# TCP 协议详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/tcp.c` —— TCP 通用层与 socket 接口
> - `net/ipv4/tcp_ipv4.c` —— IPv4 TCP 协议入口、连接管理
> - `net/ipv4/tcp_input.c` —— 接收路径、ACK 处理、拥塞控制
> - `net/ipv4/tcp_output.c` —— 发送路径、报文构造、重传
> - `net/ipv4/tcp_timer.c` —— RTO、Delayed ACK、Keepalive 等定时器
> - `include/linux/tcp.h` —— `struct tcp_sock`、`struct tcphdr`
> - `include/net/tcp.h` —— TCP 常量、宏、函数声明
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图和时序图使用 `text` 代码块表示。

---

## 1. TCP 协议基础

作用：在不可靠的 IP 网络之上提供可靠的、面向连接的、基于字节流的全双工传输服务。

核心 RFC：RFC 793（原始规范）、RFC 1122、RFC 5681（拥塞控制）、RFC 6298（RTO）、RFC 7323（时间戳与窗口缩放）、RFC 2018（SACK）。

典型交互：

```text
客户端                              服务端
  │                                  │
  │──────────── SYN  seq=x ─────────>│
  │                                  │
  │<────── SYN/ACK  seq=y, ack=x+1 ──│
  │                                  │
  │──────────── ACK  ack=y+1 ───────>│
  │                                  │
  │──────────── DATA seq=x+1 ───────>│
  │                                  │
  │<──────────────── ACK ────────────│
```

与 ARP/UDP 的本质不同：
- ARP 是无连接、无状态的地址解析协议；Linux 把它挂在 neighbour 子系统下。
- UDP 不保证可靠，不维护连接状态。
- TCP 需要维护连接状态机、序列号空间、滑动窗口、拥塞控制状态，并通过定时器驱动重传。Linux 用 `struct tcp_sock` 承载所有这些状态。

---

## 2. TCP 报文格式

### 2.1 TCP 首部

```c
struct tcphdr {
	__be16	source;
	__be16	dest;
	__be32	seq;
	__be32	ack_seq;
#if defined(__LITTLE_ENDIAN_BITFIELD)
	__u16	ae:1,
		res1:3,
		doff:4,
		fin:1,
		syn:1,
		rst:1,
		psh:1,
		ack:1,
		urg:1,
		ece:1,
		cwr:1;
#elif defined(__BIG_ENDIAN_BITFIELD)
	__u16	doff:4,
		res1:3,
		ae:1,
		cwr:1,
		ece:1,
		urg:1,
		ack:1,
		psh:1,
		rst:1,
		syn:1,
		fin:1;
#else
#error	"Adjust your <asm/byteorder.h> defines"
#endif
	__be16	window;
	__sum16	check;
	__be16	urg_ptr;
};
```

首部之后最多 40 字节的选项区，因此 `doff`（数据偏移）最小是 5（20 字节），最大是 15（60 字节）。

### 2.2 关键字段速查

| 字段 | 长度 | 含义 |
|------|------|------|
| `source/dest` | 16 bit ×2 | 源/目的端口 |
| `seq` | 32 bit | 本报文发送的第一个字节的序列号 |
| `ack_seq` | 32 bit | 期望收到的下一个字节序号（仅 ACK=1 时有效） |
| `doff` | 4 bit | TCP 首部长度，单位为 4 字节 |
| `SYN/FIN/RST/ACK/PSH/URG` | 各 1 bit | 控制位 |
| `window` | 16 bit | 接收窗口（未缩放时），通告还能收多少字节 |
| `check` | 16 bit | 校验和（覆盖伪首部 + TCP 首部 + 数据） |
| `urg_ptr` | 16 bit | 紧急指针，紧急数据偏移 |

### 2.3 常见选项

```c
#define TCPOPT_NOP		1	/* Padding */
#define TCPOPT_EOL		0	/* End of options */
#define TCPOPT_MSS		2	/* Segment size negotiating */
#define TCPOPT_WINDOW		3	/* Window scaling */
#define TCPOPT_SACK_PERM        4       /* SACK Permitted */
#define TCPOPT_SACK             5       /* SACK Block */
#define TCPOPT_TIMESTAMP	8	/* Better RTT estimations/PAWS */
#define TCPOPT_FASTOPEN		34	/* Fast open (RFC7413) */
```

- MSS（选项 2）：连接建立时协商最大报文段长度。
- Window Scale（选项 3）：把 16 bit 窗口左移，支持大于 64 KiB 的接收窗口。
- SACK（选项 4/5）：选择性确认，提高效率。
- Timestamp（选项 8）：用于 RTTM（往返时间测量）和 PAWS（防止序号回绕）。
- Fast Open（选项 34）：SYN 携带数据，减少一次 RTT。

---

## 3. Linux TCP 实现总览

### 3.1 文件与结构对应表

| 项目 | 文件/结构 |
|------|-----------|
| TCP 通用协议实例 | `struct proto tcp_prot`（`net/ipv4/tcp.c`） |
| IPv4 协议入口 | `tcp_v4_rcv()`（`net/ipv4/tcp_ipv4.c`） |
| 连接四元组查找 | `__inet_lookup_skb()`（`net/ipv4/inet_hashtables.c`） |
| 监听 SYN 处理 | `tcp_v4_conn_request() → tcp_conn_request()` |
| 主动建连 | `tcp_v4_connect() → tcp_connect()` |
| 接收状态机 | `tcp_rcv_state_process()`（`tcp_input.c`） |
| 已连接态快速路径 | `tcp_rcv_established()`（`tcp_input.c`） |
| 发送主路径 | `tcp_sendmsg() → tcp_write_xmit() → tcp_transmit_skb()` |
| 重传队列 | `sk->tcp_rtx_queue`（红黑树） |
| 发送写队列（未发送数据） | `sk->sk_write_queue` |
| 接收缓冲/乱序队列 | `sk->sk_receive_queue` / `tp->out_of_order_queue` |
| 拥塞控制算法 | `struct tcp_congestion_ops`（`include/net/tcp.h`），算法注册于 `net/ipv4/tcp_cong.c` |

### 3.2 核心数据结构：`struct tcp_sock`

`tcp_sock` 是 TCP 连接的全部状态载体，继承自 `inet_connection_sock`，而后者又继承自 `inet_sock` / `sock`。源码注释很形象地把 RFC 793 中的变量名直接写了出来：

