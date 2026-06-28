# UDP 协议详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/udp.c` —— IPv4 UDP 协议主实现
> - `include/net/udp.h` —— UDP 模块内部头文件（hash 表、checksum 辅助函数）
> - `include/uapi/linux/udp.h` —— UDP 头结构与 socket 选项（用户态可见）
> - `include/linux/udp.h` —— `struct udp_sock` 定义
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图和时序图使用 `text` 代码块表示。

---

## 1. UDP 协议基础

UDP（User Datagram Protocol，RFC 768）在 IP 之上提供**无连接、不可靠、基于数据报**的端到端传输：不保证到达/顺序、不重传、无流控；保留报文边界（一次 `sendto()` 对应一个数据报，`recvfrom()` 读一个完整数据报，除非被截断）；头部仅 8 字节，适合 DNS、QUIC、音视频等场景。丢包或校验失败通常静默丢弃。

---

## 2. UDP 报文格式

```c
struct udphdr {
	__be16	source;
	__be16	dest;
	__be16	len;
	__sum16	check;
};
```

| 字段 | 长度 | 含义 |
|------|------|------|
| `source` | 2B | 源端口 |
| `dest`   | 2B | 目的端口 |
| `len`    | 2B | UDP 头 + UDP 数据的总长度（字节） |
| `check`  | 2B | 校验和（可选，IPv4 允许为 0；IPv6 一般必须） |

以太网上的完整 UDP 帧布局：

```text
| 以太网头 (14B) | IP 头 (20B/可变) | UDP 头 (8B) | UDP 数据 |
```

**校验和计算范围**：UDP 伪首部 + UDP 首部 + UDP 数据。伪首部包含：
- 源 IP 地址（4B）
- 目的 IP 地址（4B）
- 零（1B）
- 协议号（1B，IPPROTO_UDP = 17）
- UDP 长度（2B）

---

## 3. Linux 内核 UDP 实现总览

| 项目 | 文件/结构 |
|------|-----------|
| UDP 协议主文件 | `net/ipv4/udp.c` |
| UDP 内部头文件 | `include/net/udp.h` |
| UDP socket 结构 | `struct udp_sock`（`include/linux/udp.h`） |
| UDP 端口哈希表 | `struct udp_table udp_table` |
| 发送入口 | `udp_sendmsg()` |
| 接收入口 | `udp_rcv()` |
| socket 查找 | `__udp4_lib_lookup()` |
| 校验和辅助 | `udp_set_csum()`、`udp4_hwcsum()`、`udp_v4_check()` |
| 封装/隧道 | `udp_encap_needed_key`、`encap_rcv` |

**与 TCP 最大的不同**：UDP 没有复杂的状态机、滑动窗口、重传定时器；核心复杂度集中在 **端口/socket 查找、校验和、GSO/GRO、封装隧道** 上。

---

## 4. UDP socket 与哈希表

### 4.1 核心数据结构

```c
struct udp_hslot {
	union {
		struct hlist_head	head;
		struct hlist_nulls_head	nulls_head;
	};
	int			count;
	spinlock_t		lock;
} __aligned(2 * sizeof(long));

struct udp_hslot_main {
	struct udp_hslot	hslot;	/* must be the first member */
#if !IS_ENABLED(CONFIG_BASE_SMALL)
	u32			hash4_cnt;
#endif
} __aligned(2 * sizeof(long));

struct udp_table {
	struct udp_hslot	*hash;		/* by local port */
	struct udp_hslot_main	*hash2;		/* by (local port, local address) */
#if !IS_ENABLED(CONFIG_BASE_SMALL)
	struct udp_hslot	*hash4;		/* by 4-tuple */
#endif
	unsigned int		mask;
	unsigned int		log;
};
```

Linux 使用**三张哈希表**管理 UDP socket：

| 表 | 键 | 用途 |
|----|----|------|
| `hash`  | 本地端口 | bind / 端口分配时冲突检测；lookup fallback |
| `hash2` | (本地端口, 本地地址) | 接收路径主要查找入口 |
| `hash4` | (本地端口, 本地地址, 远端端口, 远端地址) | connected UDP socket 的 O(1) 精确匹配 |

