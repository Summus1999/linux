# ICMP 协议详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/icmp.c` —— ICMPv4 协议主实现
> - `include/uapi/linux/icmp.h` —— ICMP 报文头与类型/代码定义
> - `include/net/icmp.h` —— ICMP 模块内部接口
> - `net/ipv4/ip_input.c` —— 本地交付与协议不可达错误
> - `net/ipv4/ip_forward.c` —— TTL 超时、DF 分片Needed 等转发错误
> - `net/ipv4/route.c` —— 路由不可达、ICMP Redirect
> - `net/ipv4/ip_fragment.c` —— 分片重组超时
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图和时序图使用 `text` 代码块表示。

---

## 1. ICMP 协议基础

作用：ICMP（Internet Control Message Protocol，互联网控制报文协议）是 TCP/IP 协议栈的网络层辅助协议，用于在 IP 主机、路由器之间传递控制消息与差错报告。它不承载用户数据，而是承载“关于 IP 数据报的报告”。

核心 RFC：RFC 792；RFC 1122 对主机行为做了大量补充；RFC 1812 对路由器行为做了补充。

典型交互：

```text
主机 A  ping  主机 B
→ A 构造 ICMP Echo Request（type=8, code=0），填入 id/sequence
→ 封装到 IP 数据报（protocol=1），目的地址为 B
→ B 收到后校验 ICMP checksum，交给 icmp_rcv()
→ B 在 icmp_echo() 里把 type 改成 0（Echo Reply），原样回显数据
→ A 收到 Echo Reply，ping 程序根据 id/sequence 匹配并计算时延
```

```text
主机 A 访问一个不可达端口
→ A 发 TCP SYN / UDP 报文到某主机
→ 中间路由器无路由 / 目的主机无监听进程
→ 路由器/主机回送 ICMP Destination Unreachable（type=3, code=0/1/2/3）
→ A 的传输层把该错误映射为 EHOSTUNREACH / ECONNREFUSED 等 errno
```

重要原则：ICMP 本身不保证可靠传输，也没有重传机制；它依赖 IP 进行“尽力而为”交付。

---

## 2. ICMP 报文格式

### 2.1 ICMP 通用头部

```c
struct icmphdr {
  __u8		type;
  __u8		code;
  __sum16	checksum;
  union {
	struct {
		__be16	id;
		__be16	sequence;
	} echo;
	__be32	gateway;
	struct {
		__be16	__unused;
		__be16	mtu;
	} frag;
	__u8	reserved[4];
  } un;
};
```

| 字段 | 长度 | 含义 |
|------|------|------|
| `type` | 1B | ICMP 类型，决定报文语义 |
| `code` | 1B | 类型的子代码，进一步细化原因 |
| `checksum` | 2B | 覆盖整个 ICMP 报文的校验和 |
| `un` | 4B | 随类型变化：Echo 用 `id`+`sequence`；Redirect 用 `gateway`；Frag Needed 用 `mtu` |

### 2.2 常见类型与代码

```c
#define ICMP_ECHOREPLY		0	/* Echo Reply			*/
#define ICMP_DEST_UNREACH	3	/* Destination Unreachable	*/
#define ICMP_SOURCE_QUENCH	4	/* Source Quench		*/
#define ICMP_REDIRECT		5	/* Redirect (change route)	*/
#define ICMP_ECHO		8	/* Echo Request			*/
#define ICMP_TIME_EXCEEDED	11	/* Time Exceeded		*/
#define ICMP_PARAMETERPROB	12	/* Parameter Problem		*/
#define ICMP_TIMESTAMP		13	/* Timestamp Request		*/
#define ICMP_TIMESTAMPREPLY	14	/* Timestamp Reply		*/
#define ICMP_INFO_REQUEST	15	/* Information Request		*/
#define ICMP_INFO_REPLY		16	/* Information Reply		*/
#define ICMP_ADDRESS		17	/* Address Mask Request		*/
#define ICMP_ADDRESSREPLY	18	/* Address Mask Reply		*/
```

Destination Unreachable 的常用 code：

```c
#define ICMP_NET_UNREACH	0
#define ICMP_HOST_UNREACH	1
#define ICMP_PROT_UNREACH	2
#define ICMP_PORT_UNREACH	3
#define ICMP_FRAG_NEEDED	4	/* Fragmentation Needed/DF set */
#define ICMP_SR_FAILED		5
```

Time Exceeded 的 code：

```c
#define ICMP_EXC_TTL		0	/* TTL count exceeded */
#define ICMP_EXC_FRAGTIME	1	/* Fragment Reass time exceeded */
```

### 2.3 封装关系

```text
| 以太网头 | IP 头 (protocol=1) | ICMP 头 | ICMP payload |
```

- IP 头中的 `protocol` 字段值为 `IPPROTO_ICMP = 1`。
- ICMP 报文紧跟 IP 头之后，IP 头长度由 `ihl*4` 决定。
- ICMP 校验和覆盖 ICMP 头 + payload，不包含 IP 头。

---

## 3. Linux 内核 ICMP 实现总览

| 项目 | 文件/结构 |
|------|-----------|
| ICMP 协议主文件 | `net/ipv4/icmp.c` |
| ICMP 头定义 | `include/uapi/linux/icmp.h` |
| ICMP 内部接口 | `include/net/icmp.h` |
| 协议注册 | `net/ipv4/af_inet.c` 中的 `icmp_protocol` |
| 收包入口 | `icmp_rcv()` |
| 发包入口 | `icmp_send()` / `__icmp_send()`（错误报文）；`icmp_reply()`（回复报文） |
| Echo Reply 处理 | `ping_rcv()`（`net/ipv4/ping.c`） |
| 差错分发 | `icmp_socket_deliver()` 把错误交给传输层 |
| 全局速率限制 | `icmp_global_allow()` + `icmpv4_xrlim_allow()` |

协议注册：

```c
static const struct net_protocol icmp_protocol = {
	.handler =	icmp_rcv,
	.err_handler =	icmp_err,
	.no_policy =	1,
};
```

