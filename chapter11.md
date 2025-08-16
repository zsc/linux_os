# 第11章：并发编程模型

## 本章导读

在现代多核处理器系统中，Linux 内核必须高效地处理并发执行的挑战。本章深入探讨内核中的并发编程模型，从内核抢占机制到无锁数据结构，从 per-CPU 变量到内存一致性模型。我们将分析内核如何在保证正确性的前提下，最大化利用硬件并行能力，减少锁竞争，优化缓存使用。通过学习本章，您将掌握内核并发编程的核心技术，理解如何在复杂的多处理器环境中编写高性能、可扩展的内核代码。

## 11.1 内核抢占与临界区

### 11.1.1 内核抢占的演进历程

Linux 内核抢占机制经历了从完全不可抢占到完全可抢占的演变过程。早期 Linux 内核（2.4 及之前）采用不可抢占设计，一旦进程进入内核态执行，必须主动让出 CPU 或返回用户态才能被调度。这种设计简化了内核同步，但严重影响了系统响应性。

2.6 版本引入了内核抢占机制，通过在 `thread_info` 结构中维护 `preempt_count` 计数器来控制抢占：

```
preempt_count 位域分配：
[31:28] - HARDIRQ    硬中断嵌套计数
[27:20] - SOFTIRQ    软中断嵌套计数  
[19:16] - NMI        NMI中断计数
[15:8]  - PREEMPT    抢占禁用计数
[7:0]   - 保留
```

当 `preempt_count` 为 0 时，表示内核可以被抢占。任何非零值都会禁止抢占，这包括：
- 持有自旋锁时（`spin_lock` 会增加计数）
- 处理中断时（硬中断和软中断都会增加对应计数）
- 显式禁用抢占时（`preempt_disable()` 增加 PREEMPT 域）

### 11.1.2 抢占点与调度时机

内核抢占发生在特定的抢占点（preemption points）：

1. **中断返回时**：从中断处理程序返回内核态时，检查是否需要调度
2. **解锁时**：释放自旋锁后，如果 `preempt_count` 降为 0 且有更高优先级任务等待
3. **显式抢占点**：调用 `cond_resched()` 或 `might_sleep()` 时
4. **系统调用返回**：从系统调用返回用户态前

抢占检查的核心逻辑：
```
need_resched = test_tsk_need_resched(current)
can_preempt = (preempt_count == 0) && !irqs_disabled()
should_preempt = need_resched && can_preempt
```

### 11.1.3 临界区保护策略

内核提供多层次的临界区保护机制：

**1. 禁用抢占（最轻量级）**
适用于纯内核态临界区，不涉及中断处理：
- `preempt_disable()` / `preempt_enable()`
- 开销最小，仅修改 per-CPU 变量
- 不影响中断，仍可被中断打断

**2. 禁用软中断**
保护软中断上下文共享的数据：
- `local_bh_disable()` / `local_bh_enable()`
- 禁止软中断和下半部执行
- 硬中断仍可执行

**3. 禁用硬中断（最重量级）**
完全的原子操作保护：
- `local_irq_disable()` / `local_irq_enable()`
- `local_irq_save(flags)` / `local_irq_restore(flags)`
- 禁止所有中断，影响系统响应性

### 11.1.4 抢占模型配置

Linux 内核支持多种抢占模型，通过编译选项配置：

**CONFIG_PREEMPT_NONE（服务器模式）**
- 传统非抢占内核
- 最大吞吐量，最差响应延迟
- 适合批处理和高性能计算

**CONFIG_PREEMPT_VOLUNTARY（桌面模式）**
- 在关键路径插入显式抢占点
- 平衡吞吐量和响应性
- 默认配置选项

**CONFIG_PREEMPT（低延迟模式）**
- 完全可抢占内核
- 最佳响应性，略有性能损失
- 适合交互式系统

**CONFIG_PREEMPT_RT（实时模式）**
- PREEMPT_RT 补丁集
- 将自旋锁转换为可睡眠锁
- 硬实时保证，显著性能开销

### 11.1.5 抢占调试与分析

内核提供了丰富的抢占调试工具：

