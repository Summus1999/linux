# Linux Netfilter 子系统详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/netfilter/core.c` —— hook 注册/注销/调用核心（`nf_hook_slow`、`nf_register_net_hook`）
> - `net/netfilter/nf_conntrack_core.c` —— 连接跟踪核心（`nf_conntrack_in`、`resolve_normal_ct`）
> - `net/netfilter/nf_conntrack_proto_*.c` —— 各协议连接跟踪状态机
> - `net/netfilter/nf_nat_core.c` —— NAT 核心（`nf_nat_inet_fn`、`nf_nat_setup_info`）
> - `net/ipv4/netfilter/ip_tables.c` —— iptables/xtables 后端（`ipt_do_table`）
> - `net/netfilter/nf_tables_api.c` —— nftables 核心 API
> - `include/linux/netfilter.h` —— `struct nf_hook_ops`、`NF_HOOK` 宏、hook 调用入口
> - `include/uapi/linux/netfilter.h` —— 5 个 `nf_inet_hooks`、verdict 定义
> - `include/linux/netfilter/nf_conntrack.h` —— `struct nf_conn`、`enum ip_conntrack_info`
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、状态图、时序图使用 `text` 代码块表示。

---

## 1. Netfilter 是什么、解决什么问题

作用：在内核协议栈的固定位置埋下"钩子"（hook），让外部模块能在不修改协议栈代码的前提下，对经过的报文执行过滤 / 修改 / 转移 / 记账 / NAT / 连接跟踪。它是 Linux 防火墙、NAT、流量计费、容器网络策略（Cilium 在 eBPF 之前的方案、kube-proxy iptables 模式）的共同底座。

与协议栈的关系：Netfilter 不实现任何协议，它只提供"拦截点"。协议栈在 `ip_rcv`/`ip_output`/`ip_forward` 等关键函数里调用 `NF_HOOK(...)` 宏，把报文交给 Netfilter 处理后再决定继续走还是丢弃。

为什么单独成篇学：前序笔记里 `NF_INET_PRE_ROUTING`/`NF_INET_LOCAL_IN`/`NF_INET_FORWARD`/`NF_INET_LOCAL_OUT`/`NF_INET_POST_ROUTING` 这 5 个 hook 至少出现 6 次但都一笔带过。本文把这层"债"补上，也是后续 eBPF/XDP 学习的前置——XDP 的很多设计正是对 Netfilter 性能瓶颈的回应。

核心 RFC / 文档：RFC 7511（无关搞笑，跳过）；真实设计文档是 Rusty Russell 的 "Linux Netfilter Hacking HOWTO"；nftables 见 `man 8 nft` 与 `Documentation/networking/nf_flowtable.rst`。

---

## 2. 5 个 Hook 点：报文必经的 5 道关卡

```text
                                    本机产生报文
                                        │
                                        ▼
                              ┌───────────────────┐
                              │ NF_INET_LOCAL_OUT │  ← ip_output.c:120
                              └─────────┬─────────┘
                                        │
                                        ▼
                              ┌───────────────────┐
                              │NF_INET_POST_ROUTING│ ← ip_output.c:401/417/422
                              └─────────┬─────────┘
                                        │
               ┌────────────────────────┴───────────────────────────┐
               ▼                                                     ▼
        ┌─────────────┐  转发                       ┌──────────────┐
        │  NIC 收包    │                            │ 本机接收队列  │
        └──────┬──────┘                             └──────┬───────┘
               │                                           │
               ▼                                           ▼
     ┌────────────────────┐                     ┌────────────────────┐
     │NF_INET_PRE_ROUTING │ ← ip_input.c:612    │ NF_INET_LOCAL_IN   │ ← ip_input.c:262
     └─────────┬──────────┘                     └────────────────────┘
               │
        路由判定 dst_input()
               │
       ┌───────┴───────┐
       ▼               ▼
 ┌───────────┐   ┌──────────────┐
 │  本机交付  │   │NF_INET_FORWARD│ ← ip_forward.c:162
 │ (LOCAL_IN)│   └──────┬───────┘
 └───────────┘          │
                        ▼
                ┌────────────────────┐
                │NF_INET_POST_ROUTING│
                └────────────────────┘
```

