.. SPDX-License-Identifier: GPL-2.0-only

.. include:: <isonum.txt>


AMD NPU
=======

:版权: |copy| 2024 Advanced Micro Devices, Inc.
:Author: Sonal Santan <sonal.santan@amd.com>

概述
====

AMD NPU（神经处理单元）是一种多用户AI推理加速器
集成到 AMD 客户端 APU 中。NPU 可实现机器学习的高效执行
学习类应用如CNN、LLM等。NPU基于
`AMD XDNA 架构`_. NPU 由 **amdxdna** 驱动程序管理。


硬件描述
========

AMD NPU 由以下硬件组件组成：

AMD XDNA 数组
-------------

AMD XDNA 阵列由计算和内存单元组成的二维阵列构成，采用
`AMD AI Engine Technology`_。每列包含4行计算单元和1
内存单元的一行。每个计算单元包含一个具有独立
专用的程序和数据存储器。存储器单元充当L2存储器。2D
数组可以在列边界处进行划分，从而创建空间上隔离的
可绑定到工作负载上下文的分区。

每列还配备了专用的DMA引擎，用于在主机DDR和
内存单元。

AMD Phoenix 和 AMD Hawk Point 客户端 NPU 采用 4x5 拓扑结构，即 4 行
计算单元排列成5列。AMD Strix Point客户端APU具有4x8
拓扑结构，即4行计算单元排列成8列。

共享L2内存
----------

单行内存单元构成一个由软件管理的片上L2内存池
内存。DMA引擎用于在主机DDR和内存单元之间传输数据。
AMD Phoenix 和 AMD Hawk Point NPU 拥有总计 2560 KB 的 L2 内存。
AMD Strix Point NPU 总共拥有 4096 KB 的 L2 内存。

微控制器
--------

微控制器运行负责命令处理的NPU固件，
XDNA阵列分区设置，XDNA阵列配置，工作负载上下文
管理和工作负载编排。

NPU 固件使用隔离的非特权上下文的专用实例
调用ERT来为每个工作负载上下文提供服务。ERT还用于执行用户
提供与工作负载上下文关联的 ``ctrlcode``。

NPU 固件使用一个称为 MERT 的独立特权上下文来提供服务
来自 amdxdna 驱动程序的管理命令。

邮箱
----

微控制器和amdxdna驱动程序使用特权通道进行管理
诸如设置上下文、遥测、查询、错误处理以及配置等任务
用户通道等。如前所述，特权通道请求是
由 MERT 提供服务。特权通道与单个邮箱绑定。

微控制器和amdxdna驱动程序为每个专用用户通道使用
工作负载上下文。用户通道主要用于提交工作
NPU。如前所述，用户通道的请求由以下方式提供服务：
ERT 的实例。每个用户通道都绑定到其专用的邮箱。

PCIe EP
-------

NPU 对 x86 主机 CPU 而言是一个具有多个 BAR 和一些其他功能的 PCIe 设备
MSI-X 中断向量。NPU 使用专用的高带宽 SoC 级互连结构
用于读取或写入主机内存。每个 ERT 实例都有其独立的
专用 MSI-X 中断，而多个 ERT 实例各自独占该中断资源；MERT 仅共享一个 MSI-X 中断实例。

PCIe BAR 的数量因具体设备而异。根据它们的
函数，PCIe BAR 通常可以分为以下几类。

* PSP BAR：启用 AMD PSP（平台安全处理器）功能
* SMU BAR：启用 AMD SMU（系统管理单元）功能
* SRAM BAR：为邮箱暴露环形缓冲区
* 邮箱BAR：暴露邮箱控制寄存器（头、尾和ISR）
寄存器等）
* 公共寄存器 BAR：公开公共寄存器

在特定设备上，上述 BAR 类型可能会被组合成一个
单个物理PCIe BAR。或者一个模块可能需要两个物理PCIe BAR来
功能完全正常。例如，

* 在 AMD Phoenix 设备上，PSP、SMU 和公共寄存器 BAR 位于 PCIe BAR 索引 0 上。
* 在 AMD Strix Point 设备上，邮箱和公共寄存器 BAR 位于 PCIe BAR 上
索引 0。PSP 在 PCIe BAR 索引 0（公共寄存器 BAR）中有一些寄存器
以及PCIe BAR索引4（PSP BAR）。

进程隔离硬件
------------

如前所述，XDNA阵列可以被动态划分为隔离的
空间分区，每个分区可能包含一个或多个列。该空间
分区通过编程列隔离寄存器来设置
微控制器。每个空间分区都关联一个PASID，该PASID
同样由微控制器编程。因此，在空间上存在多个分区
NPU 可以通过 PASID 保护并发的主机访问。

NPU 固件本身使用微控制器 MMU 强制隔离的上下文来
处理用户和特权频道的请求。


混合时空调度
============

AMD XDNA 架构支持混合的空间并行和时间并行（时间共享）
2D数组的调度。这意味着可以设置空间分区。
根据动态拆分以适应各种工作负载。一种*空间*分区
可能*独占地*绑定到一个工作负载上下文，而另一个分区可能
被*临时地*绑定到多个工作负载上下文。微控制器
更新临时共享分区的 PASID，以匹配上下文
在任何时候都已绑定到分区。

资源求解器
----------

amdxdna 驱动程序的资源求解器组件负责管理资源分配
在各种工作负载中二维数组的数量
运行NPU二进制文件在其元数据中所需的列数。资源求解器
组件使用工作负载传递的提示及其自身的启发式方法来
确定二维数组的（重新）分区策略以及空间和工作负载的映射
列的时分共享。FW 强制执行上下文到列资源的分配
Resource Solver 做出的具有约束力的决定。

