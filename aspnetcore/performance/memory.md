---
title: ASP.NET Core 中的内存管理和模式
author: rick-anderson
description: 了解 ASP.NET Core 如何管理内存，以及垃圾回收器（GC）的工作方式。
ms.author: riande
ms.custom: mvc
ms.date: 4/05/2019
no-loc:
- Blazor
- Blazor Server
- Blazor WebAssembly
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: performance/memory
ms.openlocfilehash: d261a26de7b9ba77e5f9787ae2eb37293257a0fc
ms.sourcegitcommit: d65a027e78bf0b83727f975235a18863e685d902
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 06/26/2020
ms.locfileid: "85406389"
---
# <a name="memory-management-and-garbage-collection-gc-in-aspnet-core"></a>ASP.NET Core 中的内存管理和垃圾回收（GC）

作者： [Sébastien Ros](https://github.com/sebastienros)和[Rick Anderson](https://twitter.com/RickAndMSFT)

内存管理是很复杂的，即使在 .NET 等托管框架中也是如此。 分析和了解内存问题可能非常困难。 本文：

* 有很多*内存泄漏*和*GC 不起作用*的问题。 其中的大多数问题都是由不了解 .NET Core 中的内存使用情况或不了解其测量方式导致的。
* 演示内存使用情况，并提出替代方法。

## <a name="how-garbage-collection-gc-works-in-net-core"></a>如何在 .NET Core 中使用垃圾回收（GC）

GC 分配堆段，其中每个段都是一系列连续的内存。 位于堆中的对象归类为三个代之一：0、1或2。 该代确定 GC 尝试释放应用程序不再引用的托管对象上内存的频率。 较低编号的生成更为频繁。

根据对象的生存期，将对象从一代移到另一代。 随着对象的运行时间较长，它们会移到较高的代中。 如前所述，较高的版本是不太常见的垃圾回收。 短期生存期的对象始终保留在第0代中。 例如，在 web 请求过程中引用的对象的生存期很短。 应用程序级别[单一实例](xref:fundamentals/dependency-injection#service-lifetimes)通常迁移到第2代。

当 ASP.NET Core 应用启动时，GC：

* 为初始堆段保留一些内存。
* 加载运行时，提交一小部分内存。

出于性能方面的原因，上述内存分配已完成。 性能优势来自连续内存中的堆段。

### <a name="call-gccollect"></a>调用 GC.Collect

调用[调用显式GC.Collect](xref:System.GC.Collect*)：

* **不**应由生产 ASP.NET Core 应用完成。
* 调查内存泄漏时非常有用。
* 调查时，验证 GC 是否已从内存中删除所有无关联的对象，以便可以测量内存。

## <a name="analyzing-the-memory-usage-of-an-app"></a>分析应用的内存使用情况

专用工具可帮助分析内存使用量：

- 计算对象引用数
- 测量 GC 对 CPU 使用的影响程度
- 测量每代使用的内存空间

使用以下工具分析内存使用量：

* [dotnet](/dotnet/core/diagnostics/dotnet-trace)：可在生产计算机上使用。
* [分析不使用 Visual Studio 调试器情况下的内存使用情况](/visualstudio/profiling/memory-usage-without-debugging2)
* [Visual Studio 中的配置文件内存使用](/visualstudio/profiling/memory-usage)

### <a name="detecting-memory-issues"></a>检测内存问题

任务管理器可用于了解 ASP.NET 应用正在使用的内存量。 任务管理器内存值：

* 表示 ASP.NET 进程使用的内存量。
* 包括应用的活对象和其他内存使用者（如本机内存使用情况）。

如果任务管理器内存值无限增加且从未平展，则应用程序的内存泄漏。 以下部分演示并解释了几种内存使用模式。

## <a name="sample-display-memory-usage-app"></a>示例显示内存使用情况应用

GitHub 上提供了[MemoryLeak 示例应用](https://github.com/sebastienros/memoryleak)。 MemoryLeak 应用：

* 包括一个收集应用程序的实时内存和 GC 数据的诊断控制器。
* 具有显示内存和 GC 数据的索引页。 索引页每秒刷新一次。
* 包含提供各种内存负载模式的 API 控制器。
* 不是受支持的工具，但它可用于显示 ASP.NET Core 应用的内存使用模式。

运行 MemoryLeak。 分配的内存缓慢增加，直到 GC 发生。 内存增加是因为该工具分配自定义对象来捕获数据。 下图显示了 Gen 0 GC 发生时的 MemoryLeak 索引页。 此图表显示 0 RPS （每秒请求数），因为未调用 API 控制器中的任何 API 终结点。

![上图](memory/_static/0RPS.png)

此图表显示内存使用量的两个值：

- 已分配：托管对象占用的内存量
- [工作集](/windows/win32/memory/working-set)：进程的虚拟地址空间中当前驻留在物理内存中的页集。 显示的工作集与任务管理器显示的值相同。

### <a name="transient-objects"></a>暂时性对象

以下 API 创建一个 10 KB 的字符串实例，并将其返回给客户端。 对于每个请求，将在内存中分配一个新的对象，并将其写入响应中。 字符串作为 UTF-16 字符存储在 .NET 中，因此每个字符需要2个字节的内存。

```csharp
[HttpGet("bigstring")]
public ActionResult<string> GetBigString()
{
    return new String('x', 10 * 1024);
}
```

下面的关系图是使用相对较小的负载生成的，用于显示 GC 如何影响内存分配。

![上图](memory/_static/bigstring.png)

上面的图表显示：

* 4K RPS （每秒请求数）。
* 第0代垃圾回收大约每两秒发生一次。
* 工作集的大小约为 500 MB。
* CPU 为12%。
* 内存消耗和发布（通过 GC）是稳定的。

以下图表采用可由计算机处理的最大吞吐量。

![上图](memory/_static/bigstring2.png)

上面的图表显示：

* 22K RPS
* 第0代垃圾回收每秒发生多次。
* 由于每秒分配的内存量明显增加，因此将触发第1代回收。
* 工作集的大小约为 500 MB。
* CPU 为33%。
* 内存消耗和发布（通过 GC）是稳定的。
* CPU （33%）不会过度使用，因此垃圾回收可以跟上大量分配。

### <a name="workstation-gc-vs-server-gc"></a>工作站 GC 与服务器 GC

.NET 垃圾回收器具有两种不同的模式：

* **工作站 GC**：针对桌面进行了优化。
* **服务器 GC**。 ASP.NET Core 应用的默认 GC。 针对服务器进行了优化。

GC 模式可以在项目文件中或在发布的应用程序的*runtimeconfig.js*文件中显式设置。 以下标记显示 `ServerGarbageCollection` 项目文件中的设置：

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

更改 `ServerGarbageCollection` 项目文件需要重新生成应用。

**注意：** 服务器垃圾回收在具有单个核心的计算机上**不可用。** 有关详细信息，请参阅 <xref:System.Runtime.GCSettings.IsServerGC>。

下图显示了使用工作站 GC 的占用大量 RPS 的内存配置文件。

![上图](memory/_static/workstation.png)

此图表与服务器版本之间的区别非常重要：

- 工作集从 500 MB 降到 70 MB。
- GC 每秒生成0次（而不是每隔两秒）回收一次。
- GC 从 300 MB 降到 10 MB。

在典型的 web 服务器环境中，CPU 使用率比内存更重要，因此服务器 GC 更好。 如果内存使用率很高且 CPU 使用率相对较低，则工作站 GC 可能会更高的性能。 例如，在内存不足的情况下承载几个 web 应用的高密度。

<a name="sc"></a>

### <a name="gc-using-docker-and-small-containers"></a>使用 Docker 和小型容器的 GC

当在一台计算机上运行多个容器化应用程序时，工作站 GC 可能比服务器 GC 更 preformant。 有关详细信息，请参阅在小型[容器中运行服务器 gc](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-0/)和[在小型容器方案中运行服务器 GC 部分– GC 堆的硬限制](https://devblogs.microsoft.com/dotnet/running-with-server-gc-in-a-small-container-scenario-part-1-hard-limit-for-the-gc-heap/)。

### <a name="persistent-object-references"></a>持久性对象引用

GC 无法释放所引用的对象。 引用但不再需要的对象将导致内存泄露。 如果应用经常分配对象，但在不再需要对象之后无法释放它们，则内存使用量将随着时间的推移而增加。

以下 API 创建一个 10 KB 的字符串实例，并将其返回给客户端。 与上一示例的不同之处在于，此实例由静态成员引用，这意味着它不能用于收集。

```csharp
private static ConcurrentBag<string> _staticStrings = new ConcurrentBag<string>();

[HttpGet("staticstring")]
public ActionResult<string> GetStaticString()
{
    var bigString = new String('x', 10 * 1024);
    _staticStrings.Add(bigString);
    return bigString;
}
```

前面的代码：

* 典型内存泄漏的示例。
* 如果频繁调用，会导致应用内存增加，直到进程因异常而崩溃 `OutOfMemory` 。

![上图](memory/_static/eternal.png)

在上图中：

* 负载测试 `/api/staticstring` 终结点会导致内存线性增加。
* GC 在内存压力增加时，通过调用第2代回收来尝试释放内存。
* GC 无法释放泄漏的内存。 已分配和工作集增加了时间。

某些方案（如缓存）需要保留对象引用，直到内存压力强制释放它们。 <xref:System.WeakReference>类可用于这种类型的缓存代码。 `WeakReference`对象在内存压力下收集。 的默认实现 <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> 使用 `WeakReference` 。

### <a name="native-memory"></a>本机内存

某些 .NET Core 对象依赖本机内存。 GC**无法**收集本机内存。 使用本机内存的 .NET 对象必须使用本机代码释放它。

.NET 提供了 <xref:System.IDisposable> 接口，使开发人员能够释放本机内存。 即使 <xref:System.IDisposable.Dispose*> 未调用，也会 `Dispose` 在[终结器](/dotnet/csharp/programming-guide/classes-and-structs/destructors)运行时正确实现类调用。

考虑下列代码：

```csharp
[HttpGet("fileprovider")]
public void GetFileProvider()
{
    var fp = new PhysicalFileProvider(TempPath);
    fp.Watch("*.*");
}
```

[PhysicalFileProvider](/dotnet/api/microsoft.extensions.fileproviders.physicalfileprovider?view=dotnet-plat-ext-3.0)是托管类，因此将在请求结束时收集任何实例。

下图显示了连续调用 API 时的内存配置文件 `fileprovider` 。

![上图](memory/_static/fileprovider.png)

上面的图表显示了此类的实现的一个明显问题，因为它会不断增加内存使用量。 这是[此问题](https://github.com/dotnet/aspnetcore/issues/3110)中正在跟踪的已知问题。

可以通过以下方式之一在用户代码中发生相同的泄漏：

* 不能正确释放类。
* 忘记调用 `Dispose` 应释放的依赖对象的方法。

### <a name="large-objects-heap"></a>大型对象堆

频繁的内存分配/空闲周期可以分段内存，尤其是在分配大块内存时。 对象在连续内存块中分配。 为了缓解碎片，当 GC 释放内存时，它会 trys 对内存进行碎片整理。 此过程称为**压缩**。 压缩涉及移动对象。 移动大型对象会对性能产生负面影响。 出于此原因，GC 将为_大型_对象（称为[大型对象堆](/dotnet/standard/garbage-collection/large-object-heap)（LOH））创建特殊的内存区域。 大于85000字节（大约 83 KB）的对象为：

* 放置在 LOH 上。
* 未压缩。
* 在第2代 Gc 期间收集。

当 LOH 已满时，GC 将触发第2代回收。 第2代回收：

* 的速度非常慢。
* 此外，还会产生在所有其他代上触发集合的成本。

以下代码会立即压缩 LOH：

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();
```

<xref:System.Runtime.GCSettings.LargeObjectHeapCompactionMode>有关压缩 LOH 的信息，请参阅。

在使用 .NET Core 3.0 和更高版本的容器中，LOH 将自动压缩。

以下 API 演示了此行为：

```csharp
[HttpGet("loh/{size=85000}")]
public int GetLOH1(int size)
{
   return new byte[size].Length;
}
```

下图显示了 `/api/loh/84975` 在最大负载下调用终结点的内存配置文件：

![上图](memory/_static/loh1.png)

下图显示调用终结点的内存配置文件 `/api/loh/84976` ，只分配*一个字节*：

![上图](memory/_static/loh2.png)

注意：此 `byte[]` 结构具有开销字节。 这就是84976字节触发85000限制的原因。

比较上述两个图表：

* 对于这两种方案（约 450 MB），工作集都是类似的。
* LOH 请求（84975字节）下面显示第0代回收。
* Over LOH 请求生成常量第2代回收。 第2代回收成本高昂。 需要更多 CPU，吞吐量几乎会下降到50%。

临时大型对象尤其有问题，因为它们会导致 gen2 Gc。

为了获得最佳性能，应最大程度地减少使用的大型对象。 如果可能，请拆分大型对象。 例如，ASP.NET Core 中的[响应缓存](xref:performance/caching/response)中间件会将缓存项拆分为小于85000个字节的块。

以下链接显示了在 LOH 限制下保留对象的 ASP.NET Core 方法：

* [ResponseCaching/StreamUtilities](https://github.com/dotnet/AspNetCore/blob/v3.0.0/src/Middleware/ResponseCaching/src/Streams/StreamUtilities.cs#L16)
* [ResponseCaching/MemoryResponseCache](https://github.com/aspnet/ResponseCaching/blob/c1cb7576a0b86e32aec990c22df29c780af29ca5/src/Microsoft.AspNetCore.ResponseCaching/Internal/MemoryResponseCache.cs#L55)

有关详情，请参阅：

* [发现的大型对象堆](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [大型对象堆](/dotnet/standard/garbage-collection/large-object-heap)

### <a name="httpclient"></a>HttpClient

使用不当 <xref:System.Net.Http.HttpClient> 可能会导致资源泄漏。 系统资源，如数据库连接、套接字、文件句柄等：

* 比内存更稀有。
* 泄漏内存时，问题更多。

经验丰富的 .NET 开发人员知道要对 <xref:System.IDisposable.Dispose*> 实现的对象调用 <xref:System.IDisposable> 。 不释放实现的对象 `IDisposable` 通常会导致内存泄漏或泄漏系统资源。

`HttpClient`实现 `IDisposable` ，但**不**应在每次调用时都释放。 `HttpClient`应重复使用。

以下终结点创建并释放 `HttpClient` 每个请求的一个新实例：

```csharp
[HttpGet("httpclient1")]
public async Task<int> GetHttpClient1(string url)
{
    using (var httpClient = new HttpClient())
    {
        var result = await httpClient.GetAsync(url);
        return (int)result.StatusCode;
    }
}
```

在 "负载" 下，将记录以下错误消息：

```
fail: Microsoft.AspNetCore.Server.Kestrel[13]
      Connection id "0HLG70PBE1CR1", Request id "0HLG70PBE1CR1:00000031":
      An unhandled exception was thrown by the application.
System.Net.Http.HttpRequestException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted --->
    System.Net.Sockets.SocketException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted
   at System.Net.Http.ConnectHelper.ConnectAsync(String host, Int32 port,
    CancellationToken cancellationToken)
```

即使 `HttpClient` 实例被释放，实际的网络连接也需要一些时间才能由操作系统释放。 通过持续创建新的连接，会发生_端口耗尽_。 每个客户端连接都需要自己的客户端端口。

防止端口耗尽的一种方法是重复使用同一个 `HttpClient` 实例：

```csharp
private static readonly HttpClient _httpClient = new HttpClient();

[HttpGet("httpclient2")]
public async Task<int> GetHttpClient2(string url)
{
    var result = await _httpClient.GetAsync(url);
    return (int)result.StatusCode;
}
```

`HttpClient`当应用程序停止时，将释放该实例。 此示例说明，每次使用后都不应释放每个可释放资源。

请参阅以下内容，了解更好的方法来处理实例的生存期 `HttpClient` ：

* [HttpClient 和生存期管理](/aspnet/core/fundamentals/http-requests#httpclient-and-lifetime-management)
* [HTTPClient 工厂博客](https://devblogs.microsoft.com/aspnet/asp-net-core-2-1-preview1-introducing-httpclient-factory/)
 
### <a name="object-pooling"></a>对象池

前面的示例演示了如何将 `HttpClient` 实例设为静态的，并由所有请求重复使用。 重复使用会阻止资源耗尽。

对象池：

* 使用重复使用模式。
* 适用于创建成本很高的对象。

池是预初始化对象的集合，这些对象可以在线程之间保留和释放。 池可以定义分配规则，例如限制、预定义大小或增长速率。

NuGet 包[ObjectPool](https://www.nuget.org/packages/Microsoft.Extensions.ObjectPool/)包含有助于管理此类池的类。

以下 API 终结点将实例化一个 `byte` 缓冲区，该缓冲区使用每个请求的随机数字填充：

```csharp
        [HttpGet("array/{size}")]
        public byte[] GetArray(int size)
        {
            var random = new Random();
            var array = new byte[size];
            random.NextBytes(array);

            return array;
        }
```

以下图表显示了如何通过中等负载调用前面的 API：

![上图](memory/_static/array.png)

在上图中，第0代回收大约每秒发生一次。

可以通过 `byte` 使用[ArrayPool \<T> ](xref:System.Buffers.ArrayPool`1)来对缓冲池进行优化，从而优化上述代码。 静态实例可跨请求重复使用。

此方法的不同之处在于，将从 API 返回一个共用对象。 这意味着：

* 从方法返回后，将立即从控件中排除对象。
* 不能释放对象。

设置对象的释放：

* 将池数组封装到可释放对象中。
* 将此池对象注册为[RegisterForDispose](xref:Microsoft.AspNetCore.Http.HttpResponse.RegisterForDispose*)。

`RegisterForDispose`将负责调用 `Dispose` 目标对象，以便仅当 HTTP 请求完成时才会释放该对象。

```csharp
private static ArrayPool<byte> _arrayPool = ArrayPool<byte>.Create();

private class PooledArray : IDisposable
{
    public byte[] Array { get; private set; }

    public PooledArray(int size)
    {
        Array = _arrayPool.Rent(size);
    }

    public void Dispose()
    {
        _arrayPool.Return(Array);
    }
}

[HttpGet("pooledarray/{size}")]
public byte[] GetPooledArray(int size)
{
    var pooledArray = new PooledArray(size);

    var random = new Random();
    random.NextBytes(pooledArray.Array);

    HttpContext.Response.RegisterForDispose(pooledArray);

    return pooledArray.Array;
}
```

应用与非池版本相同的负载会导致以下图表：

![上图](memory/_static/pooledarray.png)

主要区别是分配的字节数，因此产生的第0代回收量更少。

## <a name="additional-resources"></a>其他资源

* [垃圾回收](/dotnet/standard/garbage-collection/)
* [利用并发可视化工具了解不同的 GC 模式](https://blogs.msdn.microsoft.com/seteplia/2017/01/05/understanding-different-gc-modes-with-concurrency-visualizer/)
* [发现的大型对象堆](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [大型对象堆](/dotnet/standard/garbage-collection/large-object-heap)