| Hook 点 | 文件:行 | 触发时机 | 典型用途 |
|---------|---------|----------|----------|
| `NF_INET_PRE_ROUTING` | `ip_input.c:612` | 刚收包、校验后、路由前 | DNAT、流量计费、rp_filter |
| `NF_INET_LOCAL_IN` | `ip_input.c:262` | 路由判定为本机后、交 L4 前 | INPUT 链过滤、本机收包计费 |
| `NF_INET_FORWARD` | `ip_forward.c:162` | 路由判定转发、TTL-- 后 | FORWARD 链过滤、转发限速 |
| `NF_INET_LOCAL_OUT` | `ip_output.c:120` | 本机产生报文、构造 IP 头后 | OUTPUT 链过滤、本机出口计费 |
| `NF_INET_POST_ROUTING` | `ip_output.c:401/417/422` | 即将进入邻居/网卡发送 | SNAT（MASQUERADE）、POSTROUTING 计费 |

记忆要点：
- 5 个 hook 形成两段：RX 段（PRE_ROUTING → 分叉为 LOCAL_IN 或 FORWARD）+ TX 段（LOCAL_OUT → POST_ROUTING）。FORWARD 路径会汇入 POST_ROUTING。
- hook 不是函数调用点，是"宏展开为条件调用"——下文 §4 详解。
- 除 `nf_inet_hooks` 外，还有 `NF_NETDEV_INGRESS`（generic netdev ingress，用于 ingress qdisc、XDP 之后的进一步处理）和 `NF_NETDEV_EGRESS`（较新）。

---

## 3. 核心数据结构

### 3.1 `struct nf_hook_ops` —— 一个 hook 注册项

文件：`include/linux/netfilter.h:98-111`

```c
struct nf_hook_ops {
	struct list_head	list;
	struct rcu_head		rcu;

	/* User fills in from here down. */
	nf_hookfn		*hook;
	struct net_device	*dev;
	void			*priv;
	u8			pf;
	enum nf_hook_ops_type	hook_ops_type:8;
	unsigned int		hooknum;
	/* Hooks are ordered in ascending priority. */
	int			priority;
};
```

字段含义：

| 字段 | 含义 |
|------|------|
| `hook` | 回调函数，签名 `unsigned int (*)(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)` |
| `priv` | 回调私有数据（如 iptables 的 `xt_table`） |
| `pf` | 协议族：`NFPROTO_IPV4`/`NFPROTO_IPV6`/`NFPROTO_ARP`/`NFPROTO_BRIDGE`/`NFPROTO_NETDEV` |
| `hooknum` | 在哪一道关卡：5 个 `nf_inet_hooks` 之一 |
| `priority` | 同一关卡多个 hook 时的执行顺序，升序；`NF_IP_PRI_FIRST=INT_MIN` 最先，常用优先级见 `include/uapi/linux/netfilter_ipv4.h`（`NF_IP_PRI_RAW=-300`、`NF_IP_PRI_CONNTRACK=-200`、`NF_IP_PRI_MANGLE=-150`、`NF_IP_PRI_NAT_DST=-100`、`NF_IP_PRI_FILTER=0`、`NF_IP_PRI_NAT_SRC=100`、`NF_IP_PRI_LAST=INT_MAX`） |
| `hook_ops_type` | 内核内部区分 iptables / nftables / BPF 注册来源 |

> 设计要点：`nf_hook_ops` 里的 `list`/`rcu` 字段不在热路径使用——热路径用的是更紧凑的 `struct nf_hook_entry`（只含 `hook` + `priv` 两个指针）。`ops` 只在注册/注销时需要，因此被放在 `nf_hook_entries` 末尾的 trailer 区。这样 cache 局部性更好。见 §3.3。

### 3.2 `struct nf_hook_entries` —— 一个关卡的 hook 数组

文件：`include/linux/netfilter.h:123-141`

