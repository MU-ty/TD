.. SPDX-License-Identifier: GPL-2.0


RCU补丁的审查清单
=================


本文档包含一个用于生成和审查补丁的检查清单
这些规则的任何一条都将导致使用RCU时出现问题。
导致与省略锁原语相同类型的问题
会造成的问题。此列表基于审查此类补丁的经验
在相当长的一段时间内，但改进始终欢迎！

0. RCU 是否应用于以读取为主的情境？如果数据
结构更新频率超过约10% 时，则
应认真考虑其他方法，除非有详细说明
性能测量表明，RCU 仍然是正确的
合适的工具。是的，RCU 通过减少读取侧开销来
写入端开销增加，这正是普通使用场景下出现此问题的原因
RCU 的读取操作将远多于更新操作。

另一个例外是性能不是问题，且使用RCU的情况。
提供了更简单的实现方式。这种情况的一个示例
是Linux 2.6内核中的动态NMI代码，至少在
NMI 罕见的架构。

另一个例外情况是RCU的低实时延迟。
读取端原语至关重要。

最后一个例外情况是使用RCU读取器来防止
ABA问题（https://en.wikipedia.org/wiki/ABA_problem）
用于无锁更新。这确实会导致轻微的
反直觉的情况，其中 rcu_read_lock() 和
rcu_read_unlock() 用于保护更新，然而，这一点
该方法可以为某些类型提供相同的简化
无锁算法，垃圾回收器所使用的那种。

1. 更新代码是否具有适当的互斥机制？

RCU 确实允许*读取者*几乎裸奔，但*写入者*必须
仍然使用某种形式的互斥机制，例如：

a. 锁定，
b. 原子操作，或者
c. 限制更新到单个任务。

如果你选择 #b，请准备好描述你是如何处理的
弱序机器上的内存屏障（几乎所有的）
它们——即使是x86架构也允许后续的加载操作被重排序到前面
（早期的商店），并准备好解释为何这一附加内容
复杂性是值得的。如果你选择#c，请做好准备
解释这一单一任务如何不会成为主要瓶颈
在大型系统中（例如，如果任务是更新信息
与自身相关的内容，其他任务可以据此读取
不能成为瓶颈。注意，“大”的定义
发生了显著变化：在2000年，八个CPU被认为是“大型”的，
但在2017年，一百个CPU却并不稀奇。

2. RCU读端临界区是否正确使用了
rcu_read_lock() 及其相关函数？这些原语是必需的
防止宽限期过早结束，这
可能导致数据被粗暴地释放出来
在你的读取端代码下方，这可能会显著增加
内核的精算风险。

作为一条粗略的经验法则，任何对RCU保护的
指针必须由 rcu_read_lock() 或 rcu_read_lock_bh() 保护。
rcu_read_lock_sched()，或由适当的更新端锁保护。
显式禁用抢占（例如，preempt_disable()）
可以充当 rcu_read_lock_sched()，但可读性较差
防止 lockdep 检测锁问题。获取一个
原始自旋锁也会进入RCU读端临界区。

guard(rcu)() 和 scoped_guard(rcu) 原语用于指定
当前作用域的剩余部分或下一条语句，
分别作为RCU读端临界区。使用
这些保护机制可能比 rcu_read_lock() 更不容易出错，
rcu_read_unlock() 及其相关函数。

请注意，你*不能*依赖已知已构建的代码
仅在不可抢占式内核中。此类代码可能且将会出错，
尤其是在启用 CONFIG_PREEMPT_COUNT=y 选项构建的内核中。

让受RCU保护的指针从RCU读端“泄露”出去
临界区的问题丝毫不亚于任其泄露出去
从锁具下方。当然，除非你已经安排了其他方式
其他保护方式，例如锁或引用计数
*在* 让它们离开RCU读端临界区 *之前*。

3. 更新代码是否支持并发访问？

