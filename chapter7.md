# 第7章：块设备层与I/O调度

## 章节大纲

### 7.1 块设备架构
- 7.1.1 块设备层在内核中的位置
- 7.1.2 核心数据结构：bio、request、gendisk
- 7.1.3 块设备驱动接口
- 7.1.4 请求处理流程

### 7.2 I/O调度器演进
- 7.2.1 传统调度器：CFQ、deadline、noop
- 7.2.2 多队列调度器：mq-deadline、bfq、kyber
- 7.2.3 调度器选择策略
- 7.2.4 性能对比与场景适用

### 7.3 多队列块层（blk-mq）
- 7.3.1 从单队列到多队列的架构变革
- 7.3.2 软件队列与硬件队列映射
- 7.3.3 请求分发与完成机制
- 7.3.4 标签管理与并发控制

### 7.4 设备映射器（device-mapper）
- 7.4.1 DM架构与目标驱动
- 7.4.2 线性映射与条带化
- 7.4.3 快照与精简配置
- 7.4.4 加密与完整性保护

### 7.5 性能优化技术
- 7.5.1 I/O合并与请求重排
- 7.5.2 预读策略与缓存管理
- 7.5.3 写回机制与脏页控制
- 7.5.4 I/O统计与性能监控

### 7.6 本章小结
### 7.7 练习题
### 7.8 常见陷阱与错误
### 7.9 最佳实践检查清单

---

## 开篇导读

块设备层是Linux内核中连接文件系统与物理存储设备的关键桥梁。从早期的简单请求队列到现代的多队列架构，块设备层经历了多次重大变革，每一次都是为了适应存储技术的发展——从机械硬盘到SSD，再到NVMe设备。本章将深入剖析块设备层的架构设计、I/O调度算法、以及如何针对不同存储介质进行性能优化。

## 学习目标

完成本章学习后，您将能够：

1. **理解块设备层架构**：掌握bio、request、gendisk等核心数据结构及其相互关系
2. **分析I/O调度算法**：对比不同调度器的设计理念和适用场景
3. **掌握blk-mq机制**：理解多队列架构如何充分发挥现代存储设备性能
4. **运用device-mapper**：实现逻辑卷管理、快照、加密等高级存储功能
5. **优化存储性能**：通过I/O合并、预读、写回等技术提升系统吞吐量

---

## 7.1 块设备架构

### 7.1.1 块设备层在内核中的位置

块设备层位于VFS和实际块设备驱动之间，提供了统一的块I/O处理框架：

```
    用户空间
        |
    系统调用 (read/write/mmap)
        |
    ┌─────┐
    │ VFS │ (虚拟文件系统)
    └──┬──┘
       │
    ┌──┴────────┐
    │ Page Cache│ (页缓存)
    └──┬────────┘
       │
    ┌──┴──────┐
    │ 文件系统 │ (ext4/xfs/btrfs...)
    └──┬──────┘
       │ submit_bio()
    ┌──┴────────────┐
    │  块设备层(Block Layer)  │
    │ ┌──────────┐ │
    │ │ bio 层   │ │ (块I/O层)
    │ └──┬───────┘ │
    │    │make_request
    │ ┌──┴───────┐ │
    │ │ 请求队列  │ │ (request queue)
    │ └──┬───────┘ │
    │    │         │
    │ ┌──┴───────┐ │
    │ │I/O调度器 │ │ (elevator)
    │ └──┬───────┘ │
    └────┴──────────┘
         │
    ┌────┴──────┐
    │ 块设备驱动 │ (scsi/nvme/virtio-blk...)
    └────┬──────┘
         │
    ┌────┴──────┐
    │ 硬件设备  │ (HDD/SSD/NVMe...)
    └───────────┘
```

### 7.1.2 核心数据结构

#### bio（Block I/O）

bio是块I/O操作的基本单位，描述了一次I/O操作的所有信息：

```c
struct bio {
    struct bio        *bi_next;      /* 请求链表中的下一个bio */
    struct gendisk    *bi_disk;      /* 目标磁盘 */
    unsigned int      bi_opf;        /* 操作标志(REQ_OP_*) */
    unsigned short    bi_flags;      /* bio标志 */
    unsigned short    bi_ioprio;     /* I/O优先级 */
    
    struct bvec_iter  bi_iter;       /* 当前迭代器位置 */
    unsigned int      bi_size;       /* 剩余I/O字节数 */
    
    unsigned short    bi_vcnt;       /* bio_vec数量 */
    unsigned short    bi_max_vecs;   /* 最大bio_vec数量 */
    atomic_t          bi_cnt;        /* 引用计数 */
    
    struct bio_vec    *bi_io_vec;    /* 实际的vec列表 */
    struct bio_set    *bi_pool;      /* bio内存池 */
    
    /* 回调和私有数据 */
    bio_end_io_t      *bi_end_io;    /* I/O完成回调 */
    void              *bi_private;   /* 私有数据 */
};

struct bio_vec {
    struct page     *bv_page;     /* 物理页 */
    unsigned int    bv_len;       /* 数据长度 */
    unsigned int    bv_offset;    /* 页内偏移 */
};
```

bio的关键特性：
- **散列聚集支持**：通过bio_vec数组支持非连续内存
- **链式结构**：多个bio可以链接描述大块I/O
- **零拷贝**：直接引用用户空间页面，避免数据复制

#### request（I/O请求）

request是经过合并优化后的I/O请求单位：

```c
struct request {
    struct request_queue *q;         /* 所属请求队列 */
    struct blk_mq_ctx    *mq_ctx;    /* blk-mq上下文 */
    struct blk_mq_hw_ctx *mq_hctx;   /* 硬件队列上下文 */
    
    unsigned int cmd_flags;          /* 命令标志 */
    req_flags_t  rq_flags;           /* 请求标志 */
    
    sector_t     __sector;           /* 起始扇区 */
    unsigned int __data_len;         /* 数据长度 */
    
    struct bio   *bio;               /* bio链表头 */
    struct bio   *biotail;           /* bio链表尾 */
    
    struct list_head queuelist;      /* 队列链表 */
    
    /* 用于I/O统计 */
    u64 start_time_ns;              /* 开始时间 */
    u64 io_start_time_ns;           /* I/O开始时间 */
    
    /* 用于超时处理 */
    unsigned int timeout;            /* 超时时间 */
    struct timer_list timeout_timer; /* 超时定时器 */
};
```

