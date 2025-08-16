# 第9章：网络协议栈

## 章节大纲

### 9.1 引言
- Linux 网络栈架构概述
- 从网卡到应用程序的数据流
- 内核网络子系统的演进

### 9.2 sk_buff 与网络数据流
- 9.2.1 sk_buff 结构体详解
- 9.2.2 sk_buff 的生命周期管理
- 9.2.3 数据包在各层的处理流程
- 9.2.4 零拷贝与 scatter-gather I/O

### 9.3 TCP/IP 协议栈实现
- 9.3.1 网络层：IP 协议处理
- 9.3.2 传输层：TCP 状态机与拥塞控制
- 9.3.3 套接字层：socket API 实现
- 9.3.4 连接管理与 TIME_WAIT 优化

### 9.4 Netfilter 框架与 iptables
- 9.4.1 Netfilter 钩子机制
- 9.4.2 连接跟踪（conntrack）
- 9.4.3 NAT 实现原理
- 9.4.4 iptables 规则匹配引擎

### 9.5 高性能网络技术
- 9.5.1 NAPI：中断缓解机制
- 9.5.2 XDP：内核快速数据路径
- 9.5.3 io_uring 网络编程
- 9.5.4 eBPF 网络应用

### 9.6 本章小结

### 9.7 练习题

### 9.8 常见陷阱与错误

### 9.9 最佳实践检查清单

---

## 9.1 引言

Linux 网络协议栈是内核中最复杂、最精妙的子系统之一。从早期的简单 TCP/IP 实现，到今天支持每秒处理千万级数据包的高性能架构，Linux 网络栈经历了翻天覆地的变化。本章将深入剖析网络栈的核心数据结构、关键算法和性能优化技术，帮助读者理解从网卡硬件中断到应用程序 socket 调用的完整数据流。

### 学习目标

完成本章学习后，您将能够：

1. **理解 sk_buff 核心数据结构**：掌握网络数据包在内核中的表示和管理方式
2. **分析 TCP/IP 协议栈实现**：深入理解各层协议的处理流程和状态机制
3. **掌握 Netfilter 框架**：理解防火墙规则匹配、NAT 转换和连接跟踪原理
4. **应用高性能网络技术**：使用 NAPI、XDP、eBPF 等技术优化网络性能
5. **诊断网络性能问题**：定位和解决常见的网络瓶颈和延迟问题

### 架构概览

Linux 网络栈采用分层架构，每层都有明确的职责：

```
应用层     ┌─────────────────────────────────────┐
          │     用户空间应用程序（socket API）      │
          └─────────────────────────────────────┘
                           ↑↓
═══════════════════ 系统调用界面 ═══════════════════
                           ↑↓
套接字层   ┌─────────────────────────────────────┐
          │    BSD Socket 层（struct socket）     │
          │         协议无关的接口抽象             │
          └─────────────────────────────────────┘
                           ↑↓
传输层     ┌─────────────────────────────────────┐
          │    TCP / UDP / SCTP / DCCP          │
          │    拥塞控制、流量控制、可靠传输         │
          └─────────────────────────────────────┘
                           ↑↓
网络层     ┌─────────────────────────────────────┐
          │         IPv4 / IPv6 / ICMP           │
          │      路由选择、分片重组、转发           │
          └─────────────────────────────────────┘
                           ↑↓
Netfilter ┌─────────────────────────────────────┐
          │   防火墙钩子、NAT、连接跟踪            │
          └─────────────────────────────────────┘
                           ↑↓
链路层     ┌─────────────────────────────────────┐
          │    ARP / Neighbor Discovery          │
          │      网络设备抽象（net_device）        │
          └─────────────────────────────────────┘
                           ↑↓
驱动层     ┌─────────────────────────────────────┐
          │      网卡驱动（e1000、ixgbe等）       │
          │         DMA、中断处理、队列管理        │
          └─────────────────────────────────────┘
```

### 关键创新

Linux 网络栈的几个重要创新点：

1. **sk_buff 设计**：高效的零拷贝数据结构，支持数据包在各层间传递而无需复制
2. **NAPI 机制**：自适应的中断缓解技术，在高负载时切换到轮询模式
3. **RCU 在网络栈的应用**：无锁的路由表和连接表查找
4. **XDP 框架**：在驱动层提供可编程的快速数据路径
5. **eBPF 网络程序**：安全的内核态可编程网络处理

### 性能演进

网络栈性能的几个里程碑：

- **2.4 内核时代**：基本的 TCP/IP 实现，单核性能约 100Mbps
- **2.6 内核引入 NAPI**：中断缓解技术，千兆网卡成为可能
- **3.x 内核多队列支持**：利用多核并行处理，达到 10Gbps
- **4.x 内核 XDP 出现**：内核旁路技术，单核 14Mpps
- **5.x 内核 io_uring**：异步 I/O 革命，接近硬件极限性能

## 9.3 TCP/IP 协议栈实现

Linux 的 TCP/IP 协议栈是一个完整的、符合标准的实现，支持 IPv4、IPv6、TCP、UDP 以及众多扩展协议。本节深入分析协议栈的核心实现，包括网络层的路由机制、传输层的状态机管理、套接字层的 API 实现，以及连接管理的优化策略。

### 9.3.1 网络层：IP 协议处理

#### IP 协议架构

Linux 的 IP 层负责数据包的路由、转发、分片和重组。其核心数据结构和处理流程经过精心设计，支持高效的查找和转发：

