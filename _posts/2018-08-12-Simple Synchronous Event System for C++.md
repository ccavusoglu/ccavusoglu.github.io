---
title: Simple Synchronous Event System for C++
layout: single
excerpt: "Simple synchronous event system implementation using C++ STL"
date: "2018-08-12"
category: posts
tags:
 - c++
 - observer
 - event
 - subscribe
 - publish
 - synchronous
published: true
---

Here I'll show a basic synchronous event mechanism based on Publish/Subscribe pattern. Implementation is made to be used in a single threaded personal 2D game engine. Note that it heavily depends on polymorphism and STL. Also it's not **thread safe**. I didn't do a benchmark as I use it for simple tasks.

**The example demonstrated here are available at: <https://github.com/ccavusoglu/CppEvents>**

At core there are *IEventCallback* and *EventCallback*. Generic *EventCallback* is responsible for holding necessary callback function pointer.

```c++
class IEventCallback {

public:
    virtual EventType getEventType() const = 0;
    virtual void operator()(IEventBase const &event) const = 0;
    virtual bool operator==(IEventCallback *other) = 0;
    virtual ~IEventCallback() = default;
};

template<class T>
class EventCallback : public IEventCallback {

    T const *instance;
    EventType eventType;
    void (T::*callback)(IEventBase const &) const;

public:
    EventCallback(T const *instance, EventType eventType, void (T::*function)(IEventBase const &event) const)
            : instance(instance), eventType(eventType), callback(function) {}

    EventType getEventType() const override {
        return eventType;
    }

    void operator()(IEventBase const &event) const override {
        (instance->*callback)(event);
    }

    bool operator==(IEventCallback *other) override {
        auto *otherEventCallback = dynamic_cast<EventCallback *>(other);

        if (otherEventCallback == nullptr) return false;

        return (this->callback == otherEventCallback->callback) &&
               (this->instance == otherEventCallback->instance);
    }
};
```

*EventManager* stores subscribed actions in an *unordered_map* with *EventType* and *std::vector<const IEventCallback *>* pair. Instead of directly manipulating *unordered_map*, event additions and removals are stored in separate queues and processed when an event is fired. This way the vector holding stored actions guaranteed to not being modified when consuming an event. *EventManager::subscribe* creates and returns *EventCallback<<T>>*. The same pointer can be used when unsubscribing.

```c++
class EventManager {

    typedef std::unordered_map<int, std::vector<IEventCallback const *>> Subscribers;
    typedef std::queue<IEventCallback const *> SubscriberQueue;

    Subscribers subscribers;
    SubscriberQueue subscribersToAdd;
    SubscriberQueue subscribersToRemove;

private:
    void checkAddQueue();
    void checkRemoveQueue();

public:
    ~EventManager();

    template<class T>
    EventCallback<T> *subscribe(T const *instance, EventType eventType,
                                void (T::*function)(IEventBase const &event) const) {
        auto action = new EventCallback<T>(instance, eventType, function);

        subscribersToAdd.push(action);

        return action;
    }

    void unSubscribe(IEventCallback const *action);
    void fire(EventType type, IEventBase const &event);
};
```

Events implement *IEventBase*.

```c++
class IEventBase {

public:
    virtual EventType getEventType() const = 0;
    virtual ~IEventBase() = default;
};
```

Example events.

```c#
struct GetUserEvent : public IEventBase {

    std::string userName;

    explicit GetUserEvent(std::string userName) : userName(std::move(userName)) {}

    EventType getEventType() const override {
        return GET_USER;
    }
};

struct GetUserSettingsEvent : public IEventBase {

    std::string userType;

    explicit GetUserSettingsEvent(std::string userType) : userType(std::move(userType)) {}

    EventType getEventType() const override {
        return GET_USER_SETTINGS;
    }
};
```

Usage:

```c++
EventManager *eventManager;

struct Publisher {
    std::string userName;
    std::string userType;

    Publisher(std::string userName, std::string userType) : userName(std::move(userName)),
                                                            userType(std::move(userType)) {}

    void publishGetUser() {
        auto eventType = EventType::GET_USER;
        std::cout << "Event: " << EventTypes[eventType] << " fired: " << userName << std::endl;

        eventManager->fire(eventType, GetUserEvent(userName));
    }

    void publishGetUserSettings() {
        auto eventType = EventType::GET_USER_SETTINGS;
        std::cout << "Event: " << EventTypes[eventType] << " fired: " << userType << std::endl;

        eventManager->fire(eventType, GetUserSettingsEvent(userType));
    }
};

struct Subscriber {

    std::string name;

    explicit Subscriber(std::string name) : name(std::move(name)) {}

    void gotUser(IEventBase const &event) const {
        auto &ev = dynamic_cast<GetUserEvent const &>(event);

        std::cout << name << " handled: " << EventTypes[ev.getEventType()] << ": " << ev.userName << std::endl;
    }

    void gotUserSettings(IEventBase const &event) const {
        auto &ev = dynamic_cast<GetUserSettingsEvent const &>(event);

        std::cout << name << " handled: " << EventTypes[ev.getEventType()] << ": " << ev.userType << std::endl;
    }

    EventCallback<Subscriber> *subscribeGetUser() {
        return eventManager->subscribe(this, EventType::GET_USER, &Subscriber::gotUser);
    }

    EventCallback<Subscriber> *subscribeGetUserSettings() {
        return eventManager->subscribe(this, EventType::GET_USER_SETTINGS, &Subscriber::gotUserSettings);
    }
};

int main() {
    eventManager = new EventManager();

    auto subscriber = Subscriber("subscriber");
    auto subscriberOther = Subscriber("subscriberOther");
    auto publisher = Publisher("QQ", "Q_TYPE");
    auto event = subscriber.subscribeGetUser();
    auto eventOther = subscriberOther.subscribeGetUser();
    subscriber.subscribeGetUserSettings();
    publisher.publishGetUser();

    eventManager->unSubscribe(event);
    eventManager->unSubscribe(eventOther);
    publisher.publishGetUser();
    publisher.publishGetUserSettings();

    std::cin.ignore();

    delete eventManager;

    return 0;
}
```

Output:

```
Event: GET_USER fired: QQ
subscriber handled: GET_USER: QQ
subscriberOther handled: GET_USER: QQ
Event: GET_USER fired: QQ
No subscription for event: GET_USER
Event: GET_USER_SETTINGS fired: Q_TYPE
subscriber handled: GET_USER_SETTINGS: Q_TYPE
```

That's it. I highly welcome any kind of suggestion. Thanks!