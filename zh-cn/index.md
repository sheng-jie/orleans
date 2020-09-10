<p align="center">
  <image src="https://raw.githubusercontent.com/dotnet/orleans/gh-pages/assets/logo_full.png" alt="Orleans logo" width="600px">
</p>

[![NuGet](https://img.shields.io/nuget/v/Microsoft.Orleans.Core.svg?style=flat)](http://www.nuget.org/profiles/Orleans)
[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/dotnet/orleans?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

### Orleans是一个跨平台框架，用于构建健壮，可扩展的分布式应用程序

Orleans建立在.NET开发人员生产力的基础上，并将其带入了分布式应用程序的世界，例如云服务。 Orleans可从单个本地服务器扩展到云中全局分布的高可用性应用程序。

Orleans采用了对象，接口，async/await和try/catch等熟悉的概念，并将其扩展到多服务器环境。这样，它可以帮助具有单服务器应用程序经验的开发人员过渡到构建弹性，可扩展的云服务和其他分布式应用程序。因此，Orleans通常被称为“分布式.NET”。

它是由[Microsoft Research](http://research.microsoft.com/projects/orleans/) 创建的，并介绍了[Virtual Actor Model](http://research.microsoft.com/apps/pubs/default.aspx?id=210931)作为一种新方法来构建面向云时代的新一代分布式系统。 Orleans的核心贡献是它的编程模型，它在不限制功能，以及对开发人员施加繁重约束的情况下，降低了高并发分布式系统固有的复杂性。

文档位于[此处](Documentation/index.md)。


### 文档说明
* Orleans中文文档，是通过机器翻译，加上人工校对而成，因为个人精力有限，校对工作目前只做了一部分，但会继续利用闲暇时间做下去，也欢迎各位读者积极参与进来。
* 中文文档目前位于个人仓库[sheng-jie/orleans docs分支的zh-cn目录下](https://github.com/sheng-jie/orleans/tree/docs/zh-cn)，其中1.5下的文件夹未翻译。
* 一些专业术语因无合适翻译予以保留，例如：Orleans，Silo，Grain，Actor 等等。
* 计划是在维护一段时间后，文档翻译通顺后再提PR合并到Orlans官方仓库下。[[doc] Multiple language support](https://github.com/dotnet/orleans/issues/6075)。
* 如有兴趣加入Orleans中国社区交流群，可以扫码加我微信，我邀请您进群。

![](./images/weixin.jpg)

### 贡献指引
感谢您的参与，主要有两种方式可以参与到文档改进中：
1. 点击文档网页右上方**[Improve this Doc]**链接，即可打开GitHub中对应当前文档的原生Markdown页面，再点击页面上的[🖍]铅笔图标，即可进行在线修改，修改完毕，提交创建PR即可。
2. Fork仓库[sheng-jie/orleans](https://github.com/sheng-jie/orleans)并Clone到本地目录，签出docs分支（`checkout docs`），进入`zh-cn`目录，即可修改，修改完毕后，创建PR即可。
