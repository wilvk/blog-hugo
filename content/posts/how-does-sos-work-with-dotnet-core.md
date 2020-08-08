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

We will look at SoS running as a plugin to LLDB under the Linux platform.

The SoS plugin is in a file called:

`/.dotnet/sos/libsosplugin.so`

That is contained with a full install of dotnet.

## Looking for an entrypoint to SoS using DAC as a starting-point:

The main corpus of code for SoS is in strike.cpp, located here:

https://github.com/dotnet/diagnostics.git/src/SOS/Strike/strike.cpp :

For the purposees of this investigation, we will be using the master git branch as of git commit `daf7ecb7d5042b9001cb9c52f2ae362ac4cf9d30`.

The DAC is a useful hook into finding the entrypoint as we can use it to locate functions related to debugging. One of the main interfaces for the DAC is `ICorDebug`. We will use this interface to grep around to find functions that contain debugging functionality.

`ICorDebug` has lots of derrivatives. A logical entrypoint is `ICorDebugProcess` as it is a debugging primitive that is attached to a live dotnet process.

Grepping the source code, some places ICorDebug is used are:

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

`./runtime.cpp` has the following function for obtaining an `ICorDebugProcess` debugging object:

```
471 /**********************************************************************\
472  * Loads and initializes the public ICorDebug interfaces. This should be
473  * called at least once per debugger stop state to ensure that the
474  * interface is available and that it doesn't hold stale data. Calling
475  * it more than once isn't an error, but does have perf overhead from
476  * needlessly flushing memory caches.
477 \**********************************************************************/
478
479 HRESULT Runtime::GetCorDebugInterface(ICorDebugProcess** ppCorDebugProcess)
```

`::GetCorDebugInterface()` is called from `strike.cpp` in a couple of places:

```
./strike.cpp:13320:        if (FAILED(Status = g_pRuntime->GetCorDebugInterface(&pCorDebugProcess)))
./strike.cpp:15758:    if (FAILED(hr = g_pRuntime->GetCorDebugInterface(&pCorDebugProcess)))
```

The second call on line 15758 is used for Native Image Generation (NGEN). We will ignore this invocation in our analysis and focus on the call from line 13320.

The call to `GetCorDebugInterface` is in the method `ClrStackFromPublicInterface` on line 13315.

```
13306    // This is the main worker function used if !clrstack is called with "-i" to indicate
13307    // that the public ICorDebug* should be used instead of the private DAC interface. NOTE:
13308    // Currently only bParams is supported. NOTE: This is a work in progress and the
13309    // following would be good to do:
13310    //     * More thorough testing with interesting stacks, especially with transitions into
13311    //         and out of managed code.
13312    //     * Consider interleaving this code back into the main body of !clrstack if it turns
13313    //         out that there's a lot of duplication of code between these two functions.
13314    //         (Still unclear how things will look once locals is implemented.)
13315    static HRESULT ClrStackFromPublicInterface(BOOL bParams, BOOL bLocals, BOOL bSuppressLines, __in_z WCHAR* varToExpand = NULL, int onlyShowFrame = -1)
```

And `ClrStackFromPublicInterface` is called in `strike.cpp` as:

```
14396         return ClrStackImplWithICorDebug::ClrStackFromPublicInterface(bParams, bLocals, FALSE, wvariableName, frameToDumpVariablesFor);
```

From the function:

```
14312 DECLARE_API(ClrStack)
```

This declares the llvm-sos API command `ClrStack`. For example, when someone is in llvm with sos loaded and types in a command of the form:

```
clrstack ...
```

The C++ macro `DECLARE_API` is defined in: `./inc/wdbgexts.h` at line 1658 as:

```
1658 #define DECLARE_API(s) DECLARE_API64(s)
```

... and `DCLARE_API64` is defined in: `./inc/wdbgexts.h` at line 1686 as:

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

... and `CPPMOD` is defined at linee 1645 as:

```
#ifdef __cplusplus
#define CPPMOD extern "C"
#else
#define CPPMOD
#endif
```

... and this (`CPPMOD`), finally, is the entrypoint of lldb-sos for the plugin.

