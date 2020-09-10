---
layout: page
title: Cluster Management in Orleans
---

# Orleans的集群管理

Orleans通过一个内置的成员协议提供集群管理，我们有时将其称为**silos成员资格**. 为了让新Orleans的所有服务器都加入到silos(silos)的协议中，目前还没有达成一致，即允许新Orleans的silos加入到silos中。

协议依赖于一个外部服务来提供`成员资格表`. `成员资格表`是一个平面的非SQL类型的持久表，我们有两个用途。首先，它被用作一个集合点，供各自为政的人互相寻找，而Orleans的客户则用来寻找silos。其次，它用于存储当前的成员资格视图(活动silos列表)，并帮助协调成员资格视图上的协议。我们目前有6个`成员资格表`：基于[Azure表存储](https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-how-to-use-tables)，SQL server，[阿帕奇动物园管理员](https://ZooKeeper.apache.org/), [IO领事](https://www.consul.io), [AWS发电机B](https://aws.amazon.com/dynamodb/)，以及内存仿真以进行开发。除了`成员资格表`每个silos都参与完全分布式的对等成员身份协议，该协议检测出故障的silos，并就“设置活动silos”达成一致。我们首先在下面描述Orleans成员协议的内部实现，然后描述`成员资格表`.

### 基本成员协议：

1.  在启动时，每个silos都将自己写入一个著名的`成员资格表`(通过配置传递)。silos标识的组合(`ip地址：端口：epoch`)和服务部署id用作表中的唯一键。Epoch就是这个silos开始时的计时单位`ip地址：端口：epoch`保证在给定的Orleans部署中是独一无二的。

2.  silos通过应用程序ping相互直接监视(“你还活着吗”`心跳`). ping作为直接消息从silos发送到silos，通过silos通信的同一个TCP套接字。这样，ping与实际的网络问题和服务器运行状况完全相关。每一个silos都会发出另一个silos的声音。一个silos通过计算其他silos标识上的一致散列值来挑选要ping的对象，形成一个包含所有标识的虚拟环，并在该环上选择X个后继silo(这是一种著名的分布式技术，称为[一致哈希](https://en.wikipedia.org/wiki/Consistent_hashing)在许多分布式哈希表中被广泛使用，比如[弦式DHT](https://en.wikipedia.org/wiki/Chord_(peer-to-peer))).

3.  如果silosS没有从受监视的服务器P获得Y ping响应，它会通过将其带有时间戳的怀疑写入P的行中来怀疑它`成员资格表`.

4.  如果P在K秒内有超过Z个怀疑，那么S会将P写入P的行中，并向所有silos广播一个重新读取成员表的请求(无论如何，它们都会定期执行)。

5.  更多详情：

    5.1怀疑被写入`成员资格表`，在对应于P的行中的一个特殊列中。当S怀疑P时，它写下：“at time TTT S successed P”。

    5.2一次怀疑不足以宣告P死亡。您需要在一个可配置的时间窗口T(通常为3分钟)内，从不同的silos中得到Z个怀疑，才能将P声明为dead。使用由`成员资格表`.

    5.3可疑silosS显示P行。

    5.4如果S是最后一个嫌疑犯(如嫌疑栏中所述，在时间段T内已经有Z-1嫌疑犯)，S决定宣布P死亡。在本例中，S将自己添加到suspencers列表中，并在P的Status列中写入P is Dead。

    5.5否则，如果S不是最后一个suspencers，则S只将自己添加到suspencers列中。

    5.6在任何一种情况下，回写都使用读取的版本号或etag，因此对此行的更新是序列化的。如果由于版本/etag不匹配而导致写入失败，S将重试(再次读取并尝试写入，除非P已标记为dead)。

    5.7在较高的层次上，“读取、本地修改、回写”序列是一个事务。但是，我们没有使用存储事务来实现这一点。“事务”代码在服务器上本地执行，我们使用`成员资格表`以确保隔离和原子性。

6.  每个silos定期读取整个成员表以进行部署。通过这种方式，silos了解到新的silos加入以及其他silos被宣布死亡。

7.  **配置**：我们提供了一个默认配置，它是在我们在Azure中的生产使用过程中手动调整的。目前的默认值是：每个silos由另外3个silos监视，2个怀疑足以宣布一个silos失效，怀疑仅从最后3分钟开始(否则它们就过时了)。每10秒发送一次ping，您需要错过3次ping才能怀疑一个silos。

8.  **实施完美故障检测**–理论上，如果一个silos与其他silos失去通信，而silos进程本身仍在运行，则该silos将被宣布为死机。为了解决这个问题，一旦silos在表中被声明为dead，每个人都认为它已经死了，即使它实际上没有死(只是暂时分区或者心跳消息丢失)。每个人都停止与它交流，一旦它知道它死了(通过从表中读取它自己的新状态)，它就会自杀并关闭进程。因此，必须有一个适当的基础设施来重新启动silos作为一个新的进程(在启动时会生成一个新的epoch编号)。当它托管在Azure中时，会自动发生这种情况。如果没有，就需要另一个基础设施。例如，配置为在发生故障时自动重新启动的Windows服务。

9.  **优化以减少定期表读取的频率，并加快所有silos学习新的连接silos和死silos**. 每当任何silos成功地向表中写入任何内容(怀疑、新联接，…)时，它也会广播到所有其他silos—“现在就去重新读取表”。silos不会告诉其他人它在表中写了什么(因为这些信息可能已经过时/错误)，它只是告诉他们重新读取表。这样我们就可以很快地了解成员身份的变化，而不需要等待整个周期性的读取周期。我们仍然需要定期读取，以防“重新读取表”消息丢失。

### 基本成员协议的属性和常见问题解答：

1.  **可以处理任何数量的失败**–我们的算法可以处理任何数量的失败(即f\<=n)，包括完全重启集群。这与“传统”形成鲜明对比[帕克索斯](https://en.wikipedia.org/wiki/Paxos_(computer_science))基于解决方案，需要法定人数，通常是多数。我们已经看到在生产情况下，一半以上的silos都关闭了。我们的系统仍然正常运行，而基于Paxos的会员制将无法取得进展。
2.  **去餐桌的交通非常少**-实际的ping直接在服务器之间进行，而不是发送到表中。这将产生大量的流量，而且从故障检测的角度来看是不够准确的-如果一个silos无法到达表，它将无法写入它的“我是活着的”心跳，其他人会杀死他。
3.  **可调精度与完整性** – [一般来说，完美和准确的故障检测是不可能的](http://www.cs.yale.edu/homes/aspnes/pinewiki/FailureDetectors.html). 人们通常希望能够在准确性和完整性之间进行权衡(不想将一个真正活着的silos声明为死silos)和完整性(希望尽快声明一个实际上已经死了的silos)。可配置的声明死亡和错过ping的投票允许交易这两个。

4.  **比例尺**-基本协议可以处理数千个甚至可能是数万个服务器。这与传统的[帕克索斯](https://en.wikipedia.org/wiki/Paxos_(computer_science))基于解决方案，如组通信协议，已知其规模不会超过10个。

5.  **诊断学**-该表也非常便于诊断和故障排除。系统管理员可以立即在表中找到活动的silos的当前列表，以及查看所有被杀死的silos和怀疑的历史。这在诊断问题时特别有用。

6.  **为什么我们需要可靠的持久存储来实现`成员资格表`?**-我们为`成员资格表`有两个目的。首先，它被用作一个集合点，供各自为政的人互相寻找，而Orleans的客户则用来寻找silos。第二，我们使用可靠的存储来帮助我们协调成员观点上的协议。当我们以对等的方式在silos之间直接执行故障检测时，我们将成员资格视图存储在一个可靠的存储器中，并使用该存储器提供的并发控制机制来达成谁活着谁死的协议。这样，在某种意义上，我们的协议将分布式共识的难题外包给了云。因为我们充分利用了底层云平台的力量，真正将其作为“平台即服务”。

7.  **如果一段时间内无法访问该表，会发生什么情况？**(存储服务关闭、不可用或存在通信问题)–在这种情况下，我们的协议不会错误地宣布silos死机。目前运行的silos将继续工作，没有任何问题。但是，我们不能声明一个silos死机(如果我们通过丢失的ping检测到一些silos死机，我们将无法将此事实写入表中)，也无法允许新的silos加入。所以完整性会受到影响，但是准确性不会——从表中进行分区永远不会导致我们错误地宣布思洛死了。另外，在部分网络分区的情况下(如果一些silos可以访问表，而另一些silos不能访问表)，我们可能会将一个死silos声明为死silos，但在所有其他silos了解到它之前，还需要一些时间。所以检测可以延迟，但我们绝不会因为表格不可用而误杀他人。

8.  **直接IAmAlive写入表仅用于诊断**-除了在silos之间发送的心跳信号外，每个silos还定期更新表中其行中的“我还活着”列。“我还活着”一栏只供使用**用于手动故障排除和诊断**并且不被成员协议本身使用。它通常以较低的频率编写(每5分钟一次)，它是系统管理员检查集群活动性或轻松找出silos上次活动的时间的非常有用的工具。

### 完全订购成员资格视图的扩展：

上面描述的基本成员协议后来被扩展为支持完全有序的成员关系视图。我们将简要描述这个扩展的原因以及它是如何实现的。扩展在上述设计中没有任何改变，只是添加了一个附加属性，即所有成员配置都是全局完全有序的。

**为什么完全订购成员视图是有用的？**

-   这允许序列化新silos到集群的连接。这样，当一个新的silos加入集群时，它就可以验证与其他已经启动的silos的双向连接。如果一些已加入的silos没有应答(可能表明新silos存在网络连接问题)，则不允许新silos加入。这可以确保至少在一个silos启动时，集群中所有silos之间都有完整的连接(这是实现的)。

-   silos中的更高级别协议(如分布式grain directory)可以利用成员关系视图被排序的事实，并使用这些信息来执行更智能的重复激活解析。特别是，当directory发现在成员身份不断变化时创建了2个激活，它可能会决定停用基于现在过时的成员身份信息创建的旧激活(目前尚未实现)。

**扩展成员协议：**

1.  为了实现此功能，我们利用了`成员资格表`..

2.  我们向跟踪表更改的表添加成员身份版本行。

3.  当silo S想为silo P写怀疑或死亡声明时：

    3.1s读取最新的表格内容。如果P已经死了，什么也不做。否则，

    3.2在同一事务中，将更改写入P的行，并增加版本号并将其写回表中。

    3.3两次写入都使用ETag进行调节。

    3.4如果事务因P行或版本行上的eTag不匹配而中止，请重试。

4.  所有对表的写入都会修改和增加版本行。这样，所有对表的写入都被序列化(通过将更新序列化到version行)，并且由于silos只增加版本号，所以写入操作也完全按递增顺序排列。

**扩展成员协议的可扩展性：**

在协议的扩展版本中，所有写操作都通过一行进行序列化。这可能会损害集群managemenet协议的可伸缩性，因为它增加了并发表写入之间发生冲突的风险。为了部分缓解这个问题，silos通过使用指数回退来重试它们对表的所有写入操作。我们已经观察到扩展的协议在Azure中有多达200个silos的生产环境中能够顺利地工作。然而，我们确实认为该协议在扩展到一千个silos之外可能存在问题。在这样的大型设置中，对版本行的更新很容易被禁用，基本上保留了集群managemenet协议的其余部分，并放弃了total ordering属性。还请注意，这里我们指的是集群管理协议的可伸缩性，而不是Orleans的其他地方。我们相信Orleans运行时的其他部分(消息传递、分布式目录、grain托管、客户端到网关连接)的可扩展性远远超过了数百个Silo。

### 成员表：

如前所述，`成员资格表`它被用作一个集合点，供思洛寻找彼此和Orleans客户查找思洛，还帮助协调成员关系视图上的协议。我们目前有6个`成员资格表`：基于Azure Table、SQL server、Apache ZooKeeper、Consult IO、AWS DynamoDB和内存仿真进行开发。的接口`成员资格表`在中定义[**`IMembershipTable`**](https://github.com/dotnet/orleans/blob/master/src/Orleans/SystemTargetInterfaces/IMembershipTable.cs).

1.  [Azure表存储](https://docs.microsoft.com/en-us/azure/storage/storage-dotnet-how-to-use-tables)-在这个实现中，我们使用Azure部署ID作为分区键和silos标识(`ip地址：端口：epoch`)作为行键。它们一起保证每个silos都有一个唯一的密钥。对于并发控制，我们使用基于[Azure表ETag](https://docs.microsoft.com/en-us/rest/api/storageservices/Update-Entity2). 每次从表中读取时，我们为每个读取行存储etag，并在尝试回写时使用该etag。每次写入时，Azure表服务都会自动分配和检查etag。对于多行事务，我们利用[Azure表提供的批处理事务](https://docs.microsoft.com/en-us/rest/api/storageservices/Performing-Entity-Group-Transactions)，它保证在具有相同分区键的行上序列化事务。

2.  SQL Server—在此实现中，配置的部署ID用于区分部署以及哪些silos属于哪些部署。silos标识定义为`部署ID，ip，端口，epoch`在适当的表和列中。关系后端使用乐观并发控制和事务，类似于在Azure表实现上使用etag的过程。关系实现期望数据库引擎生成所使用的ETag。对于SQL Server，在SQL Server 2000上，生成的ETag是通过调用[新ID()](https://www.microsoft.com/en-us/download/details.aspx?id=51958). 在SQL Server 2005及更高版本上[行版本](https://docs.microsoft.com/en-us/sql/t-sql/data-types/rowversion-transact-sql)被使用。Orleans读写关系etag是不透明的`变量二进制(16)`标记并将它们存储在内存中作为[基准64](https://en.wikipedia.org/wiki/Base64)编码字符串。Orleans支持使用UNION ALL(对于Oracle包括DUAL)的多行插入，UNION ALL当前用于插入统计数据。SQL Server的具体实现和基本原理请参见[创建或清理马厩\_sql服务器.sql](https://github.com/dotnet/orleans/blob/ba30bbb2155168fc4b9f190727220583b9a7ae4c/src/OrleansSQLUtils/CreateOrleansTables_SqlServer.sql).

3.  [阿帕奇动物园管理员](https://ZooKeeper.apache.org/)-在这个实现中，我们使用配置的部署ID作为根节点和silos标识(`ip:端口@epoch`)作为其子节点。它们共同保证了每个silos的唯一路径。对于并发控制，我们使用基于[节点版本](http://zookeeper.apache.org/doc/r3.4.6/zookeeperOver.html#Nodes+and+ephemeral+nodes). 每次从部署根节点读取时，都会存储每个读取子思洛节点的版本，并在尝试回写时使用该版本。每当一个节点的数据发生变化时，ZooKeeper服务就会自动地增加版本号。对于多行事务，我们使用[多方法](http://zookeeper.apache.org/doc/r3.4.6/api/org/apache/zookeeper/ZooKeeper.html#multi(java.lang.Iterable))，它保证在具有相同父部署ID节点的silos节点上可序列化事务。

4.  [IO领事](https://www.consul.io)-我们用过[执政官的钥匙/价值商店](https://www.consul.io/intro/getting-started/kv.html)推动membershop表。参考[执政官部署](../deployment/consul_deployment.md)更多细节。

5.  [AWS发电机B](https://aws.amazon.com/dynamodb/)-在这个实现中，我们使用集群部署ID作为分区键和silos标识(`ip端口生成`)作为使记录统一的RangeKey。乐观并发由`ETag`属性通过在DynamoDB上进行条件写入来实现。实现逻辑与Azure表存储非常相似。我们只实现了基本成员协议(而不是扩展协议)。

6.  内存模拟开发设置。我们使用一种特殊的系统grains，叫做`成员资格表`，以便实现。这些Grains存放在指定的主silos中，该silos仅用于**开发设置**. 在任何实际生产使用情况下，主silos**不是必需的**.

### 配置：

成员协议通过`活泼`元素`全球`分段`Orleans配置.xml`文件。默认值在Azure的生产使用年限中进行了调整，我们相信它们代表了良好的默认设置。一般来说，没有必要改变它们。

配置元素示例：

```xml
<Liveness ProbeTimeout = "5s" TableRefreshTimeout ="10s  DeathVoteExpirationTimeout ="80s" NumMissedProbesLimit = "3" NumProbedSilos="3" NumVotesForDeathDeclaration="2" />
```

实现了4种类型的活跃度。活动协议的类型通过`系统存储类型`属性`系统存储`元素`全球`分段`Orleans配置.xml`文件。

1.  `成员资格表`-成员表存储在主silos上的一个Grains中。这是一个**仅开发设置**.

2.  `可定制`-成员表存储在Azure表中。

3.  `SQL服务器`-成员表存储在关系数据库中。

4.  `动物园管理员`-成员表存储在ZooKeeper中[合奏](http://zookeeper.apache.org/doc/r3.4.6/zookeeperAdmin.html#sc_zkMulitServerSetup).

5.  `执政官`-配置为自定义系统存储`MembershipTableAssembly=“orleansConsultils”`.  参考[执政官部署](../deployment/consul_deployment.md)更多细节。

6.  `发电机B`-配置为自定义系统存储`MembershipTableAssembly=“OrleansAWSUtils”`.

对于所有活动类型，公共配置变量在中定义`全球化。活力`要素：

1.  `问题时间`-我要给他们的“发射室”发送信号。默认值为10秒。

2.  `表刷新超时`-从成员资格表获取更新的秒数。默认值为60秒。

3.  `死亡呼气超时`-成员表中死亡投票的过期时间(秒)。默认值为120秒

4.  `努米塞德普斯里米`t-从一个silos丢失的“我还活着”心跳信号消息的数量，或者导致怀疑该silos已死亡的未应答探测的数量。默认值为3。

5.  `纽普罗贝斯洛斯`-每个silos探测活跃度的silos数量。默认值为3。

6.  `NumVotesForDeath声明`-宣布某个silos死亡所需的未过期投票数(最多应为numissedprobeslimit)。默认值为2。

7.  `闲言碎语`-是否使用八卦优化来加速活跃度信息的传播。默认值为true。

8.  `IAmAliveTablePublishTimeout`-定期在成员资格表中写入此silos处于活动状态的秒数。仅用于诊断。默认值为5分钟。

9.  `NumMissedTableIAmAliveLimit`-表中从silos中丢失的导致记录警告的“我还活着”更新的数量。不影响活动协议。默认值为2。

10. `MaxJoinAttemptTime`-在放弃之前尝试加入一个silos群的秒数。默认值为5分钟。

11. `预期群集大小`-群集的预期大小。不必很准确，可以高估。用于优化要写入Azure表的重试的指数退避算法。默认值为20。

### 设计原理：

一个很自然的问题是为什么不完全依赖[阿帕奇动物园管理员](https://ZooKeeper.apache.org/)对于集群成员身份实现，可能会使用[具有短暂节点的群成员关系](http://zookeeper.apache.org/doc/trunk/recipes.html#sc_outOfTheBox)? 为什么我们要费心实施我们自己的会员协议？主要有三个原因：

(一)**在云中部署/托管**-Zookeeper不是一个托管服务(至少在撰写本文的2015年7月，当然，当我们在2011年夏天首次实现该协议时，没有任何一个主流云提供商将Zookeeper作为托管服务运行)。这意味着在云环境中，Orleans的客户必须部署/运行/管理自己的ZK集群实例。这只是又一个不必要的负担，我们不想强迫我们的客户。通过使用Azure表，我们依赖于托管的托管服务，这使我们的客户的生活更加简单。*基本上，在云计算中，将云用作平台，而不是基础设施。*另一方面，在本地运行和管理自己的服务器时，依赖ZK作为`成员资格表`是一个可行的选择。

(二)**直接故障检测**-当将ZK的组成员资格用于临时节点时，将在Orleans服务器(ZK客户端)和ZK服务器之间执行故障检测。这可能不一定与Orleans服务器之间的实际网络问题相关。*我们希望故障检测能够准确地反映集群内通信的状态。*具体来说，在我们的设计中，如果Orleanssilos不能与`成员资格表`它不被认为是死的，可以继续工作。与此相反，如果我们将ZK组成员资格用于临时节点，那么与ZK服务器的断开连接可能会导致Orleans silo(ZK client)被声明为dead，而实际上它可能是活动的并且完全正常工作。

(三)**便携性和灵活性**-作为Orleans哲学的一部分，我们不想强迫任何特定的技术，而是要有一个灵活的设计，在这种设计中，不同的组件可以很容易地用不同的实现进行切换。这正是紫色`成员资格表`抽象服务。

### 致谢：

我们要感谢[亚历克斯·科根](https://www.linkedin.com/in/alex-kogan-3a2b52)本协议的第一版的设计与实现。这项工作是2011年夏天微软研究所暑期实习的一部分。基于ZooKeeper的实现`成员资格表`是由谁完成的[夏伊·哈佐](https://github.com/shayhatsor)SQL的实现`成员资格表`是由谁完成的[维科·伊娃](https://github.com/veikkoeeva)AWS DynamoDB的实现`成员资格表`是由谁完成的[古腾堡里贝罗](https://github.com/galvesribeiro/)并以consur为基础实现`成员资格表`是由谁完成的[保罗·诺斯](https://github.com/PaulNorth).