```
接收处理流程：
netif_receive_skb() → ip_rcv() → ip_rcv_finish() → ip_local_deliver() / ip_forward()
                                        ↓
                                  路由查找（FIB）
                                        ↓
                            本地接收 / 转发 / 丢弃

发送处理流程：
ip_queue_xmit() → ip_output() → ip_finish_output() → ip_finish_output2()
        ↓              ↓                ↓                    ↓
    路由查找      Netfilter钩子     分片处理            邻居子系统
```

#### 路由子系统

Linux 使用 FIB（Forwarding Information Base）和路由缓存实现高效的路由查找：

```c
/* 路由表项 */
struct fib_info {
    struct hlist_node    fib_hash;      /* 哈希链表 */
    struct list_head     fib_lhash;     /* 链表头 */
    struct net          *fib_net;       /* 网络命名空间 */
    int                  fib_treeref;   /* 引用计数 */
    atomic_t             fib_clntref;   /* 客户端引用 */
    unsigned int         fib_flags;     /* 标志位 */
    unsigned char        fib_dead;      /* 删除标记 */
    unsigned char        fib_protocol;  /* 路由协议 */
    unsigned char        fib_scope;     /* 路由范围 */
    unsigned char        fib_type;      /* 路由类型 */
    __be32               fib_prefsrc;   /* 首选源地址 */
    u32                  fib_priority;  /* 路由优先级 */
    struct dst_metrics  *fib_metrics;   /* 路由度量 */
    int                  fib_nhs;       /* 下一跳数量 */
    struct fib_nh        fib_nh[0];     /* 下一跳数组 */
};

/* 路由查找核心函数 */
static int ip_route_input_slow(struct sk_buff *skb, __be32 daddr, 
                                __be32 saddr, u8 tos, struct net_device *dev)
{
    struct fib_result res;
    struct flowi4 fl4;
    
    /* 构造查找键 */
    fl4.daddr = daddr;
    fl4.saddr = saddr;
    fl4.flowi4_tos = tos;
    fl4.flowi4_scope = RT_SCOPE_UNIVERSE;
    
    /* FIB 查找 */
    err = fib_lookup(net, &fl4, &res, 0);
    
    /* 创建路由缓存项 */
    rth = rt_dst_alloc(dev, flags);
    rth->dst.input = ip_local_deliver;
    rth->dst.output = ip_output;
}
```

#### 路由缓存优化

Linux 使用多级缓存加速路由查找：

1. **DST 缓存**：每个 sk_buff 携带 dst_entry 指针
2. **FIB Trie**：基于 LC-Trie 的路由表查找，O(W) 复杂度
3. **路由异常缓存**：处理 PMTU、重定向等异常

```c
/* LC-Trie 节点 */
struct tnode {
    struct key_vector kv[1];
    /* LC-Trie 使用路径压缩和级别压缩优化 */
};

/* 查找算法 */
static struct key_vector *fib_find_node(struct trie *t, 
                                         struct key_vector **tp, u32 key)
{
    struct key_vector *pn, *n = t->kv;
    unsigned long index;
    
    do {
        pn = n;
        index = get_index(key, pn);
        n = get_child_rcu(pn, index);
        
        if (!n)
            break;
            
        /* 路径压缩：跳过只有一个子节点的内部节点 */
        if (IS_TNODE(n) && (n->bits < KEYLENGTH)) {
            if (tkey_sub_equals(n->key, n->pos, n->bits, key))
                continue;
            break;
        }
    } while (IS_TNODE(n));
    
    return n;
}
```

#### IP 分片与重组

当数据包大小超过 MTU 时，IP 层负责分片和重组：

```c
/* IP 分片 */
static int ip_fragment(struct net *net, struct sock *sk, 
                       struct sk_buff *skb,
                       unsigned int mtu,
                       int (*output)(struct net *, struct sock *, 
                                     struct sk_buff *))
{
    struct iphdr *iph = ip_hdr(skb);
    unsigned int hlen = iph->ihl * 4;
    unsigned int ll_rs = LL_RESERVED_SPACE(skb->dev);
    unsigned int mtu = dst_mtu(&rt->dst);
    
    /* 计算每个分片的大小 */
    unsigned int frag_size = mtu - hlen;
    frag_size &= ~7;  /* 8 字节对齐 */
    
    /* 创建分片 */
    while (left > 0) {
        struct sk_buff *skb2;
        
        /* 分配新的 sk_buff */
        skb2 = alloc_skb(len + hlen + ll_rs, GFP_ATOMIC);
        
        /* 复制 IP 头部 */
        ip_copy_metadata(skb2, skb);
        
        /* 设置分片标志和偏移 */
        iph2 = ip_hdr(skb2);
        iph2->frag_off = htons((offset >> 3));
        if (left > 0)
            iph2->frag_off |= htons(IP_MF);
        
        /* 发送分片 */
        err = output(net, sk, skb2);
    }
}

/* IP 重组 */
struct ipq {
    struct inet_frag_queue q;
    u8         ecn;        /* ECN 标记 */
    u16        max_df_size; /* 最大不分片大小 */
    int        iif;        /* 输入接口 */
    unsigned int rid;      /* 路由 ID */
    struct inet_peer *peer;
};

static int ip_frag_reasm(struct ipq *qp, struct sk_buff *skb,
                         struct sk_buff *prev_tail, struct net_device *dev)
{
    struct net *net = qp->q.fqdir->net;
    struct iphdr *iph;
    void *reasm_data;
    int len, err;
    
    /* 检查所有分片是否到齐 */
    if (!(qp->q.flags & INET_FRAG_COMPLETE))
        goto out_fail;
    
    /* 重组分片 */
    reasm_data = inet_frag_reasm_prepare(&qp->q, skb, prev_tail);
    
    /* 合并所有分片 */
    inet_frag_reasm_finish(&qp->q, skb, reasm_data,
                           ip_frag_ecn_table[qp->ecn]);
    
    /* 更新 IP 头部 */
    iph = ip_hdr(skb);
    iph->tot_len = htons(len);
    iph->frag_off = 0;
}
```