RCU 的全部意义在于允许读者无需加锁即可运行
任何锁或原子操作。这意味着读者将
在更新进行期间正在运行。存在多个数量
处理此并发问题的方法有多种，具体取决于实际情况：

a. 使用RCU变体的list和hlist更新
用于添加、删除和替换元素的基本操作
一个受RCU保护的列表。或者，使用其他的
已添加到 RCU 保护的数据结构
Linux 内核。

这几乎总是最佳方法。

b. 按照上述 (a) 的步骤操作，但还需为每个元素维护
锁（由读取者和写入者获取的）
该守卫用于维护每个元素的状态。供读取器使用的字段
避免访问可能受到其他锁的保护
仅在需要时由更新器获取。

这也非常有效。

c. 使更新对读者而言呈现为原子性操作。例如，
指向正确对齐字段的指针更新将
看起来是原子操作，单个原子原语也是如此。
在锁下执行的操作序列将*不会*
对RCU读取器而言似乎具有原子性，序列也不会
多个原子原语。一种替代方案是
将多个独立字段移动到一个单独的结构中，
从而通过施加一种
额外的间接层。

这可以实现，但开始变得有点复杂了。

d. 仔细安排更新和读取的顺序，以便读者
在更新的各个阶段都能看到有效数据。这通常
比听起来更困难，尤其是在现代情况下
CPU倾向于对内存引用进行重排序。必须
通常会大量使用内存排序操作
通过代码，使其难以理解和
用于测试。在适用的情况下，最好使用这些工具
类似于 smp_store_release() 和 smp_load_acquire()，但具有
在某些情况下，需要使用 smp_mb() 完整内存屏障。

如前所述，通常最好将它们分组
将数据转换为独立的结构，以便于
通过更新指针，可使更改看似原子操作
引用一个包含更新值的新结构。

4. 弱序CPU带来了特殊的挑战。几乎所有的CPU
弱排序的——即使是x86 CPU也允许后续的加载操作
重新排序以位于之前的存储操作之前。RCU 代码必须考虑所有这些情况。
以下措施可防止内存损坏问题：

a. 读者必须保持内存的正确顺序
访问。rcu_dereference() 原语确保了
CPU 在获取数据之前先获取指针
指针所指向的内容。这确实是必要的
在 Alpha CPU 上。

rcu_dereference() 原语也是一个极佳的选择
文档辅助，让阅读者能够更好地理解
代码确切地知道哪些指针受到RCU保护。
请注意，编译器也可能会对代码进行重排序。
他们正变得越来越积极主动地采取行动
仅此而已。因此，rcu_dereference() 原语也
可防止破坏性的编译器优化。然而，
通过一些狡黠的创造力，是有可能的
错误处理来自 rcu_dereference() 的返回值。
请参阅 rcu_dereference.rst 以获取更多信息。

rcu_dereference() 原语被用于
各种 "_rcu()" 列表遍历原语，例如
与 list_for_each_entry_rcu() 相同。请注意，它
完全合法（尽管多余）的更新端代码
使用 rcu_dereference() 和 "_rcu()" 列表遍历
primitives. 这在代码中特别有用
对读者和更新者来说是常见的。然而，lockdep
如果在外部访问 rcu_dereference()，将会报错
RCU读端临界区的。参见lockdep.rst
去了解该如何处理此事。

当然，无论是 rcu_dereference() 还是 "_rcu()"
列表遍历原语可以替代一个良好的
在多个更新器之间进行协调的并发设计。

b. 如果正在使用列表宏，则使用 list_add_tail_rcu()
并且必须使用 list_add_rcu() 原语来确保顺序
防止弱序机器发生顺序错乱
结构初始化与指针赋值。
同样，如果正在使用 hlist 宏，则
需要使用 hlist_add_head_rcu() 原语。