```c
struct tcp_sock {
	/* inet_connection_sock has to be the first member of tcp_sock */
	struct inet_connection_sock	inet_conn;

	/* ... 省略缓存行对齐分组 ... */

	u32	snd_wnd;	/* The window we expect to receive	*/
	u32	snd_cwnd;	/* Sending congestion window		*/
	u32	rcv_nxt;	/* What we want to receive next		*/
	u32	snd_nxt;	/* Next sequence we send		*/
	u32	snd_una;	/* First byte we want an ack for	*/
	u32	rcv_wnd;	/* Current receiver window		*/
	u32	srtt_us;	/* smoothed round trip time << 3 in usecs */
	u32	packets_out;	/* Packets which are "in flight"	*/

	/* ... */
	struct tcp_options_received rx_opt;

	/* OOO segments go in this rbtree. Socket lock must be held. */
	struct rb_root	out_of_order_queue;

	/* SACKs data */
	struct tcp_sack_block duplicate_sack[1]; /* D-SACK block */
	struct tcp_sack_block selective_acks[4]; /* The SACKS themselves*/

	/* Slow start and congestion control */
	u32	snd_cwnd_cnt;	/* Linear increase counter		*/
	u32	snd_cwnd_clamp; /* Do not allow snd_cwnd to grow above this */
	u32	snd_ssthresh;	/* Slow start size threshold		*/
	u32	prior_cwnd;	/* cwnd right before starting loss recovery */
	u32	high_seq;	/* snd_nxt at onset of congestion	*/

	/* ... */
};
```

关键字段说明：

| 字段 | 含义 |
|------|------|
| `snd_una` | 已发送但未确认的最小序号 |
| `snd_nxt` | 下一个要发送的字节序号 |
| `write_seq` | 发送缓冲区的尾部（用户已写入但不一定已发送） |
| `rcv_nxt` | 期望收到的下一个字节序号 |
| `copied_seq` | 用户已经 `recv()` 读走的字节序号 |
| `snd_wnd` | 对端通告的接收窗口 |
| `snd_cwnd` | 拥塞窗口，发送方自己限制 |
| `rcv_wnd` | 本端通告给对端的接收窗口 |
| `srtt_us` | 平滑 RTT（左移 3 位，即 ×8） |

---

## 4. TCP 状态机

Linux 用 `enum tcp_state` 表示 RFC 793 状态：

```c
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
	TCP_CLOSING,	/* Now a valid state */
	TCP_NEW_SYN_RECV,

	TCP_MAX_STATES	/* Leave at the end! */
};
```

### 4.1 状态转换图

```text
                               +-----------+
                               |  CLOSED   |
                               +-----+-----+
                                     │ 主动 open / connect()
                                     ▼
                               +-----------+
               收到 SYN+ACK    │ SYN_SENT  │ 发送 SYN
               发送 ACK        +-----+-----+
                     │               │ 同时打开：收到 SYN 并发送 SYN+ACK
                     ▼               ▼
              +------------+   +------------+
              │ ESTABLISHED│   │ SYN_RECV   │
              +-----+------+   +-----+------+
                    │                │ 收到 ACK
                    │                ▼
                    │          +------------+
                    │          │ ESTABLISHED│
                    │          +-----+------+
                    │                │
          应用 close │                │ 对端 FIN
                    ▼                ▼
             +------------+    +------------+
             │ FIN_WAIT1  │    │ CLOSE_WAIT │ 应用 close
             +-----+------+    +-----+------+
                   │                 │
       收到 ACK    │                 ▼
         +---------+          +------------+
         │                    │ LAST_ACK   │
         ▼                    +-----+------+
    +------------+                  │ 收到 ACK
    │ FIN_WAIT2  │                  ▼
    +-----+------+            +-----------+
          │                   │  CLOSED   │
   收到 FIN│                   +-----------+
          ▼
    +------------+    同时关闭：收到 FIN，发送 ACK
    │ TIME_WAIT  │ ◄──────── +------------+
    +-----+------+           │  CLOSING   │
          │                  +-----+------+
          │                        │ 收到 ACK
          ▼                        │
    +-----------+                  │
    │  CLOSED   │                  │
    +-----------+                  │
```

> 同时关闭（Simultaneous Close）：两端几乎同时发送 FIN，FIN_WAIT1 收到对端 FIN 后进入 `CLOSING`，收到最后的 ACK 后进入 `TIME_WAIT`。

### 4.2 状态与处理函数对应

在 `tcp_v4_do_rcv()` 中，已建立连接走快速路径，其他状态进入状态机：

```c
int tcp_v4_do_rcv(struct sock *sk, struct sk_buff *skb)
{
	enum skb_drop_reason reason;

	reason = psp_sk_rx_policy_check(sk, skb);
	if (reason)
		goto err_discard;

	if (sk->sk_state == TCP_ESTABLISHED) { /* Fast path */
		struct dst_entry *dst;

		dst = rcu_dereference_protected(sk->sk_rx_dst,
						lockdep_sock_is_held(sk));

		sock_rps_save_rxhash(sk, skb);
		sk_mark_napi_id(sk, skb);
		if (dst && unlikely(dst != skb_dst(skb))) {
			if (sk->sk_rx_dst_ifindex != skb->skb_iif ||
			    !INDIRECT_CALL_1(dst->ops->check, ipv4_dst_check,
					     dst, 0)) {
				RCU_INIT_POINTER(sk->sk_rx_dst, NULL);
				dst_release(dst);
			}
		}
		tcp_rcv_established(sk, skb);
		return 0;
	}

	if (tcp_checksum_complete(skb))
		goto csum_err;

	if (sk->sk_state == TCP_LISTEN) {
		struct sock *nsk = tcp_v4_cookie_check(sk, skb);

		if (!nsk)
			return 0;
		if (nsk != sk) {
			reason = tcp_child_process(sk, nsk, skb);
			sock_put(nsk);
			if (reason)
				goto reset;
			return 0;
		}
	} else
		sock_rps_save_rxhash(sk, skb);

	reason = tcp_rcv_state_process(sk, skb);
	if (reason)
		goto reset;
	return 0;

reset:
	tcp_v4_send_reset(sk, skb, sk_rst_convert_drop_reason(reason));
discard:
	sk_skb_reason_drop(sk, skb, reason);
	return 0;

	/* 错误处理 ... */
}
```

