# 第12章：容器与命名空间

## 本章概要

容器技术已成为现代云原生应用的基石，而 Linux 内核的命名空间（namespace）和控制组（cgroups）机制是容器实现的核心基础。本章深入剖析这些内核机制的实现原理，从早期的 chroot 到现代容器运行时，探索进程隔离、资源限制和安全边界的内核级实现。通过学习本章，您将理解容器"轻量级虚拟化"背后的技术本质，掌握构建容器运行时所需的内核接口，以及如何在保证隔离性的同时优化性能。

## 12.1 命名空间的演进与设计哲学

### 12.1.1 从 chroot 到 namespace

Linux 命名空间的历史可以追溯到 1979 年的 chroot 系统调用。chroot 改变进程的根目录视图，是最早的进程隔离机制：

```
传统 Unix 进程视图：
    /
    ├── bin/
    ├── etc/
    ├── home/
    └── usr/

chroot 后的进程视图：
    / (实际上是 /var/chroot)
    ├── bin/
    ├── etc/
    └── lib/
```

然而，chroot 只隔离了文件系统视图，进程仍能看到系统的其他资源。2002 年，Eric W. Biederman 提出了命名空间的概念，将隔离扩展到系统的各个方面。命名空间的设计遵循几个关键原则：

1. **透明性**：命名空间内的进程无需修改即可运行
2. **递归性**：命名空间可以嵌套，形成层次结构
3. **独立性**：各类命名空间可以独立使用和组合
4. **兼容性**：保持 POSIX 语义和向后兼容

### 12.1.2 命名空间的类型与用途

Linux 内核目前支持 8 种命名空间，每种负责隔离特定的系统资源：

| 命名空间 | 隔离内容 | 引入版本 | 主要用途 |
|---------|---------|---------|---------|
| Mount (mnt) | 挂载点 | 2.4.19 | 文件系统视图隔离 |
| UTS | 主机名和域名 | 2.6.19 | 主机标识隔离 |
| IPC | System V IPC, POSIX 消息队列 | 2.6.19 | 进程间通信隔离 |
| PID | 进程 ID | 2.6.24 | 进程树隔离 |
| Network (net) | 网络设备、协议栈、端口 | 2.6.29 | 网络栈隔离 |
| User | 用户和组 ID | 3.8 | 用户权限隔离 |
| Cgroup | Cgroup 根目录 | 4.6 | 资源限制视图隔离 |
| Time | 系统时间 | 5.6 | 时间视图隔离 |

### 12.1.3 命名空间的内核数据结构

命名空间在内核中通过 `struct nsproxy` 结构管理，每个进程的 `task_struct` 包含一个指向 nsproxy 的指针：

```c
// include/linux/nsproxy.h
struct nsproxy {
    atomic_t count;                    // 引用计数
    struct uts_namespace *uts_ns;     // UTS 命名空间
    struct ipc_namespace *ipc_ns;     // IPC 命名空间
    struct mnt_namespace *mnt_ns;     // Mount 命名空间
    struct pid_namespace *pid_ns_for_children; // PID 命名空间
    struct net *net_ns;                // Network 命名空间
    struct time_namespace *time_ns;    // Time 命名空间
    struct time_namespace *time_ns_for_children;
    struct cgroup_namespace *cgroup_ns; // Cgroup 命名空间
};

// include/linux/sched.h
struct task_struct {
    // ... 其他字段
    struct nsproxy *nsproxy;  // 命名空间代理
    // ...
};
```

命名空间的创建和切换通过 clone、unshare 和 setns 系统调用实现：

```c
// 创建新命名空间的标志位
#define CLONE_NEWNS     0x00020000  // Mount namespace
#define CLONE_NEWUTS    0x04000000  // UTS namespace
#define CLONE_NEWIPC    0x08000000  // IPC namespace
#define CLONE_NEWPID    0x20000000  // PID namespace
#define CLONE_NEWNET    0x40000000  // Network namespace
#define CLONE_NEWUSER   0x10000000  // User namespace
#define CLONE_NEWCGROUP 0x02000000  // Cgroup namespace
#define CLONE_NEWTIME   0x00000080  // Time namespace
```

## 12.2 核心命名空间实现详解

### 12.2.1 PID 命名空间

PID 命名空间是容器实现的核心，它为进程提供独立的进程 ID 空间。在 PID 命名空间内，进程可以拥有与外部相同的 PID，实现进程树的完全隔离。

```
主机视角：
    systemd (PID 1)
    ├── sshd (PID 1234)
    ├── container-runtime (PID 5678)
    │   └── container-init (PID 5679)  <- 容器内的 PID 1
    │       ├── app (PID 5680)         <- 容器内的 PID 2
    │       └── worker (PID 5681)      <- 容器内的 PID 3

容器内视角：
    init (PID 1)
    ├── app (PID 2)
    └── worker (PID 3)
```

