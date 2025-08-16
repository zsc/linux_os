# 第14章：实时Linux

## 本章学习目标

实时系统要求任务在确定的时间约束内完成，这对传统 Linux 内核提出了严峻挑战。本章深入剖析 Linux 实时扩展的核心技术，从 PREEMPT_RT 补丁集到 SCHED_DEADLINE 调度器，从优先级继承机制到高精度定时器。通过学习本章，您将掌握如何将通用 Linux 转变为满足严格时间约束的实时操作系统，理解延迟的来源与优化技术，并能够设计和调试实时应用。

## 14.1 实时系统基础概念

### 14.1.1 实时性定义与分类

实时系统的核心是**可预测性**而非单纯的快速响应。一个实时系统必须保证：

$$\forall t_i \in T: t_{response} \leq t_{deadline}$$

其中 $T$ 是任务集合，$t_{response}$ 是响应时间，$t_{deadline}$ 是截止时间。

根据对截止时间违反的容忍度，实时系统分为：

1. **硬实时（Hard Real-time）**：错过截止时间将导致灾难性后果
   - 例：飞机控制系统、心脏起搏器
   - 要求：最坏情况执行时间（WCET）必须可证明

2. **软实时（Soft Real-time）**：偶尔错过截止时间可接受但会降低服务质量
   - 例：视频播放、VoIP 通话
   - 要求：统计意义上的延迟保证

3. **固实时（Firm Real-time）**：错过截止时间的结果无用但不灾难
   - 例：工业控制显示系统
   - 要求：及时性与资源利用率平衡

### 14.1.2 Linux 内核延迟来源

标准 Linux 内核的延迟主要来自：

```
中断延迟链：
    硬件中断 → 中断处理程序
         ↓
    软中断处理 → 任务调度
         ↓
    用户进程执行
```

具体延迟组成：

1. **中断延迟**（Interrupt Latency）
   - 中断禁用期间：临界区保护
   - 中断处理时间：ISR 执行
   - 公式：$L_{irq} = T_{disable} + T_{ISR}$

2. **调度延迟**（Scheduling Latency）
   - 内核不可抢占区：自旋锁持有期
   - 调度器开销：任务选择算法
   - 公式：$L_{sched} = T_{preempt\_disable} + T_{scheduler}$

3. **同步延迟**（Synchronization Latency）
   - 优先级反转：低优先级任务持锁
   - 锁竞争：等待资源释放

### 14.1.3 实时性演进历程

Linux 实时性改进的里程碑：

```
1991: Linux 0.01 - 完全不可抢占
  |
2001: 2.4 - 引入低延迟补丁
  |
2003: 2.6 - 内核可抢占（CONFIG_PREEMPT）
  |
2004: PREEMPT_RT 项目启动（Ingo Molnar）
  |
2005: 高精度定时器（hrtimer）
  |
2009: SCHED_DEADLINE 调度类
  |
2019: 5.0 - PREEMPT_RT 开始主线化
  |
2023: 6.x - 大部分 RT 特性已主线
```

## 14.2 PREEMPT_RT 补丁集

### 14.2.1 核心设计理念

PREEMPT_RT 的目标是将 Linux 转变为完全可抢占的内核：

```
标准内核执行流：
    用户空间 → [可抢占]
         ↓
    系统调用 → [部分可抢占]
         ↓
    中断处理 → [不可抢占]

PREEMPT_RT 执行流：
    用户空间 → [可抢占]
         ↓
    系统调用 → [完全可抢占]
         ↓
    线程化中断 → [可抢占]
```

### 14.2.2 关键组件改造

#### 自旋锁转换为可睡眠锁

标准内核的 spinlock 在 RT 中变为 rt_mutex：

```c
/* 标准内核 */
struct spinlock {
    arch_spinlock_t raw_lock;
};

/* PREEMPT_RT */
struct spinlock {
    struct rt_mutex_base lock;  /* 可睡眠的互斥锁 */
#ifdef CONFIG_DEBUG_LOCK_ALLOC
    struct lockdep_map dep_map;
#endif
};
```

这种转换的影响：
- 临界区变为可抢占
- 持锁时可以睡眠
- 支持优先级继承

#### 中断线程化

将中断处理程序转换为内核线程：

