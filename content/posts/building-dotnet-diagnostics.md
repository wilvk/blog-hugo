+++
title = "Building Dotnet Diagnostics"
date = 2020-05-31T14:18:15+10:00
draft = false
tags = []
categories = []
+++

I'm getting an error trying to build `dotnet/diagnostics`:

```bash
root@85bd9fba8925:/app/diagnostics# dotnet build
A compatible installed .NET Core SDK for global.json version [3.0.100] from [/app/diagnostics/global.json] was not found
Install the [3.0.100] .NET Core SDK or update [/app/diagnostics/global.json] with an installed .NET Core SDK:
  3.1.200-preview-014999 [/app/sdk/.dotnet/sdk]
```

Hmmm. ok, let's look at that global.json:

```bash
root@85bd9fba8925:/app/diagnostics# cat /app/diagnostics/global.json
{
  "sdk": {
    "version": "3.0.100"
  },
  "tools": {
    "dotnet": "3.0.100"
  },
  "msbuild-sdks": {
    "Microsoft.DotNet.Arcade.Sdk": "1.0.0-beta.19309.1"
  }
}
```

Ok, so it needs sdk `3.0.100`. Let's look at our sdk source tag:

```bash
root@85bd9fba8925:/app/sdk# git branch -vva
* (HEAD detached at v3.1.201)                                                       71a6432 Update branding to 3.1.201 servicing (#10782)     master                                                                            77b5b6b [origin/master] Update dependencies from https:/
/github.com/dotnet/aspnetcore build 20200423.13 (#11404)                                                                                      remotes/origin/HEAD                                                               -> origin/master
  remotes/origin/ViktorHofer-VSTest-verbosity                                       6b05bc2 Change test assumption to minimum                 remotes/origin/ViktorHofer-vstest-verbosity                                       9c50d39 Change VSTest invocation verbosity to minimal
  remotes/origin/conflicting-transitive-content-fix                                 e2b610a Fix transitive content conflicts                  remotes/origin/darc-master-10da8ea1-a2de-458e-ba17-131541dfc7db                   64d8c02 Update dependencies from https://github.com/dotn
et/aspnetcore build 20200423.18
```

Ah, we're on sdk `v3.1.201`.

Ok, so what branch is diagnostics on then?

```bash
root@85bd9fba8925:/app/diagnostics# git branch -vva
* (HEAD detached at 3.1.57502)                6767a9a Merge pull request #648 from mikem8361/release/stable
  master                                      197f647 [origin/master] Servicing testing fixes (#1061)
  remotes/origin/ForStable                    b238bd3 Cross dbi (#1036)
  remotes/origin/HEAD                         -> origin/master
  remotes/origin/master                       197f647 Servicing testing fixes (#1061)
  remotes/origin/release/3.0                  0adfc03 Various fixes from master (#554)
```

Strange. So, diagnostics is on `3.1.57502` but it needs sdk `3.0.100` to run.

Ok, so let's look for a tag for sdk `3.0.100`.


```bash
root@85bd9fba8925:/app/sdk# git tag
dev15.0-rc1
dev15.0-rc2
dev15.0-rc3
dev15.0-rc4
dev15.0.0
v
v2.0.2
v2.0.3
v2.1.2
v2.1.200
v2.1.3
v2.1.300
v2.1.300-preview1-62608--07
v2.1.4
v2.1.510
v2.1.607
v2.2.110
v2.2.207
v3.0.100-preview3.19124.1
v3.0.100-preview7.19354.1
v3.0.100-preview7.19362.3
v3.0.100-preview8.19406.1
v3.0.100-preview9.19426.11
v3.0.100-rc1.19458.1
v3.0.100-rc2.19467.3
v3.0.101
v3.0.102
v3.0.103
v3.0.2
v3.1.1
v3.1.100-preview1.19506.1
v3.1.100-preview2.19528.2
v3.1.100-preview3.19553.1
v3.1.100-rtm.19565.4
v3.1.101
v3.1.102
v3.1.103
v3.1.200
v3.1.201
v5.0.100-preview.2.20176.4
v5.0.100-preview.3.20216.1
```

