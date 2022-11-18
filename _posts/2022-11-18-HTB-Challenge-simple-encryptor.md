---
layout: post
title: " HTB | Reverse Engineering | Simple encryption:"
subtitle: "Decryption with some C"
date: 2022-11-18 23:45:13 -0400
background: "/img/posts/RE.jpg"
---

# Hack the box | simple encryptor:

The challenge says:

“_On our regular checkups of our secret flag storage server we found out that we were hit by ransomware! The original flag data is nowhere to be found, but luckily we not only have the encrypted file but also the encryption program itself.”_

_Ransomware is a type of malware that threatens to publish or blocks access to data or a computer system, usually by encrypting it, until the victim pays a ransom fee to the attacker._

Ok so we have to decrypt that file which is `flag.enc`

![Untitled](/img/posts/HTB_challenges/simple_encryptor/Untitled.png)

Let’s examine this encryptor with ghidra.

![Untitled](/img/posts/HTB_challenges/simple_encryptor/Untitled%201.png)

Okay so we got some interesting code.

After decompiling that main function we see the following code:

```c
undefined8 main(void)

{
  int iVar1;
  time_t tVar2;
  long in_FS_OFFSET;
  uint local_40;
  uint local_3c;
  long local_38;
  FILE *local_30;
  size_t local_28;
  void *local_20;
  FILE *local_18;
  long local_10;

  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_30 = fopen("flag","rb");
  fseek(local_30,0,2);
  local_28 = ftell(local_30);
  fseek(local_30,0,0);
  local_20 = malloc(local_28);
  fread(local_20,local_28,1,local_30);
  fclose(local_30);
  tVar2 = time((time_t *)0x0);
  local_40 = (uint)tVar2;
  srand(local_40);
  for (local_38 = 0; local_38 < (long)local_28; local_38 = local_38 + 1) {
    iVar1 = rand();
    *(byte *)((long)local_20 + local_38) = *(byte *)((long)local_20 + local_38) ^ (byte)iVar1;
    local_3c = rand();
    local_3c = local_3c & 7;
    *(byte *)((long)local_20 + local_38) =
         *(byte *)((long)local_20 + local_38) << (sbyte)local_3c |
         *(byte *)((long)local_20 + local_38) >> 8 - (sbyte)local_3c;
  }
  local_18 = fopen("flag.enc","wb");
  fwrite(&local_40,1,4,local_18);
  fwrite(local_20,1,local_28,local_18);
  fclose(local_18);
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return 0;
}
```

Let’s split the code into 3 parts:

### Part 1:

```c
local_10 = *(long *)(in_FS_OFFSET + 0x28);
  local_30 = fopen("flag","rb");
  fseek(local_30,0,2); //offset to the end
  local_28 = ftell(local_30); //put the size of the file to local_28
  fseek(local_30,0,0); //offset to the start
  local_20 = malloc(local_28); //allocate a buffer same as file size
  fread(local_20,local_28,1,local_30); //read from the file and put it to the buffer
  fclose(local_30); //close the file
  tVar2 = time((time_t *)0x0); // put the current calendar time tVar2
  local_40 = (uint)tVar2;
  srand(local_40); //seeds the random number generator used by the function rand in part 2
```

this part of the code is simply opening the file , then it reads from it and put its content to a buffer `local_20`.

### Part 2:

```c
for (local_38 = 0; local_38 < (long)local_28; local_38 = local_38 + 1) {
		// looping through the buffer character by character
    iVar1 = rand();
    *(byte *)((long)local_20 + local_38) = *(byte *)((long)local_20 + local_38) ^ (byte)iVar1;
    //xor operation with rand1
		local_3c = rand();
    local_3c = local_3c & 7;
    *(byte *)((long)local_20 + local_38) =
         *(byte *)((long)local_20 + local_38) << (sbyte)local_3c |
         *(byte *)((long)local_20 + local_38) >> 8 - (sbyte)local_3c;
		// shift left by rand2 or shift right by 8-rand2 which is a rotation left.
  }
```

it’s looping through the buffer character by character, and doing some logic operations to it , it is the encryption part.

### Part 3:

```c
local_18 = fopen("flag.enc","wb"); //open file
  fwrite(&local_40,1,4,local_18); //write 4 bytes of the seed to the file
  fwrite(local_20,1,local_28,local_18); //then it writes the buffer after encrypting it
  fclose(local_18); //close the file
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
```

Now that we understand the code, we have to reverse the whole thing:

- Open the file `flag.enc`
- Read the first 4 bytes of the file to obtain the seed
- Read what’s left of the file to obtain the encrypted part
- generate rand1 and rand2= rand1&7
- use rand2 for rotation right
- use rand1 for xor

Here’s the code to decrypt `flag.enc` :

```c
#include <stdio.h>
#include <stdlib.h>

int read_file(char *name, char **buff)
{
    FILE *file;
    int sz;

    file = fopen(name, "rb");
    fseek(file, 0, SEEK_END);
    sz = ftell(file);
    rewind(file);
    *buff = (char *)calloc(sz + 1, 1);
    fread(*buff, 1, sz, file);
    fclose(file);

    return sz;
}

int main()
{
    char *buff;
    int i, r1, r2, size;

    size = read_file("flag.enc", &buff);

    r1 = *(int *)buff;
    srand(r1);
    printf("%d\n", r1);
    for (i = 4; i < size; ++i)
    {
        r1 = rand();
        r2 = rand() & 7;
        buff[i] = (buff[i] << (8 - r2)) | ((unsigned char)buff[i] >> r2);
        buff[i] ^= r1;
    }
    printf("%s\n", &buff[4]);

    free(buff);
}
```

We’re almost finished we just have to compile the code and execute it.

![Untitled](/img/posts/HTB_challenges/simple_encryptor/Untitled%202.png)