#### gendisk（通用磁盘）

gendisk代表一个块设备：

```c
struct gendisk {
    int major;                      /* 主设备号 */
    int first_minor;                /* 起始次设备号 */
    int minors;                     /* 次设备号数量 */
    
    char disk_name[DISK_NAME_LEN]; /* 磁盘名称(如sda) */
    
    struct request_queue *queue;    /* 请求队列 */
    struct block_device *part0;     /* 整个磁盘的块设备 */
    
    const struct block_device_operations *fops; /* 块设备操作 */
    
    /* 分区信息 */
    struct disk_part_tbl __rcu *part_tbl;  /* 分区表 */
    struct hd_struct part0;                /* 整盘分区 */
    
    /* 容量信息 */
    sector_t capacity;              /* 磁盘容量(扇区) */
    
    /* 统计信息 */
    struct disk_stats __percpu *dkstats;   /* 磁盘统计 */
};
```

### 7.1.3 块设备驱动接口

块设备驱动需要实现block_device_operations接口：

```c
struct block_device_operations {
    int (*open)(struct block_device *, fmode_t);
    void (*release)(struct gendisk *, fmode_t);
    int (*ioctl)(struct block_device *, fmode_t, unsigned, unsigned long);
    int (*compat_ioctl)(struct block_device *, fmode_t, unsigned, unsigned long);
    int (*direct_access)(struct block_device *, sector_t, void **, unsigned long *);
    int (*media_changed)(struct gendisk *);
    int (*revalidate_disk)(struct gendisk *);
    int (*getgeo)(struct block_device *, struct hd_geometry *);
    void (*swap_slot_free_notify)(struct block_device *, unsigned long);
    struct module *owner;
};
```

### 7.1.4 请求处理流程

完整的I/O请求处理流程：

1. **bio提交阶段**：
```c
文件系统 -> submit_bio() -> generic_make_request() -> 请求队列
```

2. **请求合并阶段**：
```c
bio -> 电梯算法合并 -> request -> 插入调度队列
```

3. **请求派发阶段**：
```c
调度器 -> blk_mq_dispatch_rq_list() -> 驱动程序
```

4. **完成处理阶段**：
```c
中断处理 -> blk_mq_complete_request() -> bio_endio() -> 唤醒等待进程
```

---

## 7.2 I/O调度器演进

I/O调度器负责对块设备请求进行重排序和合并，以提高磁盘访问效率。Linux内核提供了多种调度算法，适应不同的存储设备和工作负载。

### 7.2.1 传统调度器

#### CFQ（Completely Fair Queuing）

CFQ是Linux长期的默认调度器，为每个进程维护独立的I/O队列：

```c
struct cfq_data {
    struct request_queue *queue;
    
    /* 服务树(红黑树)按优先级组织 */
    struct cfq_rb_root service_trees[2][3];  /* sync/async, 3个优先级 */
    
    /* 空闲和忙碌队列 */
    struct list_head idle_list;
    struct list_head busy_list;
    
    /* 时间片和配额 */
    unsigned int cfq_quantum;        /* 队列时间片 */
    unsigned int cfq_slice[2];       /* sync/async时间片 */
    unsigned int cfq_slice_idle;     /* 空闲等待时间 */
    
    /* 统计信息 */
    unsigned int busy_queues;
    struct cfq_io_context *active_cic;
};
```

CFQ的核心算法：
- **进程公平性**：每个进程获得公平的I/O带宽份额
- **同步优先**：同步I/O优先于异步I/O
- **时间片轮转**：类似CPU调度的时间片机制
- **预期思考时间**：为顺序读预留等待时间

#### Deadline调度器

Deadline调度器通过截止时间保证I/O延迟上界：

```c
struct deadline_data {
    /* 请求队列：读写分离，按扇区排序 */
    struct rb_root sort_list[2];     /* 读/写红黑树 */
    struct list_head fifo_list[2];   /* FIFO队列 */
    
    /* 当前批处理 */
    struct request *next_rq[2];      /* 下一个请求 */
    unsigned int batching;            /* 批处理计数 */
    unsigned int starved;             /* 饥饿计数 */
    
    /* 调度参数 */
    int fifo_expire[2];              /* 读500ms,写5000ms */
    int fifo_batch;                  /* 批大小16 */
    int writes_starved;              /* 写饥饿阈值2 */
    int front_merges;                /* 前向合并开关 */
};
```

Deadline特点：
- **双队列机制**：排序队列(红黑树) + FIFO队列
- **截止时间保证**：读请求500ms，写请求5s
- **防止饥饿**：通过starved计数防止写请求饥饿
- **批处理优化**：连续处理多个相邻请求

#### NOOP调度器

最简单的调度器，仅做基本的请求合并：

```c
struct noop_data {
    struct list_head queue;  /* 简单FIFO队列 */
};
```

适用场景：
- SSD等随机访问性能好的设备
- 虚拟机中的虚拟磁盘
- 需要上层自行控制I/O顺序的场景

### 7.2.2 多队列调度器

随着NVMe等高性能存储设备的普及，传统单队列调度器成为瓶颈。Linux引入了基于blk-mq的新一代调度器。

#### mq-deadline

多队列版本的deadline调度器：

```c
struct mq_deadline_data {
    /* per-hardware queue */
    struct deadline_data **hq_data;
    
    /* 全局统计 */
    atomic_t dispatched[2];
    
    /* 每CPU统计避免缓存行竞争 */
    struct mq_deadline_cpu_stats __percpu *stats;
};
```

#### BFQ（Budget Fair Queueing）

BFQ是专为交互式和软实时应用优化的调度器：

```c
struct bfq_data {
    /* 服务树 */
    struct bfq_group *root_group;
    
    /* 权重树 */
    struct rb_root queue_weights_tree;
    struct rb_root group_weights_tree;
    
    /* 预算和时间片 */
    int bfq_max_budget;
    int bfq_timeout;
    
    /* 交互性检测 */
    bool strict_guarantees;
    bool low_latency;
};
```

