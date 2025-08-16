# 第15章：性能分析与调试

在现代 Linux 系统中，性能分析和调试是内核开发与系统优化的核心技能。本章深入探讨 Linux 内核提供的各种观测、分析和调试机制，从传统的 ftrace 到革命性的 eBPF，从静态跟踪点到动态探针，从崩溃转储到实时调试。我们将学习如何使用这些强大的工具来诊断性能瓶颈、追踪系统行为、定位内核错误，并最终实现系统的性能优化。

## 学习目标

完成本章学习后，您将能够：

1. **掌握内核跟踪机制**：理解 ftrace、kprobes、tracepoints 的原理与使用
2. **精通性能分析工具**：熟练使用 perf、火焰图等工具进行性能剖析
3. **掌握内核调试技术**：使用 KGDB、crash、kdump 调试内核问题
4. **运用动态追踪技术**：编写 eBPF/bpftrace 程序进行系统观测
5. **性能优化实践**：识别性能瓶颈并实施针对性优化

## 章节大纲

## 15.1 内核跟踪机制

Linux 内核跟踪机制提供了观测内核行为的窗口，让我们能够在不修改内核代码的情况下，深入了解系统的运行状态。从 2.6.27 版本引入 ftrace 开始，Linux 的可观测性得到了质的飞跃。

### 15.1.1 ftrace 框架

ftrace（function tracer）是 Linux 内核的官方跟踪框架，由 Steven Rostedt 开发。它不仅可以跟踪函数调用，还集成了各种跟踪器，成为内核调试和性能分析的瑞士军刀。

**架构设计**

ftrace 基于编译时插桩和运行时修改实现：

```
    用户空间接口 (/sys/kernel/debug/tracing/)
           |
    +------v------+
    | Trace Buffer |  <- 环形缓冲区（per-CPU）
    +------+------+
           |
    +------v------+
    |   Tracers   |  <- function, function_graph, wakeup, irqsoff
    +------+------+
           |
    +------v------+
    | Trace Events |  <- 静态跟踪点
    +------+------+
           |
    +------v------+
    |   Ftrace    |  <- 核心框架
    |   Core      |
    +------+------+
           |
    +------v------+
    | Mcount/Fentry| <- 函数入口钩子
    +--------------+
```

**函数跟踪原理**

GCC 编译器的 `-pg` 选项会在每个函数入口插入 mcount 调用（新版本使用 fentry）：

```c
/* 原始函数 */
void do_something(void) {
    /* function body */
}

/* 编译后（概念性表示） */
void do_something(void) {
    mcount();  /* 或 __fentry__() */
    /* function body */
}
```

ftrace 在运行时通过动态修改这些调用点来启用/禁用跟踪：

```c
/* kernel/trace/ftrace.c 简化版 */
static int ftrace_modify_code(unsigned long ip, unsigned long old_code,
                              unsigned long new_code)
{
    unsigned char replaced[MCOUNT_INSN_SIZE];
    
    /* 使用 text_poke 或类似机制修改代码 */
    if (probe_kernel_read(replaced, (void *)ip, MCOUNT_INSN_SIZE))
        return -EFAULT;
        
    if (memcmp(replaced, &old_code, MCOUNT_INSN_SIZE) != 0)
        return -EINVAL;
        
    /* 原子性地替换指令 */
    text_poke_bp((void *)ip, &new_code, MCOUNT_INSN_SIZE, NULL);
    
    return 0;
}
```

**使用示例**

```bash
# 启用函数跟踪
echo function > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 设置函数过滤
echo 'tcp_*' > /sys/kernel/debug/tracing/set_ftrace_filter

# 查看跟踪结果
cat /sys/kernel/debug/tracing/trace

# 函数调用图跟踪
echo function_graph > /sys/kernel/debug/tracing/current_tracer
```

### 15.1.2 kprobes 动态探针

kprobes 是 Linux 内核的动态探测机制，允许在几乎任意内核代码位置设置探针，无需重新编译内核。它通过替换指令、使用断点或跳转指令来实现。

**kprobes 家族**

1. **kprobes**：在指定地址设置探针
2. **kretprobes**：在函数返回时触发
3. **jprobes**：已废弃，被 kprobes 取代

**实现机制**

kprobes 的核心是指令替换和异常处理：

```c
/* 简化的 kprobe 结构 */
struct kprobe {
    struct hlist_node hlist;
    kprobe_opcode_t *addr;        /* 探测点地址 */
    kprobe_pre_handler_t pre_handler;
    kprobe_post_handler_t post_handler;
    kprobe_opcode_t opcode;       /* 保存的原始指令 */
    struct arch_specific_insn ainsn;
};

/* 注册 kprobe 的核心流程 */
int register_kprobe(struct kprobe *p)
{
    /* 1. 解析符号地址 */
    addr = kprobe_addr(p);
    
    /* 2. 准备探测点 */
    ret = prepare_kprobe(p);
    
    /* 3. 插入断点指令 (x86: int3) */
    arch_arm_kprobe(p);
    
    /* 4. 添加到哈希表 */
    hlist_add_head(&p->hlist, &kprobe_table[hash]);
    
    return 0;
}
```

