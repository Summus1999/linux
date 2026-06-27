# Linux 网络报文收发完整代码调用链（TCP/IPv4 主线，Linux v7.1-rc7）

> 本文在 `docs/Linux 网络框架核心.md` 的基础上，把**用户 send/recv 到网卡驱动再返回用户**的完整调用链抽出来，形成一张系统的、可逐行对照源码的调用图。
>
> 约定：
> - `text` 代码块为教学性的调用图、时序图；
> - `c` 代码块只引用内核原始源码片段；
> - 路径以 TCP/IPv4 + 以太网 + NAPI 为主线，UDP 与 IPv6 只在差异点做简要标注。

---

## 1. 总览：一张图理解收发全貌

```text
用户空间                          内核协议栈                              网卡硬件/驱动
─────────────────────────────────────────────────────────────────────────────────────────────

发送路径（TX）
─────────────
write()/sendto()/sendmsg()
  │
  ▼
SYSCALL_DEFINE3(sendmsg, ...)     ── net/socket.c
  └── __sys_sendmsg() / ____sys_sendmsg()
        └── sock_sendmsg()          ── net/socket.c:813
              └── inet_sendmsg()    ── net/ipv4/af_inet.c:857
                    └── tcp_sendmsg() ── net/ipv4/tcp.c:1447
                          └── tcp_sendmsg_locked() ── net/ipv4/tcp.c:1117
                                ├── 分配/复用 skb（tcp_stream_alloc_skb / tcp_skb_entail）
                                ├── 拷贝用户数据（skb_copy_to_page_nocache）
                                └── tcp_push() ── net/ipv4/tcp.c:741
                                      └── __tcp_push_pending_frames()
                                            └── tcp_write_xmit()  ── 按 cwnd/发送窗口/TSQ 决定发哪些发多大
                                                  └── tcp_transmit_skb()
                                                        └── __tcp_transmit_skb()
                                                              ├── 构造 TCP 头（__skb_push + tcp_options_write）
                                                              ├── skb_orphan(); skb->sk = sk; skb->destructor = tcp_wfree
                                                              └── INDIRECT_CALL_INET(queue_xmit)
                                                                    └── ip_queue_xmit() ── net/ipv4/ip_output.c:546
                                                                          └── __ip_queue_xmit()
                                                                                ├── 路由查找（ip_route_output_flow）
                                                                                ├── 构造 IP 头（skb_push + iph）
                                                                                └── ip_local_out()
                                                                                      └── __ip_local_out()
                                                                                            └── NF_INET_LOCAL_OUT（netfilter）
                                                                                                  └── dst_output()
                                                                                                        └── ip_output() ── net/ipv4/ip_output.c:428
                                                                                                              ├── NF_INET_POST_ROUTING
                                                                                                              └── ip_finish_output()
                                                                                                                    ├── BPF_CGROUP_RUN_PROG_INET_EGRESS
                                                                                                                    └── __ip_finish_output()
                                                                                                                          ├── GSO 分片（ip_finish_output_gso）
                                                                                                                          ├── IP 分片（ip_fragment）
                                                                                                                          └── ip_finish_output2()
                                                                                                                                └── dst_neigh_output()
                                                                                                                                      └── neigh_resolve_output() / neigh_direct_output()
                                                                                                                                            └── dev_queue_xmit() ── net/core/dev.c
                                                                                                                                                  └── __dev_queue_xmit() ── net/core/dev.c:4766
                                                                                                                                                        ├── TC egress / netfilter egress（sch_handle_egress / nf_hook_egress）
                                                                                                                                                        ├── 选队列 netdev_core_pick_tx()
                                                                                                                                                        ├── 入 qdisc：__dev_xmit_skb() → qdisc → sch_direct_xmit()
                                                                                                                                                        └── 或直接 dev_hard_start_xmit()
                                                                                                                                                              └── dev->netdev_ops->ndo_start_xmit()
                                                                                                                                                                    └── 驱动写入 TX ring / DMA
                                                                                                                                                                          └── 网卡发送

接收路径（RX）
─────────────
网卡收到帧
  │
  ▼
硬中断
  └── napi_schedule(napi)              ── 关硬中断，napi 入 poll_list，raise NET_RX_SOFTIRQ

NET_RX_SOFTIRQ
  └── net_rx_action()
        └── napi->poll(napi, budget)   ── 驱动轮询函数
              ├── napi_alloc_skb() / build_skb()
              ├── DMA 数据映射到 skb->data
              ├── eth_type_trans(skb, dev)  ── 设置 skb->protocol = ETH_P_IP
              └── napi_gro_receive(napi, skb)
                    └── gro_receive_skb()   ── net/core/gro.c
                          └── dev_gro_receive()
                                ├── GRO_MERGED / GRO_HELD → 聚合，稍后上送
                                └── GRO_NORMAL → gro_normal_one(&napi->gro, skb, segs)
                                      └── 攒满 gro_normal_batch → gro_normal_list(&napi->gro)
                                            └── netif_receive_skb_list_internal(&gro->rx_list)
                                                  └── __netif_receive_skb_list()
                                                        └── __netif_receive_skb_one_core()  ── net/core/dev.c
                                                              └── __netif_receive_skb_core() ── net/core/dev.c:5972
                                                                    ├── ptype_all 投递（tcpdump / AF_PACKET）
                                                                    ├── TC ingress / netfilter
                                                                    ├── VLAN 解标签
                                                                    ├── rx_handler（bridge / bonding / macvlan）
                                                                    └── deliver_ptype_list_skb() 按 ethertype 分发
                                                                          └── ip_rcv() ── net/ipv4/ip_input.c:603
                                                                                ├── ip_rcv_core() 基础校验
                                                                                ├── NF_INET_PRE_ROUTING
                                                                                └── ip_rcv_finish() ── net/ipv4/ip_input.c:478
                                                                                      └── dst_input(skb)
                                                                                            └── 转发：ip_forward()
                                                                                            └── 本机：ip_local_deliver() ── net/ipv4/ip_input.c:250
                                                                                                  ├── ip_defrag() 分片重组
                                                                                                  ├── NF_INET_LOCAL_IN
                                                                                                  └── ip_local_deliver_finish() ── net/ipv4/ip_input.c:229
                                                                                                        └── __skb_pull(IP 头)
                                                                                                        └── ip_protocol_deliver_rcu() ── 查 inet_protos[]
                                                                                                              └── tcp_v4_rcv() ── net/ipv4/tcp_ipv4.c:2068
                                                                                                                    ├── 查 socket（__inet_lookup_skb）
                                                                                                                    ├── tcp_checksum_complete()
                                                                                                                    └── tcp_v4_do_rcv() ── net/ipv4/tcp_ipv4.c:1827
                                                                                                                          └── tcp_rcv_established() ── net/ipv4/tcp_input.c:6467
                                                                                                                                └── tcp_data_queue() ── net/ipv4/tcp_input.c:5574
                                                                                                                                      ├── __skb_pull(TCP 头)
                                                                                                                                      ├── tcp_queue_rcv() ── 入 sk_receive_queue
                                                                                                                                      └── tcp_data_ready()
                                                                                                                                            └── sk->sk_data_ready = sock_def_readable → 唤醒等待进程

用户空间读取
  │
  ▼
read()/recvfrom()/recvmsg()
  │
  └── __sys_recvmsg()
        └── sock_recvmsg()
              └── inet_recvmsg() ── net/ipv4/af_inet.c:886
                    └── tcp_recvmsg() ── net/ipv4/tcp.c:2931
                          └── tcp_recvmsg_locked()
                                └── skb_copy_datagram_msg() / skb_copy_datagram_iter()
                                      └── 拷贝到用户态 iov
                                      └── consume_skb() → sock_rfree() 释放接收内存
```

