# 第10章：内核同步机制

## 章节大纲

1. **开篇介绍**
2. **原子操作与内存屏障**
   - 原子整数操作
   - 原子位操作
   - 内存屏障类型与使用
3. **自旋锁家族**
   - 基本自旋锁（spinlock）
   - 读写自旋锁（rwlock）
   - 顺序锁（seqlock）
4. **睡眠锁机制**
   - 互斥量（mutex）
   - 信号量（semaphore）
   - 完成量（completion）
5. **RCU机制深度解析**
   - RCU基本原理
   - RCU实现细节
   - RCU使用模式
6. **正确性保证工具**
   - lockdep死锁检测
   - 锁依赖图分析
   - 运行时锁验证
7. **本章小结**
8. **练习题**
9. **常见陷阱与错误**
10. **最佳实践检查清单**

---

## 10.1 开篇介绍

在多处理器系统中，内核同步机制是保证数据一致性和正确性的基石。Linux内核作为一个可抢占、支持SMP的操作系统，需要精心设计的同步原语来协调并发执行的内核路径。本章深入剖析Linux内核的同步机制，从最底层的原子操作到高层的RCU机制，揭示内核如何在性能与正确性之间取得平衡。

本章学习目标：
- 理解各种同步原语的实现原理和适用场景
- 掌握内存屏障的语义和正确使用方法
- 深入理解RCU的设计哲学和实现机制
- 学会使用lockdep等工具进行死锁检测和调试
- 建立选择合适同步机制的决策框架

## 10.2 原子操作与内存屏障

### 10.2.1 原子整数操作

Linux内核提供了一套完整的原子操作接口，这些操作保证在多处理器环境下的原子性。原子操作是构建更高级同步原语的基础。

```
typedef struct {
    int counter;
} atomic_t;

typedef struct {
    long counter;
} atomic64_t;
```

原子操作的核心实现依赖于处理器提供的原子指令。在x86架构上，主要使用LOCK前缀和XCHG、CMPXCHG等指令：

```
static inline void atomic_add(int i, atomic_t *v)
{
    asm volatile(LOCK_PREFIX "addl %1,%0"
                 : "+m" (v->counter)
                 : "ir" (i));
}

static inline int atomic_cmpxchg(atomic_t *v, int old, int new)
{
    return cmpxchg(&v->counter, old, new);
}
```

原子操作的语义保证：
- **原子性**：操作要么完全执行，要么完全不执行
- **有序性**：某些原子操作提供内存序保证
- **可见性**：操作结果对所有CPU立即可见

### 10.2.2 原子位操作

位操作在内核中广泛用于标志管理和位图操作：

```
set_bit(nr, addr)       // 原子设置第nr位
clear_bit(nr, addr)     // 原子清除第nr位
test_bit(nr, addr)      // 测试第nr位
test_and_set_bit(nr, addr)  // 测试并设置
```

这些操作在实现上使用了处理器的BTS（Bit Test and Set）、BTR（Bit Test and Reset）等指令，配合LOCK前缀实现原子性。

### 10.2.3 内存屏障类型与使用

内存屏障是控制内存访问顺序的关键机制。Linux内核定义了多种类型的屏障：

**编译器屏障**：
```
barrier()  // 阻止编译器重排序
```

**CPU内存屏障**：
```
mb()    // 全屏障：读写都不能跨越
rmb()   // 读屏障：读操作不能跨越
wmb()   // 写屏障：写操作不能跨越
smp_mb()   // SMP系统的全屏障
smp_rmb()  // SMP系统的读屏障
smp_wmb()  // SMP系统的写屏障
```

**数据依赖屏障**：
```
read_barrier_depends()  // Alpha架构特有
smp_read_barrier_depends()
```

内存屏障的典型使用模式：

```
// 生产者-消费者模式
// 生产者：
data = value;
smp_wmb();  // 确保data写入在flag之前
flag = 1;

// 消费者：
while (!flag)
    cpu_relax();
smp_rmb();  // 确保读data在读flag之后
use(data);
```

### 10.2.4 内存序模型

Linux内核采用的内存序模型考虑了不同架构的特性：

