---
layout: post
title:  "Designing .NET Libraries with developers in mind"
date:   2021-06-28 12:00:00 +0100
tags: dotnet .NET NuGet Library
---

Working in a distributed system, one often turns to packaging common code into a reusable library. At its simplest, that could just be putting the classes we want to share in a library project, publishing it as a NuGet package and to be consumed where needed.

Job done, we have achieved reusability! But should we put more thought into it?

Yes! As a library developer, we have a responsibility to our users (other developers) to provide a library that is easy, and also desirable, to use. In this article we will walk through the creation and design of a library project, discussing things we should consider, and patterns we can adopt to create a simple, yet customizable, integration experience for our users.

* TOC
{:toc}

## What Is The Purpose Of The Library?

Before we get started - what are we doing?

It may sound obvious, but especially when looking to share existing code across applications, we should make sure we are creating a module with a clear purpose.

If a library’s purpose is not well defined, over time it can become a mixed bag of unrelated classes, with an unwieldy collection of dependencies. So called “common” internal libraries like this are unfortunately all too, well, common. They force consumers to depend on packages they do not need, increasing the chance of version conflicts when building, and in some cases even restricting the version of .NET that must be targeted.

When designing a library, think in similar terms to designing a microservice; a library should have a well defined area of responsibility. With a library of such high cohesion, a more descriptive and appropriate name than “common” should be apparent.

For the purposes of this article, we will be creating a library for starting fire-and-forget background tasks. Note that the implementation is not appropriate for production scenarios, it is kept simple in order to focus on the design of the library.

With that clear purpose in mind, we can give our library an appropriate (and cool) name like [`BackgroundTaskr`](https://github.com/Haydabase/BackgroundTaskr).

## Creating the Project
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/14b529e1ed6678d0406790b27ae9ee30ff14a728){:target="_blank"}{: .right}

First thing we need is a class library project to put the code.

