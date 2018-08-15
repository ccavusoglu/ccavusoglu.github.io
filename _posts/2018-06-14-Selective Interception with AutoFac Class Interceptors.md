---
title: Customized AutoFac Class Interceptors
layout: single
excerpt: "Using AutoFac and Castle Dynamic proxy to intercept based on method attribute"
date: "2018-06-14"
category: posts
tags:
 - csharp
 - .net
 - autofac
 - interceptor
 - castle
 - dynamicproxy
published: true
---

AutoFac is a simple and very easy to use DI container. Below I'll show how to use *Castle.DynamicProxy* to intercept concrete implementations customized with method attributes.

**AutoFac** and **AutoFac.Extras.DynamicProxy** packages are needed and they both are available in NuGet.

**AutoFac** class interceptors work by extending the intercepted class. So it's mandatory to specify methods *virtual* in order to make it work. However you (like I am) may not want your all virtual methods to be intercepted by the registered interceptor. My solution to this is pretty straightforward. 

I've created a custom attribute and marked my virtual methods. I've also specified a parameter to further control it's behavior.

```c#
public class ExecutionTimeLogAttribute : Attribute
{
    public long Threshold;

    public ExecutionTimeLogAttribute(long thresholdMs) { Threshold = thresholdMs; }

    public ExecutionTimeLogAttribute() { Threshold = 0; }
}
```

Class to be intercepted:

```c#
public class ClassToIntercept
{
    [ExecutionTimeLog]
    public virtual void InterceptedMethod() { }

    [ExecutionTimeLog(1000)]
    public virtual void InterceptedWThresholdMethod(int timeout)
    {
        try
        {
            Thread.Sleep(timeout);
        }
        catch (Exception e)
        {
            Console.WriteLine(e);
        }
    }

    public virtual void NonInterceptedMethod() { }
}
```

Container registration looks like follows:

```c#
var builder = new ContainerBuilder();

builder.RegisterType<ExecutionTimeLogInterceptor>().SingleInstance();
builder.RegisterType<ClassToIntercept>()
    .EnableClassInterceptors()
    .InterceptedBy(typeof(ExecutionTimeLogInterceptor))
    .SingleInstance();

var container = builder.Build();
```

And the interceptor itself:

```c#
public class ExecutionTimeLogInterceptor : IInterceptor
{
    public void Intercept(IInvocation invocation)
    {
        if (!(invocation.Method.GetCustomAttributes(typeof(ExecutionTimeLogAttribute), false).FirstOrDefault()
            is ExecutionTimeLogAttribute perfAttribute))
        {
            invocation.Proceed();
            return;
        }

        var e = LogExecutionTime.Begin();

        invocation.Proceed();

        e.End(invocation.Method.DeclaringType.Name, invocation.Method.Name, perfAttribute.Threshold);
    }
}
```

I've always get suspicious when there's an instance check logic. However, in this case, this is the simplest solution and I couldn't think of a case where this may fail.

*GetCustomAttributes* method's second parameter *inherit* indicates whether it should get the base method's attributes or not. If you also want to intercept the overridden methods it's better to specify custom attribute also for them instead of setting this true.

Lastly, *LogExecutionTime* class is a simple timer wrapper.

```c#
public class LogExecutionTime
{
    private readonly Stopwatch timer = new Stopwatch();

    private LogExecutionTime() { }

    public static LogExecutionTime Begin()
    {
        var timerObject = new LogExecutionTime();

        timerObject.Start();

        return timerObject;
    }

    public void Start() { timer.Start(); }

    public void End([CallerFilePath] string callerClass = "", [CallerMemberName] string methodName = "",
        long threshold = 0)
    {
        var elapsedMilliseconds = timer.ElapsedMilliseconds;

        timer.Reset();

        if (elapsedMilliseconds >= threshold)
        {
            Console.WriteLine($"{callerClass}::{methodName} executed In: {elapsedMilliseconds}ms");
        }
    }
}
```

Test and result:

```c#
var objToIntercept = container.Resolve<ClassToIntercept>();

objToIntercept.InterceptedMethod();
objToIntercept.InterceptedWThresholdMethod(500);
objToIntercept.InterceptedWThresholdMethod(1500);
objToIntercept.NonInterceptedMethod();
```

```
ClassToIntercept::InterceptedMethod executed In: 0ms
ClassToIntercept::InterceptedWThresholdMethod executed In: 1500ms
```

Thanks for reading. As always, any suggestion is highly appreciated!