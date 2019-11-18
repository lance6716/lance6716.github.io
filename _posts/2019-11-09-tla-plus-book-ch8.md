---
layout: post
title:  "Specifying Systems 第八章笔记"
tags: tech
---

# 8.1 时序逻辑

本章开始，我们将会判断系统在整个时间跨度上的命题。之前的规范涉及了简单的时序逻辑，本章将沿用那些符号并进行扩展。

在此引入一些术语简化表达：状态公式表示只涉及一个状态的公式，*动作*表示只涉及一次状态转移的公式。

系统的行为定义为无限个状态通过状态转移形成的序列，时序逻辑公式`F`为系统的行为`sigma`判定真假值，这记为`sigma |= F`。从定义有`|=`对`/\`、`~`、`\E`、`\A`有分配律。接下来我们定义如何判断`|=`的真假值。

- 对于状态公式，公式在行为的第一个状态的真假定义了`|=`的真假

- 对于状态公式`F`，`F`在行为的每个状态都为真，`sigma |= []F`才为真

- 对于*动作*`N`和状态公式`v`，行为的每一次状态转移都满足`[N]_v`，`sigma |= [][N]_v`才为真

上述三种公式在前文中都已经出现，接下来我们对他们的含义进行扩展。

首先是将含义扩展到*动作*（但按照时序逻辑公式的定义，*动作*并不是时序逻辑公式）。对于*动作*`A`，`sigma |= A`表示系统的第一个状态转移满足*动作*`A`，`sigma |= []A`表示系统的每个状态转移满足*动作*`A`。依照这种方式，我们也可以将含义扩展到时序逻辑公式。

接下来是对`[]`运算符的规范定义，将系统状态转移序列的任意一个状态视作“第一个状态”，也即截断前几个状态，如果任意截断后状态转移序列都满足时序逻辑公式`F`，则记为系统的行为满足`[]F`。因此`[]`运算符实际上表达了**从某状态以后始终满足**的含义。

在 2.2 节中，为了描述现实中可能发生的情况，我们允许系统在相邻状态中所有变量都不改变，而这会使得有些时序逻辑公式为假。为了解决这个问题，TLA 仅允许表达“允许所有变量都不改变”的时序逻辑公式。通常而言，状态公式、`[][N]_v`以及它们通过`[]`运算符和布尔运算符组合而成的公式都是 TLA 允许表达的。

接下来引入五种记号简化表达：

- `<>F`表示`~[]~F`，即`F`不总是假的，即`<>`表达了**从当前状态以后至少一次满足**的含义。

- `F ~> G`表示`[](F => <>G)`，即`F`为真之后，`G`会在某时为真。

- `<><<A>>_v`表示`~[][~A]_v`，即不是每一*步*都是`(~A) \/ (v' = v)`*步*，即某一步会是`A /\ (v' /= v)`*步*。我们也可以视作`<<A>>_v == A /\ (v' /= v)`（但这不是时序逻辑公式，因为它只描述一次状态转移），并结合`<>`的含义记忆。

- `[]<>F`即在任意时刻以后，`F`会为真。这也意味着`F`无穷次为真。

- `<>[]F`表示存在某个时刻，`F`在那以后一直为真。

`[]`和`<>`的优先级比布尔运算符高，`~>`的优先级比合取、析取低。

# 8.2 时序重言式

时序逻辑中的重言式更加复杂，有的重言式可以通过含义直接判断，有的则需要借助一些转化的技巧。

*对偶*重言式是指，将时序逻辑公式的`[]`与`<>`、`/\`与`\/`相互替换，并反转蕴含的方向，就能得到一组*对偶*重言式。

需要注意`<<A>>_v`需要与`<>`组合才是时序逻辑公式，有些重言式会使得`<<A>>_v`变成独立的谓词，例如
```
[]<>(<<A>>_v \/ <<B>>_v) <=> ([]<><<A>>_v) \/ ([]<><<B>>_v)
```
其中`(<<A>>_v \/ <<B>>_v)`没有与`<>`组合，需要转化为`<<A \/ B>>_v`才是合法的公式。

# 8.3 时序逻辑证明法则

时序逻辑的证明法则可以沿用命题逻辑的证明法则，除此之外

- 时序逻辑有自己的证明法则：

  - 泛化法则：对于时序逻辑命题`F`，如果所有的行为都有`F`，那么所有的行为都有`[]F`。这可以由 8.1 节对`[]`的定义和行为通过截断关系相互转化得到

  - 蕴含的泛化法则：对于时序逻辑命题`F`、`G`，如果所有的行为都有`F => G`，那么所有的行为都有`[]F => []G`

- 命题逻辑的法则对应一个重言式，而时序逻辑不是。这是因为时序逻辑是对于系统的行为，也即无限长的状态序列进行描述，重言式在语义上是一个命题逻辑公式而不能描述行为的关系
 
  - 例如泛化法则并不是说`F => []F`是重言式。我们可以令`F`为状态公式，`sigma`的第一个状态使`F`为真，后续状态使`F`为假，容易验证`sigma |= (F => []F)`并不为真。

# 8.4 弱公平性

系统有安全性和活跃性两种特性，分别表示“某些特性总是满足”以及“某些特性最终会满足”。本书的第一部分说明 TLA 如何表达安全性，而在了解了`[]`和`<>`运算符之后，我们就可以表达活跃性了。对于第二章的时钟，一个永远运行的时钟可以表达为`[]<><<HCnxt>>_hr`。

但是在有些情况下，活跃性还需要前提条件。第三章的异步接口
```
Rcv ==  /\ chan.rdy /= chan.ack
        /\ chan' = [chan EXCEPT !.ack = 1 - @]
```
为了表示发送的值总会被接受，前提是存在发送的值，也即`chan.rdy /= chan.ack`。TLA 通过`ENABLED`关键字描述某个*动作*的前提。因此异步接口发送值总会被接受可以描述为`[](ENABLED <<Rcv>>_chan => <><<Rcv>>_chan)`或`[](ENABLED Rcv => <><<Rcv>>_chan)`或`ENABLED Rcv ~> <><<Rcv>>_chan`。

弱公平性`WF_v(A)`定义为`[]([]ENABLED <<A>>_v => <><<A>>_v)`，其含义是如果*动作*`<<A>>_v`在任意时刻 t 后都能进入永远满足前提的状态，那么`<<A>>_v`在 t 后必将发生。弱公平性与以下两种定义等价：

- `[]<>(~ENABLED <<A>>_v) \/ []<><<A>>_v`，即`<<A>>_v`在任意时刻之后都有不满足前提条件的时候，或者在任意时刻后`<<A>>_v`总会发生

- `<>[](ENABLED <<A>>_v) => []<><<A>>_v`，即`<<A>>_v`在某时刻后会永远满足前提，则在任意时刻后`<<A>>_v`总会发生

定义了弱公平性后，我们发现“发送的值总会被接受”`[](ENABLED <<Rcv>>_chan => <><<Rcv>>_chan)`与弱公平性`[]([]ENABLED <<Rcv>>_chan => <><<Rcv>>_chan)`是不同的。如果我们能证明在异步接口的规范下两者恒等，就能用弱公平性简化表达。

这里由异步接口的规范可得：如果`<<Rcv>>_chan`的前提满足，即发送了一个值后，或者发生`<<Rcv>>_chan`从而接受这个值；或者这个值永不被接受，也就是`[](ENABLED <<Rcv>>_chan)`。这可以推出两者恒等。即下面是一个重言式，表示满足前件后“发送的值总会被接受”这种叙述与弱公平性等价
```
[](E => []E \/ <>A) => ([](E => <>A) <=> []([]E => <>A))
```
公式中`E`表示*动作*`A`的前提，`A`表示弱公平性的*动作*。

# 8.5 内存的规范

#### 8.5.1 对活跃性的需求

我们对第五章的内存模型表达“发出的请求最终会得到回复”。其中`MemoryInterface`模块只给出了`Reply`操作符接口的定义，没有具体公式的实现，因此`ENABLED`关键字不能判断何时满足前提，这需要我们增加一个假设
```
ASSUME \A p, r, miOld : \E miNew : Reply(p, r, miOld, miNew)
```

请求的发送到回复需要经历两个逻辑上连续的*动作*，因此活跃性可以表达为
```
Liveness == \A p \in Proc : WF_vars(Do(p)) /\ WF_vars(Rsp(p))
```
其中`vars`表示系统的所有变量。

#### 8.5.2 另一种表达

上面的公式会让我们思考，是否`WF_v(A) /\ WF_v(B)`与`WF_v(A \/ B)`是等价的。一般而言它们不等价，但在内存的例子中，由于`Do(p)`和`Rsp(p)`的一者满足前提时，另一者前提不会被满足，直到前者被*运行*。这是一条通用的定理，具体证明见原书。

#### 8.5.3 泛化

上一小节的定理引出了弱公平性合取法则：对于*动作*`A1`，……，`An`，如果所有的`Ai`，`Aj`都满足一者满足前提时，另一者前提不会被满足，直到前者被*运行*。那么`WF_v(A1) /\ ... /\ WF_v(An)`等价于`WF_v(A1 \/ ... \/ An)`。

考虑到合取与析取与全称量词、存在量词对应，上面的弱公平性合取法则也可以表示为量词形式。

# 8.6 强公平性

强公平性`SF_v(A)`有以下两种等价定义：

- `<>[](~ENABLED <<A>>_v) \/ []<><<A>>_v`，即`<<A>>_v`在某时刻后保持不能满足前提，或者`<<A>>_v`发生无数次

- `[]<>ENABLED <<A>>_v => []<><<A>>_v`，即如果`<<A>>_v`无数次满足前提，那么`<<A>>_v`发生无数次

其中第二种定义与弱公平性`<>[](ENABLED <<A>>_v) => []<><<A>>_v`类似。由于`<>[](ENABLED <<A>>_v)`蕴含`<>[](~ENABLED <<A>>_v)`，满足强公平性的系统一定满足弱公平性。在下面的条件下，强公平性与弱公平性等价：
```
[]<>(~ENABLED <<A>>_v) => <>[](~ENABLED<<A>>_v) \/ []<><<A>>_v
```
一种常见的满足上式的条件是：*动作*的不满足前提**仅**因为*动作*的*运行*会取消自身的前提，例如第三章异步接口的`Rcv`后（如果没有其他动作）就不满足下次`Rcv`的前提。另一种满足上式的表述是：*动作*一旦满足前提，在它*运行*前会保持满足前提的状态，*运行*后则不满足前提。

强公平性的合取法则、量词形式的合取法则与弱公平性一致。

强公平性更难实现，也在规范中更少出现。在强、弱公平性等价的时候，最好表述弱公平性。此外应尽量使用强、弱公平性而不是原始的时序逻辑公式。

# 8.7 直写式缓存

直写式缓存与 8.4 节的异步接口类似，想描述“每个请求都将收到响应”，首先要尝试按照 8.4 节的等价关系转化为弱公平性。假设能满足转化，这意味着`Next`动作中与“请求-响应”相关的析取子式具有一定的公平性，而这些*动作*子式究竟应该使用弱公平性还是强公平性又需要我们仔细讨论。因此我们现在有两个问题：

1. “每个请求都将收到响应”能否转化成弱公平性

2. 这种公平性是否等价地加强为强公平性

首先考虑第一个问题。直写式缓存有读、写两种请求，这两种请求按照多种*动作*序列描述的状态转移完成“请求-响应”流程，例如`Req(p)`、`DoRd(p)`、`Rsp(p)`。这些*动作*可以通过验证 8.4 节的等价关系转化为弱公平性。但是有的*动作*并不是直接符合 8.4 节等价关系的形式。例如`DoRd(p)`表示读请求在缓存命中时的响应，它的满足前提可能会被`Evict(p, a)`清理缓存破坏，看上去不满足“动作的前提满足后，要么在未来*运行*该动作，要么该前提始终保持满足”的定义。

我们进一步考虑`Evict(p, a)`之后发生了什么。在`Evict(p, a)`破坏后，`RdMiss(p)`的前提被满足，如果`RdMiss(p)`以及后续*动作*符合 8.4 节的等价关系以及弱一致性，并在一连串*动作*后使得`DoRd(p)`的前提重新被满足，且不会被其他*动作*破坏，那么从更大的时间跨度上可以得到：`DoRd(p)`的前提满足后，要么该前提一直满足，要么`DoRd(p)`在未来会*运行*。从而能转换成弱公平性。

而`RdMiss(p)`与`DoWr(p)`的前提需要消息队列不满，因此它们的前提满足时，可能由于另一个处理器`q`的这两种*动作*使前提不满足。也就是说，它们的前提满足不能保持始终为真。为了验证“每个请求都将收到响应”的特性，这些*动作*需要强公平性。这也就是问题二。

因此直写式缓存的活跃性表示为
```
/\ \A p \in Proc: /\ WF_vars(Rsp(p))) /\ WF_vars(DoRd(p))
                  /\ SF_vars(RdMiss(p)) /\ SF_vars(DoWr(p))
