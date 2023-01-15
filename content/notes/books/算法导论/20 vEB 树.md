---
title: 算法导论 20 vEB 树
description: 
date: 2021-08-12
author: zenk
draft: false
categories: 算法导论
tags: [数据结构与算法]
---

在前面的章节中已经介绍了很多支持优先队列操作的数据结构，包括二叉堆、红黑树、斐波那契堆。以上每个数据结构中都有至少一个重要操作等最坏情况运行时间或者分摊运行时间是 $O( \lg n )$ 。这是因为这些数据结构是基于键之间的比较来设计的，在线性时间排序那一章我们证明了基于比较的排序算法运行时间是 $\Omega( n \lg n )$ 。假如基于比较的类似数据结构能够同时将 `INSERT` 和 `EXTRACT-MIN` 的运行时间限制在 $o( \lg n )$ ，那么就可以通过连续 $n$ 次 `INSERT` 和连续 $n$ 次 `EXTRACT-MIN` 操作对 $n$ 个数据进行排序，并且运行时间是 $o( n \lg n )$ 。

> 不严格地讲，各个符号的含义： $O$ 代表“小于等于”， $\Omega$ 是“大于等于”， $\Theta$ 是“等于”， $o$ 是“小于”， $\omega$ 是“大于”。

换句话说，只要避免比较就可以做到 $o( \lg n )$ 。本章就讲介绍这样一种数据结构： vEB 树（ van Emde Boas tree），它支持优先队列的各个操作以及其他的几个操作，并且所有操作的最坏情况运行时间都是 $O( \lg \lg n )$ 。为此付出代价则是 vEB 树只能保存范围在 $0$ 到 $n - 1$ 的整数，并且不可以重复。

$n$ 现在有两种含义，第一种是代表数据结构内含有的元素的数量，第二种是数据结构内可以保存的数据的范围。为了避免歧义，我们使用 $u$ 来表示第二种含义，因此 vEB 树的各个操作的运行时间是 $O( \lg \lg u )$ 。我们将 $\\{ 0, 1, 2, \dots, u - 1 \\}$ 称为**全集**（ universe ），将 $u$ 称为全集大小。此外，我们还假设 $u = 2^k$ ，其中 $k$ 是整数且 $k \geqslant 1$ 。

为了简化讨论，我们忽略元素中的卫星数据，只考虑键的问题。因此 `SEARCH` 操作可以弱化成 `MEMBER` ， `MEMBER(x)` 只关心数据结构中是否含有 `x` 。

## 20.1 初步方法

本小节会介绍几种动态集合中保存数据的方法，虽然没有一个可以达成 $O( \lg \lg n )$ 的运行时间，但是可以帮助我们理解 vEB 树的设计。

### 直接寻址

我们可以创建一个含有 $u$ 个比特的数组 `A[0..u-1]` 。如果集合中含有元素 $x$ ，那么就令 `A[x] = 1` ，否则就令 `A[x] = 0` 。这种数据结构的 `INSERT` 、 `DELETE` 和 `MEMBER` 操作都可以在 $O(1)$ 的运行时间内完成。然而其他的操作，例如 `MINIMUM` 、 `MAXIMUM` 、 `SUCCESSOR` 和 `PREDECESSOR` 在最坏情况下需要 $\Theta(u)$ 的运行时间。

### 叠加一个二叉树

