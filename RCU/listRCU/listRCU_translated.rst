.. _list_rcu_doc:

列表RCU文档

使用RCU保护以读为主的链表
=========================

保护以读为主的链表是RCU最常见的用途之一
（list.h 中的 ``struct list_head``）。这种方法的一个主要优势是
所有必需的内存排序均由列表宏提供。
本文档介绍了几种基于列表的RCU使用场景。

在持有 rcu_read_lock() 时遍历列表，写入者可能会
修改列表。读者保证能看到所有元素
在获取 rcu_read_lock() 之前被添加到列表中的
当它们调用 rcu_read_unlock() 时，仍然在列表中。
添加到列表中的元素或从列表中移除的元素可能包含，也可能不包含
被看到。如果写入者调用 list_replace_rcu()，读取者可能会看到
既不会看到旧元素，也不会看到新元素；他们不会同时看到两者。
他们也不会看到。


示例1：以读为主的列表：延迟销毁
-------------------------------

内核中RCU链表的一个广泛使用场景是无锁遍历
系统中的所有进程。``task_struct::tasks`` 表示链表中的节点
链接所有进程。该列表可以与任何列表并行遍历
增删。

使用 ``for_each_process()`` 来遍历列表，该函数已定义
由以下两个宏：

	#define next_task(p) \
		list_entry_rcu((p)->tasks.next, struct task_struct, tasks)

	#define for_each_process(p) \
		for (p = &init_task ; (p = next_task(p)) != &init_task ; )

遍历所有进程列表的代码通常如下所示：

	rcu_read_lock();
	for_each_process(p) {
		/* Do something with p */
	}
	rcu_read_unlock();

用于从...中移除进程的简化且高度内联的代码
任务列表为：

	void release_task(struct task_struct *p)
	{
		write_lock(&tasklist_lock);
		list_del_rcu(&p->tasks);
		write_unlock(&tasklist_lock);
		call_rcu(&p->rcu, delayed_put_task_struct);
	}

当进程退出时，``release_task()`` 会调用 ``list_del_rcu(&p->tasks)``，该操作在写者并发上下文中执行，可安全地与其他写操作（如添加、删除或替换节点）并行进行。
通过在 ``tasklist_lock`` 保护下的 __exit_signal() 和 __unhash_process() 执行。
写者锁保护。``list_del_rcu()`` 调用会移除
从所有任务列表中移除该任务。``tasklist_lock`` 确保写操作的互斥性，而 RCU 机制允许多个读端并发访问链表。
防止并发的列表添加/删除操作导致数据损坏
list. 使用 ``for_each_process()`` 的读者未受到保护
``tasklist_lock``。防止读者注意到列表中的变化
指针，只有在经过一次或多次之后，``task_struct`` 对象才会被释放
宽限期结束后，通过调用 call_rcu() 来实现
put_task_struct_rcu_user()。这种延迟销毁机制确保了
任何遍历该列表的读者都会看到有效的 ``p->tasks.next`` 指针
删除/释放操作可以与列表的遍历并行进行。
这种模式也被称为**存在锁**，因为RCU会避免
从调用 delayed_put_task_struct() 回调函数开始直到
所有现有读者完成，这保证了 ``task_struct``
所讨论的对象将一直存在，直至任务完成之后
所有可能持有该对象引用的RCU读取者。


示例2：在锁之外执行的读取侧操作：无原地更新
-------------------------------------------

某些读写锁的使用场景在持有锁的同时计算一个值
读取侧锁，但在释放该锁后继续使用该值
已发布。这些用例通常是转换的良好候选方案
到RCU。一个显著的例子涉及网络数据包路由。
因为数据包路由信息跟踪了外部设备的状态
由于计算机的特性，它有时会包含过时的数据。因此，一旦
路由已计算完成，无需再保留路由表
数据包传输过程中的静态状态。毕竟，你可以保持
你可以随意配置静态路由表，但这并不能阻止外部
互联网不断变化，它反映了外部互联网的状态
这才是真正重要的。此外，路由条目通常会被添加
或被删除，而不是就地修改。这是一个罕见的例子
光速的有限性以及原子具有非零尺寸的事实
有助于使同步变得更轻量。