### 9.3.2 传输层：TCP 状态机与拥塞控制

#### TCP 连接管理

TCP 使用有限状态机管理连接生命周期：

```c
/* TCP 状态枚举 */
enum {
    TCP_ESTABLISHED = 1,
    TCP_SYN_SENT,
    TCP_SYN_RECV,
    TCP_FIN_WAIT1,
    TCP_FIN_WAIT2,
    TCP_TIME_WAIT,
    TCP_CLOSE,
    TCP_CLOSE_WAIT,
    TCP_LAST_ACK,
    TCP_LISTEN,
    TCP_CLOSING,
    TCP_NEW_SYN_RECV,
};

/* TCP 控制块 */
struct tcp_sock {
    struct inet_connection_sock inet_conn;
    
    /* 序列号管理 */
    u32    rcv_nxt;      /* 期望接收的下一个序列号 */
    u32    snd_nxt;      /* 下一个要发送的序列号 */
    u32    snd_una;      /* 最早未确认的序列号 */
    u32    snd_wnd;      /* 发送窗口大小 */
    u32    rcv_wnd;      /* 接收窗口大小 */
    
    /* 拥塞控制 */
    u32    snd_cwnd;     /* 拥塞窗口 */
    u32    snd_ssthresh; /* 慢启动阈值 */
    u32    prior_cwnd;   /* 之前的拥塞窗口 */
    
    /* RTT 估算 */
    struct minmax rtt_min;  /* 最小 RTT */
    u32    srtt_us;      /* 平滑 RTT (微秒) */
    u32    mdev_us;      /* RTT 平均偏差 */
    u32    rttvar_us;    /* RTT 方差 */
    u32    rto;          /* 重传超时 */
    
    /* 拥塞控制算法 */
    const struct tcp_congestion_ops *icsk_ca_ops;
    
    /* 重传队列 */
    struct sk_buff_head out_of_order_queue;
    struct tcp_sack_block selective_acks[4];
    
    /* 时间戳 */
    u32    tsoffset;     /* 时间戳偏移 */
    u32    ts_recent;    /* 最近接收的时间戳 */
    u32    ts_recent_stamp; /* ts_recent 的时间 */
};
```

#### 三次握手实现

TCP 三次握手的内核实现涉及 SYN cookie、Fast Open 等优化：

```c
/* 服务器端：处理 SYN */
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
    struct tcp_options_received tmp_opt;
    struct tcp_sock *tp = tcp_sk(sk);
    struct request_sock *req;
    
    /* SYN flood 防护：SYN cookie */
    if (sk_acceptq_is_full(sk)) {
        want_cookie = tcp_syn_flood_action(sk, skb, "TCP");
        if (!want_cookie)
            goto drop;
    }
    
    /* 分配请求套接字 */
    req = inet_reqsk_alloc(&tcp_request_sock_ops, sk, !want_cookie);
    
    /* 解析 TCP 选项 */
    tcp_parse_options(skb, &tmp_opt, 0, NULL);
    
    /* 初始化序列号 */
    tcp_openreq_init(req, &tmp_opt, skb, sk);
    
    /* 生成 SYN cookie（如果需要） */
    if (want_cookie) {
        isn = cookie_init_sequence(req, skb, &req->mss);
        req->cookie_ts = tmp_opt.tstamp_ok;
    } else {
        /* 正常的 ISN 生成 */
        isn = tcp_v4_init_seq(skb);
    }
    
    /* 发送 SYN+ACK */
    err = tcp_v4_send_synack(sk, dst, &fl4, req, skb);
    
    /* 加入半连接队列 */
    inet_csk_reqsk_queue_hash_add(sk, req, TCP_TIMEOUT_INIT);
}

/* 客户端：发送 SYN */
int tcp_connect(struct sock *sk)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *buff;
    
    /* 分配 SYN 包 */
    buff = sk_stream_alloc_skb(sk, 0, sk->sk_allocation, true);
    
    /* 初始化 TCP 头部 */
    tcp_init_nondata_skb(buff, tp->write_seq++, TCPHDR_SYN);
    
    /* TCP Fast Open */
    if (fastopen) {
        tcp_fastopen_init_key_once();
        tp->fastopen_req = kzalloc(sizeof(*tp->fastopen_req), 
                                    sk->sk_allocation);
    }
    
    /* 设置初始拥塞窗口 */
    tp->snd_cwnd = TCP_INIT_CWND;
    
    /* 启动重传定时器 */
    inet_csk_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
                              inet_csk(sk)->icsk_rto, TCP_RTO_MAX);
    
    /* 发送 SYN */
    return tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);
}
```

#### 拥塞控制算法

Linux 支持可插拔的拥塞控制算法，默认使用 CUBIC：

