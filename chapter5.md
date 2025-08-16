# 第5章：虚拟文件系统（VFS）

## 章节大纲

1. **开篇与VFS概述**
   - VFS的设计哲学与目标
   - VFS在Linux内核中的位置
   - 统一文件模型的重要性

2. **VFS核心架构**
   - 2.1 四大核心对象
     - 超级块（superblock）
     - 索引节点（inode）
     - 目录项（dentry）
     - 文件对象（file）
   - 2.2 对象间的关系与生命周期
   - 2.3 VFS操作表（operations）

3. **文件系统注册与挂载**
   - 3.1 文件系统类型注册
   - 3.2 挂载过程分析
   - 3.3 命名空间与挂载传播
   - 3.4 根文件系统的特殊处理

4. **路径名查找与目录项缓存**
   - 4.1 路径解析算法
   - 4.2 目录项缓存（dcache）机制
   - 4.3 RCU路径遍历优化
   - 4.4 符号链接处理

5. **文件操作接口实现**
   - 5.1 系统调用到VFS的映射
   - 5.2 文件描述符管理
   - 5.3 读写操作路径
   - 5.4 内存映射（mmap）支持

6. **算法焦点**
   - 哈希表在dcache中的应用
   - LRU缓存管理策略
   - RCU在路径查找中的应用
   - 负目录项（negative dentry）优化

7. **历史故事**
   - VFS层的诞生与演进
   - Al Viro的VFS重构贡献
   - 从单一文件系统到多文件系统支持

8. **高级话题**
   - overlayfs与容器存储
   - FUSE用户态文件系统
   - VFS层的并发优化
   - 延迟分配与预读策略

## 本章学习目标

完成本章学习后，您将能够：
1. 理解VFS的抽象层设计及其在内核中的核心作用
2. 掌握VFS四大对象的数据结构和相互关系
3. 分析文件系统挂载和路径查找的实现机制
4. 理解dcache的优化策略和RCU的应用
5. 掌握从系统调用到具体文件系统的完整调用链
6. 能够实现简单的虚拟文件系统

---

## 1. 引言

虚拟文件系统（Virtual File System，VFS）是Linux内核中最优雅的设计之一。它通过在用户空间和具体文件系统实现之间建立一个抽象层，使得Linux能够支持数十种不同的文件系统，从传统的ext4、XFS，到网络文件系统NFS、SMB，再到特殊用途的proc、sysfs，所有这些都能通过统一的接口被访问。本章将深入剖析VFS的设计理念、核心数据结构、关键算法以及性能优化技术，帮助您理解Linux如何实现"一切皆文件"的设计哲学。

## 2. VFS核心架构

### 2.1 VFS的设计哲学

VFS的核心设计目标是提供一个统一的文件访问接口，隐藏底层文件系统的实现细节。这种抽象带来了三个关键优势：

1. **透明性**：应用程序无需关心文件存储在哪种文件系统上
2. **可扩展性**：新文件系统可以通过实现VFS接口轻松集成
3. **一致性**：所有文件操作遵循相同的语义和错误处理机制

VFS在内核中的位置可以用以下ASCII图表示：

```
    用户空间应用程序
         |
    系统调用接口 (open, read, write, close...)
         |
    +---------+
    |   VFS   |  <--- 抽象层
    +---------+
         |
    +----+----+----+----+
    |    |    |    |    |
   ext4 XFS  NFS proc tmpfs  <--- 具体文件系统
```

### 2.2 四大核心对象

VFS通过四个核心对象来抽象文件系统的概念：

#### 2.2.1 超级块（struct super_block）

超级块代表一个已挂载的文件系统实例，包含文件系统的全局信息：

```c
struct super_block {
    struct list_head    s_list;        /* 超级块链表 */
    dev_t               s_dev;         /* 设备标识符 */
    unsigned char       s_blocksize_bits;
    unsigned long       s_blocksize;   /* 块大小 */
    loff_t              s_maxbytes;    /* 最大文件大小 */
    struct file_system_type *s_type;   /* 文件系统类型 */
    const struct super_operations *s_op; /* 超级块操作表 */
    
    unsigned long       s_flags;       /* 挂载标志 */
    unsigned long       s_magic;       /* 魔数 */
    struct dentry       *s_root;       /* 根目录项 */
    
    int                 s_count;       /* 引用计数 */
    atomic_t            s_active;      /* 活动引用计数 */
    
    struct list_head    s_inodes;      /* inode链表 */
    struct list_head    s_dirty;       /* 脏inode链表 */
    
    struct block_device *s_bdev;       /* 块设备 */
    struct backing_dev_info *s_bdi;    /* 后备设备信息 */
    
    void                *s_fs_info;    /* 文件系统私有信息 */
    
    /* 配额、安全等其他字段... */
};
```

超级块操作表（super_operations）定义了文件系统级别的操作：