```c
/* 中断线程化结构 */
struct irq_desc {
    struct irqaction *action;
    struct task_struct *thread;  /* 中断线程 */
    wait_queue_head_t wait_for_threads;
    atomic_t threads_active;
};

/* 线程化中断处理流程 */
irqreturn_t handle_irq_event(struct irq_desc *desc)
{
    if (desc->thread) {
        /* 唤醒中断线程 */
        wake_up_process(desc->thread);
        return IRQ_WAKE_THREAD;
    }
    /* 直接处理 */
    return action->handler(irq, action->dev_id);
}
```

#### 软中断线程化

软中断也被转换为 ksoftirqd 线程：

```c
/* RT 内核的软中断处理 */
static void run_ksoftirqd(unsigned int cpu)
{
    struct task_struct *tsk = __this_cpu_read(ksoftirqd);
    
    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}
```

### 14.2.3 抢占模型对比

Linux 提供多种抢占模型：

| 模型 | CONFIG选项 | 特点 | 延迟 | 吞吐量 |
|------|------------|------|------|--------|
| 无抢占 | PREEMPT_NONE | 仅自愿调度点 | 高(~100ms) | 最高 |
| 自愿抢占 | PREEMPT_VOLUNTARY | 增加抢占点 | 中(~10ms) | 高 |
| 抢占 | PREEMPT | 内核可抢占 | 低(~1ms) | 中 |
| RT抢占 | PREEMPT_RT | 完全可抢占 | 极低(~50μs) | 较低 |

## 14.3 优先级继承与优先级天花板

### 14.3.1 优先级反转问题

优先级反转发生在高优先级任务等待低优先级任务持有的资源时：

```
时间轴展示：
T0: L(低优先级)获取锁 mutex_A
T1: H(高优先级)尝试获取 mutex_A，阻塞
T2: M(中优先级)就绪，抢占 L
T3: M 执行...
T4: M 完成，L 继续
T5: L 释放 mutex_A
T6: H 获得 mutex_A 并执行

问题：H 被 M 间接阻塞，违反优先级语义
```

### 14.3.2 优先级继承协议（PI）

PI 协议通过临时提升持锁任务的优先级解决反转：

```c
/* rt_mutex 优先级继承实现 */
struct rt_mutex_waiter {
    struct rb_node tree_node;     /* 红黑树节点 */
    struct task_struct *task;     /* 等待任务 */
    struct rt_mutex_base *lock;   /* 等待的锁 */
    int prio;                      /* 等待者优先级 */
    u64 deadline;                  /* 截止时间 */
};

/* 优先级继承链 */
static int task_blocks_on_rt_mutex(struct rt_mutex_base *lock,
                                   struct rt_mutex_waiter *waiter,
                                   struct task_struct *task)
{
    struct task_struct *owner = rt_mutex_owner(lock);
    
    /* 将等待者加入等待队列 */
    rt_mutex_enqueue(lock, waiter);
    
    /* 继承优先级 */
    if (task->prio < owner->prio) {
        /* 提升持锁者优先级 */
        rt_mutex_adjust_prio_chain(owner, task->prio);
    }
    
    return 0;
}
```

优先级继承的传递性：

```
PI链传递：
T1(prio=10) → blocks on L1 held by →
T2(prio=30) → blocks on L2 held by →
T3(prio=50) → 继承 prio=10

结果：T3 临时运行在 prio=10
```

### 14.3.3 优先级天花板协议（PC）

PC 协议预先将锁的优先级设为可能访问它的最高优先级任务：

```c
/* 优先级天花板实现伪代码 */
struct prio_ceiling_mutex {
    struct rt_mutex base;
    int ceiling_priority;  /* 天花板优先级 */
};

int pc_mutex_lock(struct prio_ceiling_mutex *pcm)
{
    struct task_struct *task = current;
    int old_prio = task->prio;
    
    /* 立即提升到天花板优先级 */
    if (pcm->ceiling_priority < task->prio) {
        task->prio = pcm->ceiling_priority;
        /* 重新入队调度器 */
        requeue_task_rt(task);
    }
    
    rt_mutex_lock(&pcm->base);
    
    return old_prio;
}
```

PI vs PC 对比：

| 特性 | 优先级继承(PI) | 优先级天花板(PC) |
|------|---------------|-----------------|
| 优先级调整时机 | 发生阻塞时 | 获取锁时 |
| 开销 | 动态，链式传递 | 静态，固定 |
| 死锁预防 | 不能 | 可以 |
| 实现复杂度 | 高 | 中 |
| Linux支持 | 完整支持 | 部分支持 |

