# 第13章：安全子系统

## 本章导读

Linux内核安全子系统是保护系统免受恶意攻击和误用的关键防线。从早期简单的Unix权限模型，到现代复杂的强制访问控制（MAC）和细粒度权能系统，Linux安全架构经历了深刻的演变。本章将深入剖析Linux安全模块（LSM）框架的设计与实现，探讨主流安全模块如SELinux、AppArmor的工作原理，分析权能系统的细粒度权限控制机制，以及eBPF在安全策略实施中的革命性应用。通过学习本章，读者将掌握内核安全机制的核心原理，理解不同安全模型的权衡，并能够设计和实现自定义的安全策略。

## 学习目标

完成本章学习后，您将能够：

1. **理解LSM框架架构**：掌握安全钩子的设计原理和实现机制
2. **分析主流安全模块**：对比SELinux、AppArmor、SMACK的设计哲学
3. **掌握权能系统**：理解细粒度权限控制和最小权限原则
4. **应用eBPF安全**：使用eBPF实现动态安全策略
5. **识别威胁模型**：分析内核安全威胁和防护机制
6. **实践安全加固**：配置和调试安全模块，实现纵深防御

## 13.1 Linux安全模块（LSM）框架

### 13.1.1 LSM的诞生背景

Linux安全模块框架诞生于2001年，源于Linux内核安全峰会的共识：内核需要支持多种安全模型，但不应该强制使用任何特定的安全策略。这一设计理念体现了Linux的开放性和灵活性。

```
传统DAC模型的局限性：
┌─────────────────────────────────────┐
│     自主访问控制（DAC）             │
├─────────────────────────────────────┤
│  • 基于文件所有者和权限位           │
│  • 用户可以修改自己文件的权限       │
│  • root用户拥有无限权限             │
│  • 无法防止信息泄露和提权攻击       │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│     强制访问控制（MAC）需求         │
├─────────────────────────────────────┤
│  • 系统强制执行安全策略             │
│  • 用户无法修改安全标签             │
│  • 细粒度的访问控制                 │
│  • 最小权限原则                     │
└─────────────────────────────────────┘
```

### 13.1.2 LSM框架架构

LSM框架通过在内核关键操作点插入安全钩子（security hooks），允许安全模块实施额外的访问控制检查：

```
内核操作流程与LSM钩子：
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  系统调用    │────▶│   DAC检查    │────▶│   LSM钩子    │
└──────────────┘     └──────────────┘     └──────────────┘
                            │                      │
                            ▼                      ▼
                     ┌──────────────┐     ┌──────────────┐
                     │  权限拒绝    │     │  安全模块    │
                     └──────────────┘     │    检查      │
                                          └──────────────┘
                                                   │
                                          ┌────────┴────────┐
                                          ▼                 ▼
                                   ┌──────────────┐ ┌──────────────┐
                                   │   允许操作   │ │   拒绝操作   │
                                   └──────────────┘ └──────────────┘
```

### 13.1.3 LSM钩子实现机制

LSM框架在内核中定义了大量的安全钩子，覆盖文件系统、进程管理、网络、IPC等所有关键子系统：

```c
// include/linux/lsm_hooks.h 中的钩子定义
struct security_hook_list {
    struct hlist_node       list;
    struct hlist_head       *head;
    union security_list_options hook;
    const char              *lsm;  // 安全模块名称
};

// 文件操作相关的钩子示例
union security_list_options {
    // 文件打开钩子
    int (*file_open)(struct file *file);
    
    // 文件权限检查钩子
    int (*file_permission)(struct file *file, int mask);
    
    // 进程执行钩子
    int (*bprm_check_security)(struct linux_binprm *bprm);
    
    // 任务创建钩子
    int (*task_create)(unsigned long clone_flags);
    
    // 网络套接字创建钩子
    int (*socket_create)(int family, int type, int protocol, int kern);
    
    // ... 数百个钩子函数指针
};
```

钩子的调用流程采用链式结构，允许多个安全模块串联（stacking）：

```
钩子调用链：
┌─────────────┐
│  内核函数   │
└──────┬──────┘
       │
       ▼
security_file_open()
       │
       ▼
┌─────────────────────────────┐
│  遍历 security_hook_heads   │
└─────────────┬───────────────┘
              │
    ┌─────────┴─────────┬─────────┬─────────┐
    ▼                   ▼         ▼         ▼
┌────────┐        ┌────────┐ ┌────────┐ ┌────────┐
│SELinux │        │AppArmor│ │ SMACK  │ │Capability│
│ hook   │        │  hook  │ │  hook  │ │   hook   │
└────────┘        └────────┘ └────────┘ └────────┘
    │                   │         │         │
    └─────────┬─────────┴─────────┴─────────┘
              ▼
        返回访问决策
```

