---
layout: post
title: "Exploit education phoenix | stack Two"
subtitle: "Overwrite something we shouldn't"
date: 2022-10-22 23:45:13 -0400
background: "/img/posts/Pheonix.jpg"
---

# Phoenix stack 2

```c
/*
 * phoenix/stack-two, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x0d0a090a
 *
 * If you're Russian to get to the bath room, and you are Finnish when you get
 * out, what are you when you are in the bath room?
 *
 * European!
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to 0x0h3mz4 brought to you by https://exploit.education"

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  char *ptr;

  printf("%s\n", BANNER);

  ptr = getenv("ExploitEducation");
  if (ptr == NULL) {
    errx(1, "please set the ExploitEducation environment variable");
  }

  locals.changeme = 0;
  strcpy(locals.buffer, ptr);

  if (locals.changeme == 0x0d0a090a) {
    puts("Well done, you have successfully set changeme to the correct value");
  } else {
    printf("Almost! changeme is currently 0x%08x, we want 0x0d0a090a\n",
        locals.changeme);
  }

  exit(0);
}
```

Equally to [stack1](https://0x0h3mz4.github.io/2022/10/22/phoenix-stack-one.html) except that an environment variable must be set.

Initially, let’s run the program without setting the environment variable:

![Untitled](/img/posts/stack2/Untitled.png)

Then Let’s set that sucker with some python scripting!

![Untitled](/img/posts/stack2/Untitled%201.png)

Let’s print that sucker

![Untitled](/img/posts/stack2/Untitled%202.png)

Tbh I don’t know if `0x0d0a090a` has been added at the end of those As, since I don’t want to convert `0x0d0a090a` to a string to make sure.

Let’s test that anyways 😅

![Untitled](/img/posts/stack2/Untitled%203.png)

Congrats, it has been set!!!

![df.png](/img/posts/stack2/df.png)
