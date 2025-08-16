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