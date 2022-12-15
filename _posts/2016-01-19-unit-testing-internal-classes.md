---
title: "Unit Testing Internal Classes in C#"
tags: "C# .NET Testing"
redirect_from:
  - /testing-internal-classes-in-c-sharp/
  - /2016/01/19/testing-internal-classes-in-c-sharp/
---

## Unit Testing Internal Classes in C#

I recently added a whole bunch of tests to [EasyLogger](https://github.com/NimaAra/Easy.Logger), if you are wondering what _EasyLogger_ is then take a look [HERE]({% post_url 2016-01-01-high-performance-logging-using-log4net %}) to find out why I built it.
I wanted to have at least _80%_ coverage and that meant having to write tests for the [Log4NetLogger.cs](https://github.com/NimaAra/Easy.Logger/blob/master/Easy.Logger/Log4NetLogger.cs) class which happens to be _internal_:

```csharp
internal sealed class Log4NetLogger : ILogger
{
    private readonly ILog _logger;

    [DebuggerStepThrough]
    internal Log4NetLogger(ILog logger)
    {
        _logger = logger;
    }

    [DebuggerStepThrough]
    public void Debug(object message)
    {
        _logger.Debug(message);
    }
    ...
}
```

This meant that in my test method I could not do:

```csharp
Mock<ILog> mockedLogger = new();
Log4NetLogger logger = new(mockedLogger.Object);
```

I intentionally defined this class as an _internal_ because I did not want the user to be able to create an instance of this outside the context of the [Log4netService](https://github.com/NimaAra/Easy.Logger/blob/master/Easy.Logger/Log4NetService.cs) but now how would I test it? hmm....

I could just write an _integration_ or even an _end-to-end_ test to make sure I meet the coverage requirement and call it a day but it just didn't feel right; I still wanted to have _unit_ tests for the class. So after a few minutes of talking to uncle _google_ I found...

### InternalsVisibleToAttribute

As [MSDN](https://msdn.microsoft.com/en-us/library/system.runtime.compilerservices.internalsvisibletoattribute.aspx) explains:

<blockquote>
  Specifies that types that are ordinarily visible only within the current assembly are visible to a specified assembly.
</blockquote>

After putting this guy inside the [AssemblyInfo.cs](https://github.com/NimaAra/Easy.Logger/blob/master/Easy.Logger/Properties/AssemblyInfo.cs), the _internal_ class became visible to the [EasyLogger.Tests.Unit](https://github.com/NimaAra/Easy.Logger/tree/master/Easy.Logger.Tests.Unit) assembly which meant I managed to unit test an _internal_ class in C# for first time! Happy Days! :-)