**preempt_count 追踪**
通过 `/proc/[pid]/status` 查看进程的抢占计数，异常值表示可能的死锁或泄漏。

**preemptirqsoff 跟踪器**
使用 ftrace 追踪最长的抢占/中断禁用区间：
```bash
echo preemptirqsoff > /sys/kernel/debug/tracing/current_tracer
echo 1 > /sys/kernel/debug/tracing/tracing_on
```

**调度延迟统计**
`/proc/sys/kernel/sched_latency_ns` 控制目标调度延迟，过小会增加上下文切换开销。

## 11.2 per-CPU 变量与本地化

### 11.2.1 per-CPU 变量原理

per-CPU 变量是内核中重要的并发优化技术，为每个 CPU 分配独立的变量副本，避免缓存行竞争和锁开销。其核心思想是用空间换时间，将共享数据转换为 CPU 本地数据。

内存布局示意：
```
CPU0 区域: [base + 0]      变量A | 变量B | 变量C | ...
CPU1 区域: [base + offset1] 变量A | 变量B | 变量C | ...
CPU2 区域: [base + offset2] 变量A | 变量B | 变量C | ...
...
```

每个 CPU 通过 `gs` 段寄存器（x86-64）或专用寄存器存储自己的 per-CPU 基址偏移。

### 11.2.2 静态与动态 per-CPU 变量

**静态定义**
使用 `DEFINE_PER_CPU` 宏在编译时分配：
```c
DEFINE_PER_CPU(int, cpu_counter);
DEFINE_PER_CPU_SHARED_ALIGNED(struct mystruct, shared_data);
```

**动态分配**
运行时通过 `alloc_percpu` 分配：
```c
int *counters = alloc_percpu(int);
struct mystruct *data = alloc_percpu_gfp(struct mystruct, GFP_KERNEL);
```

**访问方法**
必须在禁用抢占的情况下访问 per-CPU 变量：
```c
preempt_disable();
this_cpu_inc(cpu_counter);        // 原子递增当前CPU的变量
*this_cpu_ptr(&shared_data) = x;  // 访问当前CPU的结构体
preempt_enable();
```

### 11.2.3 per-CPU 操作优化

现代处理器提供了特殊指令优化 per-CPU 操作：

**this_cpu_ops 系列**
- `this_cpu_read(var)` - 读取
- `this_cpu_write(var, val)` - 写入
- `this_cpu_add(var, val)` - 原子加法
- `this_cpu_cmpxchg(var, old, new)` - 比较交换

这些操作在 x86 上编译为单指令，利用 `gs` 段前缀直接访问。

**批量操作优化**
对于需要访问所有 CPU 数据的场景：
```c
for_each_possible_cpu(cpu) {
    int *ptr = per_cpu_ptr(counters, cpu);
    total += *ptr;
}
```

### 11.2.4 NUMA 感知的 per-CPU 分配

在 NUMA 系统中，per-CPU 变量分配考虑内存节点亲和性：

```
NUMA节点0: CPU0-3 的 per-CPU 区域
NUMA节点1: CPU4-7 的 per-CPU 区域
```

内核通过 `cpu_to_node()` 映射确保 per-CPU 数据分配在对应 CPU 的本地内存节点，减少跨节点访问延迟。

### 11.2.5 per-CPU 变量的典型应用

**1. 统计计数器**
网络包统计、系统调用计数等高频更新的计数器。

**2. 缓存池**
内存分配器的 per-CPU 缓存，如 SLUB 的 per-CPU partial 链表。

**3. 延迟处理队列**
RCU 回调链表、工作队列的 per-CPU 工作池。

**4. 中断上下文数据**
中断栈、软中断待处理标志等。

## 11.3 无锁数据结构

### 11.3.1 无锁编程基础

无锁（lock-free）编程通过原子操作和精心设计的算法，实现多线程并发访问而无需传统锁。其核心优势是避免了锁带来的开销：上下文切换、优先级反转、死锁风险。

Linux 内核中的原子操作基础：
- **原子整数操作**：`atomic_t`、`atomic64_t`
- **原子位操作**：`set_bit()`、`clear_bit()`、`test_and_set_bit()`
- **比较交换**：`cmpxchg()`、`try_cmpxchg()`
- **内存屏障**：`smp_mb()`、`smp_rmb()`、`smp_wmb()`

