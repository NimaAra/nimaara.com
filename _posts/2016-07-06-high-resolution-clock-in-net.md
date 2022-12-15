---
title: "High Resolution Clock in .NET"
tags: "C# .NET Performance"
redirect_from:
  - /high-resolution-clock-in-net/
  - /2016/07/06/high-resolution-clock-in-net/
---

## High Resolution Clock in .NET

Obtaining high resolution time requires a clock which is both _precise_ as well as _accurate_ and those are different things. As _Eric Lippert_ explains:

> A stopped clock is exactly accurate twice a day, a clock a minute slow is never accurate at any time. But the clock a minute slow is always precise to the nearest minute, whereas a stopped clock has no useful precision at all.

So we [know](https://blogs.msdn.microsoft.com/ericlippert/2010/04/08/precision-and-accuracy-of-datetime/) that `DateTime` in .NET is very precise. why? well it is represented as a `struct` which internally keeps track of the number of _ticks_ since a particular start date and those ticks are represented as a 64 bit number; And just for the context, 1 tick equals 100 nanoseconds and every second equals to _10,000,000_ ticks so with such structure one could theoretically represent sub-microsecond times.

Of course having a precise `DateTime` is not all the story let us think about it for a second, I can represent the distance between point A and B as a high-precision floating point by using a `double` for example: _156.34214233234517974_ meters but do I really have a method of measuring that distance to such accuracy? The same goes for `DateTime`; [Hans Passant](http://stackoverflow.com/a/19736972/1226568) explains:

> `DateTime.UtcNow` is accurate to **15.625** milliseconds and stable over very long periods thanks to the time service updates. Going lower than that just doesn't make much sense, you can't get the execution guarantee you need in user mode to take advantage of it.
>
> You need lots of bigger tricks to get it accurate down to a millisecond. You can only get a guarantee like that for code that runs in kernel mode, running at interrupt priority so it cannot get pre-empted by other code and with its code and data pages page-locked so it can't get hit with page faults Commercial solutions use a GPS radio to read the clock signal of the GPS satellites, backed up by an oscillator that runs in an oven to provide temperature stability.

### How accurate is DateTime?

With all of the above in mind and remembering that obtaining a time matching an atomic clock is almost impossible, we can start by measuring the (relative) accuracy of `DateTime.UtcNow` by:

```csharp
var duration = TimeSpan.FromSeconds(5);
var distinctValues = new HashSet<DateTime>();
var stopWatch = Stopwatch.StartNew();

while (stopWatch.Elapsed < duration)
{
    distinctValues.Add(DateTime.UtcNow);
}

Console.WriteLine("Samples: " + distinctValues.Count);
Console.WriteLine($"Accuracy: {stopWatch.Elapsed.TotalMilliseconds / distinctValues.Count:0.000000} ms");
```

he code above is telling us how many times `DateTime.UtcNow` is giving us distinct values in a period of 5 seconds which is the same as showing us how many times its value was updated in that period.

Running the code on my laptop returns:

```shell
Samples: 320
Accuracy: 15.625348 ms
```

And on my server:

```shell
Samples: 4980
Accuracy: 1.005322 ms
```

Both machines are running `Windows Server 2012` but as you saw one is more accurate than the other so it seems the accuracy depends on the hardware as well as software as we will cover shortly.

### Can we get more accurate?

Absolutely! We are going to get some help from `StopWatch` which has a very high resolution and it is going to help us manually tune our `DateTime` in order to achieve a much higher accuracy.

The idea is to start with a base time using `DateTime` and then every time we need to obtain the value of current time we use the `Stopwatch` to supplement our base time. All good? not really!

One slight problem with `Stopwatch` is that after a while (30 minutes) it can get out of sync meaning it can get too far from the system time so in order compensate for that we need to reset our start time every let's say 10 seconds. Sounds complicated? not really, here is the code:

```csharp
public sealed class Clock
{
    private readonly long _maxIdleTime = TimeSpan.FromSeconds(10).Ticks;
    private const long TicksMultiplier = 1000 * TimeSpan.TicksPerMillisecond;

    private DateTime _startTime = DateTime.UtcNow;
    private double _startTimestamp = Stopwatch.GetTimestamp();

    public DateTime UtcNow
    {
        get
        {
            double endTimestamp = Stopwatch.GetTimestamp();

            var durationInTicks = (endTimestamp - _startTimestamp) / Stopwatch.Frequency * TicksMultiplier;
            if (durationInTicks >= _maxIdleTime)
            {
                _startTimestamp = Stopwatch.GetTimestamp();
                _startTime = DateTime.UtcNow;
                return _startTime;
            }

            return _startTime.AddTicks((long)durationInTicks);
        }
    }
}
```

So let us run our test again to measure how accurate our hybrid clock is:

```csharp
var clock = new Clock();
var duration = TimeSpan.FromSeconds(5);
var distinctValues = new HashSet<DateTime>();
var stopWatch = Stopwatch.StartNew();

while (stopWatch.Elapsed < duration)
{
    distinctValues.Add(clock.UtcNow);
}

Console.WriteLine("Samples: " + distinctValues.Count);
Console.WriteLine($"Accuracy: {stopWatch.Elapsed.TotalMilliseconds / distinctValues.Count:0.000000} ms");
```

Which gives us:

```shell
Samples: 11469019
Accuracy: 0.000436 ms
```

Great! so can you go and start using this code? Yes and No! you see this code is not thread-safe so if you are planning on using it in a multi-threaded scenario then continue reading....

### Making it thread-safe

We can easily make our clock thread-safe by using `ThreadLocal`. This guy allows each thread to have its own instance of the `_startTime` and `_startTimestamp` instead of having to resort to a _lock_. So the modified clock now looks like:

```csharp
public sealed class Clock : IDisposable
{
    private readonly long _maxIdleTime = TimeSpan.FromSeconds(10).Ticks;
        private const long TicksMultiplier = 1000 * TimeSpan.TicksPerMillisecond;

    private readonly ThreadLocal<DateTime> _startTime =
        new ThreadLocal<DateTime>(() => DateTime.UtcNow, false);

    private readonly ThreadLocal<double> _startTimestamp =
        new ThreadLocal<double>(() => Stopwatch.GetTimestamp(), false);

    public DateTime UtcNow
    {
        get
        {
            double endTimestamp = Stopwatch.GetTimestamp();

            var durationInTicks = (endTimestamp - _startTimestamp.Value) / Stopwatch.Frequency * TicksMultiplier;
            if (durationInTicks >= _maxIdleTime)
            {
                _startTimestamp.Value = Stopwatch.GetTimestamp();
                _startTime.Value = DateTime.UtcNow;
                return _startTime.Value;
            }

            return _startTime.Value.AddTicks((long)durationInTicks);
        }
    }

    public void Dispose()
    {
        _startTime.Dispose();
        _startTimestamp.Dispose();
    }
}
```

Is that it? well, yes but....!

### Can we do better?

Starting from _Windows 8_ and _Windows Server 2012_, Microsoft has made available for us a very nice API. That API is `GetSystemTimePreciseAsFileTime`. This method obtains the current system time with the highest possible level of precision (we are talking about _<1us_); Why not take advantage of this method in our hybrid clock.

The final version of our clock now looks like:

```csharp
/// <summary>
/// This class provides a high resolution clock by using the new API available in <c>Windows 8</c>/
/// <c>Windows Server 2012</c> and higher. In all other operating systems it returns time by using
/// a manually tuned and compensated <c>DateTime</c> which takes advantage of the high resolution
/// available in <see cref="Stopwatch"/>.
/// </summary>
public sealed class Clock : IDisposable
{
    private readonly long _maxIdleTime = TimeSpan.FromSeconds(10).Ticks;
    private const long TicksMultiplier = 1000 * TimeSpan.TicksPerMillisecond;

    private readonly ThreadLocal<DateTime> _startTime =
        new ThreadLocal<DateTime>(() => DateTime.UtcNow, false);

    private readonly ThreadLocal<double> _startTimestamp =
        new ThreadLocal<double>(() => Stopwatch.GetTimestamp(), false);


    [DllImport("Kernel32.dll", CallingConvention = CallingConvention.Winapi)]
    private static extern void GetSystemTimePreciseAsFileTime(out long filetime);

    /// <summary>
    /// Creates an instance of the <see cref="Clock"/>.
    /// </summary>
    public Clock()
    {
        try
        {
            long preciseTime;
            GetSystemTimePreciseAsFileTime(out preciseTime);
            IsPrecise = true;
        } catch (EntryPointNotFoundException)
        {
            IsPrecise = false;
        }
    }

    /// <summary>
    /// Gets the flag indicating whether the instance of <see cref="Clock"/> provides high resolution time.
    /// <remarks>
    /// <para>
    /// This only returns <c>True</c> on <c>Windows 8</c>/<c>Windows Server 2012</c> and higher.
    /// </para>
    /// </remarks>
    /// </summary>
    public bool IsPrecise { get; }

    /// <summary>
    /// Gets the date and time in <c>UTC</c>.
    /// </summary>
    public DateTime UtcNow
    {
        get
        {
            if (IsPrecise)
            {
                long preciseTime;
                GetSystemTimePreciseAsFileTime(out preciseTime);
                return DateTime.FromFileTimeUtc(preciseTime);
            }

            double endTimestamp = Stopwatch.GetTimestamp();

            var durationInTicks = (endTimestamp - _startTimestamp.Value) / Stopwatch.Frequency * TicksMultiplier;
            if (durationInTicks >= _maxIdleTime)
            {
                _startTimestamp.Value = Stopwatch.GetTimestamp();
                _startTime.Value = DateTime.UtcNow;
                return _startTime.Value;
            }

            return _startTime.Value.AddTicks((long)durationInTicks);
        }
    }

    /// <summary>
    /// Releases all resources used by the instance of <see cref="Clock"/>.
    /// </summary>
    public void Dispose()
    {
        _startTime.Dispose();
        _startTimestamp.Dispose();
    }
}
```

Now we are done!
