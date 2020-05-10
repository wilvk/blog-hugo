+++
title = "Understanding Dotnet Core Pdb Files"
date = 2020-05-10T20:08:21+10:00
draft = false
tags = []
categories = []
+++

The following documents how to inspect pdb symbols and metadata with the dotnet core metadata visualizer, which is part of the [metadata-tools](https://github.com/dotnet/metadata-tools). I'm running this for a sample app I wrote for debugging in SOS, called `test_debug`. The source for it is available in my [dotnet-sos testing repo](https://github.com/wilvk/dotnet-sos/tree/master/source/own/test_debug).

Building my test app as debug instructs the compiler to output both the dll application assembly and the pdb debugging file. These two files are used in the following.

## understanding a breakpoint

The goal of this investigation is to determine how to set a breakpoint in our source code. To do this we need to know the IL code associated with a line of code in our application.

The following is the source code from our app:

```csharp
 1 using System;
 2
 3 namespace console
 4 {
 5   class Program
 6   {
 7     static void Main(string[] args)
 8     {
 9       while (true)
10       {
11         var lastTicks = DateTime.Now.Ticks;
12         System.Threading.Thread.Sleep(2000);
13         var ticksElapsed = GetTicksElapsed(lastTicks);
14         Console.WriteLine("ticks elapsed: " + ticksElapsed.ToString());
15       }
16     }
17
18     static long GetTicksElapsed(long lastTicks)
19     {
20       var currentTicks = DateTime.Now.Ticks;
21       var delta = currentTicks - lastTicks;
22       return delta;
23     }
24   }
25 }
```

Let's suppose we are interested in line 21, in the method `GetTicksElapsed`, where we are subtracting `lastTicks` from `currentTicke` in order to determine the time delta. This is in the test application as the file `Program.cs`.

An abbreviated version of our debug point would be `Program.cs:21`.

We then need to determine where in our IL code the same breakpoint would be.

We start with the `.pdb` file that is generated when the application is built in Debug mode.

Using `dotnetcore/metadata-tools/src/mdv` we can generate an output of all the symbols and mappings in our application.

## Document

Each C# code file is defined in our pdb file as a `Document`. We know the filename of our source code, so we need to find the matching Document number.

```txt
Document (index: 0x30, size: 24):
=====================================================================================================================================================================================================================================================================
   Name                                                                                                                            Language  HashAlgorithm  Hash
=====================================================================================================================================================================================================================================================================
1: '/Users/willvk/source/github/wilvk/dotnet-sos/source/own/test_debug/Program.cs' (#51)                                           C# (#2)   SHA-256 (#1)   0A-ED-B2-A4-21-42-46-17-C2-58-03-22-8C-DA-2B-62-0F-85-91-72-23-43-99-61-C7-7B-7B-BF-86-E7-AF-DD (#5e)
2: '/Users/willvk/source/github/wilvk/dotnet-sos/source/own/test_debug/obj/Debug/netcoreapp3.1/test_debug.AssemblyInfo.cs' (#10b)  C# (#2)   SHA-256 (#1)   DA-07-44-CB-7E-8D-B7-0E-E4-75-FC-77-4E-37-78-C0-DE-D4-C6-9B-AC-6F-79-BE-AF-3E-E5-71-25-75-79-A8 (#11f)
3: '/var/folders/04/0cmzblh51t717k__bn1jq8qm0000gn/T/.NETCoreApp,Version=v3.1.AssemblyAttributes.cs' (#19f)                        C# (#2)   SHA-256 (#1)   C0-05-EF-EB-23-4B-50-D8-1B-2F-23-F9-D3-7E-84-53-8C-0F-C2-14-FD-9D-65-22-8C-46-AA-DB-54-2F-5A-88 (#1ae)
```

Looking at the `Document` section we can see that document `1:` is for the file `.../Program.cs`.

Using our document number, we can then use the sequence points to determine what our IL instruction is that refers to our line of code. Sequence points are essentially a mapping between sections in our C# code text and the IL representation of that code.

There are two approaches we can take from here:
- Determine the Method Name to find the sequence points for a specific method in the document.
- Scan all sequence points for the document to find our line of code.

As the former is more error-prone and complex, we'll opt for the latter approach - scanning.

Using the `MethodDebugInformation` secion, we can scan it for all methods that are for Document #1 and that have a sequence point for line 21.

We find that method `#2` has a sequence point for line `21`:

```txt
2: #b5
{
  Locals: 0x11000002 (StandAloneSig)
  Document: #1

  IL_0000: (19, 5) - (19, 6)
  IL_0001: (20, 7) - (20, 45)
  IL_000F: (21, 7) - (21, 44)
  IL_0013: (22, 7) - (22, 20)
  IL_0017: (23, 5) - (23, 6)
}
```

The line that indicates it is for line 21 is:

```txt
  IL_000F: (21, 7) - (21, 44)
```

This shows us a few things:

Method `#2` has a sequence point between line 21, column 7 and line 21, column 44.

If we look at line 21 again, this matches up with the length of the code in the line:

```csharp
21       var delta = currentTicks - lastTicks;
         ^                                   ^
     col 7                               col 44
```

The value `#b5` refers to the location in the PDB blob where the sequence points are stored.

The sequence points are in our pdb `#Blob` output and can be seen as:

```txt
  b5 (SequencePoints): 02-00-00-01-13-05-01-00-26-02-04-0E-00-25-02-00-04-00-0D-02-00-04-00-01-02-7D
```

The IL code above also tells us the IL offset in this method for line 21 is `IL_000F`. And as the next IL offest is `IL_0013`, we can determine that line 21 includes the IL code between `IL_000F` and `IL_0012`.

## Finding the actual IL for our line of code.

Using the method number `2`, we can find the corresponding method number in the dll metadata generated by `metadata-tools` for our application.

```txt
Method (0x06, 0x1C):
=========================================================================================================================================================================================================================================================
   Name                     Signature              RVA         Parameters             GenericParameters  Attributes                                                                ImplAttributes  ImportAttributes  ImportName  ImportModule
=========================================================================================================================================================================================================================================================
1: 'Main' (#1e4)            void (string[]) (#4b)  0x00002050  0x08000001-0x08000001  nil                0x00000091 (PrivateScope, Private, Static, HideBySig)                     0               0                 nil         nil (ModuleRef)
2: 'GetTicksElapsed' (#28)  int64 (int64) (#51)    0x0000209C  0x08000002-0x08000002  nil                0x00000091 (PrivateScope, Private, Static, HideBySig)                     0               0                 nil         nil (ModuleRef)
3: '.ctor' (#201)           void () (#6)           0x000020C1  nil                    nil                0x00001886 (PrivateScope, Public, HideBySig, SpecialName, RTSpecialName)  0               0                 nil         nil (ModuleRef)
```

This tells us that method `#2` is called `GetTicksElapsed` and we can find the IL code for it by reference to `#28`. Looking further down where the Method IL is shows us the IL code we are after.

```txt
Method 'GetTicksElapsed' (#28) (0x06000002)
  Locals: int64, int64, typeref#d, int64 (#3a)
{
  // Code size       25 (0x19)
  .maxstack  2
  IL_0000:  nop
  IL_0001:  call       0x0A00000B
  IL_0006:  stloc.2
  IL_0007:  ldloca.s   V_2
  IL_0009:  call       0x0A00000C
  IL_000e:  stloc.0
  IL_000f:  ldloc.0
  IL_0010:  ldarg.0
  IL_0011:  sub
  IL_0012:  stloc.1
  IL_0013:  ldloc.1
  IL_0014:  stloc.3
  IL_0015:  br.s       IL_0017
  IL_0017:  ldloc.3
  IL_0018:  ret
}
```

### An alternative approach from the pdb output to IL

_An alternative way of cross-referencing between the two would be to check the `CustomDebuggingInformation` in the pdb output, that gives us the offset directly.
In the PDB output, the method number we found before could then be used to find the offset of our method in the dll metadata._

```txt
CustomDebugInformation (index: 0x37, size: 12):
===================================================================================
   Parent                  Kind                     Value
===================================================================================
1: 0x06000001 (MethodDef)  EnC Local Slot Map (#3)  01-2A-01-80-83-00-02-09 (#ac)
2: 0x06000002 (MethodDef)  EnC Local Slot Map (#3)  01-0D-01-3A-00-16-01 (#d0)
```

_This tells us that our offset for method `#2` is `0x06000002`.
Then, in our generated dll output, we can find the method that has an offset of `0x06000002` to find our IL code._

## Interpreting the IL

Our sequence points earlier showed us that the IL was between `IL_000F` and `IL_0012`. Inspecting the specific section of IL:

```txt
  IL_000f:  ldloc.0
  IL_0010:  ldarg.0
  IL_0011:  sub
  IL_0012:  stloc.1
```

Then looking at our line of code again:

```csharp
21       var delta = currentTicks - lastTicks;
```

We could almost guess at what the IL is doing:

- loading the value at location 0 for an operation
- loading argument 1 from the method for an operation
- performing a subtraction on the two loaded values
- storing the result in location 1

This seems to match up pretty well with what we would expect and gives us some certainty that this is correct.

The next part is to use the method offset and the method's IL offset to set a breakpoint in our running application.

For that we will use `clrmd` and will be the topic of another post.
