# 第3章：内存管理架构

## 本章导读

Linux 内核的内存管理是操作系统最复杂也最关键的子系统之一。从物理页框的分配到虚拟地址空间的映射，从高效的内存分配器到智能的页面回收机制，内存管理直接影响着系统的性能、稳定性和可扩展性。本章将深入剖析 Linux 内存管理的核心机制，理解其如何在有限的物理内存上支撑起现代计算的无限可能。

学习目标：
- 掌握物理内存管理的层次结构：节点(node)、区域(zone)、页框(page frame)
- 理解虚拟内存的实现机制：页表结构、地址转换、TLB 管理
- 深入分析内存分配器的设计：伙伴系统、slab/slub/slob 分配器
- 掌握内存回收策略：LRU 算法、页面回收、OOM killer 机制
- 理解现代内存管理特性：大页支持、NUMA 优化、内存压缩

## 3.1 物理内存管理架构

### 3.1.1 内存布局与初始化

Linux 启动时，内核首先需要建立对物理内存的认知。在 x86_64 架构上，这个过程涉及多个阶段：

```
物理内存布局 (典型 x86_64 系统)：

0x0000000000000000 +------------------+
                   | 实模式中断向量表  |
0x0000000000001000 +------------------+
                   | BIOS 数据区      |
0x00000000000A0000 +------------------+
                   | VGA 显存         |
0x0000000000100000 +------------------+ <- 1MB
                   | 内核代码段       |
                   | 内核数据段       |
                   | 内核 BSS 段      |
0x0000000001000000 +------------------+ <- 16MB
                   | 可用物理内存     |
                   |                  |
                   | (主要内存区域)   |
                   |                  |
0x00000000XXXX0000 +------------------+
                   | ACPI 表          |
                   | 设备保留内存     |
0x00000000FFFF0000 +------------------+
```

内核通过 E820 内存映射（或 UEFI 的内存描述符）获取可用内存信息。这些信息存储在 `struct e820_table` 中，包含每个内存区域的起始地址、大小和类型（可用、保留、ACPI 等）。

### 3.1.2 NUMA 架构与内存节点

现代服务器普遍采用 NUMA（Non-Uniform Memory Access）架构，内存访问延迟取决于 CPU 与内存的物理距离：

```
NUMA 拓扑示例（双路服务器）：

    Node 0                          Node 1
+-------------+                 +-------------+
|   CPU 0-7   |<---QPI/UPI---->|  CPU 8-15   |
+-------------+                 +-------------+
      |                               |
+-------------+                 +-------------+
| Local Memory|                 | Local Memory|
|   (64GB)    |                 |   (64GB)    |
+-------------+                 +-------------+

访问延迟：
- 本地内存访问：~60ns
- 远程内存访问：~100-120ns
```

内核使用 `struct pglist_data`（通常称为 `pg_data_t`）表示一个 NUMA 节点：

```c
typedef struct pglist_data {
    struct zone node_zones[MAX_NR_ZONES];  // 该节点的内存区域
    struct zonelist node_zonelists[MAX_ZONELISTS]; // 内存分配备选列表
    int nr_zones;                          // 区域数量
    unsigned long node_start_pfn;          // 起始页框号
    unsigned long node_present_pages;      // 物理页面总数
    unsigned long node_spanned_pages;      // 跨度页面数（包含空洞）
    int node_id;                           // 节点 ID
    wait_queue_head_t kswapd_wait;         // kswapd 等待队列
    struct task_struct *kswapd;            // 页面回收线程
    // ... 更多字段
} pg_data_t;
```

### 3.1.3 内存区域（Zone）管理

每个 NUMA 节点的内存被划分为多个区域（zone），反映不同的硬件限制：

```c
enum zone_type {
    ZONE_DMA,      // 0-16MB，ISA 设备 DMA
    ZONE_DMA32,    // 0-4GB，32位设备 DMA
    ZONE_NORMAL,   // 常规内存，直接映射
    ZONE_HIGHMEM,  // 高端内存（仅32位系统）
    ZONE_MOVABLE,  // 可迁移页面，支持内存热插拔
    ZONE_DEVICE,   // 设备内存（如 GPU、持久内存）
    __MAX_NR_ZONES
};
```

每个区域由 `struct zone` 结构管理：

```c
struct zone {
    unsigned long _watermark[NR_WMARK];    // 水位线：min、low、high
    unsigned long watermark_boost;          // 动态水位提升
    long lowmem_reserve[MAX_NR_ZONES];     // 为高优先级预留
    struct pglist_data *zone_pgdat;        // 所属节点
    struct per_cpu_pages __percpu *per_cpu_pageset; // Per-CPU 页面缓存
    
    // 空闲页面管理（伙伴系统）
    struct free_area free_area[MAX_ORDER];
    unsigned long managed_pages;           // 被伙伴系统管理的页面
    unsigned long spanned_pages;           // 总跨度
    unsigned long present_pages;           // 实际存在的页面
    
    // 内存压缩
    unsigned long compact_cached_free_pfn; // 压缩扫描位置
    unsigned long compact_init_free_pfn;
    
    // LRU 链表（页面回收）
    struct lruvec lruvec;
    unsigned long pages_scanned;           // 已扫描页面数
    unsigned long flags;                   // 区域状态标志
    
    // 统计信息
    atomic_long_t vm_stat[NR_VM_ZONE_STAT_ITEMS];
    atomic_long_t vm_numa_stat[NR_VM_NUMA_STAT_ITEMS];
} ____cacheline_internodealigned_in_smp;
```

### 3.1.4 页框（Page Frame）与 struct page

物理内存以页框（通常 4KB）为单位管理。每个页框对应一个 `struct page` 结构：

```c
struct page {
    unsigned long flags;    // 页面状态标志（PG_locked, PG_dirty, PG_lru 等）
    
    union {
        // 多种用途的联合体，节省内存
        struct {    // 页面缓存和匿名页面
            struct list_head lru;       // LRU 链表
            struct address_space *mapping; // 所属地址空间
            pgoff_t index;              // 在映射中的偏移
            unsigned long private;      // 文件系统私有数据
        };
        struct {    // slab 分配器
            struct kmem_cache *slab_cache;
            void *freelist;             // 空闲对象链表
            void *s_mem;                // slab 首个对象
        };
        struct {    // 复合页（大页）
            unsigned long compound_head;
            unsigned char compound_dtor;
            unsigned char compound_order;
            atomic_t compound_mapcount;
        };
    };
    
    union {
        atomic_t _mapcount;     // 页表映射计数
        unsigned int page_type; // 页面类型（如 buddy、slab）
    };
    
    atomic_t _refcount;         // 引用计数
    
#ifdef CONFIG_MEMCG
    struct mem_cgroup *mem_cgroup; // 内存控制组
#endif
};
```

