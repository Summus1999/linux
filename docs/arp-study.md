# ARP 协议详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/arp.c` —— ARP 协议主实现
> - `net/core/neighbour.c` —— 通用邻居子系统
> - `include/net/neighbour.h` —— 邻居结构/状态定义
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图和时序图使用 `text` 代码块表示。

---

## 1. ARP 协议基础

**作用**：把网络层地址（IPv4）解析成链路层地址（MAC），只作用于同一二层网段。

**核心 RFC**：RFC 826。

**典型交互**：

```text
主机 A (192.148.1.2) 想发给 192.148.1.3
→ A 不知道 1.3 的 MAC
→ A 发 ARP Request（广播：谁有 192.148.1.3？）
→ 1.3 回 ARP Reply（单播：我的 MAC 是 xx:xx:xx:xx:xx:xx）
→ A 缓存到 ARP 表
```

ARP 报文格式本身**不保证可靠性**，也没有 ARP 层校验和（依赖链路层校验，如以太网 FCS）。但 Linux 的 neighbour 子系统会按 `retrans_time_ms`、`mcast_solicit`、`ucast_solicit` 等参数重复发送 ARP Request；因此不能把 Linux 实现概括为“ARP 无重传”。

---

## 2. ARP 报文格式

```c
struct arphdr {
	__be14		ar_hrd;		/* format of hardware address	*/
	__be14		ar_pro;		/* format of protocol address	*/
	unsigned char	ar_hln;		/* length of hardware address	*/
	unsigned char	ar_pln;		/* length of protocol address	*/
	__be14		ar_op;		/* ARP opcode (command)		*/

#if 0
	 /*
	  *	 Ethernet looks like this : This bit is variable sized however...
	  */
	unsigned char		ar_sha[ETH_ALEN];	/* sender hardware address	*/
	unsigned char		ar_sip[4];		/* sender IP address		*/
	unsigned char		ar_tha[ETH_ALEN];	/* target hardware address	*/
	unsigned char		ar_tip[4];		/* target IP address		*/
#endif

};
```

以太网上的完整 ARP 帧布局：

```text
| 以太网头 (12B) | arphdr (8B) | Sender MAC/SHA (6B) | Sender IP/SIP (4B) | Target MAC/THA (6B) | Target IP/TIP (4B) |
```

注意这里有两个“目标 MAC”概念：
- **以太网头的目的 MAC**：ARP Request 通常是广播 `ff:ff:ff:ff:ff:ff`。
- **ARP payload 的 target hardware address（THA）**：以太网上的 ARP Request 通常填 `00:00:00:00:00:00`，因为还不知道目标主机的 MAC。

---

## 3. Linux 内核 ARP 实现总览

| 项目 | 文件/结构 |
|------|-----------|
| ARP 协议主文件 | `net/ipv4/arp.c` |
| 通用邻居子系统 | `net/core/neighbour.c` |
| ARP 表 | `struct neigh_table arp_tbl` |
| 邻居条目 | `struct neighbour` |
| ARP 操作集 | `struct neigh_ops` |
| 收包入口 | `arp_rcv()` |
| 发包入口 | `arp_send() → arp_create() → arp_xmit()` |

**与鸿蒙最大的不同**：Linux 把 ARP 缓存、状态机、定时器、队列全部抽象到 **neighbour 子系统**，ARP 只是其中 IPv4 的一个实例。鸿蒙可能是 ARP 模块自己维护缓存和状态机，Linux 则是通过 `neigh_table` + `neighbour` 统一管理。

---

## 4. 邻居子系统（neighbour）—— 理解 ARP 的关键

### 4.1 核心数据结构

```c
struct neigh_table arp_tbl = {
	.family		= AF_INET,
	.key_len	= 4,
	.protocol	= cpu_to_be14(ETH_P_IP),
	.hash		= arp_hash,
	.key_eq		= arp_key_eq,
	.constructor	= arp_constructor,
	.proxy_redo	= parp_redo,
	.is_multicast	= arp_is_multicast,
	.id		= "arp_cache",
	.parms		= {
		.tbl			= &arp_tbl,
		.reachable_time		= 30 * HZ,
		.data	= {
			[NEIGH_VAR_MCAST_PROBES] = 3,
			[NEIGH_VAR_UCAST_PROBES] = 3,
			[NEIGH_VAR_RETRANS_TIME] = 1 * HZ,
			[NEIGH_VAR_BASE_REACHABLE_TIME] = 30 * HZ,
			[NEIGH_VAR_DELAY_PROBE_TIME] = 5 * HZ,
			[NEIGH_VAR_INTERVAL_PROBE_TIME_MS] = 5 * HZ,
			[NEIGH_VAR_GC_STALETIME] = 60 * HZ,
			[NEIGH_VAR_QUEUE_LEN_BYTES] = SK_WMEM_DEFAULT,
			[NEIGH_VAR_PROXY_QLEN] = 64,
			[NEIGH_VAR_ANYCAST_DELAY] = 1 * HZ,
			[NEIGH_VAR_PROXY_DELAY]	= (8 * HZ) / 10,
			[NEIGH_VAR_LOCKTIME] = 1 * HZ,
		},
	},
	.gc_interval	= 30 * HZ,
	.gc_thresh1	= 128,
	.gc_thresh2	= 512,
	.gc_thresh3	= 1024,
};
```

`arp_tbl` 就是 Linux 的 ARP 表容器，所有 IPv4 邻居条目都挂在这里。

### 4.2 邻居操作集

```c
static const struct neigh_ops arp_generic_ops = {
	.family =		AF_INET,
	.solicit =		arp_solicit,
	.error_report =		arp_error_report,
	.output =		neigh_resolve_output,
	.connected_output =	neigh_connected_output,
};

static const struct neigh_ops arp_hh_ops = {
	.family =		AF_INET,
	.solicit =		arp_solicit,
	.error_report =		arp_error_report,
	.output =		neigh_resolve_output,
	.connected_output =	neigh_resolve_output,
};

static const struct neigh_ops arp_direct_ops = {
	.family =		AF_INET,
	.output =		neigh_direct_output,
	.connected_output =	neigh_direct_output,
};
```

### 4.3 NUD 状态机

```c
#define NUD_INCOMPLETE	0x01
#define NUD_REACHABLE	0x02
#define NUD_STALE	0x04
#define NUD_DELAY	0x08
#define NUD_PROBE	0x10
#define NUD_FAILED	0x20

/* Dummy states */
#define NUD_NOARP	0x40
#define NUD_PERMANENT	0x80
#define NUD_NONE	0x00
```

```c
#define NUD_IN_TIMER	(NUD_INCOMPLETE|NUD_REACHABLE|NUD_DELAY|NUD_PROBE)
#define NUD_VALID	(NUD_PERMANENT|NUD_NOARP|NUD_REACHABLE|NUD_PROBE|NUD_STALE|NUD_DELAY)
#define NUD_CONNECTED	(NUD_PERMANENT|NUD_NOARP|NUD_REACHABLE)
```

状态转换（按源码实际走向）：

```text
NUD_NONE / NUD_FAILED
    └─ IP 包触发 __neigh_event_send()
       → NUD_INCOMPLETE
           ├─ 收到 ARP Reply → neigh_update(..., NUD_REACHABLE, ...)
           │                  → NUD_REACHABLE
           └─ probes >= neigh_max_probes()
                              → NUD_FAILED

NUD_REACHABLE
    └─ reachable_time 到期
       ├─ 最近仍在使用 → NUD_DELAY
       └─ 久未使用     → NUD_STALE

NUD_STALE
    └─ 新的 IP 包触发 __neigh_event_send()
       → NUD_DELAY

NUD_DELAY
    └─ DELAY_PROBE_TIME 到期
       ├─ confirmed 被上层更新 → NUD_REACHABLE
       └─ 没有确认             → NUD_PROBE

NUD_PROBE
    ├─ 收到 ARP Reply → NUD_REACHABLE
    └─ probes >= neigh_max_probes()
       ├─ NTF_EXT_VALIDATED → NUD_STALE
       └─ 其他情况          → NUD_FAILED
```

**注意**：`NUD_NONE` 不会直接收到 ARP Reply 变成 `REACHABLE`。NONE 状态没有已知 MAC，必须先进入 `INCOMPLETE` 发 Request。

---

## 5. ARP 发送流程

### 5.1 主动发送 ARP 请求

上层 IP 要发报文时，调用路由 + 邻居子系统：

```text
ip_queue_xmit()
  → ip_route_output_ports()
  → skb_dst_set()
  → dst_neigh_output()           // 通过 dst_entry 找到 neighbour
    → neigh->output(skb)
      → neigh_resolve_output()   // 如果 MAC 还没解析
        → neigh_event_send(neigh, skb)   // 把 skb 入队，启动 solicit
          → arp_solicit(neigh, skb)      // 真正发 ARP Request
            → arp_send_dst(ARPOP_REQUEST, ...)
              → arp_create()     // 构造 skb
              → arp_xmit()       // 走 netfilter + dev_queue_xmit()
```

### 5.2 arp_solicit 源码要点

