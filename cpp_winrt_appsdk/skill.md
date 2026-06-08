---
name: c++/winrt development and review guide
description: general guidelines for developing Windows apps in c++ using c++/winrt. This guideline highlights the additional considerations on top of the standard C++ when writing apps using c++/winrt. This guidelines applies to development and code review.
---
## Overview

C++/WinRT is an entirely standard modern C++17 language projection for Windows Runtime (WinRT) APIs, implemented as a header-file-based library. C++/WinRT can author and consume Windows Runtime APIs using any standards-compliant C++17 compiler.

## String types usage

With C++/WinRT, you can call Windows Runtime APIs using C++ Standard Library wide string types such as std::wstring (not with narrow string types such as std::string). C++/WinRT have a custom string type called winrt::hstring. winrt::hstring provides convertibility with std::wstring\_view to provide the interoperability that std::basic\_string\_view.

An hstring is a range, so you can use it with range-based for, or with std::for\_each. It also provides comparison operators for naturally and efficiently comparing against its counterparts in the C++ Standard Library. Use included helpers, such as winrt::to\_string and winrt::to\_hstring, for converting between winrt::hstring and std::string.

## Standard C++ data type in c++/winrt

winrt::array\_view has conversion constructors from std::vector and std::array.

C++/WinRT binds std::vector as a Windows Runtime collection parameter. So passing a std::vector[winrt::hstring](winrt::hstring) will be converted to the appropriate Windows Runtime collection of winrt::hstring. But you can't pass a std::vector[std::wstring](std::wstring) where a Windows Runtime collection is expected.

winrt::array\_view has conversion constructors from a raw array, and from a range of T\* (pointers to the element type).A host of constructors, operators, functions, and iterators are implemented for winrt::array\_view. A winrt::array\_view is a range, so you can use it with range-based for, or with std::for\_each.

You can continue to use the Parallel Patterns Library (PPL) when calling asynchronous Windows Runtime APIs. However, in many cases, C++ coroutines provide an efficient and more easily-coded idiom for interacting with asynchronous objects.

## Boxing and unboxing values

The IInspectable interface is the root interface of every runtime class in the Windows Runtime (WinRT). a function that expects IInspectable can be passed an instance of any runtime class. But you can't directly pass to such a function a scalar value (such as a numeric or text value), nor an array. Instead, a scalar or array value needs to be wrapped inside a reference class object. That wrapping process is known as boxing the value.

The winrt::box\_value function, which takes a scalar or array value, and returns the value boxed into an IInspectable. For unboxing an IInspectable back into a scalar or array value, there is the winrt::unbox\_value function. For unboxing an IInspectable back into a scalar value, there is also the winrt::unbox\_value\_or function. for example:

```
Button().Content(winrt::box_value(L"Clicked"));
```

and

```
hstring hstringValue = unbox_value<hstring>(object); // Throws if object is not a boxed string.
hstringValue = unbox_value_or<hstring>(object, L"Default"); // Returns L"Default" if object is not a boxed string.
float floatValue = unbox_value_or<float>(object, 0.f); // Returns 0.0 if object is not a boxed float.
std::optional<int> optionalInt = object.try_as<int>(); // Returns std::nullopt if object is not a boxed int.
```

For a boxed value with unknown type, query the boxed value for its IPropertyValue interface, and then call Type on that. for example:

```
auto piInspectable = winrt::box_value(pi);
auto piPropertyValue = piInspectable.as<winrt::Windows::Foundation::IPropertyValue>();
WINRT_ASSERT(piPropertyValue.Type() == winrt::Windows::Foundation::PropertyType::Single);
```

## Consume APIs

For every type in a Windows namespace defined in metadata, C++/WinRT defines a C++-friendly equivalent (called the projected type). A projected type has the same fully-qualified name as the Windows type, but it's placed in the C++ winrt namespace using C++ syntax. For example, Windows::Foundation::Uri is projected into C++/WinRT as winrt::Windows::Foundation::Uri.

Once you have a C++/WinRT projected type value, you can treat it as if it were an instance of the actual Windows Runtime type, since it has all the same members.