## 14.4 高精度定时器（hrtimer）

### 14.4.1 传统定时器的局限

Linux 传统定时器基于 jiffies，精度受 HZ 限制：

```c
/* 传统定时器精度 */
HZ=100:  精度 = 10ms
HZ=1000: 精度 = 1ms

/* jiffies 定时器问题 */
1. 精度不足：最小粒度受 HZ 限制
2. 级联开销：时间轮级联操作
3. 同步问题：所有定时器在 tick 边界触发
```

### 14.4.2 hrtimer 架构设计

高精度定时器使用红黑树组织，支持纳秒级精度：

```c
/* hrtimer 数据结构 */
struct hrtimer {
    struct timerqueue_node node;  /* 红黑树节点 */
    ktime_t _softexpires;         /* 软过期时间 */
    enum hrtimer_restart (*function)(struct hrtimer *);
    struct hrtimer_clock_base *base;
    u8 state;
    u8 is_rel;                    /* 相对/绝对定时器 */
};

/* 时钟基础 */
struct hrtimer_clock_base {
    struct hrtimer_cpu_base *cpu_base;
    int index;
    clockid_t clockid;            /* CLOCK_MONOTONIC/REALTIME */
    struct timerqueue_head active; /* 红黑树根 */
    ktime_t resolution;           /* 时钟分辨率 */
    ktime_t (*get_time)(void);    /* 获取当前时间 */
    ktime_t offset;              /* 时间偏移 */
};

### 14.4.3 时钟源与时钟事件设备

Linux 时间子系统分离了时钟源（clocksource）和时钟事件设备（clock_event_device）：

```c
/* 时钟源：提供时间读取 */
struct clocksource {
    u64 (*read)(struct clocksource *cs);
    u64 mask;
    u32 mult;                    /* 频率转换乘数 */
    u32 shift;                   /* 频率转换移位 */
    u64 max_idle_ns;
    const char *name;            /* TSC, HPET, ACPI_PM */
    int rating;                  /* 优先级评分 */
};

/* 时钟事件设备：产生定时中断 */
struct clock_event_device {
    void (*event_handler)(struct clock_event_device *);
    int (*set_next_event)(unsigned long evt,
                         struct clock_event_device *);
    int (*set_state_oneshot)(struct clock_event_device *);
    int (*set_state_periodic)(struct clock_event_device *);
    const char *name;
    unsigned int features;       /* ONESHOT, PERIODIC */
    unsigned long max_delta_ns;
    unsigned long min_delta_ns;
};
```

时钟源优先级（x86）：

```
Rating  时钟源         精度        特点
400     TSC           ~1ns        CPU时间戳计数器
250     HPET          ~100ns      高精度事件定时器  
200     ACPI_PM       ~300ns      ACPI电源管理定时器
100     PIT           ~1μs        可编程间隔定时器
```

### 14.4.4 hrtimer 操作实现

定时器的启动与到期处理：

```c
/* 启动高精度定时器 */
int hrtimer_start(struct hrtimer *timer, ktime_t tim,
                  enum hrtimer_mode mode)
{
    struct hrtimer_clock_base *base;
    unsigned long flags;
    
    base = lock_hrtimer_base(timer, &flags);
    
    /* 计算绝对过期时间 */
    if (mode & HRTIMER_MODE_REL) {
        tim = ktime_add_safe(tim, base->get_time());
    }
    
    timer->_softexpires = tim;
    
    /* 插入红黑树 */
    enqueue_hrtimer(timer, base);
    
    /* 编程硬件定时器 */
    if (timer == base->active.next) {
        hrtimer_reprogram(timer, base);
    }
    
    unlock_hrtimer_base(timer, &flags);
    return 0;
}

/* 定时器到期处理 */
void hrtimer_interrupt(struct clock_event_device *dev)
{
    struct hrtimer_cpu_base *cpu_base = this_cpu_ptr(&hrtimer_bases);
    ktime_t now = ktime_get();
    
    while ((node = timerqueue_getnext(&base->active))) {
        timer = container_of(node, struct hrtimer, node);
        
        if (timer->_softexpires > now)
            break;
            
        __remove_hrtimer(timer, base);
        fn = timer->function;
        
        /* 执行回调 */
        restart = fn(timer);
        
        if (restart == HRTIMER_RESTART) {
            hrtimer_start_expires(timer, HRTIMER_MODE_ABS);
        }
    }
}
```

### 14.4.5 动态时钟（NO_HZ）

NO_HZ 模式在系统空闲时停止周期性时钟中断：

```
传统 HZ 模式：
|tick|tick|tick|tick|tick|tick|  每 1/HZ 秒中断
     浪费功耗

