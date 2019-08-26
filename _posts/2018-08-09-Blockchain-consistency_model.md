---
title:  区块链提供什么类型的一致性？
published: true
---

### 一致性

一致性从强到弱分别为：

`linearizability > sequential consistency > causal consistency > FIFO consistency`

[各种一致性的区别](https://iswade.github.io/translate/strong_consistency_models/)

那么区块链提供什么类型的一致性？

### 为什么这个问题重要

一、弄清楚一致性类型的重要性。

1. 最初是因为我想用一些检测工具来验证设计是否有问题。分布式数据库的通常使用jepsen，但是据说这个工具只能检测线性一致性，如果不搞明白区块链提供的到底是什么类型的一致性，就没法选择相应的验证工具。
2. 区块链对外的一致性类型对用户使用方式有很大的影响。这个有并发编程经验的应该都知道，C/C++没有一个明确的内存一致性模型，导致并发编程非常多坑，go，java就好很多。虽然还没有完全弄清楚，但是从目前了解的情况看，区块链的一致性在块和交易层面是不一样的。不同类型的链也不一样。这个会给用户使用带来很大的困扰。如果我们能提供一些额外的一致性保证的功能，易用性方面会有很大的提升。比如我们说了很长时间的批量交易。批量交易其实就提供了类似编程语言中的临界区的概念。可以沿着这个方向继续研究下去。

 二、分析一个系统的一致性类型很困难。

1. 首先这个定义就不是很清楚。我看一般的文章都会说一致性是从用户角度来说的。所以像同步这种系统内部的行为，应该是不算破坏一致性的。或者说理论上来说，对于多副本分布式系统的读操作，其实假定用户会读至少一半副本的。这块儿区块链其实也可以做一些改进，比如call的时候，节点可以告诉用户，你读的并不是最新的状态，甚至可以告诉你哪个节点有最新状态。或者简单一点有一个中心服务（区块链浏览器？）可以帮你汇总最新状态。
2. 区块链其实是有多种参与方的。用户是其中一个，共识节点也是一个，PoX的链还有矿工。从不同参与方的角度看，可能也会不一样。我觉得块级别的一致性可以认为是从共识节点或者矿工角度看的；交易级别的一致性是从用户角度看的。

### 结论及思考

我看了<Why You Can’t Beat Blockchains:Consistency and High Availability in Distributed Systems>，感觉这篇论文已经解答了这个问题。 新版本改名[《Monotonic Prefix Consistency in Distributed Systems》](https://arxiv.org/pdf/1710.09209v2.pdf)，[Talk](https://project.inria.fr/epfl-Inria/files/2017/02/JadHamza-talk.pdf)

之前其实已经有一些讨论replicated data structure的一致性问题的论文。参考文献给出的结论是其一致性不可能强于某种因果一致性的变种。

这篇论文更侧重区块链(看题目就知道了)，里面提出了一种新的一致性叫MPC(Monotonic Prefix Consistency)，看字面其实就大概知道什么意思了。

论文的结论：

1. our contribution is to prove that, for a notion of behavior where updates are anonymous and their times and places of origin do not matter (as is the case in large-scale open implementations such as blockchains),  nothing stronger than MPC can be implemented in a distributed setting.
2. MPC和因果一致性是无法比较强弱的（双方都有对方不接受的事件序列），所以这个结论跟之前的结果是不冲突的。
3. 如果使用论文中trace的定义，MPC是强于因果一致性，而且跟线性一致性没有可观察的区别。这个条件的意思大概就是，client不要在意自己发送交易或者请求的顺序，而是以查询到的上链的顺序为准。

自己的一些思考：

1. 为什么replicated data structure一致性这么弱，是因为众所周知的CAP定义，replicated data structure选择了高可靠性，一致性自然要弱一些。
2. 区块链跟一般意义上的replicated data structure还不太一样，他们会有各种各样的同步策略。但是区块链是通过atomic broadcast来同步(写操作)，其一致性在整个大类里面是最强的。
3. 为什么分布式数据库能达到线性一致性？因为分布式数据库的读写操作都是由主节点排序的，而区块链的写操作是无序的，并且读操作跟写操作是完全分离的。从CAP的角度来说就是分布式数据库舍弃了部分可用性。分布式数据库主节点不可用的时候，整个系统是不可用的。但是区块链在切换出块节点的过程中是一直保持可用性的。这主要是靠节点间会相互转发交易，当然这也就造成结论的第三点中的情况，上链的顺序跟用户最初发出交易的顺序就不一致了。





