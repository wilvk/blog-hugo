+++
title = "Linking C and C++ Libraries to C#"
date = 2020-06-19T18:41:23+10:00
draft = true
tags = []
categories = []
+++

I've written a few blogs now, and was thinking of ways to make the process a bit more interesting - mostly for me - and so decided to write a blog using the [scientific report writing method.](https://www.une.edu.au/__data/assets/pdf_file/0010/10324/WE_Writing-a-scientific-report.pdf).

# Overview:

The following is an investigation into how to call C and C++ libraries from C#, as well as how to call C and C++, from both C and C++. Calling C# from C or C++ will not be discussed, as it is more involved, and there are many ways to achieve this.

The following table indicates what is covered:

| Application/Library | C | C++ | C# |
|---------------------|---|-----|----|
| C                   | ✅ | ✅   | ❌    |
| C++                 | ✅ | ✅   | ❌    |
| C#                  | ✅ | ✅   | ❌    |

# Materials:

The investigation was done using MacOS Catalina 10.15.5, however, could easily be replicated on Linux.

For compiling the binaries:

| Language | Compiler                     |
|----------|------------------------------|
|   C#     | dotnet core, version 3.1.201 |
|   C      | Apple clang 11.0.0           |
|   C++    | Apple clang 11.0.0           |

# Method:

## A C Application to a C Library

Linking to a C library is very straightforward.

![c to c lib](https://blog.seso.io/img/c_c_lib.png)

## A C Application to a C++ Library
![c to c++ lib](https://blog.seso.io/img/c_cpp_lib.png)

## A C++ Application to a C Library
![c++ to c lib](https://blog.seso.io/img/cpp_c_lib.png)

## A C++ Application to a C++ Library
![c++ to c++ lib](https://blog.seso.io/img/cpp_cpp_lib.png)

## A C# Application to a C Library
![c# to c lib](https://blog.seso.io/img/cs_c_lib.png)

## A C# Application to a C++ Library
![c# to c++ lib](https://blog.seso.io/img/cs_cpp_lib.png)

# Discussion:

# References:

https://isocpp.org/wiki/faq/mixing-c-and-cpp
https://stackoverflow.com/questions/1041866/what-is-the-effect-of-extern-c-in-c
