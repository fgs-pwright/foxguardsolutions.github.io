---
layout: post
title: Modern Buffer Overflows (Part I)
author: Austin Howard
---

This post will focus on a deep-dive of the problem domain, and show how a vulnerable program can be made to overwrite important data. Once the problem has been explained in detail, there will be a sample program to demonstrate everything covered which will be built for both 32-bit and 64-bit systems to explore the differences. Follow-up posts will expand on this knowledge by exploring: writing practical shellcode from hand, exploitation differences between 32-bit and 64-bit, building exploits for modern systems with security protections turned on, and how these vulnerabilities still show up in the wild today.


## Understanding The Problem ##

In a nutshell, the problem can be described as attempting to write too much data into too small of a buffer. That is a very high level view, but an assembly-level view of what happens during variable allocation and manipulation, and function calls will be required to fully understand the problem. Some sample C code, the assembly generated by that code, and a walk-through of the stack during execution will be provided to aid in the explanations. All code is compiled using the GNU C Compiler (gcc) with the AT&T assembly syntax on a Linux virtual machine.

### What is (on) the Stack? ###

In C and similar languages the stack is where all local variables will be stored. There are other values stored on the stack during function calls that will be crucial to understanding buffer overflows and protections against them.

At its simplest, the stack is a contiguous section of memory that respects a FILO (First In Last Out) ordering. As more variables are created, more memory is reserved on the top of the stack for those variables; likewise, when variables are destroyed their memory on the stack is freed. This is most commonly referred to as pushing (adding more) to the stack and popping (removing) off the stack. In a high-level language like C, stack management is left up to the compiler.

At the assembly level on x86(-64) systems there are two registers responsible for managing the current function's stack frame: 

 * `%ebp` (`%rbp` for 64-bit) - Known as the Base Pointer. Always points to the bottom of the stack (the first thing pushed onto the stack). 
 * `%esp` (`%rsp` for 64-bit) - Known as the Stack Pointer. Always points to the top of the stack (the very last thing pushed onto the stack)

_**Note**: For an explanation of the stack the only real difference between 32-bit and 64-bit x86 processors is the size of the registers, and therefore the amount they decrement/increment when pushing/popping. For 32-bit a register is 4 bytes (32-bits) and will decrement/increment by 4 bytes when pushing/popping. For 64-bit a register is 8 bytes (64-bits) and will decrement/increment by 8 bytes when pushing/popping. Both have other registers that are differing sizes, but they are unimportant for this explanation, so just assume these standard sizes._