```c
/* 拥塞控制操作接口 */
struct tcp_congestion_ops {
    struct list_head    list;
    u32 key;
    u32 flags;
    
    /* 必需的回调函数 */
    void (*init)(struct sock *sk);
    void (*release)(struct sock *sk);
    
    /* 拥塞窗口调整 */
    u32 (*ssthresh)(struct sock *sk);
    void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);
    
    /* 状态变化 */
    void (*set_state)(struct sock *sk, u8 new_state);
    void (*cwnd_event)(struct sock *sk, enum tcp_ca_event ev);
    
    /* 丢包处理 */
    void (*in_ack_event)(struct sock *sk, u32 flags);
    u32 (*undo_cwnd)(struct sock *sk);
    void (*pkts_acked)(struct sock *sk, const struct ack_sample *sample);
    
    /* BBR 使用的回调 */
    u32 (*min_tso_segs)(struct sock *sk);
    void (*cong_control)(struct sock *sk, const struct rate_sample *rs);
    
    char name[TCP_CA_NAME_MAX];
};

/* CUBIC 算法核心 */
static void bictcp_cong_avoid(struct sock *sk, u32 ack, u32 acked)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct bictcp *ca = inet_csk_ca(sk);
    
    if (!tcp_is_cwnd_limited(sk))
        return;
    
    if (tcp_in_slow_start(tp)) {
        /* 慢启动：指数增长 */
        acked = tcp_slow_start(tp, acked);
        if (!acked)
            return;
    }
    
    /* 拥塞避免：CUBIC 函数 */
    bictcp_update(ca, tp->snd_cwnd, acked);
    tcp_cong_avoid_ai(tp, ca->cnt, acked);
}

/* CUBIC 更新函数 */
static inline void bictcp_update(struct bictcp *ca, u32 cwnd, u32 acked)
{
    u32 delta, bic_target, max_cnt;
    u64 offs, t;
    
    ca->ack_cnt += acked;
    
    if (ca->last_cwnd == cwnd &&
        (s32)(tcp_jiffies32 - ca->last_time) <= HZ / 32)
        return;
    
    /* CUBIC 函数：W(t) = C * (t - K)^3 + Wmax */
    t = (s32)(tcp_jiffies32 - ca->epoch_start);
    t += ca->delay_min >> 3;
    
    /* 计算 K = cubic_root(Wmax * beta / C) */
    K = ca->K;
    
    offs = ca->ack_cnt / delta;
    if (t < K)
        bic_target = ca->origin_point - delta;
    else
        bic_target = ca->origin_point + delta;
    
    /* 更新拥塞窗口 */
    if (bic_target > cwnd) {
        ca->cnt = cwnd / (bic_target - cwnd);
    } else {
        ca->cnt = 100 * cwnd;
    }
}
```

#### 快速重传与恢复

TCP 使用 SACK（Selective Acknowledgment）实现高效的丢包恢复：

```c
/* 快速重传判定 */
static bool tcp_time_to_recover(struct sock *sk, int flag)
{
    struct tcp_sock *tp = tcp_sk(sk);
    
    /* 条件1：收到3个重复 ACK */
    if (tp->fackets_out > tp->reordering)
        return true;
    
    /* 条件2：SACK 检测到丢包 */
    if (tcp_is_sack(tp) && tp->lost_out > 0)
        return true;
    
    /* 条件3：FACK 检测 */
    if (tcp_is_fack(tp) && tp->fackets_out > tp->reordering)
        return true;
    
    return false;
}

/* 进入快速恢复 */
void tcp_enter_recovery(struct sock *sk, bool ece_ack)
{
    struct tcp_sock *tp = tcp_sk(sk);
    int mib_idx;
    
    /* 保存当前状态 */
    tp->prior_ssthresh = 0;
    tcp_init_undo(tp);
    
    /* 调整拥塞窗口 */
    tp->snd_ssthresh = tcp_current_ssthresh(sk);
    tp->snd_cwnd = tp->snd_ssthresh;
    tp->snd_cwnd_cnt = 0;
    tp->prior_cwnd = tp->snd_cwnd;
    tp->prr_delivered = 0;
    tp->prr_out = 0;
    
    /* 设置恢复点 */
    tp->recovery = tp->snd_nxt;
    
    /* 进入恢复状态 */
    tcp_set_ca_state(sk, TCP_CA_Recovery);
}
```

### 9.3.3 套接字层：socket API 实现

#### socket 结构体系

Linux 套接字层提供统一的编程接口，屏蔽底层协议差异：

```c
/* 通用套接字结构 */
struct socket {
    socket_state        state;      /* 套接字状态 */
    short               type;       /* SOCK_STREAM, SOCK_DGRAM 等 */
    unsigned long       flags;      /* 套接字标志 */
    
    struct file        *file;       /* 关联的文件对象 */
    struct sock        *sk;         /* 网络层套接字 */
    const struct proto_ops *ops;    /* 协议操作函数表 */
    
    struct socket_wq    wq;         /* 等待队列 */
};

/* 网络层套接字 */
struct sock {
    struct sock_common  __sk_common;
    
    /* 接收队列 */
    struct sk_buff_head sk_receive_queue;
    struct sk_buff_head sk_write_queue;
    
    /* 内存管理 */
    atomic_t            sk_rmem_alloc;  /* 接收缓冲区大小 */
    atomic_t            sk_wmem_alloc;  /* 发送缓冲区大小 */
    int                 sk_sndbuf;      /* 发送缓冲区限制 */
    int                 sk_rcvbuf;      /* 接收缓冲区限制 */
    
    /* 回调函数 */
    void (*sk_state_change)(struct sock *sk);
    void (*sk_data_ready)(struct sock *sk);
    void (*sk_write_space)(struct sock *sk);
    void (*sk_error_report)(struct sock *sk);
    
    /* 协议相关 */
    struct proto       *sk_prot;
    const struct proto_ops *sk_prot_creator;
};

/* 协议操作函数表 */
struct proto_ops {
    int (*bind)(struct socket *sock, struct sockaddr *addr, int len);
    int (*connect)(struct socket *sock, struct sockaddr *addr, int len, int flags);
    int (*accept)(struct socket *sock, struct socket *newsock, int flags);
    int (*listen)(struct socket *sock, int len);
    int (*sendmsg)(struct socket *sock, struct msghdr *m, size_t len);
    int (*recvmsg)(struct socket *sock, struct msghdr *m, size_t len, int flags);
    /* ... */
};
```

