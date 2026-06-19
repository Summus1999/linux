# IPv4 协议详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/ip_input.c` —— IPv4 接收路径
> - `net/ipv4/ip_output.c` —— IPv4 发送路径与分片
> - `net/ipv4/ip_forward.c` —— IPv4 转发
> - `net/ipv4/ip_fragment.c` —— IPv4 重组
> - `include/uapi/linux/ip.h` —— IPv4 报文头定义
> - `include/net/ip.h` —— IPv4 内部结构/宏/函数声明
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图和时序图使用 `text` 代码块表示。

---

## 1. IP 协议基础

作用：IP（Internet Protocol，互联网协议）是 TCP/IP 协议栈的网络层协议，负责把数据包从源主机路由到目的主机。它提供的是无连接、尽力而为（best-effort）的交付服务，不保证可靠性、顺序性和不重复性。

核心 RFC：RFC 791。

典型交互：

```text
主机 A (192.168.1.2) 要发一个 TCP 段给 192.168.1.3
→ TCP 把数据交给 IP 层
→ IP 封装成 IP 数据报（填源/目的 IP、TTL、协议号、校验和等）
→ 如果目的在同一二层网段，ARP 解析目的 MAC 后直接发
→ 如果在不同网段，查路由表交给下一跳网关
→ 转发时每经过一个路由器 TTL 减 1，逐跳转发直到目的主机
→ 目的主机 IP 层解封装，根据 protocol 字段交给 TCP/UDP/ICMP 等上层
```

与 ARP 的关系：ARP 是 IP 的“辅助协议”。IP 只知道下一跳的 IP 地址，真正发以太网帧时需要 ARP（或静态邻居表）把 IP 解析成 MAC。

---

## 2. IPv4 报文头格式

```c
struct iphdr {
#if defined(__LITTLE_ENDIAN_BITFIELD)
	__u8	ihl:4,
		version:4;
#elif defined (__BIG_ENDIAN_BITFIELD)
	__u8	version:4,
  		ihl:4;
#else
#error	"Please fix <asm/byteorder.h>"
#endif
	__u8	tos;
	__be16	tot_len;
	__be16	id;
	__be16	frag_off;
	__u8	ttl;
	__u8	protocol;
	__sum16	check;
	__struct_group(/* no tag */, addrs, /* no attrs */,
		__be32	saddr;
		__be32	daddr;
	);
	/*The options start here. */
};
```

### 2.1 字段解释

| 字段 | 长度 | 含义 |
|------|------|------|
| `version` | 4 bit | IP 版本，IPv4 固定为 4 |
| `ihl` | 4 bit | IP 头部长度，以 4 字节为单位，最小 5（即 20 字节），最大 15（即 60 字节） |
| `tos` | 1 B | Type of Service，现在多用于 DSCP/ECN |
| `tot_len` | 2 B | 整个 IP 数据报长度（头 + 数据），最大 65535 字节 |
| `id` | 2 B | 分片标识，同一原始报文的所有分片 ID 相同 |
| `frag_off` | 2 B | 分片偏移 + 标志位（DF/MF） |
| `ttl` | 1 B | Time to Live，转发时每经一路由器减 1；路由器发现 TTL <= 1 时丢弃并回 ICMP Time Exceeded |
| `protocol` | 1 B | 上层协议号：ICMP=1、TCP=6、UDP=17 等 |
| `check` | 2 B | IP 头部校验和（仅校验头部，不校验数据） |
| `saddr` | 4 B | 源 IP 地址 |
| `daddr` | 4 B | 目的 IP 地址 |
| options | 可变 | IP 选项，如 Record Route、Timestamp、Loose/Strict Source Route |

### 2.2 frag_off 字段相关标志

定义在 `include/net/ip.h`：

```c
#define IP_CE		0x8000		/* Flag: "Congestion"		*/
#define IP_DF		0x4000		/* Flag: "Don't Fragment"	*/
#define IP_MF		0x2000		/* Flag: "More Fragments"	*/
#define IP_OFFSET	0x1FFF		/* "Fragment Offset" part	*/
```

- `DF`（Don't Fragment）：置 1 表示不允许分片。路径上某跳 MTU 不足且 DF=1 时，路由器会丢弃并回 `ICMP_FRAG_NEEDED`（Path MTU Discovery 的基础）。
- `MF`（More Fragments）：置 1 表示后面还有分片；最后一个分片 MF=0。
- `frag_off & IP_OFFSET`：分片偏移，以 8 字节为单位。
- `IP_CE` 也是 `frag_off` 字段中的标志位，但它和拥塞指示相关，不是普通分片决策的核心标志。

### 2.3 以太网上的 IPv4 帧布局

```text
| 以太网头 (14B) | IPv4 头 (20~60B) | TCP/UDP/ICMP 头 + 数据 |
```

以太网头中 `ethertype = 0x0800` 表示上层是 IPv4。

---

## 3. Linux 内核 IPv4 实现总览

| 项目 | 文件/结构/函数 |
|------|---------------|
| IPv4 接收主文件 | `net/ipv4/ip_input.c` |
| IPv4 发送主文件 | `net/ipv4/ip_output.c` |
| IPv4 转发 | `net/ipv4/ip_forward.c` |
| IPv4 重组 | `net/ipv4/ip_fragment.c` |
| IPv4 头定义 | `include/uapi/linux/ip.h` 中的 `struct iphdr` |
| 收包入口 | `ip_rcv()` |
| 本地交付入口 | `ip_local_deliver()` |
| 上层协议分发 | `ip_protocol_deliver_rcu()` |
| 发包入口（TCP 等用） | `ip_queue_xmit()` / `__ip_queue_xmit()` |
| 构造 IP 头并发送（特定路径辅助函数） | `ip_build_and_send_pkt()` |
| 转发入口 | `ip_forward()` |
| 分片入口 | `ip_fragment()` / `ip_do_fragment()` |
| 重组入口 | `ip_defrag()` |
| 路由查找 | `ip_route_input_noref()` / `ip_route_output_flow()` |
| 头部校验和计算 | `ip_send_check()` |

与 ARP 文档的衔接：ARP 解决的是“下一跳 IP → MAC”的问题；IP 解决的是“源 IP → 目的 IP 如何路由、如何封装/解封装、如何分片/重组”的问题。两者通过 `struct rtable`/`struct dst_entry` 和 `struct neighbour` 紧密协作。

---

## 4. IP 数据报接收流程

### 4.1 整体调用链

