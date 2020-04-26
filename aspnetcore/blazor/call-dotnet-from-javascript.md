---
title: 从 ASP.NET Core Blazor 中的 JavaScript 函数调用 .NET 方法
author: guardrex
description: 了解如何在 Blazor 应用中通过 JavaScript 函数调用 .NET 方法。
monikerRange: '>= aspnetcore-3.1'
ms.author: riande
ms.custom: mvc
ms.date: 04/07/2020
no-loc:
- Blazor
- SignalR
uid: blazor/call-dotnet-from-javascript
ms.openlocfilehash: e2344dd15efd243a405373b6cf0362f28b48173a
ms.sourcegitcommit: f0aeeab6ab6e09db713bb9b7862c45f4d447771b
ms.translationtype: HT
ms.contentlocale: zh-CN
ms.lasthandoff: 04/08/2020
ms.locfileid: "80976945"
---
# <a name="call-net-methods-from-javascript-functions-in-aspnet-core-opno-locblazor"></a><span data-ttu-id="87d90-103">从 ASP.NET Core Blazor 中的 JavaScript 函数调用 .NET 方法</span><span class="sxs-lookup"><span data-stu-id="87d90-103">Call .NET methods from JavaScript functions in ASP.NET Core Blazor</span></span>

<span data-ttu-id="87d90-104">作者：[Javier Calvarro Nelson](https://github.com/javiercn)、[Daniel Roth](https://github.com/danroth27)、[Shashikant Rudrawadi](http://wisne.co) 和 [Luke Latham](https://github.com/guardrex)</span><span class="sxs-lookup"><span data-stu-id="87d90-104">By [Javier Calvarro Nelson](https://github.com/javiercn), [Daniel Roth](https://github.com/danroth27), [Shashikant Rudrawadi](http://wisne.co), and [Luke Latham](https://github.com/guardrex)</span></span>

[!INCLUDE[](~/includes/blazorwasm-preview-notice.md)]

<span data-ttu-id="87d90-105">Blazor 应用可从 .NET 方法调用 JavaScript 函数，也可从 JavaScript 函数调用 .NET 方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-105">A Blazor app can invoke JavaScript functions from .NET methods and .NET methods from JavaScript functions.</span></span> <span data-ttu-id="87d90-106">这被称为 JavaScript 互操作（JS 互操作）   。</span><span class="sxs-lookup"><span data-stu-id="87d90-106">These scenarios are called *JavaScript interoperability* (*JS interop*).</span></span>

<span data-ttu-id="87d90-107">本文介绍如何从 JavaScript 调用 .NET 方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-107">This article covers invoking .NET methods from JavaScript.</span></span> <span data-ttu-id="87d90-108">要详细了解如何通过 .NET 调用 JavaScript 函数，请参阅 <xref:blazor/call-javascript-from-dotnet>。</span><span class="sxs-lookup"><span data-stu-id="87d90-108">For information on how to call JavaScript functions from .NET, see <xref:blazor/call-javascript-from-dotnet>.</span></span>

<span data-ttu-id="87d90-109">[查看或下载示例代码](https://github.com/dotnet/AspNetCore.Docs/tree/master/aspnetcore/blazor/common/samples/)（[如何下载](xref:index#how-to-download-a-sample)）</span><span class="sxs-lookup"><span data-stu-id="87d90-109">[View or download sample code](https://github.com/dotnet/AspNetCore.Docs/tree/master/aspnetcore/blazor/common/samples/) ([how to download](xref:index#how-to-download-a-sample))</span></span>

## <a name="static-net-method-call"></a><span data-ttu-id="87d90-110">静态 .NET 方法调用</span><span class="sxs-lookup"><span data-stu-id="87d90-110">Static .NET method call</span></span>

<span data-ttu-id="87d90-111">要从 JavaScript 调用静态 .NET 方法，请使用 `DotNet.invokeMethod` 或 `DotNet.invokeMethodAsync` 函数。</span><span class="sxs-lookup"><span data-stu-id="87d90-111">To invoke a static .NET method from JavaScript, use the `DotNet.invokeMethod` or `DotNet.invokeMethodAsync` functions.</span></span> <span data-ttu-id="87d90-112">传入要调用的静态方法的标识符、包含该函数的程序集的名称以及任意自变量。</span><span class="sxs-lookup"><span data-stu-id="87d90-112">Pass in the identifier of the static method you wish to call, the name of the assembly containing the function, and any arguments.</span></span> <span data-ttu-id="87d90-113">异步版本是支持 Blazor 服务器方案的首选。</span><span class="sxs-lookup"><span data-stu-id="87d90-113">The asynchronous version is preferred to support Blazor Server scenarios.</span></span> <span data-ttu-id="87d90-114">该 .NET 方法必须是公共的静态方法，并具有 `[JSInvokable]` 特性。</span><span class="sxs-lookup"><span data-stu-id="87d90-114">The .NET method must be public, static, and have the `[JSInvokable]` attribute.</span></span> <span data-ttu-id="87d90-115">当前不支持调用开放式泛型方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-115">Calling open generic methods isn't currently supported.</span></span>

<span data-ttu-id="87d90-116">该示例应用包含一个 C# 方法，用于返回 `int` 数组。</span><span class="sxs-lookup"><span data-stu-id="87d90-116">The sample app includes a C# method to return an `int` array.</span></span> <span data-ttu-id="87d90-117">`JSInvokable` 特性应用于该方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-117">The `JSInvokable` attribute is applied to the method.</span></span>

<span data-ttu-id="87d90-118">Pages/JsInterop.razor  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-118">*Pages/JsInterop.razor*:</span></span>

```razor
<button type="button" class="btn btn-primary"
        onclick="exampleJsFunctions.returnArrayAsyncJs()">
    Trigger .NET static method ReturnArrayAsync
</button>

@code {
    [JSInvokable]
    public static Task<int[]> ReturnArrayAsync()
    {
        return Task.FromResult(new int[] { 1, 2, 3 });
    }
}
```

<span data-ttu-id="87d90-119">为客户端提供的 JavaScript 会调用 C# .net 方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-119">JavaScript served to the client invokes the C# .NET method.</span></span>

<span data-ttu-id="87d90-120">wwwroot/exampleJsInterop.js  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-120">*wwwroot/exampleJsInterop.js*:</span></span>

[!code-javascript[](./common/samples/3.x/BlazorWebAssemblySample/wwwroot/exampleJsInterop.js?highlight=8-14)]

<span data-ttu-id="87d90-121">如果选择了“触发 .NET 静态方法 ReturnArrayAsync”按钮，请在浏览器的 Web 开发人员工具中检查控制台输出  。</span><span class="sxs-lookup"><span data-stu-id="87d90-121">When the **Trigger .NET static method ReturnArrayAsync** button is selected, examine the console output in the browser's web developer tools.</span></span>

<span data-ttu-id="87d90-122">控制台输出为：</span><span class="sxs-lookup"><span data-stu-id="87d90-122">The console output is:</span></span>

```console
Array(4) [ 1, 2, 3, 4 ]
```

<span data-ttu-id="87d90-123">第四个数组值推送到 `ReturnArrayAsync` 返回的数组 (`data.push(4);`)。</span><span class="sxs-lookup"><span data-stu-id="87d90-123">The fourth array value is pushed to the array (`data.push(4);`) returned by `ReturnArrayAsync`.</span></span>

<span data-ttu-id="87d90-124">默认情况下，方法标识符为方法名称，但你可使用 `JSInvokableAttribute` 构造函数指定其他标识符：</span><span class="sxs-lookup"><span data-stu-id="87d90-124">By default, the method identifier is the method name, but you can specify a different identifier using the `JSInvokableAttribute` constructor:</span></span>

```csharp
@code {
    [JSInvokable("DifferentMethodName")]
    public static Task<int[]> ReturnArrayAsync()
    {
        return Task.FromResult(new int[] { 1, 2, 3 });
    }
}
```

<span data-ttu-id="87d90-125">在客户端 JavaScript 文件中：</span><span class="sxs-lookup"><span data-stu-id="87d90-125">In the client-side JavaScript file:</span></span>

```javascript
returnArrayAsyncJs: function () {
  DotNet.invokeMethodAsync('BlazorSample', 'DifferentMethodName')
    .then(data => {
      data.push(4);
      console.log(data);
    });
}
```

## <a name="instance-method-call"></a><span data-ttu-id="87d90-126">实例方法调用</span><span class="sxs-lookup"><span data-stu-id="87d90-126">Instance method call</span></span>

<span data-ttu-id="87d90-127">还可以从 JavaScript 调用 .NET 实例方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-127">You can also call .NET instance methods from JavaScript.</span></span> <span data-ttu-id="87d90-128">从 JavaScript 调用 .NET 实例方法：</span><span class="sxs-lookup"><span data-stu-id="87d90-128">To invoke a .NET instance method from JavaScript:</span></span>

* <span data-ttu-id="87d90-129">按引用向 JavaScript 传递 .NET 实例：</span><span class="sxs-lookup"><span data-stu-id="87d90-129">Pass the .NET instance by reference to JavaScript:</span></span>
  * <span data-ttu-id="87d90-130">对 `DotNetObjectReference.Create` 进行静态调用。</span><span class="sxs-lookup"><span data-stu-id="87d90-130">Make a static call to `DotNetObjectReference.Create`.</span></span>
  * <span data-ttu-id="87d90-131">在 `DotNetObjectReference` 实例中包装实例，并在 `DotNetObjectReference` 实例上调用 `Create`。</span><span class="sxs-lookup"><span data-stu-id="87d90-131">Wrap the instance in a `DotNetObjectReference` instance and call `Create` on the `DotNetObjectReference` instance.</span></span> <span data-ttu-id="87d90-132">处置 `DotNetObjectReference` 对象（本部分稍后会展示一个示例）。</span><span class="sxs-lookup"><span data-stu-id="87d90-132">Dispose of `DotNetObjectReference` objects (an example appears later in this section).</span></span>
* <span data-ttu-id="87d90-133">使用 `invokeMethod` 或 `invokeMethodAsync` 函数在实例上调用 .NET 实例方法。</span><span class="sxs-lookup"><span data-stu-id="87d90-133">Invoke .NET instance methods on the instance using the `invokeMethod` or `invokeMethodAsync` functions.</span></span> <span data-ttu-id="87d90-134">在从 JavaScript 调用其他 .NET 方法时，也可以将 .NET 实例作为自变量传递。</span><span class="sxs-lookup"><span data-stu-id="87d90-134">The .NET instance can also be passed as an argument when invoking other .NET methods from JavaScript.</span></span>

> [!NOTE]
> <span data-ttu-id="87d90-135">示例应用会将消息记录到客户端控制台。</span><span class="sxs-lookup"><span data-stu-id="87d90-135">The sample app logs messages to the client-side console.</span></span> <span data-ttu-id="87d90-136">对于示例应用展示的以下示例，请在浏览器的开发人员工具中检查浏览器的控制台输出。</span><span class="sxs-lookup"><span data-stu-id="87d90-136">For the following examples demonstrated by the sample app, examine the browser's console output in the browser's developer tools.</span></span>

<span data-ttu-id="87d90-137">选择“触发 .NET 实例方法 HelloHelper.SayHello”按钮时，将调用 `ExampleJsInterop.CallHelloHelperSayHello`，并将名称 `Blazor` 传递到方法  。</span><span class="sxs-lookup"><span data-stu-id="87d90-137">When the **Trigger .NET instance method HelloHelper.SayHello** button is selected, `ExampleJsInterop.CallHelloHelperSayHello` is called and passes a name, `Blazor`, to the method.</span></span>

<span data-ttu-id="87d90-138">Pages/JsInterop.razor  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-138">*Pages/JsInterop.razor*:</span></span>

```razor
<button type="button" class="btn btn-primary" @onclick="TriggerNetInstanceMethod">
    Trigger .NET instance method HelloHelper.SayHello
</button>

@code {
    public async Task TriggerNetInstanceMethod()
    {
        var exampleJsInterop = new ExampleJsInterop(JSRuntime);
        await exampleJsInterop.CallHelloHelperSayHello("Blazor");
    }
}
```

<span data-ttu-id="87d90-139">`CallHelloHelperSayHello` 使用 `HelloHelper` 的新实例调用 JavaScript 函数 `sayHello`。</span><span class="sxs-lookup"><span data-stu-id="87d90-139">`CallHelloHelperSayHello` invokes the JavaScript function `sayHello` with a new instance of `HelloHelper`.</span></span>

<span data-ttu-id="87d90-140">JsInteropClasses/ExampleJsInterop.cs  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-140">*JsInteropClasses/ExampleJsInterop.cs*:</span></span>

[!code-csharp[](./common/samples/3.x/BlazorWebAssemblySample/JsInteropClasses/ExampleJsInterop.cs?name=snippet1&highlight=11-18)]

<span data-ttu-id="87d90-141">wwwroot/exampleJsInterop.js  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-141">*wwwroot/exampleJsInterop.js*:</span></span>

[!code-javascript[](./common/samples/3.x/BlazorWebAssemblySample/wwwroot/exampleJsInterop.js?highlight=15-18)]

<span data-ttu-id="87d90-142">该名称将传递给 `HelloHelper` 的构造函数，该构造函数设置 `HelloHelper.Name` 属性。</span><span class="sxs-lookup"><span data-stu-id="87d90-142">The name is passed to `HelloHelper`'s constructor, which sets the `HelloHelper.Name` property.</span></span> <span data-ttu-id="87d90-143">执行 JavaScript 函数 `sayHello` 时，`HelloHelper.SayHello` 返回 `Hello, {Name}!` 消息，JavaScript 函数将该消息写入控制台。</span><span class="sxs-lookup"><span data-stu-id="87d90-143">When the JavaScript function `sayHello` is executed, `HelloHelper.SayHello` returns the `Hello, {Name}!` message, which is written to the console by the JavaScript function.</span></span>

<span data-ttu-id="87d90-144">JsInteropClasses/HelloHelper.cs  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-144">*JsInteropClasses/HelloHelper.cs*:</span></span>

[!code-csharp[](./common/samples/3.x/BlazorWebAssemblySample/JsInteropClasses/HelloHelper.cs?name=snippet1&highlight=5,10-11)]

<span data-ttu-id="87d90-145">浏览器 Web 开发人员工具中的控制台输出：</span><span class="sxs-lookup"><span data-stu-id="87d90-145">Console output in the browser's web developer tools:</span></span>

```console
Hello, Blazor!
```

<span data-ttu-id="87d90-146">为避免内存泄露并允许对创建 `DotNetObjectReference` 的组件进行垃圾回收，请采用以下任一方法：</span><span class="sxs-lookup"><span data-stu-id="87d90-146">To avoid a memory leak and allow garbage collection on a component that creates a `DotNetObjectReference`, adopt one of the following approaches:</span></span>

* <span data-ttu-id="87d90-147">处置类中创建了 `DotNetObjectReference` 实例的对象：</span><span class="sxs-lookup"><span data-stu-id="87d90-147">Dispose of the object in the class that created the `DotNetObjectReference` instance:</span></span>

  ```csharp
  public class ExampleJsInterop : IDisposable
  {
      private readonly IJSRuntime _jsRuntime;
      private DotNetObjectReference<HelloHelper> _objRef;

      public ExampleJsInterop(IJSRuntime jsRuntime)
      {
          _jsRuntime = jsRuntime;
      }

      public ValueTask<string> CallHelloHelperSayHello(string name)
      {
          _objRef = DotNetObjectReference.Create(new HelloHelper(name));

          return _jsRuntime.InvokeAsync<string>(
              "exampleJsFunctions.sayHello",
              _objRef);
      }

      public void Dispose()
      {
          _objRef?.Dispose();
      }
  }
  ```

  <span data-ttu-id="87d90-148">还可以在组件中实现上述 `ExampleJsInterop` 类中所示的模式：</span><span class="sxs-lookup"><span data-stu-id="87d90-148">The preceding pattern shown in the `ExampleJsInterop` class can also be implemented in a component:</span></span>

  ```razor
  @page "/JSInteropComponent"
  @using BlazorSample.JsInteropClasses
  @implements IDisposable
  @inject IJSRuntime JSRuntime

  <h1>JavaScript Interop</h1>

  <button type="button" class="btn btn-primary" @onclick="TriggerNetInstanceMethod">
      Trigger .NET instance method HelloHelper.SayHello
  </button>

  @code {
      private DotNetObjectReference<HelloHelper> _objRef;

      public async Task TriggerNetInstanceMethod()
      {
          _objRef = DotNetObjectReference.Create(new HelloHelper("Blazor"));

          await JSRuntime.InvokeAsync<string>(
              "exampleJsFunctions.sayHello",
              _objRef);
      }

      public void Dispose()
      {
          _objRef?.Dispose();
      }
  }
  ```

* <span data-ttu-id="87d90-149">如果组件或类不处置 `DotNetObjectReference`，请通过调用 `.dispose()` 在客户端上处置该对象：</span><span class="sxs-lookup"><span data-stu-id="87d90-149">When the component or class doesn't dispose of the `DotNetObjectReference`, dispose of the object on the client by calling `.dispose()`:</span></span>

  ```javascript
  window.myFunction = (dotnetHelper) => {
    dotnetHelper.invokeMethod('BlazorSample', 'MyMethod');
    dotnetHelper.dispose();
  }
  ```

## <a name="component-instance-method-call"></a><span data-ttu-id="87d90-150">组件实例方法调用</span><span class="sxs-lookup"><span data-stu-id="87d90-150">Component instance method call</span></span>

<span data-ttu-id="87d90-151">要调用组件的 .NET 方法，请执行以下操作：</span><span class="sxs-lookup"><span data-stu-id="87d90-151">To invoke a component's .NET methods:</span></span>

* <span data-ttu-id="87d90-152">使用 `invokeMethod` 或 `invokeMethodAsync` 函数对组件执行静态方法调用。</span><span class="sxs-lookup"><span data-stu-id="87d90-152">Use the `invokeMethod` or `invokeMethodAsync` function to make a static method call to the component.</span></span>
* <span data-ttu-id="87d90-153">组件的静态方法将其实例方法调用包装为已调用的 `Action`。</span><span class="sxs-lookup"><span data-stu-id="87d90-153">The component's static method wraps the call to its instance method as an invoked `Action`.</span></span>

<span data-ttu-id="87d90-154">在客户端 JavaScript 中：</span><span class="sxs-lookup"><span data-stu-id="87d90-154">In the client-side JavaScript:</span></span>

```javascript
function updateMessageCallerJS() {
  DotNet.invokeMethod('BlazorSample', 'UpdateMessageCaller');
}
```

<span data-ttu-id="87d90-155">Pages/JSInteropComponent.razor  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-155">*Pages/JSInteropComponent.razor*:</span></span>

```razor
@page "/JSInteropComponent"

<p>
    Message: @_message
</p>

<p>
    <button onclick="updateMessageCallerJS()">Call JS Method</button>
</p>

@code {
    private static Action _action;
    private string _message = "Select the button.";

    protected override void OnInitialized()
    {
        _action = UpdateMessage;
    }

    private void UpdateMessage()
    {
        _message = "UpdateMessage Called!";
        StateHasChanged();
    }

    [JSInvokable]
    public static void UpdateMessageCaller()
    {
        _action.Invoke();
    }
}
```

<span data-ttu-id="87d90-156">如果有多个组件，每个组件都有要调用的实例方法，请使用 helper 类来调用每个组件的实例方法（如 `Action`）。</span><span class="sxs-lookup"><span data-stu-id="87d90-156">When there are several components, each with instance methods to call, use a helper class to invoke the instance methods (as `Action`s) of each component.</span></span>

<span data-ttu-id="87d90-157">如下示例中：</span><span class="sxs-lookup"><span data-stu-id="87d90-157">In the following example:</span></span>

* <span data-ttu-id="87d90-158">`JSInterop` 组件包含若干 `ListItem` 组件。</span><span class="sxs-lookup"><span data-stu-id="87d90-158">The `JSInterop` component contains several `ListItem` components.</span></span>
* <span data-ttu-id="87d90-159">每个 `ListItem` 组件都由一个消息和一个按钮组成。</span><span class="sxs-lookup"><span data-stu-id="87d90-159">Each `ListItem` component is composed of a message and a button.</span></span>
* <span data-ttu-id="87d90-160">选择 `ListItem` 组件按钮后，`ListItem` 的 `UpdateMessage` 方法会更改列表项文本并隐藏该按钮。</span><span class="sxs-lookup"><span data-stu-id="87d90-160">When a `ListItem` component button is selected, that `ListItem`'s `UpdateMessage` method changes the list item text and hides the button.</span></span>

<span data-ttu-id="87d90-161">MessageUpdateInvokeHelper.cs  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-161">*MessageUpdateInvokeHelper.cs*:</span></span>

```csharp
using System;
using Microsoft.JSInterop;

public class MessageUpdateInvokeHelper
{
    private Action _action;

    public MessageUpdateInvokeHelper(Action action)
    {
        _action = action;
    }

    [JSInvokable("BlazorSample")]
    public void UpdateMessageCaller()
    {
        _action.Invoke();
    }
}
```

<span data-ttu-id="87d90-162">在客户端 JavaScript 中：</span><span class="sxs-lookup"><span data-stu-id="87d90-162">In the client-side JavaScript:</span></span>

```javascript
window.updateMessageCallerJS = (dotnetHelper) => {
    dotnetHelper.invokeMethod('BlazorSample', 'UpdateMessageCaller');
    dotnetHelper.dispose();
}
```

<span data-ttu-id="87d90-163">Shared/ListItem.razor  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-163">*Shared/ListItem.razor*:</span></span>

```razor
@inject IJSRuntime JsRuntime

<li>
    @_message
    <button @onclick="InteropCall" style="display:@_display">InteropCall</button>
</li>

@code {
    private string _message = "Select one of these list item buttons.";
    private string _display = "inline-block";
    private MessageUpdateInvokeHelper _messageUpdateInvokeHelper;

    protected override void OnInitialized()
    {
        _messageUpdateInvokeHelper = new MessageUpdateInvokeHelper(UpdateMessage);
    }

    protected async Task InteropCall()
    {
        await JsRuntime.InvokeVoidAsync("updateMessageCallerJS",
            DotNetObjectReference.Create(_messageUpdateInvokeHelper));
    }

    private void UpdateMessage()
    {
        _message = "UpdateMessage Called!";
        _display = "none";
        StateHasChanged();
    }
}
```

<span data-ttu-id="87d90-164">Pages/JSInterop.razor  ：</span><span class="sxs-lookup"><span data-stu-id="87d90-164">*Pages/JSInterop.razor*:</span></span>

```razor
@page "/JSInterop"

<h1>List of components</h1>

<ul>
    <ListItem />
    <ListItem />
    <ListItem />
    <ListItem />
</ul>
```

[!INCLUDE[Share interop code in a class library](~/includes/blazor-share-interop-code.md)]

## <a name="avoid-circular-object-references"></a><span data-ttu-id="87d90-165">避免循环引用对象</span><span class="sxs-lookup"><span data-stu-id="87d90-165">Avoid circular object references</span></span>

<span data-ttu-id="87d90-166">不能在客户端上针对以下调用就包含循环引用的对象进行序列化：</span><span class="sxs-lookup"><span data-stu-id="87d90-166">Objects that contain circular references can't be serialized on the client for either:</span></span>

* <span data-ttu-id="87d90-167">.NET 方法调用。</span><span class="sxs-lookup"><span data-stu-id="87d90-167">.NET method calls.</span></span>
* <span data-ttu-id="87d90-168">返回类型具有循环引用时，从 C# 发出的 JavaScript 方法调用。</span><span class="sxs-lookup"><span data-stu-id="87d90-168">JavaScript method calls from C# when the return type has circular references.</span></span>

<span data-ttu-id="87d90-169">有关详细信息，请查看以下问题：</span><span class="sxs-lookup"><span data-stu-id="87d90-169">For more information, see the following issues:</span></span>

* [<span data-ttu-id="87d90-170">循环引用不受支持，使用两个按钮 (dotnet/aspnetcore #20525)</span><span class="sxs-lookup"><span data-stu-id="87d90-170">Circular references are not supported, take two (dotnet/aspnetcore #20525)</span></span>](https://github.com/dotnet/aspnetcore/issues/20525)
* [<span data-ttu-id="87d90-171">建议：在序列化时添加机制来处理循环引用 (dotnet/runtime #30820)</span><span class="sxs-lookup"><span data-stu-id="87d90-171">Proposal: Add mechanism to handle circular references when serializing (dotnet/runtime #30820)</span></span>](https://github.com/dotnet/runtime/issues/30820)

## <a name="additional-resources"></a><span data-ttu-id="87d90-172">其他资源</span><span class="sxs-lookup"><span data-stu-id="87d90-172">Additional resources</span></span>

* <xref:blazor/call-javascript-from-dotnet>
* [<span data-ttu-id="87d90-173">InteropComponent.razor 示例（dotnet/AspNetCore GitHub 存储库，3.1 版本分支）</span><span class="sxs-lookup"><span data-stu-id="87d90-173">InteropComponent.razor example (dotnet/AspNetCore GitHub repository, 3.1 release branch)</span></span>](https://github.com/dotnet/AspNetCore/blob/release/3.1/src/Components/test/testassets/BasicTestApp/InteropComponent.razor)
* <span data-ttu-id="87d90-174">[在 Blazor 服务器应用中执行大型数据传输](xref:blazor/advanced-scenarios#perform-large-data-transfers-in-blazor-server-apps)</span><span class="sxs-lookup"><span data-stu-id="87d90-174">[Perform large data transfers in Blazor Server apps](xref:blazor/advanced-scenarios#perform-large-data-transfers-in-blazor-server-apps)</span></span>