### 4.2 端口分配：`udp_lib_get_port()`

```c
int udp_lib_get_port(struct sock *sk, unsigned short snum,
		     unsigned int hash2_nulladdr)
```

IPv4 包装：

```c
static int udp_v4_get_port(struct sock *sk, unsigned short snum)
{
	unsigned int hash2_nulladdr =
		ipv4_portaddr_hash(sock_net(sk), htonl(INADDR_ANY), snum);
	unsigned int hash2_partial =
		ipv4_portaddr_hash(sock_net(sk), inet_sk(sk)->inet_rcv_saddr, 0);

	udp_sk(sk)->udp_portaddr_hash = hash2_partial;
	return udp_lib_get_port(sk, snum, hash2_nulladdr);
}
```

分配策略：
- **指定端口（`snum != 0`）**：检查 `hash` 主链，若开启 `hash2_on_bind` 且该链较长，再扫描 `hash2` 两条链确认无冲突。
- **自动端口（`snum == 0`）**：在 sysctl 本地端口范围内随机起点，以 `UDP_HTABLE_SIZE` 奇数倍步进，遍历整个端口空间；用 per-bucket bitmap 跳过已知冲突端口。

冲突判断（`udp_lib_lport_inuse()`）要点：
- 同一 netns、不同 socket；
- `SO_REUSEADDR` 双方都设置可重叠；
- 绑定设备索引需兼容；
- 接收地址匹配（`inet_rcv_saddr_equal`）；
- `SO_REUSEPORT` + 同 UID 可组成 reuseport 组而非冲突。

### 4.3 接收时 socket 查找：`__udp4_lib_lookup()`

```c
struct sock *__udp4_lib_lookup(const struct net *net, __be32 saddr,
			       __be16 sport, __be32 daddr, __be16 dport,
			       int dif, int sdif, struct sk_buff *skb)
```

查找顺序：

```text
1. hash4 精确四元组匹配（connected socket）
2. hash2 按 (dport, daddr) 查找，含 reuseport 分发
3. BPF socket lookup 重定向（若启用）
4. hash2 按 (dport, INADDR_ANY) 查找 wildcard socket
5. hash1 主端口链 fallback（处理 rehash 竞态）
```

### 4.4 匹配评分：`compute_score()`

```c
static __always_inline int
compute_score(struct sock *sk, const struct net *net,
	      __be32 saddr, __be16 sport, __be32 daddr,
	      unsigned short hnum, int dif, int sdif)
```

评分规则：
- 基础分：IPv4 socket 得 2 分，IPv4-mapped IPv6 得 1 分。
- +4：远端地址精确匹配（`inet_daddr == saddr`）。
- +4：远端端口精确匹配（`inet_dport == sport`）。
- +4：绑定到具体设备且匹配。
- +1：报文到达 CPU 等于 `sk_incoming_cpu`（RFS 亲和提示）。

最高分胜出；`udp4_lib_lookup2()` 还会对非 connected socket 调用 `inet_lookup_reuseport()` 做 reuseport 组内分发。

---

## 5. UDP 发送流程

### 5.1 发送调用链

```text
sendto() / sendmsg()
  → udp_sendmsg()
    ├─ 解析目的地址（msg_name 或 connected socket）
    ├─ ip_route_output_flow()        # 路由查找
    ├─ ip_make_skb() / ip_append_data()  # 构造 skb
    ├─ udp_send_skb()                # 填 UDP 头、校验和
    │     └─ ip_send_skb()           # 交给 IPv4 层
    └─ ip_output() / ip_finish_output()  # 最终经 neigh + dev_queue_xmit() 发出
```

### 5.2 `udp_sendmsg()` 关键逻辑