**断点处理流程**

```
    执行到探测点
         |
         v
    触发 int3 异常
         |
         v
    do_int3() 处理器
         |
         v
    kprobe_int3_handler()
         |
    +----+----+
    |         |
    v         v
pre_handler  单步执行原指令
    |         |
    v         v
post_handler 返回正常执行
```

**使用示例**

```c
/* 内核模块中使用 kprobes */
static struct kprobe kp = {
    .symbol_name = "tcp_sendmsg",
};

static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    pr_info("tcp_sendmsg: sock=%p, len=%lu\n",
            (void *)regs->di, regs->si);
    return 0;
}

static int __init kprobe_init(void)
{
    kp.pre_handler = handler_pre;
    register_kprobe(&kp);
    return 0;
}
```

### 15.1.3 tracepoints 静态跟踪点

tracepoints 是内核中预定义的静态跟踪点，相比动态探针具有更高的稳定性和更低的开销。

**定义跟踪点**

```c
/* include/trace/events/sched.h */
TRACE_EVENT(sched_switch,
    TP_PROTO(struct task_struct *prev, struct task_struct *next),
    TP_ARGS(prev, next),
    
    TP_STRUCT__entry(
        __array(char, prev_comm, TASK_COMM_LEN)
        __field(pid_t, prev_pid)
        __field(int, prev_prio)
        __array(char, next_comm, TASK_COMM_LEN)
        __field(pid_t, next_pid)
        __field(int, next_prio)
    ),
    
    TP_fast_assign(
        memcpy(__entry->prev_comm, prev->comm, TASK_COMM_LEN);
        __entry->prev_pid = prev->pid;
        __entry->prev_prio = prev->prio;
        memcpy(__entry->next_comm, next->comm, TASK_COMM_LEN);
        __entry->next_pid = next->pid;
        __entry->next_prio = next->prio;
    ),
    
    TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d ==> "
              "next_comm=%s next_pid=%d next_prio=%d",
              __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,
              __entry->next_comm, __entry->next_pid, __entry->next_prio)
);
```

**跟踪点实现**

```c
/* 跟踪点的静态调用优化 */
#define DEFINE_TRACE(name)                              \
    struct tracepoint __tracepoint_##name = {           \
        .name = #name,                                  \
        .key = STATIC_KEY_INIT_FALSE,                   \
        .funcs = NULL,                                  \
    };                                                   \
                                                        \
    static inline void trace_##name(proto)              \
    {                                                    \
        if (static_key_false(&__tracepoint_##name.key)) \
            __DO_TRACE(&__tracepoint_##name, args);     \
    }
```

### 15.1.4 跟踪事件子系统

跟踪事件子系统统一了各种跟踪机制的接口，提供了标准化的事件定义和过滤机制。

**事件过滤器**

```bash
# 设置过滤条件
echo 'prev_prio < 100' > /sys/kernel/debug/tracing/events/sched/sched_switch/filter

# 复合条件
echo 'irq == 1 && ret >= 0' > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/filter
```

**触发器机制**

```bash
# 当满足条件时触发快照
echo 'snapshot if bytes_req > 4096' > \
    /sys/kernel/debug/tracing/events/kmem/kmalloc/trigger

# 触发栈追踪
echo 'stacktrace' > \
    /sys/kernel/debug/tracing/events/sched/sched_switch/trigger
```

## 15.2 性能计数器

Linux 性能计数器子系统（perf_events）提供了统一的接口来访问硬件性能计数器和软件事件，是现代性能分析的基石。

### 15.2.1 perf events 架构

perf_events 子系统由 Thomas Gleixner、Ingo Molnar 和 Arnaldo Carvalho de Melo 等人开发，从 2.6.31 版本开始成为内核的核心组件。

**系统架构**

```
    用户空间工具 (perf, pmu-tools)
              |
    +=========v=========+
    |   系统调用接口    |  <- perf_event_open()
    +===================+
              |
    +---------v---------+
    |  perf_event 核心  |  <- 事件管理、调度、采样
    +-------------------+
         /    |    \
        /     |     \
       v      v      v
    +-----+ +-----+ +-----+
    | CPU | | SW  | |Trace|
    | PMU | |Event| |Point|
    +-----+ +-----+ +-----+
```

**核心数据结构**

