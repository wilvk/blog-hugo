---
title: "Creating a Fibonacci Generator in Assembly"
date: 2018-07-18T21:12:38+10:00
draft: true
---

Stemming from my interest in Assembly Language, I was in search for a practical example to start exploring this language. I thought I'd write a simple ASM application to compute Fibonacci to a given number of iterations. I defined some criteria and restrictions for this little project and came to the following:

- It must take a value from the command line for the number of iterations
- It must not use the glibc standard library
- It must print the final output on the screen

Other criteria that may also be considered are:

- Input validation
- Error Checking
- Calculating for very long numbers
- Equation Optimisation
- Using extended maths operations

# A beginning

I found a good blog about different approaches to Fibonacci Equations [here](https://www.geeksforgeeks.org/program-for-nth-fibonacci-number/) and armed with the new knowledge from reading a couple of books on the subject I set out to implement this.

## Using some starting knowledge

The basis was decided to be a space optimised method using a dynamic programming approach. This effectively means using an array on the stack to hold our values while giving O(n) time and space complexity.

The code used as a basis is shown below:

```
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
      /* Add the previous 2 numbers in the series
         and store it */
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

# Overview of the implementation

The implementation involved the following functionality:

- Getting the length of the command line argument
- Converting the command line argument from a string to a long (unsigned 32 bit number)
- Validating the command line argument
- Calculating Fibonacci
- Printing the long as a string to stdout

# Beginning the implementation

Ok, so we have a basis for our algorithm, but how do we implement it in ASM? There are numerous formats of Assembly Language including NASM, GAS and MASM among others. I'm working on a Linux box and so MASM is out of the question. I decided to go with GAS as it is cross-platform and is a good general purpose assembler. This also gives me the ability to cheat in my implementation (which I'm not), as using `gcc`, a valid C application can be converted into intermediate GAS assembly language.

I decided the number of iterations of Fibonacci would be determined from the command line as an argument to the application. It would expect a valid number from the user and convert this from a string to a number. The Fibonaci sequence would then be iterated n number of times.

The Fibonacci algorithm would have to create an array of lenth n+2 on the stack based on the user input. Another variable (as the variable i shown above) would be used as a counter.

The first two elements of the array would be pre-populated with 0 and 1 respectively, as all Fibonacci sequences have these numbers and are used when calculating new values. This can be considered a type of 'seed' for our algorithm.

A loop is then started with the counter set to 2 that iterates until the counter is greater than or equal to the number of iterations (n); and where the counter is increased by 1 on each iteration.

Inside the loop, for the array position in the stack, and based on the counter value, the i - 1 value is added to the i - 2 value on the stack and the result is placed on the stack at the current location allowing it to be used for the next iteration.

The final value placed on the stack then needs to be converted to a string and printed to standard out. 

Once the final value is determined, a function is called to convert the number into a string and print it to stdout.

The application then needs to exit gracefully and return control to the shell.

# 1. Starting with a framework

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

We're sort of working backwards a bit here as we've implemented our entrypoint (\_start) and our syscall to exit first. The syscall using eax=1 and ebx=0 with int 0x80 allows our program to exit gracefully. As noted by the comment, we shall fill in the logic from the middle as we progress.

Ok, so we've got a framework of GAS assembly to build on, but how do we convert this into an application?

Well, it's easy. I'm glad you asked. Firstly we need to make sure our gas assembler is installed. I'm on Fedora, so the command is `sudo dnf install binutils`.

Then we create a small shell script to run the gas assembler (as) and the linker (ld) on the output of the assembler.

Say for example we call our assembly code file fib.s, to run the assembler we would run `as -gstabs -o fib.o fib.s`

- `-gstabs` adds debugging info
- `-o fib.o` specifies our object file
- `fib.s` is our source code

The object file is a binary with all the symbols and source included, which included the raw machine code to run.
We then link the resultant object file to convert our object file into a binary application. The linker also converts our symbolic references (such as \_start in our case) to relative numeric references to where the symbols will reside in memory when the binary is run as an application.

If we were to run `ld -o fib fib.o`, we should now have a new binary application with the permissions 0755 set. If you run the application it will do nothing. But more importantly, it won't segfault, which is a good thing and indicates the application has exited gracefully.

`-o fib` specifies the output application name
`fib.o` specifies the object file to use.

The linker has other features (which I won't discuss here) that make your life easier as an assembly language programmer.

# Reading from the process arguments and converting to a numeric value

The arguments are already placed into the stack for us, along with all the environment variables of the parent process. All we should have to do is find the value in the stack where the argument is and then convert it to a string and print it to stdout.

The stack is upside down, and so we traverse up the stack to find the first argument. The stack is composed as follows.

Stack |
--- |
< pointer to last string argument > |
... |
< pointer to second string argument > |
< pointer to string that contains the first environment variable > |
< pointer to last string argument > |
... |

|Stack   |Data                                                                                |
|--------|------------------------------------------------------------------------------------|
|ESP + 12| < pointer to second string argument >                                              |
|ESP + 8 | < pointer to a string that contains the first argument >                           |
|ESP + 4 | < pointer to a string containing the name of the application >                     |
|ESP + 0 | < number of arguments on command line > <- current position of Stack Pointer ESP   |

We know that we are just after the first string argument so we can predict that the pointer to the first string argument will be 2 places up the stack from the current value of ESP. We are using 32 bit locations here, where every pointer is 4 bytes ( 4 x 8 bytes = 32 bits). This means that our first command line argument is at the location ESP + 8. This is because the Stack Pointer ESP can directly reference every byte in memory, and we are looking for the location 2 up from it's current location, effectively 2 x 4 bytes.

We want to move this into a register and print it out so we know we have the correct location. This will also show us what we need to do to print the final value.

As a final step in obtaining the value from the command line, the system call to write to stdout requires the length of the string that is to be processed.

## Debugging with gdb

It would almost be impossible for a mere mortal to understand what is going on without a debugger. I chose to use GDB as it outputs to gas asm and there is plenty of information on how it works online.

We'll use the following code (as per fib2.s) and load it into gdb with the argument 'test', such that once we've made fib2, we run `gdb --args ./fib2 test`. Gdb will load the binary with the argument specified and wait at it's repl prompt for further instructions.

fib2.s
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

Gdb is very powerful and has many options that I'm still getting across, but for these examples we will use a small subset of it's functionality to:
- step through the code
- inspect memory values
- inspect registers`

At the gdb prompt, if we enter `list`, we can see the source code for the fib2 app we have loaded with line numbers. We can set a breakpoint for a corresponding line number to start our debugging at by typing `b n` where `n` is our line number. 


```
(gdb) list
warning: Source file is more recent than executable.
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

For this example, if we enter `b 5`, a breakpoint will be set at line 5, ready to execute.
line 5 is a no-operation, which means that it is a filler instruction. Older versions of gdb require this preceeding start, and so I have left it in for these examples. 

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

From here, we can step through the remaining instructions, one at a time by entering 'stepi', which will run the current instruction and list the next instruction to be run.

```
(gdb) stepi
6           movl %esp, %ebp     # take a copy of the stack pointer esp into ebp
(gdb) stepi

...

(gdb) stepi
(gdb) stepi
(gdb) stepi

...

(gdb) stepi
10          movl $4, %eax       # indicate to int 0x80 that we are doing a write
```

Entering `stepi` several times up to line 10, we shoud have the address of the first argument in the register ecx.

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

We can see there is also the register eip with the value `0x804805a` at a much lower value. This is the extended instruction pointer, that points to the current instruction that our application is running. By convention on Linux, the stack is at the top of memory and our intructions are close to the bottom of memory. The way this is done is by virtual paging so it appears to the application that it has all of physical memory, but it is only mapped pages that appear, so it is just an illusion until the application needs to read or write from a memory address.

Coming back to the ecx value, we can eXamine what is in memory at the address referenced by ecx using the `x/` notation. 
The `x/` notation takes up to 3 components. For example, 

`x/5s ($ecx)`

Prints the first 5 strings starting at the address pointed to by the register ecx and will output something similar to the following:

```
(gdb) x/5s ($ecx)
0xffffd9d3:     "test"
0xffffd9d8:     "LESSOPEN=| /usr/bin/lesspipe %s"
0xffffd9f8:     "HOSTNAME=fc9dd5d874b7"
0xffffda0e:     "SHLVL=1"
0xffffda16:     "HOME=/root"
(gdb)
```

We can clearly see our first command line argument here "test" followed by some environment variables.

The remainder of the application prints the first 4 characters of the string pointed to by the address in ecx, then exits gracefully. To run the app to completion enter `continue` or just `c`. 

We will see the command line argument printed out followed by the app exiting as below.

```
(gdb) c
Continuing.
test[Inferior 1 (process 24) exited normally]
(gdb)
```

# Actually implementing the logic

## Creating the stack space for our array

## Implementing a for loop in assembly language


## Printing our result

## Testing the performance of our application

# Conclusion 

## From here to there - improving performance

## The limits of our application - very large numbers
## Better validation