Hmmm.. where's `3.0.100`? I guess `v3.0.103` might have to do. Will this work? Let's check. (And what happened to v4? Too close in numbering to Dotnet Framework v4.5 maybe?)

```bash
root@85bd9fba8925:/app/sdk# git checkout v3.0.103
Previous HEAD position was 71a6432... Update branding to 3.1.201 servicing (#10782)
HEAD is now at 316fbdd... [release/3.0.1xx] Update dependencies from dotnet/core-setup (#4180)

root@85bd9fba8925:/app/sdk# ./build.sh
...

...
  Microsoft.NET.Build.Extensions.Tasks -> /app/sdk/artifacts/bin/Debug/Sdks/Microsoft.NET.Build.Extensions/msbuildExtensions/Microsoft/Microsoft.NET.Build.Extensions/tools/netcoreapp2.1/Microsoft.NET.Build.Extensions.Tasks.dll
  Microsoft.NET.Build.Extensions.Tasks -> /app/sdk/artifacts/bin/Debug/Sdks/Microsoft.NET.Build.Extensions/msbuildExtensions/Microsoft/Microsoft.NET.Build.Extensions/tools/net472/Microsoft.NET.Build.Extensions.Tasks.dll
  Microsoft.NET.Build.Extensions.Tasks.UnitTests -> /app/sdk/artifacts/bin/Tests/Microsoft.NET.Build.Extensions.Tasks.UnitTests/Debug/Microsoft.NET.Build.Extensions.Tasks.UnitTests.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:34.19
root@85bd9fba8925:/app/sdk#

```

Cool. So It builds. Let's make it the system dotnet sdk.

```bash
root@85bd9fba8925:/app/sdk# cd .dotnet/
root@85bd9fba8925:/app/sdk/.dotnet# chmod +x dotnet-install.sh
root@85bd9fba8925:/app/sdk/.dotnet# ./dotnet-install.sh
dotnet_install: Warning: Unable to locate zlib. Probable prerequisite missing; install zlib.
dotnet-install: .NET Core SDK version 3.1.300 is already installed.
dotnet-install: Adding to current process PATH: `/root/.dotnet`. Note: This change will be visible only when sourcing script.
dotnet-install: Installation finished successfully.
```

Ok, and what version of dotnet do we have now?

```bash
root@85bd9fba8925:/app/sdk/.dotnet# dotnet --list-sdks
3.0.103 [/app/sdk/.dotnet/sdk]
3.1.200-preview-014999 [/app/sdk/.dotnet/sdk]
```

Cool. So it looks like it's listed there now. Great.

So let's try  building diagnostics again.

```bash
root@85bd9fba8925:/app/diagnostics# ./build.sh
  Restore completed in 19.83 ms for /root/.nuget/packages/microsoft.dotnet.arcade.sdk/1.0.0-beta.19309.1/tools/Tools.proj.
  Restore completed in 9.25 ms for /app/diagnostics/src/Microsoft.Diagnostics.DebugServices/Microsoft.Diagnostics.DebugServices.csproj.
  Restore completed in 9.72 ms for /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/Microsoft.Diagnostics.TestHelpers.csproj.
  Restore completed in 9.82 ms for /app/diagnostics/src/Microsoft.Diagnostics.Tools.RuntimeClient/Microsoft.Diagnostics.Tools.RuntimeClient.
csproj.

...

 89%] Linking CXX shared library libsos.so
[100%] Built target sos
Install the project...
-- Install configuration: "DEBUG"
-- Installing: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./libsosplugin.so
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./SOS.NETCore.dll
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./SOS.NETCore.pdb
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./Microsoft.FileFormats.dll
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./Microsoft.SymbolStore.dll
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./System.Reflection.Metadata.dll
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./System.Collections.Immutable.dll
-- Installing: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./libsos.so
-- Up-to-date: /app/diagnostics/artifacts/bin/Linux.x64.Debug/./sosdocsunix.txt
/app/diagnostics
Copied SOS to /app/diagnostics/artifacts/bin/dotnet-sos/Debug/netcoreapp2.1/publish/linux-x64
Copied SOS to /app/diagnostics/artifacts/bin/dotnet-dump/Debug/netcoreapp2.1/publish/linux-x64
BUILD: Repo sucessfully built.
BUILD: Product binaries are available at /app/diagnostics/artifacts/bin/Linux.x64.Debug
```

