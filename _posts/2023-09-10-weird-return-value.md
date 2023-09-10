---
layout: post
title: A tale of a weird return value
tags: C assembly cltq gdb debugging
---

It is funny how everyone lives at their own pace, but we often go through similar problems and experiences. I guess that's why we frequently hear parents say: "I don't want you to make the same mistakes." "What is happening now has happened before..."

This week, a friend of mine approached me because he was having difficulty figuring out why a return value was being *corrupted* in a `return` statement of a C program.

The source code looked something like this (this is obviously a super simplified version). In a C file:

```c
int main(void)
{
  void *my_value = someFunction();

  printf("[%s, %s:%d] The received return value is: %p\n",
      __func__, __FILE__, __LINE__, my_value);

  return 0;
}
```

and in another C file:

```c
void *someFunction(void)
{
  void *my_return_value = (void *)0xBADC0FFEE0DDF00D;

  printf("[%s, %s:%d] My return value is: %p\n",
      __func__, __FILE__, __LINE__, my_return_value);

  return my_return_value;
}
```

(You can download the code from [here](https://gist.github.com/rzavalet/8010f7cc0ba0cbfe20b396f3cbe814bd))

The result was something like:

```console
> ./main 
[someFunction, another_compilation_unit.c:15] My return value is: 0xbadc0ffee0ddf00d
[main, main.c:16] The received return value is: 0xffffffffe0ddf00d
```

Notice that the low bits are good, but the high bits are all `1`s. 

The confusing part was that at the line containing the `return` statement inside  `someFunction()`, the return value looked ok, i.e., we had the expected value `0xbadc0ffee0ddf00d`. However, stepping out of `someFunction()` and back in `main()`, the value was becoming: `0xffffffffe0ddf00d`. The following debugging session shows it:

```console
Breakpoint 2, someFunction () at another_compilation_unit.c:12
12        void *my_return_value = 0xBADC0FFEE0DDF00D;
(gdb) n
14        printf("[%s, %s:%d] My return value is: %p\n",
(gdb) n
[someFunction, another_compilation_unit.c:15] My return value is: 0xbadc0ffee0ddf00d
17        return my_return_value;
(gdb) print my_return_value
$1 = (void *) 0xbadc0ffee0ddf00d
(gdb) n
18      }
(gdb) n
main () at main.c:14
14        printf("[%s, %s:%d] The received return value is: %p\n",
(gdb) print my_value
$2 = (void *) 0xffffffffe0ddf00d
(gdb)
```

After confirming a few things (e.g., the code was recompiled after the last modification, the types are indeed the same, etc.) I told him I'd probably see what the assembly code looks like. 

Fortunately, the problem was reproducing consistently. So, we spun up `gdb` again, used `layout split` and began executing the instructions.

We saw that before returning from `someFunction()`, the `rax` register contained the expected value:

![Step 1](/assets/images/before_cltq.png)

Similarly, right after returning to `main()`:

![Step 2](/assets/images/before_cltq_2.png)

But then there was a `cltq` instruction, and after executing it... 

![Step 3](/assets/images/after_cltq.png)

:boom::boom::boom: the value changed!

A quick google search showed that `cltq` sign-extends a 32-bit integer to a 64 one. Aha!, that should be it! It looked like, for some reason, the compiler was taking the return value as if it was a 32-bit int. It was taking the value:

```console
(gdb) p/x (int) 0xbadc0ffee0ddf00d
$3 = 0xe0ddf00d
...
(gdb) p/t (int) 0xbadc0ffee0ddf00d
$5 = 11100000110111011111000000001101
```

and it was extending the MSB to fill the 64-bits:

```console
(gdb) p/t 0xffffffffe0ddf00d
$6 = 1111111111111111111111111111111111100000110111011111000000001101
```

Unfortunately, my friend had to move to something else at that point, but he said he was to investigate more.

 The next day, he said he figured out the problem after googling some more. He found this [blog post](https://joshpeterson.github.io/the-curious-case-of-cltq) where the author goes through a similar case. Same experience... but 7 years before :sweat_smile: "What is happening now has happened before..."


The lesson learned for my friend and me is: 

:exclamation::exclamation::exclamation:**Do not just ignore compilation warnings**:exclamation::exclamation::exclamation:

Because the code he was working with already had other warnings, he didn't notice that the compiler was already complaining about the problem:

```console
> gcc -ggdb  -o main main.c another_compilation_unit.c
main.c: In function ‘main’:
main.c:13:20: warning: implicit declaration of function ‘someFunction’ [-Wimplicit-function-declaration]
   13 |   void *my_value = someFunction();
      |                    ^~~~~~~~~~~~
main.c:13:20: warning: initialization of ‘void *’ from ‘int’ makes pointer from integer without a cast [-Wint-conversion]
```

The problem was fixed by declaring the function prototype before using it, e.g. including the proper `.h` file:

```c
#include <stdio.h>
#include "another_compilation_unit.h"

int main(void)
{
  void *my_value = someFunction();

  printf("[%s, %s:%d] The received return value is: %p\n",
      __func__, __FILE__, __LINE__, my_value);

  return 0;
}
```

```console
> ./main
[someFunction, another_compilation_unit.c:15] My return value is: 0xbadc0ffee0ddf00d
[main, main.c:16] The received return value is: 0xbadc0ffee0ddf00d
```

As stated before, other people will likely run into the same problem in the future, regardless of how much we and the compiler warn them. Everyone has their own time.
