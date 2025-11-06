.. SPDX-License-Identifier: GPL-2.0


RCU 和 lockdep 检查
===================

所有类型的RCU都提供lockdep检查功能，因此lockdep可以
了解每个任务何时进入和离开任何类型的RCU读端
临界区。每种类型的RCU被单独跟踪（但请注意）
这种情况在2.6.32及更早版本中并不存在）。这使得lockdep的
跟踪以包含RCU状态，在调试时有时会有所帮助
死锁之类的问题。
此外，RCU 提供了以下用于检查 lockdep 的原语
状态::
这些函数具有保守性：当无法确定状态时（例如未启用`CONFIG_DEBUG_LOCK_ALLOC`），会默认返回1。
这可以防止类似 WARN_ON(!rcu_read_lock_held()) 的情况在 lockdep 被禁用时产生误报。
此外，一个独立的内核配置参数 CONFIG_PROVE_RCU 可启用
rcu_dereference() 原语的检查：
rcu_dereference(p):
	检查是否处于对应类型的RCU读端临界区。
rcu_dereference_bh(p):
	检查是否处于对应类型的RCU-bh读端临界区。

rcu_dereference_sched(p):
	检查是否处于对应类型的RCU-sched读端临界区。
srcu_dereference(p, sp):
	检查是否处于对应类型的SRCU读端临界区。

	rcu_read_lock_held() for normal RCU.
	rcu_read_lock_bh_held() for RCU-bh.
	rcu_read_lock_sched_held() for RCU-sched.
	rcu_read_lock_any_held() for any of normal RCU, RCU-bh, and RCU-sched.
	srcu_read_lock_held() for SRCU.
	rcu_read_lock_trace_held() for RCU Tasks Trace.

rcu_dereference_check(p, c):
	使用显式的检查表达式 "c" 以及
	rcu_read_lock_held()。这在代码中非常有用
	由RCU读取器和更新器共同调用。
rcu_dereference_bh_check(p, c):
	使用显式的检查表达式 "c" 以及
	rcu_read_lock_bh_held()。这在代码中非常有用
	由RCU-bh读取器和更新器调用。
rcu_dereference_sched_check(p, c):
	使用显式的检查表达式 "c" 以及
	rcu_read_lock_sched_held()。这在代码中非常有用
	由 RCU-sched 读取器和更新器调用。
srcu_dereference_check(p, c):
	使用显式的检查表达式 "c" 以及
	srcu_read_lock_held()。这在代码中非常有用
	由SRCU读取器和更新器调用。

rcu_dereference_raw(p):
	不要检查。（仅在必要时谨慎使用。）
rcu_dereference_raw_check(p):
	完全不要进行 lockdep。（如有必要，也应谨慎使用。）

rcu_dereference_protected(p, c):
	使用显式的检查表达式 "c"，并省略所有屏障
	以及编译器约束。当数据结构不能改变时，这非常有用，例如在仅由更新者调用的代码中，
	但前提是必须确保无并发修改风险。
rcu_access_pointer(p):
	返回指针的值并省略所有屏障，
	但保留编译器约束以防止重复
	或合并。这在测试
	指针本身的值，例如与 NULL 进行比较时非常有用。
