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
- POSIX 消息队列
- POSIX 共享内存
- POSIX 信号量
- 与 System V IPC 对比

### 4.3 信号机制实现
#### 4.3.1 信号基础架构
- 信号的产生与投递
- 信号处理器注册
- 信号掩码与pending信号

#### 4.3.2 实时信号扩展
- 标准信号 vs 实时信号
- 信号队列机制
- siginfo_t 结构详解

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