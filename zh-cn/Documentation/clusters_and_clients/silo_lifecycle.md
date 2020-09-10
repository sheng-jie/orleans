---
layout: page
title: Silo Lifecycle
---

# silos生命周期

# 概述

Orleanssilos使用可观察的生命周期(参见[Orleans生命周期](../implementation/orleans_lifecycle.md))用于Orleans系统和应用层组件的有序启动和关闭。

## 阶段

Orleanssilos和集群客户端使用一组公共的服务生命周期阶段。

```csharp
public static class ServiceLifecycleStage
{
    public const int First = int.MinValue;
    public const int RuntimeInitialize = 2000;
    public const int RuntimeServices = 4000;
    public const int RuntimeStorageServices = 6000;
    public const int RuntimeGrainServices = 8000;
    public const int ApplicationServices = 10000;
    public const int BecomeActive = Active-1;
    public const int Active = 20000;
    public const int Last = int.MaxValue;
}
```

-   服务生命周期的第一阶段
-   运行时初始化-初始化运行时环境。silos初始化线程。
-   runtime services-启动运行时服务。silo初始化网络和各种代理。
-   RuntimeStorageServices-初始化运行时存储。
-   runtimegrainservices-启动grains的运行时服务。这包括Grains类型管理、成员服务和Grains目录。
-   应用服务–应用层服务。
-   变为活动-silos加入群集。
-   活动—silos在群集中处于活动状态，并准备接受工作负载。
-   服务生命周期的最后阶段

## 登录中

由于控制权倒置，参与者加入生命周期，而不是具有一些集中的初始化步骤集的生命周期，因此代码中并不总是清楚启动/关闭顺序是什么。为了帮助解决这个问题，在silos启动之前添加了日志记录，以报告每个阶段参与的组件。这些日志记录在`Orleans.Runtime.SiloLifecycleSubject`记录员。例如：

*信息，orleans.runtime.silolifecyclesubject，“阶段2000:orleans.statistics.perfcounterevironmentstatistics，orleans.runtime.insideruntimeclient，orleans.runtime.silo”*

*信息，orleans.runtime.silolifecyclesubject，“stage 4000:orleans.runtime.silo”*

*信息，orleans.runtime.silolifecyclesubject，“阶段10000:orleans.runtime.versions.grainversionstore，orleans.storage.azuretablegrainstorage-default，orleans.storage.azuretablegrainstorage pubstore”*

此外，对每个组件的定时和错误信息也进行了类似的逐级记录。例如：

*信息，orleans.runtime.silolifecyclesubject，“生命周期观察者orleans.runtime.insideruntimeclient在2000阶段启动，耗时33毫秒。”*

*信息，orleans.runtime.silolifecyclesubject，“生命周期观察者orleans.statistics.perfcounterenvironmentstatistics开始于2000阶段，耗时17毫秒。”*

## silos生命周期参与

应用程序逻辑可以通过在silos的服务容器中注册参与服务来参与silos的生命周期。服务必须注册为ILifecycleParticipant<ISiloLifecycle>是的。

```csharp
public interface ISiloLifecycle : ILifecycleObservable
{
}

public interface ILifecycleParticipant<TLifecycleObservable>
    where TLifecycleObservable : ILifecycleObservable
{
    void Participate(TLifecycleObservable lifecycle);
}
```

silos启动时，所有参与者(`循环参与人<Isilifecycle>`)在容器中，将有机会通过调用`参与(…)`行为。一旦所有人都有机会参与，silos的可观测生命周期将依次开始所有阶段。

## 例子

随着silo生命周期的引入，用于允许应用程序开发人员在提供程序初始化阶段注入逻辑的引导提供程序不再是必需的，因为应用程序逻辑现在可以在silo启动的任何阶段注入。尽管如此，我们还是添加了一个“启动任务”fa_ade来帮助使用引导提供程序的开发人员进行转换。作为如何开发参与silos生命周期的组件的示例，我们将查看启动任务fa_ade。

启动任务只需要继承自`循环参与人<Isilifecycle>`并在指定阶段将应用程序逻辑订阅到silos生命周期。

```csharp
class StartupTask : ILifecycleParticipant<ISiloLifecycle>
{
    private readonly IServiceProvider serviceProvider;
    private readonly Func<IServiceProvider, CancellationToken, Task> startupTask;
    private readonly int stage;

    public StartupTask(
        IServiceProvider serviceProvider,
        Func<IServiceProvider, CancellationToken, Task> startupTask,
        int stage)
    {
        this.serviceProvider = serviceProvider;
        this.startupTask = startupTask;
        this.stage = stage;
    }

    public void Participate(ISiloLifecycle lifecycle)
    {
        lifecycle.Subscribe<StartupTask>(
            this.stage,
            cancellation => this.startupTask(this.serviceProvider, cancellation));
    }
}
```

从上面的实现中，我们可以看到在startuptask的`参与(…)`调用它订阅配置阶段的silos生命周期，传递应用程序回调，而不是它自己的初始化逻辑。

需要在给定阶段初始化的组件将提供自己的回调，但模式是相同的。

现在我们有了一个startuptask，它将确保在配置阶段调用应用程序的钩子，我们需要确保startuptask参与Silo生命周期。为此，我们只需要在容器中注册它。

我们使用silohost构建器上的扩展函数来实现这一点。

```csharp
public static ISiloHostBuilder AddStartupTask(
    this ISiloHostBuilder builder,
    Func<IServiceProvider, CancellationToken, Task> startupTask,
    int stage = ServiceLifecycleStage.Active)
{
    builder.ConfigureServices(services =>
        services.AddTransient<ILifecycleParticipant<ISiloLifecycle>>(sp =>
            new StartupTask(
                sp,
                startupTask,
                stage)));
    return builder;
}
```

通过在silos的服务容器中将startuptask注册为标记接口`循环参与人<Isilifecycle>`，这向silos发出信号，表明此组件需要参与silos的生命周期。
