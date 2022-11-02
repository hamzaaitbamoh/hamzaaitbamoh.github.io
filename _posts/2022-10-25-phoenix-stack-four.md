---
layout: post
title: "Exploit education phoenix | stack Four"
subtitle: "Overwrite the saved instruction pointer"
date: 2022-10-25 23:45:13 -0400
background: "/img/posts/Pheonix.jpg"
---

# Phoenix stack 4

```c
/*
 * phoenix/stack-four, by https://exploit.education
 *
 * The aim is to execute the function complete_level by modifying the
 * saved return address, and pointing it to the complete_level() function.
 *
 * Why were the apple and orange all alone? Because the bananna split.
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

void start_level() {
  char buffer[64];
  void *ret;

  gets(buffer);

  ret = __builtin_return_address(0);
  printf("and will be returning to %p\n", ret);
}

int main(int argc, char **argv) {
  printf("%s\n", BANNER);
  start_level();
}
```

Things are getting fun! but before getting into the fun part, I need to tell you a secret about the `rip` , `rsp` and `rbp` registers.

When I disassemble main using gdb:

![Untitled](/img/posts/stack4/Untitled.png)

the instruction highlighted **call** is not only calling **start_level** , it is in fact pushing the `rip` register of the next instruction into the stack then it moves to execute those instructions inside **start_level**.

In that case: 0x40068d will be pushed into the stack then above it `rbp` will be pushed, move `rsp` into `rbp` and so on…

![Untitled](/img/posts/stack4/Untitled%201.png)

The reason why it does this: is because at the end when we hit **leave,** `rsp` will be set to `rbp` and then `rbp` will be popped out of the stack which means the frame stack of **start_level** will be popped out.

![Untitled](/img/posts/stack4/Untitled%202.png)

![Untitled](/img/posts/stack4/Untitled%203.png)

At the end **ret** will be having the old saved `rip` which is 0x40068d (in main) and popped it out from the stack so that it could be executed.

Now that we know that, we can see that we have to overwrite to the ret value to be the **complete_level** function adress.

Let’s get to work!

Ok, now we’ll make breakpoint right after reading the input with **gets** function.

And then I’m going to enter a buffer of 64 characters, I am going to write different alphabets to visualize on the stack by examining the `rsp` register

![Untitled](/img/posts/stack4/Untitled%204.png)

![Untitled](/img/posts/stack4/Untitled%205.png)

We can see now the buffer, and as we said the buffer is some instruction of the **start_level** that’s popped into the stack above a saved `rip` which is **0x40068d**

This address is what the program will be returning to after executing **start_level** but we want to return to the address of **complete_level** so we will overwrite to replace **0x40068d** by **0x40061d** (which is the adress of complete_level)

![Untitled](/img/posts/stack4/Untitled%206.png)

So we can simply count how many characters we should overwrite to reach **0x40068d**

⇒ 24 more characters which means we need a buffer of 88 (64+24=88)

![Untitled](/img/posts/stack4/Untitled%207.png)

simply run this python script and pipe it into the binary:

![Untitled](/img/posts/stack4/Untitled%208.png)

That’s it for this level.

![df.png](/img/posts/stack4/df.png)
