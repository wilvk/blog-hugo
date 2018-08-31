---
title: "Creating a Fibonacci Generator in Assembly"
date: 2018-08-30T21:12:38+10:00
draft: false
---

# Contents

- [Intro](#intro)
  * [Following along at home](#following-along-at-home)
  * [A beginning](#a-beginning)
  * [Some starting knowledge](#some-starting-knowledge)
  * [Overview of the implementation](#overview-of-the-implementation)
- [Beginning the implementation](#beginning-the-implementation)
  * [Building binary files from assembly files](#building-binary-files-from-assembly-files)
  * [The structure of a GAS assembly file](#the-structure-of-a-gas-assembly-file)
  * [Sections](#sections)
  * [Comments](#comments)
  * [Instructions and opcodes](#instructions-and-opcodes)
  * [The application entry-point and labels](#the-application-entry--point-and-labels)
  * [Compiling our code](#compiling-our-code)
  * [Reading from the process arguments and converting to a numeric value](#reading-from-the-process-arguments-and-converting-to-a-numeric-value)
  * [Debugging with GDB](#debugging-with-gdb)
- [Getting the length of our argument on the command line](#getting-the-length-of-our-argument-on-the-command-line)
  * [Making things easier to understand using function calls](#making-things-easier-to-understand-using-function-calls)
- [Converting a string to a number](#converting-a-string-to-a-number)
  * [Some new functions](#some-new-functions)
  * [Zeroing registers](#zeroing-registers)
  * [Register sizes and layout](#register-sizes-and-layout)
  * [Processing the string](#processing-the-string)
- [The actual Fibonacci algorithm](#the-actual-fibonacci-algorithm)
  * [Creating the stack space for our array](#creating-the-stack-space-for-our-array)
  * [The Fibonacci logic in assembly](#the-fibonacci-logic-in-assembly)
  * [Setting up our variables](#setting-up-our-variables)
  * [Array memory allocation](#array-memory-allocation)
  * [Variable initialisation](#variable-initialisation)
  * [Indexed memory](#indexed-memory)
  * [Running the loop to completion](#running-the-loop-to-completion)
  * [Calculating the sequence](#calculating-the-sequence)
- [Printing our result](#printing-our-result)
  * [Doing things the hard way](#doing-things-the-hard-way)
- [Conclusion](#conclusion)
  * [From here to there - improving the code](#from-here-to-there---improving-the-code)

# Intro

Stemming from my interest in Assembly Language, I was in search of a practical example to start exploring this subterranean wilderland. I thought I'd write a simple ASM application to compute Fibonacci to a given number of iterations.

I had a think about some criteria and restrictions for this little project and came up with the following:

- It must take a value from the command line for the number of iterations
- It must not use the glibc standard library
- It must print the final value in the sequence to the screen

The command line was decided to be used for the number of iterations as it is easily accessible for beginners and had less overhead in setting up. The glibc library was decided to be excluded as I wanted to get down to the nuts and bolts of printing and manipulating strings. The requirement to only print the final value in the sequence was more to keep the logic of the application simple enough to express the features of assembly without unnecessarily complicating things.

## Following along at home

The source code for what I describe here can be found in [a repository](https://github.com/wilvk/gas-asm-fib) I have created that contains all the examples described. It also has shell scripts for:

- Running the scripts in a Docker container with all the required tools installed (`docker-shell`)
- Compiling the examples (`make-app`)

## A beginning

I found a good blog about different approaches to Fibonacci Equations [here](https://www.geeksforgeeks.org/program-for-nth-fibonacci-number/) and armed with the new knowledge from reading a couple of books on the subject (specifically, [Assembly Language Step-by-Step](https://www.amazon.com/Assembly-Language-Step-Step-Programming/dp/0470497025) and [Professional Assembly Language](https://www.amazon.com.au/Professional-Assembly-Language-Richard-Blum-ebook/dp/B000Q7ZETY)) I set out to implement this.

## Some starting knowledge

The basis of the algorithm to use was decided to be a space-optimised method using a dynamic programming approach. This effectively means using an array on the stack to hold our values while giving **O(n)** time and space complexity.

The code used as a basis is shown below:

```c
//Fibonacci Series using Dynamic Programming
#include<stdio.h>
 
int fib(int n)
{
  /* Declare an array to store Fibonacci numbers. */
  int f[n+2];   // 1 extra to handle case, n = 0
  int i;
 
  /* 0th and 1st number of the series are 0 and 1*/
  f[0] = 0;
  f[1] = 1;
 
  for (i = 2; i <= n; i++)
  {
    /* Add the previous 2 numbers in the series and store it */
    f[i] = f[i-1] + f[i-2];
  }
 
  return f[n];
}
 
int main ()
{
  int n = 9;
  printf("%d", fib(n));
  getchar();
  return 0;
}
``` 

## Overview of the implementation

The implementation involved the following functionality:

- Getting the length of the command line argument
- Converting the command line argument from a string to a long (unsigned 32-bit number)
- Calculating Fibonacci
- Printing the long as a string to stdout

## Beginning the implementation

Ok, so at this point we have the basis for an algorithm in assembly based on our C code but how can this be implemented in ASM?

There are numerous formats of Assembly Language including NASM, GAS and MASM among others. I'm working on a Linux box and so MASM was out of the question as it is the Microsoft Assembler.

I decided to go with GAS, the GNU Assembler, as it is cross-platform and is a good general purpose assembler. This also gave me the ability to cheat in my implementation (which I didn't), as using `gcc`, a valid C application can be converted into intermediate GAS assembly language.

I decided the number of iterations of Fibonacci would be determined from the command line as an argument to the application. It would expect a valid number from the user and convert this from a string to a number. The Fibonaci sequence would then be iterated _n_ number of times.

The Fibonacci algorithm would have to create an array of lenth _n+2_ on the stack based on the user input. Another variable (as the variable _i_ shown above) would be used as a counter.

The first two elements of the array would be pre-populated with 0 and 1 respectively, as at least these two numbers are used when calculating new values. This can be considered a type of 'seed' for our algorithm.

A loop is then started with the counter set to 2 that iterates until the counter is greater than or equal to the number of iterations, _n_, and where the counter is increased by 1 on each iteration.

Inside the loop, for the array position in the stack, and based on the counter value, the _i-1_ value is added to the _i-2_ value on the stack and the result is placed on the stack at the current location allowing it to be used for the next iteration.

Once the final value of Fibonacci is determined, a function is called to convert the number into a string and print it to stdout.

The application then needs to exit gracefully and return control to the shell.

## Building binary files from assembly files

We will start with the following assembly code. This gives us a basis for filling out our application:

*fib1.s*
```asm
# framework
.section .text
.globl _start
  _start:
    nop
    # the rest of our program goes here
    movl $1, %eax
    movl $0, %ebx
    int $0x80
```

## The structure of a GAS assembly file 

## Sections

In the code above, the line `.section .text` indicates that the lines preceeding it will be part of the 'text' section. The text section is a specific section in the compiled binary that contains all the assembly language instructions converted to machine code. The machine code is the binary representation of the assembly language instructions and is one of the main tasks of the assembler. By default in a Linux application there are also sections called `.data` and `.bss` that refer to initialised variables and uninitialised variables respectively. We won't delve into these types of variables but it is good to know they exist for more advanced development in assembly code.

## Comments

The code above introduces some of the syntax of a GAS assembly file (`.s`). Comments begin with a hash (`#`) and can be at the start of a line or after an instruction/opcode. 

## Instructions and opcodes

An instruction/opcode is essentially an instruction for the CPU to process. Examples of intructions in __fib1.s__ above include:

- `nop`
- `movl $1 %eax`
- `movl $0, %ebx`
- `int $0x80`.

I will use the terms instruction and opcode interchangeably, but they are essentially a mnemonic in assembly language that tells the CPU what to do. Opcodes/instructions can also have one or many operands depending on what the opcode does. Operands are the values that the opcode act upon and specify what the opcode does.

For some context from the example above, we can see the following delineation:

|Line          | Opcode | Operand 1  | Operand 2 |      Description          |
|--------------|--------|------------|-----------|---------------------------|
|nop           |nop     |    -       |   -       |no-operation               |
|movl $1, %eax |movl    |   $1       |  %eax     |copy 0 into register eax   |
|movl $0, %ebx |movl    |   $0       |  %ebx     |copy 1 into register ebx   |
|int $0x80     |int     |   $0x80    |   -       |call interrupt number 0x80 |

The first `movl` instruction copies a long (4-byte) value of 0 into the register `eax`. The second does the same with 1 and the register `ebx` respectively. There are a few different variations of the `mov` command for copying values to and from registers.

These are namely:

- `movl` - copy a long, 4-byte value (32-bits)
- `movw` - copy a word, 2-byte value (16-bits)
- `movb` - copy a byte, 1-byte value (8-bits)

It is important to know what type of `mov` instruction to use as there are times when you will want to move just a single byte, a word, a long (or double-word), or some other length. Larger lengths will usually require a loop or some other method when working on a 32-bit x86 architecture.

A few extra things to note about the above instructions are:

- The `$1` and `$0` operands are called immediate values and are an exact value, as specified by the `$` prefix.
- Registers are prefixed with a percentge sign (`%`).
- A value starting with a `0x` is a hexadecimal number and includes values 0-9 and a-f.

## The application entry-point and labels

For Linux applications, the `_start:` label is required by the assembler to define the entry point to the binary application. All labels finish with a colon (`:`) to indicate that it is a label. The assembler also requires the entrypoint to be made 'global' so that other binaries, and the operating system knows that the label has been exposed by the binary. This is the reason for the line `.globl _start` preceeding the label `_start` and is used more when working with multiple assembly files that need to be compiled together.

## Compiling our code

We're working backwards a bit here as we've implemented our entrypoint (`_start`) and our syscall to exit first. The syscall using `eax=1` and `ebx=0` with `int 0x80` allows our program to exit gracefully. As noted by the comment, we shall fill in the logic from the middle as we progress.

Ok, so we've got a framework of GAS assembly to build on, but how do we convert this into an application?

Well, it's easy. I'm glad you asked.

Firstly we need to make sure our GAS assembler is installed. I'm on Fedora, so the command is `sudo dnf install binutils`. From the supplementary [repository](https://github.com/wilvk/gas-asm-fib), the `docker-shell` application can spin you up an environment with the required tools already installed.

Then we create a small shell script to run the GAS assembler (as) and the linker (ld) on the output of the assembler.

Say for example we call our assembly code file fib1.s, to run the assembler we would run `as -gstabs -o fib.o fib.s`

- `-gstabs` adds debugging info
- `-o fib.o` specifies our object file
- `fib.s` is our source code

The object file is a binary with all the symbols and source included, which included the raw machine code to run.
We then link the resultant object file to convert our object file into a binary application. The linker also converts our symbolic references (such as `\_start` in our case) to relative numeric references to where the symbols will reside in memory when the binary is run as an application.

If we were to run `ld -o fib fib.o`, we should now have a new binary application with the permissions 0755 set. If you run the application it will do nothing. But more importantly, it won't segfault, which is a good thing and indicates the application has exited gracefully.

- `-o fib` specifies the output application name.
- `fib.o` specifies the object file to use.

The linker has other features (which I won't discuss here) that make your life easier as an assembly language programmer.

We will be targetting the 32 bit x86 architecture to begin with as it is easier to understand. The following is a bash script for building the binary (also in the supplementary repo as `make-app`).

_make-app_
```
#!/bin/bash

if [ -z "$1" ]; then
  app_name=fib
else
  app_name=$1
fi

rm $app_name.o $app_name 2> /dev/null

as --32 -gstabs -o $app_name.o $app_name.s
ld -m elf_i386 -o $app_name $app_name.o

```

Makefiles are the status-quo for building assembly files, but I'm more used to working with bash scripts so the preceeding is a script that can be used to compile a single assembly file into a binary targetting 32 bit x86, and as we are working with a small number of files, we do not need some of te more advanced features of make anyway.

If a name is passed on the commandline, it will attempt to compile the corresponding .s file.

For example:

` ./make-app fib2`

will build a binary called fib2.

## Reading from the process arguments and converting to a numeric value

If we call our binary from the commandline like:

`./fib2 test`

The arguments are already placed into the stack for us, courtesy of the Linux kernel, along with all the environment variables of the parent process. All we should have to do, to begin with, is find the value in the stack where the argument is and then convert it to a string and print it to stdout.

The stack is upside down in memory (with the last addition to the stack at the lower memory address), and so we traverse up the stack to find the first argument.

The stack is composed as follows:

|Stack    |Data                                                                |
|---------|--------------------------------------------------------------------|
|   ...   |                                                                    |
|ESP + n+8| < pointer to second environment variable >                         |
|ESP + n+4| < pointer to first environment variable >                          |
|ESP + n  | < pointer to last string argument >                                |
|  ...    |                                                                    |
|ESP + 12 | < pointer to second string argument >                              |
|ESP + 8  | < pointer to a string that contains the first argument >           |
|ESP + 4  | < pointer to a string containing the name of the application >     |
|ESP + 0  | < number of arguments on command line >                            |
|   ...   |                                                                    |

We know that we are just after the first string argument so we can predict that the pointer to the first string argument will be 2 places up the stack from the current value of ESP. We are using 32 bit locations here, where every pointer is 4 bytes ( 4 x 8 bytes = 32 bits ).

This means that our first command line argument is at the location ESP + 8. This is because the Stack Pointer ESP can directly reference every byte in memory, and we are looking for the location 2 up from it's current location, effectively 2 x 4 bytes.

We want to move this into a register and print it out so we know we have the correct location. This will also show us what we need to do to print the final value.

As a final step in obtaining the value from the command line, the system call (also known as a software interrupt, as `int 0x80`) to write to stdout requires the length of the string that is to be processed.

## Debugging with GDB

It would almost be impossible for a mere mortal to understand what is going on without a debugger, and remiss of me to go through this example without explaining some basics of how to use an asm debugger. I chose to use GDB as it outputs to GAS assembly and there is plenty of information on how it works online.

We'll use the following code (as per `fib2.s`) and load it into GDB with the command line argument 'test', such that once we've made `fib2`, we can run `gdb --args ./fib2 test`. Gdb will load the binary with the argument specified and wait at it's [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) prompt for further instructions.

_fib2.s_
```asm
# stack args example
.section .text
.globl _start
  _start:
    nop
    movl %esp, %ebp     # take a copy of the stack pointer esp into ebp
    addl $8, %ebp       # address of first arg in stack
    movl (%ebp), %ecx   # move the address of the first arg into ecx
    movl $4, %edx       # set the length of our string to 4
    movl $4, %eax       # indicate to int 0x80 that we are doing a write
    movl $0, %ebx       # indicate to int 0x80 that we are writing to file descriptor 0
    int $0x80           # call int 0x80 for write
    movl $1, %eax       # exit gracefully
    movl $0, %ebx       # with return code 0
    int $0x80           # call int 0x80 for exit
```

Gdb is a very powerful debugger and has many options that I'm still getting across, but for these examples we will use a small subset of it's functionality to:

- step through our code
- inspect memory values
- inspect registers

At the GDB prompt, if we enter the command `list` we can see the start of the source code for the `fib2` app we have loaded, including line numbers. We can set a breakpoint for a corresponding line number to start our debugging at by typing `breakpoint n`, or just `b n`, where `n` is our line number. 

For example:

```
(gdb) list
1       # stack args example
2       .section .text
3       .globl _start
4         _start:
5           nop
6           movl %esp, %ebp     # take a copy of the stack pointer esp into ebp
7           addl $8, %ebp       # address of first arg in stack
8           movl (%ebp), %ecx   # move the address of the first arg into ecx
9           movl $4, %edx       # set the length of our string to 4
10          movl $4, %eax       # indicate to int 0x80 that we are doing a write
(gdb)
```

For this example, if we enter `b 5`, a breakpoint will be set at line 5, ready to be executed.
Line 5 is a no-operation (or noop), which means that it is a filler instruction and doesn't actually do anything but take up a CPU clock cycle. Older versions of GDB require this proceeding `_start`, and so I have left it in for these examples. 

```
(gdb) b 5
Breakpoint 1 at 0x8048054: file fib2.s, line 5.
(gdb)
```

To run the binary up to the breakpoint, we can enter `r` or `run`. This will run through the binary up to but not including line 5.

```
(gdb) r
Starting program: /gas-asm-fib/fib2 test

Breakpoint 1, _start () at fib2.s:5
5           nop
(gdb)

```

From here, we can step through the remaining instructions, one at a time by entering 'stepi', which will run the current instruction and list the next instruction to be run. There is also the `step` instruction which is used more for high-level languages and steps over groups of instructions for each line of code; but for assembly, we can be more certain that each instruction will get hit if we use `stepi`.

```
(gdb) stepi
6           movl %esp, %ebp     # take a copy of the stack pointer esp into ebp
...
(gdb) stepi
(gdb) stepi
(gdb) stepi
...
(gdb) stepi
10          movl $4, %eax       # indicate to int 0x80 that we are doing a write
```

Entering `stepi` several times up to line 10, we shoud have the address of the first argument in the register ecx. The preceeding instruction `mov`es, or copes the decimal value 4 into the register `eax`.

To see what all the values of the registers are, we can enter `info registers` or just `info reg`:

```
gdb) info reg
eax            0x0      0
ecx            0x0      0
edx            0x0      0
ebx            0x0      0
esp            0xffffd8c0       0xffffd8c0
ebp            0xffffd8c8       0xffffd8c8
esi            0x0      0
edi            0x0      0
eip            0x804805a        0x804805a <_start+6>
eflags         0x282    [ SF IF ]
cs             0x23     35
ss             0x2b     43
ds             0x2b     43
es             0x2b     43
fs             0x0      0
gs             0x0      0
(gdb)
```
 
This shows us that the value of ebp is `0xffffd8c8` which corresponds to an address in our stack. This value is very close to that of esp (`0xffffd8c8`), as the two are usually used in conjunction to traverse the stack.

We can see there is also the register `eip` with the value `0x804805a` at a much lower value. This is the extended instruction pointer that points to the current instruction that our application is running. The application's instructions reside in an area of memory referenced in our assembly code as the `.text` section. 

Ry convention on Linux, the stack is at the top of memory and our intructions are close to the bottom of memory. The way this is done is by virtual paging so it appears to the application that it has all of physical memory, but it is only mapped pages that appear, so it is just an illusion until the application needs to read or write from a memory address. The size of virtual memory is all of the addressable memory space, can be many times larger than the physical memory space, and appears to the application that it is exclusively owned by the application.

Coming back to the `ecx` register, we can eXamine what is in memory at the address referenced by `ecx` using the `x/` notation.

The `x/` notation takes up to three components. For example:

`x/5s ($ecx)`

Examines: 
- the first 5
- strings
- starting at the address pointed to by the register `ecx`

and will output something similar to the following:

```
(gdb) x/5s ($ecx)
0xffffd9d3:     "test"
0xffffd9d8:     "LESSOPEN=| /usr/bin/lesspipe %s"
0xffffd9f8:     "HOSTNAME=fc9dd5d874b7"
0xffffda0e:     "SHLVL=1"
0xffffda16:     "HOME=/root"
(gdb)
```

Many other formats can be printed including (c)har (d)ecimal and (t)binary among others. For more information on this, enter `help x` from the GDB REPL.

We can clearly see our first command line argument here, "test" followed by some environment variables.

The remainder of the application prints the first 4 characters of the string pointed to by the address in `ecx`, then exits gracefully. The parenthesis around the register indicates we are interested in examining the address pointed to by the register and not the value of the register itself. From the GDB REPL, when referencing registers, you may have noticed that they are referenced with a dollar sign (`$`) instead of the percentage sign (`$`) used in our assembly code. This is a small but important difference to note.

To run the app to completion enter `continue` or just `c`. 

We will see the command line argument printed out followed by the app exiting as below.

```
(gdb) c
Continuing.
test[Inferior 1 (process 24) exited normally]
(gdb)
```

Notice that there is a message directly after our string. This is because the command line argument did not have a newline character (`\n`) and so the GDB message was printed immediately after it.

# Getting the length of our argument on the command line

Up to this point, we should have an app that can read and print the first four characters of the first argument passed on the commandline when our application is called.

Using our `docker-shell` environment, the following should work:

```bash
root@9a172ec363ca:/gas-asm-fib# ./make-app fib2
root@9a172ec363ca:/gas-asm-fib# ./fib2 test
testroot@9a172ec363ca:/gas-asm-fib#
```

But what if we were to enter `./fib2 testing`? We'd still just get `test` returned. Or worse still, if we enter less than 4 characters we get garbage printed from memory to make up the 4 characters. This is far from ideal for processing our commandline arguments.

What we need is a way of determining the length of the argument in the stack. This is where the instruction pair `repne scasb` comes in useful.

The following is an implementation of:

- Finding the length of our first argument
- Printing the string pointed to by the address of the first argument to the length determined

I'll show the code below then describe how it works.

_fib3.s_
```asm
# framework - get first argument from the command line and print to stdout
.section .text
.globl _start
_start:
    nop
    movl %esp, %ebp     # take a copy of esp to use
    addl $8, %ebp       # address of first arg in stack
    movl (%ebp), %edi   # move arg address into esi for scasb
    push %edi           # store the string address as edi gets clobbered

    movl $50, %ecx      # set ecx counter to a high value
    movl $0, %eax       # zero al search char
    movl %ecx, %ebx     # copy our max counter value to edx
    cld                 # set direction down
    repne scasb         # iterate until we find the al char

    movl %ecx, %edx     # move count into edx
    subl %ecx, %ebx     # subtract from our original ecx value
    dec %ebx            # remove null byte at the end of the string from the count
    pop %ecx            # restore our string address into ecx
    mov %ebx, %edx      # move our count value to edx for the int 80 call

    movl $4, %eax       # set eax to 4 for int 80 to write to file
    movl $0, %ebx       # set ebx for file to write to as stdoout (file descriptor 0)
    int $0x80           # make it so
    movl $1, %eax       # set eax for int 80 for system exit
    movl $0, %ebx       # set ebx for return code 0
    int $0x80           # make it so again
```

Building and running this from the `docker-shell` we can see that the full first argument is now displayed:

```
root@8ec496c15833:/gas-asm-fib# ./make-app fib3
root@8ec496c15833:/gas-asm-fib# ./fib3 testing
testingroot@8ec496c15833:/gas-asm-fib#
```

Excellent, I hear you say, but how does it work? I'm glad you asked.

So what happens from the line `movl $50, %ecx` up to `repne scasb`, the registers are being set up to run `repne scasb`.
The instructions `repne scasb` work on strings specifically to determine their length. It iterates through memory starting at the address pointed to by `edi` and compares each byte to the byte stored in the lower byte of `eax`.

The following is a table of what registers need to be set and what they do:

|Register | Usage |
|---|---|
|`ecx`|A value to count down from for the length of the string. We set this to 50 to assert that we are only taking strings of up to 50 bytes|
|`eax`|The byte value to search for placed in the lower byte of the register, `al` (in this case we are searching for a null byte (`0x00`)|
|`ebx`|A copy of our counter `ecx`. This is used later to determine the actual length of the string|
|`edi`|A pointer to the start of our sting array|

The register `ecx` is considered the defacto count register on the x86 architecture and `repne scasb` implicitly uses this register to iterate, starting from the address in the `edi` register until either:
- it finds a byte in memory specified in the lower byte of the `eax` register (in our case `0x00`)
- the `ecx` register reaches 0

The opcode `cld` sets a flag for the count direction as downward.

And so `repne scasb` iterates, doing the following:
- starting at the address `edi`
- decrementing `ecx` by 1
- incrementing `edi` 
- comparing the byte poined to by the new location of `edi` to the value stored in the lower byte of `eax`. 

For example, if we were to call from the commandline:

`$ ./fib3 testing`

and then we inspect our registers after stepping past `repne scasb`, we should see a value in `ecx` that is *50 - ( len('testing\0') - 1 )*. Which effectively equals 42. As `repne scasb` includes the byte it is searching for in the count, we need to subtract 1 from the length of our string. We then subtract the result in `ecx` from our original maximum length of 50 to find the actual length of the string that we want to print.

This gives **50 - 43 = 7** and is what the following lines do:

```asm
    movl %ecx, %edx     # move count into edx
    subl %ecx, %ebx     # subtract from our original ecx value
    dec %ebx            # remove null byte at the end of the string from the count
    pop %ecx            # restore our string address into ecx
    mov %ebx, %edx      # move our count value to edx for the int 80 call
```
 
Our string length is copied from `ebx` into the `edx` register so that the `int 0x80` call to write to stdout can then be performed to print the correct number of characters to the screen.
There is also a `push` and `pop` instruction in `fib3.s` that firstly stores a copy of the address of our argument in `edi` then restores it into `ecx`. This is done as `ecx` is used for the address of the string to write to stdout, and the `repne scasb` instruction loop destroys this value in the `edi` register. Using `push` and `pop`, we can preserve the address of the string on the stack from `edi` and restore it into `ecx` when we need to use it later on._

# Making things easier to understand using function calls

Assembly language has the concept of functions that allow for reuse of code. The two instructions invloved with this are `call` and `ret`. What these do is allow jumping to a label in your code with a `call` and then return to the position in the calling code using a `ret`. Each time a `call` is encountered, the current address of the instruction, pointed to by the register `eip` is pushed onto the stack, the stack counter is decremented accordingly and the `eip` register is set to the memory address pointed to by the  new instruction of the label specified by the call instruction.

For example, if we had a label somewhere in our code called `print_value:` and somewhere else in the code an instruction such as `call print_value`, this would move execution control of our application to the label specified. We can then do some processing as part of a subroutine and when we are finished printing a value, for example, we can return control back to the calling code and at the instruction just after the `call print_value` instruction.

The instruction `ret` doesn't take a label as an operand as it just uses the last address on the stack. It is for this reason that it is important to make sure the stack always keeps the return address as the last value placed on the stack when using `call` and `ret`.

It is also for this reason that when using the stack for local variables, that we don't actually use the `esp` register for any operations, but a copy of it in `ebp`. This way we can be certain that the stack pointer (`esp`) is always a correct return address. As our local variables are just that, when we return control to the calling code, the variables we have placed there are no longer considered important and the memory addresses are considered free to be written over.

The following is what we have done so far as being refactored into separate functions.

_fib4.s_
```asm
# framework - refactor into separate functions
.section .text
.globl _start

# entrypoint of application
_start:
    nop
    movl %esp, %ebp     # take a copy of esp to use
    addl $8, %ebp       # address of first arg in stack
    movl (%ebp), %edi   # move arg address into esi for scasb
    push %edi           # store the string address as edi gets clobbered
    call get_string_length
    pop %ecx            # restore our string address into ecx
    call print_string
    call exit

# get length of string pointed to by edi and place result in ebx
get_string_length:
    movl $50, %ecx      # set ecx counter to a high value
    movl $0, %eax       # zero al search char
    movl %ecx, %ebx     # copy our max counter value to edx
    cld                 # set direction down
    repne scasb         # iterate until we find the al char
    movl %ecx, %edx     # move count into edx
    subl %ecx, %ebx     # subtract from our original ecx value
    dec %ebx            # remove null byte at the end of the string from the count
    ret

# print the string in ecx to the length of ebx
print_string:
    mov %ebx, %edx      # move our count value to edx for the int 80 call
    movl $4, %eax       # set eax to 4 for int 80 to write to file
    movl $0, %ebx       # set ebx for file to write to as stdoout (file descriptor 0)
    int $0x80           # make it so
    ret

# exit the application
exit:
    movl $1, %eax       # set eax for int 80 for system exit
    movl $0, %ebx       # set ebx for return code 0
    int $0x80           # make it so again

```

We can see here that we still have the label `_start` but we also have the additional labels of:
- get_string_length
- print_string
- exit

This helps greatly for the readability of our code. We can see that all the calls are done from the `_start` label and each section of code concludes with a `ret` instruction, except for the `exit` label, which exits our application.

When using labels and functions, it helps to add comments at the start of the section of code indicating what the expected state of any registers and memory should be, what is actually done by the code and what is expected to be returned by the code. The comments above are very brief, but we will see later greater use of this approach.

Another thing to note is that breaking our code into small, reusable chunks like this allows us to split our code into separate files for reuse across multiple projects which further enables reuse.

# Converting a string to a number

Now that we can read arguments from the command line and print them to the screen (sdout), we want to actually start using the command line argument to specify the number of iterations for our Fibonacci sequence. This involves reading the string from the command line and converting it to a number. The input from the command line will be in ASCII format, and so a numeric value in a string is not an actual numeric value for use in our application - hence the conversion.

_fib5.s_
```asm
# framework - convert string to number
.section .text
.globl _start

# entrypoint of application
_start:
    nop
    movl %esp, %ebp     # take a copy of esp to use
    addl $8, %ebp       # address of first arg in stack
    movl (%ebp), %edi   # move arg address into esi for scasb
    push %edi           # store the string address as edi gets clobbered
    call get_string_length
    pop %edi             # get edi string addr then push it as we are going to clobber it
    push %edi
    call long_from_string
    cmpl $0, %eax
    je exit
    call fibonacci
    call exit

# using address of string edi
# use eax register to store our result
# jump to done if not a number
# if numeric
# subtract 48 from the byte to convert it to a number
# multiply the result by 10 and add the number to eax
# multiply the result register eax by 10
# loop until the ecx counter has reached a non-numeric (null byte) and return

long_from_string:
    xor %eax, %eax # set eax as our result register
    xor %ecx, %ecx # set ecx(cl) as our temporary byte register
.top:
    movb (%edi), %cl
    inc %edi
    # check if we have a valid number in edi
    cmpb $48, %cl  # check if value in ecx is less than ascii '0'. exit if less
    jl .done
    cmpb $57, %cl  # check if value in ecx is greater than ascii '9'. exit if greater
    jg .done
    sub $48, %cl
    imul $10, %eax
    add %ecx, %eax
    jmp .top
.done:
    ret

fibonacci:
    ret

# get length of string pointed to by edi and place result in ebx

get_string_length:
    movl $50, %ecx      # set ecx counter to a high value
    movl $0, %eax       # zero al search char
    movl %ecx, %ebx     # copy our max counter value to edx
    cld                 # set direction down
    repne scasb         # iterate until we find the al char
    movl %ecx, %edx     # move count into edx
    subl %ecx, %ebx     # subtract from our original ecx value
    dec %ebx            # remove null byte at the end of the string from the count
    ret

# print the string in ecx to the length of ebx

print_string:
    mov %ebx, %edx      # move our count value to edx for the int 80 call
    movl $4, %eax       # set eax to 4 for int 80 to write to file
    movl $0, %ebx       # set ebx for file to write to as stdoout (file descriptor 0)
    int $0x80           # make it so
    ret

# exit the application

exit:
    movl $1, %eax       # set eax for int 80 for system exit
    movl $0, %ebx       # set ebx for return code 0
    int $0x80           # make it so again
```

This is quite a bit more code than before. We'll go though it in detail now so it all makes sense.

## Some new functions

We can see here that two new functions have been added in the form of `long_from_string` and `fibonacci`. The label `fibonacci` is just a placeholder at the moment, but `long_from_string` has our logic for converting a string into a 32-bit long.

The first thing you may notice in `long_from_string` is that we now have two extra labels:
- `.top`
- `.done`

These labels start with a period (`.`) to indicate to the assembler that they are labels local to the file. This is useful when large amounts of assembly code are compiled so that the compiler doesn't confuse having two labels with the same name.

From a visual inspection of this function, we can see that there are some `cmp` instructions and some instructions staring with a `j` such as `jl`, `jg` and `jmp`. These are jump instructions and are used in conjunction with our labels to tell the `eip` register to jump to a certain location in our code to continue running. This is not too dissimilar to the `call` instructions described previously, except they don't modify the stack with a return address.

Jump instructions are very useful for loop type logic such as `for`, `while` and `do` loops, and even conditional statements like `if` and `case` that you may be familiar with from other languages. They can, however have performance impacts for performance-critical sections of code under some circumstances.

When the function `long_from_string` is called from the `_start` section, we make the assumption that the address of our string has been placed in the `edi` register, ready for processing. We also assume that the calling function knows that the numeric result will be placed into the `eax` register at completion of the function. It is a convention in assembly language that the result from a function is placed into the `eax` register. For more complex return types, an address that points to an array or struct or some other complex type can be specified in `eax`. This same convention is used in the C language for returning values from functions.

## Zeroing registers

At the very beginning of this function, we use `xor` to zero-out the `eax` and `ecx` registers as these are used for calculating our result. Using `xor` is seen as an efficient way of setting a register to zero. An alternative is to use something like `movl $0, $eax`. The `xor` method under older processors used less clock-cycles than `mov`. There is probably not much difference between the two these days.

We then enter our loop, starting at '.top', which points to the next instruction, `movb ($edi), $cl`. What this `mov` instruction does is copy the first byte from our string array into the lower part of the `ecx` register, or `cl`. This is in effect the first byte of the first argument from the command line.

For example, if the command line were `./fib5 123`, then the `cl` register would now contain the ASCII representation of the character '1'. This ASCII representation is not actually the value 1, but the decimal value 49 which represents the ASCII character '1'. There are many references online for conversions between ASCII representations and their numerical value. There is a [chart here](https://www.cs.cmu.edu/~pattis/15-1XX/common/handouts/ascii.html) for your convenience.

## Register sizes and layout

Another thing to note about this instruction is that we have only copied a single byte (8 bits) into `cl`, and that `cl` actually makes up part of the `ecx` register. There are both `cl` and `ch` registers that are a subset of `ecx` on 32 bit CPU architectures, and all 3 are a subset of `rcx` on 64 bit CPU registers.

The following is a diagram of how these fit together:

<table >
  <tr>
    <td colspan="1" align="center">bits 56-63</td>
    <td colspan="1" align="center">bits 48-55</td>
    <td colspan="1" align="center">bits 40-47</td>
    <td colspan="1" align="center">bits 32-39</td>
    <td colspan="1" align="center">bits 24-31</td>
    <td colspan="1" align="center">bits 16-23</td>
    <td colspan="1" align="center">bits 8-15</td>
    <td colspan="1" align="center">bits 0-7</td>
  </tr>
  <tr>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
    <td colspan="1" align="center">--rcx---</td>
  </tr>
  <tr>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center">--ecx---</td>
    <td colspan="1" align="center">--ecx---</td>
    <td colspan="1" align="center">--ecx---</td>
    <td colspan="1" align="center">--ecx---</td>
  </tr>
  <tr>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center"></td>
    <td colspan="1" align="center">--ch----</td>
    <td colspan="1" align="center">--cl----</td>
  </tr>
</table>

This type of layout is the same for all of the general purpose registers including `eax`, `ebx`, `ecx` and `edx`.

There is a good diagram and reference of these registers [here](http://flint.cs.yale.edu/cs421/papers/x86-asm/asm.html).

## Processing the string

Coming back to our function, we then increment the `edi` pointer to point to the next memory location in our string, ready for processing, but continue processing the value in `cl`.

Next, we compare our value in `cl` to both 48 and 57 and jump to `.done` if the value falls outside of this range. This is done as the ASCII string values `0-9` fall in this range of decimal values 48-57. 

Our conditions for exiting the loop on an invalid input are thus:

- After the comparison to 48, if the result is lower than 48, we jump to `.done` with `jl`, or jump if lower.
- After the comparison to 57, if the result is greater than 57, with 'jg', we also jump to `.done`.

As our string is null-terminated (with a value of 0x00 or `$0`), the usual case would be that we jump to `.done` after all characters have been processed or a non-numeric string character is encountered as the value 0 is outside the range of 48-57.

After these comparisons, the actual string processing is done as per below:

```asm
    sub $48, %cl
    imul $10, %eax
    add %ecx, %eax
    jmp .top
``` 

At this point it is relatively straight-forward what occurs, but I will describe it here for completeness:

- Our value in `cl` has 48 subtracted from it to convert it to a decimal value.
- Our existing value in our result register `eax` is multiplied by 10 (the first iteration will have no effect as it starts at 0)
- Our decimal value in `cl` is added to our result register `eax`
- The whole process repeats until the end of the string

One thing to note about `imul` is that it multiplies a register (e.g. `eax`) by a value (e.g. `$10`) and places the result in the same register (e.g. `eax`). This is considered a destructive operation as the initial value is not preserved.

# The actual Fibonacci algorithm

It has been a long time coming to this point where I can actually show you the implementation of Fibonacci being calculated with assembly language. In _fib6.s_ we implement this logic. I will show an excerpt of it below for brevity and describe how it works.

```asm
# input: eax holds our fibonacci n
# processing: iterate the fibonacci sequence n times
# output: return our fibonacci result in ebx

fibonacci:

    pushl %ebp                    # preserve ebp as we are going to use it to store our stack pointer for the return call
    mov %esp, %ebp                # copy the stack pointer to ebp for use

    mov %eax, %ebx                # make a copy of our fib(n) value for allocating an array on the stack
    addl $2, %ebx                 # add 2 extra spaces to the array size in case n=1 or n=0
    shl $2, %ebx                  # multiply by 4 as we are using longs (32 bits)
    subl %ebx, %esp               # add the size of our array to the stack to allocate the required space
    xor %ecx, %ecx                # set our counter to zero
    movl %ecx, (%esp, %ecx, 4)    # initialise our array with 0 for esp[0]
    incl %ecx                     # increase our counter
    movl %ecx, (%esp, %ecx, 4)    # initialise our array with 1 for esp[1]
    incl %ecx                     # our counter/iterator should be at 2 now
    addl $2, %eax                 # increment our number of iterations by two as our ecx counter will be two higher from initialising our array

.fib_loop:                        # we begin our for loop here

    cmp %eax, %ecx                # compare our counter (ecx) to n (eax) if it's greater or equal, we're done
    jge .fib_done
    movl -4(%esp, %ecx, 4), %ebx  # get the value in the stack at esp-1 from our current stack pointer location
    movl -8(%esp, %ecx, 4), %edx  # get the value in the stack esp-2 from our current stack pointer location
    addl %edx, %ebx               # add the values esp-1 and esp-2 together
    movl %ebx, (%esp, %ecx, 4)    # place the result in the current stack location
    incl %ecx                     # bump our counter
    jmp .fib_loop                 # loop again

.fib_done:

    movl %ebp, %esp               # move our copy of the stack pointer back to esp
    popl %ebp                     # retrieve the original copy of ebp from the stack
    ret
```

The first thing to note is that the comments as the top of the function specify `input`, `processing` and `output`. I find this a good habbit for functions as it makes explicit:

- what the function expects
- what the function does
- what is expected at completion of the function

## Creating the stack space for our array

The `fibonacci` function introduces something not shown in the previous sections with how to set up the `esp` and `ebp` registers in order to use a local stack for our function. Up to this point most temporary variables have just been stored in registers (with the exception of the `push` and `pop` instructions). As the Fibonacci sequence uses an array to store results, memory on the stack needs to be allocated and used for this purpose.

This is the reason for the following instructions at the start of our function:

```asm
    pushl %ebp                    # preserve ebp as we are going to use it to store our stack pointer for the return call
    mov %esp, %ebp                # copy the stack pointer to ebp for use
```

And for these at the end of our function:

```asm
    movl %ebp, %esp               # move our copy of the stack pointer back to esp
    popl %ebp                     # retrieve the original copy of ebp from the stack
```

In order to set up our local stack, the first thing that is done is that we save a copy of the base pointer `ebp` on the stack and copy the value of the stack pointer `esp` into the base pointer so that we have a copy of it and can restore it at the end of our function. At the end of our function we can copy `ebp` back into `esp` and restore the value of `ebp` from the stack as it was at the start of the function.

After the first instructions, we can modify the stack pointer as we like and all changes will only be local to the function and will essentially be destroyed, for all intents and purposes, once the function completes. Using this approach to the stack pointer with functions is somewhat of a convention in assembly language.

## The Fibonacci logic in assembly

There are essentially two parts to the logic in this section that can be seen as:

- everything between the start of the function up to `.fib_loop`, which sets up our variables
- everything between the label `.fib_loop` up to `.fib_done`, which processes the Fibonacci sequence in a loop

## Setting up our variables

Casting our minds back to the original C source code in [#Some starting knowledge](), we would like to design our assembly code to replicate the C code. The C code is similar in design with two sections where a counter and an array are defined, followed by a loop to calculate the Fibonacci number. The function assumes `eax` is set with the value for the number of iterations, _n_ (plus 2), and returns the result in the register `ebx`.

The counter _i_ is relatively simple and the register `ecx` is used for this purpose.

The addition of 2 to `eax` is done so that the number of iterations matches what the expected number in the sequence is. Our counter is the count of the index in our array, and not the number of iterations of the sequence. By adding 2 to `eax`, our `ecx` counter can remain our array index and we can compare it to `eax` to get the correct number of iterations. This is similar (but not the same as) the C code where the for loop is initialised to `i=2`.

## Array memory allocation

What is less clear is how the array _f_ is allocated and initialised. The array requires local memory on the stack as there can be more values than can fit in the registers. As a side-note, f the C code were to use `malloc()` to create the array, this would be a slightly different implementation and would allocate space on the heap, and not the stack.

For the array memory allocation, we do the following:

- copy our iteration count _n_ in `eax` into `ebx` (as we want to keep `ecx` for counting later in the loop)
- add two to the value (as we want our array size to be two more than the number of iterations)
- multiply the value by 4 (as each value is a long that requires 32 bits, or 4 bytes)
- subtract the result from `esp` to give our new 'top' of the stack. ( I say top, but in actuality the top of the stack is at a lower memory address, hence the subtract)

This can be inferred from the following instructions:

```asm
    mov %eax, %ebx                # make a copy of our fib(n) value for allocating an array on the stack
    addl $2, %ebx                 # add 2 extra spaces to the array size in case n=1 or n=0
    shl $2, %ebx                  # multiply by 4 as we are using longs (32 bits)
    subl %ebx, %esp               # add the size of our array to the stack to allocate the required space
```

It is important to note that the `shl` instruction is actually a shift-left of the value in the `ebx` register, which in effect multiplies the value in `ecx` by 2^2, or 4 by shifting all the 1 bits in the register to the left. This could have also been achieved by the instruction `imul $4, $ebx`, but as the multiplication is a power of 2, it takes less cpu cycles if we use `shl`. 

## Variable initialisation

Once this is done, we zero our `ecx` counter register:

```asm
    xor %ecx, %ecx                # set our counter to zero
```

Then initialise the first two values in the array to 0 and 1 respectively, while incrementing our `ecx` register.

```asm
    movl %ecx, (%esp, %ecx, 4)    # initialise our array with 0 for esp[0]
    incl %ecx                     # increase our counter
    movl %ecx, (%esp, %ecx, 4)    # initialise our array with 1 for esp[1]
    incl %ecx                     # our counter/iterator should be at 2 now
```

## Indexed memory

The format of `mov` above has not been shown previously as we have mainly been working with registers and pointers for counting strings. The format of `mov` with brackets around the second operand allows us to move, or copy the value of a register into an indexed memory location. This allows us to reference a greater number of memory locations than we could otherwise with a single register. This is mainly used when writing to memory as opposed to just reading from it.

The parenthesis around the second opcode tells the cpu to move the value in first opcode `ecx` into the memory address pointed to by the indexed location. The indexing between the brackets has the following form:

&nbsp;&nbsp; **(offset, index, multiplier)**

In practice this means:

- move the value in `ecx`
- into the memory location with the value of:
  - the address pointed to by `esp` + ( the value of `ecx` * 4 )

The multiplier 4 is used as we are working with long value types that are 4 bytes in length each.

For the first `mov` above, `ecx` is zero and so the value zero is placed into the lowest memory location in our stack `f[0]`. As **0 * 4 = 0**, the value of the first iteration is just that of `esp`.

Next we increment our counter `ecx` so that it contains 1. The second `mov` then places the value 1 into the next position up in memory in our stack, 4 bytes up from the previous `mov` instruction. Our `ecx` counter is then incremented again.

This is closely reflects our C code with the lines:

```c
  f[0] = 0;
  f[1] = 1;
```

It is important that the first two values are pre-populated as the loop makes the assumption that the last two values of the Fibonacci sequence already exist.

## Running the loop to completion

The logic below shows the heart of our Fibonacci implementation:

```asm
.fib_loop:                        # we begin our for loop here
    cmp %eax, %ecx                # compare our counter (ecx) to n (eax) if it's greater or equal, we're done
    jge .fib_done
    movl -4(%esp, %ecx, 4), %ebx  # get the value in the stack at esp-1 from our current stack pointer location
    movl -8(%esp, %ecx, 4), %edx  # get the value in the stack esp-2 from our current stack pointer location
    addl %edx, %ebx               # add the values esp-1 and esp-2 together
    movl %ebx, (%esp, %ecx, 4)    # place the result in the current stack location
    incl %ecx                     # bump our counter
    jmp .fib_loop                 # loop again
```

In effect, it repeats the loop for the number of times specified on the coommand line (which we have previously placed in `eax`) and jumps to `.fib_done` if it has been reached, otherwise calculates the next number in the sequence.

## Calculating the sequence

As we have already initialised our array with the values 0 and 1 for `f[0]` and `f[1]` respectively, our loop will always be able to calculate the next number in the sequence. The line `movl -4(%esp, %ecx, 4), %ebx` shows another variation of indexing into memory with the `-4` out the front indicating a second, relative offset to the address within the parenthesis. What this does in effect is get the value 4 bytes down in memory (up the stack) from the current location of (esp + ecx * 4); effectively:

&nbsp;&nbsp; ( esp + ( ecx * 4 ) ) - 4

This corresponds to the value in our C code of `f[i-1]`. The value in memory is placed into the `ebx` register for calculating the next value in the sequence.

The same is true of the line `movl -8(%esp, %ecx, 4), %edx`, however it represents the value `f[i-2]` in the array and is placed into the `edx` register.

The format of these `mov` instructions can be summarised as:

&nbsp;&nbsp; **relative\_offset(absolute\_offset, index, size)**

The `ebx` and `edx` registers are then added together and the result placed into the array in the position `f[i]`. Our counter is then incremented and the loop repeats.

The register `ecx` can thus be seen as our index into the array throughout this process. Once the loop is complete we will have our value at the highest address in our stack, pointed to by `esp` which we can use to print to stdout as our result.

## Printing our result

The final piece in the puzzle for putting together our program is almost the reciprocal for one of the functions we created earlier, namely `get_long_from_string`. The following is the function `print_long`, from _fib7.s_ which takes an unsigned long, converts it to a string then prints the string to stdout.

```asm
# print a 32-bit long integer
# input: ebx contains the long value we wish to print
# process:
#  set a counter for the number of digits to 0
#  start a loop and check if value is zero - jump to done if so
#  divide the number by 10 and take the remainder as the first digit
#  add 48 to the number to make it an ascii value
#  store the byte in the address esp + ecx
#  increment the counter
#  jump to start of loop
# output: no output registers

print_long:

    push %ebp
    mov %esp, %ebp              # copy the stack pointer to ebp for use
    add $10, %esp               # add 10 to esp to make space for our string
    mov $10, %ecx               # set our counter to the end of the stack space allocated (higher)
    mov %ebx, %eax              # our value ebx is placed into eax for division

.loop_pl:

    xor %edx, %edx              # clear edx for the dividend
    mov $10, %ebx               # ebx is our divisor so we set it to divide by 10
    div %ebx                    # do the division
    addb $48, %dl               # convert the quotient to ascii
    movb %dl, (%esp, %ecx, 1)   # place the string byte into memory
    dec %ecx                    # decrease our counter (as we are working backwards through the number)
    cmp $0, %ax                 # exit if the remainder is 0
    je .done_pl
    jmp .loop_pl                # loop again if necessary

.done_pl:

    addl %ecx, %esp             # shift our stack pointer up to the start of the buffer
    incl %esp                   # add 1 to the stack pointer for the actual first string byte
    sub $10, %ecx               # as we are counting down, we subtract 10 from ecx to give the actual number of digits
    neg %ecx                    # convert to a positive value
    mov %ecx, %edx              # move our count value to edx for the int 80 call

    mov %esp, %ecx              # move our string start address into ecx
    movl $4, %eax               # set eax to 4 for int 80 to write to file
    movl $0, %ebx               # set ebx for file to write to as stdoout (file descriptor 0)
    int $0x80                   # make it so

    movl %ebp, %esp             # move our copy of the stack pointer back to esp
    popl %ebp                   # retrieve the original copy of ebp from the stack
    ret
```

We can see the same `esp` and `ebp` store and retrieve pattern at the start and end of the function as described earlier. Additionally, there is another local loop present  in the form of `.loop_pl` and `.done_pl` for iterating the value we are printing.

The way this function works is by dividing our number by 10, taking the remainder of the division as a digit, converting it to an ASCII representation, then placing this value into our stack for printing further down in the function. The string does not need to be null-terminated with a 0x00 byte as the syscall (`int 0x80`) to print the value uses the length of the string.

After saving the stack pointer, the stack pointer is increased by 10 bytes to allow enough space for the string we would like to print. As the long value stores up to 32 bits, this means our number can be anything up to the value 4,294,967,296, which is 10 digits long.

Our value in `ebx` is then copied into `eax` ready for operating on.

The loop then begins where the heart of the transformation occurs. The key instruction here is the line `div %ebx` which both takes a bit of setting up and post-processing. This instruction implicitly takes the value in `eax`, divides it by the value specified in the operand (in our case `ebx`), then places the result in `eax` and the remainder in `edx`. This is why `edx` is `xor`ed at the start of the loop.

As we are only dividing by 10, we can be certain that our result is in the lower byte of `edx` (as 8 bytes can have a value between 0 and 255), and so we move this lower byte `dl` into the indexed stack pointer location `(%esp, %ecx, 1)`. Following this, we decrement our `ecx` counter and compare the quotient of our division to 0. If the quotient equals 0, we know there are no digits to calculate and we can jump to `.done_pl`.

## Doing things the hard way

The five lines after `.jump_pl` involve getting the correct count of the digits processed. 

```asm
    addl %ecx, %esp             # shift our stack pointer up to the start of the buffer
    incl %esp                   # add 1 to the stack pointer for the actual first string byte
    sub $10, %ecx               # as we are counting down, we subtract 10 from ecx to give the actual number of digits
    neg %ecx                    # convert to a positive value
    mov %ecx, %edx              # move our count value to edx for the int 80 call
```

By subtracting our result with the `sub` instruction then using `neg` to convert the negative number to a positive one, we will have the actual 
 count of digits to print and can place it in the `edx` register for the syscall.

There is a bit of arithmetic here as our `ecx` was being decremented by 1 from 10 for each digit, and so the actual count is **-1 * ( count - 10 )**. This is a rather rountabout way of doing this calculation and it would have actually been simpler to just count up. However this way afforded me the opportunity to introduce the `neg` instruction which takes a 2's compliment of the value in the specified register and is essentially multiplying the register by -1.

# Conclusion 

If you have followed along, congratulations! This is a rather lengthy tutorial but you should now at least know some of the basics of assembly language and how processing is done very close to the CPU. This can have all sorts of uses from optimising higher-level code to debugging complex multi-threaded applications and everything in-between.

If you have compiled the _fib7.s_ code, you should be able to see the sequence in the `docker-shell` environment similar to below:

```bash
root@608eb3f49ac5:/gas-asm-fib# ./fib7 0
root@608eb3f49ac5:/gas-asm-fib# ./fib7 1
1root@608eb3f49ac5:/gas-asm-fib# ./fib7 2
2root@608eb3f49ac5:/gas-asm-fib# ./fib7 3
3root@608eb3f49ac5:/gas-asm-fib# ./fib7 4
5root@608eb3f49ac5:/gas-asm-fib# ./fib7 5
8root@608eb3f49ac5:/gas-asm-fib# ./fib7 6
13root@608eb3f49ac5:/gas-asm-fib#
```

## From here to there - improving the code

This is by no means the ultimate Fibonacci sequence generator (or even a sequence generator at all - it only prints the last value in the sequence, after all), and there are many tricks and tips to making this faster, more robust and more extensible both in design and application.

A follow on blog from this may include:

- Optimising calculations using an alternative Fibonacci alogrithm implementation with the corresponding performance comparison
- Optimising calculations using the FPU math coprocessor, MMX, XMM, or SIMD instructions
- Handling of a higher range of numbers
- Designing for reuse with multiple files

Until next time, happy coding!

