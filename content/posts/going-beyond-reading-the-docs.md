---
title: "Going Beyond Reading the Docs"
date: 2020-04-18T14:02:20+10:00
draft: false
---

# Starting at the start

This is something I've always done, but it became more apparent to me recently while attempting to build the latest source code for the ``dotnet core v5.0`` runtime and libraries.

The goal was to get the binaries built with debugging output so that I could investigate further how lldb and sos work. This was also for a higher goal of integrating dotnet core into a library I am writing for cross-editor debugging called [Tide](https://github.com/tide-org).

Having read the Docker build instructions [here](https://github.com/dotnet/runtime/blob/master/docs/workflow/building/coreclr/linux-instructions.md) and considering it not too difficult or time-consuming, I ventured into getting a build toolchain working that could build the binaries.

The build was for the [dotnet runtime](https://github.com/dotnet/runtime/) where I intended to run the `./build.sh` command with some target settings to output the biniaries.

Having installed all the prerequisite libraries, I was getting some errors. Thus begun the rabbit-hole of experimenting with different library versions, package repositories, library configuration settings and docker settings in an attempt to get it working.

The `./build.sh` can build a subset of the full build. This is split into the `clr` component and the `libs` component. Starting with only 1, using `--subset clr` halved the build time, but this was still taking 5-10 minutes to build. Every time I changed the `apt` libraries installed, this would take another 5-10 minutes.

# Digging deeper

Blind faith on the installation instructions was taking too long. Looking through the dotnet github repositories, there are a few repositories just for doing the docker builds alone:

- Building the various dotnet core Docker images that are published to DockerHub. These depend on other prerequisite docker images that set up the toolchain. This includes all the various architectures and operating systems that dotnet core can run on.

https://github.com/dotnet/dotnet-docker

- Building the prerequisite Docker images for the various dotnet core components, including the build toolchain.

https://github.com/dotnet/dotnet-buildtools-prereqs-docker

- Build pipelines for the dotnet core Docker nightly builds:

https://github.com/dotnet/dotnet-docker/blob/nightly

- A set of common tools for building dotnet core docker containers exposed via a cli in a docker container

https://github.com/dotnet/docker-tools

- A set of build tools to build reference version of historical dotnet core packages

https://github.com/dotnet/source-build-reference-packages

- A set of build tools for setting up and running the build pipelines

https://github.com/dotnet/buildtools

- A list of manifests describing the different versions of components used in different dotent builds
https://github.com/dotnet/versions

Doing a Google search around keywords to do this build didn't give much information as to how to set up the toolchain (or maybe my Google-fu was off). But I found that by stepping back, and not blindly driving forwards, then looking at the context that the document existed in heelped greatly in resolving the issue.

- A new set of build tools to supersede `buildtools`

https://github.com/dotnet/arcade

# Light at the end of the tunnel

Doing a Google search around keywords to do this build didn't give much information as to how to set up the toolchain (or maybe my Google-fu was off). But I found that by stepping back, and not blindly driving forwards, then looking at the context that the document existed in helped greatly in resolving the issue.

Sometimes the solution is straightforward and doesn't require such tactics to resolve the issue. Other times, understanding the context around the problem one is attempting to solve can help greatly.

Sometimes, however, the problem space is so large, or it takes so long to get feedback from the process that alternatives must be entertained.

It turned out the the `dotnet-buildtools-prereqs-docker` repo had a `Dockerfile` of exactly what I was after and the binaries could be built after configuring the docker container the same way.

# Going forward

It's a useful skill, indeed, to have the perseverence and resourcefulness to not blindly accept what is presented at face value; as often things can be either incorrect or incomplete.

In investigating all this, it made me also realise that the ecosystem of dotnet core has had many changes in it's brief history on github, and while being trailblazing as it is, it's left many dark alleys and quagmires in it's path - but that's a story for another day.
