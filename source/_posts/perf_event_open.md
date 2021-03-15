---
title: "Interface: perf_event_open"
date: 2020-11-27
---

pmu的结构
```
/**
* struct pmu - generic performance monitoring unit
*/
struct pmu {
   struct list_head        entry;


   struct module            *module;
   struct device            *dev;
   const struct attribute_group    **attr_groups;
   const char            *name;
   int                type;


   /*
    * various common per-pmu feature flags
    */
   int                capabilities;


   int * __percpu            pmu_disable_count;
   struct perf_cpu_context * __percpu pmu_cpu_context;
   atomic_t            exclusive_cnt; /* < 0: cpu; > 0: tsk */
   int                task_ctx_nr;
   int                hrtimer_interval_ms;


   /* number of address filters this PMU can do */
   unsigned int            nr_addr_filters;


   /*
    * Fully disable/enable this PMU, can be used to protect from the PMI
    * as well as for lazy/batch writing of the MSRs.
    */
   void (*pmu_enable)        (struct pmu *pmu); /* optional */
   void (*pmu_disable)        (struct pmu *pmu); /* optional */


   /*
    * Try and initialize the event for this PMU.
    *
    * Returns:
    *  -ENOENT    -- @event is not for this PMU
    *
    *  -ENODEV    -- @event is for this PMU but PMU not present
    *  -EBUSY    -- @event is for this PMU but PMU temporarily unavailable
    *  -EINVAL    -- @event is for this PMU but @event is not valid
    *  -EOPNOTSUPP -- @event is for this PMU, @event is valid, but not supported
    *  -EACCESS    -- @event is for this PMU, @event is valid, but no privilidges
    *
    *  0        -- @event is for this PMU and valid
    *
    * Other error return values are allowed.
    */
   int (*event_init)        (struct perf_event *event);


   /*
    * Notification that the event was mapped or unmapped.  Called
    * in the context of the mapping task.
    */
   void (*event_mapped)        (struct perf_event *event, struct mm_struct *mm); /* optional */
   void (*event_unmapped)        (struct perf_event *event, struct mm_struct *mm); /* optional */


   /*
    * Flags for ->add()/->del()/ ->start()/->stop(). There are
    * matching hw_perf_event::state flags.
    */
#define PERF_EF_START    0x01        /* start the counter when adding    */
#define PERF_EF_RELOAD    0x02        /* reload the counter when starting */
#define PERF_EF_UPDATE    0x04        /* update the counter when stopping */

   /*
    * Adds/Removes a counter to/from the PMU, can be done inside a
    * transaction, see the ->*_txn() methods.
    *
    * The add/del callbacks will reserve all hardware resources required
    * to service the event, this includes any counter constraint
    * scheduling etc.
    *
    * Called with IRQs disabled and the PMU disabled on the CPU the event
    * is on.
    *
    * ->add() called without PERF_EF_START should result in the same state
    *  as ->add() followed by ->stop().
    *
    * ->del() must always PERF_EF_UPDATE stop an event. If it calls
    *  ->stop() that must deal with already being stopped without
    *  PERF_EF_UPDATE.
    */
   int  (*add)            (struct perf_event *event, int flags);
   void (*del)            (struct perf_event *event, int flags);


   /*
    * Starts/Stops a counter present on the PMU.
    *
    * The PMI handler should stop the counter when perf_event_overflow()
    * returns !0. ->start() will be used to continue.
    *
    * Also used to change the sample period.
    *
    * Called with IRQs disabled and the PMU disabled on the CPU the event
    * is on -- will be called from NMI context with the PMU generates
    * NMIs.
    *
    * ->stop() with PERF_EF_UPDATE will read the counter and update
    *  period/count values like ->read() would.
    *
    * ->start() with PERF_EF_RELOAD will reprogram the the counter
    *  value, must be preceded by a ->stop() with PERF_EF_UPDATE.
    */
   void (*start)            (struct perf_event *event, int flags);
   void (*stop)            (struct perf_event *event, int flags);


   /*
    * Updates the counter value of the event.
    *
    * For sampling capable PMUs this will also update the software period
    * hw_perf_event::period_left field.
    */
   void (*read)            (struct perf_event *event);


   /*
    * Group events scheduling is treated as a transaction, add
    * group events as a whole and perform one schedulability test.
    * If the test fails, roll back the whole group
    *
    * Start the transaction, after this ->add() doesn't need to
    * do schedulability tests.
    *
    * Optional.
    */
   void (*start_txn)        (struct pmu *pmu, unsigned int txn_flags);
   /*
    * If ->start_txn() disabled the ->add() schedulability test
    * then ->commit_txn() is required to perform one. On success
    * the transaction is closed. On error the transaction is kept
    * open until ->cancel_txn() is called.
    *
    * Optional.
    */
   int  (*commit_txn)        (struct pmu *pmu);
   /*
    * Will cancel the transaction, assumes ->del() is called
    * for each successful ->add() during the transaction.
    *
    * Optional.
    */
   void (*cancel_txn)        (struct pmu *pmu);


   /*
    * Will return the value for perf_event_mmap_page::index for this event,
    * if no implementation is provided it will default to: event->hw.idx + 1.
    */
   int (*event_idx)        (struct perf_event *event); /*optional */


   /*
    * context-switches callback
    */
   void (*sched_task)        (struct perf_event_context *ctx,
                   bool sched_in);
   /*
    * PMU specific data size
    */
   size_t                task_ctx_size;




   /*
    * Set up pmu-private data structures for an AUX area
    */
   void *(*setup_aux)        (int cpu, void **pages,
                    int nr_pages, bool overwrite);
                   /* optional */


   /*
    * Free pmu-private AUX data structures
    */
   void (*free_aux)        (void *aux); /* optional */

   /*
    * Validate address range filters: make sure the HW supports the
    * requested configuration and number of filters; return 0 if the
    * supplied filters are valid, -errno otherwise.
    *
    * Runs in the context of the ioctl()ing process and is not serialized
    * with the rest of the PMU callbacks.
    */
   int (*addr_filters_validate)    (struct list_head *filters);
                   /* optional */


   /*
    * Synchronize address range filter configuration:
    * translate hw-agnostic filters into hardware configuration in
    * event::hw::addr_filters.
    *
    * Runs as a part of filter sync sequence that is done in ->start()
    * callback by calling perf_event_addr_filters_sync().
    *
    * May (and should) traverse event::addr_filters::list, for which its
    * caller provides necessary serialization.
    */
   void (*addr_filters_sync)    (struct perf_event *event);
                   /* optional */


   /*
    * Filter events for PMU-specific reasons.
    */
   int (*filter_match)        (struct perf_event *event); /* optional */
};
```
&nbsp;
perf_event_open
```
/**
* sys_perf_event_open - open a performance event, associate it to a task/cpu
*
* @attr_uptr:    event_id type attributes for monitoring/sampling
* @pid:        target pid
* @cpu:        target cpu
* @group_fd:        group leader event fd
*/

event = perf_event_alloc(&attr, cpu, task, group_leader, NULL,
                NULL, NULL, cgroup_fd);
```
&nbsp;
perf_event_open -> perf_event_alloc
```
/*
* Allocate and initialize a event structure
*/
```
&nbsp;
perf_event的定义
```
/**
* struct perf_event - performance event kernel representation:
*/
```
&nbsp;
在perf_event_alloc中
```
struct pmu *pmu;
event->pmu = NULL;
pmu = NULL;
pmu = perf_init_event(event);
```
&nbsp;
perf_event_alloc -> perf_init_event
```
/* Try parent's PMU first: */
if (event->parent && event->parent->pmu)
```
如果perf_event存在父类perf_event且有注册的pmu，就直接拿来用
```
pmu = event->parent->pmu;
ret = perf_try_init_event(pmu, event);
```
如果没有的话，要先根据event的种类找一个pmu注册的位置
```
pmu = idr_find(&pmu_idr, event->attr.type);


struct perf_event_attr attr;


/*
* Major type: hardware/software/tracepoint/etc.
*/
__u32 type;
```
&nbsp;
idr_find的定义
```
/**
* idr_find - return pointer for given id
* @idr: idr handle
* @id: lookup key
*
* Return the pointer given the id it has been registered with.  A %NULL
* return indicates that @id is not valid or you passed %NULL in
* idr_get_new().
*
* This function can be called under rcu_read_lock(), given that the leaf
* pointers lifetimes are correctly managed.
*/
```
&nbsp;
perf_init_event -> perf_try_init_event
```
event->pmu = pmu;
ret = pmu->event_init(event);
```
调用pmu->event_init
&nbsp;
在core.c中，pmu是这样注册的
```
static struct pmu perf_tracepoint = {
   .task_ctx_nr    = perf_sw_context,


   .event_init    = perf_tp_event_init,
   .add        = perf_trace_add,
   .del        = perf_trace_del,
   .start        = perf_swevent_start,
   .stop        = perf_swevent_stop,
   .read        = perf_swevent_read,
};
```
&nbsp;
event_init -> perf_tp_event_init
```
err = perf_trace_init(event);
event->destroy = tp_perf_event_destroy;
```
&nbsp;
perf_tp_event_init -> perf_trace_init
对所有的ftrace_events尝试就行初始化注册
&nbsp;
perf_trace_init -> perf_trace_event_init
```
static int perf_trace_event_init(struct trace_event_call *tp_event,
                struct perf_event *p_event)
{
   int ret;

   ret = perf_trace_event_perm(tp_event, p_event);
   if (ret)
       return ret;

   ret = perf_trace_event_reg(tp_event, p_event);
   if (ret)
       return ret;

   ret = perf_trace_event_open(p_event);
   if (ret) {
       perf_trace_event_unreg(p_event);
       return ret;
   }

   return 0;
}
```
perf_trace_event_perm用来就行权限判断
&nbsp;
perf_trace_event_init -> perf_trace_event_reg
初始化trace_event_call上per_cpu的perf_event挂载链表：tp_event->perf_events 如果perf_event需要接收tracepoint的数据，需要按绑定的cpu挂载到对应的per_cpu链表上