在 `inet_init()` 中：

```c
if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
    pr_crit("inet_add_protocol(ICMP) failed\n");
```

`inet_protos[IPPROTO_ICMP]` 被设置为 `icmp_protocol`，当 `ip_local_deliver_finish()` 看到 IP 头的 `protocol == IPPROTO_ICMP` 时，就会调用 `icmp_rcv(skb)`。

---

## 4. ICMP 接收流程

### 4.1 总体调用链

```text
网卡驱动收到帧
  → __netif_receive_skb_core()
    → 根据 ethertype=0x0800 识别 IPv4
    → ip_rcv() / ip_rcv_finish()
      → 路由判定：本地、转发、丢弃
      → 若是本地路由，dst->input = ip_local_deliver()
        → ip_defrag() 重组
        → NF_INET_LOCAL_IN Netfilter 链
        → ip_local_deliver_finish()
          → 根据 ip_hdr(skb)->protocol 索引 inet_protos[]
          → icmp_rcv(skb)
            → 校验 checksum
            → 根据 type 查 icmp_pointers[] 分派
            → icmp_echo() / icmp_unreach() / icmp_redirect() / ...
```

### 4.2 icmp_rcv：入口与基础校验

```c
int icmp_rcv(struct sk_buff *skb)
{
	enum skb_drop_reason reason = SKB_DROP_REASON_NOT_SPECIFIED;
	struct rtable *rt = skb_rtable(skb);
	struct net *net = dev_net_rcu(rt->dst.dev);
	struct icmphdr *icmph;

	if (!xfrm4_policy_check(NULL, XFRM_POLICY_IN, skb)) {
		/* ... XFRM 特殊处理 ... */
	}

	__ICMP_INC_STATS(net, ICMP_MIB_INMSGS);

	if (skb_checksum_simple_validate(skb))
		goto csum_error;

	if (!pskb_pull(skb, sizeof(*icmph)))
		goto error;

	icmph = icmp_hdr(skb);

	ICMPMSGIN_INC_STATS(net, icmph->type);

	/* Check for ICMP Extended Echo (PROBE) messages */
	if (icmph->type == ICMP_EXT_ECHO) {
		reason = icmp_echo(skb);
		goto reason_check;
	}

	/* 对广播/组播目的地的过滤 */
	if (rt->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST)) {
		if ((icmph->type == ICMP_ECHO ||
		     icmph->type == ICMP_TIMESTAMP) &&
		    READ_ONCE(net->ipv4.sysctl_icmp_echo_ignore_broadcasts)) {
			reason = SKB_DROP_REASON_INVALID_PROTO;
			goto error;
		}
		if (icmph->type != ICMP_ECHO &&
		    icmph->type != ICMP_TIMESTAMP &&
		    icmph->type != ICMP_ADDRESS &&
		    icmph->type != ICMP_ADDRESSREPLY) {
			reason = SKB_DROP_REASON_INVALID_PROTO;
			goto error;
		}
	}

	if (icmph->type == ICMP_EXT_ECHOREPLY ||
	    icmph->type == ICMP_ECHOREPLY) {
		reason = ping_rcv(skb);
		return reason ? NET_RX_DROP : NET_RX_SUCCESS;
	}

	if (icmph->type > NR_ICMP_TYPES) {
		reason = SKB_DROP_REASON_UNHANDLED_PROTO;
		goto error;
	}

	reason = icmp_pointers[icmph->type].handler(skb);
reason_check:
	if (!reason)  {
		consume_skb(skb);
		return NET_RX_SUCCESS;
	}

drop:
	kfree_skb_reason(skb, reason);
	return NET_RX_DROP;
csum_error:
	reason = SKB_DROP_REASON_ICMP_CSUM;
	__ICMP_INC_STATS(net, ICMP_MIB_CSUMERRORS);
error:
	__ICMP_INC_STATS(net, ICMP_MIB_INERRORS);
	goto drop;
}
```

校验要点：
1. XFRM 策略检查（IPsec 相关）。
2. ICMP 校验和校验（`skb_checksum_simple_validate`）。
3. 拉取 ICMP 头（`pskb_pull`）。
4. 对广播/组播目的地的限制：Echo/Timestamp 受 `icmp_echo_ignore_broadcasts` 控制；其他类型默认丢弃。
5. Echo Reply / Extended Echo Reply 直接交给 `ping_rcv()`。
6. 未知类型（`type > 18`）静默丢弃，符合 RFC 1122。
7. 其余类型查 `icmp_pointers[]` 分派。

### 4.3 icmp_pointers：分派表

```c
static const struct icmp_control icmp_pointers[NR_ICMP_TYPES + 1] = {
	[ICMP_ECHOREPLY] = {
		.handler = ping_rcv,
	},
	[1] = { .handler = icmp_discard, .error = 1 },
	[2] = { .handler = icmp_discard, .error = 1 },
	[ICMP_DEST_UNREACH] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_SOURCE_QUENCH] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_REDIRECT] = {
		.handler = icmp_redirect,
		.error = 1,
	},
	[6] = { .handler = icmp_discard, .error = 1 },
	[7] = { .handler = icmp_discard, .error = 1 },
	[ICMP_ECHO] = {
		.handler = icmp_echo,
	},
	[9] = { .handler = icmp_discard, .error = 1 },
	[10] = { .handler = icmp_discard, .error = 1 },
	[ICMP_TIME_EXCEEDED] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_PARAMETERPROB] = {
		.handler = icmp_unreach,
		.error = 1,
	},
	[ICMP_TIMESTAMP] = {
		.handler = icmp_timestamp,
	},
	[ICMP_TIMESTAMPREPLY] = { .handler = icmp_discard },
	[ICMP_INFO_REQUEST] = { .handler = icmp_discard },
	[ICMP_INFO_REPLY] = { .handler = icmp_discard },
	[ICMP_ADDRESS] = { .handler = icmp_discard },
	[ICMP_ADDRESSREPLY] = { .handler = icmp_discard },
};
```