- **x86/x86-64**：强内存序（TSO），大部分操作自然有序
- **ARM/PowerPC**：弱内存序，需要显式屏障
- **Alpha**：最弱内存序，甚至数据依赖都需要屏障

内核通过抽象层屏蔽了这些差异，但理解底层差异对性能优化至关重要。

## 10.3 自旋锁家族

### 10.3.1 基本自旋锁（spinlock）

自旋锁是内核中最基本的同步原语，适用于短期持有的临界区保护：

```
typedef struct {
    arch_spinlock_t raw_lock;
} spinlock_t;

spin_lock(&lock);
// 临界区
spin_unlock(&lock);

// 禁用中断的版本
spin_lock_irqsave(&lock, flags);
// 临界区
spin_unlock_irqrestore(&lock, flags);
```

自旋锁的实现演进经历了多个阶段：

**1. 简单测试-设置锁（早期）**：
```
while (test_and_set(&lock))
    cpu_relax();
```

**2. Ticket锁（2.6.25引入）**：
```
typedef struct {
    union {
        u32 head_tail;
        struct {
            u16 head;
            u16 tail;
        };
    };
} arch_spinlock_t;
```

Ticket锁保证了FIFO顺序，避免了饥饿问题。

**3. MCS锁和队列自旋锁（3.15引入）**：
```
struct mcs_spinlock {
    struct mcs_spinlock *next;
    int locked;
};
```

队列自旋锁减少了缓存行争用，提高了NUMA系统性能。

### 10.3.2 读写自旋锁（rwlock）

读写锁允许多个读者同时访问，但写者独占：

```
rwlock_t lock;

// 读者
read_lock(&lock);
// 读临界区
read_unlock(&lock);

// 写者
write_lock(&lock);
// 写临界区
write_unlock(&lock);
```

读写锁的实现使用计数器跟踪读者数量：
- 正数表示读者数量
- 负数表示有写者
- 零表示无人持有

读写锁存在的问题：
- **写者饥饿**：持续的读者可能导致写者永远无法获得锁
- **缓存行乒乓**：读者增减计数器导致缓存行在CPU间移动

### 10.3.3 顺序锁（seqlock）

顺序锁优化了读多写少的场景，读者不会阻塞写者：

```
typedef struct {
    struct seqcount seqcount;
    spinlock_t lock;
} seqlock_t;

// 写者
write_seqlock(&sl);
// 更新数据
write_sequnlock(&sl);

// 读者
unsigned seq;
do {
    seq = read_seqbegin(&sl);
    // 读取数据
} while (read_seqretry(&sl, seq));
```

顺序锁的核心思想：
1. 写者增加序列号（奇数表示正在写）
2. 读者检查序列号是否改变
3. 如果改变，读者重试

顺序锁的适用场景：
- 读远多于写
- 数据可以安全地多次读取
- 不适合指针（可能读到无效地址）

## 10.4 睡眠锁机制

### 10.4.1 互斥量（mutex）

互斥量是可睡眠的锁，适用于可能长时间持有的临界区：

```
struct mutex {
    atomic_long_t owner;
    spinlock_t wait_lock;
    struct list_head wait_list;
};

mutex_lock(&mutex);
// 临界区，可以睡眠
mutex_unlock(&mutex);
```

互斥量的特性：
- **自适应自旋**：短暂自旋后才睡眠，减少上下文切换
- **优先级继承**：RT补丁中支持，防止优先级反转
- **调试支持**：CONFIG_DEBUG_MUTEXES提供死锁检测

互斥量的优化技术：
```
// MCS锁风格的等待队列
struct mutex_waiter {
    struct list_head list;
    struct task_struct *task;
};

// 乐观自旋
if (mutex_can_spin_on_owner(lock))
    mutex_optimistic_spin(lock);
```

### 10.4.2 信号量（semaphore）

信号量是更通用的同步原语，支持计数：

```
struct semaphore {
    raw_spinlock_t lock;
    unsigned int count;
    struct list_head wait_list;
};

// 二值信号量（类似互斥量）
down(&sem);      // P操作，可能睡眠
up(&sem);        // V操作

// 计数信号量
sema_init(&sem, count);  // 初始化为count
```

信号量与互斥量的区别：
- 信号量可以在不同上下文up/down
- 信号量支持计数（资源池）
- 互斥量有更严格的语义和更好的调试支持

