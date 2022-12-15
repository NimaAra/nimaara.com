---
title: "Practical Parallelization in C# with MapReduce, ProducerConsumer and ActorModel"
tags: "C# .NET Performance Threading"
redirect_from:
  - /practical-parallelization-with-map-reduce-in-c/
  - /2017/07/01/practical-parallelization-with-map-reduce-in-c/
---

## Practical Parallelization in C# with MapReduce, ProducerConsumer and ActorModel

The barrier of entry into multi-threading in .NET is relatively low as both _Parallel Computing_ (making programs run faster) and _Concurrent Programming_ (making programs more responsive) have been greatly simplified since the introduction of [TPL](<https://msdn.microsoft.com/en-us/library/dd537609(v=vs.110).aspx>) and its friends ([Parallel](<https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx>) and [PLINQ](<https://msdn.microsoft.com/en-us/library/dd460688(v=vs.110).aspx>)) in _.NET4.0_.

Despite this low barrier of entry, many developers fail to achieve the best performance out of their multi-threaded solutions. This is sometimes due to frequent or unnecessary _locking/synchronization_ or at other times due to incorrectly partitioning their data or using an inappropriate pattern.

In this article we are going to look at how we can squeeze the best performance out of an easily parallelizable problem by rewriting the same basic implementation using some of the most widely used multi-threading tools available in .NET. We will be covering different .NET technologies such as [PLINQ](<https://msdn.microsoft.com/en-us/library/dd460688(v=vs.110).aspx>), [BlockingCollection](<https://msdn.microsoft.com/en-us/library/dd997371(v=vs.110).aspx>), [Parallel](<https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx>) class as well as [TPL Dataflow](<https://msdn.microsoft.com/en-us/library/hh228603(v=vs.110).aspx>) in conjunction with patterns such as [MapReduce](https://en.wikipedia.org/wiki/MapReduce), [ActorModel](https://en.wikipedia.org/wiki/Actor_model) as well as [ProducerConsumer](https://en.wikipedia.org/wiki/Producer%E2%80%93consumer_problem) while trying to achieve _optimal parallelization_ and _minimize the cost of synchronization_ between the threads.

### The Scenario

The example scenario we will be solving is to count the number of words in a text file which is the [IMDB](http://www.imdb.com/) database for movie plots downloaded from [HERE](ftp://ftp.fu-berlin.de/pub/misc/movies/database/frozendata/plot.list.gz). The file (as of today) is **432MB** with **8,151,603** lines consisting of **69,275,414** words and **432,045,742** characters. So our job is to figure out what the top n words and their respective counts are in this file. The overall work required to calculate the result can be summarized as:

1. Read every line in the file
2. Convert each line into words
3. Exclude invalid words e.g. _"a"_, _"and"_, _"with"_ etc.
4. Count and track the occurrence of each word
5. Sort the words from the highest count to the lowest
6. Return the top n words e.g. _100_

Our final result should display something like this:

```shell
life   128,714
new    119,587
man    96,257
time   89,828
world  88,863
young  87,279
film   82,800
love   80,002
family 75,530
home   74,265
story  73,211
old    62,433
day    62,319
finds  61,394
father 60,055
...
```

### Approach

We will be measuring the following for each of the patterns we implement:

1. The total execution time of the program
2. The count of times _Garbage Collection_ occurs across different generations i.e. _0_, _1_ and _2_
3. The process's _Peak Working Set_ memory which is the maximum amount of physical memory (RAM) used by the process.

In addition to the above, for each implementation I will also include a _Time-Line_ profiling snapshot of the process using the awesome [dotTrace](https://www.jetbrains.com/profiler/) profiler developed by [JetBrains](https://www.jetbrains.com/) (available as a 10 day trial). This allows us to analyze the context under which our thread(s) were executing and the level of concurrency and CPU utilization we achieved for that implementation.

You can use other tools and profilers such as the [Visual Studio Performance Profiling](https://msdn.microsoft.com/en-us/library/ms182372.aspx), [ANTS Performance Profiler](http://www.red-gate.com/products/dotnet-development/ants-performance-profiler) by [RedGate](http://www.red-gate.com/) or the [Concurrency Visualizer](https://msdn.microsoft.com/en-us/library/dd537632.aspx) extension for _Visual Studio_. There is also a free new comer built using C# and WPF called [CodeTrack](http://www.getcodetrack.com/); It looks quite promising and I highly suggest taking a look and having a play.

In order to increase our application's throughput, we will set the _GC_ mode to [Server](<https://msdn.microsoft.com/en-us/library/ms229357(v=vs.110).aspx>). A _Server GC_ mode results in less frequent _GC_ across all generations but it also means that each CPU core will now have its own heap as well as a dedicated thread responsible for collecting every dead references in that heap so on my 24-core server we will have 24 threads dedicated to GC but that is not all, thanks to the [Background Server GC](https://blogs.msdn.microsoft.com/dotnet/2012/07/20/the-net-framework-4-5-includes-new-garbage-collector-enhancements-for-client-and-server-apps/) which was introduced in _.NET4.5_ the number of threads can increase even more! This feature is used to offload a significant subset of the _Gen-2_ _GC_ by using dedicated background threads resulting in a significant reduction of the total pause times on the user threads but remember, this only applies to _Gen-2_ as both _Gen-0_ and _Gen-1_ collections still require all user threads to be suspended.

Also remember that under _Server GC_, heap sizes are much larger compared to the [Workstation GC](https://docs.microsoft.com/en-us/dotnet/standard/garbage-collection/fundamentals#workstation_and_server_garbage_collection) so don't be alarmed if you see large memory usage in our implementations.

#### Note:

In order to keep the focus of this article on the concurrency costs of each implementation rather than the behavior of _GC_, I have excluded all of the GC threads from our profiling results. This includes the _Finalizer_ as well as the _background GC_ threads. But to investigate how the _GC_ performed for each of the algorithms, you can include all those threads in the snapshot file provided for each implementation (by installing & running _dotTrace_).

I will be running the application on a **24-core** dual CPU workstation with **24GB** of RAM and the input file is being read from a **7200 RPM** HDD with **64MB** of cache. Each pattern will be profiled individually and the _Windows File Cache_ will be cleared between each run.

Finally, You can find the code [HERE](https://github.com/NimaAra/Blog.Samples/tree/master/Practical%20Parallelization). Okay, now that we have all those covered, let's get started!

### 1. Sequential

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    Dictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);

    // process lines as IEnumerable<string> read one after another from the disk.
    foreach (var line in File.ReadLines(InputFile.FullName))
    {
        foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        {
            if (!IsValidWord(word)) { continue; }
            TrackWordsOccurrence(result, word);
        }
    }

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}

private static void TrackWordsOccurrence(IDictionary<string, uint> wordCounts, string word)
{
    if (wordCounts.TryGetValue(word, out uint count))
    {
        wordCounts[word] = count + 1;
    }
    else
    {
        wordCounts[word] = 1;
    }
}

private static bool IsValidWord(string word) =>
    !InvalidCharacters.Contains(word[0]) && !StopWords.Contains(word);
```

There is nothing complicated about this code, all we are doing here is reading a bunch of lines splitting them into words, doing some filtering and keeping track of each word in a `Dictionary<string, uint>`. We then order the result by the count of each word in descending order and return n entries from the result.

Running the above produces the following:

```shell
Execution time: 87,025 ms
Gen-0: 27, Gen-1: 10, Gen-2: 4
Peak WrkSet: 617,037,824
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-1.sequential.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-1.sequential.png" | relative_url}}" width="700">
    </a>
</p>

As we can see, the program achieved an average CPU utilization of **_4.0%_**, took **_87,025_** milliseconds out of which **_0.6%_** was spent on _GC_. **_91.3%_** of the _main_ thread time was in the _Running_ state and **_8.7%_** in _Waiting_. The process reached a peak memory of **_~600MB_**. Since this implementation is a straight-forward single-threaded execution, there was no locking involved therefore no cost associated to locking-contention.

### 2. Sequential (LINQ)

Next, the same sequential algorithm implemented using _LINQ_:

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    return File.ReadLines(InputFile.FullName)
        .SelectMany(l => l.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        .Where(IsValidWord)
        .ToLookup(x => x, StringComparer.InvariantCultureIgnoreCase)
        .Select(x => new { Word = x.Key, Count = (uint)x.Count() })
        .OrderByDescending(kv => kv.Count)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Word, kv => kv.Count);
}
```

Here we are projecting each line into a sequence of words, filter out the invalid words and then group the words using `ToLookup` (or `GroupBy`) to get their count; finally we order them and take the top n items.

Here's the result:

```shell
Execution time: 92,174 ms
Gen-0: 6, Gen-1: 4, Gen-2: 3
Peak WrkSet: 4,579,205,120
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-2.sequential.linq.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-2.sequential.linq.png" | relative_url}}" width="700">
    </a>
</p>

This time, the program achieved an average CPU utilization of **_7.6%_**, took **_92,174_** milliseconds out of which **8.7%** was spent on _GC_. **_82.5%_** of all our threads time were in the _Running_ state and **_17.5%_** in _Waiting_. Exploring the snapshot file reveals that around **_40%_** was used doing LINQ operations so there is no surprise to see the inferior performance (due to LINQ's overhead). Just as above, there is no locking involved. Also note our process resulted in a whopping **_4.6GB_** of peak memory.

### 3. PLINQ (Naive)

As the title suggests here we will build a hybrid of _PLINQ_ and a single-threaded solution. The code is similar to the sequential _LINQ_ with the addition of `.AsParallel()` which does all the magic!

The work relating to reading each line, projecting the words and filtering will be distributed across all the available system cores and the work relating to keeping track of each of the word's occurrence is done on the _main_ thread. In other words the main thread in our `foreach` loop _pulls_ each line from the disk all the way through the _PLINQ_ parallelized pipeline.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    var words = File.ReadLines(InputFile.FullName)
        .AsParallel()
        .SelectMany(l => l.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        .Where(IsValidWord);

    Dictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);
    foreach (var word in words)
    {
        TrackWordsOccurrence(result, word);
    }

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

Running the above results in the following:

```shell
Execution time: 80,620 ms
Gen-0: 10, Gen-1: 7, Gen-2: 4
Peak WrkSet: 1,677,287,424
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-3.plinq.naive.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-3.plinq.naive.png" | relative_url}}" width="700">
    </a>
</p>

This time, the program achieved an average CPU utilization of **_17.5%_**, took **_80,620_** milliseconds out of which **_1.0%_** was spent on _GC_. **_17%_** of all our threads time were in the _Running_ state and **_83%_** in _Waiting_. Exploring the snapshot file reveals that around **_18%_** was spent doing _LINQ_ operations there is also very little locking contention (**_0.07%_**) since the main part of the app which requires synchronization is actually running on the main thread. Also note our process resulted in a **_1.7GB_** of peak memory.

Okay, so not much of an improvement but be ready as things are about to change!

#### A bit about Partitioning

Before continuing any further I feel I should mention an important aspect of _PLINQ_ which is responsible for splitting and assigning each input element to a worker thread. This is known as _Partitioning_ and there are currently four different partitioning strategies in _TPL_ all of which are supported by _PLINQ_; These are:

1. Chunk partitioning
1. Range partitioning
1. Hash partitioning
1. Striped partitioning

**Chunk partitioning** uses _dynamic_ element allocation as opposed to _static_ element allocation (which is used by both _Range_ and _Hash_ partitioning). _Dynamic_ allocation allows partitions to be created on the fly as opposed to being allocated in advance which means that a _dynamic_ partitioner needs to request elements more frequently from the input sequence resulting in more synchronization; In contrast, _static_ partitioning allows elements to be assigned all at once which results in less overhead and no additional synchronization after the initial work required for creating the partitions. Note that `Parallel.ForEach` only supports _dynamic_ partitioning ([unless you implement one yourself](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/how-to-implement-a-partitioner-for-static-partitioning)).

_PLINQ_ chooses **Range partitioning** for any sequence which can be indexed into i.e `IList<T>` (or `ParallelEnumerable.Range`) which _usually_ results in a better performance unless the amount of processing required for each element is not constant. For example in a scenario where there is a non-uniform processing time (think of ray-tracing), threads can finish work on their assigned partitions early and sit there idle unable to process more items. For such scenarios with non-uniform processing times, _Chunk_ partitioning may be a better strategy. It is possible to instruct _PLINQ_ to choose _Chunk_ partitioning for an _indexable_ sequence by using `Partitioner.Create(myArray, true)`.

**Hash partitioning** is used by query operators that require comparing elements (e.g. `GroupBy`, `Join`, `GroupJoin`, `Intersect`, `Except`, `Union` and `Distinct`). _Hash_ partitioning can be inefficient in that it must precalculate the _hashcode_ for each element (so that elements with identical codes are assigned and processed on the same thread).

**Striped partitioning** is the strategy used by `SkipWhile` and `TakeWhile` and is optimized for processing items at the head of a data source. In striped partitioning, each of the n worker threads is allocated a small number of items (sometimes 1) from each block of n items. The set of items belonging to a single thread is called a _stripe_, hence the name. A useful feature of this scheme is that there is no inter-thread synchronization required as each worker thread can determine its data via simple arithmetic. This is really a special case of range partitioning and only works on arrays and types that implement `IList<T>`.

Here is a diagram showing an overview of the most widely used _Chunk_ and _Range_ partitioning taken from _Albahari's_ must read [Threading in C#](http://www.albahari.com/threading/part5.aspx#_Optimizing_PLINQ).

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-plinq.partitioning.png" | relative_url}}" target="_blank">
    <img alt="picture of plinq partitioning" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-plinq.partitioning.png" | relative_url}}" width="700">
    </a>
</p>

For more details on _Partitioning_, I highly suggest having a look at [Partitioning in PLINQ](https://blogs.msdn.microsoft.com/pfxteam/2009/05/28/partitioning-in-plinq/), [Chunk partitioning vs range partitioning in PLINQ](https://blogs.msdn.microsoft.com/pfxteam/2007/12/02/chunk-partitioning-vs-range-partitioning-in-plinq/), [Partitioning of Work](http://reedcopsey.com/2010/01/26/parallelism-in-net-part-5-partitioning-of-work/) and [Custom Partitioners for PLINQ and TPL](https://docs.microsoft.com/en-us/dotnet/standard/parallel-programming/custom-partitioners-for-plinq-and-tpl).

Okay, now that we know a little more about partitioning we can look at more _PLINQ_ implementations.

### 4. PLINQ

This time we are going to let _PLINQ_ do all the work by letting it partition the input sequence of lines (using _Chunk_ partitioning) and see if we can do any better than what we have done so far.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    return File.ReadLines(InputFile.FullName)
        .AsParallel()
        .SelectMany(l => l.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        .Where(IsValidWord)
        .ToLookup(x => x, StringComparer.InvariantCultureIgnoreCase)
        .AsParallel()
        .Select(x => new { Word = x.Key, Count = (uint)x.Count() })
        .OrderByDescending(kv => kv.Count)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Word, kv => kv.Count);
}
```

Running the above results in the following:

```shell
Execution time: 22,480 ms
Gen-0: 4, Gen-1: 2, Gen-2: 2
Peak WrkSet: 4,579,401,728
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-4.plinq.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-4.plinq.png" | relative_url}}" width="700">
    </a>
</p>

And just like that the program managed to achieve an average CPU utilization of **_60.7%_**, took **_22,480_** milliseconds out of which **_6.1%_** was spent on _GC_. **_62.4%_** of all our threads time were in the _Running_ state and **_37.6%_** in _Waiting_. Exploring the snapshot file reveals that around **_47%_** was spent executing _LINQ_ operations and **_14.5%_** on locking contention. Note the size of the _WorkingSet_ is still at around **_4.5GB_**.

Despite the great reduction in execution time we can still observe a substantial cost in locking contention. A large part of this cost is incurred during the grouping of the words (the step calling `ToLookup(x => x, StringComparer.InvariantCultureIgnoreCase)`).

### 5. PLINQ (MapReduce)

As we saw above, despite the simplicity and succinctness of the _PLINQ_ implementation, we still incurred a high cost of locking so now we will use a different flavour of _PLINQ_ which allows us to perform _MapReduce_. We are going to have a _map_ phase where our problem can be split into smaller independent steps which can then be executed on any thread without the need for synchronization.

Each worker thread will then do the work of splitting the lines and excluding the invalid words and when the time comes to group them all back together, each thread will then merge _reduce_ the result by locking over the shared collection which will happen per partition.

Let us see how this can be implemented using the `Aggregate` method in _PLINQ_:

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    return File.ReadLines(InputFile.FullName)
        .AsParallel()
        .Aggregate(
/*#1*/      () => new Dictionary<string, uint>(StringComparer.InvariantCultureIgnoreCase),
/*#2*/      (localDic, line) =>
            {
                foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
                {
                    if (!IsValidWord(word)) { continue; }
                    TrackWordsOccurrence(localDic, word);
                }
                return localDic;
            },
/*#3*/      (finalResult, localDic) =>
            {
                foreach (var pair in localDic)
                {
                    var key = pair.Key;
                    if (finalResult.ContainsKey(key))
                    {
                        finalResult[key] += pair.Value;
                    }
                    else
                    {
                        finalResult[key] = pair.Value;
                    }
                }
                return finalResult;
            },
/*#4*/      finalResult => finalResult
                .OrderByDescending(kv => kv.Value)
                .Take((int)TopCount)
                .ToDictionary(kv => kv.Key, kv => kv.Value)
        );
}
```

Still confusing? Here are the main steps again:

- Step #1: First we supply a `Dictionary<string, uint>` for each of the threads running our code so this means every thread will have a local collection to work with, eliminating the need for locking.

- Step #2: Here we have the main logic which will be executed by each thread for every line in our file, the scary looking `Func<Dictionary<string, uint>, string, Dictionary<string, uint>>` allows the dictionary declared at step _#1_ to be passed as `localDic` along with the line read from the disk to the body of the delegate. After the thread is done adding the words to the dictionary it will return the `localDic` so that it can be available to the same thread operating on the next line in the same partition.

- Step #3: Once every thread has finished processing all the items in its partition then all the items on its local dictionary is then merged (reduced) into a final result-set. Hence, the delegate is of type `Func<Dictionary<string, uint>, Dictionary<string, uint>>`. By this step our _MapReduce_ implementation is actually complete however the `Aggregate` method offers us one final step.

- Step #4: This is where you have the chance to do any modification to the final result before returning it. In our case we are doing a simple ordering followed by a `Take` and projection back into a dictionary.

So let us see how we did this time:

```shell
Execution time: 21,915 ms
Gen-0: 12, Gen-1: 5, Gen-2: 3
Peak WrkSet: 1,082,503,168
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-5.plinq.map.reduce.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-5.plinq.map.reduce.png" | relative_url}}" width="700">
    </a>
</p>

This time the program achieved an average CPU utilization of **_45.9%_**, it took **_21,915_** milliseconds out of which only **_1.4%_** was spent on _GC_. **_55.1%_** of all our threads time were in the _Running_ state and **_44.9%_** in _Waiting_. So despite the lower CPU utilization in comparison to the previous step the lower time in _GC_ did actually improve the overall execution time. Exploring the snapshot file reveals that around **_37%_** was spent executing _LINQ_ operations and **_6.9%_** on locking contention. You can also see that our peak memory usage was **_4x_** smaller than the previous method. So while we may not have improved the execution time significantly we have managed to achieve more by using less CPU as well as a much lower memory footprint.

Not too shabby ha! :-)

### 6. PLINQ (ConcurrentDictionary)

_PLINQ_'s biggest advantage/convenience is thanks to its ability to collate the result of the parallelized work into a single output sequence and that is indeed what we have used until now. For this pattern we are going to do things a bit differently; First, we are not going to use _PLINQ_'s result collation. Instead, we are going to give it a function and tell it to run multiple threads each applying that function to our data. The method allowing us to achieve this is `ForAll`. That was the first part, now for the second part we are going to choose a different data type for tracking the count of words.

Sometimes choosing the right datatype (in our case collection), can make all the difference. There is no better example to demonstrate this than what we will see in this section. We are going to pick the `ConcurrentDictionary` as our collection and let it handle all the synchronization for us.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);

    File.ReadLines(InputFile.FullName)
        .AsParallel()
        .ForAll(line =>
        {
            foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
            {
                if (!IsValidWord(word)) { continue; }
                result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
            }
        });

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

There is not much to the code except the use of `AddOrUpdate`. Each thread is using this method to _Add_ a given word with a count of _1_, if the word was already added, it then takes the value (count) of that word and adds _1_ to it. All of this is done in a thread-safe manner and any synchronization is handled by the `ConcurrentDictionary`. Okay, how did we do this time?

```shell
Execution time: 19,229 ms
Gen-0: 16, Gen-1: 8, Gen-2: 4
Peak WrkSet: 863,531,008
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-6.plinq.concurrent.dictionary.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-6.plinq.concurrent.dictionary.png" | relative_url}}" width="700">
    </a>
</p>

This time the program achieved an average CPU utilization of **_53%_**, it took **_19,229_** milliseconds out of which only **_7%_** was spent on _GC_. **_65.7%_** of all our threads time were in the _Running_ state and **_34.3%_** in _Waiting_. Exploring the snapshot file reveals that around **_33%_** was spent executing _LINQ_ operations and **_10.3%_** in locking.

### 7. PLINQ (ProducerConsumer)

In this final section of using _PLINQ_ to achieve parallelism, we are going to cover my favorite pattern which I frequently use to deal with problems where I have one or more producers producing data and one or more workers consuming them.

For this algorithm, we have a single _producer_ reading and adding (producing) the lines into some sort of a collection. We then have a bunch of _consumers_ taking (consuming) and processing each line. The collection which allows such dance between the producer and its consumers is called the `BlockingCollection<T>` which by default is very similar to a [ConcurrentQueue<T>](<https://msdn.microsoft.com/en-us/library/dd267265(v=vs.110).aspx>). There is more to `BlockingCollection<T>` than I will cover today but the bits we are interested in are the _Add_ method which the producer will use to add each line to the collection; The `GetConsumingEnumerable()` which blocks each thread invoking it until there is an item in the collection and finally the `CompleteAdding` method which notifies the consumers that no more item will be added by the publisher(s) therefore unblocking every thread that is waiting on `GetConsumingEnumerable()`.

As far as _PLINQ_ is concerned we are telling it to use at most _12_ threads and also to not buffer the input so that as soon as the _publisher_ adds a line, _PLINQ_ can pass that to one of its _12_ _consumers_ and through the rest of the pipeline.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    const int WorkerCount = 12;
    const int BoundedCapacity = 10_000;
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);

    // Setup the queue
    BlockingCollection<string> blockingCollection = new(BoundedCapacity);

    // Declare the worker
    Action<string> work = line =>
    {
        foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        {
            if (!IsValidWord(word)) { continue; }
            result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
        }
    };

    Task.Run(() =>
    {
        // Begin producing
        foreach (var line in File.ReadLines(InputFile.FullName))
        {
            blockingCollection.Add(line);
        }
        blockingCollection.CompleteAdding();
    });

    // Start consuming
    blockingCollection
        .GetConsumingEnumerable()
        .AsParallel()
        .WithDegreeOfParallelism(WorkerCount)
        .WithMergeOptions(ParallelMergeOptions.NotBuffered)
        .ForAll(work);

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

With the output of:

```shell
Execution time: 18,858 ms
Gen-0: 16, Gen-1: 9, Gen-2: 4
Peak WrkSet: 857,141,248
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-7.plinq.producer.consumer.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-7.plinq.producer.consumer.png" | relative_url}}" width="700">
    </a>
</p>

This time the program achieved an average CPU utilization of **_30.7%_**, it took **_18,858_** milliseconds out of which only **_7.2%_** was spent on _GC_. **_67.9%_** of all our threads time were in the _Running_ state and **_32.1%_** in _Waiting_. Exploring the snapshot file reveals that around **_15%_** was spent executing _LINQ_ operations and **_2.2%_** in locking contention.

As you can see, we managed to achieve very good concurrency by only using half of our cores and reduced the overall _Waiting_ time and locking cost.

I cannot think of any other ways to squeeze any more performance juice out of _PLINQ_; (I am happy to be challenged by anyone on this, let me know in the comments) therefore, we are now going to start playing with the `Parallel` class.

### 8. Parallel.ForEach (MapReduce)

The [Parallel](<https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx>) class has a bunch of methods for us to use but the one we are interested in is the `Parallel.ForEach`. In it's simplest form it accepts an `IEnumerable<T>` and an `Action<T>` which it then invokes in parallel for every item in the collection. Using this method however is not suitable for our use-case as we would need to synchronize access to our dictionary responsible for tracking the occurrence of each word therefore diminishing any benefits of parallelization. Instead, we are going to use the overload of the method which enables us to implement _MapReduce_. Here's the code:

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    Dictionary<string, uint> result = new (StringComparer.InvariantCultureIgnoreCase);
    Parallel.ForEach(
        File.ReadLines(InputFile.FullName),
        new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
        () => new Dictionary<string, uint>(StringComparer.InvariantCultureIgnoreCase),
        (line, state, index, localDic) =>
        {
            foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
            {
                if (!IsValidWord(word)) { continue; }
                TrackWordsOccurrence(localDic, word);
            }
            return localDic;
        },
        localDic =>
        {
            lock (result)
            {
                foreach (var pair in localDic)
                {
                    var key = pair.Key;
                    if (result.ContainsKey(key))
                    {
                        result[key] += pair.Value;
                    }
                    else
                    {
                        result[key] = pair.Value;
                    }
                }
            }
        }
    );

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

The implementation is very similar to the _MapReduce_ we implemented using _PLINQ_ and as you saw before, the main idea behind this pattern is to ensure each thread has it's local data to work with and then when all the threads have processed all their items they will then merge (_reduce_) their results into a single sequence therefore greatly reducing synchronization.

Here is what this implementation gets us:

```shell
Execution time: 19,606 ms
Gen-0: 23, Gen-1: 13, Gen-2: 7
Peak WrkSet: 942,002,176
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-8.parallel.foreach.map.reduce.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-8.parallel.foreach.map.reduce.png" | relative_url}}" width="700">
    </a>
</p>

Okay, not bad. We have an average CPU utilization of **_35.9%_**, an execution time of **_19,606_** milliseconds out of which **_3.4%_** was spent on _GC_. **_40.5%_** of all our threads time were in the _Running_ state and **_59.5%_** in _Waiting_. Looking at the snapshot we can see that the cost of locking contention is a whopping **_35%_**, why? This is mainly due to the _Chunk Partitioning_ if we were using _Range Partitioning_, we would only need to _lock_ over the `result` once per thread at the end of its partition but for that we would need to load all the lines into an indexable collection such as an `Array` or an `IList` which would be fine if you had all the available RAM. In our case however, I was happy to pay the cost of having to _lock_ more frequently (due to much smaller chunks).

### 9. Parallel.ForEach (ConcurrentDictionary)

In this section we are going to again use the `ConcurrentDictionary` and see what it gets us when used with `Parallel.ForEach`.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);
    Parallel.ForEach(
        File.ReadLines(InputFile.FullName),
        new ParallelOptions { MaxDegreeOfParallelism = Environment.ProcessorCount },
        (line, state, index) =>
        {
            foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
            {
                if (!IsValidWord(word)) { continue; }
                result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
            }
        }
    );

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

Which outputs:

```shell
Execution time: 19,622 ms
Gen-0: 13, Gen-1: 6, Gen-2: 3
Peak WrkSet: 853,209,088
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-9.parallel.foreach.concurrent.dictionary.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-9.parallel.foreach.concurrent.dictionary.png" | relative_url}}" width="700">
    </a>
</p>

I don't know about you but I quite like the `ConcurrentDictionary`! We have an average CPU utilization of **_52.8%_**, an execution time of **_19,622_** milliseconds out of which **_6.7%_** was spent on _GC_. **_61.6%_** of all our threads time were in the _Running_ state and **_38.4%_** in _Waiting_. Looking at the snapshot reveals the cost of locking contention to be **_9.4%_**.

Despite the great result we can see from the profiling graph that we have _over loaded_ the CPU. How can I tell? Well each of those pink sections on the graph is showing us the time intervals where the thread is ready to run on the next available CPU core; Therefore, long ready intervals (the time the thread is waiting so that it can switch its state from _Waiting_ to _Running_) can mean thread starvation and CPU overload. To address it, we can try reducing the number of threads by playing with the `MaxDegreeOfParallelism`. So remember that CPU overloading/thread starvation _can_ and _will_ hurt the performance.

### 10. ProducerConsumer with Tasks

We talked about the _ProducerConsumer_ pattern in the _PLINQ_ section, what we are doing here is pretty much identical to what we did before except we are going to use [Tasks](<https://msdn.microsoft.com/en-us/library/system.threading.tasks.task(v=vs.110).aspx>) to execute our _consumers_.

Note the most important thing to remember about this pattern is that once the worker threads finish processing each item they get from the queue, they will stay blocked waiting for more items until the queue is marked as _complete_. So depending on the size of your input, you need to make sure that when you are assigning worker threads using `Task`s, you mark them with the [TaskCreationOptions.LongRunning])(https://msdn.microsoft.com/en-us/library/system.threading.tasks.taskcreationoptions(v=vs.110).aspx) flag otherwise the [Hill Climbing](https://github.com/dotnet/coreclr/blob/master/src/vm/hillclimbing.cpp) aka [Thread Injection](http://mattwarren.org/2017/04/13/The-CLR-Thread-Pool-Thread-Injection-Algorithm/) algorithm employed by _TPL_ thinks that threads in the `ThreadPool` are blocked so in order to prevent against _dead-locks_, it kicks in and inject extra threads to the `ThreadPool` at a [rate of one thread every 500 milliseconds](https://msdn.microsoft.com/en-gb/library/ff963549.aspx) which is not what we want. By using this flag you are telling _TPL_ to spin up a dedicated thread for each of your workers/consumers.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    const int WorkerCount = 12;
    const int BoundedCapacity = 10_000;
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);

    // Setup the queue
    BlockingCollection<string> blockingCollection = new(BoundedCapacity);

    // Declare the worker
    Action work = () =>
    {
        foreach (var line in blockingCollection.GetConsumingEnumerable())
        {
            foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
            {
                if (!IsValidWord(word)) { continue; }
                result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
            }
        }
    };

    // Start the workers
    var tasks = Enumerable.Range(1, WorkerCount)
        .Select(n =>
            Task.Factory.StartNew(
                work,
                CancellationToken.None,
                TaskCreationOptions.LongRunning,
                TaskScheduler.Default))
        .ToArray();

    // Begin producing
    foreach (var line in File.ReadLines(InputFile.FullName))
    {
        blockingCollection.Add(line);
    }
    blockingCollection.CompleteAdding();
    // End of producing

    // Wait for workers to finish their work
    Task.WaitAll(tasks);

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

Which outputs:

```shell
Execution time: 31,979 ms
Gen-0: 15, Gen-1: 8, Gen-2: 4
Peak WrkSet: 935,489,536
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-10.task.producer.consumer.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-10.task.producer.consumer.png" | relative_url}}" width="700">
    </a>
</p>

We have an average CPU utilization of **_26.5%_**, an execution time of **_31,979_** milliseconds out of which **_4.3%_** was spent on _GC_. **_59.1%_** of all our threads time were in the _Running_ state and **_40.9%_** in _Waiting_. Looking at the snapshot we can see that cost of locking contention was at **_2%_**.

### 11. ProducerConsumer with Tasks (Easier)

As I mentioned before, I use this pattern regularly therefore, over the years I have built an _opinionated_ wrapper around it to reduce boiler plate as well as providing easy exception handling and control over the producers and consumers. It can also be used as a _single-thread_ worker (_sequencer_) which is another pattern of its own that I may write about in another article. It is part of my [Easy.Common](https://github.com/NimaAra/Easy.Common) library available on [NuGet](https://www.nuget.org/packages/Easy.Common/). So let us implement what we did in the previous section by using the [ProducerConsumerQueue](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/ProducerConsumerQueue.cs).

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    const int WorkerCount = 12;
    const int BoundedCapacity = 10_000;
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);

    // Declare the worker
    Action<string> work = line =>
    {
        foreach (var word in line.Split(Separators, StringSplitOptions.RemoveEmptyEntries))
        {
            if (!IsValidWord(word)) { continue; }
            result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
        }
    };

    // Setup the queue
    ProducerConsumerQueue<string> pcq = new(work, WorkerCount, BoundedCapacity);
    pcq.OnException += (sender, ex) => Console.WriteLine("Oooops: " + ex.Message);

    // Begin producing
    foreach (var line in File.ReadLines(InputFile.FullName))
    {
        pcq.Add(line);
    }
    pcq.CompleteAdding();
    // End of producing

    // Wait for workers to finish their work
    pcq.Completion.Wait();

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

With the following output:

```shell
Execution time: 39,124 ms
Gen-0: 14, Gen-1: 7, Gen-2: 4
Peak WrkSet: 934,985,728
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-11.task.producer.consumer.easier.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-11.task.producer.consumer.easier.png" | relative_url}}" width="700">
    </a>
</p>

We have an average CPU utilization of **_25.1%_**, an execution time of **_39,124_** milliseconds out of which **_3.4%_** was spent on _GC_. **_53.8%_** of all our threads time were in the _Running_ state and **_46.2%_** in _Waiting_. The snapshot shows the cost of locking contention as **_2%_**.

### 12. TPL Dataflow

Finally and for the sake of completion we are going to implement the same solution using _TPL Dataflow_ which provides a pipeline based execution and is great for implementing _ActorModel_ scenarios as it provides (among other things) _message passing_. I am not going to focus much on _Dataflow_ or _ActorModel_ as there is a lot to cover.

We are going to start by creating the `bufferBlock` to which we push our lines, the block is then linked to `splitLineToWordsBlock` which as its name suggests is responsible for turning our lines into words. The `splitLineToWordsBlock` itself is linked to the `batchWordsBlock` which batches up our words into arrays of length _5000_. Finally, each batch is then fed through the `trackWordsOccurrencBlock` which tracks the occurrence of each word.

Once we have all our blocks linked together, all we have to do is read our lines and feed them through the `bufferBlock`; When we are done pushing all the lines, we mark the `bufferBlock` as complete and wait for the `trackWordsOccurrencBlock` to finish consuming all its batches. Finally, just like the previous steps we order the result and return the top n entries.

```csharp
private static IDictionary<string, uint> GetTopWords()
{
    const int WorkerCount = 12;
    ConcurrentDictionary<string, uint> result = new(StringComparer.InvariantCultureIgnoreCase);
    const int BoundedCapacity = 10_000;

    BufferBlock<string> bufferBlock = new(new DataflowBlockOptions { BoundedCapacity = BoundedCapacity });

    TransformManyBlock<string, string> splitLineToWordsBlock = new(
        line => line.Split(Separators, StringSplitOptions.RemoveEmptyEntries),
        new ExecutionDataflowBlockOptions
        {
            MaxDegreeOfParallelism = 1,
            BoundedCapacity = BoundedCapacity
        });

    BatchBlock<string> batchWordsBlock = new(5_000);

    ActionBlock<string[]> trackWordsOccurrencBlock = new(words =>
        {
            foreach (var word in words)
            {
                if (!IsValidWord(word)) { continue; }
                result.AddOrUpdate(word, 1, (key, oldVal) => oldVal + 1);
            }
        },
        new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = WorkerCount });

    DataflowLinkOptions defaultLinkOptions = new () { PropagateCompletion = true };
    bufferBlock.LinkTo(splitLineToWordsBlock, defaultLinkOptions);
    splitLineToWordsBlock.LinkTo(batchWordsBlock, defaultLinkOptions);
    batchWordsBlock.LinkTo(trackWordsOccurrencBlock, defaultLinkOptions);

    // Begin producing
    foreach (var line in File.ReadLines(InputFile.FullName))
    {
        bufferBlock.SendAsync(line).Wait();
    }

    bufferBlock.Complete();
    // End of producing

    // Wait for workers to finish their work
    trackWordsOccurrencBlock.Completion.Wait();

    return result
        .OrderByDescending(kv => kv.Value)
        .Take((int)TopCount)
        .ToDictionary(kv => kv.Key, kv => kv.Value);
}
```

Which outputs:

```shell
Execution time: 31,447 ms
Gen-0: 18, Gen-1: 10, Gen-2: 4
Peak WrkSet: 868,356,096
```

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-12.tpl.dataflow.png" | relative_url}}" target="_blank">
    <img alt="picture of profiling result using dotTrace" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-12.tpl.dataflow.png" | relative_url}}" width="700">
    </a>
</p>

We have an average CPU utilization of **_21.6%_**, an execution time of **_31,447_** milliseconds out of which **_4.3%_** was spent on _GC_. **_25.1%_** of all our threads time were in the _Running_ state and **_74.9%_** in _Waiting_. Looking at the snapshot we can see the cost of locking contention was at **_0.01%_** which is not surprising as our entire processing was pipelined.

### Conclusion

Here is a quick a summary of what each implementation achieved:

<p style="text-align: center;">
    <a href="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-result.summary.png" | relative_url}}" target="_blank">
    <img alt="picture of a table showing the result of profiling of each variation compared to one another" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-result.summary.png" | relative_url}}" width="700">
    </a>
</p>

We saw that both the `Parallel` class and _PLINQ_ can efficiently partition tasks and execute them across all the available cores in our system. We also learned how choosing a different data type, library or even different overloads of the same library can help us achieve better performance and CPU utilization.

We then looked at some of the other approaches to achieving parallelism mainly _ProducerConsumer_ and _ActorModel_ which may not have necessarily performed well for our [DataParallelism](https://en.wikipedia.org/wiki/Data_parallelism) scenario but can be suitable for [TaskParallelism](https://en.wikipedia.org/wiki/Task_parallelism) problems you may face in the future.

Finally, in order to help you choose between _PLINQ_ and `Parallel`, I would like to mention an important difference between the two which goes back to how they were designed.

`Parallel` uses a concept referred to as _Replicating Tasks_ which means a loop will start with one task for processing the loop body and if more threads are freed up to assist with the processing, additional tasks will be created to run on those threads which minimizes resource consumption in scenarios where most of the threads in the `ThreadPool` are busy doing other work. _PLINQ_ on the other hand requires that a specific number of threads are actively involved for the query to make progress. This difference in design is the reason why there are [ParallelOptions.MaxDegreeOfParallelism & PLINQ’s WithDegreeOfParallelism](https://blogs.msdn.microsoft.com/pfxteam/2009/05/29/paralleloptions-maxdegreeofparallelism-vs-plinqs-withdegreeofparallelism/).

I hope you enjoyed reading this article as much as I enjoyed writing it and I invite you to share the patterns and techniques that you use for _multi-threading_ and _parallelism_ in the comments section below.

Have fun and let's:

<p style="text-align: center;">
    <img alt="picture of a meme being excited to use parallelization in everything" src="{{"/assets/imgs/2017-06-01-practical-parallelization-in-csharp-parallelize.all.the.things.jpg" | relative_url}}" width="400">    
</p>