```c
static void arp_solicit(struct neighbour *neigh, struct sk_buff *skb)
{
	__be32 saddr = 0;
	u8 dst_ha[MAX_ADDR_LEN], *dst_hw = NULL;
	struct net_device *dev = neigh->dev;
	__be32 target = *(__be32 *)neigh->primary_key;
	int probes = atomic_read(&neigh->probes);
	struct in_device *in_dev;
	struct dst_entry *dst = NULL;

	rcu_read_lock();
	in_dev = __in_dev_get_rcu(dev);
	if (!in_dev) {
		rcu_read_unlock();
		return;
	}
	switch (IN_DEV_ARP_ANNOUNCE(in_dev)) {
	default:
	case 0:		/* By default announce any local IP */
		if (skb && inet_addr_type_dev_table(dev_net(dev), dev,
					  ip_hdr(skb)->saddr) == RTN_LOCAL)
			saddr = ip_hdr(skb)->saddr;
		break;
	case 1:		/* Restrict announcements of saddr in same subnet */
		if (!skb)
			break;
		saddr = ip_hdr(skb)->saddr;
		if (inet_addr_type_dev_table(dev_net(dev), dev,
					     saddr) == RTN_LOCAL) {
			/* saddr should be known to target */
			if (inet_addr_onlink(in_dev, target, saddr))
				break;
		}
		saddr = 0;
		break;
	case 2:		/* Avoid secondary IPs, get a primary/preferred one */
		break;
	}
	rcu_read_unlock();

	if (!saddr)
		saddr = inet_select_addr(dev, target, RT_SCOPE_LINK);

	probes -= NEIGH_VAR(neigh->parms, UCAST_PROBES);
	if (probes < 0) {
		if (!(READ_ONCE(neigh->nud_state) & NUD_VALID))
			pr_debug("trying to ucast probe in NUD_INVALID\n");
		neigh_ha_snapshot(dst_ha, neigh, dev);
		dst_hw = dst_ha;
	} else {
		probes -= NEIGH_VAR(neigh->parms, APP_PROBES);
		if (probes < 0) {
			neigh_app_ns(neigh);
			return;
		}
	}

	if (skb && !(dev->priv_flags & IFF_XMIT_DST_RELEASE))
		dst = skb_dst(skb);
	arp_send_dst(ARPOP_REQUEST, ETH_P_ARP, target, dev, saddr,
		     dst_hw, dev->dev_addr, NULL, dst);
}
```

ARP probe分为以下三种情况：
- PROBE 阶段、有旧 MAC → 内核单播 ARP 确认过期条目
- INCOMPLETE 首次解析、无 MAC → 内核广播 ARP
- 配置了 app_solicit → 内核通过 netlink 委托用户态，自己暂不发 ARP；用户态进程负责解析/更新，内核仍管 neighbour 表

**关键参数**：
- `arp_announce`：选哪个本机 IP 作为 ARP 请求的 sender IP
- `arp_ignore`：收到 ARP 请求时是否响应
- `mcast_probes / ucast_probes / app_probes`：不同阶段的探测次数上限

### 5.3 arp_create 构造报文

`arp_create`分配一个 sk_buff，把二层头 + ARP 头 + ARP 载荷填好，返回可直接交给 arp_xmit() 发送的 skb

```c
struct sk_buff *arp_create(int type, int ptype, __be32 dest_ip,
			   struct net_device *dev, __be32 src_ip,
			   const unsigned char *dest_hw,
			   const unsigned char *src_hw,
			   const unsigned char *target_hw)
{
	struct sk_buff *skb;
	struct arphdr *arp;
	unsigned char *arp_ptr;
	int hlen = LL_RESERVED_SPACE(dev);
	int tlen = dev->needed_tailroom;

	/*
	 *	Allocate a buffer
	 */

	skb = alloc_skb(arp_hdr_len(dev) + hlen + tlen, GFP_ATOMIC);
	if (!skb)
		return NULL;

	skb_reserve(skb, hlen);
	skb_reset_network_header(skb);
	skb_put(skb, arp_hdr_len(dev));
	skb->dev = dev;
	skb->protocol = htons(ETH_P_ARP);
	if (!src_hw)
		src_hw = dev->dev_addr;
	if (!dest_hw)
		dest_hw = dev->broadcast;

	/* Fill the device header for the ARP frame.
	 * Note: skb->head can be changed.
	 */
	if (dev_hard_header(skb, dev, ptype, dest_hw, src_hw, skb->len) < 0)
		goto out;

	arp = arp_hdr(skb);
	/*
	 * Fill out the arp protocol part.
	 *
	 * The arp hardware type should match the device type, except for FDDI,
	 * which (according to RFC 1390) should always equal 1 (Ethernet).
	 */
	/*
	 *	Exceptions everywhere. AX.25 uses the AX.25 PID value not the
	 *	DIX code for the protocol. Make these device structure fields.
	 */
	switch (dev->type) {
	default:
		arp->ar_hrd = htons(dev->type);
		arp->ar_pro = htons(ETH_P_IP);
		break;

#if IS_ENABLED(CONFIG_AX25)
	case ARPHRD_AX25:
		arp->ar_hrd = htons(ARPHRD_AX25);
		arp->ar_pro = htons(AX25_P_IP);
		break;

#if IS_ENABLED(CONFIG_NETROM)
	case ARPHRD_NETROM:
		arp->ar_hrd = htons(ARPHRD_NETROM);
		arp->ar_pro = htons(AX25_P_IP);
		break;
#endif
#endif

#if IS_ENABLED(CONFIG_FDDI)
	case ARPHRD_FDDI:
		arp->ar_hrd = htons(ARPHRD_ETHER);
		arp->ar_pro = htons(ETH_P_IP);
		break;
#endif
	}

	arp->ar_hln = dev->addr_len;
	arp->ar_pln = 4;
	arp->ar_op = htons(type);

	arp_ptr = (unsigned char *)(arp + 1);

	memcpy(arp_ptr, src_hw, dev->addr_len);
	arp_ptr += dev->addr_len;
	memcpy(arp_ptr, &src_ip, 4);
	arp_ptr += 4;

	switch (dev->type) {
#if IS_ENABLED(CONFIG_FIREWIRE_NET)
	case ARPHRD_IEEE1394:
		break;
#endif
	default:
		if (target_hw)
			memcpy(arp_ptr, target_hw, dev->addr_len);
		else
			memset(arp_ptr, 0, dev->addr_len);
		arp_ptr += dev->addr_len;
	}
	memcpy(arp_ptr, &dest_ip, 4);

	return skb;

out:
	kfree_skb(skb);
	return NULL;
}
```
ARP报文封装流程图：
```text
arp_create()
    │
    ├─ alloc_skb()           分配缓冲区
    ├─ skb_reserve(hlen)     给二层头留空
    ├─ skb_put(arp_len)      划出 ARP 区
    │
    ├─ dev_hard_header()     填以太网头 (dst/src MAC + 0x0806)
    │
    ├─ 填 arphdr             ar_hrd/ar_pro/ar_hln/ar_pln/ar_op
    │
    ├─ 填 SHA + SIP          发送方 MAC/IP
    ├─ 填 THA + TIP         目标 MAC/IP
    │
    └─ return skb            交给 arp_xmit() 发送
```

### 5.4 arp_xmit 发送

```c
static int arp_xmit_finish(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	return dev_queue_xmit(skb);
}

/*
 *	Send an arp packet.
 */
void arp_xmit(struct sk_buff *skb)
{
	rcu_read_lock();
	/* Send it off, maybe filter it using firewalling first.  */
	NF_HOOK(NFPROTO_ARP, NF_ARP_OUT,
		dev_net_rcu(skb->dev), NULL, skb, NULL, skb->dev,
		arp_xmit_finish);
	rcu_read_unlock();
}
```

ARP 报文也会走 **Netfilter ARP 链**（`NF_ARP_OUT` / `NF_ARP_IN`），这是做 ARP 防火墙的地方。

---

## 6. ARP 接收流程

```text
网卡驱动收到帧
  → __netif_receive_skb_core()
    → 根据 ethertype = 0x0806 找到 packet_type
    → arp_rcv(skb, dev, pt, orig_dev)
      → 基础校验（NOARP、otherhost、loopback、长度）
      → NF_HOOK(NFPROTO_ARP, NF_ARP_IN, ..., arp_process)
        → arp_process(skb)
```

### 6.1 arp_rcv 入口

```c
static int arp_rcv(struct sk_buff *skb, struct net_device *dev,
		   struct packet_type *pt, struct net_device *orig_dev)
{
	enum skb_drop_reason drop_reason;
	const struct arphdr *arp;

	/* do not tweak dropwatch on an ARP we will ignore */
	if (dev->flags & IFF_NOARP ||
	    skb->pkt_type == PACKET_OTHERHOST ||
	    skb->pkt_type == PACKET_LOOPBACK)
		goto consumeskb;

	skb = skb_share_check(skb, GFP_ATOMIC);
	if (!skb)
		goto out_of_mem;

	/* ARP header, plus 2 device addresses, plus 2 IP addresses.  */
	drop_reason = pskb_may_pull_reason(skb, arp_hdr_len(dev));
	if (drop_reason != SKB_NOT_DROPPED_YET)
		goto freeskb;

	arp = arp_hdr(skb);
	if (arp->ar_hln != dev->addr_len || arp->ar_pln != 4) {
		drop_reason = SKB_DROP_REASON_NOT_SPECIFIED;
		goto freeskb;
	}

	memset(NEIGH_CB(skb), 0, sizeof(struct neighbour_cb));

	return NF_HOOK(NFPROTO_ARP, NF_ARP_IN,
		       dev_net(dev), NULL, skb, dev, NULL,
		       arp_process);

consumeskb:
	consume_skb(skb);
	return NET_RX_SUCCESS;
freeskb:
	kfree_skb_reason(skb, drop_reason);
out_of_mem:
	return NET_RX_DROP;
}
```

