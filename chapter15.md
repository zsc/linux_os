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

内核调试是系统开发中最具挑战性的任务之一。与用户空间程序不同，内核运行在特权级别，一个错误就可能导致系统崩溃。Linux 提供了从源码级调试到崩溃分析的完整工具链。

### 15.3.1 KGDB 内核调试器

KGDB 是 Linux 内核的源码级调试器，通过串口或网络连接远程 GDB，实现对运行中内核的调试。它由 Jason Wessel 维护，从 2.6.26 版本开始进入主线。

**架构设计**

KGDB 采用客户端-服务器架构，内核作为调试存根（stub），外部 GDB 作为调试器：

```
    开发机器                     目标机器
    +-------+                   +--------+
    |  GDB  | <--- 串口/网络 --> |  KGDB  |
    +-------+                   | Kernel |
        |                       +--------+
        v                           |
    源码符号                     断点/单步
```

**核心组件**

```c
/* kernel/debug/debug_core.c */
struct kgdb_state {
    int                     cpu;
    int                     pass_exception;
    unsigned long           threadid;
    long                    kgdb_usethreadid;
    struct pt_regs          *linux_regs;
    atomic_t                *send_ready;
};

/* KGDB 架构接口 */
struct kgdb_arch {
    unsigned char           gdb_bpt_instr[BREAK_INSTR_SIZE];
    unsigned long           flags;
    
    int  (*set_breakpoint)(unsigned long addr, char *saved_instr);
    int  (*remove_breakpoint)(unsigned long addr, char *bundle);
    int  (*set_hw_breakpoint)(unsigned long addr, int len, enum kgdb_bptype type);
    int  (*remove_hw_breakpoint)(unsigned long addr, int len, enum kgdb_bptype type);
    void (*disable_hw_break)(struct pt_regs *regs);
    void (*correct_hw_break)(void);
    void (*enable_nmi)(bool on);
};
```

**断点实现机制**

KGDB 支持软件断点和硬件断点：

```c
/* 软件断点：替换指令为 int3 (x86) */
int kgdb_arch_set_breakpoint(struct kgdb_bkpt *bpt)
{
    int err;
    char opc[BREAK_INSTR_SIZE];
    
    /* 保存原始指令 */
    err = probe_kernel_read(opc, (char *)bpt->bpt_addr, BREAK_INSTR_SIZE);
    if (err)
        return err;
    memcpy(bpt->saved_instr, opc, BREAK_INSTR_SIZE);
    
    /* 写入断点指令 */
    err = probe_kernel_write((char *)bpt->bpt_addr, 
                            arch_kgdb_ops.gdb_bpt_instr, BREAK_INSTR_SIZE);
    return err;
}

/* 硬件断点：使用 CPU 调试寄存器 */
static int hw_break_reserve_slot(int breakno)
{
    int cpu, i;
    struct perf_event **pevent;
    
    for_each_online_cpu(cpu) {
        pevent = per_cpu_ptr(breakinfo[breakno].pev, cpu);
        *pevent = register_wide_hw_breakpoint(&attr, NULL, NULL);
        if (IS_ERR(*pevent))
            return PTR_ERR(*pevent);
    }
    return 0;
}
```

**异常处理流程**

当触发断点或收到调试信号时，KGDB 接管系统：

```c
/* 进入 KGDB 的核心函数 */
static int kgdb_cpu_enter(struct kgdb_state *ks, struct pt_regs *regs,
                         int exception_state)
{
    unsigned long flags;
    int cpu = raw_smp_processor_id();
    
    /* 1. 停止其他 CPU */
    if (!kgdb_single_step)
        kgdb_roundup_cpus(flags);
    
    /* 2. 禁用中断 */
    local_irq_save(flags);
    
    /* 3. 进入调试循环 */
    while (1) {
        /* 等待 GDB 命令 */
        error = gdb_serial_stub(ks);
        
        if (error == DBG_PASS_EVENT) {
            /* 继续执行 */
            break;
        }
        
        /* 处理命令 */
        switch (ks->gdb_cmd) {
        case 'c':  /* continue */
        case 's':  /* single step */
            return 0;
        }
    }
    
    /* 4. 恢复执行 */
    kgdb_restore_cpus();
    local_irq_restore(flags);
}
```