rcu_dereference_check() 的检查表达式可以是任何布尔值
表达式，但通常会包含一个lockdep表达式。对于一个
较为华丽的例子，考虑以下内容：
此表达式以RCU安全的方式获取指针“fdt->fd[fd]”，
并且，如果配置了 CONFIG_PROVE_RCU，则验证该表达式
用于：
1. 一个RCU读端临界区（隐式），或者
2. 在持有 files->file_lock 的情况下，或者
3. 在一个非共享的 files_struct 上。
在情况 (1) 中，指针以 RCU 安全的方式被获取，适用于 vanilla
RCU读端临界区，在情况(2)中，->file_lock防止
任何变化发生，最后，在情况（3）中，当前任务
是唯一访问 file_struct 的任务，从而再次防止任何更改
发生。如果上述语句仅从更新者
代码中调用，也可以按如下方式编写：
这将验证上述情况#2和#3，并且lockdep将进一步
抱怨，即使这在RCU读端临界区中使用，除非
这两种情况之一成立。因为 rcu_dereference_protected() 会省略
所有屏障和编译器约束，它生成的代码比
rcu_dereference() 的其他变体更高效。另一方面，如果RCU保护的指针
或它所指向的受RCU保护的数据可能被并发修改，则使用 rcu_dereference_protected() 是非法的。
与 rcu_dereference() 类似，当启用 lockdep 时，RCU 列表和 hlist
遍历原语会检查是否在RCU读端临界区内被调用
临界区。然而，可以向它们传递一个 lockdep 表达式
作为一个额外的可选参数。使用此 lockdep 表达式时，这些
遍历原语将仅在lockdep表达式为假
且它们是在任何RCU读端临界区之外被调用时发出警告。
例如，工作队列的 for_each_pwq() 宏旨在被使用
要么在RCU读端临界区内，要么持有wq->mutex。
因此，其实现方式如下：
使用显式的检查表达式 "c" 以及
rcu_read_lock_held()。这在代码中非常有用
由RCU读取器和更新器共同调用。
rcu_dereference_bh_check(p, c):
使用显式的检查表达式 "c" 以及
rcu_read_lock_bh_held()。这在代码中非常有用
由RCU-bh读取器和更新器调用。
rcu_dereference_sched_check(p, c):
使用显式的检查表达式 "c" 以及
rcu_read_lock_sched_held()。这在代码中非常有用
由 RCU-sched 读取器和更新器调用。
srcu_dereference_check(p, c):
使用显式的检查表达式 "c" 以及
srcu_read_lock_held()。这在代码中非常有用
由SRCU读取器和更新器调用。
rcu_dereference_raw(p):
不要检查。（仅在必要时谨慎使用。）
rcu_dereference_raw_check(p):
完全不要进行 lockdep。（如有必要，也应谨慎使用。）
rcu_dereference_protected(p, c):
使用显式的检查表达式 "c"，并省略所有屏障
以及编译器约束。当数据时，这非常有用
结构不能改变，例如在代码中
仅由更新器调用。
rcu_access_pointer(p):
返回指针的值并省略所有屏障，
但保留编译器约束以防止重复
或合并。这在测试时非常有用。
指针本身的值，例如与 NULL 进行比较。

rcu_dereference_check() 的检查表达式可以是任何布尔值
表达式，但通常会包含一个lockdep表达式。对于一个
较为华丽的例子，考虑以下内容：

	file = rcu_dereference_check(fdt->fd[fd],
				     lockdep_is_held(&files->file_lock) ||
				     atomic_read(&files->count) == 1);

此表达式以RCU安全的方式获取指针“fdt->fd[fd]”，
并且，如果配置了 CONFIG_PROVE_RCU，则验证该表达式
用于：

1. 一个RCU读端临界区（隐式），或者
2. 在持有 files->file_lock 的情况下，或者
3. 在一个非共享的 files_struct 上。

在情况 (1) 中，指针以 RCU 安全的方式被获取，适用于 vanilla
RCU读端临界区，在情况(2)中，->file_lock防止
任何阻止变化发生的情况，最后，在情况（3）中，当前任务
是唯一访问 file_struct 的任务，从而再次防止任何更改
以防止其发生。如果上述语句仅从updater中调用
代码，也可以按如下方式编写：

	file = rcu_dereference_protected(fdt->fd[fd],
					 lockdep_is_held(&files->file_lock) ||
					 atomic_read(&files->count) == 1);

这将验证上述情况#2和#3，并且lockdep将进一步
即使这在RCU读端临界区中使用，也不要抱怨，除非
这两种情况之一成立。因为 rcu_dereference_protected() 会省略
所有障碍和编译器限制，它生成的代码比
rcu_dereference() 的其他变体。另一方面，这是非法的
如果RCU保护的指针，则使用 rcu_dereference_protected()
或它所指向的受RCU保护的数据可能同时被更改。

与 rcu_dereference() 类似，当启用 lockdep 时，RCU 列表和 hlist
遍历原语会检查是否在RCU读端临界区内被调用
临界区。然而，可以向它们传递一个 lockdep 表达式
作为一个额外的可选参数。使用此 lockdep 表达式时，这些
遍历原语将仅在lockdep表达式为真时发出警告
错误，并且它们是在任何RCU读端临界区之外被调用的。

例如，工作队列的 for_each_pwq() 宏旨在被使用
要么在RCU读端临界区内，要么持有wq->mutex。
因此，其实现方式如下：

	#define for_each_pwq(pwq, wq)
		list_for_each_entry_rcu((pwq), &(wq)->pwqs, pwqs_node,
					lock_is_held(&(wq->mutex).dep_map))


==================================================

由 Qwen-plus 及 LT agent 翻译