```text
网卡驱动收到帧
  → __netif_receive_skb_core()
    → 根据 ethertype = 0x0800 找到 packet_type
    → ip_rcv(skb, dev, pt, orig_dev)
      → ip_rcv_core()           做基础校验
      → NF_HOOK(NF_INET_PRE_ROUTING, ip_rcv_finish)
        → ip_rcv_finish()
          → ip_rcv_finish_core()   路由查找
          → dst_input(skb)
            ├─ 目的为本机 → ip_local_deliver()
            │              → ip_defrag() 重组
            │              → NF_HOOK(NF_INET_LOCAL_IN, ip_local_deliver_finish)
            │              → ip_protocol_deliver_rcu() 交给 TCP/UDP/ICMP
            └─ 需要转发  → ip_forward()
```

### 4.2 ip_rcv_core：报文合法性校验

`ip_rcv_core()` 是接收路径的“第一道关卡”，位于 `net/ipv4/ip_input.c`：

```c
static struct sk_buff *ip_rcv_core(struct sk_buff *skb, struct net *net)
{
	const struct iphdr *iph;
	int drop_reason;
	u32 len;

	/* When the interface is in promisc. mode, drop all the crap
	 * that it receives, do not try to analyse it.
	 */
	if (skb->pkt_type == PACKET_OTHERHOST) {
		dev_core_stats_rx_otherhost_dropped_inc(skb->dev);
		drop_reason = SKB_DROP_REASON_OTHERHOST;
		goto drop;
	}

	__IP_UPD_PO_STATS(net, IPSTATS_MIB_IN, skb->len);

	skb = skb_share_check(skb, GFP_ATOMIC);
	if (!skb) {
		__IP_INC_STATS(net, IPSTATS_MIB_INDISCARDS);
		goto out;
	}

	drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;
	if (!pskb_may_pull(skb, sizeof(struct iphdr)))
		goto inhdr_error;

	iph = ip_hdr(skb);

	if (iph->ihl < 5 || iph->version != 4)
		goto inhdr_error;

	if (!pskb_may_pull(skb, iph->ihl*4))
		goto inhdr_error;

	iph = ip_hdr(skb);

	if (unlikely(ip_fast_csum((u8 *)iph, iph->ihl)))
		goto csum_error;

	len = iph_totlen(skb, iph);
	if (skb->len < len) {
		drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
		__IP_INC_STATS(net, IPSTATS_MIB_INTRUNCATEDPKTS);
		goto drop;
	} else if (len < (iph->ihl*4))
		goto inhdr_error;

	if (pskb_trim_rcsum(skb, len)) {
		__IP_INC_STATS(net, IPSTATS_MIB_INDISCARDS);
		goto drop;
	}

	iph = ip_hdr(skb);
	skb->transport_header = skb->network_header + iph->ihl*4;

	memset(IPCB(skb), 0, sizeof(struct inet_skb_parm));
	IPCB(skb)->iif = skb->skb_iif;

	if (!skb_sk_is_prefetched(skb))
		skb_orphan(skb);

	return skb;

/* ... error handling ... */
}
```

校验要点：
1. 不是发给自己的包（`PACKET_OTHERHOST`）直接丢弃。
2. `skb_share_check()`：如果 skb 是 clone/shared，则复制一份私有副本（去克隆化），避免后续修改影响共享者。
3. 头部至少 20 字节，且 `version == 4`、`ihl >= 5`。
4. 实际拉取到 `ihl*4` 字节后做 IP 头部校验和检查。
5. `tot_len` 必须落在合理范围：`ihl*4 <= tot_len <= skb->len`。
6. 把 `skb->len` 截断到 `tot_len`，去掉以太网填充。
7. 设置 `transport_header`，清空 `IPCB(skb)`（IP 控制块）。

### 4.3 ip_rcv_finish / ip_rcv_finish_core：路由判定

```c
static int ip_rcv_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	int ret;

	skb = l3mdev_ip_rcv(skb);
	if (!skb)
		return NET_RX_SUCCESS;

	ret = ip_rcv_finish_core(net, skb, dev, NULL);
	if (ret != NET_RX_DROP)
		ret = dst_input(skb);
	return ret;
}
```

`ip_rcv_finish_core()` 的核心任务是为报文找到路由（`dst_entry`）：

```c
static int ip_rcv_finish_core(struct net *net,
			      struct sk_buff *skb, struct net_device *dev,
			      const struct sk_buff *hint)
{
	const struct iphdr *iph = ip_hdr(skb);
	struct rtable *rt;
	int drop_reason;

	if (ip_can_use_hint(skb, iph, hint)) {
		drop_reason = ip_route_use_hint(skb, iph->daddr, iph->saddr,
						ip4h_dscp(iph), dev, hint);
		if (unlikely(drop_reason))
			goto drop_error;
	}

	if (READ_ONCE(net->ipv4.sysctl_ip_early_demux) &&
	    !skb_dst(skb) &&
	    !skb->sk &&
	    !ip_is_fragment(iph)) {
		switch (iph->protocol) {
		case IPPROTO_TCP:
			if (READ_ONCE(net->ipv4.sysctl_tcp_early_demux)) {
				tcp_v4_early_demux(skb);
				iph = ip_hdr(skb);
			}
			break;
		case IPPROTO_UDP:
			if (READ_ONCE(net->ipv4.sysctl_udp_early_demux)) {
				drop_reason = udp_v4_early_demux(skb);
				if (unlikely(drop_reason))
					goto drop_error;
				iph = ip_hdr(skb);
			}
			break;
		}
	}

	if (!skb_valid_dst(skb)) {
		drop_reason = ip_route_input_noref(skb, iph->daddr, iph->saddr,
						   ip4h_dscp(iph), dev);
		if (unlikely(drop_reason))
			goto drop_error;
	}

	/* ... */

	if (iph->ihl > 5) {
		drop_reason = ip_rcv_options(skb, dev);
		if (drop_reason)
			goto drop;
	}

	rt = skb_rtable(skb);
	if (rt->rt_type == RTN_MULTICAST) {
		__IP_UPD_PO_STATS(net, IPSTATS_MIB_INMCAST, skb->len);
	} else if (rt->rt_type == RTN_BROADCAST) {
		__IP_UPD_PO_STATS(net, IPSTATS_MIB_INBCAST, skb->len);
	}

	return NET_RX_SUCCESS;
/* ... */
}
```