**使用配置**

```bash
# 内核配置
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_KGDB_KDB=y

# 启动参数
kgdboc=ttyS0,115200 kgdbwait

# GDB 连接
(gdb) target remote /dev/ttyS0
(gdb) set debug remote 1
(gdb) break sys_open
(gdb) continue
```

### 15.3.2 crash 工具与内核转储

crash 是 Red Hat 开发的内核崩溃分析工具，由 Dave Anderson 维护。它可以分析运行中的系统或崩溃转储文件（vmcore）。

**工作原理**

crash 通过解析内核符号表和数据结构，提供类似 GDB 的调试接口：

```
    vmcore/live system
           |
    +------v------+
    | crash tool  |
    +-------------+
           |
    解析内核结构
           |
    +------v------+
    | 符号表      |
    | debuginfo   |
    +-------------+
```

**核心功能实现**

```c
/* crash 内部数据结构解析 */
struct task_context {
    ulong task;           /* task_struct 地址 */
    ulong thread_info;    /* thread_info 地址 */
    ulong pid;
    char comm[TASK_COMM_LEN];
    int processor;
    ulong ptask;          /* 父进程 */
    ulong mm_struct;      /* 内存描述符 */
    struct task_context *tc_next;
};

/* 读取内核内存 */
static int readmem(ulonglong addr, int memtype, void *buffer, 
                   long size, char *type, ulong error_handle)
{
    switch (memtype) {
    case KVADDR:
        /* 内核虚拟地址 */
        return read_kdump(addr, buffer, size);
    case PHYSADDR:
        /* 物理地址 */
        return read_physmem(addr, buffer, size);
    case FILEADDR:
        /* 文件偏移 */
        return read_vmcore(addr, buffer, size);
    }
}
```

**常用命令实现**

```c
/* bt - 显示调用栈 */
void cmd_bt(void)
{
    struct bt_info bt_info, *bt;
    struct task_context *tc;
    ulong sp, ip;
    
    /* 获取当前任务上下文 */
    tc = CURRENT_CONTEXT();
    
    /* 初始化栈帧信息 */
    bt = &bt_info;
    bt->task = tc->task;
    bt->stackbase = GET_STACKBASE(tc->task);
    
    /* 展开调用栈 */
    sp = tc->thread_info + OFFSET(thread_info_cpu_context);
    ip = GET_PC(sp);
    
    while (sp < bt->stackbase) {
        /* 解析栈帧 */
        frame = GET_FRAME_POINTER(sp);
        function = closest_symbol(ip);
        
        fprintf(fp, "#%d [%016lx] %s at %016lx\n", 
                level++, sp, function, ip);
        
        /* 下一帧 */
        sp = frame;
        ip = GET_RETURN_ADDRESS(frame);
    }
}
```

**使用示例**

```bash
# 分析 vmcore
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/vmcore

# 常用命令
crash> sys          # 系统信息
crash> bt           # 当前任务调用栈
crash> ps           # 进程列表
crash> log          # 内核日志
crash> dis -l function_name  # 反汇编
crash> struct task_struct ffff88007c8a0000  # 查看数据结构
crash> foreach bt   # 所有任务的调用栈
```

### 15.3.3 kdump 机制

kdump 是 Linux 的内核崩溃转储机制，使用 kexec 在系统崩溃时启动第二个内核来收集转储。

**工作流程**

```
    生产内核崩溃
         |
         v
    触发 panic()
         |
         v
    machine_kexec()
         |
         v
    加载捕获内核
         |
         v
    捕获内核启动
         |
         v
    收集 vmcore
         |
         v
    保存到磁盘
```

**实现机制**