```c
struct nf_hook_entries {
	u16				num_hook_entries;
	/* padding */
	struct nf_hook_entry		hooks[];

	/* trailer: pointers to original orig_ops of each hook,
	 * followed by rcu_head and scratch space used for freeing
	 * the structure via call_rcu.
	 *
	 *   This is not part of struct nf_hook_entry since its only
	 *   needed in slow path (hook register/unregister):
	 * const struct nf_hook_ops     *orig_ops[]
	 *
	 *   For the same reason, we store this at end -- its
	 *   only needed when a hook is deleted, not during
	 *   packet path processing:
	 * struct nf_hook_entries_rcu_head     head
	 */
};
```

布局图：

```text
┌──────────────────────────────────────────────────────────┐
│ struct nf_hook_entries                                   │
│   num_hook_entries = N                                   │
│   hooks[0]: { hook fn, priv }   ← 热路径，紧凑           │
│   hooks[1]: { hook fn, priv }                           │
│   ...                                                    │
│   hooks[N-1]                                             │
├──────────────────────────────────────────────────────────┤
│ trailer（仅注册/注销慢路径用）：                          │
│   orig_ops[0..N-1]: 指回 nf_hook_ops（含 priority 等）    │
│   rcu_head: 用于 call_rcu 释放                           │
└──────────────────────────────────────────────────────────┘
```

为什么这样设计：报文路径只需 `hook` 函数指针和 `priv`，不需要 `priority`（注册时已按 priority 排序）。把冷数据挪到尾部，热路径扫描 `hooks[]` 数组时 cache 命中率最高。

per-netns 存储：每个 netns 的 `struct net` 里有：

```c
struct net {
    ...
    struct netns_nf nf;
    ...
};

struct netns_nf {
    struct nf_hook_entries __rcu *hooks_ipv4[NF_INET_NUMHOOKS];
    struct nf_hook_entries __rcu *hooks_ipv6[NF_INET_NUMHOOKS];
    struct nf_hook_entries __rcu *hooks_arp[NF_ARP_NUMHOOKS];
    struct nf_hook_entries __rcu *hooks_bridge[NF_INET_NUMHOOKS];
    ...
};
```

即每个 netns、每个协议族、每个 hook 关卡，都独立有一个 `nf_hook_entries` 数组。这就是 `iptables` 在不同 netns 里规则互不干扰的根本原因。

### 3.3 verdict：hook 回调的返回值

文件：`include/uapi/linux/netfilter.h:10-33`

```c
/* Responses from hook functions. */
#define NF_DROP 0
#define NF_ACCEPT 1
#define NF_STOLEN 2
#define NF_QUEUE 3
#define NF_REPEAT 4
#define NF_STOP 5	/* Deprecated, for userspace nf_queue compatibility. */
```

| verdict | 语义 | 协议栈后续行为 |
|---------|------|---------------|
| `NF_DROP` | 丢弃；高 16 位可编码 errno | 释放 skb，立即返回；`NF_DROP_ERR(x)` 把 errno 编进去 |
| `NF_ACCEPT` | 放行 | 继续执行下一个 hook；最后一个 hook ACCEPT 后调用 `okfn`（如 `ip_rcv_finish`） |
| `NF_STOLEN` | hook 接管了 skb，协议栈别管了 | 协议栈不再继续，skb 由 hook 模块自行处理（如 queue 给用户态、延迟处理） |
| `NF_QUEUE` | 入队到用户态（nfqueue） | 通过 `nf_queue` 机制发给用户态进程（`NFQUEUE` target） |
| `NF_REPEAT` | 重新跑当前 hook | 很少用 |

> 关键约定：`NF_ACCEPT == 1`，而 `okfn` 在 `NF_HOOK` 宏里靠 `ret == 1` 触发——见 §4。

---

## 4. Hook 调用机制：`NF_HOOK` 宏与 `nf_hook_slow`

### 4.1 `NF_HOOK` 宏展开

文件：`include/linux/netfilter.h:311-320`

```c
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	int ret = nf_hook(pf, hook, net, sk, skb, in, out, okfn);
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```

语义：先调 `nf_hook()` 跑所有 hook；若全部 `ACCEPT`，`nf_hook` 返回 1，宏里再调 `okfn`（如 `ip_rcv_finish`）继续协议栈。否则（DROP/STOLEN/QUEUE）`ret != 1`，不调 `okfn`，skb 由 hook 层处理。