---

## 2. 发送路径：用户态 → 网卡（TX）

### 2.1 系统调用层 → socket 层

```text
用户态 send()/sendto()/sendmsg()
  │
  ▼
SYSCALL_DEFINE3(sendto, ...)      net/socket.c
SYSCALL_DEFINE3(sendmsg, ...)
  └── __sys_sendto() / __sys_sendmsg()
        ├── sockfd_lookup_light()     fd → struct socket
        └── ____sys_sendmsg()
              └── sock_sendmsg()      net/socket.c:813
                    └── sock_sendmsg_nosec()
                          └── sock->ops->sendmsg(sock, msg, size)
                                └── inet_sendmsg()   net/ipv4/af_inet.c:857
                                      └── sk->sk_prot->sendmsg(sk, msg, size)
                                            └── tcp_sendmsg()   net/ipv4/tcp.c:1447
```

`inet_sendmsg()` 是 `AF_INET` 所有 socket 类型（TCP/UDP/raw）共用的分发点：

```c
int inet_sendmsg(struct socket *sock, struct msghdr *msg, size_t size)
{
    struct sock *sk = sock->sk;
    const struct proto *prot;

    if (unlikely(inet_send_prepare(sk)))
        return -EAGAIN;

    /* IPV6_ADDRFORM 可在并发下把 sk->sk_prot 从 tcpv6 切到 tcpv4，
     * 因此用 READ_ONCE 取到局部变量 prot 后再解引用。 */
    prot = READ_ONCE(sk->sk_prot);
    return INDIRECT_CALL_2(prot->sendmsg, tcp_sendmsg, udp_sendmsg,
                   sk, msg, size);
}
```

注意准备阶段调的是单参数的 `inet_send_prepare(sk)`（负责 RPS 记流 + 必要时自动 bind），而不是带 `msg` 的版本；返回 `-EAGAIN` 通常意味着自动 bind 失败。

### 2.2 TCP 层：构造 skb、排队、推送