```c
/* kernel/kexec_core.c */
void __crash_kexec(struct pt_regs *regs)
{
    /* 已经有 CPU 在处理 */
    if (mutex_trylock(&kexec_mutex)) {
        if (kexec_crash_image) {
            struct pt_regs fixed_regs;
            
            /* 保存崩溃时的寄存器 */
            crash_setup_regs(&fixed_regs, regs);
            crash_save_vmcoreinfo();
            
            /* 关闭其他 CPU */
            machine_crash_shutdown(&fixed_regs);
            
            /* 跳转到捕获内核 */
            machine_kexec(kexec_crash_image);
        }
        mutex_unlock(&kexec_mutex);
    }
}

/* 保留内存区域 */
static int __init reserve_crashkernel(void)
{
    unsigned long long crash_size, crash_base;
    
    /* 解析 crashkernel= 参数 */
    ret = parse_crashkernel(boot_command_line, memblock_phys_mem_size(),
                           &crash_size, &crash_base);
    
    /* 预留内存 */
    memblock_reserve(crash_base, crash_size);
    
    /* 设置全局变量 */
    crashk_res.start = crash_base;
    crashk_res.end = crash_base + crash_size - 1;
}
```

**vmcore 生成**

```c
/* fs/proc/vmcore.c */
static int __init vmcore_init(void)
{
    int rc = 0;
    
    /* 解析 ELF 头 */
    rc = parse_crash_elf_headers();
    if (rc)
        return rc;
    
    /* 创建 /proc/vmcore */
    proc_vmcore = proc_create("vmcore", S_IRUSR, NULL, &proc_vmcore_operations);
    
    return 0;
}

/* 读取 vmcore */
static ssize_t read_vmcore(struct file *file, char __user *buffer,
                          size_t buflen, loff_t *fpos)
{
    /* 读取 ELF 头 */
    if (*fpos < elfcorebuf_sz) {
        tsz = min(elfcorebuf_sz - (size_t)*fpos, buflen);
        if (copy_to_user(buffer, elfcorebuf + *fpos, tsz))
            return -EFAULT;
    }
    
    /* 读取程序段 */
    list_for_each_entry(m, &vmcore_list, list) {
        if (*fpos < m->offset + m->size) {
            tsz = min_t(size_t, m->offset + m->size - *fpos, buflen);
            start = m->paddr + (*fpos - m->offset);
            
            /* 拷贝物理内存 */
            if (copy_oldmem_page(pfn, buffer, tsz, offset))
                return -EFAULT;
        }
    }
}
```

**配置使用**

```bash
# 安装 kdump 工具
yum install kexec-tools

# 配置预留内存
# /etc/default/grub
GRUB_CMDLINE_LINUX="crashkernel=256M"

# 启动 kdump 服务
systemctl enable kdump
systemctl start kdump

# 测试崩溃
echo 1 > /proc/sys/kernel/sysrq
echo c > /proc/sysrq-trigger
```

### 15.3.4 动态打印与 pr_debug

动态调试（dynamic debug）允许在运行时控制调试信息的输出，无需重新编译内核。

**实现原理**

```c
/* include/linux/dynamic_debug.h */
struct _ddebug {
    const char *modname;
    const char *function;
    const char *filename;
    const char *format;
    unsigned int lineno:18;
    unsigned int flags:8;
} __attribute__((aligned(8)));

/* pr_debug 宏展开 */
#define pr_debug(fmt, ...)                                      \
    dynamic_pr_debug(fmt, ##__VA_ARGS__)

#define dynamic_pr_debug(fmt, ...)                              \
do {                                                            \
    DEFINE_DYNAMIC_DEBUG_METADATA(descriptor, fmt);             \
    if (DYNAMIC_DEBUG_BRANCH(descriptor))                       \
        __dynamic_pr_debug(&descriptor, pr_fmt(fmt),           \
                          ##__VA_ARGS__);                       \
} while (0)
```

**控制接口**

