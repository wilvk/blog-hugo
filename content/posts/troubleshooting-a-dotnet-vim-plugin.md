+++
title = "Troubleshooting a Dotnet Vim Plugin"
date = 2020-02-16T17:34:02+11:00
draft = false
tags = []
categories = []
+++

## problem

I had a problem on a new MacBook.

I was trying to get C# working in Vim. I found a package online called [omnisharp-vim](https://github.com/OmniSharp/omnisharp-vim) and decided to give it a go.

After installing the plugin in my `.vimrc` file, I opened a project file. The syntax highlighting was there, although there was a small problem. Some of the imports wouldn't import:

The project was a dotnet core mvc application and a few of the relevant libraries weren't showing.

<image>

Libraries like:

```
using microsoft.extensions.hosting
using microsoft.extensions.loggin
using microsoft.aspnetcore.http
```

were showing the ubiquitous:

```
The type or namespace cannot be found (are you missing a using directive or an assembly reference?)
```

## investigation

This was strange to me as I was of the impression that once dotnet core was installed, all the dependencies should be there too. This was my first incorrect assumption that would turn into a rabbit hole of investigation.

I ran the build locally with `dotnet build` and it built as expected. So what was causing the dependencies to be missing?

Firstly, I looked at the `omnisharp-vim code` to determine how it works.

The plugin has two options - stdio and http for sending messages from the debugger to Vim. The plugin called it from the command line and the output json is consumed by the plugin to define features of the code.

I decided to blow away all the `omnisharp-vim` files in my `~/.vim/bundle/omnisharp-vim` path as well as the `~/.cache/omnisharp-roslyn` path.

No luck - the same issue occurred.

I began to think that it may have been the version of dotnet core on my laptop so uninstalled all the dotnet components using the dotnet sdk uninstall tool. Then removing all files from `/usr/local/share/dotnet`, as well as deleting `/usr/local/bin/dotnet`, and all references to it in my environment.

I reinstalled the [dotnet-sdk](https://formulae.brew.sh/cask/dotnet-sdk) from brew.

The same references were still not being resolved. I couldn't figure it out.

Looking in the path `/usr/local/share/dotnet/packs/Microsoft.AspNetCore.App.Ref/3.1.0/ref/netcoreapp3.1/` I could actually see the binaries for the missing imports there, yet for some reason `omnisharp-vim` couldn't find them.

## assumptions

The plugin `omnisharp-vim` also installs a package called [omnisharp-roslyn](https://github.com/OmniSharp/omnisharp-roslyn) that is used to communicate with the debugger and uses the .net roslyn workspaces for defining type safety and other things in code.

Stepping back from the issue a bit and thinking about all the parts in the chain, I decided to look into how `omnisharp-roslyn` actually worked.

Looking around in the source code I could see that there were references to a packaged version of Mono, but not dotnet core. Initially I thought that this was because there may be licencing or other things preventing dotnet core being packaged as well. 

I then searched online as it  may have been that a dotnet core version of `omnisharp-roslyn` existed that just hadn't installedfor some reason. To my surprise, it turns out that `omnisharp-roslyn` only targets Mono, as mentioned [here](https://github.com/OmniSharp/omnisharp-roslyn/issues/1489).  This confused me a bit but reading further, this is the only way that full compatibility with both .net framework and dotnet core can currently be achieved.

Reading the docs, I could see there was a setting called `g:OmniSharp_server_use_mono` - my previous assumption was that this was a toggle between Mono and dotnet core.  Reading the docs closer I could see that it toggles between the packaged and system-installed versions of Mono, and not between Mono and dotnet core. A rookie mistake in the new, open source .net world.

![connection between omnisharp-vim and mono](https://blog.seso.io/img/omnisharp.png)

## solution

Well, that answered that, so I duly removed all the versions of Mono that I had and did a full reinstall of `omnisharp-vim`. However, the plugin  was still not resolving. 

I was quite perplexed until I read a couple of github issues for the plugin:

- https://github.com/OmniSharp/omnisharp-roslyn/issues/1693
- https://github.com/OmniSharp/omnisharp-vim/issues/556

and realised that there was another system-wide Mono version being referenced by brew. Doing a system-wide search using on any file with Mono in it (`find / -name "*mono*"`) showed me there was a Mono brew cellar still present.

```
brew uninstall --force mono
```

Boom. All sorted and back in business.

## understanding

There are quite a lot of variations of .Net now. It's an entire ecosystem of different flavours - similar in some ways to the Linux Distro ecosystem (although not as diverse). What I understand now is that these flavors consist of:

- dotnet core
- dotnet-core-sdk
- Mono
- Mono without asp.net
- Legacy .Net Framework
- and others

There are also a variety of ways these can be downloaded and installed onto a computer, placing copies in different locations in the filesystem, including through:

- Brew
- Direct download
- Plugins (such as omnisharp-vim)
- IDEs (such as VS Code and Visual Studio for Mac)

This all adds to increasing the surface area for error and frustration in the troubleshooting process.

## learnings

This highlighted to me a few things around:

__Name your assumptions__  - My initial assumption was that as it was a dotnet core application, the debugger would also be targeting dotnet core. This was incorrect. Only by defining what my assumption was could I challenge it and come to a new understanding.

__Check your assumptions__ - At one stage through this process I was thinking that the Omnisharp-vim plugin was broken (which is is, but not in the way I thought). I tried running the plugin on another laptop and it worked. This gave me the confidence to continue troubleshooting and fully understand and resolve the issue.

__Read closely__ - Read the docs carefully - going back to the omnisharp-vim docs it clearly states that it must be run on mono. I must have skimmed over this in my initial research of the problem. The problem was that the version I was using didn't have the asp.net libraries.

__The code never lies__ - Getting to the bottom of the problem required understanding what was possible with the plugin and what was not.

__Have patience__ - It took me about a day to figure all this out but through perseverence, there comes a solution.

*Happy vimming!*