```text
tcp_sendmsg(sk, msg, size)
  └── tcp_sendmsg_locked(sk, msg, size)     net/ipv4/tcp.c:1117
        ├── tcp_send_mss()                  计算 mss_now，并确定 size_goal
        ├── while (msg_data_left(msg))       循环拆分用户数据
        │     ├── tcp_write_queue_tail(sk)   尝试向队尾 skb 追加
        │     ├── tcp_stream_alloc_skb()     不够则新建 skb
        │     ├── tcp_skb_entail(sk, skb)    挂入 write_queue
        │     └── skb_copy_to_page_nocache() 拷贝用户数据到 skb
        └── tcp_push(sk, flags, mss_now, ...)  net/ipv4/tcp.c:741
              └── __tcp_push_pending_frames()
                    └── tcp_write_xmit()    net/ipv4/tcp_output.c:2963
                          │  按 cwnd/发送窗口/TSQ 决定发哪些、发多大：
                          ├── tcp_cwnd_test()          拥塞窗口配额是否够
                          ├── tcp_snd_wnd_test()       对端接收窗口是否够
                          ├── tcp_tso_should_defer()   TSO 聚合判定，减少小包
                          ├── tcp_mss_split_point()    按配额把大 skb 拆成多段
                          └── 对每个可发的 skb → tcp_transmit_skb()    net/ipv4/tcp_output.c:1730
                                └── __tcp_transmit_skb()  net/ipv4/tcp_output.c:1529
```

`__tcp_transmit_skb()` 是真正给 skb 套上 TCP 头并下传到 IP 层的函数。注意此时 skb 刚从 `write_queue` 出来，**头部还是空的**（headerless），函数会：

1. 调用 `__skb_push(skb, tcp_header_size)` 为 TCP 头腾空间；
2. 填充 `tcphdr`（源/目的端口、seq、ack_seq、flags、window、options 等）；
3. 计算 TCP checksum；
4. 把 skb 关联回 socket 的发送内存：`skb_orphan(); skb->sk = sk; skb->destructor = tcp_wfree; refcount_add(skb->truesize, &sk->sk_wmem_alloc);`
5. 通过 `icsk->icsk_af_ops->queue_xmit` 调用 `ip_queue_xmit()`。

```c
/* net/ipv4/tcp_output.c:1529 */
static int __tcp_transmit_skb(struct sock *sk, struct sk_buff *skb,
                  int clone_it, gfp_t gfp_mask, u32 rcv_nxt)
{
    ...
    __skb_push(skb, tcp_header_size);
    skb_reset_transport_header(skb);

    skb_orphan(skb);
    skb->sk = sk;
    skb->destructor = skb_is_tcp_pure_ack(skb) ? __sock_wfree : tcp_wfree;
    refcount_add(skb->truesize, &sk->sk_wmem_alloc);
    ...
    err = INDIRECT_CALL_INET(icsk->icsk_af_ops->queue_xmit,
                 inet6_csk_xmit, ip_queue_xmit,
                 sk, skb, &inet->cork.fl);
    ...
}
```

### 2.3 IP 层：路由 + 构造 IP 头 + netfilter

```text
ip_queue_xmit(sk, skb, fl)        net/ipv4/ip_output.c:546
  └── __ip_queue_xmit(sk, skb, fl, tos)   net/ipv4/ip_output.c:463
        ├── 路由：__sk_dst_check() / ip_route_output_flow()
        ├── skb_dst_set_noref(skb, &rt->dst)
        ├── skb_push(skb, sizeof(struct iphdr) + optlen)  为 IP 头（+ IP 选项）腾空间
        ├── skb_reset_network_header(skb)
        ├── 填充 iphdr（version/ihl/tos/ttl/protocol/saddr/daddr/id/frag_off）
        └── ip_local_out(net, sk, skb)
              └── __ip_local_out(net, sk, skb)
                    └── NF_HOOK(NF_INET_LOCAL_OUT) → ip_output()
```

`ip_output()` 会经过 `NF_INET_POST_ROUTING`，然后进入 `ip_finish_output()`：

```text
ip_output()   net/ipv4/ip_output.c:428
  └── NF_INET_POST_ROUTING
        └── ip_finish_output()   net/ipv4/ip_output.c:318
              ├── BPF cgroup egress
              └── __ip_finish_output()   net/ipv4/ip_output.c:297
                    ├── ip_finish_output_gso()  若 skb_is_gso()
                    ├── ip_fragment()           若 len > mtu
                    └── ip_finish_output2()
                          └── dst_neigh_output()
                                └── neigh_resolve_output() / neigh_direct_output()
                                      └── dev_queue_xmit(skb)
```

### 2.4 设备层：选队列、qdisc、驱动