- `ESTABLISHED` → `tcp_rcv_established(sk, skb)`，快速路径（v7.1-rc7 起精简为两参数）。
- `LISTEN` → `tcp_v4_cookie_check()` 处理 SYN 或交给子 socket。
- 其他 → `tcp_rcv_state_process()`，通用状态机。

---

## 5. 连接建立（三次握手）

### 5.1 主动打开：从 `connect()` 到 SYN

应用调用 `connect()` 后，沿以下路径进入内核：

```text
sys_connect()
  → inet_stream_connect()
    → tcp_v4_connect()          // net/ipv4/tcp_ipv4.c
      → ip_route_connect()      // 查路由、选源 IP
      → tcp_set_state(sk, TCP_SYN_SENT)
      → inet_hash_connect()     // 分配源端口、加入 hash
      → secure_tcp_seq_and_ts_off() // 生成初始序列号 ISN 与时间戳偏移
      → tcp_connect(sk)         // net/ipv4/tcp_output.c
        → tcp_connect_init(sk)  // 初始化 MSS、窗口、RTO、拥控等
        → tcp_init_nondata_skb(buff, ..., TCPHDR_SYN)
        → tcp_transmit_skb(sk, buff, 1, ...)   // 发送 SYN
        → tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS, icsk_rto, false)
```

`tcp_v4_connect()` 的关键代码：

```c
int tcp_v4_connect(struct sock *sk, struct sockaddr_unsized *uaddr, int addr_len)
{
	struct sockaddr_in *usin = (struct sockaddr_in *)uaddr;
	struct inet_sock *inet = inet_sk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	__be32 daddr, nexthop;
	struct rtable *rt;
	/* ... */

	nexthop = daddr = usin->sin_addr.s_addr;
	rt = ip_route_connect(fl4, nexthop, inet->inet_saddr,
			      sk->sk_bound_dev_if, IPPROTO_TCP, orig_sport,
			      orig_dport, sk);
	if (IS_ERR(rt)) {
		err = PTR_ERR(rt);
		if (err == -ENETUNREACH)
			IP_INC_STATS(net, IPSTATS_MIB_OUTNOROUTES);
		return err;
	}

	/* ... */

	tcp_set_state(sk, TCP_SYN_SENT);
	err = inet_hash_connect(tcp_death_row, sk);
	if (err)
		goto failure;

	/* ... */

	if (likely(!tp->repair)) {
		union tcp_seq_and_ts_off st;

		st = secure_tcp_seq_and_ts_off(net,
					       inet->inet_saddr,
					       inet->inet_daddr,
					       inet->inet_sport,
					       usin->sin_port);
		if (!tp->write_seq)
			WRITE_ONCE(tp->write_seq, st.seq);
		WRITE_ONCE(tp->tsoffset, st.ts_off);
	}

	/* ... */

	err = tcp_connect(sk);
	/* ... */
}
```

`tcp_connect()` 构造并发送 SYN：

```c
int tcp_connect(struct sock *sk)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct sk_buff *buff;
	int err;

	tcp_connect_init(sk);

	buff = tcp_stream_alloc_skb(sk, sk->sk_allocation, true);
	if (unlikely(!buff))
		return -ENOBUFS;

	tcp_init_nondata_skb(buff, sk, tp->write_seq, TCPHDR_SYN);
	tcp_mstamp_refresh(tp);
	tp->retrans_stamp = tcp_time_stamp_ts(tp);
	tcp_connect_queue_skb(sk, buff);
	tcp_ecn_send_syn(sk, buff);
	tcp_rbtree_insert(&sk->tcp_rtx_queue, buff);

	/* Send off SYN; include data in Fast Open. */
	err = tp->fastopen_req ? tcp_send_syn_data(sk, buff) :
	      tcp_transmit_skb(sk, buff, 1, sk->sk_allocation);

	WRITE_ONCE(tp->snd_nxt, tp->write_seq);
	tp->pushed_seq = tp->write_seq;

	TCP_INC_STATS(sock_net(sk), TCP_MIB_ACTIVEOPENS);

	/* Timer for repeating the SYN until an answer. */
	tcp_reset_xmit_timer(sk, ICSK_TIME_RETRANS,
			     inet_csk(sk)->icsk_rto, false);
	return 0;
}
```

注意：
- `tcp_init_nondata_skb()` 把 `TCP_SKB_CB(skb)->seq` 设为 `tp->write_seq`，并把 `end_seq` 设为 `seq + 1`，因为 SYN 占用一个序列号。
- SYN 被插入 `tcp_rtx_queue`，后续若未收到 SYN+ACK 会由 RTO 定时器重传。
- 初始 RTO 由 `TCP_TIMEOUT_INIT` 定义，v7.1-rc7 中仍为 1 秒：

```c
#define TCP_TIMEOUT_INIT ((unsigned)(1*HZ))	/* RFC6298 2.1 initial RTO value */
```

### 5.2 被动打开：从 `listen()` 到 SYN+ACK

服务端调用 `listen()` 后，socket 进入 `TCP_LISTEN`。收到 SYN 时，内核在软中断中处理：

```text
网卡驱动
  → __netif_receive_skb_core()
    → ip_rcv() → ip_rcv_finish() → ip_local_deliver()
      → tcp_v4_rcv()
        → __inet_lookup_skb()        // 四元组查找
          → 命中 LISTEN socket
        → tcp_v4_do_rcv()
          → tcp_v4_cookie_check(sk, skb)  // LISTEN 状态：SYN 直接返回 sk
          → tcp_rcv_state_process(sk, skb)
            → icsk->icsk_af_ops->conn_request(sk, skb)  // 即 tcp_v4_conn_request()
              → tcp_conn_request()     // net/ipv4/tcp_input.c
                → 解析选项（MSS/WS/SACK/Timestamp/ECN/FastOpen）
                → 分配 request_sock
                → 生成 ISN
                → tcp_send_synack()    // 发送 SYN+ACK
                → 把 req 加入半连接队列（syn queue）
```

`tcp_conn_request()` 是被动打开的核心：