此类RCU用例的一个简单示例如下所示：
系统调用审计支持。例如，一个读写锁
``audit_filter_task()`` 的实现可能如下所示::

	static enum audit_state audit_filter_task(struct task_struct *tsk, char **key)
	{
		struct audit_entry *e;
		enum audit_state   state;

		read_lock(&auditsc_lock);
		/* Note: audit_filter_mutex held by caller. */
		list_for_each_entry(e, &audit_tsklist, list) {
			if (audit_filter_rules(tsk, &e->rule, NULL, &state)) {
				if (state == AUDIT_STATE_RECORD)
					*key = kstrdup(e->rule.filterkey, GFP_ATOMIC);
				read_unlock(&auditsc_lock);
				return state;
			}
		}
		read_unlock(&auditsc_lock);
		return AUDIT_BUILD_CONTEXT;
	}

此处列表在锁的保护下进行搜索，但在之后锁被释放。
相应的值将被返回。当该值被处理时
on，列表很可能已被修改。这是合理的，因为如果
你正在关闭审计功能，多审计几个系统调用是没问题的。

这意味着RCU可以很容易地应用于读取端，如下所示：

	static enum audit_state audit_filter_task(struct task_struct *tsk, char **key)
	{
		struct audit_entry *e;
		enum audit_state   state;

		rcu_read_lock();
		/* Note: audit_filter_mutex held by caller. */
		list_for_each_entry_rcu(e, &audit_tsklist, list) {
			if (audit_filter_rules(tsk, &e->rule, NULL, &state)) {
				if (state == AUDIT_STATE_RECORD)
					*key = kstrdup(e->rule.filterkey, GFP_ATOMIC);
				rcu_read_unlock();
				return state;
			}
		}
		rcu_read_unlock();
		return AUDIT_BUILD_CONTEXT;
	}

read_lock() 和 read_unlock() 调用已变为 rcu_read_lock()
以及 ``rcu_read_lock()`` 和 ``rcu_read_unlock()``，分别对应读临界区的开始与结束；而原来的 ``list_for_each_entry()``
已变为 ``list_for_each_entry_rcu()``。**_rcu()** 列表遍历
原语增加了 ``READ_ONCE()`` 语义保障及针对错误使用的诊断检查，确保在弱内存序架构（如 ARM、PowerPC）上的正确性。
不在RCU读端临界区之内。

更新侧的更改也同样简单。读者-写者锁
可能在这些简化的删除和插入操作中如下使用
audit_del_rule() 和 audit_add_rule() 的版本：

	static inline int audit_del_rule(struct audit_rule *rule,
					 struct list_head *list)
	{
		struct audit_entry *e;

		write_lock(&auditsc_lock);
		list_for_each_entry(e, list, list) {
			if (!audit_compare_rule(rule, &e->rule)) {
				list_del(&e->list);
				write_unlock(&auditsc_lock);
				return 0;
			}
		}
		write_unlock(&auditsc_lock);
		return -EFAULT;		/* No matching rule */
	}

	static inline int audit_add_rule(struct audit_entry *entry,
					 struct list_head *list)
	{
		write_lock(&auditsc_lock);
		if (entry->rule.flags & AUDIT_PREPEND) {
			entry->rule.flags &= ~AUDIT_PREPEND;
			list_add(&entry->list, list);
		} else {
			list_add_tail(&entry->list, list);
		}
		write_unlock(&auditsc_lock);
		return 0;
	}

以下是这两个函数的RCU等价形式：

	static inline int audit_del_rule(struct audit_rule *rule,
					 struct list_head *list)
	{
		struct audit_entry *e;

		/* No need to use the _rcu iterator here, since this is the only
		 * deletion routine. */
		list_for_each_entry(e, list, list) {
			if (!audit_compare_rule(rule, &e->rule)) {
				list_del_rcu(&e->list);
				call_rcu(&e->rcu, audit_free_rule);
				return 0;
			}
		}
		return -EFAULT;		/* No matching rule */
	}

	static inline int audit_add_rule(struct audit_entry *entry,
					 struct list_head *list)
	{
		if (entry->rule.flags & AUDIT_PREPEND) {
			entry->rule.flags &= ~AUDIT_PREPEND;
			list_add_rcu(&entry->list, list);
		} else {
			list_add_tail_rcu(&entry->list, list);
		}
		return 0;
	}