PID 命名空间的关键实现细节：

1. **层次结构**：PID 命名空间形成树状层次，子命名空间中的进程在父命名空间中可见
2. **PID 映射**：每个进程在其所在的命名空间及所有祖先命名空间中都有 PID
3. **init 进程**：每个 PID 命名空间都有自己的 init 进程（PID 1），负责回收孤儿进程

```c
// kernel/pid.c - PID 分配的核心结构
struct pid {
    atomic_t count;
    unsigned int level;           // PID 命名空间的层级
    /* 每个命名空间层级的 PID 号 */
    struct upid numbers[1];
};

struct upid {
    int nr;                       // 在该命名空间中的 PID
    struct pid_namespace *ns;    // 所属的命名空间
};

// PID 命名空间结构
struct pid_namespace {
    struct kref kref;
    struct pidmap pidmap[PIDMAP_ENTRIES];
    struct task_struct *child_reaper;  // init 进程
    unsigned int level;                // 层级深度
    struct pid_namespace *parent;      // 父命名空间
    // ...
};
```

### 12.2.2 Network 命名空间

网络命名空间提供完全独立的网络栈，包括网络设备、IP 地址、路由表、防火墙规则等：

```c
// include/net/net_namespace.h
struct net {
    atomic_t count;               // 引用计数
    struct list_head list;        // 命名空间链表
    
    struct net_device *loopback_dev;  // 环回设备
    struct list_head dev_base_head;   // 网络设备链表
    
    struct sock *rtnl;            // rtnetlink 套接字
    struct sock *genl_sock;       // generic netlink
    
    struct netns_ipv4 ipv4;      // IPv4 相关
    struct netns_ipv6 ipv6;      // IPv6 相关
    struct netns_packet packet;   // packet 套接字
    struct netns_unix unx;        // Unix 域套接字
    
    struct net_generic *gen;      // 通用存储
    // ...
};
```

网络命名空间的典型使用模式：

```
主机网络命名空间
    eth0 (10.0.0.1)
    docker0 (172.17.0.1) ──┐
                           │
                      veth pair
                           │
容器网络命名空间          │
    eth0 (172.17.0.2) ────┘
    lo (127.0.0.1)
```

### 12.2.3 Mount 命名空间

Mount 命名空间隔离文件系统挂载点，使每个容器拥有独立的文件系统视图：

```c
// fs/mount.h
struct mnt_namespace {
    atomic_t count;
    struct mount *root;           // 根挂载点
    struct list_head list;        // 挂载点链表
    struct user_namespace *user_ns;
    u64 seq;                      // 序列号
    wait_queue_head_t poll;
    int event;
};

struct mount {
    struct hlist_node mnt_hash;
    struct mount *mnt_parent;     // 父挂载点
    struct dentry *mnt_mountpoint; // 挂载位置
    struct vfsmount mnt;
    // ...
};
```

挂载传播机制：

- **private**：挂载事件不传播
- **shared**：挂载事件双向传播
- **slave**：单向接收传播
- **unbindable**：不可绑定挂载

### 12.2.4 User 命名空间

User 命名空间是安全容器的关键，它允许将容器内的 root 用户映射为主机上的普通用户：

```c
// kernel/user_namespace.c
struct user_namespace {
    struct uid_gid_map uid_map;   // UID 映射
    struct uid_gid_map gid_map;   // GID 映射
    struct uid_gid_map projid_map; // 项目 ID 映射
    atomic_t count;
    struct user_namespace *parent;
    kuid_t owner;                  // 创建者的 UID
    kgid_t group;                  // 创建者的 GID
    unsigned int proc_inum;
    unsigned long flags;
    // ...
};

struct uid_gid_extent {
    u32 first;      // 映射的起始 ID
    u32 lower_first; // 父命名空间中的起始 ID
    u32 count;      // 映射数量
};
```

UID/GID 映射示例：

```
容器内 UID    主机 UID
0 (root)  ->  100000
1         ->  100001
999       ->  100999
```

## 12.3 Cgroups：资源控制与限制

### 12.3.1 Cgroups v1 架构

Cgroups（Control Groups）提供了对进程组的资源使用进行限制、统计和隔离的机制。v1 采用多层次的目录结构，每个子系统独立挂载：