To consume Windows namespace APIs from C++/WinRT, you include headers from the %WindowsSdkDir%Include<WindowsTargetPlatformVersion>\\cppwinrt\\winrt folder. You must include the headers corresponding to each namespace you use.

With the C++/WinRT projection, the runtime representation of a Windows Runtime class is no more than the underlying ABI interfaces. But, for your convenience, you can code against classes in the way that their author intended. For example, you can call the ToString method of a Uri as if that were a method of the class.

In C++/WinRT, each projected type has a special C++/WinRT std::nullptr\_t constructor. With the exception of that one, all projected-type constructors—including the default constructor—cause a backing Windows Runtime object to be created, and give you a smart pointer to it.

## Ahthor APIs

author C++/WinRT APIs by using the winrt::implements base struct, either directly or indirectly. Synonyms for author in this context are produce, or implement.

declare a runtime class in a Microsoft Interface Definition Language (IDL) (.idl) file (see Factoring runtime classes into Midl files (.idl)).

Each IDL file results in a .winmd file, and Visual Studio merges all of those into a single file with the same name as your root namespace. That final .winmd file will be the one that the consumers of your component will reference.

Here's an example of declaring a runtime class in an IDL file.

```
// MyRuntimeClass.idl
namespace MyProject
{
    runtimeclass MyRuntimeClass
    {
        // Declaring a constructor (or constructors) in the IDL causes the runtime class to be
        // activatable from outside the compilation unit.
        MyRuntimeClass();
        String Name;
    }
}
```

This IDL declares a Windows Runtime (runtime) class. A runtime class is a type that can be activated and consumed via modern COM interfaces, typically across executable boundaries. When you add an IDL file to your project, and build, the C++/WinRT toolchain (midl.exe and cppwinrt.exe) generates not only an implementation type but also a projected type.

Each constructor you declare in your IDL causes a constructor to be generated on both your implementation type and on your projected type. IDL-declared constructors are used to consume the runtime class from a different compilation unit.

Whether you have IDL-declared constructor(s) or not, a constructor overload that takes std::nullptr\_t is generated on your projected type. Calling the std::nullptr\_t constructor is the first of two steps in consuming the runtime class from the same compilation unit.

the workflow is to use IDL to declare your runtime class and its members, and then the tooling generates prototypes and stub implementations for you. As for those autogenerated prototypes for the members of your runtime class, you can edit them so that they pass around different types from the types that you declare in your IDL.

## Error handling

We recommend that you continue to write exception-safe code, but that you prefer to avoid catching and throwing exceptions whenever possible.

Don't throw an exception that you expect to catch. And don't use exceptions for expected failures. Throw an exception only when an unexpected runtime error occurs, and handle everything else with error/result codes—directly, and close to the source of the failure.

An error condition that arises at the Windows Runtime ABI layer is returned in the form of a HRESULT value. But you don't need to handle HRESULTs in your code. The C++/WinRT projection code that's generated for an API on the consuming side detects an error.

All Windows Runtime Application Binary Interface boundaries (or ABI boundaries) must be noexcept—meaning that exceptions must never escape there. When you author an API, you should always mark the ABI boundary with the C++ noexcept keyword. noexcept has specific behavior in C++. If a C++ exception hits a noexcept boundary, then the process will fail fast with std::terminate.

## Author events

Event is declared in the \*.idl file. For example, declare an event whose delegate type is EventHandler with an argument of a single-precision floating-point number

```
event Windows.Foundation.EventHandler<Single> TemperatureIsBelowFreezing;
```

In C++/WinRT, an IDL-declared event is implemented as a set of overloaded functions (similar to the way a property is implemented as a pair of overloaded get and set functions). One overload takes a delegate to be registered, and returns a token (a winrt::event\_token). The other takes a token, and revokes the registration of the associated delegate. The corresponding event in the header file is represented by the winrt::event struct template, parameterized by a particular delegate type (which itself can be parameterized by an args type).

If the event must be accessible across an application binary interface (ABI)—such as between a component and its consuming application, then the event must use a Windows Runtime delegate type.

