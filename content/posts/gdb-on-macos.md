+++
title = "Gdb on MacOS"
date = 2019-06-29T17:22:07+10:00
draft = false
tags = []
categories = []
+++

GDB, or the GNU Debugger has been around for years. It was first written by Richard Stallman in 1986 and has a long heritage on Unix, Linux and other Operating Systems. I’ve started using it recently for debugging applications in assembly from my MacBook Pro.

I wanted to be able to debug both local MacOS binary applications as well as remotely debug Linux applications in Docker containers. This is not as simple as it seems as the format of binaries used on MacOS and Linux are different - MacOS uses the MachO binary format and Linux uses the ELF binary format. This on its own isn’t a problem, but it turns out that different versions of GDB are needed to interpret the different binary formats.

Adding to the confusion, Apple has been moving away from GDB for low-level debugging on MacOS in favour of LLVM, where since Mac OS X version 10.4 (Maverick) applications like Eclipse which rely on GDB started to fail.

# My Use-Cases

I essentially have 2 use-cases I need to be able to debug for:

![gdb use cases](http://me.wvk.au/img/gdb.drawio.png)

# Installing GDB to debug MachO binaries locally:

There are some great instructions on how to do this [here](https://www.ics.uci.edu/~pattis/common/handouts/macmingweclipse/allexperimental/mac-gdb-install.html), that go through the code-signing process required when installing gdb from brew.

In addition, since GDB vesion 8.2 in conjunction with MacOS X Mojave 10.14, the GDB libraries have not been kept up to date on MacOS, and so further instructions, with a specific brew formula for installing a patched version of GDB have been necessary to get this to work. Take a look [here](https://stackoverflow.com/a/53586598/512965) and [here](https://raw.githubusercontent.com/timotheecour/homebrew-timutil/master/gdb_tim.rb) for instructions on this.

So, this should get you most of the way towards debugging a MacOS MachO binary locally, but what about if we want to remotely debug a Linux binary in a Docker container from our Mac?

# Remote debugging Linux applications from MacOS

Well, it turns out that remote targets needs to be enabled in the build of gdb we install via brew. There is a great blog post on this [here](http://tomszilagyi.github.io/2018/03/Remote-gdb-with-stl-pp). The post also goes through the process of remote debugging via an exposed port, which is a useful reference.

# Switching between the two versions of gdb:

So, it’s great that we can now debug locally and remotely with GDB, but they are both different versions of GDB and will need to be switched between within brew in order to be run.

## For remote Linux debugging we want to run:

```
brew unlink gdb && brew link gdb
```

## For local MacOS debugging we want to run:

```
brew unlink gdb_tim && brew link gdb_tim
```

I hope this is useful to someone out there who was as lost as I was encountering this.
