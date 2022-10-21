---
layout: post
title: "Exploit education phoenix | stack Zero"
subtitle: "Overwrite something we shouldn't"
date: 2022-10-20 23:45:13 -0400
background: "/img/posts/Pheonix.jpg"
---

# Phoenix exploit education

# **Stack 0:**

```c
/*
 * phoenix/stack-zero, by https://exploit.education
 *
 * The aim is to change the contents of the changeme variable.
 *
 * Scientists have recently discovered a previously unknown species of
 * kangaroos, approximately in the middle of Western Australia. These
 * kangaroos are remarkable, as their insanely powerful hind legs give them
 * the ability to jump higher than a one story house (which is approximately
 * 15 feet, or 4.5 metres), simply because houses can't can't jump.
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define BANNER \
  "Welcome to " LEVELNAME ", brought to you by https://exploit.education"

char *gets(char *);

int main(int argc, char **argv) {
  struct {
    char buffer[64];
    volatile int changeme;
  } locals;

  printf("%s\n", BANNER);

  locals.changeme = 0;
  gets(locals.buffer);

  if (locals.changeme != 0) {
    puts("Well done, the 'changeme' variable has been changed!");
  } else {
    puts(
        "Uh oh, 'changeme' has not yet been changed. Would you like to try "
        "again?");
  }

  exit(0);
}
```

So basically we have a struct composed of two variables:

- a string named `buffer` that has 64 in its length
- an integer named `changeme`

The code gives us an input to enter which will be assign to the string `buffer` , with that being said we are in control of the variable `buffer` as a user.

then there is a condition that checks if `changeme` is changed to something other than 0.

Our task in this challenge is to simply change the variable `changeme`

![My Image](/img/posts/stack0/Untitled.png)

Above I displayed the stack, and how it stores local variables.

which means that the string `buffer` is above `changeme`

![Untitled](/img/posts/stack0/Untitled%201.png)

Therefore if we could manage to write on the string `buffer` some string longer than 64 characters we can overwrite it and thus we will be writing on `changeme` because it’s bellow it 😎

ok let’s try it with a string of 64 characters:

![Untitled](/img/posts/stack0/Untitled%202.png)

And let’s now add one more:

![Untitled](/img/posts/stack0/Untitled%203.png)

Congrats !!! we changed the `changeme` variable.

![Untitled](/img/posts/stack0/Untitled%204.png)