BFQ特性：
- **预算公平**：基于预算而非时间的公平性
- **低延迟保证**：为交互式应用提供响应性保证
- **cgroup支持**：与cgroup v2深度集成

#### Kyber

Kyber是为快速NVMe设备设计的轻量级调度器：

```c
struct kyber_data {
    /* 令牌桶 */
    struct kyber_token_bucket tb[KYBER_NUM_DOMAINS];
    
    /* 延迟目标 */
    unsigned int read_lat_nsec;   /* 2ms */
    unsigned int write_lat_nsec;  /* 10ms */
    
    /* 自适应调整 */
    struct timer_list timer;
    atomic_t samples[KYBER_NUM_DOMAINS];
};
```

Kyber使用令牌桶算法控制I/O延迟：
- **延迟目标**：读2ms，写10ms
- **动态调整**：根据实际延迟调整令牌发放速率
- **极简设计**：适合高速设备的低开销调度

### 7.2.3 调度器选择策略

不同存储设备和工作负载的最佳调度器选择：

| 存储类型 | 工作负载 | 推荐调度器 | 理由 |
|---------|---------|-----------|------|
| HDD | 服务器混合负载 | mq-deadline | 平衡吞吐量和延迟 |
| HDD | 桌面应用 | bfq | 优秀的交互性 |
| SATA SSD | 通用负载 | mq-deadline | 稳定可靠 |
| NVMe SSD | 高性能服务器 | none/kyber | 最小开销 |
| 虚拟磁盘 | 虚拟机 | none | 避免重复调度 |

运行时切换调度器：
```bash
# 查看当前调度器
cat /sys/block/sda/queue/scheduler

# 切换调度器
echo mq-deadline > /sys/block/sda/queue/scheduler
```

### 7.2.4 性能对比与场景适用

关键性能指标对比（相对值）：

| 调度器 | 顺序吞吐量 | 随机IOPS | 平均延迟 | CPU开销 | 公平性 |
|--------|-----------|----------|---------|---------|--------|
| noop | 中 | 高 | 低 | 极低 | 差 |
| deadline | 高 | 中 | 可控 | 低 | 中 |
| cfq | 中 | 低 | 中 | 高 | 优秀 |
| mq-deadline | 高 | 高 | 可控 | 低 | 中 |
| bfq | 中 | 中 | 极低 | 较高 | 优秀 |
| kyber | 高 | 极高 | 可控 | 极低 | 中 |

---

## 7.3 多队列块层（blk-mq）

### 7.3.1 从单队列到多队列的架构变革

传统单队列架构的瓶颈：

```
    所有CPU
        ↓
    [全局锁]
        ↓
   单一请求队列
        ↓
    I/O调度器
        ↓
     设备驱动
```

多队列架构的革新：

```
  CPU0    CPU1    CPU2    CPU3
    ↓       ↓       ↓       ↓
  SWQ0    SWQ1    SWQ2    SWQ3  (软件队列)
    ↓       ↓       ↓       ↓
    └───────┴───────┴───────┘
              ↓
         [映射算法]
              ↓
    ┌─────────────────────┐
    │  HWQ0    HWQ1       │     (硬件队列)
    └──┬────────┬─────────┘
       ↓        ↓
    [标签0]  [标签1]            (标签管理)
       ↓        ↓
    ┌──────────────┐
    │   NVMe设备   │
    └──────────────┘
```

### 7.3.2 软件队列与硬件队列映射

blk-mq的核心数据结构：

```c
struct blk_mq_hw_ctx {
    struct {
        spinlock_t      lock;
        struct list_head dispatch;     /* 派发队列 */
        unsigned long    state;         /* 队列状态 */
    } ____cacheline_aligned_in_smp;
    
    struct delayed_work run_work;      /* 异步运行 */
    struct delayed_work delay_work;    /* 延迟处理 */
    
    unsigned long       flags;         /* BLK_MQ_F_* */
    
    void               *driver_data;   /* 驱动私有数据 */
    
    struct sbitmap      ctx_map;       /* 软件队列位图 */
    
    struct blk_mq_ctx  **ctxs;        /* 软件队列数组 */
    unsigned int        nr_ctx;        /* 软件队列数量 */
    
    atomic_t            wait_index;    /* 等待索引 */
    
    struct blk_mq_tags *tags;          /* 标签集 */
    struct blk_mq_tags *sched_tags;   /* 调度器标签 */
    
    unsigned long       queued;        /* 排队请求数 */
    unsigned long       run;           /* 运行次数 */
    
    unsigned int        numa_node;     /* NUMA节点 */
    unsigned int        queue_num;     /* 队列编号 */
};

struct blk_mq_ctx {
    struct {
        spinlock_t      lock;
        struct list_head rq_list;      /* 请求列表 */
    } ____cacheline_aligned_in_smp;
    
    unsigned int        cpu;           /* 所属CPU */
    unsigned int        index_hw;      /* 硬件队列索引 */
    
    /* 统计信息 */
    struct blk_mq_ctxs  *ctxs;
    struct kobject      kobj;
} ____cacheline_aligned_in_smp;
```

CPU到硬件队列的映射算法：

```c
/* 默认映射：轮询分配 */
static int blk_mq_map_queues(struct blk_mq_tag_set *set)
{
    unsigned int *map = set->mq_map;
    unsigned int nr_queues = set->nr_hw_queues;
    unsigned int cpu, queue;
    
    queue = 0;
    for_each_possible_cpu(cpu) {
        map[cpu] = queue;
        queue = (queue + 1) % nr_queues;
    }
    return 0;
}

/* NUMA感知映射 */
static int blk_mq_map_queues_numa(struct blk_mq_tag_set *set)
{
    const struct cpumask *mask;
    unsigned int queue, cpu;
    
    queue = 0;
    for_each_node_mask(node, node_possible_map) {
        mask = cpumask_of_node(node);
        for_each_cpu(cpu, mask) {
            set->mq_map[cpu] = queue % set->nr_hw_queues;
        }
        queue++;
    }
    return 0;
}
```

### 7.3.3 请求分发与完成机制

请求分发流程：