```
/sys/fs/cgroup/
├── cpu/           # CPU 使用限制
│   ├── docker/
│   │   └── <container-id>/
│   │       ├── cpu.shares
│   │       └── cpu.cfs_quota_us
├── memory/        # 内存限制
│   ├── docker/
│   │   └── <container-id>/
│   │       ├── memory.limit_in_bytes
│   │       └── memory.usage_in_bytes
└── blkio/         # 块设备 I/O 限制
```

核心子系统功能：

| 子系统 | 功能 | 关键参数 |
|-------|------|---------|
| cpu | CPU 时间分配 | cpu.shares, cpu.cfs_period_us, cpu.cfs_quota_us |
| cpuset | CPU 和内存节点绑定 | cpuset.cpus, cpuset.mems |
| memory | 内存使用限制 | memory.limit_in_bytes, memory.soft_limit_in_bytes |
| blkio | 块设备 I/O 控制 | blkio.weight, blkio.throttle.read_bps_device |
| devices | 设备访问控制 | devices.allow, devices.deny |
| freezer | 暂停/恢复进程组 | freezer.state |
| net_cls | 网络分类标记 | net_cls.classid |
| pids | 进程数量限制 | pids.max |

### 12.3.2 Cgroups v2 统一层次

Cgroups v2 采用统一的层次结构，解决了 v1 的设计缺陷：

```c
// kernel/cgroup/cgroup.c
struct cgroup {
    struct cgroup_subsys_state self;
    unsigned long flags;
    int id;
    int level;                    // 层级深度
    
    struct cgroup *parent;        // 父 cgroup
    struct kernfs_node *kn;       // kernfs 节点
    
    struct cgroup_file procs_file; // cgroup.procs 文件
    struct cgroup_file events_file; // cgroup.events 文件
    
    u16 subtree_control;          // 子树控制掩码
    u16 subtree_ss_mask;          // 子树子系统掩码
    
    struct list_head pidlists;    // PID 列表
    struct cgroup_rstat_cpu __percpu *rstat_cpu;
    // ...
};
```

v2 的改进特性：

1. **统一层次**：所有控制器共享同一层次结构
2. **更好的资源模型**：基于权重的资源分配
3. **压力指标（PSI）**：memory.pressure, cpu.pressure, io.pressure
4. **eBPF 集成**：支持 BPF 程序进行细粒度控制

```
/sys/fs/cgroup/unified/
├── cgroup.controllers      # 可用控制器
├── cgroup.subtree_control  # 子树启用的控制器
├── user.slice/
│   └── user-1000.slice/
│       ├── memory.max
│       ├── memory.current
│       ├── cpu.max
│       └── io.max
└── system.slice/
    └── docker.service/
```

### 12.3.3 资源限制的内核实现

内存限制实现（以 memory cgroup 为例）：

```c
// mm/memcontrol.c
struct mem_cgroup {
    struct cgroup_subsys_state css;
    
    struct mem_cgroup_id id;
    
    /* 内存计数器 */
    struct page_counter memory;    // 内存使用
    struct page_counter swap;      // 交换使用
    struct page_counter memsw;     // 内存+交换
    struct page_counter kmem;      // 内核内存
    struct page_counter tcpmem;    // TCP 缓冲区
    
    /* 软限制 */
    unsigned long soft_limit;
    
    /* 统计信息 */
    struct mem_cgroup_stat_cpu __percpu *stat_cpu;
    
    /* OOM 控制 */
    struct mem_cgroup_oom_info oom_info;
    bool oom_kill_disable;
    
    /* 内存回收 */
    struct work_struct high_work;
    unsigned long memory_pressure;
    // ...
};
```

CPU 限制实现（CFS 带宽控制）：

```c
// kernel/sched/fair.c
struct cfs_bandwidth {
    raw_spinlock_t lock;
    ktime_t period;               // 周期（默认 100ms）
    u64 quota;                    // 配额
    u64 runtime;                  // 剩余运行时间
    s64 hierarchical_quota;
    
    struct hrtimer period_timer;  // 周期定时器
    struct hrtimer slack_timer;   // 松弛定时器
    struct list_head throttled_cfs_rq;
    
    int nr_periods;
    int nr_throttled;
    u64 throttled_time;
};
```

## 12.4 容器运行时原理

### 12.4.1 容器创建的系统调用序列

容器的创建涉及一系列精心编排的系统调用，从进程克隆到资源限制设置。以下是典型的容器启动流程：