```
   for_each_possible_cpu(cpu)
       INIT_HLIST_HEAD(per_cpu_ptr(list, cpu));
   tp_event->perf_events = list;
```
通过tp_event->class->reg对event就行注册
```
ret = tp_event->class->reg(tp_event, TRACE_REG_PERF_REGISTER, NULL);
```

实际上，在对event_class定义时，trace_event_class的reg函数指针就注册为trace_event_reg
```
static struct trace_event_class __used __refdata event_class_##call = { \
   .system            = TRACE_SYSTEM_STRING,            \
   .define_fields        = trace_event_define_fields_##call,    \
   .fields            = LIST_HEAD_INIT(event_class_##call.fields),\
   .raw_init        = trace_event_raw_init,            \
   .probe            = trace_event_raw_event_##call,        \
   .reg            = trace_event_reg,            \
   _TRACE_PERF_INIT(call)                        \
};
```
&nbsp;
reg -> trace_event_reg
此时传入的type为TRACE_REG_PERF_REGISTER，将perf_probe探针挂到tracepoint上
```
case TRACE_REG_PERF_REGISTER:
   return tracepoint_probe_register(call->tp, call->class->perf_probe, call);
```
&nbsp;
trace_event_reg -> tracepoint_probe_register
```
/**
* tracepoint_probe_register -  Connect a probe to a tracepoint
* @tp: tracepoint
* @probe: probe handler
* @data: tracepoint data
* @prio: priority of this function over other registered functions
*
* Returns 0 if ok, error value on error.
* Note: if @tp is within a module, the caller is responsible for
* unregistering the probe before the module is gone. This can be
* performed either with a tracepoint module going notifier, or from
* within module exit functions.
*/
```
&nbsp;
tracepoint_probe_register -> tracepoint_probe_register_prio
```
/**
* tracepoint_probe_register -  Connect a probe to a tracepoint
* @tp: tracepoint
* @probe: probe handler
* @data: tracepoint data
* @prio: priority of this function over other registered functions
*
* Returns 0 if ok, error value on error.
* Note: if @tp is within a module, the caller is responsible for
* unregistering the probe before the module is gone. This can be
* performed either with a tracepoint module going notifier, or from
* within module exit functions.
*/
```
&nbsp;
tracepoint_probe_register_prio -> tracepoint_add_func
```
/*
* Add the probe function to a tracepoint.
*/
```
&nbsp;
此时perf_event的回调函数注册到对应的tracepoint上。
在使用perf tracepoint event时，随着task的调度，perf才能record每个tracepoint对应的perf_event的启动与停止。
所以需要pmu->add()来进行数据的采集。
从perf_event_task_sched_in开始
perf_event_task_sched_in -> \_\_perf_event_task_sched_in
```
/*
* Called from scheduler to add the events of the current task
* with interrupts disabled.
*
* We restore the event value and then enable it.
*
* This does not protect us against NMI, but enable()
* sets the enabled bit in the control field of event _before_
* accessing the event control register. If a NMI hits, then it will
* keep the event running.
*/
```
&nbsp;
\_\_perf_event_task_sched_in -> perf_event_context_sched_in -> perf_event_sched_in -> ctx_sched_in
先对高优先级的ctx进行调度
```
is_active ^= ctx->is_active   
/*
* First go through the list and put on any pinned groups
* in order to give them the best chance of going on.
*/
if (is_active & EVENT_PINNED)
   ctx_pinned_sched_in(ctx, cpuctx);
/* Then walk through the lower prio flexible groups */
if (is_active & EVENT_FLEXIBLE)
   ctx_flexible_sched_in(ctx, cpuctx);
```
&nbsp;
ctx_sched_in -> ctx_pinned_sched_in/ctx_flexible_sched_in -> group_sched_in -> event_sched_in
```
perf_pmu_disable(event->pmu);
if (event->pmu->add(event, PERF_EF_START));
perf_pmu_enable(event->pmu);
```
&nbsp;
add -> perf_trace_add
```
list = this_cpu_ptr(pcpu_list);
hlist_add_head_rcu(&p_event->hlist_entry, list);
return tp_event->class->reg(tp_event, TRACE_REG_PERF_ADD, p_event);
```
将perf_event加入到tp_event->perf_events的当前cpu链表中
reg -> trace_event_reg
```
case TRACE_REG_PERF_OPEN:
case TRACE_REG_PERF_CLOSE:
case TRACE_REG_PERF_ADD:
case TRACE_REG_PERF_DEL:
   return 0;
```
&nbsp;
在DECLARE_EVENT_CLASS时候，已经通过_TRACE_PERF_PROTO(call, PARAMS(proto))和_TRACE_PERF_INIT(call)定义了perf_probe
```
#define _TRACE_PERF_PROTO(call, proto)                    \
   static notrace void                        \
   perf_trace_##call(void *__data, proto);

#define _TRACE_PERF_INIT(call)                        \
   .perf_probe = perf_trace_##call,
```
&nbsp;
在perf_trace\_##call中，perf_trace_run_bpf_submit()负责提交数据给this_cpu_ptr(event_call->perf_events)链表上等待的perf_event
perf_trace_run_bpf_submit -> perf_tp_event
raw data结构
```
struct perf_raw_record raw = {
   .frag = {
       .size = entry_size,
       .data = record,
   },
};
```
把sample数据逐个发送给当前cpu perf_events链表上链接的perf_event
如果指定了目标task，还要迭代它的上下文将其数据传送到task上绑定的perf_event
&nbsp;
perf_tp_event -> perf_swevent_event
perf_swevent_event通过local64_add增加event的count
判断是否是period模式，根据其设置数据进行上报
&nbsp;
perf_swevent_event -> perf_swevent_overflow -> \_\_perf_event_overflow
```
READ_ONCE(event->overflow_handler)(event, data, regs);
```
&nbsp;
在perf_event_alloc中
```
if (overflow_handler) {
    event->overflow_handler    = overflow_handler;
    event->overflow_handler_context = context;
} else if (is_write_backward(event)){
    event->overflow_handler = perf_event_output_backward;
    event->overflow_handler_context = NULL;
} else {
    event->overflow_handler = perf_event_output_forward;
    event->overflow_handler_context = NULL;
}
```
&nbsp;
overflow_handler -> perf_event_output_forward/perf_event_output_backward -> \_\_perf_event_output -> perf_output_sample
把数据输出到ringbuffer中，根据data->type来判断要输出那些信息