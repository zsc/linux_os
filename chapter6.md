# 第6章：具体文件系统实现

## 章节大纲

### 6.1 开篇与 ext2/3/4 演进
- ext 文件系统家族历史
- ext2 基础架构：超级块、块组、inode表
- ext3 日志机制引入
- ext4 现代化改进：extent、延迟分配、多块分配

### 6.2 日志文件系统原理
- 日志机制设计动机
- JBD/JBD2 日志层实现
- 三种日志模式：journal、ordered、writeback
- 崩溃恢复与一致性保证

### 6.3 现代文件系统：Btrfs、XFS
- XFS：高性能与可扩展性
- Btrfs：写时复制与高级特性
- ZFS 影响与设计理念
- 性能特征对比

### 6.4 特殊文件系统
- proc：进程信息与内核接口
- sysfs：设备模型导出
- debugfs：内核调试接口
- tmpfs/ramfs：内存文件系统

### 6.5 本章小结
- 关键数据结构总结
- 算法复杂度分析
- 选型决策指南

### 6.6 练习题（8道）
- 基础题：文件系统结构理解
- 进阶题：性能分析与优化
- 挑战题：设计权衡与实现

### 6.7 常见陷阱与错误
- 文件系统损坏与修复
- 性能问题诊断
- 容量规划失误

### 6.8 最佳实践检查清单
- 文件系统选择准则
- 调优参数配置
- 监控与维护要点

---

## 6.1 ext2/3/4 文件系统演进与实现

本章深入探讨 Linux 中各种文件系统的具体实现，从经典的 ext 系列到现代的 Btrfs、XFS，再到特殊用途的 proc、sysfs 等伪文件系统。通过理解这些实现细节，我们将掌握文件系统设计的核心权衡：性能、可靠性、功能特性之间的平衡。每种文件系统都是特定需求下的产物，理解它们的设计决策对于系统架构师至关重要。

### 6.1.1 ext 文件系统家族历史

ext（extended filesystem）家族是 Linux 最重要的文件系统演进脉络。从 1992 年 Rémy Card 开发的 ext 开始，历经 ext2（1993）、ext3（2001）、ext4（2008），每一代都在前代基础上解决关键问题：

```
时间线：
1992: ext  - 替代 MINIX FS，支持 2GB 文件
1993: ext2 - 成为 Linux 标准文件系统，引入块组概念
2001: ext3 - 增加日志功能，提供崩溃一致性
2008: ext4 - 支持更大容量，引入 extent、延迟分配等现代特性
```

**历史故事**：ext2 的设计深受 BSD FFS（Fast File System）影响，特别是块组（block group）概念。Theodore Ts'o 在 1994 年接手维护后，ext2 成为 Linux 事实标准。而 ext3 的开发则是由 Stephen Tweedie 主导，他提出的 JBD（Journaling Block Device）层成为 Linux 日志文件系统的基础。

### 6.1.2 ext2 基础架构

ext2 确立了 ext 家族的基本布局，理解它是掌握后续版本的关键：

#### 磁盘布局

```
+------------------+
| 引导块 (1KB)      |  - 保留给引导加载器
+------------------+
| 块组 0           |  ┐
|   超级块         |  │
|   组描述符表      |  │
|   数据块位图      |  │  块组结构
|   inode 位图     |  │  (重复 N 次)
|   inode 表       |  │
|   数据块         |  │
+------------------+  ┘
| 块组 1           |
|   ...            |
+------------------+
| ...              |
+------------------+
| 块组 N-1         |
+------------------+
```

#### 核心数据结构