```c
// 简化的容器创建流程
pid_t create_container() {
    // 1. 创建新的命名空间
    int flags = CLONE_NEWNS | CLONE_NEWPID | CLONE_NEWNET | 
                CLONE_NEWIPC | CLONE_NEWUTS | CLONE_NEWUSER;
    
    // 2. 克隆进程并创建命名空间
    pid_t pid = clone(container_init, stack_top, flags | SIGCHLD, NULL);
    
    if (pid > 0) {  // 父进程
        // 3. 配置 user namespace 映射
        write_uid_map(pid);
        write_gid_map(pid);
        
        // 4. 设置 cgroups
        setup_cgroups(pid);
        
        // 5. 配置网络
        setup_network(pid);
    }
    
    return pid;
}

int container_init(void *arg) {
    // 容器内的 init 进程
    // 1. 设置主机名
    sethostname("container", 9);
    
    // 2. 挂载必要的文件系统
    mount("proc", "/proc", "proc", 0, NULL);
    mount("sys", "/sys", "sysfs", 0, NULL);
    
    // 3. pivot_root 切换根文件系统
    pivot_root("/new_root", "/new_root/.old_root");
    
    // 4. 卸载旧根
    umount2("/.old_root", MNT_DETACH);
    
    // 5. 执行容器应用
    execve("/bin/sh", argv, envp);
}
```

### 12.4.2 OCI 运行时规范

Open Container Initiative (OCI) 定义了容器运行时的标准规范，包括运行时规范（runtime-spec）和镜像规范（image-spec）：

```json
// config.json - OCI 运行时配置示例
{
    "ociVersion": "1.0.2",
    "process": {
        "terminal": true,
        "user": {
            "uid": 0,
            "gid": 0
        },
        "args": ["/bin/sh"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "TERM=xterm"
        ],
        "cwd": "/",
        "capabilities": {
            "bounding": ["CAP_NET_BIND_SERVICE"],
            "effective": ["CAP_NET_BIND_SERVICE"],
            "permitted": ["CAP_NET_BIND_SERVICE"]
        },
        "rlimits": [{
            "type": "RLIMIT_NOFILE",
            "hard": 1024,
            "soft": 1024
        }]
    },
    "root": {
        "path": "rootfs",
        "readonly": false
    },
    "mounts": [{
        "destination": "/proc",
        "type": "proc",
        "source": "proc"
    }],
    "linux": {
        "namespaces": [
            {"type": "pid"},
            {"type": "network"},
            {"type": "ipc"},
            {"type": "uts"},
            {"type": "mount"}
        ],
        "resources": {
            "memory": {
                "limit": 536870912
            },
            "cpu": {
                "shares": 1024,
                "quota": 100000,
                "period": 100000
            }
        }
    }
}
```

### 12.4.3 runc 实现分析

runc 是 OCI 运行时规范的参考实现，其核心流程：

```go
// libcontainer/process_linux.go 简化版
func (p *initProcess) start() error {
    // 1. 创建管道用于父子进程通信
    parentPipe, childPipe := newPipe()
    
    // 2. 启动进程
    cmd := p.cmd
    cmd.ExtraFiles = []*os.File{childPipe}
    cmd.Env = append(cmd.Env, 
        fmt.Sprintf("_LIBCONTAINER_INITPIPE=%d", 3))
    
    if err := cmd.Start(); err != nil {
        return err
    }
    
    // 3. 发送配置到子进程
    if err := p.sendConfig(); err != nil {
        return err
    }
    
    // 4. 等待子进程准备就绪
    if err := p.waitForChildExit(); err != nil {
        return err
    }
    
    // 5. 配置 cgroups
    if err := p.manager.Apply(p.pid()); err != nil {
        return err
    }
    
    // 6. 设置网络
    if err := p.createNetworkInterfaces(); err != nil {
        return err
    }
    
    // 7. 通知子进程继续
    if err := p.sendContinue(); err != nil {
        return err
    }
    
    return nil
}
```

### 12.4.4 容器网络模型

容器网络通常采用 veth pair（虚拟以太网对）连接容器和主机网络：

```
   主机网络命名空间                    容器网络命名空间
   ┌──────────────┐                  ┌──────────────┐
   │              │                  │              │
   │  docker0     │                  │    eth0      │
   │  (bridge)    │◄────veth pair───►│  172.17.0.2  │
   │  172.17.0.1  │                  │              │
   │              │                  │              │
   │     eth0     │                  │              │
   │  10.0.0.100  │                  │              │
   └──────────────┘                  └──────────────┘
          │
          ▼
    外部网络 (NAT)
```

veth 设备创建和配置：

