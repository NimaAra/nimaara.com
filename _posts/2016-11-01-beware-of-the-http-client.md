---
title: "Beware of the .NET HttpClient"
tags: "C# .NET Performance"
redirect_from:
  - /beware-of-the-net-httpclient/
  - /2016/11/01/beware-of-the-net-httpclient/
---

## Beware of the .NET HttpClient

In the old days of .NET (pre _4.5_) sending a HTTP request to a server could be accomplished by either using the [WebClient](<https://msdn.microsoft.com/en-us/library/system.net.webclient(v=vs.110).aspx>) or at a much lower level via the [HttpWebRequest](<https://msdn.microsoft.com/en-us/library/system.net.httpwebrequest(v=vs.110).aspx>). In 2009 and as part of the _REST Starter Kit (RSK)_ a new abstraction was born called the [HttpClient](<https://msdn.microsoft.com/en-us/library/system.net.http.httpclient(v=vs.118).aspx>); It took until the release of _.NET 4.5_ however for it to be available to the wider audience.

This abstraction provides easier methods of communicating with a HTTP server in full asynchrony as well as allowing to set default headers for each and every request. This is all awesome an all but there are dark secrets about this class that if not addressed can cause serious performance and at times mind boggling bugs! In this article we will explore the ~~problems~~ _subtleties_ that one would need to be aware of when working with the `HttpClient` class.

### So what's wrong?

The `HttpClient` class implements `IDisposable` suggesting any object of this type must be disposed of after use; With that in mind, let us have a look at how I would use this class assuming that I am not aware of the problem:

```csharp
Uri endpoint = new("http://localhost:1234/");
for (int i = 0; i < 10; i++)
{
    using HttpClient client = new();
    string response = await client.GetStringAsync(endpoint);
    Console.WriteLine(response);
}
```

So here we are sending 10 requests to an endpoint sequentially and assuming there is a listener serving the requests on port 1234 or any other endpoint you choose to hit, you will see 10 responses written to the `Console`; This is all going well, right? **WRONG!**

Let us run the command: `netstat -abn` on _CMD_ which should return (depending on the endpoint you hit):

```shell
...
TCP    [::1]:40968            [::1]:1234             TIME_WAIT
TCP    [::1]:40969            [::1]:1234             TIME_WAIT
TCP    [::1]:40970            [::1]:1234             TIME_WAIT
TCP    [::1]:40971            [::1]:1234             TIME_WAIT
TCP    [::1]:40972            [::1]:1234             TIME_WAIT
TCP    [::1]:40973            [::1]:1234             TIME_WAIT
TCP    [::1]:40975            [::1]:1234             TIME_WAIT
TCP    [::1]:40976            [::1]:1234             TIME_WAIT
TCP    [::1]:40977            [::1]:1234             TIME_WAIT
TCP    [::1]:40978            [::1]:1234             TIME_WAIT
...
```

What is this you ask? Well, this is showing us that our application has opened up 10 sockets to the server so **one** for **every request** but more importantly even though our application has now ended, the OS has 10 sockets still occupied and in the _TIME\_\_WAIT_ state.

This is due to the way [TCP/IP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) has been designed to work as connections are not closed immediately to allow the packets to arrive out of order or re-transmitted after the connection has been closed. _TIMEWAIT_ indicates that the local endpoint (the one on our side) has closed the connections but the connections are being kept around so that any delayed packets can be handled correctly. Once this happens the connections will then be removed after their timeout period of **4 minutes**. Remember, we sent 10 requests to the **same** endpoint and yet we have **10 individual sockets** still busy for at least 4 minutes!

The above example is an overly simplified one but have a look at this:

```csharp
public class ProductController : ApiController
{
    public async Task<Product> GetProductAsync(string id)
    {
        using HttpClient httpClient = new();
        string result = await httpClient.GetStringAsync("http://somewhere/api/...");
        return new Product { Name = result };
    }
}
```

Doing this per incoming request will eventually result in `SocketException`, don't believe me? Just run a load test, sit back and watch how many requests you can serve before you run out of sockets!

### What can we do?

Well, the first obvious thing that comes to mind is reusing our client instead of creating a new one for every request but as you will see later in the post, that **can cause yet another problem**. Before we get to that point, let us first find out if we can even re-use a single instance. Is the `HTTPClient` thread-safe? The answer is _YES_ it is, at least the following methods have been [documented](<https://msdn.microsoft.com/en-us/library/system.net.http.httpclient(v=vs.110).aspx#Anchor_5>) to be thread-safe:

- `CancelPendingRequests`
- `DeleteAsync`
- `GetAsync`
- `GetByteArrayAsync`
- `GetStreamAsync`
- `GetStringAsync`
- `PostAsync`
- `PutAsync`
- `SendAsync`

However the following are not thread-safe and cannot be changed once the first request has been made:

- `BaseAddress`
- `Timeout`
- `MaxResponseContentBufferSize`

In fact on the same documentation page, under the _Remarks_ section, it explains:

> _HttpClient_ is intended to be instantiated once and re-used throughout the life of an application. Instantiating an HttpClient class for every request will exhaust the number of sockets available under heavy loads. This will result in _SocketException_ errors.

Okay so is that it? Create and reuse a single instance of our client and happy days? Well, _NO!_ There is yet another very subtle but **serious** issue you may face.

### A Singleton HttpClient does not respect DNS changes

Re-using an instance of `HttpClient` means that it holds on to the socket until it is closed so if you have a DNS record update occurring on the server the client will never know until that socket is closed and let me tell you DNS records change for different reasons all the time, for example a fail-over scenario is just one of them (albeit in such case the connection/socket would have faulted and closed) or an _Azure_ deployment when you swap different instances e.g. _Production/Staging_ in this case your client would still be hitting the old instance! In fact there is an issue on the [dotnet/corefx](https://github.com/dotnet/corefx/issues/11224) repo about this behaviour.

`HTTPClient` (for valid reasons) does not check the DNS records when the connection is open so how can we fix this? One ~~naive~~ easy workaround is to set the _keep-alive_ header to `false` so the socket will be closed after each request, this obviously results in sub-optimal performance but if you do not care, then there is your answer; However, I think we can do better.

There is the lesser known [ServicePoint](<https://msdn.microsoft.com/en-us/library/system.net.servicepoint(v=vs.110).aspx>) class which holds the solution to our problem. This class is responsible for managing different properties of a TCP connection and one of such properties is the [ConnectionLeaseTimeout](<https://msdn.microsoft.com/en-us/library/system.net.servicepoint.connectionleasetimeout(v=vs.110).aspx>). This guy as its name suggests specifies how long (in ms) the TCP socket can stay open. By default the value of this property is set to **-1** resulting in the socket staying open indefinitely (relatively speaking) so all we have to do is set it to a more realistic value:

```csharp
ServicePointManager.FindServicePoint(endpoint)
    .ConnectionLeaseTimeout = (int)TimeSpan.FromMinutes(1).TotalMilliseconds;
```

The above override needs to be applied **once** for **each endpoint**. Note the method only cares about the _host_, _schema_ and _port_ everything else is ignored.

### Almost There...

So far we have taken care of force-closing the connections after a given period but that was just the first part. If our singleton client opens another connection it may still be pointed to the old server, why you ask? well all the DNS entries are cached which by default does not refresh for 2 minutes. So we also need to reduce the cache timeout which we can do by setting the `DnsRefreshTimeout` on the `ServicePointManager` class like so:

```csharp
ServicePointManager.DnsRefreshTimeout = (int)1.Minutes().TotalMilliseconds;
```

I wanted to have a better abstraction and not have to remember to do all of this on every request, I also wanted the abstraction to implement an interface for dependency injection between my services.

### RestClient

[RestClient](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/RestClient.cs) is a thread-safe wrapper around `HttpClient` and internally keeps a cache of endpoints that it has already sent a request to and if it is asked to send a request to an endpoint it does not have in its cache, it updates the `ConnectionLeaseTimeout` for that endpoint. Here is a simple usage example:

```csharp
// This is to show that IRestClient implements IDisposable
// just like HttpClient, you should not dispose it per request.
using IRestClient client = new RestClient();
await client.SendAsync(new HttpRequestMessage(HttpMethod.Get, new Uri("http://localhost/api")));
```

Now you can safely hold on to the client and/or register it using your favorite _IoC_ container and inject it where ever you require.

The class supports the same constructors as `HttpClient` and also provides a safe way of setting its default properties:

```csharp
var defaultHeaders = new Dictionary<string, string>
{
    {"Accept", "application/json"},
    {"User-Agent", "foo-bar"}
};

using IRestClient client = new RestClient(defaultHeaders, timeout: 15.Seconds(), axResponseContentBufferSize: 10);
client.DefaultRequestHeaders.Count.ShouldBe(defaultHeaders.Count);
client.DefaultRequestHeaders["Accept"].ShouldBe("application/json");
client.DefaultRequestHeaders["UserAgent"].ShouldBe("foo-bar");

client.Endpoints.ShouldBeEmpty();
client.MaxResponseContentBufferSize.ShouldBe((uint)10);
client.Timeout.ShouldBe(15.Seconds());
```

The code is on [GitHub](https://github.com/NimaAra/Easy.Common/blob/master/Easy.Common/RestClient.cs) and is available on [NuGet](https://www.nuget.org/packages/Easy.Common/) as part of the [Easy.Common](https://github.com/NimaAra/Easy.Common) library used in my other projects.

### Update 2019

Starting from _.NET Core 2.1_, Microsoft addressed some of the issues covered in this article by making available [HttpClientFactory](https://docs.microsoft.com/en-us/dotnet/standard/microservices-architecture/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests). Despite the various features offered in this class, in my opinion, there is a little too much of ceremony involved with using this type also we would still need to deal with setting the DNS refresh timeout ourselves; Therefore, I still prefer to use `RestClient` in my projects.

`HttpClient` was also overhauled in _.NET Core 2.1_ with a rewritten _HttpMessageHandler_ called [SocketsHttpHandler](https://docs.microsoft.com/en-us/dotnet/core/whats-new/dotnet-core-2-1) which results in significant performance improvements it also introduces the [PooledConnectionLifetime](https://apisof.net/catalog/System.Net.Http.SocketsHttpHandler.PooledConnectionLifetime) property which allows us to set the connection timeout without having to set the `ConnectionLeaseTimeout` for each endpoint.

As of version [3.0.0 of Easy.Common](https://www.nuget.org/packages/Easy.Common/3.0.0), the `RestClient` no longer needs to set the `ConnectionLeaseTimeout` when running on _.NET Core 2.1_ or higher.

Have fun and happy **REST**ing.

#### BTW

This post was inspired by [YOU'RE USING HTTPCLIENT WRONG AND IT IS DESTABILIZING YOUR SOFTWARE](http://aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/) by [Simon Timms](https://twitter.com/stimms) and [Singleton HttpClient](http://byterot.blogspot.co.uk/2016/07/singleton-httpclient-dns.html) by [Ali Ostad](http://twitter.com/aliostad) as well as various great posts by [Darrel Miller](http://stackoverflow.com/users/6819/darrel-miller).
