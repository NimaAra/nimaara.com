---
title: "Local Functions, a C# 7 Feature"
tags: "C# .NET"
redirect_from:
  - /local-functions-a-new-c/
  - /2017/04/01/local-functions-a-new-c/
---

## Local Functions, a C# 7 Feature

_C# 7_ has introduced a number of features which you can explore HERE; In this article we are going to take a look at **Local Functions** and why you may want to use them.

### What are they?

_Local Functions_ enable the definition of functions inside other functions. You can think of them as helper functions that would only make sense within a given context. Such context could be a function, constructor or a property _getter_ or _setter_. They are transformed by the compiler to _private_ methods meaning there is no overhead in calling them.

The main goal of _Local Functions_ is _encapsulation_ so the compiler is enforcing that such functions cannot be called from anywhere else in the class. Consider the following overly simplified example:

```csharp
public string Address
{
    get
    {
        return $@"{GetHouseNumber()}, {GetStreet()},
                  {GetCity()},
                  {GetCountry()}";

        int GetHouseNumber() => 10;
        string GetStreet() => "Liberty St";
        string GetCity() => "Amsterdam";
        string GetCountry() => "Jamaica";
    }
}
```

Here we have a _getter_ property which is composing an address comprised of 4 different segments. Each segment is being retrieved via a separate method, we are also taking advantage of the [Expression Bodied](https://msdn.microsoft.com/en-gb/magazine/dn802602.aspx) and [String Interpolation](https://msdn.microsoft.com/en-gb/magazine/dn879355.aspx) syntax introduced as part of _C# 6_ to keep our property clear and condensed.

Had we declared those methods as _private_, they could be invoked by any member in the class so by using _Local Functions_ we have encapsulated their usage.

### But we could do this with Lambdas, no?

Yes, if you ignore the constraint forcing us to declare our lambdas before using them, you could technically achieve the same result:

```csharp
public string Address
{
    get
    {
        Func<int> getHouseNumber = () => 10;
        Func<string> getStreet = () => "Liberty St";
        Func<string> getCity = () => "Amsterdam";
        Func<string> getCountry = () => "Jamaica";

        return $@"{getHouseNumber()}, {getStreet()},
                  {getCity()},
                  {getCountry()}";
    }
}
```

_BUT_, doing so is not such a good idea for the following reasons:

- **Allocation**, By declaring those lambdas you are essentially allocating objects for the delegates on the heap so if your property is being invoked many times then you will end up putting non trivial unnecessary pressure on the _GC_.
- **No support for** `ref`, `out`, `params` or _Optional Parameters_.
- **No recursion**, in order to be able to call a Lambda recursively you would need to declare it in 2 steps which does not seem elegant to me:

```csharp
private uint GetFactorial(uint number)
{
    var factorial = default(Func<uint, uint>);
    factorial = n =>
    {
        if (n < 2) { return 1; }
        return n * factorial(n - 1);
    };

    return factorial(number);
}
```

### Limitations

_Local Functions_ can be _generic_, _asynchronous_, _dynamic_ and they can even have access to variables accessible in their enclosing scope however despite their flexibility they come with the following limitations:

- **Cannot be static**, you cannot declare a _Local Function_ as static. (no longer true as of _C# 8_)
- **No attributes allowed**, Unlike normal methods you cannot use attributes inside a _Local Function_. This means you cannot use [Caller Information](https://msdn.microsoft.com/en-us/library/hh534540.aspx) so the following example does not compile:

```csharp
private void Greet()
{
    string GetCaller([CallerMemberName] string name = null) => name;
    Console.WriteLine("Hello " + GetCaller());
}
```
