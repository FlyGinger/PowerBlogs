---
title: PyNN 文档笔记
description: 
date: 2021-05-22
author: zenk
draft: false
categories: PyNN
tags: [PyNN, 神经拟态计算]
---

[PyNN](http://neuralensemble.org/PyNN/)（发音与 pine 相同）是一种用于构建神经网络模型的语言，且独立于模拟器。使用 PyNN 的 API 和 Python 编程语言编写的模型代码，无需修改即可运行于多种 PyNN 支持的模拟器后端或者神经拟态（neuronmorphic）硬件系统。

## 安装

待续。

## 构建神经元

一般来说，神经网络构建的基本思路是自底向上的。先描述各个神经元，然后再描述连接各个神经元的网络。PyNN 也遵循这个基本思路。

### 神经元类型（Cell Type）

在 PyNN 中，定义神经元模型的等式组被封装在 `CellType` 类中。PyNN 提供一个标准神经元类型库，其中的各种类型在所有的模拟器后端上都能一致运行。需要注意的是，这里的神经元类型是数学意义上的神经元类型。生物学意义上不同的神经元类型可能可以用参数不同的同一种数学神经元类型表示。下文中会混淆使用数学神经元类型和生物神经元类型两个概念，统称为神经元类型。

举例来说，标准库中提供了神经元类型 `EIF_cond_exp_isfa_ista`。通过不同的参数，可以得到不同的神经元类型 thalamocortical relay neurons 和 cortical neurons。

``` python
>>> # 参数一
>>> ctx_parameters = {'cm': 0.25, 'tau_m': 20.0, 'v_rest': -60, 'v_thresh': -50, 'tau_refrac': RandomDistribution('uniform', [2.0, 3.0], rng=NumpyRNG(seed=4242)), 'v_reset': -60, 'v_spike': -50.0, 'a': 1.0, 'b': 0.005, 'tau_w': 600, 'delta_T': 2.5, 'tau_syn_E': 5.0, 'e_rev_E': 0.0, 'tau_syn_I': 10.0, 'e_rev_I': -80}
>>> # 参数二，基本与参数一相同，只有两项不同
>>> tc_parameters = ctx_parameters.copy().update({'a': 20.0, 'b': 0.0})
>>> # 获取神经元类型
>>> thalamocortical_type = EIF_cond_exp_isfa_ista(**tc_parameters)
>>> cortical_type = EIF_cond_exp_isfa_ista(**ctx_parameters)
```

注意，通过 `CellType` 创建的是**神经元类型**而不是神经元。

可以使用 `get_parameter_names()` 方法查看神经元类型的所有参数。

``` python
>>> IF_cond_exp.get_parameter_names()
['tau_refrac', 'cm', 'tau_syn_E', 'v_rest', 'tau_syn_I', 'tau_m', 'e_rev_E', 'i_offset', 'e_rev_I', 'v_thresh', 'v_reset']
```

可以使用 `default_parameters` 属性查看所有参数的默认值。

``` python
>>> print(IF_cond_exp.default_parameters)
{'tau_refrac': 0.1, 'cm': 1.0, 'tau_syn_E': 5.0, 'v_rest': -65.0, 'tau_syn_I': 5.0, 'tau_m': 20.0, 'e_rev_E': 0.0, 'i_offset': 0.0, 'e_rev_I': -70.0, 'v_thresh': -50.0, 'v_reset': -65.0}
```

### 族群（Population）

> Population 的翻译暂定为族群。

PyNN 是为神经元网络建模而设计的，因此默认的抽象层次是**相同类型神经元构成的族群**，而不是单个神经元。在 PyNN 中，使用 `Population` 类来表示族群。

``` python
>>> tc_cells = Population(100, thalamocortical_type)
>>> ctx_cells = Population(500, cortical_type)
```

创建一个族群时至少需要指定神经元数量和神经元类型。除此之外，还可以指定该族群中神经元的空间结构，神经元状态变量的初值，还有族群的标签。

``` python
>>> from pyNN.space import RandomStructure, Sphere
>>> tc_cells = Population(
...     100, thalamocortical_type,
...     structure=RandomStructure(boundary=Sphere(radius=200.0)),
...     initial_values={'v': -70.0},
...     label="Thalamocortical neurons")
```

``` python
>>> from pyNN.space import Grid2D
>>> from pyNN.random import RandomDistribution
>>> v_init = RandomDistribution('uniform', (-70.0, -60.0))
>>> ctx_cells = Population(
...     500, cortical_type,
...     structure=Grid2D(dx=10.0, dy=10.0),
...     initial_values={'v': v_init},
...     label="Cortical neurons")
```

为了向后兼容，PyNN 也提供了 `create()` 函数作为 `Population` 的别名。以下两行等价（注意参数顺序不同）：

``` python
>>> cells = create(my_cell_type, n=100)
>>> cells = Population(100, my_cell_type)
```

### 视图（View）

对族群的子集进行某些操作的需求也是经常出现的，例如修改参数值，添加连接或者记录数值变化等。可以通过 Python 的切片索引创建视图。

``` python
>>> n = ctx_cells[47]            # 族群中的第 48 个神经元
>>> view = ctx_cells[:80]        # 前 80 个神经元
>>> view = ctx_cells[::2]        # 第 1，3，5……个神经元
>>> view = ctx_cells[45, 91, 7]  # 特定神经元集合
```

也可以通过 `sample()` 函数获取随机神经元集合构成的视图。

``` python
>>> view = ctx_cells.sample(50, rng=NumpyRNG(seed=6538))  # 随机选择 50 个神经元
```

以上所有 `view` 都是一个 `PopulationView` 对象。`PopulationView` 中包含 `Population` 神经元子集的引用。因此通过 `PopulationView` 对神经元做出的任何修改都可以在 `Population` 中感知到，反之亦然。

大部分情况下，`PopulationView` 都可以当作 `Population` 来使用。`PopulationView` 中有一个 `parent` 引用，指向神经元的 `Population` 源。`PopulationView` 中还有一个 `mask` 属性，列出了其中的所有神经元。

``` python
>>> view.parent.label
'Cortical neurons'
>>> view.mask
array([150, 181, 53, 149, 496, 499, 240, 444, 13, 100, 28, 19, 101, 122, 143, 486, 467, 492, 406, 90, 136, 173, 8, 341, 5, 348, 188, 63, 129, 416, 307, 298, 60, 180, 382, 47, 484, 370, 223, 147, 72, 32, 261, 193, 249, 212, 58, 87, 86, 456])
```

### 集群（Assembly）

> Assembly 的翻译暂定为集群。这个翻译不太好，会和 cluster 混淆。

`Assembly` 是 `Population` 和 `PopulationView` 的集合。`Assembly` 可以通过 `Population` 或 `PopulationView` 对象之间的加法得到，也可以直接构造：

``` python
>>> all_cells = tc_cells + ctx_cells
>>> cells_for_plotting = tc_cells[:10] + ctx_cells[:50]
>>> all_cells = Assembly(tc_cells, ctx_cells)
```

可以通过标签获取集群中的族群。

``` python
>>> all_cells.get_population("Thalamocortical neurons")
Population(100, EIF_cond_exp_isfa_ista(<parameters>), structure=RandomStructure(origin=(0.0, 0.0, 0.0), boundary=Sphere(radius=200.0), rng=NumpyRNG(seed=None)), label='Thalamocortical neurons')
```

也可以通过迭代 `Assembly` 的 `populations` 属性来访问各个族群。

### 查看和修改参数值和初始条件

可以使用 `get()` 方法查看族群、视图和集群的参数。

``` python
>>> ctx_cells.get('tau_m')
20.0
>>> all_cells[0:10].get('v_reset')
-60.0
>>> ctx_cells.get(['tau_m', 'cm'])
[20.0, 0.25]
```

如果该集合中的所有神经元的对应参数值相同，那么 `get()` 会返回单个数值，否则会返回一个 `NumPy` 数组。

``` python
>>> ctx_cells.get('tau_refrac')
array([ 2.64655001,  2.15914942,  2.53500179])
```

可以使用 `set()` 方法修改参数的值。

``` python
>>> ctx_cells.set(a=2.0, b=0.2)
```

`initialize()` 方法可以修改模型的初值。

``` python
>>> ctx_cells.initialize(v=RandomDistribution('normal', (-65.0, 2.0)), w=0.0)
```

模型默认的初值可以通过如下方式查看。

``` python
>>> ctx_cells.celltype.default_initial_values
{'gsyn_exc': 0.0, 'gsyn_inh': 0.0, 'w': 0.0, 'v': -70.6}
```

### 向神经元输送电流

PyNN 提供多种电流相关的类，均为 `CurrentSource` 的子类。可以通过 `CurrentSource` 的 `inject_into()` 方法向神经元输送电流。

``` python
>>> pulse = DCSource(amplitude=0.5, start=20.0, stop=80.0)
>>> pulse.inject_into(tc_cells)
```

或者使用族群、视图、集群的`inject()` 方法向神经元输送电流。

``` python
>>> import numpy
>>> times = numpy.arange(0.0, 100.0, 1.0)
>>> amplitudes = 0.1*numpy.sin(times*numpy.pi/100.0)
>>> sine_wave = StepCurrentSource(times=times, amplitudes=amplitudes)
>>> ctx_cells[80:90].inject(sine_wave)
```

### 记录变量

`CellType` 的 `recordable` 属性给出了该神经元类型中的所有可记录的状态变量。

``` python
>>> ctx_cells.celltype.recordable
['spikes', 'v', 'w', 'gsyn_exc', 'gsyn_inh']
```

使用 `record()` 方法指定需要被记录的变量。

``` python
>>> all_cells.record('spikes')
>>> ctx_cells.sample(10).record(('v', 'w'))
```

在模拟结束后，可以使用 `get_data()` 方法取回记录的数据。

``` python
>>> t = run(0.2)
>>> data_block = all_cells.get_data()
```

或者可以使用 `write_data()` 方法把数据直接写到文件中。

``` python
>>> from neo.io import NeoHdf5IO
>>> h5file = NeoHdf5IO("my_data.h5")
>>> ctx_cells.write_data(h5file)
>>> h5file.close()
```

`get_data()` 返回一个 [Neo](http://neuralensemble.org/neo/) `Block` 对象。`write_data()` 方法同样利用了 Neo。

### 单个神经元

单个神经元是一个 `ID` 对象。注意 `ID` 在不同的模拟器后端中可能有不同的值。

``` python
>>> tc_cells[47]
709
```

可以通过 `parent` 属性访问神经元所在的族群。

``` python
>>> a_cell = tc_cells[47]
>>> a_cell.parent.label
'Thalamocortical neurons'
```

使用 `Population.id_to_index()` 可以获取神经元在族群中的原索引值。

``` python
>>> tc_cells.id_to_index(a_cell)
47
```

可以直接通过 `ID` 访问神经元的各个参数，也可以通过 `set_parameters()` 方法来修改参数的值：

``` python
>>> a_cell.tau_m
20.0
>>> a_cell.set_parameters(tau_m=10.0, cm=0.5)
>>> a_cell.tau_m
10.0
>>> a_cell.cm
0.5
```

## 构建网络

从概念上来说，突触由突触前部（pre-synaptic structure）、突触间隙（synaptic cleft）和突出后部（post-synaptic structure）构成。在 PyNN 中，突触后反应的时间动态（temporal dynamics）由突触后神经元模型处理。突触后反应的烈度（即突触权重，synaptic weight），权重的时间动态（即突触可塑性，synaptic plasticity）和连接的延迟由突触模型处理。

PyNN 中的突触连接至少需要提供两种属性，权重和延迟。

本节中用神经元群指代族群、视图或集群对象。

### 突触类型（Synapse Type）

突触模型被封装在 `SynapseType` 类中。PyNN 提供一个突触类型的标准库，能够保证在各个后端模拟器上的模拟结果是一致的。

PyNN 的默认突触类型，也是最简单的突触类型，就是固定权重突触。

``` python
syn = StaticSynapse(weight=0.04, delay=0.5)
```

权重一般分为两类。一类是基于电流的权重，单位是纳安（nanoamps）。抑制性的突触中，基于电流的权重是负数。另一类是基于电导（电导是电阻的倒数）的权重，单位是微西门子（microsiemens）。基于电导的权重总是正数。另外，延迟的单位是毫秒。

可以使用随机数作为参数值，也可以把参数值定义为突触前和突触后神经元距离的函数：

``` python
w = RandomDistribution('gamma', [10, 0.004], rng=NumpyRNG(seed=4242))
syn = StaticSynapse(weight=w, delay=0.5)
```

``` python
syn = StaticSynapse(weight=w, delay="0.2 + 0.01*d")
```

PyNN 提供一个短期突触可塑性（short-term synaptic plasticity）的标准模型。

``` python
# depressing - 抑制
depressing_synapse = TsodyksMarkramSynapse(weight=w, delay=0.2, U=0.5, tau_rec=800.0, tau_facil=0.0)

# facilitating - 促进
tau_rec = RandomDistribution('normal', [100.0, 10.0])
facilitating_synapse = TsodyksMarkramSynapse(weight=w, delay=0.5, U=0.04, tau_rec=tau_rec)
```

脉冲时序依赖可塑性（STDP，spike-timing-dependent plasticity）与其他的标准模型略有不同。STDP 突触类型由两个独立的权重依赖（weight-dependence）和时序依赖（timing-dependence）组件构成。

``` python
stdp = STDPMechanism(
    weight=0.02,  # 权重初值
    delay="0.2 + 0.01*d",
    timing_dependence=SpikePairRule(tau_plus=20.0, tau_minus=20.0, A_plus=0.01, A_minus=0.012),
    weight_dependence=AdditiveWeightDependence(w_min=0, w_max=0.04))
```

### 连接算法

PyNN 中每种用于确定哪些突触前神经元连接哪些突触后神经元的算法（connection algorithm，也称为 connection method 或 wiring method）被封装在单独的类中。

每一个这样的类都是 `Connector` 的子类，并且实现了 `connect(Projection)` 方法。注意 `Connector` 对象只是一种连接方法，并未建立实际的连接。

突触前神经元群中的每个神经元都连接到突触后神经元群的每个神经元。`AllToAllConnector` 类提供了这样的连接，其构造器只需要一个可选参数 `allow_self_connections`，其默认值为 `True`，代表突触前神经元群和突触后神经元群是同一个神经元群。

``` python
connector = AllToAllConnector(allow_self_connections=False)
```

`OneToOneConnector` 要求突触前和突触后神经元群中的神经元数量相同。两个神经元群中索引相同的神经元将被连接到一起。

``` python
connector = OneToOneConnector()
```

连接集代数（CSA，Connection Set Algebra）是一个复杂的系统，可以用精简的语法描述精细的连接模式。

``` python
import csa
cset = csa.full - csa.oneToOne
connector = CSAConnector(cset)
```

可以通过列表列出所有的连接。

``` python
connections = [
  (0, 0, 0.0, 0.1),
  (0, 1, 0.0, 0.1),
  (0, 2, 0.0, 0.1),
  (1, 5, 0.0, 0.1)
]
connector = FromListConnector(connections, column_names=["weight", "delay"])
```

连接模式也可以从文件中读取，但是文件开头需要有一行“表头”，指定每一列的含义。

``` python
# columns = ["i", "j", "weight", "delay", "U", "tau_rec"]
```

表体中的列需要使用空格间隔。

``` python
connector = FromFileConnector("connections.txt")
```

连接矩阵。

``` python
connections = numpy.array([[0, 1, 1, 0],
                           [1, 1, 0, 1],
                           [0, 0, 1, 0]], dtype=bool)
connector = ArrayConnector(connections)
```

### 投影（Projection）

投影是两个神经元群之间的连接集合。构造投影的同时，PyNN 会进行实际的连接。构造投影必须指定突触前神经元群，突触后神经元群，连接方法，突触类型。

还有一些可选参数。首先是突触后机制，如果不指定，PyNN 会根据突触类型的权重参数自动选择。然后是标签，不指定时会自动生成。最后是 `Space` 对象，指定了突触前和突触后神经元间的距离应该如何计算。

简单的例子。

``` python
excitatory_connections = Projection(
    pre, post, AllToAllConnector(), StaticSynapse(weight=0.123))
```

复杂的例子。

``` python
rng = NumpyRNG(seed=64754)
sparse_connectivity = FixedProbabilityConnector(0.1, rng=rng)
weight_distr = RandomDistribution('normal', [0.01, 1e-3], rng=rng)
facilitating = TsodyksMarkramSynapse(
    U=0.04, tau_rec=100.0, tau_facil=1000.0,
    weight=weight_distr, delay=lambda d: 0.1+d/100.0)
space = Space(axes='xy')
inhibitory_connections = Projection(
    pre, post,
    connector=sparse_connectivity,
    synapse_type=facilitating,
    receptor_type='inhibitory',
    space=space,
    label="inhibitory connections")
```

使用 `get()` 方法访问投影中的连接。

``` python
>>> excitatory_connections.get('weight', format='list')[3:7]
[(3, 0, 0.123), (4, 0, 0.123), (5, 0, 0.123), (6, 0, 0.123)]
>>> inhibitory_connections.get('delay', format='array')[:3,:5]
array([[  nan,   nan,   nan,   nan,  0.14],
       [  nan,   nan,   nan,  0.12,  0.13],
       [ 0.12,   nan,   nan,   nan,   nan]])
```

`format` 是 `list` 时，可以使用 `with_address` 来控制是否输出神经元索引。

``` python
>>> excitatory_connections.get('weight', format='list', with_address=False)[3:7]
[0.123, 0.123, 0.123, 0.123]
```

使用 `set()` 方法设置投影中连接的参数。

``` python
>>> excitatory_connections.set(weight=0.02)
>>> excitatory_connections.set(weight=RandomDistribution('gamma', [1, 0.1]), delay=0.3)
>>> inhibitory_connections.set(U=numpy.linspace(0.4, 0.6, len(inhibitory_connections)), tau_rec=500.0, tau_facil=0.1)
```

也可以直接访问 `connections`。

``` python
>>> for c in list(inhibitory_connections.connections)[:5]:
...     c.weight *= 2
```