```c
struct super_operations {
    struct inode *(*alloc_inode)(struct super_block *sb);
    void (*destroy_inode)(struct inode *);
    void (*dirty_inode)(struct inode *, int flags);
    int (*write_inode)(struct inode *, struct writeback_control *wbc);
    int (*drop_inode)(struct inode *);
    void (*evict_inode)(struct inode *);
    void (*put_super)(struct super_block *);
    int (*sync_fs)(struct super_block *sb, int wait);
    int (*statfs)(struct dentry *, struct kstatfs *);
    int (*remount_fs)(struct super_block *, int *, char *);
    /* 更多操作... */
};
```

#### 2.2.2 索引节点（struct inode）

inode代表文件系统中的一个对象（文件、目录、设备等），包含对象的元数据：

```c
struct inode {
    umode_t             i_mode;        /* 文件类型和权限 */
    unsigned short      i_opflags;
    kuid_t              i_uid;         /* 所有者用户ID */
    kgid_t              i_gid;         /* 所有者组ID */
    unsigned int        i_flags;
    
    const struct inode_operations *i_op;  /* inode操作表 */
    struct super_block  *i_sb;         /* 所属超级块 */
    struct address_space *i_mapping;   /* 页缓存映射 */
    
    unsigned long       i_ino;         /* inode号 */
    union {
        const unsigned int i_nlink;    /* 硬链接计数 */
        unsigned int __i_nlink;
    };
    dev_t               i_rdev;        /* 设备号（设备文件） */
    loff_t              i_size;        /* 文件大小 */
    struct timespec64   i_atime;       /* 访问时间 */
    struct timespec64   i_mtime;       /* 修改时间 */
    struct timespec64   i_ctime;       /* 状态改变时间 */
    
    spinlock_t          i_lock;        /* 保护inode的自旋锁 */
    unsigned short      i_bytes;       /* 最后一个块的字节数 */
    u8                  i_blkbits;     /* 块大小的位数 */
    
    atomic_t            i_count;       /* 引用计数 */
    
    union {
        struct pipe_inode_info *i_pipe;
        struct block_device *i_bdev;
        struct cdev         *i_cdev;
    };
    
    const struct file_operations *i_fop;  /* 文件操作表 */
    struct file_lock_context *i_flctx;    /* 文件锁上下文 */
    struct address_space i_data;          /* 设备的地址空间 */
    struct list_head    i_devices;        /* 设备链表 */
    
    void                *i_private;       /* 文件系统私有数据 */
};
```

#### 2.2.3 目录项（struct dentry）

目录项是路径名的一个组成部分，它将文件名与对应的inode关联起来。目录项的设计是VFS性能优化的关键：

```c
struct dentry {
    unsigned int        d_flags;       /* 目录项标志 */
    seqcount_spinlock_t d_seq;         /* 用于RCU查找的序列计数器 */
    struct hlist_bl_node d_hash;       /* 哈希表链接 */
    struct dentry       *d_parent;     /* 父目录项 */
    struct qstr         d_name;        /* 文件名 */
    struct inode        *d_inode;      /* 关联的inode */
    
    unsigned char       d_iname[DNAME_INLINE_LEN]; /* 短文件名优化 */
    
    struct lockref      d_lockref;     /* 引用计数和自旋锁 */
    const struct dentry_operations *d_op;  /* 目录项操作表 */
    struct super_block  *d_sb;         /* 所属超级块 */
    unsigned long       d_time;        /* 重验证时间 */
    void                *d_fsdata;     /* 文件系统私有数据 */
    
    union {
        struct list_head d_lru;        /* LRU链表 */
        wait_queue_head_t *d_wait;     /* 等待队列 */
    };
    struct list_head    d_child;       /* 父目录的子项链表 */
    struct list_head    d_subdirs;     /* 子目录链表 */
    
    union {
        struct hlist_node d_alias;     /* inode别名链表 */
        struct hlist_bl_node d_in_lookup_hash; /* 查找哈希表 */
        struct rcu_head d_rcu;         /* RCU回收头 */
    } d_u;
};
```

目录项有三种状态：
- **使用中（in use）**：d_inode指向有效inode，引用计数>0
- **未使用（unused）**：d_inode指向有效inode，引用计数=0，保留在缓存中
- **负目录项（negative）**：d_inode为NULL，表示不存在的文件，用于加速查找失败

#### 2.2.4 文件对象（struct file）

文件对象代表进程打开的一个文件，包含文件的当前状态：

```c
struct file {
    union {
        struct llist_node   fu_llist;   /* 文件对象链表 */
        struct rcu_head     fu_rcuhead; /* RCU回收头 */
    } f_u;
    struct path         f_path;        /* 包含dentry和vfsmount */
    struct inode        *f_inode;      /* 缓存的inode指针 */
    const struct file_operations *f_op; /* 文件操作表 */
    
    spinlock_t          f_lock;        /* 保护文件对象的锁 */
    enum rw_hint        f_write_hint;  /* 写入提示 */
    atomic_long_t       f_count;       /* 引用计数 */
    unsigned int        f_flags;       /* 文件标志（O_RDONLY等） */
    fmode_t             f_mode;        /* 文件模式 */
    struct mutex        f_pos_lock;    /* 保护f_pos */
    loff_t              f_pos;         /* 文件位置 */
    struct fown_struct  f_owner;       /* 异步I/O的所有者 */
    const struct cred   *f_cred;       /* 打开文件时的凭证 */
    struct file_ra_state f_ra;         /* 预读状态 */
    
    u64                 f_version;     /* 版本号 */
    void                *private_data; /* 私有数据 */
    
    struct address_space *f_mapping;   /* 页缓存映射 */
    errseq_t            f_wb_err;      /* 写回错误 */
    errseq_t            f_sb_err;      /* 超级块错误 */
};
```