### 10.4.3 完成量（completion）

完成量用于等待特定事件的发生：

```
struct completion {
    unsigned int done;
    wait_queue_head_t wait;
};

// 等待方
wait_for_completion(&comp);

// 通知方
complete(&comp);
complete_all(&comp);  // 唤醒所有等待者
```

完成量的典型用途：
- 等待模块初始化完成
- 等待内核线程退出
- 等待硬件操作完成

完成量与信号量的区别：
- 完成量语义更清晰（一次性事件）
- 完成量自动处理虚假唤醒
- 完成量支持超时等待

## 10.5 RCU机制深度解析

### 10.5.1 RCU基本原理

RCU（Read-Copy-Update）是Linux内核中最精妙的同步机制之一，实现了读者零开销：

```
// 读者
rcu_read_lock();
ptr = rcu_dereference(gptr);
// 使用ptr，不能睡眠
rcu_read_unlock();

// 写者
old_ptr = gptr;
new_ptr = kmalloc(...);
*new_ptr = *old_ptr;
// 修改new_ptr
rcu_assign_pointer(gptr, new_ptr);
synchronize_rcu();  // 等待所有读者完成
kfree(old_ptr);
```

RCU的核心思想：
1. **读者不加锁**：通过禁用抢占标记临界区
2. **写者复制**：创建新版本而不是就地修改
3. **延迟回收**：等待所有读者退出后才回收旧版本

### 10.5.2 宽限期（Grace Period）

宽限期是RCU的核心概念，表示所有CPU都经历了静止状态：

```
     CPU0        CPU1        CPU2
      |           |           |
   rcu_read_lock  |           |
      |        rcu_read_lock  |
      |           |      synchronize_rcu()
      |           |           |
   rcu_read_unlock|           |
      |---------- GP ---------|
      |        rcu_read_unlock|
      |           |           |
      |           |      (GP结束)
```

静止状态的定义：
- 用户态执行
- 空闲循环
- 上下文切换
- 显式声明（rcu_quiescent_state）

### 10.5.3 RCU实现变体

Linux内核包含多种RCU实现：

**1. RCU-sched**：
- 禁用抢占作为读端标记
- 调度器时钟中断检测静止状态
- 适用于大部分内核代码

**2. RCU-bh**：
- 禁用软中断作为读端标记
- 专门优化软中断处理路径

**3. SRCU（Sleepable RCU）**：
- 允许读者睡眠
- 使用per-CPU计数器
- 开销较大但更灵活

**4. Tasks RCU**：
- 等待所有任务经历上下文切换
- 用于跟踪点和BPF

### 10.5.4 RCU回调机制

RCU使用回调机制进行延迟释放：

```
struct rcu_head {
    struct rcu_head *next;
    void (*func)(struct rcu_head *head);
};

call_rcu(&obj->rcu_head, free_func);
```

回调批处理优化：
```
struct rcu_data {
    struct rcu_head *nxtlist;     // 新回调
    struct rcu_head *curlist;     // 当前批次
    struct rcu_head *donelist;    // 完成批次
};
```

### 10.5.5 RCU调试支持

内核提供了丰富的RCU调试工具：

```
CONFIG_RCU_TRACE         // RCU跟踪
CONFIG_RCU_CPU_STALL_DETECTOR  // 停滞检测
CONFIG_PROVE_RCU         // 运行时验证

// 调试接口
rcu_dereference_check()  // 带检查的解引用
rcu_access_pointer()     // 只访问指针值
```

## 10.6 正确性保证工具

### 10.6.1 lockdep死锁检测

lockdep是Linux内核的运行时锁验证系统：

```
struct lockdep_map {
    struct lock_class_key *key;
    struct lock_class *class_cache;
    const char *name;
};

// 锁类定义
static struct lock_class_key key;
spin_lock_init_key(&lock, &key);
```

lockdep检测的问题类型：
1. **死锁**：循环锁依赖
2. **锁顺序违反**：AB-BA模式
3. **中断安全违反**：中断上下文锁使用错误
4. **睡眠原语误用**：原子上下文中睡眠

### 10.6.2 锁依赖图分析

