# 第4章：进程间通信机制

进程间通信（IPC）是操作系统的核心功能之一，它使得独立的进程能够交换数据、同步执行和协调资源访问。Linux 内核提供了丰富的 IPC 机制，从经典的 Unix 管道到现代的 futex 和 io_uring，每种机制都有其独特的设计哲学和适用场景。本章将深入剖析 Linux IPC 机制的内核实现，揭示其背后的算法原理、性能特征和演进历程。

## 学习目标

完成本章学习后，您将能够：

1. **理解传统 IPC 机制**：掌握管道、消息队列、共享内存的内核实现和数据结构
2. **分析信号系统架构**：理解信号的产生、投递、处理全流程，包括实时信号扩展
3. **掌握 futex 原理**：深入理解用户态/内核态混合同步机制，包括 PI-futex 和 robust futex
4. **使用现代 IPC 接口**：熟练运用 eventfd、signalfd、timerfd 构建高效事件驱动程序
5. **优化 IPC 性能**：根据场景选择合适的 IPC 机制，实现零拷贝和 NUMA 优化
6. **理解前沿技术**：掌握 io_uring、RDMA 等新一代高性能通信机制

## 4.1 传统 IPC 机制

Linux 内核继承并扩展了 Unix 的传统 IPC 机制。这些机制虽然历史悠久，但在现代系统中仍然扮演着重要角色。理解它们的实现原理对于系统编程和性能优化至关重要。

### 4.1.1 管道（Pipe）机制深度剖析

#### 匿名管道实现原理

管道是 Unix 最早的 IPC 机制之一，其优雅的设计体现了 "一切皆文件" 的哲学。在 Linux 内核中，管道通过特殊的文件系统 pipefs 实现。

**核心数据结构**

```c
// fs/pipe.c
struct pipe_inode_info {
    struct mutex mutex;           // 保护管道状态的互斥锁
    wait_queue_head_t rd_wait;    // 读者等待队列
    wait_queue_head_t wr_wait;    // 写者等待队列
    unsigned int head;            // 环形缓冲区头部
    unsigned int tail;            // 环形缓冲区尾部
    unsigned int max_usage;       // 最大缓冲区数量
    unsigned int ring_size;       // 环形缓冲区大小（必须是2的幂）
    unsigned int readers;         // 读者计数
    unsigned int writers;         // 写者计数
    unsigned int files;           // 引用计数
    unsigned int r_counter;       // 读计数器（用于 poll）
    unsigned int w_counter;       // 写计数器（用于 poll）
    struct page *tmp_page;        // 临时页面（优化小数据传输）
    struct fasync_struct *fasync_readers;  // 异步通知读者
    struct fasync_struct *fasync_writers;  // 异步通知写者
    struct pipe_buffer *bufs;     // 环形缓冲区数组
    struct user_struct *user;     // 创建者用户结构
};

struct pipe_buffer {
    struct page *page;            // 数据页
    unsigned int offset;          // 页内偏移
    unsigned int len;             // 有效数据长度
    const struct pipe_buf_operations *ops;  // 缓冲区操作函数
    unsigned int flags;           // 标志位
    unsigned long private;        // 私有数据
};
```

**管道创建流程**

当进程调用 `pipe()` 或 `pipe2()` 系统调用时，内核执行以下步骤：

1. **分配 pipe_inode_info 结构**：初始化管道的元数据
2. **创建两个 file 结构**：分别用于读端和写端
3. **设置文件操作函数**：读端使用 `read_pipe_fops`，写端使用 `write_pipe_fops`
4. **分配文件描述符**：将两个 fd 返回给用户空间

```c
// fs/pipe.c 简化版
static int do_pipe2(int __user *fildes, int flags)
{
    struct file *files[2];
    int fd[2];
    int error;

    error = __do_pipe_flags(fd, files, flags);
    if (!error) {
        // 将文件描述符复制到用户空间
        if (copy_to_user(fildes, fd, sizeof(fd))) {
            // 错误处理
            fput(files[0]);
            fput(files[1]);
            put_unused_fd(fd[0]);
            put_unused_fd(fd[1]);
            error = -EFAULT;
        } else {
            // 安装文件描述符
            fd_install(fd[0], files[0]);
            fd_install(fd[1], files[1]);
        }
    }
    return error;
}
```

**环形缓冲区管理**

 Linux 管道使用环形缓冲区（circular buffer）管理数据，这种设计有以下优势：

1. **空间效率**：避免数据移动，通过调整指针实现循环使用
2. **时间效率**：O(1) 的入队和出队操作
3. **缓存友好**：连续的内存访问模式

```
环形缓冲区示意图：
        head
          |
          v
    +---+---+---+---+---+---+---+---+
    | D | E |   |   |   | A | B | C |
    +---+---+---+---+---+---+---+---+
              ^               ^
              |               |
           tail         wrapped data
```

**读写操作实现**

管道的读写操作涉及复杂的同步机制：

```c
// 简化的管道写操作
static ssize_t
pipe_write(struct kiocb *iocb, struct iov_iter *from)
{
    struct file *filp = iocb->ki_filp;
    struct pipe_inode_info *pipe = filp->private_data;
    unsigned int head, tail, max_usage, mask;
    ssize_t ret = 0;
    size_t total_len = iov_iter_count(from);
    ssize_t chars;
    bool was_empty = false;

    mutex_lock(&pipe->mutex);

    // 检查是否有读者
    if (!pipe->readers) {
        send_sig(SIGPIPE, current, 0);
        ret = -EPIPE;
        goto out;
    }

    head = pipe->head;
    tail = pipe->tail;
    max_usage = pipe->max_usage;
    mask = pipe->ring_size - 1;

    // 计算可用空间
    if (!pipe_full(head, tail, max_usage)) {
        unsigned int head_buf = head & mask;
        struct pipe_buffer *buf = &pipe->bufs[head_buf];
        
        // 分配页面并复制数据
        buf->page = alloc_page(GFP_HIGHUSER | __GFP_ACCOUNT);
        if (!buf->page) {
            ret = -ENOMEM;
            goto out;
        }
        
        // 从用户空间复制数据
        chars = copy_page_from_iter(buf->page, 0, PAGE_SIZE, from);
        buf->offset = 0;
        buf->len = chars;
        
        pipe->head = head + 1;
        ret += chars;
        
        // 唤醒等待的读者
        if (was_empty) {
            wake_up_interruptible_sync_poll(&pipe->rd_wait, EPOLLIN);
            kill_fasync(&pipe->fasync_readers, SIGIO, POLL_IN);
        }
    }

out:
    mutex_unlock(&pipe->mutex);
    return ret;
}
```