```c
/* include/linux/perf_event.h */
struct perf_event {
    struct list_head        event_entry;
    struct perf_event       *group_leader;
    struct pmu              *pmu;           /* 性能监控单元 */
    
    enum perf_event_state   state;
    atomic64_t              count;          /* 事件计数 */
    atomic64_t              child_count;
    
    struct perf_event_attr  attr;           /* 用户配置 */
    u64                     hw_event;       /* 硬件事件配置 */
    
    struct perf_sample_data sample_data;    /* 采样数据 */
    struct ring_buffer      *rb;            /* 环形缓冲区 */
    
    /* 回调函数 */
    perf_overflow_handler_t overflow_handler;
    struct perf_event_context *ctx;
};

/* 性能监控单元抽象 */
struct pmu {
    struct list_head        entry;
    const char              *name;
    int                     type;
    
    /* PMU 操作接口 */
    void (*pmu_enable)      (struct pmu *pmu);
    void (*pmu_disable)     (struct pmu *pmu);
    int  (*event_init)      (struct perf_event *event);
    void (*add)             (struct perf_event *event, int flags);
    void (*del)             (struct perf_event *event, int flags);
    void (*start)           (struct perf_event *event, int flags);
    void (*stop)            (struct perf_event *event, int flags);
    void (*read)            (struct perf_event *event);
};
```

### 15.2.2 硬件性能监控单元（PMU）

现代处理器提供专门的硬件计数器来统计各种微架构事件，如缓存命中/未命中、分支预测、指令执行等。

**Intel PMU 架构**

```c
/* arch/x86/events/intel/core.c */
static struct event_constraint intel_core_event_constraints[] = {
    INTEL_EVENT_CONSTRAINT(0x11, 0x2),  /* FP_ASSIST */
    INTEL_EVENT_CONSTRAINT(0x12, 0x2),  /* MUL */
    INTEL_EVENT_CONSTRAINT(0x13, 0x2),  /* DIV */
    INTEL_EVENT_CONSTRAINT(0x14, 0x1),  /* CYCLES_DIV_BUSY */
    INTEL_EVENT_CONSTRAINT(0x19, 0x2),  /* DELAYED_BYPASS */
};

/* 通用性能计数器配置 */
#define ARCH_PERFMON_EVENTSEL_EVENT    0x000000FFULL
#define ARCH_PERFMON_EVENTSEL_UMASK    0x0000FF00ULL
#define ARCH_PERFMON_EVENTSEL_USR       (1ULL << 16)
#define ARCH_PERFMON_EVENTSEL_OS        (1ULL << 17)
#define ARCH_PERFMON_EVENTSEL_EDGE      (1ULL << 18)
#define ARCH_PERFMON_EVENTSEL_INT       (1ULL << 20)
#define ARCH_PERFMON_EVENTSEL_ENABLE    (1ULL << 22)
```

**硬件事件映射**

```c
/* 通用硬件事件到具体 CPU 事件的映射 */
static u64 intel_pmu_event_map(int hw_event)
{
    switch (hw_event) {
    case PERF_COUNT_HW_CPU_CYCLES:
        return 0x003c;  /* UNHALTED_CORE_CYCLES */
    case PERF_COUNT_HW_INSTRUCTIONS:
        return 0x00c0;  /* INST_RETIRED.ANY_P */
    case PERF_COUNT_HW_CACHE_REFERENCES:
        return 0x4f2e;  /* LLC Reference */
    case PERF_COUNT_HW_CACHE_MISSES:
        return 0x412e;  /* LLC Misses */
    case PERF_COUNT_HW_BRANCH_INSTRUCTIONS:
        return 0x00c4;  /* BR_INST_RETIRED.ALL_BRANCHES */
    case PERF_COUNT_HW_BRANCH_MISSES:
        return 0x00c5;  /* BR_MISP_RETIRED.ALL_BRANCHES */
    }
    return 0;
}
```

**PMU 中断处理**

```c
/* 性能计数器溢出中断处理 */
static void intel_pmu_handle_irq(struct pt_regs *regs)
{
    struct cpu_hw_events *cpuc = this_cpu_ptr(&cpu_hw_events);
    int bit, loops;
    u64 status;
    
    /* 读取溢出状态 */
    rdmsrl(MSR_CORE_PERF_GLOBAL_STATUS, status);
    
    for_each_set_bit(bit, (unsigned long *)&status, X86_PMC_IDX_MAX) {
        struct perf_event *event = cpuc->events[bit];
        
        /* 处理溢出 */
        if (!intel_pmu_save_and_restart(event))
            continue;
            
        /* 生成采样数据 */
        perf_sample_data_init(&data, 0, event->hw.last_period);
        
        /* 调用溢出处理器 */
        if (perf_event_overflow(event, &data, regs))
            x86_pmu_stop(event, 0);
    }
    
    /* 清除溢出状态 */
    wrmsrl(MSR_CORE_PERF_GLOBAL_OVF_CTRL, status);
}
```

