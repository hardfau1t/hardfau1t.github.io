---
layout: post
title: Introduction to concurrency
---

In this series of article we will go through about concurrency(mainly focusing on threads) and synchronization by various methods like locks atomics and etc.  
Who am i? I am a embedded developer (mostly working in systems programming). At the time of writing this article I been doing systems programming for 2 years and
this will be my first article so constructive criticism is welcome.

Let us start with a simple example calculating number of prime numbers in the range of 2 to given number
```c
#include <stdbool.h>
#include <stdint.h>
#include <stdio.h>

bool is_prime(intmax_t num) {
    for (intmax_t i = 2; i <= (num / 2); i++) {
        if (num % i == 0) {
            return false;
        }
    }
    return true;
}

intmax_t count_prime(intmax_t num) {
    intmax_t ret = 0;
    for (intmax_t i = 2; i < num; i++) {
        if (is_prime(i))
            ret++;
    }
    return ret;
}

int main() {
    intmax_t N = 10;
    printf("number of primes of %ld is %ld\n", N, count_prime(N));
}
```
We get the output and we can be sure that our program works correctly  
<sub>All the code is available in [this repo]()</sub>
```
gcc -o main.elf main.c -g3
./main.elf
number of primes of 10 is 4
```
That's correct from 2-10 we have 2,3,5 and 7.  
<sub>**NOTE:** we will not be using any compiler optimizations intentionally(that can alter benchmarking results).</sub>

That's great, now lets try with timings and with even bigger number. Lets try with `N = 200000`.
```
hyperfine --show-output ./main.elf
Benchmark 1: ./main.elf
number of primes of 200000 is 17984
  Time (mean ± σ):      3.529 s ±  0.033 s    [User: 3.528 s, System: 0.001 s]
  Range (min … max):    3.505 s …  3.595 s    10 runs
```
<sub>**NOTE**: benchmarking can be inaccurate if we just run it normally due to caching, for this I will be using [hyperfine](https://github.com/sharkdp/hyperfine)
a command-line benchmarking tool which does warming and statistical analysis <sub>

That takes roughly 3.5s and the result is `17984`. From now onward we will use this number as a reference to make sure that our changes didn't break the code.

Ok that was good but can we improve this? Since we are doing this repetitive task of calculating factorial we can parallelize this.
Lets just write it using threads in simple way