关键逻辑：
- `sysctl_ip_early_demux`：TCP/UDP 早期解复用，根据四元组提前找到 socket，可能顺带缓存 `dst`。
- `ip_route_input_noref()`：根据目的 IP、源 IP、TOS、入接口查找路由。这是 IP 层最核心的路由判定。
- 如果头部带 options（`ihl > 5`），调用 `ip_rcv_options()` 处理。
- 后续动作由 `dst->input` 函数指针决定，而不是 `rt->rt_type`：本地路由指向 `ip_local_deliver()`，单播转发路由指向 `ip_forward()`；`rt->rt_type` 只是分类标签，用于统计和策略判断。

### 4.4 dst_input：根据路由类型分发

`dst_input()` 是通用入口，最终调用 `dst->input()`：

```text
rt->dst.input 通常指向：
  - ip_local_deliver()      目的为本机
  - ip_forward()            需要转发
  - ip_mr_input()           组播路由（CONFIG_IP_MROUTE）
  - ip_discard() / 错误路径
```

### 4.5 ip_local_deliver：本机交付

```c
int ip_local_deliver(struct sk_buff *skb)
{
	struct net *net = dev_net(skb->dev);

	if (ip_is_fragment(ip_hdr(skb))) {
		if (ip_defrag(net, skb, IP_DEFRAG_LOCAL_DELIVER))
			return 0;
	}

	return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN,
		       net, NULL, skb, skb->dev, NULL,
		       ip_local_deliver_finish);
}
EXPORT_SYMBOL(ip_local_deliver);
```

- 如果是分片，先进入 `ip_defrag()` 重组；重组完成前返回 0，原 skb 被接管。
- 重组完成后经 `NF_INET_LOCAL_IN` netfilter 链，最终调用 `ip_local_deliver_finish()`。

### 4.6 ip_local_deliver_finish：去掉 IP 头并交给上层

```c
static int ip_local_deliver_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC))) {
		__IP_INC_STATS(net, IPSTATS_MIB_INDISCARDS);
		kfree_skb_reason(skb, SKB_DROP_REASON_NOMEM);
		return 0;
	}

	skb_clear_delivery_time(skb);
	__skb_pull(skb, skb_network_header_len(skb));

	rcu_read_lock();
	ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol);
	rcu_read_unlock();

	return 0;
}
```

`__skb_pull()` 把 data 指针移到 IP 头之后，也就是传输层头部开始的位置。

### 4.7 ip_protocol_deliver_rcu：上层协议分发

```c
void ip_protocol_deliver_rcu(struct net *net, struct sk_buff *skb, int protocol)
{
	const struct net_protocol *ipprot;
	int raw, ret;

resubmit:
	raw = raw_local_deliver(skb, protocol);

	ipprot = rcu_dereference(inet_protos[protocol]);
	if (ipprot) {
		if (!ipprot->no_policy) {
			if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
				kfree_skb_reason(skb,
						 SKB_DROP_REASON_XFRM_POLICY);
				return;
			}
			nf_reset_ct(skb);
		}
		ret = INDIRECT_CALL_2(ipprot->handler, tcp_v4_rcv, udp_rcv,
				      skb);
		if (ret < 0) {
			protocol = -ret;
			goto resubmit;
		}
		__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
	} else {
		if (!raw) {
			if (xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
				__IP_INC_STATS(net, IPSTATS_MIB_INUNKNOWNPROTOS);
				icmp_send(skb, ICMP_DEST_UNREACH,
					  ICMP_PROT_UNREACH, 0);
			}
			kfree_skb_reason(skb, SKB_DROP_REASON_IP_NOPROTO);
		} else {
			__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
			consume_skb(skb);
		}
	}
}
```

要点：
- `inet_protos[protocol]` 是协议注册表，TCP 注册为 `IPPROTO_TCP`，UDP 注册为 `IPPROTO_UDP`，ICMP 注册为 `IPPROTO_ICMP`。
- 先尝试交给 raw socket（`raw_local_deliver`）。
- 如果找到上层协议，调用其 `handler`（如 `tcp_v4_rcv()`、`udp_rcv()`、`icmp_rcv()`）。
- 如果 `handler` 返回负数，表示它把报文重新注入为另一种协议（如 IP-IP 隧道），则重新分发。
- 如果没有注册协议且没有 raw socket 接收，回送 `ICMP_DEST_UNREACH / ICMP_PROT_UNREACH`。

---

## 5. IP 数据报发送流程

### 5.1 整体调用链（以 TCP 为例）

```text
tcp_write_xmit() / tcp_push_one()
  → tcp_transmit_skb()
    → ip_queue_xmit(skb, sk)
      → __ip_queue_xmit()
        → 路由查找（ip_route_output_flow）
        → 构造 IP 头
        → ip_local_out()
          → __ip_local_out()    计算 tot_len + 头部校验和
          → NF_INET_LOCAL_OUT
          → dst_output()
            → ip_output()
              → NF_INET_POST_ROUTING
              → ip_finish_output()
                → __ip_finish_output()
                  → 如果需要分片：ip_fragment() → ip_do_fragment()
                  → ip_finish_output2()
                    → neigh_output() / dst_neigh_output()
                      → ARP 解析 / 邻居子系统
                      → dev_queue_xmit()
```

### 5.2 __ip_queue_xmit：构造 IP 头

```c
int __ip_queue_xmit(struct sock *sk, struct sk_buff *skb, struct flowi *fl,
		    __u8 tos)
{
	struct inet_sock *inet = inet_sk(sk);
	struct net *net = sock_net(sk);
	struct ip_options_rcu *inet_opt;
	struct flowi4 *fl4;
	struct rtable *rt;
	struct iphdr *iph;
	int res;

	rcu_read_lock();
	inet_opt = rcu_dereference(inet->inet_opt);
	fl4 = &fl->u.ip4;
	rt = skb_rtable(skb);
	if (rt)
		goto packet_routed;

	rt = dst_rtable(__sk_dst_check(sk, 0));
	if (!rt) {
		inet_sk_init_flowi4(inet, fl4);

		fl4->flowi4_dscp = inet_dsfield_to_dscp(tos);

		rt = ip_route_output_flow(net, fl4, sk);
		if (IS_ERR(rt))
			goto no_route;
		sk_setup_caps(sk, &rt->dst);
	}
	skb_dst_set_noref(skb, &rt->dst);

packet_routed:
	if (inet_opt && inet_opt->opt.is_strictroute && rt->rt_uses_gateway)
		goto no_route;

	/* OK, we know where to send it, allocate and build IP header. */
	skb_push(skb, sizeof(struct iphdr) + (inet_opt ? inet_opt->opt.optlen : 0));
	skb_reset_network_header(skb);
	iph = ip_hdr(skb);
	*((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (tos & 0xff));
	if (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df)
		iph->frag_off = htons(IP_DF);
	else
		iph->frag_off = 0;
	iph->ttl      = ip_select_ttl(inet, &rt->dst);
	iph->protocol = sk->sk_protocol;
	ip_copy_addrs(iph, fl4);

	if (inet_opt && inet_opt->opt.optlen) {
		iph->ihl += inet_opt->opt.optlen >> 2;
		ip_options_build(skb, &inet_opt->opt, inet->inet_daddr, rt);
	}

	ip_select_ident_segs(net, skb, sk,
			     skb_shinfo(skb)->gso_segs ?: 1);

	skb->priority = READ_ONCE(sk->sk_priority);
	skb->mark = READ_ONCE(sk->sk_mark);

	res = ip_local_out(net, sk, skb);
	rcu_read_unlock();
	return res;

no_route:
	rcu_read_unlock();
	IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
	kfree_skb_reason(skb, SKB_DROP_REASON_IP_OUTNOROUTES);
	return -EHOSTUNREACH;
}
```