### 15.2.3 软件事件与跟踪点

除了硬件计数器，perf_events 还支持纯软件事件和跟踪点事件。

**软件事件类型**

```c
enum perf_sw_ids {
    PERF_COUNT_SW_CPU_CLOCK         = 0,
    PERF_COUNT_SW_TASK_CLOCK        = 1,
    PERF_COUNT_SW_PAGE_FAULTS       = 2,
    PERF_COUNT_SW_CONTEXT_SWITCHES  = 3,
    PERF_COUNT_SW_CPU_MIGRATIONS    = 4,
    PERF_COUNT_SW_PAGE_FAULTS_MIN   = 5,
    PERF_COUNT_SW_PAGE_FAULTS_MAJ   = 6,
    PERF_COUNT_SW_ALIGNMENT_FAULTS  = 7,
    PERF_COUNT_SW_EMULATION_FAULTS  = 8,
};

/* 软件事件触发 */
static inline void perf_sw_event(u32 event_id, u64 nr, struct pt_regs *regs)
{
    if (static_key_false(&perf_swevent_enabled[event_id]))
        __perf_sw_event(event_id, nr, regs, addr);
}
```

**跟踪点集成**

```c
/* 将跟踪点事件连接到 perf */
static int perf_trace_event_init(struct perf_event *event)
{
    struct trace_event_call *tp_event;
    
    tp_event = event->tp_event;
    if (!tp_event)
        return -ENOENT;
        
    /* 注册跟踪点处理器 */
    return perf_trace_event_reg(tp_event, event);
}
```

### 15.2.4 性能数据采集与分析

**采样机制**

```c
/* 周期性采样配置 */
struct perf_event_attr attr = {
    .type           = PERF_TYPE_HARDWARE,
    .config         = PERF_COUNT_HW_CPU_CYCLES,
    .sample_period  = 100000,  /* 每 100000 个周期采样一次 */
    .sample_type    = PERF_SAMPLE_IP | PERF_SAMPLE_TID |
                      PERF_SAMPLE_TIME | PERF_SAMPLE_CALLCHAIN,
    .exclude_kernel = 0,
    .exclude_user   = 0,
};
```

**环形缓冲区管理**

```c
/* kernel/events/ring_buffer.c */
struct ring_buffer {
    atomic_t                poll;
    local_t                 head;          /* 写入位置 */
    unsigned long           nest;
    local_t                 events;
    unsigned long           wakeup_stamp;
    local_t                 lost;
    
    long                    watermark;
    long                    aux_watermark;
    
    struct perf_event_mmap_page *user_page;  /* 用户映射页 */
    void                    *data_pages[];    /* 数据页数组 */
};

/* 写入采样数据 */
void perf_output_sample(struct perf_output_handle *handle,
                        struct perf_event_header *header,
                        struct perf_sample_data *data,
                        struct perf_event *event)
{
    /* 写入采样头 */
    perf_output_put(handle, *header);
    
    /* 写入 IP */
    if (sample_type & PERF_SAMPLE_IP)
        perf_output_put(handle, data->ip);
        
    /* 写入 TID */
    if (sample_type & PERF_SAMPLE_TID)
        perf_output_put(handle, data->tid_entry);
        
    /* 写入调用栈 */
    if (sample_type & PERF_SAMPLE_CALLCHAIN) {
        size_t size = data->callchain->nr;
        perf_output_put(handle, data->callchain->nr);
        perf_output_copy(handle, data->callchain->ips, size * 8);
    }
}
```

**使用 perf 工具**

```bash
# CPU 性能分析
perf record -F 99 -a -g -- sleep 10
perf report

# 缓存分析
perf stat -e cache-references,cache-misses ./program

# 调度延迟分析
perf sched record sleep 10
perf sched latency

# 锁竞争分析
perf lock record ./program
perf lock report
```

## 15.3 内核调试技术
- 15.3.1 KGDB 内核调试器
- 15.3.2 crash 工具与内核转储
- 15.3.3 kdump 机制
- 15.3.4 动态打印与 pr_debug

### 15.4 动态追踪工具
- 15.4.1 SystemTap 框架
- 15.4.2 eBPF 架构与原理
- 15.4.3 bpftrace 高级追踪
- 15.4.4 性能优化案例

### 15.5 可视化分析工具
- 15.5.1 火焰图生成与分析
- 15.5.2 延迟热图
- 15.5.3 调用图与依赖分析
- 15.5.4 实时性能监控

### 本章小结
### 练习题
### 常见陷阱与错误
### 最佳实践检查清单

---
