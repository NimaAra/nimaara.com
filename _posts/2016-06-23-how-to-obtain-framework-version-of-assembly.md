---
title: "How to obtain the framework version of a .NET assembly"
tags: "C# .NET"
redirect_from:
  - /how-to-obtain-the-framework-version-of-a-net-assembly/
  - /2016/06/23/how-to-obtain-the-framework-version-of-a-net-assembly/
---

## How to obtain the framework version of a .NET assembly

If you have ever wondered what version of the .NET framework a given assembly has been built against this post may be of some help.

When working on any application, I have the habit of logging as much information as possible not just about the process but also about the assemblies which the process is referencing and one important part of such information is the version of .NET framework the assembly has been built for e.g. _.NET 2_, _.NET 4.5.1_ etc. Note this is different from the run-time version.

### The not so accurate

If you look at the assembly object you can find the _ImageRuntimeVersion_ property which gets you the string representation of the _CLR_ saved in the file containing the assembly manifest (which by the way you can view by using a decompiler such as _ILSpy_ or _dotPeek_) however, this is not accurate as I explain below.

### The problem

The _ImageRuntimeVersion_ property returns either _.NET 2.0_ or _.NET 4.0_ regardless of what framework version you have built your assembly against; Meaning that if you have an assembly built in VisualStudio to target _.NET 4.5.1_, the value returned by the ImageRuntimeVersion is _.NET 4.0_. Not so helpful is it.

### What do we do?

There is _some_ hope otherwise why write this article!? Assemblies targeting _.NET 4_ and above are marked with the `TargetFrameworkAttribute` which conveniently includes the target framework we are after. This of course means that if your assembly is targeting a framework version less than _.NET 4_ i.e. _.NET 2_, _3_ and _3.5_, then as far as I am aware you are out of luck!

So armed with that knowledge one can obtain the version by calling an extension method such as:

```csharp
public static string GetFrameworkVersion(this Assembly assembly)
{
    var targetFrameAttribute = assembly.GetCustomAttributes(true)
        .OfType<TargetFrameworkAttribute>().FirstOrDefault();
    if (targetFrameAttribute == null)
    {
        return ".NET 2, 3 or 3.5";
    }

    return targetFrameAttribute.FrameworkDisplayName.Replace(".NET Framework", ".NET");
}
```
