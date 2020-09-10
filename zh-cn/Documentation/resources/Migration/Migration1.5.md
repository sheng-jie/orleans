---
layout: page
title: Migration from Orleans 1.5 to 2.0
---

# 从Orleans迁移到0.5

Orleans的大部分API在2.0中保持不变，或者这些API的实现留在遗留类中以实现向后兼容性。同时，新引入的api提供了一些新的功能或更好的方法来完成这些任务。当涉及到.NETSDK工具和VisualStudio支持时，还有更细微的区别，这有助于我们意识到这一点。本文档为将应用程序代码从迁移到Orleans 2.0提供了指导。

## Visual Studio和工具要求

Orleans2.0.0建立在.NET标准2.0之上。因此，您需要升级开发工具，以确保您获得愉快的开发体验。我们建议使用Visual Studio 2017或更高版本来开发Orleans 2.0.0应用程序。根据我们的经验，15.5.2及更高版本的效果最好。NET Standard 2.0.0与.NET 4.6.1及更高版本、.NET Core 2.0和其他框架列表兼容。Orleans2.0.0继承了这种兼容性。有关与其他.NET标准的兼容性信息，请参阅.NET[.NET标准文档](https://docs.microsoft.com/en-us/dotnet/standard/net-standard)：如果您正在使用Orleans开发.NET Core或.NET应用程序，则需要按照某些步骤来设置环境，例如安装.NET Core SDK。更多信息，请参考[文档](https://dotnet.github.io/).

## 配置代码的可用选项

## 群众或部队的集合

### 配置和启动silos(使用新的SiloBuilder API和旧的ClusterConfiguration对象)

Orleans 2.0中有许多新的选项类，它们为配置silos提供了一种新的方法。为了便于迁移到新的API，有一个可选的向后兼容包，`Microsoft.Orleans.Runtime.遗产`，它提供了从旧的1.x配置API到新的配置API的桥。

如果你加上`Microsoft.Orleans.Runtime.遗产`包，silos仍然可以通过传统的`群集配置`对象，然后可以传递给`SiloHostBuilder`建造和启动silos。

您仍然需要通过`配置应用程序部件`调用。

以下是如何以传统方式配置本地silos的示例：

```csharp
public class Program
{
    public static async Task Main(string[] args)
    {
        try
        {
            var host = await StartSilo();
            Console.WriteLine("Press Enter to terminate...");
            Console.ReadLine();

            await host.StopAsync();

            return 0;
        }
        catch (Exception ex)
        {
            Console.WriteLine(ex);
            return 1;
        }
    }

    private static async Task<ISiloHost> StartSilo()
    {
        // define the cluster configuration (temporarily required in the beta version,
        // will not be required by the final release)
        var config = ClusterConfiguration.LocalhostPrimarySilo();
        // add providers to the legacy configuration object.
        config.AddMemoryStorageProvider();
            
        var builder = new SiloHostBuilder()
            .UseConfiguration(config)
            // Add assemblies to scan for grains and serializers.
            // For more info read the Application Parts section
            .ConfigureApplicationParts(parts =>
                parts.AddApplicationPart(typeof(HelloGrain).Assembly)
                     .WithReferences())
            // Configure logging with any logging framework that supports Microsoft.Extensions.Logging.
            // In this particular case it logs using the Microsoft.Extensions.Logging.Console package.
            .ConfigureLogging(logging => logging.AddConsole());

        var host = builder.Build();
        await host.StartAsync();
        return host;
    }
}
```

### 配置和连接客户端(使用新的ClientBuilder API和旧的ClientConfiguration对象)

orleans2.0中有许多新的选项类，它们为配置客户端提供了一种新的方法。为了便于迁移到新的API，有一个可选的向后兼容包，`Microsoft.Orleans.Core.遗产`，它提供了从旧的1.x配置API到新的配置API的桥。

如果你添加了`Microsoft.Orleans.Core.遗产`包，客户端仍然可以通过旧版本以编程方式配置`客户端配置`对象，然后可以传递给`客户端生成器`来构建和连接客户端。

您仍然需要通过`配置应用程序部件`调用。

下面是一个示例，说明客户端如何使用传统配置连接到本地silos：

```csharp
// define the client configuration (temporarily required in the beta version,
// will not be required by the final release)
var config = ClientConfiguration.LocalhostSilo();
var builder = new ClientBuilder()
    .UseConfiguration(config)
    // Add assemblies to scan for grains interfaces and serializers.
    // For more info read the Application Parts section
    .ConfigureApplicationParts(parts => parts.AddApplicationPart(typeof(IHello).Assembly))
    .ConfigureLogging(logging => logging.AddConsole())
var client = builder.Build();
await client.Connect();
```

## 登录中

Orleans 2.0使用与ASP.NET核心2.0。您可以在中找到大多数Orleans日志记录功能的替代品ASP.NET岩芯测井。Orleans特定的日志记录功能，例如`ILogConsumer`和消息膨胀，仍然保持在`Microsoft.Orleans.Logging.遗产`包，这样你仍然可以选择使用它们。但是如何在2.0中使用Orleans配置日志记录发生了变化。让我来给你介绍一下迁移的过程。

在1.5中，日志配置是通过`客户端配置`和`节点配置`. 您可以配置`默认跟踪级别`, `跟踪文件名`, `跟踪模式`, `TraceLevelOverrides`, `跟踪控制台`, `批量消息限制`, `木材消费者`等等。在2.0中，日志记录配置与ASP.NETcore2.0日志记录，这意味着大部分配置都是通过`Microsoft.Extensions.Logging.ILoggingBuilder`. 

配置`默认跟踪级别`和`TraceLevelOverrides`，你需要申请[日志过滤](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging)到`ILoggingBuilder`. 例如，要在orleans运行时中将跟踪级别设置为“Debug”，可以使用下面的示例，

```
siloBuilder.AddLogging(builder=>builder.AddFilter("Orleans", LogLevel.Debug));
```

您可以用同样的方法为应用程序代码配置日志级别。如果要将默认的最小跟踪级别设置为调试，请使用下面的示例

```
siloBuilder.AddLogging(builder=>builder.SetMinimumLevel(LogLevel.Debug);
```

有关日志筛选的详细信息，请参阅<https://docs.microsoft.com/en-us/aspnet/core/fundamentals/logging>;

将TraceToConsole配置为`是的`，需要参考`Microsoft.Extensions.Logging.控制台`打包后使用`添加控制台()`扩展方法`ILoggingBuilder`. 同样的`跟踪文件名`和`跟踪模式`，如果要将消息记录到文件中，则需要使用`AddFile(“文件名”)`方法`ILoggingBuilder`.

如果您仍然想使用消息填充特性，您需要通过`ILoggingBuilder`也。消息填充功能存在于`Microsoft.Orleans.Logging.遗产`包裹。所以您需要首先添加对该包的依赖性。然后通过`ILoggingBuilder`. 下面是一个如何配置它的示例`ISiloHostBuilder`

```
       siloBuiler.AddLogging(builder => builder.AddMessageBulkingLoggerProvider(new FileLoggerProvider("mylog.log")));
```

此方法将对`FileLoggerProvider`，使用默认的填充配置。

由于我们将在将来最终弃用并删除LogConsumer特性支持，我们强烈建议您尽快迁移掉这个特性。有两种方法你可以采取迁移。一个选择是保持你自己的`iLogger提供程序`，这就创造了`窃听器`谁登录到所有现有的日志使用者。这和我们正在做的非常相似`Microsoft.Orleans.Logging.遗产`包裹。你可以看看`LegacyOrleanSlogger提供程序`从中借用逻辑。另一个选择是替换`ILogConsumer`现有的`iLogger提供程序`在nuget上，它提供相同或相似的功能，或者实现您自己的功能`iLogger提供程序`符合您特定的日志记录要求。并配置它们`iLogger提供程序`与`ILoggingBuilder`.

但如果您不能在短期内迁移非日志使用者，您仍然可以使用它。支持`ILogConsumer`生活在`Microsoft.Orleans.Logging.遗产`包裹。所以您需要首先添加对该包的依赖，然后通过扩展方法配置日志使用者`添加LegacyOrleansLogging`在`ILoggingBuilder`. 有本地人`添加日志记录`方法`IServiceCollection`提供单位ASP.NET供您配置[`ILoggingBuilder`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.loggingservicecollectionextensions.addlogging?view=aspnetcore-2.0#Microsoft_Extensions_DependencyInjection_LoggingServiceCollectionExtensions_AddLogging_Microsoft_Extensions_DependencyInjection_IServiceCollection_System_Action_Microsoft_Extensions_Logging_ILoggingBuilder). 我们还将该方法包装在`ISiloHostBuilder`和`IClientBuilder`. 所以你可以调用`添加日志记录`方法来配置silo builder和client builder`ILoggingBuilder`.  下面是一个例子：

```
            var severityOverrides = new OrleansLoggerSeverityOverrides();
            severityOverrides.LoggerSeverityOverrides.Add(typeof(MyType).FullName, Severity.Warning);
            siloBuilder.AddLogging(builder => builder.AddLegacyOrleansLogging(new List<ILogConsumer>()
            {
                new LegacyFileLogConsumer($"{this.GetType().Name}.log")
            }, severityOverrides));
```

如果您投资于的自定义实现，则可以使用此功能`ILogConsumer`无法将它们转换为`iLogger提供程序`短期内。

`Logger GetLogger(字符串loggerName)`方法`Grains`基类和`IProviderRuntime`，和`记录器日志{get；}`在2.0中，IStorageProvider上的方法仍作为不推荐使用的功能进行维护。您仍然可以在迁移Orleans遗留日志的过程中使用它。但我们建议你尽快离开它们。

## 提供程序配置

在Orleans 2.0中，包含的提供者的配置已经标准化，可以从`俱乐部选项`为silos或客户端配置。

服务ID是集群所代表的服务或应用程序的稳定标识符。随着时间的推移，在实施服务的群集的部署和升级之间，服务ID不会改变。

与服务ID不同，集群ID只在Silo集群的生命周期中保持不变。如果一个正在运行的集群被关闭，并且部署了同一服务的新集群，那么新集群将拥有一个新的、唯一的集群ID，但是将维护旧集群的服务ID。

服务ID通常用作密钥的一部分，用于持久化需要在整个服务生命周期中保持连续性的数据。例如Grain状态、提醒和持久流的队列。另一方面，集群成员关系表中的数据只在其集群的范围内有意义，因此通常是从集群ID中键入的。

在2.0之前，Orleans提供者的行为有时与使用服务ID和集群ID(以前也称为部署ID)不一致。由于这种统一性以及提供者配置API的整体变化，一些提供者写入存储器的数据可能会改变位置或密钥。对此更改敏感的提供程序示例是Azure队列流提供程序。

如果您正在将现有服务从1.x迁移到2.0，并且需要保持与您在服务中使用的提供程序持久化的数据的位置或键相关的向后兼容性，请验证数据是否位于您的服务或提供商预期的位置。如果您的服务碰巧依赖于1.x提供程序对服务ID和Cluster ID的错误使用，则可以重写`俱乐部选项`通过调用`ISiloHostBuilder.AddProviderClusterOptions()`或`IClientBuilder.AddProviderClusterOptions()`并强制它从存储器中的1.x位置读/写数据
