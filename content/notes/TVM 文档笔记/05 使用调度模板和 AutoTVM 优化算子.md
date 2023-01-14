---
title: TVM 文档笔记 05 使用调度模板和 AutoTVM 优化算子
description: 
date: 2021-10-09
author: zenk
draft: false
categories: TVM
tags: [TVM, 编译]
---

原文位于[此处](https://tvm.apache.org/docs/tutorials/get_started/autotvm_matmul_x86.html)。

在本教程中，我们介绍如何使用 TVM 张量表达式编写调度模板。 AutoTVM 可以搜索这样的调度模板，找到最优调度。

本教程基于前面的使用 TE 编写矩阵乘法的教程。

自动调优分为两步：

- 定义搜索空间。
- 在该搜索空间上运行一个搜索算法。

在本教程中将会介绍如何使用 TVM 完成这两步。整个工作流还是使用矩阵乘法的例子来展示。

## 安装依赖

为了使用 AutoTVM 包，我们需要安装以下额外的依赖。

``` bash
pip3 install --user psutil xgboost cloudpickle
```

为了让 TVM 更快地进行优化，建议使用 Cython 作为 TVM 的 FFI （ Foreign Function Interface ，语言交互接口）。在 TVM 的根目录运行以下命令：

``` bash
pip3 install --user cython
sudo make cython3
```

现在我们回到 Python 代码，首先引入需要的包。

``` python
import logging
import sys

import numpy as np
import tvm
from tvm import te
import tvm.testing

# the module is called `autotvm`
from tvm import autotvm
```

## 使用 TE 的基本矩阵乘法

回忆使用 TE 实现的基础版本的矩阵乘法，下面是稍作改动的版本。我们把乘法封装在一个 Python 函数中。为了简化，我们使用固定的块大小，把注意力集中在分割优化上。

``` python
def matmul_basic(N, L, M, dtype):

    A = te.placeholder((N, L), name="A", dtype=dtype)
    B = te.placeholder((L, M), name="B", dtype=dtype)

    k = te.reduce_axis((0, L), name="k")
    C = te.compute((N, M), lambda i, j: te.sum(A[i, k] * B[k, j], axis=k), name="C")
    s = te.create_schedule(C.op)

    # schedule
    y, x = s[C].op.axis
    k = s[C].op.reduce_axis[0]

    yo, yi = s[C].split(y, 8)
    xo, xi = s[C].split(x, 8)

    s[C].reorder(yo, xo, k, yi, xi)

    return s, [A, B, C]
```

## 使用 AutoTVM 进行矩阵乘法

在上面的调度代码中，我们使用了常量 8 作为贴片因子。然而，贴片因子的最佳数值取决于真实的硬件环境和输入形状。

如果你想让调度代码兼容更大的输入形状和目标硬件，最好定义一组候选值，然后根据在目标硬件上的测量结果进行选择。

在 AutoTVM 中，我们可以定义一个可调节的参数，称为一个旋钮（ knob ）。

## 基本矩阵乘法模板

我们从创建一个使用旋钮的分割调度操作开始。

``` python
# Matmul V1: List candidate values
@autotvm.template("tutorial/matmul_v1")  # 1. use a decorator
def matmul_v1(N, L, M, dtype):
    A = te.placeholder((N, L), name="A", dtype=dtype)
    B = te.placeholder((L, M), name="B", dtype=dtype)

    k = te.reduce_axis((0, L), name="k")
    C = te.compute((N, M), lambda i, j: te.sum(A[i, k] * B[k, j], axis=k), name="C")
    s = te.create_schedule(C.op)

    # schedule
    y, x = s[C].op.axis
    k = s[C].op.reduce_axis[0]

    # 2. get the config object
    cfg = autotvm.get_config()

    # 3. define search space
    cfg.define_knob("tile_y", [1, 2, 4, 8, 16])
    cfg.define_knob("tile_x", [1, 2, 4, 8, 16])

    # 4. schedule according to config
    yo, yi = s[C].split(y, cfg["tile_y"].val)
    xo, xi = s[C].split(x, cfg["tile_x"].val)

    s[C].reorder(yo, xo, k, yi, xi)

    return s, [A, B, C]
```

我们对原先的调度代码做出了四处修改，得到了一个可调节的模板。

<ol>
  <li>
    使用一个装饰器将这个函数声明为一个简单模板。
  </li>
  <li>
    获取一个配置对象 <code>cfg</code> 。你可以把 <code>cfg</code> 看成是这个函数的一个参数，只是不是从参数列表获取到的。有了这个参数，这个函数就不再是一个确定性的调度。我们可以把不同的配置传入这个函数，从而得到不同的调度。使用类似配置对象的函数就可以被称为模板。
    <br/>
    为了让模板函数更加紧凑，我们可以通过两种方式定义单个函数的搜索空间。
    <ol>
      <li>
        通过值的集合定义搜索空间。令 <code>cfg</code> 是一个 <code>ConfigSpace</code> 即可。它会收集函数中的所有旋钮，并构建出搜索空间。
      </li>
      <li>
        通过空间中的实体构建调度。令 <code>cfg</code> 是一个 <code>ConfigEntity</code> 即可。当它是一个 <code>ConfigEntity</code> 时，它会忽略所有空间定义 API （也就是说 <code>cfg.define_XXXXX(...)</code> ）。相反地，它会为每一个旋钮保存一个确定的值，并且根据这些值来构建调度。
      </li>
    </ol>
    在自动调优的过程中，我们首先会使用一个 <code>ConfigSpace</code> 对象调用该模板，构造搜索空间。接下来我们会使用不同的 <code>ConfigEntity</code> 对象来获取不同的调度。最终我们测评得到的不同调度，然后选取一个最佳的。
  </li>
  <li>
    定义两个旋钮。第一个参数是 <code>tile_y</code> ，且有 5 个可选值。第二个参数是 <code>tile_x</code> ，同样有 5 个可选值。这两个旋钮是独立的，所以搜索空间的大小一共是 25 。
  </li>
  <li>
    两个旋钮被传递进了 <code>split</code> 调度操作，允许根据这 25 个可选值进行调度。
  </li>
</ol>

## 使用进阶参数 API 的矩阵乘法模板

在前一个模板中，我们手动列出了某个旋钮的所有可选值。这是定义搜索空间的最低级 API ，显式地指出了可以枚举搜索的参数空间。然而，我们还提供了另外的一套 API ，可以让定义搜索空间更为简单和智能。我们建议尽量使用高级的 API 。

在接下来的例子中，我们使用 `ConfigSpace.define_split` 来定一个分割旋钮，尝试所有可能的分割方法。

我们也为重排序提供了 `ConfigSpace.define_reorder` 旋钮，为循环展开、向量化、线程绑定提供了 `ConfigSpace.define_annotate` 旋钮。当这些高级 API 不能满足你的要求时，你可以使用低级的 API 。

``` python
@autotvm.template("tutorial/matmul")
def matmul(N, L, M, dtype):
    A = te.placeholder((N, L), name="A", dtype=dtype)
    B = te.placeholder((L, M), name="B", dtype=dtype)

    k = te.reduce_axis((0, L), name="k")
    C = te.compute((N, M), lambda i, j: te.sum(A[i, k] * B[k, j], axis=k), name="C")
    s = te.create_schedule(C.op)

    # schedule
    y, x = s[C].op.axis
    k = s[C].op.reduce_axis[0]

    ##### define space begin #####
    cfg = autotvm.get_config()
    cfg.define_split("tile_y", y, num_outputs=2)
    cfg.define_split("tile_x", x, num_outputs=2)
    ##### define space end #####

    # schedule according to config
    yo, yi = cfg["tile_y"].apply(s, C, y)
    xo, xi = cfg["tile_x"].apply(s, C, x)

    s[C].reorder(yo, xo, k, yi, xi)

    return s, [A, B, C]
```

关于 `cfg.define_split` 的详细解释。在本模板中， `cfg.define_split("tile_y", y, num_outputs=2)` 会枚举所有可能的组合，把 `y` 轴分割成两个轴。举例来说，如果 `y` 的长度是 32 ，那么可以分割成 `(32,1)` 、 `(16,2)` 、 `(8,4)` 、 `(4,8)` 、 `(2,16)` 、 `(32,1)` 。

在调度过程中， `cfg["tile_y"]` 是一个 `SplitEntity` 对象。我们把外轴（ outer axes ）和内轴（ inner axes ）的长度保存在 `cfg["tile_y"].size` 。在本模板中，我们通过 `yo, yi = cfg['tile_y'].apply(s, C, y)` 来应用这些分割。实际上，它等价于 `yo, yi = s[C].split(y, cfg["tile_y"].size[1])` 或者 `yo, yi = s[C].split(y, nparts=cfg['tile_y"].size[0])` 。

使用 `cfg.apply` API 可以让多级分割（ `num_outputs >= 3` ）更加简单。

## 第二步：使用 AutoTVM 优化矩阵乘法

在第一步中，我们写了一个允许我们调整分割调度中使用的块大小的矩阵乘法模板。我们现在可以在这样一个参数空间中进行搜索。下一步是选择一个调优器对这种空间进行探索。

### TVM 中的自动调优器

调优器的任务可以被以下伪代码表示。

``` plaintext
ct = 0
while ct < max_number_of_trials:
    propose a batch of configs
    measure this batch of configs on real hardware and get results
    ct += batch_size
```

当提出下一批配置时，调优器可以选择不同的策略。 TVM 提供的策略有：

- `tvm.autotvm.tuner.RandomTuner` ：以随机顺序枚举这个空间。
- `tvm.autotvm.tuner.GridSearchTuner` ：以网格顺序枚举这个空间。
- `tvm.autotvm.tuner.GATuner` ：使用遗传算法搜索空间。
- `tvm.autotvm.tuner.XGBTuner` ：使用机遇模型的方法。训练一个 XGBoost 模型来预测降级后 IR 的运行速度，然后根据预测选择下一批配置。

你可以根据你的空间大小、时间和其他因子来选择调优器。比如说，如果你的空间十分小（比如小于 1000 ），网格搜索调优器就足够了。如果你的空间是 $10^9$ 数量级的（比如 CUDA GPU 上的 `conv2d` 的空间大小），那么 XGBoost 调优器可以更快速地找到更好的配置。

### 开始调优

接下来我们继续矩阵乘法的例子。首先我们创建一个调优任务。我们可以检查初始化的搜索空间。在本例中，一个 $512 \times 512$ 的矩阵乘法中，搜索空间是 $10 \times 10 = 100$ 。注意调优任务和搜索空间与选择的调优器是无关的。

``` python
N, L, M = 512, 512, 512
task = autotvm.task.create("tutorial/matmul", args=(N, L, M, "float32"), target="llvm")
print(task.config_space)
```

输出：

``` plaintext
ConfigSpace (len=100, space_map=
   0 tile_y: Split(policy=factors, product=512, num_outputs=2) len=10
   1 tile_x: Split(policy=factors, product=512, num_outputs=2) len=10
)
```

现在我们需要定义如何测评生成出的代码，还需要选择一个调优器。由于我们的空间很小，这里就选择随机调优器。

在本教程中，我们只做了 10 次测试。在实际使用时，你可以根据你的时间计划做更多的实验。我们把调优结果记录进一个日志文件。该文件还可以被以后其他调优器使用。

``` python
# logging config (for printing tuning log to the screen)
logging.getLogger("autotvm").setLevel(logging.DEBUG)
logging.getLogger("autotvm").addHandler(logging.StreamHandler(sys.stdout))
```

为了测评配置，还有两个任务：构建和运行。默认来说，我们使用所有 CPU 核心来编译程序。接下来我们串行地测试它们。为了减少误差，我们测量 5 次然后取平均。

``` python
measure_option = autotvm.measure_option(builder="local", runner=autotvm.LocalRunner(number=5))

# Begin tuning with RandomTuner, log records to file `matmul.log`
# You can use alternatives like XGBTuner.
tuner = autotvm.tuner.RandomTuner(task)
tuner.tune(
    n_trial=10,
    measure_option=measure_option,
    callbacks=[autotvm.callback.log_to_file("matmul.log")],
)
```

调优完成后，我们可以从日志文件中选择一个表现最好的配置，然后使用对应的参数编译调度。我们还可以做一个简单测试，检查该调度产生了正确结果。我们可以通过 `autotvm.apply_history_best` 来直接调用 `matmul` 函数，它会用当前参数查询上下文，然后得到相同参数的最佳配置。

``` python
# apply history best from log file
with autotvm.apply_history_best("matmul.log"):
    with tvm.target.Target("llvm"):
        s, arg_bufs = matmul(N, L, M, "float32")
        func = tvm.build(s, arg_bufs)

# check correctness
a_np = np.random.uniform(size=(N, L)).astype(np.float32)
b_np = np.random.uniform(size=(L, M)).astype(np.float32)
c_np = a_np.dot(b_np)

c_tvm = tvm.nd.empty(c_np.shape)
func(tvm.nd.array(a_np), tvm.nd.array(b_np), c_tvm)

tvm.testing.assert_allclose(c_np, c_tvm.numpy(), rtol=1e-4)
```

### 结语

本教程中，我们展示了如何构建操作模板，让 TVM 在参数空间中搜索并得出一个最优化的调度配置。为了更深入地理解它的工作原理，我们建议在这个例子的基础上进行扩展，添加其他新的搜索参数。在后面的章节中，我们会演示 AutoScheduler 。这是一种 TVM 优化常见运算的方法，不需要用户提供模板。