NO_HZ_IDLE 模式：
|tick|------idle------|tick|     空闲时无中断
          省电

NO_HZ_FULL 模式：
|tick|---userspace----|tick|     用户态运行时无中断
       降低延迟抖动
```

## 14.5 实时调度类

### 14.5.1 SCHED_FIFO：先进先出调度

SCHED_FIFO 是最简单的实时调度策略：

```c
/* SCHED_FIFO 运行队列 */
struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1);  /* 优先级位图 */
    struct list_head queue[MAX_RT_PRIO];    /* 每个优先级的任务队列 */
};

/* 选择下一个任务 - O(1) 复杂度 */
static struct task_struct *pick_next_rt_entity(struct rt_rq *rt_rq)
{
    struct rt_prio_array *array = &rt_rq->active;
    int idx;
    
    /* 找到最高优先级 */
    idx = sched_find_first_bit(array->bitmap);
    if (idx >= MAX_RT_PRIO)
        return NULL;
        
    /* 返回队首任务 */
    return list_first_entry(&array->queue[idx],
                           struct task_struct, rt_list_entry);
}
```

SCHED_FIFO 特性：
- 无时间片，运行直到主动让出
- 同优先级任务 FIFO 排队
- 优先级范围：1-99（99最高）

### 14.5.2 SCHED_RR：轮转调度

SCHED_RR 在 FIFO 基础上增加时间片：

```c
/* RR 时间片管理 */
static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)
{
    struct sched_rt_entity *rt_se = &p->rt;
    
    if (p->policy != SCHED_RR)
        return;
        
    if (--p->rt.time_slice)
        return;
        
    /* 时间片用完，重置并重新入队 */
    p->rt.time_slice = sched_rr_timeslice;
    
    /* 移到队尾 */
    requeue_task_rt(rq, p, 0);
    
    /* 设置重新调度标志 */
    set_tsk_need_resched(p);
}
```

默认 RR 时间片：100ms（可通过 /proc/sys/kernel/sched_rr_timeslice_ms 调整）

### 14.5.3 SCHED_DEADLINE：EDF 调度

SCHED_DEADLINE 实现最早截止时间优先（EDF）算法：

```c
/* DEADLINE 任务参数 */
struct sched_dl_entity {
    struct rb_node rb_node;      /* 红黑树节点 */
    
    /* 三个关键参数 */
    u64 dl_runtime;              /* 最大运行时间 */
    u64 dl_deadline;             /* 相对截止时间 */
    u64 dl_period;               /* 周期 */
    
    /* 运行时状态 */
    u64 deadline;                /* 绝对截止时间 */
    u64 runtime;                 /* 剩余运行时间 */
    
    /* CBS (Constant Bandwidth Server) */
    u64 dl_bw;                   /* 带宽 = runtime/period */
};
```

EDF 调度算法核心：

```c
/* 选择最早截止时间的任务 */
static struct sched_dl_entity *pick_next_dl_entity(struct dl_rq *dl_rq)
{
    struct rb_node *left = rb_first_cached(&dl_rq->root);
    
    if (!left)
        return NULL;
        
    return rb_entry(left, struct sched_dl_entity, rb_node);
}

/* 准入控制 - 保证可调度性 */
static int dl_overflow(struct task_struct *p, int policy,
                      const struct sched_attr *attr)
{
    u64 period = attr->sched_period;
    u64 runtime = attr->sched_runtime;
    u64 new_bw = runtime / period;
    
    /* 检查 CPU 利用率上界 */
    if (dl_rq->total_bw + new_bw > dl_rq->max_bw)
        return -EBUSY;  /* 拒绝任务 */
        
    return 0;
}
```

可调度性分析（单处理器）：

$$U = \sum_{i=1}^{n} \frac{C_i}{T_i} \leq 1$$

其中 $C_i$ 是任务 $i$ 的计算时间，$T_i$ 是周期。

### 14.5.4 调度类优先级

Linux 调度类的优先级顺序：

```
调度类优先级（从高到低）：
1. SCHED_DEADLINE  - EDF实时调度
2. SCHED_FIFO      - 固定优先级实时
3. SCHED_RR        - 轮转实时
4. SCHED_NORMAL    - CFS公平调度
5. SCHED_BATCH     - 批处理
6. SCHED_IDLE      - 空闲