#### 系统调用实现

套接字系统调用的内核实现：

```c
/* socket() 系统调用 */
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
    struct socket *sock;
    int retval;
    
    /* 创建套接字 */
    retval = sock_create(family, type, protocol, &sock);
    if (retval < 0)
        return retval;
    
    /* 分配文件描述符 */
    retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
    if (retval < 0)
        goto out_release;
    
    return retval;
}

/* bind() 实现 */
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
    struct socket *sock;
    struct sockaddr_storage address;
    int err;
    
    /* 获取套接字 */
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        return err;
    
    /* 复制地址 */
    err = move_addr_to_kernel(umyaddr, addrlen, &address);
    if (err >= 0)
        err = sock->ops->bind(sock, (struct sockaddr *)&address, addrlen);
    
    fput_light(sock->file, fput_needed);
    return err;
}

/* send() 数据发送路径 */
SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
                unsigned int, flags, struct sockaddr __user *, addr,
                int, addr_len)
{
    struct socket *sock;
    struct msghdr msg;
    struct iovec iov;
    
    /* 构造消息结构 */
    iov.iov_base = buff;
    iov.iov_len = len;
    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_flags = flags;
    
    /* 设置目标地址 */
    if (addr) {
        err = move_addr_to_kernel(addr, addr_len, &address);
        msg.msg_name = (struct sockaddr *)&address;
        msg.msg_namelen = addr_len;
    }
    
    /* 调用协议层发送 */
    err = sock->ops->sendmsg(sock, &msg, len);
    
    return err;
}
```

#### 异步 I/O 支持

Linux 5.1 引入的 io_uring 为网络编程带来革命性改进：

```c
/* io_uring 网络操作 */
static int io_send(struct io_kiocb *req, unsigned int issue_flags)
{
    struct io_sr_msg *sr = &req->sr_msg;
    struct socket *sock;
    struct msghdr msg;
    int ret;
    
    sock = sock_from_file(req->file);
    if (unlikely(!sock))
        return -ENOTSOCK;
    
    /* 构造消息 */
    ret = io_setup_async_msg(req, &msg, issue_flags);
    if (ret)
        return ret;
    
    /* 非阻塞发送 */
    msg.msg_flags = sr->msg_flags | MSG_NOSIGNAL;
    if (issue_flags & IO_URING_F_NONBLOCK)
        msg.msg_flags |= MSG_DONTWAIT;
    
    ret = sock_sendmsg(sock, &msg);
    
    /* 处理 EAGAIN */
    if (ret == -EAGAIN && (issue_flags & IO_URING_F_NONBLOCK))
        return -EAGAIN;
    
    /* 完成请求 */
    io_req_complete(req, ret);
    return 0;
}
```

### 9.3.4 连接管理与 TIME_WAIT 优化

#### TIME_WAIT 状态管理

TIME_WAIT 状态确保延迟数据包不会干扰新连接，但大量 TIME_WAIT 会消耗资源：

```c
/* TIME_WAIT 套接字结构（精简版） */
struct inet_timewait_sock {
    struct sock_common  __tw_common;
    
    int                 tw_timeout;     /* TIME_WAIT 超时时间 */
    volatile unsigned char tw_substate; /* 子状态 */
    unsigned char       tw_rcv_wscale;  /* 接收窗口缩放 */
    
    /* 这些字段从 struct tcp_sock 复制 */
    __be16              tw_sport;       /* 源端口 */
    __be16              tw_dport;       /* 目的端口 */
    __u32               tw_rcv_nxt;     /* 期望接收序列号 */
    __u32               tw_snd_nxt;     /* 下一个发送序列号 */
    __u32               tw_rcv_wnd;     /* 接收窗口 */
    __u32               tw_ts_recent;   /* 时间戳 */
    
    struct timer_list   tw_timer;       /* TIME_WAIT 定时器 */
    struct inet_bind_bucket *tw_tb;     /* 绑定哈希桶 */
};

/* 进入 TIME_WAIT 状态 */
void tcp_time_wait(struct sock *sk, int state, int timeo)
{
    struct inet_timewait_sock *tw;
    struct tcp_sock *tp = tcp_sk(sk);
    
    /* 分配 timewait 套接字 */
    tw = inet_twsk_alloc(sk, state);
    if (tw) {
        /* 复制必要字段 */
        tw->tw_rcv_wscale = tp->rx_opt.rcv_wscale;
        tw->tw_rcv_nxt = tp->rcv_nxt;
        tw->tw_snd_nxt = tp->snd_nxt;
        tw->tw_rcv_wnd = tcp_receive_window(tp);
        tw->tw_ts_recent = tp->rx_opt.ts_recent;
        tw->tw_ts_recent_stamp = tp->rx_opt.ts_recent_stamp;
        
        /* 设置 TIME_WAIT 定时器 */
        if (timeo < 0)
            timeo = TCP_TIMEWAIT_LEN;  /* 默认 60 秒 */
        
        inet_twsk_schedule(tw, timeo);
        
        /* 将原始套接字转换为 timewait */
        inet_twsk_hashdance(tw, sk, &tcp_hashinfo);
    }
    
    /* 销毁原始套接字 */
    tcp_done(sk);
}
```

