---
title: SpikingJelly 文档笔记
description: 
date: 2021-09-22
author: zenk
draft: false
categories: SpikingJelly
tags: [SpikingJelly, 神经拟态计算]
---

[原文](https://spikingjelly.readthedocs.io/zh_CN/latest/) 位于此处。

SpikingJelly 是一个基于 PyTorch 的脉冲神经网络深度学习框架。

## 安装

SpikingJelly 是基于 PyTorch 的，因此需要确保环境中已经安装了 PyTorch 。

``` bash
pip3 install torch torchvision torchaudio
```

然后才能安装 SpikingJelly 。 PyPI 上提供稳定版的 SpikingJelly ，当然也可以通过源码安装开发版。

``` bash
pip3 install spikingjelly
```

> 使用 M1 芯片 MacBook 的用户在这一步可能遇到问题，原因是 SpikingJelly 的依赖 scipy 无法成功安装。首先，不要使用 Python 3.9 ，本文写作时 Python 3.9 还不能通过以下方法安装 scipy 。推荐使用 Python 3.8 ，通过以下命令可以安装 scipy 。
>
> ``` bash
> brew install openblas gfortran
> pip3 install pybind11 cython pythran
> export OPENBLAS=/opt/homebrew/opt/openblas/lib/
> pip3 install scipy --no-use-pep517
> ```
>
> 安装成功后使用 `pip3 install spikingjelly` 即可安装 SpikingJelly 。

## 时钟驱动

时间驱动主要位于 `spikingjelly.clock_driven` ，本小结主要介绍时钟驱动的仿真方法、梯度替代法、可微分 SNN 神经元的使用方式。

梯度替代法是一种训练 SNN 的方法，详见如下综述：

> Neftci E, Mostafa H, Zenke F, et al. Surrogate Gradient Learning in Spiking Neural Networks: Bringing the Power of Gradient-based optimization to spiking neural networks[J]. IEEE Signal Processing Magazine, 2019, 36(6): 51-63.

### SNN

SNN 的输入是电压增量，隐藏状态是膜电压，输出是脉冲。这样的 SNN 神经元是具有马尔可夫性的：当前时刻的输出只与当前时刻的输入、神经元自身的状态有关。

可以用充电、放电、重置这三个离散方程描述任意的离散脉冲神经元：

$$
\begin{aligned}
H(t) & = f(V(t-1), X(t)) \\\\
S(t) & = g(H(t) - V_{threshold}) = \Theta(H(t) - V_{threshold}) \\\\
V(t) & = H(t) \cdot (1 - S(t)) + V_{reset} \cdot S(t)
\end{aligned}
$$

其中 $V(t)$ 是神经元的膜电压， $X(t)$ 是外部输入， $H(t)$ 是神经元的隐藏状态，是还没有发放脉冲前的瞬时状态。 $f(V(t-1),X(t))$ 是神经元的状态更新方程，不同的神经元的区别就是更新方程不同。

例如 LIF 神经元的微分方程是：

$$
\tau_m \frac{dV(t)}{dt} = -(V(t) - V_{reset}) + X(t)
$$

其差分形式是：

$$
\tau_m (V(t) - V(t-1)) = -(V(t-1) - V_{reset}) + X(t)
$$

因此充电方程为：

$$
f(V(t-1), X(t)) = V(t-1) + \frac{1}{\tau_m} (-(V(t-1)-V_{reset}) - X(t))
$$

放电过程中的 $S(t)$ 是神经元发放的脉冲， $g(x) = \Theta(x)$ 是阶跃函数，在 SNN 中习惯称为脉冲函数。

$$
\Theta(x) = \begin{cases}
1 & x \geqslant 0 \\\\
0 & x < 0
\end{cases}
$$

重置表示电压重置过程，发放脉冲则电压重置为 $V_{reset}$ ；没有发放脉冲则电压不变。

### 梯度替代法

SNN 的脉冲函数 $g(x) = \Theta(x)$ 是不可微分的，所以不能使用梯度下降、反向传播的方法来训练。但是可以用一个与 $g(x) = \Theta(x)$ 非常相似但是可以微分的函数 $\sigma(x)$ 来替代它。

梯度替代法的核心思想是：在前向传播时，使用 $g(x) = \Theta(x)$ ，神经元的输出仍然是 SNN 。反向传播时，使用梯度替代函数的梯度 $g'(x) = \sigma'(x)$ 来代替脉冲函数的梯度。最常见的梯度替代函数是 sigmoid 函数 $\sigma(\alpha x) = 1 / (1 + \exp(-\alpha x))$ ， $\alpha$ 可以控制函数的平滑程度。越大的 $\alpha$ 会越逼近 $\Theta(x)$ ，但是容易在 $x = 0$ 时梯度爆炸， $x \ne 0$ 时梯度消失，导致网络难以训练。下图展示了不同的 $\alpha$ 对应的替代函数的图像。

![梯度替代](/notes/SpikingJelly%20文档笔记/images/00.png#center)

### 将脉冲神经元嵌入到深度网络

`clock_driven.neuron` 中已经实现了一些经典神经元，可以方便地搭建各种网络，例如一个简单的全连接网络：

``` python
net = nn.Sequential(
        nn.Linear(100, 10, bias=False),
        neuron.LIFNode(tau=100.0, v_threshold=1.0, v_reset=5.0))
```

## 事件驱动

事件驱动主要位于 `spikingjelly.event_driven` ，本小结主要介绍事件驱动和 Tempotron 神经元。

`clock_driven` 使用时间驱动的方法对 SNN 进行仿真，因此在代码存在时间上的循环，例如：

``` python
for t in range(T):
    if t == 0:
        out_spikes_counter = net(encoder(img).float())
    else:
        out_spikes_counter += net(encoder(img).float())
```

而在事件驱动的 SNN 仿真中，并不需要在时间上进行循环，神经元的状态更新由事件触发。不同的神经元的活动可以一步计算，不需要在时钟上保持同步。

在脉冲响应模型（ Spiking Response Model ， SRM ）中，使用显式的 $V - t$ 方程来描述神经元的活动，而不是使用微分方程来描述神经元的充电过程。由于 $V - t$ 是已知的，因此给定任何输入 $X(t)$ ，神经元的响应 $V(t)$ 都可以被直接算出。

### Tempotron 神经元

> Gütig R, Sompolinsky H. The tempotron: a neuron that learns spike timing–based decisions[J]. Nature neuroscience, 2006, 9(3): 420-428.

上文中提出了一种 SNN 神经元， Tempotron 神经元。这个名称来自于 ANN 中的感知器（ Perceptron ）。感知器是最简单的 ANN 神经元，对输入数据进行加权求和，输出二值 0 或 1 来表示数据的分类结果。

Tempotron 的膜电位定义是：

$$
V(t) = \sum_i w_i \sum_{t_i} K(t - t_i) + V_{reset}
$$

$w_i$ 是第 $i$ 个输入的权重，或者说突触的权重。 $t_i$ 是第 $i$ 个输入脉冲发放的时刻， $K(t-t_i)$ 是输入脉冲所引发的突触后膜电位（ PostSynaptic Potentials ， PSPs）。 $V_{reset}$ 是 Tempotron 的重置电位，或者说静息电位。

$K(t-t_i)$ 是一个关于 $t_i$ 的函数（ PSP Kernel ），上文中使用的函数如下：

$$
K(t-t_i) = \begin{cases}
V_0 (\exp(-\frac{t-t_i}{\tau}) - \exp(-\frac{t-t_i}{\tau_s})) & t \geqslant t_i \\\\
0 & t < t_i
\end{cases}
$$

![1](/notes/SpikingJelly%20文档笔记/images/01.png#center)
![2](/notes/SpikingJelly%20文档笔记/images/02.png#center)
![3](/notes/SpikingJelly%20文档笔记/images/03.png#center)

上图是不同参数下 $K(t-t_i)$ 函数的形状。 $V_0$ 是归一化系数，使得函数的最大值为 1 。 $\tau$ 是膜电位时间常数，输入的脉冲会使 Tempotron 的电位瞬时激增，但之后会指数衰减。 $\tau_s$ 是突触电流的时间常数，它表示突触上传导的电流也会随着时间衰减。

## 时间驱动：神经元

这一部分介绍了泄漏集火模型的基本知识，我就不重复了。

## 时间驱动：编码器

这一部分介绍了柏松编码器、周期编码器等编码模块。

## 时间驱动：使用单层全连接SNN识别MNIST

使用惊蛰看起来是非常简单的，只需要将激活函数替换为神经元即可。

``` python
net = nn.Sequential(
    nn.Flatten(),
    nn.Linear(28 * 28, 10, bias=False),
    nn.Softmax()
    )
```

``` python
net = nn.Sequential(
    nn.Flatten(),
    nn.Linear(28 * 28, 10, bias=False),
    neuron.LIFNode(tau=tau)
    )
```

> 不再继续调研它了，所以文档到此为止。