选择逻辑：
for_each_class(class) {
    if (class->pick_next_task(rq))
        return task;
}
```

## 14.6 延迟分析与优化

### 14.6.1 延迟测量工具

Linux 提供多种延迟追踪机制：

```bash
# cyclictest - RT 延迟测试标准工具
cyclictest -p 99 -m -n -l 100000 -h 1000

# ftrace 延迟追踪器
echo 'irqsoff' > /sys/kernel/debug/tracing/current_tracer
echo 'preemptoff' > /sys/kernel/debug/tracing/current_tracer
echo 'wakeup_rt' > /sys/kernel/debug/tracing/current_tracer

# hwlat_detector - 硬件/固件延迟检测
echo 'hwlat' > /sys/kernel/debug/tracing/current_tracer
```

### 14.6.2 中断延迟优化

减少中断延迟的技术：

```c
/* 1. 中断合并（NAPI） */
static int napi_poll(struct napi_struct *napi, int budget)
{
    int work_done = 0;
    
    /* 批量处理包，减少中断频率 */
    while (work_done < budget) {
        if (!process_packet())
            break;
        work_done++;
    }
    
    if (work_done < budget)
        napi_complete(napi);
        
    return work_done;
}

/* 2. 中断亲和性设置 */
echo 2 > /proc/irq/24/smp_affinity  # 绑定到 CPU1

/* 3. 中断线程优先级调整 */
chrt -f -p 99 $(pidof irq/24-eth0)
```

### 14.6.3 调度延迟优化

降低调度延迟的方法：

```c
/* 1. 减少抢占禁用区 */
// 不好的做法
preempt_disable();
for (i = 0; i < 10000; i++) {
    do_something();
}
preempt_enable();

// 改进做法
for (i = 0; i < 10000; i++) {
    preempt_disable();
    do_something();
    preempt_enable();
    cond_resched();  /* 自愿调度点 */
}

/* 2. 使用 local_lock 替代 preempt_disable */
DEFINE_PER_CPU(struct local_lock, my_lock);

local_lock(&my_lock);
/* 临界区 - RT 内核中可抢占 */
local_unlock(&my_lock);
```

### 14.6.4 最坏情况执行时间（WCET）

WCET 分析对硬实时系统至关重要：

```
WCET 组成：
├── 任务执行时间
│   ├── 基本块执行
│   └── 缓存效应
├── 内核开销
│   ├── 系统调用
│   └── 上下文切换
└── 干扰时间
    ├── 中断处理
    └── 高优先级抢占

总 WCET = max(执行路径) + max(内核开销) + Σ(干扰)
```

## 14.7 实时系统集成技术

### 14.7.1 CPU 隔离（Isolation）

将 CPU 专用于实时任务，避免干扰：

```bash
# 内核启动参数
isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3

# 运行时绑定
taskset -c 2 ./realtime_app
```

CPU 隔离效果：
- 无调度器负载均衡
- 无 RCU 回调
- 无定时器中断（NO_HZ_FULL）

### 14.7.2 内存锁定

防止页面换出导致的不确定延迟：

```c
/* 锁定所有内存页 */
if (mlockall(MCL_CURRENT | MCL_FUTURE) != 0) {
    perror("mlockall failed");
    exit(1);
}

