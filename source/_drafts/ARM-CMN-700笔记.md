---
title: ARM CMN-700笔记
date: 2023-08-14 16:13:50
categories: 学习笔记
tags: ARM
---

> 参考文档：
>
> - [Arm Neoverse CMN‐700 Coherent Mesh Network](https://developer.arm.com/documentation/102308/latest/)，本文基于Revision r3p2。

按我理解，CMN（Coherent Mesh Network）是一种片上网络系统，把芯片中的各种组件连接起来，并提供缓存一致性等问题的解决方案。

> The CMN‐700 product is a scalable configurable coherent interconnect that is designed to meet the Power, Performance, and Area (PPA) requirements for Coherent Mesh Network systems that are used in high-end networking and enterprise compute applications.

## 组件

### XP（Crosspoint）

XP有时也称为MXP（Mesh Crosspoint），是一种路由逻辑模块。XP模块以二维矩阵的拓扑结构排列，构成了CMN-700。每个XP可以通过网络端口（mesh port）最多连接上下左右四个相邻的XP，还可以通过设备端口（device port）连接两个设备P0和P1。

``` ascii
     |   P1
     | /
---- XP ----
   / |
P0   |
```

使用了全部4个网络端口的XP可以使用最多2个设备端口；位于CMN边缘的XP可能只使用了3个或2个网络端口，通过配置，它们可以使用最多3个或4个设备端口。

``` ascii
P3   |   P1
   \ | /
---- XP ----
   / | \
P0   |   P2
```

每个XP支持四个CHI通道用于传输从源设备到目标设备的`flit`：REQ（Request）、RSP（Response）、SNP（Snoop）、DAT（Data）。

CMN-700支持最大12x12共144个XP的网络，每个XP可以用`(X,Y)`坐标来引用。左下角是`(0,0)`，水平为X轴，竖直为Y轴。

``` ascii
 [Y]

 ...      ...
  |        |
(0,1) -- (1,1) -- ...
  |        |
(0,0) -- (1,0) -- ...  [X]
```

### RN-I（I/O-coherent Request Node）

RN-I将I/O-coherent AMBA管理器连接到CMN-700。每个RN-I包含三个ACE-Lite或者ACE-Lite-with-DVM子端口。RN-I只能作为没有硬件一致性缓存的管理器的代理，它不能向RN-I发送监听事务（snoop transaction）。

### HN-F（Fully-coherent Home Node）

HN-F负责管理部分地址空间。

HN-F由以下部分组成：

- SLC（System Level Cache），最后一级cache。
- Combined PoS/PoC（Point-of-Serialization/Point-of-Coherency），重排发送到HN-F的内存请求。
- SF（Snoop Filter），跟踪RN-F中的cache line。

### HN-I（I/O-coherent Home Node）

HN-I是为所有针对AMBA下属设备的CHI事务服务的节点。HN-I是所有RN的代理，将CHI事务转换为ACE5-Lite事务。HN-I不支持缓存下游ACE5-Lite I/O子系统的数据。

### HN-P（I/O-coherent Home Node with PCIe optimization）

HN-P只能用于连接PCIe子设备。HN-P中包含HN-I功能和PCIe peer-to-peer流量的专用跟踪器。

### SBSX（AMBA 5 CHI to ACE5-Lite bridge）

SBSX具有CHI的SN-F功能，使CMN-700可以使用ACE5-Lite子设备。

### CML（Coherent Multichip Link）

CML提供了多芯片通信的功能。CCG（CML device）可以是CML_SMP连接或者CXL设备。

### CFG（Configuration Node）

CFG与HN-D节点临近，实现了CMN-700的配置、控制和监视特性。CFG没有专用的CHI端口，它和HN-D共用一个端口。

### PCCB（Power/Clock Control Block）

PCCB也和HN-D放在一起，提供独立的通信通道，用于片上网络和SoC之间的电源和时钟管理。PCCB也和HN-D共用一个端口，没有专用的CHI端口。

### SAM（System Address Map）

所有CHI命令都必须含有完全解析后的片上网络地址，包含源ID和目标ID。目标ID是通过SAM得到的，SAM可以将内存或者I/O地址映射到目标设备。

每个请求设备都需要SAM功能。

### DTC（Debug and Trace Controller）

DTC控制分布式DTM（Debug and Trace Monitors），并使用ATB接口生成时间戳。

### QoS（Quality-of-Service）

CMN-700支持端到端的QoS。QoS支持使用RN请求包的QoS字段来影响包在每个QoS决策点的仲裁优先级。

### CAL（Component Aggregation Layer）

CAL可以让多个设备连接到XP上的同一个设备端口。

### Credited Slices

CMN-700中可以使用多种可选的creditd register slices，这些credited slices可以帮助完成时序闭包，但会提高通信延迟。