要点：
- 优先使用 socket 缓存的路由（`__sk_dst_check`），失效则调用 `ip_route_output_flow()` 重新查找。
- `skb_push()` 腾出 IP 头空间，`skb_reset_network_header()` 设置网络层头指针。
- 一次写两个字节 `*((__be16 *)iph) = htons((4 << 12) | (5 << 8) | (tos & 0xff));` 同时设置 version=4、ihl=5、tos。
- 根据 `ip_dont_fragment()` 决定是否置 `IP_DF`。
- `ip_select_ttl()` 选择 TTL（通常 64，可由 socket 或路由覆盖）。
- `ip_select_ident_segs()` 设置 IP ID。

### 5.3 ip_local_out / __ip_local_out：校验和与 Local Out 钩子

```c
void ip_send_check(struct iphdr *iph)
{
	iph->check = 0;
	iph->check = ip_fast_csum((unsigned char *)iph, iph->ihl);
}
EXPORT_SYMBOL(ip_send_check);

int __ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct iphdr *iph = ip_hdr(skb);

	IP_INC_STATS(net, IPSTATS_MIB_OUTREQUESTS);

	iph_set_totlen(iph, skb->len);
	ip_send_check(iph);

	skb = l3mdev_ip_out(sk, skb);
	if (unlikely(!skb))
		return 0;

	skb->protocol = htons(ETH_P_IP);

	return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT,
		       net, sk, skb, NULL, skb_dst_dev(skb),
		       dst_output);
}

int ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	int err;

	err = __ip_local_out(net, sk, skb);
	if (likely(err == 1))
		err = dst_output(net, sk, skb);

	return err;
}
```

- `iph_set_totlen()` 把 `skb->len` 写入 `iph->tot_len`。
- `ip_send_check()` 计算并填入 IP 头部校验和。
- 经 `NF_INET_LOCAL_OUT` netfilter 链后，调用 `dst_output()`。

### 5.4 ip_output → ip_finish_output2：最终交给邻居子系统

```c
int ip_output(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev, *indev = skb->dev;
	int ret_val;

	rcu_read_lock();
	dev = skb_dst_dev_rcu(skb);
	skb->dev = dev;
	skb->protocol = htons(ETH_P_IP);

	ret_val = NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING,
				net, sk, skb, indev, dev,
				ip_finish_output,
				!(IPCB(skb)->flags & IPSKB_REROUTED));
	rcu_read_unlock();
	return ret_val;
}
```

`ip_finish_output()` → `__ip_finish_output()` 处理分片和 GSO：

```c
static int __ip_finish_output(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	unsigned int mtu;

#if defined(CONFIG_NETFILTER) && defined(CONFIG_XFRM)
	if (skb_dst(skb)->xfrm) {
		IPCB(skb)->flags |= IPSKB_REROUTED;
		return dst_output(net, sk, skb);
	}
#endif
	mtu = ip_skb_dst_mtu(sk, skb);
	if (skb_is_gso(skb))
		return ip_finish_output_gso(net, sk, skb, mtu);

	if (skb->len > mtu || IPCB(skb)->frag_max_size)
		return ip_fragment(net, sk, skb, mtu, ip_finish_output2);

	return ip_finish_output2(net, sk, skb);
}
```

`ip_finish_output2()` 找到邻居并调用 `neigh_output()`：

```c
static int ip_finish_output2(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct dst_entry *dst = skb_dst(skb);
	struct rtable *rt = dst_rtable(dst);
	struct net_device *dev = dst_dev(dst);
	unsigned int hh_len = LL_RESERVED_SPACE(dev);
	struct neighbour *neigh;
	bool is_v6gw = false;

	if (rt->rt_type == RTN_MULTICAST) {
		IP_UPD_PO_STATS(net, IPSTATS_MIB_OUTMCAST, skb->len);
	} else if (rt->rt_type == RTN_BROADCAST)
		IP_UPD_PO_STATS(net, IPSTATS_MIB_OUTBCAST, skb->len);

	IP_UPD_PO_STATS(net, IPSTATS_MIB_OUT, skb->len);

	if (unlikely(skb_headroom(skb) < hh_len && dev->header_ops)) {
		skb = skb_expand_head(skb, hh_len);
		if (!skb)
			return -ENOMEM;
	}

	rcu_read_lock();
	neigh = ip_neigh_for_gw(rt, skb, &is_v6gw);
	if (!IS_ERR(neigh)) {
		int res;

		sock_confirm_neigh(skb, neigh);
		res = neigh_output(neigh, skb, is_v6gw);
		rcu_read_unlock();
		return res;
	}
	rcu_read_unlock();

	net_dbg_ratelimited("%s: No header cache and no neighbour!\n",
			    __func__);
	kfree_skb_reason(skb, SKB_DROP_REASON_NEIGH_CREATEFAIL);
	return PTR_ERR(neigh);
}
```

- `ip_neigh_for_gw()` 根据路由找到下一跳网关对应的 `struct neighbour`。
- `neigh_output()` 进入邻居子系统，最终调用 `neigh->output()`（可能是 `neigh_resolve_output()` 触发 ARP）。
- 从这里开始，就是 ARP 文档详细讲的内容：ARP 解析 MAC、NUD 状态机、排队、重试、最终 `dev_queue_xmit()`。