lockdep维护锁类之间的依赖图：

```
     A -----> B
     ^         |
     |         v
     D <----- C
     
// 检测到循环：死锁！
```

依赖规则：
- 前向依赖：持有A时获取B，创建A→B边
- 传递性：A→B且B→C，则A→C
- 独立类：不同key的锁属于不同类

### 10.6.3 运行时锁验证

内核提供了多种运行时检查：

```
// 自旋锁检查
CONFIG_DEBUG_SPINLOCK
- 检测未初始化的锁
- 检测重复解锁
- 检测错误的CPU

// 互斥量检查
CONFIG_DEBUG_MUTEXES
- 检测API误用
- 检测优先级反转

// 原子睡眠检查
CONFIG_DEBUG_ATOMIC_SLEEP
- 检测原子上下文中的睡眠
```

## 10.7 本章小结

Linux内核同步机制构成了一个完整的层次体系，从底层的原子操作到高层的RCU机制，每种同步原语都有其适用场景和性能特征。

**关键概念总结**：

1. **原子操作与内存屏障**：
   - 原子操作是所有同步机制的基础
   - 内存屏障控制操作的可见性顺序
   - 不同架构的内存模型差异巨大

2. **自旋锁选择原则**：
   - spinlock：短临界区，不能睡眠
   - rwlock：读多写少，注意写者饥饿
   - seqlock：读远多于写，数据可重读

3. **睡眠锁使用指南**：
   - mutex：首选的睡眠锁，有调试支持
   - semaphore：需要计数或跨上下文释放
   - completion：等待一次性事件

4. **RCU核心思想**：
   - 读者零开销，写者承担复杂性
   - 通过延迟回收保证安全性
   - 适合读多写少的数据结构

5. **正确性保证**：
   - lockdep提供运行时死锁检测
   - 各种DEBUG选项帮助发现并发错误
   - 遵循锁层次避免死锁

**性能考量公式**：

临界区选择决策：
$$\text{同步开销} = \text{获取开销} + \text{持有时间} + \text{释放开销} + \text{争用开销}$$

RCU读写性能比：
$$\text{RCU收益} = \frac{\text{读频率} \times \text{读开销}_{传统}}{\text{读频率} \times \text{读开销}_{RCU} + \text{写频率} \times \text{写开销}_{RCU}}$$

锁粒度优化：
$$\text{并发度} = \frac{1}{P + \frac{1-P}{N}}$$
其中P是串行部分比例，N是处理器数量（Amdahl定律）

## 10.8 练习题

### 基础理解题

**练习10.1**：解释为什么在x86架构上大多数原子操作需要LOCK前缀，而XCHG指令不需要？

<details>
<summary>答案</summary>

XCHG指令在x86架构上具有隐式的LOCK语义，即使没有LOCK前缀也会锁定总线或缓存行。这是x86的历史设计决定。其他指令如ADD、SUB等需要显式的LOCK前缀才能保证原子性。这种设计简化了某些原子操作的实现，但XCHG的隐式锁定也意味着它总是比普通MOV指令慢。

</details>

**练习10.2**：在什么情况下读写锁（rwlock）的性能可能比普通自旋锁（spinlock）更差？

<details>
<summary>答案</summary>

1. 写操作频繁时：读写锁的写者需要等待所有读者完成，可能导致写者饥饿
2. 读临界区很短时：读写锁的获取/释放开销比spinlock大
3. NUMA系统上：读者计数器的更新导致缓存行在节点间移动
4. 读者和写者交替时：频繁的模式切换增加开销
5. 争用激烈时：多个读者更新同一个计数器造成缓存行乒乓

</details>

**练习10.3**：为什么RCU读端临界区内不能睡眠？如果需要睡眠应该使用什么？

<details>
<summary>答案</summary>

经典RCU使用禁用抢占来标记读端临界区。如果读者睡眠，会导致：
1. 抢占计数不平衡，系统错误
2. 宽限期无法结束，因为睡眠的读者不会经历静止状态
3. 可能导致系统停滞或死锁

需要睡眠时应使用SRCU（Sleepable RCU），它使用per-CPU计数器而非禁用抢占，允许读者睡眠，代价是更高的读端开销。

</details>