```c
/* lib/dynamic_debug.c */
static int ddebug_proc_write(struct file *file, const char __user *ubuf,
                            size_t len, loff_t *offp)
{
    char *words[MAXWORDS];
    struct ddebug_query query;
    
    /* 解析控制命令 */
    nwords = ddebug_tokenize(tmpbuf, words, MAXWORDS);
    
    /* 解析查询条件 */
    if (ddebug_parse_query(words, nwords, &query))
        return -EINVAL;
    
    /* 应用修改 */
    nfound = ddebug_change(&query, flags, mask);
    
    return len;
}

/* 修改调试标志 */
static int ddebug_change(const struct ddebug_query *query,
                        unsigned int flags, unsigned int mask)
{
    struct ddebug_table *dt;
    struct _ddebug *dp;
    
    list_for_each_entry(dt, &ddebug_tables, link) {
        for (dp = dt->ddebugs; dp < dt->ddebugs + dt->num_ddebugs; dp++) {
            /* 匹配查询条件 */
            if (!match_wildcard(query->module, dp->modname))
                continue;
            if (!match_wildcard(query->function, dp->function))
                continue;
            if (!match_wildcard(query->filename, dp->filename))
                continue;
                
            /* 更新标志 */
            newflags = (dp->flags & ~mask) | flags;
            if (newflags != dp->flags) {
                dp->flags = newflags;
                matched++;
            }
        }
    }
    return matched;
}
```

**使用方法**

```bash
# 启用特定文件的调试
echo 'file tcp_input.c +p' > /sys/kernel/debug/dynamic_debug/control

# 启用特定函数
echo 'func tcp_receive_skb +p' > /sys/kernel/debug/dynamic_debug/control

# 启用模块调试
echo 'module e1000e +p' > /sys/kernel/debug/dynamic_debug/control

# 查看当前设置
cat /sys/kernel/debug/dynamic_debug/control

# 带条件的调试
echo 'file drivers/net/* line 1-200 +p' > /sys/kernel/debug/dynamic_debug/control
```

## 15.4 动态追踪工具

动态追踪技术让我们能够在生产环境中安全地观测系统行为，无需停机或重新编译。从 SystemTap 到 eBPF 的演进，标志着 Linux 可观测性的革命性进步。

### 15.4.1 SystemTap 框架

SystemTap 是 Red Hat 开发的动态追踪系统，通过将高级脚本语言编译成内核模块来实现系统追踪。尽管现在 eBPF 更受欢迎，但 SystemTap 仍在许多企业环境中使用。

**架构概览**

```
    SystemTap 脚本 (.stp)
           |
    +------v------+
    |   stap 编译器|  <- 语法分析、语义检查
    +------+------+
           |
    +------v------+
    | C 代码生成   |  <- 生成内核模块源码
    +------+------+
           |
    +------v------+
    |  gcc 编译    |  <- 编译成 .ko 模块
    +------+------+
           |
    +------v------+
    | staprun 加载 |  <- 插入内核执行
    +------+------+
           |
    运行时数据收集
```

**脚本语言特性**

```systemtap
/* SystemTap 脚本示例 */
global reads

probe vfs.read {
    reads[execname(), pid()] <<< $count
}

probe timer.s(5) {
    printf("Top processes reading:\n")
    foreach ([name, pid] in reads- limit 10) {
        printf("%s[%d]: %d bytes\n", 
               name, pid, @sum(reads[name, pid]))
    }
    delete reads
}

/* 探针语法 */
probe kernel.function("tcp_sendmsg") {
    printf("TCP send from %s: %d bytes\n", 
           execname(), $size)
}

probe kernel.function("tcp_sendmsg").return {
    if ($return < 0)
        printf("TCP send failed: %d\n", $return)
}
```

**运行时实现**

```c
/* runtime/transport/transport.c */
static int _stp_transport_init(void)
{
    /* 创建 relayfs 通道 */
    _stp_relay_data.rchan = relay_open("trace", 
                                       _stp_get_module_dir(),
                                       _stp_bufsize,
                                       n_subbufs,
                                       &_stp_relay_callbacks,
                                       NULL);
    
    /* 注册探针 */
    for (i = 0; i < _stp_num_probes; i++) {
        struct stap_probe *p = &_stp_probes[i];
        register_kprobe(&p->kprobe);
    }
}

/* 探针处理器模板 */
static int probe_NNNN(struct kprobe *inst, struct pt_regs *regs)
{
    struct context *c = per_cpu_ptr(contexts, smp_processor_id());
    
    /* 获取上下文 */
    c->regs = regs;
    c->pi = inst;
    
    /* 执行用户脚本逻辑 */
    probe_NNNN_enter(c);
    
    /* 输出数据 */
    _stp_print_flush();
    
    return 0;
}
```