If you don't need to pass any parameters or arguments with event, then define your own simple Windows Runtime delegate type.

A common pattern in the Windows Runtime is the deferrable event. An event handler takes a deferral by calling the event argument's GetDeferral method. Doing so indicates to the event source that post-event activities should be postponed until the deferral is completed. This allows an event handler to perform asynchronous actions in response to an event.

The winrt::deferrable\_event\_args struct template is a helper class for implementing (producing) the Windows Runtime deferral pattern

We recommend that you pass events, and not delegates, as function parameters. The add function of winrt::event is the one exception, because you must pass a delegate in that case. The reason for this guideline is because delegates can take different forms across different Windows Runtime languages (in terms of whether they support one client registration, or multiple). Events, with their multiple subscriber model, constitute a much more predictable and consistent option.

The signature for an event handler delegate should consist of two parameters: sender (IInspectable), and args (some event argument type, for example RoutedEventArgs).

## Collections

IVector is the Windows Runtime interface implemented by any random-access collection of elements. If you were to implement IVector yourself, you'd also need to implement IIterable, IVectorView, and IIterator.

To retrieve a new object of a type that implements a general-purpose collection, you can call the winrt::single\_threaded\_vector function template. The object is returned as an IVector, and that's the interface via which you call the returned object's functions and properties.

If you need an immutable view over the collection, then you can call IVector::GetView

Pass a temporary object containing data to winrt::single\_threaded\_vector, or move a std::vector into the function will pass an rvalue into the function. That enables the compiler to be efficient and to avoid copying the data.

To retrieve a new object of a type that implements an observable collection, call the winrt::single\_threaded\_observable\_vector function template with any element type. But to make an observable collection suitable for binding to a XAML items control, use IInspectable as the element type.

## Concurrency and asynchronous operations

Any Windows Runtime API that has the potential to take more than 50 milliseconds to complete is implemented as an asynchronous function (with a name ending in "Async"). The implementation of an asynchronous function initiates the work on another thread, and returns immediately with an object that represents the asynchronous operation. When the asynchronous operation completes, that returned object contains any value that resulted from the work. The Windows::Foundation Windows Runtime namespace contains four types of asynchronous operation object.

C++/WinRT integrates C++ coroutines into the programming model to provide a natural way to cooperatively wait for a result. You can produce your own Windows Runtime asynchronous operation by writing a coroutine.

A coroutine is a function that can be suspended and resumed.

If you're asynchronously returning a Windows Runtime type, then you should return an IAsyncOperation<TResult> or an IAsyncOperationWithProgress<TResult, TProgress>. Any first- or third-party runtime class qualifies, or any type that can be passed to or from a Windows Runtime function (for example, int, or winrt::hstring).

If a coroutine doesn't have at least one co\_await statement then, in order to qualify as a coroutine, it must have at least one co\_return or one co\_yield statement.

If you're asynchronously returning a type that's not a Windows Runtime type, then you should return a Parallel Patterns Library (PPL) concurrency::task.

For synchronous functions, you should use const\& parameters by default. That will avoid the overhead of copies (which involve reference counting, and that means interlocked increments and decrements).

In a coroutine, execution is synchronous up until the first suspension point, where control is returned to the caller and the calling frame goes out of scope. By the time the coroutine resumes, anything might have happened to the source value that a reference parameter references. From the coroutine's perspective, a reference parameter has uncontrolled lifetime. So, in the example above, we're safe to access value up until the co\_await, but not after it. In the event that value is destructed by the caller, attempting to access it inside the coroutine after that results in a memory corruption. Nor can we safely pass value to DoOtherWorkAsync if there's any risk that that function will in turn suspend and then try to use value after it resumes.

## Bind XMAL control to c++/Winrt property

A property that can be effectively bound to a XAML control is known as an observable property. This idea is based on the software design pattern known as the observer pattern

## Naming conventions

C++/WinRT has established the following naming conventions:

The winrt::impl namespace is reserved for C++/WinRT, and you shouldn't use it in your application.