AMD Phoenix 和 AMD Hawk Point 客户端 NPU 可支持 6 个并发工作负载
contexts。AMD Strix Point 最多可支持 16 个并发工作负载上下文。


应用程序二进制文件
==================

NPU 应用工作负载由两个独立的二进制文件组成，它们是
由NPU编译器生成。

1. AMD XDNA 阵列覆盖层，用于配置 NPU 空间分区。
叠加层包含设置流切换的说明
计算单元的配置和ELF文件。该覆盖层被加载到
由关联的ERT实例绑定到工作负载的空间分区。
请参阅
`Versal 自适应 SoC AIE-ML 架构手册 (AM020)`_ 了解更多详细信息。

2. ``ctrlcode``，用于协调加载到空间中的叠加层
分区。``ctrlcode`` 由以保护模式运行的 ERT 执行
工作负载上下文中的微控制器。``ctrlcode`` 由以下部分组成
由名为 ``XAie_TxnOpcode`` 的一系列操作码组成。请参考
`AI Engine Run Time`_ 以获取更多详细信息。


特殊主机缓冲区
==============

上下文相关指令缓冲区
--------------------

每个工作负载上下文使用一个驻留在主机上的 64 MB 缓冲区，该缓冲区为内存
映射到为处理工作负载而创建的ERT实例中。``ctrlcode``
工作负载所使用的数据会被复制到这种特殊内存中。该缓冲区
受PASID保护，与该工作负载使用的其他所有输入/输出缓冲区一样。
指令缓冲区也被映射到工作负载的用户空间中。

全局特权缓冲区
--------------

此外，驱动程序还为维护任务分配了一个单独的缓冲区
例如记录来自 MERT 的错误信息。此全局缓冲区使用全局 IOMMU
域，且仅可由 MERT 访问。


高级使用流程
============

以下是 在 AMD NPU 上运行工作负载的步骤：

1.  将工作负载编译成一个overlay和一个``ctrlcode``二进制文件。
2.  用户空间在驱动程序中打开一个上下文并提供叠加层。
3.  驱动程序向资源求解器查询以配置一组列
对于工作负载。
4.  随后，驱动程序要求 MERT 在设备上创建一个具有所需上下文的环境
列。
5.  MERT 随后创建一个 ERT 实例。MERT 还会映射指令缓冲区
到ERT内存中。
6.  接着，用户空间将 ``ctrlcode`` 复制到指令缓冲区。
7.  用户空间随后创建一个包含指向输入、输出以及的命令缓冲区，
指令缓冲区；然后将其与驱动程序一起提交命令缓冲区并继续执行
等待完成时进入睡眠状态。
8.  驱动程序通过邮箱将命令发送给 ERT。
9.  ERT *执行* 指令缓冲区中的 ``ctrlcode``。
10. 执行 ``ctrlcode`` 会启动与主机 DDR 之间的 DMA 传输，
AMD XDNA 阵列正在运行。
11. 当ERT到达“ctrlcode”末尾时，会触发MSI-X以发送完成信号
向驾驶员发出信号，从而唤醒等待的工作负载。


启动流程
========

amdxdna 驱动程序使用 PSP 安全地加载已签名的 NPU 固件并启动引导过程
NPU微控制器的。随后，amdxdna驱动程序等待存活信号
BAR 0 上的一个特殊位置。NPU 在 SoC 挂起期间会关闭
从休眠恢复后开启，此时NPU固件被重新加载，并且握手过程重新进行
再次执行。


用户空间组件
============

编译器
------

Peano 是一个基于 LLVM 的开源单核编译器，用于 AMD XDNA 阵列
计算单元。Peano 可在以下地址获取：
https://github.com/Xilinx/llvm-aie

IRON 是一个基于 AMD XDNA 架构 NPU 的开源阵列编译器，其使用
Peano underneath。IRON 可在以下位置获取：
https://github.com/Xilinx/mlir-aie

用户模式驱动程序 (UMD)
----------------------

开源的XRT运行时栈与amdxdna内核驱动程序接口。XRT
可以在以下位置找到：
https://github.com/Xilinx/XRT

开源的NPU XRT shim可以在以下位置找到：
https://github.com/amd/xdna-driver


DMA操作
=======

DMA操作指令编码在``ctrlcode``中，如下所示：
``XAIE_IO_BLOCKWRITE`` 操作码。当 ERT 执行 ``XAIE_IO_BLOCKWRITE`` 时，DMA
主机DDR与L2内存之间的操作得以实现。


错误处理
========

当 MERT 在 AMD XDNA 阵列中检测到错误时，它会暂停该阵列的执行
工作负载上下文并通过异步消息发送给驱动程序
特权通道。然后驱动程序发送一个缓冲区指针给 MERT 以进行捕获
故障工作负载上下文所绑定分区的寄存器状态。
驱动程序随后通过读取缓冲区指针的内容来解码错误。


遥测
====

MERT 可以报告各种类型的遥测信息，例如以下内容：

* L1 中断计数器
* DMA 计数器
* 深度睡眠计数器
* 等等。


参考文献
========

- `AMD XDNA 架构 <https://www.amd.com/en/technologies/xdna.html>`_
- `AMD AI Engine 技术 <https://www.xilinx.com/products/technology/ai-engine.html>`_
- `Peano <https://github.com/Xilinx/llvm-aie>`_
- `Versal 自适应 SoC AIE-ML 架构手册 (AM020) <https://docs.amd.com/r/en-US/am020-versal-aie-ml>`_
- `AI Engine Run Time <https://github.com/Xilinx/aie-rt/tree/release/main_aig>`_


==================================================

由 Qwen-plus 及 LT agent 翻译