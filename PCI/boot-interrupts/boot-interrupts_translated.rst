.. SPDX-License-Identifier: GPL-2.0


启动中断
========

:作者: - Sean V Kelley <sean.v.kelley@linux.intel.com>

概述
====

在PCI Express上，中断通过MSI或入站中断消息（Assert_INTx/Deassert_INTx）表示
中断消息（Assert_INTx/Deassert_INTx）。集成在其中的 IO-APIC 在
给定核心IO将来自PCI Express的旧版中断消息转换为
MSI 中断。如果 IO-APIC 被禁用（通过 IO-APIC 表项中的掩码位）
IO-APIC 表项），消息被路由到传统的 PCH。这
带内中断机制传统上是那些系统的必要组成部分
不支持IO-APIC和启动。英特尔过去曾使用过
使用术语“启动中断”来描述此机制。此外，PCI Express
协议描述了这种带内传统的线中断 INTx 机制，用于
I/O设备用于发出PCI风格的电平中断信号。后续段落
描述核心IO处理中将INTx消息路由到的问题
PCH以及在BIOS和操作系统中的缓解措施。


问题
====

当带内传统 INTx 消息被转发到 PCH 时，它们会随之
触发一个新的中断，而操作系统很可能没有相应的处理程序。当一个
如果中断长时间未被处理，它们将被 Linux 内核跟踪
虚假中断。该IRQ将被Linux内核禁用
达到特定计数时出现错误“nobody cared”。此IRQ被禁用
现在阻止了现有中断的合法使用，而该中断可能恰好共享
中断请求线::

irq 19：无人处理（尝试使用 "irqpoll" 选项启动）
CPU: 0 PID: 2988 Comm: irq/34-nipalk 已污染: 4.14.87-rt49-02410-g4a640ec-dirty #1
硬件名称：National Instruments NI PXIe-8880/NI PXIe-8880，BIOS 2.1.5f1 2020/01/09
调用跟踪：

<IRQ>
? dump_stack+0x46/0x5e
? __report_bad_irq+0x2e/0xb0
? note_interrupt+0x242/0x290
? nNIKAL100_memoryRead16+0x8/0x10 [nikal]
? handle_irq_event_percpu+0x55/0x70
? handle_irq_event+0x4f/0x80
? handle_fasteoi_irq+0x81/0x180
? handle_irq+0x1c/0x30
? do_IRQ+0x41/0xd0
? common_interrupt+0x84/0x84
</IRQ>

handlers:
irq_default_primary_handler 线程化 usb_hcd_irq
禁用 IRQ #19


条件
====

使用线程化中断是最有可能触发该条件的情况
今天这个问题。IRQ之后可能无法重新启用线程化中断
处理程序被唤醒。这些“一次性触发”条件意味着线程化中断
需要保持中断线屏蔽状态，直到线程化处理程序运行完毕。
特别是处理高数据速率中断时，线程需要
运行至完成；否则某些处理器最终会导致堆栈溢出
由于发送设备的中断仍然处于活动状态。

受影响的芯片组
==============

传统的中断转发机制目前存在于多种系统中
设备包括但不限于来自 AMD/ATI、Broadcom 的芯片组，以及
Intel。通过以下缓解措施所做的更改已应用到
drivers/pci/quirks.c

从ICX开始，核心I/O中不再有任何IO-APIC。
设备。IO-APIC 仅位于 PCH 中。连接到 Core IO 的设备
PCIe 根端口将使用原生 MSI/MSI-X 机制。

缓解措施
========

缓解措施以PCI异常处理的形式实现。优先选择的方案是
首先识别并利用一种方法来禁用通往PCH的路由。
在这种情况下，可以通过一个特殊设置来禁用启动中断的生成
已添加。[1]_

Intel® 6300ESB I/O 控制器中心
备用基地址寄存器：
BIE：启动中断使能

	  ==  ===========================
0   启用启动中断。
1   引导中断已禁用。
	  ==  ===========================

基于Intel® Sandy Bridge至Sky Lake的Xeon服务器：
Coherent Interface Protocol中断控制
dis_intx_route2pch/dis_intx_route2ich/dis_intx_route2dmi2:
当此位被置位时，从本地接收到的 INTx 消息
Intel® Quick Data DMA/PCI Express 端口未路由到传统接口
PCH - 它们要么通过集成的 IO-APIC 转换为 MSI
（如果在相应的表项中 IO-APIC 掩码位是清除的）
或导致不采取进一步操作（当掩码位被设置时）

在无法直接禁用路由的情况下，另一种方法
已利用 PCI 中断引脚到 INTx 的映射表进行
重定向中断处理程序至重新路由的中断的目的
默认情况下为线路。因此，在无法进行此 INTx 路由的芯片组上
禁用后，Linux 内核会将有效的中断重新路由到其传统模式
中断。此处理程序的重定向将防止\n
通常会禁用IRQ的虚假中断检测
由于未处理计数过多而导致的行。[2]_

配置选项 X86_REROUTE_FOR_BROKEN_BOOT_IRQS 存在的目的是为了启用（或
禁用中断处理程序重定向到PCH中断
行。该选项可以被 pci=ioapicreroute 覆盖，或者
pci=noioapicreroute. [3]_


更多文档
========

在几份数据手册中都有对传统中断处理的概述
（见下方的6300ESB和6700PXH）。尽管大体相同，但它提供了关于该机制随芯片组演进的深入见解
芯片组对其处理方式的演变。

禁用启动中断的示例
------------------

- Intel® 6300ESB I/O 控制器中心（文档编号 # 300641-004US）
5.7.3 启动中断
https://www.intel.com/content/dam/doc/datasheet/6300esb-io-controller-hub-datasheet.pdf

- Intel® Xeon® 处理器 E5-1600/2400/2600/4600 v3 产品系列
数据手册 - 第2卷：寄存器（文档编号 # 330784-003）
6.6.41 cipintrc 一致性接口协议中断控制
https://www.intel.com/content/dam/www/public/us/en/documents/datasheets/xeon-e5-v3-datasheet-vol-2.pdf

处理器重定向示例
----------------

- Intel® 6700PXH 64位 PCI Hub（文档编号 #302628）
2.15.2 PCI Express 传统 INTx 支持和启动中断
https://www.intel.com/content/dam/doc/datasheet/6700pxh-64-bit-pci-hub-datasheet.pdf


如果你有任何未解答的关于传统PCI中断的问题，请给我发邮件。

此致，

敬礼，
Sean V Kelley
sean.v.kelley@linux.intel.com

.. [1] https://lore.kernel.org/r/12131949181903-git-send-email-sassmann@suse.de/
.. [2] https://lore.kernel.org/r/12131949182094-git-send-email-sassmann@suse.de/
.. [3] https://lore.kernel.org/r/487C8EA7.6020205@suse.de/


==================================================

由 Qwen-plus 及 LT agent 翻译