/\ WF_vars(MemQWr) /\ WF_vars(MemQRd)
```
考虑到`MemQWr`在全是写请求以及队列未满的时候无需*运行*，我们也可以通过合取这个限制条件进一步弱化活跃性。

# 8.8 量词

在时序逻辑下，命题逻辑的量词表示所限定的变量在所有状态下保持不变。如果想要描述随着状态改变而改变的变量，应该使用时序逻辑量词`\EE`和`\AA`。

`CHOOSE`运算符不能扩展到时序逻辑中。

# 8.9 检验时序逻辑

#### 8.9.1 回顾

时序逻辑公式既可以表达规范，也可以表达要证明的性质。一般而言，通过弱公平性和强公平性表示规范，通过基础的运算符表达要证明的性质。规范的完整形式如下：
```
Init /\ [][Next]_vars /\ Liveness
```
在表达不同抽象层级的规范时，常常使用`\E`、`\EE`隐藏一些中间变量。

#### 8.9.2 状态机闭包

描述系统的活跃性时应当尽量使用弱公平性和强公平性，而不是基本的时序逻辑公式，这是因为在构造时序逻辑公式时容易出错。这种出错常常会限制`Init`或`[][Next]_vars`中的一些状态或*动作*不能发生，因此使得规范不能描述真实的系统。在大多数情况下，公平性描述的动作应该是`Next`的子动作。一个不影响`Init`和`[][Next]_vars`的活跃性被称为状态机绑定的。

在有些复杂情况下，确实存在一些系统不能通过弱一致性和强一致性描述活跃性。这将在 11.2 节描述。

#### 8.9.3 状态机闭包与概率

可以把活跃性视作一种概率表达，例如对于一对状态机绑定的`S`和`[]<><<A>>_v`，`S`所描述的系统总有可能发生无穷多次`<<A>>_v`。在使用 TLA 表示规范时，要注意真实系统的行为符不符合规范，以及规范的公平性足够推导出由预期的系统行为引发。

#### 8.9.4 实例化映射和公平性

5.8 节介绍了规范 A 实现规范 B 时，要证明规范 A 蕴含了这种实例化的规范 B。在增加了活跃性后，实例化时的变量映射能否对弱、强公平性分配呢，也即：

由`<<vars0>>`实例化的`<<vars>>`，是否满足规范 B 的`WF_v(A)`等于规范 A 实例化的`WF_v0(A0)`呢

答案是不等于，因为其他的运算符满足分配律，但`ENABLED`关键字不满足分配律。例如实例化为两个接口变量分配了同一个实际变量，接口中的前提就会产生干扰。

因此，在规范中出现实例化映射和公平性时，需要我们还原成时序逻辑公式并对`ENABLED`运算符按如下规则手动计算：

1. 对于任意*动作*，`ENABLED(A \/ B) <=> (ENABLED A) \/ (ENABLED B)`

2. 对于单状态公式`P`和*动作*`A`，`ENABLED(P /\ A) <=> P /\ (ENABLED A)`

3. 对于*动作*`A`和`B`，如果`A`和`B`不同时更改同一个变量，`ENABLED(A /\ B) <=> (ENABLED A) /\ (ENABLED B)`

4. 对于任意变量`x`和单状态表达式`exp`，`ENABLED(x' = exp) <=> TRUE`，`ENABLED(x \in exp) <=> (exp /= {})`

#### 8.9.5 活跃性不重要之处

实际应用中，安全性比活跃性更重要。

#### 8.9.6 引起困惑的时序逻辑

时序逻辑表达方法众多，可读性差，最好按照统一的方法写时序逻辑公式。