关键概念：
- **页面标志（flags）**：记录页面状态，如脏页、锁定、正在回收等
- **引用计数（_refcount）**：追踪页面使用者数量
- **映射计数（_mapcount）**：记录页面被多少个页表映射
- **LRU 链表**：用于页面回收算法

### 3.1.5 伙伴系统（Buddy System）

伙伴系统是 Linux 物理内存分配的核心算法，通过将内存块组织成 2^n 页面的块来减少外部碎片：

```
伙伴系统原理：

Order 0:  □ □ □ □ □ □ □ □  (单页，4KB)
Order 1:  □□ □□ □□ □□      (2页，8KB)
Order 2:  □□□□ □□□□        (4页，16KB)
Order 3:  □□□□□□□□          (8页，32KB)
...
Order 10: □□□□□□□□...       (1024页，4MB)

分配过程（申请 8KB）：
1. 检查 Order 1 是否有空闲块 -> 无
2. 检查 Order 2 是否有空闲块 -> 有
3. 分裂 Order 2 块：
   - 取一半（8KB）满足请求
   - 另一半放入 Order 1 空闲链表
```

伙伴系统的核心数据结构：

```c
struct free_area {
    struct list_head free_list[MIGRATE_TYPES]; // 按迁移类型分类的空闲链表
    unsigned long nr_free;                     // 空闲块数量
};

// 页面迁移类型（减少碎片化）
enum migratetype {
    MIGRATE_UNMOVABLE,     // 不可迁移（内核数据结构）
    MIGRATE_MOVABLE,       // 可迁移（用户页面）
    MIGRATE_RECLAIMABLE,   // 可回收（文件缓存）
    MIGRATE_PCPTYPES,      // Per-CPU 页面类型数
    MIGRATE_HIGHATOMIC,    // 高优先级原子分配
    MIGRATE_CMA,           // 连续内存分配器
    MIGRATE_ISOLATE,       // 隔离页面（内存热插拔）
    MIGRATE_TYPES
};
```

分配函数调用链：
```
alloc_pages()
  -> alloc_pages_current()  // NUMA 策略
    -> __alloc_pages_nodemask()  // 核心分配函数
      -> get_page_from_freelist()  // 快速路径
        -> rmqueue()  // 从伙伴系统取页
          -> __rmqueue()  // 实际分配
      -> __alloc_pages_slowpath()  // 慢速路径（内存紧张）
        -> __alloc_pages_direct_reclaim()  // 直接回收
        -> __alloc_pages_direct_compact()  // 内存压缩
        -> oom_kill_process()  // OOM killer
```

## 3.2 虚拟内存机制

### 3.2.1 虚拟地址空间布局

Linux 为每个进程提供独立的虚拟地址空间。在 x86_64 架构上，虽然硬件支持 48 位虚拟地址（256TB），但内核采用了分割式布局：

```
x86_64 虚拟地址空间布局（5级页表，57位地址）：

0x0000000000000000 +------------------+ <- 用户空间开始
                   | 程序代码段(.text) |
                   | 程序数据段(.data) |
                   | BSS 段(.bss)      |
                   | 堆（向上增长）↓    |
                   |                   |
                   | ↓ MMAP 区域 ↓     |
                   |                   |
                   | ↑ 栈（向下增长）   |
0x00007FFFFFFFFFFF +------------------+ <- 用户空间结束 (128TB)
                   | 不可访问区域       |
0xFFFF800000000000 +------------------+ <- 内核空间开始
                   | vmalloc 区域      |
                   | 持久映射区         |
                   | 固定映射区         |
                   | 模块区域          |
0xFFFF888000000000 +------------------+
                   | 直接映射区        |
                   | (所有物理内存)     |
0xFFFFFFFFFFFFFFFF +------------------+ <- 内核空间结束
```

用户进程的虚拟地址空间由 `struct mm_struct` 管理：

```c
struct mm_struct {
    struct vm_area_struct *mmap;       // VMA 链表
    struct rb_root mm_rb;               // VMA 红黑树（快速查找）
    unsigned long mmap_base;            // mmap 区域基地址
    unsigned long task_size;            // 用户空间大小
    
    pgd_t *pgd;                         // 页全局目录
    atomic_t mm_users;                  // 用户计数（线程数）
    atomic_t mm_count;                  // 引用计数
    
    unsigned long total_vm;             // 总虚拟内存页数
    unsigned long locked_vm;            // 锁定页数
    unsigned long pinned_vm;            // 固定页数
    unsigned long data_vm;              // 数据段页数
    unsigned long exec_vm;              // 可执行页数
    unsigned long stack_vm;             // 栈页数
    
    unsigned long start_code, end_code;     // 代码段范围
    unsigned long start_data, end_data;     // 数据段范围
    unsigned long start_brk, brk;           // 堆范围
    unsigned long start_stack;              // 栈起始地址
    unsigned long arg_start, arg_end;       // 参数范围
    unsigned long env_start, env_end;       // 环境变量范围
    
    spinlock_t page_table_lock;        // 页表锁
    struct rw_semaphore mmap_sem;       // mmap 信号量
    
    struct list_head mmlist;            // 所有 mm_struct 链表
    
    unsigned long hiwater_rss;          // RSS 高水位
    unsigned long hiwater_vm;           // 虚拟内存高水位
    
    // 更多字段...
};
```

### 3.2.2 虚拟内存区域（VMA）

虚拟地址空间被划分为多个虚拟内存区域（VMA），每个 VMA 代表一段连续的虚拟地址范围：

```c
struct vm_area_struct {
    unsigned long vm_start;             // VMA 起始地址
    unsigned long vm_end;               // VMA 结束地址
    
    struct vm_area_struct *vm_next, *vm_prev;  // 链表指针
    struct rb_node vm_rb;               // 红黑树节点
    
    struct mm_struct *vm_mm;            // 所属的 mm_struct
    pgprot_t vm_page_prot;              // 页面保护标志
    unsigned long vm_flags;             // VMA 标志
    
    // 共享映射
    struct {
        struct rb_node rb;
        unsigned long rb_subtree_last;
    } shared;
    
    struct list_head anon_vma_chain;    // 匿名 VMA 链
    struct anon_vma *anon_vma;          // 反向映射
    
    const struct vm_operations_struct *vm_ops;  // VMA 操作
    unsigned long vm_pgoff;             // 文件偏移（页单位）
    struct file *vm_file;               // 映射的文件
    void *vm_private_data;              // 私有数据
    
    // NUMA 策略
    struct mempolicy *vm_policy;
    struct vm_userfaultfd_ctx vm_userfaultfd_ctx;
};

// VMA 标志
#define VM_READ     0x00000001  // 可读
#define VM_WRITE    0x00000002  // 可写
#define VM_EXEC     0x00000004  // 可执行
#define VM_SHARED   0x00000008  // 共享映射
#define VM_MAYREAD  0x00000010  // 可能可读
#define VM_MAYWRITE 0x00000020  // 可能可写
#define VM_MAYEXEC  0x00000040  // 可能可执行
#define VM_GROWSDOWN 0x00000100 // 向下增长（栈）
#define VM_LOCKED   0x00002000  // 页面锁定在内存
#define VM_HUGETLB  0x00400000  // 大页映射
```

