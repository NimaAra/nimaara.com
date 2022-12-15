---
title: "Per Object Garbage Collection Notification in .NET"
tags: "C# .NET GC"
redirect_from:
  - /per-object-garbage-collection-notification-in-net/
  - /2016/01/16/per-object-garbage-collection-notification-in-net/
---

## Per Object Garbage Collection Notification in .NET

Ever since _.NET 3.5_ we have been able to get notifications when a _Garbage Collection_ is pending, this can be achieved by using: [GC.RegisterForFullGCNotification](https://msdn.microsoft.com/en-us/library/system.gc.registerforfullgcnotification%28v=vs.100%29.aspx)

<blockquote>
  This feature is only available when <strong>concurrent garbage collection</strong> is disabled which is enabled by default in most cases unless specified otherwise or <strong>gcServer</strong> is used. The reason this is not available is that allocations are allowed while concurrent GC is in progress.
</blockquote>

Receiving notifications of a GC can be useful in scenarios where a full GC could impact performance and/or increase latency on a critical path so having knowledge of a pending GC enables the developer to redirect requests or offload the workload to other server(s).

You can find out more about the [Garbage Collection Notifications](https://msdn.microsoft.com/en-us/library/cc713687%28v=vs.100%29.aspx).

In this article however, we are going to find out how to get notified when an **individual object** is collected. I have used this method in debugging and learning about the behavior of GC in some of the applications and algorithms that I had been working on over the years.

### Meet ConditionalWeakTable

The [`ConditionalWeakTable<TKey, TValue>`](https://msdn.microsoft.com/en-us/library/dd287757%28v=vs.110%29.aspx) allows a developer to associate an arbitrary data (`TValue`) to another object (`TKey`). It was initially introduced to help implement [DLR](https://msdn.microsoft.com/en-us/library/dd233052%28v=vs.110%29.aspx) allowing you to add properties to an object. You may be tempted to implement this yourself by creating a structure which internally stores a [WeakReference](https://msdn.microsoft.com/en-us/library/system.weakreference%28v=vs.110%29.aspx) to the passed in object (`TKey`) and indeed that is how `ConditionalWeakTable` internally stores the reference but it goes beyond that, it guarantees that as long as the key is in the memory the associated value will also be alive.

`ConditionalWeakTable<TKey, TValue>` is _thread-safe_ and looks similar to a `Dictionary<TKey, TValue>` but it is far from a general purpose collection, it does not use `GetHashCode` and/or `Equals` for the equality comparisons instead it relies on `ReferenceEquals` plus it is not _enumerable_.

Today I am using this relatively unknown structure to build us an object `GCWatcher`!

```csharp
internal static class GCWatcher
{
    private readonly static ConditionalWeakTable<Object, NotifyWhenGCd> weakTable =
        new ConditionalWeakTable<Object, NotifyWhenGCd>();

    private sealed class NotifyWhenGCd
    {
        private readonly Action _value;
        internal NotifyWhenGCd(Action value) { _value = value; }

        ~NotifyWhenGCd() { _value(); }
    }

    public static T GCWatch<T>(this T objectToWatch, Action tag) where T : class
    {
        NotifyWhenGCd existing;
        if (!weakTable.TryGetValue(objectToWatch, out existing))
        {
            weakTable.Add(objectToWatch, new NotifyWhenGCd(tag));
        }
        return objectToWatch;
    }
}
```

You can then use it as:

```csharp
void Main()
{
    var o = new Exception()
        .GCWatch(() => Console.WriteLine("Object released at: {0:yyyy-MM-dd HH:mm:ss.fff}", DateTime.UtcNow));

    GC.Collect();       // There should NOT be any notifications because of the next line
    GC.KeepAlive(o);    // We are making sure that our object is kept alive up to here
    o = null;           // We are now killing our object!
    GC.Collect();       // We should be getting our notification some time after this line executes

    Console.ReadLine();
}
```

### Fancy a Puzzle?

Run the code above but instead of watching an instance of an `Exception`, use a piece of `String` whatever you like, will you see the _Object release at..._ message? Why?!

### Important

It is important to understand what this code does, it takes advantage of a _Finalizer/Destructor_ which means the internal `NotifyWhenGCd` object will require **at least two collections** to be reclaimed by the _GC_. So only use this if you know what to expect and always drive responsibly :-)