---

## 6. IP 转发流程

### 6.1 触发条件

当 `dst->input()` 指向 `ip_forward()` 时，报文目的不是本机，需要继续转发。

### 6.2 ip_forward 源码要点

```c
int ip_forward(struct sk_buff *skb)
{
	u32 mtu;
	struct iphdr *iph;	/* Our header */
	struct rtable *rt;	/* Route we use */
	struct ip_options *opt = &(IPCB(skb)->opt);
	struct net *net;
	SKB_DR(reason);

	if (skb->pkt_type != PACKET_HOST)
		goto drop;

	if (unlikely(skb->sk))
		goto drop;

	if (skb_warn_if_lro(skb))
		goto drop;

	if (!xfrm4_policy_check(NULL, XFRM_POLICY_FWD, skb)) {
		SKB_DR_SET(reason, XFRM_POLICY);
		goto drop;
	}

	if (IPCB(skb)->opt.router_alert && ip_call_ra_chain(skb))
		return NET_RX_SUCCESS;

	skb_forward_csum(skb);
	net = dev_net(skb->dev);

	if (ip_hdr(skb)->ttl <= 1)
		goto too_many_hops;

	if (!xfrm4_route_forward(skb)) {
		SKB_DR_SET(reason, XFRM_POLICY);
		goto drop;
	}

	rt = skb_rtable(skb);

	if (opt->is_strictroute && rt->rt_uses_gateway)
		goto sr_failed;

	__IP_INC_STATS(net, IPSTATS_MIB_OUTFORWDATAGRAMS);

	IPCB(skb)->flags |= IPSKB_FORWARDED;
	mtu = ip_dst_mtu_maybe_forward(&rt->dst, true);
	if (ip_exceeds_mtu(skb, mtu)) {
		IP_INC_STATS(net, IPSTATS_MIB_FRAGFAILS);
		icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED,
			  htonl(mtu));
		SKB_DR_SET(reason, PKT_TOO_BIG);
		goto drop;
	}

	if (skb_cow(skb, LL_RESERVED_SPACE(rt->dst.dev)+rt->dst.header_len))
		goto drop;
	iph = ip_hdr(skb);

	ip_decrease_ttl(iph);

	if (IPCB(skb)->flags & IPSKB_DOREDIRECT && !opt->srr &&
	    !skb_sec_path(skb))
		ip_rt_send_redirect(skb);

	if (READ_ONCE(net->ipv4.sysctl_ip_fwd_update_priority))
		skb->priority = rt_tos2priority(iph->tos);

	return NF_HOOK(NFPROTO_IPV4, NF_INET_FORWARD,
		       net, NULL, skb, skb->dev, rt->dst.dev,
		       ip_forward_finish);

/* ... error handling ... */
}
```

转发要点：
1. 只处理单播给自己的包（`PACKET_HOST`）。
2. 本机 socket 产生的包不走 forward。
3. TTL <= 1 时丢弃并回 `ICMP_TIME_EXCEEDED / ICMP_EXC_TTL`。
4. 如果报文大于出接口 MTU、DF=1 且不能忽略 DF，丢弃并回 `ICMP_FRAG_NEEDED`（PMTU 发现）；`skb->ignore_df` 等内部场景可绕过 DF。
5. `ip_decrease_ttl(iph)`：TTL 减 1，同时重新计算头部校验和（利用增量更新）。
6. 可选发送 ICMP Redirect（若发现更优路由）。
7. 经 `NF_INET_FORWARD` 钩子后进入 `ip_forward_finish()` → `dst_output()`。

### 6.3 ip_exceeds_mtu

```c
static bool ip_exceeds_mtu(const struct sk_buff *skb, unsigned int mtu)
{
	if (skb->len <= mtu)
		return false;

	if (unlikely((ip_hdr(skb)->frag_off & htons(IP_DF)) == 0))
		return false;

	if (unlikely(IPCB(skb)->frag_max_size > mtu))
		return true;

	if (skb->ignore_df)
		return false;

	if (skb_is_gso(skb) && skb_gso_validate_network_len(skb, mtu))
		return false;

	return true;
}
```

---

## 7. IP 分片与重组

### 7.1 为什么要分片

IPv4 数据报最大 65535 字节，但链路层 MTU 通常只有 1500（以太网）。当 IP 数据报大于出接口 MTU 且允许分片时，需要切成多个小片发送；通常这意味着 DF=0，Linux 内部也可能因 `skb->ignore_df` 允许分片。

### 7.2 分片：ip_fragment / ip_do_fragment

```c
static int ip_fragment(struct net *net, struct sock *sk, struct sk_buff *skb,
		       unsigned int mtu,
		       int (*output)(struct net *, struct sock *, struct sk_buff *))
{
	struct iphdr *iph = ip_hdr(skb);

	if ((iph->frag_off & htons(IP_DF)) == 0)
		return ip_do_fragment(net, sk, skb, output);

	if (unlikely(!skb->ignore_df ||
		     (IPCB(skb)->frag_max_size &&
		      IPCB(skb)->frag_max_size > mtu))) {
		IP_INC_STATS(net, IPSTATS_MIB_FRAGFAILS);
		icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED,
			  htonl(mtu));
		kfree_skb(skb);
		return -EMSGSIZE;
	}

	return ip_do_fragment(net, sk, skb, output);
}
```

- 如果 `DF=0`，直接分片。
- 如果 `DF=1` 但 `skb->ignore_df` 为真（某些隧道/内部使用），也允许分片。
- 否则丢弃并回 ICMP。

`ip_do_fragment()` 有两种路径：
1. Fast path：原 skb 已有 `frag_list`，直接遍历并填充各分片 IP 头。
2. Slow path：逐个分配新 skb，从原 skb 拷贝数据，再填 IP 头。

慢路径核心片段：

