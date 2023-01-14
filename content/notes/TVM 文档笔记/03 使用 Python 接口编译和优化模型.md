---
title: TVM 文档笔记 03 使用 Python 接口编译和优化模型
description: 
date: 2021-09-27
author: zenk
draft: false
categories: TVM
tags: [TVM, 编译]
---

原文位于[此处](https://tvm.apache.org/docs/tutorials/get_started/autotvm_relay_x86.html)。

在上一篇教程中，我们介绍了如何使用 TVM 的命令行接口 TVMC 来编译、运行和调优一个预训练的视觉模型 ResNet-50-v2 。但是 TVM 不仅是一个命令行工具，它是一个为多种语言提供可用 API 的优化框架，为机器学习模型的工作提供了极高的灵活性。

本篇教程中将会进行与 TVMC 篇相同的实验，但是使用 Python API 来完成。在本篇教程中，我们会用 TVM 的 Python API 完成以下任务：

- 为 TVM 运行时编译预训练 ResNet 50 v2 模型。
- 在编译好的模型上输入一个真实的图片，然后解释模型输出，评估模型性能。
- 使用 TVM 在 CPU 上调优模型。
- 使用 TVM 生成的调优数据重新编译模型。
- 在优化后的模型上输入一个真实的图片，然后解释模型输出，评估模型性能。

本节的目的是给出 TVM 能力的概览，并介绍如何通过 Python API 使用这些能力。

TVM 是一个深度学习编译框架，具有大量不同的模块用于深度学习模型和算子。在本教程中，我们将会使用 Python API 加载、编译和优化模型。

首先我们从引入依赖开始，包括用于加载和转换模型的 `onnx` ，用于下载测试数据的工具，处理图像数据的 Python Image Library ，预处理和后处理图像数据的 `numpy` ，以及 TVM Relay 框架和 TVM Graph Executor 。

``` python
import onnx
from tvm.contrib.download import download_testdata
from PIL import Image
import numpy as np
import tvm.relay as relay
import tvm
from tvm.contrib import graph_executor
```

## 下载、加载 ONNX 模型

在本教程中，我们使用 ResNet-50 v2 模型。 ResNet-50 是一个 50 层的卷积神经网络，用于图像分类。该模型已经使用了涵盖 1000 种的超过一百万张图片进行预训练。网络的输入图像大小是 $224 \times 224$ 。如果你想要查看 ResNet-50 的具体结构，我们推荐你下载 [Netron](https://netron.app/) ，一个免费的机器学习模型查看器。

TVM 提供一个库用于下载预训练模型。给定模型的 URL 、文件名称和模型类型之后， TVM 将会把它下载到硬盘上。以 ONNX 模型为例，你可以使用 ONNX 运行时把它加载到内存中。

处理其他格式的模型。 TVM 支持许多种流行的模型格式。支持列表可以在 TVM 文档的 [编译深度学习模型](https://tvm.apache.org/docs/tutorials/index.html#compile-deep-learning-models) 中查看。

``` python
model_url = "".join(
    [
        "https://github.com/onnx/models/raw/",
        "master/vision/classification/resnet/model/",
        "resnet50-v2-7.onnx",
    ]
)

model_path = download_testdata(model_url, "resnet50-v2-7.onnx", module="onnx")
onnx_model = onnx.load(model_path)
```

## 下载、预处理、加载测试图片

每个模型都要要求得到指定的输入张量形状，格式和数据类型。由于这个原因，大多数模型都需要预处理和后处理的过程，以验证输入合法性和解释输出含义。 TVMC 接受 NumPy 的 `.npz` 格式作为输入输出。这是一个受到良好支持的 NumPy 格式，可以将多个数组序列化为一个文件。

本教程的输入是一张猫的图片，你也可以选择其他图片。

![猫](/notes/TVM/images/02-00.jpg#center)

下载测试图片，并将其转换成一个 `numpy` 数组以便后续作为模型的输入。

``` python
img_url = "https://s3.amazonaws.com/model-server/inputs/kitten.jpg"
img_path = download_testdata(img_url, "imagenet_cat.png", module="data")

# Resize it to 224x224
resized_image = Image.open(img_path).resize((224, 224))
img_data = np.asarray(resized_image).astype("float32")

# Our input image is in HWC layout while ONNX expects CHW input, so convert the array
img_data = np.transpose(img_data, (2, 0, 1))

# Normalize according to the ImageNet input specification
imagenet_mean = np.array([0.485, 0.456, 0.406]).reshape((3, 1, 1))
imagenet_stddev = np.array([0.229, 0.224, 0.225]).reshape((3, 1, 1))
norm_img_data = (img_data / 255 - imagenet_mean) / imagenet_stddev

# Add the batch dimension, as we are expecting 4-dimensional input: NCHW.
img_data = np.expand_dims(norm_img_data, axis=0)
```

## 使用 Relay 编译模型

下一步是编译 ResNet 模型。首先我们通过 `from_onnx` 将模型引入 Relay ，然后使用标准优化将模型构建为一个 TVM 库。最终我们根据这个库创建一个 TVM 图运行时模块。

``` python
target = "llvm"
```

定义正确的目标。定义正确的目标让 TVM 可以使用针对特定硬件特性的优化，对于编译后的模型性能有巨大的影响。阅读 [在 x86 CPU 上自动调优卷积网络](https://tvm.apache.org/docs/tutorials/autotvm/tune_relay_x86.html#define-network) 获取更多信息。我们建议标明你正在使用的 CPU 和可选特性，恰当地设定目标。比如说，对于某些处理器可以使用 `target = "llvm -mcpu=skylake"` ，如果还支持 AVX-512 向量指令集那么就可以使用 `target = "llvm -mcpu=skylake-avx512"` 。

``` python
# The input name may vary across model types. You can use a tool
# like netron to check input names
input_name = "data"
shape_dict = {input_name: img_data.shape}

mod, params = relay.frontend.from_onnx(onnx_model, shape_dict)

with tvm.transform.PassContext(opt_level=3):
    lib = relay.build(mod, target=target, params=params)

dev = tvm.device(str(target), 0)
module = graph_executor.GraphModule(lib["default"](dev))
```

## 在 TVM 运行时上执行

现在我们已经编译好了模型，可以使用 TVM 运行时来进行预测了。使用 TVM 运行模型做出预测需要两个条件：

- 编译好的模型，
- 合法的模型输入。

``` python
dtype = "float32"
module.set_input(input_name, img_data)
module.run()
output_shape = (1, 1000)
tvm_output = module.get_output(0, tvm.nd.empty(output_shape)).numpy()
```

## 收集基础性能数据

我们希望收集一些基础性能数据，用于之后比较未优化模型和优化后模型的性能。为了排除 CPU 的噪声，我们多次多批地进行计算过程，然后收集平均值、中位数和标准差等基础统计数据。

``` python
import timeit

timing_number = 10
timing_repeat = 10
unoptimized = (
    np.array(timeit.Timer(lambda: module.run()).repeat(repeat=timing_repeat, number=timing_number))
    * 1000
    / timing_number
)
unoptimized = {
    "mean": np.mean(unoptimized),
    "median": np.median(unoptimized),
    "std": np.std(unoptimized),
}

print(unoptimized)
```

输出：

``` python
{'mean': 226.42689668997264, 'median': 226.24714619996666, 'std': 0.646916386317958}
```

## 输出后处理

之前已经提到过，每个模型都有自己提供输出的方式。

在本教程中，我们需要运行输出后处理的脚本把来自 ResNet 50 V2 的输出借助模型中提供的查询表转换成更具可读性的形式。

``` python
from scipy.special import softmax

# Download a list of labels
labels_url = "https://s3.amazonaws.com/onnx-model-zoo/synset.txt"
labels_path = download_testdata(labels_url, "synset.txt", module="data")

with open(labels_path, "r") as f:
    labels = [l.rstrip() for l in f]

# Open the output and read the output tensor
scores = softmax(tvm_output)
scores = np.squeeze(scores)
ranks = np.argsort(scores)[::-1]
for rank in ranks[0:5]:
    print("class='%s' with probability=%f" % (labels[rank], scores[rank]))
```

输出：

``` python
# class='n02123045 tabby, tabby cat' with probability=0.610553
# class='n02123159 tiger cat' with probability=0.367179
# class='n02124075 Egyptian cat' with probability=0.019365
# class='n02129604 tiger, Panthera tigris' with probability=0.001273
# class='n04040759 radiator' with probability=0.000261
```

## 模型调优

前面的模型是为了 TVM 运行时而编译的，并没有引入任何平台特定的优化。在本节中，我们会展示如何使用 TVMC 来构建一个针对你的平台优化的模型。

某些时候，使用编译好的模块运行推理时可能无法达到我们的预期性能。在这种情况下，我们可以利用自动调优器寻找一个对于我们的模型更好的配置，以提升性能。使用 TVM 进行调优指的是优化模型使其在特定目标机器上运行得更快。这与训练过程和微调（ fine-tuning ）过程不同， TVM 中的调优不会影响模型的准确性，只会影响运行性能。在调优过程中， TVM 会尝试运行多种不同的算子实现来查看它们的性能。运行结果将会保存在调优记录文件中，该文件是 `tune` 命令最终的输出。

在最简单的形式中，调优需要提供以下条件：

- 需要运行模型的设备的详细规格，
- 调优记录文件的保存路径，
- 需要调优的模型文件路径。

``` python
import tvm.auto_scheduler as auto_scheduler
from tvm.autotvm.tuner import XGBTuner
from tvm import autotvm

# Set up some basic parameters for the runner. The runner takes compiled code
# that is generated with a specific set of parameters and measures the
# performance of it. ``number`` specifies the number of different
# configurations that we will test, while ``repeat`` specifies how many
# measurements we will take of each configuration. ``min_repeat_ms`` is a value
# that specifies how long need to run configuration test. If the number of
# repeats falls under this time, it will be increased. This option is necessary
# for accurate tuning on GPUs, and is not required for CPU tuning. Setting this
# value to 0 disables it. The ``timeout`` places an upper limit on how long to
# run training code for each tested configuration.

number = 10
repeat = 1
min_repeat_ms = 0  # since we're tuning on a CPU, can be set to 0
timeout = 10  # in seconds

# create a TVM runner
runner = autotvm.LocalRunner(
    number=number,
    repeat=repeat,
    timeout=timeout,
    min_repeat_ms=min_repeat_ms,
    enable_cpu_cache_flush=True,
)

# Create a simple structure for holding tuning options. We use an XGBoost
# algorithim for guiding the search. For a production job, you will want to set
# the number of trials to be larger than the value of 10 used here. For CPU we
# recommend 1500, for GPU 3000-4000. The number of trials required can depend
# on the particular model and processor, so it's worth spending some time
# evaluating performance across a range of values to find the best balance
# between tuning time and model optimization. Because running tuning is time
# intensive we set number of trials to 10, but do not recommend a value this
# small. The ``early_stopping`` parameter is the minimum number of trails to
# run before a condition that stops the search early can be applied. The
# measure option indicates where trial code will be built, and where it will be
# run. In this case, we're using the ``LocalRunner`` we just created and a
# ``LocalBuilder``. The ``tuning_records`` option specifies a file to write
# the tuning data to.

tuning_option = {
    "tuner": "xgb",
    "trials": 10,
    "early_stopping": 100,
    "measure_option": autotvm.measure_option(
        builder=autotvm.LocalBuilder(build_func="default"), runner=runner
    ),
    "tuning_records": "resnet-50-v2-autotuning.json",
}
```

定义调优搜索算法。默认的搜索算法是 XGBoost Grid 算法。根据你的模型复杂性和预期的调优时间，你可以选择不同的算法。可以通过 `tvmc tune --help` 查看可选的完成列表。

设定调优参数。在本例中，考虑到运行时间，我们把测试次数（ trails ）和提前停止（ early stopping ） 设定为 10 。把它们设定成更高的数值可以获得更好的性能，然而调优时间将会变长。根据模型和平台的不同，所需的测试次数也会发生变化。

``` python
# begin by extracting the taks from the onnx model
tasks = autotvm.task.extract_from_program(mod["main"], target=target, params=params)

# Tune the extracted tasks sequentially.
for i, task in enumerate(tasks):
    prefix = "[Task %2d/%2d] " % (i + 1, len(tasks))
    tuner_obj = XGBTuner(task, loss_type="rank")
    tuner_obj.tune(
        n_trial=min(tuning_option["trials"], len(task.config_space)),
        early_stopping=tuning_option["early_stopping"],
        measure_option=tuning_option["measure_option"],
        callbacks=[
            autotvm.callback.progress_bar(tuning_option["trials"], prefix=prefix),
            autotvm.callback.log_to_file(tuning_option["tuning_records"]),
        ],
    )
```

可能的输出如下：

``` python
# [Task  1/24]  Current/Best:   10.71/  21.08 GFLOPS | Progress: (60/1000) | 111.77 s Done.
# [Task  1/24]  Current/Best:    9.32/  24.18 GFLOPS | Progress: (192/1000) | 365.02 s Done.
# [Task  2/24]  Current/Best:   22.39/ 177.59 GFLOPS | Progress: (960/1000) | 976.17 s Done.
# [Task  3/24]  Current/Best:   32.03/ 153.34 GFLOPS | Progress: (800/1000) | 776.84 s Done.
# [Task  4/24]  Current/Best:   11.96/ 156.49 GFLOPS | Progress: (960/1000) | 632.26 s Done.
# [Task  5/24]  Current/Best:   23.75/ 130.78 GFLOPS | Progress: (800/1000) | 739.29 s Done.
# [Task  6/24]  Current/Best:   38.29/ 198.31 GFLOPS | Progress: (1000/1000) | 624.51 s Done.
# [Task  7/24]  Current/Best:    4.31/ 210.78 GFLOPS | Progress: (1000/1000) | 701.03 s Done.
# [Task  8/24]  Current/Best:   50.25/ 185.35 GFLOPS | Progress: (972/1000) | 538.55 s Done.
# [Task  9/24]  Current/Best:   50.19/ 194.42 GFLOPS | Progress: (1000/1000) | 487.30 s Done.
# [Task 10/24]  Current/Best:   12.90/ 172.60 GFLOPS | Progress: (972/1000) | 607.32 s Done.
# [Task 11/24]  Current/Best:   62.71/ 203.46 GFLOPS | Progress: (1000/1000) | 581.92 s Done.
# [Task 12/24]  Current/Best:   36.79/ 224.71 GFLOPS | Progress: (1000/1000) | 675.13 s Done.
# [Task 13/24]  Current/Best:    7.76/ 219.72 GFLOPS | Progress: (1000/1000) | 519.06 s Done.
# [Task 14/24]  Current/Best:   12.26/ 202.42 GFLOPS | Progress: (1000/1000) | 514.30 s Done.
# [Task 15/24]  Current/Best:   31.59/ 197.61 GFLOPS | Progress: (1000/1000) | 558.54 s Done.
# [Task 16/24]  Current/Best:   31.63/ 206.08 GFLOPS | Progress: (1000/1000) | 708.36 s Done.
# [Task 17/24]  Current/Best:   41.18/ 204.45 GFLOPS | Progress: (1000/1000) | 736.08 s Done.
# [Task 18/24]  Current/Best:   15.85/ 222.38 GFLOPS | Progress: (980/1000) | 516.73 s Done.
# [Task 19/24]  Current/Best:   15.78/ 203.41 GFLOPS | Progress: (1000/1000) | 587.13 s Done.
# [Task 20/24]  Current/Best:   30.47/ 205.92 GFLOPS | Progress: (980/1000) | 471.00 s Done.
# [Task 21/24]  Current/Best:   46.91/ 227.99 GFLOPS | Progress: (308/1000) | 219.18 s Done.
# [Task 22/24]  Current/Best:   13.33/ 207.66 GFLOPS | Progress: (1000/1000) | 761.74 s Done.
# [Task 23/24]  Current/Best:   53.29/ 192.98 GFLOPS | Progress: (1000/1000) | 799.90 s Done.
# [Task 24/24]  Current/Best:   25.03/ 146.14 GFLOPS | Progress: (1000/1000) | 1112.55 s Done.
```

## 使用调优数据编译模型

以上调优过程的输出，调优记录将会被存储在 `resnet-50-v2-autotuning.json` 中。编译器可以使用这个结果来生成在特定平台上具有更好性能表现的模型代码。

现在针对该模型的调优数据已经收集完成，我们可以使用优化后的算子重新编译模型以提高计算速度。

``` python
with autotvm.apply_history_best(tuning_option["tuning_records"]):
    with tvm.transform.PassContext(opt_level=3, config={}):
        lib = relay.build(mod, target=target, params=params)

dev = tvm.device(str(target), 0)
module = graph_executor.GraphModule(lib["default"](dev))
```

验证一下优化后模型的输出和优化前是一样的：

``` python
dtype = "float32"
module.set_input(input_name, img_data)
module.run()
output_shape = (1, 1000)
tvm_output = module.get_output(0, tvm.nd.empty(output_shape)).numpy()

scores = softmax(tvm_output)
scores = np.squeeze(scores)
ranks = np.argsort(scores)[::-1]
for rank in ranks[0:5]:
    print("class='%s' with probability=%f" % (labels[rank], scores[rank]))

# Verifying that the predictions are the same:
#
# .. code-block:: bash
#
#   # class='n02123045 tabby, tabby cat' with probability=0.610550
#   # class='n02123159 tiger cat' with probability=0.367181
#   # class='n02124075 Egyptian cat' with probability=0.019365
#   # class='n02129604 tiger, Panthera tigris' with probability=0.001273
#   # class='n04040759 radiator' with probability=0.000261
```

## 比较调优前后的模型

我们希望收集一些优化前后的模型表现的基础数据。根据你使用的硬件、迭代的次数等因素的不同，你可能会看到不同程度的性能表现提升。

``` python
import timeit

timing_number = 10
timing_repeat = 10
optimized = (
    np.array(timeit.Timer(lambda: module.run()).repeat(repeat=timing_repeat, number=timing_number))
    * 1000
    / timing_number
)
optimized = {"mean": np.mean(optimized), "median": np.median(optimized), "std": np.std(optimized)}


print("optimized: %s" % (optimized))
print("unoptimized: %s" % (unoptimized))
```

## 结语

在本教程中，我们简短地介绍了如何使用 TVM Python API 来编译、运行和调优模型。我们也讨论了输入预处理和输出后处理的过程。最后，我们展示了优化前后的模型性能表现。

我们使用 ResNet 50 V2 作为一个简单的例子。然而， TVM 支持更多特性，包括交叉编译、远程执行、分析测试等。
