.. SPDX-License-Identifier: GPL-2.0


瑞萨R-Car V4H的PCIe控制器固件
=============================

Renesas R-Car V4H (r8a779g0) 配备了一个 PCIe 控制器，需要特定的
启动期间固件下载。

然而，瑞萨目前无法免费分发该固件。

固件文件 "104_PCIe_fw_addr_data_ver1.05.txt"（请注意，该文件名可能因**数据手册版本不同**而略有差异）
可能在不同版本的数据手册之间有所不同）可以在
以文本形式编码的数据表，因此必须转换文件内容
返回二进制形式。可以使用以下示例脚本实现：

.. code-block:: sh

$ awk '/^\\s*0x[0-9A-Fa-f]{4}\\s+0x[0-9A-Fa-f]{4}/ { print substr($2,5,2) substr($2,3,2) }' \\\
104_PCIe_fw_addr_data_ver1.05.txt | \\\
xxd -p -r > rcar_gen4_pcie.bin

一旦文本内容被转换为二进制固件文件后，请进行验证
其校验和如下：

.. code-block:: sh

$ sha1sum rcar_gen4_pcie.bin
1d0bd4b189b4eb009f5d564b1f93a79112994945  rcar_gen4_pcie.bin

生成的名为 "rcar_gen4_pcie.bin" 的二进制文件应放置在
在驱动程序运行之前的“/lib/firmware”目录。


==================================================

由 Qwen-plus 及 LT agent 翻译