```text
dev_queue_xmit(skb)      net/core/dev.c
  └── __dev_queue_xmit(skb, NULL)    net/core/dev.c:4766
        ├── skb_reset_mac_header(skb)
        ├── [CONFIG_NET_EGRESS] 若 egress_needed_key：
        │     ├── nf_hook_egress(skb)        netfilter egress 钩子
        │     └── sch_handle_egress(skb)     TC egress（clsact/egress，tc filter 在此分流）
        ├── txq = netdev_core_pick_tx(dev, skb, sb_dev)   选发送队列
        ├── q = rcu_dereference_bh(txq->qdisc)
        │
        ├── [有 qdisc]
        │     └── __dev_xmit_skb(skb, q, dev, txq)
        │           └── qdisc->enqueue / dequeue → sch_direct_xmit()
        │                 └── dev_hard_start_xmit(skb, dev, txq, &rc)
        │                       └── dev->netdev_ops->ndo_start_xmit(skb, dev)
        │
        └── [无 qdisc，如 loopback]
              └── dev_hard_start_xmit(skb, dev, txq, &rc)
                    └── dev->netdev_ops->ndo_start_xmit(skb, dev)
                          └── 驱动：DMA map → 写 TX descriptor → kick doorbell
                                └── 网卡发送 → 发送完成中断
                                      └── napi_consume_skb() / consume_skb()
                                            └── skb->destructor = tcp_wfree / __sock_wfree
                                                  └── refcount_sub(skb->truesize, &sk->sk_wmem_alloc)
                                                        └── 当 sk_wmem_alloc 归零 → __sk_free(sk)
```

**发送完成释放内存**：驱动在 TX completion 中断里调用 `napi_consume_skb()` 或 `consume_skb()`，触发 `tcp_wfree()`（或纯 ACK 的 `__sock_wfree()`），把 `sk_wmem_alloc` 减回来。计数归零时由最后一个 `tcp_wfree()` 调用 `__sk_free(sk)` 完成 socket 延迟释放。

---

## 3. 接收路径：网卡 → 用户态（RX）

### 3.1 硬中断 → NAPI 轮询

```text
网卡收到帧 → 触发硬中断
  │
  └── 驱动中断处理函数
        └── napi_schedule(napi)               include/linux/netdevice.h:558
              ├── napi_schedule_prep(n)        net/core/dev.c:6729  test_and_set NAPI_STATE_SCHED
              └── __napi_schedule(n)           net/core/dev.c:6710
                    └── ____napi_schedule(sd, n)  net/core/dev.c:4957
                          ├── list_add_tail(&n->poll_list, &sd->poll_list)
                          └── raise_softirq_irqoff(NET_RX_SOFTIRQ)

ksoftirqd / 退出硬中断
  └── net_rx_action()   net/core/dev.c:7914
        └── 遍历 poll_list
              └── napi->poll(napi, budget)   驱动轮询函数
                    ├── 从 RX ring 取描述符
                    ├── napi_alloc_skb() / build_skb() 构造 skb
                    ├── DMA 数据已填充到 skb->data
                    ├── eth_type_trans(skb, dev)
                    │     └── 设置 skb->protocol = ETH_P_IP
                    │     └── 设置 skb->pkt_type（单播/广播/多播）
                    └── napi_gro_receive(napi, skb)
                          └── gro_receive_skb()
```

注意 `napi_schedule()` 并不是直接做三件事，而是一个 inline wrapper：先用 `napi_schedule_prep()` 通过 `try_cmpxchg` 原子地尝试置位 `NAPI_STATE_SCHED`（若已调度则置 `NAPI_STATE_MISSED` 后返回 false），只有 prep 成功才调 `__napi_schedule()`，后者关中断后调 `____napi_schedule()` 把 napi 挂到本 CPU 的 `softnet_data.poll_list` 并 raise 软中断。三个函数原始源码如下：

```c
/* include/linux/netdevice.h:558 */
static inline bool napi_schedule(struct napi_struct *n)
{
	if (napi_schedule_prep(n)) {
		__napi_schedule(n);
		return true;
	}

	return false;
}

/* net/core/dev.c:6729 */
bool napi_schedule_prep(struct napi_struct *n)
{
	unsigned long new, val = READ_ONCE(n->state);

	do {
		if (unlikely(val & NAPIF_STATE_DISABLE))
			return false;
		new = val | NAPIF_STATE_SCHED;

		/* Sets STATE_MISSED bit if STATE_SCHED was already set */
		new |= (val & NAPIF_STATE_SCHED) / NAPIF_STATE_SCHED *
				   NAPIF_STATE_MISSED;
	} while (!try_cmpxchg(&n->state, &val, new));

	return !(val & NAPIF_STATE_SCHED);
}

/* net/core/dev.c:6710 */
void __napi_schedule(struct napi_struct *n)
{
	unsigned long flags;

	local_irq_save(flags);
	____napi_schedule(this_cpu_ptr(&softnet_data), n);
	local_irq_restore(flags);
}

/* net/core/dev.c:4957 */
static inline void ____napi_schedule(struct softnet_data *sd,
				     struct napi_struct *napi)
{
	struct task_struct *thread;

	lockdep_assert_irqs_disabled();

	if (test_bit(NAPI_STATE_THREADED, &napi->state)) {
		thread = READ_ONCE(napi->thread);
		if (thread) {
			if (use_backlog_threads() && thread == raw_cpu_read(backlog_napi))
				goto use_local_napi;

			set_bit(NAPI_STATE_SCHED_THREADED, &napi->state);
			wake_up_process(thread);
			return;
		}
	}

use_local_napi:
	DEBUG_NET_WARN_ON_ONCE(!list_empty(&napi->poll_list));
	list_add_tail(&napi->poll_list, &sd->poll_list);
	WRITE_ONCE(napi->list_owner, smp_processor_id());
	/* If not called from net_rx_action()
	 * we have to raise NET_RX_SOFTIRQ.
	 */
	if (!sd->in_net_rx_action)
		raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```