We now need to go the other way to determine how a breakpoint is set from the `ICorDebug` interface on a process.

Setting up a test `ICorDebugProcess` app in C++ could help with this. It could then be used as a proxy to any other language. This is now starting to get interesting. This might be the topic of another blog later on. The diagnostics test suite also uses a Python wrapper to `lldb-sos`, and so a similar approach could be taken to attach to a live process - but I digress.

## Going from our bpmd command to it's actual implementation in lldb-sos:

In the dotnet-sos repository, under `./scripts/startup.lldb` and `./scripts/code_line_breakpoint.lldb`

the first call to SOS is:

```
bpmd Program.cs:21
```

Looking through the source again, `bpmd` is defined in `./strike.cpp:8048-8426`, and this is where the command line parsing occurs.

The following is an analysis of the actual bpmd method in llvm and the path it takes to setting a breakpoint. I find this stuff fascinating. Your Mileage May Vary.

Still in strike.cpp we can see the entrypoint for the api of the command `bpmd`.

`8048 DECLARE_API(bpmd)`

`bpmd` sets a breeakpoint for a method descriptor based on either a filename and line numbere or a function and IL offset.

`bpmd` uses both _pending_ and _set_ breakpoints.

A pending breakpoint is for when code is not yet loaded by the JIT-er and set is when the code has been JIT-ed (loaded into memory).

TODO: add notes of program flow for below:

`8063 // bpmd acts as a feeder of breakpoints to bp when the time is right.`

```
8190 // is available? If so, don't add to our pending list, just go ahead and
8091 // set the real breakpoint.
```

this is where the fun begins:

`8216 // Get modules that may need a breakpoint bound`

module-bound breakpoints - what are these?

JIT notifications - what are these and how do they work?

```
8216    // Get modules that may need a breakpoint bound
        if ((Status = CheckEEDll()) == S_OK)
        {
            if ((Status = LoadClrDebugDll()) != S_OK)
            {
                // if the EE is loaded but DAC isn't we should stop.
                DACMessage(Status);
                return Status;
            }
```

the two checks above are for:

- check that the Execution Engine is present and accessible
- check that that DAC is present and accessible

then:
We get the module object from the filename - L8228
then iterate each module
we are only interested in when we have filename:line_number, (fIsFilename = true)
this is implemented in L8264-8293

```
   SymbolReader symbolReader;
                symbolsLoaded = g_bpoints.LoadSymbolsForModule(moduleList[iModule], &symbolReader);
                if(symbolsLoaded == S_OK &&
                   g_bpoints.ResolvePendingNonModuleBoundBreakpoint(Filename, lineNumber, moduleList[iModule], &symbolReader) == S_OK)
                {
                    // if we have symbols then get the function name so we can lookup the MethodDescs
                    mdMethodDef methodDefToken;
                    ULONG32 ilOffset;
                    if(SUCCEEDED(symbolReader.ResolveSequencePoint(Filename, lineNumber, &methodDefToken, &ilOffset)))
                    {
                        ToRelease<IXCLRDataMethodDefinition> pMethodDef = NULL;
                        if (SUCCEEDED(ModDef->GetMethodDefinitionByToken(methodDefToken, &pMethodDef)))
                        {
                            ULONG32 nameLen = 0;
                            pMethodDef->GetName(0, mdNameLen, &nameLen, FunctionName);

                            // get the size of the required buffer
                            int buffSize = WideCharToMultiByte(CP_ACP, 0, FunctionName, -1, TypeName.data, 0, NULL, NULL);

                            TypeName.data = new NOTHROW char[buffSize];
                            if (TypeName.data != NULL)
                            {
                                int bytesWritten = WideCharToMultiByte(CP_ACP, 0, FunctionName, -1, TypeName.data, buffSize, NULL, NULL);
                                _ASSERTE(bytesWritten == buffSize);
                            }
                        }
                    }
                }
````

if we can:
 - resolve symbols and
 - resolve any non-pending non-module bound breakpoints

we then:
 - check if we can resolve sequence points to a method definition token and an IL offset

then:
  - use the method definition token with the module object to get the method definition object
  - get the method table name


L8295 - 8321:

util.cpp - GetMethodDescsFromName - Find the EE data given a name.

strike.cpp - Update //returns true if updates are still needed for this module, FALSE if all BPs are now bound
  - LoadSymbolsForModule
  - ResolvePendingNonModuleBoundBreakpoint
  - ResolvePendingBreakpoint

if the Breakpoint isn't set, we need to depend on the DAC raising a notification when the code block is JITted and ready for a breakpoint to be set:

`8337 bNeedNotificationExceptions = TRUE;`

8352-8354:

```
// if we've got an explicit MD, then we better have runtime and dac loaded
        INIT_API_EE()
        INIT_API_DAC();
