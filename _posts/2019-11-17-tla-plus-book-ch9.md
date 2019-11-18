---
layout: post
title:  "Specifying Systems 第九章笔记"
tags: tech
---

# 9.1 回顾时钟

上一章学习了如何描述未来一定会发生某特性，但是更具有现实意义的是**实时性**，即描述在未来的某段时间里会发生。实时性可以通过增加辅助变量进行描述。以时钟为例，如果我们想要描述`hr`在某段时间内会改变，首先要引入一个`now`变量，并明确新增的诸多细节，例如相对于`now`，`hr`是瞬时改变的吗？`now`的改变是否有最小或者最大粒度……为了描述`hr`两次改变间`now`的变化，可以使用`t`变量记录当下到上次`hr`改变（或者系统启动）经过的时间。

*动作*是公式，因此具有真假值的含义，`TNext == t' = IF HCnxt THEN 0 ELSE t + (now' - now)`中`HCnxt`表达`HCnxt`*步*，因此本公式表示当`HCnxt`发生与否时`t`的变化。

模块中可以继续定义子模块，子模块定义上方的父模块部分在子模块中可见。

当引入`Real`域的时候，对于`now`等增长的变量需要注意上界的问题。为了在规范中排除有上界的`now`数列，可以使用`\A r \in Real : WF_now(NowNext /\ (now' > r))`。即
```
RTnow(v) == LET NowNext == /\ now' \in {r \in Real : r > now} 
                           /\ UNCHANGED v
            IN  /\ now \in Real 
                /\ [][NowNext]_now
                /\ \A r \in Real : WF_now(NowNext /\ (now' > r))
```

# 9.2 更通用的实时性规范

对于实时性的要求，一个更通用的描述是，*动作*`<<A>>_v`在前提满足累计或连续`D`秒到`E`秒的这段时间里必定发生。累计发生在实际中较少使用，因此我们只考虑连续`E`、`D`秒的情况。
```
RTBound(A, v, D, E) == 
  LET TNext(t)  ==  t' = IF <<A>>_v \/ ~(ENABLED <<A>>_v)'
                           THEN 0 
                           ELSE t + (now' - now)

      Timer(t) == (t = 0)  /\  [][TNext(t)]_<<t, v, now>>

      MaxTime(t) == [](t =< E) 

      MinTime(t) == [][A => t >= D]_v 
  IN \EE t : Timer(t) /\ MaxTime(t) /\ MinTime(t)
```
由定义可得，如果`E =< Infinity`，`RTBound(A, v, D, E)`蕴含`WF_v(A)`。当`A`定义为`A1 \/ A2 \/ ...`而`Ai`的满足前提不并发以及前提的不满足只能通过*运行*该*动作*时，有重言式
```
RTBound(A, v, D, E) = RTBound(A1, v, D, E) /\ RTBound(A2, v, D, E) /\ ...
```

# 9.3 实时缓存

前面两节定义的`RTnow`和`RTBound`可以作为通用的描述实时性的模块，例如内存可以将`RTBound(A, v, D, E)`的`A`定义为`(ctl[p] /= "rdy") /\ (ctl'[p] = "rdy")`，即描述处理器`p`恢复空闲的时间。

对于内存的实现——直写式缓存内存而言，如果它可以增加实时性的描述并蕴含上面的实时内存，就可以认为它是实时性内存的实现。然而目前的直写式缓存每个*动作*并不能满足实时性的要求，这是因为由于没有指明调度算法，有的处理器可能始终抢占`memQ`使得其他处理器资源始终受限。这可以对影响`memQ`的*动作*添加 round-robin 等调度算法解决。

具体而言，`RdMiss`和`DoWr`会添加消息到`memQ`上从而抢占其他处理器，所以这两个*动作*要添加 round-robin 作为前提，并更新`lastP`记录行动的上次处理器。round-robin 具体条件如下：
```
position(p) == CHOOSE i \in 1..N : p = (lastP + i) % N
canGoNext(p) == \A q \in Proc : (position(q) < position(p)) => ~ENABLED(RdMiss(q) \/ DoWr(q))
```

# 9.4 芝诺式规范

一个`now`无限增长但是有上界的行为叫做“芝诺式的”。公式`\A r \in Real : WF_now(NowNext /\ (now' > r))`阻止了芝诺式行为的发生，这个公式叫做`NZ`（Non-Zeno）。在仅允许芝诺式行为的系统中，`NZ`始终为`FALSE`。

仅允许芝诺式行为的规范叫做芝诺式规范，例如`RTBound(A, v, D, E)`中`D > E`，意味着`now`不断趋近于`E`。芝诺式规范的形式化定义是：存在有限的行为满足规范的安全性部分，但是不存在无限的行为同时满足安全性和`NZ`。非芝诺式规范等价于状态机绑定的规范。芝诺式规范往往是因为实时性条件限制了一些系统行为的发生，这类似于非状态机闭包。

非芝诺式规范更有用，一般而言，由下面三个合取的规范是非芝诺式的：

1. `Init /\ [][Next]_vars`
2. `RTnow(vars)`
3. 有限个`RTBound(Ai, vars, Di, Ei)`公式，其中
  - `0 =< Di =< Ei =< Infinity`
  - `Ai`是`Next`的子行为
  - 没有*步*同时满足不同的`Ai`

描述更具体实现的规范能容易满足上面的要求，描述更抽象的规范往往找不到合适的`Ai`满足`Ai`是`Next`的子行为。

# 9.5 混合系统规范

TLA+ 将系统描述成了一个一个离散的状态，但是真实世界中的系统往往是有连续物理量的。具有连续变量的规范叫做混合系统规范。这可以通过在离散的时间点上应用积分运算符`Integrate`计算两个状态间的变量变化描述。具体而言，所有离散的变量形成元组`vd`，连续变量（除了`now`）形成元组`vc`，`RTnow`通过如下的公式替代
```
/\ now' \in {r \in Real: r > now}
/\ vc' = Integrate(D, now, now', vc)
/\ UNCHANGED vd
```
其中`D`是微分方程。这个公式的含义是，在公式*运行*的瞬间`vd`不变，通过积分计算`vc`的新状态。

# 9.6 实时性的评论

实时性可以看作是比活跃性更强的描述，在简单系统中可以取代活跃性。通常而言，使用`RTnow`和`RTBound`能满足规范对实时性的描述和证明。