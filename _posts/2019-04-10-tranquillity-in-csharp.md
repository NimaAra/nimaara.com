---
title: "Tranquillity in C# with EasyDictionary"
tags: "C# .NET"
redirect_from:
  - /easier-dictionary-in-c/
  - /2019/04/10/easier-dictionary-in-c/
---

### Tranquillity in C# with EasyDictionary

[`Dictionary<TKey, TValue>`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.dictionary-2) is probably among the most used collections in C#. If you are reading this blog then I am assuming you know the fundamentals of this data structure and if you are not, you can have a look at [The .NET Dictionary](https://www.red-gate.com/simple-talk/blogs/the-net-dictionary/) which covers it in detail. As a summary however:

- `Dictionary<TKey, TValue>` is a [Hash table](https://en.wikipedia.org/wiki/Hash_table) mapping a set of keys to a set of values.
- Each key in the dictionary must be unique according to the [`IEqualityComparer<TKey>`](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.iequalitycomparer-1) (used to determine the equality of the keys in the dictionary).
- Retrieving a value by using its key has a complexity of _O(1)_ (very fast).

### Typical use-case

Most of the time, I use a dictionary to map a model to one of its properties; Let's say an _Id_ or _Name_. For example, given the model:

```csharp
public sealed record class Person(Guid Id, string Name, uint Age);
```

And given a collection of people from somewhere:

```csharp
IEnumerable<Person> people = Enumerable.Range(1, 10)
    .Select(n => new Person(Guid.NewGuid(), "P-" + n.ToString(), (uint)n));
```

We may now want to make `people` available for fast lookups based on the _Id_ of a given person.

Well that's easy, let's new-up a `Dictionary` and add our people to it:

```csharp
Dictionary<Guid, Person> dic = new();
foreach(Person p in people)
{
    dic[p.Id] = p;
    // or if you don't like the index syntax then you can instead do:
    // dic.Add(p.Id, p);
}
```

We can also create a dictionary directly from the `IEnumerable<Person>` by doing:

```csharp
Dictionary<Guid, Person> dic = Enumerable.Range(1, 10)
    .Select(n => new Person(Guid.NewGuid(), "P-" + n.ToString(), (uint)n))
    .ToDictionary(p => p.Id, p => p);
```

Okay not the most verbose code to complain about but if you take into account that every time you want to `Add`, `TryGet` or check for existence (`Contains`) of a given value you need to specify the `TKey` explicitly then you will be able to also hear my _OCD_ screaming to make things simpler.

<p style="text-align: center;">
    <img alt="picture of a meme showing a creature with a twitching eye" src="{{"/assets/imgs/2019-04-10-tranquillity-in-csharp.ocd.gif" | relative_url}}" width="400">    
</p>

### What if...

Would it not be more _elegant_ if we could just add a person and let the dictionary get the _Id_ from it? Something similar to:

```csharp
EasyDictionary<Guid, Person> dic = new (...);
foreach(Person p in people)
{
    dic.Add(p)
}
```

In order to do this, we would need some sort of a key selector which for a given `TValue` would return a `TKey`. In other words, we need a `Func<TValue, TKey>` which in our example would be a `Func<Person, Guid>`.

Now that we know what a key selector would look like, we can start thinking about the structure of our new abstraction:

```csharp
public sealed class EasyDictionary<TKey, TValue>
{
    public EasyDictionary(Func<TValue, TKey> keySelector)
    {
        KeySelector = keySelector;
    }

    public Func<TValue, TKey> KeySelector { get; }
    ...
}
```

All we have to do now is put a `Dictionary<TKey, TValue>` inside our wrapper, implement `IReadOnlyDictionary<TKey, TValue>` as well as `ICollection<TValue>` and then redirect all the required methods and properties to our internal dictionary.

### Job done!

I am going to spare you the unexciting code here and instead, point you to the completed version in [`EasyDictionary`](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/EasyDictionary.cs) available as part of the [Easy.Common](https://github.com/NimaAra/Easy.Common) [NuGet package](https://www.nuget.org/packages/Easy.Common).

Armed with our much smarter dictionary, let us see what we can do with it (look at the [unit tests](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common.Tests.Unit/EasyDictionary/EasyDictionaryTests.cs) for more examples):

```csharp
Person[] people = Enumerable.Range(1, 10)
    .Select(n => new Person(Guid.NewGuid(), "P-" + n.ToString(), (uint)n))
    .ToArray();

EasyDictionary<Guid, Person> easyDic = new(p => p.Id);
foreach(Person p in people)
{
    easyDic.Add(p);
}

// Let's lookup an existing value.
Person firstPerson = people[0];

easyDic.Contains(firstPerson);                          // true
easyDic.ContainsKey(firstPerson.Id);                    // true

easyDic[firstPerson.Id];                                // firstPerson

easyDic.TryGetValue(firstPerson.Id, out Person yep);    // true (yep is firstPerson)

easyDic.Remove(firstPerson.Id);                         // true (removed)
easyDic.Remove(firstPerson);                            // false (no longer there)

easyDic.TryGetValue(firstPerson.Id, out Person _);      // false (no longer there)
```

We can also enumerate over the items:

```csharp
foreach (Person p in easyDic) { /* do what you want with p */ }
```

But what about over-writing an existing item? Well, we can use `AddOrReplace` for that:

```csharp
Person olderPerson = new(
    firstPerson.Id,
    firstPerson.Name,
    firstPerson.Age + 41);

easyDic.AddOrReplace(olderPerson); // true;
```

### Bonus point

We can even throw a bunch of [extension methods](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/Extensions/EnumerableExtensions.cs#L148,#L159) to be able to do:

```csharp
EasyDictionary<Guid, Person> dic = Enumerable.Range(1, 10)
    .Select(n => new Person(Guid.NewGuid(), "P-" + n.ToString(), (uint)n))
    .ToEasyDictionary(p => p.Id);
```

Now this is tranquillity!

<p style="text-align: center;">
    <img alt="picture of a meme showing a calm horse" src="{{"/assets/imgs/2019-04-10-tranquillity-in-csharp.horse.gif" | relative_url}}" width="400">    
</p>