### 代码分析题

**练习10.4**：分析以下代码可能存在的并发问题：

```c
struct foo {
    spinlock_t lock;
    struct list_head list;
    int counter;
};

void process_foo(struct foo *f)
{
    spin_lock(&f->lock);
    if (f->counter > 0) {
        spin_unlock(&f->lock);
        schedule();  // 让出CPU
        spin_lock(&f->lock);
        f->counter--;
        // 处理list
    }
    spin_unlock(&f->lock);
}
```

<details>
<summary>答案</summary>

问题：检查条件和执行操作之间释放了锁，存在TOCTOU（Time-Of-Check-Time-Of-Use）竞态。

具体场景：
1. 线程A检查counter > 0，成功
2. 线程A释放锁，调用schedule()
3. 线程B获得锁，将counter减到0
4. 线程A重新获得锁，错误地将counter减到负数

修复方案：不要在检查和操作之间释放锁，或重新检查条件：
```c
spin_lock(&f->lock);
if (f->counter > 0) {
    f->counter--;
    // 如果必须让出CPU，保存状态
}
spin_unlock(&f->lock);
```

</details>

**练习10.5**：解释以下RCU代码模式的正确性，指出可能的问题：

```c
struct foo {
    struct rcu_head rcu;
    int data;
};

void reader(void)
{
    struct foo *p;
    int val;
    
    rcu_read_lock();
    p = rcu_dereference(global_foo);
    val = p->data;
    rcu_read_unlock();
    
    if (val > 100) {
        rcu_read_lock();
        p = rcu_dereference(global_foo);
        p->data = val + 1;  // 问题行
        rcu_read_unlock();
    }
}
```

<details>
<summary>答案</summary>

问题：RCU读者不能修改数据！第二个rcu_read_lock()块中试图修改p->data违反了RCU语义。

具体错误：
1. RCU是读-复制-更新，读者只能读
2. 两次rcu_dereference()可能得到不同的指针
3. 修改共享数据需要适当的锁保护

正确的更新模式：
```c
void updater(int increment)
{
    struct foo *old, *new;
    
    new = kmalloc(sizeof(*new), GFP_KERNEL);
    
    spin_lock(&update_lock);
    old = global_foo;
    *new = *old;
    new->data += increment;
    rcu_assign_pointer(global_foo, new);
    spin_unlock(&update_lock);
    
    synchronize_rcu();
    kfree(old);
}
```

</details>

### 设计实现题

**练习10.6**：设计一个读多写少的计数器，要求读操作开销最小。比较至少三种实现方案的优缺点。

<details>
<summary>答案</summary>

方案1：原子变量
```c
atomic_t counter;
读：atomic_read(&counter)  // 单指令
写：atomic_inc(&counter)   // 原子操作
优点：简单，开销小
缺点：写操作在多CPU下有缓存行争用
```

方案2：顺序锁保护
```c
seqlock_t lock;
int counter;
读：循环读取直到序列号稳定
写：获取写锁更新
优点：读者不阻塞写者
缺点：读者可能需要重试
```

方案3：Per-CPU计数器 + RCU
```c
DEFINE_PER_CPU(int, counters);
读：累加所有CPU的值（RCU保护）
写：更新本地CPU计数器
优点：写操作无争用，扩展性好
缺点：读操作开销O(NR_CPUS)，计数可能不精确
```

方案4：分片计数器（最优）
```c
struct counter {
    atomic_t shards[NR_SHARDS];
};
读：累加所有分片
写：hash选择分片更新
优点：平衡读写性能，减少争用
缺点：实现复杂，需要选择合适的分片数
```

选择依据：
- 极端读多：原子变量
- 需要精确值：顺序锁
- 高并发写：Per-CPU或分片

</details>

**练习10.7**：实现一个支持超时的读写信号量，要求支持读者升级为写者。给出伪代码和关键设计决策。

<details>
<summary>答案</summary>

关键设计决策：
1. 使用等待队列支持超时
2. 读者升级需要原子性保证
3. 防止升级导致的死锁

