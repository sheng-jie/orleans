---
layout: page
title: Tutorial 1 Hello World
---

# 概述：Hello World

此概述与可用的Hello World示例应用程序相关联[这里](https://github.com/dotnet/orleans/tree/master/Samples/2.0/HelloWorld)。

Orleans的主要概念包括silos，客户和一种或多种Grains。创建Orleans应用程序涉及配置silos，配置客户端和开发Grain。

## 配置silos

通过以下方式以编程方式配置silos`SiloHostBuilder`和许多追加的选项（option）类。可以找到所有选项的列表[这里。](http://dotnet.github.io/orleans/Documentation/clusters_and_clients/configuration_guide/list_of_options_classes.html)

```csharp
[...]
        private static async Task<ISiloHost> StartSilo()
        {
            // define the cluster configuration
            var builder = new SiloHostBuilder()
                .UseLocalhostClustering()
                .Configure<ClusterOptions>(options =>
                {
                    options.ClusterId = "dev";
                    options.ServiceId = "HelloWorldApp";
                })
                .Configure<EndpointOptions>(options => options.AdvertisedIPAddress = IPAddress.Loopback)
                .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(HelloGrain).Assembly).WithReferences())
                .ConfigureLogging(logging => logging.AddConsole());

            var host = builder.Build();
            await host.StartAsync();
            return host;
        }
```

| 选项 | 用于 |
| --- | --- |
| `.UseLocalhostClustering()` | 将客户端配置为连接到本地主机上的silos。 |
| `集群选项` | ClusterId是Orleans群集的名称，对于silos和客户端，名称必须相同，以便彼此对话。ServiceId是用于应用程序的ID，并且在部署之间不得更改 |
| `EndpointOptions` | 这告诉silos在哪里听。在此示例中，我们使用了`回送`。 |
| `配置应用程序部件` | 将Grain类和接口程序集作为应用程序部分添加到您的orleans应用程序中。 |

加载配置后，将构建SiloHost，然后异步启动。

## 配置客户端

与silos类似，客户端通过以下方式配置`ClientBuilder`以及类似的追加的选项（option）类集合。

```csharp
        private static async Task<IClusterClient> StartClientWithRetries()
        {
            attempt = 0;
            IClusterClient client;
            client = new ClientBuilder()
                .UseLocalhostClustering()
                .Configure<ClusterOptions>(options =>
                {
                    options.ClusterId = "dev";
                    options.ServiceId = "HelloWorldApp";
                })
                .ConfigureLogging(logging => logging.AddConsole())
                .Build();

            await client.Connect(RetryFilter);
            Console.WriteLine("Client successfully connect to silo host");
            return client;
        }
```

| 选项 | 用于 |
| --- | --- |
| `.UseLocalhostClustering()` | 与SiloHost相同 |
| `集群选项` | 与SiloHost相同 |

可以找到有关配置客户端的更深入的指南[在配置指南的客户端配置部分中。](http://dotnet.github.io/orleans/Documentation/clusters_and_clients/configuration_guide/client_configuration.html)

## 开发一个grain

Grains是Orleans编程模型的关键原语。Grains是Orleans应用程序的基础，它们是隔离，分布和持久化的原子单位。Grains是代表应用程序实体的对象。就像经典的面向对象编程一样，grain封装了实体的状态，并在代码逻辑中对其行为进行了编码。Grains可以相互保持引用，并可以通过调用彼此通过接口公开的方法进行交互。

您可以在[Orleans文档的“核心概念”部分。](http://dotnet.github.io/orleans/Documentation/core_concepts/index.html)

这是Hello World Grain的代码主体：

```csharp
[...]
namespace HelloWorld.Grains
{
    public class HelloGrain : Orleans.Grain, IHello
    {
        Task<string> IHello.SayHello(string greeting)
        {
            logger.LogInformation($"SayHello message received: greeting = '{greeting}'");
            return Task.FromResult($"You said: '{greeting}', I say: Hello!");
        }
    }
}
```

您可以阅读，grain类实现一个或多个grain接口[在这里，在Grains部分。](http://dotnet.github.io/orleans/Documentation/grains/index.html))

```csharp
[...]
namespace HelloWorld.Interfaces
{
    public interface IHello : Orleans.IGrainWithIntegerKey
    {
        Task<string> SayHello(string greeting);
    }
}
```

## 组件如何协同工作

建立此编程模型是我们分布式面向对象编程的核心概念的一部分。SiloHost首先启动。然后，启动OrleansClient程序。OrleansClient的Main方法调用启动客户端的方法，`StartClientWithRetries()。`客户端被传递给`DoClientWork()`方法。

```csharp
        private static async Task DoClientWork(IClusterClient client)
        {
            // example of calling grains from the initialized client
            var friend = client.GetGrain<IHello>(0);
            var response = await friend.SayHello("Good morning, my friend!");
            Console.WriteLine("\n\n{0}\n\n", response);
        }
```

此时，OrleansClient创建对IHellograins的引用，并通过其接口IHello调用其SayHello()方法。此调用激活silos中的Grains。OrleansClient向激活的Grains发送问候语。Grains返回问候作为对OrleansClient的响应，OrleansClient在控制台上显示该问候。

## 运行示例应用

要运行示例应用程序，请参阅[自述文件。](https://github.com/dotnet/orleans/tree/master/Samples/2.0/HelloWorld)