通常情况下，write_lock() 和 write_unlock() 将被替换为
spin_lock() 和 spin_unlock()。但在这种情况下，所有调用者都持有
``audit_filter_mutex`` 已足以同步写者，因此不需要额外的锁定。RCU 的引入使得原用于排他写的 ``auditsc_lock`` 完全可以被移除，从而显著简化了同步逻辑。
因此可以彻底消除 ``auditsc_lock``，因为使用 RCU 后不再需要写者排除读者的锁机制，实现了无锁读取和写操作的解耦。
需要写者排除读者的传统需求。

``list_del()``、``list_add()`` 和 ``list_add_tail()`` 原语已被
由 ``list_del_rcu()``、``list_add_rcu()`` 和 ``list_add_tail_rcu()`` 替代，这些 RCU 版本原语隐含必要的内存屏障和宽限期管理支持。
**_rcu()** 列表操作原语添加了内存屏障，这些屏障用于确保在弱内存序 CPU 上的数据可见性和顺序性，并配合 ``READ_ONCE()`` 防止编译器优化导致的重排序问题。
在弱排序CPU上需要。list_del_rcu() 原语省略了
会导致并发问题的指针中毒调试辅助代码
读者惨败。

因此，当读者能够容忍陈旧数据，并且条目仅被添加或
已删除，无需原地修改，使用RCU非常简单！


示例3：处理就地更新
-------------------

系统调用审计代码不会就地更新审计规则。然而，
如果确实如此，用于实现该功能的读写锁代码可能如下所示
（假设只有 ``field_count`` 被更新，否则添加的字段将会
需要填写::

	static inline int audit_upd_rule(struct audit_rule *rule,
					 struct list_head *list,
					 __u32 newaction,
					 __u32 newfield_count)
	{
		struct audit_entry *e;
		struct audit_entry *ne;

		write_lock(&auditsc_lock);
		/* Note: audit_filter_mutex held by caller. */
		list_for_each_entry(e, list, list) {
			if (!audit_compare_rule(rule, &e->rule)) {
				e->rule.action = newaction;
				e->rule.field_count = newfield_count;
				write_unlock(&auditsc_lock);
				return 0;
			}
		}
		write_unlock(&auditsc_lock);
		return -EFAULT;		/* No matching rule */
	}

RCU 版本会创建一个副本，更新该副本，然后替换旧的
用新更新的条目替换原条目。这一系列操作，允许
在进行复制以执行更新的同时支持并发读取，正是这一点提供了
RCU（*read-copy update*）的名称。

audit_upd_rule() 的RCU版本如下::

	static inline int audit_upd_rule(struct audit_rule *rule,
					 struct list_head *list,
					 __u32 newaction,
					 __u32 newfield_count)
	{
		struct audit_entry *e;
		struct audit_entry *ne;

		list_for_each_entry(e, list, list) {
			if (!audit_compare_rule(rule, &e->rule)) {
				ne = kmalloc(sizeof(*entry), GFP_ATOMIC);
				if (ne == NULL)
					return -ENOMEM;
				audit_copy_rule(&ne->rule, &e->rule);
				ne->rule.action = newaction;
				ne->rule.field_count = newfield_count;
				list_replace_rcu(&e->list, &ne->list);
				call_rcu(&e->rcu, audit_free_rule);
				return 0;
			}
		}
		return -EFAULT;		/* No matching rule */
	}

同样，这假设调用者持有 ``audit_filter_mutex``。通常情况下，
写入锁在这种代码中会变成自旋锁。

update_lsm_rule() 所做的工作与此非常相似，适用于那些需要的人
更倾向于查看真实的 Linux 内核代码。