**超级块（Super Block）**：
```c
struct ext2_super_block {
    __le32  s_inodes_count;       /* inode 总数 */
    __le32  s_blocks_count;       /* 块总数 */
    __le32  s_r_blocks_count;     /* 保留块数 */
    __le32  s_free_blocks_count;  /* 空闲块数 */
    __le32  s_free_inodes_count;  /* 空闲 inode 数 */
    __le32  s_first_data_block;   /* 第一个数据块号 */
    __le32  s_log_block_size;     /* 块大小 = 1024 << s_log_block_size */
    __le32  s_blocks_per_group;   /* 每组块数 */
    __le32  s_inodes_per_group;   /* 每组 inode 数 */
    __le32  s_mtime;              /* 挂载时间 */
    __le32  s_wtime;              /* 写入时间 */
    __le16  s_magic;              /* 魔数 0xEF53 */
    __le16  s_state;              /* 文件系统状态 */
    /* ... 更多字段 ... */
};
```

**inode 结构**：
```c
struct ext2_inode {
    __le16  i_mode;        /* 文件模式 */
    __le16  i_uid;         /* 用户 ID */
    __le32  i_size;        /* 文件大小（字节）*/
    __le32  i_atime;       /* 访问时间 */
    __le32  i_ctime;       /* 创建时间 */
    __le32  i_mtime;       /* 修改时间 */
    __le32  i_dtime;       /* 删除时间 */
    __le16  i_gid;         /* 组 ID */
    __le16  i_links_count; /* 硬链接数 */
    __le32  i_blocks;      /* 512 字节块数 */
    __le32  i_block[15];   /* 数据块指针 */
    /* i_block[0-11]: 直接块
       i_block[12]: 一级间接块
       i_block[13]: 二级间接块  
       i_block[14]: 三级间接块 */
};
```

#### 块组设计哲学

块组（Block Group）是 ext2 的核心创新，其设计目标是：

1. **局部性优化**：相关数据放在同一块组，减少磁头移动
2. **并行化**：不同块组可以并行操作
3. **故障隔离**：单个块组损坏不影响整个文件系统

块分配算法遵循以下策略：
- 目录：在父目录所在块组或负载最轻的块组创建
- 文件：优先在父目录所在块组分配
- 大文件：跨块组分配时保持连续性

### 6.1.3 ext3 日志机制引入

ext3 最重要的贡献是引入日志，解决了 ext2 的崩溃一致性问题：

#### JBD 层架构

```
     应用程序
         ↓
    VFS 层接口
         ↓
    ext3 文件系统
         ↓
    JBD 日志层  ← 事务管理
         ↓
    块设备层
```

#### 日志事务流程

```
事务生命周期：
T_RUNNING → T_LOCKED → T_FLUSH → T_COMMIT → T_FINISHED

1. T_RUNNING：收集修改操作
2. T_LOCKED：停止接受新操作
3. T_FLUSH：写入日志区
4. T_COMMIT：提交事务
5. T_FINISHED：可以回收日志空间
```

#### 三种日志模式对比

| 模式 | 日志内容 | 性能 | 一致性保证 |
|------|---------|------|-----------|
| journal | 元数据+数据 | 最慢 | 最强 |
| ordered | 仅元数据，数据先写 | 中等 | 较强 |
| writeback | 仅元数据 | 最快 | 仅元数据一致 |

默认使用 ordered 模式，平衡了性能和一致性。

### 6.1.4 ext4 现代化改进

ext4 在 2008 年成为默认文件系统，引入多项关键改进：

#### Extent 树替代块映射

传统块映射的问题：
- 大文件需要多级间接块
- 碎片化严重时性能下降
- 元数据开销大

Extent 解决方案：
```c
struct ext4_extent {
    __le32  ee_block;     /* 逻辑块号 */
    __le16  ee_len;       /* extent 长度 */
    __le16  ee_start_hi;  /* 物理块号高 16 位 */
    __le32  ee_start_lo;  /* 物理块号低 32 位 */
};

/* Extent 树示例：
   一个 100MB 连续文件只需要一个 extent 记录
   而块映射需要 25600 个块指针 */
```

#### 延迟分配（Delayed Allocation）

```
传统分配：
write() → 立即分配块 → 写入页缓存 → 后台刷新

延迟分配：
write() → 写入页缓存 → 后台刷新时分配块

优势：
1. 更好的块分配决策（知道实际大小）
2. 减少碎片
3. 避免短生命文件的块分配
4. 提高小文件写入性能
```