### 13.1.4 LSM的关键数据结构

每个安全模块需要维护自己的安全上下文，LSM框架提供了灵活的安全blob机制：

```c
// 任务安全信息
struct task_security_struct {
    u32 osid;           // 对象安全标识符
    u32 sid;            // 主体安全标识符
    u32 exec_sid;       // 执行域SID
    u32 create_sid;     // 创建文件的SID
    u32 keycreate_sid;  // 创建密钥的SID
    u32 sockcreate_sid; // 创建套接字的SID
};

// inode安全信息
struct inode_security_struct {
    struct inode *inode;    // 反向指针
    struct list_head list;  // 用于延迟初始化
    u32 task_sid;          // 创建任务的SID
    u32 sid;               // inode的SID
    u16 sclass;            // 安全类别
    unsigned char initialized;  // 初始化状态
    spinlock_t lock;
};
```

## 13.2 主流安全模块详解

### 13.2.1 SELinux（Security-Enhanced Linux）

SELinux是最复杂也是最强大的Linux安全模块，由美国国家安全局（NSA）开发，实现了类型强制（Type Enforcement）和多级安全（MLS）模型。

#### 类型强制（TE）模型

SELinux的核心是基于类型的访问控制，每个进程和对象都被赋予一个安全上下文：

```
SELinux安全上下文格式：
user:role:type:level

示例：
system_u:system_r:httpd_t:s0       # Apache进程
unconfined_u:object_r:user_home_t:s0  # 用户主目录文件

访问控制决策：
┌────────────────┐        ┌────────────────┐
│   主体(Subject) │        │   客体(Object)  │
│   httpd_t      │───────▶│   httpd_config_t│
└────────────────┘        └────────────────┘
         │                         │
         └────────┬────────────────┘
                  ▼
          ┌──────────────┐
          │  策略规则    │
          │ allow httpd_t│
          │ httpd_config_t│
          │ :file read;  │
          └──────────────┘
```

#### 策略语言与编译

SELinux策略使用特定的策略语言编写，然后编译成二进制格式加载到内核：

```
# 类型声明
type httpd_t;
type httpd_exec_t;
type httpd_config_t;
type httpd_log_t;

# 域转换规则
type_transition init_t httpd_exec_t:process httpd_t;

# 访问向量规则
allow httpd_t httpd_config_t:file { read getattr };
allow httpd_t httpd_log_t:file { create write append };
allow httpd_t self:tcp_socket { create bind listen accept };

# 宏定义简化策略编写
define(`apache_domain', `
    type $1_t;
    type $1_exec_t;
    domain_type($1_t)
    domain_entry_file($1_t, $1_exec_t)
')
```

#### AVC（Access Vector Cache）

为了提高性能，SELinux实现了访问向量缓存：

```c
struct avc_node {
    struct avc_entry    ae;      // 访问向量条目
    struct hlist_node   list;    // 哈希链表
    struct rcu_head     rhead;   // RCU头
};

struct avc_entry {
    u32         ssid;       // 源安全标识符
    u32         tsid;       // 目标安全标识符
    u16         tclass;     // 目标类别
    struct av_decision  avd;  // 访问决策
};

// AVC查询流程
static inline int avc_has_perm(u32 ssid, u32 tsid,
                               u16 tclass, u32 requested,
                               struct common_audit_data *auditdata)
{
    struct av_decision avd;
    int rc;
    
    // 快速路径：查询缓存
    rc = avc_has_perm_noaudit(ssid, tsid, tclass, requested, 0, &avd);
    
    // 慢速路径：查询策略
    if (rc)
        rc = security_compute_av(ssid, tsid, tclass, requested, &avd);
    
    // 审计记录
    avc_audit(ssid, tsid, tclass, requested, &avd, rc, auditdata);
    
    return rc;
}
```

### 13.2.2 AppArmor

AppArmor采用基于路径的访问控制，相比SELinux更容易理解和配置：

```
AppArmor配置文件示例：
#include <tunables/global>