### 2.3 VFS操作表

VFS通过操作表（operations structure）实现多态性。每个对象都有对应的操作表：

1. **super_operations**：文件系统级操作（创建/删除inode、同步等）
2. **inode_operations**：inode级操作（创建、删除、查找等）
3. **dentry_operations**：目录项操作（比较、哈希、验证等）
4. **file_operations**：文件操作（读、写、打开、关闭等）
5. **address_space_operations**：地址空间操作（页面读写、内存映射等）

这种设计允许不同文件系统提供自己的实现，同时保持接口的一致性。

### 2.4 对象间的关系

VFS对象之间存在复杂的引用关系：

```
    super_block
         |
         | s_root
         ↓
      dentry (/)
         |
         | d_inode
         ↓
       inode
         |
         | i_mapping
         ↓
   address_space
   
   进程 → file → dentry → inode
             ↓
          f_path.mnt → vfsmount → super_block
```

这些关系确保了：
- 文件系统的层次结构
- 引用计数的正确管理
- 内存回收的安全性
- 并发访问的一致性

## 3. 文件系统注册与挂载

### 3.1 文件系统类型注册

Linux支持的每种文件系统都必须先注册到内核中。文件系统类型由`struct file_system_type`表示：

```c
struct file_system_type {
    const char *name;               /* 文件系统名称 */
    int fs_flags;                   /* 文件系统标志 */
#define FS_REQUIRES_DEV         1   /* 需要块设备 */
#define FS_BINARY_MOUNTDATA     2   /* 二进制挂载数据 */
#define FS_HAS_SUBTYPE          4   /* 有子类型 */
#define FS_USERNS_MOUNT         8   /* 可在用户命名空间挂载 */
#define FS_DISALLOW_NOTIFY_PERM 16  /* 禁用fanotify权限事件 */
    
    struct dentry *(*mount)(struct file_system_type *, int,
                           const char *, void *);
    void (*kill_sb)(struct super_block *);
    struct module *owner;           /* 所属模块 */
    struct file_system_type *next;  /* 链表中的下一个 */
    struct hlist_head fs_supers;    /* 该类型的超级块链表 */
    
    struct lock_class_key s_lock_key;
    struct lock_class_key s_umount_key;
    struct lock_class_key s_vfs_rename_key;
    struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];
    
    struct lock_class_key i_lock_key;
    struct lock_class_key i_mutex_key;
    struct lock_class_key i_mutex_dir_key;
};
```

文件系统注册通过`register_filesystem()`完成：

```c
int register_filesystem(struct file_system_type *fs)
{
    int res = 0;
    struct file_system_type **p;
    
    if (fs->next)
        return -EBUSY;
    
    write_lock(&file_systems_lock);
    p = find_filesystem(fs->name, strlen(fs->name));
    if (*p)
        res = -EBUSY;
    else
        *p = fs;
    write_unlock(&file_systems_lock);
    return res;
}
```

### 3.2 挂载过程分析

文件系统挂载是将存储设备上的文件系统连接到目录树的过程。挂载涉及以下关键步骤：

#### 3.2.1 挂载系统调用

```c
SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name,
                char __user *, type, unsigned long, flags, void __user *, data)
{
    return ksys_mount(dev_name, dir_name, type, flags, data);
}
```

#### 3.2.2 挂载过程的核心步骤

1. **路径查找**：找到挂载点的dentry
2. **文件系统查找**：根据类型名找到file_system_type
3. **超级块创建**：调用文件系统的mount方法创建超级块
4. **挂载连接**：创建mount结构，连接到挂载树

```c
struct mount {
    struct hlist_node mnt_hash;     /* 挂载哈希表 */
    struct mount *mnt_parent;        /* 父挂载点 */
    struct dentry *mnt_mountpoint;   /* 挂载点目录项 */
    struct vfsmount mnt;             /* VFS挂载信息 */
    
    union {
        struct rcu_head mnt_rcu;
        struct llist_node mnt_llist;
    };
    
    struct mnt_pcp __percpu *mnt_pcp;
    struct list_head mnt_mounts;    /* 子挂载链表 */
    struct list_head mnt_child;     /* 父挂载的子项 */
    struct list_head mnt_instance;  /* 挂载实例链表 */
    
    const char *mnt_devname;        /* 设备名 */
    struct list_head mnt_list;
    struct list_head mnt_expire;    /* 过期链表 */
    struct list_head mnt_share;     /* 共享挂载链表 */
    struct list_head mnt_slave_list;/* 从属挂载链表 */
    struct list_head mnt_slave;     /* 从属链表 */
    struct mount *mnt_master;        /* 主挂载点 */
    struct mnt_namespace *mnt_ns;   /* 挂载命名空间 */
    struct mountpoint *mnt_mp;      /* 挂载点 */
    union {
        struct hlist_node mnt_mp_list;
        struct hlist_node mnt_umount;
    };
    struct list_head mnt_umounting;
    struct fsnotify_mark_connector __rcu *mnt_fsnotify_marks;
    __u32 mnt_fsnotify_mask;
    int mnt_id;                     /* 挂载ID */
    int mnt_group_id;               /* 组ID */
    int mnt_expiry_mark;            /* 过期标记 */
    struct hlist_head mnt_pins;
    struct hlist_head mnt_stuck_children;
};
```