### 15.4.2 eBPF 架构与原理

eBPF（extended Berkeley Packet Filter）是 Linux 内核的革命性技术，提供了安全、高效的内核可编程能力。由 Alexei Starovoitov 主导开发，已成为云原生可观测性的基石。

**eBPF 架构**

```
    用户空间                     内核空间
    +--------+                +------------+
    | eBPF   |                | eBPF 程序  |
    | 程序   | -- bpf() -->   | (字节码)   |
    +--------+                +-----+------+
                                   |
                              +----v-----+
                              | Verifier |  <- 安全验证
                              +----+-----+
                                   |
                              +----v-----+
                              | JIT 编译 |  <- 编译成机器码
                              +----+-----+
                                   |
                              +----v-----+
                              | 执行引擎 |  <- 在钩子点执行
                              +----------+
```

**eBPF 指令集**

```c
/* include/uapi/linux/bpf.h */
struct bpf_insn {
    __u8    code;    /* 操作码 */
    __u8    dst_reg:4;  /* 目标寄存器 */
    __u8    src_reg:4;  /* 源寄存器 */
    __s16   off;     /* 偏移 */
    __s32   imm;     /* 立即数 */
};

/* eBPF 寄存器 */
#define BPF_REG_0   0   /* 返回值 */
#define BPF_REG_1   1   /* 参数 1 */
#define BPF_REG_2   2   /* 参数 2 */
#define BPF_REG_3   3   /* 参数 3 */
#define BPF_REG_4   4   /* 参数 4 */
#define BPF_REG_5   5   /* 参数 5 */
#define BPF_REG_6   6   /* callee saved */
#define BPF_REG_7   7   /* callee saved */
#define BPF_REG_8   8   /* callee saved */
#define BPF_REG_9   9   /* callee saved */
#define BPF_REG_10  10  /* 栈指针 */

/* 指令示例 */
BPF_MOV64_REG(BPF_REG_6, BPF_REG_1),     /* r6 = r1 */
BPF_LD_ABS(BPF_B, ETH_HLEN + 0),         /* 加载字节 */
BPF_JMP_IMM(BPF_JEQ, BPF_REG_0, 0x80, 2), /* if r0 == 0x80 jump */
BPF_MOV64_IMM(BPF_REG_0, 0),             /* r0 = 0 */
BPF_EXIT_INSN(),                          /* return r0 */
```

**验证器（Verifier）**

eBPF 验证器确保程序安全性，防止内核崩溃：

```c
/* kernel/bpf/verifier.c */
static int do_check(struct bpf_verifier_env *env)
{
    struct bpf_insn *insns = env->prog->insnsi;
    struct bpf_verifier_state *state;
    int insn_cnt = env->prog->len;
    
    /* 遍历所有可能的执行路径 */
    for (i = 0; i < insn_cnt; i++) {
        struct bpf_insn *insn = &insns[i];
        u8 class = BPF_CLASS(insn->code);
        
        /* 检查内存访问 */
        if (class == BPF_LDX || class == BPF_STX) {
            err = check_mem_access(env, insn_idx, regno, off, size,
                                 t, value_regno, strict_alignment);
            if (err)
                return err;
        }
        
        /* 检查函数调用 */
        if (insn->code == (BPF_JMP | BPF_CALL)) {
            err = check_helper_call(env, insn->imm, insn_idx);
            if (err)
                return err;
        }
        
        /* 检查跳转 */
        if (BPF_CLASS(insn->code) == BPF_JMP) {
            err = check_cond_jmp_op(env, insn, &insn_idx);
            if (err)
                return err;
        }
    }
    
    /* 确保程序有返回 */
    if (state->curframe) {
        verbose(env, "program didn't return\n");
        return -EINVAL;
    }
    
    return 0;
}

/* 检查内存访问安全性 */
static int check_mem_access(struct bpf_verifier_env *env, int insn_idx,
                           u32 regno, int off, int bpf_size,
                           enum bpf_access_type t,
                           int value_regno, bool strict_alignment)
{
    struct bpf_reg_state *regs = cur_regs(env);
    struct bpf_reg_state *reg = regs + regno;
    
    /* 检查指针类型 */
    switch (reg->type) {
    case PTR_TO_MAP_VALUE:
        /* 检查 map 访问边界 */
        if (check_map_access(env, regno, off, size, false))
            return -EACCES;
        break;
    case PTR_TO_CTX:
        /* 检查上下文访问 */
        if (check_ctx_access(env, insn_idx, off, size, t, &reg_type))
            return -EACCES;
        break;
    case PTR_TO_STACK:
        /* 检查栈访问 */
        if (check_stack_boundary(env, regno, off, size, access_type))
            return -EACCES;
        break;
    }
    return 0;
}
```

