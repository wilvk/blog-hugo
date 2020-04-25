+++
title = "Simple Sos Debugging"
date = 2020-04-25T21:18:10+10:00
draft = false
tags = []
categories = []
+++

## Follow along at home:

The following is based on the docker images and dotnet core application located at the repository [here](https://github.com/wilvk/dotnet-sos)

## Getting started:

To begin with, start up the docker container:

```
make llvm
```

Then when in the container, build the solution and start the lldb debugger

```
make d-build-debug

make d-llvm-method-breakpoint

```

Once the program has started up, get a list of the clr stack with registers:

```
clrstack -r
```

We will see the `Child SP` and the `IP Call Site`

Using the `IP Call Site`, call `ip2md` to get the address of the method descriptor for the currently running instruction.

```
ip2md 00007FFF7CE92690
```

This gives us a lot of useful information including the `Class`, `MethodTable`, `Module` and `CurrentCodeAddr`

e.g.

```
(lldb) ip2md 00007FFF7CE92690
MethodDesc:   00007fff7cf3ff60
Method Name:          console.Program.GetTicksElapsed(Int64)
Class:                00007fff7cf81dc0
MethodTable:          00007fff7cf3ff88
mdToken:              0000000006000002
Module:               00007fff7cf3db98
IsJitted:             yes
Current CodeAddr:     00007fff7ce92690
Version History:
  ILCodeVersion:      0000000000000000
  ReJIT ID:           0
  IL Addr:            0000000000000000
     CodeAddr:           00007fff7ce92690  (MinOptJitted)
     NativeCodeVersion:  0000000000000000
Source file:  /work/source/own/test_debug/Program.cs @ 19
```


Then, if we want to look at all the methods in the object associated with this table, we can dump the method table at the address of `MethodTable` shown.

e.g.

```
(lldb) dumpmt -MD 00007fff7cf3ff88
EEClass:         00007FFF7CF81DC0
Module:          00007FFF7CF3DB98
Name:            console.Program
mdToken:         0000000002000002
File:            /work/source/own/test_debug/bin/Debug/netcoreapp3.1/test_debug.dll
BaseSize:        0x18
ComponentSize:   0x0
DynamicStatics:  false
ContainsPointers false
Slots in VTable: 7
Number of IFaces in IFaceMap: 0
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
00007FFF7CE80090 00007FFF7C48C728   NONE System.Object.Finalize()
00007FFF7CE80098 00007FFF7C48C738   NONE System.Object.ToString()
00007FFF7CE800A0 00007FFF7C48C748   NONE System.Object.Equals(System.Object)
00007FFF7CE800B8 00007FFF7C48C788   NONE System.Object.GetHashCode()
00007FFF7CE8BEA0 00007FFF7CF3FF78   NONE console.Program..ctor()
00007FFF7CE908A0 00007FFF7CF3FF48    JIT console.Program.Main(System.String[])
00007FFF7CE92690 00007FFF7CF3FF60    JIT console.Program.GetTicksElapsed(Int64)
```

In the column `JIT` we can see that our two methods in our application have been JITT-ed at runtime, whereas other methods that are part of this object have not been.

## Tracing an object back to it's ancestors

From `dumpmt`, we can see the value `EEClass` which is the address of the descriptor for the object `console.Program`.

If we then run `dumpobj` on this address we can see more info about this class.

```
(lldb) dumpclass 00007FFF7CF81DC0
Class Name:      console.Program
mdToken:         0000000002000002
File:            /work/source/own/test_debug/bin/Debug/netcoreapp3.1/test_debug.dll
Parent Class:    00007fff7ce77fe8
Module:          00007fff7cf3db98
Method Table:    00007fff7cf3ff88
Vtable Slots:    4
Total Method Slots:  5
Class Attributes:    100000
NumInstanceFields:   0
NumStaticFields:     0
```

We can see the `Parent Class` address here and `Module` address. the `Method Table` is the same as what we just saw.

Running `dumpclass` again with the `Parent Class` address this time, we can see what `console.Program` is inherited from:

```
(lldb) dumpclass 00007fff7ce77fe8
Class Name:      System.Object
mdToken:         0000000002000057
File:            /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/System.Private.CoreLib.dll
Parent Class:    0000000000000000
Module:          00007fff7c484020
Method Table:    00007fff7c48c798
Vtable Slots:    4
Total Method Slots:  5
Class Attributes:    102001
NumInstanceFields:   0
NumStaticFields:     0
```

This shows that `parent.Console` inherits from `System.Object`. we can also see that the name of `File` has changed from our application to `System.Private.CoreLib.dll`, indicating that it is a dotnet core framework library.

We can also see that the `Parent Class` value is `0` indicating this is the base object of our application (and of the dotnet core framework library).

If `program.Console` inherits from `System.Object`, and our source code only has two methods, namely:

- `console.Program.Main(System.String[])`
- `console.Program.GetTicksElapsed(Int64)`

We would expect that the other methods shown earlier must be inherited from `System.Object`. to verify this, we can run `dump -MD` with the address of the `Method Table` shown in order to do so:

```
(lldb) dumpmt -MD 00007fff7c48c798
EEClass:         00007FFF7CE77FE8
Module:          00007FFF7C484020
Name:            System.Object
mdToken:         0000000002000057
File:            /usr/share/dotnet/shared/Microsoft.NETCore.App/3.1.1/System.Private.CoreLib.dll
BaseSize:        0x18
ComponentSize:   0x0
DynamicStatics:  false
ContainsPointers false
Slots in VTable: 9
Number of IFaces in IFaceMap: 0
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
00007FFF7CE80090 00007FFF7C48C728   NONE System.Object.Finalize()
00007FFF7CE80098 00007FFF7C48C738   NONE System.Object.ToString()
00007FFF7CE800A0 00007FFF7C48C748   NONE System.Object.Equals(System.Object)
00007FFF7CE800B8 00007FFF7C48C788   NONE System.Object.GetHashCode()
00007FFF7CE80088 00007FFF7C48C718   NONE System.Object..ctor()
00007FFF7CE80078 00007FFF7C48C6E8   NONE System.Object.GetType()
00007FFF7CE80080 00007FFF7C48C700   NONE System.Object.MemberwiseClone()
00007FFF7CE800A8 00007FFF7C48C758   NONE System.Object.Equals(System.Object, System.Object)
00007FFF7CE800B0 00007FFF7C48C770   NONE System.Object.ReferenceEquals(System.Object, System.Object)
```

Looking at the `MethodDesc` table from the output does infact show the same method names and associated addresses:

```
console.Program:

00007FFF7CE80090 00007FFF7C48C728   NONE System.Object.Finalize()
00007FFF7CE80098 00007FFF7C48C738   NONE System.Object.ToString()
00007FFF7CE800A0 00007FFF7C48C748   NONE System.Object.Equals(System.Object)
00007FFF7CE800B8 00007FFF7C48C788   NONE System.Object.GetHashCode()

System.Object:

00007FFF7CE80090 00007FFF7C48C728   NONE System.Object.Finalize()
00007FFF7CE80098 00007FFF7C48C738   NONE System.Object.ToString()
00007FFF7CE800A0 00007FFF7C48C748   NONE System.Object.Equals(System.Object)
00007FFF7CE800B8 00007FFF7C48C788   NONE System.Object.GetHashCode()

```

### Inspecting other components:


From our `dumpmt` of `console.Program` earlier, we can use the `Module Address` to get more details about our application's assembly loaded into memory:

```
(lldb) dumpmodule 00007fff7cf3db98
Name:       /work/source/own/test_debug/bin/Debug/netcoreapp3.1/test_debug.dll
Attributes: PEFile SupportsUpdateableMethods
Assembly:   0000000000691510
PEFile:                  000000000068F7C0
ModuleId:                00007FFF7CF3E150
ModuleIndex:             0000000000000001
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 00007FFF7CF279B0
TypeRefToMethodTableMap: 00007FFF7CF279C8
MethodDefToDescMap:      00007FFF7CF27A58
FieldDefToDescMap:       00007FFF7CF27A78
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       00007FFF7CF27A88
AssemblyReferencesMap:   00007FFF7CF27A90
MetaData start address:  00007FFFF7E3F2CC (1540 bytes)
```

Then using the `Assembly` address, we can dump the assembly details:

```
(lldb) dumpassembly 0000000000691510
Parent Domain:      000000000062d820
Name:               /work/source/own/test_debug/bin/Debug/netcoreapp3.1/test_debug.dll
ClassLoader:        0000000000721890
  Module
  00007fff7cf3db98    /work/source/own/test_debug/bin/Debug/netcoreapp3.1/test_debug.dll

```

Well, that was fun. I hope you enjoyed following along!