#### 命名管道（FIFO）

命名管道通过文件系统提供持久化的 IPC 通道，允许无亲缘关系的进程通信。

**FIFO 创建与打开**

```c
// fs/pipe.c
static int fifo_open(struct inode *inode, struct file *filp)
{
    struct pipe_inode_info *pipe;
    bool is_pipe = inode->i_sb->s_magic == PIPEFS_MAGIC;
    int ret;

    filp->f_version = 0;

    spin_lock(&inode->i_lock);
    if (inode->i_pipe) {
        pipe = inode->i_pipe;
        pipe->files++;
        spin_unlock(&inode->i_lock);
    } else {
        spin_unlock(&inode->i_lock);
        pipe = alloc_pipe_info();
        if (!pipe)
            return -ENOMEM;
        pipe->files = 1;
        spin_lock(&inode->i_lock);
        if (unlikely(inode->i_pipe)) {
            // 另一个进程已经创建了管道
            inode->i_pipe->files++;
            spin_unlock(&inode->i_lock);
            free_pipe_info(pipe);
            pipe = inode->i_pipe;
        } else {
            inode->i_pipe = pipe;
            spin_unlock(&inode->i_lock);
        }
    }
    filp->private_data = pipe;

    // 根据打开模式更新读者/写者计数
    mutex_lock(&pipe->mutex);
    if (filp->f_mode & FMODE_READ)
        pipe->readers++;
    if (filp->f_mode & FMODE_WRITE)
        pipe->writers++;
    mutex_unlock(&pipe->mutex);

    return 0;
}
```

#### splice 和 tee 系统调用

splice 和 tee 是 Linux 2.6.17 引入的零拷贝数据传输机制，大幅提升了管道的性能。

**splice 原理**

splice 通过在内核空间移动页面引用而非复制数据，实现高效的数据传输：

```c
// fs/splice.c 核心逻辑
ASMLINKAGE long sys_splice(int fd_in, loff_t __user *off_in,
                           int fd_out, loff_t __user *off_out,
                           size_t len, unsigned int flags)
{
    struct fd in, out;
    long error;

    if (unlikely(!len))
        return 0;

    error = -EBADF;
    in = fdget(fd_in);
    if (in.file) {
        out = fdget(fd_out);
        if (out.file) {
            error = do_splice(in.file, off_in, out.file, off_out,
                            len, flags);
            fdput(out);
        }
        fdput(in);
    }
    return error;
}
```

**零拷贝数据流**

```
传统方式（4次拷贝）：
磁盘 → 内核缓冲区 → 用户缓冲区 → 内核缓冲区 → 网络

splice方式（2次拷贝）：
磁盘 → 内核缓冲区 → 网络
         ^
         |
    仅移动页面引用
```

### 4.1.2 System V IPC 机制详解

System V IPC 是 AT&T 在 System V Release 3 中引入的三种 IPC 机制的统称，包括消息队列、共享内存和信号量。尽管 POSIX 试图提供更现代的替代方案，System V IPC 因其广泛的应用仍然是 Linux 内核的重要组成部分。

#### 通用架构与数据结构

System V IPC 的三种机制共享相似的架构设计：

```c
// include/linux/ipc.h
struct kern_ipc_perm {
    spinlock_t      lock;
    bool            deleted;
    int             id;             // IPC 标识符
    key_t           key;            // IPC 键值
    kuid_t          uid;            // 所有者 UID
    kgid_t          gid;            // 所有者 GID
    kuid_t          cuid;           // 创建者 UID
    kgid_t          cgid;           // 创建者 GID
    umode_t         mode;           // 访问权限
    unsigned long   seq;            // 序列号
    void            *security;      // LSM 安全标签
    struct rhash_head khtnode;      // 哈希表节点
    struct rcu_head rcu;            // RCU 头
    refcount_t      refcount;       // 引用计数
} ____cacheline_aligned_in_smp;

// IPC 命名空间结构
struct ipc_namespace {
    refcount_t      count;
    struct ipc_ids  ids[3];        // 三种 IPC 机制的 ID 表
    
    int             sem_ctls[4];    // 信号量限制参数
    int             used_sems;      // 已使用的信号量数
    
    unsigned int    msg_ctlmax;     // 消息最大字节数
    unsigned int    msg_ctlmnb;     // 队列最大字节数
    unsigned int    msg_ctlmni;     // 最大队列数
    atomic_t        msg_bytes;      // 当前消息字节总数
    atomic_t        msg_hdrs;       // 当前消息头总数
    
    size_t          shm_ctlmax;     // 共享内存段最大大小
    size_t          shm_ctlall;     // 共享内存总大小限制
    unsigned long   shm_tot;        // 当前共享内存页数
    int             shm_ctlmni;     // 最大共享内存段数
    int             shm_rmid_forced;// 强制删除标志
    
    struct notifier_block ipcns_nb;
    struct vfsmount *mq_mnt;        // POSIX 消息队列挂载点
    
    unsigned int    mq_queues_count;
    unsigned int    mq_queues_max;
    unsigned int    mq_msg_max;
    unsigned int    mq_msgsize_max;
    unsigned int    mq_msg_default;
    unsigned int    mq_msgsize_default;
    
    struct user_namespace *user_ns;
    struct ucounts  *ucounts;
    
    struct llist_node async_free_work;
    struct work_struct free_work;
} __randomize_layout;
```

#### 消息队列（Message Queue）

消息队列提供了一种进程间传递格式化数据的机制，每个消息都有类型标识，接收者可以选择性地接收特定类型的消息。

**核心数据结构**

