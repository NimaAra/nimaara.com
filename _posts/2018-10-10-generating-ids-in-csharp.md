---
title: "Generating IDs in C#, 'safely' and efficiently"
tags: "C# .NET Performance"
redirect_from:
  - /generating-ids-in-csharp/
  - /2018/10/10/generating-ids-in-csharp/
---

### Generating IDs in C#, 'safely' and efficiently

Recently I needed to find an efficient algorithm for generating unique IDs in a highly concurrent and low latency component. After looking at several options I settled for the algorithm used by the [Kestrel HTTP](https://github.com/aspnet/KestrelHttpServer) server. _Kestrel_ generates request IDs that are stored in the `TraceIdentifier` property hanging off the [HTTPContext](https://github.com/aspnet/HttpAbstractions/blob/07d115400e4f8c7a66ba239f230805f03a14ee3d/src/Microsoft.AspNetCore.Http.Abstractions/HttpContext.cs).

The IDs are base-32 encoded increments of a `long` (which is seeded based on the `DateTime.UtcNow.Ticks`) using the characters _1_ to _9_ and _A_ to _V_.

Let's look at the [code](https://github.com/aspnet/KestrelHttpServer/blob/6fde01a825cffc09998d3f8a49464f7fbe40f9c4/src/Kestrel.Core/Internal/Infrastructure/CorrelationIdGenerator.cs#L9):

```csharp
internal static class CorrelationIdGenerator
{
    private static readonly string _encode32Chars = "0123456789ABCDEFGHIJKLMNOPQRSTUV";

    private static long _lastId = DateTime.UtcNow.Ticks;

    public static string GetNextId() => GenerateId(Interlocked.Increment(ref _lastId));

    private static unsafe string GenerateId(long id)
    {
        char* charBuffer = stackalloc char[13];

        charBuffer[0] = _encode32Chars[(int)(id >> 60) & 31];
        charBuffer[1] = _encode32Chars[(int)(id >> 55) & 31];
        charBuffer[2] = _encode32Chars[(int)(id >> 50) & 31];
        charBuffer[3] = _encode32Chars[(int)(id >> 45) & 31];
        charBuffer[4] = _encode32Chars[(int)(id >> 40) & 31];
        charBuffer[5] = _encode32Chars[(int)(id >> 35) & 31];
        charBuffer[6] = _encode32Chars[(int)(id >> 30) & 31];
        charBuffer[7] = _encode32Chars[(int)(id >> 25) & 31];
        charBuffer[8] = _encode32Chars[(int)(id >> 20) & 31];
        charBuffer[9] = _encode32Chars[(int)(id >> 15) & 31];
        charBuffer[10] = _encode32Chars[(int)(id >> 10) & 31];
        charBuffer[11] = _encode32Chars[(int)(id >> 5) & 31];
        charBuffer[12] = _encode32Chars[(int)id & 31];

        return new string(charBuffer, 0, 13);
    }
}
```

If we invoke it like so:

```csharp
CorrelationIdGenerator.GetNextId();
Thread.Sleep(50);
CorrelationIdGenerator.GetNextId();
Thread.Sleep(150);
CorrelationIdGenerator.GetNextId();
```

We get:

```shell
0HLH7QN5JTC87
0HLH7QN5JTC88
0HLH7QN5JTC89
```

Overall, this code is very efficient in how it constructs a `string` representation of an ID based on a `long` by avoiding the `.ToString()` call which (according to the [comments on the code](https://github.com/aspnet/KestrelHttpServer/blob/6fde01a825cffc09998d3f8a49464f7fbe40f9c4/src/Kestrel.Core/Internal/Infrastructure/CorrelationIdGenerator.cs#L23)) would be **_~310%_** and **_~600%_** slower on _x64_ and _x86_ respectively.

It also avoids allocating memory on the _heap_ avoiding frequent _GC_ collections (when invoked on a hot path) by allocating a `char[]` on the _stack_, populating it with the encoded values then passing the `char` pointer to the `string` constructor.

### Impressive stuff, but...

Allocating memory on the _stack_ (using `stackalloc`) has long been used as a performance trick; Not only it can be faster than _heap_ allocation but it also avoids _GC_. However, there are some caveats:

- `stackalloc` is explicitly unsafe (hence why I used `'safe'` in the title of this article) which means you would have to compile your assembly as such (by using the `unsafe` flag). Doing so means that your assembly may be black-listed by some environments where loading of `unsafe` assemblies are not permitted. e.g. _SQLServer_ ([although there are workarounds](https://docs.microsoft.com/en-us/sql/relational-databases/clr-integration/assemblies/creating-an-assembly?view=sql-server-2017)). (Also note that as of _C#7.2_ `unsafe` is no longer required as long as you use `Span` when using `stackalloc`)

- The amount of data you are allowed to allocate is very limited. Managed stacks are limited to **_1MB_** in size and can be even less when running under _IIS_. Each stack frame takes a chunk of that space so the deeper the _stack_ the less memory will be available.

- If an _exception_ is thrown during allocation, you will get an `StackOverflowException` which is best known as an exception that cannot be caught. To be fair, _Kestrel_ allocates a small block of memory when generating an ID, therefore, the chances of overflowing is rare; However, I still prefer to be on the _safe_ side if and when I can.

With the above points in mind, I got curious to see if I can avoid using `unsafe` and:

1. Measure the performance impact of going without it.
1. Try close the gap in the performance as much as possible and bring it to a reasonable level.

### A bit about how we are going to measure the performance

If you have not yet read my previous articles I suggest having a look so that you can get familiar with some of the terms, tools and _my_ overall approach to profiling.

- [Practical Parallelization in C#]({% post_url 2017-06-01-practical-parallelization-in-csharp %})
- [High Performance Logging using log4net]({% post_url 2016-01-01-high-performance-logging-using-log4net %})
- [Counting Lines of a Text File in C#, the Smart Way]({% post_url 2018-03-20-counting-lines-of-a-text-file-in-csharp %})
- [Stuff Every .NET App Should be Logging at Startup]({% post_url 2017-11-07-stuff-every-dotnet-app-should-log %})

We are going to start by doing some _coarse-grained/high-level_ measurements just so that we get a general idea of the behaviour of _GC_ and the running time for our baseline algorithm before we start playing with things.

We will be running a _.NET Core 2.1.5_ application hosted on _Windows Server 2016_. Our machine is a _dual CPU Xeon processor_ with _12_ cores (_24_ logical) clocking at _2.67GHz_.

Also note the _GC_ mode of our application will be set to `Workstation` to emphasise the impact of the _GC_ collections on the process (refer to the links above for finding out the difference between `Workstation` and `Server`). The overall result of our benchmark should still be valid with `Server` mode enabled.

Let's start by generating **_100 million_** IDs while measuring the execution time and the _GC_ behaviour. We will repeat this **_10 times_** and look at the average values across all executions.

Here is the code for generating the IDs:

```csharp
Stopwatch sw = Stopwatch.StartNew();
for(int i = 0; i < 100_000_000; i++)
{
    string id = CorrelationIdGenerator.GetNextId();
}
sw.Stop();

using Process process = Process.GetCurrentProcess();

Console.WriteLine("Execution time: {0}\r\n  - Gen-0: {1}, Gen-1: {2}, Gen-2: {3}",
        sw.Elapsed.ToString(),
        GC.CollectionCount(0),
        GC.CollectionCount(1),
        GC.CollectionCount(2));
```

This _not-so-fancy_ code starts a timer, generates a lot of IDs then measures how many times _GC_ collections occurred across the _3_ generations.

### 1. Stackalloc

With all the above out of the way, here's the average values for our baseline:

```shell
- Execution time: 00:00:06.0511525
- Gen-0: 890, Gen-1: 0, Gen-2: 0
```

Okay, so it took around **_6 seconds_** and **_890_** _Gen0_ _GC_ to generate the IDs. If you are wondering why we have those _Gen0_ collections despite allocating on the _stack_, well, the answer is; We are only allocating the `char[]` on the _stack_ but we still need to allocate **_100 million_** `string` objects on the _heap_.

Now that we know how our baseline behaves, let us measure how we would have done if we had allocated the `char[]` on the _heap_ instead of the _stack_.

### 2. Heap allocation

All we have to do for this version is remove the `unsafe` keyword and new up a `char[]`, with everything else unchanged.

```csharp
private static string GenerateId(long id)
{
    var buffer = new char[13];

    buffer[0] = _encode32Chars[(int)(id >> 60) & 31];
    buffer[1] = _encode32Chars[(int)(id >> 55) & 31];
    buffer[2] = _encode32Chars[(int)(id >> 50) & 31];
    buffer[3] = _encode32Chars[(int)(id >> 45) & 31];
    buffer[4] = _encode32Chars[(int)(id >> 40) & 31];
    buffer[5] = _encode32Chars[(int)(id >> 35) & 31];
    buffer[6] = _encode32Chars[(int)(id >> 30) & 31];
    buffer[7] = _encode32Chars[(int)(id >> 25) & 31];
    buffer[8] = _encode32Chars[(int)(id >> 20) & 31];
    buffer[9] = _encode32Chars[(int)(id >> 15) & 31];
    buffer[10] = _encode32Chars[(int)(id >> 10) & 31];
    buffer[11] = _encode32Chars[(int)(id >> 5) & 31];
    buffer[12] = _encode32Chars[(int)id & 31];

    return new string(buffer, 0, 13);
}
```

Right, let us measure this version:

```shell
- Execution time: 00:00:06.1883507
- Gen-0: 1780, Gen-1: 0, Gen-2: 0
```

Allocating on the _heap_ seems to be about **_140 milliseconds_** slower than allocating on the _stack_; However, we have doubled the number of \_GC_s. You may now be saying:

> Dude, it doesn't matter. Gen0 GCs are super fast...

And you would be right; Both _Gen0_ and _Gen1_ collections are fast but unlike the _Gen2_ collections, they are _blocking_; So, the more IDs you generate, the more frequent blocking collections wasting CPU cycles reducing your overall throughput. This may not be a problem for your application and if it is not then fair enough but there are scenarios where you need to reduce allocations as much as possible.

Okay, what is the most simple method of reducing those allocations? Reusing the `char[]` you say? Sure, let's try that...

### 3. Re-using the buffer

What we need to do here is declare a field for our re-usable `char[]` buffer and overwrite it for every ID we generate.

```csharp
private static readonly char[] _buffer = new char[13];

private static string GenerateId(long id)
{
    char[] buffer = _buffer;
    ...
}
```

And this is how it does:

```shell
- Execution time: 00:00:05.5621535
- Gen-0: 890, Gen-1: 0, Gen-2: 0
```

<p style="text-align: center;">
    <img alt="picture of a meme showing 2 colleagues saying how easy their task was" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.that.was.easy.gif" | relative_url}}" width="400">    
</p>

Excellent! Awesome gains! Super fast! No allocation! Let's wrap up!

### But wait a second

The `unsafe` version was _thread-safe_ and our version is not. What if we want to generate the IDs from different threads and in parallel?

No worries, all we have to do is ensure access to the `char[]` is _synchronised_ across all threads and for that we are going to use a simple `lock`, let's go!

### 4. Locking the buffer

With the rest of the code unchanged, Let us `lock` the body of our method like so:

```csharp
private static string GenerateId(long id)
{
    lock(_buffer)
    {
        char[] buffer = _buffer;
        ...
    }
}
```

And run the benchmark:

```shell
- Execution time: 00:00:07.7254997
- Gen-0: 890, Gen-1: 0, Gen-2: 0
```

Okay, no more allocation other than the actual `string` representation of our IDs. As you can see the cost of using the `lock` is an **_extra second_** or so of overhead.

### But this is not a valid test

Even though we now have a _thread-safe_ method, we have so far only generated the IDs from a single thread and that is not representative of how our code can and will be used out in the wild.

### A more realistic scenario

We are going to sprinkle a bit of parallelism to our benchmarking code by generating **_1 billion_** IDs across **_12_** threads in parallel.

Here is the updated code for our benchmarker:

```csharp
const int ITERATION_COUNT = 1_000_000_000;
const int THREAD_COUNT = 12;

Stopwatch sw = Stopwatch.StartNew();

ParallelEnumerable
    .Range(1, ITERATION_COUNT)
    .WithDegreeOfParallelism(THREAD_COUNT)
    .WithExecutionMode(ParallelExecutionMode.ForceParallelism)
    .ForAll(_ => {
        string id = CorrelationIdGenerator.GetNextId();
    });

sw.Stop();

using Process process = Process.GetCurrentProcess();
Console.WriteLine("Execution time: {0}\r\n  - Gen-0: {1}, Gen-1: {2}, Gen-2: {3}",
        sw.Elapsed.ToString(),
        GC.CollectionCount(0),
        GC.CollectionCount(1),
        GC.CollectionCount(2));
```

### 5. Stackalloc (Parallel)

Now that we have updated our benchmarker, we need to run it again for our baseline to see how it does in parallel.

```shell
- Execution time: 00:00:58.9092805
- Gen-0: 8980, Gen-1: 5, Gen-2: 0
```

Okay, so **_8,980_** _Gen0_, **_5_** _Gen1_ and **_00:00:58.9092805_** is what we are up against.

### 6. New buffer (Parallel)

As a reminder, this is the version where we are creating a new `char[]` each time we are generating an ID. So let us see the performance of this _allocaty_ method in parallel:

```shell
- Execution time: 00:01:04.6934614
- Gen-0: 17959, Gen-1: 19, Gen-2: 0
```

Again, we can see the high number of allocations except this time the _Gen1_ allocations are also increasing.

### 7. Locking the buffer (Parallel)

Not to worry, we have it all under control. We will be avoiding all those nasty allocations thanks to our friend Mr `lock`! Let's go:

```shell
- Execution time: 00:04:14.3642255
- Gen-0: 9008, Gen-1: 25, Gen-2: 2
```

<p style="text-align: center;">
    <img alt="picture of a meme showing an old woman adjusting her glasses while looking at a laptop screen" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.what.did.I.just.read.jpg" | relative_url}}" width="400">    
</p>

Yep, looks like Mr `lock` was not much of a friend after all! not only it took us **_4 times_** longer but also **_25_** _Gen1_ and **_2_** _Gen2_ collections. But why? Isn't acquiring a lock fast? Well, that depends; Acquiring a _non-contended_ lock is extremely fast (_~20 ns_) but if contended, the context switch depending on the number of competing threads can quickly increase this to microseconds which in our case resulted in a noticeable performance hit.

Indeed, if we profile this app using [dotTrace](https://www.jetbrains.com/profiler/) (refer to my previous blog posts for more details) we can see that we spent a whopping **_75%_** of our total execution time _waiting_ in _Lock contention_ (click to enlarge):

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.1.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.1.png" | relative_url}}" width="700">
    </a>
</p>

Those _12_ threads are barely utilising the CPU. Let's compare that with our `stackalloc` version:

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.2.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.2.png" | relative_url}}" width="700">
    </a>
</p>

See how nice and healthy those threads are? Those context switches are wasting the CPU cycles. If only we could somehow reduce it...

### 8. Spin Locking

Guess what? We can avoid those context switches by using a [SpinLock](https://docs.microsoft.com/en-us/dotnet/api/system.threading.spinlock?view=netframework-4.7.2).

> SpinLock lets you lock without incurring the cost of a context switch, at the expense of keeping a thread spinning (uselessly busy) - [Albahari](http://www.albahari.com/threading/part5.aspx#_SpinLock_and_SpinWait)

And this is how we are going to use it in our code:

```csharp
private static readonly char[] _buffer = new char[13];
private static SpinLock _spinner = new SpinLock(false);

private static string GenerateId(long id)
{
    bool lockTaken = false;
    try
    {
        _spinner.Enter(ref lockTaken);

        char[] buffer = _buffer;
        ...
        return new string(buffer, 0, 13);
    } finally
    {
        if (lockTaken)
        {
            _spinner.Exit();
        }
    }
}
```

### `<warning>`IMPORTANT`</warning>`

You should almost never use a `SpinLock` as it comes with many gotchas one of which is that unlike a `lock` (aka [Monitor](https://docs.microsoft.com/en-us/dotnet/api/system.threading.monitor)) a `SpinLock` is not _re-entrant_; So, if you cannot figure out why I have set its constructor parameter to `false` or why the `_spinner` field has not been declared as `readonly`, then it means you should not be using `SpinLock` until you fully understand how it works (I still do not!) and once you do, you should totally erase any memory of `SpinLock` from your head. Seriously, DO NOT USE IT! Nevertheless, you can have a look at [THIS](https://stackoverflow.com/questions/11224130/spinlock-throwing-synchronizationlockexception) and [THIS](https://stackoverflow.com/questions/5869825/when-should-one-use-a-spinlock-instead-of-mutex) to satisfy your curiosity.

Let's see what difference `SpinLock` makes:

```shell
- Execution time: 00:01:34.5641002
- Gen-0: 8962, Gen-1: 9, Gen-2: 0
```

It looks like this time we did much better by not having to do as many context switches as we did when we were locking; But let us have a closer look at what our process was actually doing:

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.3.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.3.png" | relative_url}}" width="700">
    </a>
</p>

Even though we had no contentions, we spent only **_13.5%_** of our total execution time running and the rest spinning and wasting precious CPU time. **NOT GOOD**!

### The Evil Shared State

Sharing the `char[]` across our worker threads does not seem to be a good choice after all. How can we eliminate this shared resource and at the same time avoid allocating? If only there was a way for each thread to have its own buffer to work with...

Well, .NET has a facility which allows us to do just that.

### 9. ThreadStatic

Any _static field_ marked with a [ThreadStatic](https://docs.microsoft.com/en-us/dotnet/api/system.threadstaticattribute) attribute is attached to a thread and each thread will then have its own copy of it. You can think of it as a container which holds a separate instance of our field for every thread.

> A _static field_ marked with `ThreadStaticAttribute` is not shared between threads. Each executing thread has a separate instance of the field, and independently sets and gets values for that field. If the field is accessed on a different thread, it will contain a different value. [MSDN](https://docs.microsoft.com/en-us/dotnet/api/system.threadstaticattribute)

When using `ThreadStatic`, we are not required to specify an initial value for our field; if we do, then the initialisation occurs once the static constructor of the class is executed, therefore, the value is only set for the thread executing that constructor; When accessed in all subsequent threads, it will be initialised to its default value. So, in our case, before each thread tries to use the buffer for the first time, we need to make sure the field for that thread is initialised to a new `char[]` otherwise its value will be `null`.

This is what our code looks like with `ThreadStatic`:

```csharp
[ThreadStatic]
private static char[] _buffer;

private static string GenerateId(long id)
{
    if (_buffer is null)
    {
        _buffer = new char[13];
    }

    char[] buffer = _buffer;
    ...
}
```

Let's see how this one does:

```shell
- Execution time: 00:01:11.0138624
- Gen-0: 8981, Gen-1: 7, Gen-2: 0
```

Okay, we are getting there! We reduced the execution time by around **_20 seconds_**. Let us see how our threads performed:

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.4.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.4.png" | relative_url}}" width="700">
    </a>
</p>

Now we are talking! As you can see from the image, all our threads are showing a healthy utilisation and overall spent **_94.6%_** of their time _running_. This is what we wanted as eliminating that shared buffer also removes any contention between our worker threads enabling them to each do their work independently with no bottleneck.

Okay good, but can we do better? Just a little bit faster?

### 10. ThreadLocal

Starting from _.NET4.0_, [ThreadLocal](https://docs.microsoft.com/en-us/dotnet/api/system.threading.threadlocal-1) provides _Thread Local Storage_ (_TLS_) of data. It also improves on `ThreadStatic` by providing a strongly typed, locally scoped container to store values specific to each thread. Unlike `ThreadStatic`, you can mark both _instance_ and _static fields_ with it. It also allows you to use a factory method to create or initialise its value _lazily_ and on the first access from each thread.

#### Side note

I have used this `ThreadLocal` trick many times in the past including in [Easy.MessageHub](https://github.com/NimaAra/Easy.MessageHub) (a high-performance _Event Aggregator_ which you can read about [HERE]({% post_url 2017-11-07-stuff-every-dotnet-app-should-log %})) to avoid locking and remove contention between threads.

Armed with our new friend, I present to you the final version of our code:

```csharp
private static readonly ThreadLocal<char[]> _buffer =
    new ThreadLocal<char[]>(() => new char[13]);

private static string GenerateId(long id)
{
    char[] buffer = _buffer.Value;

    buffer[0] = Encode_32_Chars[(int)(id >> 60) & 31];
    buffer[1] = Encode_32_Chars[(int)(id >> 55) & 31];
    buffer[2] = Encode_32_Chars[(int)(id >> 50) & 31];
    buffer[3] = Encode_32_Chars[(int)(id >> 45) & 31];
    buffer[4] = Encode_32_Chars[(int)(id >> 40) & 31];
    buffer[5] = Encode_32_Chars[(int)(id >> 35) & 31];
    buffer[6] = Encode_32_Chars[(int)(id >> 30) & 31];
    buffer[7] = Encode_32_Chars[(int)(id >> 25) & 31];
    buffer[8] = Encode_32_Chars[(int)(id >> 20) & 31];
    buffer[9] = Encode_32_Chars[(int)(id >> 15) & 31];
    buffer[10] = Encode_32_Chars[(int)(id >> 10) & 31];
    buffer[11] = Encode_32_Chars[(int)(id >> 5) & 31];
    buffer[12] = Encode_32_Chars[(int)id & 31];

    return new string(buffer, 0, buffer.Length);
}
```

Let's take it for a spin:

```shell
- Execution time: 00:00:58.7741476
- Gen-0: 8980, Gen-1: 5, Gen-2: 0
```

And just like that, we managed to beat the `stackalloc` version by **_135ms_**!

<p style="text-align: center;">
    <img alt="picture of a meme showing a group of people clapping" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.clapping.gif" | relative_url}}" width="400">    
</p>

Here's the profiling snapshot where once again we can see a clean and healthy utilisation across all our threads.

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.5.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.5.png" | relative_url}}" width="700">
    </a>
</p>

### Bonus point

Shortly after publishing this article, [Ben Adams](https://twitter.com/ben_a_adams) the author of the original [pull request](https://github.com/aspnet/Hosting/pull/385) for _Kestrel_'s ID generation reminded me of the new [`String.Create`](https://msdn.microsoft.com/en-us/magazine/mt814808.aspx?f=255&MSPPError=-2147217396) method which allows us to avoid both stack and heap allocation for our buffer and write directly to the `string` memory.

You will need to be on _.NET Core 2.1_ or higher and know the length of your `string` in advance. The method then allocates a `string` and gives you a `Span<char>` allowing you to essentially write directly into the `string`'s memory on the _heap_.

This is how that version would look like:

```csharp
public static string GenerateId(long id) =>
    string.Create(13, id, _writeToStringMemory);

private static readonly SpanAction<char, long> _writeToStringMemory =
    // DO NOT convert to method group otherwise will allocate
    (span, id) => WriteToStringMemory(span, id);

private static void WriteToStringMemory(Span<char> span, long id)
{
    span[0] = Encode_32_Chars[(int) (id >> 60) & 31];
    span[1] = Encode_32_Chars[(int) (id >> 55) & 31];
    span[2] = Encode_32_Chars[(int) (id >> 50) & 31];
    span[3] = Encode_32_Chars[(int) (id >> 45) & 31];
    span[4] = Encode_32_Chars[(int) (id >> 40) & 31];
    span[5] = Encode_32_Chars[(int) (id >> 35) & 31];
    span[6] = Encode_32_Chars[(int) (id >> 30) & 31];
    span[7] = Encode_32_Chars[(int) (id >> 25) & 31];
    span[8] = Encode_32_Chars[(int) (id >> 20) & 31];
    span[9] = Encode_32_Chars[(int) (id >> 15) & 31];
    span[10] = Encode_32_Chars[(int) (id >> 10) & 31];
    span[11] = Encode_32_Chars[(int) (id >> 5) & 31];
    span[12] = Encode_32_Chars[(int) id & 31];
}
```

Ready to see the result?

```shell
- Execution time: 00:01:00.1041126
- Gen-0: 8980, Gen-1: 6, Gen-2: 0
```

And here's how the process was doing:

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.6.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2018-10-10-generating-ids-in-csharp.profile.result.6.png" | relative_url}}" width="700">
    </a>
</p>

Hmm... this is not what I expected. Could it be that we are spending a little too much time [Bounds Checking](https://en.wikipedia.org/wiki/Bounds_checking)? Let's give the _JIT_ a little help by reversing the buffer population:

```csharp
private static void WriteToStringMemory(Span<char> span, long id)
{
    span[12] = Encode_32_Chars[(int) id & 31];
    ...
    span[0] = Encode_32_Chars[(int) (id >> 60) & 31];
}
```

And how about now?

```shell
- Execution time: 00:00:56.6800000
- Gen-0: 8980, Gen-1: 5, Gen-2: 0
```

Excellent!

### Conclusion

As we discussed before, our focus was to avoid using `unsafe` whilst trying to minimise any potential drop in the performance. The goal was not to be faster than the baseline but as it turned out we managed to shave a little bit off which is also nice.

I have included the final version of this method ([`IDGenerator`](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/IDGenerator.cs)) in [Easy.Common](https://github.com/NimaAra/Easy.Common). You can also find the benchmark code [HERE](https://github.com/NimaAra/Blog.Samples/tree/master/IDGenerator).

I hope you enjoyed this article. Happy `ThreadLocal`ing all the things!