#### TIME_WAIT 优化策略

Linux 提供多种机制优化 TIME_WAIT：

```c
/* 1. TIME_WAIT 重用（tw_reuse） */
static int tcp_tw_reuse_check(const struct inet_timewait_sock *tw,
                               const struct tcp_options_received *rx_opt)
{
    /* 检查时间戳，确保不会接收旧数据包 */
    if (rx_opt->ts_recent &&
        get_seconds() - tw->tw_ts_recent_stamp > 1)
        return 1;
    
    return 0;
}

/* 2. TIME_WAIT 回收（tw_recycle）- 已废弃 */
/* 由于 NAT 环境问题，4.12 内核已移除 */

/* 3. TIME_WAIT 暗杀（tw_kill） */
static void inet_twsk_kill(struct inet_timewait_sock *tw)
{
    struct inet_hashinfo *hashinfo = tw->tw_dr->hashinfo;
    spinlock_t *lock = inet_ehash_lockp(hashinfo, tw->tw_hash);
    
    spin_lock(lock);
    if (hlist_nulls_unhashed(&tw->tw_node)) {
        spin_unlock(lock);
        return;
    }
    
    /* 从哈希表移除 */
    hlist_nulls_del_rcu(&tw->tw_node);
    sk_nulls_node_init(&tw->tw_node);
    spin_unlock(lock);
    
    /* 释放端口 */
    inet_twsk_bind_unhash(tw, hashinfo);
    
    /* 释放内存 */
    inet_twsk_put(tw);
}

/* 4. SO_LINGER 选项 */
struct linger {
    int l_onoff;    /* 0 = off, 非0 = on */
    int l_linger;   /* 延迟时间（秒） */
};

/* 设置 SO_LINGER 跳过 TIME_WAIT */
if (sk->sk_linger && !sk->sk_lingertime) {
    /* l_onoff = 1, l_linger = 0: 立即关闭，发送 RST */
    tcp_send_active_reset(sk, gfp_any());
    tcp_done(sk);
} else {
    /* 正常关闭流程 */
    tcp_close_state(sk);
}
```

#### 连接表管理

Linux 使用哈希表管理大量并发连接：

```c
/* TCP 连接哈希表 */
struct inet_hashinfo tcp_hashinfo = {
    .ehash = NULL,          /* established 哈希表 */
    .ehash_locks = NULL,    /* 哈希锁数组 */
    .ehash_mask = 0,        /* 哈希掩码 */
    .ehash_locks_mask = 0,
    
    .bhash = NULL,          /* bind 哈希表 */
    .bhash_size = 0,
    
    .lhash2 = NULL,         /* listen 哈希表 */
    .lhash2_mask = 0,
};

/* 连接查找 */
struct sock *__inet_lookup_established(struct net *net,
                                        struct inet_hashinfo *hashinfo,
                                        const __be32 saddr, const __be16 sport,
                                        const __be32 daddr, const u16 hnum,
                                        const int dif)
{
    INET_ADDR_COOKIE(acookie, saddr, daddr);
    const __portpair ports = INET_COMBINED_PORTS(sport, hnum);
    struct sock *sk;
    const struct hlist_nulls_node *node;
    
    /* 计算哈希值 */
    unsigned int hash = inet_ehashfn(net, daddr, hnum, saddr, sport);
    unsigned int slot = hash & hashinfo->ehash_mask;
    struct inet_ehash_bucket *head = &hashinfo->ehash[slot];
    
    /* RCU 读锁保护 */
    rcu_read_lock();
begin:
    sk_nulls_for_each_rcu(sk, node, &head->chain) {
        if (sk->sk_hash != hash)
            continue;
        
        /* 快速路径：cookie 匹配 */
        if (likely(INET_MATCH(sk, net, acookie, saddr, daddr, 
                              ports, dif))) {
            if (unlikely(!refcount_inc_not_zero(&sk->sk_refcnt)))
                goto out;
            
            /* 再次检查（防止并发删除） */
            if (unlikely(!INET_MATCH(sk, net, acookie, saddr, daddr,
                                     ports, dif))) {
                sock_gen_put(sk);
                goto begin;
            }
            goto found;
        }
    }
    
    /* 检查是否需要重试（NULLS 保护） */
    if (get_nulls_value(node) != slot)
        goto begin;
    
out:
    sk = NULL;
found:
    rcu_read_unlock();
    return sk;
}
```

#### 端口管理与复用

Linux 支持 SO_REUSEADDR 和 SO_REUSEPORT 实现端口复用：