/* 预分配堆栈 */
void stack_prefault(void) {
    unsigned char dummy[MAX_STACK_SIZE];
    memset(dummy, 0, MAX_STACK_SIZE);
}
```

### 14.7.3 实时应用模板

标准实时应用初始化流程：

```c
void setup_realtime(void)
{
    struct sched_param param;
    cpu_set_t cpuset;
    
    /* 1. 设置实时调度策略 */
    param.sched_priority = 99;
    if (sched_setscheduler(0, SCHED_FIFO, &param) < 0) {
        perror("sched_setscheduler");
        exit(1);
    }
    
    /* 2. 绑定 CPU */
    CPU_ZERO(&cpuset);
    CPU_SET(2, &cpuset);
    if (sched_setaffinity(0, sizeof(cpuset), &cpuset) < 0) {
        perror("sched_setaffinity");
        exit(1);
    }
    
    /* 3. 锁定内存 */
    if (mlockall(MCL_CURRENT | MCL_FUTURE) < 0) {
        perror("mlockall");
        exit(1);
    }
    
    /* 4. 预热缓存 */
    stack_prefault();
}
```

## 本章小结

本章深入探讨了 Linux 实时扩展的核心技术。从 PREEMPT_RT 补丁集的完全可抢占内核设计，到优先级继承协议解决优先级反转问题，再到高精度定时器和 SCHED_DEADLINE 调度器的实现，我们系统地分析了将通用 Linux 转变为实时操作系统的关键机制。

**核心要点回顾**：

1. **PREEMPT_RT 三大改造**：自旋锁睡眠化、中断线程化、软中断线程化
2. **优先级继承链**：动态提升持锁任务优先级，防止优先级反转
3. **高精度定时器**：纳秒级精度，红黑树组织，支持 NO_HZ 模式
4. **实时调度类层次**：DEADLINE > FIFO > RR > NORMAL
5. **延迟优化技术**：CPU 隔离、内存锁定、中断亲和性

**关键公式总结**：

- EDF 可调度性：$U = \sum \frac{C_i}{T_i} \leq 1$
- 响应时间：$R_i = C_i + \sum_{j \in hp(i)} \lceil \frac{R_i}{T_j} \rceil C_j$
- 优先级继承：$prio_{effective} = \min(prio_{nominal}, prio_{waiters})$

## 练习题

### 基础题

**1. PREEMPT_RT 机制理解**
分析以下代码在标准内核和 RT 内核中的行为差异：
```c
spin_lock(&lock);
msleep(10);  /* 睡眠 10ms */
spin_unlock(&lock);
```

<details>
<summary>答案</summary>

标准内核：编译错误或内核崩溃。spin_lock 持有时不能睡眠，因为会导致死锁。

RT 内核：正常执行。spin_lock 在 RT 中是可睡眠的 rt_mutex，允许持锁睡眠并支持优先级继承。

关键差异：RT 内核将自旋锁转换为互斥锁，实现完全可抢占。
</details>

**2. 优先级反转场景分析**
三个任务 L(优先级=10)、M(优先级=50)、H(优先级=90) 共享互斥锁 mutex_A。如果 L 先获得锁，然后 H 请求锁，最后 M 就绪，描述任务执行顺序。

<details>
<summary>答案</summary>

无优先级继承：
1. L 持锁运行
2. H 请求锁，阻塞
3. M 就绪，抢占 L
4. M 执行完成
5. L 继续，释放锁
6. H 获得锁执行

有优先级继承：
1. L 持锁运行
2. H 请求锁，L 继承优先级 90
3. M 就绪但优先级低于 L(90)
4. L 以优先级 90 执行，释放锁
5. H 获得锁执行
6. M 最后执行

PI 避免了中优先级任务延迟高优先级任务。
</details>

**3. hrtimer 精度计算**
系统使用 TSC 作为时钟源（频率 2.4GHz），计算理论最高定时精度。

<details>
<summary>答案</summary>

TSC 周期 = 1 / 2.4GHz = 0.417ns

理论精度 = TSC 周期 = 0.417ns

实际精度受限于：
- 中断延迟（~1μs）
- 调度延迟（~10μs）
- 时钟事件设备编程开销（~100ns）

典型实际精度：1-10μs
</details>

**4. SCHED_DEADLINE 准入控制**
系统有两个 DEADLINE 任务：
- T1: runtime=2ms, period=10ms
- T2: runtime=3ms, period=15ms
问：能否再加入 T3: runtime=4ms, period=20ms？

<details>
<summary>答案</summary>

计算 CPU 利用率：
- U1 = 2/10 = 0.2
- U2 = 3/15 = 0.2
- U3 = 4/20 = 0.2

总利用率 = 0.2 + 0.2 + 0.2 = 0.6 < 1

答：可以加入 T3。系统总利用率 60%，满足 EDF 可调度条件。

注意：多核系统中每个 CPU 独立计算利用率。
</details>

### 挑战题

**5. 实时调度器设计**
设计一个混合调度器，同时支持周期任务和零星任务，要求：
- 周期任务使用 EDF
- 零星任务使用固定优先级
- 保证两类任务的隔离

*提示：考虑使用服务器（server）机制*

<details>
<summary>答案</summary>

混合调度器架构：

1. **CBS (Constant Bandwidth Server) 用于零星任务**
   - 为零星任务预留带宽
   - 转换为周期性预算补充

2. **层次化调度**
   ```
   根调度器（EDF）
   ├── 周期任务1
   ├── 周期任务2
   └── CBS服务器
       ├── 零星任务1（优先级）
       └── 零星任务2（优先级）
   ```

3. **准入控制**
   - 周期任务：Σ(Ci/Ti) ≤ α
   - CBS预算：Qs/Ts ≤ 1-α
   - α 是周期任务预留比例

4. **实现要点**
   - CBS 作为 EDF 任务参与调度
   - CBS 内部使用优先级队列
   - 预算耗尽时 CBS 暂停
</details>

**6. 延迟抖动分析**
实时任务每 10ms 执行一次，测量到延迟分布：
- 99%: < 50μs
- 0.9%: 50-100μs  
- 0.1%: > 1ms

分析可能的原因和优化方案。

<details>
<summary>答案</summary>

**延迟尖峰原因分析**：

1. **SMI (System Management Interrupt)**
   - 检测：hwlat_detector
   - 解决：BIOS 设置禁用 SMI

2. **CPU 频率调节**
   - 检测：/sys/devices/system/cpu/cpu*/cpufreq/
   - 解决：performance governor

3. **内存页错误**
   - 检测：缺页统计
   - 解决：mlockall() + 预分配

4. **缓存未命中**
   - 检测：perf stat -e cache-misses
   - 解决：缓存预热，NUMA 绑定

5. **中断风暴**
   - 检测：/proc/interrupts
   - 解决：中断合并，NAPI

**优化方案**：
```bash
# 1. CPU 配置
echo performance > /sys/devices/system/cpu/cpu2/cpufreq/scaling_governor
echo 0 > /sys/devices/system/cpu/cpu2/online  # 隔离 CPU