#### 多块分配（Multiblock Allocation）

使用 mballoc（Multiblock Allocator）：
- **伙伴系统**：管理空闲块
- **预分配**：为文件预留连续空间
- **局部性组**：相关文件放在一起

```c
/* 预分配策略 */
struct ext4_prealloc_space {
    struct list_head pa_inode_list;  /* inode 预分配链表 */
    struct list_head pa_group_list;  /* 块组预分配链表 */
    union {
        struct rb_node pa_node;       /* 用于 inode 预分配 */
        struct list_head pa_tmp_list;/* 临时链表 */
    } u;
    spinlock_t pa_lock;
    atomic_t pa_count;
    unsigned pa_deleted;
    ext4_fsblk_t pa_pstart;          /* 物理起始块 */
    ext4_lblk_t pa_lstart;           /* 逻辑起始块 */
    ext4_grpblk_t pa_len;            /* 长度 */
    ext4_grpblk_t pa_free;           /* 剩余空闲块 */
};
```

#### 其他重要特性

1. **更大的文件系统支持**：
   - 最大卷：1 EB（ext3：32 TB）
   - 最大文件：16 TB（ext3：2 TB）

2. **快速 fsck**：
   - 未使用的 inode 表部分不检查
   - inode 表初始化可延迟到使用时

3. **纳秒级时间戳**：
   - 支持纳秒精度
   - 额外的创建时间字段

4. **持久预分配**：
   - fallocate() 系统调用支持
   - 数据库等应用可预留空间

### 6.1.5 ext4 内核实现分析

让我们深入内核代码，理解 ext4 的关键操作：

#### inode 操作实现

```c
/* fs/ext4/inode.c */
const struct inode_operations ext4_file_inode_operations = {
    .setattr    = ext4_setattr,
    .getattr    = ext4_getattr,
    .listxattr  = ext4_listxattr,
    .get_acl    = ext4_get_acl,
    .set_acl    = ext4_set_acl,
    .fiemap     = ext4_fiemap,
};

/* 文件写入路径 */
static int ext4_write_begin(struct file *file, 
                            struct address_space *mapping,
                            loff_t pos, unsigned len, unsigned flags,
                            struct page **pagep, void **fsdata)
{
    struct inode *inode = mapping->host;
    int ret, needed_blocks;
    
    /* 1. 计算需要的块数 */
    needed_blocks = ext4_writepage_trans_blocks(inode);
    
    /* 2. 开始日志事务 */
    handle = ext4_journal_start(inode, EXT4_HT_WRITE_PAGE, needed_blocks);
    
    /* 3. 准备写入页 */
    if (ext4_should_dioread_nolock(inode))
        ret = ext4_write_begin_inline(mapping, inode, pos, len, 
                                      flags, pagep);
    else
        ret = __block_write_begin(page, pos, len, ext4_get_block);
    
    /* 4. 延迟分配处理 */
    if (ret == 0 && ext4_should_journal_data(inode))
        ret = ext4_walk_page_buffers(handle, page_buffers(page),
                                     from, to, NULL, 
                                     do_journal_get_write_access);
    return ret;
}
```

#### Extent 操作

```c
/* fs/ext4/extents.c */
int ext4_ext_map_blocks(handle_t *handle, struct inode *inode,
                        struct ext4_map_blocks *map, int flags)
{
    struct ext4_ext_path *path = NULL;
    struct ext4_extent newex, *ex;
    
    /* 1. 查找 extent */
    path = ext4_find_extent(inode, map->m_lblk, NULL, 0);
    
    /* 2. 检查是否命中缓存 */
    if ((ex = path[depth].p_ext)) {
        ext4_lblk_t ee_block = le32_to_cpu(ex->ee_block);
        ext4_fsblk_t ee_start = ext4_ext_pblock(ex);
        unsigned short ee_len = ext4_ext_get_actual_len(ex);
        
        if (in_range(map->m_lblk, ee_block, ee_len)) {
            /* 命中：直接映射 */
            map->m_pblk = ee_start + map->m_lblk - ee_block;
            map->m_len = ee_len - (map->m_lblk - ee_block);
            goto out;
        }
    }
    
    /* 3. 未命中：分配新块 */
    if (flags & EXT4_GET_BLOCKS_CREATE) {
        newex.ee_block = cpu_to_le32(map->m_lblk);
        cluster = ext4_ext_map_clusters(inode, map, flags);
        
        /* 4. 插入新 extent */
        err = ext4_ext_insert_extent(handle, inode, &path, &newex, flags);
    }
    
out:
    ext4_ext_drop_refs(path);
    return err;
}
```

