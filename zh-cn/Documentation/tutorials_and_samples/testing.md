---
layout: page
title: 单元测试
---

# 单元测试

本教程显示如何对Grains进行单元测试，以确保它们的行为正确。对Grains进行单元测试的主要方法有两种，选择的方法将取决于要测试的功能类型。这个`Microsoft.Orleans.TestingHost`NuGet包可用于为Grains创建测试silos，也可以使用模拟框架，例如[moq](https://github.com/moq/moq)模拟您与之交互的Orleans运行时的各个部分。

## 使用TestCluster

这个`Microsoft.Orleans.TestingHost`NuGet软件包包含`测试集群（TestCluster）`可以用来创建一个内存集群，默认情况下它由两个silos组成，可以用来测试Grains。

```csharp
using System;
using System.Threading.Tasks;
using Orleans;
using Orleans.TestingHost;
using Xunit;

namespace Tests
{
    public class HelloGrainTests
    {
        [Fact]
        public async Task SaysHelloCorrectly()
        {
            var cluster = new TestCluster();
            cluster.Deploy();

            var hello = cluster.GrainFactory.GetGrain<IHelloGrain>(Guid.NewGuid());
            var greeting = await hello.SayHello();

            cluster.StopAllSilos();

            Assert.Equal("Hello, World", greeting);
        }
    }
}
```

由于启动内存集群的开销，您可能希望创建一个`测试集群`并在多个测试案例中重复使用。例如，可以使用xUnit的类或集合夹具来完成此操作(请参见<https://xunit.github.io/docs/shared-context.html>更多细节)。

为了共享一个`测试集群`在多个测试用例之间，首先创建一个夹具类型：

```csharp
public class ClusterFixture : IDisposable
{
    public ClusterFixture()
    {
        this.Cluster = new TestCluster();
        this.Cluster.Deploy();
    }

    public void Dispose()
    {
        this.Cluster.StopAllSilos();
    }

    public TestCluster Cluster { get; private set; }
}
```

接下来创建一个集合夹具：

```csharp
[CollectionDefinition(ClusterCollection.Name)]
public class ClusterCollection : ICollectionFixture<ClusterFixture>
{
    public const string Name = "ClusterCollection";
}
```

您现在可以重复使用`测试集群`在您的测试用例中：

```csharp
using System;
using System.Threading.Tasks;
using Orleans;
using Xunit;

namespace Tests
{
    [Collection(ClusterCollection.Name)]
    public class HelloGrainTests
    {
        private readonly TestCluster _cluster;

        public HelloGrainTests(ClusterFixture fixture)
        {
            _cluster = fixture.Cluster;
        }

        [Fact]
        public async Task SaysHelloCorrectly()
        {
            var hello = _cluster.GrainFactory.GetGrain<IHelloGrain>(Guid.NewGuid());
            var greeting = await hello.SayHello();

            Assert.Equal("Hello, World", greeting);
        }
    }
}
```

xUnit将调用`ClusterFixture`类型的`Dispose`方法，当所有测试都已完成并且在内存中群集silos将停止时。`测试集群`也有一个接受的构造函数`TestClusterOptions`可用于配置集群中的silos。

如果您在silos中使用依赖注入来使服务可用于Grains，则也可以使用以下模式：

```csharp
public class ClusterFixture : IDisposable
{
    public ClusterFixture()
    {
        var builder = new TestClusterBuilder();
        builder.AddSiloBuilderConfigurator<TestSiloConfigurations>();
        this.Cluster = builder.Build();
        this.Cluster.Deploy();
    }

    public void Dispose()
    {
        this.Cluster.StopAllSilos();
    }

    public TestCluster Cluster { get; private set; }
}

public class TestSiloConfigurations : ISiloBuilderConfigurator {
    public void Configure(ISiloHostBuilder hostBuilder) {
        hostBuilder.ConfigureServices(services => {
            services.AddSingleton<T, Impl>(...);
        });
    }
}
```

## 使用Mocks

Orleans还使模拟系统的许多部分成为可能，并且在许多情况下，这是对grains进行单元测试的最简单方法。这种方法确实有局限性(例如，围绕调度重入和序列化)，并且可能要求Grain包含仅由单元测试使用的代码。的[OrleansTestKit](https://github.com/OrleansContrib/OrleansTestKit)提供了一种替代方法，可以绕开许多这些限制。

例如，让我们想象一下我们正在测试的Grains与其他Grains交互。为了能够模拟其他Grains，我们还需要模拟`grain工厂`被测Grains的成员。默认`grain工厂`是正常的`protected`属性，但大多数模拟框架要求将属性设置为`public`和`virtual`才能嘲笑他们。所以我们要做的第一件事就是`grain工厂`都`public`和`virtual`属性：

```csharp
public new virtual IGrainFactory GrainFactory
{
    get { return base.GrainFactory; }
}
```

现在，我们可以在Orleans运行时之外创建Grains，并使用模拟来控制`grain工厂`：

```csharp
using System;
using System.Threading.Tasks;
using Orleans;
using Xunit;
using Moq;

namespace Tests
{
    public class WorkerGrainTests
    {
        [Fact]
        public async Task RecordsMessageInJournal()
        {
            var data = "Hello, World";

            var journal = new Mock<IJournalGrain>();

            var worker = new Mock<WorkerGrain>();
            worker
                .Setup(x => x.GrainFactory.GetGrain<IJournalGrain>(It.IsAny<Guid>()))
                .Returns(journal.Object);

            await worker.DoWork(data)

            journal.Verify(x => x.Record(data), Times.Once());
        }
    }
}
```

在这里，我们创建受测Grains`WorkerGrain`，使用Moq表示我们可以覆盖`grain工厂`以便它返回一个模拟`IJournalGrain`。然后，我们可以验证我们的`WorkerGrain`与`IJournalGrain`如我们所料。