c. 如果正在使用列表宏，则调用 list_del_rcu()
必须使用 primitive 来保持 list_del() 的指针
对并发操作造成毒性影响而引发的中毒
读者也是如此。同样，如果正在使用 hlist 宏，
需要使用 hlist_del_rcu() 原语。

list_replace_rcu() 和 hlist_replace_rcu() 原语
可用于用新结构替换旧结构
在各自类型的RCU保护列表中。

d. 类似于 (4b) 和 (4c) 的规则也适用于 "hlist_nulls"
RCU保护的链表类型。

e. 更新必须确保对给定内容的初始化
结构体的定义必须在其指针使用之前完成
公开化。使用 rcu_assign_pointer() 原语
当公开指向一个结构体的指针时
可以被RCU读端临界区遍历。

5.  如果调用了 call_rcu()、call_srcu()、call_rcu_tasks()、call_rcu_tasks_rude() 或 call_rcu_tasks_trace()，则
当使用 call_rcu_tasks()、call_rcu_tasks_rude() 或 call_rcu_tasks_trace() 时，回调函数可能会
在软中断上下文中调用，并且无论如何都在下半部执行，包括任何类型的中断请求（irq）上下文。该规则同样适用于
已禁用。特别是，此回调函数不能阻塞。
如果需要回调函数阻塞，请在工作队列中运行该代码
从回调中调度的处理程序。队列中的 rcu_work()
在调用 call_rcu() 的情况下，该函数会为你完成此操作。

6. 由于 synchronize_rcu() 可能会阻塞，因此不能在无法睡眠的上下文中调用
从任何类型的中断请求（irq）上下文中。同样的规则适用
用于 synchronize_srcu()、synchronize_rcu_expedited()、
synchronize_srcu_expedited()、synchronize_rcu_tasks()、synchronize_rcu_tasks_rude() 和 synchronize_rcu_tasks_trace()。
synchronize_rcu_tasks_rude() 和 synchronize_rcu_tasks_trace()。

这些原语的加速形式具有相同的语义
与非加速版本相同，但加速过程对CPU的消耗更大。
应将快速原语的使用限制在极少数情况下
通常不会的配置更改操作
在实时工作负载运行时执行。请注意
对IPI敏感的实时工作负载可以使用 rcupdate.rcu_normal
完全禁用快速宽限期的内核启动参数
时期，尽管这可能会对性能产生影响。

特别是，如果你发现自己正在调用其中一个加速的
在循环中反复使用基本类型时，请大家行个方便：
重构你的代码，使其批量处理更新，从而允许
一个非加急的单一原语操作来覆盖整个批次。
这很可能比包含循环的速度更快
快速的原始方法，对其他部分来说会简单得多
系统的性能，尤其是对在系统上运行的实时工作负载
系统的其余部分。或者，可以改用异步方式
诸如 call_rcu() 之类的原语。

7. 从 v4.20 版本开始，一个给定的内核仅实现一种 RCU 风格，
是用于 PREEMPTION=n 的 RCU-sched 和用于 PREEMPTION=y 的 RCU-preempt。
如果更新器使用 call_rcu() 或 synchronize_rcu()，那么
相应的读者可以使用：(1) rcu_read_lock() 和
rcu_read_unlock()，以及任何一对禁用的原语
并重新启用软中断，例如 rcu_read_lock_bh() 和
rcu_read_unlock_bh()，或 (3) 任何一对禁用的原语
并重新启用抢占，例如 rcu_read_lock_sched() 和
rcu_read_unlock_sched()。如果更新者使用 synchronize_srcu()
或调用 call_srcu()，则相应的读取者必须使用
srcu_read_lock() 和 srcu_read_unlock()，并且具有相同的
srcu_struct。关于快速RCU宽限期等待的规则
基本类型与其非加急版本相同。

同样，必须正确使用RCU Tasks的变体：

a. 如果更新器使用 synchronize_rcu_tasks() 或
调用 call_rcu_tasks() 后，读者必须避免
执行自愿的上下文切换，即从
阻塞。

