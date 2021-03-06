---
layout: post
title:  "Specifying Systems 第五章笔记"
tags: tech
---

# 5.1 内存的接口

本节通过一个规范表示接口，不涉及实现，也即不涉及判断真假的具体公式。

首先选择抽象的层级，第二章把“发送”操作用三个变量（`val`、`rdy`、`ack`）进行描述，并把`val`和`rdy`同时更改表示为一个原子操作。本章把“发送”操作用一个变量进行描述：`val`。

接口试图规定“发送”操作是某一类*动作*，这需要描述怎样的*动作*是符合本接口的，类似于在“函数是一等公民”的编程语言中描述函数。但 TLA+ 只能把`CONSTANT`、`VARIABLE`定义的变量作为模块的参数，而不能把*动作*定义的公式作为模块的参数。作为替代，TLA+ 中可以定义运算符以达到这个目的。把*动作*涉及的 N 个变量列举出来，定义一个 N 元运算符
```
CONSTANTS  operator_as_action(_, _, _, _)
```
即可。

运算符的返回值不限定类型，而*动作*是一个公式，返回值是布尔型，所以还需要一个表示类型的假设
```
ASSUME \A p1, p2, p3, p4 : 
        operator_as_action(p1, p2, p3, p4) \in BOOLEAN
```

在学习了需要的语法知识后，我们可以定义内存的规范了。
```
-------------------------- MODULE MemoryInterface ---------------------------
VARIABLE   memInt                 \* represent memory
CONSTANTS  Send(_, _, _, _),
           Reply(_, _, _, _),
           InitMemInt,            \* set of possible initial values of memInt
           Proc,                  \* set of processor identiers
           Adr,                   \* set of memory addresses
           Val                    \* set of memory values

ASSUME \A p, d, miOld, miNew : 
        /\ Send(p,d,miOld,miNew)  \in BOOLEAN
        /\ Reply(p,d,miOld,miNew) \in BOOLEAN  

-----------------------------------------------------------------------------
MReq == [op : {"Rd"}, adr: Adr] 
          \union [op : {"Wr"}, adr: Adr, val : Val]          \* available req

NoVal == CHOOSE v : v \notin Val                       \* return value for Rd
=============================================================================
```
其中`MReq`使用记录描述了两种内存请求，记录的`:`限定了域的类型。`NoVal`通过`CHOOSE`关键字在不引入新的模块参数的前提下描述了空返回值。`CHOOSE`关键字更详细的特性将在第六章介绍。

通过`EXTEND`这个接口，就可以利用本接口定义的相关运算符和公式了。

# 5.2 函数

TLA+ 的函数是映射关系的表达，类似于数组的下标映射到值，或者*记录*的域映射到值。显然*记录*就是一种函数。与*记录*不同，函数的定义域不必是预先列举的字符串（域），函数的值也可以是与自变量相关的表达式。

函数通过方括号`f[x]`表示自变量为`x`时函数的值，所以`record.field`等于`record["field"]`。函数的定义域通过`DOMAIN function`表示。定义域是`S`、值域是`T`的函数族通过`[S -> T]`表示。定义域是`S`、值为`expression`（包含`x`的表达式）的函数通过`[x \in S |-> expression]`表示。`@`符号依然表示函数的旧值。

可以在现有的函数上构造新的函数，使其保持部分旧自变量取值，这类似修改数组中单个元素。`[f EXCEPT ![c] = e]`即`[x \in DOMAIN f |-> IF x = c THEN e ELSE f[x]]`。TLA+ 同样支持`[f EXCEPT ![c1] = e1, ![c2] = e2, ...]`。

类似于多维数组，TLA+ 同样支持`f[a][b]`等语法。

在大多数情况下，多维数组不如用元组的函数表达方便，TLA+ 可以将`f[<<element1, element2, ...>>]`缩写为`f[element1, element2, ...]`，并由`[element1 \in set1, element2 \in set2, ... |-> expression]`定义函数。

# 5.3 线性内存

本节的规范通过`ctl`描述处理器状态从而控制状态转移，通过`buf`表示处理器与内存交互的缓冲区。 

