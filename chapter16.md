# 第16章：引导过程与初始化

## 本章导读

系统启动是 Linux 内核最神秘也最关键的阶段。从按下电源按钮到用户登录界面出现，期间经历了复杂的硬件初始化、内核加载、子系统启动等过程。本章将深入剖析从 BIOS/UEFI 固件到 systemd 用户空间的完整启动链路，重点解析内核如何从一个压缩的二进制文件变成运行中的操作系统核心。通过学习本章，您将掌握内核启动的每个关键节点、理解 initcall 机制的设计哲学，并能够优化系统启动性能。

## 16.1 BIOS/UEFI 引导阶段

### 16.1.1 传统 BIOS 引导流程

当 x86 系统上电时，CPU 被硬件强制设置为实模式，并从物理地址 0xFFFFFFF0（Reset Vector）开始执行。这个地址映射到 BIOS ROM，包含一条跳转指令，跳转到 BIOS 初始化代码。

```
启动顺序：
1. Power On → CPU Reset → CS:IP = 0xF000:0xFFF0
2. BIOS POST (Power-On Self-Test)
3. 内存检测与初始化
4. 搜索启动设备（按 Boot Order）
5. 加载 MBR (Master Boot Record) 到 0x7C00
6. 跳转到 0x7C00 执行引导代码
```

MBR 结构（512 字节）：
```
偏移    大小     描述
0x000   446B    引导代码（bootloader stage 1）
0x1BE   64B     分区表（4个16字节分区项）
0x1FE   2B      魔数 0xAA55（小端序）
```

### 16.1.2 UEFI 现代引导

UEFI（Unified Extensible Firmware Interface）提供了更强大的引导机制：

```
UEFI 启动流程：
SEC (Security) → PEI (Pre-EFI) → DXE (Driver Execution) → BDS (Boot Device Select) → OS Loader
     ↓              ↓               ↓                        ↓                      ↓
  验证固件      初始化内存      加载驱动和协议           选择启动项            加载内核
```

UEFI 的关键优势：
- **GPT 分区支持**：突破 MBR 的 2TB 限制
- **安全启动**：通过签名验证防止 rootkit
- **网络启动**：原生支持 PXE 和 HTTP 启动
- **运行时服务**：为 OS 提供持续的固件服务

ESP（EFI System Partition）文件布局：
```
/EFI/
├── BOOT/
│   └── BOOTX64.EFI    # 默认引导器
├── ubuntu/
│   ├── grubx64.efi     # GRUB UEFI 引导器
│   ├── shimx64.efi     # Secure Boot shim
│   └── grub.cfg        # GRUB 配置
└── Microsoft/
    └── Boot/           # Windows 引导器
```

### 16.1.3 Bootloader 架构

现代 Linux 系统主要使用 GRUB2（GRand Unified Bootloader）：

GRUB2 多阶段加载：
```
Stage 1 (boot.img):
  - 大小：512 字节（刚好一个扇区）
  - 位置：MBR 或 GPT 的 BIOS Boot Partition
  - 功能：加载 Stage 1.5

Stage 1.5 (core.img):
  - 大小：约 32KB
  - 位置：MBR 后的空闲扇区或专用分区
  - 功能：包含文件系统驱动，能够读取 /boot

Stage 2 (grub modules):
  - 位置：/boot/grub/
  - 功能：提供完整的引导环境
```

GRUB2 传递给内核的关键参数：
```c
struct boot_params {
    struct screen_info screen_info;    // 显示模式信息
    struct apm_bios_info apm_bios_info; // APM BIOS 信息
    __u32 cmd_line_ptr;                // 内核命令行指针
    __u32 initrd_addr_max;             // initrd 最大地址
    __u32 kernel_alignment;            // 内核对齐要求
    // ... E820 内存映射表
    struct e820_entry e820_table[E820_MAX_ENTRIES];
};
```

## 16.2 内核加载与解压

### 16.2.1 bzImage 格式解析

Linux 内核编译后生成的 bzImage（big zImage）是一个自解压的可执行文件：

```
bzImage 结构：
┌─────────────────┐ 0x0000
│   Setup Header  │ 实模式设置代码（arch/x86/boot/header.S）
├─────────────────┤ 0x1F1 (497)
│   Boot Protocol │ 引导协议版本和标志
├─────────────────┤ 0x200 (512)
│   Setup Code    │ 16位/32位混合代码
├─────────────────┤ setup_sects * 512
│  Protected Mode │ 32/64位压缩内核
│  Kernel (vmlinux.gz)│
└─────────────────┘
```

关键的 setup_header 字段：
```c
struct setup_header {
    __u8  setup_sects;      // setup 代码的扇区数
    __u16 root_flags;       // 根文件系统标志
    __u32 syssize;          // 压缩内核大小（16字节为单位）
    __u16 ram_size;         // 废弃
    __u16 vid_mode;         // 视频模式
    __u16 root_dev;         // 根设备号
    __u16 boot_flag;        // 0xAA55 魔数
    __u16 jump;             // 跳转指令
    __u32 header;           // "HdrS" 魔数
    __u16 version;          // 引导协议版本
    __u32 realmode_swtch;   // 切换到实模式的钩子
    __u16 start_sys_seg;    // 废弃
    __u16 kernel_version;   // 内核版本字符串指针
    __u8  type_of_loader;   // 引导器类型
    __u8  loadflags;        // 引导标志
    __u16 setup_move_size;  // 移动到 setup 代码大小
    __u32 code32_start;     // 32位代码起始地址
    __u32 ramdisk_image;    // initrd 加载地址
    __u32 ramdisk_size;     // initrd 大小
    __u32 bootsect_kludge;  // 废弃
    __u16 heap_end_ptr;     // setup heap 结束位置
    __u8  ext_loader_ver;   // 扩展引导器版本
    __u8  ext_loader_type;  // 扩展引导器类型
    __u32 cmd_line_ptr;     // 内核命令行地址
    __u32 initrd_addr_max;  // initrd 最高地址
    __u32 kernel_alignment; // 内核对齐要求
    __u8  relocatable_kernel; // 内核是否可重定位
    __u8  min_alignment;    // 最小对齐（2^n）
    __u16 xloadflags;       // 扩展引导标志
    __u32 cmdline_size;     // 命令行最大长度
    __u32 hardware_subarch; // 硬件子架构
    __u64 hardware_subarch_data; // 子架构数据指针
    __u32 payload_offset;   // 压缩载荷偏移
    __u32 payload_length;   // 压缩载荷长度
    __u64 setup_data;       // setup_data 链表
    __u64 pref_address;     // 首选加载地址
    __u32 init_size;        // 初始化所需内存大小
    __u32 handover_offset;  // EFI handover 协议入口
};
```

