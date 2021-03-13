---
layout: post
title:  "TIL about the cpuid instruction"
date:   2021-03-10 20:37:47 -0500
categories: assembly
---

So I've been trying to learn Rust more deeply lately, and one thing I found
that I really enjoyed was Philipp Oppermann's blog series, ["Writing an OS in Rust"](https://os.phil-opp.com/).
I enjoyed it so much that I sponsored his work on GitHub and so I get the monthly
newsletter, [This Month in Rust OSDev](https://rust-osdev.com/this-month/2021-02/). This issue linked to a repo where someone
is writing a [hobby OS](https://gitlab.com/cdrzewiecki/celos) based off of the blog series. I thought this was super
cool, so I took a look at the code.

The first thing I checked out was the [bootloader code](https://gitlab.com/cdrzewiecki/celos/-/blob/master/kernel/src/arch/x86_64/boot.asm). This is the first
non-firmware code that gets run and is responsible for initializing the stack,
transitioning the processor, loading the kernel, and setting up the page table.
This code is written in x86 assembly and looked reasonably familiar, with `mov`,
`jmp`, `push`, `pull` instructions. Reading through a bit more, I came across
an instruction named `cpuid`. This stuck out to me because it wasn't a "simple"
operation like moving data into a register or jumping to a memory location.
Here's what it looked like in context:

{% highlight nasm %}
check_long_mode:
  ; test if extended processor info in available
  mov eax, 0x80000000    ; implicit argument for cpuid
  cpuid                  ; get highest supported argument
  cmp eax, 0x80000001    ; it needs to be at least 0x80000001
  jb .no_long_mode       ; if it's less, the CPU is too old for long mode

  ; use extended info to test if long mode is available
  mov eax, 0x80000001    ; argument for extended processor info
  cpuid                  ; returns various feature bits in ecx and edx
  test edx, 1 << 29      ; test if the LM-bit is set in the D-register
  jz .no_long_mode       ; If it's not set, there is no long mode
  ret
.no_long_mode:
  mov al, "2"
  jmp error
{% endhighlight %}

From the comments, it looks like it gets used to see if the processor can
transition into ["long mode"](https://en.wikipedia.org/wiki/Long_mode), which is essentially 64-bit mode. This got me
curious if there was other information that we could get from the CPU itself.
I checked out the [Wikipedia page](https://en.wikipedia.org/wiki/CPUID) for it and became even more fascinated with
all of the information that you can get out of your processor. There was even
some example assembly code:

{% highlight nasm %}
  .data

s0: .asciz  "CPUID: %x\n"

  .text

  .align  32
  .globl  main

main:
  pushq %rbp
  movq  %rsp,%rbp
  subq  $16,%rsp

  movl  $1,%eax
  cpuid

  movq  $s0,%rdi
  movl  %eax,%esi
  xorl  %eax,%eax
  call  printf
{% endhighlight %}

I copied this into `cpuid.s` and found [this Stack Overflow article](https://stackoverflow.com/questions/4288921/hello-world-using-x86-assembler-on-mac-0sx) about compiling
assembly code on MacOS (using `gcc -o cpuid cpuid.s`). Unfortunately, there were
errors! The first error was an `invalid alignment value` on the `.align 32` instruction.
I figured that it wasn't necessary, so I removed it and moved on to the next error!
That got me to `32-bit absolute addressing is not supported in 64-bit mode` on the
`movq $s0,%rdi` instruction. After a fair bit of searching, I managed to come across
the [`lea` instruction](https://code-examples.net/en/q/319865), which loads the effective address of a static value using
relative addressing. Changing the `movq` to `leaq  s0(%rip), %rdi` got it to compile
and work! Here's the output from my laptop:

```
$ gcc -o cpuid cpuid.s
$ ./cpuid
CPUID: 0x0306d4
```

I added an extra call to get the ["highest function number,"](https://en.wikipedia.org/wiki/CPUID#EAX=0:_Highest_Function_Parameter_and_Manufacturer_ID) which is what the
original bootloader code is using to see if the CPU can be transitioned from
["real mode"](https://en.wikipedia.org/wiki/Real_mode) (16-bit mode) into ["long mode"](https://en.wikipedia.org/wiki/Long_mode) (64-bit mode). Check out the
[full code](https://github.com/jasondew/cpuid) and try it for yourself!