```c
// ipc/msg.c
struct msg_queue {
    struct kern_ipc_perm q_perm;
    time64_t q_stime;           // 最后发送时间
    time64_t q_rtime;           // 最后接收时间
    time64_t q_ctime;           // 最后修改时间
    unsigned long q_cbytes;     // 队列中当前字节数
    unsigned long q_qnum;       // 队列中消息数
    unsigned long q_qbytes;     // 队列最大字节数
    struct pid *q_lspid;        // 最后发送者 PID
    struct pid *q_lrpid;        // 最后接收者 PID
    
    struct list_head q_messages;    // 消息链表
    struct list_head q_receivers;   // 接收者等待队列
    struct list_head q_senders;     // 发送者等待队列
} __randomize_layout;

struct msg_msg {
    struct list_head m_list;
    long m_type;                // 消息类型
    size_t m_ts;                // 消息大小
    struct msg_msgseg *next;    // 大消息的下一段
    void *security;             // 安全标签
    // 消息数据紧随其后
};

// 大消息分段结构
struct msg_msgseg {
    struct msg_msgseg *next;
    // 分段数据紧随其后
};
```

**消息发送流程**

```c
// 简化的 msgsnd 实现
static long do_msgsnd(int msqid, long mtype, void __user *mtext,
                     size_t msgsz, int msgflg)
{
    struct msg_queue *msq;
    struct msg_msg *msg;
    int err;
    
    // 分配消息结构
    msg = load_msg(mtext, msgsz);
    if (IS_ERR(msg))
        return PTR_ERR(msg);
    
    msg->m_type = mtype;
    msg->m_ts = msgsz;
    
    // 查找消息队列
    msq = msq_obtain_object_check(ns, msqid);
    if (IS_ERR(msq)) {
        err = PTR_ERR(msq);
        goto out_free;
    }
    
    ipc_lock_object(&msq->q_perm);
    
    // 检查权限
    err = -EACCES;
    if (ipcperms(ns, &msq->q_perm, S_IWUGO))
        goto out_unlock;
    
    // 检查队列空间
    if (msgsz + msq->q_cbytes > msq->q_qbytes) {
        if (msgflg & IPC_NOWAIT) {
            err = -EAGAIN;
            goto out_unlock;
        }
        // 阻塞等待空间
        // ...
    }
    
    // 将消息加入队列
    list_add_tail(&msg->m_list, &msq->q_messages);
    msq->q_cbytes += msgsz;
    msq->q_qnum++;
    atomic_add(msgsz, &ns->msg_bytes);
    atomic_inc(&ns->msg_hdrs);
    
    // 唤醒等待的接收者
    ss_wakeup(msq, &wake_q, false);
    
    ipc_unlock_object(&msq->q_perm);
    wake_up_q(&wake_q);
    
    return 0;
}
```

**消息队列的 $O(1)$ 查找优化**

Linux 使用基数树（radix tree）实现 IPC ID 到对象的快速映射：

```
IPC ID 结构：
+--------+--------+--------+
| seq(16)| idx(15)| use(1) |
+--------+--------+--------+

基数树查找：
     root
    /    \
   /      \
node1    node2
  |        |
 obj1     obj2
```

#### 共享内存（Shared Memory）

共享内存是最快的 IPC 机制，因为进程直接访问同一块物理内存，避免了数据复制。

**核心实现**

```c
// ipc/shm.c
struct shmid_kernel {
    struct kern_ipc_perm shm_perm;
    struct file *shm_file;      // 关联的文件对象
    unsigned long shm_nattch;    // 当前附加数
    unsigned long shm_segsz;     // 段大小
    time64_t shm_atim;          // 最后附加时间
    time64_t shm_dtim;          // 最后分离时间
    time64_t shm_ctim;          // 最后修改时间
    struct pid *shm_cprid;       // 创建者 PID
    struct pid *shm_lprid;       // 最后操作者 PID
    struct ucounts *mlock_ucounts;  // mlock 计数
    
    // 任务列表，用于 task->sysvshm.shm_clist
    struct list_head shm_clist;
    struct ipc_namespace *ns;
} __randomize_layout;
```

**共享内存映射过程**

```c
// shmat 系统调用的核心逻辑
long do_shmat(int shmid, char __user *shmaddr, int shmflg,
             ulong *raddr, unsigned long shmlba)
{
    struct shmid_kernel *shp;
    unsigned long addr = (unsigned long)shmaddr;
    unsigned long size;
    struct file *file, *base;
    int    err;
    unsigned long flags = MAP_SHARED;
    unsigned long prot;
    int acc_mode;
    struct ipc_namespace *ns;
    struct shm_file_data *sfd;
    int f_flags;
    unsigned long populate = 0;
    
    // 获取共享内存对象
    shp = shm_obtain_object_check(ns, shmid);
    if (IS_ERR(shp)) {
        err = PTR_ERR(shp);
        goto out;
    }
    
    // 权限检查
    if (ipcperms(ns, &shp->shm_perm, acc_mode))
        goto out_unlock;
    
    // 获取关联的文件
    base = get_file(shp->shm_file);
    shp->shm_nattch++;
    size = i_size_read(file_inode(base));
    
    // 创建 shm_file_data 结构
    sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);
    if (!sfd) {
        err = -ENOMEM;
        goto out_put;
    }
    
    file = alloc_file_clone(base, f_flags,
                           is_file_hugepages(base) ?
                           &shm_file_operations_huge :
                           &shm_file_operations);
    
    // 使用 do_mmap 映射到进程地址空间
    addr = do_mmap(file, addr, size, prot, flags, 0,
                  &populate, NULL);
    
    *raddr = addr;
    err = 0;
    
    if (populate)
        mm_populate(addr, populate);
        
    return err;
}
```

#### 信号量（Semaphore）

System V 信号量支持信号量集合和原子操作序列，比 POSIX 信号量功能更强大但也更复杂。

**信号量数据结构**

```c
// ipc/sem.c
struct sem_array {
    struct kern_ipc_perm sem_perm;  // IPC 权限
    time64_t         sem_ctime;     // 最后修改时间
    struct list_head pending_alter;  // 待处理的修改操作
    struct list_head pending_const;  // 待处理的常量操作
    struct list_head list_id;       // undo 列表
    int              sem_nsems;     // 信号量数量
    int              complex_count; // 复杂操作计数
    unsigned int     use_global_lock;// 全局锁标志
    
    struct sem       sems[];        // 信号量数组
} __randomize_layout;

struct sem {
    int semval;                     // 当前值
    struct pid *sempid;             // 最后操作的 PID
    spinlock_t lock;                // 每个信号量的锁
    struct list_head pending_alter; // 等待值改变的操作
    struct list_head pending_const; // 等待值不变的操作
    time64_t sem_otime;             // 最后操作时间
} ____cacheline_aligned_in_smp;
```

