---
title: "Properly Casting in C"
date: 2022-08-18T19:25:19-04:00
tags: [C, "Software Design", "Writing Good C"]
draft: true
---

I believe the most common mistake I see in both new and experienced users of C
is incorrect or dangerous integer casting. It appears to me that these troubled
developers are equating casting to conversion without realizing they're
introducing bugs into their software. I'm unsure if this is because of lack of
knowledge, laziness, or forgetfulness due to working in higher-level languages,
but whatever the reason I see it often (and I'm guilty of it too!).

## The problem

Let's look at an example to get started.

```c
#include <stdio.h>
#include <stdint.h>
#include <limits.h>

uint16_t a = 100;
int16_t b = (int16_t)a;
printf("a = %u b = %d\n", a, b);
// a = 100 b = 100
```

The above does what we expect. We're casting an unsigned to a signed, but
since the number is still within the bounds of the signed value, it works as
expected.

```c
uint16_t a = INT16_MAX;
int16_t b = (int16_t)a;
printf("a = %u b = %d\n", a, b);
// a = 32767 b = 32767
```

Even with the `INT16_MAX` value, the cast still works fine, no bugs here, but
what happens if we add `1` to that number?

```c
uint16_t a = INT16_MAX + 1;
int16_t b = (int16_t)a;
printf("a = %u b = %d\n", a, b);
// a = 32768 b = -32768
```

Wow, it goes negative! Why is that happening if `a` was a positive number? Most
people I question about this figure a cast like this would just clamp `a` to
`b`'s highest positive value, but this is clearly not true. This is the major
(and bug-inducing) difference between a cast and a conversion.

A cast is not elegant. There is no validation being done for you (and why would
it? Since when has C ever done anything for you?). A cast literally chops bits
off and slams them into a box. A cast is when have your friend sit on your
suitcase, so you can jam in one more outfit for your trip -- it'll close, but
you might end up with a shirt cut up by the zipper.

Let's [look at these numbers bit-by-bit](https://bitwisecmd.com/). These
numbers are represented in their unsigned form.

```plaintext
unsigned        bits          hex
-----------------------------------
100       00000000 01100100  0x0064
32767     01111111 11111111  0x7fff
32768     10000000 00000000  0x8000
```

And now let's annotate our source with what's being set in the actual
registers.

```c
uint16_t a = INT16_MAX + 1;  // a = 10000000 00000000
int16_t b = (int16_t)a;      // b = 10000000 00000000
printf("a = %u b = %d\n", a, b);
// a = 32768 b = -32768
```

What we're seeing here is because the MSB of `32768` is `1` this tells the
computer this is actually a negative number (as long as it's treated as signed
which `printf` is doing). This system of storing negative numbers is called
[two's complement](https://en.wikipedia.org/wiki/Two%27s_complement). The
takeaway here is a cast does not care if it's going from unsigned to signed,
nor does it care if it's going from a wider container to a narrower one.

Casting is not only dangerous when casting from unsigned to signed, it can be
dangerous when casting from a wider container to a narrower one. Here's an
example which shows casting is like a guillotine, and we lose data permanently.

```c
int16_t a = -32768;   // a = 10000000 00000000
int8_t b = (int8_t)a; // b =          00000000
printf("a = %d b = %d\n", a, b);
// a = -32768 b = 0
```

```c
int16_t a = -32768 + 20; // a = 10000000 00010100
int8_t b = (int8_t)a;    // b =          00010100
printf("a = %d b = %d\n", a, b);
// a = -32748 b = 20
```

I have a real-world example which I have seen too many times. The problem
is mostly Microsoft's fault for creating a POSIX named function which is not
POSIX compliant.

POSIX declares the `write()` function as such:

```c
// POSIX
ssize_t write(int fd, const void *buf, size_t count);
```

And Microsoft's implementation is:

```c
// Microsoft
int write(int fd, const void *buffer, unsigned int count);
```

The problematic usage I often see is in usages like this:

```c
  // defined above
  size_t count;
  ssize_t bytes_written;

#ifdef _WIN32
  bytes_written = (ssize_t)write(fd, buffer, (unsigned int)count);
#else
  bytes_written = write(fd, buffer, count);
#endif
```

We can see that there's a very real chance `count` will be bigger than
the max value of `unsigned int`. On both 32 and 64-bit Windows, `int` is
32-bits wide. On 32-bit Windows `size_t` is 32-bits wide and on 64-bit Windows
`size_t` is 64-bits wide. So this incorrect cast will not only lead to
interoperability between Windows and Linux but also between 32/64-bit Windows.

## The solution

The steps to perform proper conversion are not magic, they're exactly what you
may expect. Basically all you really need is bounds checking. The resulting
code will be exhaustive, but that's what C is.

1. When converting from signed to signed or unsigned to unsigned, simply ensure
  the receiving container is of the same width or larger. If the resulting
  container is smaller then check for out-of-bounds input values and clamp
  or error depending on your situation.
1. what




**TODO: Show the correct way**




http://tzimmermann.org/2018/04/20/safe-integer-conversion-in-c/

C is hard

Overflow and underflow