**JIT 编译器**

将 eBPF 字节码编译成本地机器码：

```c
/* arch/x86/net/bpf_jit_comp.c */
static int do_jit(struct bpf_prog *bpf_prog, int *addrs, u8 *image,
                 int oldproglen, struct jit_context *ctx)
{
    struct bpf_insn *insn = bpf_prog->insnsi;
    int insn_cnt = bpf_prog->len;
    u8 *prog = temp;
    
    for (i = 0; i < insn_cnt; i++, insn++) {
        const s32 imm32 = insn->imm;
        u32 dst_reg = insn->dst_reg;
        u32 src_reg = insn->src_reg;
        u8 b2 = 0, b3 = 0;
        s64 jmp_offset;
        u8 jmp_cond;
        
        switch (insn->code) {
        /* ALU 操作 */
        case BPF_ALU64 | BPF_ADD | BPF_X:
            EMIT3(0x48, 0x01, add_2reg(0xC0, dst_reg, src_reg));
            break;
            
        /* 内存加载 */
        case BPF_LDX | BPF_MEM | BPF_DW:
            EMIT3(0x48, 0x8B, add_2reg(0x40, dst_reg, src_reg));
            EMIT1(insn->off);
            break;
            
        /* 函数调用 */
        case BPF_JMP | BPF_CALL:
            func = (u8 *) __bpf_call_base + imm32;
            jmp_offset = func - (prog + 5);
            EMIT1_off32(0xE8, jmp_offset);
            break;
            
        /* 条件跳转 */
        case BPF_JMP | BPF_JEQ | BPF_X:
            EMIT3(0x48, 0x39, add_2reg(0xC0, dst_reg, src_reg));
            goto emit_cond_jmp;
        }
    }
}
```

**eBPF Maps**

Maps 是 eBPF 程序与用户空间通信的关键机制：

```c
/* kernel/bpf/hashtab.c */
struct bpf_htab {
    struct bpf_map map;
    struct bucket *buckets;
    void *elems;
    atomic_t count;
    u32 n_buckets;
    u32 elem_size;
    struct lock_class_key lockdep_key;
};

/* 查找元素 */
static void *htab_map_lookup_elem(struct bpf_map *map, void *key)
{
    struct bpf_htab *htab = container_of(map, struct bpf_htab, map);
    struct hlist_nulls_head *head;
    struct htab_elem *l;
    u32 hash, key_size;
    
    /* 计算哈希值 */
    hash = htab_map_hash(key, key_size, htab->hashrnd);
    head = select_bucket(htab, hash);
    
    /* 查找链表 */
    hlist_nulls_for_each_entry_rcu(l, n, head, hash_node) {
        if (l->hash == hash && !memcmp(&l->key, key, key_size))
            return l->key + round_up(key_size, 8);
    }
    
    return NULL;
}
```

### 15.4.3 bpftrace 高级追踪

bpftrace 是 Brendan Gregg 和 Alastair Robertson 开发的高级追踪语言，提供了类似 DTrace 的简洁语法。

