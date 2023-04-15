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
<sub>All the code is available in [this repo](https://github.com/hardfau18/blog-examples/tree/main/concurrency)</sub>
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

Lets modify our code and add threads.
```c
#include <threads.h>

#define NUM_THREADS 10
intmax_t result = 0;
const intmax_t N = 200000;

...

int count_prime(void* arg) {
    long int id = (long int)arg;
    for (intmax_t i = 2 + id; i < N; i += NUM_THREADS) {
        if (is_prime(i))
            result++;
    }
    return 0;
}

int main() {
    thrd_t handles[NUM_THREADS] = {0};
    for (long int i = 0; i < NUM_THREADS; i++) {
        thrd_create(&handles[i], count_prime, (void*)i);
    }
    for (long int i = 0; i < NUM_THREADS; i++) {
        thrd_join(handles[i], NULL);
    }
    printf("number of primes of %ld is %ld\n", N, result);
}
```
in main we create 10 threads and in each thread we call `count_prime`. We divide the whole number range into equally for each thread and in each thread
we count the number of primes in its range and increment result. Then we wait for all the threads and check the results. Lets benchmark it again.
```
Benchmark 1: ./main.elf
  Time (mean ± σ):      1.104 s ±  0.021 s    [User: 4.323 s, System: 0.001 s]
  Range (min … max):    1.074 s …  1.147 s    10 runs
```
That's great we made our code ~3 times faster😃. Parallelism is wonderful and let's use it everywhere and that's done.

Is it? Lets check the output.
```
./main.elf
number of primes of 200000 is 17966
```
That's not what we expected :(. If you have noticed, you might be say that `result` should not be global and it should not be accessed by all the threads.
Then you are correct. This is due to **Data race** or **Race condition**.

**Data race** occurs when 2 or more threads tries to read and modify a shared location and here all the threads are writing to globally shared `result` variable.  
Lets take an example where 2 threads **A** and **B** are adding some value to `result` with initial value of 10.  

![Datarace](/assets/concurrency/datarace_example.png)  
In above diagram
1. In step 1 **A** reads `result` as 10 for modification.
3. Now **A** adds 5 to its copy of `result`.
2. In step 3 **B** also reads `result` as 10, and **A** haven't yet committed the changes.
4. **B** adds 8 to its local `result`.
5. **A** writes to the `result` as 15.
6. **B** writes to the `result` as 18.

Thus at the end result is 18. But that's not what we expected, we expected to be like this
![Datarace](/assets/concurrency/expected_behaviour.png)  
where **A** reads the `result` and modifies and writes back, after that **B** should have operated on `result`. A section/timeframe in which a thread accesses a shared data is called as **critical section**.
In this **critical section** only one thread is allowed to enter. For more information check this [wiki article](https://en.wikipedia.org/wiki/Race_condition#In_software)

So how should we achieve this? You might have learned that for these kind of situation we can use **locks**. When we say locks there are many kinds of locks but in this situation 
we will use **Mutexes** which allows us to access a shared region mutably during a **critical section**.
When we lock a mutex we get ownership for the given critical section and no other thread can enter while we hold the lock ( or at least should not read/write the shared data).
In early days pthread library used to provide mutexes, but after c11 we have mutexes in stdlib. Lets use that
```c
...
const intmax_t N = 200000;
mtx_t lock;
...

int count_prime(void* arg) {
    long int id = (long int)arg;
    for (intmax_t i = 2 + id; i < N; i += NUM_THREADS) {
        if (is_prime(i)){
            mtx_lock(&lock);
            result++;
            mtx_unlock(&lock);
        }
    }
    return 0;
}

int main() {
    thrd_t handles[NUM_THREADS] = {0};
    mtx_init(&lock, mtx_plain);
    for (long int i = 0; i < NUM_THREADS; i++) {
        thrd_create(&handles[i], count_prime, (void*)i);
    }
    for (long int i = 0; i < NUM_THREADS; i++) {
        thrd_join(handles[i], NULL);
    }
    printf("number of primes of %ld is %ld\n", N, result);
    mtx_destroy(&lock);
}
```
Now we initialize the mutex in main and everytime before incrementing result, we lock that section so that no one can read it and modify result and unlock.
Lets test our code now.
```
./main.elf list
number of primes of 200000 is 17984
```
Now lets benchmark our code.
```
Benchmark 1: ./main.elf
  Time (mean ± σ):      1.092 s ±  0.025 s    [User: 4.259 s, System: 0.004 s]
  Range (min … max):    1.054 s …  1.143 s    10 runs
```
Still ~3 times faster than our single threaded code🥳. But it surprises it finishes in the same time as without locks, marginally faster than without locks but that can be error in testing <sub>(benchmarking is hard🥵)</sub>.
When we have threads and they need synchronize with each other, it slows down processing. When it comes to mutexes, they can do systemcall which introduces indeterminism, for this reason realtime applications tries to avoid mutex
as much as possible.

### Conclusion
Single threaded code is safe and easy to write. But if its possible to use threads and parallelize then we could use mutexes and there are other ways for synchronization. Only thing to keep in mind that avoid sharing as much as possible, sharing is bad.
In our case why threaded code runs at the same speed even with locks that's for another day, we will look into upcoming articles. If you like these articles feel free to share your opinion.