```c
// 创建 veth pair
int create_veth_pair(const char *veth1, const char *veth2) {
    struct nl_sock *sock = nl_socket_alloc();
    struct rtnl_link *link = rtnl_link_alloc();
    struct rtnl_link *peer = rtnl_link_alloc();
    
    // 设置 veth 类型
    rtnl_link_set_type(link, "veth");
    rtnl_link_set_name(link, veth1);
    rtnl_link_set_name(peer, veth2);
    
    // 创建 veth pair
    return rtnl_link_add(sock, link, NLM_F_CREATE);
}

// 将 veth 一端移到容器命名空间
int move_veth_to_netns(const char *veth, pid_t pid) {
    char path[256];
    snprintf(path, sizeof(path), "/proc/%d/ns/net", pid);
    
    int netns_fd = open(path, O_RDONLY);
    struct nl_sock *sock = nl_socket_alloc();
    struct rtnl_link *link = rtnl_link_get_by_name(sock, veth);
    
    rtnl_link_set_ns_fd(link, netns_fd);
    return rtnl_link_change(sock, link, link, 0);
}
```

## 12.5 安全容器技术

### 12.5.1 容器安全挑战

传统容器共享主机内核，存在固有的安全风险：

1. **内核漏洞**：容器逃逸可能获得主机权限
2. **资源耗尽**：恶意容器可能耗尽系统资源
3. **侧信道攻击**：共享 CPU 缓存可能泄露信息
4. **权限提升**：不当的权能配置可能导致提权

### 12.5.2 gVisor：用户态内核

gVisor 通过在用户态实现兼容的系统调用接口，提供额外的隔离层：

```
应用程序
    │
    ▼ 系统调用
Sentry (用户态内核)
    │
    ▼ 有限的系统调用
主机内核
```

gVisor 的关键组件：

```go
// Sentry 系统调用拦截
func (k *Kernel) SyscallEnter(t *Task, sysno uintptr, args arch.SyscallArguments) {
    // 检查系统调用是否允许
    if !k.featureSet.HasFeature(sysno) {
        return syserror.ENOSYS
    }
    
    // 执行系统调用模拟
    switch sysno {
    case syscall.SYS_OPEN:
        return k.sysOpen(t, args)
    case syscall.SYS_READ:
        return k.sysRead(t, args)
    case syscall.SYS_WRITE:
        return k.sysWrite(t, args)
    // ...
    }
}
```

### 12.5.3 Kata Containers：轻量级虚拟机

Kata Containers 为每个容器创建独立的轻量级虚拟机，提供硬件级隔离：

```
容器运行时 (containerd/CRI-O)
         │
         ▼
    kata-runtime
         │
    ┌────┴────┐
    ▼         ▼
kata-proxy  kata-shim
    │         │
    └────┬────┘
         ▼
    kata-agent (VM内)
         │
         ▼
    Guest Kernel
         │
         ▼
    Container App
```

Kata 的内存优化技术：

```c
// DAX (Direct Access) 内存映射
struct dax_device {
    struct inode *inode;
    struct cdev cdev;
    void *private;
    unsigned long flags;
    const struct dax_operations *ops;
};

// 内存去重 (KSM - Kernel Same-page Merging)
struct ksm_scan {
    struct mm_struct *mm_slot;
    unsigned long address;
    struct rmap_item **rmap_list;
    unsigned long seqnr;
};
```

### 12.5.4 安全增强机制

#### Seccomp-BPF：系统调用过滤

```c
// 定义 seccomp 过滤规则
struct sock_filter filter[] = {
    // 加载系统调用号
    BPF_STMT(BPF_LD | BPF_W | BPF_ABS, 
             offsetof(struct seccomp_data, nr)),
    
    // 允许的系统调用
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
    
    // 默认拒绝
    BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
};

struct sock_fprog prog = {
    .len = sizeof(filter) / sizeof(filter[0]),
    .filter = filter,
};

// 应用过滤规则
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);
```

#### AppArmor/SELinux 强制访问控制

```
# AppArmor 容器配置文件示例
profile docker-container flags=(attach_disconnected,mediate_deleted) {
  # 网络访问
  network inet,
  network inet6,
  
  # 文件访问
  deny @{PROC}/* w,
  deny @{PROC}/.* w,
  deny /sys/[^f]*/** w,
  deny /sys/f[^s]*/** w,
  
  # 挂载权限
  mount,
  umount,
  pivot_root,
  
  # 权能限制
  capability net_bind_service,
  capability setuid,
  capability setgid,
  deny capability sys_admin,
}
```

#### Capabilities 细粒度权限控制