`error = 1` 表示该 ICMP 类型属于差错报文。这个标记在 `__icmp_send()` 中用于防止“对 ICMP 差错报文再回 ICMP 差错报文”的无限循环。

---

## 5. ICMP Echo 处理

### 5.1 Echo Request → Echo Reply

```c
static enum skb_drop_reason icmp_echo(struct sk_buff *skb)
{
	DEFINE_RAW_FLEX(struct icmp_bxm, icmp_param, replyopts.opt.__data,
			IP_OPTIONS_DATA_FIXED_SIZE);
	struct net *net;

	net = skb_dst_dev_net_rcu(skb);
	if (READ_ONCE(net->ipv4.sysctl_icmp_echo_ignore_all))
		return SKB_NOT_DROPPED_YET;

	icmp_param->data.icmph	   = *icmp_hdr(skb);
	icmp_param->skb		   = skb;
	icmp_param->offset	   = 0;
	icmp_param->data_len	   = skb->len;
	icmp_param->head_len	   = sizeof(struct icmphdr);

	if (icmp_param->data.icmph.type == ICMP_ECHO)
		icmp_param->data.icmph.type = ICMP_ECHOREPLY;
	else if (!icmp_build_probe(skb, &icmp_param->data.icmph))
		return SKB_NOT_DROPPED_YET;

	icmp_reply(icmp_param, skb);
	return SKB_NOT_DROPPED_YET;
}
```

要点：
1. 受 `icmp_echo_ignore_all` 控制：设为 1 时本机不响应任何 ping。
2. 直接把收到的 `icmphdr` 复制一份，将 `type` 由 8 改为 0。
3. `data_len = skb->len` 表示要把 Echo Request 的 payload 原样回显。
4. 调用 `icmp_reply()` 构造并发送回复。

### 5.2 Echo Reply → ping_rcv

当本机发出 Echo Request 后，对端返回的 Echo Reply 由 `ping_rcv()` 处理：

```c
enum skb_drop_reason ping_rcv(struct sk_buff *skb)
{
	struct net *net = dev_net(skb->dev);
	struct icmphdr *icmph = icmp_hdr(skb);
	struct sock *sk;

	/* Push ICMP header back */
	skb_push(skb, skb->data - (u8 *)icmph);

	sk = ping_lookup(net, skb, ntohs(icmph->un.echo.id));
	if (sk)
		return __ping_queue_rcv_skb(sk, skb);

	kfree_skb_reason(skb, SKB_DROP_REASON_NO_SOCKET);
	return SKB_DROP_REASON_NO_SOCKET;
}
```

`ping_lookup()` 根据 `(net, saddr, daddr, id)` 找到对应的 ping socket，把 skb 放入该 socket 的接收队列，用户态 `ping`/`recvfrom` 即可读到数据。

### 5.3 ping 请求/回复完整时序

```text
用户态：ping 程序
  │
  ▼
创建 ICMP raw socket（或通过 ping socket）
  │
  ▼
sendto() 构造 ICMP Echo Request
  │
  ▼
内核：ping_v4_sendmsg() → ip_push_pending_frames() → ip_output()
  │
  ▼
网络 → 对端主机
  │
  ▼
对端：ip_local_deliver() → icmp_rcv()
  │
  ▼
icmp_echo() 生成 ICMP Echo Reply
  │
  ▼
icmp_reply() → ip_route_output_key() → ip_append_data() → ip_push_pending_frames()
  │
  ▼
网络 → 本机
  │
  ▼
本机：ip_local_deliver() → icmp_rcv()
  │
  ▼
ping_rcv() → ping_lookup() → sock_queue_rcv_skb()
  │
  ▼
用户态 recvfrom() 收到 Echo Reply
```

---

## 6. ICMP 差错报文处理（icmp_unreach）

`icmp_unreach()` 集中处理 `ICMP_DEST_UNREACH`、`ICMP_TIME_EXCEEDED`、`ICMP_PARAMETERPROB` 和 `ICMP_SOURCE_QUENCH`。

```c
static enum skb_drop_reason icmp_unreach(struct sk_buff *skb)
{
	enum skb_drop_reason reason = SKB_NOT_DROPPED_YET;
	const struct iphdr *iph;
	struct icmphdr *icmph;
	struct net *net;
	u32 info = 0;

	net = skb_dst_dev_net_rcu(skb);

	if (!pskb_may_pull(skb, sizeof(struct iphdr)))
		goto out_err;

	icmph = icmp_hdr(skb);
	iph   = (const struct iphdr *)skb->data;

	if (iph->ihl < 5)  {
		reason = SKB_DROP_REASON_IP_INHDR;
		goto out_err;
	}

	switch (icmph->type) {
	case ICMP_DEST_UNREACH:
		switch (icmph->code & 15) {
		case ICMP_NET_UNREACH:
		case ICMP_HOST_UNREACH:
		case ICMP_PROT_UNREACH:
		case ICMP_PORT_UNREACH:
			break;
		case ICMP_FRAG_NEEDED:
			switch (READ_ONCE(net->ipv4.sysctl_ip_no_pmtu_disc)) {
			default:
				net_dbg_ratelimited("... fragmentation needed ...\n");
				break;
			case 2:
				goto out;
			case 3:
				if (!icmp_tag_validation(iph->protocol))
					goto out;
				fallthrough;
			case 0:
				info = ntohs(icmph->un.frag.mtu);
			}
			break;
		case ICMP_SR_FAILED:
			net_dbg_ratelimited("... Source Route Failed\n");
			break;
		default:
			break;
		}
		if (icmph->code > NR_ICMP_UNREACH)
			goto out;
		break;
	case ICMP_PARAMETERPROB:
		info = ntohl(icmph->un.gateway) >> 24;
		break;
	case ICMP_TIME_EXCEEDED:
		__ICMP_INC_STATS(net, ICMP_MIB_INTIMEEXCDS);
		if (icmph->code == ICMP_EXC_FRAGTIME)
			goto out;
		break;
	}

	/* 防御bogus广播错误响应 */
	if (!READ_ONCE(net->ipv4.sysctl_icmp_ignore_bogus_error_responses) &&
	    inet_addr_type_dev_table(net, skb->dev, iph->daddr) == RTN_BROADCAST) {
		net_warn_ratelimited("... invalid ICMP error to a broadcast ...\n");
		goto out;
	}

	icmp_socket_deliver(skb, info);

out:
	return reason;
out_err:
	__ICMP_INC_STATS(net, ICMP_MIB_INERRORS);
	return reason ?: SKB_DROP_REASON_NOT_SPECIFIED;
}
```

