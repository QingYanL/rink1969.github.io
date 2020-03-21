---
title: Linear Logic 与 FRP
published: true
---

### 交互与UI

在CKB的[Dapp开发框架](https://github.com/rink1969/ckb-generator)中，现阶段只有非常简单的命令式交互，也没有涉及到多方交互。

而实际上一个Dapp除了跟区块链交互，还会涉及跟其他参与方交互的流程。Cell Model这方面更甚，比如多签。

一个Dapp的流程也有可能是动态的，根据链上状态的不同而不同。

出于用户友好的考虑，最好是有一个UI来做信息的展示和用户交互。

### Linear Logic Programming

最近看到一个[视频](https://www.youtube.com/watch?v=bFeJZRdhKcI)，讲利用linear logic programming进行复杂规则游戏的原型验证。

整个游戏世界由一堆初始配置和一些演化规则组成。其中一些规则是关于用户交互调度，随机等的处理。

游戏的进行就是在玩家的动作下，世界状态从初始配置开始，遵循演化规则，一步一步变化。

规则可以使用linear logic来描述，因为游戏世界也跟现实世界很类似，死掉的角色不会再出现，物品用掉之后就没有了。

---

交互方面，最简单的做法是把当前所有可能的选项列出来，让用户选择并输入对应的序号。

这样的交互太简单了，处理不了更复杂的交互，比如需要用户输入多个值。

另外因为是多玩家，并且分角色或者分队伍，很多选项其实是无效的。

更好的做法是像回合制游戏一样，做一个回合的调度。这样在自己的回合里，用户可以做很多复杂的交互，比如输入。知道当前是哪个角色的回合，以及在整个流程里面的进度，就可以给出比较少的更有针对性的选项。

---

系统中通常会有一些重要的不变量。基本的比如，每个玩家只有一次条生命；每个玩家都会有自己的回合等。

规则设计的时候，维护系统的这些不变量是很重要的。

如果初始配置是符合这些不变量，且每一条规则对世界状态进行变化的时候也保持这些不变量。那么在整个游戏过程中，这些不变量将会一致成立。

使用linear logic来描述规则对于维持不变量非常有帮助。

### 对于框架的意义

处理多个参与方的交互，我们可以借鉴上面的一些做法。

linear logic用于Dapp开发，目前不是很紧急。

因为区块链本身就是一个linear logic的系统（参见[上一篇文章](https://rink1969.github.io/Linear-Logic-Everywhere)）。

Dapp通常也都以链上数据为准，客户端如果跟链上状态没有同步，只要保证后续操作报错，不要有误操作即可。

当然使用linear logic之后，就可以在开发阶段发现一些破坏不变量的行为，这对提高Dapp质量是非常有帮助的。

### FRP

Functional reactive programming是最近几年非常火的一项技术，多被用在交互式和UI领域。

有一篇[文章](https://techsingular.net/2016/01/13/functional-ui-programming/)，我觉得总结的不错。

一提到UI编程，肯定都会想到MVC架构。很多人觉得FRP是代替MVC的，但是其实在FRP里面还是可以看到MVC的影子。FRP可以认为是一种多层Model之间同步的一种技术。也可以认为是嵌套的MVC，这里V并不是直接渲染出可以显示的内容（bitmap），而是将上一层的Model渲染为下一层Model。

Cell Model相比Account Model，在MVC的M上面会更复杂。

Account Model可以认为合约里表示状态的数据结构是核心的M，一般客户端不需要再单独维护自己的M。

但是Cell Model下，链上没有一个显式的全局的M，这项工作必须放到客户端。

在之前的[文章](https://rink1969.github.io/Account-Model-VS-UTXO-Model-3)中，也提到类似的情况，当时是类比为Arrow。

其实Arrow跟FRP有很密切的关系。在FRP中经常会出现Signal的概念，`SF a b = Signal a -> Signal b`，这里的SF类型就是一个Arrow，而不是Monad。

而且我们可以从FRP框架中借鉴一些做法。用户看到的是更上层的Model，开发的也只是上层Model之间的转换函数。我们可以把Cell相关的概念隐藏起来，严格限制对Cell的操作，这样可以让Dapp更加的安全和可靠。