As an entire paper could be written on x86 registers and their uses, I will refer anyone who would like to know more about them to [the wiki books page on x86 registers](https://en.wikibooks.org/wiki/X86_Assembly/X86_Architecture). The page covers explanations of all the x86 registers used in 32-bit and 64-bit, as well as their intended and actual uses.

On x86 processors the stack always grows from higher memory addresses to lower memory addresses; that is to say, when a value is pushed on to the stack `%esp` is decremented thus increasing the size of the stack, and incremented when things are popped off the stack. Each push will expand the stack by the size of one register (4 bytes for 32-bit, 8 bytes for 64-bit), likewise one pop will shrink the stack by the size of one register.

Through the lifetime of a function `%ebp` should never change, it points to the first value pushed onto the stack when that function was entered. When a new function is called the function prolog will push the current value of `%ebp` onto the stack (directly below the last function's stack frame), and then copy the current value of `%esp` into `%ebp`. That looks like this:

```
foo:
    # Begin function prolog
    push %ebp
    mov %esp, %ebp
    # End function prolog
    ....
```

This ensures that when the function returns, it is capable of restoring the caller's stack frame (`%ebp` and `%esp`). In order for a function to return to the calling function at the call site, the location of the call site must also be saved somewhere. On x86 systems this value is stored on the stack by the `call` instruction. The caller knows where the callee should return to by its current value of `%eip`, or the instruction pointer. As instructions are fetched from memory, `%eip` holds the location of the next instruction to fetch (and execute), as execution continues this increments. There are several instructions in assembly that can change the value of `%eip` directly, but we will focus on the `call` and `ret` instructions.

As mentioned above, the callee function is responsible for restoring the caller's stack frame, this is handled by the function epilog:

```
    ...
    pop %ebp
    ret
```

The `pop` instruction will pop the value in the location pointed to by `%esp` into the register `%ebp`, which will also increment `%esp` (remember `%esp` increments as things are taken off the stack). The `ret` instruction will pop one value off the stack and set `%eip` to this value, it is equivalent to `pop %eip`; however you cannot directly modify `%eip` so that is an invalid instruction. But where did that saved `%eip` value come from? It is pushed onto the stack by the `call` instruction just before `%eip` is set to the callee's location.

Putting all of those pieces together one can determine how the stack will look before, during, and after a function call.

```
Before call to Function1:

Register     Value on stack        Memory Address     Notes

         -----------------------                
        |    saved %eip         |  0xbfffffd0
         -----------------------                ----   
%ebp -> |    saved %ebp         |  0xbfffffcc       | 
         -----------------------                    | Main's stack frame
        |    local variables    |  0xbfffffc8       |
%esp -> |    ..........         |  0xbfffffc4       |
         -----------------------                ----



After call to Function1:
         -----------------------                
        |    saved %eip         |  0xbfffffd0   
         -----------------------                ----
        |    saved %ebp         |  0xbfffffcc       |
         -----------------------                    | Main's stack frame
        |    local variables    |  0xbfffffc8       |
        |    ..........         |  0xbfffffc4       |
         -----------------------                ----
        |    saved %eip         |  0xbfffffc0
         -----------------------                ----
%ebp -> |    saved %ebp         |  0xbfffffbc       |
         -----------------------                    |
        |    local variables    |  0xbfffffb8       | Function1's stack frame
        |    ..........         |  0xbfffffb4       |
%esp -> |    ..........         |  0xbfffffb0       |
         -----------------------                ----


After Function1 returns:
         -----------------------                
        |    saved %eip         |  0xbfffffd0
         -----------------------                ----   
%ebp -> |    saved %ebp         |  0xbfffffcc       | 
         -----------------------                    | Main's stack frame
        |    local variables    |  0xbfffffc8       |
%esp -> |    ..........         |  0xbfffffc4       |
         -----------------------                ----
        |    saved %eip         |  0xbfffffc0
         -----------------------
        |    saved %ebp         |  0xbfffffbc
         -----------------------
        |    local variables    |  0xbfffffb8
        |    ..........         |  0xbfffffb4
        |    ..........         |  0xbfffffb0
         -----------------------

```


### A Vulnerable Program ###

The following will look at a program that is susceptible to a buffer overflow and examine the stack and registers as the overflow occurs. The program below will demonstrate all the knowledge gained from above, by executing the program in the GNU Debugger (gdb).


```c
/** Easily exploitable Buffer Overflow for learning purposes
 *
 * Compilation:
 *  gcc -fno-stack-protector -z execstack -m32 -o easy easy.c
**/

#include <string.h>
#include <stdio.h>
#include <stdlib.h>

void vulnerable(char * input) {
    char buffer[1024];

    strcpy(buffer, input);

    printf("Input: %s\n", buffer);
}

int main (int argc, char * argv[]) {

    if (argc != 2)
        exit(1);

    vulnerable(argv[1]);
}

```

An explanation of the compiler flags used is in order to understand what follows:

 * `-fno-stack-protector` - Turns off the stack protector function. That is a small piece of code that places a specially crafted value on the stack just below the saved `%eip` value.
 * `-z execstack` - Adds the executable bit to the permissions of the memory space mapped to the stack (i.e. makes the stack a valid place to put executable code).
 * `-m32` - On a 64-bit system this will force the compiler to compile the source as a 32-bit process.

For the purposes of learning about this particular program, Address Space Layout Randomization (ASLR) should also be turned off:
`sysctl -w kernel.randomize_va_space=0`

A later post will explore more deeply what each of these flags does and the impact they have on the resulting binary file. For now, suffice it to say that they severely weakened the executable code produced by the compiler.

GDB is capable of showing the assembly instructions that the compiler generated for each function, which will be crucial to understanding how to exploit this program:

```
$ gdb -q ./easy
(gdb) disas main
Dump of assembler code for function main:
0x0804843a <main+0>:    lea    0x4(%esp),%ecx
0x0804843e <main+4>:    and    $0xfffffff0,%esp
0x08048441 <main+7>:    pushl  -0x4(%ecx)
0x08048444 <main+10>:   push   %ebp
0x08048445 <main+11>:   mov    %esp,%ebp
0x08048447 <main+13>:   push   %ecx
0x08048448 <main+14>:   sub    $0x14,%esp
0x0804844b <main+17>:   mov    %ecx,-0xc(%ebp)
0x0804844e <main+20>:   mov    -0xc(%ebp),%eax
0x08048451 <main+23>:   cmpl   $0x2,(%eax)
0x08048454 <main+26>:   je     0x8048462 <main+40>
0x08048456 <main+28>:   movl   $0x1,(%esp)
0x0804845d <main+35>:   call   0x8048340 <exit@plt>
0x08048462 <main+40>:   mov    -0xc(%ebp),%edx
0x08048465 <main+43>:   mov    0x4(%edx),%eax
0x08048468 <main+46>:   add    $0x4,%eax
0x0804846b <main+49>:   mov    (%eax),%eax
0x0804846d <main+51>:   mov    %eax,(%esp)
0x08048470 <main+54>:   call   0x8048404 <vulnerable>
0x08048475 <main+59>:   add    $0x14,%esp
0x08048478 <main+62>:   pop    %ecx
0x08048479 <main+63>:   pop    %ebp
0x0804847a <main+64>:   lea    -0x4(%ecx),%esp
0x0804847d <main+67>:   ret
End of assembler dump.
```

This shows the assembly that gcc converted the `main` function into. From the left the output shows the address of the instruction, an interpreted name for that address, and the raw assembly that is located at that address. The function prolog can be seen At the instruction labeled `<main+10>` (`0x08048444`), and the function epilog is at `<main+63>` (`0x08048479`). These may look slightly different for the main function than was spelled out above, but something more familiar is in the `vulnerable` function.

```
(gdb) disas vulnerable
Dump of assembler code for function vulnerable:
0x08048404 <vulnerable+0>:  push   %ebp
0x08048405 <vulnerable+1>:  mov    %esp,%ebp
0x08048407 <vulnerable+3>:  sub    $0x408,%esp
0x0804840d <vulnerable+9>:  mov    0x8(%ebp),%eax
0x08048410 <vulnerable+12>: mov    %eax,0x4(%esp)
0x08048414 <vulnerable+16>: lea    -0x400(%ebp),%eax
0x0804841a <vulnerable+22>: mov    %eax,(%esp)
0x0804841d <vulnerable+25>: call   0x8048320 <strcpy@plt>
0x08048422 <vulnerable+30>: lea    -0x400(%ebp),%eax
0x08048428 <vulnerable+36>: mov    %eax,0x4(%esp)
0x0804842c <vulnerable+40>: movl   $0x8048540,(%esp)
0x08048433 <vulnerable+47>: call   0x8048330 <printf@plt>
0x08048438 <vulnerable+52>: leave  
0x08048439 <vulnerable+53>: ret    
End of assembler dump.
```

This looks more like the expected structure, the function prolog is exactly what was expected, but the epilog still doesn't look right. `pop %ebp` has been replaced with `leave`, which does effectively the same thing. The next thing to notice is the instruction directly after the function prolog `sub $0x408, %esp`, since `%esp` points to the top of the stack and the stack grows down toward lower memory addresses this instruction is creating more space on the stack. Converting 1024 (the size of our buffer) to hex, yields exactly `0x400`, that must mean that the compiler reserved another 8 bytes for something else.

Exploring the contents of the stack before and after the call to `vulnerable` will yield the purpose of these extra 8 bytes:

```
(gdb) b *main+54
Breakpoint 1 at 0x8048470
(gdb) run hello
Starting program: /tmp/easy/easy hello

Breakpoint 1, 0x08048470 in main ()
Current language:  auto; currently asm
```

This sets a breakpoint on the line that calls the function `vulnerable`, and then runs the program with the input string "hello". The debugger stops at the breakpoint and prints the address of `%eip` as `0x08048470`. When setting a breakpoint in gdb, it will stop when `%eip` has the value of the breakpoint that was set (i.e. before that instruction executes).

```
(gdb) info reg
eax            0xbfffda82   -1073751422
ecx            0xbfffd8f0   -1073751824
edx            0xbfffd8f0   -1073751824
ebx            0x26eff4 2551796
esp            0xbfffd8c0   0xbfffd8c0
ebp            0xbfffd8d8   0xbfffd8d8
esi            0x8048490    134513808
edi            0x8048350    134513488
eip            0x8048470    0x8048470 <main+54>
eflags         0x286    [ PF SF IF ]
cs             0x73 115
ss             0x7b 123
ds             0x7b 123
es             0x7b 123
fs             0x0  0
gs             0x33 51
(gdb) si
0x08048404 in vulnerable ()
(gdb) i r
eax            0xbfffda82   -1073751422
ecx            0xbfffd8f0   -1073751824
edx            0xbfffd8f0   -1073751824
ebx            0x26eff4 2551796
esp            0xbfffd8bc   0xbfffd8bc
ebp            0xbfffd8d8   0xbfffd8d8
esi            0x8048490    134513808
edi            0x8048350    134513488
eip            0x8048404    0x8048404 <vulnerable>
eflags         0x286    [ PF SF IF ]
cs             0x73 115
ss             0x7b 123
ds             0x7b 123
es             0x7b 123
fs             0x0  0
gs             0x33 51
```

**Notes:**

 * `info reg` (or `i r`)  - displays the current values of the registers
 * `si` - Executes a single instruction and the pauses execution agian.


This examines the registers, take note of `eip`, `esp`, and `ebp` before and after the call to vulnerable. At this point of execution, only the `call` to `vulnerable` has executed, so the function prolog has not executed and the stack frame for `vulnerable` has not been setup. Just as was explained in the previous section, the value of `esp` was decremented by 4 after the `call` instruction was executed. This can be confirmed by examining the value stored on top of the stack.


```
(gdb) x/xw $esp
0xbfffd8bc: 0x08048475
(gdb) si
0x08048405 in vulnerable ()
(gdb) 
0x08048407 in vulnerable ()
(gdb) i r
eax            0xbfffda82   -1073751422
ecx            0xbfffd8f0   -1073751824
edx            0xbfffd8f0   -1073751824
ebx            0x26eff4 2551796
esp            0xbfffd8b8   0xbfffd8b8
ebp            0xbfffd8b8   0xbfffd8b8
esi            0x8048490    134513808
edi            0x8048350    134513488
eip            0x8048407    0x8048407 <vulnerable+3>
eflags         0x286    [ PF SF IF ]
cs             0x73 115
ss             0x7b 123
ds             0x7b 123
es             0x7b 123
fs             0x0  0
gs             0x33 51
(gdb) x/xw $ebp
0xbfffd8b8: 0xbfffd8d8
```

The first command examines the memory one word at a time starting at the value pointed to by `%esp`. Exactly as expected, the disassembly output of `main` will confirm that `0x08048475` is the instruction directly after the `call vulnerable` instruction.

The next command `si` is executed twice (if you don't specify a command, gdb executes the last command again), this executes through the end of the function prolog. At this point `%esp` and `%ebp` both point to `0xbfffd8b8`. An examination of the contents of that memory space will show that it holds the value of `0xbfffd8d8`, which was the value of `%ebp` before the call to the `vulnerable` function. Just as suspected, the saved value of `%ebp` is being pointed to by `%ebp`, and just above that is the return address (saved `%eip`).

So what are those extra 8 bytes of data that the compiler saved room for in `vulnerable`s stack frame? As it turns out, on 32-bit x86 processors the C Application Binary Interface calls for passing all parameters to functions on the stack. So it is saving 4 bytes for a pointer to the input string ("hello") and 4 bytes for the destination pointer (`buffer`). A careful examination of the assembly of that function will reveal that `buffer` is the 1024 bytes immediately following the saved `%ebp` (therefore `%ebp-4`), and the two arguments to `memcpy` directly follow that. Looking back at the `vulnerable` function will show the preparation for the call to `memcpy`:


```
0x0804840d <vulnerable+9>:  mov    0x8(%ebp),%eax
0x08048410 <vulnerable+12>: mov    %eax,0x4(%esp)
0x08048414 <vulnerable+16>: lea    -0x400(%ebp),%eax
0x0804841a <vulnerable+22>: mov    %eax,(%esp)
```

The first two lines load the address of the first parameter passed to `vulnerable` into the memory at location `%esp + 4`. The next 2 instructions load the address of `%ebp - 0x400` into the memory at location `%esp`. This reveals a subtle detail of this architecture: the `memcpy` function is going to start copying `argv[1]` into `buffer` at the lowest memory address occupied by `buffer`. That is, it copies memory into the buffer starting at the memory address closest the top of stack (`%esp`) working its way to the bottom of the stack (`%ebp`).

Knowing that `buffer` occupies 1024 bytes on the stack, and directly above it is the saved `%ebp` followed by the saved `%eip`, it should be fairly easy to calculate the amount of data needed to exactly overwrite the saved `%eip`. Recalling that the `ret` instruction at the end of this function will load that saved `%eip` value from the stack and attempt to set the instruction pointer to that value, it should be fairly easy to get the program to jump to any arbitrary location.

To fill the buffer should require 1024 bytes of data, add an additional 4 bytes to overwrite the saved `%ebp`, 4 more bytes will overwrite the saved `%eip`. All that totals 1032 bytes of data.


```
(gdb) r `perl -e 'print "A"x1028, "B"x4'`
The program being debugged has been started already.
Start it from the beginning? (y or n) y
Starting program: /tmp/easy/easy `perl -e 'print "A"x1028, "B"x4'`
Input: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBB

Program received signal SIGSEGV, Segmentation fault.
0x42424242 in ?? ()
(gdb) i r
eax            0x410    1040
ecx            0x0  0
edx            0x2700f0 2556144
ebx            0x26eff4 2551796
esp            0xbfffd4c0   0xbfffd4c0
ebp            0x41414141   0x41414141
esi            0x8048490    134513808
edi            0x8048350    134513488
eip            0x42424242   0x42424242
eflags         0x10292  [ AF SF IF RF ]
cs             0x73 115
ss             0x7b 123
ds             0x7b 123
es             0x7b 123
fs             0x0  0
gs             0x33 51
```

The program crashes with a segmentation fault. This is the error thrown when a memory access violation occurs, in this case it was because the program tried to access memory at location 0x42424242 ("BBBB"). The output above shows that the buffer and saved `%ebp` were successfully filled with "A"s, and the saved `%eip` was overwritten with "B"s. The program then tried to return from the `vulnerable` function to the location `0x42424242` which is what it found to be the saved `%eip` which caused the segementation fault.


### Next Steps ###

The ultimate goal of a buffer overflow attack is to take control of the program and force it to execute arbitrary, attacker-controlled, instructions. The provious section outlined how to take control of `%eip` which is often at least half the battle, but only forced the program to crash which is of little use. The next logical step would be to write some shellcode and force the vulnerable program to execute it. 

Shellcode is a small set of compiled instructions designed to be used in an exploit, which most often results in executing a shell; hence, **shell**code. Shellcode is unique in that it most often must be very small, contain no null bytes, and be stand-alone. The next article will cover the basics of shell coding on 32-bit and 64-bit systems, and using the resulting shellcodes to attack the vulnerable program above, which will wrap up the discussions about how to execute buffer overflows in a very insecure environment.

After the conclusion of the completely insecure setup, we will revisit the `easy.c` program and look at how the program can be exploited without the executable stack (removal of the `-z noexec`). That will introduce the concept of Return Oriented Programming, and the return to libc attack which will be revisted later in the series.

Following the discussion of rop (return oriented programming) will be a discussion of the challenges that can be faced by the return oriented programmer in a modern environment with ASLR. A discussion of what ASLR is, what it does, the challenges it creates for writing exploits, and ways to abuse it will follow introducing the Global Offset Table (GOT). The use and abuse of the GOT can effectively allow an attacker to completely bypass the protections of ASLR, so it is only fitting to examine why that is true and how to do it.

The series will wrap up with a discussion of the stack canary (turned off by `-fno-stack-protector`), what it is, how it works, and why it is (mostly) secure. This discussion will naturally lend itself to a discussion of exploit primitives and chaining abuses of software to create an exploit. When all of the pieces have been put back together and all of the protections of a modern day system have been turned back on, some recent examples of buffer overflows and how they ended up in production code will be shared.

### Conclusions ###

This has been a fairly detailed explanation of what a buffer overflow is and what causes it to occur on 32-bit x86 systems. With all of the modern day protections turned off, these exploits are nearly identical for 64-bit x86 systems with a few minor adjustments.

Because the exact size of the stack frame can be calculated, this post did not touch on what happens when data past the saved `%eip` is overwritten. For a 32-bit x86 system this will cause no problems; however 64-bit x86 systems will stop execution when an attempt is made to set the instruction pointer to any value larger than `0x00007fffffffffff`. The implication is that when executing a buffer overflow on a 64-bit x86 system, one must calculate exactly how much data to write or else the program will unintentionally crash.

This post has detailed everything from what a buffer overflow is to how one can be used to make a program start executing instructions at an arbitrary location. Albeit no new instructions were executed, a future post will follow up on how write instructions to the program and execute them, as well as how to generate those instructions from hand (read shellcoding). After a complete understanding of this old style of buffer overflow is gained, the posts will evolve into using newer techniques to evade all the security protections we turned off for this post.