```c
void ip_frag_init(struct sk_buff *skb, unsigned int hlen,
		  unsigned int ll_rs, unsigned int mtu, bool DF,
		  struct ip_frag_state *state)
{
	struct iphdr *iph = ip_hdr(skb);

	state->DF = DF;
	state->hlen = hlen;
	state->ll_rs = ll_rs;
	state->mtu = mtu;

	state->left = skb->len - hlen;	/* Space per frame */
	state->ptr = hlen;		/* Where to start from */

	state->offset = (ntohs(iph->frag_off) & IP_OFFSET) << 3;
	state->not_last_frag = iph->frag_off & htons(IP_MF);
}

struct sk_buff *ip_frag_next(struct sk_buff *skb, struct ip_frag_state *state)
{
	unsigned int len = state->left;
	struct sk_buff *skb2;
	struct iphdr *iph;

	if (len > state->mtu)
		len = state->mtu;
	if (len < state->left)
		len &= ~7;

	skb2 = alloc_skb(len + state->hlen + state->ll_rs, GFP_ATOMIC);
	if (!skb2)
		return ERR_PTR(-ENOMEM);

	ip_copy_metadata(skb2, skb);
	skb_reserve(skb2, state->ll_rs);
	skb_put(skb2, len + state->hlen);
	skb_reset_network_header(skb2);
	skb2->transport_header = skb2->network_header + state->hlen;

	if (skb->sk)
		skb_set_owner_w(skb2, skb->sk);

	skb_copy_from_linear_data(skb, skb_network_header(skb2), state->hlen);

	if (skb_copy_bits(skb, state->ptr, skb_transport_header(skb2), len))
		BUG();
	state->left -= len;

	iph = ip_hdr(skb2);
	iph->frag_off = htons((state->offset >> 3));
	if (state->DF)
		iph->frag_off |= htons(IP_DF);

	if (state->left > 0 || state->not_last_frag)
		iph->frag_off |= htons(IP_MF);
	state->ptr += len;
	state->offset += len;

	iph->tot_len = htons(len + state->hlen);

	ip_send_check(iph);

	return skb2;
}
```

分片规则：
- 除最后一片外，每片数据长度必须是 8 的倍数（`len &= ~7`）。
- 每片都拷贝原始 IP 头，修改 `tot_len`、`frag_off`、`MF`、校验和。
- 只有偏移 0 的分片包含 IP payload 的起始字节，因此通常包含传输层头部的开始；但不保证完整传输层头都在第一片中。
- RFC 层面按 `(saddr, daddr, protocol, id)` 判断同一份原始报文；Linux 实现额外加入 `user` 和 `vif`，实际用 6 元组 `(saddr, daddr, protocol, id, user, vif)` 作为重组队列键。

### 7.3 重组：ip_defrag

```c
int ip_defrag(struct net *net, struct sk_buff *skb, u32 user)
{
	struct net_device *dev;
	struct ipq *qp;
	int vif;

	__IP_INC_STATS(net, IPSTATS_MIB_REASMREQDS);

	rcu_read_lock();
	dev = skb->dev ? : skb_dst_dev_rcu(skb);
	vif = l3mdev_master_ifindex_rcu(dev);
	qp = ip_find(net, ip_hdr(skb), user, vif);
	if (qp) {
		int ret, refs = 0;

		spin_lock(&qp->q.lock);

		ret = ip_frag_queue(qp, skb, &refs);

		spin_unlock(&qp->q.lock);
		rcu_read_unlock();
		inet_frag_putn(&qp->q, refs);
		return ret;
	}
	rcu_read_unlock();

	__IP_INC_STATS(net, IPSTATS_MIB_REASMFAILS);
	kfree_skb(skb);
	return -ENOMEM;
}
```

`ip_find()` 根据 6 元组查找或创建重组队列：

```c
static struct ipq *ip_find(struct net *net, struct iphdr *iph,
			   u32 user, int vif)
{
	struct frag_v4_compare_key key = {
		.saddr = iph->saddr,
		.daddr = iph->daddr,
		.user = user,
		.vif = vif,
		.id = iph->id,
		.protocol = iph->protocol,
	};
	struct inet_frag_queue *q;

	q = inet_frag_find(net->ipv4.fqdir, &key);
	if (!q)
		return NULL;

	return container_of(q, struct ipq, q);
}
```

### 7.4 重组队列插入：ip_frag_queue

```c
static int ip_frag_queue(struct ipq *qp, struct sk_buff *skb, int *refs)
{
	struct net *net = qp->q.fqdir->net;
	int ihl, end, flags, offset;
	struct sk_buff *prev_tail;
	struct net_device *dev;
	unsigned int fragsize;
	int err = -ENOENT;
	SKB_DR(reason);
	u8 ecn;

	if (qp->q.flags & INET_FRAG_COMPLETE) {
		SKB_DR_SET(reason, DUP_FRAG);
		goto err;
	}

	ecn = ip4_frag_ecn(ip_hdr(skb)->tos);
	offset = ntohs(ip_hdr(skb)->frag_off);
	flags = offset & ~IP_OFFSET;
	offset &= IP_OFFSET;
	offset <<= 3;		/* offset is in 8-byte chunks */
	ihl = ip_hdrlen(skb);

	end = offset + skb->len - skb_network_offset(skb) - ihl;

	if ((flags & IP_MF) == 0) {
		if (end < qp->q.len ||
		    ((qp->q.flags & INET_FRAG_LAST_IN) && end != qp->q.len))
			goto discard_qp;
		qp->q.flags |= INET_FRAG_LAST_IN;
		qp->q.len = end;
	} else {
		if (end&7) {
			end &= ~7;
			if (skb->ip_summed != CHECKSUM_UNNECESSARY)
				skb->ip_summed = CHECKSUM_NONE;
		}
		if (end > qp->q.len) {
			if (qp->q.flags & INET_FRAG_LAST_IN)
				goto discard_qp;
			qp->q.len = end;
		}
	}
	if (end == offset)
		goto discard_qp;

	if (!pskb_pull(skb, skb_network_offset(skb) + ihl))
		goto discard_qp;

	if (pskb_trim_rcsum(skb, end - offset))
		goto discard_qp;

	dev = skb->dev;
	barrier();

	prev_tail = qp->q.fragments_tail;
	err = inet_frag_queue_insert(&qp->q, skb, offset, end);
	if (err)
		goto insert_error;

	if (dev)
		qp->iif = dev->ifindex;

	qp->q.stamp = skb->tstamp;
	qp->q.tstamp_type = skb->tstamp_type;
	qp->q.meat += skb->len;
	qp->ecn |= ecn;
	add_frag_mem_limit(qp->q.fqdir, skb->truesize);
	if (offset == 0)
		qp->q.flags |= INET_FRAG_FIRST_IN;

	fragsize = skb->len + ihl;

	if (fragsize > qp->q.max_size)
		qp->q.max_size = fragsize;

	if (ip_hdr(skb)->frag_off & htons(IP_DF) &&
	    fragsize > qp->max_df_size)
		qp->max_df_size = fragsize;

	if (qp->q.flags == (INET_FRAG_FIRST_IN | INET_FRAG_LAST_IN) &&
	    qp->q.meat == qp->q.len) {
		unsigned long orefdst = skb->_skb_refdst;

		skb->_skb_refdst = 0UL;
		err = ip_frag_reasm(qp, skb, prev_tail, dev, refs);
		skb->_skb_refdst = orefdst;
		if (err)
			inet_frag_kill(&qp->q, refs);
		return err;
	}

	skb_dst_drop(skb);
	skb_orphan(skb);
	return -EINPROGRESS;

/* ... error handling ... */
}
```