## 6.2 日志文件系统原理

日志文件系统是现代文件系统的核心特性，它解决了传统文件系统最大的痛点：崩溃一致性。在 ext2 时代，系统崩溃后的 fsck 可能需要数小时，而日志机制将恢复时间缩短到秒级。本节深入剖析日志机制的设计原理和实现细节。

### 6.2.1 为什么需要日志

#### 崩溃一致性问题

考虑一个简单的文件创建操作，涉及多个元数据更新：

```
1. 分配 inode，更新 inode 位图
2. 初始化 inode 结构
3. 在父目录中创建目录项
4. 更新父目录的修改时间
5. 更新超级块的计数器
```

如果在步骤 3 之后崩溃，会出现：
- inode 已分配但无法访问（孤儿 inode）
- 空间泄漏
- 文件系统不一致

#### 传统解决方案的问题

**同步元数据更新**：
- 每次操作都同步写磁盘
- 性能极差（10-100 倍慢）
- 仍无法保证原子性

**软更新（Soft Updates）**：
- BSD 的解决方案
- 复杂的依赖跟踪
- 实现困难，仍需要后台 fsck

### 6.2.2 日志机制核心概念

#### Write-Ahead Logging (WAL)

日志的核心思想来自数据库的 WAL：

```
原理：先写日志，后写数据
1. 将要执行的操作写入日志
2. 确保日志落盘
3. 执行实际的数据/元数据更新
4. 标记日志项完成
```

#### 事务（Transaction）

将相关的修改组织成事务，保证原子性：

```c
/* JBD2 事务结构 */
struct transaction_s {
    journal_t *t_journal;           /* 所属日志 */
    tid_t t_tid;                    /* 事务 ID */
    enum {
        T_RUNNING,                  /* 接受新的修改 */
        T_LOCKED,                   /* 不再接受修改，准备提交 */
        T_FLUSH,                    /* 正在写入日志 */
        T_COMMIT,                   /* 提交中 */
        T_COMMIT_DFLUSH,           /* 提交后刷盘 */
        T_COMMIT_JFLUSH,           /* 日志刷盘 */
        T_FINISHED                  /* 完成，可回收 */
    } t_state;
    
    struct journal_head *t_buffers;        /* 缓冲区链表 */
    struct journal_head *t_forget;         /* 需要忘记的缓冲区 */
    struct journal_head *t_checkpoint_list;/* 检查点链表 */
    
    unsigned long t_expires;               /* 超时时间 */
    ktime_t t_start_time;                  /* 开始时间 */
    atomic_t t_updates;                    /* 活跃更新数 */
    atomic_t t_outstanding_credits;        /* 使用的日志空间 */
};
```

### 6.2.3 JBD2 架构详解

JBD2（Journaling Block Device 2）是 ext4 和 OCFS2 使用的日志层：

#### 日志布局

```
日志区域（循环缓冲区）：
+------------+------------+------------+------------+
| 超级块     | 描述符块   | 数据/元数据 | 提交块     |
+------------+------------+------------+------------+
     ↑                                         ↑
   head                                      tail
   (最旧的未检查点事务)                    (最新提交位置)
```

#### 日志超级块