关键逻辑：
1. 校验 ICMP payload 中至少包含原始 IP 头。
2. 对 `ICMP_FRAG_NEEDED`，提取 `icmph->un.frag.mtu` 作为 `info`，后续交给 `ipv4_update_pmtu()` 做路径 MTU 更新。
3. 对 `ICMP_PARAMETERPROB`，`info` 是出错字节偏移（高 8 位）。
4. 对 `ICMP_TIME_EXCEEDED`，code=1（分片重组超时）直接丢弃，不再向传输层报告。
5. 默认开启 `icmp_ignore_bogus_error_responses=1`，对发往广播地址的错误响应直接丢弃。
6. 最后调用 `icmp_socket_deliver()` 把错误交给原始套接字和对应传输层。

### 6.1 icmp_socket_deliver：把错误交给上层

```c
static void icmp_socket_deliver(struct sk_buff *skb, u32 info)
{
	const struct iphdr *iph = (const struct iphdr *)skb->data;
	const struct net_protocol *ipprot;
	int protocol = iph->protocol;

	if (!pskb_may_pull(skb, iph->ihl * 4 + 8))
		goto out;

	if (protocol == IPPROTO_RAW)
		goto out;

	raw_icmp_error(skb, protocol, info);

	ipprot = rcu_dereference(inet_protos[protocol]);
	if (ipprot && ipprot->err_handler)
		ipprot->err_handler(skb, info);
	return;

out:
	__ICMP_INC_STATS(dev_net_rcu(skb->dev), ICMP_MIB_INERRORS);
}
```

- 提取原始报文的 `protocol` 字段。
- 先通知监听该协议的 raw socket。
- 再调用对应协议注册的 `err_handler`：TCP 是 `tcp_v4_err()`，UDP 是 `udp_err()`，ICMP 自身是 `icmp_err()`。
- 错误码 `info` 对 TCP/UDP 的语义不同：对 Frag Needed 是 MTU；对 Parameter Problem 是偏移；其他情况多为 0。

### 6.2 ICMP 差错报文携带的原始数据

RFC 1122 规定：ICMP 差错报文必须包含原始 IP 头 + 至少 8 字节原始传输层头。Linux 实际会尽可能多地携带原始数据（不超过 576 字节），以便上层精确定位是哪条连接/哪个报文出错。

---

## 7. ICMP 发送流程

ICMP 发送有两条路径：
1. 回复路径：`icmp_reply()` —— 用于 Echo Reply、Timestamp Reply 等，以收到的 Request 为模板。
2. 错误路径：`icmp_send()` / `__icmp_send()` —— 用于 Destination Unreachable、Time Exceeded、Redirect 等，以触发错误的 skb 为输入。

### 7.1 icmp_reply：构造回复报文

```c
static void icmp_reply(struct icmp_bxm *icmp_param, struct sk_buff *skb)
{
	struct rtable *rt = skb_rtable(skb);
	struct net *net = dev_net_rcu(rt->dst.dev);
	bool apply_ratelimit = false;
	struct ipcm_cookie ipc;
	struct flowi4 fl4;
	struct sock *sk;
	__be32 daddr, saddr;
	u32 mark = IP4_REPLY_MARK(net, skb->mark);
	int type = icmp_param->data.icmph.type;
	int code = icmp_param->data.icmph.code;

	if (ip_options_echo(net, &icmp_param->replyopts.opt, skb))
		return;

	local_bh_disable();

	if (!icmpv4_global_allow(net, type, code, &apply_ratelimit))
		goto out_bh_enable;

	sk = icmp_xmit_lock(net);
	if (!sk)
		goto out_bh_enable;

	icmp_param->data.icmph.checksum = 0;

	ipcm_init(&ipc);
	ipc.tos = ip_hdr(skb)->tos;
	ipc.sockc.mark = mark;
	daddr = ipc.addr = ip_hdr(skb)->saddr;
	saddr = fib_compute_spec_dst(skb);

	if (icmp_param->replyopts.opt.optlen) {
		ipc.opt = &icmp_param->replyopts;
		if (ipc.opt->opt.srr)
			daddr = icmp_param->replyopts.opt.faddr;
	}
	memset(&fl4, 0, sizeof(fl4));
	fl4.daddr = daddr;
	fl4.saddr = saddr;
	fl4.flowi4_mark = mark;
	fl4.flowi4_uid = sock_net_uid(net, NULL);
	fl4.flowi4_dscp = ip4h_dscp(ip_hdr(skb));
	fl4.flowi4_proto = IPPROTO_ICMP;
	fl4.flowi4_oif = l3mdev_master_ifindex(skb->dev);
	security_skb_classify_flow(skb, flowi4_to_flowi_common(&fl4));
	rt = ip_route_output_key(net, &fl4);
	if (IS_ERR(rt))
		goto out_unlock;
	if (icmpv4_xrlim_allow(net, rt, &fl4, type, code, apply_ratelimit))
		icmp_push_reply(sk, icmp_param, &fl4, &ipc, &rt);
	ip_rt_put(rt);
out_unlock:
	icmp_xmit_unlock(sk);
out_bh_enable:
	local_bh_enable();
}
```

要点：
1. `ip_options_echo()` 把请求中的 IP 选项（记录路由、时间戳、严格/宽松源路由等）构造为回复所需选项。
2. 源地址通过 `fib_compute_spec_dst(skb)` 计算，通常取入接口上的本地地址。
3. 目的地址是请求报文的源地址。
4. 路由查找使用 `ip_route_output_key()`。
5. 经过全局/每主机速率限制后，调用 `icmp_push_reply()` 实际发送。