重组逻辑：
- 用 `(saddr, daddr, protocol, id, user, vif)` 作为重组队列键。
- 把每个分片按 `offset` 插入红黑树，检测重叠/冲突。
- `MF=0` 的分片标志 `INET_FRAG_LAST_IN` 并记录总长度 `qp->q.len`。
- 当 `FIRST_IN | LAST_IN` 都收到且 `meat == len` 时，调用 `ip_frag_reasm()` 拼回完整报文。
- 重组超时（默认 30 秒）触发 `ip_expire()`。只有本机交付等特定场景会回送 `ICMP_TIME_EXCEEDED / ICMP_EXC_FRAGTIME`；转发路径通常静默丢弃：

```c
static void ip_expire(struct timer_list *t)
{
    /* ... */
    if (frag_expire_skip_icmp(qp->q.key.v4.user) &&
        (skb_rtable(head)->rt_type != RTN_LOCAL))
        goto out;

    icmp_send(head, ICMP_TIME_EXCEEDED, ICMP_EXC_FRAGTIME, 0);
    /* ... */
}
```

### 7.5 分片/重组的状态机

```text
发送端：
  IP 数据报 > MTU
      │
      ├─ DF=1 且不能忽略 DF ──→ 丢弃 + ICMP_FRAG_NEEDED（PMTU 发现）
      └─ DF=0 或允许忽略 DF ──→ ip_do_fragment()
                    ├─ Fast path：利用 frag_list 直接生成多个分片 skb
                    └─ Slow path：逐个 alloc_skb 拷贝数据 + 填 IP 头
                          │
                          ▼
                    每片：tot_len = 本片总长
                          frag_off = offset/8 (+ MF 或 DF)
                          id = 原报文 id
                          check = 新校验和
                          → ip_finish_output2 → 邻居子系统

接收端：
  收到分片
      │
      ▼
  ip_defrag() → ip_find() 找/建 ipq
      │
      ▼
  ip_frag_queue() 按 offset 插入
      │
      ├─ 还缺分片 → -EINPROGRESS（skb 被接管）
      └─ 全部到齐 → ip_frag_reasm() 拼回完整 skb
                          │
                          ▼
                    继续 ip_local_deliver() → 上层协议
```

---

## 8. 路由查找基础

### 8.1 路由在 IP 层的作用

IP 层几乎所有关键决策都依赖路由：
- 收包：判断目的 IP 是本机、广播、组播还是需转发。
- 发包：决定出接口、下一跳网关、MTU、TTL、TOS 映射优先级。
- 转发：决定下一跳和是否发送 ICMP Redirect。

### 8.2 核心结构

`struct rtable`（定义在 `include/net/route.h`）是 IPv4 路由查询结果常用的 `dst_entry` 封装：

```c
struct rtable {
	struct dst_entry	dst;

	int			rt_genid;
	unsigned int		rt_flags;
	__u16			rt_type;
	__u8			rt_is_input;
	__u8			rt_uses_gateway;

	int			rt_iif;

	u8			rt_gw_family;

	/* Info on neighbour */
	union {
		__be32		rt_gw4;
		struct in6_addr	rt_gw6;
	};

	/* Miscellaneous cached information */
	u32			rt_mtu_locked:1,
				rt_pmtu:31;
};
```

关键字段：
- `rt_type`：`RTN_LOCAL`、`RTN_UNICAST`、`RTN_BROADCAST`、`RTN_MULTICAST`、`RTN_BLACKHOLE` 等。
- `rt_gw_family` / `rt_gw4` / `rt_gw6`：路由使用的网关地址；直连网段通常没有网关，取下一跳时会回退到目的地址。
- `dst.dev`：出接口网卡。
- `rt_flags`：`RTCF_LOCAL`、`RTCF_GATEWAY`、`RTCF_BROADCAST`、`RTCF_MULTICAST` 等。

### 8.3 接收路径路由查找

```c
ip_route_input_noref(skb, iph->daddr, iph->saddr, ip4h_dscp(iph), dev);
```

输入路由根据目的地址、源地址、TOS、入接口查找，结果写入 `skb_dst(skb)`。

### 8.4 发送路径路由查找

```c
ip_route_output_flow(net, fl4, sk);
```

`fl4` 是 `struct flowi4`，包含 saddr、daddr、tos、oif、mark 等流信息。输出路由决定从哪个接口发、下一跳是谁。

### 8.5 FIB 与路由缓存

Linux IPv4 路由子系统主要分两层：
- FIB（Forwarding Information Base）：由 `ip route` 配置的真实路由表，通过 `fib_lookup()` 查询。
- `dst` / `rtable`：根据 FIB 查询结果构造出的转发路径对象，会挂到 skb、socket dst cache 或 nexthop exception 等位置被复用。不要把它理解成旧式全局 route cache。

---

## 9. 重要 sysctl 参数

| 参数 | 文件 | 含义 |
|------|------|------|
| `ip_forward` | `/proc/sys/net/ipv4/ip_forward` | 是否开启 IPv4 转发 |
| `ip_no_pmtu_disc` | `/proc/sys/net/ipv4/ip_no_pmtu_disc` | 是否禁用 Path MTU Discovery（默认 0，即启用） |
| `ip_default_ttl` | `/proc/sys/net/ipv4/ip_default_ttl` | 默认 TTL（64） |
| `ip_early_demux` | `/proc/sys/net/ipv4/ip_early_demux` | 是否启用 TCP/UDP 早期解复用 |
| `tcp_early_demux` | `/proc/sys/net/ipv4/tcp_early_demux` | TCP 早期解复用 |
| `udp_early_demux` | `/proc/sys/net/ipv4/udp_early_demux` | UDP 早期解复用 |
| `conf/*/rp_filter` | `/proc/sys/net/ipv4/conf/*/rp_filter` | 反向路径过滤 |
| `conf/*/accept_source_route` | `/proc/sys/net/ipv4/conf/*/accept_source_route` | 是否接受源路由选项 |
| `conf/*/send_redirects` | `/proc/sys/net/ipv4/conf/*/send_redirects` | 是否发送 ICMP Redirect |
| `conf/*/log_martians` | `/proc/sys/net/ipv4/conf/*/log_martians` | 是否记录异常报文 |
| `ipfrag_high_thresh` | `/proc/sys/net/ipv4/ipfrag_high_thresh` | 分片重组内存上限 |
| `ipfrag_low_thresh` | `/proc/sys/net/ipv4/ipfrag_low_thresh` | 分片重组内存下限 |
| `ipfrag_time` | `/proc/sys/net/ipv4/ipfrag_time` | 分片重组超时时间（默认 30 秒） |
| `ipfrag_max_dist` | `/proc/sys/net/ipv4/ipfrag_max_dist` | 防止分片攻击的最大 ID 距离 |