### 3.3 命名空间与挂载传播

Linux支持挂载命名空间，允许不同进程看到不同的文件系统视图：

#### 3.3.1 挂载命名空间

```c
struct mnt_namespace {
    atomic_t count;                  /* 引用计数 */
    struct ns_common ns;
    struct mount *root;              /* 根挂载点 */
    struct list_head list;           /* 挂载点链表 */
    struct user_namespace *user_ns;
    struct ucounts *ucounts;
    u64 seq;                         /* 序列号 */
    wait_queue_head_t poll;
    u64 event;
    unsigned int mounts;             /* 挂载点数量 */
    unsigned int pending_mounts;
};
```

#### 3.3.2 挂载传播类型

Linux支持四种挂载传播类型：

1. **MS_SHARED**：共享挂载，挂载/卸载操作会传播到同组其他挂载点
2. **MS_PRIVATE**：私有挂载，挂载/卸载操作不传播
3. **MS_SLAVE**：从属挂载，单向接收主挂载点的传播
4. **MS_UNBINDABLE**：不可绑定挂载，不能作为bind mount的源

传播机制的实现：

```c
static int propagate_one(struct mount *m)
{
    struct mount *child;
    int type;
    
    if (IS_MNT_NEW(m))
        return 0;
    
    /* 判断传播类型 */
    type = CL_SLAVE;
    if (IS_MNT_SHARED(m))
        type |= CL_MAKE_SHARED;
    
    /* 复制挂载 */
    child = copy_tree(source, source->mnt.mnt_root, type);
    if (IS_ERR(child))
        return PTR_ERR(child);
    
    /* 附加到目标 */
    mnt_set_mountpoint(m, mp, child);
    child->mnt_parent = m;
    child->mnt_master = m->mnt_master;
    
    /* 加入挂载哈希表 */
    list_add_tail(&child->mnt_hash, tree_list);
    return count_mounts(m->mnt_ns, child);
}
```

### 3.4 根文件系统的特殊处理

根文件系统是系统启动时第一个挂载的文件系统，具有特殊的初始化过程：

#### 3.4.1 早期根文件系统（initramfs）

```c
static int __init populate_rootfs(void)
{
    char *err;
    
    /* 解压内置的initramfs */
    err = unpack_to_rootfs(__initramfs_start,
                           __initramfs_size);
    if (err)
        panic("Failed to unpack initramfs: %s\n", err);
    
    /* 如果有外部initrd */
    if (initrd_start) {
        #ifdef CONFIG_BLK_DEV_RAM
        /* 创建/initrd.image */
        int fd = ksys_open("/initrd.image",
                          O_WRONLY|O_CREAT, 0700);
        if (fd >= 0) {
            ksys_write(fd, (char *)initrd_start,
                      initrd_end - initrd_start);
            ksys_close(fd);
        }
        #else
        /* 直接解压 */
        err = unpack_to_rootfs((char *)initrd_start,
                               initrd_end - initrd_start);
        #endif
    }
    return 0;
}
rootfs_initcall(populate_rootfs);
```

#### 3.4.2 真实根文件系统切换

从initramfs切换到真实根文件系统的过程：

```c
void __init prepare_namespace(void)
{
    if (root_delay) {
        printk(KERN_INFO "Waiting %d sec before mounting root device...\n",
               root_delay);
        ssleep(root_delay);
    }
    
    /* 等待根设备出现 */
    wait_for_device_probe();
    
    /* 挂载根文件系统 */
    mount_root();
    
    /* 切换根目录 */
    ksys_chdir("/root");
    mount_block_root();
    mount_root_generic();
    
    /* 移动挂载点 */
    ksys_mount(".", "/", NULL, MS_MOVE, NULL);
    ksys_chroot(".");
}
```

## 4. 路径名查找与目录项缓存

路径名查找是VFS最频繁的操作之一，其性能直接影响系统整体性能。Linux通过目录项缓存（dcache）和RCU优化实现了高效的路径查找。

### 4.1 路径解析算法

路径名解析将文件路径转换为对应的dentry和inode。核心函数是`path_lookupat()`：

```c
static int path_lookupat(struct nameidata *nd, unsigned flags, struct path *path)
{
    const char *s = path_init(nd, flags);
    int err;
    
    if (IS_ERR(s))
        return PTR_ERR(s);
    
    while (!(err = link_path_walk(s, nd)) &&
           (s = lookup_last(nd)) != NULL)
        ;
    
    if (!err) {
        *path = nd->path;
        nd->path.mnt = NULL;
        nd->path.dentry = NULL;
    }
    
    terminate_walk(nd);
    return err;
}
```