```c
int udp_sendmsg(struct sock *sk, struct msghdr *msg, size_t len)
{
	int corkreq = udp_test_bit(CORK, sk) || msg->msg_flags & MSG_MORE;
	...
	if (len > 0xFFFF)
		return -EMSGSIZE;
	...
	ulen += sizeof(struct udphdr);

	/* 获取目的地址 */
	if (usin) {
		daddr = usin->sin_addr.s_addr;
		dport = usin->sin_port;
	} else {
		/* connected socket */
		daddr = inet->inet_daddr;
		dport = inet->inet_dport;
		connected = 1;
	}

	ipc.gso_size = READ_ONCE(up->gso_size);
	if (msg->msg_controllen) {
		err = udp_cmsg_send(sk, msg, &ipc.gso_size);
		...
	}

	/* 路由查找 */
	if (connected)
		rt = dst_rtable(sk_dst_check(sk, 0));

	if (!rt) {
		flowi4_init_output(fl4, ipc.oif, ipc.sockc.mark,
				   ipc.tos & INET_DSCP_MASK, scope,
				   IPPROTO_UDP, flow_flags, faddr, saddr,
				   dport, inet->inet_sport,
				   sk_uid(sk));
		rt = ip_route_output_flow(net, fl4, sk);
		...
	}

	/* 非 cork 快速路径 */
	if (!corkreq) {
		skb = ip_make_skb(sk, fl4, ip_generic_getfrag, msg, ulen,
				  sizeof(struct udphdr), &ipc, &rt,
				  &cork, msg->msg_flags);
		err = PTR_ERR(skb);
		if (!IS_ERR_OR_NULL(skb))
			err = udp_send_skb(skb, fl4, &cork);
		goto out;
	}

	/* cork 路径 */
	up->len += ulen;
	err = ip_append_data(sk, fl4, ip_generic_getfrag, msg, ulen,
			     sizeof(struct udphdr), &ipc, &rt,
			     corkreq ? msg->msg_flags|MSG_MORE : msg->msg_flags);
	if (err)
		udp_flush_pending_frames(sk);
	else if (!corkreq)
		err = udp_push_pending_frames(sk);
	...
}
```

### 5.3 `udp_send_skb()`：填头与校验和

```c
static int udp_send_skb(struct sk_buff *skb, struct flowi4 *fl4,
			struct inet_cork *cork)
{
	struct sock *sk = skb->sk;
	int offset, len, datalen;
	struct udphdr *uh;
	int err;

	offset = skb_transport_offset(skb);
	len = skb->len - offset;
	datalen = len - sizeof(*uh);

	/* Create a UDP header */
	uh = udp_hdr(skb);
	uh->source = inet_sk(sk)->inet_sport;
	uh->dest = fl4->fl4_dport;
	uh->len = htons(len);
	uh->check = 0;

	if (cork->gso_size) {
		...
		if (datalen > cork->gso_size) {
			skb_shinfo(skb)->gso_size = cork->gso_size;
			skb_shinfo(skb)->gso_type = SKB_GSO_UDP_L4;
			skb_shinfo(skb)->gso_segs = DIV_ROUND_UP(datalen,
								 cork->gso_size);
			goto csum_partial;
		}
	}

	if (sk->sk_no_check_tx) {			 /* UDP csum off */
		skb->ip_summed = CHECKSUM_NONE;
		goto send;
	} else if (skb->ip_summed == CHECKSUM_PARTIAL) { /* HW csum */
csum_partial:
		udp4_hwcsum(skb, fl4->saddr, fl4->daddr);
		goto send;
	}

	/* add protocol-dependent pseudo-header */
	uh->check = csum_tcpudp_magic(fl4->saddr, fl4->daddr, len,
				      IPPROTO_UDP, udp_csum(skb));
	if (uh->check == 0)
		uh->check = CSUM_MANGLED_0;

send:
	err = ip_send_skb(sock_net(sk), skb);
	if (unlikely(err)) {
		if (err == -ENOBUFS &&
		    !inet_test_bit(RECVERR, sk)) {
			UDP_INC_STATS(sock_net(sk), UDP_MIB_SNDBUFERRORS);
			err = 0;
		}
	} else {
		UDP_INC_STATS(sock_net(sk), UDP_MIB_OUTDATAGRAMS);
	}
	return err;
}
```