### 6.2 arp_process 核心逻辑

这是 ARP 包处理的“大脑”。

#### 6.2.1 报文合法性校验

```c
static int arp_process(struct net *net, struct sock *sk, struct sk_buff *skb)
{
	struct net_device *dev = skb->dev;
	struct in_device *in_dev = __in_dev_get_rcu(dev);
	struct arphdr *arp;
	unsigned char *arp_ptr;
	struct rtable *rt;
	unsigned char *sha;
	unsigned char *tha = NULL;
	__be32 sip, tip;
	u14 dev_type = dev->type;
	int addr_type;
	struct neighbour *n;
	struct dst_entry *reply_dst = NULL;
	bool is_garp = false;

	/* arp_rcv below verifies the ARP header and verifies the device
	 * is ARP'able.
	 */

	if (!in_dev)
		goto out_free_skb;

	arp = arp_hdr(skb);

	switch (dev_type) {
	default:
		if (arp->ar_pro != htons(ETH_P_IP) ||
		    htons(dev_type) != arp->ar_hrd)
			goto out_free_skb;
		break;
	case ARPHRD_ETHER:
	case ARPHRD_FDDI:
	case ARPHRD_IEEE802:
		/*
		 * ETHERNET, and Fibre Channel (which are IEEE 802
		 * devices, according to RFC 2625) devices will accept ARP
		 * hardware types of either 1 (Ethernet) or 6 (IEEE 802.2).
		 * This is the case also of FDDI, where the RFC 1390 says that
		 * FDDI devices should accept ARP hardware of (1) Ethernet,
		 * however, to be more robust, we'll accept both 1 (Ethernet)
		 * or 6 (IEEE 802.2)
		 */
		if ((arp->ar_hrd != htons(ARPHRD_ETHER) &&
		     arp->ar_hrd != htons(ARPHRD_IEEE802)) ||
		    arp->ar_pro != htons(ETH_P_IP))
			goto out_free_skb;
		break;
	case ARPHRD_AX25:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_AX25))
			goto out_free_skb;
		break;
	case ARPHRD_NETROM:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_NETROM))
			goto out_free_skb;
		break;
}
if (arp->ar_op != htons(ARPOP_REPLY) &&
	    arp->ar_op != htons(ARPOP_REQUEST))
		goto out_free_skb;
```

#### 6.2.2 提取字段

```c
	arp_ptr = (unsigned char *)(arp + 1);
	sha	= arp_ptr;
	arp_ptr += dev->addr_len;
	memcpy(&sip, arp_ptr, 4);
	arp_ptr += 4;
	switch (dev_type) {
#if IS_ENABLED(CONFIG_FIREWIRE_NET)
	case ARPHRD_IEEE1394:
		break;
#endif
	default:
		tha = arp_ptr;
		arp_ptr += dev->addr_len;
	}
	memcpy(&tip, arp_ptr, 4);
```

#### 6.2.3 对 ARP Request 的处理

```c
	/* Special case: IPv4 duplicate address detection packet (RFC2131) */
	if (sip == 0) {
		if (arp->ar_op == htons(ARPOP_REQUEST) &&
		    inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL &&
		    !arp_ignore(in_dev, sip, tip))
			arp_send_dst(ARPOP_REPLY, ETH_P_ARP, sip, dev, tip,
				     sha, dev->dev_addr, sha, reply_dst);
		goto out_consume_skb;
	}

	if (arp->ar_op == htons(ARPOP_REQUEST) &&
	    ip_route_input_noref(skb, tip, sip, 0, dev) == 0) {

		rt = skb_rtable(skb);
		addr_type = rt->rt_type;

		if (addr_type == RTN_LOCAL) {
			int dont_send;

			dont_send = arp_ignore(in_dev, sip, tip);
			if (!dont_send && IN_DEV_ARPFILTER(in_dev))
				dont_send = arp_filter(sip, tip, dev);
			if (!dont_send) {
				n = neigh_event_ns(&arp_tbl, sha, &sip, dev);
				if (n) {
					arp_send_dst(ARPOP_REPLY, ETH_P_ARP,
						     sip, dev, tip, sha,
						     dev->dev_addr, sha,
						     reply_dst);
					neigh_release(n);
				}
			}
			goto out_consume_skb;
		} else if (IN_DEV_FORWARD(in_dev)) {
			if (addr_type == RTN_UNICAST  &&
			    (arp_fwd_proxy(in_dev, dev, rt) ||
			     arp_fwd_pvlan(in_dev, dev, rt, sip, tip) ||
			     (rt->dst.dev != dev &&
			      pneigh_lookup(&arp_tbl, net, &tip, dev)))) {
				n = neigh_event_ns(&arp_tbl, sha, &sip, dev);
				if (n)
					neigh_release(n);

				if (NEIGH_CB(skb)->flags & LOCALLY_ENQUEUED ||
				    skb->pkt_type == PACKET_HOST ||
				    NEIGH_VAR(in_dev->arp_parms, PROXY_DELAY) == 0) {
					arp_send_dst(ARPOP_REPLY, ETH_P_ARP,
						     sip, dev, tip, sha,
						     dev->dev_addr, sha,
						     reply_dst);
				} else {
					pneigh_enqueue(&arp_tbl,
						       in_dev->arp_parms, skb);
					goto out_free_dst;
				}
				goto out_consume_skb;
			}
		}
	}
```

重点：
- 收到请求自己的 ARP Request 时，先学习对方（`neigh_event_ns` 创建/更新邻居为 `NUD_STALE`），再回复。
- `arp_ignore` 控制是否回复（常见 `arp_ignore=1` 只在target IP 属于入接口时才回）。
- 这里的 `ip_route_input_noref()` 是接收路径里的路由判定，用于判断 `tip` 是本机地址、可转发地址还是代理 ARP 候选地址。

#### 6.2.4 对 ARP Reply / 免费 ARP 的处理

```c
	/* Update our ARP tables */

	n = __neigh_lookup(&arp_tbl, &sip, dev, 0);

	addr_type = -1;
	if (n || arp_accept(in_dev, sip)) {
		is_garp = arp_is_garp(net, dev, &addr_type, arp->ar_op,
				      sip, tip, sha, tha);
	}

	if (arp_accept(in_dev, sip)) {
		/* Unsolicited ARP is not accepted by default.
		   It is possible, that this option should be enabled for some
		   devices (strip is candidate)
		 */
		if (!n &&
		    (is_garp ||
		     (arp->ar_op == htons(ARPOP_REPLY) &&
		      (addr_type == RTN_UNICAST ||
		       (addr_type < 0 &&
			/* postpone calculation to as late as possible */
			inet_addr_type_dev_table(net, dev, sip) ==
				RTN_UNICAST)))))
			n = __neigh_lookup(&arp_tbl, &sip, dev, 1);
	}

	if (n) {
		int state = NUD_REACHABLE;
		int override;

		/* If several different ARP replies follows back-to-back,
		   use the FIRST one. It is possible, if several proxy
		   agents are active. Taking the first reply prevents
		   arp trashing and chooses the fastest router.
		 */
		override = time_after(jiffies,
				      n->updated +
				      NEIGH_VAR(n->parms, LOCKTIME)) ||
			   is_garp;

		/* Broadcast replies and request packets
		   do not assert neighbour reachability.
		 */
		if (arp->ar_op != htons(ARPOP_REPLY) ||
		    skb->pkt_type != PACKET_HOST)
			state = NUD_STALE;
		neigh_update(n, sha, state,
			     override ? NEIGH_UPDATE_F_OVERRIDE : 0, 0);
		neigh_release(n);
	}
```

- 收到 Reply：把邻居状态改为 `NUD_REACHABLE`。
- `LOCKTIME`：防止 ARP 快速抖动，短时间内不覆盖。
- 广播 Reply / Request：只标 `NUD_STALE`，不标 `REACHABLE`。

### 6.3 免费 ARP（Gratuitous ARP）

判断条件：

```c
static bool arp_is_garp(struct net *net, struct net_device *dev,
			int *addr_type, __be14 ar_op,
			__be32 sip, __be32 tip,
			unsigned char *sha, unsigned char *tha)
{
	bool is_garp = tip == sip;

	/* Gratuitous ARP _replies_ also require target hwaddr to be
	 * the same as source.
	 */
	if (is_garp && ar_op == htons(ARPOP_REPLY))
		is_garp =
			/* IPv4 over IEEE 1394 doesn't provide target
			 * hardware address field in its ARP payload.
			 */
			tha &&
			!memcmp(tha, sha, dev->addr_len);

	if (is_garp) {
		*addr_type = inet_addr_type_dev_table(net, dev, sip);
		if (*addr_type != RTN_UNICAST)
			is_garp = false;
	}
	return is_garp;
}
```

**免费ARP用途**：
1. IP 冲突检测：启动时发 GARP，看是否有人回。
2. 更新别人 ARP 缓存：HA/主备切换时主动刷新 MAC。

Linux 默认 **`arp_accept=0`**：不根据 GARP 创建新邻居，只更新已有条目。设 `arp_accept=1` 才会创建。

---

## 7. arp_constructor：邻居条目创建时初始化

