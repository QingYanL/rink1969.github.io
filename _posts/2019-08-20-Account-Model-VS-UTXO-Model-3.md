---
title:  理论篇
published: true
---

> 天之道，损有余而补不足。 --《道德经》

### 为什么要写这一系列文章

 其实关于这个主题，网上已经有很多相关文章了，随便一搜都一大把。这些文章有的是描述两种模式各自的优缺点，有的是通过打比方，让人更容易理解两种模式是什么样的。
但是这些文章都只有How，没有Why。

没有文章能说明白两种模式的本质是什么？他们的关键区别是什么？

我做研究有个习惯，喜欢把问题映射到PLT（编程语言理论）。然后看PLT中是否有相关的结论，然后再映射回来，从而得到一些有意义的结论。

我觉得UTXO/Cell Model对应于一种叫[Arrow](https://wiki.haskell.org/Arrow)的计算模型；Account Model对应于一种可能更为大家熟悉的计算模型——[Monad](https://wiki.haskell.org/Monad)。

### 本文的内容

Arrow比Monad更加General。因此，UTXO/Cell Model也应该比Account Model更加General。

我们会详细描述这些概念之间是如何对应的。

通过更详细的比较，我们还能找出两者更多的差异。

比如哪些是UTXO/Cell Model可以表达，而Account Model表达不了的。

本文会假设读者对Haskell以及Arrow和Monad有一定了解。

如果不了解，可以跳过相关的讨论，直接看结论。

### 形式化

我们先对原生token进行一个形式化的描述。

用户标识沿用eth的名词，用addr来表示，其类型为 `Address` 。

对用户身份校验，即验签，表示为一个函数`checkSig : [Signature] → Bool`

这里之所以是 `Signature` 的数组，是为了能表达多签等场景。

eth的账本类型为 `Ledger = [(Address, u256)]`

tx本身是数据，包含from，to，amount。类型为 `Data Tx = Tx Address Address u256`

但是经过vm解释之后就变成了 `Ledger -> Ledger` 的函数。

即 `interpreter :: Tx -> Ledger -> Ledger`

eth的世界状态是一个 `State Monad`。类型为 `WorldState = State Ledger ()`。

当然也可以理解为eth是采用了类似OOP的思路，将账户类比为一个对象，既有成员变量，又有成员方法。

其实这两种描述方法是一样的，State Monad就是为了能在Haskell这样的纯函数式编程语言里面描述状态。

本文之所以采用Haskell的方式，是为了后续更好的与bitcoin进行对比。

在eth上执行交易的过程可以用如下函数描述：

```haskell
eth_apply :: Signature -> Tx -> WorldState -> WorldState
eth_apply sig tx ws =
  if checkSig sig
    execState (modify $ interpreter tx) ws
```

bitcoin的账本类型为 `Ledger = [UTXO]`

交易由inputs和outputs组成，类型为 `Tx [UTXO] [UTXO]`。

在bitcoin上执行交易的过程可以用如下函数描述：

```haskell
bitcoin_apply :: [Signature] -> Tx -> Ledger -> Ledger
bitcoin_apply sigs tx utxo_set =
  if checksig sigs
    utxo_set - (inputs tx) + (outputs tx)
```

### 一般化

eth的一般化就是提供了smart contract的能力。

继续沿用Account Model，采用类似分形的方式，将世界状态分成两个层次。

世界状态由多个Account组成一棵Merkel Tree。

Account里面有code和保存contract状态的Storage。

Storage由合约内的多个变量组成一棵Merkel Tree。

每个合约都有自己的业务类型，不再只是 Ledger 了。

`ContractType = [StorageVar]`

但是合约状态类型类似一个小的世界状态，还是一个State Monad。

`ContractState = State ContractType ()`

每个合约里的操作类型都是 `func : ContractType -> ContractType`。

UTXO Model的一般化就是ckb提出的Cell Model，将balance一般化为data，并且加上Type。

CellSet变为由各种不同CellType的Cell组成一个松散的集合。

交易类型变为 Tx Input Output。其中Input和Output由各种不同CellType随意组合，且两者无需保持一致。

Account Model显然对应Monad，具体来说就是State Monad。

而UTXO一般化之后的Cell Model，对应的是一种叫Arrow的计算模型，Arrow是Function的一般化。

Function有输入（参数）类型和输出（返回值）类型。Arrow的两个类型参数，第一个表示输入类型，第二个表述输出类型。

### 差异

通过一些讨论Arrow和Monad区别的文章，我们可以反推Cell Model和Account Model的差异。

---

首先是对输入类型的限制上的差别。

我们可以看到Monad对与输入类型没什么限制。对应的，eth就Tx的类型并没有限制，只要evm可以将其解释成`func : ContractType -> ContractType`就可以了。

而Arrow是需要明确输入类型的。对应的，ckb的tx的inputs类型是确定的。

从这方面来说，Monad/Account Model会更灵活。

这种灵活对于eth这样链上计算的运行方式来说还可以，vm会在执行的时候对输入进行检查。

对于ckb这样链下计算链上验证的运行方式就不行了，因为计算结果中并不包含input的类型，根本无法对用户输入进行限制。

---

接下来是输出类型。

我们可以看到Monad并没有针对一个交易的显式的输出。

或者说像eth里，一个交易的输出就是整个Storage。这个交易具体改了Storage里的哪些东西是被封装起来的。

而Cell Model针对每个交易是有具体的输出的。

在跨链操作中，Cell Model可以方便的针对一个交易的输出给出证明，将其转移到其他链上。而Account Model就要困难的多。

---

对于计算方式来说，Monad整体是一种Fold的方式，类似归纳的方式将嵌套的结构合并到一层。每一个操作都是curry化的，局部的。

而Arrow则不一定需要curry化，可以同时操作多种输入/输出类型。

具体的例子就是之前提到的多对多的转账。eth只能一个账户一个账户的处理，而bitcoin可以直接完成多人到多人的转账。

---

组合性方面，Arrow的组合性更好。因为Arrow是类似Function的，所以可以像Function一样去组合。只要上一个函数的输出类型跟下一个函数的输入类型是匹配的就可以组合，类似于链式函数调用。

而Monad之前说过他是一种嵌套的层级结构，它的组合也是嵌套的。

以eth为例，合约之间的组合只能是跨合约调用的方式，会有很多限制。比如对用户身份的鉴权操作只能在最外面做一次，意味者两个合约必须使用同一个身份标识。同时内层合约执行出错的时候，无法提前返回，还必须返回外层合约。外层合约会拿到内层合约的返回值或者错误信息，处理起来比较麻烦。

---

依赖方面，Monad的依赖是被封装起来的，或者说像在eth里面，一个交易的依赖是整个Storage。而Arrow是明确了依赖哪些输入的。所以交易并行执行方面，UTXO/Cell Model会更有优势。

Monad依赖之前操作的结果，比如说在eth里，后一个交易依赖的Storage是前一个交易修改之后的Storage。整个依赖关系是动态，前一个交易改了合约里面的一个flag，就会导致后一个交易的行为发生变化。

而Arrow的依赖关系是静态的，当然也可以依赖之前的交易，但是并不用等到之前的交易执行。所以bitcoin和ckb可以实现[HTLC](https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts)这类操作，在一个交易没上链之前就开始构造其后续的交易。