### 3.2.3 多级页表机制

x86_64 使用 4 级或 5 级页表实现虚拟到物理地址转换：

```
4级页表地址转换（48位虚拟地址）：

虚拟地址：
+--------+--------+--------+--------+--------+------------+
| Sign   | PGD    | PUD    | PMD    | PTE    | Offset     |
| Extend | Index  | Index  | Index  | Index  | (12 bits)  |
| (16)   | (9)    | (9)    | (9)    | (9)    | 4KB page   |
+--------+--------+--------+--------+--------+------------+

转换过程：
CR3 寄存器 -> PGD 基地址
    |
    v
PGD[pgd_index] -> PUD 基地址
    |
    v
PUD[pud_index] -> PMD 基地址
    |
    v
PMD[pmd_index] -> PT 基地址
    |
    v
PTE[pte_index] -> 物理页框
    |
    v
物理地址 = 页框地址 + offset
```

页表项格式（x86_64 PTE）：

```
63                                                             0
+---+---+---+-----+---+---+---+---+---+---+---+---+-----------+
|NX |   | Reserved |   Physical Page Frame Number   | Flags    |
+---+---+---+-----+---+---+---+---+---+---+---+---+-----------+
  |                                                      |
  |                                                      +-> P(resent)
  |                                                      +-> R/W
  |                                                      +-> U/S
  |                                                      +-> PWT
  |                                                      +-> PCD
  |                                                      +-> A(ccessed)
  |                                                      +-> D(irty)
  |                                                      +-> PAT
  |                                                      +-> G(lobal)
  +-> No eXecute
```

内核页表操作宏：

```c
// 页表级别定义
typedef struct { unsigned long pgd; } pgd_t;
typedef struct { unsigned long pud; } pud_t;
typedef struct { unsigned long pmd; } pmd_t;
typedef struct { unsigned long pte; } pte_t;

// 地址转换辅助函数
#define pgd_index(addr) (((addr) >> PGDIR_SHIFT) & (PTRS_PER_PGD - 1))
#define pud_index(addr) (((addr) >> PUD_SHIFT) & (PTRS_PER_PUD - 1))
#define pmd_index(addr) (((addr) >> PMD_SHIFT) & (PTRS_PER_PMD - 1))
#define pte_index(addr) (((addr) >> PAGE_SHIFT) & (PTRS_PER_PTE - 1))

// 页表遍历
static inline pud_t *pud_offset(pgd_t *pgd, unsigned long address) {
    return (pud_t *)pgd_page_vaddr(*pgd) + pud_index(address);
}

static inline pmd_t *pmd_offset(pud_t *pud, unsigned long address) {
    return (pmd_t *)pud_page_vaddr(*pud) + pmd_index(address);
}

static inline pte_t *pte_offset_kernel(pmd_t *pmd, unsigned long address) {
    return (pte_t *)pmd_page_vaddr(*pmd) + pte_index(address);
}
```

### 3.2.4 TLB 管理

TLB（Translation Lookaside Buffer）缓存虚拟到物理地址的映射，避免每次访问内存都要遍历页表：

```
TLB 结构示例：

+----------+----------+--------+-------+-------+
| Valid    | ASID/PCID| VPN    | PFN   | Flags |
+----------+----------+--------+-------+-------+
| 1        | 0x123    | 0x7fff | 0x5432| RWX   |
| 1        | 0x123    | 0x8000 | 0x6789| R-X   |
| 0        | -        | -      | -     | -     |
+----------+----------+--------+-------+-------+

TLB 命中率对性能的影响：
- TLB 命中：~1 CPU 周期
- TLB 未命中：~100-200 CPU 周期（4级页表遍历）
```

Linux TLB 刷新策略：

```c
// TLB 刷新接口
void flush_tlb_all(void);                    // 刷新所有 TLB 项
void flush_tlb_mm(struct mm_struct *mm);     // 刷新特定进程的 TLB
void flush_tlb_page(struct vm_area_struct *vma, unsigned long addr);
void flush_tlb_range(struct vm_area_struct *vma, 
                     unsigned long start, unsigned long end);

// 懒惰 TLB 模式（内核线程优化）
struct mm_struct init_mm = {
    .mm_rb      = RB_ROOT,
    .pgd        = swapper_pg_dir,  // 内核页表
    .mm_users   = ATOMIC_INIT(2),
    .mm_count   = ATOMIC_INIT(1),
    .mmap_sem   = __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock = __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist     = LIST_HEAD_INIT(init_mm.mmlist),
};
```

### 3.2.5 缺页异常处理

当访问的虚拟地址没有对应的物理页面时，CPU 触发缺页异常：

```c
// x86_64 缺页异常处理入口
dotraplinkage void do_page_fault(struct pt_regs *regs, unsigned long error_code) {
    unsigned long address = read_cr2();  // 获取引起异常的地址
    
    // 错误码解析
    // bit 0: 0=页不存在, 1=保护违例
    // bit 1: 0=读访问, 1=写访问
    // bit 2: 0=内核模式, 1=用户模式
    // bit 3: 1=使用保留位
    // bit 4: 1=指令获取
    
    if (unlikely(kmmio_fault(regs, address)))
        return;
        
    if (unlikely(fault_in_kernel_space(address))) {
        // 内核空间缺页
        if (vmalloc_fault(address) >= 0)
            return;
    }
    
    // 用户空间缺页
    __do_page_fault(regs, error_code, address);
}

// 缺页处理核心逻辑
static int handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
                          unsigned int flags) {
    if (unlikely(is_vm_hugetlb_page(vma)))
        return hugetlb_fault(vma->vm_mm, vma, address, flags);
        
    pgd = pgd_offset(mm, address);
    pud = pud_alloc(mm, pgd, address);
    pmd = pmd_alloc(mm, pud, address);
    
    return handle_pte_fault(mm, vma, address, pte, pmd, flags);
}
```

缺页异常类型：
1. **次要缺页（Minor Fault）**：页面在内存中但未映射
2. **主要缺页（Major Fault）**：需要从磁盘读取
3. **写时复制（COW）**：写入共享页面时复制
4. **需求分页（Demand Paging）**：首次访问时分配