---

## 10. 完整收发时序图

### 10.1 本机接收一个 TCP 报文

```text
时间轴 ──────────────────────────────────────────────>

网卡驱动    收到以太网帧
  │
  ▼
__netif_receive_skb_core()
  │
  ▼
ip_rcv()
  │
  ▼
ip_rcv_core()
  ├─ 检查 version/ihl
  ├─ 校验 IP 头部 checksum
  ├─ 按 tot_len 截断 skb
  └─ 设置 transport_header
  │
  ▼
ip_rcv_finish() → ip_rcv_finish_core()
  ├─ early_demux（可选）
  ├─ ip_route_input_noref()   决定是本机/转发/广播/组播
  ├─ 处理 IP options（如果有）
  │
  ▼
dst_input()
  │
  ▼
ip_local_deliver()
  ├─ 如果是分片 → ip_defrag() 等待重组
  ├─ NF_INET_LOCAL_IN
  │
  ▼
ip_local_deliver_finish()
  ├─ __skb_pull() 去掉 IP 头
  │
  ▼
ip_protocol_deliver_rcu()
  ├─ raw_local_deliver()
  └─ inet_protos[IPPROTO_TCP] → tcp_v4_rcv()
```

### 10.2 本机发送一个 TCP 报文

```text
时间轴 ──────────────────────────────────────────────>

tcp_write_xmit()
  │
  ▼
tcp_transmit_skb()
  │
  ▼
ip_queue_xmit() → __ip_queue_xmit()
  ├─ 查路由（socket dst 缓存 或 ip_route_output_flow）
  ├─ skb_push() / skb_reset_network_header()
  ├─ 填充 IP 头：version/ihl/tos/ttl/protocol/saddr/daddr/id/frag_off
  ├─ 处理 IP options（如果有）
  │
  ▼
ip_local_out() → __ip_local_out()
  ├─ iph_set_totlen()
  ├─ ip_send_check()
  ├─ NF_INET_LOCAL_OUT
  │
  ▼
dst_output() → ip_output()
  ├─ NF_INET_POST_ROUTING
  │
  ▼
ip_finish_output() → __ip_finish_output()
  ├─ GSO 处理
  ├─ 如果 skb->len > mtu → ip_fragment() → ip_do_fragment()
  │
  ▼
ip_finish_output2()
  ├─ ip_neigh_for_gw() 找到下一跳 neighbour
  │
  ▼
neigh_output() → 邻居子系统
  ├─ 需要 ARP → neigh_resolve_output() → ARP Request
  └─ MAC 已缓存 → neigh_connected_output()
       │
       ▼
  dev_hard_header() 填以太网头
       │
       ▼
  dev_queue_xmit()
```

### 10.3 IP 转发一个报文

```text
ip_rcv() → ip_rcv_finish() → dst_input()
  │
  ▼
ip_forward()
  ├─ 检查 TTL
  ├─ xfrm4_route_forward() 重新路由到出接口
  ├─ 检查 MTU / DF
  ├─ ip_decrease_ttl()        TTL 减 1，更新 checksum
  ├─ 可选 ip_rt_send_redirect()
  │
  ▼
NF_INET_FORWARD
  │
  ▼
ip_forward_finish() → dst_output()
  │
  ▼
ip_output() → ip_finish_output() → ip_finish_output2()
  │
  ▼
邻居子系统 / ARP / dev_queue_xmit()
```

---

## 11. 与 ARP 协议的衔接点

学完 ARP 后，理解 IP 层最关键的一个衔接就是：IP 层不直接处理 MAC，它只处理到 “下一跳 IP” 这一层。

```text
IP 层视角：
  源 IP ──────→ 目的 IP
       路由后：源 IP ──→ 下一跳网关 IP（或直连目的 IP）

ARP/邻居子系统视角：
  下一跳 IP ───→ MAC

合并后：
  IP 头：saddr/daddr（端到端）
  以太网头：src MAC / dst MAC（逐跳变化）
```

ARP 文档里讲的 `neigh_resolve_output()`、`arp_solicit()`、`arp_process()`、`NUD` 状态机，正是把 IP 层算出的“下一跳 IP”最终转换成以太网帧的“目的 MAC”的过程。

---

## 12. 小结

| 主题 | 核心函数/结构 | 关键理解 |
|------|--------------|---------|
| IPv4 头 | `struct iphdr` | version/ihl/ttl/protocol/saddr/daddr/frag_off |
| 接收校验 | `ip_rcv_core()` | version=4、ihl>=5、checksum、tot_len 合法 |
| 接收路由 | `ip_rcv_finish_core()` | `ip_route_input_noref()` 决定报文去向 |
| 本机交付 | `ip_local_deliver()` | 先重组，再经 LOCAL_IN，最后 `ip_protocol_deliver_rcu()` |
| 发送构造 | `__ip_queue_xmit()` | 查路由、填 IP 头、选 TTL/DF/ID |
| 发送输出 | `ip_local_out()` → `ip_output()` | 校验和、Local Out、Post Routing、邻居子系统 |
| 转发 | `ip_forward()` | TTL 减 1、MTU/DF 检查、可能发 ICMP |
| 分片 | `ip_fragment()` / `ip_do_fragment()` | 通常 DF=0 才分片，`skb->ignore_df` 等内部场景可例外，偏移 8 字节对齐 |
| 重组 | `ip_defrag()` / `ip_frag_queue()` | Linux 按 `(saddr,daddr,protocol,id,user,vif)` 排队，超时 30s |
| 路由 | `struct rtable` / FIB | 决定本机/转发/广播/组播、下一跳、出接口 |