#### 4.1.1 路径查找数据结构

```c
struct nameidata {
    struct path     path;           /* 当前路径 */
    struct qstr     last;           /* 最后一个组件 */
    struct path     root;           /* 根目录 */
    struct inode    *inode;         /* 当前inode */
    unsigned int    flags;          /* 查找标志 */
    unsigned        seq;            /* 序列号（RCU） */
    int             last_type;      /* 最后组件类型 */
    unsigned        depth;          /* 符号链接深度 */
    int             total_link_count; /* 总链接数 */
    struct saved {
        struct path link;
        struct delayed_call done;
        const char *name;
        unsigned seq;
    } *stack, internal[EMBEDDED_LEVELS];
    struct filename *name;
    struct nameidata *saved;
    unsigned        root_seq;
    int             dfd;
    kuid_t          dir_uid;
    umode_t         dir_mode;
};
```

#### 4.1.2 路径遍历核心算法

```c
static int link_path_walk(const char *name, struct nameidata *nd)
{
    int err;
    
    if (IS_ERR(name))
        return PTR_ERR(name);
    
    while (*name == '/')
        name++;
    if (!*name)
        return 0;
    
    for(;;) {
        u64 hash_len;
        int type;
        
        err = may_lookup(nd);
        if (err)
            return err;
        
        hash_len = hash_name(nd->path.dentry, name);
        
        type = LAST_NORM;
        if (name[0] == '.') switch (hashlen_len(hash_len)) {
            case 2:
                if (name[1] == '.') {
                    type = LAST_DOTDOT;
                    nd->flags |= LOOKUP_JUMPED;
                }
                break;
            case 1:
                type = LAST_DOT;
        }
        
        if (likely(type == LAST_NORM)) {
            struct dentry *parent = nd->path.dentry;
            nd->flags &= ~LOOKUP_JUMPED;
            if (unlikely(parent->d_flags & DCACHE_OP_HASH)) {
                struct qstr this = { { .hash_len = hash_len }, .name = name };
                err = parent->d_op->d_hash(parent, &this);
                if (err < 0)
                    return err;
                hash_len = this.hash_len;
                name = this.name;
            }
        }
        
        nd->last.hash_len = hash_len;
        nd->last.name = name;
        nd->last_type = type;
        
        name += hashlen_len(hash_len);
        if (!*name)
            return 0;
        
        do {
            name++;
        } while (unlikely(*name == '/'));
        if (unlikely(!*name)) {
            nd->flags |= LOOKUP_DIRECTORY | LOOKUP_FOLLOW | LOOKUP_DOTDOT;
            nd->last_type = LAST_ROOT;
            return 0;
        }
        
        err = walk_component(nd, WALK_MORE);
        if (err < 0)
            return err;
        
        if (err) {
            const char *s = get_link(nd);
            if (IS_ERR(s))
                return PTR_ERR(s);
            if (likely(s)) {
                nd->stack[nd->depth - 1].name = name;
                name = s;
                continue;
            }
        }
    }
}
```

### 4.2 目录项缓存（dcache）机制

dcache是VFS性能的关键，它缓存了最近使用的目录项，避免重复的磁盘访问。

#### 4.2.1 dcache哈希表

dcache使用全局哈希表快速查找目录项：

```c
static struct hlist_bl_head *dentry_hashtable __read_mostly;

static inline struct hlist_bl_head *d_hash(unsigned int hash)
{
    return dentry_hashtable + (hash >> (32 - d_hash_shift));
}

static inline unsigned int d_hash_name(const struct dentry *parent,
                                       const struct qstr *name)
{
    unsigned int hash = init_name_hash(parent);
    unsigned int len = name->len;
    const unsigned char *str = name->name;
    
    while (len--)
        hash = partial_name_hash(*str++, hash);
    
    return end_name_hash(hash);
}
```

#### 4.2.2 dcache查找

```c
struct dentry *__d_lookup(const struct dentry *parent, const struct qstr *name)
{
    unsigned int hash = d_hash_name(parent, name);
    struct hlist_bl_head *b = d_hash(hash);
    struct hlist_bl_node *node;
    struct dentry *dentry;
    
    rcu_read_lock();
    hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
        if (dentry->d_name.hash != hash)
            continue;
        
        spin_lock(&dentry->d_lock);
        if (dentry->d_parent != parent)
            goto next;
        if (d_unhashed(dentry))
            goto next;
        
        if (!d_same_name(dentry, parent, name))
            goto next;
        
        dentry->d_lockref.count++;
        found = dentry;
        spin_unlock(&dentry->d_lock);
        break;
next:
        spin_unlock(&dentry->d_lock);
    }
    rcu_read_unlock();
    
    return found;
}
```

### 4.3 RCU路径遍历优化

RCU（Read-Copy-Update）允许无锁的路径查找，极大提升了并发性能。

#### 4.3.1 RCU模式的路径查找