```c
int tcp_conn_request(struct request_sock_ops *rsk_ops,
		     const struct tcp_request_sock_ops *af_ops,
		     struct sock *sk, struct sk_buff *skb)
{
	struct tcp_fastopen_cookie foc = { .len = -1 };
	u32 isn = TCP_SKB_CB(skb)->tcp_tw_isn;
	struct tcp_options_received tmp_opt;
	const struct tcp_sock *tp = tcp_sk(sk);
	struct net *net = sock_net(sk);
	struct request_sock *req;
	bool want_cookie = false;
	/* ... */

	if (!isn) {
		syncookies = READ_ONCE(net->ipv4.sysctl_tcp_syncookies);

		if (syncookies == 2 || inet_csk_reqsk_queue_is_full(sk)) {
			want_cookie = tcp_syn_flood_action(sk,
						   rsk_ops->slab_name);
			if (!want_cookie)
				goto drop;
		}
	}

	if (sk_acceptq_is_full(sk))
		goto drop;

	req = inet_reqsk_alloc(rsk_ops, sk, !want_cookie);
	if (!req)
		goto drop;

	tcp_clear_options(&tmp_opt);
	tmp_opt.mss_clamp = af_ops->mss_clamp;
	tmp_opt.user_mss  = READ_ONCE(tp->rx_opt.user_mss);
	tcp_parse_options(sock_net(sk), skb, &tmp_opt, 0,
			  want_cookie ? NULL : &foc);

	/* ... */
	tcp_openreq_init(req, &tmp_opt, skb, sk);

	dst = af_ops->route_req(sk, skb, &fl, req, isn);
	/* ... */

	tcp_rsk(req)->snt_isn = isn;
	tcp_openreq_init_rwin(req, sk, dst);
	/* ... */

	if (!want_cookie) {
		tcp_reqsk_record_syn(sk, req, skb);
		fastopen_sk = tcp_try_fastopen(sk, skb, req, &foc, dst);
	}
	if (fastopen_sk) {
		af_ops->send_synack(fastopen_sk, dst, &fl, req,
				    &foc, TCP_SYNACK_FASTOPEN, skb);
		/* Add the child socket directly into the accept queue */
		/* ... */
	} else {
		tcp_rsk(req)->tfo_listener = false;
		if (!want_cookie &&
		    unlikely(!inet_csk_reqsk_queue_hash_add(sk, req))) {
			reqsk_free(req);
			dst_release(dst);
			return 0;
		}
		af_ops->send_synack(sk, dst, &fl, req, &foc,
				    !want_cookie ? TCP_SYNACK_NORMAL :
						   TCP_SYNACK_COOKIE,
				    skb);
		/* ... */
	}
	/* ... */
}
```

重点：
- 半连接队列（syn queue）：存放 `request_sock`，由 `tcp_max_syn_backlog` 控制。
- 全连接队列（accept queue）：存放已建立连接的子 socket，由 `listen()` 的 `backlog` 和 `somaxconn` 控制。
- Syncookies：当半连接队列满时，Linux 可启用 syncookie，不保留 `request_sock`，只把状态编码到 SYN+ACK 的 ISN 中，收到 ACK 时再解码验证。
- Fast Open：若客户端之前获得过 cookie，SYN 可携带数据，服务端验证后直接创建子 socket 并放入 accept queue。

### 5.3 第三次握手：ACK 到达

客户端收到 SYN+ACK 后，从 `tcp_v4_rcv()` 查到 `TCP_SYN_SENT` 状态的 socket：

```text
tcp_v4_rcv()
  → __inet_lookup_skb()
  → tcp_v4_do_rcv(sk, skb)
    → tcp_rcv_state_process(sk, skb)   // 因为 sk_state == TCP_SYN_SENT
      → 检查 ack_seq 是否合法
      → 处理窗口缩放、SACK、时间戳选项
      → tcp_finish_connect(sk, skb)
        → tcp_set_state(sk, TCP_ESTABLISHED)
        → 唤醒阻塞在 connect() 上的进程
        → 发送 ACK（如果有数据待发则随数据一起发）
```

服务端收到第三次 ACK 后：

```text
tcp_v4_rcv()
  → __inet_lookup_skb() 命中 TCP_NEW_SYN_RECV 状态的 request_sock
  → tcp_check_req(sk, skb, req, ...)
    → 如果 ACK 合法，创建子 socket（tcp_v4_syn_recv_sock）
    → tcp_child_process(listener, child, skb)
      → tcp_v4_do_rcv(child, skb)
        → tcp_rcv_state_process(child, skb) 把子 socket 置为 ESTABLISHED
      → 把 child 加入 accept queue
      → 唤醒 accept() 等待的进程
```

---

## 6. 数据发送流程

### 6.1 应用写数据：`tcp_sendmsg()`

```text
sys_sendmsg() / sys_write()
  → sock_sendmsg()
    → inet_sendmsg()
      → tcp_sendmsg(sk, msg, size)
        → tcp_sendmsg_locked(sk, msg, size)
          → 把用户数据拷贝到 sk_buff（可能合并到现有 skb）
          → 如果满足发送条件，调用 tcp_push()
            → __tcp_push_pending_frames(sk, mss_now, nonagle)
              → tcp_write_xmit(sk, mss_now, nonagle, ...)
                → tcp_transmit_skb(sk, skb, 1, gfp)
                  → 构造 IP + TCP 首部、计算校验和
                  → ip_queue_xmit() / ip6_xmit()
```

`tcp_sendmsg_locked()` 的简略逻辑：

```c
int tcp_sendmsg_locked(struct sock *sk, struct msghdr *msg, size_t size)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct ubuf_info *uarg = NULL;
	/* ... */

	flags = msg->msg_flags;

	/* ... 拷贝用户数据到 sk->sk_write_queue 中的 skb ... */

	tcp_push(sk, flags & ~MSG_MORE, mss_now,
		 tcp_sk(sk)->nonagle, size_goal);
	return copied;
}
```

Nagle 算法：
- 如果 `TCP_NODELAY` 未设置（默认启用 Nagle），且存在已发送但未确认的小包，当前数据不会立即发送，而是等待 ACK 到来或凑够一个 MSS。
- `tp->nonagle` 字段控制：

```c
#define TCP_NAGLE_OFF		1	/* Nagle's algo is disabled */
#define TCP_NAGLE_CORK		2	/* Socket is corked	    */
#define TCP_NAGLE_PUSH		4	/* Cork is overridden for already queued data */
```

### 6.2 发送引擎：`tcp_write_xmit()`

`tcp_write_xmit()` 决定当前能不能发、发多少：