b. 如果更新器使用 call_rcu_tasks_trace()
或 synchronize_rcu_tasks_trace()，然后的
相应的读者必须使用 rcu_read_lock_trace()
以及 rcu_read_unlock_trace()。

c. 如果更新者使用 synchronize_rcu_tasks_rude()，
那么相应的读者必须使用任何内容
禁用抢占，例如 preempt_disable()
并启用抢占。

混合在一起会导致混乱和内核损坏，
甚至导致了一个可被利用的安全问题。因此，
使用不明显的原语组合时，应添加注释
当然必不可少。一个不太明显的搭配示例如下：
网络中的XDP功能，该功能从
网络驱动 NAPI（softirq）上下文。BPF 高度依赖 RCU
对其数据结构的保护，但由于BPF程序
调用完全在单个 local_bh_disable() 内发生
在 NAPI 轮询周期的某个阶段，这种用法是安全的。原因在于
这种用法是安全的，因为读者可以使用任何东西
当更新器使用 call_rcu() 或 synchronize_rcu() 时禁用 BH。

8. 尽管 synchronize_rcu() 比 call_rcu() 更慢，
通常会使代码更简洁。因此，除非需要更新
性能至关重要，更新器不能阻塞，
或 synchronize_rcu() 的延迟在用户空间是可见的，
应优先使用 synchronize_rcu() 而不是 call_rcu()。
此外，kfree_rcu() 和 kvfree_rcu() 通常会导致
比使用 synchronize_rcu() 的代码更简单
synchronize_rcu() 的多毫秒级延迟。因此请
kfree_rcu() 和 kvfree_rcu() 的“即发即忘”优势
在适用的情况下具备释放内存的功能。

同步原语 synchronize_rcu() 的一个特别重要的特性
primitive 是因为它会自动自我限制：如果宽限期
由于某种原因被延迟，那么 synchronize_rcu()
primitive 也会相应地延迟更新。相比之下，
使用 call_rcu() 的代码应明确限制更新速率
宽限期内延期的情况，因为未能这样做可能会
导致过高的实时延迟，甚至出现内存溢出（OOM）情况。

使用 call_rcu() 时获得这种自限特性的方法，
kfree_rcu() 或 kvfree_rcu() 包括：

a. 记录数据结构元素的数量
由RCU保护的数据结构使用，包括
那些等待宽限期结束的人。强制执行一项
对此数量的限制，必要时暂停更新以允许
之前延迟的释放操作完成。或者，
仅限制等待延迟释放的数量，而不是其他
元素的总数。

一种延缓更新的方法是获取更新端
互斥锁。（不要对自旋锁尝试此操作——其他CPU
锁上的旋转操作可能会阻止宽限期
（来自永无止境的循环。）另一种拖延更新的方法
用于更新时在外部使用一个包装函数
内存分配器，以便该包装函数
当有过多内存正在等待时，模拟内存不足（OOM）情况
RCU 宽限期。当然还有许多其他
这一主题的变体。

b. 限制更新速率。例如，如果更新仅在特定条件下发生
每小时一次，之后则不再进行明确的速率限制
必需的，除非你的系统已经严重损坏。
较旧版本的 dcache 子系统采用了这种方法，
用全局锁保护更新，限制其速率。

c. 可信更新——如果更新只能通过手动方式进行
超级用户或其他受信任的用户，那么可能不会
有必要自动限制它们。该理论
超级用户已经有多种方式会导致崩溃
机器。

d. 定期调用 rcu_barrier()，允许有限的
每个宽限期内的更新次数。