### 7.2 icmp_push_reply：组包与校验和

```c
static void icmp_push_reply(struct sock *sk,
			    struct icmp_bxm *icmp_param,
			    struct flowi4 *fl4,
			    struct ipcm_cookie *ipc, struct rtable **rt)
{
	struct sk_buff *skb;

	if (ip_append_data(sk, fl4, icmp_glue_bits, icmp_param,
			   icmp_param->data_len + icmp_param->head_len,
			   icmp_param->head_len,
			   ipc, rt, MSG_DONTWAIT) < 0) {
		__ICMP_INC_STATS(sock_net(sk), ICMP_MIB_OUTERRORS);
		ip_flush_pending_frames(sk);
	} else if ((skb = skb_peek(&sk->sk_write_queue)) != NULL) {
		struct icmphdr *icmph = icmp_hdr(skb);
		__wsum csum;
		struct sk_buff *skb1;

		csum = csum_partial_copy_nocheck((void *)&icmp_param->data,
						 (char *)icmph,
						 icmp_param->head_len);
		skb_queue_walk(&sk->sk_write_queue, skb1) {
			csum = csum_add(csum, skb1->csum);
		}
		icmph->checksum = csum_fold(csum);
		skb->ip_summed = CHECKSUM_NONE;
		ip_push_pending_frames(sk, fl4);
	}
}
```

- `ip_append_data()` 把 ICMP 头和 payload 分片写入发送队列。
- `icmp_glue_bits()` 负责把原始 skb 的数据拷贝到待发送 skb 中。
- 对所有分片累加校验和，最后写入 `icmph->checksum`。
- `ip_push_pending_frames()` 把报文交给 IP 层发送。

### 7.3 __icmp_send：构造错误报文

`icmp_send()` 是内联包装，最终调用 `__icmp_send()`：

```c
void __icmp_send(struct sk_buff *skb_in, int type, int code, __be32 info,
		 const struct inet_skb_parm *parm)
{
	DEFINE_RAW_FLEX(struct icmp_bxm, icmp_param, replyopts.opt.__data,
			IP_OPTIONS_DATA_FIXED_SIZE);
	struct iphdr *iph;
	int room;
	struct rtable *rt = skb_rtable(skb_in);
	bool apply_ratelimit = false;
	struct sk_buff *ext_skb;
	struct ipcm_cookie ipc;
	struct flowi4 fl4;
	__be32 saddr;
	u8  tos;
	u32 mark;
	struct net *net;
	struct sock *sk;

	if (!rt)
		return;

	rcu_read_lock();

	if (rt->dst.dev)
		net = dev_net_rcu(rt->dst.dev);
	else if (skb_in->dev)
		net = dev_net_rcu(skb_in->dev);
	else
		goto out;

	iph = ip_hdr(skb_in);

	if ((u8 *)iph < skb_in->head ||
	    (skb_network_header(skb_in) + sizeof(*iph)) >
	    skb_tail_pointer(skb_in))
		goto out;

	/* 不回应物理层多播/广播 */
	if (skb_in->pkt_type != PACKET_HOST)
		goto out;

	/* 协议层多播/广播 */
	if (rt->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST))
		goto out;

	/* 只回应第一个分片 */
	if (iph->frag_off & htons(IP_OFFSET))
		goto out;

	/* 防止对 ICMP 差错报文再回 ICMP 差错报文 */
	if (icmp_pointers[type].error) {
		if (iph->protocol == IPPROTO_ICMP) {
			u8 _inner_type, *itp;

			itp = skb_header_pointer(skb_in,
						 skb_network_header(skb_in) +
						 (iph->ihl << 2) +
						 offsetof(struct icmphdr, type) -
						 skb_in->data,
						 sizeof(_inner_type),
						 &_inner_type);
			if (!itp)
				goto out;

			if (*itp > NR_ICMP_TYPES ||
			    icmp_pointers[*itp].error)
				goto out;
		}
	}

	local_bh_disable();

	/* 全局速率限制，回环设备除外 */
	if (!(skb_in->dev && (skb_in->dev->flags & IFF_LOOPBACK)) &&
	      !icmpv4_global_allow(net, type, code, &apply_ratelimit))
		goto out_bh_enable;

	sk = icmp_xmit_lock(net);
	if (!sk)
		goto out_bh_enable;

	/* 构造源地址与选项 */
	saddr = iph->daddr;
	if (!(rt->rt_flags & RTCF_LOCAL)) {
		/* 非本地路由时，可能使用入接口地址作为源 */
		struct net_device *dev = NULL;

		rcu_read_lock();
		if (rt_is_input_route(rt) &&
		    READ_ONCE(net->ipv4.sysctl_icmp_errors_use_inbound_ifaddr))
			dev = dev_get_by_index_rcu(net, parm->iif ? parm->iif :
						   inet_iif(skb_in));

		if (dev)
			saddr = inet_select_addr(dev, iph->saddr, RT_SCOPE_LINK);
		else
			saddr = 0;
		rcu_read_unlock();
	}

	tos = icmp_pointers[type].error ? (RT_TOS(iph->tos) |
					   IPTOS_PREC_INTERNETCONTROL) :
					   iph->tos;
	mark = IP4_REPLY_MARK(net, skb_in->mark);

	if (__ip_options_echo(net, &icmp_param->replyopts.opt, skb_in,
			      &parm->opt))
		goto out_unlock;

	/* 准备 ICMP 头数据 */
	icmp_param->data.icmph.type	 = type;
	icmp_param->data.icmph.code	 = code;
	icmp_param->data.icmph.un.gateway = info;
	icmp_param->data.icmph.checksum	 = 0;
	icmp_param->skb			 = skb_in;
	icmp_param->offset		 = skb_network_offset(skb_in);
	ipcm_init(&ipc);
	ipc.tos = tos;
	ipc.addr = iph->saddr;
	ipc.opt = &icmp_param->replyopts;
	ipc.sockc.mark = mark;

	rt = icmp_route_lookup(net, &fl4, skb_in, iph, saddr,
			       inet_dsfield_to_dscp(tos), mark, type, code,
			       icmp_param);
	if (IS_ERR(rt))
		goto out_unlock;

	if (rt->rt_flags & (RTCF_BROADCAST | RTCF_MULTICAST))
		goto ende;

	/* 每主机速率限制 */
	if (!icmpv4_xrlim_allow(net, rt, &fl4, type, code, apply_ratelimit))
		goto ende;

	/* RFC 1122：返回的数据不超过 576 字节 */
	room = dst4_mtu(&rt->dst);
	if (room > 576)
		room = 576;
	room -= sizeof(struct iphdr) + icmp_param->replyopts.opt.optlen;
	room -= sizeof(struct icmphdr);
	if (room <= (int)sizeof(struct iphdr))
		goto ende;

	ext_skb = icmp_ext_append(net, skb_in, &icmp_param->data.icmph, room,
				  parm->iif);
	if (ext_skb)
		icmp_param->skb = ext_skb;

	icmp_param->data_len = icmp_param->skb->len - icmp_param->offset;
	if (icmp_param->data_len > room)
		icmp_param->data_len = room;
	icmp_param->head_len = sizeof(struct icmphdr);

	if (!fl4.saddr)
		fl4.saddr = htonl(INADDR_DUMMY);

	trace_icmp_send(skb_in, type, code);

	icmp_push_reply(sk, icmp_param, &fl4, &ipc, &rt);

	if (ext_skb)
		consume_skb(ext_skb);
ende:
	ip_rt_put(rt);
out_unlock:
	icmp_xmit_unlock(sk);
out_bh_enable:
	local_bh_enable();
out:
	rcu_read_unlock();
}
```