## 3.3 内存分配器

### 3.3.1 Slab 分配器架构

Slab 分配器是建立在伙伴系统之上的高效小对象分配器，解决了内部碎片问题：

```
Slab 分配器层次结构：

    kmem_cache (缓存描述符)
         |
    +----|----+----+
    |    |    |    |
  slab slab slab slab  (每个 slab 包含多个对象)
    |
  +---+---+---+---+
  |obj|obj|obj|obj|  (相同大小的对象)
  +---+---+---+---+

特点：
- 对象重用：避免频繁初始化/销毁
- CPU 缓存友好：热对象保持在缓存中
- 减少碎片：相同大小对象聚集
- 着色优化：避免缓存行冲突
```

核心数据结构：

```c
struct kmem_cache {
    struct kmem_cache_cpu __percpu *cpu_slab;  // Per-CPU 缓存
    unsigned long flags;                       // 缓存标志
    unsigned long min_partial;                  // 最小部分空闲 slab 数
    unsigned int size;                         // 对象大小
    unsigned int object_size;                  // 用户请求的大小
    unsigned int offset;                       // 空闲指针偏移
    
    struct kmem_cache_order_objects oo;        // slab 大小和对象数
    struct kmem_cache_order_objects max;       // 最大 slab 配置
    struct kmem_cache_order_objects min;       // 最小 slab 配置
    
    gfp_t allocflags;                          // 分配标志
    int refcount;                              // 引用计数
    void (*ctor)(void *);                      // 构造函数
    
    const char *name;                          // 缓存名称
    struct list_head list;                     // 全局缓存链表
    
    // SLUB 特有字段
    struct kmem_cache_node *node[MAX_NUMNODES]; // 节点数据
    
#ifdef CONFIG_SLAB_FREELIST_HARDENED
    unsigned long random;                      // 随机化种子（安全）
#endif
};

// Per-CPU slab 缓存
struct kmem_cache_cpu {
    void **freelist;        // 本地空闲对象链表
    unsigned long tid;      // 事务 ID（无锁操作）
    struct page *page;      // 当前 slab 页
    struct page *partial;   // 部分空闲 slab 链表
#ifdef CONFIG_SLUB_STATS
    unsigned stat[NR_SLUB_STAT_ITEMS];  // 统计信息
#endif
};
```

### 3.3.2 SLUB 分配器优化

SLUB 是 Slab 的简化版本，成为现代 Linux 的默认分配器：

```c
// SLUB 分配快速路径（无锁）
static __always_inline void *slab_alloc_node(struct kmem_cache *s,
        gfp_t gfpflags, int node, unsigned long addr) {
    void *object;
    struct kmem_cache_cpu *c;
    struct page *page;
    unsigned long tid;
    
redo:
    // 禁用抢占，获取 Per-CPU 数据
    preempt_disable();
    c = this_cpu_ptr(s->cpu_slab);
    
    // 事务开始
    tid = READ_ONCE(c->tid);
    barrier();
    
    object = c->freelist;
    page = c->page;
    
    if (unlikely(!object || !node_match(page, node))) {
        // 慢速路径：需要新的 slab
        object = __slab_alloc(s, gfpflags, node, addr, c);
    } else {
        // 快速路径：从 freelist 取对象
        void *next_object = get_freepointer(s, object);
        
        // 无锁更新（使用 cmpxchg）
        if (unlikely(!this_cpu_cmpxchg_double(
                s->cpu_slab->freelist, s->cpu_slab->tid,
                object, tid,
                next_object, next_tid(tid)))) {
            goto redo;  // CAS 失败，重试
        }
        
        stat(s, ALLOC_FASTPATH);
    }
    
    preempt_enable();
    return object;
}
```

SLUB 与 SLAB 的对比：

| 特性 | SLAB | SLUB | SLOB |
|------|------|------|------|
| 复杂度 | 高 | 中 | 低 |
| 内存开销 | 较大 | 中等 | 最小 |
| Per-CPU 缓存 | 复杂 | 简单 | 无 |
| 调试支持 | 有限 | 丰富 | 基本 |
| 适用场景 | 大型系统 | 通用 | 嵌入式 |
| 碎片控制 | 最好 | 好 | 一般 |

### 3.3.3 kmalloc 和通用缓存

kmalloc 是内核最常用的内存分配接口，基于预定义的通用缓存：

```c
// 通用缓存大小（2^n 字节）
struct kmem_cache *kmalloc_caches[KMALLOC_SHIFT_HIGH + 1];

// kmalloc 实现
static __always_inline void *kmalloc(size_t size, gfp_t flags) {
    if (__builtin_constant_p(size)) {
        // 编译时常量大小，优化路径
        if (size > KMALLOC_MAX_CACHE_SIZE)
            return kmalloc_large(size, flags);
            
        unsigned int index = kmalloc_index(size);
        return kmem_cache_alloc_trace(kmalloc_caches[index], flags, size);
    }
    return __kmalloc(size, flags);
}

// 大小到索引的映射
static __always_inline unsigned int kmalloc_index(size_t size) {
    if (size <= 8) return 3;       // kmalloc-8
    if (size <= 16) return 4;      // kmalloc-16
    if (size <= 32) return 5;      // kmalloc-32
    if (size <= 64) return 6;      // kmalloc-64
    if (size <= 128) return 7;     // kmalloc-128
    if (size <= 256) return 8;     // kmalloc-256
    if (size <= 512) return 9;     // kmalloc-512
    if (size <= 1024) return 10;   // kmalloc-1024
    if (size <= 2048) return 11;   // kmalloc-2048
    if (size <= 4096) return 12;   // kmalloc-4096
    if (size <= 8192) return 13;   // kmalloc-8192
    // ... 更大的尺寸
}
```

### 3.3.4 vmalloc 虚拟连续分配

vmalloc 分配虚拟地址连续但物理不连续的内存，适合大块分配：

```c
void *vmalloc(unsigned long size) {
    return __vmalloc_node(size, 1, GFP_KERNEL, PAGE_KERNEL,
                         NUMA_NO_NODE, __builtin_return_address(0));
}

static void *__vmalloc_area_node(struct vm_struct *area, gfp_t gfp_mask,
                                 pgprot_t prot, int node) {
    unsigned long size = get_vm_area_size(area);
    unsigned long addr = (unsigned long)area->addr;
    unsigned long end = addr + size;
    unsigned int nr_pages = size >> PAGE_SHIFT;
    struct page **pages;
    
    // 分配页面指针数组
    pages = kvmalloc_array(nr_pages, sizeof(struct page *), GFP_KERNEL);
    
    // 分配物理页面
    for (i = 0; i < nr_pages; i++) {
        pages[i] = alloc_page(gfp_mask | __GFP_HIGHMEM);
        if (!pages[i])
            goto fail;
    }
    
    // 建立页表映射
    if (map_vm_area(area, prot, pages))
        goto fail;
        
    return area->addr;
}
```