```c
// 权能集合管理
struct cred {
    // ... 其他字段
    kernel_cap_t cap_inheritable;  // 可继承权能
    kernel_cap_t cap_permitted;    // 允许的权能
    kernel_cap_t cap_effective;    // 生效的权能
    kernel_cap_t cap_bset;        // 边界集
    kernel_cap_t cap_ambient;     // 环境权能
    // ...
};

// 常用容器权能配置
#define CONTAINER_CAPS ( \
    CAP_TO_MASK(CAP_CHOWN) | \
    CAP_TO_MASK(CAP_DAC_OVERRIDE) | \
    CAP_TO_MASK(CAP_FSETID) | \
    CAP_TO_MASK(CAP_FOWNER) | \
    CAP_TO_MASK(CAP_SETGID) | \
    CAP_TO_MASK(CAP_SETUID) | \
    CAP_TO_MASK(CAP_SETPCAP) | \
    CAP_TO_MASK(CAP_NET_BIND_SERVICE) | \
    CAP_TO_MASK(CAP_KILL) \
)
```

## 12.6 容器编排与集群管理

### 12.6.1 容器编排的内核支持

Kubernetes 等容器编排系统依赖多项内核特性实现集群管理：

1. **CRI（Container Runtime Interface）**：标准化容器运行时接口
2. **CNI（Container Network Interface）**：标准化网络插件接口
3. **CSI（Container Storage Interface）**：标准化存储插件接口

### 12.6.2 Pod 的内核实现

Kubernetes Pod 中的容器共享某些命名空间：

```c
// Pod 内容器的命名空间配置
struct pod_namespace_config {
    // 共享的命名空间
    struct net *net_ns;        // 共享网络命名空间
    struct ipc_namespace *ipc_ns; // 共享 IPC 命名空间
    struct uts_namespace *uts_ns; // 共享 UTS 命名空间
    
    // 独立的命名空间
    struct pid_namespace *pid_ns; // 各自的 PID 命名空间
    struct mnt_namespace *mnt_ns; // 各自的挂载命名空间
};

// Pause 容器（基础设施容器）
void create_pause_container() {
    // 创建共享的命名空间
    unshare(CLONE_NEWNET | CLONE_NEWIPC | CLONE_NEWUTS);
    
    // Pause 容器只是持有命名空间
    while (1) {
        pause();  // 永久休眠
    }
}
```

### 12.6.3 容器间通信优化

```c
// 共享内存 IPC
struct shm_container_ipc {
    int shmid;
    void *addr;
    size_t size;
};

// Unix 域套接字
struct unix_socket_ipc {
    int sock_fd;
    struct sockaddr_un addr;
};

// 内存映射文件
struct mmap_ipc {
    int fd;
    void *addr;
    size_t length;
};
```

## 12.7 性能优化与调优

### 12.7.1 命名空间性能开销

不同命名空间的性能影响：

| 命名空间 | 性能开销 | 主要影响 |
|---------|---------|---------|
| PID | 低 (~1%) | 进程创建时的 PID 分配 |
| Network | 中 (~5%) | 网络栈遍历、veth 开销 |
| Mount | 低 (~2%) | 路径解析、挂载点查找 |
| IPC | 极低 (<1%) | IPC 对象查找 |
| UTS | 极低 (<1%) | 几乎无开销 |
| User | 中 (~3%) | 权限检查、ID 映射 |

### 12.7.2 Cgroups 性能优化

```c
// 使用 cgroup v2 的压力指标进行动态调整
struct psi_group {
    struct mutex trigger_lock;
    struct psi_trigger *triggers;
    u64 total[NR_PSI_TASK_COUNTS][NR_PSI_STATES];
    u64 avg[NR_PSI_STATES][3];  // 10s, 60s, 300s
};

// 内存压力监控
void monitor_memory_pressure(struct cgroup *cgrp) {
    struct psi_group *psi = cgrp->psi;
    u64 pressure = psi->avg[PSI_MEM_SOME][0];  // 10s average
    
    if (pressure > THRESHOLD_HIGH) {
        // 触发内存回收
        reclaim_memory(cgrp);
    } else if (pressure < THRESHOLD_LOW) {
        // 可以增加内存限制
        increase_memory_limit(cgrp);
    }
}
```

### 12.7.3 容器启动优化

1. **镜像层缓存**：利用 overlayfs 的层次结构
2. **延迟挂载**：按需挂载文件系统
3. **预创建容器池**：提前创建待用容器
4. **checkpoint/restore**：使用 CRIU 快速恢复

```bash
# CRIU 容器检查点
criu dump -t $PID -D /checkpoint --shell-job

# 快速恢复
criu restore -D /checkpoint --shell-job
```

## 本章小结

本章深入探讨了 Linux 容器技术的内核实现机制。我们从命名空间的设计哲学出发，详细分析了 8 种命名空间的实现原理和相互关系。通过对 cgroups v1 和 v2 的对比，理解了资源控制的演进过程。容器运行时部分涵盖了从系统调用到 OCI 规范的完整实现链路。安全容器技术展示了 gVisor 和 Kata Containers 如何通过不同方式增强隔离性。

