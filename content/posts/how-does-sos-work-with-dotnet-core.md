+++
title = "How Does SoS Work With Dotnet Core"
date = 2020-07-11T16:02:56+10:00
draft = true
tags = []
categories = []
+++

# What is SoS?

SoS is an acronym for Son Of Strike, and is the prededessor of Strike, which previously interacted with the original .NET CLR called 'Lightning". There is a back story to why SoS is called what it is [here](https://stackoverflow.com/a/3573049/512965), but it is essentially a low-level debugger for .NET and dotnet core frameworks.

Under the .NET Framework, it integrated with WinDbg for debugging .NET applications. Under dotnet core, it is integrated into LLDB. LLDB is a low-level debugger that is part of the [LLVM framework](https://llvm.org/). SoS for LLVM is open-source and is available in the [diagnostics repository](https://github.com/dotnet/diagnostics).

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

## The Debugging Interface implementation


# How does SoS interact with dac?

We will look at SoS running as a plugin to LLDB under the Linux platform.

The SoS plugin is in a file called:

.dotnet/sos/libsosplugin.so


Looking at the SoS code in https://github.com/dotnet/diagnostics.git/src/SOS/Strike/strike.cpp :



it would be good to know how this calls mscorwksdac.so and implement something similar in dotnet core or python.

docs also show:

```
clrstack -i -a           : This will show you all parameters and locals for all frames
```

clrmd used to reference ICorDebug. It doesn't anymore. Not sure why.


util.h:191 references MSCORWKS
./sosdocs.txt has a lot of info and references ICorDebug

ICorDebug has lots of derrivatives. A logical entrypoint is ICorDebugProcess

some places this is used are:

```
./ExpressionNode.cpp:11:ICorDebugProcess* ExpressionNode::s_pCorDebugProcess = nullptr;

./ExpressionNode.h:91:    static ICorDebugProcess* s_pCorDebugProcess;
./runtime.h:90:    virtual HRESULT GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess) = 0;
./runtime.h:189:    ICorDebugProcess* m_pCorDebugProcess;
./runtime.h:285:    HRESULT GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess);
./strike.cpp:13319:        ICorDebugProcess* pCorDebugProcess;
./strike.cpp:15756:    ToRelease<ICorDebugProcess2> proc2;
./strike.cpp:15757:    ICorDebugProcess* pCorDebugProcess;
./strike.cpp:15762:    else if (FAILED(pCorDebugProcess->QueryInterface(__uuidof(ICorDebugProcess2), (void**) &proc2)))
./runtime.cpp:479:HRESULT Runtime::GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess)
./runtime.cpp:488:        // ICorDebugProcess4 is currently considered a private experimental interface on ICorDebug, it might go away so
./runtime.cpp:490:        ToRelease<ICorDebugProcess4> pProcess4 = NULL;
./runtime.cpp:491:        if (SUCCEEDED(m_pCorDebugProcess->QueryInterface(__uuidof(ICorDebugProcess4), (void**)&pProcess4)))
./runtime.cpp:550:        IID_ICorDebugProcess,
./runtime.cpp:558:    hr = pUnkProcess->QueryInterface(IID_ICorDebugProcess, (PVOID*)&m_pCorDebugProcess);
./cordebugdatatarget.h:10:* in order to get an ICorDebugProcess back.
```

```
The -i options uses DML output for a better debugging experience, so typically you
```

runtime.cpp has:

```
/**********************************************************************\
 * Loads and initializes the public ICorDebug interfaces. This should be
 * called at least once per debugger stop state to ensure that the
 * interface is available and that it doesn't hold stale data. Calling
 * it more than once isn't an error, but does have perf overhead from
 * needlessly flushing memory caches.
\**********************************************************************/

479 HRESULT Runtime::GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess)
```

is called from:

```
./strike.cpp:13320:        if (FAILED(Status = g_pRuntime->GetCorDebugInterface(&pCorDebugProcess)))
./strike.cpp:15758:    if (FAILED(hr = g_pRuntime->GetCorDebugInterface(&pCorDebugProcess)))
./runtime.cpp:479:HRESULT Runtime::GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess)
```


and it's main entry point is:

```
    // This is the main worker function used if !clrstack is called with "-i" to indicate
    // that the public ICorDebug* should be used instead of the private DAC interface. NOTE:
    // Currently only bParams is supported. NOTE: This is a work in progress and the
    // following would be good to do:
    //     * More thorough testing with interesting stacks, especially with transitions into
    //         and out of managed code.
    //     * Consider interleaving this code back into the main body of !clrstack if it turns
    //         out that there's a lot of duplication of code between these two functions.
    //         (Still unclear how things will look once locals is implemented.)
    static HRESULT ClrStackFromPublicInterface(BOOL bParams, BOOL bLocals, BOOL bSuppressLines, __in_z WCHAR* varToExpand = NULL, int onlyShowFrame = -1)
```

`ClrStackFromPublicInterface`

and that is called from:

```
14312 DECLARE_API(ClrStack)
```

and that is defined in:

./inc/wdbgexts.h

at:

```
1658 #define DECLARE_API(s) DECLARE_API64(s)
```

and that is defined in:

./inc/wdbgexts.h

at 1686:

```
#define DECLARE_API64(s)                           \
    CPPMOD VOID                                    \
    s(                                             \
        HANDLE                 hCurrentProcess,    \
        HANDLE                 hCurrentThread,     \
        ULONG64                dwCurrentPc,        \
        ULONG                  dwProcessor,        \
        PCSTR                  args                \
     )
```

and CPPMOD is defined at 1645:

```
#ifdef __cplusplus
#define CPPMOD extern "C"
#else
#define CPPMOD
#endif
```

and this is the entrypoint of lldb-sos for the plugin.

we now need to go the other way to determine how a breakpoint is set from the ICorDebug interface on a process.

Setting up a test ICorDebugProcess app in C++ could help with this. it could then be used as a proxy to any other language. This is now starting to get interesting.

The diagnostics tests also use a python wrapper to lldb-sos, and so a similar approach  could be taken.
