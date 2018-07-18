---
title: "Creating a Fibonacci Generator in Assembly"
date: 2018-07-18T21:12:38+10:00
draft: true
---

Stemming from my interest in Assembly Language and in search for a practical example to start exploring this language, I thought I'd write a simple ASM application to compute Fibonacci to a given number of iterations. I defined some criteria and restrictions for this little project and came to the following:

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

## Using some staring knowledge

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

# Beginning the implementation

Ok, so we have a basis for our algorithm, but how do we implement it in ASM? There are numerous formats of Assembly Language including NASM, GAS and MASM among others. I'm working on a Linux box and so MASM is out of the question. I decided to go with GAS as it is cross-platform and is a good general purpose assembler. This also gives me the ability to cheat in my implementation (which I'm not), as using `gcc`, a valid C application can be converted into intermediate GAS assembly language.

I decided the number of iterations of Fibonacci would be determined from the command line as an argument to the application. It would expect a valid number from the user and convert this from a string to a number.

The Fibonacci algorithm would then have to create an array of lenth n+2 on the stack based on the user input. Another variable (as i shown above) would be used as a counter this counter would likely use the ECX register.

The first two elements of the array would be pre-populated with 0 and 1 respectively, as all Fibonacci sequences have these numbers and are used when calculating new values.

A loop is then started with the counter set to 2; which iterates until the counter is less than or equal to the number of iterations (n); and where the conuter is increased by 1 on each iteration.

Inside the loop, for the array position in the stack, and based on the counter value, the i - 1 value is added to the i - 2 value on the stack and the result is placed on the stack at the current location allowing it to be used for the next iteration.

The final value placed on the stack then needs to be converted to a string and printed to standard out. 

The application then needs to exit gracefully and return control to the shell.


# Starting with a framework

```
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

Ok, so we've got a framework of GAS assembly to build on, but ow do we convert this into an application?

Well, it's easy. I'm glad you asked. Firstly we need to make sure our gas assembler is installed. I'm on Fedora, so the command is `sudo dnf install binutils`.

Then we create a small shell script to run the gas assembler (as) and the linker (ld) on the output of the assembler.

Say for example we call our assembly code file fib.s, to run the assembler we would run `as -gstabs -o fib.o fib.s`

- `-gstabs` adds debugging info
- `-o fib.o` specifies our object file
- `fib.s` is our source code

The object file is a binary with all the symbols and source included as well as the raw machine code to run.

Then, we link the resultant object file to convert our object file into a binary application. The linker also converts our symbolic references (such as \_start in our case) to relative numeric references to where the symbols will reside in memory when the binary is run as an application.

If we were to run `ld -o fib fib.o`, we should now have a new binary application with the permissions 0755 set. If you run the application it will do nothing. But more importantly, it won't segfault (which is a good thing).

`-o fib` specifies the output application name
`fib.o` specifies the object file to use.

The linker has other features (which I won't discuss here) that make your life easier as an assembly language programmer.

# Reading from STDIN and converting to a numeric value

# Actually implementing the logic

## Creating the stack space for our array

## Implementing a for loop in assembly language

## Debugging with gdb

## Printing our result

## Testing the performance of our application

# Conclusion 

## From here to there - improving performance

## The limits of our application - very large numbers

## Better validation