### 4.2 `nf_hook` 快路径

文件：`include/linux/netfilter.h`（`nf_hook` 内联）

```c
static inline int nf_hook(uint8_t pf, unsigned int hook, struct net *net,
			  struct sock *sk, struct sk_buff *skb,
			  struct net_device *in, struct net_device *out,
			  int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	struct nf_hook_entries *hook_head = NULL;
#ifdef CONFIG_JUMP_LABEL
	...
#endif
	rcu_read_lock();
	switch (pf) {
	case NFPROTO_IPV4:
		hook_head = rcu_dereference(net->nf.hooks_ipv4[hook]);
		break;
	/* ... IPV6, ARP, BRIDGE ... */
	}
	if (hook_head) {
		struct nf_hook_state state;
		nf_hook_state_init(&state, hook, pf, in, out, sk, net, okfn);
		ret = nf_hook_slow(skb, &state, hook_head);
	}
	rcu_read_unlock();
	return ret;
}
```

两个性能优化点（务必记住，这是 Netfilter 在生产环境"还能用"的关键）：

1. `CONFIG_JUMP_LABEL` + `nf_hooks_needed[pf][hook]` 静态键：当某 `(pf, hook)` 关卡没有任何 hook 注册时，静态键关闭，`NF_HOOK_LIST` 直接 `return`，几乎零开销。只有注册过 hook 的关卡才进入慢路径。这就是"没开 iptables 时几乎不影响性能"的原因。
2. `NF_HOOK_LIST` 批量处理：NAPI 攒一批 skb 后用 `NF_HOOK_LIST` 一次 `rcu_read_lock` 处理整批，避免每个包单独锁。见 `ip_input.c:681` 的 `ip_list_rcv` 路径。

### 4.3 `nf_hook_slow` 逐个 hook 调用

文件：`net/netfilter/core.c`（`nf_hook_slow`）

```text
nf_hook_slow(skb, state, hook_head):
    for (i = 0; i < hook_head->num_hook_entries; i++) {
        verdict = hook_head->hooks[i].hook(hooks[i].priv, skb, state);
        switch (verdict & NF_VERDICT_MASK) {
        case NF_ACCEPT:   continue;            // 走下一个 hook
        case NF_DROP:     kfree_skb(skb); return verdict;  // 丢弃
        case NF_STOLEN:   return verdict;       // hook 接管，不再继续
        case NF_QUEUE:    return nf_queue(skb, state, verdict);  // 入用户态队列
        case NF_REPEAT:   i--; continue;        // 重跑当前 hook
        }
    }
    return 1;  // 全部 ACCEPT，让 NF_HOOK 宏去调 okfn
```

> 注意 `verdict & NF_VERDICT_MASK`：verdict 高位可能编码了 queue number 或 errno，必须 mask 取低 8 位判断。

---

## 5. Hook 注册与注销

### 5.1 `nf_register_net_hook`

文件：`net/netfilter/core.c`

注册流程：

```text
nf_register_net_hook(net, ops):
    1. 取出该 (pf, hooknum) 关卡当前的 nf_hook_entries（旧）
    2. 分配新 nf_hook_entries，大小 = 旧 N + 1
    3. 按 ops->priority 升序插入新数组
    4. 更新 nf_hooks_needed 静态键（若从 0 → 1，开启静态键）
    5. rcu_assign_pointer(net->nf.hooks_ipv4[hook], new)  // RCU 原子替换
    6. synchronize_rcu / call_rcu 释放旧数组
```

关键设计：整个 hook 数组是不可变的——任何注册/注销都重新分配整个 `nf_hook_entries`，用 RCU 原子替换指针。这样读路径（报文路径）完全无锁，只在写路径（规则变更）有开销。代价是规则频繁变更时内存分配压力较大。

### 5.2 `nf_unregister_net_hook`

注销流程类似，分配 N-1 的新数组替换。注意必须等 RCU 宽限期结束才能释放旧数组，因为可能有报文还在读旧数组。

### 5.3 常见注册来源