关键要点：

1. **命名空间**：提供进程视图隔离，是容器的基础
2. **Cgroups**：实现资源限制和统计，保证公平性
3. **运行时规范**：OCI 标准化了容器生态系统
4. **安全机制**：多层防护确保容器隔离性
5. **性能权衡**：隔离性与性能的平衡设计

公式总结：

- 容器开销 = $\sum_{ns \in namespaces} overhead(ns) + cgroup\_overhead + network\_overhead$
- 内存限制 = $min(cgroup\_limit, system\_available, oom\_threshold)$
- CPU 配额 = $\frac{quota}{period} \times available\_cpus$

## 练习题

### 基础题

1. **命名空间理解**
   - 解释 PID 命名空间的层次结构，为什么容器内的进程在主机上有不同的 PID？
   - **Hint**: 考虑 struct pid 的 level 字段和 upid 数组
   <details>
   <summary>答案</summary>
   PID 命名空间形成树状层次结构。每个进程在其所在命名空间及所有祖先命名空间中都有 PID。struct pid 包含多个 upid 结构，每个对应一个命名空间层级。容器内 PID 1 的进程在主机命名空间可能是 PID 5679，通过 pid->numbers[] 数组维护多个 PID 映射。
   </details>

2. **Cgroups 版本对比**
   - 比较 cgroups v1 和 v2 的主要区别，说明 v2 解决了 v1 的哪些问题？
   - **Hint**: 考虑层次结构、控制器管理和资源模型
   <details>
   <summary>答案</summary>
   v1 问题：多个层次结构导致管理复杂、控制器间缺乏协调、无统一的资源压力指标。v2 改进：统一层次结构、所有控制器共享同一树、引入 PSI 压力指标、更好的资源模型（基于权重而非限制）、支持 eBPF 程序进行细粒度控制。
   </details>

3. **容器网络基础**
   - 描述 veth pair 的工作原理，以及容器如何通过 docker0 网桥访问外网
   - **Hint**: 考虑数据包在命名空间间的流动路径
   <details>
   <summary>答案</summary>
   veth pair 是成对的虚拟网卡，数据从一端进入会从另一端出来。容器 eth0 连接到 veth 一端，另一端连接到 docker0 网桥。出站流量：容器 eth0 → veth → docker0 → iptables NAT → 主机 eth0。入站流量反向，通过 iptables DNAT 规则转发到容器。
   </details>

4. **系统调用序列**
   - 列出创建一个最小容器需要的系统调用序列，并说明每个调用的作用
   - **Hint**: clone、unshare、setns、mount、pivot_root
   <details>
   <summary>答案</summary>
   1. clone(CLONE_NEW*): 创建新进程和命名空间
   2. setns(): 加入已有命名空间
   3. unshare(): 创建新命名空间
   4. mount(): 挂载 proc、sys 等伪文件系统
   5. pivot_root(): 切换根文件系统
   6. prctl(PR_SET_NO_NEW_PRIVS): 设置安全属性
   7. setrlimit(): 设置资源限制
   8. execve(): 执行容器进程
   </details>

### 挑战题

5. **实现简单容器**
   - 设计并实现一个最小容器运行时，支持进程隔离和简单的资源限制
   - **Hint**: 使用 clone() 创建命名空间，通过 /sys/fs/cgroup 设置限制
   <details>
   <summary>答案</summary>
   核心步骤：
   1. 使用 clone(CLONE_NEWPID|CLONE_NEWNS|CLONE_NEWNET) 创建隔离环境
   2. 在子进程中 mount proc、创建设备节点
   3. 使用 pivot_root 切换根目录
   4. 写入 cgroup 文件设置内存和 CPU 限制
   5. drop capabilities 降低权限
   6. execve 运行用户程序
   关键点：处理好父子进程通信、正确设置挂载传播属性、清理环境变量
   </details>

6. **性能分析题**
   - 分析为什么 Kubernetes Pod 中的容器共享网络命名空间而不共享 PID 命名空间？这种设计的权衡是什么？
   - **Hint**: 考虑容器间通信、进程隔离、故障域
   <details>
   <summary>答案</summary>
   共享网络命名空间优势：1) localhost 通信无需经过网络栈 2) 共享端口空间便于服务发现 3) 减少网络配置复杂度。
   不共享 PID 命名空间原因：1) 进程隔离防止相互干扰 2) 独立的 PID 1 处理信号和孤儿进程 3) 故障隔离，一个容器崩溃不影响其他。
   权衡：网络性能 vs 进程隔离，通过 pause 容器持有共享命名空间实现最佳平衡。
   </details>

