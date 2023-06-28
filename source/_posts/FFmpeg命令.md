---
title: FFmpeg命令
date: 2022-12-01
categories: 解决方案
tags: [FFmpeg]
---

## 查看硬件支持

使用如下命令可以查看硬件支持、编码器、解码器等。

``` bash
ffmpeg -hwaccels
ffmpeg -encoders
ffmpeg -decoders
```

使用如下命令可以查看单个编码器的参数。

``` bash
ffmpeg -h encoder=libx265
```

## 裁切黑边

可以使用`cropdetect`自动检测黑边，用`crop`进行黑边裁切。

`cropdetect`后的数值是灵敏度，如果黑边不是纯黑，可以适当提高数值。

``` bash
ffplay -i xxx.mp4 -an -vf cropdetect=20
ffmpeg -i xxx.mp4 -vf crop=xxx
```
