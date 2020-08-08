+++
title = "How Does Debugging Work in Dotnet Core"
date = 2020-07-04T17:45:28+10:00
draft = true
tags = []
categories = []
+++


Topic areas:

- SOS
  - dead-end with llvm breakpoints
- past history - dbgeng
- DAC
- ICorDebug
- No COM on linux?
- coreclr/src/debug
  - dbgshim
    - setting a breakpoint
    - breakpoint hit event emitted

In dotnet core, all debugging is abstracted through the Data Access Component (DAC) via 2(?) different interfaces:
- ICorDebug
- ICLRDataTarget

ICLRDataTarget is used for backwards compatibility with DbgEng, which was originally used with WinDbg. ICorDebug is the more recent way of interacting with the DAC and is used with VS Code. (Refs?)

The [Book Of The Runtime (BOTR)](https://github.com/dotnet/coreclr/tree/release/3.1/Documentation/botr) is a great place to start for all things CLR related. There is a [well referenced diagram](https://github.com/dotnet/coreclr/blob/release/3.1/Documentation/botr/dac-notes.md#marshaling-specifics) of the DAC including a definition of what is considered the Left and Right-hand side (LHS and RHS) of the debugging architecture.

# Code locations:

There are some important entrypoints into the SoS and the dotnet core debugging code. They are:

- [diagnostics/src/SOS](https://github.com/dotnet/diagnostics/tree/master/src/SOS) - location of the source code for SoS
- [coreclr/src/debug](https://github.com/dotnet/coreclr/tree/release/3.1/src/debug) - location of the debug interfaces in the CLR.
- [coreclr/src/dll](https://github.com/dotnet/coreclr/tree/release/3.1/src/dll) - location of: dbgshim.so,

For the purposes of this discussion, we will be looking at coreclr version 3.1 and diagnostics SOS from latest.

# Build libraries:

Debugging the CLR depends on several libraries that need to be present in order to complete the debugging experience.

As dotnet core is cross-platform, these manifest as differently named files, depending on the platform dotnet is run on.

           | Linux                                                            | MacOS                 | Windows
Debug Shim | ./bin/obj/Linux.x64.Debug/src/dlls/dbgshim/libdbgshim.so         | libdbgshim.dylib      | dbgshim.dll
DAC DBI    | ./bin/obj/Linux.x64.Debug/src/dlls/mscordbi/libmscordbi.so       |
DAC        | ./bin/obj/Linux.x64.Debug/src/dlls/mscordac/libmscordaccore.so   | libmscordaccore.dylib |
Core CLR   | ./bin/obj/Linux.x64.Debug/src/dlls/mscoree/coreclr/libcoreclr.so | libcoreclr.dylib      |

## The Debug Shim (DbgShim)

From the code:

```
// DbgShim.cpp
//
// This contains the APIs for creating a telesto managed-debugging session. These APIs serve to locate an
// mscordbi.dll for a given telesto dll and then instantiate the ICorDebug object.
```

This is essentially the entrypoint for any debugger into the dotnet core ecosystem for launching and attaching to processes for the purpose of debugging. Note the use of the term `Telesto`, which I have tried to track down it's meaning, but is most likely a CLR instance running in a process.

We can also see mention of `mscordbi.dll`, or `libmscordbi.so` on Linux, which is the binary interface that interacts with the DAC.

The code comments are interesting, and show the default process for launching and attaching to a process.

```
// Here's a High-level overview of the API usage

From the debugger:
A debugger calls GetStartupNotificationEvent(pid of debuggee) to get an event, which is signalled when
that process loads a Telesto.  The debugger thus waits on that event, and when it's signalled, it can call
EnumerateCLRs / CloseCLREnumeration to get an array of Telestos in the target process (including the one
that was just loaded). It can then call CreateVersionStringFromModule, CreateDebuggingInterfaceFromVersion
to attach to any or all Telestos of interest.

From the debuggee:
When a new Telesto spins up, it checks for the startup event (created via GetStartupNotificationEvent), and
if it exists, it will:
- signal it
- wait on the "Continue" event, thus giving a debugger a chance to attach to the telesto

Notes:
- There is no CreateProcess (Launch) case. All Launching is really an "Early-attach case".
```

I won't delve into this too much, but it shows that the debugging process depends on callbacks, or 'signals' to be emitted during the debug process.

## The Debugger Intereface (DBI)

Again, starting with the comments in the code for ./src/dlls/mscordbi/mscordbi.cpp:

```
// COM+ Debugging Services -- Debugger Interface DLL
//
// Dll* routines for entry points, and support for COM framework.

...


//*****************************************************************************
// The main dll entry point for this module.  This routine is called by the
// OS when the dll gets loaded.  Control is simply deferred to the main code.
//*****************************************************************************
extern "C"
#ifdef FEATURE_PAL
DLLEXPORT // For Win32 PAL LoadLibrary emulation
#endif
BOOL WINAPI DllMain(HINSTANCE hInstance, DWORD dwReason, LPVOID lpReserved)
{
	// Defer to the main debugging code.
    return DbgDllMain(hInstance, dwReason, lpReserved);
}
```

This indicates that the implentation of this is elsewhere. But where?

Looking around the code for a matching method signature, we find the entrypoint in the file: `./src/drbug/di/cordb.cpp`.

```
grep -rni DbgDllMain .
./src/dlls/mscordbi/mscordbi.cpp:14:extern BOOL WINAPI DbgDllMain(HINSTANCE hInstance, DWORD dwReason,
.src/dlls/mscordbi/mscordbi.cpp:28:    return DbgDllMain(hInstance, dwReason, lpReserved);
./src/debug/di/cordb.cpp:196:BOOL WINAPI DbgDllMain(HINSTANCE hInstance, DWORD dwReason, LPVOID lpReserved)
./src/debug/di/cordb.cpp:230:                    "DI::DbgDllMain: load right side support from file '%s'\n",
```

It is in `./src/debug/di/cordb.cpp` that all the public APIs are defined for intracting with the DAC. Gleaning this path, we can see thre is a lot of thee implementation defined for interacting with the DAC, a data target, and the CLR. We will run through how this all works in the following section.