```c
static bool tcp_write_xmit(struct sock *sk, unsigned int mss_now, int nonagle,
			   int push_one, gfp_t gfp)
{
	struct tcp_sock *tp = tcp_sk(sk);
	struct sk_buff *skb;
	unsigned int tso_segs, sent_pkts;
	int cwnd_quota;
	int result;
	bool is_cwnd_limited = false;

	sent_pkts = 0;

	tcp_mstamp_refresh(tp);
	if (!push_one) {
		/* Do MTU probing. */
		result = tcp_mtu_probe(sk);
		if (!result) {
			return false;
		} else if (result > 0) {
			sent_pkts = 1;
		}
	}

	while ((skb = tcp_send_head(sk))) {
		unsigned int limit;

		tso_segs = tcp_init_tso_segs(skb, mss_now);
		if (tso_segs < 0)
			return false;

		cwnd_quota = tcp_cwnd_test(tp, skb);
		if (!cwnd_quota) {
			is_cwnd_limited = true;
			break;
		}

		if (unlikely(!tcp_wnd_end(tp)) ||
		    !tcp_pacing_check(sk, skb, nonagle, &cwnd_quota)) {
			is_cwnd_limited = true;
			break;
		}

		limit = mss_now;
		if (tso_segs > 1 && !sk->sk_gso_disabled)
			limit = tcp_mss_split_point(sk, skb, mss_now,
						    min_t(unsigned int,
							  cwnd_quota,
							  sk->sk_gso_max_segs));

		if (skb->len > limit &&
		    unlikely(tso_fragment(sk, skb, limit, mss_now, gfp)))
			break;

		if (tcp_small_queue_check(sk, skb, 0))
			break;

		if (unlikely(tcp_transmit_skb(sk, skb, 1, gfp)))
			break;

		tcp_event_new_data_sent(sk, skb);

		tcp_minshall_update(tp, mss_now, skb);
		if (tcp_in_cwnd_reduction(sk))
			tp->prr_out += tcp_skb_pcount(skb);

		if (!push_one)
			tcp_pacing_check_after_send(sk, skb, &cwnd_quota);

		sent_pkts += tcp_skb_pcount(skb);

		if (inet_csk(sk)->icsk_pending == ICSK_TIME_LOSS_PROBE)
			tp->tlp_high_seq = tp->snd_nxt;

		if (push_one)
			break;
	}

	if (is_cwnd_limited)
		tp->is_cwnd_limited = true;

	if (likely(sent_pkts)) {
		tcp_cwnd_validate(sk, is_cwnd_limited);
		return false;
	}
	return !tp->packets_out && tcp_send_head(sk);
}
```

核心约束（发送窗口取最小值）：

```text
本次能发的字节数 = min(对端通告窗口 snd_wnd, 拥塞窗口 snd_cwnd, TSO/GSO 限制, pacing 限制)
```

`tcp_event_new_data_sent()` 把 skb 从 `sk_write_queue` 移到 `tcp_rtx_queue`（红黑树），更新 `snd_nxt` 和 `packets_out`：

```c
static void tcp_event_new_data_sent(struct sock *sk, struct sk_buff *skb)
{
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	unsigned int prior_packets = tp->packets_out;

	WRITE_ONCE(tp->snd_nxt, TCP_SKB_CB(skb)->end_seq);

	__skb_unlink(skb, &sk->sk_write_queue);
	tcp_rbtree_insert(&sk->tcp_rtx_queue, skb);

	if (tp->highest_sack == NULL)
		tp->highest_sack = skb;

	tp->packets_out += tcp_skb_pcount(skb);
	if (!prior_packets || icsk->icsk_pending == ICSK_TIME_LOSS_PROBE)
		tcp_rearm_rto(sk);

	NET_ADD_STATS(sock_net(sk), LINUX_MIB_TCPORIGDATASENT,
		      tcp_skb_pcount(skb));
	tcp_check_space(sk);
}
```

### 6.3 报文构造：`tcp_transmit_skb()`

`tcp_transmit_skb()` 负责填充 TCP 首部、选项、校验和，并交给 IP 层：

```c
static int tcp_transmit_skb(struct sock *sk, struct sk_buff *skb, int clone_it,
			    gfp_t gfp_mask)
{
	const struct inet_connection_sock *icsk = inet_csk(sk);
	struct inet_sock *inet;
	struct tcp_sock *tp = tcp_sk(sk);
	struct tcp_skb_cb *tcb;
	struct tcphdr *th;
	int err;

	/* ... */

	tcb = TCP_SKB_CB(skb);
	memset(&opts, 0, sizeof(opts));

	if (unlikely(tcb->tcp_flags & TCPHDR_SYN))
		tcp_options_size = tcp_syn_options(sk, skb, &opts, &md5);
	else
		tcp_options_size = tcp_established_options(sk, skb, &opts,
							   &md5);
	tcp_header_size = tcp_options_size + sizeof(struct tcphdr);

	/* ... */

	th = (struct tcphdr *)skb->data;
	th->source	= inet->inet_sport;
	th->dest	= inet->inet_dport;
	th->seq		= htonl(tcb->seq);
	th->ack_seq	= htonl(tp->rcv_nxt);
	*(((__be16 *)th) + 6)	= htons(((tcp_header_size >> 2) << 12) | tcb->tcp_flags);
	th->window	= htons(min(tp->rcv_wnd, 65535U));
	th->check	= 0;
	th->urg_ptr	= 0;

	/* ... */
	tcp_options_write((__be32 *)(th + 1), tp, &opts);

	/* ... */

	err = icsk->icsk_af_ops->queue_xmit(sk, skb, &inet->cork.fl);
	/* ... */
}
```

实际源码中 `tcp_transmit_skb()` 较长，核心就是：
1. 根据 SYN 或已连接态选择需要携带的选项。
2. 填充 `seq`、`ack_seq`、`window`、控制位。
3. 调用 `tcp_options_write()` 写入选项。
4. 通过 `icsk_af_ops->queue_xmit`（IPv4 下是 `ip_queue_xmit`）交给 IP 层。

---

## 7. 数据接收流程

### 7.1 入口：`tcp_v4_rcv()`

IPv4 TCP 报文到达本机后，由 `tcp_v4_rcv()` 处理：