对 call_srcu()、call_rcu_tasks()、call_rcu_tasks_rude() 和 call_rcu_tasks_trace() 同样适用上述注意事项，其中：call_rcu_tasks() 要求读者不得阻塞（即不能发生上下文切换）；call_rcu_tasks_trace() 必须与 rcu_read_lock_trace() 配对使用；而 call_rcu_tasks_rude() 要求读者禁用抢占。
call_rcu_tasks_trace()。这就是为什么存在 srcu_barrier()、rcu_barrier_tasks() 和 rcu_barrier_tasks_trace() 的原因，分别对应 SRCU、Tasks RCU 和 Tracing Tasks RCU 的屏障机制。
rcu_barrier_tasks() 和 rcu_barrier_tasks_trace()，分别用于确保对应类型 RCU 的所有回调已完成。

请注意，尽管这些原语确实采取了措施以避免
当任一CPU的回调过多时导致内存耗尽，
一个意志坚定的用户或管理员仍然可能耗尽内存。
这在具有大量系统的场景中尤为如此
已将CPU配置为将其所有RCU回调卸载到
单个CPU，或者系统空闲内存相对较少时。

9. 所有RCU列表遍历原语，包括
rcu_dereference()、list_for_each_entry_rcu() 和
list_for_each_safe_rcu() 必须位于 RCU 读端临界区内
临界区或必须由适当的更新侧进行保护
锁。RCU 读端临界区由以下内容界定
rcu_read_lock() 和 rcu_read_unlock()，或类似的原语
例如 rcu_read_lock_bh() 和 rcu_read_unlock_bh()，其中
因此必须使用匹配的 rcu_dereference() 原语
为了使 lockdep 满意，在这种情况下，使用 rcu_dereference_bh()。

使用RCU列表遍历是允许的原因
在持有更新侧锁时使用原子操作的问题在于这样做
在减少代码冗余方面非常有帮助，特别是当存在重复代码时
在读者和更新者之间共享。其他原语
此情况下的相关说明已在 lockdep.rst 中讨论。

此规则的一个例外情况是，数据仅被添加时
链接的数据结构，并且在任何过程中都不会被移除
读者可能访问该结构的时间。在这种情况下
在某些情况下，可以使用 READ_ONCE() 来替代 rcu_dereference()
以及读端标记（rcu_read_lock() 和 rcu_read_unlock()，
例如）可以省略。

10. 相反，如果你处于RCU读端临界区中，
并且你没有持有适当的更新侧锁，你*必须*
使用列表宏的 "_rcu()" 变体。如果未能这样做
会破坏Alpha，导致激进的编译器生成错误的代码，
并且会让人在试图理解你的代码时感到困惑。

11. 任何由RCU回调获取的锁，必须在其他地方获取
在禁用软中断的情况下，例如通过 spin_lock_bh()。未能
在获取该锁时禁用软中断将导致
一旦RCU软中断处理程序恰好运行，就会立即陷入死锁
在中断该获取的关键阶段时调用你的RCU回调
章节。

12.  RCU 回调可以并且确实会并行执行。在许多情况下，
回调代码只是对 kfree() 的简单封装，以便
不是问题（或者更准确地说，即使是个问题，其程度也）
一个问题，内存分配器的锁会处理它）。然而，
如果回调函数要操作共享的数据结构，它们
必须使用所需的任何锁定或其他同步机制
安全地访问和/或修改该数据结构。

不要假设RCU回调将在相同的
执行相应 call_rcu()、call_srcu() 的 CPU
call_rcu_tasks() 或 call_rcu_tasks_trace()。例如，如果
当某个CPU在仍有待处理的RCU回调时进入离线状态，
那么该RCU回调将在某个仍在运行的CPU上执行。
（如果情况并非如此，一个自生成的RCU回调将
防止受害CPU进入离线状态。此外，
由 rcu_nocbs= 指定的 CPU 可能*始终*处于
RCU回调实际上在其他一些CPU上执行
实时工作负载，这正是使用它的全部意义所在
`rcu_nocbs=` 是内核启动参数。

