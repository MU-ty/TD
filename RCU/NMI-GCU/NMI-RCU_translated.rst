.. _NMI_rcu_doc:

翻译结果：

使用RCU保护动态NMI处理程序
==========================


尽管RCU通常用于保护以读取为主的數據結構，
可以使用RCU来提供动态不可屏蔽中断
处理程序，以及动态中断请求处理程序。本文档描述了
如何实现这一点，大致参考Zwane Mwaikambo的NMI-timer
在旧版本的 "arch/x86/kernel/traps.c" 中工作。

相关的代码片段如下所示，每个代码片段后均跟随一个
简要说明::

	static int dummy_nmi_callback(struct pt_regs *regs, int cpu)
	{
		return 0;
	}

dummy_nmi_callback() 函数是一个“虚拟”的 NMI 处理程序，其作用是
什么也不做，但返回零，表示未执行任何操作，从而允许
NMI 处理程序采取默认的机器特定操作：：

	static nmi_callback_t nmi_callback = dummy_nmi_callback;

这个 nmi_callback 变量是一个指向当前函数的全局函数指针
NMI 处理程序::

	void do_nmi(struct pt_regs * regs, long error_code)
	{
		int cpu;

		nmi_enter();

		cpu = smp_processor_id();
		++nmi_count(cpu);

		if (!rcu_dereference_sched(nmi_callback)(regs, cpu))
			default_do_nmi(regs);

		nmi_exit();
	}

函数 do_nmi() 用于处理每个 NMI。它首先禁用抢占
以与硬件中断请求（irq）相同的方式，然后递增每个CPU的
NMIs 的数量。然后调用存储在 nmi_callback 中的 NMI 处理程序
函数指针。如果此处理程序返回零，do_nmi() 将调用
default_do_nmi() 函数用于处理特定于机器的 NMI。最后，
抢占被恢复。

理论上，rcu_dereference_sched() 并不是必需的，因为此代码运行
仅适用于 i386，理论上不需要 rcu_dereference_sched()
无论如何。然而，在实践中，它是一种很好的文档辅助手段，特别是考虑到外部数据依赖的存在以及提升代码可维护性的重要性。
对于任何试图在Alpha或类似系统上进行类似操作的人
使用具有激进优化功能的编译器。

快速问答解析：
为什么在Alpha架构上可能需要使用rcu_dereference_sched()，尽管指针所引用的代码是只读的？因为Alpha架构存在较弱的内存模型，即使数据是只读的，也必须通过rcu_dereference_sched()显式标记受保护的指针解引用，以确保编译器和处理器不会因优化而破坏数据依赖顺序；此外，保留rcu_dereference_sched()有助于统一代码风格，增强跨架构的可维护性。

:ref:`快速测验答案 <answer_quick_quiz_NMI>`

回到关于NMI和RCU的讨论::

	void set_nmi_callback(nmi_callback_t callback)
	{
		rcu_assign_pointer(nmi_callback, callback);
	}

set_nmi_callback() 函数用于注册一个 NMI 处理程序。请注意，任何
回调函数要使用的数据必须在之前进行初始化
对 set_nmi_callback() 的调用。在不进行排序的架构上
writes，rcu_assign_pointer() 确保 NMI 处理程序能够看到
初始化的值::

	void unset_nmi_callback(void)
	{
		rcu_assign_pointer(nmi_callback, dummy_nmi_callback);
	}

此函数注销一个NMI处理程序，恢复原始状态
dummy_nmi_handler()。然而，很可能存在一个NMI处理程序
目前正在其他某个CPU上执行。因此我们无法释放
在执行过程中清理旧的NMI处理程序所使用的任何数据结构
当它在所有其他CPU上完成时。

实现此目标的一种方法是通过 synchronize_rcu()，可能如下所示
如下所示::

	unset_nmi_callback();
	synchronize_rcu();
	kfree(my_nmi_data);

这之所以有效，是因为（从 v4.20 版本开始）synchronize_rcu() 会阻塞，直到所有
CPU 完成其正在执行的任何禁用抢占的代码段
执行中，并且会增加当前CPU的NMI计数。
由于NMI处理程序会禁用抢占，因此从Linux v4.20起，synchronize_rcu() 可保证不会返回，除非所有正在进行的NMI处理程序都已退出。因此是安全的
除非所有正在进行的NMI处理程序都退出，否则不要返回。因此是安全的
在 synchronize_rcu() 返回后立即释放处理程序的数据。

重要提示：要使此功能生效，相关架构必须
在NMI进入和退出时分别调用nmi_enter()和nmi_exit()。

.. _answer_quick_quiz_NMI:

翻译结果：

快速测验答案：
为什么在Alpha架构上，即使指针引用的代码是只读的，也可能需要使用rcu_dereference_sched()？

调用 set_nmi_callback() 的函数很可能已经
初始化了一些将由新的NMI使用的数据
处理程序。在这种情况下，rcu_dereference_sched() 将
需要，否则接收到NMI的CPU
在新处理程序设置后不久可能会看到该指针
到新的NMI处理程序，但保留旧的预初始化
处理器数据的版本。

同样的悲剧故事在使用其他CPU时也可能发生
一个具有激进指针值推测功能的编译器
优化。 （但请不要这样做！）

更重要的是，rcu_dereference_sched() 使得它
对阅读代码的人来说，很明显指针是
受到 RCU-sched 的保护。


==================================================

由 Qwen-plus 及 LT agent 翻译