vmalloc vs kmalloc：

```
kmalloc:
- 物理连续
- 直接映射区
- 快速（无需修改页表）
- 适合小分配（<128KB）
- DMA 友好

vmalloc:
- 仅虚拟连续
- vmalloc 区域
- 较慢（需要修改页表）
- 适合大分配
- 可能跨 NUMA 节点
```

### 3.3.5 Per-CPU 内存分配

Per-CPU 变量避免缓存竞争，提高多核性能：

```c
// 静态 Per-CPU 变量
DEFINE_PER_CPU(int, my_counter);

// 动态 Per-CPU 分配
void __percpu *__alloc_percpu(size_t size, size_t align) {
    return pcpu_alloc(size, align, false, GFP_KERNEL);
}

// Per-CPU 内存布局
/*
 * CPU0: [---- unit 0 ----]
 * CPU1: [---- unit 1 ----]
 * CPU2: [---- unit 2 ----]
 * CPU3: [---- unit 3 ----]
 * 
 * 每个 unit 包含相同的变量副本
 */

// 访问 Per-CPU 变量
int cpu = get_cpu();  // 禁用抢占
per_cpu(my_counter, cpu)++;
put_cpu();  // 恢复抢占

// 或使用便捷宏
this_cpu_inc(my_counter);  // 原子操作，无需禁用抢占
```

## 3.4 内存回收机制

### 3.4.1 LRU 算法实现

Linux 使用近似 LRU（Least Recently Used）算法管理页面回收：

```
LRU 链表组织：

         活跃链表                    非活跃链表
    (Active LRU Lists)          (Inactive LRU Lists)
    
    +-------------+              +-------------+
    | Active Anon |              |Inactive Anon|
    | (匿名页面)   |              | (匿名页面)   |
    +-------------+              +-------------+
         |                            |
    频繁访问的                    较少访问的
    进程内存页                    进程内存页
    
    +-------------+              +-------------+
    | Active File |              |Inactive File|
    | (文件页面)   |              | (文件页面)   |
    +-------------+              +-------------+
         |                            |
    热点文件缓存                  冷文件缓存
```

LRU 链表管理结构：

```c
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];  // LRU 链表数组
    struct zone_reclaim_stat reclaim_stat;
    
    // LRU 链表类型
    enum lru_list {
        LRU_INACTIVE_ANON = LRU_BASE,      // 非活跃匿名页
        LRU_ACTIVE_ANON = LRU_BASE + LRU_ACTIVE,    // 活跃匿名页
        LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,    // 非活跃文件页
        LRU_ACTIVE_FILE = LRU_BASE + LRU_FILE + LRU_ACTIVE,  // 活跃文件页
        LRU_UNEVICTABLE,                    // 不可回收页
        NR_LRU_LISTS
    };
    
    atomic_long_t nr_zone_inactive_anon;
    atomic_long_t nr_zone_active_anon;
    atomic_long_t nr_zone_inactive_file;
    atomic_long_t nr_zone_active_file;
    atomic_long_t nr_zone_unevictable;
    
    spinlock_t lru_lock;
};

// 页面在 LRU 链表间的移动
void activate_page(struct page *page) {
    if (PageLRU(page) && !PageActive(page) && !PageUnevictable(page)) {
        struct lruvec *lruvec = mem_cgroup_page_lruvec(page);
        
        del_page_from_lru_list(page, lruvec, page_lru(page));
        SetPageActive(page);
        add_page_to_lru_list(page, lruvec, page_lru(page));
        
        __count_vm_event(PGACTIVATE);
        update_page_reclaim_stat(lruvec, page);
    }
}
```

### 3.4.2 页面回收流程

内存压力触发页面回收，kswapd 守护进程负责后台回收：

```c
// kswapd 主循环
static int kswapd(void *p) {
    pg_data_t *pgdat = (pg_data_t*)p;
    struct task_struct *tsk = current;
    
    set_freezable();
    
    for ( ; ; ) {
        bool ret;
        
        // 等待内存压力事件
        ret = try_to_freeze();
        if (kthread_should_stop())
            break;
            
        // 检查水位线
        if (prepare_kswapd_sleep(pgdat, order, classzone_idx)) {
            // 睡眠等待唤醒
            schedule();
            continue;
        }
        
        // 执行页面回收
        balance_pgdat(pgdat, order, classzone_idx);
    }
    
    return 0;
}

// 页面回收核心函数
static unsigned long shrink_node(pg_data_t *pgdat,
                                 struct scan_control *sc) {
    struct lruvec *lruvec = node_lruvec(pgdat);
    unsigned long nr_reclaimed = 0;
    
    // 扫描控制参数
    struct scan_control {
        unsigned long nr_to_reclaim;   // 目标回收页数
        unsigned long nr_scanned;      // 已扫描页数
        unsigned int priority;         // 扫描优先级（0-12）
        unsigned int may_writepage:1;  // 是否可写回脏页
        unsigned int may_unmap:1;      // 是否可解除映射
        unsigned int may_swap:1;       // 是否可换出
        unsigned int hibernation_mode:1;
    };
    
    // 1. 回收文件页缓存
    nr_reclaimed += shrink_list(LRU_INACTIVE_FILE, nr_to_scan,
                                lruvec, sc);
    
    // 2. 回收匿名页（需要 swap）
    if (sc->may_swap)
        nr_reclaimed += shrink_list(LRU_INACTIVE_ANON, nr_to_scan,
                                   lruvec, sc);
    
    // 3. 回收 slab 缓存
    nr_reclaimed += shrink_slab(sc->gfp_mask, pgdat->node_id,
                               NULL, sc->priority);
    
    return nr_reclaimed;
}
```

水位线机制：

```
内存水位线（Watermarks）：

页面数 ^
      |
      |............... high (开始后台回收)
      |
      |............... low  (唤醒 kswapd)
      |
      |............... min  (直接回收，可能 OOM)
      |___________________________
                                时间 ->

计算公式：
min = (内存大小 * 比例) / 10000
low = min + (min / 4)
high = min + (min / 2)
```

### 3.4.3 直接回收与回写

当内存分配失败时，进程直接参与页面回收：

