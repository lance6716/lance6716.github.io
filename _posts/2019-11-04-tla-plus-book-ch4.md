---
layout: post
title:  "Specifying Systems 第四章笔记"
tags: tech
---

本章介绍一个先进先出 FIFO 系统。

# 4.1 内部结构的规范

一个 FIFO 队列内部是一个缓冲区，对外提供单个消息的入队与出队接口。对于外部接口，TLA+ 可以复用之前的模块，这类似于面向对象编程中实例的概念。我们复用上一章的模块`Channel`
```
InChan == INSTANCE Channel WITH Data <- Message, chan <- in
```
`InChan`现在是一个`Channel`，除了`Data`和`chan`通过相应的新标识符访问，其余标识符（以及运算符）通过`InChan!symbol`访问。

缓冲区用 TLA+ 的内置数据结构`Sequences`进行表示。`Sequences`表示一个有限序列，字面量用元组表示，并有如下的与之相关的标识符：

`Seq(S)`表示`S`集合能组成的所有有限序列。

`Head(S)`、`Tail(S)`表示序列的首元素及剩余序列。

`Append(S)`、`S \o T`表示序列与元素、另一个序列连接。

`Len(S)`表示序列的长度。

了解上述语法后，可以把 FIFO 描述为三个组件：依次相连的入队接口、内部先进先出缓冲区、出队接口。FIFO 的状态转移将由相连两组件各自的状态转移以及两组件交互组成。

# 4.2 实例化的细节

与`EXTENDS`不同，实例化需要导入的所有符号（主要是`CONSTANT`、`VARIABLE`等关键词定义的标识符）都指定本模块中对应标识符。有以下几个方面需要注意。

#### 4.2.1 实例化即替换

TLA+ 的实例化是进行表达式展开和替换。为了满足语法，原模块替换完成后，所有符号在本模块中都要有定义，这可以通过`EXTENDS`引入运算符等符号以及使用`<-`为原模块未匹配的符号指定本模块的标识符、表达式等符号来满足。

#### 4.2.2 实例化的参数化

实例化时可以包含参数，即
```
InChan  == INSTANCE Channel WITH Data <- Message, chan <- in
OutChan == INSTANCE Channel WITH Data <- Message, chan <- out
```
可以使用
```
Chan(ch) == INSTANCE Channel WITH Data <- Message, chan <- ch
```
代替，`Chan(in)`在替代后即为`InChan`。

#### 4.2.3 隐式替换

本模块与导入模块的同名符号可以自动隐式替换。所有符号都隐式替换时`WITH`可以省略。

#### 4.2.4 同名实例化

```
INSTANCE Module WITH symbol_from_module <- symbol_here, ...
```
进行了一次同名实例化，这与`EXTENDS`的效果类似，可以直接访问导入符号而无需`instance!symbol`，并对指定的符号进行了替换。

# 4.3 隐藏内部队列

FIFO 的三个组件中，外部接口并不需要了解内部缓冲区的信息。时序逻辑存在量词`\EE`可以把一个符号的作用域转化为这条公式，从而减少向外部暴露的符号。例如`InnerFIFO`向外部暴露了 4 个符号
```
CONSTANT  Message
VARIABLES in, out, q
```
其中`q`是内部缓冲区的序列。我们可以新建模块`FIFO`，通过`\EE`将`q`转化为一个由 TLA+ 自动推导、无需指明的符号。但需要注意，`\EE`不能隐藏本作用域定义的变量，因此一般与实例化或者`LET`结合使用。
```
------------------------ MODULE FIFO -------------------------
CONSTANT  Message
VARIABLES in, out
Inner(q) == INSTANCE InnerFIFO 
Spec == \EE q : Inner(q)!Spec
==============================================================
```

# 4.4 有限容量 FIFO

本节描述一个内部缓冲区长度有限的 FIFO。在复用`InnerFIFO`的基础上，可以在状态转移中表达新的公式完成这一点。新的状态转移定义为
```
Next /\ (BufRev => (Len(q) < N))
```
即内部缓冲区可以接收消息意味着缓冲区未满（长度小于 N）。

为了使规范有意义，我们指定外部参数`N`是正整数。正整数的限制通过`ASSUME`关键字表达。
```
CONSTANT N
ASSUME   (N \in Nat) /\ (N > 0)
```
`ASSUME`在 TLA+ 进行推导时生效，一般应用在给`CONSTANT`增加更强的假设时。错误应用`ASSUME`会导致逻辑矛盾。

TLA+ 隐含了一些假设，例如所有的对象都是集合（TLA+ 以 ZF 集合论为基础）。如果需要表达比隐含的假设更强的描述，例如 TLA+ 某对象是有限集（而不是不限定有限或无限的集合），需要用`ASSUME`表达。

# 4.5 规范表明了什么

本章完成的 FIFO 模块规范实际上不单单包含了一个满足先进先出特性的队列，还包含了对队列的发送、接收方的描述，而这是 FIFO 的*环境*，并不是 FIFO 队列本身。包含环境的规范叫做*闭系统*，不包含的叫做*开系统*。一般而言，这两者可以相互转换，但*闭系统*描述更加简单，也更为常用。