```c
/* 1. bio提交到软件队列 */
void blk_mq_make_request(struct request_queue *q, struct bio *bio)
{
    struct blk_mq_ctx *ctx = blk_mq_get_ctx(q);
    struct request *rq;
    
    /* 分配请求 */
    rq = blk_mq_get_request(q, bio, ctx);
    
    /* 初始化请求 */
    blk_mq_bio_to_request(rq, bio);
    
    /* 插入软件队列 */
    blk_mq_insert_request(rq, ctx);
    
    /* 触发运行 */
    blk_mq_run_hw_queue(hctx, async);
}

/* 2. 从软件队列派发到硬件队列 */
bool blk_mq_dispatch_rq_list(struct blk_mq_hw_ctx *hctx,
                             struct list_head *list)
{
    struct request *rq;
    int errors = 0;
    
    while (!list_empty(list)) {
        rq = list_first_entry(list, struct request, queuelist);
        list_del_init(&rq->queuelist);
        
        /* 调用驱动的queue_rq */
        ret = q->mq_ops->queue_rq(hctx, rq);
        
        if (ret == BLK_STS_OK) {
            queued++;
        } else if (ret == BLK_STS_RESOURCE) {
            /* 资源不足，重新入队 */
            blk_mq_requeue_request(rq);
            break;
        }
    }
    
    return queued > 0;
}
```

请求完成处理：

```c
/* 中断上下文的完成处理 */
void blk_mq_complete_request(struct request *rq)
{
    struct blk_mq_ctx *ctx = rq->mq_ctx;
    
    /* 标记完成 */
    if (!blk_mark_rq_complete(rq))
        return;
    
    /* 本地CPU处理 */
    if (rq->cpu == smp_processor_id()) {
        __blk_mq_complete_request(rq);
    } else {
        /* IPI到目标CPU */
        blk_mq_trigger_softirq(rq);
    }
}

/* 软中断中的处理 */
static void __blk_mq_complete_request(struct request *rq)
{
    struct request_queue *q = rq->q;
    
    /* 更新统计 */
    blk_account_io_completion(rq);
    
    /* 调用完成回调 */
    if (rq->end_io)
        rq->end_io(rq, error);
    else
        blk_mq_free_request(rq);
}
```

### 7.3.4 标签管理与并发控制

标签管理确保请求数量受控：

```c
struct blk_mq_tags {
    unsigned int nr_tags;       /* 标签总数 */
    unsigned int nr_reserved;   /* 保留标签数 */
    
    struct sbitmap_queue bitmap_tags;    /* 普通标签位图 */
    struct sbitmap_queue breserved_tags; /* 保留标签位图 */
    
    struct request **rqs;       /* 请求数组 */
    struct list_head page_list; /* 内存页列表 */
};

/* 标签分配 */
unsigned int blk_mq_get_tag(struct blk_mq_alloc_data *data)
{
    struct blk_mq_tags *tags = data->tags;
    struct sbitmap_queue *bt;
    unsigned int tag;
    
    /* 选择位图：普通或保留 */
    bt = &tags->bitmap_tags;
    if (data->flags & BLK_MQ_REQ_RESERVED)
        bt = &tags->breserved_tags;
    
    /* 从位图分配 */
    tag = __sbitmap_queue_get(bt);
    
    if (tag == -1 && !(data->flags & BLK_MQ_REQ_NOWAIT)) {
        /* 等待标签释放 */
        blk_mq_wait_for_tag(data);
        tag = __sbitmap_queue_get(bt);
    }
    
    return tag;
}
```

标签等待和唤醒机制：

```c
/* 等待标签 */
static int blk_mq_wait_for_tag(struct blk_mq_alloc_data *data)
{
    struct sbq_wait_state *ws;
    DEFINE_WAIT(wait);
    
    ws = bt_wait_ptr(bt, data->hctx);
    
    do {
        prepare_to_wait(&ws->wait, &wait, TASK_UNINTERRUPTIBLE);
        
        tag = __sbitmap_queue_get(bt);
        if (tag != -1)
            break;
        
        io_schedule();
    } while (1);
    
    finish_wait(&ws->wait, &wait);
    return tag;
}

/* 释放标签并唤醒等待者 */
void blk_mq_put_tag(struct blk_mq_hw_ctx *hctx, unsigned int tag)
{
    struct blk_mq_tags *tags = hctx->tags;
    
    sbitmap_queue_clear(&tags->bitmap_tags, tag);
    
    /* 唤醒等待者 */
    sbitmap_queue_wake_up(&tags->bitmap_tags);
}
```

---

## 7.4 设备映射器（Device Mapper）

Device Mapper是Linux内核中的一个通用框架，用于创建虚拟块设备。它为LVM2、软件RAID、加密等提供底层支持。

### 7.4.1 DM架构与目标驱动

DM的三层架构：

```
    用户空间工具 (lvm2, cryptsetup, dmsetup)
            ↓ ioctl
    ┌─────────────────────────┐
    │    Device Mapper Core    │
    │  (映射表管理，I/O路由)    │
    └───────────┬─────────────┘
                ↓
    ┌───────────────────────────────┐
    │       Target Drivers          │
    │  (linear, striped, mirror,    │
    │   snapshot, crypt, thin...)   │
    └───────────┬───────────────────┘
                ↓
    ┌───────────────────────────────┐
    │     Underlying Devices        │
    │   (物理磁盘、分区、其他DM设备)  │
    └───────────────────────────────┘
```

核心数据结构：

