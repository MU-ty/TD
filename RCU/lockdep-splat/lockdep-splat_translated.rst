.. SPDX-License-Identifier: GPL-2.0


Lockdep-RCU Splat
=================

Lockdep-RCU 于2010年初被添加到Linux内核中
(http://lwn.net/Articles/371986/). 此功能会检查一些常见情况
RCU API 的误用，最值得注意的是使用了 `rcu_dereference()` 系列函数之一
在没有适当保护的情况下访问受 RCU 保护的指针。
当检测到此类误用时，会发出一个 Lockdep-RCU splat 警告。

通常导致 Lockdep-RCU splat 警告的原因是有人访问了一个
RCU 保护的数据结构，既不满足 (1) 处于正确类型的
RCU 读端临界区（直到完成引用计数或其他安全机制的建立），也不满足 (2) 持有正确的更新端锁。
因此，这个问题可能很严重：它可能导致随机内存
覆盖或更糟的情况。当然，也可能会出现误报，这种情况
毕竟这是真实世界，诸如此类。

那么让我们来看一个来自3.0-rc5的RCU lockdep报错示例，其中一个
早已修复::


警告：可疑的 RCU 使用
                     
block/cfq-iosched.c:2776 存在可疑的 rcu_dereference_protected() 使用！

可能有助于我们调试此问题的其他信息：

    rcu_scheduler_active = 1, debug_locks = 0
    3 locks held by scsi_scan_6/1552:
    #0:  (&shost->scan_mutex){+.+.}, at: [<ffffffff8145efca>]
    scsi_scan_host_selected+0x5a/0x150
    #1:  (&eq->sysfs_lock){+.+.}, at: [<ffffffff812a5032>]
    elevator_exit+0x22/0x60
    #2:  (&(&q->__queue_lock)->rlock){-.-.}, at: [<ffffffff812b6233>]
    cfq_exit_queue+0x43/0x190

    stack backtrace:
    Pid: 1552, comm: scsi_scan_6 Not tainted 3.0.0-rc5 #17
    Call Trace:
    [<ffffffff810abb9b>] lockdep_rcu_dereference+0xbb/0xc0
    [<ffffffff812b6139>] __cfq_exit_single_io_context+0xe9/0x120
    [<ffffffff812b626c>] cfq_exit_queue+0x7c/0x190
    [<ffffffff812a5046>] elevator_exit+0x36/0x60
    [<ffffffff812a802a>] blk_cleanup_queue+0x4a/0x60
    [<ffffffff8145cc09>] scsi_free_queue+0x9/0x10
    [<ffffffff81460944>] __scsi_remove_device+0x84/0xd0
    [<ffffffff8145dca3>] scsi_probe_and_add_lun+0x353/0xb10
    [<ffffffff817da069>] ? error_exit+0x29/0xb0
    [<ffffffff817d98ed>] ? _raw_spin_unlock_irqrestore+0x3d/0x80
    [<ffffffff8145e722>] __scsi_scan_target+0x112/0x680
    [<ffffffff812c690d>] ? trace_hardirqs_off_thunk+0x3a/0x3c
    [<ffffffff817da069>] ? error_exit+0x29/0xb0
    [<ffffffff812bcc60>] ? kobject_del+0x40/0x40
    [<ffffffff8145ed16>] scsi_scan_channel+0x86/0xb0
    [<ffffffff8145f0b0>] scsi_scan_host_selected+0x140/0x150
    [<ffffffff8145f149>] do_scsi_scan_host+0x89/0x90
    [<ffffffff8145f170>] do_scan_async+0x20/0x160
    [<ffffffff8145f150>] ? do_scsi_scan_host+0x90/0x90
    [<ffffffff810975b6>] kthread+0xa6/0xb0
    [<ffffffff817db154>] kernel_thread_helper+0x4/0x10
    [<ffffffff81066430>] ? finish_task_switch+0x80/0x110
    [<ffffffff817d9c04>] ? retint_restore_args+0xe/0xe
    [<ffffffff81097510>] ? __kthread_init_worker+0x70/0x70
    [<ffffffff817db150>] ? gs_change+0xb/0xb

v3.0-rc5 版本中 block/cfq-iosched.c 文件的第 2776 行如下：

	if (rcu_dereference(ioc->ioc_data) == cic) {

此表格注明必须位于一个纯正的RCU读端临界区中
但上面的“其他信息”列表显示情况并非如此
case。相反，我们持有三个锁，其中一个可能与RCU相关。
也许这个锁确实保护了这个引用。如果是这样，修复方法
是通知RCU，也许可以通过修改 __cfq_exit_single_io_context() 来实现
将结构体 request_queue "q" 作为参数从 cfq_exit_queue() 中取出，
这将允许我们按如下方式调用 rcu_dereference_protected：

	if (rcu_dereference_protected(ioc->ioc_data,
				      lockdep_is_held(&q->queue_lock)) == cic) {

通过此更改，如果出现这种情况，则不会发出 lockdep-RCU splat。
代码是在RCU读端临界区内部被调用的
或在持有 ->queue_lock 的情况下。特别是，这会抑制
上述 lockdep-RCU 报错是因为 ->queue_lock 被持有（参见 #2）
如上所列。

另一方面，也许我们确实需要一个RCU读端临界区
部分。在这种情况下，临界区必须覆盖该使用的整个范围
来自 rcu_dereference() 的返回值，或至少直到有某些情况发生
引用计数递增或类似操作。处理此问题的一种方法是
添加 rcu_read_lock() 和 rcu_read_unlock() 如下：

	rcu_read_lock();
	if (rcu_dereference(ioc->ioc_data) == cic) {
		spin_lock(&ioc->lock);
		rcu_assign_pointer(ioc->ioc_data, NULL);
		spin_unlock(&ioc->lock);
	}
	rcu_read_unlock();

通过此更改，rcu_dereference() 始终位于 RCU 保护区域内
读端临界区，这同样会抑制
上述 lockdep-RCU 报错信息。

但在这种特定情况下，我们实际上并没有解引用该指针
从 rcu_dereference() 返回。相反，该指针仅被进行比较
指向cic指针，这意味着rcu_dereference()可以被替换
由 rcu_access_pointer() 如下所示：

	if (rcu_access_pointer(ioc->ioc_data) == cic) {

因为可以在没有保护的情况下合法调用 rcu_access_pointer()，
这一更改还将抑制上述 lockdep-RCU 报错信息。


==================================================

由 Qwen-plus 及 LT agent 翻译