发送限制（防止滥用）：
1. 必须是对单播给自己（`PACKET_HOST`）的报文。
2. 不回应目的地址为广播/组播的报文。
3. 只回应第一个分片（`IP_OFFSET == 0`）。
4. 不回应 ICMP 差错报文：如果触发报文本身是 ICMP 且 `error=1`，则不再生成 ICMP 差错，防止循环。
5. 回环设备豁免全局速率限制。
6. 错误报文的 TOS 字段固定加上 `IPTOS_PREC_INTERNETCONTROL`。
7. 携带的原始数据长度不超过 576 字节（含 IP 头）。

---

## 8. 速率限制

ICMP 差错报文容易被用于放大攻击或洪水，因此 Linux 实现了两级限速：

### 8.1 全局令牌桶：icmp_global_allow

```c
bool icmp_global_allow(struct net *net)
{
	u32 delta, now, oldstamp;
	int incr, new, old;

	if (atomic_read(&net->ipv4.icmp_global_credit) > 0)
		return true;

	now = jiffies;
	oldstamp = READ_ONCE(net->ipv4.icmp_global_stamp);
	delta = min_t(u32, now - oldstamp, HZ);
	if (delta < HZ / 50)
		return false;

	incr = READ_ONCE(net->ipv4.sysctl_icmp_msgs_per_sec);
	incr = div_u64((u64)incr * delta, HZ);
	if (!incr)
		return false;

	if (cmpxchg(&net->ipv4.icmp_global_stamp, oldstamp, now) == oldstamp) {
		old = atomic_read(&net->ipv4.icmp_global_credit);
		do {
			new = min(old + incr, READ_ONCE(net->ipv4.sysctl_icmp_msgs_burst));
		} while (!atomic_try_cmpxchg(&net->ipv4.icmp_global_credit, &old, new));
	}
	return true;
}
```

- `icmp_global_credit`：当前可用令牌。
- `icmp_global_stamp`：上次补充令牌的时间。
- `icmp_msgs_per_sec`：每秒补充速率（默认 10000）。
- `icmp_msgs_burst`：令牌桶容量（默认 10000）。
- `icmp_global_consume()` 每次发送随机消耗 0~2 个令牌。

### 8.2 哪些类型不受全局限速

```c
static bool icmpv4_mask_allow(struct net *net, int type, int code)
{
	if (type > NR_ICMP_TYPES)
		return true;

	/* Don't limit PMTU discovery. */
	if (type == ICMP_DEST_UNREACH && code == ICMP_FRAG_NEEDED)
		return true;

	/* Limit if icmp type is enabled in ratemask. */
	if (!((1 << type) & READ_ONCE(net->ipv4.sysctl_icmp_ratemask)))
		return true;

	return false;
}
```

- `ICMP_FRAG_NEEDED`（PMTU 发现）永远不限速。
- 默认 `icmp_ratemask = 0x1818`，即 type 3、4、11、12 被限速：
  - `0x1818 = 0b0001 1000 0001 1000`
  - bit 3 = Destination Unreachable
  - bit 4 = Source Quench
  - bit 11 = Time Exceeded
  - bit 12 = Parameter Problem

### 8.3 每主机令牌桶：icmpv4_xrlim_allow

```c
static bool icmpv4_xrlim_allow(struct net *net, struct rtable *rt,
			       struct flowi4 *fl4, int type, int code,
			       bool apply_ratelimit)
{
	struct dst_entry *dst = &rt->dst;
	struct inet_peer *peer;
	struct net_device *dev;
	int peer_timeout;
	bool rc = true;

	if (!apply_ratelimit)
		return true;

	peer_timeout = READ_ONCE(net->ipv4.sysctl_icmp_ratelimit);
	if (!peer_timeout)
		goto out;

	rcu_read_lock();
	dev = dst_dev_rcu(dst);
	if (dev && (dev->flags & IFF_LOOPBACK))
		goto out_unlock;

	peer = inet_getpeer_v4(net->ipv4.peers, fl4->daddr,
			       l3mdev_master_ifindex_rcu(dev));
	rc = inet_peer_xrlim_allow(peer, peer_timeout);

out_unlock:
	rcu_read_unlock();
out:
	if (!rc)
		__ICMP_INC_STATS(net, ICMP_MIB_RATELIMITHOST);
	else
		icmp_global_consume(net);
	return rc;
}
```