**语言特性**

```bpftrace
#!/usr/bin/env bpftrace

/* 追踪系统调用延迟 */
tracepoint:raw_syscalls:sys_enter
{
    @start[tid] = nsecs;
}

tracepoint:raw_syscalls:sys_exit
/@start[tid]/
{
    @ns[comm] = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}

/* 追踪 TCP 重传 */
kprobe:tcp_retransmit_skb
{
    @retrans[comm, pid] = count();
}

/* 追踪块 I/O 延迟 */
kprobe:blk_account_io_start
{
    @iotime[arg0] = nsecs;
}

kprobe:blk_account_io_done
/@iotime[arg0]/
{
    @usecs = hist((nsecs - @iotime[arg0]) / 1000);
    delete(@iotime[arg0]);
}

END
{
    clear(@iotime);
    clear(@start);
}
```

**内部实现**

bpftrace 将高级语言编译成 eBPF 字节码：

```c
/* AST 到 LLVM IR 的转换 */
void CodegenLLVM::visit(Call &call)
{
    if (call.func == "count") {
        // 生成计数器更新代码
        auto *map_lookup = b_.CreateCall(lookup_func, 
                                        {map_ptr, key_ptr});
        auto *counter = b_.CreateLoad(map_lookup);
        auto *incremented = b_.CreateAdd(counter, 
                                        b_.getInt64(1));
        b_.CreateStore(incremented, map_lookup);
    }
    else if (call.func == "hist") {
        // 生成直方图更新代码
        generate_histogram_update(call);
    }
}
```

### 15.4.4 性能优化案例

**案例1：追踪内存分配热点**

```c
/* eBPF C 程序 */
#include <uapi/linux/ptrace.h>
#include <linux/mm.h>

BPF_HASH(allocs, u64, u64);
BPF_STACK_TRACE(stack_traces, 1024);

int trace_kmalloc(struct pt_regs *ctx, size_t size)
{
    u64 pid = bpf_get_current_pid_tgid();
    u64 stackid = stack_traces.get_stackid(ctx, 0);
    
    struct alloc_info info = {};
    info.size = size;
    info.stackid = stackid;
    info.timestamp = bpf_ktime_get_ns();
    
    allocs.update(&pid, &info);
    return 0;
}
```

**案例2：网络延迟分析**

```bpftrace
#!/usr/bin/env bpftrace

/* TCP 连接延迟分析 */
kprobe:tcp_v4_connect
{
    @start[tid] = nsecs;
    @daddr[tid] = ((struct sockaddr_in *)arg1)->sin_addr.s_addr;
}

kretprobe:tcp_v4_connect
/@start[tid]/
{
    $lat = (nsecs - @start[tid]) / 1000;
    @conn_lat = hist($lat);
    
    if ($lat > 1000000) {  // > 1秒
        printf("Slow connection to %s: %d us\n",
               ntop(@daddr[tid]), $lat);
    }
    
    delete(@start[tid]);
    delete(@daddr[tid]);
}
```

**案例3：文件系统性能瓶颈**

```c
/* 追踪 ext4 写入延迟 */
struct data_t {
    u64 ts;
    u64 delta;
    u32 pid;
    char comm[16];
    char filename[32];
};

BPF_PERF_OUTPUT(events);
BPF_HASH(start, u32);

int trace_ext4_write_begin(struct pt_regs *ctx)
{
    u32 pid = bpf_get_current_pid_tgid();
    u64 ts = bpf_ktime_get_ns();
    start.update(&pid, &ts);
    return 0;
}

int trace_ext4_write_end(struct pt_regs *ctx)
{
    u32 pid = bpf_get_current_pid_tgid();
    u64 *tsp = start.lookup(&pid);
    if (!tsp)
        return 0;
        
    struct data_t data = {};
    data.ts = bpf_ktime_get_ns();
    data.delta = data.ts - *tsp;
    data.pid = pid;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    
    events.perf_submit(ctx, &data, sizeof(data));
    start.delete(&pid);
    return 0;
}
```

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
