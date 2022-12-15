---
title: Beware of the IDictionary<TKey, TValue>
tags: "C# .NET GC Performance"
redirect_from:
  - /beware-of-the-idictionary-tkey-tvalue/
  - /2016/03/06/beware-of-the-idictionary-tkey-tvalue/
---

## Beware of the IDictionary<TKey, TValue>

### Background

As it is the case in most scenarios the source of latency is usually due to the .NET _Garbage Collection_, so we always try very hard to minimize object allocations specially in our performance-critical components.

We have a messaging API which is used by many of our micro services. Some of those services use the API to publish thousands of messages per second to the market so it is important to keep the latency as low as possible.

In my quest to reduce latency and CPU consumption of our components, I came across a very interesting and probably not so well known performance gotcha!

### Problem

The API has a method which accepts an `IDictionary<string, string>` and publishes them to a message bus like so:

```csharp
public void Publish(IDictionary<string, string> data)
{
    foreach (var item in data)
    {
        _publisher.Publish(item);
    }
}
```

Nothing really complicated right? **WRONG!**

### What is going on?

Let's do a quick and dirty profiling of an even simpler example:

```csharp
void Main()
{
    Dictionary<int, int> dic = new();

    var sw = Stopwatch.StartNew();
    for (var i = 0; i < 100000000; i++)
    {
        foreach (var item in dic) { ; }
    }
    sw.Stop();

    // compose the result
    StringBuilder result = new();
    result.AppendFormat("[Time]\t{0}", sw.Elapsed);
    result.AppendLine();
    result.AppendFormat("[Gen-0]\t{0}", GC.CollectionCount(0));
    result.AppendLine();
    result.AppendFormat("[Gen-1]\t{0}", GC.CollectionCount(1));
    result.AppendLine();
    result.AppendFormat("[Gen-2]\t{0}", GC.CollectionCount(2));

    Console.WriteLine(result.ToString());
}
```

What we have here is an empty dictionary where both the keys and values are of type `int`. We are then simply enumerating the dictionary a **hundred million** times. Note we are not doing anything in the body of the _foreach_ so that we can focus only on the enumeration.

Finally, we record the total time it took for the enumeration as well as the number of times _GC_ performed collections of our _GEN 0_ to _GEN 2_. Running the above on a machine with a _Quad-Core 3.4GHz and 8GB of RAM_ prints:

```shell
[Time]  00:00:01.8128806
[Gen-0] 0
[Gen-1] 0
[Gen-2] 0
```

So it took almost **2 seconds** and resulted in no garbage collection, so far so good. Now let us try again but this time change our _dic_ pointer to an `IDictionary<int, int>` instead:

```csharp
... same as above
    IDictionary<int, int> dic = new Dictionary<int, int>();
... same as above
```

Which prints:

```shell
[Time]  00:00:04.3902405
[Gen-0] 765
[Gen-1] 2
[Gen-2] 1
```

This time it took more than twice than enumerating a `Dictionary<int, int>` and resulted in **765** collections of _GEN 0_, **2** collections of _GEN 1_ and **1** collection of _GEN 2_. This is **MADNESS!**

<p style="text-align: center;"><img alt="picture of homer simpson screaming" src="https://i.imgur.com/DuChpLR.gif?1"></p>

### And here is why

Let us look at the _IL_ generated in the case when we are using a `Dictionary<int, int>`:

```csharp
... not important
IL_0013:  ldloc.0     // dic
IL_0014:  callvirt    System.Collections.Generic.Dictionary<System.Int32,System.Int32>.GetEnumerator
IL_0019:  stloc.s     04
IL_001B:  br.s        IL_0029
IL_001D:  ldloca.s    04
IL_001F:  call        System.Collections.Generic.Dictionary<System.Int32,System.Int32>+Enumerator.get_Current
IL_0024:  stloc.s     05 // item
...
IL_0029:  ldloca.s    04
IL_002B:  call        System.Collections.Generic.Dictionary<System.Int32,System.Int32>+Enumerator.MoveNext
IL_0030:  brtrue.s    IL_001D
IL_0032:  leave.s     IL_0043
IL_0034:  ldloca.s    04
IL_0036:  constrained. System.Collections.Generic.Dictionary<,>.Enumerator
... not important
```

Here we can see a call to `System.Collections.Generic.Dictionary<System.Int32,System.Int32>+Enumerator.get_Current` to get the current `Enumerator` and on every iteration we have a call to `System.Collections.Generic.Dictionary<System.Int32,System.Int32>+Enumerator.MoveNext` to move the `Enumerator` forward.

Let us compare that with the _IL_ for the case of an `IDictionary<int, int>`:

```csharp
... not important
IL_0013:  ldloc.0     // dic
IL_0014:  callvirt    System.Collections.Generic.IEnumerable<System.Collections.Generic.KeyValuePair<System.Int32,System.Int32>>.GetEnumerator
IL_0019:  stloc.s     04
IL_001B:  br.s        IL_0029
IL_001D:  ldloc.s     04
IL_001F:  callvirt    System.Collections.Generic.IEnumerator<System.Collections.Generic.KeyValuePair<System.Int32,System.Int32>>.get_Current
IL_0024:  stloc.s     05 // item
...
IL_0029:  ldloc.s     04
IL_002B:  callvirt    System.Collections.IEnumerator.MoveNext
IL_0030:  brtrue.s    IL_001D
IL_0032:  leave.s     IL_0041
... not important
```

Very interesting! This time we have a `callvirt` to `System.Collections.Generic.IEnumerable<System.Collections.Generic.KeyValuePair<System.Int32,System.Int32>>.GetEnumerator` followed by more `callvirt` to `System.Collections.Generic.IEnumerator`.

Aha! Allow me to explain in case it is not that obvious.

### Virtual Call / Interface Dispatch

The first issue is the _Interface Dispatch_ (the parts that do a non-inlineable `callvirt` to `System.Collections.Generic.IEnumerator`) which is in contrast to the case of `Dictionary<int, int>` where we are doing fast, non-virtual and inlineable calls.

### Boxing

The second issue and by far the most significant performance hit is due to \__boxing_. Instead of returning a `Dictionary<int, int>.Enumerator`, we are boxing the `struct` and return an `IEnumerator<KeyValuePair<int, int>>` which explains the high number of _GC_ collections across our heap.

<p style="text-align: center;">
<img alt="picture of sherlock holmes smoking a pipe" src="https://i.imgur.com/nVrH5Nk.jpg">
</p>

### How about IList<T>, IReadOnlyList<T>, ICollection<T> & IEnumerable<T> vs List<T>?

`IList<T>` and `IReadOnlyList<T>` implement `ICollection<T>` which in turn implements `IEnumerable<T>` and therefore using any of those interfaces for the enumeration incurs the similar performance cost we covered above.

This is even more clear if you look at the source of `List<T>`:

```csharp
public Enumerator GetEnumerator() => new Enumerator(this);

IEnumerator<T> IEnumerable<T>.GetEnumerator() => new Enumerator(this);

System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => new Enumerator(this);
```

You can see in the last two `GetEnumerator()`s the `Enumerator struct` is being boxed to a generic an a non generic `IEnumerator`.

### Summary

It is a good idea to abstract yourself from an implementation, by that I mean programming against an interface rather than a concrete implementation _however_ there are times where this can have a performance impact and what we covered in this article was one of such examples. In our case the problem was fixed by simply switching to a `Dictionary<string, string>` which is the cost we are prepared to pay in return of a more efficient API.