```c
/* 映射设备 */
struct mapped_device {
    struct rw_semaphore io_lock;
    struct mutex suspend_lock;
    
    struct request_queue *queue;
    struct gendisk *disk;
    
    atomic_t holders;
    atomic_t open_count;
    
    struct dm_table *map;
    
    /* I/O统计 */
    struct dm_stats stats;
    
    /* 事件处理 */
    atomic_t event_nr;
    wait_queue_head_t eventq;
    
    /* 工作队列 */
    struct workqueue_struct *wq;
};

/* 映射表 */
struct dm_table {
    struct mapped_device *md;
    unsigned type;
    
    /* 目标设备数组 */
    unsigned int num_targets;
    struct dm_target *targets;
    
    /* 设备限制 */
    struct queue_limits limits;
    
    /* 底层设备列表 */
    struct list_head devices;
};

/* 目标驱动 */
struct target_type {
    const char *name;
    struct module *module;
    unsigned version[3];
    
    /* 构造和析构 */
    int (*ctr)(struct dm_target *ti, unsigned argc, char **argv);
    void (*dtr)(struct dm_target *ti);
    
    /* I/O处理 */
    int (*map)(struct dm_target *ti, struct bio *bio);
    int (*end_io)(struct dm_target *ti, struct bio *bio, int error);
    
    /* 管理操作 */
    int (*preresume)(struct dm_target *ti);
    void (*resume)(struct dm_target *ti);
    void (*presuspend)(struct dm_target *ti);
    void (*postsuspend)(struct dm_target *ti);
    
    /* 状态查询 */
    void (*status)(struct dm_target *ti, status_type_t type,
                   char *result, unsigned maxlen);
    
    struct list_head list;
};
```

### 7.4.2 线性映射与条带化

#### 线性映射（dm-linear）

最简单的目标类型，将虚拟设备的连续区域映射到物理设备：

```c
struct linear_c {
    struct dm_dev *dev;
    sector_t start;  /* 物理设备起始扇区 */
};

static int linear_map(struct dm_target *ti, struct bio *bio)
{
    struct linear_c *lc = ti->private;
    
    /* 重定向到底层设备 */
    bio->bi_bdev = lc->dev->bdev;
    bio->bi_iter.bi_sector = lc->start + dm_target_offset(ti, bio);
    
    return DM_MAPIO_REMAPPED;
}
```

使用示例：
```bash
# 创建线性映射：将/dev/sdb1和/dev/sdc1连接
echo "0 1000000 linear /dev/sdb1 0" | dmsetup create linear1
echo "1000000 1000000 linear /dev/sdc1 0" | dmsetup create linear2
```

#### 条带化（dm-stripe）

将I/O分散到多个设备以提高并行性：

```c
struct stripe_c {
    uint32_t stripes;        /* 条带数量 */
    uint32_t stripe_width;   /* 条带宽度(扇区) */
    
    /* 条带数组 */
    struct stripe {
        struct dm_dev *dev;
        sector_t physical_start;
    } stripe[0];
};

static int stripe_map(struct dm_target *ti, struct bio *bio)
{
    struct stripe_c *sc = ti->private;
    sector_t sector = bio->bi_iter.bi_sector;
    uint32_t stripe;
    
    /* 计算条带索引 */
    stripe = sector_div(sector, sc->stripe_width) % sc->stripes;
    
    /* 重定向到对应条带 */
    bio->bi_bdev = sc->stripe[stripe].dev->bdev;
    bio->bi_iter.bi_sector = sc->stripe[stripe].physical_start +
                             (sector * sc->stripe_width) +
                             (bio->bi_iter.bi_sector % sc->stripe_width);
    
    return DM_MAPIO_REMAPPED;
}
```

### 7.4.3 快照与精简配置

#### 快照（dm-snapshot）

实现时间点快照功能：

```c
struct dm_snapshot {
    struct rw_semaphore lock;
    
    struct dm_dev *origin;    /* 原始设备 */
    struct dm_dev *cow;       /* COW设备 */
    
    /* 异常存储 */
    struct dm_exception_store *store;
    
    /* 异常哈希表 */
    struct exception_table complete;
    struct exception_table pending;
    
    /* 块大小 */
    uint32_t chunk_size;
    uint32_t chunk_shift;
    
    /* 位图跟踪 */
    unsigned long *tracked_chunk_bitset;
};

/* COW处理 */
static int snapshot_map(struct dm_target *ti, struct bio *bio)
{
    struct dm_snapshot *s = ti->private;
    chunk_t chunk;
    
    chunk = bio->bi_iter.bi_sector >> s->chunk_shift;
    
    /* 查找异常映射 */
    e = dm_lookup_exception(&s->complete, chunk);
    if (e) {
        /* 已复制，重定向到COW设备 */
        remap_exception(s, e, bio, chunk);
        return DM_MAPIO_REMAPPED;
    }
    
    /* 首次写入，触发COW */
    if (bio_data_dir(bio) == WRITE) {
        pe = __lookup_pending_exception(s, chunk);
        if (!pe) {
            /* 创建挂起异常 */
            pe = alloc_pending_exception(s);
            start_copy(pe);
        }
        bio_list_add(&pe->origin_bios, bio);
        return DM_MAPIO_SUBMITTED;
    }
    
    /* 读操作，直接访问原始设备 */
    bio->bi_bdev = s->origin->bdev;
    return DM_MAPIO_REMAPPED;
}
```

#### 精简配置（dm-thin）

实现存储过度分配和高效快照：

```c
struct thin_c {
    struct dm_pool_metadata *pmd;    /* 元数据 */
    struct dm_bio_prison *prison;    /* bio监狱 */
    
    struct pool {
        struct bio_list deferred_bios;
        struct workqueue_struct *wq;
        
        /* 数据块分配 */
        dm_block_t offset_mask;
        dm_block_t low_water_blocks;
        
        /* 精简设备列表 */
        struct list_head active_thins;
    } *pool;
    
    dm_thin_id dev_id;
    struct dm_thin_device *td;
};

/* 精简映射 */
static int thin_map(struct dm_target *ti, struct bio *bio)
{
    struct thin_c *tc = ti->private;
    dm_block_t block = bio->bi_iter.bi_sector >> tc->pool->block_shift;
    struct dm_thin_lookup_result result;
    
    /* 查找映射 */
    r = dm_thin_find_block(tc->td, block, 1, &result);
    
    if (r == 0) {
        /* 块已分配 */
        remap_to_pool(tc, bio, result.block);
        return DM_MAPIO_REMAPPED;
    }
    
    if (r == -ENODATA) {
        /* 块未分配 */
        if (bio_data_dir(bio) == READ) {
            /* 读零 */
            zero_fill_bio(bio);
            bio_endio(bio);
            return DM_MAPIO_SUBMITTED;
        }
        
        /* 写时分配 */
        schedule_zero_fill_copy(tc, bio);
        return DM_MAPIO_SUBMITTED;
    }
    
    return DM_MAPIO_KILL;
}
```

