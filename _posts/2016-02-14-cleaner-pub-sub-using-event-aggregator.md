---
title: "Cleaner Pub-Sub using the Event Aggregator Pattern"
tags: "C# .NET PubSub Messaging"
redirect_from:
  - /cleaner-pub-sub-using-the-event-aggregator-pattern/
  - /2016/02/14/cleaner-pub-sub-using-the-event-aggregator-pattern/
---

## Cleaner Pub-Sub using the Event Aggregator Pattern

If you have been working with .NET events then you should be familiar with the following pattern (overly simplified for brevity):

```csharp
public sealed class OrderSubscriber
{
    private readonly IOrderPublisher _publisher;

    public OrderSubscriber(IOrderPublisher publisher)
    {
        _publisher = publisher;
        _publisher.NewOrder += OnNewOrder;
    }

    private void OnNewOrder(object sender, OrderPlacedEventArgs eArgs)
    {
        /* do something with the order */
    }
}

public sealed class OrderPublisher : IOrderPublisher
{
    public event EventHandler<OrderPlacedEventArgs> NewOrder;

    public void Publish()
    {
        var snapshot = NewOrder;
        if (snapshot == null) { return; }

        snapshot(this, new OrderPlacedEventArgs());
    }
}

public sealed class OrderPlacedEventArgs : EventArgs { ... not really important }
```

What we have here is a simple publisher publishing an _OrderPlaced_ event and a subscriber which handles the orders when the events are published. I have seen this pattern being used in many code bases and is pretty much the standard way of writing pub-sub applications in the enterprise. However, there are a couple of things that I do not like about this approach.

1. The main problem is the direct coupling between the publisher and the subscriber meaning as the application grows, we will not be able to introduce new subscribers, publishers or events without having to break a few places.

2. We have to inherit from _EventArgs_ for our _OrderPlaced_ event; This restriction was removed in _.NET 45_ but _.NET 4_ is still strong in many enterprises plus I think we can do better.

3. We also have to worry about NewOrder delegate not being null and making sure there is at least one subscriber before publishing. There are many ways to make that part cleaner (specially in _C# 6_) but why do this when we can do better!

So how can we improve what we currently have?

### Event Aggregator

An Event Aggregator is a service which sits between your publishers and subscribers acting as an intermediary pushing your messages (events) from one entity to another.

[Martin Fowler](http://martinfowler.com/eaaDev/EventAggregator.html) describes it as:

> An Event Aggregator is a simple element of indirection. In its simplest form you have it register with all the source objects you are interested in, and have all target objects register with the Event Aggregator. The Event Aggregator responds to any event from a source object by propagating that event to the target objects.

Here is a diagram to make it even more clear:

<p style="text-align: center;">
<img alt="diagram of event aggregator" src="https://i.imgur.com/adqe1m4.png">
</p>

So what this means is that your publishers and subscribers are no longer coupled together and they only need to hold a reference to the _Event Aggregator_.

### Easy Message Hub

I hope by now you accept the benefits that the _Event Aggregator_ pattern can bring to your publishers and subscribers. I very much like this pattern and how it can result in a much cleaner code-base. In my journey to find a suitable implementation in C#, I found a number of candidates:

- [Event Aggregator (Prism)](https://prismlibrary.com/docs/event-aggregator.html)
- [Async Event Aggregator](https://github.com/timothy-makarov/AsyncEventAggregator) by Timothy Makarov
- [Event Aggregator with Reactive Extensions](https://github.com/shiftkey/Reactive.EventAggregator) by José F. Romaniello

But they either lacked the optimal performance or the kind of API I was hoping for; So I came up with the [Easy.MessageHub](https://github.com/NimaAra/Easy.MessageHub).

The API surface-area is very simple, there is the `MessageHub` representing the _Event Aggregator_ which can be registered as an `IMessageHub` using your favourite _IoC_ framework and injected into your publishers, subscribers or anyone else who might be interested in the messages coming in or going out.

In it's simplest form one could publish using:

```csharp
MessageHub hub = new();
// OrderPlaced and OrderDeleted both inherit from Order
hub.Publish(new OrderPlaced());
hub.Publish(new OrderDeleted());

// let us send a string message on the same hub
hub.Publish("A Very Important Message!");
```

And for the subscription:

```csharp
Action<Order> onNewOrder = o => { /* do whatever you want with the Order */ };
Guid token = hub.Subscribe(onNewOrder);
```

To Un-Subscribe:

```csharp
// By Token
hub.Unsubscribe(token);
hub.IsSubscribed(token); // returns false

// Clear all subscriptions
hub.ClearSubscriptions();
```

There is also the concept of a _global event handler_ allowing you to have a single hook into every message that is published by the hub perhaps for the purpose of logging, audit etc.

```csharp
Action<Type, object> logHandler = (type, msg) => logger.Log("Type of message: {Type}, Message: {Message}", type, msg);
hub.RegisterGlobalHandler(logHandler);
```

And finally if any of the subscribers throw an exception while receiving a message, instead of breaking all the others you can subscribe to the error by:

```csharp
Action<Guid, Exception> errorHandler = (token, e) => logger.Log("Token: {Token}, Error: {Error}", token, e);
hub.RegisterGlobalErrorHandler(errorHandler);
```

Now armed with our new friend, we can rewrite the original example as:

```csharp
public sealed class OrderSubscriber
{
    private readonly IMessageHub _hub;
    private readonly Guid _subscriptionToken;

    public OrderSubscriber(IMessageHub hub)
    {
        _hub = hub;
        _subscriptionToken = _hub.Subscribe<Order>(OnNewOrder);
    }

    public void Unsubscribe() => _hub.Unsubscribe(_subscriptionToken);

    private void OnNewOrder(Order order) => /* do something with the order */
}

public sealed class OrderPublisher
{
    private readonly IMessageHub _hub;

    public OrderPublisher(IMessageHub hub) => _hub = hub;

    public void Publish() => _hub.Publish(new OrderPlaced());
}

public class Order { }
public sealed class OrderPlaced : Order { }
```

Note in this version:

- Publishers and Subscribers no longer hold a reference to each other therefore reducing coupling.
- Any class can now be an event e.g. `Order`, `OrderPlaced`.
- We are no longer limited to using .NET events or `EventArg`.
- `MessageHub` is mockable therefore publisher and subscriber can easily be tested.
- We do not need to check if the publisher has any subscribers before publishing, we just fire the message and `MessageHub` takes care of the rest.

The source together with some benchmarking and usage examples are available on [GitHub](https://github.com/NimaAra/Easy.MessageHub) and NuGet package available [HERE](https://www.nuget.org/packages/Easy.MessageHub).

Good luck and happy messaging!
