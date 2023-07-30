+++
title = "Demystifying pointers arrays and functions"
date = 2023-07-31

draft = true

[ taxonomies]
categories = ["c"]
tags = ["c", "pointers"]
+++

In this article we go through whats a pointer, how it differs from arrays and function pointer.  
Highly recommend [this](http://www.unixwiz.net/techtips/reading-cdecl.html "Reading C type declarations") article,
which deals with understanding  complex type declaration.

In C pointer is a type of variable which can point to some area in the memory. For example
```c
    int a = 0;
    int *ptr = &a;
```

Here `ptr` will be pointing to `a`.

That was simple. Lets take another example
```c
    int arr[4] = { 3, 2, 1, 0};
    int *ptr1 = arr;
    int (*ptr1)[] = arr;

```

<!---
# can't create int foo [] =  arr;
# int* != int []
-->