| 来源 | hook_ops_type | 典型模块 |
|------|---------------|----------|
| iptables (xtables) | `NF_HOOK_OP_UNDEFINED` | `iptable_filter`/`iptable_nat`/`iptable_mangle`（每个表注册一个 hook，回调里跑 `ipt_do_table`） |
| nftables | `NF_HOOK_OP_NF_TABLES` | `nf_tables_chain_type` |
| BPF | `NF_HOOK_OP_BPF` | `BPF_PROG_TYPE_NETFILTER`（较新） |
| 内核内置模块 | — | `nf_conntrack`（注册 PRE_ROUTING/LOCAL_OUT 做 conntrack）、`nf_nat`、`iptable_raw` |

---

## 6. iptables（xtables）后端：`ipt_do_table`

### 6.1 表与链的关系

iptables 的 5 张表（filter/nat/mangle/raw/security）× 5 条链（INPUT/OUTPUT/FORWARD/PREROUTING/POSTROUTING），但不是所有表都挂在所有链上：

| 表 | 有效 hook | 用途 |
|----|-----------|------|
| raw | PRE_ROUTING, LOCAL_OUT | 在 conntrack 之前做标记（`NOTRACK`） |
| mangle | 全部 5 个 | 修改报文头（TOS/TTL/mark） |
| nat | PRE_ROUTING(DNAT), POST_ROUTING(SNAT), LOCAL_OUT(DNAT), LOCAL_IN | NAT |
| filter | LOCAL_IN, FORWARD, LOCAL_OUT | 过滤 |
| security | LOCAL_IN, FORWARD, LOCAL_OUT | SELinux 标记 |

每张表在初始化时调用 `ipt_register_table` 注册到一个 hook，回调统一是 `ipt_do_table`。表内部用 `ipt_replace` 结构存规则。

### 6.2 `ipt_do_table` 匹配流程

文件：`net/ipv4/netfilter/ip_tables.c`

```text
ipt_do_table(skb, state, table):
    遍历 table->entries 的规则链：
        对每条规则：
            1. 匹配 IP 头基本字段（src/dst/proto/iface）
            2. 调用所有 match 的 ->match()（如 tcp match、state match）
            3. 全部匹配则执行 target：
                - 标准 target：ACCEPT/DROP/RETURN/JUMP（跳到另一条链）
                - 扩展 target：调用 ->target()（如 REJECT/LOG/MASQUERADE）
        不匹配则继续下一条
    链遍历完默认走 policy（ACCEPT/DROP）
```

性能要点：iptables 规则线性匹配，O(N)。K8s 里 kube-proxy iptables 模式一个 Service 可能上千条规则，这就是后来 IPVS 模式和 eBPF 模式出现的根本原因。

---

## 7. 连接跟踪（conntrack）：状态机与表

### 7.1 conntrack 解决的问题

iptables 的 `state`/`conntrack` match 能写出"只放行已建立连接的回包"这种规则，靠的就是 conntrack。它为每条流（双向五元组）维护一个 `struct nf_conn`，记录状态、超时、NAT 映射。

### 7.2 核心结构

- `struct nf_conn`：一条连接的跟踪条目，含 tuple、状态、超时、NAT 信息、协议私有数据
- `struct nf_conntrack_tuple`：单向五元组（src/dst/proto/端口），一条连接有 original + reply 两个 tuple
- `enum ip_conntrack_info`：skb 相对连接的方向与位置（`IP_CT_ESTABLISHED`/`IP_CT_RELATED`/`IP_CT_NEW`/`IP_CT_IS_REPLY`）

### 7.3 状态机

```text
                 NEW
                  │  （首包，original 方向）
                  ▼
     ┌──────────────────────┐
     │  ESTABLISHED + RELATED│  ← 收到 reply 方向回包后转 ESTABLISHED
     │  （双向都见过报文）     │
     └──────────┬───────────┘
                │  超时（不同协议不同超时）
                ▼
            链表移除、kmem_cache_free
```

### 7.4 核心函数

文件：`net/netfilter/nf_conntrack_core.c`