三种校验和路径：
1. **`sk_no_check_tx` 开启**：校验和字段填 0，标记 `CHECKSUM_NONE`。
2. **硬件卸载（`CHECKSUM_PARTIAL`）**：调用 `udp4_hwcsum()` 设置 `csum_start`/`csum_offset`，网卡完成最终计算。
3. **软件计算**：`udp_csum()` 累加 UDP 头+数据，`csum_tcpudp_magic()` 加伪首部。

### 5.4 `UDP_SEGMENT` 与 UDP GSO

```c
static int __udp_cmsg_send(struct cmsghdr *cmsg, u16 *gso_size)
{
	switch (cmsg->cmsg_type) {
	case UDP_SEGMENT:
		if (cmsg->cmsg_len != CMSG_LEN(sizeof(__u16)))
			return -EINVAL;
		*gso_size = *(__u16 *)CMSG_DATA(cmsg);
		return 0;
	default:
		return -EINVAL;
	}
}
```

`UDP_SEGMENT` 允许应用通过 `sendmsg()` 的 cmsg 指定 GSO 分段大小。当数据长度超过该值时，`udp_send_skb()` 把 skb 标记为 `SKB_GSO_UDP_L4`，由后续 `__udp_gso_segment()` 在合适时机分段。最大段数受 `UDP_MAX_SEGMENTS`（128）限制。

---

## 6. UDP 接收流程

### 6.1 接收调用链

```text
网卡驱动
  → __netif_receive_skb_core()
    → ip_rcv() → ip_rcv_finish_core()
      → [可选] udp_v4_early_demux()   # 提前查找 socket 并 attach 到 skb->sk
      → ip_local_deliver_finish()
        → ip_protocol_deliver_rcu()
          → udp_rcv(skb)
            → 校验 UDP 头长度、uh->len、trim
            → udp4_csum_init()
            → inet_steal_sock()       # 复用 early demux 找到的 socket
              → udp_unicast_rcv_skb()
                → udp_queue_rcv_skb()
                  → udp_queue_rcv_one_skb()
                    → xfrm4_policy_check, sk_filter_trim_cap
                    → __udp_queue_rcv_skb()
                      → __udp_enqueue_schedule_skb()
                        → sk_receive_queue / reader_queue

用户 recvmsg()
  → udp_recvmsg()
    → __skb_recv_udp()
    → 校验和检查 + copy_to_iter
    → skb_consume_udp()
```

### 6.2 `udp_rcv()` 入口

```c
int udp_rcv(struct sk_buff *skb)
{
	struct rtable *rt = skb_rtable(skb);
	struct net *net = dev_net(skb->dev);
	struct sock *sk = NULL;
	unsigned short ulen;
	__be32 saddr, daddr;
	struct udphdr *uh;
	bool refcounted;
	int drop_reason;

	drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;

	/* Validate the packet. */
	if (!pskb_may_pull(skb, sizeof(struct udphdr)))
		goto drop;

	uh   = udp_hdr(skb);
	ulen = ntohs(uh->len);
	saddr = ip_hdr(skb)->saddr;
	daddr = ip_hdr(skb)->daddr;

	if (ulen > skb->len)
		goto short_packet;

	if (ulen < sizeof(*uh))
		goto short_packet;

	if (ulen < skb->len) {
		if (pskb_trim_rcsum(skb, ulen))
			goto short_packet;
		uh = udp_hdr(skb);
	}

	if (udp4_csum_init(skb, uh))
		goto csum_error;

	sk = inet_steal_sock(net, skb, sizeof(struct udphdr), saddr, uh->source,
			     daddr, uh->dest, &refcounted, udp_ehashfn);
	if (IS_ERR(sk))
		goto no_sk;

	if (sk) {
		...
		ret = udp_unicast_rcv_skb(sk, skb, uh);
		if (refcounted)
			sock_put(sk);
		return ret;
	}

	if (rt->rt_flags & (RTCF_BROADCAST|RTCF_MULTICAST))
		return __udp4_lib_mcast_deliver(net, skb, uh, saddr, daddr);

	sk = __udp4_lib_lookup_skb(skb, uh->source, uh->dest);
	if (sk)
		return udp_unicast_rcv_skb(sk, skb, uh);
no_sk:
	...
	if (udp_lib_checksum_complete(skb))
		goto csum_error;

	drop_reason = SKB_DROP_REASON_NO_SOCKET;
	__UDP_INC_STATS(net, UDP_MIB_NOPORTS);
	icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);
	...
}
```