### 16.2.2 解压过程详解

内核解压发生在保护模式下，主要代码在 `arch/x86/boot/compressed/` 目录：

```c
// arch/x86/boot/compressed/head_64.S
startup_64:
    // 1. 清除 BSS 段
    xorl %eax, %eax
    leaq _bss(%rip), %rdi
    leaq _ebss(%rip), %rcx
    subq %rdi, %rcx
    shrq $3, %rcx
    rep stosq

    // 2. 建立身份映射页表
    call paging_prepare

    // 3. 调用 C 语言解压函数
    pushq %rsi          // boot_params 指针
    movq %rsi, %rdi     // 第一个参数
    leaq boot_heap(%rip), %rsi  // 堆空间
    call extract_kernel

    // 4. 跳转到解压后的内核
    popq %rsi
    jmp *%rax
```

解压算法支持多种格式：
- **GZIP**：默认选择，压缩率和速度平衡
- **BZIP2**：更好的压缩率，解压较慢
- **LZMA/XZ**：最佳压缩率，内存需求大
- **LZO/LZ4**：快速解压，适合嵌入式系统

### 16.2.3 早期内存布局

解压后的内存布局（x86_64）：
```
物理内存布局：
0x00000000 - 0x000FFFFF  实模式内存（1MB）
  0x00000000 - 0x000003FF  中断向量表（IVT）
  0x00000400 - 0x000004FF  BIOS 数据区（BDA）
  0x00000500 - 0x00007BFF  可用内存
  0x00007C00 - 0x00007DFF  MBR 加载区
  0x00080000 - 0x0009FFFF  扩展 BIOS 数据区（EBDA）
  0x000A0000 - 0x000BFFFF  视频内存
  0x000C0000 - 0x000FFFFF  BIOS ROM

0x00100000+              高端内存
  0x00100000              _text（内核代码段开始）
  ...                     内核代码
  _etext                  内核代码段结束
  ...                     内核数据
  _edata                  初始化数据结束
  __bss_start             BSS 段开始
  _end                    内核镜像结束
```

## 16.3 早期初始化

### 16.3.1 CPU 模式切换

x86 处理器的模式演进路径：

```
实模式(16位) → 保护模式(32位) → 长模式(64位)
    ↓              ↓                ↓
  20位地址      4GB地址空间      48位虚拟地址
  1MB内存       分段+分页        仅分页
  无保护        特权级保护       NX位支持
```

实模式到保护模式切换（arch/x86/boot/pm.c）：
```c
// 设置 GDT（全局描述符表）
static void setup_gdt(void)
{
    static const uint64_t boot_gdt[] __attribute__((aligned(16))) = {
        [GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),  // 代码段
        [GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),  // 数据段
        [GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),  // TSS
    };
    
    struct gdt_ptr gdt;
    gdt.len = sizeof(boot_gdt) - 1;
    gdt.ptr = (uint32_t)&boot_gdt;
    asm volatile("lgdtl %0" : : "m" (gdt));
}

// 切换到保护模式
protected_mode_jump:
    movl %cr0, %eax
    orb $X86_CR0_PE, %al    // 设置 PE (Protection Enable) 位
    movl %eax, %cr0
    ljmp $__BOOT_CS, $in_pm32  // 长跳转刷新 CS
```

32位到64位长模式切换：
```asm
// arch/x86/boot/compressed/head_64.S
startup_32:
    // 1. 检查 CPU 是否支持长模式
    movl $0x80000001, %eax
    cpuid
    testl $(1<<29), %edx    // 检查 LM (Long Mode) 位
    jz no_longmode

    // 2. 设置 PAE（物理地址扩展）
    movl %cr4, %eax
    orl $X86_CR4_PAE, %eax
    movl %eax, %cr4

    // 3. 加载 PML4 页表
    movl $early_level4_pgt, %eax
    movl %eax, %cr3

    // 4. 启用 EFER.LME（长模式使能）
    movl $MSR_EFER, %ecx
    rdmsr
    orl $EFER_LME, %eax
    wrmsr

    // 5. 启用分页和保护模式
    movl %cr0, %eax
    orl $(X86_CR0_PG | X86_CR0_PE), %eax
    movl %eax, %cr0

    // 6. 长跳转进入64位模式
    ljmp $__KERNEL_CS, $startup_64
```

### 16.3.2 早期页表建立

Linux 使用 4 级页表（x86_64）：
```
虚拟地址分解（48位）：
┌────────┬────────┬────────┬────────┬────────┬──────────┐
│ 符号扩展│  PML4  │  PDPT  │   PD   │   PT   │  Offset  │
│ 16 bits│ 9 bits │ 9 bits │ 9 bits │ 9 bits │ 12 bits  │
└────────┴────────┴────────┴────────┴────────┴──────────┘
          ↓         ↓         ↓         ↓         ↓
        PML4E     PDPTE      PDE       PTE    物理页内偏移
```

早期页表初始化（arch/x86/kernel/head_64.S）：
```c
// 早期页表数据结构
__INITDATA
NEXT_PAGE(early_level4_pgt)
    .quad   level3_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
    .fill   511,8,0
    
NEXT_PAGE(level3_ident_pgt)
    .quad   level2_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
    .fill   511,8,0

NEXT_PAGE(level2_ident_pgt)
    // 映射前 1GB 物理内存（2MB 大页）
    PMDS(0, __PAGE_KERNEL_IDENT_LARGE_EXEC, PTRS_PER_PMD)
```

动态页表建立流程：
```c
void __init early_alloc_pgt_buf(void)
{
    unsigned long tables = INIT_PGT_BUF_SIZE;
    phys_addr_t base;

    base = memblock_find_in_range(0, 1ULL<<32, tables, PAGE_SIZE);
    if (!base)
        panic("Cannot find space for the kernel page tables");

    memblock_reserve(base, tables);
    pgt_buf_start = base;
    pgt_buf_end = base + tables;
    pgt_buf_top = base;
}
```

### 16.3.3 中断和异常初始化

