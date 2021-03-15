---
title: SIGKILL Memory Reclaim Expedite
date: 2021-01-26
---

## 方案一设计(Design Proposal v1)

Andorid系统在杀死进程并回收内存时，并不是即时操作，同时还要保证可用内存保留在一个范围内防止在回收进程内存时系统性能下降。假如进程退出前在不间断的睡眠中阻塞或者由于其他一些操作耗时较长，此时的内存回收时间可能会不可估计。

通过提高内存回收的速率可以解决上述问题。提出一个方案，修改kill_pid_info的流程，在对pending的状态发送SIGKILL信号后，立即将task_struct标记为受害者，同时将该task_struct挂到内核线程oom_reaper的链表上，之后oom_reaper会接管该进程的vma，隔离内存执行内存回收。使用oom_reaper线程处理victim进程时，即使它无法运行也能执行回收操作。

![](/images/sigkill_memory_reclaim_expedite/top.jpg)

结果调查，Android lowmemorykiller由于内核版本的不同，通常会选择不同的策略来实施lowmemorykiller。无论哪种措施，都要对系统中的进程进行一系列的优先级判断，然后根据设定好的处理逻辑执行kill操作。

通过init进程lmkd执行杀死进程时，通过系统调用kill对指定要杀的进程pid发送信号，通过驱动lowmemorykiller执行杀死进程时，通过内核函数send_sig传输task_struct和SIGKILL信号。可以看出，无论系统采用哪种方式执行LMK，都会将SIGKILL和对应的进程信息发送给内核的信号模块。

9652机芯采用的内核版本为4.9，其Android lowmemorykiller的方案为通过lowmemorykiller驱动处理进程。原始的send_sig接口需要传入task_struct，同时比系统调用kill更快的进入signal模块，而oom_reaper需要访问mm_struct的mmap_sem，而在signal模块中对mm_struct进行处理可能会导致信号阻塞，引起其他的风险。

因此本方案修改了lowmemorykiller驱动中SIGKILL信号的发送方式，使用内核函数kill_pid替代。这样就能够在信号发送之后再对进程的task_struct进行处理，降低了持锁的风险。但是如果受害者进程正在进行coredump时，可能会导致coredump的数据缺失。

![](/images/sigkill_memory_reclaim_expedite/trace.jpg)

由于lowmemorykiller触发具有不确定性，需要判断bad进程来进行kill，所以对于新方案的性能测试需要从两方面入手。一方面是在lowmemorykiller触发的场景下查看匿名页和文件页在开启和关闭优化下的变动情况，以及lowmemorykiller触发时的回收内存的大小和kswapd进程的资源占用。另一方面是测量kill进程时发送信号到进程退出经过的时间，通过执行多次kill操作采集样本分析数据。

所需开启的trace event为:sched:sched_process_exit、sched:sched_switch、signal:signal_generate、kmem:rss_stat。其中kmem:rss_stat用来查看对应pid对内存的操作，查看mm_struct中4类内存的变化信息。signal:signal_generate用来捕获信号9，确定\_\_signal_send 调用的时机。sched:sched_process_exit和sched:sched_switch分别对进程退出do_exit和do_task_dead时候的时间点进行捕获。

匿名页与文件页使用情况变化:

|                                      | 优化功能关闭    | 优化功能开启    |
|--------------------------------------|-----------|-----------|
| Active(anon) + Active(file) 增加的值     | 11933372B | 11307024B |
| Active(anon) + Active(file) 减少的值     | 11400280B | 10529300B |
| Inactive(anon) + Inactive(file) 增加的值 | 40040B    | 711944B   |
| Inactive(anon) + Inactive(file) 减少的值 | 136656B   | 742036B   |

优化功能对进程消亡时间的影响:

|                                         | 优化功能关闭      | 优化功能开启      | 提升倍数  |
|-----------------------------------------|-------------|-------------|-------|
| signal_generate到sched_process_exit平均时间差 | 0.01505937s | 0.00110683s | 13.61 |
| signal_generate到sched_switch平均时间差       | 0.02152393s | 0.00781319s | 2.75  |

根据测试结果，当优化功能开启后，对于Inactive的内存空间的处理吞吐量大大增大。另外，以\_\_send_signal到\_\_schedule(prev_state=x)为采集时间段比较开关优化功能时的耗时，可看出在未开启优化耗时是开启优化耗时的2.75倍。

## 方案二设计(Design Proposal v2)

在oom_reaper线程中，有一个值MAX_OOM_REAP_RETRIES，用来控制oom_reaper的重试次数，其中oom_reaper只使用一个try_lock，并会在短暂的睡眠时间尝试10(MAX_OOM_REAP_RETRIES)次。但假如mm_struct中的读写信号量mmap_sem被阻止时，由于采取的策略为try_lock，可能会出现锁竞争的情况，降低内存回收的效率。另外如果产生mmap_sem的争用也会导致lowmemorykiller延迟的进一步扩大。

该方案不再通过oom_reaper线程执行内存回收，而是在发送SIGKILL信号的进程的上下文中进行内存回收。

![](/images/sigkill_memory_reclaim_expedite/top2.jpg)

通过对系统节点/sys/kernel/lmk_sigkill/lmk_sigkill_on进行写入(0、1、2)可对优化功能进行开启与关闭，方案二兼容方案一的优化。

优化功能对应标志:

| 序号 | 优化功能                  | tcl_lmk_sigkill.h            | oom.h(enum sigkill_state) |
|----|-----------------------|------------------------------|---------------------------|
| 0  | 关闭                    | LMK_SIGKILL_CTRL_OFF         | SIGKILL_STATE_OFF         |
| 1  | 方案一(expedite_reclaim) | LMK_SIGKILL_CTRL_EXPEDITE_ON | SIGKILL_STATE_EXPEDITE    |
| 2  | 方案二(reap_mm)          | LMK_SIGKILL_CTRL_REAP_ON     | SIGKILL_STATE_REAP        |

![](/images/sigkill_memory_reclaim_expedite/reap_mm.jpg)

优化功能对进程消亡时间的影响:

|                                         | 优化功能关闭      | 优化功能开启      | 提升倍数 |
|-----------------------------------------|-------------|-------------|------|
| signal_generate到sched_process_exit平均时间差 | 0.01505937s | 0.00360875s | 4.17 |
| signal_generate到sched_switch平均时间差       | 0.02152393s | 0.02127370s | 1.01 |

优化方案二与未开启优化方案时相比较具有明显的提升，但是相比较与优化方案一的测试结果，优化方案二的测试结果没有优化方案一的提升倍数大，但是避免了之前内核线程oom_reaper处理任务超时的影响，不在内核线程中进行匿名内存和swap内存的回收，而是在对victim进程发出SIGKILL信号的进程的上下文中进行匿名内存、非VM_SHARED内存和swap内存(check_swap_entry)的回收。