Let's try our dotnet build again.

```bash
root@85bd9fba8925:/app/diagnostics# dotnet build
Microsoft (R) Build Engine version 16.3.2+e481bbf88 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

MSBUILD : error MSB1011: Specify which project or solution file to use because this folder contains more than one project or solution file.
```

Ok, well that looks better than last time. Let's specify the solution as well this time.

```bash
root@85bd9fba8925:/app/diagnostics# dotnet build ./diagnostics.sln
Microsoft (R) Build Engine version 16.3.2+e481bbf88 for .NET Core
Copyright (C) Microsoft Corporation. All rights reserved.

  Restore completed in 16.26 ms for /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/Microsoft.Diagnostics.TestHelpers.csproj.
  Restore completed in 16.23 ms for /app/diagnostics/src/Microsoft.Diagnostics.Tools.RuntimeClient/Microsoft.Diagnostics.Tools.RuntimeClient.csproj.
  Restore completed in 16.23 ms for /app/diagnostics/src/Microsoft.Diagnostics.DebugServices/Microsoft.Diagnostics.DebugServices.csproj.
  Restore completed in 16.23 ms for /app/diagnostics/src/Microsoft.Diagnostics.Repl/Microsoft.Diagnostics.Repl.csproj.
  Restore completed in 0.79 ms for /app/diagnostics/src/SOS/SOS.Package/SOS.Package.csproj.

  ...

    Microsoft.Diagnostics.Repl -> /app/diagnostics/artifacts/bin/Microsoft.Diagnostics.Repl/Debug/netstandard2.0/Microsoft.Diagnostics.Repl.dll
  SOS.Hosting -> /app/diagnostics/artifacts/bin/SOS.Hosting/Debug/netstandard2.0/SOS.Hosting.dll
  dotnet-dump -> /app/diagnostics/artifacts/bin/dotnet-dump/Debug/netcoreapp2.1/dotnet-dump.dll
  dotnet-counters -> /app/diagnostics/artifacts/bin/dotnet-counters/Debug/netcoreapp2.1/dotnet-counters.dll
  DotnetCounters.UnitTests -> /app/diagnostics/artifacts/bin/DotnetCounters.UnitTests/Debug/netcoreapp3.0/DotnetCounters.UnitTests.dll

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:03.74
```

And I think it's working now. That wasn't too difficult, now, was it?

Well, let's run our tests, just to be sure.


```bash
root@85bd9fba8925:/app/diagnostics# ./build.sh --test

  Restore completed in 17.87 ms for /root/.nuget/packages/microsoft.dotnet.arcade.sdk/1.0.0-beta.19309.1/tools/Tools.proj.
  Restore completed in 189.66 ms for /app/diagnostics/src/Microsoft.Diagnostics.Tools.RuntimeClient/Microsoft.Diagnostics.Tools.RuntimeClien
t.csproj.
  Restore completed in 189.81 ms for /app/diagnostics/src/Microsoft.Diagnostics.Repl/Microsoft.Diagnostics.Repl.csproj.
  Restore completed in 189.77 ms for /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/Microsoft.Diagnostics.TestHelpers.csproj.
  Restore completed in 189.6 ms for /app/diagnostics/src/Microsoft.Diagnostics.DebugServices/Microsoft.Diagnostics.DebugServices.csproj.
  Restore completed in 17.88 ms for /app/diagnostics/src/SOS/SOS.Package/SOS.Package.csproj.
  Restore completed in 22.99 ms for /app/diagnostics/src/SOS/SOS.InstallHelper/SOS.InstallHelper.csproj.

  ...

    Tests succeeded: /app/diagnostics/artifacts/bin/DotnetTrace.UnitTests/Debug/netcoreapp3.0/DotnetTrace.UnitTests.dll [netcoreapp3.0|x64]
  Tests succeeded: /app/diagnostics/artifacts/bin/DotnetCounters.UnitTests/Debug/netcoreapp3.0/DotnetCounters.UnitTests.dll [netcoreapp3.0|x64]
  Tests succeeded: /app/diagnostics/artifacts/bin/EventPipe.UnitTests/Debug/netcoreapp3.0/EventPipe.UnitTests.dll [netcoreapp3.0|x64]
XUnit : error : Tests failed: /app/diagnostics/artifacts/TestResults/Debug/SOS.UnitTests_netcoreapp2.0_x64.html [netcoreapp2.0|x64] [/app/diagnostics/src/SOS/SOS.UnitTests/SOS.UnitTests.csproj]

Build FAILED.

XUnit : error : Tests failed: /app/diagnostics/artifacts/TestResults/Debug/SOS.UnitTests_netcoreapp2.0_x64.html [netcoreapp2.0|x64] [/app/diagnostics/src/SOS/SOS.UnitTests/SOS.UnitTests.csproj]
    0 Warning(s)
    1 Error(s)

Time Elapsed 00:00:09.84
Build failed (exit code '1').
```