- 每个目的 IP 有一个 `inet_peer` 结构，内部维护令牌桶。
- `icmp_ratelimit` 默认 `1*HZ`：同一对端每秒最多一个受限 ICMP。
- 回环设备豁免。

---

## 9. 内核中常见的 ICMP 错误生成场景

### 9.1 TTL 耗尽：ip_forward

```c
int ip_forward(struct sk_buff *skb)
{
	/* ... */
	if (ip_hdr(skb)->ttl <= 1)
		goto too_many_hops;
	/* ... */

too_many_hops:
	__IP_INC_STATS(net, IPSTATS_MIB_INHDRERRORS);
	icmp_send(skb, ICMP_TIME_EXCEEDED, ICMP_EXC_TTL, 0);
	SKB_DR_SET(reason, IP_INHDR);
drop:
	kfree_skb_reason(skb, reason);
	return NET_RX_DROP;
}
```

路由器在转发前发现 TTL <= 1，丢弃并向源主机发送 `ICMP_TIME_EXCEEDED / ICMP_EXC_TTL`。

### 9.2 DF 位置且需要分片：ip_forward / ip_fragment / ip_output

转发路径 `ip_forward`：

```c
mtu = ip_dst_mtu_maybe_forward(&rt->dst, true);
if (ip_exceeds_mtu(skb, mtu)) {
	IP_INC_STATS(net, IPSTATS_MIB_FRAGFAILS);
	icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED,
		  htonl(mtu));
	SKB_DR_SET(reason, PKT_TOO_BIG);
	goto drop;
}
```

本地输出路径 `ip_fragment`：

```c
if ((iph->frag_off & htons(IP_DF)) == 0)
	return ip_do_fragment(net, sk, skb, output);

if (unlikely(!skb->ignore_df ||\t     (IPCB(skb)->frag_max_size &&
	      IPCB(skb)->frag_max_size > mtu))) {
	IP_INC_STATS(net, IPSTATS_MIB_FRAGFAILS);
	icmp_send(skb, ICMP_DEST_UNREACH, ICMP_FRAG_NEEDED,
		  htonl(mtu));
	kfree_skb(skb);
	return -EMSGSIZE;
}
```

### 9.3 协议不可达：ip_local_deliver_finish

```c
ret = INDIRECT_CALL_2(ipprot->handler, tcp_v4_rcv, udp_rcv, skb);
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
	}
}
```

当 IP 头中的 `protocol` 字段没有对应注册协议时，发送 `ICMP_DEST_UNREACH / ICMP_PROT_UNREACH`。

### 9.4 端口不可达：udp

```c
icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);
```

UDP 收到报文但无对应 socket 时，发送端口不可达。

### 9.5 分片重组超时：ip_expire

```c
static void ip_expire(struct timer_list *t)
{
	/* ... */
	if (frag_expire_skip_icmp(qp->q.key.v4.user) &&
	    (skb_rtable(head)->rt_type != RTN_LOCAL))
		goto out;

	spin_unlock(&qp->q.lock);
	icmp_send(head, ICMP_TIME_EXCEEDED, ICMP_EXC_FRAGTIME, 0);
	/* ... */
}
```

IP 分片重组队列超时（默认 30 秒），如果属于本地交付场景，则发送 `ICMP_TIME_EXCEEDED / ICMP_EXC_FRAGTIME`。

### 9.6 ICMP Redirect：ip_rt_send_redirect

```c
void ip_rt_send_redirect(struct sk_buff *skb)
{
	/* ... */
	if (!in_dev || !IN_DEV_TX_REDIRECTS(in_dev)) {
		rcu_read_unlock();
		return;
	}
	/* ... */
	if (peer->n_redirects == 0 ||
	    time_after(jiffies,
		       (peer->rate_last +
			(ip_rt_redirect_load << peer->n_redirects)))) {
		__be32 gw = rt_nexthop(rt, ip_hdr(skb)->daddr);

		icmp_send(skb, ICMP_REDIRECT, ICMP_REDIR_HOST, gw);
		peer->rate_last = jiffies;
		++peer->n_redirects;
	}
	/* ... */
}
```

路由器发现源主机使用了非最优路由时，发送 ICMP Redirect，建议源主机把后续报文直接发给更优网关 `gw`。

### 9.7 路由不可达：ip_error

```c
static int ip_error(struct sk_buff *skb)
{
	/* ... */
	switch (rt->dst.error) {
	case EHOSTUNREACH:
		code = ICMP_HOST_UNREACH;
		break;
	case ENETUNREACH:
		code = ICMP_NET_UNREACH;
		break;
	case EACCES:
		code = ICMP_PKT_FILTERED;
		break;
	}

	/* 每主机速率限制 ... */
	if (send)
		icmp_send(skb, ICMP_DEST_UNREACH, code, 0);

out:
	kfree_skb_reason(skb, reason);
	return 0;
}
```

当路由查找失败（如无路由、策略拒绝）时，`dst->input` 指向 `ip_error()`，它把错误映射为 ICMP Destination Unreachable 后发送。

---

## 10. 重要 sysctl 参数