此外，不要假设按特定顺序排队的回调函数
将按该顺序被调用，即使它们全部被加入队列。
相同的CPU。此外，请勿假设相同CPU的回调将
被顺序调用。例如，在近期的内核中，CPU 可以
在卸载和重新加载回调调用之间切换，
并且当某个CPU正在进行此类切换时，其回调函数
可能会被该CPU的软中断处理程序并发调用
该CPU的rcuo内核线程。此时，该CPU的回调函数
可能会被并发执行且执行顺序不确定。

13. 与大多数RCU变体不同，*允许*在其中阻塞
SRCU 读端临界区（由 srcu_read_lock() 标记）
以及 srcu_read_unlock())，因此称为“SRCU”：“可睡眠的 RCU”。
与RCU一样，guard(srcu)() 和 scoped_guard(srcu) 的形式是
可获得，且通常更易于使用。请注意
如果你不需要在读端临界区中睡眠，
你应该使用RCU而不是SRCU，因为RCU几乎
总是比SRCU更快、更易于使用。

也与其他形式的RCU不同，需要显式初始化和
清理工作需要在构建时通过 DEFINE_SRCU() 完成
或使用 DEFINE_STATIC_SRCU()，或在运行时通过 init_srcu_struct()
以及 cleanup_srcu_struct()。最后这两个函数会传入一个
"struct srcu_struct" 定义了给定范围的
SRCU 域。一旦初始化，srcu_struct 将被传递
到 srcu_read_lock()、srcu_read_unlock() 和 synchronize_srcu()，
synchronize_srcu_expedited() 和 call_srcu()。给定的
synchronize_srcu() 仅等待 SRCU 读端临界区完成
由 srcu_read_lock() 和 srcu_read_unlock() 保护的代码段
已传递相同 srcu_struct 的调用。此属性
是让睡眠状态下的读侧临界区变得可接受的原因——
给定的子系统仅延迟自身的更新，而不影响其他子系统的更新
使用SRCU的子系统。因此，SRCU不太容易发生OOM
比RCU的读端临界区更高效的系统
被允许睡觉。

在读端临界区中睡眠的能力并不存在
免费获取。首先，对应的 srcu_read_lock() 和
srcu_read_unlock() 调用必须传入相同的 srcu_struct。
其次，宽限期检测的开销仅被分摊
在共享给定 srcu_struct 的那些更新之上，而不是
在全球范围内进行摊销，就像其他形式的RCU一样。
因此，应优先使用 SRCU 而不是读写信号量（rw_semaphore）
仅在极端的读取密集型情况下，或在以下情况下
需要SRCU读端的死锁免疫或低读端开销
实时延迟。你还应考虑 percpu_rw_semaphore
当你需要轻量级阅读器时。

SRCU的快速原语（synchronize_srcu_expedited()）
从不向其他CPU发送IPI，因此对系统更友好
比 synchronize_rcu_expedited() 更适合实时工作负载。

RCU Tasks Trace读侧也允许睡眠
临界区，由 rcu_read_lock_trace() 分隔
以及 rcu_read_unlock_trace()。然而，这是一个专门化的
RCU的一种变体，使用前应先进行检查
与其当前用户。在大多数情况下，你应该
使用 SRCU。与 RCU 和 SRCU 一样，使用 guard(rcu_tasks_trace)() 和
scoped_guard(rcu_tasks_trace) 可用，并且通常提供
使用更加简便。

请注意，rcu_assign_pointer() 与 SRCU 的关系与其和其它 RCU 的关系相同。
其他形式的RCU，但你应该使用rcu_dereference()
使用 srcu_dereference() 以避免 lockdep 警告。

14.  call_rcu()、synchronize_rcu() 及其相关函数的全部意义
是等待所有已存在的读取者完成之后
执行某些原本具有破坏性的操作。它是
因此，首先移除任何路径至关重要
读者可以关注的可能受到影响的内容
破坏性操作，然后*才*调用 call_rcu()
synchronize_rcu()，或其他类似函数。