```c
static int arp_constructor(struct neighbour *neigh)
{
	__be32 addr;
	struct net_device *dev = neigh->dev;
	struct in_device *in_dev;
	struct neigh_parms *parms;
	u32 inaddr_any = INADDR_ANY;

	if (dev->flags & (IFF_LOOPBACK | IFF_POINTOPOINT))
		memcpy(neigh->primary_key, &inaddr_any, arp_tbl.key_len);

	addr = *(__be32 *)neigh->primary_key;
	rcu_read_lock();
	in_dev = __in_dev_get_rcu(dev);
	if (!in_dev) {
		rcu_read_unlock();
		return -EINVAL;
	}

	neigh->type = inet_addr_type_dev_table(dev_net(dev), dev, addr);

	parms = in_dev->arp_parms;
	__neigh_parms_put(neigh->parms);
	neigh->parms = neigh_parms_clone(parms);
	rcu_read_unlock();

	if (!dev->header_ops) {
		neigh->nud_state = NUD_NOARP;
		neigh->ops = &arp_direct_ops;
		neigh->output = neigh_direct_output;
	} else {
		/* Good devices (checked by reading texts, but only Ethernet is
		   tested)

		   ARPHRD_ETHER: (ethernet, apfddi)
		   ARPHRD_FDDI: (fddi)
		   ARPHRD_IEEE802: (tr)
		   ARPHRD_METRICOM: (strip)
		   ARPHRD_ARCNET:
		   etc. etc. etc.

		   ARPHRD_IPDDP will also work, if author repairs it.
		   I did not it, because this driver does not work even
		   in old paradigm.
		 */

		if (neigh->type == RTN_MULTICAST) {
			neigh->nud_state = NUD_NOARP;
			arp_mc_map(addr, neigh->ha, dev, 1);
		} else if (dev->flags & (IFF_NOARP | IFF_LOOPBACK)) {
			neigh->nud_state = NUD_NOARP;
			memcpy(neigh->ha, dev->dev_addr, dev->addr_len);
		} else if (neigh->type == RTN_BROADCAST ||
			   (dev->flags & IFF_POINTOPOINT)) {
			neigh->nud_state = NUD_NOARP;
			memcpy(neigh->ha, dev->broadcast, dev->addr_len);
		}

		if (dev->header_ops->cache)
			neigh->ops = &arp_hh_ops;
		else
			neigh->ops = &arp_generic_ops;

		if (neigh->nud_state & NUD_VALID)
			neigh->output = neigh->ops->connected_output;
		else
			neigh->output = neigh->ops->output;
	}
	return 0;
}
```

关键：组播、广播、lo、P2P（IFF_POINTOPOINT 点对点接口）、NOARP 设备不需要发 ARP，直接 `NUD_NOARP`。

---

## 8. 重要 sysctl 参数

| 参数 | 文件 | 含义 |
|------|------|------|
| `arp_ignore` | `/proc/sys/net/ipv4/conf/*/arp_ignore` | 收到 ARP 请求时是否响应 |
| `arp_announce` | `/proc/sys/net/ipv4/conf/*/arp_announce` | ARP 请求源 IP 选择策略 |
| `arp_accept` | `/proc/sys/net/ipv4/conf/*/arp_accept` | 是否接受主动/免费 ARP 创建新条目 |
| `arp_filter` | `/proc/sys/net/ipv4/conf/*/arp_filter` | 是否做反向路由过滤 |
| `proxy_arp` | `/proc/sys/net/ipv4/conf/*/proxy_arp` | 是否开启代理 ARP |
| `proxy_arp_pvlan` | `/proc/sys/net/ipv4/conf/*/proxy_arp_pvlan` | RFC 3069 Private VLAN 代理 |
| `base_reachable_time_ms` | `/proc/sys/net/ipv4/neigh/*/base_reachable_time_ms` | REACHABLE 超时 |
| `retrans_time_ms` | `/proc/sys/net/ipv4/neigh/*/retrans_time_ms` | INCOMPLETE/PROBE 探测重试间隔 |
| `ucast_solicit` | `/proc/sys/net/ipv4/neigh/*/ucast_solicit` | unicast probe 次数 |
| `mcast_solicit` | `/proc/sys/net/ipv4/neigh/*/mcast_solicit` | multicast probe 次数 |

---

## 9. 解析 `arp_process()`

`arp_process()` 是 ARP 报文处理的真正核心，位于 `net/ipv4/arp.c:702`。它由 `arp_rcv()` 经过 Netfilter `NF_ARP_IN` 链后调用。函数签名：这里 `sk` 恒为 `NULL`。

```c
static int arp_process(struct net *net, struct sock *sk, struct sk_buff *skb)
```

### 9.1 局部变量与入口

```c
static int arp_process(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct net_device *dev = skb->dev;
    struct in_device *in_dev = __in_dev_get_rcu(dev);
    struct arphdr *arp;
    unsigned char *arp_ptr;
    struct rtable *rt;
    unsigned char *sha;
    unsigned char *tha = NULL;
    __be32 sip, tip;
    u14 dev_type = dev->type;
    int addr_type;
    struct neighbour *n;
    struct dst_entry *reply_dst = NULL;
    bool is_garp = false;
```

| 变量 | 含义 |
|------|------|
| `dev` | 接收报文的网卡 |
| `in_dev` | 网卡的 IPv4 配置块，包含 `arp_ignore`、`arp_accept` 等参数 |
| `arp` | ARP 报文头指针 |
| `sha/tha` | Sender/Target Hardware Address |
| `sip/tip` | Sender/Target Protocol Address（即 IP） |
| `addr_type` | 路由类型：`RTN_LOCAL`、`RTN_UNICAST`、`RTN_BROADCAST` 等 |
| `n` | 查到的 `struct neighbour` |
| `reply_dst` | 隧道元数据回复用的 dst，普通场景为 NULL |
| `is_garp` | 是否为免费 ARP |

### 9.2 第一阶段：基础合法性校验

```c
    if (!in_dev)
        goto out_free_skb;
```

没有 IPv4 配置的网卡不处理 ARP。

```c
    arp = arp_hdr(skb);

    switch (dev_type) {
    default:
        if (arp->ar_pro != htons(ETH_P_IP) ||
            htons(dev_type) != arp->ar_hrd)
            goto out_free_skb;
        break;
    case ARPHRD_ETHER:
    case ARPHRD_FDDI:
    case ARPHRD_IEEE802:
        if ((arp->ar_hrd != htons(ARPHRD_ETHER) &&
             arp->ar_hrd != htons(ARPHRD_IEEE802)) ||
            arp->ar_pro != htons(ETH_P_IP))
            goto out_free_skb;
        break;
	case ARPHRD_AX25:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_AX25))
			goto out_free_skb;
		break;
	case ARPHRD_NETROM:
		if (arp->ar_pro != htons(AX25_P_IP) ||
		    arp->ar_hrd != htons(ARPHRD_NETROM))
			goto out_free_skb;
		break;
    }
```

**解释**：
- 默认情况下，要求报文为`0x0800`的协议报文且硬件网卡类型匹配
- 以太网/FDDI/IEEE802 做了兼容处理：允许硬件类型为 Ethernet(1) 或 IEEE802(6)。
- AX.25、NET/ROM 等链路层有各自的判断。
- 不匹配的报文直接丢弃。

```c
    if (arp->ar_op != htons(ARPOP_REPLY) &&
        arp->ar_op != htons(ARPOP_REQUEST))
        goto out_free_skb;
```

只处理 Request(1) 和 Reply(2)，其他操作码（如 RARP）丢弃。

### 10.3 第二阶段：字段提取

```c
    arp_ptr = (unsigned char *)(arp + 1);
    sha     = arp_ptr;
    arp_ptr += dev->addr_len;
    memcpy(&sip, arp_ptr, 4);
    arp_ptr += 4;
    switch (dev_type) {
#if IS_ENABLED(CONFIG_FIREWIRE_NET)
    case ARPHRD_IEEE1394:
        break;
#endif
    default:
        tha = arp_ptr;
        arp_ptr += dev->addr_len;
    }
    memcpy(&tip, arp_ptr, 4);
```

**注意 IEEE1394(FireWire)**：这种链路层没有 target hardware address 字段，所以 `tha = NULL`。

### 10.4 第三阶段：目标地址 sanity check

```c
    if (ipv4_is_multicast(tip) ||
        (!IN_DEV_ROUTE_LOCALNET(in_dev) && ipv4_is_loopback(tip)))
        goto out_free_skb;
```

- 目标 IP 是组播：丢弃。
- 目标 IP 是 127.x.x.x 且未开启 `route_localnet`：丢弃。

```c
    if (sip == tip && IN_DEV_ORCONF(in_dev, DROP_GRATUITOUS_ARP))
        goto out_free_skb;
```

- 如果 sip == tip 是免费 ARP，且接口/全局配置 `drop_gratuitous_arp=1`：丢弃。

```c
    if (dev_type == ARPHRD_DLCI)
        sha = dev->broadcast;
```

Frame Relay 特殊处理：源二层地址用广播地址。

### 9.5 第四阶段：处理 ARP Request

#### 9.5.1 DAD（重复地址检测）特殊处理

```c
    if (sip == 0) {
        if (arp->ar_op == htons(ARPOP_REQUEST) &&
            inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL &&
            !arp_ignore(in_dev, sip, tip))
            arp_send_dst(ARPOP_REPLY, ETH_P_ARP, sip, dev, tip,
                         sha, dev->dev_addr, sha, reply_dst);
        goto out_consume_skb;
    }
```

- `sip == 0` 对应 IPv4 Address Conflict Detection / ARP Probe 场景，规范应参考 RFC 5227。源码注释写的是 `RFC2131`，这是 DHCP 客户端冲突检测语境下的历史注释。
- 如果 `tip` 是本机地址，且 `arp_ignore` 允许，则回复。
- 回复的 target IP 是 0，这是合法的。