In the winrt namespace, names that begin with a lowercase letter belong to C++/WinRT, but you may use them in your application. The documentation calls out those names that you can overload or specialize. For example, your application is permitted to specialize the winrt::is\_guid\_of function template.

In sub-namespaces of the winrt namespace (except for winrt::impl), names that begin with an uppercase letter are available to your application.

In all namespaces, names beginning with WINRT\_IMPL\_ are reserved for C++/WinRT, and you shouldn't use them in your application.

In all namespaces, names beginning with WINRT\_ (except those that begin with WINRT\_IMPL\_) are reserved for C++/WinRT. You may use them, and the documentation calls out those names that may be defined by your application, such as WINRT\_LEAN\_AND\_MEAN.

## Threading and async programming

use the thread pool to accomplish work asynchronously in parallel threads. The thread pool manages a set of threads and uses a queue to assign work items to threads as they become available. The thread pool is similar to the asynchronous programming patterns available in the Windows Runtime but the thread pool offers more control than the asynchronous programming patterns. use it to complete multiple work items in parallel.

Submit work items, control their priority, and cancel work items.

Schedule work items using timers and periodic timers.

Set aside resources for critical work items.

Run work items in response to named events and semaphores.

The thread pool is more efficient at managing threads because it reduces the overhead of creating and destroying threads. The means it has access to optimize threads across multiple CPU cores, and it can balance thread resources between apps and when background tasks are running. Using the built-in thread pool is convenient because you focus on writing code that accomplishes a task instead of the mechanics of thread management.

### Do's

Use the thread pool to do parallel work in your app.

Use work items to accomplish extended tasks without blocking the UI thread.

Create work items that are short-lived and independent. Work items run asynchronously and they can be submitted to the pool in any order from the queue.

Dispatch updates to the UI thread with the Windows.UI.Core.CoreDispatcher.

Use ThreadPoolTimer.CreateTimer instead of the Sleep function.

Use the thread pool instead of creating your own thread management system. The thread pool runs at the OS level with advanced capability and is optimized to dynamically scale according to device resources and activity within the process and across the system.

In C++, ensure that work item delegates use the agile threading model (C++ delegates are agile by default).

Use pre-allocated work items when you can't tolerate a resource allocation failure at time of use.

### Don'ts

Don't create periodic timers with a period value of <1 millisecond (including 0). This will cause the work item to behave as a single-shot timer.

Don't submit periodic work items that take longer to complete than the amount of time you specified in the period parameter.

Don't try to send UI updates (other than toasts and notifications) from a work item dispatched from a background task. Instead, use background task progress and completion handlers - for example, IBackgroundTaskInstance.Progress.

When you use work-item handlers that use the async keyword, don't assume that all code in the handler has executed when the complete state has been set on the work item. The thread pool work item may be set to the complete state before all of the code in the handler has executed. Code following an await keyword within the handler may execute after the work item has been set to the complete state.

Don't try to run a pre-allocated work item more than once without reinitializing it. Create a periodic work item

### Create the periodic work item

Use the CreatePeriodicTimer method to create a periodic work item. Supply a lambda that accomplishes the work, and use the period parameter to specify the interval between submissions. The period is specified using a TimeSpan structure. The work item will be resubmitted every time the period elapses, so make sure the period is long enough for work to complete

### Cancel the timer

When necessary, call the Cancel method to stop the periodic work item from repeating. If the work item is running when the periodic timer is cancelled it is allowed to complete. The TimerDestroyedHandler (if provided) is called when all instances of the periodic work item have completed.

### Create and submit the work item

Create a work item by calling RunAsync. Supply a delegate to do the work (you can use a lambda, or a delegate function). Note that RunAsync returns an IAsyncAction object; store this object for use in the next step.

Three versions of RunAsync are available so that you can optionally specify the priority of the work item, and control whether it runs concurrently with other work items.

### Create a single-shot timer
Use the CreateTimer method to create a timer for the work item. Supply a lambda that accomplishes the work, and use the delay parameter to specify how long the thread pool waits before it can assign the work item to an available thread. The delay is specified using a TimeSpan structure.

Note  You can use CoreDispatcher.RunAsync to access the UI and show progress from the work item.