![叠一个二叉树](/notes/算法导论/images/20-00.png#center)

如上图所示，我们可以在直接寻址方法中的 `A` 上叠加一个二叉树，令 `A` 中的每个元素称为二叉树的一个叶节点。二叉树的内部节点为 1 ，当且仅当以它为根节点的树的叶节点中含有 1 。换句话说，节点中的值等于两个孩子中的值的逻辑或。如此以来，表现差的四个操作都可以得到优化：

- `MINIMUM` ：从根节点开始，总是访问含有 1 的最左边的子节点。
- `MAXIMUM` ：从根节点开始，总是访问含有 1 的最右边的子节点。
- `SUCCESSOR` ：从节点 `x` 开始，不断向上访问，直到发现当前节点 `x'` 是某节点的左孩子，并且 `x'` 的兄弟节点 `z` 是 1 。此时从 `z` 开始，总是访问含有 1 的最左边的子节点。
- `PREDECESSOR`：与 `SUCCESSOR` 的流程对称。

上面的图中展示了寻找节点 14 的前任的过程。然而，插入和删除操作也需要做出一些修改，因为不但要修改 `A` 中的内容，还要修改二叉树中的内容。

最终我们得到的数据结构中， `MEMBER` 操作可以在 $O(1)$ 内完成，而其他操作都需要 $O(\lg u)$ 的时间。这似乎比红黑树还要更快一些，不过如果树中保存的数据数量 $n$ 远小于 $u$ 时，还是红黑树会更快一些。

### 叠加一个高度固定的树

为什么我们不叠加一个度更大的树呢？假设全集的大小 $u = 2^{ 2k }$ ，那么 $\sqrt{u}$ 就是一个整数。这次我们叠加的不是二叉树，而是一棵度为 $\sqrt{u}$ 的树。

![叠加一个高度固定的树](/notes/算法导论/images/20-01.png#center)

上图中就展示了这样一棵树，它的高度固定为 $2$ 。和从前一样，每个节点中的值等于它所有子节点中值的逻辑或。现在，每个深度为 1 的 $\sqrt{u}$ 个节点都汇总了 $\sqrt{u}$ 个叶节点中的值。我们可以把这 $\sqrt{u}$ 个节点看成是一个数组 $summary[ 0 .. \sqrt{u} - 1]$ ，并且 $summary[i]$ 是 1 当且仅当 $A[ i \sqrt{u} .. ( i + 1 ) \sqrt{u} - 1 ]$ 中含有 1 。我们把 `A` 中这 $\sqrt{u}$ 个比特称为一个**集群**（ cluster ）。对于给定键值 $x$ ，它应该属于第 $\lfloor x / \sqrt{u} \rfloor$ 个集群。

![叠加一个高度固定的树](/notes/算法导论/images/20-02.png#center)

现在 `INSERT` 操作可以在 $O(1)$ 内完成：设置 $A[x]$ 和 $summary[ \lfloor x / \sqrt{u} \rfloor ]$ 为 1 即可。而其他的操作可以在 $O(\sqrt{u})$ 的运行时间内完成：

- 为了寻找最小值，我们首先找到 `summary` 中最左边的 1 ，比如说是 `summary[i]` ，然后再到第 $i$ 个集群中搜索最左边的 1 。寻找最大值的过程类似。
- 为了寻找 $x$ 的后继，首先寻找本集群中 $x$ 右边是否有 1 。如果有，那么它就是 $x$ 的后继，否则我们到 `summary` 数组中搜索 $x$ 对应的位置的右边的第一个 1 ，然后到对应的集群中寻找第一个 1 。寻找前任的过程类似。
- 为了删除 $x$ ，首先令 $i = \lfloor x / \sqrt{u} \rfloor$ 。将 $A[x]$ 设置为 0 ，然后令 $summary[i]$ 是第 $i$ 个集群中所有比特的逻辑或。

这种数据结构的大多数操作都需要 $O(\sqrt{u})$ 的运行时间，似乎比上一种数据结构要差。但是建立度为 $\sqrt{u}$ 的树的思想是构建 vEB 树的非常重要的点。

## 20.2 递归结构

上面我们定义了一种结构，其中有 $\sqrt{u}$ 个 `summary` 结构，每个 `summary` 结构管理 $\sqrt{u}$ 个子结构。在本章中，我们定义一种递归结构。首先还是构造 $\sqrt{u} = u^{ 1 / 2 }$ 个结构，每个管理 $\sqrt{u}$ 个子结构。子结构递归地管理 $u^{ 1 / 4 }$ 个子子结构，子子结构递归地管理 $u^{ 1 / 8 }$ 个子子子结构，直到某层递归中的结构只管理 2 个元素。

为了简化讨论，我们假设 $u = 2^{ 2^k }$ ，以使得每层子结构的大小都是整数。在实际使用中这个限制有些太严格，下一小节中我们将会把这个限制放宽到 $u = 2^k$ 。做出这样的限制只是为了本章能够专注于介绍 vEB 树本身。

虽然有些突兀，但是请看下面的递推式：

$$
T(u) = T(\sqrt{u}) + O(1)
$$

我们可以证明 $T(u) = O( \lg \lg u )$ 。首先，令 $m = \lg u$ ，那么 $u = 2^m$ ，带入上式：

$$
T( 2^m ) = T( 2^{ m / 2 } ) + O(1)
$$

使用新的符号 $S(m) = T( 2^m )$ ，那么递推式会变成：

$$
S(m) = S( m / 2 ) + O(1)
$$

根据主定理的第二种情况， $S(m) = O( \lg m )$ ，所以 $T(u) = T( 2^m ) = S(m) = O( \lg m ) = O( \lg \lg u )$ 。

上面这个递推式可以帮助我们设计数据结构。根据递推式，我们需要把问题规模为 $u$ 的问题缩减为问题规模为 $\sqrt{u}$ 的问题，同时这个缩减过程要在常数时间内完成。另一方面，我们可以这样理解 $\lg \lg u$ 这个结构：假设 $u$ 是 $\lg u$ 位的数字，每次在递归中下降一级， $u$ 的位数就会减少一半，最终变成只剩一位。举例来说，如果一开始有 $b$ 位，然后每次减少一半，那么经过 $\lg b$ 次之后就会只剩一位。由于 $b = \lg u$ ，所以经过 $\lg \lg u$ 次之后我们就到达了 vEB 树中的最小单位，也就是 $u = 2$ 的那一级。

为了简化讨论，我们定义一些小函数。给定数字 $x$ ，容易知道它位于第 $\lfloor x / \sqrt{u} \rfloor$ 个集群，并且是该集群中的 $x \bmod \sqrt{u}$ 。如果我们知道某个元素是第 $x$ 个集群中的第 $y$ 个元素，那么它应该是 $x \sqrt{u} + y$ 。

$$
\begin{aligned}
high(x) & = \lfloor x / \sqrt{u} \rfloor \\\\
low(x) & = x \bmod \sqrt{u} \\\\
index(x, y) & = x \sqrt{u} + y
\end{aligned}
$$

### 原始 vEB 结构

![原始 vEB 结构](/notes/算法导论/images/20-03.png#center)

对于全集 $\\{ 0, 1, 2, \dots, u - 1 \\}$ ，定义**原始 vEB 结构**（ proto-vEB structure, proto van Emde Boas structure ），写作 $proto-vEB(u)$ 。每个 $proto-vEB(u)$ 包含一个属性 $u$ ，代表了它的全集大小。此外，

- 如果 $u = 2$ ，那么它是基础大小，此时 $proto-vEB(u)$ 只有 $A[ 0 .. 1 ]$ 这两个比特。
- 否则，如果 $u = 2^{ 2^k }$ ，其中 $k \geqslant 1$ ，那么 $u \geqslant 4$ 。此时 $proto-vEB(u)$ 中还包含：
  - 一个 $summary$ 指针，指向一个 $proto-vEB(\sqrt{u})$ 。
  - 一个数组 $cluster[ 0 .. u - 1 ]$ ，包含了 $\sqrt{u}$ 指针，每一个都指向一个 $proto-vEB(\sqrt{u})$ 。

下图中展示了一个完整的例子，在 $u = 16$ 的 vEB 树中储存了 $\\{ 2, 3, 4, 5, 7, 14, 15 \\}$ 这些元素。

![一棵 vEB 树](/notes/算法导论/images/20-04.png#center)

### 原始 vEB 树上的操作

#### 搜索

``` plaintext
PROTO-VEB-MEMBER(V, x)
  if V.u == 2
    return V.A[x]
  else
    return PROTO-VEB-MEMBER(V.cluster[high(x)], low(x))
```

首先是 `MEMBER` 操作。 `MEMBER` 的思路非常简单，就是一个递归地搜索过程。由于运行时间满足递推式 $T(u) = T(\sqrt{u}) + O(1)$ ，所以 $T(u) = O( \lg \lg u )$ 。

#### 最值

``` plaintext
PROTO-VEB-MINIMUM(V)
  if V.u == 2
    if V.A[0] == 1
      return 0
    else if V.A[1] == 1
      return 1
    else
      return nil
  else
    min-cluster = PROTO-VEB-MINIMUM(V.summary)
    if min-cluster == nil
      return nil
    else
      offset = PROTO-VEB-MINIMUM(V.cluster[min-cluster])
      return index(min-cluster, offset)
```

然后是 `MINIMUM` 操作。 `MINIMUM` 较为复杂一些：如果当前已经到了最小块，那么直接检查两个比特位；否则就在 `summary` 中寻找最小值（等于真正的最小值所在的集群的下标），然后再到对应的集群中搜索最小值。唯一的问题在于， `MINIMUM`  的递推式是：

$$
T(u) = 2T(\sqrt{u}) + O(1)
$$

令 $u = 2^m$ ：

$$
T( 2^m ) = 2T( 2^{ m / 2 } ) + O(1)
$$

令 $S(m) = T(2^m)$ ：

$$
S(m) = 2S( m / 2 ) + O( 1 )
$$

根据主定理情况 1 ， $S(m) = \Theta(m)$ 。所以 $T(u) = T(2^m) = S(m) = \Theta(m) = \Theta(\lg u)$ 。因此寻找最小值的操作的运行时间是 $\Theta(\lg u)$ 。

#### 前任和后继

``` plaintext
PROTO-VEB-SUCCESSOR(V, x)
  if V.u == 2
    if x == 0 and V.A[1] == 1
      return 1
    else
      return nil
  else
    offset = PROTO-VEB-SUCCESSOR(V.cluster[high(x)], low(x))
    if offset != nil
      return index(high(x), offset)
    else
      succ-cluster = PROTO-VEB-SUCCESSOR(V.summary, high(x))
      if succ-cluster = nil
        return nil
      else
        offset = PROTO-VEB-MINIMUM(V.cluster[succ-cluster])
        return index(succ-cluster, offset)
```

`SUCCESSOR` 更为复杂一些。首先，如果 $u = 2$ ，那么作为一种简单情况直接处理。否则，我们先在本集群内寻找 $x$ 的后继者，也就是 `PROTO-VEB-SUCCESSOR(V.cluster[high(x)], low(x))` 。如果本集群内没有后继者，那么就搜索后继者所在的集群，这是通过 `PROTO-VEB-SUCCESSOR(V.summary, high(x))` 实现的。最后，我们搜索后继者所在集群的最小值，也就找到了后继者。在最坏情况下，运行时间的递推式：

$$
\begin{aligned}
T(u) & = 2T(\sqrt{u}) + \Theta(\lg\sqrt{u})
& = 2T(\sqrt{u}) + \Theta(\lg u)
\end{aligned}
$$

通过和之前类似的证明方法，可以得到 $T(u) = \Theta(\lg u \lg \lg u)$ ，这比寻找最小值还要更慢一些。

#### 插入

``` plaintext
PROTO-VEB-INSERT(V, x)
  if V.u == 2
    V.A[x] = 1
  else
    PROTO-VEB-INSERT(V.cluster[high(x)], low(x))
    PROTO-VEB-INSERT(V.summary, high(x))
```

插入操作与最值类似，由于递归调用了两次，所以运行时间是 $O(\lg u)$ 。

#### 删除

我们甚至无法写出 $O(\lg u)$ 的删除操作。因为插入某元素时，可以简单地令 $summary$ 中对应的位为 1 ，而删除时却不能直接让其为 0 。为了完成删除操作，我们需要遍历某些位置才能得到最终的值。

显然，为了达成 $O(\lg \lg u)$ 的运行时间，我们需要让所有的操作都只递归调用一次。

## 20.3 vEB 树

熟悉了 vEB 树的基本结构之后，我们将 $u$ 的限制放宽到 $u = 2^k$ ，其中 $k$ 是任意正整数。但是这样一来，求平方根是就会遇到结果不是整数的情况。当 $u = 2^{2k}$ 时，可以求得整数的平方根。但是当 $u = 2^{2k+1}$ 时平方根就不是整数。注意到此时 $u = 2^{k+1} 2^k$ ，我们令 $\sqrt[\uparrow]{u} = 2^{k+1}$ ， $\sqrt[\downarrow]{u} = 2^k$ 。扩展到任意情况，当 $u = 2^k$ 时， $k = \lg u$ ，有 $\sqrt[\uparrow]{u} = 2^{\lceil (\lg u)/2 \rceil}$ ， $\sqrt[\downarrow]{u} = 2^{\lfloor (\lg u)/2 \rfloor}$ 。上一小节中定义的符号也需要改动一下：

$$
\begin{aligned}
high(x) & = \lfloor x / \sqrt[\downarrow]{u} \rfloor \\\\
low(x) & = x \bmod \sqrt[\downarrow]{u} \\\\
index(x, y) & = x \sqrt[\downarrow]{u} + y
\end{aligned}
$$

### vEB 树

![vEB 树的节点](/notes/算法导论/images/20-05.png#center)

本节中的 vEB 树节点相较于 proto-vEB 树的节点多了两个属性：`max` 和 `min` ，分别保存了以本节点为根节点的树中的最大值和最小值。注意，保存在 `min` 中的数据不会再出现在集群中，而保存在 `max` 中的数据还会出现在集群中。如果严格地说， `min` “保存”了 vEB 树中的最小值，而 `max` “记录”了 vEB 树中的最大值。在最小规模的节点中， $u = 2$ 时，我们不需要 `A` 数组，只需要把数据保存在 `max` 和 `min` 中即可。

![一棵 vEB 树](/notes/算法导论/images/20-06.png#center)

上图中给出了一棵真正的 vEB 树，其中保存了 $\\{2,3,4,5,7,14,15\\}$ 。 `max` 和 `min` 可以通过如下作用来帮助我们减少递归：

1. 求取最值的操作无需递归，直接访问 `max` 和 `min` 即可。
2. 求取前任和后继的操作中，通过比较 `x` 和 `max` 、 `min` ，可以直接知道前任或后继是否位于本集群，从而减少递归次数。
3. 通过 `max` 和 `min` ，我们可以直接知道一个 vEB 是为空（ `max` 和 `min` 都是 `nil` ），还是只保存了一个数据（ `max` 和 `min` 不是 `nil` 并且相等），还是至少保存了两个数据（ `max` 和 `min` 不是 `nil` 并且不相等）。
4. 如果我们知道 vEB 树为空，我们可以通过直接修改 `max` 和 `min` 的方式插入新节点。类似的，当 vEB 树中只有一个数据时，我们可以通过只修改 `max` 和 `min` 的方式删除该数据。

再回来看递推式，此时无法满足严格的相等关系了：

$$
T(u) \leqslant T(\sqrt[\uparrow]{u}) + O(1)
$$

令 $u = 2^m$ ：

$$
T(2^m) \leqslant T(2^{\lceil m/2 \rceil}) + O(1)
$$

注意到对于任意 $m \geqslant 2$ 都有 $\lceil m / 2 \rceil \leqslant 2m/3$ ：

$$
T(2^m) \leqslant T(2^{2m/3}) + O(1)
$$

令 $S(m) = T(2^m)$ ：

$$
S(m) \leqslant S(2m/3) + O(1)
$$

根据主定理的情况 2 ，有 $S(m) = O(\lg m)$ ，从而 $T(u) = O(\lg \lg u)$ 。

### vEB 树上的操作

#### vEB 树上的最值

``` plaintext
VEB-TREE-MINIMUM(V)
  return V.min

VEB-TREE-MAXIMUM(V)
  return V.max
```

显然，以上操作只需要 $O(1)$ 的运行时间。

#### vEB 树上的搜索

``` plaintext
VEB-TREE-MEMBER(V, x)
  if x == V.min or x == V.max
    return TRUE
  else if V.u == 2
    return FALSE
  else
    return VEB-TREE-MEMBER(V.cluster[high(x)], low(x))
```

搜索的运行时间是 $O(\lg \lg u)$ 。

#### vEB 树上的前任和后继

``` plaintext
VEB-TREE-SUCCESSOR(V, x)
  if V.u == 2
    if x == 0 and V.max == 1
      return 1
    else
      return nil
  else if V.min != nil and x < V.min
    return V.min
  else
    max-low = VEB-TREE-MAXIMUM(V.cluster(high(x)))
    if max-low != nil and low(x) < max-low
      offset = VEB-TREE-SUCCESSOR(V.cluster[high(x)], low(x))
      return index(high(x), offset)
    else
      succ-cluster = VEB-TREE-SUCCESSOR(V.summary, high(x))
      if succ-cluster == nil
        return nil
      else
        offset = VEB-TREE-MINIMUM(V.cluster[succ-cluster])
        return index(succ-cluster, offset)
```

在 `max` 和 `min` 的帮助下，我们避免了两次递归，现在搜索后继的算法只会递归一次了。由于 `min` 的特殊性，寻找前任的算法流程稍微有些不同。

``` plaintext
VEB-TREE-PREDECESSOR(V, x)
  if V.u == 2
    if x == 1 and V.min == 0
      return 0
    else
      return nil
  else if V.max != nil and x > V.max
    return V.max
  else
    min-low = VEB-TREE-MINIMUM(V.cluster(high(x)))
    if min-low != nil and low(x) > min-low
      offset = VEB-TREE-PREDECESSOR(V.cluster[high(x)], low(x))
      return index(high(x), offset)
    else
      pred-cluster = VEB-TREE-PREDECESSOR(V.summary, high(x))
      if pred-cluster == nil
        if V.min != nil and x > V.min
          return V.min
        else
          return nil
      else
        offset = VEB-TREE-MAXIMUM(V.cluster[pred-cluster])
        return index(pred-cluster, offset)
```

在 `VEB-TREE-PREDECESSOR` 中，多出来的几行是为了检查一个特殊情况。在 `VEB-TREE-SUCCESSOR` 中，如果后继不在本集群，那么一定处于后面的集群（如果存在的话）。而在 `VEB-TREE-PREDECESSOR` 中，如果前任不在本集群，那么它有可能位于前面的集群，也有可能位于 `V.min` 。

#### vEB 树上的插入

在 `PROTO-VEB-INSERT` 中，我们需要两次递归调用，一次向集群中插入，一次向 `summary` 中插入。借助 `max` 和 `min` ，我们可以将两次递归调用减少为一次。当利用 `max` 和 `min` 得知当前树为空时，我们只需要设定 `max` 和 `min` ，然后递归调用向 `summary` 中插入即可；当利用 `max` 和 `min` 得知树不为空时，可知 `summary` 中已经被插入好了，我们只需要递归调用插入到集群中即可。

``` plaintext
VEB-TREE-INSERT(V, x)
  if V.min == nil
    V.min = x
    V.max = x
  else
    if x < V.min
      exchange x and V.min
    if V.u > 2
      if VEB-TREE-MINIMUM(V.cluster[high(x)]) == nil
        VEB-TREE-INSERT(V.summary, high(x))
        V.cluster[high(x)].min = low(x)
        V.cluster[high(x)].max = low(x)
      else
        VEB-TREE-INSERT(V.cluster[high(x)], low(x))
    if x > V.max
      V.max = x
```

#### vEB 树上的删除

``` plaintext
VEB-TREE-DELETE(V, x)
  if V.min == V.max
    V.min = nil
    V.max = nil
  else if V.u == 2
    if x == 0
      V.min = 1
    else
      V.max = V.min
  else
    if x == V.min
      first-cluster = VEB-TREE-MINIMUM(V.summary)
      x = index(first-cluster, VEB-TREE-MINIMUM(V.cluster[first-cluster]))
      V.min = x
    VEB-TREE-DELETE(V.cluster[high(x)], low(x)) // recursive 1
    if VEB-TREE-MINIMUM(V.cluster[high(x)]) == nil
      VEB-TREE-DELETE(V.summary, high(x)) // recursive 2
      if x == V.max
        summary-max = VEB-TREE-MAXIMUM(V.summary)
        if summary-max == nil
          V.max = V.min
        else
          V.max = index(summary-max, VEB-TREE-MAXIMUM(V.cluster[summary-max]))
    else if x == V.max
      V.max = index(high(x), VEB-TREE-MAXIMUM(V.cluster[high(x)]))
```

删除的过程如上。第一个分支处理只含有一个元素的情况，此时只需要设置 `max` 和 `min` 。如果不满足第一个分支的条件，说明当前树中最少有两个元素。第二个分支处理 $u = 2$ 的情况，这时候也只需要设置 `max` 和 `min` 。第三个分支则是更普遍的情况。

如果 `x == V.min` ，我们可以直接删除 `V.min` 中保存的元素。但是我们还需要找到一个元素来填补 `V.min` 的位置，不过这也容易，我们只要寻找除了 `min` 之外的最小值来填补当前的空缺，然后删除那个元素即可。删除目标元素之后，可能会发生某个集群为空的情况。此时，我们需要额外处理 `summary` 中的数据。最后，还要考虑 `x == V.max` 的情况。

第一眼看时，我们会发现删除的算法中有两个递归调用。但是仔细分析一下，第二个递归被调用，当且仅当 `x` 所在的集群被删空。如果 `x` 所在的集群删掉 `x` 之后为空，那说明 `x` 所在的集群只有 `x` 这一个元素。如果 `x` 所在的集群只有一个元素，那么第一个递归就只需要常数时间。综上所述，第二个递归被调用当且仅当第一个递归仅花费常数时间。因此，删除操作的运行时间也是 $O(\lg \lg u)$ 。

### vEB 树的空间复杂度

为了简化讨论，这里令 $u = 2^{2^k}$ ，其中 $k$ 是非负整数。假设大小为 $u$ 的 vEB 树占用的空间是 $P(u)$ ，可以写出递推式：

$$
P(u) = (\sqrt{u} + 1)P(\sqrt{u}) + \Theta(\sqrt{u})
$$

其中，大小为 $u$ 的节点中有一个 `cluster` 数组，其长度为 $\Theta(\sqrt{u})$ ，数组中每个指针都指向一个大小为 $\sqrt{u}$ 的节点，再加上一个大小为 $\sqrt{u}$ 的 `summary` 节点，就完成了上述递推式。

接下来通过数学归纳法证明 $P(u) = O(u)$ 。首先是基础情况 $u = 2$ ，我们知道此时只是用了 `max` 和 `min` ，占用的空间是一个常数。然后是递推情况，假设对于所有的 $v < u$ ，都有 $P(v) = O(v)$ 。

$$
\begin{aligned}
P(u) & = (\sqrt{u} + 1)P(\sqrt{u}) + \Theta(\sqrt{u}) \\\\
& = (\sqrt{u} + 1)O(\sqrt{u}) + \Theta(\sqrt{u}) \\\\
& = \sqrt{u} O(\sqrt{u}) + O(\sqrt{u}) + \Theta(\sqrt{u}) \\\\
& = O(u) + O(\sqrt{u}) + \Theta(\sqrt{u}) \\\\
& = O(u + 2\sqrt{u}) \\\\
& = O(u)
\end{aligned}
$$

因此， vEB 树会占用与 $O(u)$ 的空间。另外，由于 vEB 树需要在使用之前先初始化完成，初始化所需要的时间与 vEB 树所占用的空间成正比，因此初始化所需要的运行时间也是 $O(u)$ 。
