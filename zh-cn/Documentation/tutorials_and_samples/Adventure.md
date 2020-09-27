---
layout: page
title: Adventure
---

# 冒险

一个简单的多人文本冒险游戏，其灵感来自老式的、基于文本的冒险游戏。

### 说明书

1.  在 Visual Studio 打开 OrleansAdventure.sln 项目。你可以 [在这里](https://github.com/dotnet/orleans/tree/master/Samples/2.0/Adventure) 找到他。
2.  启动“AdventureSetup”项目。
3.  当AdventureSetup运行成功后，启动“AdventureClient”项目。
4.  然后系统会提示您在命令行中输入您的姓名。进入并开始游戏。

### 概述

AdventureSetup程序从 AdventureConfig.txt 中读取游戏配置("地图")。

它设置了一系列的“房间”，如森林、海滩、洞穴、空地等。这些位置连接到其他房间，以模拟游戏的位置和布局。该示例配置仅描述了少数位置。

房间里可以放钥匙、剑等“东西”。

AdventureClient程序设置您的播放器，并提供一个简单的基于文本的用户界面，允许您玩游戏。

您可以使用简单的命令语言在房间中移动并与事物进行交互，例如说“go north”或“take brass key”。

### 为什么使用Orleans？

Orleans允许游戏通过非常简单的C#代码来描述游戏，同时允许它扩展到大型多人游戏。为了使这个动机有意义，房间的迷宫需要非常大，并且需要同时支持大量玩家。使用Orleans的一个优点是，该服务可以应对增量而设计，以小规模运行它的开销并不显著，而且您可以确信，当需要时它将是可以扩展的。

### 它是如何建模的？

玩家和房间被建模为Grains。这些Grains使我们能够用每个Grains模型状态和功能来分发游戏。

像密钥这样的东西被建模为“plain old objects”——它们实际上只是简单的不可变的数据结构，并且可以在房间里和玩家之间移动，所以它们不需要设计为Grains。

### 可能的改进

1.  把地图弄得非常、非常大
2.  让铜钥匙打开一些东西
3.  允许玩家互相留言
4.  使食物和饮用水成为可能和有实际的作用。