So we’ll need to decide on the .NET version we are going to target. Here there is a trade off between compatibility, and availability of features. Given the current state of .NET, it can be a little confusing, but [this article](https://devblogs.microsoft.com/dotnet/the-future-of-net-standard/#what-you-should-target) summarizes our choice down to 3 targets:
- .NET Standard 2.0 - to share code between .NET Framework and all other platforms.
- .NET Standard 2.1 - to share code between Mono, Xamarin, and .NET Core 3.x.
- .NET 5.0 - for code sharing moving forward.
.NET 5.0 may seem like the right way to go in future, but there are many existing applications still using older flavours of .NET. We should not make upgrading the version of .NET a prerequisite to using our library, if it can be reasonably avoided.

So for our project, we’ll start off targeting .NET Standard 2.0, which is actually still the [official guidance](https://devblogs.microsoft.com/dotnet/the-future-of-net-standard/). We can revisit this decision if we find we need to use newer features, or wish to depend on other libraries which force our hand.

## Open Source?

We may only wish to share our library within our own organisation, but even so, it is often a good exercise to explore if it would be suitable for open-sourcing.

_Would the library we are creating expose application/organisation specific logic if it were open-sourced?_
_If we have existing code, would we be happy to open-source it as is?_

This thought process might help identify domain specific aspects of our library that would restrict its use in other applications, even within our own organisation. It could perhaps lead to a more general purpose solution, enabling future as yet unforeseen use cases. It may also highlight areas of improvement in existing code from a readability, structure, or test coverage perspective.

Say we already have an implementation of a background task factory we are going to use as a basis for our library, with the following task creation method:
{% highlight csharp %}
public void CreateBackgroundTask(string name, Func<IServiceProvider, Task> runTask)
{
    Task.Run(async () =>
    {
        // Must use newly resolved dependencies in our background task to avoid accessing disposed scoped services.
        using var scope = _serviceProvider.CreateScope();
        try
        {
            await runTask(scope.ServiceProvider);
        }
        catch (Exception exception)
        {
            await scope.ServiceProvider.GetRequiredService<ISlackNotifier>().NotifyTaskFailure(name, exception);
        }
    });
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/8ab329d1b728eada64ad6997d36cf5d7070d714b/ExistingCodebase/BackgroundTaskFactory.cs){:target="_blank"}{: .right}

Here we are notifying slack on failure of a task, which is obviously an organisational (if not application) specific piece of logic. If open sourcing, we would certainly not want such logic to be included in our library. Even within our own organisation, there could be use cases where we may not want that behaviour for all tasks. So in our library we will not include this behaviour, but instead design for [extensibility](#extensibility), allowing for such behaviours to be injected.

## Designing the library interface

### Usage Interface
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/18f52cdefc8250827e2a4141efa119e9785e74a7){:target="_blank"}{: .right}

Given good developers test their code, we can assume users of our library will want to code against an interface for creating background tasks, to allow for injecting a different implementation in unit tests. Exposing our task factory as an interface also enables useful patterns like decorators, giving the potential for some extensibility without any effort on our part.

So we’ll define our interface as:
{% highlight csharp %}
public interface IBackgroundTaskr
{
   void CreateBackgroundTask(string name, Func<IServiceProvider, Task> runTask);
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/18f52cdefc8250827e2a4141efa119e9785e74a7/Haydabase.BackgroundTaskr/IBackgroundTaskr.cs){:target="_blank"}{: .right}

And for reference, the initial implementation:
{% highlight csharp %}
internal class BackgroundTaskFactory : IBackgroundTaskr
{
   private readonly IServiceProvider _serviceProvider;

   public BackgroundTaskFactory(IServiceProvider serviceProvider)
   {
       _serviceProvider = serviceProvider;
   }

   public void CreateBackgroundTask(string name, Func<IServiceProvider, Task> runTask)
   {
       Task.Run(async () =>
       {
           try
           {
               // Must use newly resolved dependencies in our background task to avoid accessing disposed scoped services.
               using var scope = _serviceProvider.CreateScope();
               await runTask(scope.ServiceProvider);
           }
           catch
           {
           }
       });
   }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/18f52cdefc8250827e2a4141efa119e9785e74a7/Haydabase.BackgroundTaskr/BackgroundTaskFactory.cs){:target="_blank"}{: .right}

### Minimal Surface Area

Note that we keep the above class `internal`, as we do not expect it to be instantiated directly (see next section…). This is good practice in general for any class that is not required to be directly referenced when using the library.

Any public type in our library that we wish to change, could result in a compile time break for consumers when upgrading. Therefore we should minimise the surface area of the library interface as much as is practicable, to allow us to safely make changes to the implementation, knowing no consumers are referencing it.

So as we build out our library, we’ll mark each new type as `internal`, unless or until we find a need to make it `public`.

### Registration Interface
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/9dc163501e907ce87c691599060838e5eb3bda5b){:target="_blank"}{: .right}

A good library works out of the box, with minimal to no configuration. An intuitive interface, with little effort to start using it, lowers the barrier to adoption.

Many official .NET libraries make use of the built in [dependency injection](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection) support, and developers are accustomed to registering libraries with extension methods on the `IServiceCollection`. So we’ll design our library to be registered in a similarly intuitive way:
{% highlight csharp %}
services.AddBackgroundTaskr();
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/9dc163501e907ce87c691599060838e5eb3bda5b/DemoApp/Startup.cs#L32){:target="_blank"}{: .right}

Initially implemented as such:
{% highlight csharp %}
namespace Microsoft.Extensions.DependencyInjection
{
   public static class BackgroundTaskrServiceCollectionExtensions
   {
       public static void AddBackgroundTaskr(this IServiceCollection services)
       {
           services.TryAddSingleton<IBackgroundTaskr, BackgroundTaskFactory>();
       }
   }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/9dc163501e907ce87c691599060838e5eb3bda5b/Haydabase.BackgroundTaskr/BackgroundTaskrServiceCollectionExtensions.cs){:target="_blank"}{: .right}

There are a couple of things to note here:
- We put this class in the `Microsoft.Extensions.DependencyInjection namespace`, as with other libraries’ registration methods, making the method more easily discoverable by intellisense.
- We use `TryAddSingleton` which prevents multiple registrations if our method is called multiple times by mistake.

We now have a basic version of our library coded up that we could publish, but remember we needed a way to inject custom behaviour for our slack notification use case...

## Extensibility

Libraries often need to be extensible, allowing developers to tailor the behaviour to their specific use case. There are a couple of patterns in the .NET ecosystem that we can use to allow our libraries behaviour to be customized:
- [Registration Builder](#registration-builder) - for example, adding extra or custom behaviours
- [Configuration Options](#configuration-options) - for example, configuring settings or toggling features on/off, but also can be used to inject behaviour via funcs/actions

In practice, many libraries will use a combination of the above approaches, and so will we...

## Middleware
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/0f6fc745e0839054c3af90eebee4411b076d8925){:target="_blank"}{: .right}

One of the most common .NET libraries with a rich builder interface is `HttpClient`. This allows for much configuration, including adding [outgoing request middleware](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-5.0#outgoing-request-middleware) (`DelegatingHandler`):
{% highlight csharp %}
services.AddHttpClient("some-service")
    .AddHttpMessageHandler<SomeMiddlewareHandler>();
{% endhighlight %}

We will use a similar approach in our library, allowing middlewares to be added around the running of background tasks. Firstly let’s define a middleware interface:
{% highlight csharp %}
public interface IBackgroundTaskrMiddleware
{
   public Task OnNext(string name, Func<Task> next);
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/0f6fc745e0839054c3af90eebee4411b076d8925/Haydabase.BackgroundTaskr/IBackgroundTaskrMiddleware.cs){:target="_blank"}{: .right}

We will need to chain multiple middlewares together, so we will define a class to do so. Each middleware needs to be invoked with a next func which invokes the next middleware in the chain, or runs the background task if it is the innermost middleware. If we knew there were always 2 middleware, an implementation might look something like this:
{% highlight csharp %}
internal sealed class MiddlewareInvoker : IMiddlewareInvoker
{
   private readonly IReadOnlyList<IBackgroundTaskrMiddleware> _middlewares;

   public MiddlewareInvoker(IEnumerable<IBackgroundTaskrMiddleware> middlewares)
   {
       _middlewares = middlewares.ToList();
   }

   public Task InvokeAsync(string name, Func<Task> runTask) =>
       _middlewares[0].OnNext(name, () => _middlewares[1].OnNext(name, runTask));
}
{% endhighlight %}


Now we’ve figured out a simple case, we can see how to extrapolate to the general case, using [LINQ](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/query-syntax-and-method-syntax-in-linq) to do the heavy lifting: reversing the list and using the [`Aggregate`](https://docs.microsoft.com/en-us/dotnet/api/system.linq.enumerable.aggregate?view=netstandard-2.0) method to build our middleware pipeline from the innermost layer outwards:
{% highlight csharp %}
internal sealed class MiddlewareInvoker : IMiddlewareInvoker
{
   private readonly IReadOnlyList<IBackgroundTaskrMiddleware> _middlewaresInReverse;

   public MiddlewareInvoker(IEnumerable<IBackgroundTaskrMiddleware> middlewares)
   {
       _middlewaresInReverse = middlewares.Reverse().ToList();
   }

   public Task InvokeAsync(string name, Func<Task> runTask) =>
       _middlewaresInReverse.Aggregate(runTask, (next, middleware) => () => middleware.OnNext(name, next))();
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/0f6fc745e0839054c3af90eebee4411b076d8925/Haydabase.BackgroundTaskr/MiddlewareInvoker.cs){:target="_blank"}{: .right}

Note that we will leverage the fact that .NET dependency injection automatically resolves all registered `T` ([in order of registration](https://docs.microsoft.com/en-us/dotnet/core/extensions/dependency-injection#service-registration-methods)) when injecting an `IEnumerable<T>`.

We now must register the `MiddlewareInvoker` in our registration method, so we can resolve and use it:
{% highlight csharp %}
services.TryAddScoped<IMiddlewareInvoker, MiddlewareInvoker>();
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/0f6fc745e0839054c3af90eebee4411b076d8925/Haydabase.BackgroundTaskr/BackgroundTaskrServiceCollectionExtensions.cs#L12){:target="_blank"}{: .right}

Then we update our `BackgroundTaskFactory` to use it, by adding tweaking how we invoke the `runTask` func as such:
{% highlight csharp %}
var invoker = scope.ServiceProvider.GetRequiredService<IMiddlewareInvoker>();
await invoker.InvokeAsync(name, () => runTask(scope.ServiceProvider));
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/0f6fc745e0839054c3af90eebee4411b076d8925/Haydabase.BackgroundTaskr/BackgroundTaskFactory.cs#L24){:target="_blank"}{: .right}

## Registration Builder
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/2661be4f74424955f85c5b96a1c6849a250a76bd){:target="_blank"}{: .right}

Now we need a way of registering middlewares, and for that we will use the builder pattern. Firstly, we’ll define an interface for our builder:
{% highlight csharp %}
public interface IBackgroundTaskrBuilder
{
   public IServiceCollection Services { get; }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/2661be4f74424955f85c5b96a1c6849a250a76bd/Haydabase.BackgroundTaskr/IBackgroundTaskrBuilder.cs){:target="_blank"}{: .right}

All we really need here is a reference to the `IServiceCollection` so we can add more registrations to it. If we had a more complicated library where we could add more than one instance of something (e.g. `HttpClient`) we’d likely also have a `Name` property here to indicate the instance the builder is related to.
Note that we also do not define any methods on this interface, instead all builder functionality will be provided through extension methods. Since adding a method to an interface would be a compile time break, using extension methods here allows us to add more functionality in future without a breaking change.

So we define a few extension methods to allow registering middlewares as such:
{% highlight csharp %}
public static IBackgroundTaskrBuilder UsingMiddleware(
   this IBackgroundTaskrBuilder builder,
   Func<IServiceProvider, IBackgroundTaskrMiddleware> resolveMiddleware)
{
   builder.Services.AddScoped<IBackgroundTaskrMiddleware>(resolveMiddleware);
   return builder;
}

public static IBackgroundTaskrBuilder UsingMiddleware<TMiddleware>(this IBackgroundTaskrBuilder builder)
   where TMiddleware : IBackgroundTaskrMiddleware =>
   builder.UsingMiddleware(x => x.GetRequiredService<TMiddleware>());

public static IBackgroundTaskrBuilder UsingMiddleware(
   this IBackgroundTaskrBuilder builder,
   Func<string, Func<Task>, Task> onNext) => builder.UsingMiddleware(_ => new FuncMiddleware(onNext));

public static IBackgroundTaskrBuilder UsingMiddleware(
   this IBackgroundTaskrBuilder builder,
   Func<IServiceProvider, string, Func<Task>, Task> onNext) =>
   builder.UsingMiddleware(s => new FuncMiddleware((name, next) => onNext(s, name, next)));


internal class FuncMiddleware : IBackgroundTaskrMiddleware
{
   private readonly Func<string, Func<Task>, Task> _onNext;

   public FuncMiddleware(Func<string, Func<Task>, Task> onNext)
   {
       _onNext = onNext;
   }

   public Task OnNext(string name, Func<Task> next) => _onNext(name, next);
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/2661be4f74424955f85c5b96a1c6849a250a76bd/Haydabase.BackgroundTaskr/BackgroundTaskrBuilderExtensions.cs){:target="_blank"}{: .right}

These allow for multiple ways of registering middlewares, respectively:
- resolving/creating the instance explicitly,
- providing just the implementation type to be resolved,
- or providing an inline func defining the behaviour of the middleware (with or without resolving dependencies).

Note each builder method returns the builder, to allow chaining. We must also return a builder from the initial registration method, so it can be used:
{% highlight csharp %}
public static IBackgroundTaskrBuilder AddBackgroundTaskr(this IServiceCollection services)
{
   services.TryAddSingleton<IBackgroundTaskr, BackgroundTaskFactory>();
   services.TryAddScoped<IMiddlewareInvoker, MiddlewareInvoker>();
   return new BackgroundTaskrBuilder(services);
}

internal class BackgroundTaskrBuilder : IBackgroundTaskrBuilder
{
   public BackgroundTaskrBuilder(IServiceCollection services) => Services = services;

   public IServiceCollection Services { get; }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/2661be4f74424955f85c5b96a1c6849a250a76bd/Haydabase.BackgroundTaskr/BackgroundTaskrServiceCollectionExtensions.cs#L13){:target="_blank"}{: .right}

We can now specify our application specific slack notification in a custom middleware when registering the library, for example:
{% highlight csharp %}
services.AddBackgroundTaskr()
   .UsingMiddleware(async (serviceProvider, name, next) =>
   {
       try
       {
           await next();
       }
       catch (Exception exception)
       {
           await serviceProvider.GetRequiredService<ISlackNotifier>().NotifyTaskFailure(name, exception);
       }
   });
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/2661be4f74424955f85c5b96a1c6849a250a76bd/DemoApp/Startup.cs#L34){:target="_blank"}{: .right}

If this turns out to be something we need in multiple services, we could publish a separate internal library with an extension method on the `IBackgroundTaskrBuilder` for easy registering of such a middleware. _(See [this commit](https://github.com/Haydabase/BackgroundTaskr/commit/91a62c8de52bbcd167487ea88eccc95bf017445f){:target="_blank"} for an example implementation.)_

## Observability

Running a reliable production system requires us to be able to understand the behaviour of the live system, and detect and analyze failures. When writing code, we should think about how we will be able to ascertain what the code is doing when running in production. The key to this is having our code produce signals at key events (at a minimum: at the beginning and end of an operation, and on errors), with [high dimensionality and high cardinality](https://www.honeycomb.io/blog/so-you-want-to-build-an-observability-tool/).

Looking at our existing `BackgroundTaskFactory` implementation, right now if we ran this in production, we’d have no way to know when a task was triggered, run, failed etc., let alone start to understand the reason for any failures. So let’s look at ways we can make our library observable out of the box.

### OpenTelemetry
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/85f83cb8bd15b73bc0aafa5f0253da45dab64d01){:target="_blank"}{: .right}

[OpenTelemetry](https://opentelemetry.io/) has been designed as a language/platform agnostic standard for observability, with “[the inspiration of the project [being] to make every library and application observable out of the box](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/overview.md#instrumentation-libraries)”. This sounds like it is exactly what we want.

The [.NET implementation of the OpenTelemetry API](https://github.com/open-telemetry/opentelemetry-dotnet) uses the existing `Activity` to represent an OpenTelemetry [Span](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#span), which essentially represents the current operation (e.g. HTTP request). Because of this, some OpenTelemetry concepts have [different terminology](https://github.com/open-telemetry/opentelemetry-dotnet/issues/947) in the .NET implementation, for example:

| OpenTelemetry Terminology | .NET Terminology |
| ------------------------- | ---------------- |
| [Tracer](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#tracer) | ActivitySource |
| [Span](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#span) | Activity |
| [Attribute](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/common/common.md#attributes) | Tag |

  
Many of the existing official .NET libraries (e.g. [`HttpClient`](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry.Instrumentation.Http/README.md#step-1-install-package)) have separate packages to add OpenTelemetry instrumentation, but the [guidance for new libraries](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/docs/trace/extending-the-sdk/README.md#writing-own-instrumentation-library) is to bake the instrumentation in, without the need for a separate package. So we will do accordingly with our library.

We have two operations to instrument in our library; the creation and running of the background task. In order to create activities for them, we will need an `ActivitySource`:
{% highlight csharp %}
private static ActivitySource ActivitySource { get; } = new ActivitySource("Haydabase.BackgroundTaskr");
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/85f83cb8bd15b73bc0aafa5f0253da45dab64d01/Haydabase.BackgroundTaskr/BackgroundTaskFactory.cs#L12){:target="_blank"}{: .right}

We can then instrument our `CreateBackgroundTask` method as such:
{% highlight csharp %}
public void CreateBackgroundTask(string name, Func<IServiceProvider, Task> runTask)
{
   using var createActivity = ActivitySource.StartActivity(
       $"Create BackgroundTask: {name}",
       ActivityKind.Internal,
       default(ActivityContext),
       new Dictionary<string, object?> {["backgroundtaskr.name"] = name});
   var propagationContext = createActivity?.Context ?? Activity.Current?.Context ?? default(ActivityContext);
   Task.Run(async () =>
   {
       using var runActivity = ActivitySource.StartActivity(
           $"Run BackgroundTask {name}",
           ActivityKind.Internal,
           propagationContext,
           new Dictionary<string, object?> {["backgroundtaskr.name"] = name});
       try
       {
           using var scope = _serviceProvider.CreateScope();
           var invoker = scope.ServiceProvider.GetRequiredService<IMiddlewareInvoker>();
           await invoker.InvokeAsync(name, () => runTask(scope.ServiceProvider));
           runActivity?.SetStatus(Status.Ok);
       }
       catch (Exception exception)
       {
           runActivity?.RecordException(exception);
           runActivity?.SetStatus(Status.Error);
       }
   });
   createActivity?.SetStatus(Status.Ok);
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/85f83cb8bd15b73bc0aafa5f0253da45dab64d01/Haydabase.BackgroundTaskr/BackgroundTaskFactory.cs#L23){:target="_blank"}{: .right}

For more information on how to the Activity API should be used, see the [official documentation](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry.Api/README.md#instrumenting-a-libraryapplication-with-net-activity-api), but here are some things to note:
- `StartActivity` may return `null`, if no listeners are registered on the `ActivitySource` (typically if the application has not set up any exporters for our telemetry data), hence the use of the [null-conditional operator](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-) when accessing the `Activity` properties/methods.
- We explicitly propagate the `createActivity`’s context when creating `runActivity`. This is not strictly required here, as any existing `Activity`’s context is automatically used as the parent on creation of a new `Activity`. If we’d implemented this library by storing jobs in a database to ensure resiliency to service restarts, we would also have to store the context in order to propagate it when fetching the job out of the database. See the official documentation’s example code for how to [inject](https://github.com/open-telemetry/opentelemetry-dotnet/blob/7723fa04cffbf9aca662be87e4599991b8b0d9cc/examples/MicroserviceExample/Utils/Messaging/MessageSender.cs#L78) and [extract](https://github.com/open-telemetry/opentelemetry-dotnet/blob/7723fa04cffbf9aca662be87e4599991b8b0d9cc/examples/MicroserviceExample/Utils/Messaging/MessageReceiver.cs#L61) context using the [Propagators API](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/context/api-propagators.md).

In order for users of OpenTelemetry to collect traces from our library, they must use the [`AddSource` method of the `TracerProviderBuilder`](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry/README.md#activity-source) in their setup. Given we want configuring our library to feel intuitive to use, we’ll add a convenience extension method for adding the instrumentation, using the same semantics as other instrumentations (e.g. [`AspNetCore`](https://github.com/open-telemetry/opentelemetry-dotnet/tree/main/src/OpenTelemetry.Instrumentation.AspNetCore#step-2-enable-aspnet-core-instrumentation-at-application-startup)):
{% highlight csharp %}
services.AddOpenTelemetryTracing(
    x => x
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("BackgroundTaskr.DemoApp"))
        .AddBackgroundTaskrInstrumentation()
        .AddAspNetCoreInstrumentation()
        .AddConsoleExporter()
);
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/85f83cb8bd15b73bc0aafa5f0253da45dab64d01/DemoApp/Startup.cs#L40){:target="_blank"}{: .right}

Simply implemented as such:
{% highlight csharp %}
namespace OpenTelemetry.Trace
{
   public static class TracerProviderBuilderExtensions
   {
       public static TracerProviderBuilder AddBackgroundTaskrInstrumentation(this TracerProviderBuilder builder) =>
           builder.AddSource("Haydabase.BackgroundTaskr");
   }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/85f83cb8bd15b73bc0aafa5f0253da45dab64d01/Haydabase.BackgroundTaskr/TracerProviderBuilderExtensions.cs){:target="_blank"}{: .right}

In our instrumentation, we set a single tag on creation of each `Activity`, using the only information we have about the task; the name. Additional tags can be added to at any point during the operation, enabling enrichment of telemetry data to help our users achieve the high dimensionality and cardinality we were looking for. There are already methods for enriching the task running `Activity`; either in a [middleware](#middleware), or in the task itself, by accessing `Activity.Current`. There aren’t, however, any methods for enriching the task creation `Activity`. We can again take inspiration from other instrumentation libraries to add such a capability...

## Configuration Options
[_Section commit_](https://github.com/Haydabase/BackgroundTaskr/commit/5ffdf4d3831f9efb4b9911bd1253c5831051cca8){:target="_blank"}{: .right}

Most of the official .NET instrumentation libraries provide a mechanism for enriching the activities they instrument via the [Options pattern](https://docs.microsoft.com/en-us/dotnet/core/extensions/options) (e.g. [`HttpClient`](https://github.com/open-telemetry/opentelemetry-dotnet/blob/main/src/OpenTelemetry.Instrumentation.Http/README.md#enrich)). We will use the same approach in our library. Firstly we’ll define an options class:
{% highlight csharp %}
public class BackgroundTaskrOptions
{
   public Action<Activity, string, object>? EnrichCreate { get; set; }
   public Action<Activity, string, object>? EnrichRun { get; set; }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/5ffdf4d3831f9efb4b9911bd1253c5831051cca8/Haydabase.BackgroundTaskr/BackgroundTaskrOptions.cs){:target="_blank"}{: .right}

The standard mechanism for configuring options that developers will expect, is via a configure `Action` on the registration method, so we’ll update ours with some overloads allowing that:
{% highlight csharp %}
public static IBackgroundTaskrBuilder AddBackgroundTaskr(this IServiceCollection services,
   Action<IServiceProvider, BackgroundTaskrOptions> configure)
{
   services.AddTransient<IConfigureOptions<BackgroundTaskrOptions>>(serviceProvider =>
       new ConfigureOptions<BackgroundTaskrOptions>(options => configure(serviceProvider, options)));
   services.TryAddSingleton<IBackgroundTaskr, BackgroundTaskFactory>();
   services.TryAddScoped<IMiddlewareInvoker, MiddlewareInvoker>();
   return new BackgroundTaskrBuilder(services);
}

public static IBackgroundTaskrBuilder AddBackgroundTaskr(this IServiceCollection services,
   Action<BackgroundTaskrOptions> configure) =>
   services.AddBackgroundTaskr((_, options) => configure(options));

public static IBackgroundTaskrBuilder AddBackgroundTaskr(this IServiceCollection services) =>
   services.AddBackgroundTaskr(_ => { });
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/5ffdf4d3831f9efb4b9911bd1253c5831051cca8/Haydabase.BackgroundTaskr/BackgroundTaskrServiceCollectionExtensions.cs){:target="_blank"}{: .right}

As with previous extnesions, we provide multiple overloads for convenience of consumers, respectively:
- one to allow resolving dependencies for the enrichment,
- another simpler configuration action if no dependencies are required,
- and finally leaving the original overload for if no configuration is desired.

In order to use the configured enrichment actions, we must inject an `IOptions<>` instance to our `BackgroundTaskFactory`:
{% highlight csharp %}
internal class BackgroundTaskFactory : IBackgroundTaskr
{
   private static ActivitySource ActivitySource { get; } = new ActivitySource("Haydabase.BackgroundTaskr");
  
   private readonly IServiceProvider _serviceProvider;
   private readonly IOptions<BackgroundTaskrOptions> _options;

   public BackgroundTaskFactory(IServiceProvider serviceProvider, IOptions<BackgroundTaskrOptions> options)
   {
       _serviceProvider = serviceProvider;
       _options = options;
   }

   public void CreateBackgroundTask(string name, Func<IServiceProvider, Task> runTask)
   {
       var options = _options.Value;
       using var createActivity = ActivitySource.StartActivity(
           $"Create BackgroundTask: {name}",
           ActivityKind.Internal,
           default(ActivityContext),
           new Dictionary<string, object?> {["backgroundtaskr.name"] = name});
       options.EnrichCreate?.InvokeSafe(createActivity, "OnStartActivity", name);
       var propagationContext = createActivity?.Context ?? Activity.Current?.Context ?? default(ActivityContext);
       Task.Run(async () =>
       {
           using var runActivity = ActivitySource.StartActivity(
               $"Run BackgroundTask {name}",
               ActivityKind.Internal,
               propagationContext,
               new Dictionary<string, object?> {["backgroundtaskr.name"] = name});
           options.EnrichRun?.InvokeSafe(runActivity, "OnStartActivity", name);
           try
           {
               // Must use newly resolved dependencies in our background task to avoid accessing disposed scoped services.
               using var scope = _serviceProvider.CreateScope();
               var invoker = scope.ServiceProvider.GetRequiredService<IMiddlewareInvoker>();
               await invoker.InvokeAsync(name, () => runTask(scope.ServiceProvider));
               runActivity?.SetStatus(Status.Ok);
           }
           catch (Exception exception)
           {
               runActivity?.RecordException(exception);
               runActivity?.SetStatus(Status.Error);
               options.EnrichRun?.InvokeSafe(runActivity, "OnException", exception);
           }
           finally
           {
               options.EnrichRun?.InvokeSafe(runActivity, "OnStopActivity", name);
           }
       });
       createActivity?.SetStatus(Status.Ok);
       options.EnrichCreate?.InvokeSafe(createActivity, "OnStopActivity", name);
   }
}
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/5ffdf4d3831f9efb4b9911bd1253c5831051cca8/Haydabase.BackgroundTaskr/BackgroundTaskFactory.cs){:target="_blank"}{: .right}

Note that each access of `IOptions<>.Value` evaluates all registered configuration actions, so we only call it once per task we create. 

Also note that we are calling a custom [`InvokeSafe` extension method](https://github.com/Haydabase/BackgroundTaskr/blob/5ffdf4d3831f9efb4b9911bd1253c5831051cca8/Haydabase.BackgroundTaskr/EnrichmentExtensions.cs#L8){:target="_blank"} on the enrich actions. This is simply to ensure we only call the action if the `Activity` is not null, and that we swallow any exceptions occurring during enrichment. The OpenTelemetry [error handling specification](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/error-handling.md#basic-error-handling-principles) explicitly mentions that instrumentions should not throw unhandled exceptions, and particularly must take care when accepting external callbacks, as we are here.

Now users of our library can add custom enrichment while registering as such:
{% highlight csharp %}
services.AddBackgroundTaskr(options =>
   {
       options.EnrichCreate = (activity, eventName, rawObject) =>
       {
           switch (eventName)
           {
               case "OnStartActivity" when rawObject is string taskName:
               {
                   activity.SetTag("custom_task_name", taskName);
                   break;
               }
               case "OnStopActivity" when rawObject is string taskName:
               {
                   activity.SetTag("total_milliseconds", activity.Duration.TotalMilliseconds);
                   break;
               }
           }
       };
       options.EnrichRun = (activity, eventName, rawObject) =>
       {
           switch (eventName)
           {
               case "OnStartActivity" when rawObject is string taskName:
               {
                   activity.SetTag("custom_task_name", taskName);
                   break;
               }
               case "OnException" when rawObject is Exception exception:
               {
                   activity.SetTag("stack_trace", exception.StackTrace);
                   break;
               }
               case "OnStopActivity" when rawObject is string taskName:
               {
                   activity.SetTag("total_milliseconds", activity.Duration.TotalMilliseconds);
                   break;
               }
           }
       };
   });
{% endhighlight %}[_Source link_](https://github.com/Haydabase/BackgroundTaskr/blob/5ffdf4d3831f9efb4b9911bd1253c5831051cca8/DemoApp/Startup.cs#L36){:target="_blank"}{: .right}

Hopefully we can now see how we could easily add more configuration settings/toggles to our options class to enable further custom behaviours.

## Versioning

Inevitably libraries evolve over time, as our library has during the previous few sections, to include new features, or fix bugs. So we must consider versioning...

[SemVer](https://semver.org/) is the de facto standard for versioning libraries, as recommended in the [.NET guidelines](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/versioning). This enables developers to easily infer the level of changes between versions, most crucially which versions are backwards compatible and so can be safely upgraded to, and which may require changes in order to upgrade.

Even when using SemVer, thought must be put into which is the appropriate next version. Whilst techniques like [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) can help keep track of changes, developers still need to mark their changes appropriately. So it is vital that any developer maintaining a library be aware of what qualifies as a [breaking change](https://docs.microsoft.com/en-us/dotnet/standard/library-guidance/breaking-changes). For example, breaking changes do not have to be compilation errors; they can equally be changes in behaviour when coding against the same interface.

Where possible, we should provide a mechanism to opt-in to new/different behaviour rather than making a breaking change. For example, imagine we wanted to update our library so that it retries failed tasks. This could have undesirable consequences for users that may reasonably not have coded their tasks to handle being retried. We have a couple of simple ways that we have seen to make this update backwards compatible:
- Add a boolean flag to our `BackgroundTaskrOptions` class to control if the retry behaviour is enabled (`false` by default)
- Add an `AddRetries` method to our builder, which could have the same effect, but with its own options for configuring numbers of attempts, back-off policies etc. It could even simply add a middleware which implements the retries.

## Summary

We have seen how we can design a .NET library that is intuitive to configure, use, and extend, as well as easy to adopt and maintain by:

- Ensuring it has a well defined raison d'être, and avoiding unnecessary dependencies
- Designing with an open-source attitude, even if we may not be open-sourcing
- Choosing a .NET target as wide reaching as possible, considering required features
- Creating types as `internal` by default
- Using concepts developers will be familiar with from other libraries:
  - Middleware
  - Builders
  - Options
- Providing Observability out of the box with OpenTelemetry
- Versioning with SemVer and avoiding breaking changes where practicable 

### Further Reading

This is by no means an exhaustive list of concerns, omitting other important aspects like documentation, CI/CD, testing etc, so here are some links covering other relevant topics:

- [Guidance for library authors](https://devblogs.microsoft.com/dotnet/guidance-for-library-authors/) - .NET Blog
- [Document your C## code with XML comments](https://docs.microsoft.com/en-us/dotnet/csharp/codedoc) - .NET Guidelines
- [How To Build & Publish NuGet Packages With GitHub Actions](https://www.jamescroft.co.uk/how-to-build-publish-nuget-packages-with-github-actions/) - jamescroft.co.uk blog
- [Level up your .NET libraries](https://benfoster.io/blog/level-up-your-dotnet-libraries/) - benfoster.io blog