### 3.2 GRO → L2 分发

```text
napi_gro_receive(napi, skb)
  └── gro_receive_skb(&napi->gro, skb)
        └── dev_gro_receive(gro, skb)
              ├── 找到匹配 flow → GRO_MERGED / GRO_HELD（聚合）
              └── 无匹配 → GRO_NORMAL
                    └── gro_normal_one(gro, skb, segs)   include/net/gro.h:537
                          ├── list_add_tail(&skb->list, &gro->rx_list)  先挂到批量链表
                          ├── gro->rx_count += segs
                          └── 若 rx_count >= gro_normal_batch（默认 64）
                                └── gro_normal_list(gro)   include/net/gro.h:519
                                      └── netif_receive_skb_list_internal(&gro->rx_list)
                                            └── __netif_receive_skb_list()
                                                  └── __netif_receive_skb_one_core(skb, false)
                                                        ├── __netif_receive_skb_core(&skb, false, &pt_prev)
                                                        └── 若 pt_prev：pt_prev->func(skb,...) → ip_rcv()
```

注意 `gro_normal_one()` 的第一个参数是 `struct gro_node *gro`（即 `&napi->gro`），不是 `napi` 本身；它**并不立即上送**，而是把 skb 挂进 `gro->rx_list` 并累加 `rx_count`，只有攒够 `gro_normal_batch`（默认 64）时才由 `gro_normal_list()` 批量调用 `netif_receive_skb_list_internal()` 上送协议栈。GRO 在 `napi->poll()` 结束时（`napi_complete_done` 之前）也会再调一次 `gro_normal_list()` 把剩余的 flush 上去。

对于非 NAPI 设备或绕过 GRO 的场景，直接走 `netif_receive_skb(skb)`，最终也会进入 `__netif_receive_skb_core()`。

### 3.3 IP 层：校验、路由、分片、L3→L4 分发

```text
ip_rcv(skb, dev, pt, orig_dev)   net/ipv4/ip_input.c:603
  └── ip_rcv_core(skb, net)      做 IP 头校验、长度检查
        └── NF_INET_PRE_ROUTING
              └── ip_rcv_finish()   net/ipv4/ip_input.c:478
                    ├── 路由查找：ip_route_input_noref() / skb_dst_set()
                    └── dst_input(skb)
                          ├── 转发路径：ip_forward()
                          └── 本机路径：ip_local_deliver()   net/ipv4/ip_input.c:250
                                ├── ip_defrag(net, skb, IP_DEFRAG_LOCAL_DELIVER)  分片重组
                                ├── NF_INET_LOCAL_IN
                                └── ip_local_deliver_finish()   net/ipv4/ip_input.c:229
                                      ├── __skb_pull(skb, skb_network_header_len(skb))  剥 IP 头
                                      └── ip_protocol_deliver_rcu(net, skb, ip_hdr(skb)->protocol)
                                            ├── raw_local_deliver(skb, protocol)   RAW socket
                                            └── ipprot = rcu_dereference(inet_protos[protocol])
                                                  └── INDIRECT_CALL_2(ipprot->handler, tcp_v4_rcv, udp_rcv, skb)
                                                        └── tcp_v4_rcv()   net/ipv4/tcp_ipv4.c:2068
```

`inet_protos[]` 是一张 256 项的表，下标就是 IP 头 `protocol` 字段。关键注册在 `net/ipv4/af_inet.c` 的 `inet_init()` 中：

```c
net_hotdata.tcp_protocol = (struct net_protocol) {
    .handler    = tcp_v4_rcv,
    .err_handler    = tcp_v4_err,
    .no_policy  = 1,
    .icmp_strict_tag_validation = 1,
};
inet_add_protocol(&net_hotdata.tcp_protocol, IPPROTO_TCP);
```

### 3.4 TCP 层：找 socket、校验、入接收队列