### 11.3.2 无锁链表实现

内核中的无锁单链表（`llist`）实现：

```
结构定义：
struct llist_node {
    struct llist_node *next;
};

struct llist_head {
    struct llist_node *first;
};
```

**无锁入队（压栈）操作**：
```c
bool llist_add(struct llist_node *new, struct llist_head *head) {
    struct llist_node *first;
    do {
        first = READ_ONCE(head->first);
        new->next = first;
    } while (cmpxchg(&head->first, first, new) != first);
    return first == NULL;
}
```

**批量出队操作**：
```c
struct llist_node *llist_del_all(struct llist_head *head) {
    return xchg(&head->first, NULL);
}
```

这种设计适合单生产者-单消费者（SPSC）或多生产者-单消费者（MPSC）场景。

### 11.3.3 无锁环形缓冲区

内核的 `kfifo` 实现了高效的无锁环形缓冲区：

```
核心数据结构：
struct kfifo {
    unsigned char *buffer;  // 缓冲区指针
    unsigned int size;      // 缓冲区大小（2的幂）
    unsigned int in;        // 写入位置
    unsigned int out;       // 读取位置
};
```

**关键设计点**：
1. 大小必须是 2 的幂，利用无符号整数溢出特性
2. 单生产者-单消费者模型，无需锁
3. 使用内存屏障保证顺序

**写入操作**：
```c
unsigned int kfifo_in(struct kfifo *fifo, const void *buf, unsigned int len) {
    unsigned int l;
    len = min(len, fifo->size - (fifo->in - fifo->out));
    
    // 复制数据到环形缓冲区
    l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), buf, l);
    memcpy(fifo->buffer, buf + l, len - l);
    
    smp_wmb(); // 确保数据写入完成后才更新in
    fifo->in += len;
    return len;
}
```

### 11.3.4 RCU 保护的数据结构

RCU（Read-Copy-Update）是 Linux 内核中最重要的无锁同步机制：

**RCU 链表遍历**（读端无锁）：
```c
rcu_read_lock();
list_for_each_entry_rcu(pos, &head, list) {
    // 安全访问 pos，不会被释放
    process(pos);
}
rcu_read_unlock();
```

**RCU 保护的更新**：
```c
struct foo *new = kmalloc(sizeof(*new), GFP_KERNEL);
*new = *old;
new->field = new_value;

rcu_assign_pointer(global_ptr, new);
synchronize_rcu();  // 等待所有读者完成
kfree(old);
```

### 11.3.5 无锁队列与栈

**Michael & Scott 队列算法**的简化实现：
```c
struct msqueue {
    struct node *head;
    struct node *tail;
};

void enqueue(struct msqueue *q, struct node *new) {
    struct node *tail, *next;
    new->next = NULL;
    
    for (;;) {
        tail = READ_ONCE(q->tail);
        next = READ_ONCE(tail->next);
        
        if (tail != q->tail) continue;
        
        if (next != NULL) {
            // 尾部不是真正的尾，尝试推进
            cmpxchg(&q->tail, tail, next);
            continue;
        }
        
        // 尝试链接新节点
        if (cmpxchg(&tail->next, NULL, new) == NULL)
            break;
    }
    
    // 尝试推进尾指针
    cmpxchg(&q->tail, tail, new);
}
```

### 11.3.6 无锁编程的挑战

**ABA 问题**
指针在 A→B→A 变化后，CAS 操作无法检测到中间状态。解决方案：
- 使用版本号或标记指针
- 使用 hazard pointers
- 依赖 RCU 的延迟回收

**内存回收**
无锁数据结构的最大挑战是安全回收内存。Linux 内核主要使用 RCU 机制解决。

**正确性验证**
无锁算法极易出错，需要：
- 形式化验证
- 压力测试
- 内存模型分析工具

## 11.4 内存模型与一致性

### 11.4.1 Linux 内核内存模型（LKMM）

Linux 内核内存模型定义了并发内存操作的语义，规定了：
- 内存操作的可见性规则
- 编译器和 CPU 的重排序限制
- 同步原语的语义保证

