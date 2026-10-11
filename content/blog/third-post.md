+++
title = "Code Translation: From Source Code to Machine Code"
date = 2026-10-11
description = "What actually happens between gcc main.c and ./a.out: preprocessing, compilation, assembly and linking, with real output at every stage."

[taxonomies]
tags = ["c", "compilation", "gcc", "embedded"]
+++

## Motivation

In an interview, I was once asked to explain the compilation steps, and my mind went blank. I still can't say why. Maybe I didn't know the steps. Maybe I knew them, but only on the surface. Or maybe I was just nervous. I left the room without knowing which one it was, and that bothered me more than the question did.

Most of the time, compiling is one command and a program appears. It works, so it's easy to never look inside. That interview showed me that using a tool and understanding it are not the same thing.

So I decided to settle it. In this post I go through each stage myself, run it, and look at what comes out. If I can explain every step in my own words and show its output, Its understood completely then.


## Why compile at all?

Here is a small C program. Anyone can read it.

```c
#include <stdio.h>

int main(void)
{
    int obj1 = 1;
    int obj2 = 2;

    printf("%d \n", obj1 + obj2);
    return 0;
}
```

A CPU can't. It only runs machine instructions, which are just numbers in binary. So the .c file has to be turned into those instructions before anything can happen, and that translation is what compilation is.

## The four stages

When I type gcc main.c, it feels like one step, but it is really four programs running one after another, each handing a file to the next.

```
main.c  ->  main.i  ->  main.s  ->  main.o  ->  main
        gcc -E      gcc -S      gcc -c      gcc
```

Preprocessing, compilation, assembly and linking. I'll go through them in order. All the output below comes from GCC 13.3 on x86-64 Linux, so on another machine, say an ARM microcontroller, the instructions will look different but the stages are the same.

To give the preprocessor something to do, I added a macro and a comment to the program.

```c
#include <stdio.h>

#define GREETING_NUM 1

int main(void)
{
    int obj1 = GREETING_NUM; /* first value */
    int obj2 = 2;

    printf("%d \n", obj1 + obj2);
    return 0;
}
```

## Preprocessing

The preprocessor works on text only. It doesn't understand C yet. It just looks for lines starting with #, like #include, #define and #ifdef, and acts on them. An #include is replaced by the whole contents of that header, a #define macro is replaced wherever it's used, code inside #if and #ifdef blocks is kept or thrown away depending on the condition, and comments are removed.

```sh
gcc -E main.c -o main.i
```

The result is still plain C. Mine came out at 825 lines, almost all of it from stdio.h being pasted in. The part I care about is at the bottom.

```c
# 5 "main.c"
int main(void)
{
    int obj1 = 1;
    int obj2 = 2;

    printf("%d \n", obj1 + obj2);
    return 0;
}
```

GREETING_NUM has become 1 and the comment is gone. The line starting with # 5 is a marker that tells the compiler which file and line the code came from, which is how error messages can point back to your original source.

## Compilation

This is the stage the word usually refers to. The compiler reads the preprocessed code, checks the syntax and the types, optimizes it, and writes assembly for the target CPU.

```sh
gcc -S main.i -o main.s
```

While writing this I noticed I had typed onj1 instead of obj1 in my notes. That's the kind of mistake this stage catches. The compiler stops with an error about an undeclared name and never produces assembly. (Errors about a function that's declared but never defined show up later, at linking.)

Here is main.s, with no optimization.

```asm
.LC0:
        .string "%d \n"
main:
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $16, %rsp
        movl    $1, -8(%rbp)        # obj1 = 1
        movl    $2, -4(%rbp)        # obj2 = 2
        movl    -8(%rbp), %edx
        movl    -4(%rbp), %eax
        addl    %edx, %eax          # obj1 + obj2
        movl    %eax, %esi          # second argument to printf
        leaq    .LC0(%rip), %rax    # address of the format string
        movq    %rax, %rdi          # first argument to printf
        movl    $0, %eax
        call    printf@PLT
        movl    $0, %eax            # return 0
        leave
        ret
```