/usr/sbin/nginx {
    #include <abstractions/base>
    #include <abstractions/nameservice>
    
    capability net_bind_service,
    capability setuid,
    capability setgid,
    
    /etc/nginx/ r,
    /etc/nginx/** r,
    /var/log/nginx/*.log w,
    /var/www/html/** r,
    
    network inet stream,
    network inet6 stream,
    
    # 子配置文件
    ^/usr/sbin/nginx-worker {
        /var/www/html/** r,
        /tmp/ r,
        /tmp/** rw,
    }
}
```

AppArmor的实现相对简单，主要依赖于DFA（确定有限自动机）进行路径匹配：

```
路径匹配DFA：
        /etc
         │
    ┌────┴────┐
    │         │
  nginx/    passwd
    │         │
  *.conf    [accept]
    │
  [accept]
```

### 13.2.3 SMACK（Simplified Mandatory Access Control Kernel）

SMACK提供了更简单的标签式访问控制：

```
SMACK规则格式：
主体标签  客体标签  访问权限

示例：
WebServer Database  r     # WebServer可以读Database
User      WebServer wx    # User可以写和执行WebServer
System    _         rwxa  # System对所有对象有完全权限
```

## 13.3 Linux权能系统（Capabilities）

### 13.3.1 权能的设计理念

Linux权能系统将传统的超级用户权限分解为细粒度的权能，实现最小权限原则：

```
传统模型 vs 权能模型：

传统模型：                    权能模型：
┌──────────────┐           ┌──────────────────┐
│    root      │           │   CAP_NET_ADMIN  │
│  (所有权限)   │   ───▶    │   CAP_SYS_ADMIN  │
└──────────────┘           │   CAP_DAC_OVERRIDE│
                           │   CAP_SETUID      │
┌──────────────┐           │   ...            │
│  普通用户     │           │   (38个权能)      │
│  (受限权限)   │           └──────────────────┘
└──────────────┘
```

### 13.3.2 权能的实现机制

每个进程维护多个权能集合：

```c
// include/linux/cred.h
struct cred {
    // ... 其他字段
    kernel_cap_t    cap_inheritable;  // 可继承权能集
    kernel_cap_t    cap_permitted;    // 允许权能集
    kernel_cap_t    cap_effective;    // 有效权能集
    kernel_cap_t    cap_bset;        // 边界集
    kernel_cap_t    cap_ambient;     // 环境权能集
};

// 权能检查宏
#define capable(cap) (capable_wrt_inode_uidgid(current_cred(), \
                     NULL, NULL, cap))

// 文件权能存储在扩展属性中
struct vfs_cap_data {
    __le32 magic_etc;            // 魔数和标志
    struct {
        __le32 permitted;        // 允许权能
        __le32 inheritable;      // 可继承权能
    } data[VFS_CAP_U32];
};
```

权能的传递规则遵循复杂的公式：

```
P'(ambient)     = (file caps || P(ambient))
P'(permitted)   = (P(inheritable) & F(inheritable)) | 
                  (F(permitted) & cap_bset) | P'(ambient)
P'(effective)   = F(effective) ? P'(permitted) : P'(ambient)
P'(inheritable) = P(inheritable)
P'(bset)        = P(bset)

其中：
P  = 父进程权能
P' = 子进程权能
F  = 文件权能
```

### 13.3.3 常用权能详解

```c
// 网络管理权能
CAP_NET_ADMIN       // 配置网络接口、路由表、防火墙规则
CAP_NET_BIND_SERVICE // 绑定特权端口（<1024）
CAP_NET_RAW         // 使用原始套接字和包套接字

// 系统管理权能
CAP_SYS_ADMIN       // 执行系统管理操作（最强大的权能）
CAP_SYS_MODULE      // 加载和卸载内核模块
CAP_SYS_PTRACE      // 追踪任意进程
CAP_SYS_CHROOT      // 使用chroot()

// 文件系统权能
CAP_DAC_OVERRIDE    // 绕过文件读、写、执行权限检查
CAP_DAC_READ_SEARCH // 绕过文件读权限和目录搜索权限检查
CAP_FOWNER          // 绕过文件所有者检查
CAP_CHOWN           // 改变文件所有者

// 进程管理权能
CAP_SETUID          // 设置UID
CAP_SETGID          // 设置GID
CAP_KILL            // 发送信号给任意进程
CAP_SETPCAP         // 修改进程权能
```

## 13.4 eBPF与安全策略

### 13.4.1 eBPF在安全中的应用

eBPF（extended Berkeley Packet Filter）为Linux安全提供了可编程的、动态的安全策略实施机制：

```c
// 使用eBPF实现系统调用过滤
SEC("lsm/file_open")
int BPF_PROG(file_open_security, struct file *file)
{
    struct task_struct *task = (struct task_struct *)bpf_get_current_task();
    char comm[16];
    bpf_get_current_comm(&comm, sizeof(comm));
    
    // 获取文件路径
    char path[256];
    struct dentry *dentry = file->f_path.dentry;
    bpf_d_path(&file->f_path, path, sizeof(path));
    
    // 安全策略：禁止特定进程访问敏感文件
    if (strncmp(comm, "suspicious", 10) == 0) {
        if (strstr(path, "/etc/shadow")) {
            bpf_printk("Blocked: %s accessing %s\n", comm, path);
            return -EPERM;
        }
    }
    
    return 0;
}
```

### 13.4.2 BPF LSM框架

BPF LSM允许通过eBPF程序实现LSM钩子：

```
BPF LSM架构：
┌──────────────────────────────┐
│      用户空间BPF程序          │
└────────────┬─────────────────┘
             │ bpf()系统调用
             ▼
┌──────────────────────────────┐
│      BPF验证器               │
│  • 类型检查                  │
│  • 边界检查                  │
│  • 循环检测                  │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│      BPF JIT编译器           │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│      LSM钩子点               │
│  • file_open                 │
│  • task_alloc                │
│  • socket_connect            │
└──────────────────────────────┘
```

### 13.4.3 使用eBPF进行运行时安全监控

```c
// 监控进程创建
SEC("tracepoint/sched/sched_process_fork")
int trace_fork(struct trace_event_raw_sched_process_fork *ctx)
{
    u32 pid = ctx->parent_pid;
    u32 child_pid = ctx->child_pid;
    
    // 记录进程树
    struct process_info info = {
        .parent_pid = pid,
        .child_pid = child_pid,
        .timestamp = bpf_ktime_get_ns(),
    };
    
    bpf_map_update_elem(&process_tree, &child_pid, &info, BPF_ANY);
    
    // 检测异常fork行为
    u64 *fork_count = bpf_map_lookup_elem(&fork_stats, &pid);
    if (fork_count && *fork_count > FORK_THRESHOLD) {
        // 触发告警
        bpf_send_signal(SIGKILL);
    }
    
    return 0;
}

// 监控网络连接
SEC("lsm/socket_connect")
int BPF_PROG(socket_connect_security, struct socket *sock,
             struct sockaddr *address, int addrlen)
{
    if (address->sa_family != AF_INET)
        return 0;
        
    struct sockaddr_in *addr = (struct sockaddr_in *)address;
    u32 dest_ip = addr->sin_addr.s_addr;
    u16 dest_port = ntohs(addr->sin_port);
    
    // 检查黑名单
    if (bpf_map_lookup_elem(&blacklist_ips, &dest_ip)) {
        bpf_printk("Blocked connection to %pI4:%d\n", &dest_ip, dest_port);
        return -EPERM;
    }
    
    return 0;
}
```

## 13.5 内核安全威胁模型

### 13.5.1 常见攻击向量

Linux内核面临多种安全威胁：

```
攻击向量分类：
┌─────────────────────────────────────────┐
│            内核攻击面                    │
├─────────────────────────────────────────┤
│  1. 系统调用接口                        │
│     • 输入验证不足                      │
│     • 竞态条件                          │
│     • 整数溢出                          │
│                                         │
│  2. 设备驱动                            │
│     • 未初始化内存                      │
│     • 缓冲区溢出                        │
│     • UAF（Use-After-Free）             │
│                                         │
│  3. 网络协议栈                          │
│     • 包处理漏洞                        │
│     • 协议状态机错误                    │
│     • 拒绝服务攻击                      │
│                                         │
│  4. 文件系统                            │
│     • 元数据损坏                        │
│     • 符号链接攻击                      │
│     • 权限绕过                          │
└─────────────────────────────────────────┘
```

### 13.5.2 权限提升攻击

权限提升是最严重的安全威胁之一：

```c
// 典型的权限提升漏洞模式
struct vulnerable_ioctl_data {
    void __user *user_ptr;
    size_t size;
};

// 错误的权限检查
static long vulnerable_ioctl(struct file *file, unsigned int cmd,
                            unsigned long arg)
{
    struct vulnerable_ioctl_data data;
    
    // 缺少权限检查！
    if (copy_from_user(&data, (void __user *)arg, sizeof(data)))
        return -EFAULT;
    
    // 危险：直接使用用户提供的指针
    memcpy(data.user_ptr, kernel_secret, data.size);  // 信息泄露
    
    return 0;
}

// 正确的实现
static long secure_ioctl(struct file *file, unsigned int cmd,
                         unsigned long arg)
{
    struct vulnerable_ioctl_data data;
    
    // 权限检查
    if (!capable(CAP_SYS_ADMIN))
        return -EPERM;
    
    // 输入验证
    if (copy_from_user(&data, (void __user *)arg, sizeof(data)))
        return -EFAULT;
    
    if (data.size > MAX_SIZE)
        return -EINVAL;
    
    // 使用安全的内核API
    if (copy_to_user(data.user_ptr, safe_data, data.size))
        return -EFAULT;
    
    return 0;
}
```

### 13.5.3 侧信道攻击

现代CPU的侧信道漏洞对内核安全构成新挑战：

```
Spectre/Meltdown缓解措施：

1. 页表隔离（PTI）：
   ┌──────────────┐    ┌──────────────┐
   │  用户页表     │    │  内核页表     │
   │              │    │              │
   │  用户空间映射 │    │  用户空间映射 │
   │              │    │  +           │
   │              │    │  内核空间映射 │
   └──────────────┘    └──────────────┘
   
2. 推测执行屏障：
   asm volatile("lfence" ::: "memory");  // Intel
   asm volatile("dsb sy" ::: "memory");  // ARM
   
3. 间接分支预测屏障（IBPB）：
   wrmsrl(MSR_IA32_PRED_CMD, PRED_CMD_IBPB);
```

### 13.5.4 内存损坏防护

内核实现了多种内存损坏防护机制：

```c
// KASLR（内核地址空间布局随机化）
// arch/x86/boot/compressed/kaslr.c
static unsigned long find_random_phys_addr(unsigned long minimum,
                                          unsigned long image_size)
{
    unsigned long random_addr;
    
    // 获取随机数
    random_addr = kaslr_get_random_long("Physical");
    
    // 对齐到内核对齐边界
    random_addr &= ~(CONFIG_PHYSICAL_ALIGN - 1);
    
    // 确保在有效范围内
    if (random_addr < minimum)
        random_addr = minimum;
    
    return random_addr;
}

// 栈溢出保护（Stack Canary）
void __stack_chk_fail(void)
{
    panic("stack-protector: Kernel stack is corrupted in: %pS\n",
          __builtin_return_address(0));
}

// KFENCE（内核电栅栏）
static void kfence_guarded_alloc(void *addr, size_t size)
{
    // 在分配区域前后设置保护页
    set_memory_valid((unsigned long)addr - PAGE_SIZE, 1, false);
    set_memory_valid((unsigned long)addr + size, 1, false);
}

## 13.6 历史故事：Linux安全的演进之路

### 13.6.1 NSA贡献SELinux的争议

2000年，美国国家安全局（NSA）向Linux社区贡献SELinux代码时引发巨大争议。许多开发者质疑："为什么NSA要帮助加强Linux安全？是否有后门？"

Linus Torvalds最初的态度是谨慎的："我不会接受任何我不理解的安全代码。"经过长达3年的代码审查和重构，SELinux最终在2003年进入2.6内核。这个过程中，LSM框架应运而生——它允许SELinux作为可选模块存在，而不是强制所有人使用。

有趣的是，Edward Snowden在2013年的爆料显示，NSA确实有各种监控项目，但没有证据表明SELinux存在后门。相反，SELinux的开源特性使其成为最透明、最被审查的安全系统之一。

### 13.6.2 Linus对安全框架的态度转变

早期的Linus对复杂的安全框架持怀疑态度。他在2001年的邮件中写道："安全不应该妨碍正常使用。如果安全措施让系统变得难用，人们就会关闭它。"

这种实用主义哲学深刻影响了Linux安全架构的设计：
- 安全功能必须是可选的
- 默认配置不能破坏现有系统
- 性能影响必须最小化

随着云计算和容器技术的兴起，Linus的态度有所软化。2018年，他在一次采访中承认："安全已经变得如此重要，我们不能再把它当作次要功能。"

### 13.6.3 权能系统的漫长征程

Linux权能系统的实现经历了近20年的演进：

- **1997年**：首次引入权能概念（2.1内核）
- **2008年**：文件权能支持（2.6.24）
- **2015年**：环境权能（Ambient Capabilities）引入（4.3内核）

Andy Lutomirski在实现环境权能时说："权能系统的问题是它太复杂了。我们需要让它对普通开发者更友好。"这促成了环境权能的设计，使得非root进程也能保持某些权能。

## 13.7 高级话题：现代内核安全技术

### 13.7.1 内核自保护项目（KSPP）

Kernel Self Protection Project由Kees Cook领导，致力于将各种安全加固技术整合到主线内核：

```c
// 控制流完整性（CFI）
// 编译器在间接调用点插入类型检查
static void cfi_check_fn(void (*fn)(int), int arg)
{
    // 编译器生成的CFI检查
    if (!__cfi_check(fn, CFI_TYPE_FN))
        __cfi_fail();
    
    fn(arg);  // 安全的间接调用
}

// 内核地址空间隔离（KASI）
struct kasi_struct {
    struct mm_struct *isolated_mm;  // 隔离的地址空间
    pte_t *isolated_ptes;           // 独立的页表项
    unsigned long isolated_start;   // 隔离区域起始
    unsigned long isolated_size;    // 隔离区域大小
};

// 初始化完整性测量（IMA）
static int ima_calc_file_hash(struct file *file, struct ima_digest_data *hash)
{
    loff_t i_size = i_size_read(file_inode(file));
    struct crypto_shash *tfm;
    struct shash_desc *desc;
    
    tfm = ima_alloc_tfm(hash->algo);
    desc = ima_alloc_desc(tfm);
    
    // 计算文件哈希
    rc = crypto_shash_init(desc);
    while (offset < i_size) {
        rbuf_len = min(i_size - offset, rbuf_size);
        rc = integrity_kernel_read(file, offset, rbuf, rbuf_len);
        rc = crypto_shash_update(desc, rbuf, rbuf_len);
        offset += rbuf_len;
    }
    
    rc = crypto_shash_final(desc, hash->digest);
    return rc;
}
```

### 13.7.2 基于硬件的安全特性

现代处理器提供了越来越多的安全特性：

```
Intel CET（控制流强制技术）：
┌─────────────────────────────────┐
│      Shadow Stack               │
├─────────────────────────────────┤
│  • 硬件维护的返回地址栈         │
│  • 防止ROP攻击                  │
│  • 自动验证返回地址             │
└─────────────────────────────────┘

ARM Pointer Authentication：
┌─────────────────────────────────┐
│   指针认证码（PAC）              │
├─────────────────────────────────┤
│  • 在指针高位嵌入认证码         │
│  • 使用密钥生成和验证           │
│  • 防止指针篡改                 │
└─────────────────────────────────┘
```

### 13.7.3 容器安全与沙箱技术

```c
// seccomp-bpf沙箱实现
static int install_seccomp_filter(void)
{
    struct sock_filter filter[] = {
        // 检查系统调用号
        BPF_STMT(BPF_LD | BPF_W | BPF_ABS,
                offsetof(struct seccomp_data, nr)),
        
        // 允许的系统调用白名单
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_read, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_write, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        
        BPF_JUMP(BPF_JMP | BPF_JEQ | BPF_K, __NR_exit, 0, 1),
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_ALLOW),
        
        // 默认拒绝
        BPF_STMT(BPF_RET | BPF_K, SECCOMP_RET_KILL),
    };
    
    struct sock_fprog prog = {
        .len = ARRAY_SIZE(filter),
        .filter = filter,
    };
    
    // 安装过滤器
    if (prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0))
        return -1;
        
    if (prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog))
        return -1;
        
    return 0;
}

// Landlock LSM - 用户空间沙箱
struct landlock_ruleset_attr {
    __u64 handled_access_fs;  // 文件系统访问权限
    __u64 handled_access_net; // 网络访问权限
};

static int create_landlock_sandbox(void)
{
    struct landlock_ruleset_attr attr = {
        .handled_access_fs = LANDLOCK_ACCESS_FS_READ_FILE |
                            LANDLOCK_ACCESS_FS_READ_DIR,
    };
    
    int ruleset_fd = landlock_create_ruleset(&attr, sizeof(attr), 0);
    
    // 添加规则
    struct landlock_path_beneath_attr path_beneath = {
        .allowed_access = LANDLOCK_ACCESS_FS_READ_FILE,
        .parent_fd = open("/usr", O_PATH | O_CLOEXEC),
    };
    
    landlock_add_rule(ruleset_fd, LANDLOCK_RULE_PATH_BENEATH,
                      &path_beneath, 0);
    
    // 应用限制
    landlock_restrict_self(ruleset_fd, 0);
    
    return 0;
}
```

## 13.8 本章小结

Linux内核安全子系统是一个复杂而精妙的体系，从LSM框架的灵活架构，到SELinux的强制访问控制，从细粒度的权能系统，到可编程的eBPF安全策略，每一层都体现了深思熟虑的设计。

**核心要点回顾：**

1. **LSM框架**提供了统一的安全钩子机制，支持多种安全模型共存
2. **SELinux**实现了最严格的强制访问控制，适用于高安全需求场景
3. **AppArmor**采用基于路径的访问控制，配置更加直观
4. **权能系统**将root权限细分，实现最小权限原则
5. **eBPF安全**提供了动态、可编程的安全策略实施
6. **威胁模型**涵盖权限提升、侧信道、内存损坏等多种攻击向量

**关键公式：**

访问控制决策：
$$\text{Decision} = \text{DAC}(S,O,P) \land \text{MAC}(S,O,P) \land \text{Capability}(S,P)$$

其中：
- $S$ = 主体（Subject）
- $O$ = 客体（Object）  
- $P$ = 权限（Permission）

权能传递公式：
$$P'_{permitted} = (P_{inheritable} \cap F_{inheritable}) \cup (F_{permitted} \cap \text{cap\_bset}) \cup P'_{ambient}$$

安全强度评估：
$$\text{Security\_Strength} = \sum_{i=1}^{n} w_i \cdot \text{Control}_i$$

其中$w_i$为各安全控制的权重。

## 13.9 练习题

### 基础题

**练习13.1**：解释LSM框架中security_hook_heads的作用，以及它如何支持多个安全模块的串联。

<details>
<summary>提示</summary>
考虑钩子链表的遍历顺序和返回值处理机制。
</details>

<details>
<summary>答案</summary>
security_hook_heads是一个包含所有安全钩子链表头的结构。每个钩子位置可以注册多个安全模块的处理函数，形成链表。调用时按注册顺序遍历，任何模块返回错误都会终止操作。这种设计允许多个安全模块协同工作，实现纵深防御。
</details>

**练习13.2**：编写一个简单的AppArmor配置文件，限制nginx只能访问/var/www目录和监听80端口。

<details>
<summary>提示</summary>
需要考虑文件访问权限、网络权限和必要的系统资源。
</details>

<details>
<summary>答案</summary>
配置包括：/var/www/** r权限用于读取网页文件，/var/log/nginx/* w权限用于写日志，capability net_bind_service用于绑定80端口，network inet stream用于TCP连接。还需包含基础抽象如abstractions/base。
</details>

**练习13.3**：说明为什么CAP_SYS_ADMIN被称为"新的root"，并给出三个应该避免授予此权能的原因。

<details>
<summary>提示</summary>
查看CAP_SYS_ADMIN允许的操作列表。
</details>

<details>
<summary>答案</summary>
CAP_SYS_ADMIN允许挂载文件系统、配置交换空间、执行各种系统管理操作，几乎等同于root。应避免授予因为：1)违背最小权限原则，2)可能通过挂载绕过其他安全限制，3)增大攻击面。应该使用更具体的权能如CAP_NET_ADMIN或CAP_SYS_MODULE。
</details>

### 挑战题

**练习13.4**：设计一个基于eBPF的安全监控系统，检测并阻止进程的异常fork行为（fork炸弹攻击）。

<details>
<summary>提示</summary>
需要跟踪每个进程的fork频率，设置阈值，使用BPF map存储状态。
</details>

<details>
<summary>答案</summary>
方案包括：使用BPF_MAP_TYPE_HASH存储每个PID的fork计数和时间戳，在sched_process_fork跟踪点更新计数，实现滑动窗口算法计算fork速率，超过阈值时通过bpf_send_signal发送SIGKILL。需要考虑map清理和父子进程关系。
</details>

**练习13.5**：分析Dirty COW（CVE-2016-5195）漏洞的原理，解释它如何绕过Linux的权限系统，以及内核是如何修复的。

<details>
<summary>提示</summary>
关注写时复制（COW）机制和竞态条件。
</details>

<details>
<summary>答案</summary>
Dirty COW利用了get_user_pages()和COW机制之间的竞态条件。通过madvise(MADV_DONTNEED)和写入只读映射的竞争，可以直接修改页缓存中的原始页面。修复方案是在处理COW时增加额外的标志位FOLL_COW，确保写操作总是触发页面复制，避免直接修改共享页面。
</details>

**练习13.6**：实现一个LSM模块，限制特定进程只能创建特定类型的文件（如只能创建.log文件）。

<details>
<summary>提示</summary>
需要hook文件创建相关的LSM钩子，检查文件名后缀。
</details>

<details>
<summary>答案</summary>
实现需要：1)注册inode_create和inode_mknod钩子，2)在钩子函数中获取当前进程名和目标文件名，3)使用配置的白名单检查文件扩展名，4)维护进程到允许扩展名的映射表，5)处理符号链接和硬链接的特殊情况。
</details>

**练习13.7**：设计一个基于SELinux的多租户隔离方案，确保不同租户的进程和文件完全隔离。

<details>
<summary>提示</summary>
使用MCS（Multi-Category Security）或MLS（Multi-Level Security）。
</details>

<details>
<summary>答案</summary>
方案设计：为每个租户分配唯一的类别（如c0,c1,c2），创建租户专用的类型（tenant1_t, tenant2_t），使用MCS约束确保进程只能访问相同类别的资源。策略规则包括类型转换规则、文件上下文规则和网络标签规则。还需考虑共享资源的处理和管理域的设置。
</details>

**练习13.8**：分析并比较KASLR、KPTI、CET三种内核保护机制的性能开销和安全效益。

<details>
<summary>提示</summary>
考虑各机制防护的攻击类型和实现代价。
</details>

<details>
<summary>答案</summary>
KASLR随机化内核地址，开销小（<1%），但可通过侧信道泄露。KPTI隔离页表防Meltdown，开销大（5-30%），效果确定。CET防控制流劫持，需硬件支持，开销中等（2-5%），防护ROP/JOP攻击效果好。实际部署需根据威胁模型和性能要求权衡。
</details>

## 13.10 常见陷阱与错误

### 陷阱1：过度依赖DAC权限检查

```c
// 错误：只检查DAC权限
if (capable(CAP_DAC_OVERRIDE))
    return 0;  // 危险！绕过了所有LSM检查

// 正确：DAC和LSM都要检查
ret = generic_permission(inode, mask);
if (ret)
    return ret;
return security_inode_permission(inode, mask);
```

### 陷阱2：忽视权能的继承规则

```bash
# 错误：期望子进程继承父进程的所有权能
setcap cap_net_admin+ep /usr/bin/parent
# 子进程不会自动获得cap_net_admin

# 正确：设置继承位
setcap cap_net_admin+eip /usr/bin/parent
```

### 陷阱3：SELinux上下文转换错误

```c
// 错误：忘记设置执行上下文
execve("/usr/bin/service", argv, envp);
// 进程仍在原域中运行

// 正确：确保类型转换规则存在
// type_transition init_t service_exec_t:process service_t;
```

### 陷阱4：eBPF验证器限制

```c
// 错误：无界循环
for (i = 0; i < user_provided_value; i++) {
    // BPF验证器会拒绝
}

// 正确：有界循环
#pragma unroll
for (i = 0; i < 100 && i < user_value; i++) {
    // 验证器可以证明循环有界
}
```

### 陷阱5：安全模块加载顺序

```bash
# 错误：期望后加载的模块优先级更高
insmod security_module1.ko
insmod security_module2.ko  # 不一定优先

# 正确：使用security=参数指定主模块
# 在grub中：security=selinux
```

## 13.11 最佳实践检查清单

### 设计阶段
- [ ] 明确定义安全需求和威胁模型
- [ ] 选择合适的安全模型（DAC/MAC/RBAC）
- [ ] 设计最小权限的权能集合
- [ ] 规划安全模块的部署策略
- [ ] 考虑性能影响和可用性平衡

### 实现阶段
- [ ] 正确实现LSM钩子，避免TOCTOU问题
- [ ] 使用内核加固选项（CONFIG_HARDENED_USERCOPY等）
- [ ] 实施输入验证和边界检查
- [ ] 避免信息泄露（清零敏感数据）
- [ ] 使用安全的内核API（如copy_from_user）

### 配置阶段
- [ ] 配置SELinux/AppArmor策略
- [ ] 设置适当的权能边界集
- [ ] 启用审计日志记录
- [ ] 配置seccomp过滤器
- [ ] 实施网络安全策略

### 运行阶段
- [ ] 监控安全事件和审计日志
- [ ] 定期更新安全策略
- [ ] 进行安全漏洞扫描
- [ ] 实施入侵检测系统
- [ ] 制定应急响应计划

### 审计阶段
- [ ] 审查权限和权能分配
- [ ] 检查安全策略的有效性
- [ ] 分析性能影响
- [ ] 评估安全合规性
- [ ] 更新威胁模型

---

通过本章的学习，我们深入理解了Linux内核的安全架构，从灵活的LSM框架到强大的SELinux，从细粒度的权能系统到动态的eBPF安全。安全是一个持续演进的领域，需要在可用性、性能和安全性之间找到平衡。下一章，我们将探讨实时Linux，了解如何在保证安全的同时满足严格的实时性要求。
```
