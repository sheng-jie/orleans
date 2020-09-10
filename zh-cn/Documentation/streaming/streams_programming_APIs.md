---
layout: page
title: Orleans Streams Programming APIs
---

# Orleans流编程API

应用程序通过与众所周知的API非常相似的API与流交互[NET中的无功扩展(Rx)](https://msdn.microsoft.com/en-us/data/gg577609.aspx). 主要的区别是Orleans河的延伸是**异步**，以使处理在Orleans的分布式和可扩展计算结构中更高效。

### 异步流<a name="Async-Stream"></a>

应用程序通过使用*流提供程序*获取流的句柄。您可以阅读有关流提供程序的更多信息[在这里](stream_providers.md)，但现在您可以将其视为允许实现者自定义流行为和语义的流工厂：

```csharp
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(Guid, "MyStreamNamespace");
```

应用程序可以通过调用`GetStreamProvider`方法`Grains`类，或者调用`GrainClient.GetStreamProvider()`方法。

[**`Orleans.Streams.IAsyncStream<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncStream.cs)是一个**虚拟流的逻辑强类型句柄**. 它在精神上类似于OrleansGrains引用。访问`GetStreamProvider`和`GetStream`完全是本地的。关于`GetStream`是一个GUID和一个额外的字符串，我们称之为流命名空间(可以为null)。GUID和名称空间字符串一起构成流标识(与`GrainFactory.GetGrain`). GUID和命名空间字符串的组合为确定流标识提供了额外的灵活性。就像Grains7可能存在于Grains类型中一样`播放器`而不同的grains7可能存在于该grains类型中`聊天室Grains`，流123可能与流命名空间一起存在`播放事件流`并且不同的流123可以存在于流命名空间中`聊天室消息流`.

### 生产和消费<a name="Producing-and-Consuming"></a>

`IAsyncStream<T>`同时实现[**`Orleans.Streams.IAsyncObserver<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObserver.cs)和[**`Orleans.Streams.IAsyncObservable<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/IAsyncObservable.cs)接口。这样，应用程序就可以使用流通过使用`Orleans.Streams.IAsyncObserver<T>`或者使用`Orleans.Streams.IAsyncObservable<T>`.

```csharp
public interface IAsyncObserver<in T>
{
    Task OnNextAsync(T item, StreamSequenceToken token = null);
    Task OnCompletedAsync();
    Task OnErrorAsync(Exception ex);
}

public interface IAsyncObservable<T>
{
    Task<StreamSubscriptionHandle<T>> SubscribeAsync(IAsyncObserver<T> observer);
}
```

为了在流中生成事件，应用程序只调用

```csharp
await stream.OnNextAsync<T>(event)
```

为了订阅流，应用程序调用

```csharp
StreamSubscriptionHandle<T> subscriptionHandle = await stream.SubscribeAsync(IAsyncObserver)
```

争论`订阅同步`可以是实现`IAsyncObserver服务器`接口或lambda函数的组合来处理传入事件。更多选项`订阅同步`可通过[**`AsyncObservableExtensions`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Extensions/AsyncObservableExtensions.cs)班级。`订阅同步`返回[**`StreamSubscriptionHandle<T>`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/StreamSubscriptionHandle.cs)，这是一个不透明的句柄，可用于从流中取消订阅(在精神上类似于异步版本的`不可分解`).

```csharp
await subscriptionHandle.UnsubscribeAsync()
```

需要注意的是**订阅是为了Grains，而不是为了激活**. 一旦grain代码被订阅到流中，这个订阅将超过这个激活的生命周期，并且永远保持持久，直到grain代码(可能在不同的激活中)显式取消订阅。这是一个**虚拟流抽象**：不仅所有流在逻辑上始终存在，而且流订阅也是持久的，并且在创建订阅的特定物理激活之后仍然有效。

### 多重性<a name="Multiplicity"></a>

一条Orleans河可能有多个生产者和多个消费者。生产者发布的消息将被传递给在消息发布之前订阅流的所有使用者。

此外，消费者可以多次订阅同一个流。每次订阅它都会返回一个唯一的`StreamSubscriptionHandle<T>`. 如果一个grain(或客户端)订阅同一个流X次，它将接收相同的事件X次，每次订阅一次。消费者还可以从单个订阅中取消订阅。它可以通过调用以下命令来查找其当前的所有订阅：

```csharp
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

### 从故障中恢复<a name="Recovering-From-Failures"></a>

如果流的生产者死了(或者它的grains被停用)，它就不需要做什么了。下次这个grain想要生成更多事件时，它可以再次获得流句柄，并以相同的方式生成新的事件。

消费者的逻辑更复杂一些。正如我们前面所说的，一旦消费者的Grains被订阅到一个流中，这个订阅在该Grains明确取消订阅之前是有效的。如果流的消费者死亡(或其Grain被停用)并且流上生成了一个新事件，消费Grain将被自动重新激活(就像任何普通的OrleansGrains在向其发送消息时自动激活一样)。grain代码现在只需要提供一个`IAsyncObserver<T>`处理数据。使用者基本上需要重新附加处理逻辑作为`非激活异步`方法。为此，它可以调用：

```csharp
StreamSubscriptionHandle<int> newHandle = await subscriptionHandle.ResumeAsync(IAsyncObserver)
```

消费者使用它第一次订阅时获得的前一个句柄来“继续处理”。注意到`恢复异步`仅使用的新实例更新现有订阅`IAsyncObserver服务器`逻辑，并不会改变此使用者已订阅此流的事实。

消费者如何获得旧的订阅句柄？有两种选择。消费者可能保留了从原始状态返回的句柄`订阅同步`操作，现在可以使用它。或者，如果消费者没有把手，它可以询问`IAsyncStream<T>`对于其所有活动订阅句柄，通过调用：

```csharp
IList<StreamSubscriptionHandle<T>> allMyHandles = await IAsyncStream<T>.GetAllSubscriptionHandles()
```

消费者现在可以恢复所有这些服务，如果愿意，也可以取消订阅。

**评论：**如果消费Grains实施`IAsyncObserver服务器`直接接口(`公共类MyGrain<T>：Grain，IAsyncObserver<T>`)，理论上不应要求重新附加`IAsyncObserver服务器`因此不需要调用`恢复异步`. 流式运行时应该能够自动发现grain已经实现了`IAsyncObserver服务器`只会调用那些`IAsyncObserver服务器`方法。但是，流式运行时当前不支持此功能，并且grain代码仍然需要显式调用`恢复异步`即使Grains工具`IAsyncObserver服务器`直接。支持这一点是我们的待办事项。

### 显式和隐式订阅<a name="Explicit-and-Implicit-Subscriptions"></a>

默认情况下，流使用者必须显式订阅流。此订阅通常由grain(或客户端)收到的指示其订阅的外部消息触发。例如，在聊天服务中，当用户加入聊天室时，他的Grains收到`加入聊天组`带有聊天名称的消息，这将导致用户grain订阅此聊天流。

此外，OrleansStreams也支持**“隐式订阅”**. 在这个模型中，grains并没有显式地订阅流。这个grain是自动的，隐式的，仅仅基于它的grain标识和`隐式itstreamsubscription`属性。隐式订阅的主要价值是允许流活动自动触发Grain激活(从而触发订阅)。例如，使用SMS流，如果一个grain要生成一个流，而另一个grain要处理这个流，则生产者需要知道消费Grains的身份，并对其进行grain调用，告诉它订阅流。只有在那之后，它才能开始发送事件。相反，使用隐式订阅，生产者只需开始为流生成事件，消费Grain将自动激活并订阅流。在这种情况下，制作人根本不在乎谁在看事件

类型的Grains实现类`我的grains类型`可以声明属性`[隐式itstreamsubscription(“MyStreamNamespace”)]`. 这将告诉流式运行时，当在标识为GUID XXX和`“MyStreamNamespace”`命名空间，则应将其传递到标识为XXX类型的grains`我的grains类型`. 也就是说，运行时映射流`<XXX，MyStreamNamespace>`消费grain`<XXX，我的grains类型>`.

存在`隐式itstreamsubscription`使流式处理执行阶段自动将此grains订阅到流，并将流事件传递给它。但是，grain代码仍然需要告诉运行时它希望如何处理事件。本质上，它需要附加`IAsyncObserver服务器`. 因此，当Grains被激活时，里面的Grains代码`非激活异步`需要调用：

```csharp
IStreamProvider streamProvider = base.GetStreamProvider("SimpleStreamProvider");
IAsyncStream<T> stream = streamProvider.GetStream<T>(this.GetPrimaryKey(), "MyStreamNamespace");
StreamSubscriptionHandle<T> subscription = await stream.SubscribeAsync(IAsyncObserver<T>);
```

### 写入订阅逻辑<a name="Writing-Subscription-Logic"></a>

下面是关于如何为各种情况编写订阅逻辑的指南：显式和隐式订阅、可倒带和不可倒带流。显式订阅和隐式订阅的主要区别在于，对于隐式订阅，grain对于每个流命名空间始终只有一个隐式订阅；无法创建多个订阅(没有订阅多重性)，也无法取消订阅，而grain逻辑总是只需要附加处理逻辑。这也意味着对于隐式订阅，永远不需要恢复订阅。另一方面，对于显式订阅，需要恢复订阅，否则如果再次订阅Grain，则会导致订阅多次。

**隐式订阅：**

对于隐式订阅，grain需要订阅以附加处理逻辑。这应该在谷仓里做`非激活异步`方法。Grains应该简单地执行`等待流式订阅同步(下一页…)`在它的`非激活异步`方法。这将导致此特定激活附加`OnNext`函数来处理该流。grains可以选择指定`StreamSequenceToken`作为`订阅同步`，这将导致此隐式订阅从该令牌开始使用。永远不需要隐式订阅来调用`恢复异步`.

```csharp
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    await stream.SubscribeAsync(OnNextAsync)
}
```

**显式订阅：**

对于显式订阅，grain必须调用`订阅同步`订阅流。这将创建一个订阅，并附加处理逻辑。显式订阅将一直存在，直到取消订阅该Grain，因此，如果某个Grain被停用并重新激活，则该Grain仍然显式订阅，但不会附加任何处理逻辑。在这种情况下，Grains需要重新附加处理逻辑。为了做到这一点`非激活异步`，grain首先需要通过调用`stream.GetAllSubscriptionHandles()`. Grains必须执行`恢复异步`对于每个句柄，它希望继续处理或取消订阅已处理的任何句柄的异步。grains也可以选择指定`StreamSequenceToken`作为`恢复异步`调用，这将导致此显式订阅从该令牌开始使用。

```csharp
public async override Task OnActivateAsync()
{
    var streamProvider = GetStreamProvider(PROVIDER_NAME);
    var stream = streamProvider.GetStream<string>(this.GetPrimaryKey(), "MyStreamNamespace");
    var subscriptionHandles = await stream.GetAllSubscriptionHandles();
    if (!subscriptionHandles.IsNullOrEmpty())
        subscriptionHandles.ForEach(async x => await x.ResumeAsync(OnNextAsync));
}
```

### 流顺序和序列标记<a name="Stream-Order-and-Sequence-Tokens"></a>

单个生产者和单个消费者之间的事件传递顺序取决于流提供者。

对于SMS，生产者通过控制生产者发布事件的方式来显式地控制消费者看到的事件顺序。默认情况下(如果`火上浇油`SMS provider的选项设置为false)，并且如果生产者等待`OnNextAsync`访问，事件按先进先出顺序到达。在SMS中，由制作者决定如何处理将由中断表示的传递失败`Task`由`OnNextAsync`调用。

Azure队列流不保证FIFO顺序，因为底层的Azure队列不保证故障情况下的顺序。(它们确实保证了无故障执行中的先进先出顺序。)当生产者将事件生成到Azure队列中时，如果排队操作失败，则由生产者尝试另一次排队，然后再处理潜在的重复消息。在传递端，Orleans流式处理运行时将事件从队列中取出，并尝试将其传递给消费者进行处理。Orleans流式处理运行时仅在成功处理后从队列中删除事件。如果传递或处理失败，则不会从队列中删除事件，稍后将自动重新出现在队列中。流式运行时将再次尝试传递它，因此可能会破坏FIFO顺序。上面的行为符合Azure队列的正常语义。

**应用程序定义的顺序**：要处理上述排序问题，应用程序可以选择指定自己的排序。这是通过[**`StreamSequenceToken`**](https://github.com/dotnet/orleans/blob/master/src/Orleans.Core.Abstractions/Streams/Core/StreamSubscriptionHandle.cs)，它是不透明的`i可比较`对象，可用于对事件排序。制作者可以通过一个可选的`StreamSequenceToken`致`OnNext`调用。这个`StreamSequenceToken`将一直传递给消费者，并与活动一起交付。这样，应用程序就可以独立于流式运行时推理和重建其顺序。

### 可倒流<a name="Rewindable-Streams"></a>

有些流只允许应用程序从最新的时间点开始订阅它们，而其他流则允许“返回时间”。后一种功能取决于底层队列技术和特定的流提供程序。例如，Azure队列只允许使用最新的排队事件，而EventHub允许从任意时间点(直到某个过期时间)重放事件。支持时间倒流的流称为**可倒流**.

可倒带流的使用者可以传递`StreamSequenceToken`致`订阅同步`调用。运行时将从这个开始向它传递事件`StreamSequenceToken`. 空令牌表示消费者希望接收从最新开始的事件。

回放流的能力在恢复场景中非常有用。例如，考虑订阅流并定期检查其状态和最新序列标记的Grain。当从失败中恢复时，grain可以从最新的检查点序列令牌重新订阅同一个流，从而恢复而不会丢失自上一个检查点以来生成的任何事件。

[事件中心提供程序](https://www.nuget.org/packages/Microsoft.Orleans.OrleansServiceBus/)可倒带。你可以找到它的代码[在这里](https://github.com/dotnet/orleans/tree/master/src/Azure/Orleans.Streaming.EventHubs). [短讯服务](https://www.nuget.org/packages/Microsoft.Orleans.OrleansProviders/)和[Azure队列](https://www.nuget.org/packages/Microsoft.Orleans.Streaming.AzureStorage/)提供程序不可倒带。

### 无状态自动扩展处理<a name="Stateless-Automatically-Scaled-Out-Processing"></a>

默认情况下，Orleans流的目标是支持大量相对较小的流，每个流由一个或多个有状态Grain处理。总的来说，所有流的处理在大量规则的(有状态的)grains之间被切分。应用程序代码通过分配流ID和GrainID以及显式订阅来控制这种分片。**目标是分片状态处理**.

然而，还有一个有趣的场景**自动扩展无状态处理**. 在这个场景中，应用程序有少量的流(甚至一个大的流)，目标是无状态处理。例如，事件的全局流，其中的处理涉及对每个事件进行解码，并可能将其转发到其他流以进行进一步的有状态处理。在Orleans，可以通过`无状态工作者`Grains。

**无状态自动扩展处理的当前状态：**这还没有实施。从订阅流的尝试`无状态工作者`grains将导致未定义的行为。[我们正在考虑支持这一选择](https://github.com/dotnet/orleans/issues/433).

### Grains和Orleans客户<a name="Grains-and-Orleans-Clients"></a>

OrleansStreams**在Grains和Orleans的客户中都是一致的**. 也就是说，可以在grain内部和Orleans客户端中使用完全相同的api来生成和消费事件。这大大简化了应用程序逻辑，使得特殊的客户端api(如Grain observer)变得多余。

### 全面管理和可靠的流媒体酒吧<a name="Fully-Managed-and-Reliable-Streaming-Pub-Sub"></a>

为了跟踪流订阅，Orleans使用一个名为**流媒体酒吧**作为流消费者和流生产者的交汇点。Pub-Sub跟踪所有流订阅，持久化它们，并将流消费者与流生产者匹配。

应用程序可以选择发布订阅数据的存储位置和存储方式。Pub-Sub组件本身被实现为grains(称为`公共地下室`)，它使用Orleans声明性持久化。`公共地下室`使用名为的存储提供程序`PubSubStore酒店`. 与任何Grain一样，您可以为存储提供程序指定实现。对于流式发布订阅，您可以更改`PubSubStore酒店`在silos构建时，使用思洛主机生成器：

下面将Pub-Sub配置为将其状态存储在Azure表中。

```csharp
hostBuilder.AddAzureTableGrainStorage("PubSubStore", 
    options=>{ options.ConnectionString = "Secret"; });
```

这样，发布订阅数据将持久地存储在Azure表中。对于初始开发，您也可以使用内存存储。除了Pub-Sub之外，Orleans流媒体运行时还从生产者向消费者传递事件，管理分配给活动使用流的所有运行时资源，并透明地从未使用的流中垃圾收集运行时资源。

### 配置<a name="Configuration"></a>

为了使用流，您需要通过思洛主机或群集客户端构建器启用流提供程序。您可以阅读有关流提供程序的更多信息[在这里](stream_providers.md). 示例流提供程序设置：

```csharp
hostBuilder.AddSimpleMessageStreamProvider("SMSProvider")
  .AddAzureQueueStreams<AzureQueueDataAdapterV2>("AzureQueueProvider",
    optionsBuilder => optionsBuilder.Configure(
      options=>{ options.ConnectionString = "Secret"; }))
  .AddAzureTableGrainStorage("PubSubStore",
    options=>{ options.ConnectionString = "Secret"; });
```

## 下一个

[Orleans河流供应商](stream_providers.md)
