---
title: "Going Beyond Reading the Docs"
date: 2020-04-18T14:02:20+10:00
draft: false
---

This is something I've always done, but it became more apparent to me recently while attempting to build the latest source code for the dotnet core v5.0 runtime and libraries.

The goal was to get the binaries built with debugging output so that I could investigate further how lldb and sos work. This was also for a higher goal of integrating dotnet core into a library I am writing for cross-editor debugging called [Tide](https://github.com/tide-org).

Having read the Docker build instructions [here](https://github.com/dotnet/runtime/blob/master/docs/workflow/building/coreclr/linux-instructions.md) and considering it not too difficult or time-consuming, I ventured into getting a build toolchain working that could build the binaries.

The build was for the [dotnet runtime](https://github.com/dotnet/runtime/) where I intended to run the `./build.sh` command with some target settings to output the biniaries.

Having installed all the prerequisite libraries, I was getting some errors. Thus begun the rabbit-hole of experimenting with different library versions, package repositories, library configuration settings and docker settings in an attempt to get it working.

The `./build.sh` can build a subset of the full build. This is split into the `clr` component and the `libs` component. Starting with only 1, uaing `--subset clr` halved the build time, but this was still taking 5-10 minutes to build. Every time I changed the `apt` libraries installed, this would take another 5-10 minutes.

Blind faith on the installation instructions was taking too long. Looking through the dotnet github repositories, there are a few repositories just for doing the docker builds alone:

- Building the various dotnet core Docker images that are published to DockerHub. These depend on other prerequisite docker images that set up thee toolchain. This includes all the various architectures and operating systems that dotnet core can run on.
https://github.com/dotnet/dotnet-docker

- Building the prerequisite Docker images for the various dotnet core components, including the build toolchain.
https://github.com/dotnet/dotnet-buildtools-prereqs-docker

- Build pipelines for the dotnet core Docker nightly builds:
https://github.com/dotnet/dotnet-docker/blob/nightly

- A set of common tools for building dotnet core docker containers exposed via a cli in a docker container
https://github.com/dotnet/docker-tools

Doing a Google search around keywords to do this build didn't give much information as to how to set up the toolchain (or maybe my Google-fu was off). But I found that by stepping back, and not blindly driving forwards, then looking at the context that the document existed in heelped greatly in resolving the issue.

Somtimes the solution is straightforward and doesn't require such tactics to resolve the issue. Other times, understanding the context around the problem one is atteempting to solve can help greatly.

Sometimes, however, the problem space is so large, or it takes so long to get feedback from the process that alternatives must be entertained.

It turned out the the `dotneet-buildtools-prereeqs-dockeer` repo had a `Dockerfile` of exactly what I was after and the binaries could be built after configuring the docker container the same way.

It's a useful skill, indeed, to have the perseverence and resourcefulness to not blindly accept what is presented at face value; as often things can be either incorrect or incomplete.
