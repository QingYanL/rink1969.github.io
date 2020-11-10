---
title: CRDT与Local First
published: true 
---

## 问题

在开发[CITA-Cloud](https://github.com/cita-cloud)的过程中，一个比较复杂的功能就是交易池和分叉树的维护。

用户发送的交易会通过广播扩散到各个节点，经过检查之后进入交易池。在合适的时间，节点会从交易池中挑选部分交易，打包成`proposal`加入分叉树，等待最终确认。

因为是分布式系统，交易进入各个节点交易池的顺序可能不同。打包`proposal`时，因为是各个节点自己随意挑选的交易，不同块之间也会存在重复的情况。各个节点往分叉树上增加新的`proposal`时，都是在自己认为的最长链后面增加，所以也会存在分叉的情况。

等过了一段时间之后，共识确认出了最长链，可能跟本节点原先认为的不同，这时就需要一个类似`git`切换分支的操作。这个操作比较复杂，分叉树以及交易池的状态都需要相应的更新。目前的实现简单粗暴，所以一直在思考这里应该是怎么样的一种算法和数据结构。

## `CRDT`

前一段时间看了参考资料1这篇文章，里面提到了`CRDT`。虽然描述的问题场景是协作软件，比如`google doc`，但是感觉跟前述问题非常相似。于是产生了深入了解`CRDT`的想法，接着就找到了参考资料2。

在分布式系统领域有一个`CAP`原则，大致是说`C`（一致性），`A`（可用性）和`P`（容忍网络分区）这三个诉求，只能满足其中的两个，而无法同时满足三个。通常认为在设计系统的时候`P`是一定要满足的，因为网络分区的情况并不是软件系统能够控制的，那么就只能在`C`和`A`之间二选一了。

但是参考资料2提出了一个新的角度。

首先`CAP`的粒度太粗了，一致性有不同程度的一致性，可用性也可以分成很多等级，这个很早之前已经有人提出批评了。其实网络分区也可以按照时延细分出很多不同情况，比如共识算法里常说的同步网络，半同步网络，异步网络（参见[区块链提供什么类型的一致性？](https://rink1969.github.io/Blockchain-consistency_model)）。

很多选择也不像表面看起来那么简单。比如`PBFT`类共识算法，通常针对半同步网络，采用超时翻倍的方式进行无限重试。表面上看好像是选择了`A`，系统一直在活动，并没有停下来，但是从实际效果上就是选择了`C`，并没有可用性可言，因为没有实际出块。

文章提出的思路是，虽然`P`是不受控制的，但是我们也不能逃避，而是应该正面面对它。软件应该主动去检测网络分区情况是否发生，如果发生了，则在此期间需要对操作进行一些限制，保证系统的一些不变量不被破坏，以便将来网络分区恢复之后，可以自然的将节点各自的修改合并到一起。这其实是一种在分区期间保持可用性，但是一致性降低为最终一致性的做法。

这个思路对于区块链来说非常有意义。回头看我们前面描述的问题，因为是一种去中心化的场景，各个节点都是根据自己本地的情况做决策的，其实就相当于时刻处于一种网络分区的情况。也许我们可以使用`CRDT`技术来尽量减少各个节点的无效工作。

## Local First

与`CRDT`一起出现的还有一个名词`Local First`。它指的是类似`google doc`这样一种应用，网络连接正常的时候可以多人协同编辑一份文档；而网络中断的情况下，每个参与者还是可以正常的编辑文档，软件的功能基本不受影响，只是期间看不到其他协作者的修改；等到网络连接恢复，会自动同步并合并多个参与者期间对文档的修改。当然对于开发者来说，还有一个更加熟悉的例子，那就是`git`。

`Local First`有很多优势：可用性高，离线状态也可以继续使用；安全，本地保存数据；隐私，只传递必要的交互数据。参见参考资料3和4.

其实[CITA-Cloud闲谈](https://rink1969.github.io/cita-cloud)中描述的未来的场景，就跟`Local First`很像。之前还在想这样的架构跟直接远程调用`SaaS`服务有什么区别，`Local First`倒是做了一个很好的总结。

另外一个类比是暗黑之类的战网形式的网络游戏。单机也可以玩，接入战网就可以和其他玩家互动。这要求本地有完整的游戏逻辑，数据可以来自本地，也可以来自网络。实际要求的是业务逻辑和数据在设计上的分离。战网其实提供了游戏之外额外的价值，比如单机打到的设备无法交易，但是战网上打到的设备可以交易，类似于区块链上的数字资产。

## 未来工作

`CRDT`有两种实现方式：一种是操作本身是可交换的，自然可以合并。比如超市买东西，最终的总价和你买东西的顺序是无关的；另一种是描述的值在一个半格结构上面，`merge`的时候有汇聚性。对于前一种方式，因为操作会比较受限，可能在某些特性的场景会有应用。对于后一种情况，其实在区块链里并不鲜见，其实pow就是赋予各个分叉链一个半格结构，从而可以自然地从多个分叉链中选择出主链。

`CRDT`的核心是如何解决冲突，目前针对的都是比较简单的协作场景。在区块链这样的拜占庭场景下，所有的操作都需要是可信的，因此会有很多哈希等操作，而这些操作很多是不具有可交换性的。另外，节点之间是相互不信任的，可能会拒绝别其他节点增加的交易，这种情况下如何解决冲突就变得很复杂了。

要在区块链中应用`CRDT`，有两种方式：一种是在共识前阶段，比如交易池和分叉树使用，这是要解决操作受限和拜占庭情况引发的冲突的问题。另外一种是在共识后，因为共识本身就是解决冲突的，可以认为共识之后的应用层就不存在拜占庭情况了，可以尝试基于`CRDT`实现一个应用编程模式。

## 参考

1. [I was wrong. CRDTs are the future](https://josephg.com/blog/crdts-are-the-future/)
2. [CAP Twelve Years Later: How the "Rules" Have Changed](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed)
3. [不要在云上保存你的数据（一）：本地优先的七个理念](https://www.infoq.cn/article/kpiK-qYmJGuzX12ejvaG)
4. [不要在云上保存你的数据（二）：未来软件发展途径](https://www.infoq.cn/article/TAJhsV1-BZWmbLJ7NrYr)