入口要点：
- 先拉取并校验 UDP 头；
- 根据 `uh->len` 裁剪 skb 尾部（处理 IP 尾填充）；
- 初始化校验和相关状态；
- 优先复用 early demux 已 attach 的 socket；
- 未命中再按广播/组播或单播路径查找；
- 仍无 socket 则校验 checksum 后回 `ICMP_PORT_UNREACH`。

### 6.3 单播交付：`udp_unicast_rcv_skb()`

```c
static int udp_unicast_rcv_skb(struct sock *sk, struct sk_buff *skb,
			       struct udphdr *uh)
{
	int ret;

	if (inet_get_convert_csum(sk) && uh->check)
		skb_checksum_try_convert(skb, IPPROTO_UDP, inet_compute_pseudo);

	ret = udp_queue_rcv_skb(sk, skb);

	if (ret > 0)
		return -ret;
	return 0;
}
```

### 6.4 入队：`udp_queue_rcv_one_skb()` / `__udp_queue_rcv_skb()`

```c
static int udp_queue_rcv_one_skb(struct sock *sk, struct sk_buff *skb)
{
	struct udp_sock *up = udp_sk(sk);
	struct net *net = sock_net(sk);

	if (!xfrm4_policy_check(sk, XFRM_POLICY_IN, skb))
		goto drop;
	nf_reset_ct(skb);

	if (static_branch_unlikely(&udp_encap_needed_key) &&
	    READ_ONCE(up->encap_type)) {
		int (*encap_rcv)(struct sock *sk, struct sk_buff *skb);

		encap_rcv = READ_ONCE(up->encap_rcv);
		if (encap_rcv) {
			int ret;

			/* Verify checksum before giving to encap */
			if (udp_lib_checksum_complete(skb))
				goto csum_error;

			ret = encap_rcv(sk, skb);
			if (ret <= 0) {
				__UDP_INC_STATS(net, UDP_MIB_INDATAGRAMS);
				return -ret;
			}
		}
		/* FALLTHROUGH -- it's a UDP Packet */
	}

	prefetch(&sk->sk_rmem_alloc);
	if (rcu_access_pointer(sk->sk_filter) &&
	    udp_lib_checksum_complete(skb))
			goto csum_error;

	drop_reason = sk_filter_trim_cap(sk, skb, sizeof(struct udphdr));
	if (drop_reason)
		goto drop;

	udp_csum_pull_header(skb);

	ipv4_pktinfo_prepare(sk, skb, true);
	return __udp_queue_rcv_skb(sk, skb);
	...
}
```

封装 socket 优先：若全局 `udp_encap_needed_key` 启用且该 socket 是封装 socket，`encap_rcv` 返回值：
- `0`：包已被封装层消费，不再进入普通 UDP 队列；
- `>0`：fallthrough，继续按普通 UDP 处理；
- `<0`：按协议号 `-ret` 重新提交（例如转交 ESP）。

```c
static int __udp_queue_rcv_skb(struct sock *sk, struct sk_buff *skb)
{
	int rc;

	if (inet_sk(sk)->inet_daddr) {
		sock_rps_save_rxhash(sk, skb);
		sk_mark_napi_id(sk, skb);
		sk_incoming_cpu_update(sk);
	} else {
		sk_mark_napi_id_once(sk, skb);
	}

	rc = __udp_enqueue_schedule_skb(sk, skb);
	if (rc < 0) {
		if (rc == -ENOMEM) {
			UDP_INC_STATS(net, UDP_MIB_RCVBUFERRORS);
			drop_reason = SKB_DROP_REASON_SOCKET_RCVBUFF;
		} else {
			UDP_INC_STATS(net, UDP_MIB_MEMERRORS);
			drop_reason = SKB_DROP_REASON_PROTO_MEM;
		}
		UDP_INC_STATS(net, UDP_MIB_INERRORS);
		...
		return -1;
	}

	return 0;
}
```