伪代码实现：
```c
struct timed_rwsem {
    atomic_t count;  // >0:读者数，-1:写者，0:空闲
    spinlock_t wait_lock;
    struct list_head read_waiters;
    struct list_head write_waiters;
    struct task_struct *owner;  // 当前写者
};

// 读者获取（支持超时）
int down_read_timeout(struct timed_rwsem *sem, long timeout)
{
    while (atomic_read(&sem->count) < 0) {
        // 有写者，需要等待
        if (wait_event_timeout(sem->read_waiters,
                              atomic_read(&sem->count) >= 0,
                              timeout) == 0)
            return -ETIMEDOUT;
    }
    atomic_inc(&sem->count);
    return 0;
}

// 读者升级为写者
int upgrade_read_to_write(struct timed_rwsem *sem)
{
    // 原子地从读者变为升级等待者
    spin_lock(&sem->wait_lock);
    
    // 标记为升级中，阻止新读者
    if (atomic_read(&sem->count) != 1) {
        // 还有其他读者，等待
        atomic_dec(&sem->count);  // 释放读锁
        
        // 等待成为唯一访问者
        wait_event(sem->write_waiters,
                  atomic_read(&sem->count) == 0);
        
        atomic_set(&sem->count, -1);  // 获取写锁
        sem->owner = current;
    } else {
        // 唯一读者，直接升级
        atomic_set(&sem->count, -1);
        sem->owner = current;
    }
    
    spin_unlock(&sem->wait_lock);
    return 0;
}

// 防止死锁策略
1. 升级失败时回退到读者
2. 使用try_upgrade，失败则释放读锁再获取写锁
3. 限制同时升级的读者数量
```

</details>

**练习10.8**：设计一个调试工具，检测内核中潜在的锁顺序违规。要求能处理动态分配的锁和条件加锁。

<details>
<summary>答案</summary>

设计思路：扩展lockdep，支持动态场景

核心数据结构：
```c
struct lock_pattern {
    void *lock_class_ids[MAX_DEPTH];
    int depth;
    char *stack_trace;
    int count;  // 出现次数
};

struct dynamic_lock_class {
    void *key;  // 动态生成的key
    char *allocation_site;
    struct list_head patterns;  // 此锁参与的模式
};

// 运行时检测
1. Hook所有锁操作
2. 记录锁获取序列
3. 构建动态依赖图
4. 检测环和异常模式

// 条件加锁处理
struct conditional_lock {
    bool (*condition)(void *);
    void *context;
    struct lock_pattern *true_path;
    struct lock_pattern *false_path;
};

// 检测算法
detect_violations() {
    // 1. 收集所有锁序列
    for_each_thread(t) {
        record_lock_sequence(t);
    }
    
    // 2. 构建全局锁序图
    build_lock_order_graph();
    
    // 3. 检测违规
    // a) 直接环
    detect_cycles();
    
    // b) 条件死锁
    for_each_conditional_pattern(p) {
        if (can_form_cycle(p))
            report_conditional_deadlock(p);
    }
    
    // c) 锁层次违反
    check_lock_hierarchy();
    
    // 4. 生成报告
    generate_violation_report();
}

// 输出格式
LOCK ORDER VIOLATION:
Thread A: Lock1 -> Lock2
Thread B: Lock2 -> Lock1
Conditional: if (condition) Lock3
Recommendation: Establish lock hierarchy
```

高级功能：
1. 概率性死锁检测（基于运行时条件概率）
2. 性能影响评估（锁持有时间分析）
3. 自动修复建议（基于模式库）

</details>

## 10.9 常见陷阱与错误（Gotchas）

### 10.9.1 死锁陷阱

**1. ABBA死锁**
```c
// CPU 0              // CPU 1
spin_lock(&A);        spin_lock(&B);
spin_lock(&B);        spin_lock(&A);  // 死锁！
```
**预防**：建立全局锁顺序

**2. 中断死锁**
```c
spin_lock(&lock);
// 中断发生
  -> interrupt_handler()
     spin_lock(&lock);  // 死锁！
```
**预防**：使用spin_lock_irqsave()

**3. 优先级反转**
```c
低优先级任务持有锁 -> 高优先级任务等待 -> 中优先级任务抢占低优先级
```
**预防**：使用优先级继承协议

### 10.9.2 RCU陷阱