#### 9.5.2 请求本机地址

```c
    if (arp->ar_op == htons(ARPOP_REQUEST) &&
        ip_route_input_noref(skb, tip, sip, 0, dev) == 0) {

        rt = skb_rtable(skb);
        addr_type = rt->rt_type;

        if (addr_type == RTN_LOCAL) {
            int dont_send;

            dont_send = arp_ignore(in_dev, sip, tip);
            if (!dont_send && IN_DEV_ARPFILTER(in_dev))
                dont_send = arp_filter(sip, tip, dev);
            if (!dont_send) {
                n = neigh_event_ns(&arp_tbl, sha, &sip, dev);
                if (n) {
                    arp_send_dst(ARPOP_REPLY, ETH_P_ARP,
                                 sip, dev, tip, sha,
                                 dev->dev_addr, sha,
                                 reply_dst);
                    neigh_release(n);
                }
            }
            goto out_consume_skb;
        }
```

关键逻辑链：
1. `ip_route_input_noref(skb, tip, sip, 0, dev)`：在接收网卡上查路由，看 `tip` 是不是自己。
2. 如果 `rt_type == RTN_LOCAL`，说明请求的是自己。
3. `arp_ignore(in_dev, sip, tip)`：根据 `arp_ignore` 决定是否响应。
4. `arp_filter(sip, tip, dev)`：如果开启 `arp_filter`，做反向路由检查，防止源 IP 路由从另一个接口出去。
5. `neigh_event_ns(&arp_tbl, sha, &sip, dev)`：
   - 查找/创建 sender 的邻居条目。
   - 用 `sha` 更新其 MAC，状态设为 `NUD_STALE`。
   - 这是“先学习对方，再回复”的优化。
6. `arp_send_dst(...)`：构造并发送 ARP Reply。
7. `neigh_release(n)`：释放 `neigh_event_ns` 获得的引用。

#### 9.5.3 代理 ARP 处理

```c
        } else if (IN_DEV_FORWARD(in_dev)) {
            if (addr_type == RTN_UNICAST  &&
                (arp_fwd_proxy(in_dev, dev, rt) ||
                 arp_fwd_pvlan(in_dev, dev, rt, sip, tip) ||
                 (rt->dst.dev != dev &&
                  pneigh_lookup(&arp_tbl, net, &tip, dev)))) {
                n = neigh_event_ns(&arp_tbl, sha, &sip, dev);
                if (n)
                    neigh_release(n);

                if (NEIGH_CB(skb)->flags & LOCALLY_ENQUEUED ||
                    skb->pkt_type == PACKET_HOST ||
                    NEIGH_VAR(in_dev->arp_parms, PROXY_DELAY) == 0) {
                    arp_send_dst(ARPOP_REPLY, ETH_P_ARP,
                                 sip, dev, tip, sha,
                                 dev->dev_addr, sha,
                                 reply_dst);
                } else {
                    pneigh_enqueue(&arp_tbl,
                                   in_dev->arp_parms, skb);
                    goto out_free_dst;
                }
                goto out_consume_skb;
            }
        }
    }
```

**代理 ARP 触发条件**（需同时满足）：
- 设备开启 IP 转发（`IN_DEV_FORWARD`）。
- 路由类型是 `RTN_UNICAST`。
- 满足以下之一：
  - `arp_fwd_proxy()`：全局/接口 `proxy_arp=1` 且出接口不是本接口。
  - `arp_fwd_pvlan()`：Private VLAN 代理（`proxy_arp_pvlan=1`）。
  - `pneigh_lookup()`：存在代理邻居条目（通过 `ip neigh add proxy` 添加）。

**代理延迟**：如果 `proxy_delay != 0` 且不是本机队列重入、不是单播给自己的报文，则把 skb 入代理队列 `pneigh_enqueue`，延迟后再由 `parp_redo()` 重新处理。这是为了防御 ARP 代理泛洪。

### 9.6 第五阶段：处理 ARP Reply / 更新 ARP 表

```c
    n = __neigh_lookup(&arp_tbl, &sip, dev, 0);
```

用 sender IP 查 ARP 表。第四个参数 `0` 表示查不到不创建。

```c
    addr_type = -1;
    if (n || arp_accept(in_dev, sip)) {
        is_garp = arp_is_garp(net, dev, &addr_type, arp->ar_op,
                              sip, tip, sha, tha);
    }
```

- 如果已有邻居条目，或者 `arp_accept` 允许接受主动 ARP，则判断是否为免费 ARP。
- `arp_is_garp()` 逻辑：
  - `tip == sip` 初步判定为 GARP。
  - 如果是 Reply，还要求 `tha == sha`（RFC 规定）。
  - 最后检查 sender IP 是 unicast，否则不算。

```c
    if (arp_accept(in_dev, sip)) {
        if (!n &&
            (is_garp ||
             (arp->ar_op == htons(ARPOP_REPLY) &&
              (addr_type == RTN_UNICAST ||
               (addr_type < 0 &&
                inet_addr_type_dev_table(net, dev, sip) == RTN_UNICAST)))))
            n = __neigh_lookup(&arp_tbl, &sip, dev, 1);
    }
```

- `arp_accept=0`（默认）：不基于主动 ARP/Reply 创建新邻居。
- `arp_accept=1`：免费 ARP 或单播 Reply 可以创建新邻居。
- `arp_accept=2`：只有 sender IP 和本接口在同一子网时才创建。

```c
    if (n) {
        int state = NUD_REACHABLE;
        int override;

        override = time_after(jiffies,
                              n->updated +
                              NEIGH_VAR(n->parms, LOCKTIME)) ||
                   is_garp;

        if (arp->ar_op != htons(ARPOP_REPLY) ||
            skb->pkt_type != PACKET_HOST)
            state = NUD_STALE;
        neigh_update(n, sha, state,
                     override ? NEIGH_UPDATE_F_OVERRIDE : 0, 0);
        neigh_release(n);
    }
```

更新策略：
- 默认 `state = NUD_REACHABLE`。
- 但如果报文不是单播给自己的 Reply（比如广播 Reply、Request），则只标 `NUD_STALE`。
- `override`：
  - 如果距离上次更新超过 `locktime`，允许覆盖。
  - 如果是 GARP，强制覆盖（用于 HA 切换主动刷新 MAC）。
- 最后 `neigh_update()` 更新 MAC 和状态。

### 9.7 出口路径

```c
out_consume_skb:
	consume_skb(skb);

out_free_dst:
	dst_release(reply_dst);
	return NET_RX_SUCCESS;

out_free_skb:
	kfree_skb(skb);
	return NET_RX_DROP;
```

注意：`out_consume_skb` 后面没有 `return`，会自然 fall through 到 `out_free_dst`，正常消费 skb 后继续释放 `reply_dst` 并返回 `NET_RX_SUCCESS`。代理 ARP 延迟路径直接 `goto out_free_dst`，不走 `out_consume_skb`，因为 skb 已经被 `pneigh_enqueue()` 接管。

---

## 10. 邻居子系统

### 10.1 `struct neighbour` 关键字段

```c
struct neighbour {
	struct hlist_node	hash;
	struct hlist_node	dev_list;
	struct neigh_table	*tbl;
	struct neigh_parms	*parms;
	unsigned long		confirmed;
	unsigned long		updated;
	rwlock_t		lock;
	refcount_t		refcnt;
	unsigned int		arp_queue_len_bytes;
	struct sk_buff_head	arp_queue;
	struct timer_list	timer;
	unsigned long		used;
	atomic_t		probes;
	u8			nud_state;
	u8			type;
	u8			dead;
	u8			protocol;
	u32			flags;
	seqlock_t		ha_lock;
	unsigned char		ha[ALIGN(MAX_ADDR_LEN, sizeof(unsigned long))] __aligned(8);
	struct hh_cache		hh;
	int			(*output)(struct neighbour *, struct sk_buff *);
	const struct neigh_ops	*ops;
	struct list_head	gc_list;
	struct list_head	managed_list;
	struct rcu_head		rcu;
	struct net_device	*dev;
	netdevice_tracker	dev_tracker;
	u8			primary_key[];
} __randomize_layout;
```

| 字段 | 作用 |
|------|------|
| `primary_key` | 邻居键值，ARP 是 IPv4 地址（4 字节） |
| `ha` | 解析到的链路层地址 |
| `nud_state` | NUD 状态 |
| `output` | 当前发包函数指针 |
| `ops` | 协议相关操作集 |
| `arp_queue` | 等待 ARP 解析的 IP 包队列 |
| `timer` | 邻居状态定时器 |
| `probes` | 已发送探测次数 |
| `confirmed` | 上次确认可达时间 |
| `used` | 上次使用时间，用于 GC |

### 10.2 邻居创建流程

```text
上层需要发送 IP 包
  → dst_neigh_output() 或 neigh_event_send()
    → __neigh_lookup(tbl, pkey, dev, 0)   // 查表
      → 未找到
        → __neigh_create(tbl, pkey, dev, true)
          → neigh_alloc()                 // 分配并初始化 neighbour
          → tbl->constructor(n)           // 对 ARP 就是 arp_constructor()
          → 插入哈希表
```

`neigh_alloc()` 初始化关键字段：