该模式的另一个应用可以在 openswitch 驱动程序的 *connection 中找到
``ct_limit_set()`` 中的跟踪表*代码。该表用于存储连接跟踪
条目，并对最大条目数有限制。存在这样一个表
每个区域一个，因此每个区域一个*限制*。这些区域被映射到其对应的限制值。
通过使用由RCU管理的hlist作为哈希链的哈希表。当一个新
如果设置了限制，则会分配一个新的限制对象并调用 ``ct_limit_set()``
使用 list_replace_rcu() 将旧的 limit 对象替换为新的对象。
在经过一个宽限期后，旧的限制对象将通过 kfree_rcu() 被释放。


示例4：消除过时数据
-------------------

上述审计示例可以容忍过时的数据，大多数算法也是如此
正在跟踪外部状态。毕竟，存在延迟
从外部状态改变到Linux意识到之前的时间
变化的量，因此如前所述，会有少量额外的
RCU引起的陈旧性通常不是问题。

然而，有许多情况下无法容忍陈旧的数据。
Linux 内核中的一个例子是 System V IPC（参见 shm_lock()）
function in ipc/shm.c)。此代码在锁的保护下检查一个 *deleted* 标志。
每个条目一个自旋锁，如果设置了 *deleted* 标志，则假装该条目不存在
条目不存在。为了使其更有帮助，搜索功能必须
返回时持有每个条目的自旋锁，正如 shm_lock() 实际所做的那样。

.. _quick_quiz:

快速测验：

快速测验：
对于删除标记技术要发挥作用，为什么必须满足以下条件
在从搜索函数返回时是否需要持有每个条目的锁？

:ref:`快速测验答案 <quick_quiz_answer>`

如果系统调用审计模块需要拒绝过时数据，一种方法是
要实现这一点，需要添加一个 ``deleted`` 标志和一个 ``lock`` 自旋锁到
``audit_entry`` 结构，并按如下方式修改 audit_filter_task()::

	static struct audit_entry *audit_filter_task(struct task_struct *tsk, char **key)
	{
		struct audit_entry *e;
		enum audit_state   state;

		rcu_read_lock();
		list_for_each_entry_rcu(e, &audit_tsklist, list) {
			if (audit_filter_rules(tsk, &e->rule, NULL, &state)) {
				spin_lock(&e->lock);
				if (e->deleted) {
					spin_unlock(&e->lock);
					rcu_read_unlock();
					return NULL;
				}
				rcu_read_unlock();
				if (state == AUDIT_STATE_RECORD)
					*key = kstrdup(e->rule.filterkey, GFP_ATOMIC);
				/* As long as e->lock is held, e is valid and
				 * its value is not stale */
				return e;
			}
		}
		rcu_read_unlock();
		return NULL;
	}

函数 ``audit_del_rule()`` 需要在下方设置 ``deleted`` 标志
自旋锁如下：

	static inline int audit_del_rule(struct audit_rule *rule,
					 struct list_head *list)
	{
		struct audit_entry *e;

		/* No need to use the _rcu iterator here, since this
		 * is the only deletion routine. */
		list_for_each_entry(e, list, list) {
			if (!audit_compare_rule(rule, &e->rule)) {
				spin_lock(&e->lock);
				list_del_rcu(&e->list);
				e->deleted = 1;
				spin_unlock(&e->lock);
				call_rcu(&e->rcu, audit_free_rule);
				return 0;
			}
		}
		return -EFAULT;		/* No matching rule */
	}

这也假设调用者持有 ``audit_filter_mutex``。

请注意，此示例假设条目仅被添加和删除。
需要额外的机制来正确处理就地更新
由 audit_upd_rule() 执行。一方面，audit_upd_rule() 会
需要同时持有旧的 ``audit_entry`` 及其替换项的锁
在执行 list_replace_rcu() 时。


示例5：跳过陈旧对象
-------------------