```c
int tcp_v4_rcv(struct sk_buff *skb)
{
	struct net *net = dev_net_rcu(skb->dev);
	const struct iphdr *iph;
	const struct tcphdr *th;
	struct sock *sk = NULL;
	bool refcounted;
	int ret;

	if (skb->pkt_type != PACKET_HOST)
		goto discard_it;

	__TCP_INC_STATS(net, TCP_MIB_INSEGS);

	if (!pskb_may_pull(skb, sizeof(struct tcphdr)))
		goto discard_it;

	th = (const struct tcphdr *)skb->data;

	if (unlikely(th->doff < sizeof(struct tcphdr) / 4)) {
		drop_reason = SKB_DROP_REASON_PKT_TOO_SMALL;
		goto bad_packet;
	}
	if (!pskb_may_pull(skb, th->doff * 4))
		goto discard_it;

	if (skb_checksum_init(skb, IPPROTO_TCP, inet_compute_pseudo))
		goto csum_error;

	th = (const struct tcphdr *)skb->data;
	iph = ip_hdr(skb);
lookup:
	sk = __inet_lookup_skb(skb, __tcp_hdrlen(th), th->source,
			       th->dest, sdif, &refcounted);
	if (!sk)
		goto no_tcp_socket;

	if (sk->sk_state == TCP_TIME_WAIT)
		goto do_time_wait;

	if (sk->sk_state == TCP_NEW_SYN_RECV) {
		/* 第三次握手或同时 open，交给 tcp_check_req() */
		/* ... */
	}

process:
	/* XFRM、BPF 过滤 ... */

	if (sk->sk_state == TCP_LISTEN) {
		ret = tcp_v4_do_rcv(sk, skb);
		goto put_and_return;
	}

	sk_incoming_cpu_update(sk);

	bh_lock_sock_nested(sk);
	tcp_segs_in(tcp_sk(sk), skb);
	ret = 0;
	if (!sock_owned_by_user(sk)) {
		ret = tcp_v4_do_rcv(sk, skb);
	} else {
		drop_reason = tcp_add_backlog(sk, skb);
		if (drop_reason)
			goto discard_and_relse;
	}
	bh_unlock_sock(sk);

put_and_return:
	if (refcounted)
		sock_put(sk);
	return ret;

	/* 错误处理 ... */
}
```

关键步骤：
1. 校验报文长度、`doff`、校验和。
2. `__inet_lookup_skb()` 根据 `(src_ip, src_port, dst_ip, dst_port)` 查找 socket。
3. 如果 socket 正在被用户态持有，skb 入 `sk_backlog`，等释放锁后再处理。
4. 否则直接 `tcp_v4_do_rcv()`。

### 7.2 已连接态快速路径：`tcp_rcv_established()`

```text
tcp_v4_do_rcv()
  → tcp_rcv_established(sk, skb)
    → 头部预测（Header Prediction）
      ├─ 快速路径：期望的下一个报文、ACK 合法、无乱序、无 SACK 事件
      │   → tcp_queue_rcv() 直接追加到接收队列
      │   → 必要时触发 delayed ACK
      └─ 慢速路径：进入 tcp_ack() + tcp_data_queue()
```

`tcp_rcv_established()` 是 TCP 接收最热的函数，源码里用 `tcp_flag_word(th) & TCP_HP_BITS` 等做快速判断。如果满足条件，直接调用 `tcp_queue_rcv()`；否则走完整路径处理 ACK、乱序、SACK 等。

### 7.3 乱序与 SACK 处理

当收到的报文序号不等于 `rcv_nxt` 时，数据被放入乱序红黑树：

```c
struct tcp_sock {
	/* OOO segments go in this rbtree. Socket lock must be held. */
	struct rb_root	out_of_order_queue;
};
```

乱序 skb 按序列号排序，等待中间缺段到达后再合并到有序接收队列。

如果启用了 SACK，接收方在 ACK 中携带 SACK block，告诉发送方已经收到了哪些不连续的段。发送方在 `tcp_input.c` 的 SACK 处理逻辑中把已 SACK 的段标记为 `TCPCB_SACKED_ACKED`，避免不必要的重传。

### 7.4 ACK 与窗口更新：`tcp_ack()`

```text
tcp_rcv_established() 慢速路径
  → tcp_ack(sk, skb, FLAG_SLOWPATH)
    → 检查 ack_seq 是否合法（不能确认还没发的数据）
    → 如果 ack_seq > snd_una：
        → 更新 snd_una
        → 从重传队列中移除已被确认的 skb
        → 更新 RTT 样本（如果有时间戳或未被重传过的段）
        → 调用 tcp_cong_avoid() / tcp_fastretrans_alert()
    → 处理窗口更新（snd_wnd）
```

`tcp_ack()` 是拥塞控制、快速重传、快速恢复的核心触发点。

---

## 8. 拥塞控制

### 8.1 核心变量

| 变量 | 含义 |
|------|------|
| `snd_cwnd` | 拥塞窗口，当前网络允许发送的段数 |
| `snd_ssthresh` | 慢启动阈值，cwnd 超过它后进入拥塞避免 |
| `snd_cwnd_cnt` | 拥塞避免阶段的线性增长计数器 |
| `packets_out` | 已发送但未确认的段数（在途） |
| `sacked_out` | 被 SACK 确认的段数 |
| `lost_out` | 被判定为丢失的段数 |
| `retrans_out` | 已重传的段数 |

### 8.2 拥塞状态机

Linux TCP 的拥塞控制抽象成 `struct tcp_congestion_ops`：

```c
struct tcp_congestion_ops {
	u32	flags;
	/* initialize private data (optional) */
	void (*init)(struct sock *sk);
	/* clean up private data  (optional) */
	void (*release)(struct sock *sk);

	/* return slow start threshold (required) */
	u32 (*ssthresh)(struct sock *sk);
	/* do new cwnd calculation (required) */
	void (*cong_avoid)(struct sock *sk, u32 ack, u32 acked);
	/* call when ack arrives (optional) */
	void (*pkts_acked)(struct sock *sk, const struct ack_sample *sample);
	/* ... */
	const char name[TCP_CA_NAME_MAX];
};
```

常见算法：
- `reno`：基础兜底算法，其他算法未加载时用它兜底（`tcp_cong.c` 中作为 fallback 注册）。
- `cubic`：Linux 默认拥塞控制算法（`CONFIG_DEFAULT_TCP_CONG` 默认值为 `"cubic"`）。
- `bbr`、`bbr2`：基于带宽和 RTT 建模的算法。