```c
n = kzalloc(tbl->entry_size + dev->neigh_priv_len, GFP_ATOMIC);
if (!n)
	goto out_entries;

__skb_queue_head_init(&n->arp_queue);
rwlock_init(&n->lock);
seqlock_init(&n->ha_lock);
n->updated	  = n->used = now;
n->nud_state	  = NUD_NONE;
n->output	  = neigh_blackhole;
n->flags	  = flags;
seqlock_init(&n->hh.hh_lock);
n->parms	  = neigh_parms_clone(&tbl->parms);
timer_setup(&n->timer, neigh_timer_handler, 0);

NEIGH_CACHE_STAT_INC(tbl, allocs);
n->tbl		  = tbl;
refcount_set(&n->refcnt, 1);
n->dead		  = 1;
INIT_LIST_HEAD(&n->gc_list);
INIT_LIST_HEAD(&n->managed_list);
```

`arp_constructor()` 随后根据地址类型和设备标志设置初始状态：
- 组播 → `NUD_NOARP`，MAC 用 `arp_mc_map()` 计算。
- 广播/lo/P2P/NOARP → `NUD_NOARP`。
- 普通单播 → `NUD_NONE`，output 指向 `neigh_resolve_output`。

### 10.3 邻居查找

```c
struct neighbour *__neigh_lookup(struct neigh_table *tbl,
                                 const void *pkey,
                                 struct net_device *dev,
                                 int creat);
```

- `creat=0`：只查找，不创建。
- `creat=1`：找不到时创建（内部调用 `__neigh_create`）。

查找通过 `tbl->hash(pkey, dev, hash_rnd)` 计算哈希桶，再遍历链表比较 `dev` + `primary_key`。

### 10.4 垃圾回收

#### 周期性 GC：`neigh_periodic_work()`

由 delayed work 周期性执行，间隔约为 `BASE_REACHABLE_TIME / 2`。

```c
static void neigh_periodic_work(struct work_struct *work)
{
	struct neigh_table *tbl = container_of(work, struct neigh_table, gc_work.work);
	struct neigh_hash_table *nht;
	struct hlist_node *tmp;
	struct neighbour *n;
	unsigned int i;

	NEIGH_CACHE_STAT_INC(tbl, periodic_gc_runs);

	spin_lock_bh(&tbl->lock);
	nht = rcu_dereference_protected(tbl->nht,
					lockdep_is_held(&tbl->lock));

	/*
	 *	periodically recompute ReachableTime from random function
	 */

	if (time_after(jiffies, tbl->last_rand + 300 * HZ)) {
		struct neigh_parms *p;

		WRITE_ONCE(tbl->last_rand, jiffies);
		list_for_each_entry(p, &tbl->parms_list, list)
			neigh_set_reach_time(p);
	}

	if (atomic_read(&tbl->entries) < READ_ONCE(tbl->gc_thresh1))
		goto out;

	for (i = 0 ; i < (1 << nht->hash_shift); i++) {
		neigh_for_each_in_bucket_safe(n, tmp, &nht->hash_heads[i]) {
			unsigned int state;

			write_lock(&n->lock);

			state = n->nud_state;
			if ((state & (NUD_PERMANENT | NUD_IN_TIMER)) ||
			    (n->flags &
			     (NTF_EXT_LEARNED | NTF_EXT_VALIDATED))) {
				write_unlock(&n->lock);
				continue;
			}

			if (time_before(n->used, n->confirmed) &&
			    time_is_before_eq_jiffies(n->confirmed))
				n->used = n->confirmed;

			if (refcount_read(&n->refcnt) == 1 &&
			    (state == NUD_FAILED ||
			     !time_in_range_open(jiffies, n->used,
						 n->used + NEIGH_VAR(n->parms, GC_STALETIME)))) {
				hlist_del_rcu(&n->hash);
				hlist_del_rcu(&n->dev_list);
				neigh_mark_dead(n);
				write_unlock(&n->lock);
				neigh_cleanup_and_release(n);
				continue;
			}
			write_unlock(&n->lock);
		}
		/*
		 * It's fine to release lock here, even if hash table
		 * grows while we are preempted.
		 */
		spin_unlock_bh(&tbl->lock);
		cond_resched();
		spin_lock_bh(&tbl->lock);
		nht = rcu_dereference_protected(tbl->nht,
						lockdep_is_held(&tbl->lock));
	}
out:
	/* Cycle through all hash buckets every BASE_REACHABLE_TIME/2 ticks.
	 * ARP entry timeouts range from 1/2 BASE_REACHABLE_TIME to 3/2
	 * BASE_REACHABLE_TIME.
	 */
	queue_delayed_work(system_power_efficient_wq, &tbl->gc_work,
			      NEIGH_VAR(&tbl->parms, BASE_REACHABLE_TIME) >> 1);
	spin_unlock_bh(&tbl->lock);
}
```

清理条件：
- 引用计数为 1（只有表本身持有）。
- 状态是 `NUD_FAILED`。
- 或 `used` 时间超过 `gc_staletime`。

#### 强制 GC：`neigh_forced_gc()`

当条目数超过 `gc_thresh3`，或超过 `gc_thresh2` 且 5 秒内没清理过时触发。优先清理 FAILED/NOARP/过期的条目。

---

## 11. ARP 请求触发、排队、超时完整链路

这是 Linux ARP 最精髓的部分。一句话总结：IP 包先发到一个“未解析”的 neighbour，neighbour 子系统把包挂起并触发 ARP 探测，收到 Reply 后再把挂起的包发出去。

### 11.1 触发：从 IP 层到 `neigh_resolve_output()`

当 IP 层要发送一个报文时：

```text
ip_queue_xmit(skb)
  → 路由查找，得到 dst
  → dst->neighbour 可能已存在或需要查找
  → 调用 neigh->output(skb)
```

如果 neighbour 状态不是 `NUD_CONNECTED`，`output` 指向 `neigh_resolve_output()`。

### 11.2 `neigh_resolve_output()`：解析入口

```c
int neigh_resolve_output(struct neighbour *neigh, struct sk_buff *skb)
{
	int rc = 0;

	if (!neigh_event_send(neigh, skb)) {
		int err;
		struct net_device *dev = neigh->dev;
		unsigned int seq;

		if (dev->header_ops->cache && !READ_ONCE(neigh->hh.hh_len))
			neigh_hh_init(neigh);

		do {
			__skb_pull(skb, skb_network_offset(skb));
			seq = read_seqbegin(&neigh->ha_lock);
			err = dev_hard_header(skb, dev, ntohs(skb->protocol),
					      neigh->ha, NULL, skb->len);
		} while (read_seqretry(&neigh->ha_lock, seq));

		if (err >= 0)
			rc = dev_queue_xmit(skb);
		else
			goto out_kfree_skb;
	}
out:
	return rc;
out_kfree_skb:
	rc = -EINVAL;
	kfree_skb_reason(skb, SKB_DROP_REASON_NEIGH_HH_FILLFAIL);
	goto out;
}
```

`neigh_event_send()` 是触发器：

```c
static inline int neigh_event_send(struct neighbour *neigh, struct sk_buff *skb)
{
	return neigh_event_send_probe(neigh, skb, true);
}

static __always_inline int neigh_event_send_probe(struct neighbour *neigh,
						  struct sk_buff *skb,
						  const bool immediate_ok)
{
	unsigned long now = jiffies;

	if (READ_ONCE(neigh->used) != now)
		WRITE_ONCE(neigh->used, now);
	if (!(READ_ONCE(neigh->nud_state) & (NUD_CONNECTED | NUD_DELAY | NUD_PROBE)))
		return __neigh_event_send(neigh, skb, immediate_ok);
	return 0;
}
```

- 更新 `used` 时间戳（用于 GC）。
- 如果已经是 CONNECTED/DELAY/PROBE 状态，不重复触发，直接返回 0。
- 否则进入 `__neigh_event_send()`。

### 11.3 `__neigh_event_send()`：状态机启动与排队

```c
int __neigh_event_send(struct neighbour *neigh, struct sk_buff *skb,
		       const bool immediate_ok)
{
	int rc;
	bool immediate_probe = false;

	write_lock_bh(&neigh->lock);

	rc = 0;
	if (neigh->nud_state & (NUD_CONNECTED | NUD_DELAY | NUD_PROBE))
		goto out_unlock_bh;
	if (neigh->dead)
		goto out_dead;
```

再次检查：加锁后再次检查状态，防止并发。

#### 11.3.1 从未解析状态启动探测

```c
	if (!(neigh->nud_state & (NUD_STALE | NUD_INCOMPLETE))) {
		if (NEIGH_VAR(neigh->parms, MCAST_PROBES) +
		    NEIGH_VAR(neigh->parms, APP_PROBES)) {
			unsigned long next, now = jiffies;

			atomic_set(&neigh->probes,
				   NEIGH_VAR(neigh->parms, UCAST_PROBES));
			neigh_del_timer(neigh);
			WRITE_ONCE(neigh->nud_state, NUD_INCOMPLETE);
			neigh->updated = now;
			if (!immediate_ok) {
				next = now + 1;
			} else {
				immediate_probe = true;
				next = now + max(NEIGH_VAR(neigh->parms,
							   RETRANS_TIME),
						 HZ / 100);
			}
			neigh_add_timer(neigh, next);
		} else {
			WRITE_ONCE(neigh->nud_state, NUD_FAILED);
			neigh->updated = jiffies;
			write_unlock_bh(&neigh->lock);

			kfree_skb_reason(skb, SKB_DROP_REASON_NEIGH_FAILED);
			return 1;
		}
	}
```