**信号量操作的原子性保证**

```c
// semop 系统调用处理多个信号量操作
static int perform_atomic_semop(struct sem_array *sma,
                               struct sem_queue *q)
{
    struct sembuf *sop;
    struct sem *curr;
    int nsops = q->nsops;
    int i;
    int semval, result = 0;
    
    // 第一遍：检查所有操作是否可以执行
    for (i = 0; i < nsops; i++) {
        sop = &q->sops[i];
        curr = &sma->sems[sop->sem_num];
        semval = curr->semval;
        
        if (sop->sem_op + semval < 0) {
            // 操作会导致信号量值为负，不能执行
            return 1;  // 需要等待
        }
    }
    
    // 第二遍：执行所有操作
    for (i = 0; i < nsops; i++) {
        sop = &q->sops[i];
        curr = &sma->sems[sop->sem_num];
        
        curr->semval += sop->sem_op;
        curr->sempid = q->pid;
    }
    
    return 0;  // 成功执行
}
```

### 4.1.3 POSIX IPC

POSIX IPC 是 IEEE 1003.1b-1993 标准定义的进程间通信机制，旨在提供比 System V IPC 更清洁、更一致的接口。POSIX IPC 的设计充分吸取了 System V IPC 的经验教训，提供了基于文件描述符的操作模型，更好地集成到 Unix 的 "一切皆文件" 哲学中。

#### POSIX 消息队列

POSIX 消息队列克服了 System V 消息队列的诸多限制，提供了消息优先级、异步通知等高级特性。

**核心数据结构**

```c
// ipc/mqueue.c
struct mqueue_inode_info {
    spinlock_t lock;
    struct inode vfs_inode;     // VFS inode
    wait_queue_head_t wait_q;   // 等待队列
    
    struct rb_root msg_tree;    // 消息红黑树（按优先级排序）
    struct rb_node *msg_tree_rightmost;  // 最右节点缓存
    struct posix_msg_tree_node *node_cache;  // 节点缓存
    
    struct mq_attr attr;        // 队列属性
    
    struct sigevent notify;     // 异步通知配置
    struct pid *notify_owner;   // 通知接收进程
    struct user_namespace *notify_user_ns;
    struct ucounts *ucounts;    // 用户计数
    
    unsigned long qsize;        // 队列当前大小（字节）
};

struct mq_attr {
    long mq_flags;      // 队列标志（O_NONBLOCK）
    long mq_maxmsg;     // 最大消息数
    long mq_msgsize;    // 最大消息大小
    long mq_curmsgs;    // 当前消息数
};

// 消息节点结构
struct msg_msg {
    struct rb_node m_rb_node;   // 红黑树节点
    struct list_head m_list;    // 同优先级消息链表
    long m_type;                // 消息优先级
    size_t m_ts;                // 消息大小
    struct msg_msgseg *next;    // 大消息的下一段
    void *security;             // 安全标签
};
```

**优先级队列实现**

POSIX 消息队列使用红黑树实现 $O(\log n)$ 的优先级队列：

```c
// 消息插入（按优先级）
static int msg_insert(struct msg_msg *msg,
                     struct mqueue_inode_info *info)
{
    struct rb_node **p, *parent = NULL;
    struct posix_msg_tree_node *leaf;
    long k = msg->m_type;  // 优先级作为键
    
    // 查找插入位置
    p = &info->msg_tree.rb_node;
    while (*p) {
        parent = *p;
        leaf = rb_entry(parent, struct posix_msg_tree_node, rb_node);
        
        if (k < leaf->priority) {
            p = &(*p)->rb_left;
        } else if (k > leaf->priority) {
            p = &(*p)->rb_right;
        } else {
            // 相同优先级，加入链表尾部（FIFO）
            list_add_tail(&msg->m_list, &leaf->msg_list);
            return 0;
        }
    }
    
    // 创建新的优先级节点
    if (info->node_cache) {
        leaf = info->node_cache;
        info->node_cache = NULL;
    } else {
        leaf = kmalloc(sizeof(*leaf), GFP_ATOMIC);
        if (!leaf)
            return -ENOMEM;
    }
    
    INIT_LIST_HEAD(&leaf->msg_list);
    leaf->priority = k;
    rb_link_node(&leaf->rb_node, parent, p);
    rb_insert_color(&leaf->rb_node, &info->msg_tree);
    
    // 更新最右节点缓存（最低优先级）
    if (parent == info->msg_tree_rightmost)
        info->msg_tree_rightmost = &leaf->rb_node;
        
    list_add_tail(&msg->m_list, &leaf->msg_list);
    return 0;
}
```

**异步通知机制**

POSIX 消息队列支持三种通知方式：

1. **信号通知**：向指定进程发送信号
2. **线程通知**：创建新线程执行通知函数
3. **信号值通知**：发送带值的实时信号

```c
// mq_notify 实现
SYSCALL_DEFINE2(mq_notify, mqd_t, mqdes,
               const struct sigevent __user *, u_notification)
{
    struct fd f;
    struct mqueue_inode_info *info;
    struct sigevent notification;
    
    f = fdget(mqdes);
    if (!f.file)
        return -EBADF;
        
    info = MQUEUE_I(file_inode(f.file));
    
    if (u_notification) {
        if (copy_from_user(&notification, u_notification,
                          sizeof(struct sigevent)))
            return -EFAULT;
            
        switch (notification.sigev_notify) {
        case SIGEV_NONE:
            break;
        case SIGEV_SIGNAL:
            // 设置信号通知
            info->notify.sigev_signo = notification.sigev_signo;
            info->notify.sigev_value = notification.sigev_value;
            info->notify_owner = get_pid(task_pid(current));
            break;
        case SIGEV_THREAD:
            // 线程通知需要用户空间库支持
            break;
        }
    } else {
        // 取消通知
        put_pid(info->notify_owner);
        info->notify_owner = NULL;
    }
    
    fdput(f);
    return 0;
}
```

#### POSIX 共享内存

POSIX 共享内存通过 tmpfs 文件系统实现，提供了更灵活的内存映射机制。

**实现架构**