# 2. 中断绑定
echo 1 > /proc/irq/default_smp_affinity

# 3. 内核参数
nohz_full=2 rcu_nocbs=2 isolcpus=2

# 4. 应用优化
- 内存预分配
- 缓存预热循环
- 避免系统调用
```
</details>

**7. 实时补丁移植**
将 PREEMPT_RT 补丁移植到嵌入式 ARM 平台，需要考虑哪些架构相关问题？

<details>
<summary>答案</summary>

**ARM 架构特殊考虑**：

1. **中断控制器差异**
   - GIC vs NVIC
   - 中断优先级位数
   - FIQ vs IRQ 处理

2. **原子操作实现**
   - LDREX/STREX 指令
   - 内存屏障差异（DMB/DSB/ISB）
   - Thumb vs ARM 模式

3. **定时器架构**
   - Generic Timer vs SysTick
   - 时钟源精度（32.768kHz vs MHz）

4. **缓存架构**
   - 缓存行大小（32B vs 64B）
   - 缓存一致性协议

5. **移植步骤**
   ```
   1. 架构相关代码（arch/arm/）
      - 上下文切换
      - 中断入口/出口
      
   2. 平台驱动
      - 定时器驱动
      - 中断控制器
      
   3. 配置选项
      - Kconfig 依赖
      - 默认配置
      
   4. 测试验证
      - cyclictest
      - stress-ng
   ```

6. **常见问题**
   - FPU 上下文保存
   - Thumb-2 代码优化
   - 能耗管理冲突
</details>

**8. 形式化验证**
使用模型检查工具验证优先级继承协议的正确性，建立什么样的模型？

<details>
<summary>答案</summary>

**UPPAAL 时间自动机模型**：

1. **任务自动机**
   ```
   状态：Ready, Running, Blocked
   时钟：exec_time, deadline
   迁移：
   - Ready → Running [guard: highest_prio]
   - Running → Blocked [sync: lock_request]
   - Blocked → Ready [sync: lock_release]
   ```

2. **互斥锁自动机**
   ```
   状态：Free, Locked
   变量：owner, wait_queue
   迁移：
   - Free → Locked [sync: lock_request]
   - Locked → Free [sync: lock_release]
   ```

3. **验证属性（CTL）**
   ```
   // 无死锁
   A[] not deadlock
   
   // 优先级反转bounded
   A[] (Task_H.Blocked imply 
        Task_H.wait_time <= MAX_INVERSION)
   
   // 互斥性
   A[] not (Task1.cs and Task2.cs)
   
   // 实时性
   A[] (Task.exec_time <= Task.deadline)
   ```

4. **模型参数**
   - 任务数：3-5
   - 优先级：离散值
   - 执行时间：区间
   - 临界区：非确定选择

验证可发现死锁、无界优先级反转等问题。
</details>

## 常见陷阱与错误

### 1. RT 内核不等于硬实时
**陷阱**：认为使用 PREEMPT_RT 就能保证硬实时
```bash
# 错误想法：装了 RT 内核就是硬实时系统
```
**正解**：RT 内核只是降低延迟，硬实时需要：
- WCET 分析
- 资源预留
- 错误隔离
- 认证标准（DO-178C、IEC 61508）

### 2. 优先级设置错误
**陷阱**：随意设置实时优先级
```c
// 危险：最高优先级可能饿死系统任务
param.sched_priority = 99;
sched_setscheduler(0, SCHED_FIFO, &param);
```
**正解**：预留优先级空间：
- 99: 紧急中断线程
- 90-98: 关键系统服务
- 50-89: 实时应用
- 1-49: 软实时任务

### 3. 忽视内存延迟
**陷阱**：只关注 CPU 调度
```c
// 页错误导致毫秒级延迟
char *buffer = malloc(LARGE_SIZE);
memset(buffer, 0, LARGE_SIZE);  // 触发缺页
```
**正解**：完整内存管理：
```c
mlockall(MCL_CURRENT | MCL_FUTURE);
// 预分配并预触碰
posix_memalign(&buffer, PAGE_SIZE, size);
memset(buffer, 0, size);
```

### 4. 中断处理不当
**陷阱**：实时任务被中断风暴影响
```bash
# 网卡中断影响实时性
cat /proc/interrupts  # 每秒数万中断
```
**正解**：中断管理策略：
- 中断亲和性绑定
- NAPI 轮询模式
- 中断合并（ethtool -C）
- 中断线程化优先级调整

### 5. 调试工具使用不当
**陷阱**：使用 printk 调试实时代码
```c
// printk 可能导致数百微秒延迟
printk("Enter critical section\n");
```
**正解**：使用无锁追踪：
- ftrace 事件
- 循环缓冲区
- LTTng/SystemTap

## 最佳实践检查清单

### 系统配置
- [ ] 内核编译选项
  - CONFIG_PREEMPT_RT 启用
  - CONFIG_DEBUG_PREEMPT 调试期启用
  - CONFIG_NO_HZ_FULL 配置
- [ ] 启动参数设置
  - isolcpus 隔离 RT CPU
  - nohz_full 禁用时钟中断
  - rcu_nocbs 移除 RCU 回调
- [ ] BIOS/UEFI 配置
  - 禁用 SMI
  - 禁用 CPU 频率调节
  - 禁用 C-states

### 应用设计
- [ ] 实时任务初始化
  - 调度策略设置（FIFO/RR/DEADLINE）
  - CPU 亲和性绑定
  - 内存锁定（mlockall）
  - 栈预分配
- [ ] 资源管理
  - 避免动态内存分配
  - 使用内存池
  - 预分配所有资源
- [ ] 同步机制
  - 优先使用 rt_mutex
  - 避免优先级反转
  - 最小化临界区

### 性能监控
- [ ] 延迟测量
  - cyclictest 基准测试
  - 应用内延迟统计
  - 最坏情况记录
- [ ] 系统追踪
  - ftrace 配置
  - perf 事件监控
  - eBPF 探针部署
- [ ] 资源监控
  - CPU 利用率 < 70%
  - 内存锁定量检查
  - 中断频率监控

### 测试验证
- [ ] 功能测试
  - 所有实时约束满足
  - 降级模式正确
  - 错误恢复机制
- [ ] 压力测试
  - CPU 压力（stress-ng）
  - 内存压力
  - I/O 压力
  - 组合压力场景
- [ ] 长期稳定性
  - 72 小时连续运行
  - 内存泄漏检查
  - 优先级继承链检查

### 文档规范
- [ ] 设计文档
  - 任务模型定义
  - 时序图
  - WCET 分析
- [ ] 配置文档
  - 内核配置
  - 系统调优参数
  - 应用配置
- [ ] 运维手册
  - 监控指标
  - 故障诊断流程
  - 性能调优指南
```