IDT（中断描述符表）设置：
```c
// arch/x86/kernel/idt.c
gate_desc idt_table[IDT_ENTRIES] __page_aligned_bss;

static const __initconst struct idt_data def_idts[] = {
    INTG(X86_TRAP_DE,       divide_error),        // 除零异常
    INTG(X86_TRAP_DB,       debug),               // 调试异常
    INTG(X86_TRAP_NMI,      nmi),                 // 不可屏蔽中断
    INTG(X86_TRAP_BP,       int3),                // 断点
    INTG(X86_TRAP_OF,       overflow),            // 溢出
    INTG(X86_TRAP_BR,       bounds),              // 边界检查
    INTG(X86_TRAP_UD,       invalid_op),          // 无效操作码
    INTG(X86_TRAP_NM,       device_not_available), // 设备不可用
    INTG(X86_TRAP_DF,       double_fault),        // 双重故障
    INTG(X86_TRAP_OLD_MF,   coprocessor_segment_overrun),
    INTG(X86_TRAP_TS,       invalid_TSS),         // 无效 TSS
    INTG(X86_TRAP_NP,       segment_not_present), // 段不存在
    INTG(X86_TRAP_SS,       stack_segment),       // 栈段错误
    INTG(X86_TRAP_GP,       general_protection),  // 通用保护
    INTG(X86_TRAP_PF,       page_fault),          // 页面错误
    INTG(X86_TRAP_MF,       coprocessor_error),   // 协处理器错误
    INTG(X86_TRAP_AC,       alignment_check),     // 对齐检查
    INTG(X86_TRAP_MC,       machine_check),       // 机器检查
    INTG(X86_TRAP_XF,       simd_coprocessor_error), // SIMD 异常
};

void __init idt_setup_early_traps(void)
{
    idt_setup_from_table(idt_table, early_idts, 
                        ARRAY_SIZE(early_idts), true);
    load_idt(&idt_descr);
}
```

## 16.4 start_kernel 流程分析

### 16.4.1 start_kernel 总体架构

`start_kernel()` 是内核 C 代码的入口点，负责初始化所有核心子系统：

```c
// init/main.c
asmlinkage __visible void __init start_kernel(void)
{
    char *command_line;
    char *after_dashes;

    set_task_stack_end_magic(&init_task);  // 设置栈魔数
    smp_setup_processor_id();               // 设置 CPU ID
    debug_objects_early_init();             // 调试对象初始化

    cgroup_init_early();                    // cgroup 早期初始化

    local_irq_disable();                    // 禁用本地中断
    early_boot_irqs_disabled = true;

    // 架构相关的早期初始化
    boot_cpu_init();
    page_address_init();
    pr_notice("%s", linux_banner);          // 打印内核版本
    
    setup_arch(&command_line);              // 架构相关设置
    setup_command_line(command_line);       // 处理命令行参数
    setup_nr_cpu_ids();                     // 设置 CPU 数量
    setup_per_cpu_areas();                  // 设置 per-CPU 区域

    smp_prepare_boot_cpu();                 // 准备启动 CPU
    boot_cpu_hotplug_init();

    build_all_zonelists(NULL);              // 构建内存区域链表
    page_alloc_init();                      // 页分配器初始化

    pr_notice("Kernel command line: %s\n", saved_command_line);
    
    parse_early_param();                    // 解析早期参数
    after_dashes = parse_args("Booting kernel",
                             static_command_line, __start___param,
                             __stop___param - __start___param,
                             -1, -1, NULL, &unknown_bootoption);
    
    jump_label_init();                      // 静态键初始化
    
    setup_log_buf(0);                       // 设置日志缓冲区
    vfs_caches_init_early();                // VFS 缓存早期初始化
    sort_main_extable();                    // 排序异常表
    trap_init();                            // 陷阱门初始化
    mm_init();                              // 内存管理初始化

    ftrace_init();                          // 函数跟踪初始化

    sched_init();                           // 调度器初始化
    
    preempt_disable();                      // 禁用抢占
    if (WARN(!irqs_disabled(),
         "Interrupts were enabled early\n"))
        local_irq_disable();
    early_boot_irqs_disabled = false;
    local_irq_enable();                     // 启用中断

    kmem_cache_init_late();                 // slab 后期初始化

    console_init();                         // 控制台初始化
    if (panic_later)
        panic("Too many boot %s vars at `%s'", panic_later,
              panic_param);

    lockdep_init();                         // 锁依赖初始化

    locking_selftest();                     // 锁自测

    mem_encrypt_init();                     // 内存加密初始化

    #ifdef CONFIG_BLK_DEV_INITRD
    if (initrd_start && !initrd_below_start_ok &&
        page_to_pfn(virt_to_page((void *)initrd_start)) < min_low_pfn) {
        pr_crit("initrd overwritten (0x%08lx < 0x%08lx) - disabling it.\n",
            page_to_pfn(virt_to_page((void *)initrd_start)),
            min_low_pfn);
        initrd_start = 0;
    }
    #endif

    setup_per_cpu_pageset();                // 设置 per-CPU 页集
    numa_policy_init();                     // NUMA 策略初始化
    acpi_early_init();                      // ACPI 早期初始化
    if (late_time_init)
        late_time_init();
    sched_clock_init();                     // 调度时钟初始化
    calibrate_delay();                      // 校准延迟循环
    pid_idr_init();                         // PID IDR 初始化
    anon_vma_init();                        // 匿名 VMA 初始化
    #ifdef CONFIG_X86
    if (efi_enabled(EFI_RUNTIME_SERVICES))
        efi_enter_virtual_mode();           // EFI 虚拟模式
    #endif
    thread_stack_cache_init();              // 线程栈缓存初始化
    cred_init();                            // 凭证初始化
    fork_init();                            // fork 初始化
    proc_caches_init();                     // 进程缓存初始化
    uts_ns_init();                          // UTS 命名空间初始化
    buffer_init();                          // 缓冲区初始化
    key_init();                             // 密钥管理初始化
    security_init();                        // 安全模块初始化
    dbg_late_init();                        // 调试后期初始化
    vfs_caches_init();                      // VFS 缓存初始化
    pagecache_init();                       // 页缓存初始化
    signals_init();                         // 信号初始化
    seq_file_init();                        // seq_file 初始化
    proc_root_init();                       // proc 根目录初始化
    nsfs_init();                            // 命名空间文件系统初始化
    cpuset_init();                          // cpuset 初始化
    cgroup_init();                          // cgroup 初始化
    taskstats_init_early();                 // 任务统计初始化
    delayacct_init();                       // 延迟统计初始化

    poking_init();                          // 代码修改初始化
    check_bugs();                           // 检查 CPU 缺陷

    acpi_subsystem_init();                  // ACPI 子系统初始化
    arch_post_acpi_subsys_init();           // 架构 ACPI 后初始化
    sfi_init_late();                        // SFI 后期初始化

    /* 完成调度器初始化 */
    scheduler_enable();

    /* 执行所有 initcall */
    rest_init();
}
```

### 16.4.2 关键子系统初始化顺序

初始化顺序的依赖关系：

```
内存管理基础设施
    ↓
