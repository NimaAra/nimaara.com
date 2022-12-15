---
title: .NET Core 5, Entity Framework 7, ASP.NET 5 (vNext) & MVC 6 are all Dead!
tags: "C# .NET EF ASP.NET"
redirect_from:
  - /entity-framework-7-asp-net-5mvc-6-is-dead-asp-net-5-are-dead/
  - /2016/01/20/entity-framework-7-asp-net-5mvc-6-is-dead-asp-net-5-are-dead/
---

That's it, it is official. Today in their weekly community stand-up the _ASP.NET_ team [announced](https://www.youtube.com/watch?v=FSf83_TU5Yg) what many .NET developers had been waiting to hear ever since _ASP.NET vNext_ was [announced](http://www.hanselman.com/blog/IntroducingASPNETVNext.aspx) back in 2014, that is there is no longer:

- .NET Core 5
- ASP.NET 5 (vNext)
- MVC 6
- Entity Framework 7

Instead we now have:

- .NET Core 1.0
- ASP.NET Core 1.0
- ASP.NET Core MVC 1.0
- Entity Framework Core 1.0

_Scott Hanselman_ from the _ASP.NET_ team explains in his blog [post](http://www.hanselman.com/blog/ASPNET5IsDeadIntroducingASPNETCore10AndNETCore10.aspx) the motivation behind the name change but in summary the new naming is more aligned with the brand new _.NET Core_ concept as a whole.

Just to note, this is not only a simple change of name, it is going to affect all of the .NET packages available on NuGet, APIs and even the namespaces making it very clear which flavour of .NET you wish to use.

For those of you who are not familiar with .NET Core, it is Microsoft's official (unlike [Mono](http://www.mono-project.com)) cross -platform implementation of the [CLI](https://en.wikipedia.org/wiki/Common_Language_Infrastructure) running on _Windows_, _OSX_ and _Linux_. And here is the good part...it is all open source together with the rest of .NET. Don't believe me? take a [look](https://github.com/dotnet) for yourself.

In short .NET Core can be summarized in the following diagram:

<p style="text-align: center;">
<img alt="diagram of .net core components" src="https://i.imgur.com/lsr7Ta3.png">
<p style="text-align: center;">

So in other words .NET Core consists of:

1. #### CoreFX

   Includes the new _BCL_ i.e. `System.*` things like `System.Collections`, `System.Xml` etc.

2. #### CoreCLR

   The runtime implementation which includes _RyuJIT_, the _GC_, native interop and much more which runs cross-platform.

3. #### CoreRT

   The native runtime which is part of the new _.NET Native_ announced back in [April 2014](http://blogs.msdn.com/b/dotnet/archive/2014/04/02/announcing-net-native-preview.aspx) which is going to compile .NET natively ahead of time.

4. #### CoreCLI
   The Command Line Interface giving a new cross-platform command line experience on _Windows_, _OSX_ and _Linux_ which you can use to build .NET Core applications but it is also capable of building class libraries and _Console_ applications which can run on the full .NET framework.

ASP.NET Core is the brand new framework built from the ground up on top of .NET Core which can also run on the full .NET framework, here's another nice diagram by _Scott_ to make it more clear in the larger .NET universe.

<p style="text-align: center;">
<img alt="diagram of asp.net 4.6 and asp.net core 1.0" src="https://www.hanselman.com/blog/content/binary/Windows-Live-Writer/Reintroducing-ASP.NET-Core-1.0-and-.NE.0_B840/image_0e978596-bd85-42ed-8d27-c16e119bca5d.png">
</p>

I like what I am seeing in this new world of .NET specially the great work that the ASP.NET team has been doing for the past year which includes [ridiculous](https://github.com/aspnet/benchmarks) performance; The future of .NET looks quite promising :-)