Hmmm again.. it looks like it needs dotnet sdk v2.0 now. Ok, so let's try building and installing that now.

Let's use the latest patch version matching 2.0:


```bash
root@85bd9fba8925:/app/sdk# git checkout v2.0.3

Previous HEAD position was 316fbdd... [release/3.0.1xx] Update dependencies from dotnet/core-setup (#4180)
HEAD is now at fc3e52d... Publish the SDK NuGet packages to the Transport Feed (#1687)
```

And build our repository:

```bash
root@85bd9fba8925:/app/sdk# ./build.sh
dotnet-install: Calling: machine_has curl
dotnet-install: Calling: calculate_vars
dotnet-install: Calling: get_normalized_architecture_from_architecture <auto>
dotnet-install: Calling: get_machine_architecture
dotnet-install: Calling: get_normalized_architecture_from_architecture x64
dotnet-install: normalized_architecture=x64

...
```

2 hours later (well not quite), we can install our v2.0 sdk.

```bash
root@85bd9fba8925:/app/sdk/.dotnet# ./dotnet-install.sh

dotnet_install: Warning: Unable to locate zlib. Probable prerequisite missing; install zlib.
dotnet-install: .NET Core SDK version 3.1.300 is already installed.
dotnet-install: Adding to current process PATH: `/root/.dotnet`. Note: This change will be visible only when sourcing script.
dotnet-install: Installation finished successfully.
```

So let's check it's listed with dotnet:

```bash
root@85bd9fba8925:/app/sdk/.dotnet# dotnet --list-sdks
3.0.103 [/app/sdk/.dotnet/sdk]
3.1.200-preview-014999 [/app/sdk/.dotnet/sdk]
```

Well, that's strange. Where is it?

Let's have a look at our build log for a hint:

```bash
dotnet-install: Extracting zip from https://dotnetcli.azureedge.net/dotnet/Runtime/1.1.2/dotnet-ubuntu.16.04-x64.1.1.2.tar.gz               dotnet-install: Adding to current process PATH: `/app/sdk/.dotnet_cli`. Note: This change will be visible only when sourcing script.
dotnet-install: Installation finished successfully.
```

Ok, so it looks like it's installed in a different path so the two sdks can be installed side-by-side. Fair enough.

Let's go to that path and check the version:

```bash
root@85bd9fba8925:/app/sdk/.dotnet_cli# ./dotnet --version
2.0.2-vspre-006963
```

Ok, so version `v2.0.3` of the sdk is built from branch `2.0.2-vspre-006963`. Maybe this was before pre-release versions were tagged separately.

So let's try to run our diagnostics tests again.