```c
static int lookup_fast(struct nameidata *nd,
                      struct path *path, struct inode **inode,
                      unsigned *seqp)
{
    struct vfsmount *mnt = nd->path.mnt;
    struct dentry *dentry, *parent = nd->path.dentry;
    int status = 1;
    
    if (nd->flags & LOOKUP_RCU) {
        unsigned seq;
        dentry = __d_lookup_rcu(parent, &nd->last, &seq);
        if (unlikely(!dentry)) {
            if (unlazy_walk(nd))
                return -ECHILD;
            return 0;
        }
        
        *inode = d_backing_inode(dentry);
        if (unlikely(read_seqcount_retry(&dentry->d_seq, seq)))
            return -ECHILD;
        
        if (unlikely(__read_seqcount_retry(&parent->d_seq, nd->seq)))
            return -ECHILD;
        
        *seqp = seq;
        status = d_revalidate(dentry, nd->flags);
        if (likely(status > 0)) {
            path->mnt = mnt;
            path->dentry = dentry;
            if (unlikely(!*inode))
                return -ENOENT;
            if (likely(__follow_mount_rcu(nd, path, inode, seqp)))
                return 1;
        }
        if (unlazy_child(nd, dentry, seq))
            return -ECHILD;
        if (unlikely(status == -ECHILD))
            status = d_revalidate(dentry, nd->flags);
    } else {
        dentry = __d_lookup(parent, &nd->last);
        if (unlikely(!dentry))
            return 0;
        status = d_revalidate(dentry, nd->flags);
    }
    
    if (unlikely(status <= 0)) {
        if (!status)
            d_invalidate(dentry);
        dput(dentry);
        return status;
    }
    
    path->mnt = mnt;
    path->dentry = dentry;
    return 1;
}
```

#### 4.3.2 序列锁保护

RCU模式使用序列锁检测并发修改：

```c
static inline int read_seqcount_retry(const seqcount_t *s, unsigned start)
{
    smp_rmb();
    return unlikely(s->sequence != start);
}

static inline unsigned read_seqbegin(const seqlock_t *sl)
{
    unsigned ret = READ_ONCE(sl->sequence);
    if (unlikely(ret & 1)) {
        cpu_relax();
        goto repeat;
    }
    smp_rmb();
    return ret;
}
```

### 4.4 符号链接处理

符号链接的处理需要特别注意避免无限循环：

```c
static const char *get_link(struct nameidata *nd)
{
    struct saved *last = nd->stack + nd->depth - 1;
    struct dentry *dentry = last->link.dentry;
    struct inode *inode = nd->link_inode;
    int error;
    const char *res;
    
    if (unlikely(nd->total_link_count >= MAXSYMLINKS)) {
        path_put(&last->link);
        return ERR_PTR(-ELOOP);
    }
    
    nd->total_link_count++;
    
    touch_atime(&last->link);
    
    error = security_inode_follow_link(dentry, inode,
                                       nd->flags & LOOKUP_RCU);
    if (unlikely(error))
        return ERR_PTR(error);
    
    res = READ_ONCE(inode->i_link);
    if (!res) {
        const char * (*get)(struct dentry *, struct inode *,
                           struct delayed_call *);
        get = inode->i_op->get_link;
        if (nd->flags & LOOKUP_RCU) {
            res = get(NULL, inode, &last->done);
            if (res == ERR_PTR(-ECHILD)) {
                if (unlazy_walk(nd))
                    return ERR_PTR(-ECHILD);
                res = get(dentry, inode, &last->done);
            }
        } else {
            res = get(dentry, inode, &last->done);
        }
        if (IS_ERR_OR_NULL(res))
            return res;
    }
    
    if (*res == '/') {
        error = nd_jump_root(nd);
        if (unlikely(error))
            return ERR_PTR(error);
    }
    
    return res;
}
```

### 4.5 负目录项优化

负目录项（negative dentry）缓存不存在的文件，避免重复的查找失败：

```c
static void d_instantiate_negative(struct dentry *dentry, struct inode *inode)
{
    BUG_ON(dentry->d_inode);
    BUG_ON(!hlist_unhashed(&dentry->d_u.d_alias));
    
    spin_lock(&dentry->d_lock);
    dentry->d_inode = NULL;
    dentry->d_flags |= DCACHE_NEGATIVE;
    spin_unlock(&dentry->d_lock);
}
```

负目录项的好处：
1. 减少对不存在文件的重复查找
2. 加速编译等需要搜索大量路径的操作
3. 提高include路径搜索效率

## 5. 文件操作接口实现

VFS通过系统调用为用户空间提供文件操作接口。这些系统调用最终转换为对具体文件系统的操作。

### 5.1 系统调用到VFS的映射

#### 5.1.1 open系统调用

```c
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
    if (force_o_largefile())
        flags |= O_LARGEFILE;
    return do_sys_open(AT_FDCWD, filename, flags, mode);
}

long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
    struct open_how how = build_open_how(flags, mode);
    return do_sys_openat2(dfd, filename, &how);
}

static long do_sys_openat2(int dfd, const char __user *filename,
                           struct open_how *how)
{
    struct filename *tmp;
    int fd;
    
    tmp = getname(filename);
    if (IS_ERR(tmp))
        return PTR_ERR(tmp);
    
    fd = get_unused_fd_flags(how->flags);
    if (fd >= 0) {
        struct file *f = do_filp_open(dfd, tmp, &how);
        if (IS_ERR(f)) {
            put_unused_fd(fd);
            fd = PTR_ERR(f);
        } else {
            fsnotify_open(f);
            fd_install(fd, f);
        }
    }
    putname(tmp);
    return fd;
}
```