LKMM 采用 "happens-before" 关系定义操作顺序：
```
A happens-before B 意味着：
- A 的效果对 B 可见
- A 和 B 之间存在因果关系
```

### 11.4.2 内存屏障类型与语义

**编译器屏障**
`barrier()` 阻止编译器重排序，但不影响 CPU：
```c
#define barrier() __asm__ __volatile__("": : :"memory")
```

**CPU 内存屏障**
- `smp_mb()` - 全屏障，禁止所有重排序
- `smp_rmb()` - 读屏障，禁止读-读重排序
- `smp_wmb()` - 写屏障，禁止写-写重排序
- `smp_read_barrier_depends()` - 地址依赖屏障（多数架构为空）

**获取-释放语义**
- `smp_load_acquire()` - 获取语义的读
- `smp_store_release()` - 释放语义的写

这对操作建立同步关系：release 之前的操作对 acquire 之后可见。

### 11.4.3 缓存一致性协议

现代处理器使用 MESI（或其变种）协议维护缓存一致性：

```
状态转换：
M (Modified)   - 独占且已修改
E (Exclusive)  - 独占未修改
S (Shared)     - 共享只读
I (Invalid)    - 无效

缓存行状态机：
I --读--> E/S
E --写--> M
S --写--> M (需要使其他副本无效)
M --其他CPU读--> S
```

**伪共享问题**
不同 CPU 访问同一缓存行的不同变量导致性能下降：
```c
struct bad {
    int cpu0_data;  // CPU0 频繁更新
    int cpu1_data;  // CPU1 频繁更新
}; // 两个变量在同一缓存行

struct good {
    int cpu0_data;
    char pad[CACHE_LINE_SIZE - sizeof(int)];
    int cpu1_data;
} __aligned(CACHE_LINE_SIZE);
```

### 11.4.4 内存序与原子操作

C11/C++11 内存序在内核中的对应：

```
memory_order_relaxed  -> READ_ONCE/WRITE_ONCE
memory_order_acquire  -> smp_load_acquire
memory_order_release  -> smp_store_release  
memory_order_seq_cst  -> smp_mb() + 原子操作
```

**原子操作的内存序保证**：
```c
atomic_add(&v, 1);           // 默认全序
atomic_add_return(&v, 1);    // 全屏障语义
atomic_add_relaxed(&v, 1);   // 无序，仅原子性
```

### 11.4.5 NUMA 架构的一致性挑战

NUMA 系统中，跨节点内存访问带来额外的一致性开销：

```
本地访问延迟: ~10ns
远程访问延迟: ~20-100ns

缓存目录协议：
Node0 目录 --> 跟踪 Node0 内存的缓存位置
Node1 目录 --> 跟踪 Node1 内存的缓存位置
```

**NUMA 优化策略**：
1. **节点本地分配**：`alloc_pages_node()`
2. **CPU 亲和性**：将线程绑定到数据所在节点
3. **复制而非共享**：per-node 数据结构
4. **批量传输**：减少跨节点同步频率

### 11.4.6 内存模型验证工具

**litmus 测试**
使用 herd7 工具验证特定的内存访问模式：
```c
C example
{
    int x = 0;
    int y = 0;
}

P0(int *x, int *y) {
    WRITE_ONCE(*x, 1);
    smp_wmb();
    WRITE_ONCE(*y, 1);
}

P1(int *x, int *y) {
    int r1 = READ_ONCE(*y);
    smp_rmb();
    int r2 = READ_ONCE(*x);
}

exists (1:r1=1 /\ 1:r2=0)  // 这种结果可能吗？
```

**KCSAN（Kernel Concurrency Sanitizer）**
运行时检测数据竞争：
```bash
CONFIG_KCSAN=y
CONFIG_KCSAN_REPORT_ONCE_IN_MS=3000
```

## 本章小结

本章深入探讨了 Linux 内核的并发编程模型，涵盖了从基础的抢占机制到高级的无锁数据结构。关键要点包括：

**内核抢占机制**
- `preempt_count` 计数器控制抢占时机，包含硬中断、软中断、NMI 和抢占禁用计数
- 抢占点包括中断返回、解锁、显式抢占点和系统调用返回
- 提供多层次临界区保护：禁用抢占、禁用软中断、禁用硬中断
- 支持四种抢占模型：NONE、VOLUNTARY、PREEMPT、RT