```text
tcp_v4_rcv(skb)   net/ipv4/tcp_ipv4.c:2068
  ├── __inet_lookup_skb(net, &tcp_hashinfo, skb, ...)  根据四元组找 sock
  ├── tcp_checksum_complete(skb)  校验和检查
  ├── tcp_filter(sk, skb)  socket filter / cgroup BPF 入站过滤（sk_filter_trim_cap）
  ├── 若 sk_state == TCP_LISTEN：直接 tcp_v4_do_rcv(sk, skb)
  ├── 若 sock_owned_by_user(sk)：tcp_add_backlog(sk, skb)  socket 被用户态锁住，入 backlog 延后处理
  └── 否则：bh_lock_sock_nested → tcp_v4_do_rcv(sk, skb)   net/ipv4/tcp_ipv4.c:1827
        ├── 若 sk_state == TCP_ESTABLISHED：
        │     └── tcp_rcv_established(sk, skb)   net/ipv4/tcp_input.c:6467
        │           ├── tcp_header_len 校验
        │           ├── tcp_data_queue(sk, skb)   net/ipv4/tcp_input.c:5574
        │           │     ├── __skb_pull(skb, tcp_hdrlen)  剥 TCP 头
        │           │     ├── tcp_queue_rcv(sk, skb, &fragstolen)  入 sk_receive_queue
        │           │     ├── tcp_event_data_recv(sk, skb)
        │           │     ├── tcp_ofo_queue(sk)  处理乱序包
        │           │     └── tcp_data_ready(sk)  唤醒用户态
        │           └── tcp_send_ack() / tcp_send_delayed_ack()  准备 ACK
        │
        ├── 若 sk_state == TCP_LISTEN：
        │     └── tcp_v4_cookie_check() / tcp_child_process() / tcp_rcv_state_process()
        │
        └── 其它状态：
              └── tcp_rcv_state_process(sk, skb)   状态机处理 SYN/FIN/RST 等
```

`tcp_queue_rcv()` 把 skb 挂到 `sk->sk_receive_queue`，然后 `tcp_data_ready()` 调用 `sk->sk_data_ready`（默认 `sock_def_readable`）唤醒阻塞在 `recvmsg()` 上的进程。

### 3.5 用户态读取

```text
read()/recvfrom()/recvmsg()
  │
  └── __sys_recvmsg()
        └── sock_recvmsg()
              └── inet_recvmsg()   net/ipv4/af_inet.c:886
                    └── tcp_recvmsg(sk, msg, len, flags)   net/ipv4/tcp.c:2931
                          └── tcp_recvmsg_locked()
                                ├── 从 sk_receive_queue 取出 skb
                                ├── skb_copy_datagram_msg(skb, offset, msg, copied)
                                │     └── 拷贝数据到用户 iov
                                └── consume_skb(skb)
                                      └── sock_rfree(skb)   释放接收内存
                                            └── atomic_sub(skb->truesize, &sk->sk_rmem_alloc)
```

---

## 4. 关键分发表：三张表决定报文去向

| 表 | 结构 | 索引 | 作用 | 关键函数 | 位置 |
|----|------|------|------|----------|------|
| `ptype_base[]` / `ptype_specific` / `ptype_all` | `struct packet_type` | `skb->protocol`（ethertype） | L2 → L3 分发 | `__netif_receive_skb_core()` → `deliver_ptype_list_skb()` | `net/core/dev.c` |
| `inet_protos[]` | `struct net_protocol` | IP 头 `protocol` 字段 | IPv4 L3 → L4 分发 | `ip_protocol_deliver_rcu()` | `net/ipv4/protocol.c` |
| `inetsw[]` | `struct inet_protosw` | `socket type`（SOCK_STREAM/DGRAM/RAW） | socket 创建时匹配 `type + protocol` | `inet_create()` | `net/ipv4/af_inet.c` |

### 4.1 L2 → L3 分发流程（`__netif_receive_skb_core`）

```text
__netif_receive_skb_core(pskb, pfmemalloc, ppt_prev)
  │
  ├── 1. ptype_all 投递    → tcpdump / AF_PACKET
  ├── 2. TC ingress / netfilter
  ├── 3. VLAN 处理
  ├── 4. rx_handler（bridge / bonding / macvlan）
  └── 5. 按 ethertype 分发
        ├── deliver_ptype_list_skb(skb, &pt_prev, type, &ptype_base[hash])
        ├── deliver_ptype_list_skb(skb, &pt_prev, type, &net->ptype_specific)
        └── deliver_ptype_list_skb(skb, &pt_prev, type, &orig_dev->ptype_specific)
              └── 匹配到 ip_packet_type / arp_packet_type / ipv6_packet_type
                    └── pt_prev->func = ip_rcv / arp_rcv / ipv6_rcv
```

### 4.2 L3 → L4 分发流程（`ip_protocol_deliver_rcu`）

```text
ip_protocol_deliver_rcu(net, skb, protocol)
  │
  ├── raw_local_deliver(skb, protocol)   RAW socket
  ├── ipprot = rcu_dereference(inet_protos[protocol])
  │     ├── IPPROTO_TCP → tcp_v4_rcv(skb)
  │     ├── IPPROTO_UDP → udp_rcv(skb)
  │     └── IPPROTO_ICMP → icmp_rcv(skb)
  └── 若无 handler 且无 raw socket：icmp_send(ICMP_PROT_UNREACH) + drop
```

---