#### 5.1.2 文件打开的核心过程

```c
struct file *do_filp_open(int dfd, struct filename *pathname,
                         const struct open_flags *op)
{
    struct nameidata nd;
    int flags = op->lookup_flags;
    struct file *filp;
    
    set_nameidata(&nd, dfd, pathname);
    filp = path_openat(&nd, op, flags | LOOKUP_RCU);
    if (unlikely(filp == ERR_PTR(-ECHILD)))
        filp = path_openat(&nd, op, flags);
    if (unlikely(filp == ERR_PTR(-ESTALE)))
        filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
    restore_nameidata();
    return filp;
}

static struct file *path_openat(struct nameidata *nd,
                               const struct open_flags *op, unsigned flags)
{
    struct file *file;
    int error;
    
    file = alloc_empty_file(op->open_flag, current_cred());
    if (IS_ERR(file))
        return file;
    
    if (unlikely(file->f_flags & __O_TMPFILE)) {
        error = do_tmpfile(nd, flags, op, file);
    } else if (unlikely(file->f_flags & O_PATH)) {
        error = do_o_path(nd, flags, file);
    } else {
        const char *s = path_init(nd, flags);
        while (!(error = link_path_walk(s, nd)) &&
               (s = open_last_lookups(nd, file, op)) != NULL)
            ;
        if (!error)
            error = do_open(nd, file, op);
        terminate_walk(nd);
    }
    
    if (likely(!error)) {
        if (likely(file->f_mode & FMODE_OPENED))
            return file;
        WARN_ON(1);
        error = -EINVAL;
    }
    
    fput(file);
    if (error == -EOPENSTALE) {
        if (flags & LOOKUP_RCU)
            error = -ECHILD;
        else
            error = -ESTALE;
    }
    return ERR_PTR(error);
}
```

### 5.2 文件描述符管理

#### 5.2.1 文件描述符表

每个进程维护一个文件描述符表：

```c
struct files_struct {
    atomic_t count;
    bool resize_in_progress;
    wait_queue_head_t resize_wait;
    
    struct fdtable __rcu *fdt;
    struct fdtable fdtab;
    
    spinlock_t file_lock ____cacheline_aligned_in_smp;
    unsigned int next_fd;
    unsigned long close_on_exec_init[1];
    unsigned long open_fds_init[1];
    unsigned long full_fds_bits_init[1];
    struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

struct fdtable {
    unsigned int max_fds;
    struct file __rcu **fd;
    unsigned long *close_on_exec;
    unsigned long *open_fds;
    unsigned long *full_fds_bits;
    struct rcu_head rcu;
};
```

#### 5.2.2 分配文件描述符

```c
int get_unused_fd_flags(unsigned flags)
{
    return __get_unused_fd_flags(flags, 0, rlimit(RLIMIT_NOFILE));
}

int __get_unused_fd_flags(unsigned flags, unsigned long nofile)
{
    struct files_struct *files = current->files;
    int fd;
    
    spin_lock(&files->file_lock);
repeat:
    fd = find_next_fd(files, files->next_fd);
    
    if (fd >= nofile) {
        fd = -EMFILE;
        goto out;
    }
    
    error = expand_files(files, fd);
    if (error < 0)
        goto out;
    
    if (error)
        goto repeat;
    
    __set_open_fd(fd, files->fdt);
    if (flags & O_CLOEXEC)
        __set_close_on_exec(fd, files->fdt);
    else
        __clear_close_on_exec(fd, files->fdt);
    
    files->next_fd = fd + 1;
    
out:
    spin_unlock(&files->file_lock);
    return fd;
}
```

### 5.3 读写操作路径

#### 5.3.1 read系统调用

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
    return ksys_read(fd, buf, count);
}

ssize_t ksys_read(unsigned int fd, char __user *buf, size_t count)
{
    struct fd f = fdget_pos(fd);
    ssize_t ret = -EBADF;
    
    if (f.file) {
        loff_t pos, *ppos = file_ppos(f.file);
        if (ppos) {
            pos = *ppos;
            ppos = &pos;
        }
        ret = vfs_read(f.file, buf, count, ppos);
        if (ret >= 0 && ppos)
            f.file->f_pos = pos;
        fdput_pos(f);
    }
    return ret;
}
```

#### 5.3.2 VFS读操作

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    ssize_t ret;
    
    if (!(file->f_mode & FMODE_READ))
        return -EBADF;
    if (!(file->f_mode & FMODE_CAN_READ))
        return -EINVAL;
    if (unlikely(!access_ok(buf, count)))
        return -EFAULT;
    
    ret = rw_verify_area(READ, file, pos, count);
    if (ret)
        return ret;
    
    if (count > MAX_RW_COUNT)
        count = MAX_RW_COUNT;
    
    if (file->f_op->read)
        ret = file->f_op->read(file, buf, count, pos);
    else if (file->f_op->read_iter)
        ret = new_sync_read(file, buf, count, pos);
    else
        ret = -EINVAL;
    
    if (ret > 0) {
        fsnotify_access(file);
        add_rchar(current, ret);
    }
    inc_syscr(current);
    return ret;
}
```