**per-CPU 变量技术**
- 为每个 CPU 分配独立变量副本，消除缓存行竞争
- 静态（`DEFINE_PER_CPU`）和动态（`alloc_percpu`）两种分配方式
- 必须在禁用抢占下访问，使用 `this_cpu_ops` 系列操作
- NUMA 系统中考虑内存节点亲和性，减少跨节点访问

**无锁数据结构**
- 基于原子操作（`atomic_t`、`cmpxchg`）和内存屏障构建
- 内核实现了无锁链表（`llist`）、环形缓冲区（`kfifo`）
- RCU 机制提供高效的读端无锁访问
- 需要解决 ABA 问题和内存安全回收挑战

**内存模型与一致性**
- LKMM 定义了内存操作的可见性和因果关系
- 提供多种内存屏障：编译器屏障、CPU 屏障、获取-释放语义
- MESI 协议维护缓存一致性，需注意伪共享问题
- NUMA 架构需要特殊优化策略减少远程访问

**关键公式与度量**

1. **抢占延迟计算**：
   $$T_{preempt} = T_{disable} + T_{critical} + T_{enable}$$
   其中 $T_{disable}$ 和 $T_{enable}$ 通常为常数时间

2. **缓存行伪共享开销**：
   $$Overhead = N_{invalidations} \times (T_{invalidate} + T_{refetch})$$
   
3. **NUMA 访问延迟比**：
   $$R_{NUMA} = \frac{T_{remote}}{T_{local}} \approx 2-10$$

4. **无锁算法的进展保证**：
   - **Wait-free**：每个操作在有限步内完成
   - **Lock-free**：系统整体持续进展
   - **Obstruction-free**：单独运行时有限步完成

## 练习题

### 基础题

**练习 11.1：抢占计数分析**
分析以下代码片段的 `preempt_count` 值变化：
```c
spin_lock(&lock1);        // (1)
local_bh_disable();       // (2)  
spin_lock(&lock2);        // (3)
spin_unlock(&lock2);      // (4)
local_bh_enable();        // (5)
spin_unlock(&lock1);      // (6)
```
问：在每个标记点，`preempt_count` 的值是多少？假设初始值为 0。

*Hint*：`spin_lock` 增加 1，`local_bh_disable` 增加 0x100（SOFTIRQ 域）。

<details>
<summary>参考答案</summary>

- (1) 后：0x00000001（PREEMPT 域 +1）
- (2) 后：0x00000101（SOFTIRQ 域 +1）  
- (3) 后：0x00000102（PREEMPT 域再 +1）
- (4) 后：0x00000101（PREEMPT 域 -1）
- (5) 后：0x00000001（SOFTIRQ 域 -1）
- (6) 后：0x00000000（PREEMPT 域 -1）

关键理解：不同域独立计数，`spin_lock` 只影响 PREEMPT 域。

</details>

**练习 11.2：per-CPU 变量访问**
以下代码有什么问题？如何修正？
```c
DEFINE_PER_CPU(int, my_counter);

void increment_counter(void) {
    int cpu = smp_processor_id();
    per_cpu(my_counter, cpu)++;
}
```

*Hint*：考虑抢占发生的时机。

<details>
<summary>参考答案</summary>

问题：获取 CPU ID 和访问 per-CPU 变量之间可能被抢占，导致在错误的 CPU 上执行递增。

修正方案：
```c
void increment_counter(void) {
    preempt_disable();
    this_cpu_inc(my_counter);
    preempt_enable();
}
// 或使用更简洁的：
void increment_counter(void) {
    this_cpu_inc(my_counter);  // 内部已处理抢占
}
```

</details>

**练习 11.3：内存屏障选择**
为以下场景选择合适的内存屏障：
1. 确保数据写入后再设置标志位
2. 确保读取标志位后再读取数据
3. 确保两个写操作的顺序
4. 在自旋锁的实现中

*Hint*：考虑单向屏障和双向屏障的区别。

<details>
<summary>参考答案</summary>

