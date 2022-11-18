---
layout: post
title: " HTB | Reverse Engineering | behind the scenes:"
subtitle: "UD2"
date: 2022-11-17 23:45:13 -0400
background: "/img/posts/RE.jpg"
---

# HTB | Reverse Engineering | behind the scenes:

the challenge quotes:
“_After struggling to secure our secret strings for a long time, we finally figured out the solution to our problem: Make decompilation harder. It should now be impossible to figure out how our programs work!”_

We will see if that's impossible xD

So we have our executable file here.

Let’s test that out.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled.png)

Let’s now print text strings embedded in our binary.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%201.png)

Hmmm. Nothing really special other than the fact that there’s some C functions like strcmp,puts…

Let’s examine this file with ghidra.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%202.png)

After checking our main function, when clicking `invalidInstructionException()` the instruction UD2 seems to be highlighted.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%203.png)

Okay according to intel’s documentation UD2 generates an invalid opcode exception

My assumption is that the code executes fine till it bumps into this instruction and then it exits.

So as you can see there’s some instructions after UD2 let’s disassemble it.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%204.png)

select it and press **d**.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%205.png)

OK so after that if we could replace all those UD2 with NOP we can pass to next instructions.

Now do that to all those UD2 that exists out there.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%206.png)

ALRIGHT! We’re almost finished you can see the flag it’s just a concatenation of those strings.

![Untitled](/img/posts/HTB_challenges/behind_the_scenes/Untitled%207.png)

Now export the patched program and execute it,finally you got your flag.