`__udp_enqueue_schedule_skb()` 把 skb 推入 per-socket-per-NUMA-node 的 lockless `udp_prod_queue`（`udp_sk(sk)->udp_prod_queue[numa_node_id()]`，生产端用 llist 无锁追加），再在持锁侧拼接到 `sk_receive_queue`，并唤醒等待者。

### 6.5 Early Demux：`udp_v4_early_demux()`

在 `ip_rcv_finish_core()` 中，若 `sysctl_udp_early_demux` 开启且 skb 未分片，会提前调用 `udp_v4_early_demux()`：

```text
ip_rcv_finish_core()
  → udp_v4_early_demux(skb)
    → 拉取 UDP 头
    → 单播：__udp4_lib_demux_lookup() 在 hash2 中快速查第一个匹配
    → 组播：__udp4_lib_mcast_demux_lookup()
    → 找到 socket 后：skb->sk = sk, skb->destructor = sock_pfree
    → 可选缓存 sk->sk_rx_dst 到 skb，避免后续再查路由
```

early demux 的收益：把 socket 查找从协议交付阶段提前到 IP 层，减少缓存未命中，并可直接复用 socket 缓存的 dst。

---

## 7. UDP 校验和

### 7.1 发送侧

```c
static inline __wsum udp_csum(struct sk_buff *skb)
{
	__wsum csum = csum_partial(skb_transport_header(skb),
				   sizeof(struct udphdr), skb->csum);

	for (skb = skb_shinfo(skb)->frag_list; skb; skb = skb->next) {
		csum = csum_add(csum, skb->csum);
	}
	return csum;
}
```

```c
void udp4_hwcsum(struct sk_buff *skb, __be32 src, __be32 dst)
{
	struct udphdr *uh = udp_hdr(skb);
	int offset = skb_transport_offset(skb);
	int len = skb->len - offset;
	...
	if (!skb_has_frag_list(skb)) {
		skb->csum_start = skb_transport_header(skb) - skb->head;
		skb->csum_offset = offsetof(struct udphdr, check);
		uh->check = ~csum_tcpudp_magic(src, dst, len,
					       IPPROTO_UDP, 0);
	} else {
		/* 多 fragment：手动累加后回退为软件校验和 */
		...
		uh->check = csum_tcpudp_magic(src, dst, len, IPPROTO_UDP, csum);
		if (uh->check == 0)
			uh->check = CSUM_MANGLED_0;
	}
}
```

### 7.2 通用封装辅助：`udp_set_csum()`

```c
void udp_set_csum(bool nocheck, struct sk_buff *skb,
		  __be32 saddr, __be32 daddr, int len)
{
	struct udphdr *uh = udp_hdr(skb);

	if (nocheck) {
		uh->check = 0;
	} else if (skb_is_gso(skb)) {
		uh->check = ~udp_v4_check(len, saddr, daddr, 0);
	} else if (skb->ip_summed == CHECKSUM_PARTIAL) {
		uh->check = 0;
		uh->check = udp_v4_check(len, saddr, daddr, lco_csum(skb));
		if (uh->check == 0)
			uh->check = CSUM_MANGLED_0;
	} else {
		skb->ip_summed = CHECKSUM_PARTIAL;
		skb->csum_start = skb_transport_header(skb) - skb->head;
		skb->csum_offset = offsetof(struct udphdr, check);
		uh->check = ~udp_v4_check(len, saddr, daddr, 0);
	}
}
```

### 7.3 接收侧

