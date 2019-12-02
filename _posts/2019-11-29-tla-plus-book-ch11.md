---
layout: post
title:  "Specifying Systems 第十一章笔记"
tags: tech
---

# 11.1 数据结构的规范

#### 11.1.1 定义局部标识符

`LOCAL`关键词可以控制所修饰的**定义**标识符以及**实例化**`INSTANCE`不导出到其他模块中，避免与其他模块的标识符冲突。但是`LOCAL`不能修饰变量、常量的声明，以及`EXTENDS`。

在`LOCAL`的语法上，`INSTANCE`和`EXTENDS`是不等同的。但是如果模块没有变量、常量的声明或者子模块，那么该模块的`INSTANCE`和`EXTENDS`效果一致（见 4.2.4 同名实例化），因此可以用`INSTANCE`导入并用`LOCAL`修饰。

#### 11.1.2 图

我们以图为例解释如何定义一个数据结构，具体而言，定义一个没有外部声明的有向图`Graphs`模块，以方便图与图之间的运算。`Graphs`模块通过*记录*访问点集`N`、边集`E`，需要对外提供如下运算符：

- `IsDirectedGraph(G)`：当且仅当`G`的`E`是`N \X N`的子集。

- `DirectedSubgraph(G)`：返回有向图`G`的所有子图。

- `IsUndirectedGraph(G)`、`UndirectedSubgraph(G)`：将上述的运算符扩展到无向图，在`Graphs`模块中，无向边通过两条形成环的边表示。

- `Path(G)`：图的路径。

- `AreConnectedIn(m, n, G)`：两个点`m`和`n`是否是连通的。

在此省略定义的其他的运算符。`Graphs`模块如下：
```
------------------------------- MODULE Graphs ------------------------------- 
LOCAL INSTANCE Naturals
LOCAL INSTANCE Sequences

IsDirectedGraph(G) ==
   /\ G = [node |-> G.node, edge |-> G.edge]
   /\ G.edge \subseteq (G.node \X G.node)

DirectedSubgraph(G) ==    
  {H \in [node : SUBSET G.node, edge : SUBSET (G.node \X G.node)] :
     IsDirectedGraph(H) /\ H.edge \subseteq G.edge}
-----------------------------------------------------------------------------
IsUndirectedGraph(G) ==
   /\ IsDirectedGraph(G)
   /\ \A e \in G.edge : <<e[2], e[1]>> \in G.edge

UndirectedSubgraph(G) == {H \in DirectedSubgraph(G) : IsUndirectedGraph(H)}
-----------------------------------------------------------------------------
Path(G) == {p \in Seq(G.node) :
             /\ p /= << >>
             /\ \A i \in 1..(Len(p)-1) : <<p[i], p[i+1]>> \in G.edge}

AreConnectedIn(m, n, G) == 
  \E p \in Path(G) : (p[1] = m) /\ (p[Len(p)] = n)
=============================================================================
```

#### 11.1.3 解微分方程

9.5 节说明了使用`Integrate`运算符描述混合系统，为了表达`Integrate`运算符，需要在 TLA+ 中表述微分等高级的数学运算。在此仅摘录极限的 TLA+ 表达。

邻域`Nbhd(r, e) == {s \in Real: (r - e < s) /\ (s < r + e)}`。

δ-ε 极限表示法求导数`df(r)`
```
\A e \in PosReal :
  \E d \in PosReal : 
    \A s \in Nbhd(r,d) \ {r} : (f[s] - f[r]) / (s - r) \in Nbhd(df[r], e)
```

完整的微分方程表示以及`Integrate`运算符定义见原书。

#### 11.1.4 BNF 语法

TLA+ 的语法可以表示 BNF 语法，例如`L & M == {s \o t : s \in L, t \in M}`，`L | M == L \cup M`。更复杂的表达 BNF 语法的方法见原书。

# 11.2 其他内存规范

#### 11.2.1 接口

为了尽量模拟真实世界中的内存，我们首先明确一个内存应当具有什么性质，并使用接口描述。本节描述一个多处理器、异步通信、每个处理器通过多个寄存器与内存交互的接口。

#### 11.2.2 正确性条件

思考如何使用正确性描述这个接口的行为。我们规定对于单个处理器，内存的行为如同串行按照发起请求的顺序执行一样。处理器只有“读”操作会从内存返回值，我们以“读”操作为例更具体讨论如何描述内存的行为。

如果两个处理器对相同地址写入各自的值，然后进行有限或无限次的读操作，始终读到各自的值。这样的行为需要我们回应：某些优先顺序需要在无限长的行为中保持，还是仅在有限长的行为中保持？如果允许“有限长的行为中保持”，可以解释为内存对某个处理器的请求始终优先于另一个处理，因此发生了上述有限长的行为。但无论如何也不能解释无限长的行为，因此我们决定仅在有限长的行为中保持某些优先顺序。