- `nf_conntrack_in(net, pf, hooknum, skb)` —— hook 回调，PRE_ROUTING 和 LOCAL_OUT 调用
- `resolve_normal_ct(skb, dataoff, protonum, &ctinfo)` —— 查 tuple hash 表，找到或新建 `nf_conn`
- `nf_ct_set(skb, ct, ctinfo)` —— 把 conntrack 指针挂到 `skb->_nfct`，后续 hook/target 可直接读

### 7.5 conntrack 表的 hash 与扩容

- `nf_conntrack_hash`：全局 hash 表（per-netns），key 是 tuple
- 大小由 `nf_conntrack_max` sysctl 控制，默认 `nf_conntrack_buckets * 4`
- 满 75% 触发 GC，`nf_conntrack_gc_begin` 清理超时条目
- `nf_conntrack_count` 接近 `nf_conntrack_max` 时新连接会被丢，日志 `nf_conntrack: table full, dropping packet`

---

## 8. NAT：`nf_nat_core`

### 8.1 NAT 与 conntrack 的关系

NAT 不是一个独立 hook，而是挂在 conntrack 之上的附加信息。每条 `nf_conn` 可携带一个 `struct nf_conn_nat`，记录 original/reply 两个方向的 NAT 映射。NAT hook 只在 conntrack 之后执行：

- DNAT：PRE_ROUTING（`nf_nat_inet_fn` in `iptable_nat` 的 PRE_ROUTING hook）
- SNAT：POST_ROUTING（同函数，POST_ROUTING hook）

### 8.2 核心函数

文件：`net/netfilter/nf_nat_core.c`

- `nf_nat_inet_fn(skb, state)` —— NAT 主入口，根据 conntrack 状态决定是否做、做 DNAT 还是 SNAT
- `nf_nat_setup_info(ct, &range, maniptype)` —— 真正改 tuple，并把原 tuple 记录到 `ct->nat`，便于 reply 方向还原
- `manip_pkt(skb, &target, maniptype, ...)` —— 改 IP/TCP/UDP 头里的地址端口

### 8.3 MASQUERADE vs SNAT

- `SNAT --to-source IP`：固定源 IP，性能好（conntrack 建立后映射固定）
- `MASQUERADE`：源 IP 取自出接口当前地址，接口 IP 变（如拨号）时自动适配；但每次发包都要查接口地址，且接口 down 时会立即清 conntrack

---

## 9. nftables：iptables 的继任者

### 9.1 为什么要替换 iptables

| 维度 | iptables (xtables) | nftables |
|------|--------------------|----------|
| 规则存储 | 平铺数组，线性扫描 | 字节码虚拟机，支持 map/set/字典 |
| 匹配语法 | 每 match 一个内核模块 | 统一表达式（`nft_expr`），无新模块 |
| 性能 | O(N) 线性 | map/set 查找 O(1)，规则可优化 |
| 事务 | 无（规则逐条原子） | 有（`NFT_MSG_NEWRULE` 事务批提交） |
| IPv4/IPv6 | 两套（iptables/ip6tables） | 统一 `nf_tables` family |

### 9.2 核心结构

- `struct nft_chain`：一条链，挂在某个 hook 上
- `struct nft_rule`：一条规则，由一串 `nft_expr` 组成
- `struct nft_expr_ops`：表达式操作（cmp/lookup/counter/immediate/...）
- `nft_do_chain(skb, state)` —— hook 回调，执行链上所有规则的字节码

### 9.3 执行模型

nftables 把规则编译成字节码，`nft_do_chain` 是一个解释器，逐条执行表达式。相比 iptables 的函数指针调用，字节码更紧凑、cache 更友好、且能在用户态预编译优化。

---

## 10. 与协议栈的衔接点全表

把前序笔记里出现的 `NF_HOOK` 全部对齐：

