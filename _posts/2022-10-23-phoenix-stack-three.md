---
layout: post
title: "Exploit education phoenix | stack Three"
subtitle: "Overwrite function pointers stored on the stack"
date: 2022-10-23 23:45:13 -0400
background: "/img/posts/Pheonix.jpg"
---

# Phoenix stack 3

```c
/*
 * phoenix/stack-three, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable to 0x0d0a090a
 *
 * When does a joke become a dad joke?
 *   When it becomes apparent.
 *   When it's fully groan up.
 *
 */

#include <err.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

void complete_level() {
  printf("Congratulations, you've finished " LEVELNAME " :-) Well done!\n");
  exit(0);
}

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int (*fp)();
  } locals;

  printf("%s\n", BANNER);

  locals.fp = NULL;
  gets(locals.buffer);

  if (locals.fp) {
    printf("calling function pointer @ %p\n", locals.fp);
    fflush(stdout);
    locals.fp();
  } else {
    printf("function pointer remains unmodified :~( better luck next time!\n");
  }

  exit(0);
}
```

_In C, like normal data pointers (int _, char _, etc), we can have pointers to functions as well._

Our challenge here is to make `fp` points to the function `complete_level`

Let’s run the code and see

![Untitled](/img/posts/stack3/Untitled.png)

Ok now we need to retrieve the address memory of the function `complete_level`

Let’s disassemble the main function with gdb

![Untitled](/img/posts/stack3/Untitled%201.png)

cool! we can understand the pattern of the code just by following those assembly instruction.

But nothing here special about the wanted function `complete_level`

We can otherwise just type print to retrieve the address.

![Untitled](/img/posts/stack3/Untitled%202.png)

Cool cool! Now that we have the address we can simply execute the same attack we did on the previous stacks.

![Untitled](/img/posts/stack3/Untitled%203.png)

Great job!

![df.png](/img/posts/stack3/df.png)