1. `smp_wmb()` 或 `smp_store_release()`（写屏障）
2. `smp_rmb()` 或 `smp_load_acquire()`（读屏障）
3. `smp_wmb()`（写-写屏障）
4. `smp_mb()`（全屏障，锁的获取和释放需要全序）

选择原则：尽量使用最轻量的屏障满足需求，acquire-release 对适合生产者-消费者模式。

</details>

### 挑战题

**练习 11.4：无锁计数器设计**
设计一个无锁的全局计数器，支持：
- 多个 CPU 并发递增
- 读取全局总和（可以是近似值）
- 最小化缓存行竞争

*Hint*：结合 per-CPU 变量和原子操作。

<details>
<summary>参考答案</summary>

设计思路：
1. 使用 per-CPU 计数器减少竞争
2. 定期或按需聚合到全局计数器
3. 读取时遍历所有 per-CPU 值

```c
DEFINE_PER_CPU(unsigned long, local_counter);
atomic64_t global_counter;

void increment(void) {
    unsigned long *ptr = this_cpu_ptr(&local_counter);
    (*ptr)++;
    
    // 每 1024 次更新全局计数器
    if ((*ptr & 0x3ff) == 0) {
        atomic64_add(1024, &global_counter);
        *ptr -= 1024;
    }
}

unsigned long read_counter(void) {
    unsigned long sum = atomic64_read(&global_counter);
    int cpu;
    
    for_each_possible_cpu(cpu) {
        sum += per_cpu(local_counter, cpu);
    }
    return sum;
}
```

</details>

**练习 11.5：RCU 链表遍历竞态**
分析以下 RCU 保护的链表遍历代码，找出潜在问题：
```c
struct foo {
    struct list_head list;
    int data;
    struct bar *ptr;
};

void reader(void) {
    struct foo *entry;
    
    rcu_read_lock();
    list_for_each_entry_rcu(entry, &head, list) {
        if (entry->data > 100) {
            process(entry->ptr->value);  // 这里安全吗？
        }
    }
    rcu_read_unlock();
}
```

*Hint*：考虑 `entry->ptr` 指向的对象生命周期。

<details>
<summary>参考答案</summary>

问题：`entry->ptr` 指向的 `bar` 对象可能不受 RCU 保护，访问 `entry->ptr->value` 可能导致使用已释放内存。

解决方案：
1. 确保 `bar` 对象也通过 RCU 延迟释放
2. 或在 `foo` 结构中嵌入 `bar` 数据
3. 使用引用计数保护 `bar` 对象

修正代码：
```c
void reader(void) {
    struct foo *entry;
    struct bar *bar;
    
    rcu_read_lock();
    list_for_each_entry_rcu(entry, &head, list) {
        if (entry->data > 100) {
            bar = rcu_dereference(entry->ptr);
            if (bar) {
                process(bar->value);
            }
        }
    }
    rcu_read_unlock();
}
```

</details>

**练习 11.6：缓存行对齐优化**
某内核模块定义了如下结构：
```c
struct stats {
    atomic_t rx_packets;
    atomic_t rx_bytes;
    atomic_t tx_packets;
    atomic_t tx_bytes;
    spinlock_t lock;
    unsigned long last_update;
};
```
网络接收路径频繁更新 `rx_*`，发送路径频繁更新 `tx_*`。如何优化以减少缓存行竞争？

*Hint*：计算各字段大小，考虑缓存行边界。

<details>
<summary>参考答案</summary>

优化方案：将频繁并发访问的字段分离到不同缓存行：

```c
struct stats {
    // RX 路径专用缓存行
    atomic_t rx_packets;
    atomic_t rx_bytes;
    char pad1[CACHE_LINE_SIZE - 2*sizeof(atomic_t)];
    
    // TX 路径专用缓存行  
    atomic_t tx_packets;
    atomic_t tx_bytes;
    char pad2[CACHE_LINE_SIZE - 2*sizeof(atomic_t)];
    
    // 管理字段单独缓存行
    spinlock_t lock;
    unsigned long last_update;
} __aligned(CACHE_LINE_SIZE);
```

