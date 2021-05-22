---
title: OpenFOAM初探（一）
date: 2020-02-14T14:21:00.000Z
author: Xiaoxiao
category: 知识
summary:  运行环境及编译安装
tags:
  - CFD
  - OpenFOAM
  - Linux

mathjax: false
cover: true
---

## 前言

笔者早年的工作主要在win10平台进行，使用过诸如CFX、Fluent等商业软件，后续论文工作主要操练的是Fluent UDF。诚然，Fluent依托UDF在可操作性上已经远超其余商业CFD软件(待考证)，但是其缺陷依然明显：1, 界面操作繁琐，对于一些参数化的研究工作，很难通过脚本实现(GUI依赖版本，TUI懒得学)；2, 新功能的整合程度参差不齐，以笔者常用的DPM为例，功能限制较多，BUG不少...；3, 很难修改顶层求解器，很多中间步骤需要手工；4, 安装包太大，正版太贵。
OpenFOAM是一款流行的开源CFD工具箱，最初萌生体验的想法主要还是觉得自己的代码水平有些上路了(自我感觉)，此外觉得用命令行和脚本操作很有一种Geek的感觉～

## 基本介绍

OpenFOAM主要有两大分支，一是[基金会版(.org)](https://www.openfoam.org)，版本号为OF6,7,8...，似乎是由帝国理工维护，从"血统"上讲更纯；二是[ESI版(.com)](https://www.openfoam.com)，版本号为v1912,v2006,v2012...(前两位数字代表年份，后两位数字代表月份，第一次接触乍一看还以为这版本好几年没更新了)，由ESI Group维护，有那么一点点商业化的感觉。通过使用OpenFOAM 7/8/v2006/v2012这四个版本, 我大致了解了一下这两个分支的区别：

* **手册** [ESI版的手册](https://openfoam.com/documentation/guides/latest/doc/)要远比基金会版的全面，后者基本就是看源码。当然总体来说OpenFOAM是我用过的大型开源软件里文档做的最差的，大概是为了培训收入吧。
* **算例** ESI版自带的Tutorials要比基金会版的多，比如大气边界层的建模，基金会版似乎没有。
* **后处理** ESI版DPM类的求解器输出的lagrangian粒子场能够通过外部安装的Paraview读取，而基金会版的只能通过在ThirdParty中编译的Paraview打开，而这个版本的显示性能较差。同时，如果使用的是WSL1这种没有显示接口的系统，使用Windows下的Paraview做颗粒后处理的话，ESI版是唯一的选择。
* **求解器** 基金会版似乎对一些求解器做了整合，比如将动网格功能合并进了一些原始的求解器，这一点比较友好。求解器功能上两者似乎差异不大。
* **第三方包** ESI版支持更多的第三方内容，比如比较知名的科学计算套件Petsc(很高端，很想学，但没空)、基于一些奇妙方法的多相流求解器OpenQBMM(但是算例好像跑不了...)、优化大规模IO的adios(听着很妙但没钱玩超算)...但是鲁迅说过，我可以不用，但它必须要有。
* **平台支持** 参考基金会官网，OpenFOAM主要面向Linux，同时也一定程度支持Windows 10和macOS。基金会提供了面向Linux的源码包，以及Ubuntu等主流Linux发行版的二进制包。在Win10上安装OpenFOAM(不包括虚拟机)需要安装Linux子系统，即WSL。而ESI版提供基于MSYS的OpenCFD,能在Windows上做到开箱即用(似乎就是个docker?)，针对不同发行版的二进制包也更多。不过平台的支持程度并不是很关键，因为想要好好学习OpenFOAM还是得从编译源码包开始，使用原生Linux无论从性能和兼容性来看都是最佳选择，WSL可以作为Windows系统上的备胎。否则OpenFOAM对比Fluent似乎没有明显优势。

## 环境配置

### Arch/Manjaro

```bash
sudo pacman -S gcc flex cmake git openmpi
```

### Debian/Ubuntu

```bash
sudo apt-get update
sudo apt-get install build-essential flex bison cmake zlib1g-dev libboost-system-dev libboost-thread-dev libopenmpi-dev openmpi-bin gnuplot libreadline-dev libncurses-dev libxt-dev
sudo apt-get install qt4-dev-tools libqt4-dev libqt4-opengl-dev freeglut3-dev libqtwebkit-dev
```