在某些使用场景下，通过跳过部分内容可以提升读取性能
读取侧列表遍历时的陈旧对象，其中陈旧对象
将在一个或多个宽限期后被移除和销毁
时期。一个这样的例子可以在 timerfd 子系统中找到。当一个
``CLOCK_REALTIME`` 时钟被重新编程（例如由于设置）
（系统时间）那么所有依赖于
这个时钟被触发后，等待它的进程将被唤醒
在计划到期之前。为了便于实现这一点，所有此类定时器
当它们被设置时，会被添加到RCU管理的``cancel_list``中
``timerfd_setup_cancel()``::

	static void timerfd_setup_cancel(struct timerfd_ctx *ctx, int flags)
	{
		spin_lock(&ctx->cancel_lock);
		if ((ctx->clockid == CLOCK_REALTIME ||
		     ctx->clockid == CLOCK_REALTIME_ALARM) &&
		    (flags & TFD_TIMER_ABSTIME) && (flags & TFD_TIMER_CANCEL_ON_SET)) {
			if (!ctx->might_cancel) {
				ctx->might_cancel = true;
				spin_lock(&cancel_lock);
				list_add_rcu(&ctx->clist, &cancel_list);
				spin_unlock(&cancel_lock);
			}
		} else {
			__timerfd_remove_cancel(ctx);
		}
		spin_unlock(&ctx->cancel_lock);
	}

当 timerfd 被释放（fd 被关闭）时，``might_cancel``
timerfd对象的标志被清除，该对象从
``cancel_list`` 并被销毁，如下所示的简化和内联代码
timerfd_release() 的版本::

	int timerfd_release(struct inode *inode, struct file *file)
	{
		struct timerfd_ctx *ctx = file->private_data;

		spin_lock(&ctx->cancel_lock);
		if (ctx->might_cancel) {
			ctx->might_cancel = false;
			spin_lock(&cancel_lock);
			list_del_rcu(&ctx->clist);
			spin_unlock(&cancel_lock);
		}
		spin_unlock(&ctx->cancel_lock);

		if (isalarm(ctx))
			alarm_cancel(&ctx->t.alarm);
		else
			hrtimer_cancel(&ctx->t.tmr);
		kfree_rcu(ctx, rcu);
		return 0;
	}

如果 ``CLOCK_REALTIME`` 时钟被设置，例如通过时间服务器，
hrtimer 框架调用 ``timerfd_clock_was_set()``，该函数会遍历
``cancel_list`` 并唤醒在 timerfd 上等待的进程。在迭代过程中
在 ``cancel_list`` 中，会参考 ``might_cancel`` 标志以跳过过时的条目
对象::

	void timerfd_clock_was_set(void)
	{
		ktime_t moffs = ktime_mono_to_real(0);
		struct timerfd_ctx *ctx;
		unsigned long flags;

		rcu_read_lock();
		list_for_each_entry_rcu(ctx, &cancel_list, clist) {
			if (!ctx->might_cancel)
				continue;
			spin_lock_irqsave(&ctx->wqh.lock, flags);
			if (ctx->moffs != moffs) {
				ctx->moffs = KTIME_MAX;
				ctx->ticks++;
				wake_up_locked_poll(&ctx->wqh, EPOLLIN);
			}
			spin_unlock_irqrestore(&ctx->wqh.lock, flags);
		}
		rcu_read_unlock();
	}

关键是，由于受RCU保护的遍历过程
``cancel_list`` 与对象的添加和移除同时进行，
有时遍历可能会访问一个已从
列表。在此示例中，使用了一个标志来跳过此类对象。


摘要
----

可以容忍陈旧数据的以读为主的基于列表的数据结构是
最适用于使用RCU的情况。最简单的情形是条目被
要么从数据结构中添加或删除（或原子性地修改）
（就地操作），但非原子性的就地修改可以通过以下方式处理：
一份副本，更新该副本，然后用副本替换原始文件。
如果无法容忍陈旧数据，则可以使用 *deleted* 标志
与每个条目的自旋锁结合使用，以允许搜索
拒绝新删除数据的函数。

.. _quick_quiz_answer:

翻译结果：

快速测验答案：
对于删除标记技术要发挥作用，为什么必须满足以下条件
在从搜索函数返回时，是否需要持有每个条目的锁？

如果搜索函数在返回前释放了每个条目的锁，
那么调用方无论如何都将处理过时的数据。如果
如果处理过期数据确实没问题，那么你就不需要一个
*deleted* 标志。如果处理过期数据确实是个问题，
那么你需要在整个代码中持有每个条目的锁
使用了返回值。

:ref:`返回快速测验 <quick_quiz>`


==================================================

由 Qwen-plus 及 LT agent 翻译