另一个问题是：内存没有规定多个处理器之间的顺序，那么某个处理器能否使用另一个处理器未来的值？这自然也是不现实的。不使用未来的值也意味着在某一时刻等价的串行化顺序不因为未来的请求改变。

#### 11.2.3 串行内存

对于“与某种串行化等价”的性质，常常使用一个序列记录这种串行化的历史。我们使用`opQ[p][i]`表示处理器`p`的第`i`个操作，使用`p`、`i`表示`opId`，即
```
opId == {oiv \in [proc: Proc, idx: Nat]: 
            oiv.idx \in DOMAIN opQ[oiv.proc]}
```

系统的行为可以概括为`IssueRequest(proc, req, reg)`、`RespondToRequest(proc, reg)`、`Internal`三种，后两者应当用公平性对系统的特性进一步描述。其中`RespondToRequest`在产生结果后会对`opQ`进行就地修改，因此每个寄存器至多有一个对应的未完成的`opQ`中的操作。读操作会有一个`source`域记录读到数值的来源，这也由`RespondToRequest`进行修改。

在描述规范时，尽量把内部状态描述得可以通过单状态公式表达安全性。本节这个单状态公式就是`Serializable`。

为了固定多个处理器请求的相对串行顺序，使用`opOrder`集合进行记录。如果二元组属于该集合，则表示发生了二元组的两个元素具有相对顺序。`Serializable`使用`opOrder`进行系统性质的描述：`opOrder`包含的元素个数只能增多；`opQ[p]`的顺序符合`opOrder`；读操作会读到相同地址的最后一次写入，或者初始值。最后一个性质表示为存在一个全序`R`满足：
```
\A oi \in opId:
    ("source" \in DOMAIN opQ[oi.proc][oi.idx]) =>
        ~(\E oj \in goodSource(oi):                         /* not exsit such oj
            /\ <<oj, oi>> \in R
            /\ (opQ[oi.proc][oi.idx].source /= InitWr) =>
                (<<opQ[oi.proc][oi.idx].source, oj>> \in R) /* that issued after the source of oi
        )
```

`goodSource(oi)`表示某个`opId`可能作为`source`域的所有请求。读操作的`RespondToRequest`会断定`goodSource(oi)`中存在符合`opOrder'`的请求作为响应。

接下来描述系统的活跃性，比较难的一点是表示内存最终能得到确定、唯一的串行化顺序。一种尝试是
```
\A oi, oj:
    (oi \in opId) /\ (oj in opId) =>
        ((oi /= oj) =>
            WF_<<...>>(/\ Internal
                       /\ (<<oi, oj>> \in opOrder') \/ (<<oj, oi>> \in opOrder'))
        )
```
但是要注意该公式是时序逻辑公式，`(oi \in opId) /\ (oj in opId)`这种单状态公式只是描述初始状态，而不是每个状态。因此需要将其放入公平性之中应用到每个状态。

#### 11.2.4 串行一致性内存

如果取消“某个处理器使用另一个处理器未来的值”的限制，也即某一时刻等价的串行化顺序可以因为未来的请求改变，就得到更简单的串行一致性内存。首先我们使`opQ`变为一个先入先出队列，“队列”仍然保证了单个处理器需要满足的顺序关系，新增的可以出队的特性可以在一定条件下获取未来的处理器值。

此外新增`mem`记录内存中的值。如果在读操作的`opQ`出队的同时，读操作返回的值是`mem`中地址对应的值，多个处理器就能通过`mem`通信，从而满足读操作结果的解释性。而为了表示获取未来的值这一特性，读操作返回的值不必是**当前**`mem`的值，如果它返回一个非当前值则需要等待未来另一个处理器写入这个值时，`opQ`才能出队。我们使用活跃性限定这种对未来的等待不能是毫无根据的无限的等待。

#### 11.2.5 对内存规范的思考

上面提到的内存规范，有的能实现，有的需要过高的计算复杂度（例如在队列中搜索合法状态）才能实现，有的则不能实现。例如串行一致性内存在这种满足规范的场景下是无法实现的：多个处理器相互依赖未来值。

那些不能实现的规范往往是因为其不是状态机绑定的，也就是说某些实现中的公式会阻止一些*步*的发生。按照 8.9.2 节的解释，非状态机绑定是因为公平性描述的动作不能蕴含`Next`。但即便如此，有时为了简单的描述系统，我们还是会使用那些无法实现的规范。