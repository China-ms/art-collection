---
layout: post
toc: true
title: iOS 开发中的重构
date: 2016-03-10 17:42:11.000000000 +08:00
permalink: /:title
tags: iOS
---


随着一个项目的开发的进行, 项目中代码行数会随着时间的推移, 而逐渐增长. 但是由于一些历史原因, 先人在软件开发的过程中, 没有注意到项目中可以复用的模块以及代码. 这样一个工程中就出现了很多复制粘贴产生的代码, 违反了 `DRY` 的原则, 使得项目难以维护.

## 为什么要重构

为什么要重构, 重构其实是一件持续性的工作, 工程师在进行开发的过程中难免会有无法考虑周全的地方, 而目前的很多 iOS, Android 项目其实都是业务驱动的, 而业务和需求的更改导致原来模块的设计不再适用于当前的状况.

而大多数工程师是懒惰的. 因为没有足够的推动力去修改当前模块的设计, 所以通过打补丁的方式实现需求, 最终可能会导致整个项目代码的失控, 陷入焦油坑.

重构, 在我看来就是改变原有的设计, 减少项目中冗余的代码的过程. 重构应该伴随着项目的开发进行, 而不是在整个项目要失控时, 才去做这件事情.

> 早重构, 及时重构

## 如何在 iOS 开发中完成项目的重构

因为在 iOS 开发中 `viewController` 非常不便于测试, 所以绝大多数 iOS 的工程中都是没有单元测试, 同时也是没有单元测试这个意识的, 集成测试什么的就更不必多说了. 如果我们的工程项目是用 Ruby 或者 Python 写的, 那么补齐所有的单元测试是重构之前最好要做额事情, 不过对于后端代码的重构, 我由于没有太多的经验只能说这么多了.

在 iOS 项目中, 既然没有单元测试, 如何能保证重构前后的模块效果完全一样呢. 这是一个非常麻烦的问题, 因为没有各种自动化测试工具或者代码, 我们真的无法保证重构前后的功能 100% 的相同, 只能通过"人肉测试"的方式来检验重构的正确性.

重构一个模块之前我们要首先了解什么样的模块是需要重构的, 我想需要重构的最常见的场景就是**违反了 DRY 原则**.

> 当一个项目中很多代码都是复制粘贴的, 那么这个项目就是有问题的了, 就要考虑复制粘贴的这部分代码是否需要抽象成一个可复用的模块.

而我遇到的这种情况就是违反了 `DRY` 原则, 大量复制粘贴的代码遍布整个工程的每一个角落, 使得整个项目难以维护.

### 设计模块

在重构之前我们不仅要熟悉需要重构的这部分代码的业务逻辑, 而且更有对于重构后的模块有着什么样的功能有着清晰而且明确的定义.

### 分步重构

如果需要重构的模块过于庞大, 你可以通过重构后的代码, 一部分一部分的代替原有的代码, 并删除掉废弃的功能和逻辑.

有的模块可能同时包含 `UI` 和逻辑, 那么我们可以选择先重构其中的 `UI` 部分代码, 将 `UI` 从原有的模块中剥离出来, 改变原有的设计. 再删除原有模块中的代码.


## 总结

重构其实是一个非常庞大的话题, 如果没有亲自重构一个模块和项目很难去理解如何去重构一个项目和重构的作用.

往往有的人会想重写整个项目的代码, 然而在大多数情况下, 原有的逻辑可能由于历史原因已经不知道该如何重写, 而且目前的项目代码并不是完全不可用. 重写项目相对于重构也往往更加容易, 但是需要的周期确实太长, 所以说是要重写还是重构是一个需要权衡的问题.

<iframe src="http://ghbtns.com/github-btn.html?user=draveness&type=follow&size=large" height="30" width="240" frameborder="0" scrolling="0" style="width:240px; height: 30px;" allowTransparency="true"></iframe>

Blog: [draveness.me](http://draveness.me)
