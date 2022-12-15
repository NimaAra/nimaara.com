---
title: "High Performance Logging using log4net"
tags: "C# .NET Logging"
redirect_from:
  - /high-performance-logging-log4net/
  - /2016/01/01/high-performance-logging-log4net/
---

## High Performance Logging using log4net

When it comes to logging in _.NET_ all you generally need to do is choose between [log4net](https://logging.apache.org/log4net) or [NLog](http://nlog-project.org) and start logging, however there are times when logging can become a bottleneck which can be a big problem specially if you are dealing with a low-latency component; This was the case with one of our applications.

### The problem

We have a multi-threaded server component which produces a large number of events most of which need to be logged; It also needs to process those events within a specified SLA therefore only very little latency can be tolerated. Recently after adding a bunch of new features the amount of log entries increased by around 60% and that was enough to introduce big latencies which made a lot of people unhappy! :-(

The app uses the [`RollingFileAppender`](https://logging.apache.org/log4net/release/config-examples.html) which persists the log events to the disk synchronously and one by one. After some profiling sessions it was clear to me that the threads producing the log events are waiting for the appender to persist every entry to the disk one after another.

### How bad can it be?

I created a project to be able to track the progress of the solution in different scenarios. [*link at the end*]

The most important scenario to address was the throughput, here we have a single thread (_main_) producing as many log events as it possibly can within a specified duration (_10sec_):

```csharp
...
var sw = Stopwatch.StartNew();
long counter = 0;
while (sw.Elapsed < Duration)
{
    counter++;
    _logger.DebugFormat("Counter is: {0}", counter);
}
...
```

The machine running the benchmark has the following spec:

- CPU 2.2GHz 4 cores
- RAM 8GB
- SSD Read: 282(MB/s) Write: 271(MB/s)

Before running each benchmark the Windows File Cache was cleared using the awesome [RAMMap](https://technet.microsoft.com/en-us/sysinternals/ff700229.aspx) by SysInternals.

Using the `RollingFileAppender`:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<log4net>
  <root>
      <level value="ALL"/>
      <appender-ref ref="RollingFile" />
  </root>

  <appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
    <file type="log4net.Util.PatternString" value="Benchmarker-%date{yyyy-MM-dd}.log" />
    <appendToFile value="false"/>
    <rollingStyle value="Composite"/>
    <maxSizeRollBackups value="-1"/>
    <maximumFileSize value="50MB"/>
    <staticLogFileName value="true"/>
    <datePattern value="yyyy-MM-dd"/>
    <preserveLogFileNameExtension value="true"/>
    <countDirection value="1"/>
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date{ISO8601} [%-5level] [%2thread] %logger{1} - %message%newline%exception"/>
    </layout>
  </appender>

</log4net>
```

results in:

```shell
-------------------Testing Throughput-------------------
Counter reached: 701,017, Time Taken: 00:00:10.0000111
Gen 0: 72
Gen 1: 1
Gen 2: 0
```

As the numbers show the _main_ thread managed to log around **700k** entries and the GC behaves as expected with many short lived objects on the _Gen 0_ most of which are of type `String`, note there is only **1** collection of _Gen 1_ which makes sense as the _main_ thread is blocked until all of the log events it just created are persisted by the appender **One by One** so they do not stay alive long enough to get promoted to _Gen 1_ triggering more collections.

### What can we do?

So it was clear that I needed some mechanism to do both _asynchronous_ and _batch_ logging; If logging to a file was not a requirement and we had a listener, I could simply use the [`UdpAppender`](https://logging.apache.org/log4net/release/config-examples.html) and spend the rest of the day at the bar and maybe worry about occasional packet drops! however I was not that lucky :-(

### Options...

The first option would be to use the [`BufferingForwardingAppender`](https://logging.apache.org/log4net/release/config-examples.html) I like the _log4net_ forwarders in general as they can be very powerful and help do all kind of composition, filtering and logging to different destinations so to try this option all I had to do was to edit the _log4net.config_ to look like:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<log4net>
  <root>
      <level value="ALL"/>
      <appender-ref ref="BufferingForwarder" />
  </root>

  <!--  http://stackoverflow.com/a/11351400/1226568 -->
  <appender name="BufferingForwarder" type="log4net.Appender.BufferingForwardingAppender">
    <bufferSize value="512" />
    <lossy value="false" />
    <Fix value="268" />

    <evaluator type="log4net.Core.LevelEvaluator">
      <threshold value="WARN"/>
    </evaluator>

    <appender-ref ref="RollingFile" />
    <!-- or any additional appenders or other forwarders -->
  </appender>

  <appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
    <file type="log4net.Util.PatternString" value="Benchmarker-%date{yyyy-MM-dd}.log" />
    <appendToFile value="false"/>
    <rollingStyle value="Composite"/>
    <maxSizeRollBackups value="-1"/>
    <maximumFileSize value="50MB"/>
    <staticLogFileName value="true"/>
    <datePattern value="yyyy-MM-dd"/>
    <preserveLogFileNameExtension value="true"/>
    <countDirection value="1"/>
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date{ISO8601} [%-5level] [%2thread] %logger{1} - %message%newline%exception"/>
    </layout>
  </appender>
</log4net>
```

As you may guess from the available config properties of the `BufferingForwader`, log entries are batched into a buffer of size **512**; We also set the _lossy_ flag to be _false_ otherwise the forwarder would only log the last **512** log entries when it sees a log entry with a threshold of _WARN_ as specified by the:

```xml
<evaluator type="log4net.Core.LevelEvaluator">
    <threshold value="WARN"/>
</evaluator>
```

Note we are not using this feature in our benchmarking and we could remove the `<evaluator>` but I am including it here as something you might want to try.

Finally since this forwarder uses batching, the log events will be persisted beyond the normal scope of the _log4net_ internal `Append()` method as opposed to being persisted one by one therefore we need to fix some of the properties of those events that contain volatile data such as _Thread Name(Id)_, _Message_, _Exception_ etc. In fact the full list of the properties that can be fixed is:

```csharp
[Flags]
public enum FixFlags
{
  [Obsolete("Replaced by composite Properties")] Mdc = 1,
  Ndc = 2,
  Message = 4,
  ThreadName = 8,
  LocationInfo = 16,
  UserName = 32,
  Domain = 64,
  Identity = 128,
  Exception = 256,
  Properties = 512,
  None = 0,
  All = 268435455,
  Partial = Properties | Exception | Domain | ThreadName | Message,
}

```

But it is important to note that the more properties you fix the slower the logging will be. I have found that _Partial_ works best in most cases even though we almost never include _Domain_ or _Properties_ in our layout pattern (see [HERE](https://logging.apache.org/log4net/log4net-1.2.13/release/sdk/log4net.Layout.PatternLayout.html) for the full list) but for this test I am fixing _Message_, _ThreadName_ and _Exception_ which is `4 + 8 + 256 = 268`. If you find these numbers ambiguous then you can simply set it to `Message, ThreadName, Exception` and it will be correctly parsed as `268` (another _.NET_ hidden gem!).

Okay, so now that we have the config updated lets run the same benchmarks with the `BufferingForwarder`:

```shell
-------------------Testing Throughput-------------------
Counter reached: 2,218,724, Time Taken: 00:00:10.0015448
Gen 0: 227
Gen 1: 1
Gen 2: 0
```

Not bad! we managed to log around **2.2m** entries with still **1** _Gen 1_ collection; So far so good however there are some subtleties here....

### Problems with BufferingForwardingAppender

The `BufferingForwardingAppender` only flushes its events once the _bufferSize_ is full unless you specify _lossy_ to be `true` so this means if you have an application producing a single log event every **1** second you would have to wait **512** seconds for the batch to be flushed out. This may not be a problem for you and if it is not then I highly recommend using this forwarder as it has great GC and CPU performance but if you need real-time monitoring of every log entry and a higher throughput then continue reading.

Another point we should be aware of is that this forwarder is not asynchronous it merely batches the events so the application thread(s) producing the log events will be blocked while the buffer is being flushed out.

### The solution

Having considered the points above, I thought I could do better so say hello to <strike>my little friend</strike> `AsyncBufferingForwardingAppender`.

This buffering forwarder uses a worker thread which batches up and flushes the log events in the background, it also detects if it has been idle meaning if there has not been enough log events to cause the buffer to be flushed it will invoke a manual flush which addresses the problem I mentioned above and since it is developed as a forwarder you can add additional appenders to it for example it can forward the log events to the _Console_, _DB_ or any other appender that you might have all at the same time.

I let you look at the code to find out how this forwarder works but let's get down to the numbers.

To test this forwarder I updated the _log4net.config_ to be:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<log4net>
  <root>
    <level value="ALL"/>
      <appender-ref ref="AsyncBufferingForwarder"/>
  </root>

  <appender name="AsyncBufferingForwarder" type="Easy.Logger.AsyncBufferingForwardingAppender">
    <lossy value="false" />
    <bufferSize value="512" />
    <Fix value="268" />

    <appender-ref ref="RollingFile"/>
    <!--Any other appender or forwarder...-->
  </appender>

  <appender name="RollingFile" type="log4net.Appender.RollingFileAppender">
    <file type="log4net.Util.PatternString" value="Benchmarker-%date{yyyy-MM-dd}.log" />
    <appendToFile value="false"/>
    <rollingStyle value="Composite"/>
    <maxSizeRollBackups value="-1"/>
    <maximumFileSize value="50MB"/>
    <staticLogFileName value="true"/>
    <datePattern value="yyyy-MM-dd"/>
    <preserveLogFileNameExtension value="true"/>
    <countDirection value="1"/>
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date{ISO8601} [%-5level] [%2thread] %logger{1} - %message%newline%exception"/>
    </layout>
  </appender>
</log4net>
```

And the result is:

```shell
-------------------Testing Throughput-------------------
Counter reached: 3,146,474, Time Taken: 00:00:10.0000011
Gen 0: 503
Gen 1: 172
Gen 2: 0
```

This time we managed to log **3.1m** entries at the cost of **503** _Gen 0_ and **8172** _Gen 1_ collections. The high number of _Gen 1_ is due to the asynchronous logging of this forwarder as the log events are now kept alive long enough to be pushed to _Gen 1_ but not long enough to end up at _Gen 2_ so the good news is that we do not have any _Gen 2_ collections which would be really costly.

### Bonus point

Given our component lives on the server, we could leverage the [`gcServer`](https://msdn.microsoft.com/en-us/library/ms229357%28v=vs.110%29.aspx) to optimize the overall GC across all generations provided you have enough RAM and more than one core otherwise there will be no noticeable gains. To do this we need to update our _app.config_:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <runtime>
      <gcServer enabled="true"/>
  </runtime>
</configuration>
```

This time the benchmark says:

```shell
-------------------Testing Throughput-------------------
Counter reached: 3,379,161, Time Taken: 00:00:10.0000011
Gen 0: 9
Gen 1: 4
Gen 2: 0
```

<p style="text-align: center;"><img width="400" src="https://i.imgur.com/Tvm9aWs.jpg"/></p>

### Summary

In this article we saw how we progressed from the non batching `RollingFileAppender` producing ~**700k** log events to a synchronous batching `BufferingForwardingAppender` producing ~**2.2m** log events we then took this further by using the custom `AsyncBufferingForwardingAppender` to achieve **3.1million** log events in **10s**.

We also looked at the effect of the _gcServer_ to optimize the overall collections across both _Gen 0_ and _Gen 1_ achieving a throughput of ~**3.4m**.

In writing this forwarder my focus was purely on the throughput both in single and multi threaded scenarios. All of this comes at the cost of slightly more CPU usage due to _Gen 1_ collection; So is the extra **1 million** log events worth the extra CPU cost? that is for you to decide but at least we now have the option :-)

Source [HERE](https://github.com/NimaAra/Easy.Logger), NuGet package [HERE](https://www.nuget.org/packages/Easy.Logger).