或使用 per-CPU 统计进一步优化：
```c
struct stats {
    DEFINE_PER_CPU_ALIGNED(atomic_t, rx_packets);
    DEFINE_PER_CPU_ALIGNED(atomic_t, rx_bytes);
    DEFINE_PER_CPU_ALIGNED(atomic_t, tx_packets);
    DEFINE_PER_CPU_ALIGNED(atomic_t, tx_bytes);
};
```

</details>

**练习 11.7：内存序分析**
分析以下代码在弱内存序架构（如 ARM）上的行为：
```c
int data = 0;
int flag = 0;

// CPU 0
void producer(void) {
    data = 42;
    flag = 1;
}

// CPU 1
void consumer(void) {
    while (flag == 0);
    use(data);
}
```
问：`consumer` 看到的 `data` 值一定是 42 吗？如何保证？

*Hint*：考虑编译器和 CPU 重排序。

<details>
<summary>参考答案</summary>

不一定。存在两个问题：
1. 编译器可能重排序 `producer` 中的写操作
2. CPU 可能重排序或延迟可见性

修正方案使用 acquire-release 语义：
```c
int data = 0;
int flag = 0;

// CPU 0
void producer(void) {
    data = 42;
    smp_store_release(&flag, 1);
}

// CPU 1  
void consumer(void) {
    while (smp_load_acquire(&flag) == 0);
    use(data);  // 保证看到 data = 42
}
```

或使用显式屏障：
```c
// CPU 0
void producer(void) {
    data = 42;
    smp_wmb();  // 确保 data 写入在 flag 之前
    WRITE_ONCE(flag, 1);
}

// CPU 1
void consumer(void) {
    while (READ_ONCE(flag) == 0);
    smp_rmb();  // 确保 flag 读取在 data 之前
    use(data);
}
```

</details>

**练习 11.8：NUMA 优化策略**
某服务器有 2 个 NUMA 节点，各 32 个 CPU。一个网络处理模块需要维护连接状态表。设计数据结构和访问策略，优化 NUMA 访问。

*Hint*：考虑连接的亲和性和负载均衡。

<details>
<summary>参考答案</summary>

优化策略：

1. **分区哈希表**：每个 NUMA 节点独立哈希表
```c
struct conn_table {
    struct hlist_head *buckets;
    spinlock_t *locks;
    int node_id;
} __cacheline_aligned;

struct conn_table conn_tables[MAX_NUMA_NODES];

// 初始化
for_each_online_node(node) {
    conn_tables[node].buckets = alloc_pages_node(node, ...);
    conn_tables[node].locks = alloc_percpu_on_node(node, ...);
}
```

2. **连接亲和性**：根据连接特征分配到特定节点
```c
int get_conn_node(struct connection *conn) {
    // 基于源 IP 哈希到 NUMA 节点
    return hash_32(conn->src_ip, NUMA_SHIFT) % nr_numa_nodes;
}
```

3. **CPU 绑定**：网卡中断和处理线程绑定到同一节点
```c
// 中断亲和性
echo $NODE0_CPUS > /proc/irq/$IRQ/smp_affinity

// 工作线程 CPU 亲和性
CPU_SET(cpu, &cpuset);
pthread_setaffinity_np(thread, sizeof(cpuset), &cpuset);
```

4. **批量处理**：减少跨节点同步
```c
// 本地批量处理后再同步到远程节点
if (local_batch_size >= BATCH_THRESHOLD) {
    flush_to_remote_node();
}
```

</details>

## 常见陷阱与错误

### 1. 抢占相关错误

**错误：抢占计数泄漏**
```c
void buggy_function(void) {
    preempt_disable();
    if (error_condition)
        return;  // 忘记 preempt_enable()！
    preempt_enable();
}
```
**后果**：系统最终死锁，调度器无法运行。

**调试方法**：
- 检查 `/proc/[pid]/status` 中的 preempt 计数
- 使用 `CONFIG_DEBUG_PREEMPT` 检测不平衡

### 2. per-CPU 变量误用

**错误：跨 CPU 访问**
```c
int cpu = smp_processor_id();
// ... 可能被抢占迁移到其他 CPU ...
per_cpu(var, cpu)++;  // 访问错误的 CPU 变量！
```

