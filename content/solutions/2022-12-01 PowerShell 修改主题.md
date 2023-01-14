---
title: PowerShell 修改主题
description: 
date: 2022-12-01
author: zenk
draft: false
categories: 解决方案
tags: [PowerShell]
---

本文档描述了按照[官方文档](https://ohmyposh.dev/docs/installation/windows)安装Oh My Posh，并修改PowerShell的主题的步骤。

## 安装Oh My Posh并修改主题

打开PowerShell，输入以下命令。该命令会获取并执行安装Oh My Posh的脚本。

``` powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://ohmyposh.dev/install.ps1'))
```

继续在PowerShell中输入以下命令。该命令会打开PowerShell配置文件。

``` powershell
notepad $PROFILE
```

如果出现文件不存在等错误，那么说明配置文件不存在。输入以下命令来创建配置文件。

``` powershell
New-Item -Path $PROFILE -Type File -Force
```

在打开的配置文件中添加以下内容。这些内容会在启动Powershell时告知Oh My Posh应该使用何种主题。

``` powershell
oh-my-posh init pwsh --config 'C:/Users/Posh/paradox.omp.json' | Invoke-Expression
```

上述命令中的`paradox`可以替换成其他主题，[这里](https://ohmyposh.dev/docs/themes)展示了许多可选的主题。

为了使某些字符正确显示，需要安装[Nerd Font](https://github.com/ryanoasis/nerd-fonts)。Nerd Font是一类字体的统称，Oh My Posh推荐Meslo LGM NF。

打开使用PowerShell的应用，例如Windows Terminal、Visual Studio Code等，设置显示字体为安装好的Nerd Font。

## 升级Oh My Posh

打开PowerShell，输入以下命令。该命令会获取并执行升级Oh My Posh的脚本。

``` powershell
Set-ExecutionPolicy Bypass -Scope Process -Force; Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://ohmyposh.dev/install.ps1'))
```