### 7.4.4 加密与完整性保护

#### dm-crypt

提供透明的块设备加密：

```c
struct crypt_config {
    struct dm_dev *dev;
    sector_t start;
    
    /* 加密参数 */
    struct crypto_skcipher *tfm;
    char *cipher_string;
    unsigned key_size;
    u8 *key;
    
    /* IV生成 */
    struct iv_operations {
        int (*ctr)(struct crypt_config *cc);
        void (*dtr)(struct crypt_config *cc);
        int (*generator)(struct crypt_config *cc, u8 *iv,
                        struct dm_crypt_request *dmreq);
    } *iv_gen_ops;
    
    /* 工作队列 */
    struct workqueue_struct *queue;
    struct workqueue_struct *crypt_queue;
    
    /* 内存池 */
    mempool_t *req_pool;
    mempool_t *page_pool;
};

/* 加密I/O处理 */
static int crypt_map(struct dm_target *ti, struct bio *bio)
{
    struct crypt_config *cc = ti->private;
    struct dm_crypt_io *io;
    
    /* 分配加密I/O上下文 */
    io = mempool_alloc(cc->io_pool, GFP_NOIO);
    io->cc = cc;
    io->base_bio = bio;
    io->sector = bio->bi_iter.bi_sector;
    
    if (bio_data_dir(bio) == READ) {
        /* 读：先读取加密数据，后解密 */
        kcryptd_queue_read(io);
    } else {
        /* 写：先加密，后写入 */
        kcryptd_queue_crypt(io);
    }
    
    return DM_MAPIO_SUBMITTED;
}
```

使用示例：
```bash
# 创建加密设备
cryptsetup luksFormat /dev/sdb1
cryptsetup open /dev/sdb1 encrypted
mkfs.ext4 /dev/mapper/encrypted
```

---

## 7.5 性能优化技术

### 7.5.1 I/O合并与请求重排

请求合并减少设备访问次数：

```c
/* 前向合并：新bio合并到已有request前面 */
bool blk_attempt_front_merge(struct request_queue *q,
                             struct request *rq, struct bio *bio)
{
    if (!blk_rq_merge_ok(rq, bio))
        return false;
    
    if (!ll_front_merge_fn(q, rq, bio))
        return false;
    
    /* 更新request */
    rq->__sector = bio->bi_iter.bi_sector;
    rq->__data_len += bio->bi_iter.bi_size;
    
    /* 链接bio */
    bio->bi_next = rq->bio;
    rq->bio = bio;
    
    return true;
}

/* 后向合并：新bio合并到已有request后面 */
bool blk_attempt_back_merge(struct request_queue *q,
                            struct request *rq, struct bio *bio)
{
    if (!blk_rq_merge_ok(rq, bio))
        return false;
    
    if (!ll_back_merge_fn(q, rq, bio))
        return false;
    
    /* 更新request */
    rq->__data_len += bio->bi_iter.bi_size;
    
    /* 链接bio */
    rq->biotail->bi_next = bio;
    rq->biotail = bio;
    
    return true;
}
```

### 7.5.2 预读策略与缓存管理

预读机制提高顺序读性能：

```c
struct file_ra_state {
    pgoff_t start;          /* 预读窗口起始 */
    unsigned int size;      /* 当前窗口大小 */
    unsigned int async_size;/* 异步预读大小 */
    unsigned int ra_pages;  /* 最大预读页数 */
    unsigned int mmap_miss; /* mmap缓存未命中计数 */
    loff_t prev_pos;       /* 上次读取位置 */
};

/* 预读算法 */
static unsigned long get_next_ra_size(struct file_ra_state *ra,
                                      unsigned long max)
{
    unsigned long cur = ra->size;
    unsigned long newsize;
    
    if (cur < max / 16)
        newsize = 4 * cur;    /* 快速增长 */
    else
        newsize = 2 * cur;    /* 线性增长 */
    
    return min(newsize, max);
}
```

### 7.5.3 写回机制与脏页控制

脏页写回的触发条件：

```c
struct bdi_writeback {
    struct backing_dev_info *bdi;
    
    unsigned long state;
    unsigned long last_old_flush;
    
    struct list_head b_dirty;      /* 脏inode列表 */
    struct list_head b_io;         /* 正在写回列表 */
    struct list_head b_more_io;    /* 需要更多I/O列表 */
    struct list_head b_dirty_time; /* 只有时间戳变化 */
    
    struct percpu_counter stat[NR_WB_STAT_ITEMS];
};

/* 写回控制参数 */
int dirty_background_ratio = 10;  /* 后台写回阈值10% */
int dirty_ratio = 20;             /* 前台写回阈值20% */
unsigned int dirty_expire_interval = 30 * HZ; /* 30秒过期 */
```

### 7.5.4 I/O统计与性能监控

块设备统计信息：

```c
struct disk_stats {
    unsigned long sectors[NR_STAT_GROUPS];  /* 读写扇区数 */
    unsigned long ios[NR_STAT_GROUPS];      /* I/O次数 */
    unsigned long merges[NR_STAT_GROUPS];   /* 合并次数 */
    unsigned long ticks[NR_STAT_GROUPS];    /* I/O时间 */
    unsigned long io_ticks;                 /* 总I/O时间 */
    unsigned long time_in_queue;            /* 队列时间 */
};
```

通过/proc/diskstats监控：
```bash
# 字段说明：
# 1-3: 主次设备号和设备名
# 4: 读完成次数
# 5: 合并读次数
# 6: 读扇区数
# 7: 读耗时(ms)
# 8-11: 写相关统计
# 12: 正在进行的I/O
# 13: I/O耗时
# 14: 加权I/O耗时

cat /proc/diskstats
```

---

## 7.6 本章小结

本章深入探讨了Linux块设备层的设计与实现，主要内容包括：

### 核心概念回顾

1. **块设备架构**：
   - bio作为基本I/O单位，支持散列聚集
   - request经过合并优化，减少设备访问
   - gendisk抽象物理设备，提供统一接口

2. **I/O调度演进**：
   - 传统调度器(CFQ/Deadline)适合机械硬盘
   - 多队列调度器(mq-deadline/bfq/kyber)为SSD优化
   - 不同工作负载需要选择合适的调度策略