- 当前状态既不是 STALE 也不是 INCOMPLETE（即 NONE 或其他无效态）。
- 设置 `probes = ucast_probes` 是为了让 `arp_solicit()` 跳过 unicast probe 阶段。首次解析通常没有旧 MAC，应直接发广播 ARP Request。
- 状态设为 `NUD_INCOMPLETE`。
- 启动定时器：
  - `immediate_ok=true`（常规路径）：定时器在 `RETRANS_TIME` 后触发，但会立即发第一次 probe（见下文）。
  - `immediate_ok=false`：定时器 1 个 jiffy 后触发。

#### 11.3.2 STALE 状态的处理

```c
	} else if (neigh->nud_state & NUD_STALE) {
		neigh_dbg(2, "neigh %p is delayed\n", neigh);
		neigh_del_timer(neigh);
		WRITE_ONCE(neigh->nud_state, NUD_DELAY);
		neigh->updated = jiffies;
		neigh_add_timer(neigh, jiffies +
				NEIGH_VAR(neigh->parms, DELAY_PROBE_TIME));
	}
```

- 如果条目是 STALE（有旧 MAC 但过期），不立即发 ARP，先进入 DELAY 状态。
- 在 `DELAY_PROBE_TIME` 内如果上层继续使用，则直接发；超时后再转 PROBE。
- 这是为了批量流量时减少 ARP 请求。

#### 11.3.3 排队待解析的 skb

```c
	if (neigh->nud_state == NUD_INCOMPLETE) {
		if (skb) {
			while (neigh->arp_queue_len_bytes + skb->truesize >
			       NEIGH_VAR(neigh->parms, QUEUE_LEN_BYTES)) {
				struct sk_buff *buff;

				buff = __skb_dequeue(&neigh->arp_queue);
				if (!buff)
					break;
				neigh->arp_queue_len_bytes -= buff->truesize;
				kfree_skb_reason(buff, SKB_DROP_REASON_NEIGH_QUEUEFULL);
				NEIGH_CACHE_STAT_INC(neigh->tbl, unres_discards);
			}
			skb_dst_force(skb);
			__skb_queue_tail(&neigh->arp_queue, skb);
			neigh->arp_queue_len_bytes += skb->truesize;
		}
		rc = 1;
	}
```

- 只有在 `NUD_INCOMPLETE` 时才排队。
- 队列按字节数限制（默认 `SK_WMEM_DEFAULT`）。
- 超限时从队首丢弃旧包。
- `skb_dst_force(skb)`：确保 dst 在队列期间不会被释放。
- `rc = 1` 表示 skb 已被接管，调用方不要再释放。

```c
out_unlock_bh:
	if (immediate_probe)
		neigh_probe(neigh);
	else
		write_unlock(&neigh->lock);
	local_bh_enable();
	trace_neigh_event_send_done(neigh, rc);
	return rc;
```

- `immediate_ok=true` 时，这里立即调用 `neigh_probe()` 发第一次 ARP Request。
- 定时器仍然会在 `RETRANS_TIME` 后触发，用于重传。

### 11.4 `neigh_probe()`：真正发送 ARP 请求

```c
static void neigh_probe(struct neighbour *neigh)
	__releases(neigh->lock)
{
	struct sk_buff *skb = skb_peek_tail(&neigh->arp_queue);
	/* keep skb alive even if arp_queue overflows */
	if (skb)
		skb = skb_clone(skb, GFP_ATOMIC);
	write_unlock(&neigh->lock);
	if (neigh->ops->solicit)
		neigh->ops->solicit(neigh, skb);
	atomic_inc(&neigh->probes);
	consume_skb(skb);
}
```

- 从队列尾部取一个 skb 克隆，传给 `arp_solicit()`。
- `arp_solicit()` 根据 skb 的源 IP 决定 sender IP（受 `arp_announce` 控制）。
- `probes` 计数加 1。

### 11.5 `neigh_timer_handler()`：定时器超时与重传

```c
static void neigh_timer_handler(struct timer_list *t)
{
	unsigned long now, next;
	struct neighbour *neigh = timer_container_of(neigh, t, timer);
	bool skip_probe = false;
	unsigned int state;
	int notify = 0;

	write_lock(&neigh->lock);

	state = neigh->nud_state;
	now = jiffies;
	next = now + HZ;

	if (!(state & NUD_IN_TIMER))
		goto out;
```

`NUD_IN_TIMER` 包含 `NUD_INCOMPLETE | NUD_REACHABLE | NUD_DELAY | NUD_PROBE`。非这些状态不处理。

#### 11.5.1 REACHABLE 超时

```c
	if (state & NUD_REACHABLE) {
		if (time_before_eq(now,
				   neigh->confirmed + neigh->parms->reachable_time)) {
			neigh_dbg(2, "neigh %p is still alive\n", neigh);
			next = neigh->confirmed + neigh->parms->reachable_time;
		} else if (time_before_eq(now,
					  neigh->used +
					  NEIGH_VAR(neigh->parms, DELAY_PROBE_TIME))) {
			neigh_dbg(2, "neigh %p is delayed\n", neigh);
			WRITE_ONCE(neigh->nud_state, NUD_DELAY);
			neigh->updated = jiffies;
			neigh_suspect(neigh);
			next = now + NEIGH_VAR(neigh->parms, DELAY_PROBE_TIME);
		} else {
			neigh_dbg(2, "neigh %p is suspected\n", neigh);
			WRITE_ONCE(neigh->nud_state, NUD_STALE);
			neigh->updated = jiffies;
			neigh_suspect(neigh);
			notify = 1;
		}
	}
```

#### 11.5.2 DELAY 超时

```c
	} else if (state & NUD_DELAY) {
		if (time_before_eq(now,
				   neigh->confirmed +
				   NEIGH_VAR(neigh->parms, DELAY_PROBE_TIME))) {
			neigh_dbg(2, "neigh %p is now reachable\n", neigh);
			WRITE_ONCE(neigh->nud_state, NUD_REACHABLE);
			neigh->updated = jiffies;
			neigh_connect(neigh);
			notify = 1;
			next = neigh->confirmed + neigh->parms->reachable_time;
		} else {
			neigh_dbg(2, "neigh %p is probed\n", neigh);
			WRITE_ONCE(neigh->nud_state, NUD_PROBE);
			neigh->updated = jiffies;
			atomic_set(&neigh->probes, 0);
			notify = 1;
			next = now + max(NEIGH_VAR(neigh->parms, RETRANS_TIME),
					 HZ/100);
		}
	}
```

#### 11.5.3 INCOMPLETE / PROBE 超时

```c
	} else {
		/* NUD_PROBE|NUD_INCOMPLETE */
		next = now + max(NEIGH_VAR(neigh->parms, RETRANS_TIME), HZ/100);
	}
```

#### 11.5.4 探测次数上限检查

```c
	if ((neigh->nud_state & (NUD_INCOMPLETE | NUD_PROBE)) &&
	    atomic_read(&neigh->probes) >= neigh_max_probes(neigh)) {
		if (neigh->nud_state == NUD_PROBE &&
		    neigh->flags & NTF_EXT_VALIDATED) {
			WRITE_ONCE(neigh->nud_state, NUD_STALE);
			neigh->updated = jiffies;
		} else {
			WRITE_ONCE(neigh->nud_state, NUD_FAILED);
			neigh_invalidate(neigh);
		}
		notify = 1;
		skip_probe = true;
	}
```

`neigh_max_probes()`：

```c
static __inline__ int neigh_max_probes(struct neighbour *n)
{
	struct neigh_parms *p = n->parms;
	return NEIGH_VAR(p, UCAST_PROBES) + NEIGH_VAR(p, APP_PROBES) +
	       (n->nud_state & NUD_PROBE ? NEIGH_VAR(p, MCAST_REPROBES) :
	        NEIGH_VAR(p, MCAST_PROBES));
}
```

- 初始从 `NUD_NONE` 进入 `NUD_INCOMPLETE` 时，`probes` 被设为 `UCAST_PROBES`，因此 `arp_solicit()` 会跳过 unicast 阶段，直接进入 APP/MCAST 阶段。
- `NUD_PROBE` 只由 `NUD_DELAY` 超时进入；进入时 `probes` 被重置为 0，然后按 `UCAST_PROBES`、`APP_PROBES`、`MCAST_REPROBES` 的上限检查。
- 达到上限后通常标记 `NUD_FAILED`；如果 `NUD_PROBE` 且带 `NTF_EXT_VALIDATED`，则退回 `NUD_STALE`。

#### 11.5.5 重设定时器并继续探测

```c
	if (neigh->nud_state & NUD_IN_TIMER) {
		if (time_before(next, jiffies + HZ/100))
			next = jiffies + HZ/100;
		if (!mod_timer(&neigh->timer, next))
			neigh_hold(neigh);
	}
	if (neigh->nud_state & (NUD_INCOMPLETE | NUD_PROBE)) {
		neigh_probe(neigh);
	} else {
out:
		write_unlock(&neigh->lock);
	}
```

### 11.6 解析成功：`neigh_update()` 与 `process_arp_queue`

**总结：ARP Reply 到来 → 邻居表写入 MAC 并变为 REACHABLE → 把 ARP 解析期间缓存的包重新 lookup 邻居后发出。**

收到 ARP Reply 后，`arp_process()` 调用：

```c
neigh_update(n, sha, NUD_REACHABLE, NEIGH_UPDATE_F_OVERRIDE, 0);
```

`neigh_update()` 内部调用 `__neigh_update()`：