```c
// 直接回收路径
static int __perform_reclaim(gfp_t gfp_mask, unsigned int order,
                            const struct alloc_context *ac) {
    struct reclaim_state reclaim_state;
    int progress;
    unsigned int noreclaim_flag;
    
    // 避免递归回收
    noreclaim_flag = memalloc_noreclaim_save();
    reclaim_state.reclaimed_slab = 0;
    current->reclaim_state = &reclaim_state;
    
    // 执行直接回收
    progress = try_to_free_pages(ac->zonelist, order, gfp_mask,
                                 ac->nodemask);
    
    current->reclaim_state = NULL;
    memalloc_noreclaim_restore(noreclaim_flag);
    
    return progress;
}

// 页面回写机制
static pageout_t pageout(struct page *page, struct address_space *mapping,
                         struct scan_control *sc) {
    // 检查页面是否脏
    if (!PageDirty(page))
        return PAGE_CLEAN;
        
    // 检查是否允许写回
    if (!sc->may_writepage)
        return PAGE_KEEP;
        
    // 正在回写中
    if (PageWriteback(page))
        return PAGE_KEEP;
        
    // 触发异步回写
    if (mapping->a_ops->writepage == NULL)
        return PAGE_ACTIVATE;
        
    SetPageReclaim(page);
    return mapping->a_ops->writepage(page, &wbc);
}
```

### 3.4.4 OOM Killer 机制

当内存回收无法满足分配需求时，OOM Killer 选择进程终止：

```c
// OOM 评分算法
unsigned long oom_badness(struct task_struct *p, 
                         struct mem_cgroup *memcg,
                         const nodemask_t *nodemask,
                         unsigned long totalpages) {
    long points;
    long adj;
    
    // 获取进程内存使用量
    points = get_mm_rss(p->mm) + get_mm_counter(p->mm, MM_SWAPENTS) +
            mm_pgtables_bytes(p->mm) / PAGE_SIZE;
    
    // 考虑 oom_score_adj 调整值
    adj = (long)p->signal->oom_score_adj;
    if (adj == OOM_SCORE_ADJ_MIN) {
        // -1000 表示永不杀死
        return 0;
    }
    
    // 计算最终分数（0-1000）
    adj *= totalpages / 1000;
    points += adj;
    
    // 特殊进程保护
    if (has_capability_noaudit(p, CAP_SYS_ADMIN))
        points -= (points * 3) / 100;  // root 进程降低 3%
        
    return points > 0 ? points : 1;
}

// OOM Killer 选择受害者
static struct task_struct *select_bad_process(struct oom_control *oc) {
    struct task_struct *p;
    struct task_struct *chosen = NULL;
    long chosen_points = 0;
    
    for_each_process(p) {
        unsigned long points;
        
        // 跳过不可杀死的进程
        if (oom_unkillable_task(p, NULL, oc->nodemask))
            continue;
            
        // 计算 badness 分数
        points = oom_badness(p, NULL, oc->nodemask, oc->totalpages);
        if (!points || points < chosen_points)
            continue;
            
        chosen = p;
        chosen_points = points;
    }
    
    return chosen;
}

// 执行 OOM Kill
static void oom_kill_process(struct oom_control *oc, const char *message) {
    struct task_struct *victim = oc->chosen;
    struct task_struct *p;
    
    // 打印 OOM 信息
    pr_err("%s: Kill process %d (%s) score %ld\n",
           message, task_pid_nr(victim), victim->comm,
           oom_badness(victim, NULL, oc->nodemask, oc->totalpages));
           
    // 发送 SIGKILL
    do_send_sig_info(SIGKILL, SEND_SIG_PRIV, victim, PIDTYPE_PID);
    
    // 标记为 OOM 受害者
    mark_oom_victim(victim);
    
    // 尝试回收内存
    try_oom_reaper(victim);
}
```

### 3.4.5 内存压缩与去碎片化

内存压缩通过移动页面来创建连续空闲内存：

```c
// 内存压缩算法
static unsigned long compact_zone(struct zone *zone, 
                                  struct compact_control *cc) {
    unsigned long migrate_pfn = cc->migrate_pfn;
    unsigned long free_pfn = cc->free_pfn;
    
    // 扫描可移动页面（从低地址向高地址）
    while (migrate_pfn < free_pfn) {
        // 隔离可移动页面
        nr_isolated = isolate_migratepages(zone, cc);
        
        if (!nr_isolated)
            break;
            
        // 迁移页面到空闲位置
        migrate_pages(&cc->migratepages, compaction_alloc,
                     compaction_free, (unsigned long)cc,
                     cc->mode, MR_COMPACTION);
    }
    
    return COMPACT_COMPLETE;
}

// 页面迁移类型管理
static void set_pageblock_migratetype(struct page *page, int migratetype) {
    if (unlikely(page_group_by_mobility_disabled &&
                migratetype < MIGRATE_PCPTYPES))
        migratetype = MIGRATE_UNMOVABLE;
        
    set_pageblock_flags_group(page, (unsigned long)migratetype,
                             PB_migrate, PB_migrate_end);
}
```

## 3.5 本章小结

Linux 内存管理是一个复杂而精妙的系统，通过多层次的抽象和优化实现了高效的内存利用：

### 核心概念回顾

1. **物理内存管理层次**：
   - NUMA 节点 → 内存区域（Zone）→ 页框（Page）
   - 伙伴系统解决外部碎片，支持大块连续分配
   - struct page 是物理页面管理的核心数据结构

2. **虚拟内存机制**：
   - 多级页表实现虚拟到物理地址映射
   - VMA 管理进程地址空间的不同区域
   - TLB 缓存加速地址转换
   - 缺页异常实现按需分配

3. **内存分配器**：
   - Slab/SLUB：高效的小对象分配，减少内部碎片
   - kmalloc：通用内核内存分配接口
   - vmalloc：虚拟连续的大块内存分配
   - Per-CPU 分配：避免缓存竞争

4. **内存回收机制**：
   - LRU 算法管理页面老化
   - kswapd 后台回收与直接回收
   - OOM Killer 作为最后手段
   - 内存压缩创建连续空闲空间

### 关键公式

1. **页表索引计算**：
   $$\text{PGD\_index} = \frac{\text{VA} \gg 39}{512} \bmod 512$$
   $$\text{物理地址} = \text{PFN} \times 4096 + \text{offset}$$

2. **伙伴系统块大小**：
   $$\text{块大小} = 2^{\text{order}} \times \text{PAGE\_SIZE}$$

3. **OOM 评分**：
   $$\text{score} = \frac{\text{RSS} + \text{Swap}}{\text{Total}} \times 1000 + \text{oom\_score\_adj}$$

4. **水位线计算**：
   $$\text{min} = \frac{\text{zone\_pages} \times \text{min\_free\_kbytes}}{10000}$$
   $$\text{low} = \text{min} \times 1.25, \quad \text{high} = \text{min} \times 1.5$$