## 5. skb 内存生命周期与 socket 的绑定

### 5.1 发送路径：skb_set_owner_w / tcp_wfree

```text
__tcp_transmit_skb()
  ├── skb_orphan(skb)               清除旧 socket 关联
  ├── skb->sk = sk
  ├── skb->destructor = tcp_wfree   （纯 ACK 用 __sock_wfree）
  └── refcount_add(skb->truesize, &sk->sk_wmem_alloc)
        └── sk_wmem_alloc 记账「在途发送包」占用的内存

网卡 TX completion
  └── napi_consume_skb(skb) / consume_skb(skb)
        └── skb_unref() 减 users → 0
              └── __kfree_skb()
                    └── skb_release_head_state()
                          └── skb->destructor(skb) = tcp_wfree()
                                └── refcount_sub(skb->truesize, &sk->sk_wmem_alloc)
                                      └── 若归零 → __sk_free(sk)
```

### 5.2 接收路径：skb_set_owner_r / sock_rfree

```text
tcp_queue_rcv(sk, skb, &fragstolen)   net/ipv4/tcp_input.c:5499
  ├── tcp_try_coalesce(sk, tail, skb)  尝试合并到 sk_receive_queue 队尾 skb
  │     └── 合并成功（eaten=1）：skb 数据并入 tail，新 skb 被释放，不单独记账
  └── 合并失败（!eaten）：
        ├── tcp_add_receive_queue(sk, skb)  挂入 sk_receive_queue
        └── skb_set_owner_r(skb, sk)        include/net/sock.h:2450
              ├── skb_orphan(skb)
              ├── skb->sk = sk
              ├── skb->destructor = sock_rfree
              ├── atomic_add(skb->truesize, &sk->sk_rmem_alloc)
              └── sk_mem_charge(sk, skb->truesize)

tcp_add_backlog(sk, skb)   net/ipv4/tcp_ipv4.c:1899
  └── skb_set_owner_r(skb, sk)   backlog 中的 skb 同样计入 sk_rmem_alloc

用户 recvmsg()
  └── consume_skb(skb)
        └── sock_rfree(skb)
              └── atomic_sub(skb->truesize, &sk->sk_rmem_alloc)
                    └── sk_mem_uncharge(sk, skb->truesize)
```

注意：`tcp_queue_rcv()` 只有在 **未 coalesce**（`!eaten`）时才对单独的 skb 调 `skb_set_owner_r()`；coalesce 成功时新 skb 被合并进队尾 skb 释放掉，其 `truesize` 不会重复计入 `sk_rmem_alloc`。这是接收路径减少内存记账开销的关键优化。

---

## 6. 时序图：一次 TCP echo 请求的收发全过程

```text
用户进程 A                      内核协议栈                       网卡
   │                               │                              │
   │ sendmsg("hello")              │                              │
   │──────────────────────────────▶│                              │
   │                               │ tcp_sendmsg()                │
   │                               │ tcp_transmit_skb()           │
   │                               │ ip_queue_xmit()              │
   │                               │ dev_queue_xmit()             │
   │                               │ ndo_start_xmit()             │
   │                               │─────────────────────────────▶│
   │                               │                              │ DMA → 网卡发送
   │                               │◀─────────────────────────────│ TX completion
   │                               │ napi_consume_skb()           │
   │                               │ tcp_wfree()                  │
   │                               │                              │
   │                               │◀──── 对端回包（"hello"）─────│
   │                               │ 硬中断 → napi_schedule()     │
   │                               │ net_rx_action()              │
   │                               │ napi->poll()                 │
   │                               │ napi_gro_receive()           │
   │                               │ netif_receive_skb_list_internal()│
   │                               │ ip_rcv()                     │
   │                               │ ip_local_deliver()           │
   │                               │ tcp_v4_rcv()                 │
   │                               │ tcp_rcv_established()        │
   │                               │ tcp_data_queue()             │
   │                               │ tcp_data_ready() → wakeup    │
   │ recvmsg()                     │                              │
   │──────────────────────────────▶│                              │
   │                               │ tcp_recvmsg()                │
   │                               │ skb_copy_datagram_msg()      │
   │                               │ consume_skb() → sock_rfree() │
   │◀──────────────────────────────│ 返回 "hello"                 │
```

---

## 7. 函数/文件索引速查表