```c
/* 日志超级块 */
typedef struct journal_superblock_s {
    /* 静态信息 */
    journal_header_t s_header;
    __be32 s_blocksize;            /* 日志块大小 */
    __be32 s_maxlen;               /* 日志区总块数 */
    __be32 s_first;                /* 第一个日志块 */
    
    /* 动态信息 */
    __be32 s_sequence;             /* 第一个提交 ID */
    __be32 s_start;                /* 第一个日志块的块号 */
    __be32 s_errno;                /* 错误码 */
    
    /* 特性标志 */
    __be32 s_feature_compat;       /* 兼容特性 */
    __be32 s_feature_incompat;     /* 不兼容特性 */
    __be32 s_feature_ro_compat;    /* 只读兼容特性 */
    
    /* 校验和 */
    __be32 s_checksum;             /* CRC32C 校验和 */
} journal_superblock_t;
```

#### 日志记录类型

```c
/* 日志块标签 */
typedef struct journal_block_tag_s {
    __be32 t_blocknr;              /* 磁盘块号 */
    __be16 t_checksum;             /* 块校验和 */
    __be16 t_flags;                /* 标志 */
    __be32 t_blocknr_high;         /* 块号高 32 位 */
} journal_block_tag_t;

#define JBD2_FLAG_ESCAPE    1      /* 块内容需要转义 */
#define JBD2_FLAG_SAME_UUID 2      /* UUID 相同 */
#define JBD2_FLAG_DELETED   4      /* 块已删除 */
#define JBD2_FLAG_LAST_TAG  8      /* 最后一个标签 */
```

### 6.2.4 三种日志模式深度分析

#### Journal 模式（数据日志）

```
操作流程：
1. 元数据 + 数据 → 日志
2. 日志落盘
3. 元数据 + 数据 → 最终位置
4. 提交事务

优点：
- 最强的一致性保证
- 数据和元数据都不会丢失

缺点：
- 所有数据写两次（日志 + 最终位置）
- 性能开销大
- 日志空间需求高

适用场景：
- 关键数据，不容许任何丢失
- 小文件为主的工作负载
```

#### Ordered 模式（默认）

```
操作流程：
1. 数据 → 最终位置
2. 数据落盘
3. 元数据 → 日志
4. 日志落盘
5. 元数据 → 最终位置
6. 提交事务

优点：
- 平衡性能和一致性
- 防止垃圾数据暴露
- 日志空间需求小

缺点：
- 数据写入成为瓶颈
- 大文件写入延迟事务提交

实现要点：
- 使用 JBD2_FEATURE_INCOMPAT_ASYNC_COMMIT
- 数据块写入使用 bio 批量提交
```

#### Writeback 模式

```
操作流程：
1. 元数据 → 日志
2. 日志落盘
3. 元数据 → 最终位置
4. 数据异步写入（不保证顺序）

优点：
- 最佳性能
- 最小日志开销

缺点：
- 可能暴露垃圾数据
- 崩溃后文件内容不确定

适用场景：
- 临时文件
- 可重新生成的数据
- 性能优先的场景
```

### 6.2.5 日志恢复机制

#### 恢复流程

```c
/* fs/jbd2/recovery.c */
int jbd2_journal_recover(journal_t *journal)
{
    int err, err2;
    journal_superblock_t *sb;
    struct recovery_info info;
    
    /* 1. 读取日志超级块 */
    sb = journal->j_superblock;
    
    /* 2. 扫描日志，找到有效事务范围 */
    err = do_one_pass(journal, &info, PASS_SCAN);
    
    /* 3. 重放日志（REDO） */
    if (!err)
        err = do_one_pass(journal, &info, PASS_REVOKE);
    if (!err)
        err = do_one_pass(journal, &info, PASS_REPLAY);
    
    /* 4. 清理日志区域 */
    journal->j_transaction_sequence = info.end_transaction;
    jbd2_journal_clear_revoke(journal);
    
    /* 5. 同步文件系统 */
    sync_blockdev(journal->j_fs_dev);
    
    return err;
}
```

#### 检查点机制

检查点（Checkpoint）将内存中的脏数据写入磁盘，释放日志空间：