3. **blk-mq革新**：
   - 软件队列与硬件队列分离，减少锁竞争
   - 标签管理控制并发度
   - NUMA感知的队列映射提升性能

4. **Device Mapper灵活性**：
   - 目标驱动架构支持多种虚拟设备
   - 快照和精简配置实现高级存储功能
   - 透明加密保护数据安全

### 关键性能优化

- **I/O合并**：$合并率 = \frac{合并请求数}{总请求数} \times 100\%$
- **预读效率**：$命中率 = \frac{预读命中页数}{总预读页数} \times 100\%$
- **队列深度**：$平均队列深度 = \frac{\sum 队列长度 \times 时间}{总时间}$
- **延迟分析**：$总延迟 = 队列延迟 + 服务延迟$

### 发展趋势

1. **新硬件支持**：ZNS SSD、持久内存、CXL存储
2. **智能调度**：机器学习驱动的I/O预测
3. **容器优化**：cgroup v2的I/O隔离增强
4. **可编程存储**：eBPF在块层的应用

---

## 7.7 练习题

### 基础题

**1. bio与request的关系**
分析以下代码片段，解释bio如何转换为request：
```c
void blk_mq_bio_to_request(struct request *rq, struct bio *bio)
{
    rq->__sector = bio->bi_iter.bi_sector;
    rq->__data_len = bio->bi_iter.bi_size;
    rq->bio = rq->biotail = bio;
}
```

<details>
<summary>提示</summary>
考虑bio链表结构和request的聚合特性
</details>

<details>
<summary>答案</summary>

bio是基本I/O单位，request是优化后的请求单位。转换过程：
1. 单个或多个bio可以组成一个request
2. request记录起始扇区和总数据长度
3. bio通过链表连接，biotail指向最后一个bio
4. 相邻的bio可以合并到同一个request中
</details>

**2. I/O调度器选择**
某服务器运行PostgreSQL数据库，存储设备是SATA SSD，应该选择哪种I/O调度器？说明理由。

<details>
<summary>提示</summary>
考虑数据库的I/O特征：随机读写多、延迟敏感
</details>

<details>
<summary>答案</summary>

推荐使用mq-deadline调度器：
1. 数据库以随机I/O为主，CFQ的顺序优化效果有限
2. mq-deadline保证延迟上界，适合事务处理
3. 比none调度器提供更好的公平性
4. 相比bfq开销更小，适合服务器环境
</details>

**3. blk-mq队列映射**
系统有8个CPU，NVMe设备支持4个硬件队列，描述默认的CPU到硬件队列映射。

<details>
<summary>提示</summary>
轮询分配算法：CPU % 硬件队列数
</details>

<details>
<summary>答案</summary>

默认轮询映射：
- CPU 0, 4 → HWQ 0
- CPU 1, 5 → HWQ 1  
- CPU 2, 6 → HWQ 2
- CPU 3, 7 → HWQ 3

每个硬件队列服务2个CPU，保证负载均衡。
</details>

### 挑战题

**4. 设计I/O合并算法**
设计一个改进的I/O合并算法，同时考虑：
- 空间局部性（相邻扇区）
- 时间局部性（请求时间接近）
- 公平性（防止某些进程饥饿）

<details>
<summary>提示</summary>
可以使用时间窗口和距离阈值的组合策略
</details>

<details>
<summary>答案</summary>

改进的合并算法设计：

1. **双重阈值策略**：
   - 距离阈值：扇区距离 < 128KB可合并
   - 时间阈值：100ms内的请求可合并

2. **公平性保证**：
   - 每个进程维护独立的合并窗口
   - 设置最大合并次数限制(如16次)
   - 超时自动派发防止无限等待

3. **自适应调整**：
   - 监测命中率动态调整阈值
   - 顺序I/O增大窗口，随机I/O减小窗口

4. **优先级考虑**：
   - 高优先级I/O降低合并等待时间
   - 实时任务直接派发不等待合并
</details>

**5. 实现简单的设备映射目标**
设计一个mirror（镜像）目标驱动的核心逻辑，要求：
- 读操作负载均衡
- 写操作同步复制
- 处理设备故障

<details>
<summary>提示</summary>
需要考虑读写分离、故障检测和降级模式
</details>

<details>
<summary>答案</summary>

Mirror目标驱动设计：

1. **数据结构**：
   - 镜像设备数组
   - 设备状态标志(正常/故障)
   - 轮询索引用于负载均衡

2. **读操作处理**：
   - 轮询选择健康设备
   - 失败时自动切换到备份
   - 记录读错误用于故障检测

3. **写操作处理**：
   - 克隆bio到所有健康设备
   - 等待所有写完成
   - 任一失败则标记设备故障

4. **故障恢复**：
   - 定期尝试恢复故障设备
   - 支持在线重同步
   - 维护脏块位图加速恢复

5. **性能优化**：
   - 异步写入提高吞吐量
   - 读请求NUMA亲和性
   - 写合并减少开销
</details>

**6. 分析blk-mq性能瓶颈**
某NVMe设备在高并发场景下性能不及预期，如何分析和优化？

<details>
<summary>提示</summary>
从软硬件队列映射、标签分配、中断亲和性等方面分析
</details>

<details>
<summary>答案</summary>

性能分析和优化方案：

1. **诊断步骤**：
   - 检查队列深度是否充分利用
   - 分析软中断分布是否均衡
   - 监测标签分配等待时间
   - 评估NUMA节点访问模式

2. **可能的瓶颈**：
   - 队列映射不当导致热点
   - 标签数量不足限制并发
   - 中断处理集中在少数CPU
   - 跨NUMA访问增加延迟

3. **优化措施**：
   - 调整nr_hw_queues匹配CPU数
   - 增加queue_depth提高并发度
   - 设置中断亲和性分散负载
   - 使用NUMA感知的队列映射

4. **参数调优**：
```bash
echo 256 > /sys/block/nvme0n1/queue/nr_requests
echo 0 > /sys/block/nvme0n1/queue/rq_affinity
echo 2 > /sys/block/nvme0n1/queue/nomerges
```

5. **性能验证**：
   - 使用fio测试不同队列深度
   - 监控iostat查看利用率
   - 分析blktrace追踪I/O路径