**1. 过早释放**
```c
rcu_read_lock();
p = rcu_dereference(gp);
rcu_read_unlock();
use(p);  // 错误！p可能已被释放
```

**2. 引用计数与RCU混用**
```c
rcu_read_lock();
p = rcu_dereference(gp);
atomic_inc(&p->refcount);  // 危险！p可能正在释放
rcu_read_unlock();
```
**正确方式**：在RCU临界区内完成引用计数增加

**3. RCU与睡眠**
```c
rcu_read_lock();
kmalloc(GFP_KERNEL);  // 错误！可能睡眠
rcu_read_unlock();
```

### 10.9.3 内存屏障陷阱

**1. 编译器优化**
```c
while (!flag);  // 编译器可能优化为 if(!flag) while(1);
data = shared;
```
**修复**：使用READ_ONCE(flag)

**2. 依赖排序**
```c
p = READ_ONCE(gp);
if (p)
    val = p->data;  // ARM/Alpha可能乱序
```
**修复**：使用smp_read_barrier_depends()（已废弃）或rcu_dereference()

### 10.9.4 性能陷阱

**1. 锁粒度过细**
```c
for (i = 0; i < 1000; i++) {
    spin_lock(&lock);
    do_something_tiny();
    spin_unlock(&lock);
}
```
**问题**：锁开销超过实际工作

**2. 读写锁误用**
```c
read_lock(&rwlock);
lengthy_computation();  // 阻塞所有写者
read_unlock(&rwlock);
```

**3. 虚假共享**
```c
struct percpu_counter {
    int cpu0_count;  // 同一缓存行
    int cpu1_count;  // 造成乒乓
} __attribute__((packed));
```
**修复**：使用____cacheline_aligned

## 10.10 最佳实践检查清单

### 设计阶段

- [ ] **锁层次设计**
  - 定义清晰的锁获取顺序
  - 文档化锁之间的依赖关系
  - 避免嵌套锁超过3层

- [ ] **锁粒度决策**
  - 评估临界区大小
  - 考虑NUMA影响
  - 平衡并发度与开销

- [ ] **同步原语选择**
  - 临界区能睡眠？→ mutex
  - 读多写少？→ RCU/seqlock
  - 需要计数？→ semaphore
  - 简单保护？→ spinlock

### 实现阶段

- [ ] **正确性保证**
  - 启用lockdep（CONFIG_PROVE_LOCKING）
  - 启用死锁检测（CONFIG_DEBUG_DEADLOCKS）
  - 运行时检查（CONFIG_DEBUG_ATOMIC_SLEEP）

- [ ] **中断安全**
  - 中断处理程序使用的锁需要irqsave
  - 软中断使用的锁需要bh_disable
  - 注意本地中断与远程中断的区别

- [ ] **内存序保证**
  - 使用适当的屏障
  - 注意编译器优化
  - 测试弱内存序架构

### 优化阶段

- [ ] **缓存行对齐**
  ```c
  struct optimized {
      spinlock_t lock ____cacheline_aligned;
      int shared_data;
  } ____cacheline_aligned;
  ```

- [ ] **per-CPU优化**
  - 考虑per-CPU变量减少争用
  - 使用local_t进行本地计数
  - 注意CPU热插拔处理

- [ ] **RCU转换检查**
  - 读写比例是否合适（10:1以上）
  - 能否接受过期数据
  - 更新是否可以复制

### 调试阶段

- [ ] **死锁排查**
  - 检查/proc/lockdep
  - 分析lockdep报告
  - 使用ftrace跟踪锁事件

- [ ] **性能分析**
  - perf记录锁争用
  - 分析锁持有时间
  - 检查锁等待队列长度

- [ ] **压力测试**
  - 高并发场景测试
  - CPU热插拔测试
  - 内存压力测试

### 维护阶段

- [ ] **文档更新**
  - 记录锁的用途和保护的数据
  - 更新锁依赖图
  - 记录已知的性能特征

- [ ] **代码审查要点**
  - 所有锁路径都有对应的解锁
  - 错误路径正确释放锁
  - 没有递归锁的使用

- [ ] **版本兼容**
  - 检查内核版本差异
  - 注意API变化（如自旋锁初始化）
  - 测试实时补丁（PREEMPT_RT）兼容性