接收路径校验和采用**延迟校验**策略：
- 若 skb 标记 `CHECKSUM_UNNECESSARY`（网卡已验证或 loopback），跳过；
- 否则在 `udp_queue_rcv_one_skb()` 需要时（encap、BPF filter）调用 `udp_lib_checksum_complete()`；
- 最终在 `udp_recvmsg()` 中通过 `skb_copy_and_csum_datagram_msg()` 边拷贝边完成校验。

这样避免了对被截断/丢弃的包做完整校验和计算。

---

## 8. UDP 封装与隧道

### 8.1 静态键与开关

```c
DEFINE_STATIC_KEY_FALSE(udp_encap_needed_key);

void udp_encap_enable(void)
{
	static_branch_inc(&udp_encap_needed_key);
}
EXPORT_SYMBOL(udp_encap_enable);

void udp_encap_disable(void)
{
	static_branch_dec(&udp_encap_needed_key);
}
EXPORT_SYMBOL(udp_encap_disable);
```

static key 保证：系统不存在 UDP 封装 socket 时，接收路径的封装分支完全零开销。

### 8.2 封装 socket 注册

```c
struct udp_sock {
	struct inet_sock inet;
	...
	__u8		 encap_type;

	int (*encap_rcv)(struct sock *sk, struct sk_buff *skb);
	void (*encap_err_rcv)(struct sock *sk, struct sk_buff *skb, int err,
			      __be16 port, u32 info, u8 *payload);
	int (*encap_err_lookup)(struct sock *sk, struct sk_buff *skb);
	void (*encap_destroy)(struct sock *sk);
	...
};
```

典型封装类型：
- `UDP_ENCAP_ESPINUDP`（2）：IPsec NAT-T
- `UDP_ENCAP_L2TPINUDP`（3）：L2TP over UDP
- `UDP_ENCAP_OVPNINUDP`（8）：OpenVPN

### 8.3 封装包处理

在 `udp_queue_rcv_one_skb()` 中：

```text
if (udp_encap_needed_key 启用 && up->encap_type 非 0 && up->encap_rcv 存在)
    → 先完成 UDP 校验和
    → ret = encap_rcv(sk, skb)
        ├─ ret == 0：包被封装层消费，直接返回
        ├─ ret > 0：fallthrough 到普通 UDP 路径
        └─ ret < 0：按 -ret 协议号重新提交（如 IPPROTO_ESP）
```

例如 IPsec ESP-in-UDP：

```c
int xfrm4_udp_encap_rcv(struct sock *sk, struct sk_buff *skb)
{
	int ret;

	ret = __xfrm4_udp_encap_rcv(sk, skb, true);
	if (!ret)
		return xfrm4_rcv_encap(skb, IPPROTO_ESP, 0,
				       udp_sk(sk)->encap_type);

	if (ret < 0) {
		kfree_skb(skb);
		return 0;
	}

	return ret;
}
```

### 8.4 销毁

```c
static void udp_destroy_sock(struct sock *sk)
{
	struct udp_sock *up = udp_sk(sk);
	...
	if (static_branch_unlikely(&udp_encap_needed_key)) {
		if (up->encap_type) {
			void (*encap_destroy)(struct sock *sk);

			encap_destroy = READ_ONCE(up->encap_destroy);
			if (encap_destroy)
				encap_destroy(sk);
		}
		if (udp_test_bit(ENCAP_ENABLED, sk)) {
			static_branch_dec(&udp_encap_needed_key);
			udp_tunnel_cleanup_gro(sk);
		}
	}
}
```

---

## 9. 重要 sysctl 参数与 socket 选项

### 9.1 全局内存限制

| 参数 | 路径 | 含义 |
|------|------|------|
| `udp_mem` | `/proc/sys/net/ipv4/udp_mem` | 全局 UDP 内存三级阈值 `[low, pressure, hard limit]` |
| `udp_rmem_min` | `/proc/sys/net/ipv4/udp_rmem_min` | 每个 socket 接收缓冲最小值 |
| `udp_wmem_min` | `/proc/sys/net/ipv4/udp_wmem_min` | 每个 socket 发送缓冲最小值 |