因为这些原语仅等待已存在的读取器，
是调用者的责任，以确保任何后续操作
读者将安全执行。

15. 各种RCU读端原语不一定包含
内存屏障。因此，你应该为CPU做好规划
并且允许编译器自由地将代码重排到RCU内外
读端临界区。这是由
RCU 更新侧原语用于处理此问题。

对于SRCU读者，可以使用smp_mb__after_srcu_read_unlock()
在调用 srcu_read_unlock() 后立即执行以获得完整的内存屏障。

16. 使用 CONFIG_PROVE_LOCKING、CONFIG_DEBUG_OBJECTS_RCU_HEAD 和
__rcu 稀疏检查以验证你的 RCU 代码。这些检查可以帮助
发现以下问题：

CONFIG_PROVE_LOCKING:
检查对RCU保护的数据结构的访问
在适当的RCU读端临界区下执行
在持有正确锁组合的同时，该部分
或其他适当的条件。

CONFIG_DEBUG_OBJECTS_RCU_HEAD:
确保不要将同一个对象传递给 call_rcu()
（或朋友）在RCU宽限期结束之前
自上次你将同一对象传递以来
call_rcu()（或类似函数）。

CONFIG_RCU_STRICT_GRACE_PERIOD:
结合 KASAN 来检查是否有指针泄露出去
RCU读端临界区。此Kconfig
该选项在性能和可扩展性方面都存在较大挑战。
因此仅限于四CPU系统。

__rcu 稀疏检查：
标记指向RCU保护数据结构的指针
使用 __rcu，并且如果你访问它，sparse 会发出警告
没有使用任一变体服务的指针
rcu_dereference() 的。

这些调试辅助工具可以帮助你发现以下问题：
否则极难发现。

17.  如果你传递了一个在模块内定义的回调函数
调用 call_rcu()、call_srcu() 或 call_rcu_tasks() 之一，
call_rcu_tasks_trace()，那么就需要等待所有
等待在卸载该模块之前被调用的回调函数。
请注意，等待宽限期是绝对*不够*的
周期！例如，synchronize_rcu() 的实现就*不是*
保证等待通过其他CPU注册的回调函数
call_rcu()。或者，如果当前CPU最近...
已离线并重新上线。

你需要使用其中一个屏障函数：

- call_rcu() -> rcu_barrier()
- call_srcu() -> srcu_barrier()
- call_rcu_tasks() -> rcu_barrier_tasks()
- call_rcu_tasks_rude() -> rcu_barrier_tasks_rude()
- call_rcu_tasks_trace() -> rcu_barrier_tasks_trace()

此外，kfree_rcu() 和 kvfree_rcu() 提供“即发即忘”（fire-and-forget）的释放语义，具有更低的延迟开销。
- call_rcu_tasks_trace() -> rcu_barrier_tasks_trace()

然而，这些屏障函数绝对*不*被保证
等待一个宽限期。例如，如果没有
系统中任何位置排队的 call_rcu() 回调函数，rcu_barrier()
可以并且会立即返回。

因此，如果你需要同时等待一个宽限期以及所有其他条件
已存在的回调函数，您需要同时调用这两个函数。
根据RCU的类型，具体配对方式有所不同：

- 要么使用 synchronize_rcu()，要么使用 synchronize_rcu_expedited()，
与 rcu_barrier() 一起使用
- 要么使用 synchronize_srcu()，要么使用 synchronize_srcu_expedited()，
与 srcu_barrier() 一起使用
- synchronize_rcu_tasks() 和 rcu_barrier_tasks()
- synchronize_tasks_trace() 和 rcu_barrier_tasks_trace()

如有必要，你可以使用类似工作队列（workqueues）的方式来执行
所需的一对函数同时执行。

有关更多信息，请参见 rcubarrier.rst。


==================================================

由 Qwen-plus 及 LT agent 翻译