```c
/* 检查点处理 */
int jbd2_log_do_checkpoint(journal_t *journal)
{
    transaction_t *transaction;
    struct journal_head *jh;
    struct buffer_head *bh;
    int result;
    
    /* 获取最旧的未检查点事务 */
    transaction = journal->j_checkpoint_transactions;
    
    while (transaction) {
        jh = transaction->t_checkpoint_list;
        
        while (jh) {
            bh = jh2bh(jh);
            
            /* 写入脏缓冲区 */
            if (buffer_dirty(bh)) {
                get_bh(bh);
                spin_unlock(&journal->j_list_lock);
                ll_rw_block(REQ_OP_WRITE, 1, &bh);
                put_bh(bh);
                spin_lock(&journal->j_list_lock);
            }
            
            /* 等待 I/O 完成 */
            if (buffer_locked(bh)) {
                get_bh(bh);
                spin_unlock(&journal->j_list_lock);
                wait_on_buffer(bh);
                put_bh(bh);
                spin_lock(&journal->j_list_lock);
            }
            
            /* 从检查点链表移除 */
            __jbd2_journal_remove_checkpoint(jh);
            jh = next_jh;
        }
        
        /* 移动到下一个事务 */
        transaction = transaction->t_cpnext;
    }
    
    return result;
}
```

### 6.2.6 性能优化技术

#### 批量提交（Batching）

```c
/* 批量提交优化 */
void jbd2_journal_commit_transaction(journal_t *journal)
{
    transaction_t *commit_transaction;
    struct buffer_head **wbuf;
    int bufs = 0;
    
    /* 收集所有需要写入的缓冲区 */
    wbuf = kmalloc(sizeof(*wbuf) * MAX_WRITEBACK_PAGES, GFP_NOFS);
    
    list_for_each_entry(jh, &commit_transaction->t_buffers, b_tnext) {
        wbuf[bufs++] = jh2bh(jh);
        
        if (bufs == MAX_WRITEBACK_PAGES) {
            /* 批量提交 */
            ll_rw_block(REQ_OP_WRITE | REQ_SYNC, bufs, wbuf);
            bufs = 0;
        }
    }
    
    /* 提交剩余的缓冲区 */
    if (bufs)
        ll_rw_block(REQ_OP_WRITE | REQ_SYNC, bufs, wbuf);
}
```

#### 并行日志写入

JBD2 支持多个 CPU 并行准备日志记录：

```c
/* 并行日志准备 */
handle_t *jbd2__journal_start(journal_t *journal, int nblocks, 
                              unsigned int type, int line)
{
    handle_t *handle;
    transaction_t *transaction;
    
    /* 每个 CPU 独立的 handle */
    handle = new_handle(nblocks);
    
    /* 获取当前运行事务 */
    read_lock(&journal->j_state_lock);
    transaction = journal->j_running_transaction;
    
    /* 原子地增加引用计数 */
    atomic_inc(&transaction->t_updates);
    
    /* 预留日志空间 */
    atomic_sub(nblocks, &transaction->t_outstanding_credits);
    
    read_unlock(&journal->j_state_lock);
    
    handle->h_transaction = transaction;
    return handle;
}
```

#### 快速提交（Fast Commit）

ext4 引入的快速提交机制，用于加速 fsync：

```c
/* 快速提交实现 */
int ext4_fc_commit(journal_t *journal, tid_t tid)
{
    struct ext4_fc_commit_info *fc_info;
    
    /* 1. 检查是否可以快速提交 */
    if (!ext4_fc_can_commit(journal))
        return jbd2_complete_transaction(journal, tid);
    
    /* 2. 准备快速提交信息 */
    fc_info = ext4_fc_prepare(journal);
    
    /* 3. 写入快速提交日志 */
    ext4_fc_write_inode(fc_info);
    ext4_fc_write_data(fc_info);
    
    /* 4. 提交快速日志 */
    ext4_fc_submit(fc_info);
    
    return 0;
}
```