```bash
root@85bd9fba8925:/app/diagnostics# ./build.sh --test
  Restore completed in 29.21 ms for /root/.nuget/packages/microsoft.dotnet.arcade.sdk/1.0.0-beta.19309.1/tools/Tools.proj.
  Restore completed in 19.94 ms for /app/diagnostics/src/Microsoft.Diagnostics.Tools.RuntimeClient/Microsoft.Diagnostics.Tools.RuntimeClient
.csproj.
  Restore completed in 23.06 ms for /app/diagnostics/src/Microsoft.Diagnostics.Repl/Microsoft.Diagnostics.Repl.csproj.
  Restore completed in 25.67 ms for /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/Microsoft.Diagnostics.TestHelpers.csproj.
  Restore completed in 25.33 ms for /app/diagnostics/src/Microsoft.Diagnostics.DebugServices/Microsoft.Diagnostics.DebugServices.csproj.
  Restore completed in 1.59 ms for /app/diagnostics/src/SOS/SOS.InstallHelper/SOS.InstallHelper.csproj.

...

  Tests succeeded: /app/diagnostics/artifacts/bin/DotnetCounters.UnitTests/Debug/netcoreapp3.0/DotnetCounters.UnitTests.dll [netcoreapp3.0|x64]
  Tests succeeded: /app/diagnostics/artifacts/bin/EventPipe.UnitTests/Debug/netcoreapp3.0/EventPipe.UnitTests.dll [netcoreapp3.0|x64]
XUnit : error : Tests failed: /app/diagnostics/artifacts/TestResults/Debug/SOS.UnitTests_netcoreapp2.0_x64.html [netcoreapp2.0|x64] [/app/diagnostics/src/SOS/SOS.UnitTests/SOS.UnitTests.csproj]

Build FAILED.

XUnit : error : Tests failed: /app/diagnostics/artifacts/TestResults/Debug/SOS.UnitTests_netcoreapp2.0_x64.html [netcoreapp2.0|x64] [/app/diagnostics/src/SOS/SOS.UnitTests/SOS.UnitTests.csproj]
    0 Warning(s)
    1 Error(s)

Time Elapsed 00:00:10.65
Build failed (exit code '1').
```

Hmmm.. still failing for the same reason. It probably doesn't know about sdk v2.0.

Let's dig a bit deeper into the error log:

```bash
root@85bd9fba8925:/app/diagnostics# cat /app/diagnostics/artifacts/TestResults/Debug/SOS.UnitTests_netcoreapp2.0_x64.html

...


        00:00.390: [40m[32minfo[39m[22m[49m: Microsoft.AspNetCore.Routing.EndpointMiddleware[0]
        00:00.390:       Executing endpoint '/ HTTP: GET'
        00:00.395: Connecting to pipe SOSRunner.37508.WebApp3
        Running Process: /app/diagnostics/.dotnet/dotnet /app/diagnostics/artifacts/bin/dotnet-dump/Debug/netcoreapp2.1/publish/dotnet-dump.
dll collect --process-id 37656 --output /app/diagnostics/artifacts/tmp/Debug/dumps/ProjectK/3.0.0/netcoreapp3.0/WebApp3.Heap.dmp
        Working Directory:
        {
            00:00.219: Writing minidump with heap to /app/diagnostics/artifacts/tmp/Debug/dumps/ProjectK/3.0.0/netcoreapp3.0/WebApp3.Heap.dm
p
        00:03.947: Writing minidump with heap to file /app/diagnostics/artifacts/tmp/Debug/dumps/ProjectK/3.0.0/netcoreapp3.0/WebApp3.Heap.d
mp
        00:03.947: Written 188710912 bytes (46072 pages) to core file
            00:03.550: Complete
        }
        Exit code: 0 ( 00:03.560 elapsed)
        }
        Killing process: 00:03.961: Kill() was called
    }                                                                                                                                           Exit code: 137 ( 00:03.975 elapsed)

}                                                                                                                                           SOSRunner processing SOS.WebApp3
{                                                                                                                                           System.IO.FileNotFoundException: Native debugger path not set or does not exist:
   at SOSRunner.StartDebugger(TestInformation information, DebuggerAction action) in /app/diagnostics/src/SOS/SOS.UnitTests/SOSRunner.cs:lin
e 413

...

```

It looks like one of the tests is running a process:

```bash
/app/diagnostics/.dotnet/dotnet /app/diagnostics/artifacts/bin/dotnet-dump/Debug/netcoreapp2.1/publish/dotnet-dump.
dll collect --process-id 37656 --output /app/diagnostics/artifacts/tmp/Debug/dumps/ProjectK/3.0.0/netcoreapp3.0/WebApp3.Heap.dmp
```

So firstly, let's try running that single project's tests:

```bash
root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# cd /app/diagnostics/src/SOS/SOS.UnitTests
root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# dotnet test
Test run for /app/diagnostics/artifacts/bin/SOS.UnitTests/Debug/netcoreapp2.0/SOS.UnitTests.dll(.NETCoreApp,Version=v2.0)
Microsoft (R) Test Execution Command Line Tool Version 16.3.0
Copyright (c) Microsoft Corporation.  All rights reserved.

Starting test execution, please wait...

A total of 1 test files matched the specified pattern.

[xUnit.net 00:00:00.85]     SOS.TaskNestedException(config: projectk.sdk.prebuilt.5.0.0-alpha.1.19564.1) [FAIL]

...

  X SOS.LLDBPluginTests(config: projectk.sdk.prebuilt.3.0.0) [3ms]
  Error Message:
   System.ArgumentException : LLDB_PATH environment variable not set
  Stack Trace:
     at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 245
--- End of stack trace from previous location where exception was thrown ---
  Standard Output Messages:
 Starting SOS.LLDBPluginTests
 {
 System.ArgumentException: LLDB_PATH environment variable not set
    at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 245
 }


                                                                                                                                              X SOS.LLDBPluginTests(config: projectk.sdk.prebuilt.2.1.14) [6ms]
  Error Message:
   System.ArgumentException : LLDB_PATH environment variable not set
  Stack Trace:
     at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 245
--- End of stack trace from previous location where exception was thrown ---
  Standard Output Messages:
 Starting SOS.LLDBPluginTests
 {
 System.ArgumentException: LLDB_PATH environment variable not set
    at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 245
 }



Test Run Failed.
Total tests: 47
     Failed: 47
 Total time: 9.8197 Seconds
 ```

 Ohhh.. everything failed that time.. Hmmm..

 Well, what is the error then?

 ```bash
   Error Message:
   System.ArgumentException : LLDB_PATH environment variable not set
  Stack Trace:
     at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 245
--- End of stack trace from previous location where exception was thrown ---
```

Ah, it looks like there is a dependency on lldb.

Let's look at the offending file:

```bash
root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# cat -n SOS.cs

...

   244              // Get lldb path
   245              arguments.AppendFormat("--lldb {0} ", Environment.GetEnvironmentVariable("LLDB_PATH") ?? throw new ArgumentException("LL
DB_PATH environment variable not set"));

...

```

Does it just need to be set? Let's try.

```bash

root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# export LLDB_PATH

root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# dotnet test                                                                       Test run for /app/diagnostics/artifacts/bin/SOS.UnitTests/Debug/netcoreapp2.0/SOS.UnitTests.dll(.NETCoreApp,Version=v2.0)                   Microsoft (R) Test Execution Command Line Tool Version 16.3.0
Copyright (c) Microsoft Corporation.  All rights reserved.

Starting test execution, please wait...

...

  X SOS.LLDBPluginTests(config: projectk.sdk.prebuilt.2.1.14) [52ms]
  Error Message:
   System.Exception : Process returned exit code 2, expected 0
Command Line: /usr/bin/python /app/diagnostics/src/SOS/lldbplugin.tests/test_libsosplugin.py --lldb  --host "/app/diagnostics/.dotnet/dotnet --fx-version 2.1.14 " --plugin /app/diagnostics/artifacts/bin/Linux.x64.Debug/libsosplugin.so --logfiledir /app/diagnostics/artifacts/TestResults/Debug/sos.unittests_2020_05_29_01_35_52_2983/2.1.14 --assembly /app/diagnostics/artifacts/bin/TestDebuggee/Debug/netcoreapp2.1/TestDebuggee.dll
Working Directory: /app/diagnostics/src/SOS/lldbplugin.tests
  Stack Trace:
     at Microsoft.Diagnostics.TestHelpers.ProcessRunner.InternalWaitForExit(Task`1 startProcessTask, Task stdOutTask, Task stdErrTask) in /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/ProcessRunner.cs:line 436
   at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 286