```c
void __init udp_init(void)
{
	unsigned long limit;

	udp_table_init(&udp_table, "UDP");
	limit = nr_free_buffer_pages() / 8;
	limit = max(limit, 128UL);
	sysctl_udp_mem[0] = limit / 4 * 3;
	sysctl_udp_mem[1] = limit;
	sysctl_udp_mem[2] = sysctl_udp_mem[0] * 2;
	...
}
```

### 9.2 常用 UDP socket 选项

| 选项 | 层级 | 含义 |
|------|------|------|
| `UDP_CORK` | `SOL_UDP` |  cork：累积数据，直到 `UDP_CORK` 关闭或 socket 关闭才一次性发送 |
| `UDP_ENCAP` | `SOL_UDP` |  开启封装模式（IPsec/L2TP 等） |
| `UDP_SEGMENT` | `SOL_UDP` |  设置 UDP GSO 分段大小 |
| `UDP_GRO` | `SOL_UDP` |  允许接收 UDP GRO 聚合包 |
| `UDP_NO_CHECK6_TX/RX` | `SOL_UDP` |  IPv6 发送/接收时关闭校验和 |
| `SO_REUSEADDR` | `SOL_SOCKET` |  允许地址复用 |
| `SO_REUSEPORT` | `SOL_SOCKET` |  允许多个 socket 绑定同一端口，内核做负载均衡 |
| `SO_BINDTODEVICE` | `SOL_SOCKET` |  绑定到指定网卡 |

---

## 10. UDP SNMP 统计

```c
enum {
	UDP_MIB_NUM = 0,
	UDP_MIB_INDATAGRAMS,		/* InDatagrams */
	UDP_MIB_NOPORTS,		/* NoPorts */
	UDP_MIB_INERRORS,		/* InErrors */
	UDP_MIB_OUTDATAGRAMS,		/* OutDatagrams */
	UDP_MIB_RCVBUFERRORS,		/* RcvbufErrors */
	UDP_MIB_SNDBUFERRORS,		/* SndbufErrors */
	UDP_MIB_CSUMERRORS,		/* InCsumErrors */
	UDP_MIB_IGNOREDMULTI,		/* IgnoredMulti */
	UDP_MIB_MEMERRORS,		/* MemErrors */
	__UDP_MIB_MAX
};
```

| 计数器 | 典型触发场景 |
|--------|--------------|
| `InDatagrams` | 成功收包 |
| `NoPorts` | 无匹配 socket，回 ICMP port unreachable |
| `InErrors` | 入包通用错误（校验失败、队列满、xfrm 失败等） |
| `OutDatagrams` | 成功发送 |
| `RcvbufErrors` | socket 接收缓冲不足 |
| `SndbufErrors` | socket 发送缓冲不足 |
| `InCsumErrors` | UDP 校验和错误 |
| `IgnoredMulti` | 组播报文无 socket 被丢弃 |
| `MemErrors` | 全局/协议内存限制丢包 |

对应 `/proc/net/snmp` 中的 `Udp:` 行。

---

## 11. 小结

- **无连接、无状态**是 UDP 的核心特征，Linux 内核没有为它维护类似 TCP 的连接状态机。
- **端口/socket 查找**是 UDP 的关键路径：Linux 用 `hash` + `hash2` + `hash4` 三张表加速，connected socket 可走 O(1) 精确匹配，wildcard socket 通过 `compute_score()` 评分择优。
- **校验和**支持软件、硬件卸载、关闭三种模式；接收侧延迟校验以节省 CPU。
- **GSO/GRO** 让 UDP 也能做批量分段/聚合；`UDP_SEGMENT` cmsg 是高性能 UDP 发送（如 QUIC）的常用手段。
- **封装/隧道**通过 `udp_encap_needed_key` 实现零开销快速路径，IPsec NAT-T、L2TP、OpenVPN 等都依赖这套机制。
- **内存 accounting** 与全局 `udp_mem`、per-socket `udp_rmem_min`/`udp_wmem_min` 共同决定 UDP 在高负载下是否丢包。