```c
// POSIX 共享内存基于 tmpfs
static struct file_system_type shmem_fs_type = {
    .owner      = THIS_MODULE,
    .name       = "tmpfs",
    .init_fs_context = shmem_init_fs_context,
    .kill_sb    = kill_litter_super,
};

// shm_open 实际上是在 /dev/shm 下创建文件
int shm_open(const char *name, int oflag, mode_t mode)
{
    char pathname[PATH_MAX];
    int fd;
    
    // 构造路径 /dev/shm/name
    snprintf(pathname, PATH_MAX, "/dev/shm/%s", name);
    
    // 使用 open 系统调用
    fd = open(pathname, oflag, mode);
    
    return fd;
}
```

**与 System V 共享内存的对比**

| 特性 | System V | POSIX |
|------|----------|-------|
| 命名空间 | IPC 键值 | 文件系统路径 |
| 持久性 | 直到显式删除 | 可选持久化 |
| 大小调整 | 创建时固定 | ftruncate 动态调整 |
| 权限管理 | IPC 权限位 | 文件系统权限 |
| 同步机制 | 需要信号量 | 可用文件锁 |

**内存映射优化**

POSIX 共享内存支持大页（Huge Pages）映射：

```c
// 使用大页的共享内存
int shm_fd = shm_open("/hugepage_shm", O_CREAT | O_RDWR, 0666);

// 设置大页标志
struct statfs fs_stats;
fstatfs(shm_fd, &fs_stats);
if (fs_stats.f_type == HUGETLBFS_MAGIC) {
    // 自动使用大页
}

// 映射时指定 MAP_HUGETLB
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                 MAP_SHARED | MAP_HUGETLB, shm_fd, 0);
```

#### POSIX 信号量

POSIX 信号量提供了两种类型：命名信号量和未命名信号量，相比 System V 信号量更加简洁。

**信号量实现**

```c
// kernel/locking/semaphore.c
struct semaphore {
    raw_spinlock_t      lock;
    unsigned int        count;
    struct list_head    wait_list;
};

// POSIX 信号量的用户空间表示
typedef union {
    char __size[__SIZEOF_SEM_T];
    long int __align;
} sem_t;

// 内核中的 POSIX 信号量操作
struct posix_sem_ops {
    int (*wait)(struct semaphore *sem);
    int (*timedwait)(struct semaphore *sem, 
                    const struct timespec *abs_timeout);
    int (*trywait)(struct semaphore *sem);
    int (*post)(struct semaphore *sem);
    int (*getvalue)(struct semaphore *sem, int *sval);
};
```

**未命名信号量的共享内存实现**

```c
// 进程间共享的未命名信号量
struct shared_sem {
    struct semaphore sem;
    int pshared;            // PTHREAD_PROCESS_SHARED
    atomic_t refcount;      // 引用计数
};

// sem_init 实现
int sem_init(sem_t *sem, int pshared, unsigned int value)
{
    struct shared_sem *s;
    
    if (pshared == PTHREAD_PROCESS_SHARED) {
        // 分配共享内存段
        s = mmap(NULL, sizeof(*s), PROT_READ | PROT_WRITE,
                MAP_SHARED | MAP_ANONYMOUS, -1, 0);
        if (s == MAP_FAILED)
            return -1;
    } else {
        // 进程私有信号量
        s = malloc(sizeof(*s));
        if (!s)
            return -1;
    }
    
    sema_init(&s->sem, value);
    s->pshared = pshared;
    atomic_set(&s->refcount, 1);
    
    *(struct shared_sem **)sem = s;
    return 0;
}
```

**自适应等待优化**

POSIX 信号量实现了自适应等待策略，在轻度竞争时自旋，重度竞争时睡眠：

```c
// 自适应等待实现
static int adaptive_sem_wait(struct semaphore *sem)
{
    int spin_count = 0;
    const int MAX_SPINS = 1000;
    
    // 快速路径：尝试获取
    if (likely(raw_spin_trylock(&sem->lock))) {
        if (sem->count > 0) {
            sem->count--;
            raw_spin_unlock(&sem->lock);
            return 0;
        }
        raw_spin_unlock(&sem->lock);
    }
    
    // 自适应自旋
    while (spin_count++ < MAX_SPINS) {
        if (sem->count > 0) {
            if (raw_spin_trylock(&sem->lock)) {
                if (sem->count > 0) {
                    sem->count--;
                    raw_spin_unlock(&sem->lock);
                    return 0;
                }
                raw_spin_unlock(&sem->lock);
            }
        }
        cpu_relax();  // 处理器特定的自旋等待
    }
    
    // 慢速路径：睡眠等待
    return __sem_wait_slowpath(sem);
}
```

#### 与 System V IPC 对比

**架构差异**

```
System V IPC 架构：
    用户空间
        |
    系统调用接口 (msgget, shmat, semget)
        |
    IPC 命名空间
        |
    IPC 对象管理器
        |
    内核数据结构

POSIX IPC 架构：
    用户空间
        |
    文件系统接口 (open, mmap, unlink)
        |
    VFS 层
        |
    特殊文件系统 (mqueue, tmpfs)
        |
    内核数据结构
```

**性能对比**

| 操作 | System V | POSIX | 性能差异原因 |
|------|----------|-------|------------|
| 创建开销 | 中等 | 较低 | POSIX 利用文件系统缓存 |
| 查找速度 | O(1) | O(log n) | System V 使用哈希表 |
| 内存占用 | 较高 | 较低 | POSIX 共享 VFS 结构 |
| 并发性能 | 一般 | 较好 | POSIX 细粒度锁 |
| 持久化 | 自动 | 可选 | System V 默认持久 |

**选择建议**

1. **使用 POSIX IPC 的场景**：
   - 新开发的应用程序
   - 需要与文件系统集成
   - 需要精确的超时控制
   - 跨平台可移植性要求高

2. **使用 System V IPC 的场景**：
   - 维护遗留系统
   - 需要信号量集合的原子操作
   - 需要消息类型过滤
   - 已有大量 System V IPC 代码

## 4.3 信号机制实现

信号是 Unix 系统最古老的进程间通信机制之一，也是异步事件通知的核心机制。Linux 内核不仅完整实现了 POSIX.1 标准信号，还扩展了实时信号支持，提供了更丰富的信号信息传递能力。理解信号机制的内核实现对于编写健壮的系统程序至关重要。

### 4.3.1 信号基础架构

Linux 信号机制建立在精巧的数据结构和算法之上，实现了高效的信号产生、投递和处理流程。

#### 信号的产生与投递

**核心数据结构**