</details>

**7. 设计智能I/O调度器**
基于机器学习设计一个自适应I/O调度器，描述：
- 特征提取方案
- 模型选择理由
- 在线学习机制

<details>
<summary>提示</summary>
考虑I/O模式识别、工作负载分类、参数自动调优
</details>

<details>
<summary>答案</summary>

智能I/O调度器设计：

1. **特征工程**：
   - I/O大小分布(4K/64K/1M比例)
   - 顺序/随机比例
   - 读写比例
   - 进程间I/O相关性
   - 时间模式(周期性/突发性)

2. **模型架构**：
   - 使用决策树进行工作负载分类
   - LSTM预测未来I/O模式
   - 强化学习优化调度参数

3. **在线学习**：
   - 滑动窗口收集最近N秒统计
   - 增量更新模型参数
   - A/B测试验证改进效果

4. **调度策略**：
   - 数据库负载→优化延迟
   - 批处理→最大化吞吐量
   - 混合负载→动态权衡

5. **实现考虑**：
   - eBPF收集特征避免开销
   - 模型推理使用查找表加速
   - 降级机制应对异常情况
</details>

**8. 优化Device Mapper性能**
分析dm-crypt的性能开销，提出优化方案。

<details>
<summary>提示</summary>
考虑加密算法、并行处理、硬件加速等因素
</details>

<details>
<summary>答案</summary>

dm-crypt性能优化：

1. **性能瓶颈分析**：
   - CPU密集的加密运算
   - 内存拷贝开销
   - 单线程处理限制
   - IV生成计算

2. **算法优化**：
   - 使用AES-NI硬件加速
   - 选择XTS模式提高并行度
   - 优化IV生成算法

3. **并行化改进**：
   - 多CPU并行加密
   - 异步加密队列
   - per-CPU工作队列

4. **内存优化**：
   - 使用内存池减少分配
   - 零拷贝技术
   - 对齐优化cache性能

5. **配置建议**：
```bash
# 使用硬件加速
cryptsetup --cipher aes-xts-plain64 \
          --key-size 512 \
          --use-random

# 调整工作队列
echo 4 > /sys/module/dm_crypt/parameters/num_threads

# 设置更大的块大小
cryptsetup --perf-no_read_workqueue \
          --perf-no_write_workqueue
```

6. **性能监控**：
   - 测量加密吞吐量
   - 分析CPU利用率分布
   - 追踪I/O延迟增加
</details>

---

## 7.8 常见陷阱与错误（Gotchas）

### 调试技巧

1. **I/O挂起诊断**
```bash
# 检查挂起的I/O
cat /proc/<pid>/stack
cat /sys/kernel/debug/block/<dev>/hctx*/busy

# 使用blktrace追踪
blktrace -d /dev/sda -o trace
blkparse -i trace
```

2. **性能问题定位**
```bash
# 监控队列深度
iostat -x 1

# 查看调度器统计
cat /sys/block/sda/queue/scheduler
grep . /sys/block/sda/queue/iosched/*
```

### 常见错误

1. **bio使用错误**
   - ❌ 修改已提交的bio
   - ❌ 忘记调用bio_put释放引用
   - ✅ 克隆bio进行修改

2. **请求队列配置**
   - ❌ 超过设备能力的队列深度
   - ❌ 不匹配的扇区大小
   - ✅ 根据设备特性设置限制

3. **DM目标实现**
   - ❌ 同步I/O阻塞映射函数
   - ❌ 不处理设备热插拔
   - ✅ 异步处理和错误恢复

4. **性能调优误区**
   - ❌ 盲目增加预读大小
   - ❌ 所有设备用同一调度器
   - ✅ 基于测试数据调优

---

## 7.9 最佳实践检查清单

### 设计审查要点

#### 块设备驱动
- [ ] 正确设置队列限制（max_sectors、max_segments）
- [ ] 实现超时处理机制
- [ ] 支持blk-mq多队列模式
- [ ] 处理电源管理事件
- [ ] 提供合理的默认队列深度

#### I/O路径优化
- [ ] 最小化内存分配
- [ ] 避免不必要的内存拷贝
- [ ] 使用per-CPU数据减少竞争
- [ ] 批量处理提高效率
- [ ] 正确设置内存屏障

#### Device Mapper目标
- [ ] 支持在线扩容
- [ ] 处理底层设备错误
- [ ] 实现状态持久化
- [ ] 提供有意义的状态信息
- [ ] 支持非阻塞I/O路径

#### 性能监控
- [ ] 导出关键性能指标
- [ ] 支持tracepoint调试
- [ ] 提供sysfs调优接口
- [ ] 实现I/O统计收集
- [ ] 集成到标准监控工具

#### 可靠性保证
- [ ] 处理设备移除场景
- [ ] 实现I/O错误重试
- [ ] 支持I/O超时取消
- [ ] 保证数据一致性
- [ ] 优雅处理资源耗尽

### 部署建议

1. **生产环境配置**
```bash
# 设置合理的脏页比例
echo 5 > /proc/sys/vm/dirty_background_ratio
echo 10 > /proc/sys/vm/dirty_ratio

# 调整预读大小
blockdev --setra 256 /dev/sda

# 设置I/O调度器
echo mq-deadline > /sys/block/sda/queue/scheduler

# 启用多队列
echo 8 > /sys/block/sda/queue/nr_requests
```

2. **性能基准测试**
```bash
# 顺序读写
fio --name=seq --rw=write --bs=1M --size=10G

# 随机IOPS
fio --name=rand --rw=randread --bs=4K --iodepth=32

# 混合负载
fio --name=mixed --rw=randrw --rwmixread=70 --bs=4K
```

3. **故障注入测试**
```bash
# 模拟I/O错误
echo 1 > /sys/block/sda/make-it-fail

# 模拟I/O延迟
echo "100" > /sys/kernel/debug/fail_io_timeout/probability
```

---

*第7章完成。本章详细剖析了Linux块设备层的架构演进，从传统的单队列到现代的blk-mq，从简单的调度算法到智能化的I/O优化。通过学习本章，您应该能够理解存储栈的关键组件，分析I/O性能问题，并针对不同场景选择合适的配置策略。*