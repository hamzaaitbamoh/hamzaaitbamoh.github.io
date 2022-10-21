---
layout: post
title: "Exploit education phoenix | stack One"
subtitle: "Overwrite something we shouldn't"
date: 2022-10-21 23:45:13 -0400
background: "/img/posts/Pheonix.jpg"
---

# Phoenix stack 1

```c
/*
 * phoenix/stack-one, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x496c5962
 *
 * Did you hear about the kid napping at the local school?
 * It's okay, they woke up.
 *
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  if (argc < 2) {
    errx(1, "specify an argument, to be copied into the \"buffer\"");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, argv[1]);

  if (locals.changeme == 0x496c5962) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Getting closer! changeme is currently 0x%08x, we want 0x496c5962\n",
        locals.changeme);
  }

  exit(0);
}
```

we have pretty much the same code as [stack 0](https://0x0h3mz4.github.io/2022/10/19/phoenix-stack-zero.html) ,what’s been added is:

- Here we have to enter the buffer as an argument so that it would be assigned to `buffer` with the help of the function **strcpy.**
- Note that **strcpy** is vulnerable because the source size could be more than destination size.
- the if condition is checking whether the integer `changeme` is equal to `0x496c5962`

So our task here is not only to change the variable `changeme` to some random value but to specify the value to be `0x496c5962`

So we overwrite the buffer with a string which must have `0x496c5962` at the end so that it will take the value of `changeme` .

Imagine it like that:

![Untitled](/img/posts/stack1/Untitled.png)

1-_`changeme` is still 0._

![Untitled](/img/posts/stack1/Untitled%201.png)

2- We added 4 charcters “B” and let’s see if **BBBB** is assigned to `changeme`

![Untitled](/img/posts/stack1/Untitled%202.png)

So basically **BBBB** in hexadecimal is **42424242**

![Untitled](/img/posts/stack1/Untitled%203.png)

Nice!!! so all we have to do is to replace **BBBB** with `0x496c5962`

![Untitled](/img/posts/stack1/Untitled%204.png)

_you got to reverse `0x496c5962` to be `\x62\x59\x6c\x49` with this weird syntax_

Congrats!!!

![df.png](/img/posts/stack1/df.png)