```c
/* SO_REUSEPORT 实现负载均衡 */
struct sock *inet_lookup_listener(struct net *net,
                                   struct inet_hashinfo *hashinfo,
                                   struct sk_buff *skb, int doff,
                                   const __be32 saddr, __be16 sport,
                                   const __be32 daddr, const unsigned short hnum,
                                   const int dif, const int sdif)
{
    struct sock *sk, *result = NULL;
    struct inet_listen_hashbucket *ilb;
    unsigned int hash = inet_lhashfn(net, hnum);
    
    ilb = &hashinfo->listening_hash[hash];
    
    /* 查找监听套接字 */
    sk_for_each_rcu(sk, &ilb->head) {
        score = compute_score(sk, net, hnum, daddr, dif, sdif);
        
        if (score > hiscore) {
            hiscore = score;
            result = sk;
            
            /* SO_REUSEPORT 负载均衡 */
            if (sk->sk_reuseport) {
                phash = inet_ehashfn(net, daddr, hnum, saddr, sport);
                
                /* 使用包哈希选择套接字 */
                result = reuseport_select_sock(sk, phash, skb, doff);
            }
        }
    }
    
    return result;
}

/* REUSEPORT 组管理 */
struct sock_reuseport {
    struct rcu_head     rcu;
    
    u16                 max_socks;      /* 数组大小 */
    u16                 num_socks;      /* 当前套接字数 */
    unsigned int        synq_overflow_ts; /* SYN 队列溢出时间戳 */
    unsigned int        reuseport_id;   /* ID */
    unsigned int        bind_inany:1;   /* 绑定任意地址 */
    unsigned int        has_conns:1;    /* 有已建立连接 */
    struct bpf_prog __rcu *prog;        /* eBPF 程序 */
    struct sock        *socks[0];       /* 套接字数组 */
};
```

---

## 9.2 sk_buff 与网络数据流

sk_buff（socket buffer）是 Linux 网络栈的核心数据结构，每个网络数据包在内核中都由一个 sk_buff 结构表示。这个精心设计的数据结构支持高效的零拷贝操作、协议头部的动态添加删除，以及数据包的分片与重组。

### 9.2.1 sk_buff 结构体详解

sk_buff 结构体包含了数据包的所有元信息和指向实际数据的指针：

```c
struct sk_buff {
    /* 链表管理 */
    struct sk_buff    *next;
    struct sk_buff    *prev;
    
    /* 时间戳 */
    ktime_t           tstamp;
    
    /* 关联的网络设备 */
    struct net_device *dev;
    
    /* 关联的 socket */
    struct sock       *sk;
    
    /* 数据指针 
     * head: 缓冲区起始地址
     * data: 有效数据起始地址  
     * tail: 有效数据结束地址
     * end:  缓冲区结束地址
     */
    unsigned char     *head,
                      *data,
                      *tail,
                      *end;
    
    /* 传输层头部指针 */
    union {
        struct tcphdr  *th;
        struct udphdr  *uh;
        struct icmphdr *icmph;
        unsigned char  *raw;
    } h;
    
    /* 网络层头部指针 */
    union {
        struct iphdr   *iph;
        struct ipv6hdr *ipv6h;
        struct arphdr  *arph;
        unsigned char  *raw;
    } nh;
    
    /* 链路层头部指针 */
    union {
        struct ethhdr  *ethernet;
        unsigned char  *raw;
    } mac;
    
    /* 路由信息 */
    struct dst_entry *dst;
    
    /* 控制信息 */
    __u32             priority;
    __u16             protocol;
    __u8              pkt_type;
    
    /* 引用计数 */
    atomic_t          users;
    
    /* 分片信息 */
    struct skb_shared_info *shinfo;
};
```

### 关键指针关系

sk_buff 中最重要的是四个数据指针的关系：

```
        head                data               tail                end
         ↓                   ↓                   ↓                   ↓
    ┌────┬──────────────┬────────────────────┬─────────────────┬────┐
    │    │   headroom    │    packet data    │    tailroom     │    │
    └────┴──────────────┴────────────────────┴─────────────────┴────┘
         ←───────────────── skb->len ──────────→
         ←──────────────────── skb->data_len ────────────────────→
         ←────────────────────── skb->truesize ──────────────────────→
```

这种设计允许在数据包前后高效地添加或删除协议头部：

- **skb_push()**：在数据前添加头部（data 指针前移）
- **skb_pull()**：移除数据前的头部（data 指针后移）
- **skb_put()**：在数据后添加内容（tail 指针后移）
- **skb_trim()**：截断数据末尾（tail 指针前移）

### 9.2.2 sk_buff 的生命周期管理

#### 分配与初始化

网络数据包的 sk_buff 通常在以下场景分配：

1. **接收路径**：网卡驱动在 DMA 缓冲区准备时预分配
2. **发送路径**：应用程序发送数据时动态分配
3. **转发路径**：可能复用原 sk_buff 或克隆

```c
/* 分配 sk_buff */
struct sk_buff *skb = alloc_skb(size, GFP_ATOMIC);

/* 为网络设备分配（包含 NET_SKB_PAD 对齐） */
struct sk_buff *skb = netdev_alloc_skb(dev, size);

/* 分配并预留头部空间 */
struct sk_buff *skb = alloc_skb(size + headroom, GFP_KERNEL);
skb_reserve(skb, headroom);
```

#### 引用计数管理

sk_buff 使用引用计数管理生命周期：

```c
/* 增加引用 */
skb_get(skb);  // atomic_inc(&skb->users)

/* 减少引用，到 0 时释放 */
kfree_skb(skb);  // 错误路径释放
consume_skb(skb); // 正常路径释放
```

#### 克隆与复制