I trimmed the directives at the top, but the rest is untouched, and you can follow the C through it. The two variables are stored on the stack, they are added, and the result and the address of the string are put into registers for the call to printf. The return value goes into eax.

One thing to hold on to is that printf is only called here. The compiler has no idea where its code lives.

## Assembly

The assembler turns this text into actual bytes and writes an object file, main.o on Linux and similar systems, or main.obj with some other toolchains.

```sh
gcc -c main.s -o main.o
```

An object file isn't readable, so I used objdump -d -M intel main.o to see the bytes next to the instructions.

```
0000000000000000 <main>:
   0:   55                      push   rbp
   1:   48 89 e5                mov    rbp,rsp
   4:   48 83 ec 10             sub    rsp,0x10
   8:   c7 45 f8 01 00 00 00    mov    DWORD PTR [rbp-0x8],0x1
   f:   c7 45 fc 02 00 00 00    mov    DWORD PTR [rbp-0x4],0x2
  16:   8b 55 f8                mov    edx,DWORD PTR [rbp-0x8]
  19:   8b 45 fc                mov    eax,DWORD PTR [rbp-0x4]
  1c:   01 d0                   add    eax,edx
  1e:   89 c6                   mov    esi,eax
  20:   48 8d 05 00 00 00 00    lea    rax,[rip+0x0]
  27:   48 89 c7                mov    rdi,rax
  2a:   b8 00 00 00 00          mov    eax,0x0
  2f:   e8 00 00 00 00          call   34 <main+0x34>
  34:   b8 00 00 00 00          mov    eax,0x0
  39:   c9                      leave
  3a:   c3                      ret
```

The middle column is the machine code. 55 is push rbp and c3 is ret. This is what the CPU actually runs.

What surprised me is the lea and call lines, where the address part is all zeros. The assembler doesn't know where the string or printf will end up, so it leaves a blank and writes a note for later. nm and objdump -r show that.

```
$ nm main.o
0000000000000000 T main
                 U printf

$ objdump -r main.o
OFFSET           TYPE              VALUE
0000000000000023 R_X86_64_PC32     .rodata-0x0000000000000004
0000000000000030 R_X86_64_PLT32    printf-0x0000000000000004
```

T means main is defined in this file. U means printf is undefined, so someone else has to supply it. The two relocation entries say that the bytes at offsets 0x23 and 0x30 need to be patched with real addresses later. So an object file has the machine code, the data, a table of symbols and these relocation notes, but it isn't a complete program yet.

## Linking

The linker takes the object files and libraries and joins them into one executable. It looks at that U printf, finds printf in the C library, fills in the blanks from the relocation notes, and lays everything out at real addresses.

```sh
gcc main.o -o main
```

Disassembling the final file shows the zeros are gone.

```
0000000000001139 <main>:
    ...
    1159:   48 8d 05 a4 0e 00 00    lea    rax,[rip+0xea4]        # 2004
    ...
    1168:   e8 c3 fe ff ff          call   1030 <printf@plt>
```

main now sits at 0x1139, the string is at 0x2004, and the call points somewhere real. The @plt part is there because gcc links to the C library dynamically by default. The code for printf isn't copied into my program. It's loaded from libc when the program starts, and ldd main confirms that by listing libc.so.6.

For a fully self-contained file you can link statically with gcc -static main.o -o main. That's the normal choice in bare-metal and embedded work, where there's no operating system to load shared libraries.

And then it runs.

```
$ ./main
3
```

## Wrapping up

Plain gcc main.c does all four stages at once. If you want to see the files in between, gcc -save-temps main.c keeps them.

If someone asks me again, the short version is this. The preprocessor expands the text, the compiler turns it into assembly, the assembler turns that into an object file, and the linker joins object files and libraries into an executable.