7. **安全设计题**
   - 设计一个容器安全方案，要求：防止容器逃逸、限制系统调用、实现细粒度权限控制
   - **Hint**: 组合使用 user namespace、seccomp、capabilities、LSM
   <details>
   <summary>答案</summary>
   多层防御方案：
   1. User namespace: 容器 root 映射为主机普通用户(uid 100000)
   2. Seccomp-BPF: 白名单过滤危险系统调用(禁止 mount、ptrace 等)
   3. Capabilities: 仅保留必要权能，drop CAP_SYS_ADMIN 等
   4. AppArmor/SELinux: 强制访问控制策略
   5. 只读根文件系统 + tmpfs
   6. 资源限制: cgroups 防止 DoS
   7. 网络隔离: 独立网络命名空间 + 防火墙规则
   实施顺序：先宽后严，逐步收紧权限
   </details>

8. **优化实践题**
   - 某容器化应用启动时间过长（>10秒），请分析可能的原因并提出优化方案
   - **Hint**: 镜像层、文件系统、网络配置、进程初始化
   <details>
   <summary>答案</summary>
   瓶颈分析：
   1. 镜像拉取：使用本地镜像缓存、镜像预热
   2. 层解压：优化层数量、使用 lazy pulling
   3. 文件系统：overlayfs 优化、减少层数
   4. 网络初始化：预创建网络命名空间池
   5. 进程启动：应用预热、JIT 预编译
   优化方案：
   - 使用 CRIU checkpoint/restore
   - 实现容器池预创建
   - 镜像分层优化，基础层共享
   - 延迟加载非关键资源
   - 并行化初始化步骤
   预期效果：启动时间降至 1-2 秒
   </details>

## 常见陷阱与错误

### 命名空间相关

1. **PID 命名空间的 init 进程**
   - 错误：忘记处理容器内的 PID 1 信号
   - 后果：子进程变成僵尸进程
   - 解决：PID 1 必须正确处理 SIGCHLD，回收子进程

2. **User namespace UID 映射**
   - 错误：先写 gid_map 再写 uid_map
   - 后果：权限错误，映射失败
   - 解决：必须先写 uid_map，且需要 CAP_SETUID 权能

3. **Mount 命名空间传播**
   - 错误：未设置正确的挂载传播类型
   - 后果：容器内挂载影响主机
   - 解决：使用 MS_PRIVATE 或 MS_SLAVE 防止传播

### Cgroups 相关

4. **Cgroups v1/v2 混用**
   - 错误：同时使用 v1 和 v2 控制器
   - 后果：行为不可预测，某些控制器失效
   - 解决：选择一个版本，避免混用

5. **内存限制与 OOM**
   - 错误：设置过低的内存限制
   - 后果：频繁 OOM，容器被杀
   - 解决：监控内存使用，设置合理的限制和 swap

### 网络相关

6. **veth 设备清理**
   - 错误：容器退出后 veth 设备残留
   - 后果：设备名冲突，资源泄露
   - 解决：确保容器退出时清理网络资源

### 安全相关

7. **Capabilities 过度授权**
   - 错误：保留 CAP_SYS_ADMIN
   - 后果：容器可以执行特权操作
   - 解决：最小权限原则，只保留必需权能

8. **Seccomp 规则顺序**
   - 错误：默认允许，再禁用特定调用
   - 后果：新的危险调用未被过滤
   - 解决：默认拒绝，白名单允许

## 最佳实践检查清单

### 容器设计审查

- [ ] **资源限制**：设置了内存、CPU、PID 数量限制
- [ ] **安全配置**：启用 user namespace，正确配置 UID 映射
- [ ] **系统调用过滤**：应用了 seccomp 规则
- [ ] **权能最小化**：仅保留必要的 capabilities
- [ ] **文件系统**：根文件系统只读，敏感路径使用 tmpfs
- [ ] **网络隔离**：独立网络命名空间，合理的防火墙规则

### 性能优化审查

- [ ] **启动优化**：镜像层优化，使用缓存
- [ ] **网络性能**：合理配置 veth 队列和缓冲区
- [ ] **存储优化**：选择合适的存储驱动（overlay2）
- [ ] **CPU 亲和性**：关键容器绑定 CPU
- [ ] **内存局部性**：NUMA 感知的容器调度

### 运维实践审查

- [ ] **日志管理**：日志轮转，避免填满磁盘
- [ ] **监控告警**：资源使用、错误率监控
- [ ] **备份恢复**：数据持久化策略
- [ ] **更新策略**：滚动更新，回滚机制
- [ ] **故障处理**：健康检查，自动重启
- [ ] **安全更新**：及时更新基础镜像和运行时