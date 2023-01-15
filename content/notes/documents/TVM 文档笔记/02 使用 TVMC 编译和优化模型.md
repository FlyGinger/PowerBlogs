---
title: TVM 文档笔记 02 使用 TVMC 编译和优化模型
description: 
date: 2021-09-18
author: zenk
draft: false
categories: TVM
tags: [TVM, 编译]
---

原文位于[此处](https://tvm.apache.org/docs/tutorials/get_started/tvmc_command_line_driver.html)。

本章将会介绍 TVMC ， TVM 命令行驱动。 TVMC 能够通过命令行接口将自动调优（ auto-tuning ）、编译、查看和执行模型等特性暴露出来。

在本章中，我们会用 TVMC 完成以下任务：

- 为 TVM 运行时编译一个预训练的 ResNet-50 v2 模型。
- 将一个真实图片输送到编译后的模型，得到输出和模型表现。
- 使用 TVM 在 CPU 上调优模型。
- 使用 TVM 得到的调优数据重编译，得到优化后模型。
- 将一个真实图片输送到优化后的模型，得到输出并比较模型表现。

本节的目标是给出 TVM 和 TVMC 能力的概览，为理解 TVM 如何工作创造条件。

## 使用 TVMC

TVMC 是一个 Python 应用，是 TVM Python 包的一部分。当你安装完 TVM 的 Python 包之后，你就获得了一个命令行应用 `tvmc` 。该命令的位置与你的平台和安装方法相关。

如果你将 TVM 当做一个 Python 模块设置到了 `$PYTHONPATH` 中，那么你就可以通过可执行 Python 模块 `python3 -m tvm.driver.tvmc` 命令来访问命令行驱动功能。

为了简化叙述，后面的教程中使用 `tvmc <options>` 来代替 `python3 -m tvm.driver.tvmc <options>` ，二者的结果是一样的。

可以通过如下命令访问帮助页面：

``` bash
tvmc --help
```

注意，实际上你需要使用 `python3 -m tvm.driver.tvmc --help` ，后面不再继续提醒 `tvmc` 这个简称的完整命令。

从 `tvmc` 可以访问的 TVM 主要特性由子命令 `compile` 、 `run` 和 `tune` 提供。使用命令 `tvmc <subcommand> --help` 可以获得子命令的帮助页面。在本教程中将会逐个介绍这些命令，但是首先我们需要下载一个预训练模型。

## 获取模型

在本教程中，我们将会使用 ResNet-50 v2 模型。 ResNet-50 是一个卷积神经网络，有 50 层，用于分类图片。我们要使用的模型已经使用 1000 个不同类别的超过一百万张图片进行了预训练。网络的输入图片大小是 $224 \times 224$ 。如果你想要探索 ResNet-50 模型的结构，我们推荐 [Netron](https://netron.app) ，一个免费的机器学习模型查看器。

本教程中使用的模型是 ONNX 格式。

``` bash
wget https://github.com/onnx/models/raw/master/vision/classification/resnet/model/resnet50-v2-7.onnx
```

使用 macOS 的可能需要 `brew install wget` 。

支持的模型格式。 TVMC 支持 Keras 、 ONNX 、 TensorFlow 、 TFLite 和 Torch 创建的模型。可以使用 `-model-format` 选项来显式指明模型的格式。使用 `tvmc compile --help` 可以查看更多信息。

TVM 的 ONNX 支持。 TVM 的 ONNX 支持依赖于系统中的 ONNX Python 库。可以通过 `pip3 install onnx` 来安装 ONNX 库。 M1 芯片的 MacBook 可能无法成功安装。如果你遇到类似的问题，你需要：

``` bash
brew install protobuf
pip3 install onnx==1.8.1 --no-use-pep517
```

## 将 ONNX 模型编译为 TVM 运行时

下载完 ResNet-50 模型后，下一步就是来编译它。我们需要通过 `tvmc compile` 命令来完成这一过程。编译过程的输出是一个 TAR 包，模型将会被编译为目标平台的一个动态库。

``` bash
tvmc compile \
--target "llvm" \
--output resnet50-v2-7-tvm.tar \
resnet50-v2-7.onnx
```

接下来我们看看 `tvmc compile` 生成了什么东西。创建一个新文件夹，然后把编译结果解压到该文件夹中。

``` bash
mkdir model
tar -xvf resnet50-v2-7-tvm.tar -C model
ls model
```

你会看到三个文件：

- `mod.so` 是模型的 C++ 库表示，可以被 TVM 运行时加载。
- `mod.json` 是 TVM Relay 计算图的文本表示。
- `mod.params` 是预训练模型中的参数。

该模块可以被你的应用直接加载，模型可以通过 TVM 运行时 API 运行。

定义正确的编译目标。指定正确的目标（ `--target` ）对于编译后模块的性能有巨大的影响，因为指定了目标之后就可以利用各种硬件特性。 [这里](https://tvm.apache.org/docs/tutorials/autotvm/tune_relay_x86.html#define-network) 有更详细的信息。我们建议确认你要使用的 CPU 以及它的可选特性，并恰当地定义目标。

## 使用 TVMC 运行后编译后模块中的模型

现在我们将模型编译为了一个模块，可以与 TVM 运行配合进行预测。 TVMC 中已经内置了 TVM 运行时，可以运行编译后的 TVM 模型。为了使用 TVMC 来运行模型并做出预测，我们需要做两件事情：

- 编译后的模块，我们刚刚已经生成好了；
- 用于预测的合法模型输入。

每个模型都要要求得到指定的输入张量形状，格式和数据类型。由于这个原因，大多数模型都需要预处理和后处理的过程，以验证输入合法性和解释输出含义。 TVMC 接受 NumPy 的 `.npz` 格式作为输入输出。这是一个受到良好支持的 NumPy 格式，可以将多个数组序列化为一个文件。

本教程的输入是一张猫的图片，你也可以选择其他图片。

![猫](/notes/TVM/images/02-00.jpg#center)

### 输入预处理

对于我们的 ResNet 50 V2 模型，输入格式要求是 ImageNet 。

你需要安装 Python Image Library ，可以通过 `pip3 install --user pillow` 来完成安装。

``` python
#!python ./preprocess.py
from tvm.contrib.download import download_testdata
from PIL import Image
import numpy as np

img_url = "https://s3.amazonaws.com/model-server/inputs/kitten.jpg"
img_path = download_testdata(img_url, "imagenet_cat.png", module="data")

# Resize it to 224x224
resized_image = Image.open(img_path).resize((224, 224))
img_data = np.asarray(resized_image).astype("float32")

# ONNX expects NCHW input, so convert the array
img_data = np.transpose(img_data, (2, 0, 1))

# Normalize according to ImageNet
imagenet_mean = np.array([0.485, 0.456, 0.406])
imagenet_stddev = np.array([0.229, 0.224, 0.225])
norm_img_data = np.zeros(img_data.shape).astype("float32")
for i in range(img_data.shape[0]):
        norm_img_data[i, :, :] = (img_data[i, :, :] / 255 - imagenet_mean[i]) / imagenet_stddev[i]

# Add batch dimension
img_data = np.expand_dims(norm_img_data, axis=0)

# Save to .npz (outputs imagenet_cat.npz)
np.savez("imagenet_cat", data=img_data)
```

### 运行编译后的模型

有了模型和输入数据，我们可以使用 TVMC 来做一个预测：

``` bash
tvmc run \
--inputs imagenet_cat.npz \
--output predictions.npz \
resnet50-v2-7-tvm.tar
```

`.tar` 模型文件包含了一个 C++ 库， Relay 模型的描述，以及模型的参数。 TVMC 中含有一个 TVM 运行时，可以加载模型并对输入做出预测。当运行以上命令时， TVMC 输出一个新的文件 `predictions.npz` ，包含了模型的输出张量，以 NumPy 格式保存。

在本例中，我们编译模型和运行模型是在同一台机器上。某些情况下，我们可能会通过 RPC Tracker 进行远程调用。可以通过 `tvmc run --help` 来获取更多选项。

### 输出后处理

之前已经提到过，每个模型都有自己提供输出的方式。

在本教程中，我们需要运行输出后处理的脚本把来自 ResNet 50 V2 的输出借助模型中提供的查询表转换成更具可读性的形式。

下面的脚本是一个后处理的例子，从编译好的模型中提取出标签。

``` python
#!python ./postprocess.py
import os.path
import numpy as np

from scipy.special import softmax

from tvm.contrib.download import download_testdata

# Download a list of labels
labels_url = "https://s3.amazonaws.com/onnx-model-zoo/synset.txt"
labels_path = download_testdata(labels_url, "synset.txt", module="data")

with open(labels_path, "r") as f:
    labels = [l.rstrip() for l in f]

output_file = "predictions.npz"

# Open the output and read the output tensor
if os.path.exists(output_file):
    with np.load(output_file) as data:
        scores = softmax(data["output_0"])
        scores = np.squeeze(scores)
        ranks = np.argsort(scores)[::-1]

        for rank in ranks[0:5]:
            print("class='%s' with probability=%f" % (labels[rank], scores[rank]))
```

运行脚本并得到输出：

``` python
python postprocess.py

# class='n02123045 tabby, tabby cat' with probability=0.610553
# class='n02123159 tiger cat' with probability=0.367179
# class='n02124075 Egyptian cat' with probability=0.019365
# class='n02129604 tiger, Panthera tigris' with probability=0.001273
# class='n04040759 radiator' with probability=0.000261
```

将猫的图片替换为其他图片，你可以得到 ResNet 模型做出的其他预测。

## 自动调优 ResNet 模型

前面的模型是为了 TVM 运行时而编译的，并没有引入任何平台特定的优化。在本节中，我们会展示如何使用 TVMC 来构建一个针对你的平台优化的模型。

某些时候，使用编译好的模块运行推理时可能无法达到我们的预期性能。在这种情况下，我们可以利用自动调优器寻找一个对于我们的模型更好的配置，以提升性能。使用 TVM 进行调优指的是优化模型使其在特定目标机器上运行得更快。这与训练过程和微调（ fine-tuning ）过程不同， TVM 中的调优不会影响模型的准确性，只会影响运行性能。在调优过程中， TVM 会尝试运行多种不同的算子实现来查看它们的性能。运行结果将会保存在调优记录文件中，该文件是 `tune` 命令最终的输出。

在最简单的形式中，调优需要提供以下条件：

- 需要运行模型的设备的详细规格，
- 调优记录文件的保存路径，
- 需要调优的模型文件路径。

以下命令展示了如何使用 TVMC 调优：

``` bash
tvmc tune \
--target "llvm" \
--output resnet50-v2-7-autotuner_records.json \
resnet50-v2-7.onnx
```

如果使用了更为详细的 `--target` 参数，那么可以得到更好的结果。举例来说，如果你使用某些型号的 Intel i7 处理器，那么可以使用 `--target "llvm -mcpu=skylake" 。在本例中，我们在本地的 CPU 上使用 LLVM 作为编译器进行调优。

TVMC 将会在该模型的参数空间中搜索，尝试对于算子的不同配置，然后选择在你的平台上运行最快的。虽然搜索是基于指定模型和 CPU 的，但是搜索过程仍然可能花费数小时来完成。输出的文件 `resnet50-v2-7-autotuner_records.json` 将在后续的操作中用于编译优化后模型。

定义调优搜索算法。默认的搜索算法是 XGBoost Grid 算法。根据你的模型复杂性和预期的调优时间，你可以选择不同的算法。可以通过 `tvmc tune --help` 查看可选的完成列表。

在一个消费级的 Skylake CPU 上，可能的输出如下：

``` bash
tvmc tune   --target "llvm -mcpu=broadwell"   --output resnet50-v2-7-autotuner_records.json   resnet50-v2-7.onnx
# [Task  1/24]  Current/Best:    9.65/  23.16 GFLOPS | Progress: (60/1000) | 130.74 s Done.
# [Task  1/24]  Current/Best:    3.56/  23.16 GFLOPS | Progress: (192/1000) | 381.32 s Done.
# [Task  2/24]  Current/Best:   13.13/  58.61 GFLOPS | Progress: (960/1000) | 1190.59 s Done.
# [Task  3/24]  Current/Best:   31.93/  59.52 GFLOPS | Progress: (800/1000) | 727.85 s Done.
# [Task  4/24]  Current/Best:   16.42/  57.80 GFLOPS | Progress: (960/1000) | 559.74 s Done.
# [Task  5/24]  Current/Best:   12.42/  57.92 GFLOPS | Progress: (800/1000) | 766.63 s Done.
# [Task  6/24]  Current/Best:   20.66/  59.25 GFLOPS | Progress: (1000/1000) | 673.61 s Done.
# [Task  7/24]  Current/Best:   15.48/  59.60 GFLOPS | Progress: (1000/1000) | 953.04 s Done.
# [Task  8/24]  Current/Best:   31.97/  59.33 GFLOPS | Progress: (972/1000) | 559.57 s Done.
# [Task  9/24]  Current/Best:   34.14/  60.09 GFLOPS | Progress: (1000/1000) | 479.32 s Done.
# [Task 10/24]  Current/Best:   12.53/  58.97 GFLOPS | Progress: (972/1000) | 642.34 s Done.
# [Task 11/24]  Current/Best:   30.94/  58.47 GFLOPS | Progress: (1000/1000) | 648.26 s Done.
# [Task 12/24]  Current/Best:   23.66/  58.63 GFLOPS | Progress: (1000/1000) | 851.59 s Done.
# [Task 13/24]  Current/Best:   25.44/  59.76 GFLOPS | Progress: (1000/1000) | 534.58 s Done.
# [Task 14/24]  Current/Best:   26.83/  58.51 GFLOPS | Progress: (1000/1000) | 491.67 s Done.
# [Task 15/24]  Current/Best:   33.64/  58.55 GFLOPS | Progress: (1000/1000) | 529.85 s Done.
# [Task 16/24]  Current/Best:   14.93/  57.94 GFLOPS | Progress: (1000/1000) | 645.55 s Done.
# [Task 17/24]  Current/Best:   28.70/  58.19 GFLOPS | Progress: (1000/1000) | 756.88 s Done.
# [Task 18/24]  Current/Best:   19.01/  60.43 GFLOPS | Progress: (980/1000) | 514.69 s Done.
# [Task 19/24]  Current/Best:   14.61/  57.30 GFLOPS | Progress: (1000/1000) | 614.44 s Done.
# [Task 20/24]  Current/Best:   10.47/  57.68 GFLOPS | Progress: (980/1000) | 479.80 s Done.
# [Task 21/24]  Current/Best:   34.37/  58.28 GFLOPS | Progress: (308/1000) | 225.37 s Done.
# [Task 22/24]  Current/Best:   15.75/  57.71 GFLOPS | Progress: (1000/1000) | 1024.05 s Done.
# [Task 23/24]  Current/Best:   23.23/  58.92 GFLOPS | Progress: (1000/1000) | 999.34 s Done.
# [Task 24/24]  Current/Best:   17.27/  55.25 GFLOPS | Progress: (1000/1000) | 1428.74 s Done.
```

`tvmc tune` 可能花费很长时间，因此它提供了许多选项用于自定义调优过程，例如重复的数量（通过 `--repeat` 或 `--number` ）、使用的调优算法等。通过 `tvmc tune --help` 查看详细信息。

## 使用调优数据编译优化模型

在以上调优过程之后，我们得到了调优记录 `resnet50-v2-7-autotuner_records.json` 。该文件可以通过两种方式使用：

- 作为以后调优的输入（ `tvmc tune --tunning-records` ）
- 作为编译器的输入

编译器可以使用调优结果生成针对你指定目标机器的高性能代码，通过 `tvmc compile --tuning-records` 可以完成这一过程。查看 `tvmc compile --help` 查看更多信息。

现在调优数据已经收集完成，我们可以使用优化后的算子重新编译模型，提升运行速度：

``` bash
tvmc compile \
--target "llvm" \
--tuning-records resnet50-v2-7-autotuner_records.json  \
--output resnet50-v2-7-tvm_autotuned.tar \
resnet50-v2-7.onnx
```

检查优化后的模型输出了同样的结果：

``` bash
tvmc run \
--inputs imagenet_cat.npz \
--output predictions.npz \
resnet50-v2-7-tvm_autotuned.tar

python postprocess.py
```

验证预测结果相同：

``` bash
# class='n02123045 tabby, tabby cat' with probability=0.610550
# class='n02123159 tiger cat' with probability=0.367181
# class='n02124075 Egyptian cat' with probability=0.019365
# class='n02129604 tiger, Panthera tigris' with probability=0.001273
# class='n04040759 radiator' with probability=0.000261
```

## 比较调优后和调优前模型

TVMC 提供了模型基础性能测试的工具。你可以指定重复的次数，可以让 TVMC 报告模型运行的时间（独立于运行时启动时间）。从这样的比较中，我们可以得知模型性能大概有多少提升。比如，在 Intel i7 测试平台上，可以看到调优后的模型比调优前快了 47% 。

``` bash
tvmc run \
--inputs imagenet_cat.npz \
--output predictions.npz  \
--print-time \
--repeat 100 \
resnet50-v2-7-tvm_autotuned.tar

# Execution time summary:
# mean (ms)   max (ms)    min (ms)    std (ms)
#     92.19     115.73       89.85        3.15

tvmc run \
--inputs imagenet_cat.npz \
--output predictions.npz  \
--print-time \
--repeat 100 \
resnet50-v2-7-tvm.tar

# Execution time summary:
# mean (ms)   max (ms)    min (ms)    std (ms)
#    193.32     219.97      185.04        7.11
```

## 结语

在本教程中，我们介绍了 TVMC ，一个 TVM 的命令行驱动。我们介绍了如何编译、运行、调优模型。我们也讨论了输入预处理和输出后处理。在调优过程之后，我们也展示了如何比较优化与未优化模型的性能。

我们使用 ResNet 50 V2 作为一个简单的例子。然而， TVMC 支持更多特性，包括交叉编译、远程执行、分析测试等。

使用 `tvmc --help` 可以查看更多可使用的选项。

在下一篇教程中，我们会介绍如何使用 Python 接口来执行相同的编译与优化步骤。