```

dacprivate.h - Request:
    GetMethodDescData

sospriv.h -
```
701 MIDL_INTERFACE("436f00f2-b42a-4b9f-870c-e73db66ae930")
 ...
786    ISOSDacInterface : public IUnknown
virtual HRESULT STDMETHODCALLTYPE GetMethodDescData(

```

8350 - /* We were given a MethodDesc already */
This is when the code is already Jitted.

8366 - only if the .net binary is native code (not NGEN).

L8368-8396 - is when we have dynamic code (we can't set a breakpoint for dynamic code).
```
8371 // Dynamic methods don't have JIT notifications. This is something we must
8372 // fix in the next release. Until then, you have a cumbersome user experience.
```

L8398 - 8412 - we have jitted code and can set a breakpoint:

```
        {
            // Must issue a pending breakpoint.
            if (g_sos->GetMethodDescName(pMD, mdNameLen, FunctionName, NULL) != S_OK)
            {
                ExtOut("Unable to get method name for MethodDesc %p\n", SOS_PTR(pMD));
                return Status;
            }

            FileNameForModule ((DWORD_PTR) MethodDescData.ModulePtr, ModuleName);

            // We didn't find code, add a breakpoint.
            g_bpoints.ResolvePendingNonModuleBoundBreakpoint(ModuleName, FunctionName, TO_TADDR(MethodDescData.ModulePtr), 0);
            g_bpoints.Update(TO_TADDR(MethodDescData.ModulePtr), FALSE);
            bNeedNotificationExceptions = TRUE;
        }
```

L8415-8423:

we then set our pending breakpoints
```
if (bNeedNotificationExceptions)
    {
        ExtOut("Adding pending breakpoints...\n");
#ifndef FEATURE_PAL
        Status = g_ExtControl->Execute(DEBUG_EXECUTE_NOT_LOGGED, "sxe -c \"!SOSHandleCLRN\" clrn", 0);
#else
        Status = g_ExtServices->SetExceptionCallback(HandleExceptionNotification);
#endif // FEATURE_PAL
    }
```

libraries:

(https://github.com/dotnet/diagnostics/blob/master/documentation/FAQ.md)

The DAC module: (libmscordaccore.so or libmscordaccore.dylib)
The runtime (libcoreclr.so or libcoreclr.dylib).
(both need to be in the same directory)

Sos libraries:
SOS (libsos.so),
the managed debugging shim (libdbgshim.so)



file name -> module -> method descriptor -> method table name -> IL offset

DAC details:

https://github.com/dotnet/runtime/blob/master/docs/design/coreclr/botr/dac-notes.md#marshaling-specifics

DAC declarations:

https://github.com/dotnet/runtime/blob/master/src/coreclr/src/inc/daccess.h

TODO: reread all this. amend and add to.

how do pending breakpoints work?
how do native breakpoints work?
what is IXClr?
what is LTTng-UST exception? https://github.com/dotnet/coreclr/issues/20205

SBFileSpec.h - the lldb loads or attaches to the dotnet process. It then iterates the debugger process to find the .net process and the loaded EE and DAC. Any standalone debugger would need to do the same.

dbgeng.h -
https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/

what is the difference between this and mscordacwks?

clrmd uses dbgeng and it compiles under osx

good article on dbgeng:
https://www.osr.com/nt-insider/2010-issue2/basics-debugger-extensions/

our startingpoint would be to look at:

IDebugControl – Evaluate expressions, disassemble, execute commands, add/remove breakpoints, get input, send output, etc.
clrmd -> DebugControl.cs has `public readonly IntPtr AddBreakpoint;`

so,

It looks like SOS Strike uses the ICorDebug interface and clrmd now uses the dbgeng APIs.

is dbgeng in dotnet core for linux?

it looks like a lot of the same functionality in dotnet/diagnostics sos:

https://github.com/dotnet/diagnostics/blob/master/src/SOS/Strike/inc/dbgeng.h

is available as a standalone dotnet library:

https://github.com/dotnet/diagnostics/tree/master/src/SOS/SOS.Hosting/dbgeng

and is also replicated in clrmd:

https://github.com/microsoft/clrmd/tree/master/src/Microsoft.Diagnostics.Runtime/src/DbgEng


the strike c++ project file also references dbgeng:

./Strike.vcxproj

ICorDebug vs dbgeng

ICorDebug:
https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/debugging/icordebug-interface

dbgeng:
https://docs.microsoft.com/en-us/windows-hardware/drivers/ddi/dbgeng/


Creeating a breakpoint with ICorDebug:

https://github.com/dotnet/runtime/issues/13259

COM interop builds for projects in mscos - not sure if it works though.

looks like it doesn't work though (or at least has been prevented from working)

```
        public static DataTarget CreateFromDbgEng(IntPtr pDebugClient)
        {
            if (!RuntimeInformation.IsOSPlatform(OSPlatform.Windows))
                throw new PlatformNotSupportedException(GetPlatformMessage(nameof(CreateFromDbgEng), RuntimeInformation.OSDescription));
```


ICorDebug cross-platform:
https://github.com/dotnet/runtime/issues/8042

The x-plat debugging API is ICorDebug. It works well minus a few features that haven't been ported yet. This is what VSCode uses for example when debugging on Linux and mac.

I wasn't sure what you were eluding to with MethodImplOptions.InternalCall? ICorDebug is implemented in mscordbi.dll and meant to be loaded into the debugger which process which is seperate from the debuggee process that loaded coreclr.dll. The runtime doesn't need to load ICorDebug to execute IL, thus there is no need for the runtime to have any ecalls defined for it.



TODO:

I know that SOS on linux works as I have proof.

Need to check: what deependencies are loaded into SOS when a breakpoint is hit.

Also need to look into how vscode debugs c# on macos, what dependencies are loaded.

--

clrmd com base is at:
./src/Microsoft.Diagnostics.Runtime/src/Utilities/COMInterop/ComHelper.cs


dbgeng vs dac


dac is in dotnet/coreclr dotnet/diagnostics interacts with this.

--
SOS:

7065 void IssueDebuggerBPCommand ( CLRDATA_ADDRESS addr )

actual execution of setting breakpoint is at:

`7112 g_ExtControl->Execute(DEBUG_EXECUTE_NOT_LOGGED, buffer, 0);`

once we have the native address of the instruction we want to set a breakpoint for, how do we actually set it at the ASM level in memory?

sos is using the Instruction Pointer memory addresses of the JIITted code to set a breakpoint via llvm:

` sprintf_s(buffer, _countof(buffer), "bp %p", (void*) (size_t) addr);`

 this is quite interesting. something similar could be done with gdb (i think I originally put this in my todos).

 --
 need to check clrmd v1 and if it can be used with the dac stuff in clrmd v2 - can't it's .net framework only.

 look at clrdbg:
 https://github.com/Microsoft/MIEngine/wiki/What-is-CLRDBG

although the docs for ICorDebug suggest it is .net framework only:
https://docs.microsoft.com/en-us/dotnet/framework/unmanaged-api/debugging/icordebugbreakpoint-interface

where is ICorDebug implemented?

./runtime/src/coreclr/src/pal/prebuilt/inc/cordebug.h

---
A few things so far, going down the rabbit hole:

Symbol files can be read from pdbs. (example at: /Users/willvk/source/github/wilvk/dotnet-sos/source/own/pdb_testing/test_pdb)

clrmd can be used to inspect live processes: (example at: /Users/willvk/source/github/wilvk/dotnet-sos/source/own/clrmd_testing/debug)

final piece of the puzzle is to control breakpoints with ICorDebug using either DAC or otherwise.
looking at coreclr for how to do this.
