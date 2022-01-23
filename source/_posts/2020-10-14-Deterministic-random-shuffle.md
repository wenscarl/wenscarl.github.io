---
layout: post
title: Deterministic-random-shuffle
date: 2020-10-14
categories: Deep Learning
tags: 
- Randomness
- Determinism
- Tensorflow
- Deep Learning
- eager execution
---

## Motivation

Determinism is a important issue in deep learning(DL) though a lot of randomness is burried here and there, e.g. initialization of weights 

It's actually easier to understand in TF1.x language, purely graph mode. This [link](https://www.tensorflow.org/api_docs/python/tf/compat/v1/set_random_seed) is highly recommended. TF2.x introduces eager mode but still keeps graph mode option by all sorts of means, e.g. `tf.function`. It's explained [here](https://www.tensorflow.org/api_docs/python/tf/random/set_seed) but to be honest way more  counterintuitive.

This post aims to list couple of senarios may trip you down when playing around with such concepts. 

<!--more-->

## Problem statement
Determinism means, in most cases, the identical results among many runs. A common approach is `tf.random.set_seed(SEED)` at the entry point of your code. To be frank, there is a lot of implications going on under the hood, but essentially it will achieve repeatable results. I am actually assuming all the ops(on CPU and GPU) are deterministic, but in fact NOT! Check [here](https://github.com/NVIDIA/framework-determinism).

This post will focus on another case: without running the program twice, can we get a repeatable results, if a op under test is executed in a loop many times. A substantial example would be 1) identical dropout between batches or epochs; 2) same order of shuffle of input data. For simplicity, the latter will be looked at. All the codes are based on TF2.x. 

### Graph mode

```
@tf.function
def shuffled_sum(input_val, seed=123):
    print("Python run")
    tf.random.set_seed(seed)
    tensorValue1 = tf.random.shuffle(input_val, seed)
    tf.print(tensorValue1)
    tensorValue2 = tf.random.shuffle(input_val, seed)
    tf.print(tensorValue2)

shuffled_sum([1,2,3,4,5,6,7,8,9,10])  # run-1
shuffled_sum([1,2,3,4,5,6,7,8,9,10])  # run-2

## output:
Python run
[8 7 3 ... 9 1 10] ---A
[8 7 3 ... 9 1 10] ---B
[6 8 10 ... 4 1 7] ---C
[6 8 10 ... 4 1 7] ---D
```

The intention of `shuffled_sum` was to set the seed as much as it can to achieve repeatable results. But why we didn't see `A==C`? 

A: TF's builds the graph first and find that input argument `seed` is not related to `tensorValue1` and `tensorValue2`. In run-2, the global seed is not seen for the shuffle op. The internal counter for shuffle op increments. For `A==B` ,  the two ops takes in the same input args, so TF regards them as identical. 

### Eager mode

This time, we comments out `tf.funciton`,and see the output as

```
## output:
Python run
[8 7 3 ... 9 1 10] ---A
[6 8 10 ... 4 1 7] ---B
Python run
[8 7 3 ... 9 1 10] ---C
[6 8 10 ... 4 1 7] ---D
```

Now `A==C` reals the global seed works. But `A!=B` means that B is the next sequence of A given the seeds combination and it's deterministic.  In eager mode, the shuffle simply runs twice(counter increments) in the function.

### How to enforce repeatability in graph mode?

```
shuffled_sum([1,2,3,4,5,6,7,8,9,10])
tf.random.set_seed(123)
shuffled_sum([1,2,3,4,5,6,7,8,9,10])
## output:
[8 7 3 ... 9 1 10]
[8 7 3 ... 9 1 10]
[8 7 3 ... 9 1 10]
[8 7 3 ... 9 1 10]
```

'tf.random.set_seed(123)' have to be run before gets into run-2 so it cannot fall into a graph. In TF1.x way, it should comes before session.run(xxx).