为了避免数据复制，sk_buff 支持高效的克隆：

```c
/* 克隆：共享数据，复制元数据 */
struct sk_buff *clone = skb_clone(skb, GFP_ATOMIC);

/* 完整复制：数据和元数据都复制 */
struct sk_buff *copy = skb_copy(skb, GFP_ATOMIC);

/* COW 复制：必要时才复制数据 */
if (skb_shared(skb))
    skb = skb_unshare(skb, GFP_ATOMIC);
```

### 9.2.3 数据包在各层的处理流程

#### 接收路径（Bottom-up）

```
网卡 DMA → 驱动 RX 中断 → NAPI poll → netif_receive_skb
    ↓
链路层处理（eth_type_trans）
    ↓
网络层处理（ip_rcv / ipv6_rcv）
    ↓
Netfilter PREROUTING 钩子
    ↓
路由决策（本地 / 转发）
    ↓
本地：传输层处理（tcp_v4_rcv / udp_rcv）
    ↓
套接字队列（sk->sk_receive_queue）
    ↓
应用程序 recv()
```

每层处理的典型操作：

```c
/* 链路层：解析以太网头部 */
static int eth_type_trans(struct sk_buff *skb, struct net_device *dev)
{
    struct ethhdr *eth = eth_hdr(skb);
    skb->protocol = eth->h_proto;
    skb_pull(skb, ETH_HLEN);  // 移除以太网头部
    return 0;
}

/* 网络层：IP 头部校验 */
static int ip_rcv(struct sk_buff *skb, struct net_device *dev, 
                  struct packet_type *pt, struct net_device *orig_dev)
{
    struct iphdr *iph = ip_hdr(skb);
    
    /* 校验 IP 头部 */
    if (iph->ihl < 5 || iph->version != 4)
        goto drop;
    
    if (!pskb_may_pull(skb, iph->ihl * 4))
        goto drop;
        
    /* 更新 skb 指针 */
    skb->transport_header = skb->network_header + iph->ihl * 4;
    
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, 
                   skb, dev, NULL, ip_rcv_finish);
}
```

#### 发送路径（Top-down）

```
应用程序 send() → socket 层
    ↓
传输层构造（tcp_transmit_skb）
    ↓
网络层处理（ip_output）
    ↓
Netfilter POSTROUTING 钩子
    ↓
邻居子系统（ARP 解析）
    ↓
设备队列（dev_queue_xmit）
    ↓
流量控制（qdisc）
    ↓
驱动 TX → DMA → 网卡发送
```

### 9.2.4 零拷贝与 scatter-gather I/O

#### 分散聚集 I/O

sk_buff 支持将数据分散在多个内存页中，避免大块连续内存分配：

```c
struct skb_shared_info {
    unsigned short  nr_frags;
    struct skb_frag_struct {
        struct page *page;
        __u32       page_offset;
        __u32       size;
    } frags[MAX_SKB_FRAGS];
};
```

这种设计支持：
- **零拷贝发送**：直接映射用户空间页面
- **TSO/GSO**：硬件分段卸载
- **GRO**：接收端数据包聚合

#### 页面翻转（Page Flipping）

高性能网卡驱动使用页面翻转技术避免数据复制：

```c
/* 接收时直接将 DMA 页面赋给 sk_buff */
static void rx_page_flip(struct sk_buff *skb, struct page *page)
{
    skb_fill_page_desc(skb, 0, page, offset, len);
    skb->data_len = len;
    skb->len += len;
}
```

#### MSG_ZEROCOPY 发送

Linux 4.14 引入的 MSG_ZEROCOPY 允许应用程序零拷贝发送：

```c
/* 用户空间 */
send(fd, buffer, len, MSG_ZEROCOPY);

/* 内核空间：直接引用用户页面 */
static int tcp_zerocopy_send(struct sock *sk, struct msghdr *msg)
{
    struct sk_buff *skb = tcp_write_queue_tail(sk);
    
    /* 将用户页面添加到 skb frags */
    err = skb_zerocopy_add_frags_iter(sk, skb, &msg->msg_iter, 
                                       len, &uarg, sk->sk_allocation);
    
    /* 设置回调通知 */
    skb_zcopy_set(skb, uarg);
}
```

### 内存管理优化

#### skb 缓存池

为了减少分配开销，内核维护了 per-CPU 的 sk_buff 缓存：

```c
struct netdev_alloc_cache {
    struct page_frag    frag;
    unsigned int        pagecnt_bias;
};

static DEFINE_PER_CPU(struct netdev_alloc_cache, netdev_alloc_cache);
```

#### 内存预分配策略

网卡驱动通常预分配接收缓冲区：

```c
/* Intel ixgbe 驱动的接收缓冲区管理 */
static bool ixgbe_alloc_rx_buffers(struct ixgbe_ring *rx_ring, u16 cleaned)
{
    union ixgbe_adv_rx_desc *rx_desc;
    struct ixgbe_rx_buffer *bi;
    
    /* 批量分配，减少内存分配开销 */
    while (cleaned--) {
        bi = &rx_ring->rx_buffer_info[i];
        bi->skb = netdev_alloc_skb_ip_align(rx_ring->netdev,
                                             rx_ring->rx_buf_len);
        
        /* 设置 DMA 映射 */
        bi->dma = dma_map_single(rx_ring->dev, bi->skb->data,
                                  rx_ring->rx_buf_len, DMA_FROM_DEVICE);
    }
}
```