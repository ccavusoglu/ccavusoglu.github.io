---
title: Simple Synchronous Event System for C#
layout: single
excerpt: "A simple thread safe event system implementation."
date: "2018-03-08"
category: posts
tags:
 - csharp
 - .net
 - observer
 - event
 - subscribe
 - publish
 - synchronous
published: true
---

.NET's event/delegate mechanism is sometimes hard to manage and could create unnecessary complexity. I've implemented a simple event system which I use almost every project I work on.

**The example demonstrated here are available at: <https://github.com/ccavusoglu/SimpleSynchronousEvents>**

The implementation contains an interface *IEventBase,* a manager *EventManager* to store observers and *Events* implementing *IEventBase* interface.

*EventManager* pretty much self explanatory. It stores *Actions* and *Type* pairs and invokes subscribed *Actions* when an event is fired. It's thread safe by utilizing .NET's *ConcurrentDictionary* and *ImmutableList*. If you think performance overhead of *ImmutableList* isn't neglectable in your use case, using a *List* and wrapping list operations with a lock can be preferred. Refer to this article <https://msdn.microsoft.com/en-us/magazine/mt795189.aspx> for a detailed analysis of *Immutable* types.

```c#
public class EventManager
{
    private static EventManager Instance;
    private readonly ConcurrentDictionary<Type, ImmutableList<Action<IEventBase>>> subscribers;

    private EventManager()
    {
        subscribers = new ConcurrentDictionary<Type, ImmutableList<Action<IEventBase>>>();
    }

    public static EventManager GetInstance()
    {
        return Instance ?? (Instance = new EventManager());
    }

    public void Fire<T>(T obj) where T : IEventBase
    {
        var type = typeof(T);

        if (subscribers.TryGetValue(type, out var actions))
        {
            foreach (var action in actions)
                action.Invoke(obj);
        }
        else
        {
            Console.WriteLine($"There isn't any subscription for event type: {type}");
        }
    }

    public Action<IEventBase> Subscribe<T>(Action<IEventBase> callback) where T : IEventBase
    {
        var type = typeof(T);

        if (!subscribers.TryGetValue(type, out var actions))
        {
            actions = ImmutableList<Action<IEventBase>>.Empty;
            subscribers.TryAdd(type, actions);
        }
        else
        {
            actions = subscribers[type];
        }

        if (!actions.Contains(callback))
            actions = actions.Add(callback);

        subscribers[type] = actions;

        return callback;
    }

    public void UnSubscribe<T>(Action<IEventBase> callback) where T : IEventBase
    {
        var type = typeof(T);

        if (!subscribers.TryGetValue(type, out var actions)) return;

        if (actions.Contains(callback))
            actions = actions.Remove(callback);

        subscribers[type] = actions;
    }
}
```

*IEventBase* is an empty interface. *Events* implement *IEventBase*.

```c#
public interface IEventBase
{
    
}
```

Example events.

```c#
public class SomeConcreteEvent : IEventBase
{
    public string Param { get; set; }
}

public class AnotherConcreteEvent : IEventBase
{
    public string Param { get; set; }
}
```

Usage:

```c#
// subscriber
eventManager.Subscribe<SomeConcreteEvent>(o => 
	Console.WriteLine($"SomeConcreteEvent handled in SubscriberClass: {((SomeConcreteEvent)o).Param}"));

// publisher
eventManager.Fire(new SomeConcreteEvent
{
    Param = "Event fired from PublisherClass::DoWork"
});
```

Usage with UnSubscribe:

```c#
// subscriber
anotherConcreteEventCallback =
    eventManager.Subscribe<AnotherConcreteEvent>(o => HandleAnotherConcreteEvent((AnotherConcreteEvent)o));

private void HandleAnotherConcreteEvent(AnotherConcreteEvent anotherConcreteEvent)
{
    eventManager.UnSubscribe<AnotherConcreteEvent>(anotherConcreteEventCallback);

    Console.WriteLine($"AnotherConcreteEvent handled in SubscriberClass: {anotherConcreteEvent.Param}");
}

// publisher
eventManager.Fire(new AnotherConcreteEvent
{
    Param = "Event fired from AnotherClass::DoAnotherWork"
});

```

That's it. Simple and loosely coupled solution for a common object communication problem.