| 出现位置 | 文件:行 | hook | okfn | 笔记出处 |
|----------|---------|------|------|----------|
| IP 收包 | `ip_input.c:612` | PRE_ROUTING | `ip_rcv_finish` | ip-study §4 / datapath RX |
| IP 批量收包 | `ip_input.c:681` | PRE_ROUTING（LIST） | `ip_list_rcv_finish` | network-framework §2.7 |
| IP 本地交付 | `ip_input.c:262` | LOCAL_IN | `ip_local_deliver_finish` | ip-study §4 |
| IP 转发 | `ip_forward.c:162` | FORWARD | `ip_forward_finish` | ip-study §6 |
| 本机出口 | `ip_output.c:120` | LOCAL_OUT | `ip_output` | ip-study §5 / datapath TX |
| 即将发送 | `ip_output.c:401/417/422` | POST_ROUTING | `ip_finish_output`/`nf_hook_slow` | datapath TX |

---

## 11. 可观测性：怎么看 Netfilter 在干什么

```bash
# 看 conntrack 表使用量
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 看各协议超时参数
ls /proc/sys/net/netfilter/nf_conntrack_*_timeout_*

# 看 iptables 规则与计数
iptables -t nat -L -n -v
iptables -t filter -L -n -v

# 看 nftables
nft list ruleset

# conntrack 工具
conntrack -L              # 列出所有条目
conntrack -E              # 实时事件流
conntrack -C              # 计数

# perf / bpftrace 定位 hook 热点
perf trace -e 'net:*nf*'
bpftrace -e 'kprobe:nf_hook_slow { @[comm] = count(); }'

# dropwatch 定位丢包
dropwatch -l kas
```

---

## 12. 常见陷阱与注意事项

1. `NF_DROP` 不会调用 `okfn`，但 skb 释放由 Netfilter 负责——协议栈函数（如 `ip_rcv`）只需把 `NF_HOOK` 的返回值往上传，不要自己 `kfree_skb`。
2. `NF_STOLEN` 协议栈什么都不能做——skb 所属权转移给 hook，连引用计数都不能动。
3. conntrack 在 raw 表之前——错了，raw 表（`NF_IP_PRI_RAW=-300`）在 conntrack（`NF_IP_PRI_CONNTRACK=-200`）之前，所以 `NOTRACK` 能生效。
4. NAT 改了 IP 头后要重算校验和——`manip_pkt` 内部会调 `nf_csum_update`，但如果你写自定义 target 改 IP 头，必须自己重算。
5. hook 优先级冲突——同优先级时注册顺序决定执行顺序，不可依赖。自定义模块应使用 `NF_IP_PRI_*` 常量而非裸数字。
6. netns 隔离——每个 netns 独立 hook 数组，`iptables -t nat -L` 在不同 netns 看到不同规则。容器网络依赖这点。
7. 性能悬崖——conntrack 表满、iptables 规则数过千、NAPI 批量被打断成单包路径，性能都会断崖式下降。

---

## 13. 总结

Netfilter 是 Linux 网络的"控制平面"，把 5 个 hook 点埋在 IP 协议栈的关键路径上，通过 `NF_HOOK` 宏把报文交给外部模块处理。核心设计：

- 不可变 hook 数组 + RCU：读路径无锁，写路径整体替换
- 静态键 `nf_hooks_needed`：未注册 hook 的关卡零开销
- conntrack + NAT 分层：conntrack 是基础，NAT 是其上的附加信息
- iptables → nftables 迁移：从线性匹配到字节码虚拟机

学完本文后，下一步是 eBPF/XDP——它把"拦截点"从 IP 层下沉到驱动 RX 软中断之前，性能比 Netfilter 高一个数量级，是现代高性能网络方案（Cilium、Katran、Cloudflare）的基础。

---

## 14. 待补充（TODO）

- [ ] §5.1 `nf_register_net_hook` 完整源码贴入与逐行注解
- [ ] §6.2 `ipt_do_table` 完整匹配循环源码
- [ ] §7.4 `resolve_normal_ct` 的 tuple hash 查找源码
- [ ] §8.2 `nf_nat_setup_info` 的 manip 类型与范围处理
- [ ] §9.3 nftables 字节码 `nft_do_chain` 源码与一条规则的完整展开
- [ ] 一张完整的 conntrack 状态机图（含 `IP_CT_RELATED`/`IP_CT_IS_REPLY`）
- [ ] nfqueue（`NF_QUEUE`）用户态收包路径
- [ ] `nf_flowtable` 硬件流卸载（fast path 旁路 Netfilter）