| 阶段 | 关键函数 | 文件 | 行号 |
|------|----------|------|------|
| 系统调用入口 | `__sys_sendmsg()` / `__sys_sendto()` / `__sys_recvmsg()` | `net/socket.c` | 2767 / 2230 / 2976 |
| socket 层 | `sock_sendmsg()` / `sock_recvmsg()` | `net/socket.c` | 813 |
| AF_INET 分发 | `inet_sendmsg()` / `inet_recvmsg()` | `net/ipv4/af_inet.c` | 857 / 886 |
| TCP 发送 | `tcp_sendmsg()` / `tcp_sendmsg_locked()` | `net/ipv4/tcp.c` | 1447 / 1117 |
| TCP 推送 | `tcp_push()` | `net/ipv4/tcp.c` | 741 |
| TCP 实际发送 | `__tcp_transmit_skb()` / `tcp_transmit_skb()` | `net/ipv4/tcp_output.c` | 1529 / 1730 |
| IP 发送 | `ip_queue_xmit()` / `__ip_queue_xmit()` | `net/ipv4/ip_output.c` | 546 / 463 |
| IP 输出 | `ip_output()` / `ip_finish_output()` / `__ip_finish_output()` | `net/ipv4/ip_output.c` | 428 / 318 / 297 |
| 邻居子系统 | `dst_neigh_output()` / `neigh_resolve_output()` | `net/core/neighbour.c` | - |
| 设备发送 | `dev_queue_xmit()` / `__dev_queue_xmit()` | `net/core/dev.c` | - / 4766 |
| 驱动接口 | `ndo_start_xmit()` | 各网卡驱动 | - |
| NAPI 调度 | `napi_schedule()` / `net_rx_action()` | `include/linux/netdevice.h` / `net/core/dev.c` | 558 / 7914 |
| GRO 接收 | `napi_gro_receive()` / `gro_receive_skb()` | `net/core/gro.c` | - |
| L2 分发核心 | `__netif_receive_skb_core()` | `net/core/dev.c` | 5972 |
| IP 接收 | `ip_rcv()` / `ip_rcv_finish()` | `net/ipv4/ip_input.c` | 603 / 478 |
| IP 本机投递 | `ip_local_deliver()` / `ip_local_deliver_finish()` | `net/ipv4/ip_input.c` | 250 / 229 |
| L3→L4 分发 | `ip_protocol_deliver_rcu()` | `net/ipv4/ip_input.c` | 189 |
| TCP 接收 | `tcp_v4_rcv()` / `tcp_v4_do_rcv()` | `net/ipv4/tcp_ipv4.c` | 2068 / 1827 |
| TCP 已连接处理 | `tcp_rcv_established()` | `net/ipv4/tcp_input.c` | 6467 |
| TCP 数据入队 | `tcp_data_queue()` / `tcp_queue_rcv()` | `net/ipv4/tcp_input.c` | 5574 / 5499 |
| TCP 读取 | `tcp_recvmsg()` | `net/ipv4/tcp.c` | 2931 |
| 数据拷贝到用户 | `skb_copy_datagram_msg()` / `skb_copy_datagram_iter()` | `net/core/datagram.c` | - |
| skb 分配 | `__alloc_skb()` / `build_skb()` / `napi_alloc_skb()` | `net/core/skbuff.c` | - |
| skb 释放 | `consume_skb()` / `kfree_skb_reason()` / `__kfree_skb()` | `net/core/skbuff.c` | - |
| socket 内存绑定 | `skb_set_owner_w()`（非 inline） / `skb_set_owner_r()`（inline） | `net/core/sock.c` / `include/net/sock.h` | - / 2450 |
| 发送内存释放 | `tcp_wfree()` / `__sock_wfree()` | `net/ipv4/tcp.c` / `net/core/sock.c` | - |
| 接收内存释放 | `sock_rfree()` | `net/core/sock.c` | - |

---

## 8. 小结

Linux 网络栈的收发路径可以用两条主线概括：

- **发送（TX）**：用户 `sendmsg()` → `struct socket` → `struct sock` → TCP 构造 skb/IP 头 → 路由 → 邻居 → `net_device` → qdisc → 驱动 `ndo_start_xmit()` → 网卡。
- **接收（RX）**：网卡 → 硬中断 → NAPI 轮询 → `napi_gro_receive()` → `__netif_receive_skb_core()` 按 ethertype 分发 → `ip_rcv()` → `ip_local_deliver()` 按 IP protocol 分发 → `tcp_v4_rcv()` → `tcp_rcv_established()` → 入 `sk_receive_queue` → 用户 `recvmsg()`。

贯穿这两条主线的核心机制是：

1. **三层分发**：`ptype_base[]`（L2→L3）和 `inet_protos[]`（L3→L4）决定报文交给哪个协议；
2. **两个 socket 抽象**：`struct socket` 面向用户/VFS，`struct sock` 面向内核协议；
3. **统一的报文容器**：`struct sk_buff` 携带头部偏移、路由、设备、GSO、引用计数、内存记账等全部元数据；
4. **发送/接收内存记账**：`skb_set_owner_w/r()` 把 skb 生命周期与 `sk_wmem_alloc` / `sk_rmem_alloc` 绑定，`tcp_wfree()` / `sock_rfree()` 在 skb 释放时归还配额；
5. **NAPI + GRO**：用软中断轮询替代每包硬中断，用 GRO 聚合减少协议栈入口开销。

把这张图和 `docs/Linux 网络框架核心.md` 里的数据结构章节对照阅读，就能从「函数调用顺序」和「数据结构协作」两个维度，建立对 Linux 网络协议栈的完整理解。