```
------------------ MODULE InternalMemory ---------------------
EXTENDS MemoryInterface
VARIABLES mem, ctl, buf
--------------------------------------------------------------
IInit == /\ mem \in [Adr->Val]
         /\ ctl = [p \in Proc |-> "rdy"] 
         /\ buf = [p \in Proc |-> NoVal] 
         /\ memInt \in InitMemInt

TypeInvariant == 
  /\ mem \in [Adr->Val]
  /\ ctl \in [Proc -> {"rdy", "busy","done"}] 
  /\ buf \in [Proc -> MReq \cup Val \cup {NoVal}]

Req(p) == /\ ctl[p] = "rdy" 
          /\ \E req \in MReq :
                /\ Send(p, req, memInt, memInt') 
                /\ buf' = [buf EXCEPT ![p] = req]
                /\ ctl' = [ctl EXCEPT ![p] = "busy"]
          /\ UNCHANGED mem 

Do(p) == 
  /\ ctl[p] = "busy" 
  /\ mem' = IF buf[p].op = "Wr"
              THEN [mem EXCEPT ![buf[p].adr] = buf[p].val] 
              ELSE mem 
  /\ buf' = [buf EXCEPT ![p] = IF buf[p].op = "Wr"
                                  THEN NoVal
                                  ELSE mem[buf[p].adr]]
  /\ ctl' = [ctl EXCEPT ![p] = "done"] 
  /\ UNCHANGED memInt 

Rsp(p) == /\ ctl[p] = "done"
          /\ Reply(p, buf[p], memInt, memInt')
          /\ ctl' = [ctl EXCEPT ![p]= "rdy"]
          /\ UNCHANGED <<mem, buf>> 

INext == \E p \in Proc: Req(p) \/ Do(p) \/ Rsp(p) 

ISpec == IInit  /\  [][INext]_<<memInt, mem, ctl, buf>>
--------------------------------------------------------------
THEOREM ISpec => []TypeInvariant
==============================================================
```
注意上述规范中`Send`和`Reply`依然没有定义，不能被运行。

# 5.4 元组即函数

n 元组实际上是定义域为 1 到 n 的函数。有限长序列是元组，因此它也是函数。元组的笛卡尔积运算符是`\X`。

# 5.5 递归函数定义

递归函数的定义有单独的语法，以阶乘为例：
```
fact[n \in Nat] == IF n = 0 THEN 1 ELSE n * fact[n - 1]
```
具体含义将在第六章讨论。

# 5.6 直写式缓存

直写式缓存是一个足够复杂的组件，可以以此体会 TLA+ 精确描述系统的能力。为了在 5.3 节基础上增加直写式缓存的特性，我们为表示处理器状态的`ctl`变量新增了一个状态标识。

本节的规范使用了一个逻辑表达的小技巧，`(x /= y) /\ (x /= z)`可以写作`x \notin {y, z}`。缓存定义为从处理器和地址到值的映射，即
```
cache \in [Proc -> [Adr -> Val \cup {NoVal}]]
```
下述公式表达了缓存的一致性
```
Coherence == \A p, q \in Proc, a \in Adr : 
                (NoVal \notin {cache[p][a], cache[q][a]})
                      => (cache[p][a] = cache[q][a])
```
模块通过`EXTEND`或者实例化导入了一些*动作*，要注意因为本模块相对于原模块有额外的状态变量时，导入的*动作*还要保证这些额外状态变量不变，即
```
action == action_from_import /\ UNCHANGED <<variable_declared_here>>
```

`LET ... IN ...`语法将`LET`从句中定义的标识符的作用域限制在`IN`从句中。

可以使用一个中间变量表示某个“最终状态”，例如消息队列全部应用后的状态，这个技巧在表达一些复杂逻辑时能够简化规范。本节的直写式缓存通过一个异步队列将各处理器的缓冲区同步到内存中，为了保证缓存一致性，某些处理器的请求需访问尚未同步到内存的、仍处于异步队列中的值，此时用一个中间变量表示“内存的最终状态”就能简化这一逻辑。详见原书。

# 5.7 不变量

目前为止我们遇到了两种不变量：*类型不变量*和表示系统属性的不变量`Coherence`，这两者具有本质上的区别。对于系统的*类型不变量*，如果一个状态（与初始状态的可达无关）满足*类型不变量*，则符合*步*的状态转移也一定满足*类型不变量*。但`Coherence`不满足这种特性。所幸在正确的规范中，这些即将不满足`Coherence`的状态是系统的初始状态不可达的。

# 5.8 证明正确实现了接口

规范 A 也可以蕴含另一个（接口）规范 B，这常常发生在“is a”关系中，例如直写式缓存是一种内存。这是通过 A 的初始化公式蕴含 B 的初始化公式，以及 A 的不变量和*动作*蕴含了 B 的*动作*或保持状态不变来证明。更细节地说，规范 A 中通过变量映射实例化了规范 B，就需要证明在这种变量映射下， A 的规范能蕴含规范 B。