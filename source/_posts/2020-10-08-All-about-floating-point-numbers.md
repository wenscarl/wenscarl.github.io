---
layout: post
title: All about floating-point numbers
date: 2020-10-02 20:57:08
tags: Floating-point
categories: Math
---
### How Floating-point number are represented in IEEE-754 standard
![IEEE754FP][2]
The maximal and minimal number 

<!--more-->

| FP | maximal     |minimal|
|----|-------|---------|
| 32 | 1.11111111111111111111111*2^127^=3.4028234663852886e038|1.0000 0000 0000 0000 0000 001*2^-126^= 1.1754944909521339e-038 |
|64|1.7976931348623157e308|2.2250738585072014e−308|
For details please check [this][3]. Never forget the exponent bits cannot be all 1s, otherwise it's regarded as NaN or Inf. Also note the difference between subnormal double and [normal double][4].
###  Some facts

```bash
$ python
>>> 0.1+0.2
0.30000000000000004
>>> e = 0.5**24
>>> a,b,c,d=1+d,-1-d,1-d,-1+d
>>> print(a+c+b+d, a+b+c+d)
>>> 0.0 0.0
>>> a1=np.float32(a); b1=np.float32(b); c1=np.float32(c); d1=np.float32(d)
>>> print(a1+c1+b1+d1, a1+b1+c1+d1)
>>> 5.9604645e-08 0.0
>>> print("%.26f, %.26f, %.26f, %.26f" %(a,b,c,d))
1.00000005960464477539062500, -1.00000005960464477539062500, 0.99999994039535522460937500, -0.99999994039535522460937500
>>> print("%.26f, %.26f, %.26f, %.26f" % (a1,b1,c1,d1))
>>> 1.00000000000000000000000000, -1.00000000000000000000000000, 0.99999994039535522460937500, -0.99999994039535522460937500
```

It's a known fact that associativity of FP's addition does NOT hold and that's why we see `a1+c1+b1+d1`, `a1+b1+c1+d1` give distinct results. Another important fact is the rounding from FP64 to FP32 behaves differently between `1+e` and `1-e`. Clearly, rounding for `1-e` is more accurate than another.  It can be explained by following table.

----------------------------------------
```bash
case: 1-e
  FP64 representation:
    _decimal: 0.9999999403954
    ____sign: 0
    exponent: 01111111110
    mantissa: 1111111111111111111111100000000000000000000000000000
  FP32 representation:
    _decimal: 0.9999999403954
    ____sign: 0
    exponent: 01111110
    mantissa: 11111111111111111111111
case: 1+e
  FP64 representation:
    _decimal: 1.0000000596046
    ____sign: 0
    exponent: 01111111111
    mantissa: 0000000000000000000000010000000000000000000000000000
  FP32 representation:
    _decimal: 1.0000000000000
    ____sign: 0
    exponent: 01111111
    mantissa: 00000000000000000000000
```
----------------------------------------

At rounding stage, the last 29 bits are rounded based on whether greater than `10...0`(29 bits). It's not hard to infer that the smallest(in abs value) number FP32 can represent without rounding error is `2^-24`. Likewise, `2^-53` for FP64. 

### Largest continuously representable integer

Precision is solely determined by number of bits for mantissa. Assume the exponent is 23, let's look at number

`a = 1.11...1*2^23`, here there are 23 bits of 1s followed by decimal. Any positive integer less than a is precisely representable in FP32. The next integer on real axis is `b=a+1` and don't worry about exponent for this moment, since it can goes way beyond 23. The mantissa for 1 is all 0s. Thus `b=1.00..0*2^24=(16777216)~10`. For the next one in binary sense, we have `c=1.00..1*2^24`,  but unfortunately this is number 16777218 where 16777217 is missing. The same induction can be carried through and not hard to imagine that there leaves more blank spaces when numbers go large.

```bash
>>> np.float32(2**24+1)
>>> 16777216.0
>>> np.float32(2**24+2)
>>> 16777218.0
```
As summary, by IEEE754 standard, the integer range which can be represented accurately is 
| FP | Range        |
|----|--------------|
| 32 | [-2^24^,2^24^] |
| 64 | [-2^53^,2^53^] |

### Why 0.1+0.2 != 0.3
We should by far aware that not all FP numbers can be represented accurately, e.g.1.25 can but 3.1415926... can not. 0.1 and 0.2 happen to sit between two representable numbers in FP64. So the sum is not a perfect 0.3.  

### Conclusion
If you're still thirsty on this topic, please check this [blog][1].

[1]: https://www.juliensobczak.com/inspect/2019/03/10/floating-point-numbers-demystified.html
[2]: /images/UnderstandingFloatingpointnumberrepresentationslide_14.jpg
[3]: https://blog.csdn.net/whzhaochao/article/details/12887779
[4]: https://en.wikipedia.org/wiki/Double-precision_floating-point_format