### 8.3 慢启动与拥塞避免

慢启动（Slow Start）：
- 初始 `snd_cwnd` 通常为 10 MSS（`TCP_INIT_CWND = 10`，RFC 6928）。
- 每收到一个 ACK，`snd_cwnd` 增加 1 MSS，指数增长。
- 直到 `snd_cwnd >= snd_ssthresh`，进入拥塞避免。

拥塞避免（Congestion Avoidance）：
- 每收到一个 ACK，`snd_cwnd_cnt` 增加本次 ACK 新确认的段数 `acked`（见 `tcp_cong_avoid_ai()`）。
- 当 `snd_cwnd_cnt >= snd_cwnd` 时，`snd_cwnd++`，并把计数器减去 `snd_cwnd`。
- 效果上每个 RTT 增加约 1 MSS，线性增长。

### 8.4 丢包恢复

Linux 支持多种丢包检测与恢复机制：

1. RTO 超时：
   - 重传 `snd_una` 开始的段。
   - `snd_ssthresh = max(flight / 2, 2)`，`snd_cwnd` 降到 1 MSS，进入慢启动。

2. 快速重传（Fast Retransmit）：
   - 收到 3 个重复 ACK（默认 `TCP_FASTRETRANS_THRESH = 3`），认为包丢失。
   - 进入 `TCP_CA_Recovery`，重传丢失段。
   - `snd_ssthresh = max(flight / 2, 2)`，`snd_cwnd` 调整为 `ssthresh + 3`（3 个段，对应 3 个重复 ACK）。

3. SACK 与 RACK：
   - SACK 提供更精确的重传信息。
   - RACK（Recent ACK）基于时间戳判断哪个段最晚发送且未被确认，从而标记丢失。

4. F-RTO / TLP：
   - F-RTO（Forward RTO）用于识别虚假 RTO。
   - TLP（Tail Loss Probe）在 RTO 到期前发送一个探测包，减少尾丢包导致的超时。

---

## 9. 连接拆除（四次挥手）

### 9.1 主动关闭

```text
主动关闭端                         被动关闭端
   │                                   │
   │──────────── FIN, seq=u ─────────>│  应用 close() 后进入 FIN_WAIT1
   │                                   │
   │<────────── ACK, ack=u+1 ─────────│  进入 CLOSE_WAIT
   │                                   │
   │ 收到 ACK 后进入 FIN_WAIT2         │
   │                                   │
   │<──────────── FIN, seq=w ─────────│  应用 close() 后进入 LAST_ACK
   │                                   │
   │────────── ACK, ack=w+1 ─────────>│  进入 TIME_WAIT
   │                                   │
   │ 等待 2MSL 后进入 CLOSED           │  收到 ACK 后进入 CLOSED
```

### 9.2 `tcp_close()` 与状态推进

应用调用 `close()` 后：

```text
tcp_close()
  → 如果接收缓冲区还有未读数据，发送 RST（符合 RFC）
  → 否则 tcp_close_state(sk)
    → 根据当前状态决定是发 FIN（TCP_ACTION_FIN）还是直接关闭
  → tcp_send_fin(sk)
    → 构造 FIN 段，放入发送队列并触发发送
  → tcp_state_change(sk)
```

`tcp_send_fin()`：

```c
void tcp_send_fin(struct sock *sk)
{
	struct sk_buff *skb, *tskb, *tail = tcp_write_queue_tail(sk);
	struct tcp_sock *tp = tcp_sk(sk);

	sk->sk_shutdown |= SEND_SHUTDOWN;

	/* 优先把 FIN 打在 write queue 尾包上；
	 * 若 write queue 为空但内存紧张，可打在已发送的 rtx_queue 尾包上。
	 */
	tskb = tail;
	if (!tskb && tcp_under_memory_pressure(sk))
		tskb = skb_rb_last(&sk->tcp_rtx_queue);

	if (tskb) {
		TCP_SKB_CB(tskb)->tcp_flags |= TCPHDR_FIN;
		TCP_SKB_CB(tskb)->end_seq++;
		tp->write_seq++;
		if (!tail) {
			/* tskb 已经发出去，只需让 snd_nxt 前进一步，
			 * 让重传路径按带 FIN 的序列号处理。
			 */
			WRITE_ONCE(tp->snd_nxt, tp->snd_nxt + 1);
			return;
		}
	} else {
		/* 发送队列为空，分配一个纯 FIN 包 */
		skb = alloc_skb_fclone(MAX_TCP_HEADER,
				       sk_gfp_mask(sk, GFP_ATOMIC |
						       __GFP_NOWARN));
		if (unlikely(!skb))
			return;

		INIT_LIST_HEAD(&skb->tcp_tsorted_anchor);
		skb_reserve(skb, MAX_TCP_HEADER);
		sk_forced_mem_schedule(sk, skb->truesize);
		tcp_init_nondata_skb(skb, sk, tp->write_seq,
				     TCPHDR_ACK | TCPHDR_FIN);
		tcp_queue_skb(sk, skb);
	}
	__tcp_push_pending_frames(sk, tcp_current_mss(sk), TCP_NAGLE_OFF);
}
```

### 9.3 TIME_WAIT 状态

`TIME_WAIT` 持续 `TCP_TIMEWAIT_LEN`（Linux 选择的值为 60 秒，RFC 793 建议的 2MSL 为 4 分钟，Linux 取了更短的值以减少资源占用）：

```c
#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-WAIT
				  * state, about 60 seconds	*/
```

作用：
1. 防止旧连接的迟到的报文被新连接误收。
2. 保证被动关闭端收到最后的 ACK；若 ACK 丢失，被动端会重发 FIN，TIME_WAIT 端能重发 ACK。

Linux 通过 `tcp_time_wait()` 把 socket 变成 `struct tcp_timewait_sock`，放在单独的 hash 中，避免占用完整的 `tcp_sock`。

---

## 10. 定时器

TCP 使用 `inet_connection_sock` 中的定时器框架：

```c
struct inet_connection_sock {
	/* ... */
	struct timer_list	icsk_retransmit_timer;
	struct timer_list	icsk_delack_timer;
	/* ... */
};
```

`tcp_sock` 还额外有：

```c
struct tcp_sock {
	struct hrtimer	pacing_timer;
	struct hrtimer	compressed_ack_timer;
};
```