中断和异常处理
    ↓
调度器核心
    ↓
时钟和定时器
    ↓
进程管理
    ↓
文件系统
    ↓
设备驱动
```

关键初始化函数详解：

**1. mm_init() - 内存管理初始化**
```c
static void __init mm_init(void)
{
    page_ext_init_flatmem();    // 页扩展初始化
    mem_init();                  // 架构相关内存初始化
    kmem_cache_init();          // slab 分配器初始化
    pgtable_init();             // 页表缓存初始化
    vmalloc_init();             // vmalloc 区域初始化
    ioremap_huge_init();        // 大页 ioremap 初始化
    kaslr_late_init();          // KASLR 后期初始化
}
```

**2. sched_init() - 调度器初始化**
```c
void __init sched_init(void)
{
    int i, j;
    unsigned long alloc_size = 0, ptr;

    // 初始化等待队列
    wait_bit_init();

    // 分配调度域拓扑
    alloc_size += 2 * nr_cpu_ids * sizeof(void **);
    
    // 分配根域
    alloc_size += 2 * nr_cpu_ids * sizeof(void **);
    
    if (alloc_size) {
        ptr = (unsigned long)kzalloc(alloc_size, GFP_NOWAIT);
        // 设置各种指针...
    }

    init_rt_bandwidth(&def_rt_bandwidth, global_rt_period(), global_rt_runtime());

    // 初始化每个 CPU 的运行队列
    for_each_possible_cpu(i) {
        struct rq *rq;

        rq = cpu_rq(i);
        raw_spin_lock_init(&rq->lock);
        rq->nr_running = 0;
        rq->calc_load_active = 0;
        rq->calc_load_update = jiffies + LOAD_FREQ;
        init_cfs_rq(&rq->cfs);          // CFS 运行队列
        init_rt_rq(&rq->rt);            // RT 运行队列
        init_dl_rq(&rq->dl);            // Deadline 运行队列
        
        INIT_LIST_HEAD(&rq->leaf_cfs_rq_list);
        INIT_LIST_HEAD(&rq->leaf_rt_rq_list);
        
        rq->cpu = i;
        rq->online = 0;
        rq->idle_stamp = 0;
        rq->avg_idle = 2*sysctl_sched_migration_cost;
        rq->max_idle_balance_cost = sysctl_sched_migration_cost;

        INIT_LIST_HEAD(&rq->cfs_tasks);

        rq_attach_root(rq, &def_root_domain);
        
        #ifdef CONFIG_NO_HZ_COMMON
        rq->nohz_flags = 0;
        #endif
        
        #ifdef CONFIG_NO_HZ_FULL
        rq->last_sched_tick = 0;
        #endif
        
        atomic_set(&rq->nr_iowait, 0);
    }

    set_load_weight(&init_task, false);    // 设置 init 任务权重

    // 初始化 CPU 优先级映射
    init_cpu_present(cpu_possible_mask);
    init_cpu_online(cpu_possible_mask);
    
    scheduler_running = 1;                  // 标记调度器运行
}
```

### 16.4.3 rest_init 与内核线程创建

`rest_init()` 创建第一批内核线程并切换到用户空间：

```c
static noinline void __ref rest_init(void)
{
    struct task_struct *tsk;
    int pid;

    rcu_scheduler_starting();       // RCU 调度器启动
    
    /*
     * 创建 kernel_init 线程（PID 1）
     * 这将成为所有用户进程的祖先
     */
    pid = kernel_thread(kernel_init, NULL, CLONE_FS);
    if (pid < 0) {
        panic("Failed to create kernel_init thread");
    }
    
    /*
     * 创建 kthreadd 线程（PID 2）
     * 负责创建其他内核线程
     */
    pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
    kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
    
    complete(&kthreadd_done);       // 通知 kthreadd 创建完成

    /*
     * 当前任务（PID 0）将成为 idle 进程
     */
    init_idle_bootup_task(current);
    schedule_preempt_disabled();    // 开始调度
    
    /* 调用 cpu_idle 循环 */
    cpu_startup_entry(CPUHP_ONLINE);
}
```

## 16.5 initcall 机制

### 16.5.1 initcall 级别体系

Linux 使用 initcall 机制来管理驱动和模块的初始化顺序。initcall 分为多个级别，确保依赖关系正确：

```c
// include/linux/init.h
typedef int (*initcall_t)(void);

/* initcall 级别定义 */
#define pure_initcall(fn)       __define_initcall(fn, 0)  // 纯初始化
#define core_initcall(fn)       __define_initcall(fn, 1)  // 核心初始化
#define core_initcall_sync(fn)  __define_initcall(fn, 1s) // 核心同步
#define postcore_initcall(fn)   __define_initcall(fn, 2)  // 核心后
#define postcore_initcall_sync(fn) __define_initcall(fn, 2s)
#define arch_initcall(fn)       __define_initcall(fn, 3)  // 架构相关
#define arch_initcall_sync(fn)  __define_initcall(fn, 3s)
#define subsys_initcall(fn)     __define_initcall(fn, 4)  // 子系统
#define subsys_initcall_sync(fn) __define_initcall(fn, 4s)
#define fs_initcall(fn)         __define_initcall(fn, 5)  // 文件系统
#define fs_initcall_sync(fn)    __define_initcall(fn, 5s)
#define device_initcall(fn)     __define_initcall(fn, 6)  // 设备驱动
#define device_initcall_sync(fn) __define_initcall(fn, 6s)
#define late_initcall(fn)       __define_initcall(fn, 7)  // 后期初始化
#define late_initcall_sync(fn)  __define_initcall(fn, 7s)

