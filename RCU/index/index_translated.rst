.. SPDX-License-Identifier: GPL-2.0

.. _rcu_handbook:
一份关于 Linux 内核中 RCU 机制的综合性技术文档集合，采用模块化结构组织内容，涵盖从基础概念到高级设计的完整知识体系。
====================================================================================================================
RCU 手册
========

.. toctree::
   :maxdepth: 2

checklist
lockdep
lockdep-splat
rcubarrier
rcu_dereference
whatisRCU
rcu
rculist_nulls
rcuref
“酷刑”（torture），即 RCU 混沌测试框架，用于对 RCU 子系统进行高强度压力测试，验证其在极端条件下的正确性与稳定性。
stallwarn
listRCU
NMI-RCU
上移

设计/内存排序/树形RCU内存排序 —— 阐述树形 RCU 在多核架构下如何保证内存访问顺序，确保跨 CPU 的读写操作满足一致性要求。
设计/快速宽限期/快速宽限期 —— 描述快速宽限期机制的实现原理及其在延迟敏感场景中的应用，支持需要低延迟同步的实时任务。
设计/需求/需求
设计/数据结构/数据结构

.. only:: subproject and html

索引
    

* :ref:`genindex`


==================================================

由 Qwen-plus 及 LT agent 翻译