### 10.1 RTO 重传定时器

类型：`ICSK_TIME_RETRANS`

- 在发送新数据（且此前无在途数据）或重传时启动/重置。
- 超时后调用 `tcp_retransmit_timer()`。
- RTO 计算遵循 RFC 6298：

```c
#define TCP_TIMEOUT_INIT ((unsigned)(1*HZ))	/* 初始 RTO 1 秒 */
#define TCP_RTO_MIN	((unsigned)(HZ / 5))	/* 最小 200ms */
#define TCP_RTO_MAX	((unsigned)(TCP_RTO_MAX_SEC * HZ)) /* 最大 120 秒 */
```

### 10.2 Delayed ACK 定时器

类型：`ICSK_TIME_DACK`

- 收到数据后不立即 ACK，等待最多 `HZ/5`（默认配置 `HZ=1000` 时为 200ms，`TCP_DELACK_MAX = HZ/5`）。
- 如果期间又有数据要发，ACK 随数据捎带。
- 若对端窗口很小、收到 PSH、进入 quickack 模式等会立即 ACK。

### 10.3 Keepalive 定时器

类型：`ICSK_TIME_KEEPOPEN`

- 连接空闲 `tcp_keepalive_time`（默认 2 小时）后发送探测包。
- 最多 `tcp_keepalive_probes` 次（默认 9 次），间隔 `tcp_keepalive_intvl`（默认 75 秒）。
- 都失败后断开连接。

### 10.4 Pacing 定时器

高精度定时器 `pacing_timer`，用于按 pacing rate 控制发包节奏，尤其在 BBR 等基于模型 pacing 的算法中重要。

---

## 11. 重要 sysctl 参数

| 参数 | 文件 | 含义 |
|------|------|------|
| `tcp_sack` | `/proc/sys/net/ipv4/tcp_sack` | 是否启用 SACK |
| `tcp_window_scaling` | `/proc/sys/net/ipv4/tcp_window_scaling` | 是否启用窗口缩放 |
| `tcp_timestamps` | `/proc/sys/net/ipv4/tcp_timestamps` | 是否启用 TCP 时间戳 |
| `tcp_fastopen` | `/proc/sys/net/ipv4/tcp_fastopen` | TFO 开关位图 |
| `tcp_syncookies` | `/proc/sys/net/ipv4/tcp_syncookies` | Syncookie 模式（0/1/2） |
| `tcp_slow_start_after_idle` | `/proc/sys/net/ipv4/tcp_slow_start_after_idle` | 空闲后是否重置 cwnd |
| `tcp_congestion_control` | `/proc/sys/net/ipv4/tcp_congestion_control` | 默认拥塞控制算法 |
| `tcp_available_congestion_control` | `/proc/sys/net/ipv4/tcp_available_congestion_control` | 可用算法列表 |
| `tcp_notsent_lowat` | `/proc/sys/net/ipv4/tcp_notsent_lowat` | 未发送数据低水位 |
| `tcp_retries1` | `/proc/sys/net/ipv4/tcp_retries1` | 重传次数阈值1 |
| `tcp_retries2` | `/proc/sys/net/ipv4/tcp_retries2` | 重传次数阈值2（最终超时） |
| `tcp_syn_retries` | `/proc/sys/net/ipv4/tcp_syn_retries` | SYN 重试次数 |
| `tcp_synack_retries` | `/proc/sys/net/ipv4/tcp_synack_retries` | SYN+ACK 重试次数 |
| `tcp_keepalive_time` | `/proc/sys/net/ipv4/tcp_keepalive_time` | 首次 keepalive 探测前空闲时间 |
| `tcp_keepalive_probes` | `/proc/sys/net/ipv4/tcp_keepalive_probes` | keepalive 探测次数 |
| `tcp_keepalive_intvl` | `/proc/sys/net/ipv4/tcp_keepalive_intvl` | keepalive 探测间隔 |
| `tcp_tw_reuse` | `/proc/sys/net/ipv4/tcp_tw_reuse` | 是否复用 TIME_WAIT 端口 |
| `tcp_fin_timeout` | `/proc/sys/net/ipv4/tcp_fin_timeout` | FIN_WAIT2 超时 |
| `tcp_mtu_probing` | `/proc/sys/net/ipv4/tcp_mtu_probing` | PLPMTUD 探测开关 |
| `tcp_base_mss` | `/proc/sys/net/ipv4/tcp_base_mss` | MTU 探测基础 MSS |
| `tcp_probe_interval` | `/proc/sys/net/ipv4/tcp_probe_interval` | MTU 探测间隔 |

---

## 12. 小结

TCP 是互联网上最复杂的传输协议之一，Linux 内核的实现也相应地分层、模块化和高度优化。理解 Linux TCP 的关键抓手：

1. 状态机：所有连接行为都由 `tcp_sock->sk_state` 和 `tcp_rcv_state_process()` 驱动。
2. 序列号空间：`snd_una / snd_nxt / write_seq / rcv_nxt / copied_seq` 是 TCP 可靠性的基础。
3. 两个窗口：发送受 `min(snd_wnd, snd_cwnd)` 限制，接收端通过 `rcv_wnd` 反压。
4. 红黑树重传队列：`tcp_rtx_queue` 是现代 Linux TCP 重传、SACK、RACK 的核心数据结构。
5. 定时器驱动：RTO、Delayed ACK、Keepalive、Pacing 共同保证连接活性、可靠性和性能。
6. 拥塞控制抽象：`tcp_congestion_ops` 把 Reno/Cubic/BBR 等算法与核心状态机解耦。

后续深入学习建议按以下顺序：
- 先精读 `tcp_v4_connect()` + `tcp_connect()`，理解主动建连。
- 再读 `tcp_conn_request()` + `tcp_check_req()`，理解被动建连。
- 然后读 `tcp_sendmsg()` + `tcp_write_xmit()` + `tcp_transmit_skb()`，理解发送。
- 最后读 `tcp_v4_rcv()` + `tcp_rcv_established()` + `tcp_ack()`，理解接收与拥塞控制。

---

> 提示：本文基于 Linux v7.1-rc7 源码整理。不同内核版本的函数名、字段名和默认参数可能略有差异，但核心架构（`tcp_sock`、状态机、红黑树重传队列、`tcp_input.c` / `tcp_output.c` 分工）长期稳定。
