---
title: "Why record_stream Matters: A Real CUDA Memory Bug in SGLang"
date: 2026-02-21
categories: GPU
tags:
  - CUDA
  - PyTorch
  - GPU
  - debugging
---

## Background

PyTorch's CUDA memory caching allocator is designed to be fast — it recycles GPU memory aggressively based on Python reference counting. This works transparently most of the time, but it has a blind spot: **it only tracks the stream on which a tensor was originally allocated**.

When a tensor is passed to a *different* CUDA stream, the allocator has no idea. If Python references to that tensor drop to zero at the wrong moment, the allocator can reclaim the memory while the second stream's kernel is still reading it. The result is a silent use-after-free on the GPU: corrupted data, out-of-bounds crashes, or worse — wrong answers with no error at all.

This exact bug was recently caught in [SGLang's speculative decoding (EAGLE v2)](https://github.com/sgl-project/sglang/issues/18744), and fixed with a single call to `record_stream`.

---

## The Bug: Index Out of Bounds in Speculative Decoding

### Symptom

SGLang users running DeepSeek-R1 on 8× B200/GB200 GPUs with speculative decoding (EAGLE, 2 steps) at high concurrency (≥ 1024) saw intermittent crashes like:

```
vectorized_gather_kernel: block: [0,1,0], thread: [64,0,0]
  Assertion `ind >= 0 && ind < ind_dim_size` failed

IndexKernel.cu:111: operator():
  Assertion `-sizes[i] <= index && index < sizes[i]` failed
```

The indices tensor used to gather from `topk_p_buf`, `topk_index_buf`, `hidden_states_buf`, etc. contained garbage — both negative values and values far exceeding the buffer size.

### Root Cause

The race condition unfolds in this sequence:

1. An `indices` tensor is **allocated on PyTorch's default stream**.
2. The object holding that tensor (`spec_info`) is **replaced** — `batch.spec_info` is assigned a new value, dropping the last Python reference to the old one.
3. PyTorch's caching allocator sees refcount → 0 and **reclaims the GPU memory**.
4. Meanwhile, the **forward/compute stream** enqueues kernels that read from those same indices.
5. The forward stream reads from memory that has already been freed and partially overwritten → **corrupted indices → out-of-bounds crash**.

The key insight: steps 3 and 4 can race because they happen on *different streams* with no synchronization between them.

### The Fix ([PR #18958](https://github.com/sgl-project/sglang/pull/18958))

```python
# Before (broken)
draft_input.topk_p = self.topk_p_buf[draft_input.future_indices.indices]

# After (fixed)
indices = draft_input.future_indices.indices
indices.record_stream(torch.get_device_module(self.device).current_stream())
draft_input.topk_p = self.topk_p_buf[indices]
```

One line. `indices.record_stream(stream)` tells the caching allocator: "the current stream is also using this tensor — don't free it until this stream has passed this point." The race is eliminated.

---

## What `record_stream` Does

```python
tensor.record_stream(stream)
```

Registers `stream` as a consumer of `tensor`'s underlying GPU memory. The caching allocator will not reuse that memory until **all enqueued operations on `stream`** at the time of the call have completed.

**When you need it:** any time a tensor is:
- allocated (or last used) on stream A
- then handed off to stream B for further GPU work
- and Python references may drop to zero before stream B finishes

**When you don't need it:** if you hold a Python reference to the tensor until after stream B is done (e.g., you explicitly synchronize before releasing it), the reference itself keeps the memory alive.

---

## Minimal Reproducer

<!-- PASTE YOUR REPRODUCER CODE HERE -->

```python
import gc
import torch

DEVICE = "cuda:0"
N = 32 * 1024 * 1024  # 128 MB


def buggy():
    side = torch.cuda.Stream(DEVICE)

    # ① Source tensor born on the DEFAULT stream.
    src = torch.full((N,), 1.0, dtype=torch.float32, device=DEVICE)
    torch.cuda.synchronize()

    out = torch.empty(N, dtype=torch.float32, device=DEVICE)
    with torch.cuda.stream(side):
        # ② Burn GPU cycles on the side stream to delay the copy,
        #    giving the default stream time to recycle src's memory.
        delay = torch.zeros(4096, 4096, device=DEVICE)
        for _ in range(20):
            delay = torch.mm(delay + 1e-6, delay + 1e-6)
        out.copy_(src)
        # ★ KEY FIX: tell the allocator src's memory is live on `side`.
        #   The block will not be recycled until all work queued on `side`
        #   up to this point (including the copy) has completed.
        #   Both streams continue running concurrently — no serialization.
        # src.record_stream(side)

    # ③ Python ref dropped. Allocator marks the block free for the default
    #    stream — unaware that the side stream will read it after the matmuls.
    del src
    gc.collect()

    # ④ Same-size allocation on the DEFAULT stream gets the recycled block
    #    and overwrites it with 2.0 while side stream is still in matmuls.
    poison = torch.full((N,), 2.0, dtype=torch.float32, device=DEVICE)

    # Sync only to observe the result — the corruption already happened above.
    torch.cuda.synchronize()
    side.synchronize()

    got = out[0].item()
    print(f"[buggy ] expected=1.0  got={got:.1f}  "
          f"→ {'CORRUPTED' if got != 1.0 else 'OK (lucky this run)'}")
    del poison, out


BUF_SIZE = 32 * 1024 * 1024

def not_succesful_buggy():
    side = torch.cuda.Stream(DEVICE)

    buf = torch.arange(BUF_SIZE, dtype=torch.float32, device=DEVICE)  # buf[i] == i
    gate = torch.cuda.Event()

    # ① Valid indices on DEFAULT stream: values in [0, BUF_SIZE).
    indices = torch.randint(0, BUF_SIZE, (N,), dtype=torch.int64, device=DEVICE)
    torch.cuda.synchronize()

    with torch.cuda.stream(side):
        # ② Delay gather so default stream can lap it.
        delay = torch.zeros(4096, 4096, device=DEVICE)
        for _ in range(20):
            delay = torch.mm(delay + 1e-6, delay + 1e-6)
        side.wait_event(gate)
        out = buf[indices]   # gather — reads indices after matmuls

    indices_ptr = indices.data_ptr()
    del indices
    gc.collect()

    # ③ Recycle with OOB values — same dtype, same count.
    poison = torch.full((N,), BUF_SIZE + 9999, dtype=torch.int64, device=DEVICE)
    assert poison.data_ptr() == indices_ptr, "allocator did not reuse block"

    gate.record()
    torch.cuda.synchronize()  # poison write finishes
    side.synchronize()        # gather runs — reads OOB indices

    # With TORCH_USE_CUDA_DSA=1 + CUDA_LAUNCH_BLOCKING=1: assertion crash above.
    # Without: silent corruption — output should equal valid gathered values
    #          but instead reflects OOB reads (garbage or wrapped values).
    got = out[0].item()
    expected = out[0].item()  # meaningless since buf[i]==i, just show it's garbage
    print(f"[buggy ] out[0]={got:.1f}  (if corrupted, will not match any valid buf value)")
    del poison, out, buf


if __name__ == "__main__":
    assert torch.cuda.is_available(), "CUDA required"
    buggy()
    not_succesful_buggy()
```
CUDA_LAUNCH_BLOCKING=1 python show_bug.py
Hides the bug;

python show_bug.py
will display the difference. And src.record_stream(side) fixes it. Not sure why another cannot reproduce.

---

## Key Takeaways

- **The PyTorch caching allocator tracks allocation stream, not usage stream.** Cross-stream tensor handoffs without `record_stream` are a latent use-after-free.
- **The bug is silent until it isn't.** At low concurrency or on slower hardware, the race window may never open. It took 8× B200s at 1024 concurrent requests to expose it reliably.
- **The fix is cheap.** `record_stream` costs essentially nothing — it just registers a stream with the allocator's internal bookkeeping.
- **Speculative decoding amplifies the risk.** SGLang's overlap scheduler runs prefill and decode on separate streams concurrently, and `spec_info` objects have short, non-obvious lifetimes — a perfect storm for this class of bug.

---

## References

- [SGLang issue #18744](https://github.com/sgl-project/sglang/issues/18744) — original bug report
- [SGLang PR #18958](https://github.com/sgl-project/sglang/pull/18958) — the fix
- [PyTorch docs: `Tensor.record_stream`](https://pytorch.org/docs/stable/generated/torch.Tensor.record_stream.html)
- [PyTorch CUDA memory management](https://pytorch.org/docs/stable/notes/cuda.html#memory-management)