**错误：未禁用抢占访问**
```c
this_cpu_ptr(&data)->field = value;  // 可能在访问中被抢占
```

**正确做法**：使用 `get_cpu()`/`put_cpu()` 或 `this_cpu_*` 原语。

### 3. 无锁编程陷阱

**ABA 问题示例**
```c
// 线程 1 读到 A
old = head;  // head = A
// 线程 2: A->B->A 的变化
// 线程 1 CAS 成功但状态已变
cmpxchg(&head, old, new);  // 错误地认为没有变化
```

**内存序错误**
```c
// 错误：缺少内存屏障
data = 1;
flag = 1;  // 其他 CPU 可能先看到 flag=1
```

### 4. 缓存行伪共享

**性能杀手示例**
```c
struct bad_design {
    int cpu0_counter;  // 频繁更新
    int cpu1_counter;  // 频繁更新，同一缓存行！
};
```
**症状**：性能急剧下降，缓存未命中率高。

### 5. RCU 误用

**错误：过早释放**
```c
rcu_assign_pointer(global_ptr, new);
kfree(old);  // 错误！还有读者在访问
// 正确：synchronize_rcu() 或 call_rcu()
```

**错误：长时间持有 rcu_read_lock**
```c
rcu_read_lock();
msleep(1000);  // 错误！阻塞 RCU grace period
rcu_read_unlock();
```

### 6. 内存屏障遗漏

**错误：初始化竞态**
```c
obj->field = value;
published = obj;  // 其他 CPU 可能看到未初始化的 field
```

**正确版本**：
```c
obj->field = value;
smp_wmb();  // 或使用 smp_store_release()
published = obj;
```

## 最佳实践检查清单

### 设计阶段

- [ ] **明确并发需求**：是否真的需要并发？能否简化为单线程？
- [ ] **选择合适的同步原语**：
  - 低竞争：自旋锁
  - 可能睡眠：互斥锁
  - 读多写少：RCU 或读写锁
  - 高频计数：per-CPU 变量
- [ ] **数据结构布局**：
  - 频繁访问的字段放在结构开头
  - 并发访问的字段分离到不同缓存行
  - 只读字段标记为 `const`
- [ ] **NUMA 考虑**：数据本地性、CPU 亲和性设置

### 实现阶段

- [ ] **抢占管理**：
  - `preempt_disable()`/`preempt_enable()` 配对
  - 临界区尽可能短
  - 避免在禁用抢占时睡眠
- [ ] **per-CPU 变量**：
  - 使用 `this_cpu_*` 操作
  - 避免跨 CPU 访问
  - 初始化时考虑 CPU 热插拔
- [ ] **无锁算法**：
  - 使用 `READ_ONCE`/`WRITE_ONCE` 防止编译器优化
  - 正确放置内存屏障
  - 处理 ABA 问题
  - 考虑弱内存序架构
- [ ] **RCU 使用**：
  - 读端不能睡眠
  - 正确的 grace period 等待
  - 避免过度延迟回收

### 测试阶段

- [ ] **并发测试**：
  - 多 CPU 压力测试
  - CPU 热插拔测试
  - 抢占点注入测试
- [ ] **性能分析**：
  - 缓存未命中率（`perf stat`）
  - 锁竞争分析（`lock_stat`）
  - 调度延迟（`schedstat`）
- [ ] **正确性验证**：
  - KCSAN 数据竞争检测
  - lockdep 死锁检测
  - RCU stall 检测

### 优化阶段

- [ ] **减少锁竞争**：
  - 细粒度锁
  - 无锁数据结构
  - per-CPU 化
- [ ] **缓存优化**：
  - 消除伪共享
  - 数据预取
  - 批量操作
- [ ] **NUMA 优化**：
  - 节点本地分配
  - 减少跨节点访问
  - 中断亲和性调优

### 文档与维护

- [ ] **并发语义文档化**：
  - 锁顺序约定
  - 内存序要求
  - RCU 保护范围
- [ ] **性能基准**：
  - 建立性能基线
  - 回归测试
  - 扩展性测试
- [ ] **调试信息**：
  - 添加 tracepoints
  - 统计计数器
  - debugfs 接口