## 3.6 练习题

### 基础理解题

**题目 1**：解释为什么 Linux 使用多级页表而不是单级页表？计算 48 位虚拟地址空间使用单级页表需要多少内存。

<details>
<summary>提示：考虑页表项大小和虚拟地址空间大小</summary>

计算单级页表的内存需求，并与多级页表的按需分配特性对比。
</details>

<details>
<summary>参考答案</summary>

单级页表需要为整个虚拟地址空间预分配页表项：
- 48位地址空间 = 2^48 字节 = 256TB
- 4KB页面 = 2^12 字节
- 需要页表项数 = 2^48 / 2^12 = 2^36 个
- 每个PTE 8字节，总需求 = 2^36 × 8 = 512GB

多级页表优势：
1. 按需分配：只为实际使用的虚拟地址创建页表
2. 稀疏地址空间高效：大部分进程只使用很小部分地址空间
3. 共享页表：内核页表可在进程间共享
4. 缓存友好：活跃页表项集中，提高TLB命中率
</details>

**题目 2**：伙伴系统中，如果请求分配 5 个页面，系统会如何处理？说明分配过程和内部碎片。

<details>
<summary>提示：伙伴系统只能分配 2^n 大小的块</summary>

考虑向上取整到最近的 2 的幂次，以及产生的内部碎片。
</details>

<details>
<summary>参考答案</summary>

分配过程：
1. 5页面向上取整到 2^3 = 8 页面
2. 查找 order=3 的空闲块
3. 如果没有，向更高 order 请求并分裂
4. 返回 8 页面的块给请求者

内部碎片：
- 请求：5 × 4KB = 20KB
- 实际分配：8 × 4KB = 32KB
- 内部碎片：12KB (37.5%)
- 这是伙伴系统简单高效的代价
</details>

### 代码分析题

**题目 3**：分析以下代码片段，解释它在内存管理中的作用：

```c
static inline struct page *alloc_pages(gfp_t gfp_mask, unsigned int order)
{
    return alloc_pages_current(gfp_mask, order);
}

#define alloc_page(gfp_mask) alloc_pages(gfp_mask, 0)
#define __get_free_page(gfp_mask) \
    __get_free_pages((gfp_mask), 0)
```

<details>
<summary>提示：注意 order 参数和宏定义的作用</summary>

这些是页面分配的不同接口，提供不同级别的抽象。
</details>

<details>
<summary>参考答案</summary>

代码作用：
1. `alloc_pages()`: 核心页面分配函数，分配 2^order 个连续页面
2. `alloc_page()`: 分配单个页面的便捷宏（order=0）
3. `__get_free_page()`: 分配单页并返回虚拟地址而非 struct page

设计理念：
- 提供多层次 API 满足不同需求
- alloc_pages 返回 struct page*，适合需要页面元数据的场景
- __get_free_page 返回地址，适合直接使用内存的场景
- 宏定义避免函数调用开销
</details>

**题目 4**：以下 SLUB 快速路径代码使用了什么优化技术？为什么这样设计？

```c
if (unlikely(!this_cpu_cmpxchg_double(
        s->cpu_slab->freelist, s->cpu_slab->tid,
        object, tid,
        next_object, next_tid(tid)))) {
    goto redo;
}
```

<details>
<summary>提示：注意 cmpxchg 和 Per-CPU 变量的使用</summary>

这是无锁编程技术，使用 CAS 操作实现原子更新。
</details>

<details>
<summary>参考答案</summary>

优化技术：
1. **无锁算法**：使用 CAS（Compare-And-Swap）避免锁开销
2. **Per-CPU 缓存**：每个 CPU 独立的 freelist，避免缓存竞争
3. **事务 ID (tid)**：检测并发修改，确保一致性
4. **双字 CAS**：原子更新 freelist 和 tid

设计原因：
- 内存分配是高频操作，锁会成为瓶颈
- Per-CPU 设计利用了 CPU 缓存局部性
- 失败重试比等待锁更高效（乐观并发）
- 适合分配频繁但竞争较少的场景
</details>

### 设计实现题

**题目 5**：设计一个简化的 Slab 分配器，支持固定大小对象的分配和释放。要求：
- 支持对象缓存和重用
- 实现简单的 Per-CPU 缓存
- 考虑内存对齐

<details>
<summary>提示：使用链表管理空闲对象，考虑批量分配优化</summary>

关键是维护空闲对象链表和 Per-CPU 缓存的同步。
</details>

<details>
<summary>参考答案</summary>

设计要点：

```c
struct simple_slab {
    void *freelist;          // 空闲对象链表
    unsigned int size;       // 对象大小
    unsigned int align;      // 对齐要求
    struct page *pages;      // slab 页面
    
    // Per-CPU 缓存
    struct {
        void *list;
        int count;
    } __percpu *cpu_cache;
};

// 分配逻辑：
1. 检查 Per-CPU 缓存
2. 如果为空，从全局 freelist 批量获取
3. 如果全局为空，分配新 slab
4. 返回对象给调用者

// 释放逻辑：
1. 放入 Per-CPU 缓存
2. 如果缓存满，批量返回给全局 freelist
3. 考虑 slab 合并和内存回收

// 关键优化：
- 批量传输减少锁竞争
- 对象构造函数避免重复初始化
- LIFO 顺序提高缓存热度
```
</details>

**题目 6**：实现一个简单的 LRU 页面置换算法，包括页面老化和回收选择。

<details>
<summary>提示：使用双向链表维护 LRU 顺序，考虑活跃/非活跃列表</summary>

重点是页面在列表间的移动策略和回收victim的选择。
</details>

<details>
<summary>参考答案</summary>

实现框架：

```c
struct lru_list {
    struct list_head active;
    struct list_head inactive;
    unsigned long nr_active;
    unsigned long nr_inactive;
    spinlock_t lock;
};

// 页面访问时：
1. 如果在 inactive 列表，移到 active
2. 如果在 active 列表，移到列表头部
3. 维护 active:inactive 比例（如 2:1）

// 页面老化：
1. 定期扫描 active 列表尾部
2. 检查页面访问位（PTE 的 Accessed 位）
3. 未访问的页面降级到 inactive

// 回收选择：
1. 优先从 inactive 列表尾部选择
2. 跳过脏页（需要写回）
3. 跳过正在使用的页面
4. 批量回收提高效率

// 优化考虑：
- 使用时钟算法近似 LRU
- 分离文件页和匿名页
- Per-zone LRU 减少锁竞争
```
</details>

### 开放思考题

**题目 7**：现代 NVMe SSD 的普及如何影响 Linux 内存管理设计？请讨论至少三个方面的影响。