```c
// include/linux/sched/signal.h
struct signal_struct {
    refcount_t      sigcnt;
    atomic_t        live;
    int             nr_threads;
    struct list_head thread_head;
    
    wait_queue_head_t wait_chldexit;
    
    // 当前进程组信号处理器
    struct k_sigaction action[_NSIG];
    
    // 共享的挂起信号
    struct sigpending shared_pending;
    
    // POSIX 定时器列表
    struct list_head posix_timers;
    
    // 实时定时器
    struct hrtimer real_timer;
    ktime_t real_timer_offset;
    
    // CPU 时间限制
    struct task_cputime cputime_expires;
    struct list_head cpu_timers[3];
    
    // 进程组信息
    struct pid *pgrp;
    struct pid *tty_old_pgrp;
    
    // 会话领导者
    int leader;
    
    struct tty_struct *tty;
    
    // 累积的资源使用统计
    seqlock_t stats_lock;
    u64 utime, stime, cutime, cstime;
    u64 gtime, cgtime;
    struct prev_cputime prev_cputime;
    unsigned long nvcsw, nivcsw, cnvcsw, cnivcsw;
    unsigned long min_flt, maj_flt, cmin_flt, cmaj_flt;
    unsigned long inblock, oublock, cinblock, coublock;
    unsigned long maxrss, cmaxrss;
    struct task_io_accounting ioac;
    
    // 审计上下文
    unsigned long long sum_sched_runtime;
    
    struct rlimit rlim[RLIM_NLIMITS];
    
    // 进程组是否为孤儿
    unsigned int is_child_subreaper:1;
    unsigned int has_child_subreaper:1;
};

// 每个线程的信号信息
struct sighand_struct {
    spinlock_t      siglock;
    refcount_t      count;
    wait_queue_head_t signalfd_wqh;
    struct k_sigaction action[_NSIG];  // 信号处理器数组
};

// 挂起信号队列
struct sigpending {
    struct list_head list;     // 信号队列链表
    sigset_t signal;           // 挂起信号位图
};

// 信号队列项
struct sigqueue {
    struct list_head list;
    int flags;
    kernel_siginfo_t info;     // 信号信息
    struct ucounts *ucounts;
};
```

**信号产生流程**

信号可以通过多种方式产生：

1. **硬件异常**：如除零错误、段错误
2. **软件条件**：如 alarm 定时器到期
3. **终端输入**：如 Ctrl+C 产生 SIGINT
4. **系统调用**：如 kill()、raise()

```c
// kernel/signal.c - kill 系统调用实现
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
    struct kernel_siginfo info;
    
    prepare_kill_siginfo(sig, &info);
    
    return kill_something_info(sig, &info, pid);
}

static int kill_something_info(int sig, struct kernel_siginfo *info, pid_t pid)
{
    int ret;
    
    if (pid > 0) {
        // 发送给指定进程
        ret = kill_pid_info(sig, info, find_vpid(pid));
    } else if (pid == 0) {
        // 发送给进程组
        ret = kill_pgrp_info(sig, info, task_pgrp(current));
    } else if (pid == -1) {
        // 发送给所有进程（除了 init）
        ret = kill_all_info(sig, info);
    } else {
        // 发送给指定进程组
        ret = kill_pgrp_info(sig, info, find_vpid(-pid));
    }
    
    return ret;
}
```

**信号投递算法**

```c
// 信号投递的核心函数
static int __send_signal(int sig, struct kernel_siginfo *info,
                        struct task_struct *t, enum pid_type type, bool force)
{
    struct sigpending *pending;
    struct sigqueue *q;
    int override_rlimit;
    int ret = 0, result;
    
    // 检查信号是否被忽略
    result = TRACE_SIGNAL_IGNORED;
    if (!prepare_signal(sig, t, force))
        goto ret;
    
    // 选择挂起信号队列（线程私有或进程共享）
    pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending;
    
    // 检查是否为传统信号且已在队列中
    if (legacy_queue(pending, sig))
        goto ret;
    
    // 分配信号队列项
    q = __sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit);
    if (q) {
        list_add_tail(&q->list, &pending->list);
        switch ((unsigned long) info) {
        case (unsigned long) SEND_SIG_NOINFO:
            clear_siginfo(&q->info);
            q->info.si_signo = sig;
            q->info.si_errno = 0;
            q->info.si_code = SI_USER;
            q->info.si_pid = task_tgid_nr_ns(current,
                                            task_active_pid_ns(t));
            q->info.si_uid = from_kuid_munged(current_user_ns(),
                                             current_uid());
            break;
        case (unsigned long) SEND_SIG_PRIV:
            clear_siginfo(&q->info);
            q->info.si_signo = sig;
            q->info.si_errno = 0;
            q->info.si_code = SI_KERNEL;
            q->info.si_pid = 0;
            q->info.si_uid = 0;
            break;
        default:
            copy_siginfo(&q->info, info);
            break;
        }
    } else if (!is_si_special(info) &&
               sig >= SIGRTMIN && info->si_code != SI_USER) {
        // 实时信号必须排队，如果内存不足则失败
        ret = -EAGAIN;
        goto ret;
    }
    
    // 设置信号位图
    sigaddset(&pending->signal, sig);
    
    // 唤醒目标进程
    complete_signal(sig, t, type);
    
ret:
    trace_signal_generate(sig, info, t, type != PIDTYPE_PID, result);
    return ret;
}
```

#### 信号处理器注册

**sigaction 系统调用**

