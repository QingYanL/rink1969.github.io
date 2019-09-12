---
title: 区块链需要逻辑编程
published: true
---

### logic programming

逻辑编程有两个特点：

1. 不区分输入和输出。可以正向计算，也可以反向计算。
2. 可以处理不确定的情况。就像解代数方程一样，如果只有一个解，会输出这个解；如果有多个解，会把所有的解都找出来，当然用户也可以控制，找到任何一个解的时候就停下来。

### 对于区块链的意义

尤其是对于Cell Model来说，非常适合，参见[文章](https://github.com/rink1969/ckb-generator/blob/master/README.md)。

对于第一点，很多时候，确实是需要从输出来反向计算输入的。比如转账，我们是先确定输出里转账的金额，然后据此才能去查找live cell作为交易的输入。

对于第二点，继续上面的例子，查找live cell的时候肯定有很多种组合可以满足要求。我们可以把所有可能的结果都列出来，然后再附加其他限制；也可以简单一点，找到任意一个结果就可以了。

### linear logic programming

在之前逻辑编程的基础上增加linear logic的限制。

下面是一些学习的笔记：

---

[linear logic programming](https://www.infoq.com/presentations/linear-logic-programming/)

linear logic programming与logic programming。

logic programming 首先有一些规则，然后用户来query一个问题的答案。

关键的一点是locally stateful computation。或者叫frame property。感觉就是前面说的运动中的一帧。

purely functional 把世界状态作为参数传给函数，然后生成新的世界状态。

imperative 直接修改替换状态

linear logic programming 是一种声明式的，high level，更贴近领域语言



logic programming proof search is execution。

反复应用规则，直到找到用户想要查询的答案，同时给出proof，也就是路径。

backward chaining 从目标开始一步一步往回搜索，拆分出一些子任务，直到子任务显然成立。prolog就是这种方法。

forward chaining 从已经得到的结论开始，类似广度优先搜索，直到碰到目标。



如前所述，很多逻辑编程语言都使用backward chaining。

为什么要关注forward chaining，因为想modeling evolving system



rules和proposition是等价的



quiescence 无法再应用任何一条规则的状态

存在不确定性，比如排序，先交换哪两个数字都是可以的。

---

[Ceptre: A Language for Modeling Generative Interactive Systems](https://www.youtube.com/watch?v=bFeJZRdhKcI)

针对复杂规则的游戏进行原型验证

模拟游戏

类似mud的游戏，通过输入命令，改变隐含的世界模型



interactive worlds 由 initial configuration（玩家的状态）和一些rules for evolving configurations（buffing增加玩家的攻击力）组成。其中一些规则用来调度用户交互，随机和其他的一些处理。



logic

facts about the world are propositions。

可以写成predicates



action and change

最基本的一点是在玩家的动作下，世界的facts如何随着时间发生改变。

可以形式化为linear logic的implication

tenser conjoin two propositions

！表示可访问多次（不同的颜色表示是否可以多次访问）

A * $B -> C  等于 A * B -> C * B



linear logic Programming

Congifuration（linear context）一些属性列表

Signature（permanent rules）



multi set rewriting    forward chaining



run program  = 从init开始应用规则，形成一个trace

直到没有规则可以再应用

这里有不确定性



interactivity

一个方法是把所有可能选项列出来，让用户选择并输入序号



但是玩家是分队对抗的

而现在的选项是混在一起的



引入stage

有点像是一个回合游戏里面，维持当前轮到谁的机制

跳出到runtime进行调度的机制

交互就增加到中间调度的时候



case studies

Narrative Generation

Multi-agent interactive social simulation

RPG

桌游



Fuzzing in game

没太大意义，可以做一些ai，统计等

进行一些平衡性的调整



关于规则的设计，Invariants非常重要，规则要保证这些不变量

INV holkds for a specification iff INV holds of all initial configurations and for all rules A -> B

if INV holds of A,contexts then INV holds of B,contexts

一个好的方法是使用linear logic（有点像libra的想法）



Lessons for PL Design

Logical Basis 逻辑是一个很好的中间语言，在人的思考和机器的中间

minimalism

逐渐修正

### Ceptre的例子

[项目仓库](https://github.com/chrisamaphone/interactive-lp)，README里面有预编译binary的下载链接。

写了一个简单的两个用户转账的例子。

编辑`transfer.cep`包含如下内容：

```
cell : type.
key : type.
user : type.

locked key cell : pred.
control user key : pred.

stage transfer = {
transfer
  : locked K C * control U K * control U1 K1 -o locked K1 C * control U K * control U1 K1.
}
#interactive transfer.

alice : user.
bob : user.

alice_key : key.
bob_key : key.

c : cell.

context init =
{ control alice alice_key, control bob bob_key, locked alice_key c }


#trace _ transfer init.
```

运行结果：

```
$ ceptre transfer.cep 
Ceptre!
cell: type.
key: type.
user: type.
locked key cell: pred.
control user key: pred.
stage transfer {
forward chaining rule transfer with 5 free variables...
}
#interactive transfer.
alice: user.
bob: user.
alice_key: key.
bob_key: key.
c: cell.
context init { (control alice alice_key), (control bob bob_key), (locked alice_key c) }
#trace ...

0: (quiesce)
1: (transfer alice_key c alice bob bob_key)
?- 1

0: (quiesce)
1: (transfer bob_key c bob alice alice_key)
?- 1

0: (quiesce)
1: (transfer alice_key c alice bob bob_key)
?- 0


Final state:
{(control alice alice_key), (control bob bob_key), (locked alice_key c), (stage transfer)}

Trace: 
let [x6, x5, x4] = transfer alice_key c alice bob bob_key [x3, [x1, [x2, []]]];
let [x9, x8, x7] = transfer bob_key c bob alice alice_key [x4, [x6, [x5, []]]];
```

整个运行过程是交互式的命令行的方式。

我们可以看到需要用户操作时，系统都会列出所有当前可以进行的操作的列表。

特别需要注意的是，系统会根据规则以及相关的类型自动匹配参数。

因为在我们的初始配置里是alice拥有一个cell，所以第一次transfer，系统自动就判断出可能的操作是alice给bob转账。

而转账之后，cell的所有权转移给了bob。接下来系统的提示就变成了bob给alice转账。

这个是逻辑编程里的unify特性，再辅以linear logic对物品所有权的感知。能够帮助用户做很多推理，去除一些无意义的选项，简化用户的负担。

这个对于Cell Model在用户交互层面会非常有用。

### Cell Model下Dapp架构

目前ckb-generator没有达到预期的效果，但是其中采用的合约组件的方式证明是可行的。

只不过组件生态如果比较成熟，用户只需要简单的进行组件的组合，就不需要DSL了。

---

整个Dapp如[上篇文章](https://rink1969.github.io/Linear-Logic-FRP)所说，采用FRP，或者说嵌套MVC架构。

SDK负责屏蔽链交互相关的信息，解析出Json格式的数据。

logic porgramming这一层负责Dapp业务规则相关的推理，根据用户操作，给出结果，或者给出用户当前可以做的操作的列表。

最后是GUI，以用户友好的方式展示相关信息，以及各种复杂的用户输入。

---

Cell Model的一个特点是输入输出都是确定的。

再加上前述逻辑编程的特点。

如果事先把Dapp相关的资源单独隔离出来，保证Dapp运行期间这些资源不会被消费掉。

将这些资源作为初始配置输入逻辑编程环境，那么用户就可以任意的操作或者回退。

比如像HTLC的例子，Cell Model可以做到交易没上链之前就得到其交易Hash，并据此构造后续的交易。

只有到了不得不跟链交互的时候，经过反复确认，再把相关交易一起提交上链。

---

这样就有点侧链的感觉。

或者说像是git里拉了一个分支，用户可以在分支上任意的修改，回退再修改，最后merge回主分支。

这印证了[之前的文章](https://rink1969.github.io/Linear-Logic-Everywhere)所说，POW是解决多条子链（分支）合并，并解决其可能出现的冲突的。