#### 5.3.3 写操作实现

```c
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
    ssize_t ret;
    
    if (!(file->f_mode & FMODE_WRITE))
        return -EBADF;
    if (!(file->f_mode & FMODE_CAN_WRITE))
        return -EINVAL;
    if (unlikely(!access_ok(buf, count)))
        return -EFAULT;
    
    ret = rw_verify_area(WRITE, file, pos, count);
    if (ret)
        return ret;
    
    if (count > MAX_RW_COUNT)
        count = MAX_RW_COUNT;
    
    file_start_write(file);
    if (file->f_op->write)
        ret = file->f_op->write(file, buf, count, pos);
    else if (file->f_op->write_iter)
        ret = new_sync_write(file, buf, count, pos);
    else
        ret = -EINVAL;
    
    if (ret > 0) {
        fsnotify_modify(file);
        add_wchar(current, ret);
    }
    inc_syscw(current);
    file_end_write(file);
    return ret;
}
```

### 5.4 内存映射（mmap）支持

#### 5.4.1 mmap系统调用

```c
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
                unsigned long, prot, unsigned long, flags,
                unsigned long, fd, unsigned long, off)
{
    if (offset_in_page(off) != 0)
        return -EINVAL;
    
    return ksys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
}

unsigned long ksys_mmap_pgoff(unsigned long addr, unsigned long len,
                              unsigned long prot, unsigned long flags,
                              unsigned long fd, unsigned long pgoff)
{
    struct file *file = NULL;
    unsigned long retval;
    
    if (!(flags & MAP_ANONYMOUS)) {
        file = fget(fd);
        if (!file)
            return -EBADF;
        if (is_file_hugepages(file)) {
            len = ALIGN(len, huge_page_size(hstate_file(file)));
        } else if (unlikely(flags & MAP_HUGETLB)) {
            retval = -EINVAL;
            goto out_fput;
        }
    } else if (flags & MAP_HUGETLB) {
        struct ucounts *ucounts = NULL;
        file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
                                 VM_NORESERVE,
                                 &ucounts, HUGETLB_ANONHUGE_INODE,
                                 (flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
        if (IS_ERR(file))
            return PTR_ERR(file);
    }
    
    retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
    
out_fput:
    if (file)
        fput(file);
    return retval;
}
```

#### 5.4.2 文件映射的VFS支持

```c
static int do_mmap(struct file *file, unsigned long addr,
                  unsigned long len, unsigned long prot,
                  unsigned long flags, unsigned long pgoff,
                  unsigned long *populate, struct list_head *uf)
{
    struct mm_struct *mm = current->mm;
    vm_flags_t vm_flags;
    int pkey = 0;
    
    *populate = 0;
    
    if (!len)
        return -EINVAL;
    
    if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
        if (!(file && path_noexec(&file->f_path)))
            prot |= PROT_EXEC;
    
    if (!(flags & MAP_FIXED))
        addr = round_hint_to_min(addr);
    
    len = PAGE_ALIGN(len);
    if (!len)
        return -ENOMEM;
    
    if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
        return -EOVERFLOW;
    
    if (mm->map_count > sysctl_max_map_count)
        return -ENOMEM;
    
    addr = get_unmapped_area(file, addr, len, pgoff, flags);
    if (IS_ERR_VALUE(addr))
        return addr;
    
    if (flags & MAP_FIXED_NOREPLACE) {
        if (find_vma_intersection(mm, addr, addr + len))
            return -EEXIST;
    }
    
    if (prot == PROT_EXEC) {
        pkey = execute_only_pkey(mm);
        if (pkey < 0)
            pkey = 0;
    }
    
    vm_flags = calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(flags) |
               mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;
    
    if (file) {
        struct inode *inode = file_inode(file);
        unsigned long flags_mask = VM_MERGEABLE | VM_MAYSHARE |
                                   VM_DONTEXPAND | VM_SOFTDIRTY;
        
        if (!file_mmap_ok(file, inode, pgoff, len))
            return -EOVERFLOW;
        
        flags_mask |= file->f_op->mmap_supported_flags;
        
        switch (flags & MAP_TYPE) {
        case MAP_SHARED:
            flags_mask |= VM_SHARED | VM_MAYSHARE;
            break;
        case MAP_SHARED_VALIDATE:
            flags_mask |= VM_SHARED | VM_MAYSHARE;
            if (flags & ~flags_mask)
                return -EOPNOTSUPP;
            break;
        case MAP_PRIVATE:
            break;
        default:
            return -EINVAL;
        }
        
        if (!file->f_op->mmap)
            return -ENODEV;
        if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
            return -EINVAL;
    }
    
    addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
    if (!IS_ERR_VALUE(addr) &&
        ((vm_flags & VM_LOCKED) ||
         (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
        *populate = len;
    
    return addr;
}
```