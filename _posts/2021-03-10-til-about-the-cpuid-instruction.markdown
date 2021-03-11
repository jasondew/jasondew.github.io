---
layout: post
title:  "TIL about the cpuid instruction"
date:   2021-03-10 20:37:47 -0500
categories: assembly
---

Earlier, I was reading through [This Month in Rust OSDev](https://rust-osdev.com/this-month/2021-02/)
and came across someone who was writing a [hobby OS](https://gitlab.com/cdrzewiecki/celos), so I had to
take a peek at the code. The first thing I looked at was the [bootloader code](https://gitlab.com/cdrzewiecki/celos/-/blob/master/kernel/src/arch/x86_64/boot.asm)
and I saw all the usual assembly instructions (`mov`, `jmp`, etc) but while
reading the code that transitions the processor into "long mode" (64-bit mode),
I noticed an instruction that didn't look like the others: `cpuid`. Here's the,
very nicely documented, code I was looking at:

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

I was intrigued, so I checked out the [Wikipedia page](https://en.wikipedia.org/wiki/CPUID) and became even more
fascinated. It turns out that most processors can return lots of information
about their capabilities and that you can write some pretty simple assembly
code to query them. There was even some sample code:

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

Unfortunately, this code doesn't work on MacOS! At this point, I was all in,
so I decided I was going to make it work.

I found out that I could use `gcc`
to try and compile it, so I copied it into `cpuid.s` and gave it a go with:
`gcc -o cpuid cpuid.s`. Unfortunately, that gave an error:
`invalid alignment value` on the `.align 32` instruction. I figured that it
wasn't necessary, so I removed it and moved on to the next error:
`32-bit absolute addressing is not supported in 64-bit mode` on `movq $s0,%rdi`.
That one took a bit more to figure out. After a fair bit of searching, I
managed to come across [the `lea` instruction](https://code-examples.net/en/q/319865), which loads the effective
address of a static value using relative addressing. So I changed the `movq`
to `leaq  s0(%rip), %rdi` and it worked! Here's the output from my laptop:

```
$ gcc -o cpuid cpuid.s
$ ./cpuid
CPUID: 0x0306d4
```

I added an extra call to get the "highest function number," which is used
to see if a CPU can go into long mode, just for fun. Check out the [full code](https://github.com/jasondew/cpuid)
and try it for yourself!