/* module_init 实际上是 device_initcall */
#define module_init(x)  __initcall(x);
```

initcall 执行顺序：
```
    0. pure_initcall        最早执行，基础设施
    1. core_initcall        核心组件
    2. postcore_initcall    核心组件之后
    3. arch_initcall        架构特定代码
    4. subsys_initcall      子系统（总线、电源管理等）
    5. fs_initcall          文件系统
    6. device_initcall      设备驱动（module_init）
    7. late_initcall        最后的初始化
```

### 16.5.2 initcall 实现机制

initcall 通过特殊的链接器脚本段实现：

```c
// include/asm-generic/vmlinux.lds.h
#define INIT_CALLS_LEVEL(level)                     \
    __initcall##level##_start = .;                  \
    KEEP(*(.initcall##level##.init))               \
    KEEP(*(.initcall##level##s.init))              \

#define INIT_CALLS                                   \
    __initcall_start = .;                          \
    KEEP(*(.initcallearly.init))                   \
    INIT_CALLS_LEVEL(0)                            \
    INIT_CALLS_LEVEL(1)                            \
    INIT_CALLS_LEVEL(2)                            \
    INIT_CALLS_LEVEL(3)                            \
    INIT_CALLS_LEVEL(4)                            \
    INIT_CALLS_LEVEL(5)                            \
    INIT_CALLS_LEVEL(rootfs)                       \
    INIT_CALLS_LEVEL(6)                            \
    INIT_CALLS_LEVEL(7)                            \
    __initcall_end = .;
```

宏展开示例：
```c
// 驱动代码
static int __init my_driver_init(void)
{
    return platform_driver_register(&my_driver);
}
module_init(my_driver_init);

// 展开后
static initcall_t __initcall_my_driver_init6 
    __used __attribute__((__section__(".initcall6.init"))) 
    = my_driver_init;
```

### 16.5.3 do_initcalls 执行流程

```c
// init/main.c
static void __init do_initcalls(void)
{
    int level;

    for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
        do_initcall_level(level);
}

static void __init do_initcall_level(int level)
{
    initcall_entry_t *fn;

    strcpy(initcall_command_line, saved_command_line);
    parse_args(initcall_level_names[level],
               initcall_command_line, __start___param,
               __stop___param - __start___param,
               level, level,
               NULL, ignore_unknown_bootoption);

    trace_initcall_level(initcall_level_names[level]);
    for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
        do_one_initcall(initcall_from_entry(fn));
}

int __init_or_module do_one_initcall(initcall_t fn)
{
    int count = preempt_count();
    char msgbuf[64];
    int ret;

    if (initcall_blacklisted(fn))
        return -EPERM;

    trace_initcall_start(fn);
    ret = fn();
    trace_initcall_finish(fn, ret);

    msgbuf[0] = 0;

    if (preempt_count() != count) {
        sprintf(msgbuf, "preemption imbalance ");
        preempt_count_set(count);
    }
    if (irqs_disabled()) {
        strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
        local_irq_enable();
    }
    WARN(msgbuf[0], "initcall %pS returned with %s\n", fn, msgbuf);

    return ret;
}
```

### 16.5.4 依赖管理与调试

initcall 调试参数：
```bash
# 内核命令行参数
initcall_debug=1        # 打印每个 initcall 的执行时间
initcall_blacklist=foo_init,bar_init  # 黑名单特定 initcall
```

依赖声明示例：
```c
// 使用 module_param 声明依赖
static int depends_on_foo = 0;
module_param(depends_on_foo, int, 0);

static int __init my_init(void)
{
    if (!depends_on_foo) {
        pr_info("Waiting for foo module\n");
        return -EPROBE_DEFER;  // 延迟探测
    }
    // 正常初始化
    return 0;
}
```

## 16.6 systemd 与用户空间启动

### 16.6.1 kernel_init 进程

kernel_init (PID 1) 是所有用户空间进程的祖先：

```c
static int __ref kernel_init(void *unused)
{
    int ret;

    kernel_init_freeable();  // 执行可释放的初始化

    /* 需要在释放内存前完成所有异步调用 */
    async_synchronize_full();
    
    kprobe_free_init_mem();
    ftrace_free_init_mem();
    free_initmem();          // 释放 __init 段内存
    
    mark_readonly();
    
    /*
     * Kernel mappings are now finalized - update the userspace
     * page-table to finalize PTI.
     */
    pti_finalize();

    system_state = SYSTEM_RUNNING;
    numa_default_policy();

    rcu_end_inkernel_boot();

    do_sysctl_args();        // 处理 sysctl 参数

    if (ramdisk_execute_command) {
        ret = run_init_process(ramdisk_execute_command);
        if (!ret)
            return 0;
        pr_err("Failed to execute %s (error %d)\n",
               ramdisk_execute_command, ret);
    }

    /*
     * 尝试运行 init 进程
     * 按顺序尝试不同的路径
     */
    if (execute_command) {
        ret = run_init_process(execute_command);
        if (!ret)
            return 0;
        panic("Requested init %s failed (error %d).",
              execute_command, ret);
    }

    if (!try_to_run_init_process("/sbin/init") ||
        !try_to_run_init_process("/etc/init") ||
        !try_to_run_init_process("/bin/init") ||
        !try_to_run_init_process("/bin/sh"))
        return 0;

    panic("No working init found. Try passing init= option to kernel.");
}
```

### 16.6.2 systemd 架构

systemd 作为现代 Linux 的 init 系统，采用并行化启动：

```
systemd 核心概念：
┌──────────────┐
│    Unit      │  基本管理单元
├──────────────┤
│   Service    │  守护进程
│   Socket     │  套接字激活
│   Target     │  同步点
│   Mount      │  挂载点
│   Timer      │  定时器
│   Path       │  路径监控
│   Slice      │  资源控制组
│   Scope      │  外部进程组
│   Device     │  设备单元
│   Swap       │  交换分区
└──────────────┘
```

systemd 启动顺序：
```
1. default.target (通常链接到 multi-user.target 或 graphical.target)
   ↓
2. basic.target (基础系统)
   ├── sysinit.target (系统初始化)
   │   ├── local-fs.target (本地文件系统)
   │   ├── swap.target (交换分区)
   │   └── cryptsetup.target (加密卷)
   ├── sockets.target (套接字)
   └── timers.target (定时器)
   ↓
3. multi-user.target (多用户模式)
   ├── network.target (网络服务)
   ├── remote-fs.target (远程文件系统)
   └── [各种服务...]
   ↓
4. graphical.target (图形界面，可选)
```

### 16.6.3 启动优化技术

**1. 并行化启动**
```ini
# systemd service 文件
[Unit]
Description=My Service
After=network.target       # 依赖声明
Wants=other.service        # 弱依赖

[Service]
Type=notify                # 支持 sd_notify
ExecStart=/usr/bin/myservice
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

**2. 套接字激活**
```c
// 使用 systemd 套接字激活
int sd_listen_fds(int unset_environment);

int main() {
    int n = sd_listen_fds(0);
    if (n > 0) {
        // 使用 systemd 传递的文件描述符
        int fd = SD_LISTEN_FDS_START + 0;
        // ...
    }
}
```

**3. 启动时间分析**
```bash
# systemd 启动分析工具
systemd-analyze                 # 总体启动时间
systemd-analyze blame           # 每个服务的启动时间
systemd-analyze critical-chain  # 关键路径分析
systemd-analyze plot > boot.svg # 生成启动时序图
```

### 16.6.4 initramfs 与早期用户空间

initramfs（initial RAM filesystem）提供早期用户空间环境：

```c
// init/initramfs.c
static int __init populate_rootfs(void)
{
    char *err;

    /* 如果没有 initramfs，直接返回 */
    if (!initrd_start)
        return 0;

    /* 检查是否是 cpio 格式 */
    if (initrd_start && !IS_ENABLED(CONFIG_INITRAMFS_FORCE)) {
        err = unpack_to_rootfs((char *)initrd_start,
                              initrd_size);
        if (!err) {
            free_initrd_mem(initrd_start, initrd_end);
            return 0;
        }
    }

    /* 解压 initramfs */
    err = unpack_to_rootfs(__initramfs_start,
                          __initramfs_size);
    if (err)
        panic("%s", err);

    return 0;
}
rootfs_initcall(populate_rootfs);
```

initramfs 生成脚本：
```bash
#!/bin/bash
# 创建 initramfs

# 创建基本目录结构
mkdir -p initramfs/{bin,sbin,etc,proc,sys,dev,lib,lib64}

# 复制必要的二进制文件
cp /bin/busybox initramfs/bin/
for cmd in sh ls mount umount; do
    ln -s busybox initramfs/bin/$cmd
done

# 创建 init 脚本
cat > initramfs/init << 'EOF'
#!/bin/sh
/bin/mount -t proc none /proc
/bin/mount -t sysfs none /sys
/bin/mount -t devtmpfs none /dev

# 挂载真实根文件系统
/bin/mount -t ext4 /dev/sda1 /newroot

# 切换根文件系统
exec switch_root /newroot /sbin/init
EOF
chmod +x initramfs/init

# 打包成 cpio 归档
cd initramfs
find . | cpio -o -H newc | gzip > ../initramfs.cpio.gz
```

## 本章小结

本章深入探讨了 Linux 内核从上电到用户空间启动的完整过程。我们了解了：

1. **固件引导阶段**：BIOS/UEFI 的工作原理，bootloader 的多阶段加载机制，以及引导协议的设计
2. **内核解压和早期初始化**：bzImage 格式、解压算法选择、CPU 模式切换（实模式→保护模式→长模式）
3. **start_kernel 执行流程**：各个子系统的初始化顺序和依赖关系，从内存管理到调度器的建立
4. **initcall 机制**：驱动和模块的分级初始化系统，通过链接器脚本实现的优雅设计
5. **用户空间启动**：kernel_init 进程的创建，systemd 的并行化启动架构，initramfs 的作用

关键技术要点：
- 页表的建立过程：从早期恒等映射到完整的虚拟内存系统
- 中断系统的初始化：IDT 表的设置和异常处理机制
- 进程 0、1、2 的特殊角色：idle、init 和 kthreadd
- systemd 的 target 依赖图和套接字激活机制

启动优化的核心思想：
- 并行化：利用多核 CPU 并行执行初始化任务
- 延迟加载：将非关键组件推迟到需要时才初始化
- 缓存优化：合理安排代码和数据布局，减少缓存失效

## 练习题

### 基础题

**练习 16.1**：解释 BIOS 和 UEFI 启动的主要区别是什么？为什么现代系统倾向于使用 UEFI？

<details>
<summary>答案</summary>

主要区别：
1. **启动模式**：BIOS 使用 16 位实模式启动，UEFI 可以直接在 32/64 位模式下运行
2. **分区支持**：BIOS 使用 MBR（最大 2TB），UEFI 使用 GPT（支持超大硬盘）
3. **引导过程**：BIOS 需要多阶段引导（MBR→活动分区），UEFI 可以直接从 ESP 分区加载引导器
4. **安全性**：UEFI 支持安全启动（Secure Boot），可以验证引导器签名
5. **界面**：UEFI 提供图形化设置界面，支持鼠标操作
6. **驱动**：UEFI 有标准的驱动模型，BIOS 需要 INT 13h 等中断调用

选择 UEFI 的原因：支持大容量硬盘、更快的启动速度、更好的安全性、网络启动支持
</details>

**练习 16.2**：在 x86_64 架构下，内核从实模式切换到长模式需要经过哪些步骤？每个步骤的关键操作是什么？

<details>
<summary>答案</summary>

切换步骤：
1. **实模式→保护模式**：
   - 设置 GDT（全局描述符表）
   - 设置 CR0.PE 位（保护模式使能）
   - 执行长跳转刷新 CS 段选择器

2. **保护模式→长模式兼容**：
   - 检查 CPUID 确认 CPU 支持长模式
   - 设置 CR4.PAE 位（物理地址扩展）
   - 建立 4 级页表（PML4）
   - 设置 CR3 指向页表基址

3. **启用长模式**：
   - 设置 EFER.LME 位（长模式使能）
   - 设置 CR0.PG 位（启用分页）
   - 执行长跳转进入 64 位代码段

关键点：必须先启用 PAE，然后设置 LME，最后启用分页
</details>

**练习 16.3**：解释 initcall 的七个级别及其用途，为什么需要这种分级机制？

<details>
<summary>答案</summary>

initcall 级别：
1. **pure_initcall (0)**：最基础的初始化，如调试设施
2. **core_initcall (1)**：核心组件，如 CPU、内存检测
3. **postcore_initcall (2)**：核心后初始化，如 IRQ 子系统
4. **arch_initcall (3)**：架构特定初始化，如平台设备
5. **subsys_initcall (4)**：子系统初始化，如总线、电源管理
6. **fs_initcall (5)**：文件系统初始化
7. **device_initcall (6)**：设备驱动（module_init）
8. **late_initcall (7)**：最后的清理和优化

需要分级的原因：
- 确保依赖关系正确（如总线必须在设备之前初始化）
- 避免初始化顺序错误导致的崩溃
- 提供确定性的初始化顺序
- 便于调试和问题定位
</details>

### 进阶题

**练习 16.4**：分析 start_kernel 中为什么要先禁用中断，然后在 sched_init 之后才启用？这个顺序如果改变会有什么问题？

<details>
<summary>答案</summary>

禁用中断的原因：
1. **数据结构未就绪**：早期初始化时，中断处理所需的数据结构（如 IDT、per-CPU 变量）还未建立
2. **调度器未运行**：中断处理可能需要调度，但调度器还未初始化
3. **避免竞态条件**：初始化过程中的全局数据结构修改需要原子性

必须在 sched_init 后启用的原因：
- 中断处理可能唤醒进程，需要调度器支持
- 时钟中断需要更新调度器的时间片
- 软中断处理需要 ksoftirqd 线程

如果过早启用中断：
- 可能触发空指针解引用（处理函数未注册）
- 导致死锁（锁机制未初始化）
- 系统崩溃（必要的数据结构未建立）

如果过晚启用中断：
- 某些依赖中断的初始化会失败
- 定时器相关功能无法工作
- 设备检测可能超时
</details>

**练习 16.5**：设计一个最小的 initramfs，要求能够：挂载真实根文件系统、建立基本的 /dev 设备节点、处理启动失败的应急 shell。给出具体实现。

<details>
<summary>答案</summary>

最小 initramfs 实现：

```bash
#!/bin/bash
# 创建目录结构
mkdir -p minimal_initramfs/{bin,dev,etc,lib,lib64,mnt,proc,sys,root,tmp}

# 安装 busybox（静态链接版本）
cp /bin/busybox minimal_initramfs/bin/
cd minimal_initramfs/bin
for cmd in sh mount umount switch_root mknod ls cat echo; do
    ln -s busybox $cmd
done
cd ../..

# 创建 init 脚本
cat > minimal_initramfs/init << 'INIT_SCRIPT'
#!/bin/sh

# 应急 shell 函数
rescue_shell() {
    echo "启动失败，进入应急 shell"
    echo "问题：$1"
    exec sh
}

# 挂载必要的文件系统
mount -t proc none /proc || rescue_shell "无法挂载 /proc"
mount -t sysfs none /sys || rescue_shell "无法挂载 /sys"
mount -t devtmpfs none /dev || {
    # 如果 devtmpfs 不可用，手动创建设备节点
    mknod /dev/null c 1 3
    mknod /dev/zero c 1 5
    mknod /dev/random c 1 8
    mknod /dev/urandom c 1 9
    mknod /dev/tty c 5 0
    mknod /dev/console c 5 1
    mknod /dev/sda b 8 0
    mknod /dev/sda1 b 8 1
}

# 解析内核命令行获取根设备
ROOT_DEV=""
for param in $(cat /proc/cmdline); do
    case $param in
        root=*)
            ROOT_DEV="${param#root=}"
            ;;
    esac
done

[ -z "$ROOT_DEV" ] && rescue_shell "未指定 root 参数"

# 等待根设备出现（最多 10 秒）
COUNTER=0
while [ ! -b "$ROOT_DEV" ]; do
    sleep 1
    COUNTER=$((COUNTER + 1))
    [ $COUNTER -ge 10 ] && rescue_shell "根设备 $ROOT_DEV 不存在"
done

# 挂载根文件系统
mount -o ro "$ROOT_DEV" /mnt || rescue_shell "无法挂载根文件系统"

# 检查 init 是否存在
[ -x /mnt/sbin/init ] || rescue_shell "/sbin/init 不存在"

# 清理并切换根
umount /proc
umount /sys
umount /dev

# 切换到真实根文件系统
exec switch_root /mnt /sbin/init || rescue_shell "switch_root 失败"
INIT_SCRIPT

chmod +x minimal_initramfs/init

# 打包
cd minimal_initramfs
find . | cpio -o -H newc | gzip > ../minimal_initramfs.cpio.gz
cd ..

echo "initramfs 创建完成：minimal_initramfs.cpio.gz"
```

关键设计点：
1. 使用静态链接的 busybox 减少依赖
2. 实现 rescue_shell 提供调试能力
3. 支持 devtmpfs 和手动创建设备节点两种方式
4. 解析内核命令行获取启动参数
5. 实现设备等待机制避免竞态条件
</details>

**练习 16.6**：systemd 的 socket activation 机制是如何工作的？设计一个支持 socket activation 的简单服务。

<details>
<summary>答案</summary>

Socket Activation 工作原理：
1. systemd 预先创建并监听套接字
2. 当有连接到达时，systemd 启动对应的服务
3. systemd 通过环境变量传递套接字文件描述符
4. 服务直接使用这些文件描述符，无需自己创建套接字

实现示例：

```c
// echo_server.c - 支持 socket activation 的回显服务器
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <systemd/sd-daemon.h>
#include <sys/socket.h>
#include <netinet/in.h>

#define BUFFER_SIZE 1024

int main() {
    int listen_fd;
    int n_fds;
    
    // 检查是否由 systemd 启动
    n_fds = sd_listen_fds(0);
    
    if (n_fds > 0) {
        // 使用 systemd 传递的套接字
        listen_fd = SD_LISTEN_FDS_START + 0;
        sd_notify(0, "READY=1\nSTATUS=Using systemd socket");
    } else {
        // 回退：自己创建套接字
        listen_fd = socket(AF_INET, SOCK_STREAM, 0);
        struct sockaddr_in addr = {
            .sin_family = AF_INET,
            .sin_port = htons(8080),
            .sin_addr.s_addr = INADDR_ANY
        };
        
        if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0 ||
            listen(listen_fd, 10) < 0) {
            perror("Failed to create socket");
            return 1;
        }
    }
    
    // 主循环
    while (1) {
        int client_fd = accept(listen_fd, NULL, NULL);
        if (client_fd < 0) continue;
        
        char buffer[BUFFER_SIZE];
        ssize_t n = read(client_fd, buffer, BUFFER_SIZE-1);
        if (n > 0) {
            buffer[n] = '\0';
            write(client_fd, buffer, n);  // 回显
        }
        
        close(client_fd);
        
        // 通知 systemd 处理了一个连接
        sd_notify(0, "STATUS=Processed a connection");
    }
    
    return 0;
}
```

systemd 配置文件：

```ini
# echo.socket
[Unit]
Description=Echo Server Socket

[Socket]
ListenStream=8080
Accept=no

[Install]
WantedBy=sockets.target

# echo.service
[Unit]
Description=Echo Server
Requires=echo.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/echo_server
StandardOutput=journal
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

优势：
- 零停机时间重启（套接字保持监听）
- 按需启动服务（节省资源）
- 并行化启动（服务可以延迟到真正需要时）
</details>

### 挑战题

**练习 16.7**：分析 Linux 内核启动过程中的内存布局变化，从实模式的 1MB 限制到最终的虚拟内存系统。画出各个阶段的内存映射图。

<details>
<summary>提示</summary>

需要考虑的阶段：
1. 实模式：段:偏移寻址，20 位地址总线
2. 保护模式早期：恒等映射，GDT 段描述符
3. 启用分页后：临时页表，内核空间映射
4. 最终状态：完整的虚拟内存布局

关键转换点：
- setup_idt：建立中断描述符表
- init_mem_mapping：建立内核直接映射
- setup_per_cpu_areas：建立 per-CPU 区域
- vmalloc_init：初始化 vmalloc 区域

需要关注的地址范围：
- 0-1MB：实模式可访问区域
- 1MB-16MB：ISA DMA 区域
- 内核代码段：_text 到 _etext
- 内核直接映射：PAGE_OFFSET 开始
- vmalloc 区域：VMALLOC_START 到 VMALLOC_END
</details>

**练习 16.8**：设计并实现一个启动时间分析工具，能够精确测量内核各个初始化阶段的耗时，并生成火焰图。考虑如何处理早期启动（时钟未初始化）的计时问题。

<details>
<summary>提示</summary>

实现思路：
1. **早期计时**：使用 TSC（Time Stamp Counter）或 HPET
2. **数据收集**：在每个 initcall 前后记录时间戳
3. **存储机制**：使用静态数组或 early_memremap
4. **数据导出**：通过 /proc 或 debugfs 接口
5. **可视化**：转换为 FlameGraph 格式

关键技术点：
- rdtsc 指令读取 CPU 时间戳
- __init 段的内存会被释放，需要持久化存储
- 考虑多 CPU 的时间同步问题
- 处理时钟频率变化（CPU 动态调频）

输出格式设计：
```
initcall_name;parent_function;duration_us
```

工具集成：
- 与 ftrace 集成获取更详细的调用栈
- 支持 bootchart 格式输出
- 生成 systemd-analyze 兼容的数据
</details>

## 常见陷阱与错误 (Gotchas)

### 1. 启动参数错误
**问题**：内核命令行参数拼写错误或格式不正确导致启动失败
```bash
# 错误：参数之间需要空格
root=/dev/sda1init=/sbin/init  

# 正确：
root=/dev/sda1 init=/sbin/init
```
**解决**：使用 `early_param` 和 `__setup` 正确注册参数处理函数

### 2. initcall 顺序依赖
**问题**：驱动初始化顺序错误导致依赖未满足
```c
// 错误：网卡驱动可能在网络子系统之前初始化
module_init(network_driver_init);  // device_initcall 级别

// 正确：使用更晚的初始化级别
late_initcall(network_driver_init);
```

### 3. 早期内存访问
**问题**：在 MMU 启用前访问虚拟地址
```c
// 错误：早期代码不能使用 kmalloc
void __init early_setup(void) {
    char *buf = kmalloc(256, GFP_KERNEL);  // 崩溃！
}

// 正确：使用 alloc_bootmem 或静态分配
void __init early_setup(void) {
    static char buf[256];  // 或 alloc_bootmem(256)
}
```

### 4. init 段内存释放
**问题**：__init 函数的指针在 init 段释放后使用
```c
// 错误：保存 __init 函数指针
static int (*saved_func)(void);
int __init init_module(void) {
    saved_func = some_init_function;  // 危险！
}

// 正确：不保存 __init 函数指针，或移除 __init
```

### 5. 设备探测竞态
**问题**：根设备还未出现就尝试挂载
```bash
# 内核 panic: VFS: Unable to mount root fs
```
**解决**：使用 rootwait 参数或在 initramfs 中实现等待逻辑

### 6. UEFI 安全启动
**问题**：未签名的内核或模块在 Secure Boot 环境下无法加载
**解决**：
- 使用 MOK (Machine Owner Key) 签名内核模块
- 或在 BIOS 中禁用 Secure Boot
- 使用 shim 加载器链

## 最佳实践检查清单

### 启动性能优化
- [ ] 选择合适的内核压缩算法（LZ4 用于快速解压）
- [ ] 精简内核配置，移除不需要的驱动
- [ ] 使用 initcall_blacklist 禁用不必要的初始化
- [ ] 启用并行化选项（如 PARALLEL_BOOT）
- [ ] 优化 initramfs 大小，只包含必要组件

### initcall 设计
- [ ] 选择正确的 initcall 级别
- [ ] 实现 -EPROBE_DEFER 支持延迟探测
- [ ] 避免在 initcall 中进行长时间阻塞操作
- [ ] 使用 async_schedule 进行异步初始化
- [ ] 添加适当的错误处理和回滚机制

### 调试和诊断
- [ ] 启用 initcall_debug 跟踪启动过程
- [ ] 使用 ftrace 的 function_graph 跟踪器
- [ ] 配置 early_printk 用于早期调试
- [ ] 保存启动日志用于后续分析
- [ ] 使用 systemd-analyze 分析用户空间启动

### 安全考虑
- [ ] 验证引导器和内核签名（Secure Boot）
- [ ] 保护内核命令行参数（避免注入）
- [ ] 限制 initramfs 的权限和内容
- [ ] 启用 KASLR（内核地址空间随机化）
- [ ] 审计 init 脚本避免安全漏洞

### 可靠性保证
- [ ] 实现 watchdog 监控启动过程
- [ ] 提供失败后的恢复机制（rescue shell）
- [ ] 记录关键启动阶段的检查点
- [ ] 实现启动失败的自动回滚
- [ ] 测试各种启动失败场景

### 文档和维护
- [ ] 记录自定义的内核参数
- [ ] 维护 initcall 依赖关系文档
- [ ] 记录启动时间基准和优化历史
- [ ] 提供启动问题的故障排除指南
- [ ] 定期更新和测试恢复流程