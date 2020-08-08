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