```c
if (lladdr != neigh->ha) {
	write_seqlock(&neigh->ha_lock);
	memcpy(&neigh->ha, lladdr, dev->addr_len);
	write_sequnlock(&neigh->ha_lock);
	neigh_update_hhs(neigh);
	if (!(new & NUD_CONNECTED))
		neigh->confirmed = jiffies -
			      (NEIGH_VAR(neigh->parms, BASE_REACHABLE_TIME) << 1);
	notify = 1;
}
if (new == old)
	goto out;
if (new & NUD_CONNECTED)
	neigh_connect(neigh);
else
	neigh_suspect(neigh);

if (!(old & NUD_VALID))
	process_arp_queue = true;
```

然后：

```c
if (process_arp_queue)
	neigh_update_process_arp_queue(neigh);
```

`neigh_update_process_arp_queue()`：

```c
static void neigh_update_process_arp_queue(struct neighbour *neigh)
	__releases(neigh->lock)
	__acquires(neigh->lock)
{
	struct sk_buff *skb;

	/* Again: avoid deadlock if something went wrong. */
	while (neigh->nud_state & NUD_VALID &&
	       (skb = __skb_dequeue(&neigh->arp_queue)) != NULL) {
		struct dst_entry *dst = skb_dst(skb);
		struct neighbour *n2, *n1 = neigh;

		write_unlock_bh(&neigh->lock);

		rcu_read_lock();

		/* Why not just use 'neigh' as-is?  The problem is that
		 * things such as shaper, eql, and sch_teql can end up
		 * using alternative, different, neigh objects to output
		 * the packet in the output path.  So what we need to do
		 * here is re-lookup the top-level neigh in the path so
		 * we can reinject the packet there.
		 */
		n2 = NULL;
		if (dst &&
		    READ_ONCE(dst->obsolete) != DST_OBSOLETE_DEAD) {
			n2 = dst_neigh_lookup_skb(dst, skb);
			if (n2)
				n1 = n2;
		}
		READ_ONCE(n1->output)(n1, skb);
		if (n2)
			neigh_release(n2);
		rcu_read_unlock();

		write_lock_bh(&neigh->lock);
	}
	__skb_queue_purge(&neigh->arp_queue);
	neigh->arp_queue_len_bytes = 0;
}
```

- 从队列中取出所有挂起的 skb。
- 对每个 skb 重新调用 `neigh->output()`，此时状态已是 REACHABLE，会走 `neigh_connected_output()` 直接发出去。
- 清空队列。

### 11.7 解析失败：`neigh_invalidate()`

总结：解析失败 → 把 `arp_queue` 里的包逐个报错丢弃 → 上层（如 TCP）感知链路失败并做相应处理。

```c
static void neigh_invalidate(struct neighbour *neigh)
	__releases(neigh->lock)
	__acquires(neigh->lock)
{
	struct sk_buff *skb;

	NEIGH_CACHE_STAT_INC(neigh->tbl, res_failed);
	neigh_dbg(2, "neigh %p is failed\n", neigh);
	neigh->updated = jiffies;

	/* It is very thin place. report_unreachable is very complicated
	   routine. Particularly, it can hit the same neighbour entry!

	   So that, we try to be accurate and avoid dead loop. --ANK
	 */
	while (neigh->nud_state == NUD_FAILED &&
	       (skb = __skb_dequeue(&neigh->arp_queue)) != NULL) {
		write_unlock(&neigh->lock);
		neigh->ops->error_report(neigh, skb);
		write_lock(&neigh->lock);
	}
	__skb_queue_purge(&neigh->arp_queue);
	neigh->arp_queue_len_bytes = 0;
}
```

`arp_error_report()`：

```c
static void arp_error_report(struct neighbour *neigh, struct sk_buff *skb)
{
	dst_link_failure(skb);
	kfree_skb_reason(skb, SKB_DROP_REASON_NEIGH_FAILED);
}
```

对 TCP 来说，`dst_link_failure()` 会记录一些错误信息，通知用户态进程。

---

## 12. 完整 ARP 解析时序图

### 12.1 正常解析成功

```text
时间轴 ──────────────────────────────────────────────>

应用层        sendto()
  │              │
  ▼              ▼
IP 层      ip_queue_xmit(skb, dst=192.148.1.3)
  │              │
  │              ▼
  │      dst_neigh_output(dst, skb)
  │              │
  │              ▼
  │      neigh->output = neigh_resolve_output
  │              │
  │              ▼
  │      neigh_event_send(neigh, skb)
  │              │
  │              ▼
  │      __neigh_event_send()
  │      nud_state = NUD_INCOMPLETE
  │      skb → arp_queue
  │      neigh_add_timer(RETRANS_TIME)
  │      neigh_probe() ──► arp_solicit() ──► arp_send_dst(REQUEST)
  │              │
  │              │  等待 ARP Reply ...
  │              │
  │              ▼
  │      arp_rcv() ──► arp_process()
  │              │
  │              ▼
  │      neigh_update(n, sha, NUD_REACHABLE)
  │              │
  │              ▼
  │      neigh_update_process_arp_queue()
  │      从 arp_queue 取出 skb
  │              │
  │              ▼
  │      neigh->output = neigh_connected_output
  │      dev_hard_header() 填充 MAC
  │      dev_queue_xmit(skb)
  │              │
  ▼              ▼
网卡驱动     发送成功
```

### 12.2 状态机转换完整图

```text
NUD_NONE / NUD_FAILED
    └─ 新 IP 包触发
       → NUD_INCOMPLETE
           ├─ 收到 ARP Reply
           │  → NUD_REACHABLE
           └─ probes >= neigh_max_probes()
              → NUD_FAILED

NUD_REACHABLE
    └─ reachable_time 到期
       ├─ neigh->used 仍在 DELAY_PROBE_TIME 窗口内
       │  → NUD_DELAY
       └─ 久未使用
          → NUD_STALE

NUD_STALE
    └─ 再次发包触发
       → NUD_DELAY

NUD_DELAY
    └─ DELAY_PROBE_TIME 到期
       ├─ confirmed 已更新
       │  → NUD_REACHABLE
       └─ confirmed 未更新
          → NUD_PROBE

NUD_PROBE
    ├─ 收到 ARP Reply
    │  → NUD_REACHABLE
    └─ probes >= neigh_max_probes()
       ├─ NTF_EXT_VALIDATED
       │  → NUD_STALE
       └─ 普通条目
          → NUD_FAILED
```

### 12.3 探测次数与类型

```text
NUD_INCOMPLETE
    ├─ 初始进入时 probes = UCAST_PROBES
    ├─ arp_solicit() 因此跳过 unicast probe
    ├─ APP_PROBES 阶段：可交给 userspace neigh_app_ns()
    ├─ MCAST_PROBES 阶段：发送广播 ARP Request
    ├─ 收到 Reply → NUD_REACHABLE
    └─ probes >= UCAST_PROBES + APP_PROBES + MCAST_PROBES
       → NUD_FAILED

NUD_PROBE
    ├─ 只能从 NUD_DELAY 超时进入
    ├─ 进入时 probes = 0
    ├─ 先尝试 UCAST_PROBES 次 unicast probe（有旧 MAC）
    ├─ 再走 APP_PROBES / MCAST_REPROBES
    ├─ 收到 Reply → NUD_REACHABLE
    └─ 达到 neigh_max_probes()
       → NUD_FAILED（或 NTF_EXT_VALIDATED 条目回到 NUD_STALE）
```

---

## 13. 常见问题与调试

### 13.1 为什么 ARP 表里有 STALE 状态还能通信？

`NUD_STALE` 表示 MAC 已过期，但 Linux 不会立即丢弃。当上层继续发包时：
- `neigh_event_send()` 发现是 `NUD_STALE`，进入 `__neigh_event_send()`。
- 转到 `NUD_DELAY`，**不丢包、不发 ARP**。
- 在 `DELAY_PROBE_TIME` 内如果成功发送出去（此时仍使用旧 MAC），且收到任何确认（如 TCP ACK），则回到 `REACHABLE`。
- 只有 delay 超时后才进入 `PROBE` 发 ARP。

这是一种性能优化，避免每个过期条目都立刻触发 ARP。

### 13.2 为什么第一次 ping 有点慢？

因为目标 IP 的 neighbour 初始是 `NUD_NONE`，需要：
1. `neigh_event_send()` 启动 INCOMPLETE。
2. 发 ARP Request。
3. 等 Reply。
4. 再从 `arp_queue` 发出第一个 ICMP Echo Request。

这个延迟就是第一次 ARP 解析时间。

### 13.3 如何观察 ARP 解析过程？

最简单的方法是虚拟机内开一个tcpdump抓包，观察从ping一个陌生ip到彻底ping通的一个状态机的变化。

```bash
# 查看 ARP 表
ip neigh show

# 抓 ARP 包
tcpdump -i eth0 -nn arp

# 查看邻居统计
cat /proc/net/stat/arp_cache

# 查看 dropwatch 风格的丢弃原因
cat /sys/kernel/debug/tracing/events/skb/skb_kfree/enable

# 查看具体邻居条目状态
cat /proc/net/arp
```

### 13.4 参数调优示例

```bash
# 缩短 ARP 探测间隔（毫秒）
sysctl -w net.ipv4.neigh.eth0.retrans_time_ms=250

# 增加重试次数
sysctl -w net.ipv4.neigh.eth0.mcast_solicit=5

# 开启免费 ARP 自动学习
sysctl -w net.ipv4.conf.eth0.arp_accept=1

# 让接口只回复属于自己的 ARP
sysctl -w net.ipv4.conf.eth0.arp_ignore=1
```

---