| 参数 | 文件 | 含义 |
|------|------|------|
| `icmp_echo_ignore_all` | `/proc/sys/net/ipv4/icmp_echo_ignore_all` | 是否忽略所有 Echo Request（默认 0） |
| `icmp_echo_ignore_broadcasts` | `/proc/sys/net/ipv4/icmp_echo_ignore_broadcasts` | 是否忽略广播/组播 Echo Request（默认 1） |
| `icmp_echo_enable_probe` | `/proc/sys/net/ipv4/icmp_echo_enable_probe` | 是否启用 ICMP Extended Echo (PROBE)（默认 0） |
| `icmp_ignore_bogus_error_responses` | `/proc/sys/net/ipv4/icmp_ignore_bogus_error_responses` | 是否忽略对广播地址的 ICMP 错误（默认 1） |
| `icmp_ratelimit` | `/proc/sys/net/ipv4/icmp_ratelimit` | 每主机 ICMP 限速间隔，0 表示不限（默认 HZ） |
| `icmp_ratemask` | `/proc/sys/net/ipv4/icmp_ratemask` | 哪些 ICMP 类型受 `icmp_ratelimit` 限制（默认 0x1818） |
| `icmp_msgs_per_sec` | `/proc/sys/net/ipv4/icmp_msgs_per_sec` | 全局每秒令牌补充速率（默认 10000） |
| `icmp_msgs_burst` | `/proc/sys/net/ipv4/icmp_msgs_burst` | 全局令牌桶容量（默认 10000） |
| `icmp_errors_use_inbound_ifaddr` | `/proc/sys/net/ipv4/icmp_errors_use_inbound_ifaddr` | 非本地路由错误是否使用入接口地址作为源（默认 0） |
| `icmp_errors_extension_mask` | `/proc/sys/net/ipv4/icmp_errors_extension_mask` | 是否在错误报文中附加 RFC 4884/5837 扩展（默认 0） |

---

## 11. ICMP 初始化

```c
int __init icmp_init(void)
{
	int err, i;

	for_each_possible_cpu(i) {
		struct sock *sk;

		err = inet_ctl_sock_create(&sk, PF_INET,
					   SOCK_RAW, IPPROTO_ICMP, &init_net);
		if (err < 0)
			return err;

		per_cpu(ipv4_icmp_sk, i) = sk;

		sk->sk_sndbuf =	2 * SKB_TRUESIZE(64 * 1024);
		sock_set_flag(sk, SOCK_USE_WRITE_QUEUE);
		inet_sk(sk)->pmtudisc = IP_PMTUDISC_DONT;
	}
	return register_pernet_subsys(&icmp_sk_ops);
}
```

- 每个 CPU 创建一个 RAW socket（`ipv4_icmp_sk`），专门用于发送 ICMP 报文。
- 发送缓冲区设置为可容纳 2 个 64KB 报文。
- `PMTUDISC_DONT`：ICMP 报文自身不进行路径 MTU 发现，避免分片。
- `icmp_sk_init()` 初始化每个 namespace 的 sysctl 默认值。

---

## 12. 完整流程图

### 12.1 ICMP Echo Request/Reply 收发

```text
发送端（本机）                          接收端（对端）
    │                                       │
    ▼                                       ▼
ping_v4_sendmsg()                   ip_local_deliver()
    │                                       │
    ▼                                       ▼
ip_push_pending_frames()            ip_local_deliver_finish()
    │                                       │
    ▼                                       ▼
ip_output() ──────────网络──────────▶ icmp_rcv()
    │                                       │
    ▼                                       ▼
dev_queue_xmit()                    skb_checksum_simple_validate()
                                            │
                                            ▼
                                    icmp_pointers[ICMP_ECHO].handler
                                            │
                                            ▼
                                        icmp_echo()
                                            │
                                            ▼
                                        icmp_reply()
                                            │
                                            ▼
                                    ip_route_output_key()
                                            │
                                            ▼
                                    icmp_push_reply()
                                            │
                                            ▼
                                    ip_push_pending_frames()
                                            │
                                            ▼
                            网络 ◀──── dev_queue_xmit()
    │
    ▼
ip_local_deliver()
    │
    ▼
icmp_rcv()
    │
    ▼
ping_rcv()
    │
    ▼
ping_lookup() → sock_queue_rcv_skb()
    │
    ▼
用户态 recvfrom() 读取 Echo Reply
```

### 12.2 ICMP 差错报文生成与上报

```text
某中间设备/主机遇到错误
  │
  ▼
调用 icmp_send(skb_in, type, code, info)
  │
  ▼
__icmp_send() 执行安全/速率检查
  │
  ▼
构造 ICMP 差错头，携带 skb_in 的 IP 头 + 部分 payload
  │
  ▼
icmp_push_reply() → IP 层 → 网络
  │
  ▼
源主机收到 ICMP 差错
  │
  ▼
ip_local_deliver() → icmp_rcv() → icmp_unreach()
  │
  ▼
icmp_socket_deliver(skb, info)
  │
  ├─→ raw socket
  └─→ inet_protos[protocol]->err_handler
        │
        ▼
    tcp_v4_err() / udp_err() / icmp_err()
        │
        ▼
    更新 socket 状态 / 通知用户态 errno
```

---

## 13. 小结

| 主题 | 核心函数/结构 | 关键记忆点 |
|------|--------------|-----------|
| 协议注册 | `icmp_protocol` / `inet_add_protocol` | `protocol=1`，处理器 `icmp_rcv` |
| 收包入口 | `icmp_rcv()` | 校验 checksum，分派 `icmp_pointers[]` |
| Echo Reply | `ping_rcv()` | 按 `(net, id)` 查找 ping socket |
| Echo Request | `icmp_echo()` | 改 type 为 0，原样回显 payload |
| 差错处理 | `icmp_unreach()` | 提取 MTU/偏移，调用 `icmp_socket_deliver` |
| 错误发送 | `__icmp_send()` | 大量安全检查，防止循环和广播风暴 |
| 回复发送 | `icmp_reply()` | 处理 IP 选项，路由查找 |
| 速率限制 | `icmp_global_allow()` / `icmpv4_xrlim_allow()` | 全局令牌桶 + 每主机令牌桶；Frag Needed 不限速 |
| 初始化 | `icmp_init()` | 每 CPU RAW socket，namespace sysctl |

ICMP 是 IP 网络中“带外控制面”的核心：ping 用它探测可达性， traceroute 用它发现路径，PMTU 发现用它避免分片，路由器和主机用它报告各种错误。Linux 内核对 ICMP 的处理既遵循 RFC，又通过严格的速率限制和过滤规则防止被滥用。