```c
// kernel/signal.c
SYSCALL_DEFINE4(rt_sigaction, int, sig,
               const struct sigaction __user *, act,
               struct sigaction __user *, oact,
               size_t, sigsetsize)
{
    struct k_sigaction new_sa, old_sa;
    int ret;
    
    // 检查信号号码有效性
    if (!valid_signal(sig) || sig < 1 || (act && sig_kernel_only(sig)))
        return -EINVAL;
    
    if (act) {
        if (copy_from_user(&new_sa.sa, act, sizeof(new_sa.sa)))
            return -EFAULT;
    }
    
    ret = do_sigaction(sig, act ? &new_sa : NULL, oact ? &old_sa : NULL);
    if (ret)
        return ret;
    
    if (oact) {
        if (copy_to_user(oact, &old_sa.sa, sizeof(old_sa.sa)))
            return -EFAULT;
    }
    
    return 0;
}

int do_sigaction(int sig, struct k_sigaction *act, struct k_sigaction *oact)
{
    struct task_struct *p = current, *t;
    struct k_sigaction *k;
    sigset_t mask;
    
    if (!valid_signal(sig) || sig < 1 || (act && sig_kernel_only(sig)))
        return -EINVAL;
    
    k = &p->sighand->action[sig-1];
    
    spin_lock_irq(&p->sighand->siglock);
    if (oact)
        *oact = *k;
    
    if (act) {
        sigdelsetmask(&act->sa.sa_mask,
                     sigmask(SIGKILL) | sigmask(SIGSTOP));
        *k = *act;
        
        // 如果设置为 SIG_IGN，清除挂起的信号
        if (sig_handler(p, sig) == SIG_IGN) {
            sigemptyset(&mask);
            sigaddset(&mask, sig);
            flush_sigqueue_mask(&mask, &p->signal->shared_pending);
            for_each_thread(p, t)
                flush_sigqueue_mask(&mask, &t->pending);
        }
    }
    
    spin_unlock_irq(&p->sighand->siglock);
    return 0;
}
```

#### 信号掩码与 Pending 信号

**信号掩码操作**

```c
// 信号掩码的原子操作
static inline void sigaddset(sigset_t *set, int _sig)
{
    unsigned long sig = _sig - 1;
    if (_NSIG_WORDS == 1)
        set->sig[0] |= 1UL << sig;
    else
        set->sig[sig / _NSIG_BPW] |= 1UL << (sig % _NSIG_BPW);
}

static inline void sigdelset(sigset_t *set, int _sig)
{
    unsigned long sig = _sig - 1;
    if (_NSIG_WORDS == 1)
        set->sig[0] &= ~(1UL << sig);
    else
        set->sig[sig / _NSIG_BPW] &= ~(1UL << (sig % _NSIG_BPW));
}

static inline int sigismember(sigset_t *set, int _sig)
{
    unsigned long sig = _sig - 1;
    if (_NSIG_WORDS == 1)
        return set->sig[0] & (1UL << sig);
    else
        return set->sig[sig / _NSIG_BPW] & (1UL << (sig % _NSIG_BPW));
}
```

**信号的阻塞与解除**

```c
// sigprocmask 系统调用
SYSCALL_DEFINE4(rt_sigprocmask, int, how, sigset_t __user *, nset,
               sigset_t __user *, oset, size_t, sigsetsize)
{
    sigset_t old_set, new_set;
    int error;
    
    // 保存旧的信号掩码
    old_set = current->blocked;
    
    if (nset) {
        if (copy_from_user(&new_set, nset, sizeof(sigset_t)))
            return -EFAULT;
        sigdelsetmask(&new_set, sigmask(SIGKILL) | sigmask(SIGSTOP));
        
        error = sigprocmask(how, &new_set, NULL);
        if (error)
            return error;
    }
    
    if (oset) {
        if (copy_to_user(oset, &old_set, sizeof(sigset_t)))
            return -EFAULT;
    }
    
    return 0;
}

// 设置信号掩码
int sigprocmask(int how, sigset_t *set, sigset_t *oldset)
{
    struct task_struct *tsk = current;
    sigset_t newset;
    
    // 根据操作类型计算新掩码
    switch (how) {
    case SIG_BLOCK:
        sigorsets(&newset, &tsk->blocked, set);
        break;
    case SIG_UNBLOCK:
        sigandnsets(&newset, &tsk->blocked, set);
        break;
    case SIG_SETMASK:
        newset = *set;
        break;
    default:
        return -EINVAL;
    }
    
    __set_current_blocked(&newset);
    return 0;
}
```

### 4.3.2 实时信号扩展

Linux 支持 POSIX.1b 实时信号扩展，提供了更可靠的信号机制。

#### 标准信号 vs 实时信号

**信号分类**

```c
// include/uapi/asm-generic/signal.h
#define SIGHUP       1  // 终端挂起
#define SIGINT       2  // 终端中断（Ctrl+C）
#define SIGQUIT      3  // 终端退出（Ctrl+\）
#define SIGILL       4  // 非法指令
#define SIGTRAP      5  // 跟踪/断点陷阱
#define SIGABRT      6  // abort() 调用
#define SIGBUS       7  // 总线错误
#define SIGFPE       8  // 浮点异常
#define SIGKILL      9  // 强制终止（不可捕获）
#define SIGUSR1     10  // 用户定义信号 1
#define SIGSEGV     11  // 段错误
#define SIGUSR2     12  // 用户定义信号 2
#define SIGPIPE     13  // 管道破裂
#define SIGALRM     14  // alarm() 定时器
#define SIGTERM     15  // 终止请求
#define SIGSTKFLT   16  // 协处理器栈错误
#define SIGCHLD     17  // 子进程状态改变
#define SIGCONT     18  // 继续执行
#define SIGSTOP     19  // 停止执行（不可捕获）
#define SIGTSTP     20  // 终端停止（Ctrl+Z）
#define SIGTTIN     21  // 后台进程读终端
#define SIGTTOU     22  // 后台进程写终端
#define SIGURG      23  // 紧急数据到达
#define SIGXCPU     24  // CPU 时间限制超出
#define SIGXFSZ     25  // 文件大小限制超出
#define SIGVTALRM   26  // 虚拟定时器
#define SIGPROF     27  // 性能分析定时器
#define SIGWINCH    28  // 终端窗口大小改变
#define SIGIO       29  // I/O 就绪
#define SIGPWR      30  // 电源故障
#define SIGSYS      31  // 系统调用参数错误

// 实时信号范围
#define SIGRTMIN    32
#define SIGRTMAX    64
```

**关键差异**

| 特性 | 标准信号（1-31） | 实时信号（32-64） |
|------|-----------------|------------------|
| 排队 | 不排队，多个相同信号合并 | 可靠排队，不丢失 |
| 优先级 | 信号编号越小优先级越高 | 可自定义优先级 |
| 信息传递 | 仅信号编号 | 可携带额外数据 |
| 顺序保证 | 无保证 | FIFO 顺序保证 |
| 用途 | 系统定义的特定事件 | 应用程序自定义 |

#### 信号队列机制

**实时信号的可靠排队**