<details>
<summary>提示：考虑延迟、带宽、寿命等特性</summary>

NVMe 的低延迟改变了内存与存储的传统边界。
</details>

<details>
<summary>参考答案</summary>

影响分析：

1. **交换策略优化**：
   - NVMe 延迟接近 DRAM（微秒级）
   - 可以更积极地使用 swap
   - zswap/zram 压缩交换更有价值
   - 考虑分层内存（内存+NVMe+HDD）

2. **页面回收算法**：
   - 主要/次要缺页的代价差距缩小
   - 可以容忍更高的缺页率
   - 预读算法需要重新调整
   - 考虑 SSD 寿命的写入均衡

3. **持久内存支持**：
   - DAX（直接访问）绕过页缓存
   - PMEM 作为内存或存储的双重角色
   - 新的编程模型（如 Intel Optane）
   - 崩溃一致性的新挑战

4. **I/O 路径优化**：
   - 减少内核态/用户态切换（io_uring）
   - 巨页支持减少 TLB 压力
   - NUMA 感知的设备分配
</details>

**题目 8**：设计一个支持内存 QoS（Quality of Service）的内存管理机制，确保关键应用的内存保障。

<details>
<summary>提示：考虑内存预留、优先级和隔离机制</summary>

参考 cgroups 的内存控制器设计。
</details>

<details>
<summary>参考答案</summary>

QoS 机制设计：

1. **内存预留机制**：
   - 最小保障（memory.min）：保护阈值
   - 软限制（memory.low）：尽力保护
   - 硬限制（memory.max）：绝对上限
   - 分层继承：父子 cgroup 约束

2. **优先级调度**：
   - 内存分配优先级队列
   - 高优先级可抢占低优先级内存
   - OOM 时按优先级选择受害者
   - 回收压力按优先级分配

3. **性能隔离**：
   - Per-cgroup LRU 链表
   - 独立的回收扫描控制
   - CPU 和内存带宽联合控制
   - NUMA 节点绑定

4. **监控反馈**：
   - 实时内存压力指标
   - 分配延迟统计
   - 缺页率监控
   - 自适应调整策略

实现考虑：
- 避免优先级反转
- 防止饿死低优先级
- 开销与精度权衡
- 与调度器集成
```
</details>

## 3.7 常见陷阱与错误

### 内存泄漏模式

1. **忘记释放内存**：
```c
// 错误：忘记 kfree
void *buf = kmalloc(size, GFP_KERNEL);
if (!buf)
    return -ENOMEM;
// ... 使用 buf
return 0;  // 泄漏！

// 正确：使用 goto 清理
void *buf = kmalloc(size, GFP_KERNEL);
if (!buf)
    return -ENOMEM;
// ... 使用 buf
ret = do_something();
if (ret)
    goto out_free;
// ...
out_free:
    kfree(buf);
    return ret;
```

2. **重复释放**：
```c
// 错误：double free
kfree(ptr);
// ... 其他代码
kfree(ptr);  // 崩溃！

// 正确：释放后置空
kfree(ptr);
ptr = NULL;
```

3. **释放栈内存**：
```c
// 错误：释放栈变量
char buf[100];
kfree(buf);  // 崩溃！

// 正确：只释放动态分配的内存
char *buf = kmalloc(100, GFP_KERNEL);
kfree(buf);
```

### 分配标志误用

1. **原子上下文使用 GFP_KERNEL**：
```c
// 错误：中断处理中睡眠
irqreturn_t my_isr(int irq, void *dev) {
    void *buf = kmalloc(size, GFP_KERNEL);  // 可能睡眠！
}

// 正确：使用 GFP_ATOMIC
irqreturn_t my_isr(int irq, void *dev) {
    void *buf = kmalloc(size, GFP_ATOMIC);
}
```

2. **内存回收死锁**：
```c
// 错误：文件系统回收路径再次分配
int writepage(struct page *page) {
    void *buf = kmalloc(size, GFP_KERNEL);  // 可能死锁！
}

// 正确：使用 GFP_NOFS
int writepage(struct page *page) {
    void *buf = kmalloc(size, GFP_NOFS);
}
```

### 并发访问错误

1. **无保护的页表操作**：
```c
// 错误：无锁访问页表
pte_t *pte = pte_offset_map(pmd, addr);
*pte = new_pte;  // 竞态条件！

// 正确：持有页表锁
spin_lock(&mm->page_table_lock);
pte_t *pte = pte_offset_map(pmd, addr);
*pte = new_pte;
spin_unlock(&mm->page_table_lock);
```

2. **Per-CPU 变量误用**：
```c
// 错误：未禁用抢占
int *counter = &per_cpu(my_counter, smp_processor_id());
(*counter)++;  // 可能被抢占到其他 CPU！

// 正确：禁用抢占
preempt_disable();
this_cpu_inc(my_counter);
preempt_enable();
```

## 3.8 最佳实践检查清单

### 内存分配决策树

- [ ] **确定分配大小**
  - 小于 PAGE_SIZE → kmalloc
  - 大于 128KB → vmalloc
  - 需要物理连续 → alloc_pages
  - 特定对象类型 → kmem_cache_alloc

- [ ] **选择正确的 GFP 标志**
  - 进程上下文 → GFP_KERNEL
  - 原子上下文 → GFP_ATOMIC
  - 文件系统 → GFP_NOFS
  - 不触发 I/O → GFP_NOIO
  - 用户空间 → GFP_USER

- [ ] **考虑 NUMA 特性**
  - 使用本地节点分配
  - 设置合适的 NUMA 策略
  - 监控跨节点访问

- [ ] **实现错误处理**
  - 检查分配失败
  - 提供降级方案
  - 正确的清理路径

### 性能优化要点

- [ ] **减少分配频率**
  - 对象池和缓存
  - 批量分配
  - 预分配策略

- [ ] **优化缓存使用**
  - 数据结构对齐
  - 避免伪共享
  - Per-CPU 变量

- [ ] **控制内存碎片**
  - 使用合适的分配器
  - 定期内存压缩
  - 监控碎片指标

- [ ] **内存回收优化**
  - 合理设置水位线
  - 优化 swappiness
  - 使用 madvise 提示

### 调试与监控

- [ ] **使用调试工具**
  - KASAN（地址消毒）
  - kmemleak（泄漏检测）
  - slub_debug（分配器调试）
  - ftrace（跟踪分配）

- [ ] **监控关键指标**
  - /proc/meminfo
  - /proc/buddyinfo
  - /proc/slabinfo
  - /proc/pagetypeinfo

- [ ] **性能分析**
  - perf 内存事件
  - 缺页率统计
  - 内存带宽使用
  - NUMA 平衡状态