--- End of stack trace from previous location where exception was thrown ---
  Standard Output Messages:
 Starting SOS.LLDBPluginTests
 {
     Running Process: /usr/bin/python /app/diagnostics/src/SOS/lldbplugin.tests/test_libsosplugin.py --lldb  --host "/app/diagnostics/.dotnet/dotnet --fx-version 2.1.14 " --plugin /app/diagnostics/artifacts/bin/Linux.x64.Debug/libsosplugin.so --logfiledir /app/diagnostics/artifacts/TestResults/Debug/sos.unittests_2020_05_29_01_35_52_2983/2.1.14 --assembly /app/diagnostics/artifacts/bin/TestDebuggee/Debug/netcoreapp2.1/TestDebuggee.dll
     Working Directory: /app/diagnostics/src/SOS/lldbplugin.tests
     Additional Environment Variables: COMPlus_LogFacility=0xffffffbf, COMPlus_LogLevel=6, COMPlus_StressLog=1, COMPlus_StressLogSize=65536
     {
         STDERROR: 00:00.032: usage: test_libsosplugin.py [-h] [--lldb LLDB] [--host HOST] [--plugin PLUGIN]
         STDERROR: 00:00.032:                             [--logfiledir LOGFILEDIR] [--assembly ASSEMBLY]
         STDERROR: 00:00.032:                             [--timeout TIMEOUT] [--regex REGEX]
         STDERROR: 00:00.032:                             [--repeat REPEAT]
         STDERROR: 00:00.032:                             [unittest_args [unittest_args ...]]
         STDERROR: 00:00.032: test_libsosplugin.py: error: argument --lldb: expected one argument
     }
     Exit code: 2 ( 00:00.036 elapsed)

 System.Exception: Process returned exit code 2, expected 0
 Command Line: /usr/bin/python /app/diagnostics/src/SOS/lldbplugin.tests/test_libsosplugin.py --lldb  --host "/app/diagnostics/.dotnet/dotnet --fx-version 2.1.14 " --plugin /app/diagnostics/artifacts/bin/Linux.x64.Debug/libsosplugin.so --logfiledir /app/diagnostics/artifacts/TestResults/Debug/sos.unittests_2020_05_29_01_35_52_2983/2.1.14 --assembly /app/diagnostics/artifacts/bin/TestDebuggee/Debug/netcoreapp2.1/TestDebuggee.dll
 Working Directory: /app/diagnostics/src/SOS/lldbplugin.tests
    at Microsoft.Diagnostics.TestHelpers.ProcessRunner.InternalWaitForExit(Task`1 startProcessTask, Task stdOutTask, Task stdErrTask) in /app/diagnostics/src/Microsoft.Diagnostics.TestHelpers/ProcessRunner.cs:line 436
    at SOS.LLDBPluginTests(TestConfiguration config) in /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs:line 286
 }



Test Run Failed.
Total tests: 47
     Failed: 47
 Total time: 9.6904 Seconds

 ```

 It does need to be set, but now we get another error later on:

 ```bash
 root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# cat -n /app/diagnostics/src/SOS/SOS.UnitTests/SOS.cs

 ...

   285              // Wait for the debuggee to finish
   286              await processRunner.WaitForExit();
```

So, this looks like a bit of a dead-end and we should probably install lldb and see if that resolves it. But, what version of lldb should we install? Let's try the default version (lldb-8).

```bash
root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# apt-get update -y && apt-get install -y llvm-8 lldb-8
Hit:2 http://security.ubuntu.com/ubuntu xenial-security InRelease
Hit:3 http://archive.ubuntu.com/ubuntu xenial InRelease
Hit:4 http://archive.ubuntu.com/ubuntu xenial-updates InRelease
Hit:5 https://apt.kitware.com/ubuntu xenial InRelease
Hit:6 http://archive.ubuntu.com/ubuntu xenial-backports InRelease
Hit:1 https://apt.llvm.org/xenial llvm-toolchain-xenial-9 InRelease


...

Preparing to unpack .../lldb-8_1%3a8-3~ubuntu16.04.1_amd64.deb ...
Unpacking lldb-8 (1:8-3~ubuntu16.04.1) ...
Processing triggers for libc-bin (2.23-0ubuntu11) ...
Setting up libllvm8:amd64 (1:8-3~ubuntu16.04.1) ...
Setting up liblldb-8 (1:8-3~ubuntu16.04.1) ...
Setting up llvm-8-runtime (1:8-3~ubuntu16.04.1) ...
Setting up llvm-8 (1:8-3~ubuntu16.04.1) ...
Setting up llvm-8-dev (1:8-3~ubuntu16.04.1) ...
Setting up python-lldb-8 (1:8-3~ubuntu16.04.1) ...
Setting up lldb-8 (1:8-3~ubuntu16.04.1) ...
Processing triggers for libc-bin (2.23-0ubuntu11) ...
```

Cool. So let's try again:

```bash
root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# export LLDB_PATH=/usr/bin/lldb-8                                                  root@85bd9fba8925:/app/diagnostics/src/SOS/SOS.UnitTests# dotnet test
Test run for /app/diagnostics/artifacts/bin/SOS.UnitTests/Debug/netcoreapp2.0/SOS.UnitTests.dll(.NETCoreApp,Version=v2.0)
Microsoft (R) Test Execution Command Line Tool Version 16.3.0                                                                               Copyright (c) Microsoft Corporation.  All rights reserved.
                                                                                                                                            Starting test execution, please wait...
                                                                                                                                            A total of 1 test files matched the specified pattern.
                                                                                                                                            [xUnit.net 00:05:29.24]     SOS.StackAndOtherTests(config: projectk.cli.5.0.0-alpha.1.19564.1) [FAIL]
                                                                                                                                              X SOS.StackAndOtherTests(config: projectk.cli.5.0.0-alpha.1.19564.1) [39s 342ms]
  Error Message:
   Microsoft.Diagnostics.TestHelpers.TestStepException : The Build Debuggee test step failed.
Original Error: Process returned exit code 1, expected 0
Command Line: /app/diagnostics/.dotnet/dotnet restore --configfile /app/diagnostics/artifacts/Debuggees/embedded/SymbolTestApp/NuGet.config
--packages "/root/.nuget/packages" /p:RuntimeFrameworkVersion=3.0.0 /p:BuildProjectFramework=netcoreapp3.0 /p:DebugType=embedded            Working Directory: /app/diagnostics/artifacts/Debuggees/embedded/SymbolTestApp
   at Microsoft.Diagnostics.TestHelpers.ProcessRunner.InternalWaitForExit(Task`1 startProcessTask, Task stdOutTask, Task stdErrTask) in /app
/diagnostics/src/Microsoft.Diagnostics.TestHelpers/ProcessRunner.cs:line 436                                                                   at Microsoft.Diagnostics.TestHelpers.DotNetBuildDebuggeeTestStep.Restore(String extraArgs, ITestOutputHelper output) in /app/diagnostics/
src/Microsoft.Diagnostics.TestHelpers/DotNetBuildDebuggeeTestStep.cs:line 208

```

.. an hour passes...

```bash


 }
 SOSRunner processing SOS.Overflow
 {
 System.IO.FileNotFoundException: Native debugger path not set or does not exist:
    at SOSRunner.StartDebugger(TestInformation information, DebuggerAction action) in /app/diagnostics/src/SOS/SOS.UnitTests/SOSRunner.cs:line 413


                                                                                                                                            [xUnit.net 00:08:44.29]     SOS.WebApp [FAIL]
                                                                                                                                              X SOS.WebApp [1ms]
  Error Message:
   System.InvalidOperationException : The test method expected 1 parameter value, but 0 parameter values were provided.


Test Run Failed.
Total tests: 47
     Passed: 41
     Failed: 6
 Total time: 16.5891 Minutes

```

Ok, so only 6 out of 47 failed this time. That's some progress.


But why did the 6 fail? I'll look at that in the next blog post.