```c
// 实时信号队列管理
struct sigqueue_cache {
    struct sigqueue *first;
    int count;
};

static struct sigqueue *__sigqueue_alloc(int sig, struct task_struct *t,
                                        gfp_t gfp_flags, int override_rlimit)
{
    struct sigqueue *q = NULL;
    struct ucounts *ucounts = NULL;
    long sigpending;
    
    // 实时信号必须排队
    if ((sig >= SIGRTMIN) && 
        (sigpending = inc_rlimit_ucounts(ucounts, UCOUNT_RLIMIT_SIGPENDING, 1)) == 0) {
        dec_rlimit_ucounts(ucounts, UCOUNT_RLIMIT_SIGPENDING, 1);
        return NULL;
    }
    
    // 从缓存或 slab 分配
    if (current->sigqueue_cache.count > 0) {
        q = current->sigqueue_cache.first;
        current->sigqueue_cache.first = q->next;
        current->sigqueue_cache.count--;
    } else {
        q = kmem_cache_alloc(sigqueue_cachep, gfp_flags);
    }
    
    if (unlikely(q == NULL)) {
        if (ucounts)
            dec_rlimit_ucounts(ucounts, UCOUNT_RLIMIT_SIGPENDING, 1);
    } else {
        INIT_LIST_HEAD(&q->list);
        q->flags = 0;
        q->ucounts = ucounts;
    }
    
    return q;
}
```

#### siginfo_t 结构详解

**信号信息结构**

```c
// include/uapi/asm-generic/siginfo.h
typedef struct kernel_siginfo {
    struct {
        int si_signo;    // 信号编号
        int si_errno;    // errno 值
        int si_code;     // 信号来源代码
        
        union {
            int _pad[SI_PAD_SIZE];
            
            // SIGKILL
            struct {
                __kernel_pid_t _pid;    // 发送进程 PID
                __kernel_uid32_t _uid;  // 发送进程 UID
            } _kill;
            
            // POSIX.1b 定时器
            struct {
                __kernel_timer_t _tid;  // 定时器 ID
                int _overrun;           // 溢出计数
                sigval_t _sigval;       // 信号值
                int _sys_private;       // 系统私有数据
            } _timer;
            
            // POSIX.1b 信号
            struct {
                __kernel_pid_t _pid;    // 发送进程 PID
                __kernel_uid32_t _uid;  // 发送进程 UID
                sigval_t _sigval;       // 信号值
            } _rt;
            
            // SIGCHLD
            struct {
                __kernel_pid_t _pid;    // 子进程 PID
                __kernel_uid32_t _uid;  // 子进程 UID
                int _status;            // 退出状态
                __kernel_clock_t _utime;
                __kernel_clock_t _stime;
            } _sigchld;
            
            // SIGILL, SIGFPE, SIGSEGV, SIGBUS, SIGTRAP
            struct {
                void __user *_addr;     // 故障地址
                
                union {
                    // BUS_MCEERR_AR, BUS_MCEERR_AO
                    int _trapno;        // TRAP 编号
                    
                    // BUS_MCEERR_AR, BUS_MCEERR_AO
                    short _addr_lsb;    // 地址 LSB
                    
                    // SEGV_BNDERR
                    struct {
                        void __user *_lower;
                        void __user *_upper;
                    } _addr_bnd;
                    
                    // SEGV_PKUERR
                    __u32 _pkey;
                };
            } _sigfault;
            
            // SIGPOLL
            struct {
                long _band;     // POLL_IN, POLL_OUT, POLL_MSG
                int _fd;        // 文件描述符
            } _sigpoll;
            
            // SIGSYS
            struct {
                void __user *_call_addr;    // 调用地址
                int _syscall;                // 系统调用号
                unsigned int _arch;          // 架构
            } _sigsys;
        } _sifields;
    };
} kernel_siginfo_t;
```

**信号代码定义**

```c
// 信号来源代码（si_code）
#define SI_USER      0      // kill(), raise()
#define SI_KERNEL    0x80   // 内核产生
#define SI_QUEUE    -1      // sigqueue()
#define SI_TIMER    -2      // POSIX.1b 定时器
#define SI_MESGQ    -3      // POSIX.1b 消息队列
#define SI_ASYNCIO  -4      // AIO 完成
#define SI_SIGIO    -5      // SIGIO 排队
#define SI_TKILL    -6      // tkill(), tgkill()
#define SI_DETHREAD -7      // SIGCHLD from execve

// SIGILL 的 si_code
#define ILL_ILLOPC  1       // 非法操作码
#define ILL_ILLOPN  2       // 非法操作数
#define ILL_ILLADR  3       // 非法寻址模式
#define ILL_ILLTRP  4       // 非法陷阱
#define ILL_PRVOPC  5       // 特权操作码
#define ILL_PRVREG  6       // 特权寄存器
#define ILL_COPROC  7       // 协处理器错误
#define ILL_BADSTK  8       // 内部栈错误

// SIGSEGV 的 si_code
#define SEGV_MAPERR 1       // 地址未映射
#define SEGV_ACCERR 2       // 权限错误
#define SEGV_BNDERR 3       // 边界检查失败
#define SEGV_PKUERR 4       // 保护键错误
```

### 4.4 futex 与用户态同步
#### 4.4.1 futex 设计原理
- 用户态快速路径
- 内核态慢速路径
- futex 哈希表组织

#### 4.4.2 futex 操作详解
- FUTEX_WAIT 与 FUTEX_WAKE
- Priority Inheritance (PI) futex
- Robust futex 机制

### 4.5 现代 IPC 机制
#### 4.5.1 eventfd
- eventfd 实现原理
- 与 epoll 集成
- eventfd 在虚拟化中的应用

#### 4.5.2 signalfd 与 timerfd
- 信号的文件描述符抽象
- 定时器的文件描述符抽象
- 统一的事件处理模型

### 4.6 性能分析与优化
- IPC 机制性能对比
- 零拷贝优化技术
- NUMA 架构下的 IPC 优化

### 4.7 历史故事
- Ulrich Drepper 与 futex 的诞生
- System V IPC 到 POSIX IPC 的标准之争
- Android Binder 的设计哲学

### 4.8 高级话题
- io_uring 的异步通信模型
- RDMA 在内核中的支持
- 用户态旁路技术（DPDK）

### 4.9 本章小结

### 4.10 练习题

### 4.11 常见陷阱与错误

### 4.12 最佳实践检查清单