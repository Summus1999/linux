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

为什么单独成篇学：前序笔记里 `NF_INET_PRE_ROUTING`/`NF_INET_LOCAL_IN`/`NF_INET_FORWARD`/`NF_INET_LOCAL_OUT`/`NF_INET_POST_ROUTING` 这 5 个 hook 至少出现 6 次但都一笔带过，本文系统讲清这层；也是后续 eBPF/XDP 学习的前置，XDP 的很多设计正是对 Netfilter 性能瓶颈的回应。

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
| `priority` | 同一关卡多个 hook 时的执行顺序，升序；`NF_IP_PRI_FIRST=INT_MIN` 最先，常用优先级见 `include/uapi/linux/netfilter_ipv4.h`（`NF_IP_PRI_RAW_BEFORE_DEFRAG=-450`、`NF_IP_PRI_RAW=-300`、`NF_IP_PRI_CONNTRACK=-200`、`NF_IP_PRI_MANGLE=-150`、`NF_IP_PRI_NAT_DST=-100`、`NF_IP_PRI_FILTER=0`、`NF_IP_PRI_NAT_SRC=100`、`NF_IP_PRI_CONNTRACK_CONFIRM=INT_MAX`、`NF_IP_PRI_LAST=INT_MAX`） |
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

两个性能优化点：

1. `CONFIG_JUMP_LABEL` + `nf_hooks_needed[pf][hook]` 静态键：当某 `(pf, hook)` 关卡没有任何 hook 注册时，静态键关闭，`NF_HOOK_LIST` 直接 `return`，几乎零开销。只有注册过 hook 的关卡才进入慢路径。这就是"没开 iptables 时几乎不影响性能"的原因。
2. `NF_HOOK_LIST` 批量处理：NAPI 攒一批 skb 后用 `NF_HOOK_LIST` 一次 `rcu_read_lock` 处理整批，避免每个包单独锁。见 `ip_input.c:681` 的 `ip_list_rcv` 路径。

### 4.3 `nf_hook_slow` 逐个 hook 调用

文件：`net/netfilter/core.c:610-645`

**这是 Netfilter 的热路径核心**——每个报文经过每个有 hook 的关卡都会进这里。

源码：

```c
/* Returns 1 if okfn() needs to be executed by the caller,
 * -EPERM for NF_DROP, 0 otherwise.  Caller must hold rcu_read_lock. */
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state,
		 const struct nf_hook_entries *e, unsigned int s)
{
	unsigned int verdict;
	int ret;

	for (; s < e->num_hook_entries; s++) {
		verdict = nf_hook_entry_hookfn(&e->hooks[s], skb, state);
		switch (verdict & NF_VERDICT_MASK) {
		case NF_ACCEPT:
			break;
		case NF_DROP:
			kfree_skb_reason(skb,
					 SKB_DROP_REASON_NETFILTER_DROP);
			ret = NF_DROP_GETERR(verdict);
			if (ret == 0)
				ret = -EPERM;
			return ret;
		case NF_QUEUE:
			ret = nf_queue(skb, state, s, verdict);
			if (ret == 1)
				continue;
			return ret;
		case NF_STOLEN:
			return NF_DROP_GETERR(verdict);
		default:
			WARN_ON_ONCE(1);
			return 0;
		}
	}

	return 1;
}
EXPORT_SYMBOL(nf_hook_slow);
```

逐行注解：

| 行 | 代码 | 说明 |
|----|------|------|
| 注释 | `Returns 1 if okfn() needs to be executed` | **返回值约定是理解整个 NF_HOOK 宏的钥匙**：`1` = 全部 ACCEPT，调用者（`NF_HOOK` 宏）去调 `okfn`；`-EPERM` 或负值 = DROP；`0` = STOLEN/QUEUE 等，调用者什么都不做 |
| 注释 | `Caller must hold rcu_read_lock` | 调用者必须已持 RCU 读锁——因为 `e`（hook 数组）是 RCU 保护的，读期间数组不能被释放 |
| 签名 | `const struct nf_hook_entries *e, unsigned int s` | `e` 是该关卡的 hook 数组；`s` 是**起始下标**（不是固定 0！）。这是为了支持 `NF_QUEUE` 后续从断点继续——见 §4.4 |
| 618 | `for (; s < e->num_hook_entries; s++)` | 从下标 `s` 开始遍历所有 hook。升序 priority 已在注册时排好（见 §5.1），这里顺序执行即按 priority 升序 |
| 619 | `nf_hook_entry_hookfn(&e->hooks[s], skb, state)` | 调 `e->hooks[s].hook(e->hooks[s].priv, skb, state)`。注意用的是 `nf_hook_entry`（只含 hook+priv 两个指针），不是 `nf_hook_ops`——热路径只取必要字段，见 §3.1 的设计要点 |
| 620 | `verdict & NF_VERDICT_MASK` | **关键**：verdict 高 16 位可能编码了 queue number 或 errno，必须 mask 取低 8 位判断。`NF_VERDICT_MASK = 0x000000ff` |
| 621-622 | `case NF_ACCEPT: break;` | ACCEPT 不是"放行通过关卡"，而是"放行到**下一个 hook**"。只有**所有** hook 都 ACCEPT，循环正常结束，返回 1，才让 `NF_HOOK` 宏调 `okfn` |
| 623-629 | `case NF_DROP:` | `kfree_skb_reason` 释放 skb（带 drop reason 便于 dropwatch 观测）；`NF_DROP_GETERR(verdict)` 从高位取 errno，没有则默认 `-EPERM`。返回负值，调用者据此不调 `okfn` |
| 630-634 | `case NF_QUEUE:` | 入队到用户态（nfqueue）。`nf_queue` 返回 1 表示"用户态裁决为 ACCEPT，继续跑后续 hook"——此时 `continue` 回到 for 循环；返回其他值表示用户态裁决为 DROP/STOLEN 或入队失败，直接返回。**这是 `s` 参数存在的意义**：`nf_queue` 会把当前下标 `s` 记下来，用户态裁决 ACCEPT 后从 `s+1` 继续跑 |
| 635-636 | `case NF_STOLEN:` | hook 接管了 skb。返回 `NF_DROP_GETERR(verdict)`（通常 0）。**注意：这里没有 `kfree_skb`**——skb 所有权已转移给 hook 模块，由它自行处理（可能延迟、可能 queue、可能转发），协议栈不能再碰 |
| 637-639 | `default:` | 未知 verdict，告警并返回 0。理论上不会到这 |
| 643 | `return 1;` | **全部 hook ACCEPT 的正常出口**。返回 1 触发 `NF_HOOK` 宏调 `okfn`（如 `ip_rcv_finish`）继续协议栈 |

**性能特征**：

1. **O(N) 线性扫描**：N = 该关卡注册的 hook 数。这就是为什么 iptables 规则多了会慢——每条规则本质都是 match+target，挂在同一关卡。
2. **无锁**：整个循环在 RCU 读侧，无任何锁。写侧（注册/注销）通过整体替换数组保证一致性，见 §5。
3. **`nf_hook_entry` 而非 `nf_hook_ops`**：热路径只碰 `hook`+`priv` 两个指针（16 字节），`priority`/`pf`/`hooknum` 等冷字段在 trailer 区，cache 友好。
4. **`NF_REPEAT` 没了**：早期版本有 `case NF_REPEAT: i--; continue;`，现代内核（v5.x+）已移除该 case，verdict=4 会落入 `default` 告警。这是历史包袱，`NF_REPEAT` 定义还在 uapi 但内核不再处理。

### 4.4 `nf_hook_slow_list`：NAPI 批量路径

文件：`net/netfilter/core.c:647-663`

```c
void nf_hook_slow_list(struct list_head *head, struct nf_hook_state *state,
		       const struct nf_hook_entries *e)
{
	struct sk_buff *skb, *next;
	LIST_HEAD(sublist);
	int ret;

	list_for_each_entry_safe(skb, next, head, list) {
		skb_list_del_init(skb);
		ret = nf_hook_slow(skb, state, e, 0);
		if (ret == 1)
			list_add_tail(&skb->list, &sublist);
	}
	/* Put passed packets back on main list */
	list_splice(&sublist, head);
}
EXPORT_SYMBOL(nf_hook_slow_list);
```

**设计意图**：NAPI 一次 poll 攒一批 skb（典型 64 个），`ip_list_rcv` 用 `NF_HOOK_LIST` 一次 `rcu_read_lock` 处理整批，避免每个包单独进 RCU 临界区。

**关键细节**：

| 行 | 说明 |
|----|------|
| `LIST_HEAD(sublist)` | 临时链表，存"所有 hook 都 ACCEPT 的包" |
| `skb_list_del_init(skb)` | 先从原链表摘下来。因为 `nf_hook_slow` 可能 DROP（`kfree_skb`）或 STOLEN，skb 可能消失，不能留在原链表上 |
| `ret = nf_hook_slow(skb, state, e, 0)` | 起始下标 `s=0`。批量路径不支持 NF_QUEUE 断点续跑（用户态 nfqueue 不适合 NAPI 批量路径） |
| `if (ret == 1) list_add_tail(...)` | 只有 ACCEPT 的包才进 sublist；DROP/STOLEN 的包已被 `nf_hook_slow` 内部处理，这里不用管 |
| `list_splice(&sublist, head)` | 把通过的包拼回主链表，交给 `okfn` 的批量版本（如 `ip_list_rcv_finish`） |

**与单包路径的关系**：`NF_HOOK`（单包）vs `NF_HOOK_LIST`（批量）的选择由协议栈决定。`ip_rcv` 走单包（兼容老驱动），`ip_list_rcv` 走批量（NAPI 新路径）。两者最终都调 `nf_hook_slow`，只是外层包装不同。

### 4.5 静态键优化：`nf_hooks_needed`

文件：`net/netfilter/core.c:30-33`

```c
#ifdef CONFIG_JUMP_LABEL
struct static_key nf_hooks_needed[NFPROTO_NUMPROTO][NF_MAX_HOOKS];
EXPORT_SYMBOL(nf_hooks_needed);
#endif
```

**这是 Netfilter"不开规则就几乎零开销"的根本原因**。

`nf_hooks_needed[pf][hook]` 是一个 `static_key`（跳转标签），每个 `(协议族, hook关卡)` 组合一个：

- **没有任何 hook 注册**时，静态键处于"未启用"分支，`NF_HOOK_LIST` 里的 `static_key_false(&nf_hooks_needed[pf][hook])` 编译为 `unlikely` 跳转，几乎不执行（单条 `jmp` 指令）。
- **首次注册 hook**时，`nf_static_key_inc`（`core.c:359`）调 `static_key_slow_inc` 启用该键，内核会**动态 patch 所有引用点的指令**，改成走慢路径。
- **最后一个 hook 注销**后，`nf_static_key_dec` 关闭该键，指令再 patch 回快路径。

**效果**：生产服务器即使编译了 Netfilter，只要没 `iptables -A` 任何规则，IP 收发路径开销几乎为零——防火墙功能默认编进内核但默认零开销。

> **对比 eBPF/XDP**：XDP 的 hook 点本身就在驱动 RX 早期，且 BPF 程序直接 JIT 成原生指令执行，没有"线性扫描 hook 数组"的开销。这就是为什么 XDP 比 Netfilter 快一个数量级——但代价是 XDP 程序不能像 Netfilter hook 那样动态增删多个，每个 hook 点同一时间只能挂一个 BPF 程序。

---

## 5. Hook 注册与注销

### 5.1 `nf_register_net_hook` 完整注册链

注册有两个层次：外层 `nf_register_net_hook` 处理 `NFPROTO_INET` 的拆分（IPv4+IPv6 各注册一次），内层 `__nf_register_net_hook` 做实际工作。

#### 5.1.1 外层 `nf_register_net_hook`

文件：`net/netfilter/core.c:550-578`

```c
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *reg)
{
	int err;

	if (reg->pf == NFPROTO_INET) {
		if (reg->hooknum == NF_INET_INGRESS) {
			err = __nf_register_net_hook(net, NFPROTO_INET, reg);
			if (err < 0)
				return err;
		} else {
			err = __nf_register_net_hook(net, NFPROTO_IPV4, reg);
			if (err < 0)
				return err;

			err = __nf_register_net_hook(net, NFPROTO_IPV6, reg);
			if (err < 0) {
				__nf_unregister_net_hook(net, NFPROTO_IPV4, reg);
				return err;
			}
		}
	} else {
		err = __nf_register_net_hook(net, reg->pf, reg);
		if (err < 0)
			return err;
	}

	return 0;
}
EXPORT_SYMBOL(nf_register_net_hook);
```

**关键点**：`NFPROTO_INET` 是"IPv4+IPv6 同时注册"的语法糖。一个 ops 声明 `pf=NFPROTO_INET`，内核会拆成两次注册（IPv4 一次、IPv6 一次）。**如果 IPv6 注册失败，必须回滚 IPv4 的注册**（第 566 行）——这是事务性注册的标准模式。

#### 5.1.2 内层 `__nf_register_net_hook`

文件：`net/netfilter/core.c:389-452`

```c
static int __nf_register_net_hook(struct net *net, int pf,
				  const struct nf_hook_ops *reg)
{
	struct nf_hook_entries *p, *new_hooks;
	struct nf_hook_entries __rcu **pp;
	int err;

	switch (pf) {
	case NFPROTO_NETDEV:
		/* ... 校验 ingress/egress 配置、dev 归属 ... */
		break;
	case NFPROTO_INET:
		if (reg->hooknum != NF_INET_INGRESS)
			break;
		err = nf_ingress_check(net, reg, NF_INET_INGRESS);
		if (err < 0)
			return err;
		break;
	}

	pp = nf_hook_entry_head(net, pf, reg->hooknum, reg->dev);
	if (!pp)
		return -EINVAL;

	mutex_lock(&nf_hook_mutex);

	p = nf_entry_dereference(*pp);
	new_hooks = nf_hook_entries_grow(p, reg);

	if (!IS_ERR(new_hooks)) {
		hooks_validate(new_hooks);
		rcu_assign_pointer(*pp, new_hooks);
	}

	mutex_unlock(&nf_hook_mutex);
	if (IS_ERR(new_hooks))
		return PTR_ERR(new_hooks);

#ifdef CONFIG_NETFILTER_INGRESS
	if (nf_ingress_hook(reg, pf))
		net_inc_ingress_queue();
#endif
#ifdef CONFIG_NETFILTER_EGRESS
	if (nf_egress_hook(reg, pf))
		net_inc_egress_queue();
#endif
	nf_static_key_inc(reg, pf);

	BUG_ON(p == new_hooks);
	nf_hook_entries_free(p);
	return 0;
}
```

逐段注解：

| 步骤 | 代码 | 说明 |
|------|------|------|
| 1. 参数校验 | `switch (pf)` | NETDEV/INET ingress 需要绑 dev，校验 `reg->dev` 归属当前 netns |
| 2. 定位 hook 头 | `pp = nf_hook_entry_head(net, pf, hooknum, dev)` | 返回 `&net->nf.hooks_ipv4[hooknum]` 这类指针的地址（`__rcu **`）。注意是**指针的地址**，不是指针本身——后面要 `rcu_assign_pointer` 替换它指向的内容 |
| 3. 加互斥锁 | `mutex_lock(&nf_hook_mutex)` | **全局**互斥锁，所有协议族共用一把。注册/注销是低频操作，不必精细化。读路径（报文）不碰这把锁 |
| 4. 取旧数组 | `p = nf_entry_dereference(*pp)` | `nf_entry_dereference` 是 `rcu_dereference_protected(..., lockdep_is_held(&nf_hook_mutex))`——断言已持锁，非 RCU 读侧 |
| 5. 生长新数组 | `new_hooks = nf_hook_entries_grow(p, reg)` | 核心：分配 N+1 大小的新数组，按 priority 升序插入 `reg`。见 §5.1.3 |
| 6. 校验 | `hooks_validate(new_hooks)` | DEBUG 模式下检查 priority 是否升序，生产环境是空函数 |
| 7. RCU 替换 | `rcu_assign_pointer(*pp, new_hooks)` | **原子地**把 `net->nf.hooks_ipv4[hook]` 指向新数组。这一刻起，新来的报文读到的是新数组；已经在读旧数组的报文继续读旧数组（RCU 保证） |
| 8. 解锁 | `mutex_unlock(&nf_hook_mutex)` | 锁只保护"生长 + 替换"这一步，释放旧数组在锁外 |
| 9. 更新计数 | `net_inc_ingress_queue` 等 | ingress/egress hook 计数，供 `netdev_get_xdp_slave` 等查询 |
| 10. 静态键 | `nf_static_key_inc(reg, pf)` | 见 §4.5。若是该关卡首个 hook，启用 `nf_hooks_needed[pf][hooknum]` 静态键 |
| 11. 释放旧数组 | `nf_hook_entries_free(p)` | **不是立即释放**，而是 `call_rcu`——等 RCU 宽限期结束（所有读者退出）后才真正 `kvfree`。见 §5.1.4 |

**`BUG_ON(p == new_hooks)`**：防御性断言。`nf_hook_entries_grow` 每次都分配新内存，不可能返回旧指针。如果触发说明内核内存损坏。

#### 5.1.3 `nf_hook_entries_grow`：按 priority 插入

文件：`net/netfilter/core.c:96-167`

```c
static struct nf_hook_entries *
nf_hook_entries_grow(const struct nf_hook_entries *old,
		     const struct nf_hook_ops *reg)
{
	unsigned int i, alloc_entries, nhooks, old_entries;
	struct nf_hook_ops **orig_ops = NULL;
	struct nf_hook_ops **new_ops;
	struct nf_hook_entries *new;
	bool inserted = false;

	alloc_entries = 1;
	old_entries = old ? old->num_hook_entries : 0;

	if (old) {
		orig_ops = nf_hook_entries_get_hook_ops(old);

		for (i = 0; i < old_entries; i++) {
			if (orig_ops[i] != &dummy_ops)
				alloc_entries++;

			/* BPF hook 必须用唯一 priority，避免两个 BPF 程序顺序歧义 */
			if (reg->priority == orig_ops[i]->priority &&
			    reg->hook_ops_type == NF_HOOK_OP_BPF)
				return ERR_PTR(-EBUSY);
		}
	}

	if (alloc_entries > MAX_HOOK_COUNT)
		return ERR_PTR(-E2BIG);

	new = allocate_hook_entries_size(alloc_entries);
	if (!new)
		return ERR_PTR(-ENOMEM);

	new_ops = nf_hook_entries_get_hook_ops(new);

	i = 0;
	nhooks = 0;
	while (i < old_entries) {
		if (orig_ops[i] == &dummy_ops) {
			++i;
			continue;
		}

		if (inserted || reg->priority > orig_ops[i]->priority) {
			new_ops[nhooks] = (void *)orig_ops[i];
			new->hooks[nhooks] = old->hooks[i];
			i++;
		} else {
			new_ops[nhooks] = (void *)reg;
			new->hooks[nhooks].hook = reg->hook;
			new->hooks[nhooks].priv = reg->priv;
			inserted = true;
		}
		nhooks++;
	}

	if (!inserted) {
		new_ops[nhooks] = (void *)reg;
		new->hooks[nhooks].hook = reg->hook;
		new->hooks[nhooks].priv = reg->priv;
	}

	return new;
}
```

**关键设计**：

1. **跳过 `dummy_ops`**：注销时旧 hook 不会被立即删除，而是被替换成 `dummy_ops`（见 §5.2），等下次 `shrink` 才真正移除。所以 grow 时要跳过这些 dummy 槽位，相当于"顺带压缩"。
2. **BPF hook 唯一 priority**：两个 BPF 程序同 priority 直接 `-EBUSY` 拒绝。普通 hook 同 priority 允许（按注册顺序），但 BPF 程序顺序不能有歧义。
3. **`MAX_HOOK_COUNT=1024`**：单关卡最多 1024 个 hook，防止恶意/bug 模块注册无限 hook 拖垮内核。
4. **插入逻辑**：`reg->priority > orig_ops[i]->priority` 时把旧 hook 拷到新数组；否则插入 `reg`。`!inserted` 分支处理 priority 最大（追加到末尾）的情况。

#### 5.1.4 旧数组释放：`nf_hook_entries_free`

文件：`net/netfilter/core.c:68-82`

```c
static void nf_hook_entries_free(struct nf_hook_entries *e)
{
	struct nf_hook_entries_rcu_head *head;
	struct nf_hook_ops **ops;
	unsigned int num;

	if (!e)
		return;

	num = e->num_hook_entries;
	ops = nf_hook_entries_get_hook_ops(e);
	head = (void *)&ops[num];
	head->allocation = e;
	call_rcu(&head->head, __nf_hook_entries_free);
}
```

**RCU 宽限期释放**：

```text
T0: rcu_assign_pointer(*pp, new)   ← 新报文读 new，老报文可能还在读 old
T1: call_rcu(old, __nf_hook_entries_free)
T2: ... RCU 宽限期（等所有读者退出）...
T3: __nf_hook_entries_free → kvfree(old)   ← 此时无任何读者，安全释放
```

**为什么必须 RCU 释放**：`nf_hook_slow` 在 RCU 读侧遍历 `e->hooks[]`。如果在遍历中途数组被释放，就是 use-after-free。RCU 保证：`rcu_assign_pointer` 后，只要经过一个 RCU 宽限期，所有"在替换前进入 RCU 临界区"的读者都已退出，此时释放旧数组才安全。

> **设计权衡**：这种"整体替换 + RCU 释放"的设计让读路径零开销，但写路径要分配/释放整块内存。对 iptables（规则少变动）友好，对"每秒增删规则"的场景不友好——这是 kube-proxy 后来弃用 iptables 模式的根因之一。

### 5.2 `nf_unregister_net_hook`：用 dummy 替换 + shrink

注销不走"分配 N-1 数组"，而是**两步走**：先把要删的 hook 替换成 `dummy_ops`（`accept_all`），再尝试 shrink。

#### 5.2.1 `nf_remove_net_hook`：标记删除

文件：`net/netfilter/core.c:463-479`

```c
static bool nf_remove_net_hook(struct nf_hook_entries *old,
			       const struct nf_hook_ops *unreg)
{
	struct nf_hook_ops **orig_ops;
	unsigned int i;

	orig_ops = nf_hook_entries_get_hook_ops(old);
	for (i = 0; i < old->num_hook_entries; i++) {
		if (orig_ops[i] != unreg)
			continue;
		WRITE_ONCE(old->hooks[i].hook, accept_all);
		WRITE_ONCE(orig_ops[i], (void *)&dummy_ops);
		return true;
	}

	return false;
}
```

**关键设计**：不是重新分配数组，而是**原地修改**：

- `WRITE_ONCE(old->hooks[i].hook, accept_all)`：把回调改成 `accept_all`（永远返回 `NF_ACCEPT`）。这样正在遍历这个数组的报文看到的是"放行"，不会调到已注销的回调。
- `WRITE_ONCE(orig_ops[i], &dummy_ops)`：标记这个槽位为 dummy，下次 grow/shrink 时清理。

**为什么这样设计**：注销是高频操作（比如 `iptables -F` 清空规则），如果每次都分配新数组，内存分配压力太大。原地替换成 dummy，等下次注册/注销时顺带 shrink（`__nf_hook_entries_try_shrink`）。

#### 5.2.2 `__nf_unregister_net_hook`

文件：`net/netfilter/core.c:481-520`

```c
static void __nf_unregister_net_hook(struct net *net, int pf,
				     const struct nf_hook_ops *reg)
{
	struct nf_hook_entries __rcu **pp;
	struct nf_hook_entries *p;

	pp = nf_hook_entry_head(net, pf, reg->hooknum, reg->dev);
	if (!pp)
		return;

	mutex_lock(&nf_hook_mutex);

	p = nf_entry_dereference(*pp);
	if (WARN_ON_ONCE(!p)) {
		mutex_unlock(&nf_hook_mutex);
		return;
	}

	if (nf_remove_net_hook(p, reg)) {
		/* ... 更新 ingress/egress 计数 ... */
		nf_static_key_dec(reg, pf);
	} else {
		WARN_ONCE(1, "hook not found, pf %d num %d", pf, reg->hooknum);
	}

	p = __nf_hook_entries_try_shrink(p, pp);
	mutex_unlock(&nf_hook_mutex);
	if (!p)
		return;

	nf_queue_nf_hook_drop(net);
	nf_hook_entries_free(p);
}
```

流程：标记删除（`nf_remove_net_hook`）→ 减静态键计数（`nf_static_key_dec`）→ 尝试 shrink（`__nf_hook_entries_try_shrink`，若 dummy 过多则分配新数组替换）→ 释放旧数组（`nf_hook_entries_free`，RCU 延迟释放）。

**`nf_queue_nf_hook_drop`**：注销时若仍有 skb 在 nfqueue 中等待用户态裁决，这些 skb 引用了已注销的 hook 上下文，必须丢弃。这是清理"悬挂的队列报文"。

### 5.3 RCU 替换模型总结

```text
注册/注销时序：
┌─────────────────────────────────────────────────────────────┐
│ Writer (持 nf_hook_mutex)                                   │
│  1. nf_hook_entries_grow: 分配新数组 new (N+1)              │
│  2. rcu_assign_pointer(*pp, new)  ← 原子替换，瞬间生效      │
│  3. mutex_unlock                                             │
│  4. call_rcu(old, free)  ← 提交延迟释放                     │
└─────────────────────────────────────────────────────────────┘
                          │
                          │  RCU 宽限期
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Reclaimer (软中断)                                          │
│  - 所有 T0 前进入 RCU 读侧的读者已退出                       │
│  - __nf_hook_entries_free: kvfree(old)                      │
└─────────────────────────────────────────────────────────────┘

Reader (nf_hook_slow, 持 rcu_read_lock):
  - T0 前: 读 old 数组 → 在 RCU 读侧，old 不会被释放
  - T0 后: 读 new 数组
  - 绝不会看到"半新半旧"——rcu_assign_pointer 是原子的
```

**三层保证**：

1. **原子性**：`rcu_assign_pointer` 保证读者要么看到 old，要么看到 new，不会看到中间态。
2. **可见性**：RCU 宽限期保证 `call_rcu` 的回调在所有旧读者退出后才执行。
3. **无锁读**：读者不获取 `nf_hook_mutex`，只持 `rcu_read_lock`（开销极低，软中断里也能用）。

### 5.4 常见注册来源

| 来源 | hook_ops_type | 典型模块 |
|------|---------------|----------|
| iptables (xtables) | `NF_HOOK_OP_UNDEFINED` | `iptable_filter`/`iptable_nat`/`iptable_mangle`（每个表注册一个 hook，回调里跑 `ipt_do_table`） |
| nftables | `NF_HOOK_OP_NF_TABLES` | `nf_tables_chain_type` |
| BPF | `NF_HOOK_OP_BPF` | `BPF_PROG_TYPE_NETFILTER`（较新） |
| 内核内置模块 | — | `nf_conntrack`（注册 PRE_ROUTING/LOCAL_OUT 做 conntrack）、`nf_nat`、`iptable_raw` |

**举例**：`iptable_filter` 模块初始化时注册 3 个 hook（INPUT/FORWARD/OUTPUT），每个 hook 的回调都是 `ipt_do_table`，`priv` 指向该表的 `xt_table` 结构。报文经过关卡时，`nf_hook_slow` 调 `ipt_do_table(skb, state, table)` 跑规则——见 §6。

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

文件：`net/ipv4/netfilter/ip_tables.c:222-362`

**这是 iptables 的热路径核心**——每张表注册到 Netfilter hook 时，回调函数都是 `ipt_do_table`，`priv` 指向该表的 `xt_table`。报文经过关卡时，`nf_hook_slow` 调用此函数遍历表内规则。

源码（核心循环部分）：

```c
unsigned int
ipt_do_table(void *priv,
	     struct sk_buff *skb,
	     const struct nf_hook_state *state)
{
	const struct xt_table *table = priv;
	unsigned int hook = state->hook;
	/* ... 初始化 ... */
	struct ipt_entry *e, **jumpstack;
	const struct xt_table_info *private;

	/* ... 设置 acpar.fragoff/thoff/hotdrop/state ... */

	local_bh_disable();
	addend = xt_write_recseq_begin();
	private = READ_ONCE(table->private); /* Address dependency. */
	cpu        = smp_processor_id();
	table_base = private->entries;
	jumpstack  = (struct ipt_entry **)private->jumpstack[cpu];

	e = get_entry(table_base, private->hook_entry[hook]);

	do {
		const struct xt_entry_target *t;
		const struct xt_entry_match *ematch;
		struct xt_counters *counter;

		if (!ip_packet_match(ip, indev, outdev,
		    &e->ip, acpar.fragoff)) {
 no_match:
			e = ipt_next_entry(e);
			continue;
		}

		xt_ematch_foreach(ematch, e) {
			acpar.match     = ematch->u.kernel.match;
			acpar.matchinfo = ematch->data;
			if (!acpar.match->match(skb, &acpar))
				goto no_match;
		}

		counter = xt_get_this_cpu_counter(&e->counters);
		ADD_COUNTER(*counter, skb->len, 1);

		t = ipt_get_target_c(e);
		WARN_ON(!t->u.kernel.target);

		/* Standard target? */
		if (!t->u.kernel.target->target) {
			int v;

			v = ((struct xt_standard_target *)t)->verdict;
			if (v < 0) {
				/* Pop from stack? */
				if (v != XT_RETURN) {
					verdict = (unsigned int)(-v) - 1;
					break;
				}
				if (stackidx == 0) {
					e = get_entry(table_base,
					    private->underflow[hook]);
				} else {
					e = jumpstack[--stackidx];
					e = ipt_next_entry(e);
				}
				continue;
			}
			if (table_base + v != ipt_next_entry(e) &&
			    !(e->ip.flags & IPT_F_GOTO)) {
				if (unlikely(stackidx >= private->stacksize)) {
					verdict = NF_DROP;
					break;
				}
				jumpstack[stackidx++] = e;
			}

			e = get_entry(table_base, v);
			continue;
		}

		acpar.target   = t->u.kernel.target;
		acpar.targinfo = t->data;

		verdict = t->u.kernel.target->target(skb, &acpar);
		if (verdict == XT_CONTINUE) {
			/* Target might have changed stuff. */
			ip = ip_hdr(skb);
			e = ipt_next_entry(e);
		} else {
			/* Verdict */
			break;
		}
	} while (!acpar.hotdrop);

	xt_write_recseq_end(addend);
	local_bh_enable();

	if (acpar.hotdrop)
		return NF_DROP;
	else return verdict;
}
```

逐段注解：

| 行 | 代码 | 说明 |
|----|------|------|
| 227 | `const struct xt_table *table = priv` | `priv` 是 `nf_hook_ops.priv`，表初始化时设为该表的 `xt_table`。一个 hook 回调对应一张表 |
| 258-259 | `local_bh_disable()` / `xt_write_recseq_begin()` | **禁软中断 + 序列锁读开始**。iptables 规则替换（`iptables -A/-F`）用 `xt_write_recseq` 做读多写少同步，这里读侧。禁软中断是因为软中断（NAPI）会调到这里 |
| 260 | `private = READ_ONCE(table->private)` | 取规则集。`READ_ONCE` 保证地址依赖——替换规则时 `rcu_assign_pointer` 更新此指针 |
| 261-263 | `cpu / table_base / jumpstack` | **per-cpu jumpstack**：JUMP 跳转栈是 per-cpu 的，避免多 CPU 互踩。TEE target 做包复制时会切换备用栈（第 272 行） |
| 275 | `e = get_entry(table_base, private->hook_entry[hook])` | 定位起始规则。每张表为每个 hook 存起始偏移（`hook_entry[hook]`）——例如 filter 表 INPUT 链从 `hook_entry[NF_INET_LOCAL_IN]` 开始 |
| 283-288 | `ip_packet_match(ip, indev, outdev, &e->ip, fragoff)` | **第一级匹配**：IP 头基本字段（src/dst/proto/iniface/outiface）。这是内联快速检查，不匹配直接下一条 |
| 290-295 | `xt_ematch_foreach(ematch, e) { if (!match->match(...)) goto no_match; }` | **第二级匹配**：遍历规则的扩展 match（tcp/udp/state/conntrack/multiport...）。任一不匹配则整条规则不匹配 |
| 297-298 | `ADD_COUNTER(*counter, skb->len, 1)` | 匹配成功，更新规则计数器（`iptables -L -v` 看到的 pkts/bytes） |
| 300 | `t = ipt_get_target_c(e)` | 取规则的 target。target 分两类：standard（内置）和 extension（模块） |
| 310 | `if (!t->u.kernel.target->target)` | **判断是否标准 target**：标准 target 的 `->target` 函数指针为 NULL（用 verdict 字段直接表示 ACCEPT/DROP/RETURN/JUMP），扩展 target 有具体函数 |
| 313-318 | `v = ((xt_standard_target *)t)->verdict; if (v < 0) { if (v != XT_RETURN) { verdict = -v - 1; break; } }` | **verdict 编码**：负值表示终止性裁决。`-NF_ACCEPT-1`、`-NF_DROP-1` 等（-1 偏移是为区分 0）。`XT_RETURN`（也是负值）表示从自定义链返回 |
| 320-326 | `if (stackidx == 0) e = underflow[hook]; else e = jumpstack[--stackidx]` | **链返回**：栈空则跳到链底（underflow，即默认 policy），否则弹栈回到跳转点继续 |
| 329-336 | `jumpstack[stackidx++] = e` | **JUMP 入栈**：跳到自定义链前，把当前规则压栈，便于 RETURN 时回来。`IPT_F_GOTO` 标志的 `-g`（goto）不入栈（不 RETURN） |
| 338 | `e = get_entry(table_base, v)` | JUMP 到偏移 v |
| 342-345 | `acpar.target = ...; verdict = t->u.kernel.target->target(skb, &acpar)` | **扩展 target 执行**：如 REJECT/LOG/MASQUERADE/DNAT/SNAT。target 返回最终 verdict |
| 346-353 | `if (verdict == XT_CONTINUE) e = ipt_next_entry(e); else break` | target 返回 `XT_CONTINUE` 表示"继续下一条规则"（如 LOG 记录后不终止）；其他值（NF_ACCEPT/NF_DROP）则 break 结束循环 |
| 354 | `} while (!acpar.hotdrop)` | **hotdrop**：match/target 可设此标志立即 DROP（如检测到异常报文），无需等规则链跑完 |
| 356-357 | `xt_write_recseq_end / local_bh_enable` | 退出临界区 |

**规则结构布局**（理解循环的关键）：

```text
table->private->entries (连续内存块)
┌─────────────────────────────────────────────────────────┐
│ ipt_entry (规则0)                                       │
│   ├─ ip: src/dst/proto/iniface/outiface (第一级匹配)    │
│   ├─ nft_entry_match[0]: tcp match (第二级匹配)         │
│   ├─ nft_entry_match[1]: state match                    │
│   └─ xt_entry_target: ACCEPT/DROP/JUMP/扩展 target      │
├─────────────────────────────────────────────────────────┤
│ ipt_entry (规则1)                                       │
│   ...                                                   │
├─────────────────────────────────────────────────────────┤
│ ...                                                     │
├─────────────────────────────────────────────────────────┤
│ underflow (链底): 标准 target verdict = -NF_ACCEPT-1    │
│   (默认 policy，规则全不匹配时到这里)                    │
└─────────────────────────────────────────────────────────┘
```

**verdict 编码**：

| 用户视角 | 内部存储 | 含义 |
|----------|----------|------|
| `-j ACCEPT` | `verdict = -NF_ACCEPT - 1 = -2` | 终止，放行 |
| `-j DROP` | `verdict = -NF_DROP - 1 = -1` | 终止，丢弃 |
| `-j RETURN` | `verdict = XT_RETURN (-NF_REPEAT - 1 = -5)` | 从当前链返回（`NF_REPEAT=4`，复用其值定义 RETURN） |
| `-j CUSTOM_CHAIN` | `verdict = 正偏移` (非负) | JUMP 到偏移 |
| `-g CUSTOM_CHAIN` | `verdict = 正偏移` + `IPT_F_GOTO` | GOTO（不 RETURN） |

负值=终止性裁决，非负=跳转。`-1` 偏移是为了让 0 也是有效偏移。

### 6.3 性能特征与瓶颈

1. **O(N) 线性匹配**：规则按顺序遍历，匹配到第一条命中即执行 target。N 条规则最坏要扫 N 次。
2. **per-cpu jumpstack + 序列锁**：读路径无锁（`xt_write_recseq` 读侧），写路径（替换规则）要等所有读者退出。规则替换是低频操作，这套设计对"规则稳定 + 高 PPS"友好。
3. **`ip_packet_match` 快速预筛**：IP 头字段匹配是内联的，比扩展 match 快。把高频规则的 IP 字段填全（而非用 `0.0.0.0/0`）能减少扩展 match 调用。
4. **`hotdrop` 逃生**：match 发现致命异常（如 truncated header）直接设 hotdrop，避免后续规则白跑。

**K8s kube-proxy iptables 模式的性能问题**：一个 Service 生成一条 `-j KUBE-SVC-xxx` 链，该链再 JUMP 到多个 `KUBE-SEP-xxx`（后端 pod）链，用 `statistic` match 做随机。上千 Service 时规则数爆炸，每次新连接首包要线性扫过大量链。这是后来 IPVS 模式（hash 查找）和 eBPF 模式（map 查找）出现的根本原因。

> **对比 nftables**：nftables 用 map/set 做 O(1) 查找，且规则编译成字节码由统一解释器执行（无函数指针间接跳转），cache 友好。这就是 §9 要讲的内容。

---

## 7. 连接跟踪（conntrack）：状态机与表

### 7.1 conntrack 解决的问题

iptables 的 `-m state --state ESTABLISHED` / `-m conntrack --ctstate ...` 能写出"只放行已建立连接的回包"这种规则，靠的就是 conntrack。它为每条流（双向五元组）维护一个 `struct nf_conn`，记录状态、超时、NAT 映射。

**conntrack 不是一个独立 hook，而是挂在 PRE_ROUTING 和 LOCAL_OUT 两个关卡、priority=`NF_IP_PRI_CONNTRACK=-200` 的 hook**。注意它早于 filter 表（priority=0），晚于 raw 表（priority=-300）——所以 raw 表的 `NOTRACK` 能让包跳过 conntrack。

### 7.2 conntrack 的 4 个 hook 注册点

文件：`net/netfilter/nf_conntrack_proto.c:229-254`

```c
/* Connection tracking may drop packets, but never alters them, so
 * make it the first hook. */
static const struct nf_hook_ops ipv4_conntrack_ops[] = {
	{
		.hook		= ipv4_conntrack_in,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_PRE_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK,        /* -200 */
	},
	{
		.hook		= ipv4_conntrack_local,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_OUT,
		.priority	= NF_IP_PRI_CONNTRACK,        /* -200 */
	},
	{
		.hook		= nf_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_POST_ROUTING,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM, /* INT_MAX，最后 */
	},
	{
		.hook		= nf_confirm,
		.pf		= NFPROTO_IPV4,
		.hooknum	= NF_INET_LOCAL_IN,
		.priority	= NF_IP_PRI_CONNTRACK_CONFIRM, /* INT_MAX，最后 */
	},
};
```

**为什么是 4 个 hook，而不是 1 个？** conntrack 分两个阶段：

| 阶段 | hook | 时机 | 作用 |
|------|------|------|------|
| **创建/查找** | `ipv4_conntrack_in` @ PRE_ROUTING | 收包早期 | 对收到的包查/建 conntrack |
| **创建/查找** | `ipv4_conntrack_local` @ LOCAL_OUT | 本机发包早期 | 对本机产生的包查/建 conntrack |
| **确认** | `nf_confirm` @ POST_ROUTING | 发包最末 | 包真的要离开本机了，把 conntrack 从"未确认"链表移到主 hash 表，永久生效 |
| **确认** | `nf_confirm` @ LOCAL_IN | 本机收包最末 | 对本机接收的包也确认（本机回环也算） |

**两阶段设计的意义**：`ipv4_conntrack_in` 创建的 conntrack 先挂在"unconfirmed"链表（per-cpu），**此时还不算正式连接**。只有当包成功穿过协议栈到达 `nf_confirm`（没被 filter 丢），才移入全局 hash 表——这避免了"建了连接却被 filter 丢包"导致 hash 表被垃圾条目塞满。

### 7.3 核心数据结构

#### 7.3.1 `struct nf_conntrack_tuple` —— 单向五元组

一条连接有 **两个** tuple：original（发起方向）+ reply（回应方向）。查 hash 表时，用收到的包构造 tuple，匹配 original 或 reply 任一即可。

```
original tuple:  src=客户端IP:port  dst=服务端IP:port  proto=TCP
reply tuple:     src=服务端IP:port  dst=客户端IP:port  proto=TCP  (src/dst 颠倒)
```

#### 7.3.2 `struct nf_conn` —— 一条连接的跟踪条目

文件：`include/net/netfilter/nf_conntrack.h:74-124`

```c
struct nf_conn {
	struct nf_conntrack ct_general;        /* 内嵌引用计数（skb->_nfct 指向这里） */

	spinlock_t	lock;
	u32 timeout;                           /* jiffies32，到期则被 GC 回收 */

#ifdef CONFIG_NF_CONNTRACK_ZONES
	struct nf_conntrack_zone zone;         /* conntrack zone，隔离不同网络 */
#endif
	/* These are my tuples; original and reply */
	struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];  /* 2 个 tuple */

	/* Have we seen traffic both ways yet? (bitset) */
	unsigned long status;                  /* IPS_* 状态位，见 §7.4 */

	possible_net_t ct_net;

#if IS_ENABLED(CONFIG_NF_NAT)
	struct hlist_node nat_bysource;        /* 挂到 NAT by-source hash */
#endif

	struct nf_conn *master;                /* 若是 expected 连接，指向主连接 */

	struct nf_ct_ext *ext;                 /* 扩展区：helper/timeout/nat/seqadj... */

	union nf_conntrack_proto proto;         /* 协议私有数据（TCP 状态机等） */
};
```

**关键字段**：

| 字段 | 作用 |
|------|------|
| `ct_general` | 内嵌引用计数。`skb->_nfct` 低 3 位存 `ctinfo`，高位存 `nf_conn` 指针——一个字段同时存"指向哪条连接"和"在这个连接里处于什么位置" |
| `tuplehash[2]` | 双向 tuple。`IP_CT_DIR_ORIGINAL=0`、`IP_CT_DIR_REPLY=1` |
| `status` | `IPS_*` 位图（见 §7.4），记录连接生命周期状态 |
| `timeout` | 到期时间。各协议有不同的超时表（TCP ESTABLISHED 5 天，TIME_WAIT 2 分钟等） |
| `master` | FTP 数据连接、ICMP 差错报文等"相关连接"指向其主连接 |
| `proto` | 协议私有数据。TCP 在这里存 `enum tcp_conntrack` 状态机（SYN_SENT/SYN_RECV/ESTABLISHED/...） |

#### 7.3.3 `enum ip_conntrack_info` —— skb 在连接中的位置

文件：`include/uapi/linux/netfilter/nf_conntrack_common.h:7-35`

```c
enum ip_conntrack_info {
	IP_CT_ESTABLISHED,      /* 0: 已建立连接（任一方向） */
	IP_CT_RELATED,          /* 1: 相关连接（如 FTP 数据、ICMP 差错） */
	IP_CT_NEW,              /* 2: 新连接的首包 */
	IP_CT_IS_REPLY,         /* 3: 分界线，>=此值表示 reply 方向 */
	IP_CT_ESTABLISHED_REPLY = IP_CT_ESTABLISHED + IP_CT_IS_REPLY,  /* 3+0=3 */
	IP_CT_RELATED_REPLY = IP_CT_RELATED + IP_CT_IS_REPLY,          /* 3+1=4 */
	/* No NEW in reply direction. */
	IP_CT_NUMBER,           /* 5: 不同 ctinfo 类型的数量 */
	IP_CT_UNTRACKED = 7,    /* 未被跟踪（raw 表 NOTRACK） */
};
```

**设计要点**：

- `ctinfo` 同时编码"连接状态"和"方向"。`ctinfo % IP_CT_IS_REPLY` 取状态部分，`ctinfo >= IP_CT_IS_REPLY` 判断方向。
- 这让 iptables `-m state` 的 `--state ESTABLISHED` 能用 `(ctinfo == IP_CT_ESTABLISHED || ctinfo == IP_CT_ESTABLISHED_REPLY)` 一次性匹配双向。
- `IP_CT_UNTRACKED=7`：raw 表 `NOTRACK` 标记的包，跳过 conntrack，`skb->_nfct` 被设成特殊值。

### 7.4 `IPS_*` 状态位：连接的生命周期

文件：`include/uapi/linux/netfilter/nf_conntrack_common.h:42-130`

```c
enum ip_conntrack_status {
	IPS_EXPECTED    = (1 << 0),   /* 是 expected 连接（如 FTP 数据通道） */
	IPS_SEEN_REPLY  = (1 << 1),   /* 见过双向流量，连接真正建立 */
	IPS_ASSURED     = (1 << 2),   /* 连接稳固，不会被早期 GC 回收 */
	IPS_CONFIRMED   = (1 << 3),   /* 已确认（包已离开本机，移入主 hash 表） */
	IPS_SRC_NAT     = (1 << 4),   /* original 方向需 SNAT */
	IPS_DST_NAT     = (1 << 5),   /* original 方向需 DNAT */
	IPS_SEQ_ADJUST  = (1 << 6),   /* NAT 改了 TCP seq，需 seqadj 修正 */
	IPS_SRC_NAT_DONE= (1 << 7),   /* SNAT 已执行 */
	IPS_DST_NAT_DONE= (1 << 8),   /* DNAT 已执行 */
	IPS_DYING       = (1 << 9),   /* 正在被回收 */
	IPS_FIXED_TIMEOUT=(1 << 10),  /* 超时固定（如 UDP 流量） */
	IPS_OFFLOAD     = (1 << 14),  /* 已卸载到 flow table（旁路 conntrack） */
	IPS_HW_OFFLOAD  = (1 << 15),  /* 已卸载到硬件 */
};
```

**完整状态机图**（含 NAT、expected、确认）：

```text
                              首包到达 PRE_ROUTING/LOCAL_OUT
                                        │
                                        ▼
                            ipv4_conntrack_in / _local
                                        │
                          resolve_normal_ct 查/建 nf_conn
                                        │
                        ┌───────────────┴───────────────┐
                        │                               │
                   新建 conntrack                  找到已有 conntrack
                   status=0 (未确认)                (已确认)
                        │                               │
                        │ ctinfo=IP_CT_NEW              │ 见 §7.5 决策树
                        ▼                               │
              挂到 per-cpu unconfirmed 链表            │
                        │                               │
                        ▼                               │
                    包继续走协议栈                       │
                        │                               │
              ┌─────────┴─────────┐                     │
              │                   │                     │
         被 filter 丢弃       到达 nf_confirm            │
              │             (POST_ROUTING/LOCAL_IN)      │
              │                   │                     │
              ▼                   ▼                     │
       unconfirmed 链表      移入全局 hash 表             │
       随 skb 释放而回收     set IPS_CONFIRMED           │
                             (此时才算正式连接)          │
                                        │               │
                                        └───────┬───────┘
                                                │
                                  收到 reply 方向回包
                                                │
                                                ▼
                              resolve_normal_ct 匹配 reply tuple
                                                │
                                  set IPS_SEEN_REPLY
                                  ctinfo = IP_CT_ESTABLISHED_REPLY
                                                │
                                  ◄── 此后双向流量都 ctinfo=IP_CT_ESTABLISHED(_REPLY)
                                                │
                                  超时（协议相关）或 FIN/RST
                                                │
                                                ▼
                                      set IPS_DYING → GC 回收
                                                │
                                                ▼
                                      从 hash 表移除、kmem_cache_free
```

**关键转换点**：

| 转换 | 触发 | 设置位 | ctinfo 变化 |
|------|------|--------|-------------|
| 新建 → 确认 | `nf_confirm`，包即将离开 | `IPS_CONFIRMED` | `IP_CT_NEW` |
| 确认 → 见过回应 | 收到 reply 方向首包 | `IPS_SEEN_REPLY` | `IP_CT_ESTABLISHED` |
| 任意 → 相关 | 是 `master` 的 expected（FTP/ICMP） | `IPS_EXPECTED` | `IP_CT_RELATED` |
| 任意 → NAT | NAT target 决定 | `IPS_SRC_NAT`/`IPS_DST_NAT` | 不变 |
| 确认 → 死亡 | 超时或协议终止（FIN/RST） | `IPS_DYING` | — |

### 7.5 `nf_conntrack_in`：conntrack 主入口

文件：`net/netfilter/nf_conntrack_core.c:2012-2096`

```c
unsigned int
nf_conntrack_in(struct sk_buff *skb, const struct nf_hook_state *state)
{
	enum ip_conntrack_info ctinfo;
	struct nf_conn *ct, *tmpl;
	u_int8_t protonum;
	int dataoff, ret;

	tmpl = nf_ct_get(skb, &ctinfo);
	if (tmpl || ctinfo == IP_CT_UNTRACKED) {
		/* Previously seen (loopback or untracked)?  Ignore. */
		if ((tmpl && !nf_ct_is_template(tmpl)) ||
		     ctinfo == IP_CT_UNTRACKED)
			return NF_ACCEPT;
		skb->_nfct = 0;
	}

	/* rcu_read_lock()ed by nf_hook_thresh */
	dataoff = get_l4proto(skb, skb_network_offset(skb), state->pf, &protonum);
	if (dataoff <= 0) {
		NF_CT_STAT_INC_ATOMIC(state->net, invalid);
		ret = NF_ACCEPT;
		goto out;
	}

	if (protonum == IPPROTO_ICMP || protonum == IPPROTO_ICMPV6) {
		ret = nf_conntrack_handle_icmp(tmpl, skb, dataoff,
					       protonum, state);
		/* ... ICMP 特殊处理 ... */
	}
repeat:
	ret = resolve_normal_ct(tmpl, skb, dataoff, protonum, state);
	if (ret < 0) {
		/* Too stressed to deal. */
		NF_CT_STAT_INC_ATOMIC(state->net, drop);
		ret = NF_DROP;
		goto out;
	}

	ct = nf_ct_get(skb, &ctinfo);
	if (!ct) {
		/* Not valid part of a connection */
		NF_CT_STAT_INC_ATOMIC(state->net, invalid);
		ret = NF_ACCEPT;
		goto out;
	}

	ret = nf_conntrack_handle_packet(ct, skb, dataoff, ctinfo, state);
	/* ... 协议状态机处理（TCP/SYN_SENT 等）... */

	if (ctinfo == IP_CT_ESTABLISHED_REPLY &&
	    !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct->status))
		nf_conntrack_event_cache(IPCT_REPLY, ct);
	/* ... */
}
```

**主流程**：

1. **跳过已跟踪包**：`nf_ct_get` 读 `skb->_nfct`。若已是 `IP_CT_UNTRACKED`（raw 表 NOTRACK）或已跟踪（loopback 重入），直接 ACCEPT。
2. **解析 L4 协议**：`get_l4proto` 返回 L4 头偏移和协议号。失败则记 `invalid` 计数并 ACCEPT（不跟踪无法识别的包，而非丢弃）。
3. **ICMP 特殊路径**：ICMP 差错报文要关联到原连接（`RELATED`），单独处理。
4. **`resolve_normal_ct`**：查/建 conntrack，设置 `skb->_nfct`。见 §7.6。
5. **`nf_conntrack_handle_packet`**：调协议状态机（如 TCP 的 `tcp_packet()` 更新 `tcp_conntrack` 状态、检查窗口、判断 SYN/SYN-ACK/FIN）。
6. **设置 `IPS_SEEN_REPLY`**：收到 reply 方向首包时设置，连接从 NEW 转 ESTABLISHED。

### 7.6 `resolve_normal_ct`：查/建 conntrack 的核心

文件：`net/netfilter/nf_conntrack_core.c:1869-1932`

**这是 conntrack 最关键的函数**——决定一个包属于哪条连接，以及在该连接里处于什么位置。

```c
/* On success, returns 0, sets skb->_nfct | ctinfo */
static int
resolve_normal_ct(struct nf_conn *tmpl,
		  struct sk_buff *skb,
		  unsigned int dataoff,
		  u_int8_t protonum,
		  const struct nf_hook_state *state)
{
	const struct nf_conntrack_zone *zone;
	struct nf_conntrack_tuple tuple;
	struct nf_conntrack_tuple_hash *h;
	enum ip_conntrack_info ctinfo;
	struct nf_conntrack_zone tmp;
	u32 hash, zone_id, rid;
	struct nf_conn *ct;

	if (!nf_ct_get_tuple(skb, skb_network_offset(skb),
			     dataoff, state->pf, protonum, state->net,
			     &tuple))
		return 0;

	/* look for tuple match */
	zone = nf_ct_zone_tmpl(tmpl, skb, &tmp);

	zone_id = nf_ct_zone_id(zone, IP_CT_DIR_ORIGINAL);
	hash = hash_conntrack_raw(&tuple, zone_id, state->net);
	h = __nf_conntrack_find_get(state->net, zone, &tuple, hash);

	if (!h) {
		rid = nf_ct_zone_id(zone, IP_CT_DIR_REPLY);
		if (zone_id != rid) {
			u32 tmp = hash_conntrack_raw(&tuple, rid, state->net);
			h = __nf_conntrack_find_get(state->net, zone, &tuple, tmp);
		}
	}

	if (!h) {
		h = init_conntrack(state->net, tmpl, &tuple,
				   skb, dataoff, hash);
		if (!h)
			return 0;
		if (IS_ERR(h))
			return PTR_ERR(h);
	}
	ct = nf_ct_tuplehash_to_ctrack(h);

	/* It exists; we have (non-exclusive) reference. */
	if (NF_CT_DIRECTION(h) == IP_CT_DIR_REPLY) {
		ctinfo = IP_CT_ESTABLISHED_REPLY;
	} else {
		unsigned long status = READ_ONCE(ct->status);

		/* Once we've had two way comms, always ESTABLISHED. */
		if (likely(status & IPS_SEEN_REPLY))
			ctinfo = IP_CT_ESTABLISHED;
		else if (status & IPS_EXPECTED)
			ctinfo = IP_CT_RELATED;
		else
			ctinfo = IP_CT_NEW;
	}
	nf_ct_set(skb, ct, ctinfo);
	return 0;
}
```

**逐段注解**：

| 行 | 代码 | 说明 |
|----|------|------|
| 1885-1888 | `nf_ct_get_tuple(..., &tuple)` | 从 skb 解析出**当前包的 tuple**（src/dst/proto/端口）。这是查表的 key |
| 1891 | `zone = nf_ct_zone_tmpl(tmpl, skb, &tmp)` | 取 conntrack zone。zone 用于隔离不同网络（容器网络常用），不同 zone 的连接互不可见 |
| 1893-1895 | `hash = hash_conntrack_raw(&tuple, zone_id, ...)`<br>`h = __nf_conntrack_find_get(...)` | **第一次查找**：用 original 方向的 zone 算 hash，查全局表。`_get` 后缀表示查到会**增加引用计数** |
| 1897-1904 | `if (!h) { rid = ...; h = __nf_conntrack_find_get(...); }` | **第二次查找**：第一次没找到，可能是**反方向**的包（reply 方向）。用 reply 方向的 zone 重新算 hash 再查。两次查找对应 tuplehash[2] 的两个方向 |
| 1906-1913 | `if (!h) { h = init_conntrack(...); }` | **两次都没找到 → 新连接**。`init_conntrack` 分配 `nf_conn`、填充双向 tuple、挂到 unconfirmed 链表。返回 `tuplehash[IP_CT_DIR_ORIGINAL]` |
| 1914 | `ct = nf_ct_tuplehash_to_ctrack(h)` | 通过 `tuplehash` 反推 `nf_conn` 指针（`container_of`） |
| 1917-1918 | `if (NF_CT_DIRECTION(h) == IP_CT_DIR_REPLY)`<br>`ctinfo = IP_CT_ESTABLISHED_REPLY;` | **匹配到的是 reply tuple** → 这个包是"已建立连接的回应方向"。注意 reply 方向永远是 `ESTABLISHED_REPLY`，没有 `NEW_REPLY` |
| 1919-1928 | `else { status = ...; if (IPS_SEEN_REPLY) ESTABLISHED; else if (IPS_EXPECTED) RELATED; else NEW; }` | **匹配到 original tuple** → 根据 `status` 位进一步分类：见过回应就是 ESTABLISHED；是 expected（FTP 数据）就是 RELATED；都没见过就是 NEW（首包） |
| 1930 | `nf_ct_set(skb, ct, ctinfo)` | 把 `ct` 指针和 `ctinfo` 编码后存入 `skb->_nfct`。后续所有 hook/target 用 `nf_ct_get(skb, &ctinfo)` 一次性读出 |

**ctinfo 决策树**（背下来）：

```text
            resolve_normal_ct 查 tuple hash 表
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
    匹配 reply     匹配 original    都没匹配
    tuplehash        tuplehash      (新连接)
          │             │             │
          ▼             ▼             ▼
  ESTABLISHED   ┌─ IPS_SEEN_REPLY?  init_conntrack
     _REPLY     │   是 → ESTABLISHED   建新 nf_conn
                │   否 → ┌─ IPS_EXPECTED?  ctinfo=NEW
                │        │   是 → RELATED
                │        │   否 → NEW
```

**关键洞察**：

1. **reply 方向的包永远是 ESTABLISHED_REPLY**——因为只有 original 方向发过包（建立了 conntrack），才可能有 reply tuple 被匹配。这是 conntrack "自动识别回包"的本质。
2. **original 方向的包可能仍是 NEW**——如果是重传的首包，匹配到 original tuple，但还没见过 reply，ctinfo=NEW。
3. **`init_conntrack` 分配的新 conntrack 挂在 unconfirmed 链表**——此时不在全局 hash 表，其他 CPU 看不到。只有 `nf_confirm` 把它移入 hash 表后才正式生效。这避免了"建连接的包被后续 filter 丢掉"造成 hash 表污染。

### 7.7 conntrack 表的 hash 与容量管理

- **`nf_conntrack_hash`**：per-netns 全局 hash 表，key = `hash_conntrack_raw(tuple, zone_id)`。bucket 数 = `nf_conntrack_buckets` sysctl。
- **容量上限**：`nf_conntrack_max` sysctl，默认 = `buckets * 4`。`nf_conntrack_count` 接近上限时新连接被 DROP，日志 `nf_conntrack: table full, dropping packet`。
- **GC**：定时器 + 每次 `nf_conntrack_in` 触发的机会式 GC，扫描 hash 表清理 `timeout` 到期条目。
- **early drop**：表满时尝试驱逐最老的 `IPS_ASSURED` 未设置条目腾空间。
- **`IPS_OFFLOAD`**：连接被 `nf_flowtable` 卸载后，conntrack 不再处理其报文（旁路到 fast path），这是高吞吐场景的关键优化。

### 7.8 可观测性

```bash
# 当前条目数 / 上限
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 列出所有连接（original + reply tuple + 状态 + 超时）
conntrack -L

# 实时事件流（新建/更新/销毁）
conntrack -E

# 各协议超时参数
ls /proc/sys/net/netfilter/nf_conntrack_*_timeout_*
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established   # 默认 432000=5天
cat /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_time_wait    # 默认 120

# 统计：invalid/drop/insert/insert_failed
cat /proc/net/netfilter/nf_conntrack | head
```

**典型排障**：`dmesg | grep "table full"` → 提高 `nf_conntrack_max` 和 `nf_conntrack_buckets`（后者需重启生效，是 hash 表 bucket 数）。

---

## 8. NAT：`nf_nat_core`

### 8.1 NAT 与 conntrack 的关系

**NAT 不是独立 hook，而是挂在 conntrack 之上的附加信息**：

- NAT 没有自己的状态表，**复用 conntrack 的 `nf_conn`**。每条 `nf_conn` 可携带一个 `struct nf_conn_nat` 扩展，记录 NAT 映射。
- NAT hook（`nf_nat_inet_fn`）只在 conntrack hook 之后执行——它依赖 `skb->_nfct` 已被设置。
- **一条连接的 NAT 映射在首包时确定一次，之后所有包都按这个映射改**。这就是为什么 NAT 必须 stateful——它要记住"这条流把源地址从 A 改成了 B"。

NAT 的两个方向对应两个 hook 关卡：

```text
PREROUTING (DNAT)                   POSTROUTING (SNAT)
     │                                    │
     ▼                                    ▼
nf_nat_inet_fn                       nf_nat_inet_fn
maniptype = NF_NAT_MANIP_DST         maniptype = NF_NAT_MANIP_SRC
(改目的地址，进站)                    (改源地址，出站)
```

`HOOK2MANIP` 宏（`include/net/netfilter/nf_nat.h:19`）定下规则：

```c
/* SRC manip occurs POST_ROUTING or LOCAL_IN */
#define HOOK2MANIP(hooknum) ((hooknum) != NF_INET_POST_ROUTING && \
			     (hooknum) != NF_INET_LOCAL_IN)
```

即 **POST_ROUTING 和 LOCAL_IN 做 SRC manip（SNAT），其余关卡做 DST manip（DNAT）**。LOCAL_OUT 也做 DNAT（本机产生的包可被 DNAT）。

### 8.2 `nf_nat_inet_fn`：NAT 主入口

文件：`net/netfilter/nf_nat_core.c:897-970`

```c
unsigned int
nf_nat_inet_fn(void *priv, struct sk_buff *skb,
	       const struct nf_hook_state *state)
{
	struct nf_conn *ct;
	enum ip_conntrack_info ctinfo;
	struct nf_conn_nat *nat;
	/* maniptype == SRC for postrouting. */
	enum nf_nat_manip_type maniptype = HOOK2MANIP(state->hook);

	ct = nf_ct_get(skb, &ctinfo);
	if (!ct || in_vrf_postrouting(state))
		return NF_ACCEPT;

	nat = nfct_nat(ct);

	switch (ctinfo) {
	case IP_CT_RELATED:
	case IP_CT_RELATED_REPLY:
		/* Only ICMPs can be IP_CT_IS_REPLY.  Fallthrough */
	case IP_CT_NEW:
		/* Seen it before?  This can happen for loopback, retrans,
		 * or local packets.  */
		if (!nf_nat_initialized(ct, maniptype)) {
			struct nf_nat_lookup_hook_priv *lpriv = priv;
			struct nf_hook_entries *e = rcu_dereference(lpriv->entries);
			unsigned int ret;
			int i;

			if (!e)
				goto null_bind;

			for (i = 0; i < e->num_hook_entries; i++) {
				ret = e->hooks[i].hook(e->hooks[i].priv, skb,
						       state);
				if (ret != NF_ACCEPT)
					return ret;
				if (nf_nat_initialized(ct, maniptype))
					goto do_nat;
			}
null_bind:
			ret = nf_nat_alloc_null_binding(ct, state->hook);
			if (ret != NF_ACCEPT)
				return ret;
		} else {
			if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
				goto oif_changed;
		}
		break;
	default:
		/* ESTABLISHED */
		WARN_ON(ctinfo != IP_CT_ESTABLISHED &&
			ctinfo != IP_CT_ESTABLISHED_REPLY);
		if (nf_nat_oif_changed(state->hook, ctinfo, nat, state->out))
			goto oif_changed;
	}
do_nat:
	return nf_nat_packet(ct, ctinfo, state->hook, skb);

oif_changed:
	nf_ct_kill_acct(ct, ctinfo, skb);
	return NF_DROP;
}
```

**两阶段设计**——这是 NAT 代码最重要的结构：

| 阶段 | 时机 | 作用 |
|------|------|------|
| **首包：建立映射** | `IP_CT_NEW`/`IP_CT_RELATED` 且 `!nf_nat_initialized` | 跑 NAT 规则（`e->hooks[i].hook`），规则匹配后调 `nf_nat_setup_info` 把映射写入 `ct`，设 `IPS_*_NAT_DONE` |
| **后续包：应用映射** | `ESTABLISHED` 或已 `nf_nat_initialized` | 直接调 `nf_nat_packet` 按 `ct` 里记录的映射改包，不再查规则 |

逐段注解：

| 行 | 说明 |
|----|------|
| 905 | `maniptype = HOOK2MANIP(state->hook)`——PRE_ROUTING→DST manip(DNAT)，POST_ROUTING→SRC manip(SNAT) |
| 907 | `ct = nf_ct_get(skb, &ctinfo)`——读 conntrack。**NAT 完全依赖 conntrack**，没 ct 就直接 ACCEPT（不 NAT） |
| 918-922 | `case IP_CT_NEW/RELATED`——首包路径。RELATED 是 ICMP 差错/FTP 数据等关联连接 |
| 926 | `if (!nf_nat_initialized(ct, maniptype))`——检查这条连接的该方向 NAT 是否已设置。**首包=否，重传/已设置=是** |
| 935-942 | `for (i...) e->hooks[i].hook(...)`——遍历 NAT 规则 hook（即 iptables nat 表的规则）。规则匹配后通过 `nf_nat_setup_info` 标记 `ct`，`nf_nat_initialized` 变真，跳出循环 |
| 943-946 | `null_bind: nf_nat_alloc_null_binding`——**没规则匹配也要做"空绑定"**：给 conntrack 记录一个"无 NAT"的占位，避免后续包重复查规则。这是性能优化 |
| 956-961 | `default: ESTABLISHED`——已建立连接的后续包，跳过规则查询 |
| 964 | `return nf_nat_packet(...)`——**真正改包的地方**，见 §8.4 |

**关键洞察**：NAT 规则（`iptables -t nat -A`）**只对每条连接的首包执行一次**。首包确定映射后写入 `ct->status` 的 `IPS_*_NAT_DONE` 位，后续包看到这个位直接跳过规则查询——规则匹配是 per-flow-first-packet 而非 per-packet。

### 8.3 `nf_nat_setup_info`：建立 NAT 映射

文件：`net/netfilter/nf_nat_core.c:765-831`

```c
nf_nat_setup_info(struct nf_conn *ct,
		  const struct nf_nat_range2 *range,
		  enum nf_nat_manip_type maniptype)
{
	struct net *net = nf_ct_net(ct);
	struct nf_conntrack_tuple curr_tuple, new_tuple;

	/* Can't setup nat info for confirmed ct. */
	if (nf_ct_is_confirmed(ct))
		return NF_ACCEPT;

	WARN_ON(maniptype != NF_NAT_MANIP_SRC &&
		maniptype != NF_NAT_MANIP_DST);

	if (WARN_ON(nf_nat_initialized(ct, maniptype)))
		return NF_DROP;

	/* What we've got will look like inverse of reply. */
	nf_ct_invert_tuple(&curr_tuple,
			   &ct->tuplehash[IP_CT_DIR_REPLY].tuple);

	get_unique_tuple(&new_tuple, &curr_tuple, range, ct, maniptype);

	if (!nf_ct_tuple_equal(&new_tuple, &curr_tuple)) {
		struct nf_conntrack_tuple reply;

		/* Alter conntrack table so will recognize replies. */
		nf_ct_invert_tuple(&reply, &new_tuple);
		nf_conntrack_alter_reply(ct, &reply);

		/* Non-atomic: we own this at this point. */
		if (maniptype == NF_NAT_MANIP_SRC)
			ct->status |= IPS_SRC_NAT;
		else
			ct->status |= IPS_DST_NAT;

		if (nfct_help(ct) && !nfct_seqadj(ct))
			if (!nfct_seqadj_ext_add(ct))
				return NF_DROP;
	}

	if (maniptype == NF_NAT_MANIP_SRC) {
		unsigned int srchash;
		spinlock_t *lock;

		srchash = hash_by_src(net, nf_ct_zone(ct),
				      &ct->tuplehash[IP_CT_DIR_ORIGINAL].tuple);
		lock = &nf_nat_locks[srchash % CONNTRACK_LOCKS];
		spin_lock_bh(lock);
		hlist_add_head_rcu(&ct->nat_bysource,
				   &nf_nat_bysource[srchash]);
		spin_unlock_bh(lock);
	}

	/* It's done. */
	if (maniptype == NF_NAT_MANIP_DST)
		ct->status |= IPS_DST_NAT_DONE;
	else
		ct->status |= IPS_SRC_NAT_DONE;

	return NF_ACCEPT;
}
```

逐段注解：

| 行 | 代码 | 说明 |
|----|------|------|
| 773 | `if (nf_ct_is_confirmed(ct)) return NF_ACCEPT` | **只能对未确认的 ct 设置 NAT**。确认后 ct 已进全局 hash 表，改 tuple 会破坏一致性。所以 NAT 必须在 `nf_confirm` 之前（priority 更小）执行 |
| 779 | `if (WARN_ON(nf_nat_initialized(ct, maniptype))) return NF_DROP` | 防止重复设置同方向 NAT |
| 787-788 | `nf_ct_invert_tuple(&curr_tuple, &ct->tuplehash[REPLY].tuple)` | 取"当前 effective tuple"=reply tuple 的逆。NAT 要改的就是这个 tuple |
| 790 | `get_unique_tuple(&new_tuple, &curr_tuple, range, ct, maniptype)` | 在 `range` 约束下选一个**可用**的新 tuple（避免端口冲突，`find_free_ipport` 等） |
| 792-808 | `if (!nf_ct_tuple_equal(&new_tuple, &curr_tuple))` | 新旧 tuple 不同才需要 NAT。**改 reply tuple**（`nf_conntrack_alter_reply`），后续 reply 方向的包靠这个新 tuple 匹配。设 `IPS_SRC_NAT`/`IPS_DST_NAT` 位 |
| 805-807 | `nfct_seqadj_ext_add` | 若连接有 helper（FTP 等）且 NAT 改了地址，TCP seq 需要调整（`IPS_SEQ_ADJUST`），否则对端因 seq 不匹配而 RST |
| 810-821 | `hlist_add_head_rcu(&ct->nat_bysource, &nf_nat_bysource[srchash])` | **SNAT 专用**：把 ct 挂到 by-source hash 表。这个表用于"给定源地址，反查已建立的 NAT 连接"，MASQUERADE 反向查找、端口复用都靠它 |
| 824-827 | `ct->status |= IPS_*_NAT_DONE` | 标记该方向 NAT 已完成。`nf_nat_inet_fn` 后续包靠这个位跳过规则查询 |

**核心机制**：NAT 不是改 skb，而是**改 conntrack 的 reply tuple**。改完后：

- **original 方向的包**：`nf_nat_packet` 根据 `ct->tuplehash[!dir]`（reply 方向）逆推出目标 tuple，改 skb 的 src/dst。
- **reply 方向的包**：因为 reply tuple 已被改，`resolve_normal_ct` 用收到的包构造 tuple 能匹配到这条 conntrack——**这就是 NAT 回包能自动找到连接的原因**。

### 8.4 `nf_nat_packet`：真正改包

文件：`net/netfilter/nf_nat_core.c:860-885`

```c
/* Do packet manipulations according to nf_nat_setup_info. */
unsigned int nf_nat_packet(struct nf_conn *ct,
			   enum ip_conntrack_info ctinfo,
			   unsigned int hooknum,
			   struct sk_buff *skb)
{
	enum nf_nat_manip_type mtype = HOOK2MANIP(hooknum);
	enum ip_conntrack_dir dir = CTINFO2DIR(ctinfo);
	unsigned int verdict = NF_ACCEPT;
	unsigned long statusbit;

	if (mtype == NF_NAT_MANIP_SRC)
		statusbit = IPS_SRC_NAT;
	else
		statusbit = IPS_DST_NAT;

	/* Invert if this is reply dir. */
	if (dir == IP_CT_DIR_REPLY)
		statusbit ^= IPS_NAT_MASK;

	/* Non-atomic: these bits don't change. */
	if (ct->status & statusbit)
		verdict = nf_nat_manip_pkt(skb, ct, mtype, dir);

	return verdict;
}
```

**这个函数决定"这个包要不要改、改哪个方向"**：

| 行 | 说明 |
|----|------|
| 865 | `mtype = HOOK2MANIP(hooknum)`——当前关卡对应的方向。POST_ROUTING→SRC，PRE_ROUTING→DST |
| 866 | `dir = CTINFO2DIR(ctinfo)`——这个包在连接里是 original 还是 reply 方向 |
| 870-873 | `statusbit = IPS_SRC_NAT / IPS_DST_NAT`——当前关卡默认检查的 NAT 位 |
| 876-877 | `if (dir == IP_CT_DIR_REPLY) statusbit ^= IPS_NAT_MASK` | **精妙设计**：reply 方向的包要改的是"相反"的地址。例如 SNAT 在 original 方向改源地址，reply 方向要改目的地址（把改后的地址改回原地址）。`^` 翻转 SRC/DST 位 |
| 880-881 | `if (ct->status & statusbit) verdict = nf_nat_manip_pkt(...)` | 只有这条连接确实配置了对应方向的 NAT 才改包。没配 NAT 的连接（如纯转发）直接 ACCEPT |

**举例**：`iptables -t nat -A POSTROUTING -j MASQUERADE`（SNAT）

- original 方向包经过 POST_ROUTING：`mtype=SRC`，`dir=ORIGINAL`，`statusbit=IPS_SRC_NAT`。ct 有此位 → 改源地址。
- reply 方向包经过 PRE_ROUTING：`mtype=DST`，`dir=REPLY`，`statusbit=IPS_DST_NAT ^ IPS_NAT_MASK = IPS_SRC_NAT`。ct 有 `IPS_SRC_NAT` → 改目的地址（还原）。

### 8.5 `nf_nat_manip_pkt` → `nf_nat_ipv4_manip_pkt`：改 IP/TCP/UDP 头

文件：`net/netfilter/nf_nat_proto.c:357-381` 和 `:291-317`

```c
unsigned int nf_nat_manip_pkt(struct sk_buff *skb, struct nf_conn *ct,
			      enum nf_nat_manip_type mtype,
			      enum ip_conntrack_dir dir)
{
	struct nf_conntrack_tuple target;

	/* We are aiming to look like inverse of other direction. */
	nf_ct_invert_tuple(&target, &ct->tuplehash[!dir].tuple);

	switch (target.src.l3num) {
	case NFPROTO_IPV6:
		if (nf_nat_ipv6_manip_pkt(skb, 0, &target, mtype))
			return NF_ACCEPT;
		break;
	case NFPROTO_IPV4:
		if (nf_nat_ipv4_manip_pkt(skb, 0, &target, mtype))
			return NF_ACCEPT;
		break;
	default:
		WARN_ON_ONCE(1);
		break;
	}

	return NF_DROP;
}
```

```c
static bool nf_nat_ipv4_manip_pkt(struct sk_buff *skb,
				  unsigned int iphdroff,
				  const struct nf_conntrack_tuple *target,
				  enum nf_nat_manip_type maniptype)
{
	struct iphdr *iph;
	unsigned int hdroff;

	if (skb_ensure_writable(skb, iphdroff + sizeof(*iph)))
		return false;

	iph = (void *)skb->data + iphdroff;
	hdroff = iphdroff + iph->ihl * 4;

	if (!l4proto_manip_pkt(skb, iphdroff, hdroff, target, maniptype))
		return false;
	iph = (void *)skb->data + iphdroff;

	if (maniptype == NF_NAT_MANIP_SRC) {
		csum_replace4(&iph->check, iph->saddr, target->src.u3.ip);
		iph->saddr = target->src.u3.ip;
	} else {
		csum_replace4(&iph->check, iph->daddr, target->dst.u3.ip);
		iph->daddr = target->dst.u3.ip;
	}
	return true;
}
```

**改包顺序**（重要）：

1. **`nf_ct_invert_tuple(&target, &ct->tuplehash[!dir].tuple)`**——目标地址 = 对方向 tuple 的逆。`!dir` 是关键：original 方向包看 reply tuple，reply 方向包看 original tuple。
2. **`skb_ensure_writable`**——确保 skb 可写（可能 `pskb_expand_head` 重新分配）。NAT 改包前必须可写，否则会破坏共享 skb。
3. **`l4proto_manip_pkt`**——先改 L4（TCP/UDP 端口、ICMP id）。**先改 L4 再改 L3**，因为 L4 校验和依赖 L3 地址，改完 L3 后要更新 L4 校验和。
4. **改 IP 头**：`csum_replace4(&iph->check, oldip, newip)` 增量更新 IP 头校验和，再写新地址。

**校验和处理**：NAT 不重算整个校验和，而是用 `csum_replace4`/`inet_proto_csum_replace4` **增量更新**——只改变化的部分，O(1) 开销。

### 8.6 完整 NAT 流程图

```text
首包（original 方向，客户端→服务端）
─────────────────────────────────
PRE_ROUTING:
  conntrack hook: 建 nf_conn (ctinfo=NEW)
  nat hook (DNAT): nf_nat_inet_fn
    ├─ ctinfo=NEW, !nf_nat_initialized → 跑 nat 规则
    ├─ 规则匹配 → nf_nat_setup_info:
    │    ├─ get_unique_tuple 选新目的地址
    │    ├─ nf_conntrack_alter_reply 改 reply tuple
    │    └─ set IPS_DST_NAT | IPS_DST_NAT_DONE
    └─ nf_nat_packet → nf_nat_manip_pkt 改 skb 目的地址
... 包穿过协议栈 ...
POST_ROUTING:
  nat hook (SNAT): nf_nat_inet_fn
    ├─ ctinfo=NEW, !nf_nat_initialized(SRC) → 跑 nat 规则
    ├─ MASQUERADE 规则匹配 → nf_nat_setup_info:
    │    ├─ 选源地址=出接口地址
    │    ├─ 改 reply tuple
    │    ├─ set IPS_SRC_NAT | IPS_SRC_NAT_DONE
    │    └─ 挂 nat_bysource hash
    └─ nf_nat_packet 改 skb 源地址
  conntrack confirm hook: nf_confirm → ct 移入全局 hash

回包（reply 方向，服务端→客户端）
─────────────────────────────────
PRE_ROUTING:
  conntrack hook: resolve_normal_ct
    └─ 用收到的包构造 tuple，匹配 reply tuplehash → ctinfo=ESTABLISHED_REPLY
  nat hook (DNAT 关卡): nf_nat_inet_fn
    ├─ ctinfo=ESTABLISHED_REPLY → default 分支
    └─ nf_nat_packet:
         ├─ mtype=DST, dir=REPLY
         ├─ statusbit = IPS_DST_NAT ^ IPS_NAT_MASK = IPS_SRC_NAT
         ├─ ct 有 IPS_SRC_NAT → 改包
         └─ nf_nat_manip_pkt: target=逆(original tuple)
              把目的地址从"SNAT后的地址"改回"原客户端地址"
... 包上交本机或转发 ...
POST_ROUTING (若转发):
  nat hook (SNAT 关卡): 类似，改源地址（若有 DNAT 则还原）
```

### 8.7 MASQUERADE vs SNAT

- `SNAT --to-source IP`：固定源 IP。映射在首包确定后固定，性能最好。
- `MASQUERADE`：源 IP **每次首包都查出接口当前地址**（`find_appropriate_src` → 出接口 IP）。接口 IP 变（拨号/DHCP）时自动适配；接口 down 时通过 `masq_index` 匹配立即清相关 conntrack（`nf_nat_masquerade_ipv4` → 遍历 `nat_bysource` hash）。

**性能差异**：MASQUERADE 首包多一次接口地址查找，且接口事件触发 conntrack 清理。常态下两者差不多（首包占比低），但 MASQUERADE 更适合动态 IP 场景。

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

- `struct nft_chain`：一条链。base chain 挂在某个 Netfilter hook 上，回调 `nft_do_chain`；普通 chain 只能被 JUMP/GOTO 调用。
- `struct nft_rule_dp`：一条规则的"数据路径"表示（无锁、紧凑）。由一串 `nft_expr` 连续排列组成。
- `struct nft_expr`：一个表达式。含 `ops`（指向 `nft_expr_ops`，即操作类型）和 `data`（操作数）。
- `struct nft_expr_ops`：表达式操作，如 `nft_cmp_fast_ops`（比较）、`nft_payload_fast_ops`（取报文字段）、`nft_immediate_ops`（设置 verdict）、`nft_lookup_ops`（查 set/map）。
- `struct nft_rule_blob`：一个链的规则打包成的连续内存块，RCU 保护，双缓冲（`blob_gen_0` / `blob_gen_1`）。
- `struct nft_regs`：表达式执行时的寄存器组，`regs.verdict.code` 存当前裁决。

### 9.3 `nft_do_chain`：字节码解释器

文件：`net/netfilter/nf_tables_core.c:249-348`

**这是 nftables 的热路径核心**。与 `ipt_do_table` 类似，每条 base chain 注册一个 Netfilter hook，回调都是 `nft_do_chain`，`priv` 指向 `nft_chain`。

源码：

```c
unsigned int
nft_do_chain(struct nft_pktinfo *pkt, void *priv)
{
	const struct nft_chain *chain = priv, *basechain = chain;
	const struct net *net = nft_net(pkt);
	const struct nft_expr *expr, *last;
	const struct nft_rule_dp *rule;
	struct nft_regs regs;
	unsigned int stackptr = 0;
	struct nft_jumpstack jumpstack[NFT_JUMP_STACK_SIZE];
	bool genbit = READ_ONCE(net->nft.gencursor);
	struct nft_rule_blob *blob;
	struct nft_traceinfo info;

	info.trace = false;
	if (static_branch_unlikely(&nft_trace_enabled))
		nft_trace_init(&info, pkt, basechain);
do_chain:
	if (genbit)
		blob = rcu_dereference(chain->blob_gen_1);
	else
		blob = rcu_dereference(chain->blob_gen_0);

	rule = (struct nft_rule_dp *)blob->data;
next_rule:
	regs.verdict.code = NFT_CONTINUE;
	for (; !rule->is_last ; rule = nft_rule_next(rule)) {
		nft_rule_dp_for_each_expr(expr, last, rule) {
			if (expr->ops == &nft_cmp_fast_ops)
				nft_cmp_fast_eval(expr, &regs);
			else if (expr->ops == &nft_cmp16_fast_ops)
				nft_cmp16_fast_eval(expr, &regs);
			else if (expr->ops == &nft_bitwise_fast_ops)
				nft_bitwise_fast_eval(expr, &regs);
			else if (expr->ops != &nft_payload_fast_ops ||
				 !nft_payload_fast_eval(expr, &regs, pkt))
				expr_call_ops_eval(expr, &regs, pkt);

			if (regs.verdict.code != NFT_CONTINUE)
				break;
		}

		switch (regs.verdict.code) {
		case NFT_BREAK:
			regs.verdict.code = NFT_CONTINUE;
			nft_trace_copy_nftrace(pkt, &info);
			continue;
		case NFT_CONTINUE:
			nft_trace_packet(pkt, &regs.verdict,  &info, rule,
					 NFT_TRACETYPE_RULE);
			continue;
		}
		break;
	}

	nft_trace_verdict(pkt, &info, rule, &regs);

	switch (regs.verdict.code & NF_VERDICT_MASK) {
	case NF_ACCEPT:
	case NF_QUEUE:
	case NF_STOLEN:
		return regs.verdict.code;
	case NF_DROP:
		return NF_DROP_REASON(pkt->skb, SKB_DROP_REASON_NETFILTER_DROP, EPERM);
	}

	switch (regs.verdict.code) {
	case NFT_JUMP:
		if (WARN_ON_ONCE(stackptr >= NFT_JUMP_STACK_SIZE))
			return NF_DROP;
		jumpstack[stackptr].rule = nft_rule_next(rule);
		stackptr++;
		fallthrough;
	case NFT_GOTO:
		chain = regs.verdict.chain;
		goto do_chain;
	case NFT_CONTINUE:
	case NFT_RETURN:
		break;
	default:
		WARN_ON_ONCE(1);
	}

	if (stackptr > 0) {
		stackptr--;
		rule = jumpstack[stackptr].rule;
		goto next_rule;
	}

	nft_trace_packet(pkt, &regs.verdict, &info, NULL, NFT_TRACETYPE_POLICY);

	if (static_branch_unlikely(&nft_counters_enabled))
		nft_update_chain_stats(basechain, pkt);

	if (nft_base_chain(basechain)->policy == NF_DROP)
		return NF_DROP_REASON(pkt->skb, SKB_DROP_REASON_NETFILTER_DROP, EPERM);

	return nft_base_chain(basechain)->policy;
}
```

逐段注解：

| 行 | 代码 | 说明 |
|----|------|------|
| 259 | `genbit = READ_ONCE(net->nft.gencursor)` | **世代游标**：nftables 用双缓冲（`blob_gen_0`/`blob_gen_1`）做无锁规则替换。读者用当前 `gencursor` 选 blob，写者改另一个 blob 后翻转 gencursor，等 RCU 宽限期再回收旧 blob。这是相比 iptables（序列锁）的一大改进 |
| 267-270 | `blob = rcu_dereference(chain->blob_gen_{genbit})` | 取当前代的规则 blob。整个 blob 是连续内存，规则和表达式紧密排列，cache 友好 |
| 272 | `rule = blob->data` | 第一条规则。blob 内规则用 `is_last` 标志结尾，无独立链表 |
| 274 | `regs.verdict.code = NFT_CONTINUE` | 每条规则开始前重置 verdict。`NFT_CONTINUE` = 继续下一个表达式 |
| 275 | `for (; !rule->is_last; rule = nft_rule_next(rule))` | 遍历链上所有规则，直到 `is_last` 标志 |
| 276 | `nft_rule_dp_for_each_expr(expr, last, rule)` | 遍历一条规则内的所有表达式 |
| 277-285 | **快速路径分发**（关键优化） | 见 §9.4 详解 |
| 287-288 | `if (regs.verdict.code != NFT_CONTINUE) break` | 表达式设了 verdict（如匹配失败设 `NFT_BREAK`，或 immediate 设 `NF_DROP`）则跳出表达式循环 |
| 291-301 | `switch (regs.verdict.code)` | **规则级 verdict 处理**：`NFT_BREAK`=本规则不匹配，重置为 CONTINUE 跑下条规则；`NFT_CONTINUE`=本规则匹配且没终止，跑下条规则；其他=终止 |
| 306-313 | `switch (regs.verdict.code & NF_VERDICT_MASK)` | **终止性 verdict**：NF_ACCEPT/QUEUE/STOLEN/DROP 直接返回。注意复用 Netfilter 的 verdict 编码 |
| 316-324 | `case NFT_JUMP / NFT_GOTO` | **跳链**：JUMP 压栈后跳到目标链（`goto do_chain`）；GOTO 不压栈直接跳。与 iptables 的 JUMP/GOTO 概念一致 |
| 332-336 | `if (stackptr > 0) { stackptr--; rule = jumpstack[stackptr].rule; goto next_rule; }` | **RETURN 弹栈**：从自定义链返回，恢复到调用点继续 |
| 340-341 | `if (nft_counters_enabled) nft_update_chain_stats(...)` | 链级计数器。用静态键控制，默认不开计数器时零开销 |
| 343-346 | `policy == NF_DROP ? DROP : policy` | **默认策略**：规则全不匹配时执行 base chain 的 policy（ACCEPT 或 DROP） |

### 9.4 快速路径优化：内联表达式

`nft_do_chain` 最精妙的设计是第 277-285 行的**快速路径分发**：

```c
if (expr->ops == &nft_cmp_fast_ops)
    nft_cmp_fast_eval(expr, &regs);           // 直接调用，无间接
else if (expr->ops == &nft_cmp16_fast_ops)
    nft_cmp16_fast_eval(expr, &regs);
else if (expr->ops == &nft_bitwise_fast_ops)
    nft_bitwise_fast_eval(expr, &regs);
else if (expr->ops != &nft_payload_fast_ops ||
         !nft_payload_fast_eval(expr, &regs, pkt))
    expr_call_ops_eval(expr, &regs, pkt);     // 慢路径：函数指针
```

**设计意图**：

- **常见表达式（cmp/payload/bitwise）直接内联调用**，不走 `expr->ops->eval` 函数指针间接跳转。函数指针跳转会破坏分支预测，内联调用让 CPU 流水线更顺畅。
- **慢路径 `expr_call_ops_eval`**：罕见表达式（lookup/counter/immediate/ct/meta/...）才走函数指针。这些表达式操作复杂，内联收益小。
- `nft_cmp_fast_ops` 是"比较 4 字节"的快速版本，`nft_cmp16_fast_ops` 是"比较 16 字节"（IPv6 地址）。用户态 `nft` 编译规则时会优先用 fast 版本。

**对比 iptables**：`ipt_do_table` 里每个 match 都是 `acpar.match->match(skb, &acpar)` 函数指针调用，无法内联。nftables 的快速路径是它比 iptables 快的核心原因之一。

### 9.5 规则的字节码布局

```text
nft_rule_blob (连续内存，RCU 保护，双缓冲)
┌────────────────────────────────────────────────────────┐
│ rule[0] (struct nft_rule_dp, is_last=false)            │
│   ├─ expr: payload (取 tcp dport)   ← nft_payload_fast │
│   ├─ expr: cmp    (== 22)           ← nft_cmp_fast     │
│   └─ expr: immediate (verdict=NF_ACCEPT)               │
├────────────────────────────────────────────────────────┤
│ rule[1]                                                 │
│   ├─ expr: payload (取 ip saddr)                        │
│   ├─ expr: lookup (查 blacklist set)  ← 慢路径          │
│   └─ expr: immediate (verdict=NF_DROP)                  │
├────────────────────────────────────────────────────────┤
│ rule[last] (is_last=true)  ← 链结束标志                  │
└────────────────────────────────────────────────────────┘
```

**执行示例**：`nft add rule ip filter input tcp dport 22 accept`

编译成 3 个表达式：`payload(tcp dport)` → `cmp(==22)` → `immediate(ACCEPT)`。报文经过时：

1. `nft_payload_fast_eval` 取 TCP 目的端口到 `regs`
2. `nft_cmp_fast_eval` 比较 regs 与 22，不匹配设 `NFT_BREAK`
3. 若匹配，`immediate` 设 `NF_ACCEPT`，跳出循环返回

### 9.6 verdict 编码

nftables 的 verdict 分两类：

| 类型 | verdict | 含义 |
|------|---------|------|
| 表达式级 | `NFT_CONTINUE` | 继续下一个表达式 |
| 表达式级 | `NFT_BREAK` | 本规则不匹配，跑下条规则 |
| 规则级 | `NF_ACCEPT`/`NF_DROP`/`NF_QUEUE`/`NF_STOLEN` | 终止，返回 Netfilter（复用 `nf_inet_hooks` 的 verdict） |
| 跳链级 | `NFT_JUMP` | 跳到自定义链，压栈（可 RETURN） |
| 跳链级 | `NFT_GOTO` | 跳到目标链，不压栈 |
| 跳链级 | `NFT_RETURN` | 从当前链返回，弹栈 |

**设计要点**：nftables 复用 Netfilter 的 `NF_ACCEPT`/`NF_DROP` 等作为终止 verdict，这样 `nft_do_chain` 的返回值可以直接被 `nf_hook_slow` 消费——无需翻译。这是 nftables 能作为 Netfilter 的"另一种后端"而非"替代品"共存的关键。

### 9.7 nftables 相比 iptables 的性能优势总结

| 维度 | iptables | nftables | 改进点 |
|------|----------|----------|--------|
| 规则匹配 | O(N) 线性，每 match 函数指针 | 快速表达式内联 + set/map O(1) 查找 | 减少 branch miss + 算法降阶 |
| 规则存储 | 平铺 `ipt_entry` + match + target | 紧凑 `nft_rule_dp` blob，双缓冲 | cache 友好 + 无锁替换 |
| 同步 | 序列锁 `xt_write_recseq` | RCU 双缓冲 + gencursor | 读路径完全无锁 |
| set/map | 无（要靠 `hashlimit`/`recent` 模块） | 内置 `nft_set`，支持 hash/interval/concat | 大规模 IP/端口集合 O(1) |
| 事务 | 逐条规则原子 | `NFT_MSG_NEWRULE` 批事务 | 一次性提交多条规则，中间态不可见 |
| IPv4/IPv6 | 两套（`iptables`/`ip6tables`） | 统一 family（inet/ipv4/ipv6） | 一套规则管两个协议 |

**生产实践**：现代发行版（Debian 11+、RHEL 8+、Ubuntu 22.04+）默认用 nftables 后端，`iptables` 命令通过 `iptables-nft` 兼容层翻译成 nftables 规则。Cilium、firewalld 已原生用 nftables API。

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

## 13. nfqueue：把报文送给用户态裁决

> nfqueue 把内核截获的报文通过 netlink 送给用户态进程（如 `suricata`、`conntrack-tools`、自写 IDS），用户态判决后再回注内核继续走协议栈。

### 13.1 触发点：`NF_QUEUE` verdict 的编码

回顾 §4.3 `nf_hook_slow` 的 `case NF_QUEUE`：

```c
case NF_QUEUE:
    ret = nf_queue(skb, state, s, verdict);
    if (ret == 1)
        continue;       /* 用户态裁决 ACCEPT，从 s+1 继续后续 hook */
    return ret;
```

verdict 的高 16 位编码了**队列号**（queue number），通过 `NF_QUEUE_NR(x)` 宏构造：

```c
#define NF_QUEUE_NR(x) ((((x) << 16) & NF_VERDICT_QMASK) | NF_QUEUE)
```

`nf_queue`（`net/netfilter/nf_queue.c:238`）取出队列号后调 `__nf_queue`：

```c
int nf_queue(struct sk_buff *skb, struct nf_hook_state *state,
	     unsigned int index, unsigned int verdict)
{
	int ret;

	ret = __nf_queue(skb, state, index, verdict >> NF_VERDICT_QBITS);
	if (ret < 0) {
		if (ret == -ESRCH &&
		    (verdict & NF_VERDICT_FLAG_QUEUE_BYPASS))
			return 1;       /* bypass: 没人监听就当 ACCEPT */
		kfree_skb(skb);
	}

	return 0;
}
```

`NF_VERDICT_FLAG_QUEUE_BYPASS`（`0x00008000`）表示"无用户态监听时放行而非丢弃"——iptables 的 `QUEUE` target 默认是 strict（无监听则丢），`NFQUEUE --queue-bypass` 是宽松模式。

### 13.2 `__nf_queue`：构造 queue entry

文件：`net/netfilter/nf_queue.c:159-235`

```c
static int __nf_queue(struct sk_buff *skb, const struct nf_hook_state *state,
		      unsigned int index, unsigned int queuenum)
{
	struct nf_queue_entry *entry = NULL;
	const struct nf_queue_handler *qh;
	unsigned int route_key_size;
	int status;

	/* QUEUE == DROP if no one is waiting, to be safe. */
	qh = rcu_dereference(nf_queue_handler);
	if (!qh)
		return -ESRCH;       /* 没注册 handler → 丢弃 */

	/* ... route_key_size 按协议族设置 ... */

	entry = kmalloc(sizeof(*entry) + route_key_size, GFP_ATOMIC);
	if (!entry)
		return -ENOMEM;

	if (skb_dst(skb) && !skb_dst_force(skb)) {
		kfree(entry);
		return -ENETDOWN;
	}

	*entry = (struct nf_queue_entry) {
		.skb	= skb,
		.skb_dev = skb->dev,
		.state	= *state,            /* 保存 hook 状态，便于回注时恢复 */
		.hook_index = index,         /* 记住断点下标，回注从这里继续 */
		.size	= sizeof(*entry) + route_key_size,
	};
	__nf_queue_entry_init_physdevs(entry);

	if (!nf_queue_entry_get_refs(entry)) {   /* 增加 hook 数组等引用计数 */
		kfree(entry);
		return -ENOTCONN;
	}

	switch (entry->state.pf) {
	case AF_INET:
		nf_ip_saveroute(skb, entry);    /* 保存路由信息，回注时重算可能需要 */
		break;
	case AF_INET6:
		nf_ip6_saveroute(skb, entry);
		break;
	}

	status = qh->outfn(entry, queuenum);     /* 调用注册的 outfn 把 entry 送给用户态 */
	if (status < 0) {
		nf_queue_entry_free(entry);
		return status;
	}

	return 0;
}
```

**关键设计**：

1. **`nf_queue_handler` 全局唯一**：整个内核同时只能有一个 nfqueue handler（即 `nfnetlink_queue` 模块）。`rcu_dereference(nf_queue_handler)` 取它，没注册直接 `-ESRCH`（丢弃）。
2. **`nf_queue_entry` 保存完整上下文**：`skb` + `hook_state` + `hook_index`（断点下标）。这是回注时从断点继续的依据——§4.3 `nf_hook_slow` 的 `s` 参数就是为这个设计。
3. **`skb_dst_force`**：skb 的路由可能被释放，这里强制持引用，保证回注时路由仍有效。
4. **`nf_ip_saveroute`**：保存路由快照。用户态可能改包（如改目的地址），回注时 `nf_reroute` 据此判断是否要重新查路由。

### 13.3 `nfnetlink_queue`：用户态通信桥梁

文件：`net/netfilter/nfnetlink_queue.c`

`nfnetlink_queue` 模块初始化时注册一个 `nf_queue_handler`：

```c
static const struct nf_queue_handler nfqh = {
	.outfn		= nfqnl_enqueue_packet,
	.nf_hook_drop	= nfqnl_nf_hook_drop,
};
```

`qh->outfn` 即 `nfqnl_enqueue_packet`（`:1051`），把 entry 封装成 netlink 消息（`NFQA_PACKET`）发给绑定了该 queue_num 的用户态 socket。用户态用 `libnetfilter_queue` 库接收。

### 13.4 `nf_reinject`：用户态裁决回注

用户态进程处理完报文后，通过 `NFQNL_MSG_VERDICT` netlink 消息把判决（ACCEPT/DROP/REPEAT）发回内核。内核调用 `nfqnl_reinject` → `nf_reinject`（`net/netfilter/nfnetlink_queue.c:355`）：

```c
static void nf_reinject(struct nf_queue_entry *entry, unsigned int verdict)
{
	const struct nf_hook_entry *hook_entry;
	const struct nf_hook_entries *hooks;
	struct sk_buff *skb = entry->skb;
	const struct net *net;
	unsigned int i;
	int err;
	u8 pf;

	net = entry->state.net;
	pf = entry->state.pf;

	hooks = nf_hook_entries_head(net, pf, entry->state.hook);

	i = entry->hook_index;                /* 从断点下标恢复 */
	if (!hooks || i >= hooks->num_hook_entries) {
		kfree_skb_reason(skb, SKB_DROP_REASON_NETFILTER_DROP);
		nf_queue_entry_free(entry);
		return;
	}

	hook_entry = &hooks->hooks[i];

	/* Continue traversal iff userspace said ok... */
	if (verdict == NF_REPEAT)
		verdict = nf_hook_entry_hookfn(hook_entry, skb, &entry->state);

	if (verdict == NF_ACCEPT) {
		if (nf_reroute(skb, entry) < 0)    /* 用户态可能改了包，重算路由 */
			verdict = NF_DROP;
	}

	if (verdict == NF_ACCEPT) {
next_hook:
		++i;                               /* 从断点+1 继续后续 hook */
		verdict = nf_iterate(skb, &entry->state, hooks, &i);
	}

	switch (verdict & NF_VERDICT_MASK) {
	case NF_ACCEPT:
	case NF_STOP:
		local_bh_disable();
		entry->state.okfn(entry->state.net, entry->state.sk, skb);  /* 调 okfn 继续协议栈 */
		local_bh_enable();
		break;
	case NF_QUEUE:
		err = nf_queue(skb, &entry->state, i, verdict);  /* 可能再次入队 */
		if (err == 1)
			goto next_hook;
		break;
	case NF_STOLEN:
		break;
	default:
		kfree_skb(skb);
	}

	nf_queue_entry_free(entry);
}
```

**`nf_iterate`**（`:259`）是 `nf_hook_slow` 的"从指定下标继续"版本：

```c
static unsigned int nf_iterate(struct sk_buff *skb,
			       struct nf_hook_state *state,
			       const struct nf_hook_entries *hooks,
			       unsigned int *index)
{
	const struct nf_hook_entry *hook;
	unsigned int verdict, i = *index;

	while (i < hooks->num_hook_entries) {
		hook = &hooks->hooks[i];
repeat:
		verdict = nf_hook_entry_hookfn(hook, skb, state);
		if (verdict != NF_ACCEPT) {
			*index = i;
			if (verdict != NF_REPEAT)
				return verdict;
			goto repeat;
		}
		i++;
	}

	*index = i;
	return NF_ACCEPT;
}
```

### 13.5 完整数据流

```text
内核协议栈                          用户态进程
─────────────                       ─────────
ip_rcv
  └─ NF_HOOK(PRE_ROUTING)
       └─ nf_hook_slow
            └─ 某 hook 返回 NF_QUEUE(5)
                 └─ nf_queue
                      └─ __nf_queue
                           ├─ 分配 nf_queue_entry (含 hook_index 断点)
                           ├─ skb_dst_force (持路由引用)
                           └─ qh->outfn = nfqnl_enqueue_packet
                                └─ netlink_unicast ──────────┐
                                                              │
                                                              ▼
                                              recvmsg() 收到 NFQA_PACKET
                                              解析报文，做 IDS/审计...
                                              决定 ACCEPT
                                                              │
                                              sendmsg(NFQNL_MSG_VERDICT) ─┐
                                                              ▲             │
                                  ┌───────────────────────────┘             │
                                  │                                         │
                       nfqnl_recv_verdict ◄─────────────────────────────────┘
                          └─ nfqnl_reinject
                               └─ nf_reinject(entry, NF_ACCEPT)
                                    ├─ i = entry->hook_index (恢复断点)
                                    ├─ nf_reroute (用户态可能改了包)
                                    ├─ ++i; nf_iterate (从断点+1 继续后续 hook)
                                    └─ okfn = ip_rcv_finish (继续协议栈)
```

**关键洞察**：

1. **断点续跑**：`hook_index` 让回注后从被中断的 hook 之后继续，而非从头跑。否则已 ACCEPT 的 hook 会重复执行。
2. **`nf_reroute`**：用户态改了目的地址（如 IDS 改写重定向），回注时必须重算路由。
3. **性能代价**：每个 nfqueue 包要经历 skb→netlink→用户态→netlink→内核 两次拷贝和两次上下文切换，不适合高 PPS 场景。XDP 的 `AF_XDP` 是其高性能替代。
4. **`nf_queue_nf_hook_drop`**：模块卸载或 netns 退出时，所有在队 skb 必须丢弃（引用的 hook 数组要释放）。`nf_unregister_net_hook` 调用此函数清理。

### 13.6 用法

```bash
# 把 FORWARD 链的包送到 queue 0
iptables -A FORWARD -j NFQUEUE --queue-num 0 --queue-bypass

# 用户态接收（需 libnetfilter_queue）
# 示例：suricata IDS
suricata -q 0 -c /etc/suricata/suricata.yaml

# nftables 等价
nft add rule ip filter forward counter queue num 0 bypass

# 查看 nfqueue 统计
cat /proc/net/netfilter/nfnetlink_queue
```

---

## 14. nf_flowtable：流卸载旁路 Netfilter

> `nf_flowtable` 对已建立的连接绕过整个 Netfilter 软件路径（hook 数组、conntrack 查找、iptables 匹配），直接在网卡或轻量 hook 里转发。

### 14.1 解决什么问题

回顾 §4.3 `nf_hook_slow` 和 §6.2 `ipt_do_table`：每个包经过 5 个 hook 关卡，每个关卡线性扫描 hook 数组，iptables 还要线性匹配规则。对已建立连接的后续包（ESTABLISHED），这些开销是浪费——连接已经过了首包判决，后续包只需按既定映射转发。

`nf_flowtable` 的思路：**首包走完整 Netfilter 流程并建立 conntrack；连接进入 ESTABLISHED 后，把流信息（双向 tuple、NAT 映射、出接口、下一跳 MAC）打包成 `flow_offload` 存入 flow table；后续包在 flow table 命中后直接转发，不再走 Netfilter**。

### 14.2 核心数据结构

#### 14.2.1 `struct nf_flowtable`

```c
struct nf_flowtable {
	struct list_head		list;      /* 注册到全局链表 */
	struct rhashtable		rhashtable; /* flow_offload 查找表 */
	const struct nf_flowtable_type	*type;
	u32				flags;
	/* ... GC 工作队列、统计 ... */
};
```

`rhashtable` 是 flow table 的核心——key 是 tuple，O(1) 查找。相比 conntrack 的 hash 表，flow table 只存"已确认可加速"的流，规模小得多。

#### 14.2.2 `struct flow_offload`

```c
struct flow_offload {
	struct nf_conn	*ct;                          /* 关联的 conntrack */
	unsigned long	flags;
	u32		timeout;
	struct flow_offload_tuple_rhash tuplehash[2];  /* 双向 tuple + rhash 节点 */
};
```

每条 `flow_offload` 对应一条连接的双向转发信息，存入 rhashtable 时两个方向都插入（类似 conntrack 的 tuplehash[2]）。

#### 14.2.3 `struct nf_flowtable_type`

文件：`net/netfilter/nf_flow_table_inet.c:68-96`

```c
static struct nf_flowtable_type flowtable_inet = {
	.family		= NFPROTO_INET,
	.init		= nf_flow_table_init,
	.setup		= nf_flow_table_offload_setup,
	.action		= nf_flow_rule_route_inet,
	.free		= nf_flow_table_free,
	.hook		= nf_flow_offload_inet_hook,   /* 快路径 hook */
	.owner		= THIS_MODULE,
};
```

`.hook` 是快路径入口——flow table 作为一个 Netfilter hook 注册（通常挂在 ingress/egress），命中 flow 的包在此被直接转发。

### 14.3 流的建立：`flow_offload_add`

文件：`net/netfilter/nf_flow_table_core.c:324-355`

```c
int flow_offload_add(struct nf_flowtable *flow_table, struct flow_offload *flow)
{
	int err;

	flow->timeout = nf_flowtable_time_stamp + flow_offload_get_timeout(flow);

	err = rhashtable_insert_fast(&flow_table->rhashtable,
				     &flow->tuplehash[0].node,
				     nf_flow_offload_rhash_params);
	if (err < 0)
		return err;

	err = rhashtable_insert_fast(&flow_table->rhashtable,
				     &flow->tuplehash[1].node,
				     nf_flow_offload_rhash_params);
	if (err < 0) {
		rhashtable_remove_fast(&flow_table->rhashtable,
				       &flow->tuplehash[0].node,
				       nf_flow_offload_rhash_params);
		return err;
	}

	nf_ct_refresh(flow->ct, NF_CT_DAY);    /* 关联 conntrack 续命 */

	if (nf_flowtable_hw_offload(flow_table)) {
		__set_bit(NF_FLOW_HW, &flow->flags);
		nf_flow_offload_add(flow_table, flow);   /* 下发到网卡硬件 */
	}

	return 0;
}
```

**触发时机**：nftables 规则用 `flow offload @f` action，在连接 ESTABLISHED 后（首包经过 `IPS_SEEN_REPLY`）调用 `flow_offload_add`。也可由 conntrack 的 `IPS_OFFLOAD` 位管理自动触发。

**双向插入**：`tuplehash[0]` 和 `[1]` 都插入 rhashtable，这样任一方向的包都能查到。

**硬件卸载**：`nf_flowtable_hw_offload` 为真时调 `nf_flow_offload_add`（offload 子模块）把流下发到网卡 TCAM，后续包完全在硬件转发，CPU 零参与。

### 14.4 快路径：`nf_flow_offload_ip_hook`

文件：`net/netfilter/nf_flow_table_ip.c:840-917`

```c
nf_flow_offload_ip_hook(void *priv, struct sk_buff *skb,
			const struct nf_hook_state *state)
{
	struct flow_offload_tuple_rhash *tuplehash;
	struct nf_flowtable *flow_table = priv;
	/* ... */

	tuplehash = nf_flow_offload_lookup(&ctx, flow_table, skb);   /* 查 flow table */
	if (!tuplehash)
		return NF_ACCEPT;        /* 未命中，走正常 Netfilter 路径 */

	ret = nf_flow_offload_forward(&ctx, flow_table, tuplehash, skb);
	if (ret < 0)
		return NF_DROP;
	else if (ret == 0)
		return NF_ACCEPT;        /* 命中但无法加速（如需分片） */

	/* ... xmit_type 分发：NEIGH / DIRECT / XFRM ... */

	switch (tuplehash->tuple.xmit_type) {
	case FLOW_OFFLOAD_XMIT_NEIGH:
		rt = dst_rtable(tuplehash->tuple.dst_cache);
		xmit.outdev = dev_get_by_index_rcu(state->net, tuplehash->tuple.ifidx);
		/* ... */
		neigh = ip_neigh_gw4(rt->dst.dev, rt_nexthop(rt, ip_daddr));
		xmit.dest = neigh->ha;                 /* 直接用缓存的下一跳 MAC */
		skb_dst_set_noref(skb, &rt->dst);
		break;
	case FLOW_OFFLOAD_XMIT_DIRECT:
		xmit.outdev = dev_get_by_index_rcu(state->net, tuplehash->tuple.out.ifidx);
		xmit.dest = tuplehash->tuple.out.h_dest;    /* 预存的 MAC */
		xmit.source = tuplehash->tuple.out.h_source;
		break;
	/* ... */
	}
	xmit.tuple = other_tuple;

	return nf_flow_queue_xmit(state->net, skb, &xmit);   /* 直接发送，不回 Netfilter */
}
```

**快路径做了什么**：

1. **`nf_flow_offload_lookup`**：rhashtable 查 tuple，O(1)。
2. **`nf_flow_offload_forward`**：做 NAT 改写（如果该流有 NAT 映射）、TTL 递减、校验和更新。这些原本在 `nf_nat_packet` + `ip_forward` 里做的事，这里内联做，无函数指针开销。
3. **`xmit_type` 分发**：预存的发送方式（NEIGH=查邻居、DIRECT=预存 MAC、XFRM=IPsec）。
4. **`nf_flow_queue_xmit`**：直接调 `dev_queue_xmit` 发送，**不返回 NF_ACCEPT**——包不再走后续 Netfilter hook。

**没做什么**（相比正常路径省掉的开销）：

- 不扫 hook 数组（5 个关卡的 `nf_hook_slow`）
- 不查 conntrack hash 表（`resolve_normal_ct`）
- 不跑 iptables/nftables 规则匹配
- 不进 qdisc 排队（直接 `dev_queue_xmit`，部分路径）

### 14.5 flow table 的挂载层次

flow table 的 hook 注册位置很关键——它必须在"conntrack 已建立"之后才能命中。典型部署：

```text
ingress hook (flow table)
    ├─ 命中 → 直接转发（旁路后续所有 Netfilter）
    └─ 未命中 → 继续走 PRE_ROUTING → conntrack → ... → POST_ROUTING
                                            │
                                  连接 ESTABLISHED 后
                                  flow_offload_add 把流加入 flow table
                                            │
                                  后续包在 ingress 即命中
```

nftables 配置示例：

```bash
# 1. 定义 flow table，挂在 ingress（或 prerouting）
nft add flowtable inet filter f { hook ingress priority 0\; devices = { eth0, eth1 }\; }

# 2. 规则：ESTABLISHED 连接走 flow offload
nft add rule inet filter forward ip protocol { tcp, udp } flow offload @f

# 3. 普通 forward 规则仍存在，但 ESTABLISHED 流量被 flow table 旁路
nft add rule inet filter forward ct state established accept
```

### 14.6 与 conntrack 的协作

| 机制 | 作用 |
|------|------|
| `IPS_OFFLOAD` | conntrack 状态位，标记该连接已卸载到 flow table。设置后 conntrack 不再处理该连接的包（避免重复 NAT） |
| `nf_ct_refresh(flow->ct, NF_CT_DAY)` | `flow_offload_add` 时给 conntrack 续命一天，防止 flow table 活跃但 conntrack 超时 |
| GC 协同 | flow table 的 GC 清理超时 flow 时，清除 conntrack 的 `IPS_OFFLOAD` 位，连接回到正常 Netfilter 路径 |
| `flow_offload_teardown` | 连接结束时（FIN/RST），拆除 flow entry 并让 conntrack 走正常超时回收 |

**关键约束**：flow table 只适合**无状态变化**的 ESTABLISHED 流量。首包、连接结束、异常包（如分片、TCP 标志异常）都回退到正常 Netfilter 路径。这是"快路径优化"而非"替代品"。

### 14.7 性能影响

| 场景 | 无 flow table | 有 flow table |
|------|---------------|---------------|
| 首包 | 完整 Netfilter | 完整 Netfilter（相同） |
| 后续包 | 5 hook × O(N) 扫描 + conntrack 查找 + 规则匹配 | 1 次 rhashtable 查找 + 内联转发 |
| 转发延迟 | 数百纳秒～微秒 | 数十纳秒 |
| CPU 占用 | 高（每包多次函数调用） | 低（快路径近乎直线代码） |

**硬件卸载更进一步**：`FLOW_OFFLOAD_XMIT_DIRECT` + 网卡 TCAM，后续包完全在网卡 ASIC 转发，CPU 零参与。这是云厂商做 SDN 网关的关键技术。

> **与 eBPF/XDP 的关系**：flow table 是 Netfilter 内部的优化，仍依赖 conntrack 和 nftables 配置。XDP 是更激进的方案——在驱动 RX 早期就用 BPF 程序转发，连 conntrack 都不碰。两者可共存：XDP 处理极速路径，flow table 处理已建立连接，Netfilter 处理首包和异常。

---

## 15. conntrack helper 与 expectation：相关连接的自动建立

> FTP 数据连接、SIP 媒体流、ICMP 差错报文等"相关连接"如何被 conntrack 自动识别并建立？答案在 helper + expectation 机制。

### 15.1 解决什么问题

FTP 主动模式下，客户端在控制连接（21 端口）上发 `PORT` 命令告诉服务器"我来监听某端口，你连过来"。服务器随后从 20 端口**主动发起**到客户端指定端口的数据连接。

问题：这条数据连接是**新连接**（五元组不同），conntrack 会把它当 `IP_CT_NEW`，防火墙规则要么放行所有新连接（不安全），要么放行特定端口（FTP 数据端口动态变化，无法预知）。

`helper` 机制解决：在控制连接上"窥探"应用层协议，解析出即将建立的数据连接五元组，预先注册一个 `expectation`。数据连接到来时，conntrack 发现它匹配某个 expectation，就把它标记为 `IP_CT_RELATED`，关联到控制连接——防火墙只需放行 `RELATED` 即可。

### 15.2 核心数据结构

#### 15.2.1 `struct nf_conntrack_helper`

文件：`include/net/netfilter/nf_conntrack_helper.h:32-64`

```c
struct nf_conntrack_helper {
	struct hlist_node hnode;

	char name[NF_CT_HELPER_NAME_LEN];   /* "ftp"、"sip" 等 */
	refcount_t refcnt;
	struct module *me;

	/* Tuple of things we will help (compared against server response) */
	struct nf_conntrack_tuple tuple;    /* 匹配条件：哪些连接需要这个 helper */

	/* Function to call when data passes; return verdict, or -1 to
           invalidate. */
	int (*help)(struct sk_buff *skb,
		    unsigned int protoff,
		    struct nf_conn *ct,
		    enum ip_conntrack_info conntrackinfo);  /* 窥探应用层的回调 */

	void (*destroy)(struct nf_conn *ct);

	const struct nf_conntrack_expect_policy *expect_policy;
	unsigned int expect_class_max;

	unsigned int flags;

	/* For user-space helpers: */
	unsigned int queue_num;             /* 也可送到用户态 helper */
	u16 data_len;
};
```

**关键字段**：

- `tuple`：匹配条件。如 FTP helper 的 tuple 是 `proto=TCP, dport=21`，只有目标 21 端口的连接才会被这个 helper 处理。
- `help`：窥探应用层载荷的回调。FTP 的 `help` 解析 `PORT`/`227` 命令，提取数据连接五元组，调 `nf_ct_expect_related` 注册 expectation。
- `expect_policy`：expectation 的超时、最大数量等策略。

#### 15.2.2 `struct nf_conn_help`

文件：`include/net/netfilter/nf_conntrack_helper.h:70-81`

```c
struct nf_conn_help {
	/* Helper. if any */
	struct nf_conntrack_helper __rcu *helper;

	struct hlist_head expectations;      /* 该连接已注册的 expectation 链表 */

	/* Current number of expected connections */
	u8 expecting[NF_CT_MAX_EXPECT_CLASSES];   /* 各 class 的计数，防止泛滥 */

	/* private helper information. */
	char data[32] __aligned(8);          /* helper 私有数据（如 FTP 解析状态） */
};
```

这是 `nf_conn` 的扩展区（`nf_ct_ext`）。有 helper 的连接才会分配此扩展。

#### 15.2.3 `struct nf_conntrack_expect`

```c
struct nf_conntrack_expect {
	struct nf_conn		*master;      /* 主连接（如 FTP 控制连接） */
	struct nf_conntrack_helper __rcu *helper;
	struct nf_conntrack_tuple	tuple;  /* 期望的数据连接五元组 */
	struct nf_conntrack_tuple_mask	mask;  /* 通配掩码（如源端口任意） */
	/* ... 超时、class、flags ... */
	struct hlist_node	hnode;        /* 挂到全局 nf_ct_expect_hash */
};
```

### 15.3 helper 的绑定：`__nf_ct_try_assign_helper`

文件：`net/netfilter/nf_conntrack_helper.c:191-238`

```c
int __nf_ct_try_assign_helper(struct nf_conn *ct, struct nf_conn *tmpl,
			      gfp_t flags)
{
	struct nf_conntrack_helper *helper = NULL;
	struct nf_conn_help *help;

	/* We already got a helper explicitly attached (e.g. nft_ct) */
	if (test_bit(IPS_HELPER_BIT, &ct->status))
		return 0;

	/* ... 从 tmpl 或按 tuple 查找匹配的 helper ... */

	if (helper == NULL) {
		if (help)
			RCU_INIT_POINTER(help->helper, NULL);
		return 0;
	}

	if (help == NULL) {
		help = nf_ct_helper_ext_add(ct, flags);   /* 分配 nf_conn_help 扩展 */
		if (help == NULL)
			return -ENOMEM;
	}
	/* ... */

	rcu_assign_pointer(help->helper, helper);      /* 绑定 */

	return 0;
}
```

**绑定时机**：`init_conntrack`（新建 conntrack）时调用。按新连接的 tuple 查找匹配的 helper——如 FTP helper 的 tuple 是 `dport=21`，新建的 21 端口连接会被绑定。

**显式 vs 自动**：

- 自动绑定：基于 tuple 匹配（默认，需 `nf_conntrack_helper` 模块加载）
- 显式绑定：iptables/nftables `-m helper --helper ftp` 或 `ct helper set "ftp"`，设置 `IPS_HELPER` 位

### 15.4 expectation 的注册：以 FTP 为例

文件：`net/netfilter/nf_conntrack_ftp.c:468-523`

FTP helper 的 `help` 回调在控制连接的报文上解析 `PORT` 命令：

```c
/* 简化伪代码 */
exp = nf_ct_expect_alloc(ct);                    /* 分配 expectation，master=控制连接 */
if (!exp) { ... }

nf_ct_expect_init(exp, NF_CT_EXPECT_CLASS_DEFAULT,
		  cmd.l3num,
		  &server_addr, &client_addr,     /* 数据连接的 src/dst */
		  IPPROTO_TCP, NULL, &data_port); /* 数据连接端口 */

if (nf_ct_expect_related(exp, 0) != 0) {          /* 注册到全局 expect hash */
	nf_ct_helper_log(skb, ct, "cannot add expectation");
}
```

**`nf_ct_expect_related`**（`net/netfilter/nf_conntrack_expect.c:514`）：

```c
int nf_ct_expect_related_report(struct nf_conntrack_expect *expect,
				u32 portid, int report, unsigned int flags)
{
	int ret;

	spin_lock_bh(&nf_conntrack_expect_lock);
	ret = __nf_ct_expect_check(expect, flags);   /* 检查数量上限 */
	if (ret < 0)
		goto out;

	nf_ct_expect_insert(expect);                 /* 插入全局 hash 表 */

	nf_ct_expect_event_report(IPEXP_NEW, expect, portid, report);
	spin_unlock_bh(&nf_conntrack_expect_lock);

	return 0;
out:
	spin_unlock_bh(&nf_conntrack_expect_lock);
	return ret;
}
```

`__nf_ct_expect_check` 检查 `cnet->expect_count >= nf_ct_expect_max`（防止 expectation 泛滥，默认 `nf_ct_expect_max = nf_ct_expect_hsize * 4`，而 `nf_ct_expect_hsize = nf_conntrack_htable_size / 256`）。

### 15.5 expectation 的匹配：`nf_ct_find_expectation`

文件：`net/netfilter/nf_conntrack_expect.c:174-205`

```c
struct nf_conntrack_expect *
nf_ct_find_expectation(struct net *net,
		       const struct nf_conntrack_zone *zone,
		       const struct nf_conntrack_tuple *tuple, bool unlink)
{
	struct nf_conntrack_net *cnet = nf_ct_pernet(net);
	struct nf_conntrack_expect *i, *exp = NULL;
	unsigned int h;

	lockdep_nfct_expect_lock_held();

	if (!cnet->expect_count)
		return NULL;                  /* 快速短路：无任何 expectation */

	h = nf_ct_expect_dst_hash(net, tuple);
	hlist_for_each_entry(i, &nf_ct_expect_hash[h], hnode) {
		if (!(i->flags & NF_CT_EXPECT_INACTIVE) &&
		    nf_ct_exp_equal(tuple, i, zone, net)) {
			exp = i;
			break;
		}
	}
	if (!exp)
		return NULL;

	/* If master is not in hash table yet (ie. packet hasn't left
	   this machine yet), how can other end know about expected?
	   Hence these are not the droids you are looking for (if
	   master ct never got confirmed, we'd hold a reference to it
	   and weird things would happen to future packets). */
	if (!nf_ct_is_confirmed(exp->master))
		return NULL;                  /* 主连接未确认，忽略 */
	/* ... */
}
```

**调用时机**：`init_conntrack`（§7.6）新建 conntrack 时，先查 expectation。若命中，新连接被标记 `IPS_EXPECTED`，`ctinfo = IP_CT_RELATED`，`ct->master = exp->master`（关联到控制连接）。

**关键约束**：`!nf_ct_is_confirmed(exp->master)` —— 主连接必须已确认（包已离开本机）。否则 FTP 控制连接的 `PORT` 命令还在路上，数据连接不可能到来，匹配到也是误报。

### 15.6 完整流程图

```text
FTP 控制连接 (客户端:12345 → 服务器:21)
─────────────────────────────────────────
首包到达 PRE_ROUTING:
  conntrack: 新建 ct_ctrl (ctinfo=NEW)
  __nf_ct_try_assign_helper: tuple 匹配 FTP helper → 绑定 helper
  nf_confirm: ct_ctrl 确认

后续包（含 PORT 命令）:
  conntrack: ct_ctrl, ctinfo=ESTABLISHED
  helper.help 回调被调用:
    ├─ 解析 PORT 命令，提取数据连接端口 4567
    ├─ nf_ct_expect_alloc(ct_ctrl)
    ├─ nf_ct_expect_init(exp, ..., dport=4567)
    └─ nf_ct_expect_related(exp) → 插入 nf_ct_expect_hash

FTP 数据连接 (服务器:20 → 客户端:4567)
─────────────────────────────────────────
首包到达 PRE_ROUTING:
  conntrack: resolve_normal_ct
    └─ init_conntrack (新建 ct_data)
         └─ nf_ct_find_expectation(tuple)
              ├─ 匹配到 exp (master=ct_ctrl)
              ├─ set IPS_EXPECTED
              └─ ct_data->master = ct_ctrl
  ctinfo = IP_CT_RELATED

防火墙规则:
  iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
  └─ 数据连接因 ctinfo=RELATED 被放行，无需为动态端口开洞

数据连接结束:
  ct_data 超时或 FIN → 回收
  expectation 被删除（一次性，匹配后即移除）
```

### 15.7 NAT 与 helper 的协作

NAT 改了控制连接的地址端口后，`PORT` 命令里**内嵌的 IP/端口**也必须改（否则服务器连不到改后的地址）。这是 NAT helper 的职责：

- `nf_conntrack_ftp.c` 同时注册 NAT helper，在 `help` 回调里检测到 `PORT` 命令时，先做 NAT 改写（改载荷里的 IP/端口），再注册对应改写后的 expectation。
- `nf_nat_setup_info` 里若连接有 helper 且改了地址，会调 `nfct_seqadj_ext_add` 准备 TCP seq 调整（载荷长度变化导致 seq 偏移）。

### 15.8 现代实践：用户态 helper 与自动 helper 的衰落

**内核 helper 的局限**：

- 只支持少数协议（FTP/SIP/IRC/H.323/AMANDA），新协议要写内核模块
- 在内核解析应用层协议，复杂且易出 bug
- 性能开销：每个包都调 `help` 回调

**现代趋势**：

1. **用户态 helper**：`queue_num` 字段允许把报文送到用户态 helper 进程（类似 nfqueue），解析后通过 ctnetlink 注册 expectation。`conntrackd`、`nfqws` 等用此机制。
2. **`nf_conntrack_helper` 默认禁用**：现代内核（v5.x+）默认 `sysctl net.netfilter.nf_conntrack_helper=0`，必须显式 `-m helper` 或 `ct helper set` 才启用自动绑定。这是安全加固——避免意外触发 helper。
3. **eBPF 取代**：Cilium 等 eBPF 方案用 BPF 程序做应用层解析和连接跟踪，完全绕开内核 helper 机制。

```bash
# 查看已加载的 helper
cat /proc/net/nf_conntrack_expect

# 显式启用 FTP helper（nftables）
nft add rule ... ct helper set "ftp"

# 临时开启自动 helper（不推荐生产）
sysctl -w net.netfilter.nf_conntrack_helper=1
```

### 15.9 相关 ICMP 处理

ICMP 差错报文（如 `destination unreachable`）内嵌了"触发差错的原始报文"的头部。conntrack 用 `nf_conntrack_handle_icmp`（§7.5）解析内嵌头部，匹配到原始连接后，把 ICMP 包标记为 `IP_CT_RELATED`——这与 helper expectation 机制独立，但概念相同：**一条连接的报文关联到另一条已存在连接**。

---

## 16. 总结

Netfilter 是 Linux 网络的"控制平面"，把 5 个 hook 点埋在 IP 协议栈的关键路径上，通过 `NF_HOOK` 宏把报文交给外部模块处理。核心设计：

- **不可变 hook 数组 + RCU**：读路径无锁，写路径整体替换（§4、§5）
- **静态键 `nf_hooks_needed`**：未注册 hook 的关卡零开销（§4.5）
- **conntrack + NAT 分层**：conntrack 是基础，NAT 是其上的附加信息（§7、§8）
- **iptables → nftables 迁移**：从线性匹配到字节码虚拟机，快速表达式内联（§6、§9）
- **nfqueue**：用户态裁决，断点续跑（§13）
- **nf_flowtable**：已建立连接旁路 Netfilter 软件路径（§14）
- **helper + expectation**：相关连接自动识别（§15）

学完本文后，下一步是 eBPF/XDP——它把"拦截点"从 IP 层下沉到驱动 RX 软中断之前，是现代高性能网络方案（Cilium、Katran、Cloudflare）的基础。但 Netfilter 只是内核包过滤的**一支**，§17 把它与 XDP、TC BPF、cgroup skb、socket filter 摆进同一张全景图，作为进入 eBPF/XDP 的地图。

---

## 17. 内核包过滤挂载点全景图

> Netfilter 只是 Linux 内核"包过滤"的**一支**——内核里还有 XDP、TC BPF、cgroup skb、socket filter 等多个挂载点。本节用一张图回答：① Netfilter 在整个过滤谱系里处于什么位置？② 各挂载点解决什么 Netfilter 解决不了的问题？③ 后续学 eBPF/XDP 时按什么顺序对位。每个挂载点的逐函数源码留到 `docs/ebpf-xdp-study.md`，本节只画清位置与边界。

### 17.1 一张图：一个包从网卡到 socket，会被谁拦

下面这张图把一个 RX 包（以及对应的 TX 包）从最底层网卡到最上层 socket 的完整路径画出来，标出每一处可挂载过滤程序的位置。**Netfilter 的 5 个 hook（§2）只是其中标 ★ 的一段**。

```text
                            RX 路径（收包方向，自下而上）
═════════════════════════════════════════════════════════════════════════════

   ┌─────────────┐
   │  NIC 硬件   │  收到以太网帧，DMA 到环形缓冲
   └──────┬──────┘
          │  触发硬中断 → NAPI poll
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ① XDP（native）   BPF_PROG_TYPE_XDP              │  ← 最早！
   │     驱动 xdp_buff（还不是 sk_buff），在 NAPI       │     在 sk_buff 诞生之前
   │     软中断上下文里运行，早于内存分配               │     就能 drop/redirect/改包
   │     入口：驱动 ndo_bpf → bpf_prog_run              │
   └──────┬──────────────────────────────────────────────┘
          │  XDP_PASS：驱动构造 sk_buff
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ② XDP（generic）  do_xdp_generic()                │  ← 没有 native 支持时的
   │     net/core/dev.c:5656                            │     软件回退，在
   │     netif_receive_generic_xdp()  dev.c:5576        │     netif_receive_skb 附近
   │     此时已是 sk_buff，比 native 晚、比下面的早     │
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ③ TC ingress（cls_bpf）  cls_bpf_classify()       │  ← qdisc ingress 钩子
   │     net/sched/cls_bpf.c:81                         │     sk_buff 已完整，
   │     BPF_PROG_TYPE_SCHED_CLS / SCHED_ACT            │     可改包、可重定向
   │     挂载点：sch_handle_ingress()  dev.c            │
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ④ Netfilter PRE_ROUTING  ★ NF_INET_PRE_ROUTING    │  ← 本文主角（§2~§9）
   │     ip_input.c:612  NF_HOOK → nf_hook_slow         │     IP 层入口，路由判定前
   │     iptables raw/mangle/nat、conntrack 在此启动     │
   ├─────────────────────────────────────────────────────┤
   │  ⑤ Netfilter LOCAL_IN  ★ NF_INET_LOCAL_IN         │
   │     ip_input.c:262                                 │     路由判定为本机、交 L4 前
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ⑥ cgroup ingress  BPF_CGROUP_RUN_PROG_INET_INGRESS│  ← 绑定到 socket 所属
   │     net/core/filter.c:148                          │     cgroup 的 BPF 程序
   │     → __cgroup_bpf_run_filter_skb  cgroup.c:1561   │     可做容器级 ACL
   │     BPF_PROG_TYPE_CGROUP_SKB                       │
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ⑦ Socket filter  sk_filter_trim_cap()             │  ← 最晚！离应用最近
   │     net/core/filter.c:133                          │     cBPF（SO_ATTACH_FILTER）
   │     被 tcp_filter()/udp.c:2403/sock.c:551 调用      │     或 eBPF（SO_ATTACH_BPF，
   │     BPF_PROG_TYPE_SOCKET_FILTER                    │     BPF_PROG_TYPE_SOCKET_FILTER）
   │     libpcap/tcpdump 的底层就是它                   │
   └──────┬──────────────────────────────────────────────┘
          ▼
      socket 接收队列 → 唤醒 recvmsg()


                            TX 路径（发包方向，自上而下）
═════════════════════════════════════════════════════════════════════════════

      sendmsg()/tcp_sendmsg() ── 构造 skb 入发送队列
          │
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ⑥' cgroup egress  BPF_CGROUP_RUN_PROG_INET_EGRESS │  ← ip_output.c:322/341
   │     → __cgroup_bpf_run_filter_skb                  │     本机出口的容器级过滤
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ④' Netfilter LOCAL_OUT  ★ NF_INET_LOCAL_OUT       │  ← ip_output.c:120
   │     本机产生报文、构造 IP 头后                       │
   ├─────────────────────────────────────────────────────┤
   │  ④' Netfilter POST_ROUTING ★ NF_INET_POST_ROUTING  │  ← ip_output.c:401/417/422
   │     即将进入邻居/网卡发送，SNAT/MASQUERADE 在此      │
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ③' TC egress（cls_bpf）  sch_handle_egress        │  ← qdisc egress 钩子
   │     BPF_PROG_TYPE_SCHED_CLS                        │
   └──────┬──────────────────────────────────────────────┘
          ▼
   ┌─────────────────────────────────────────────────────┐
   │  ②' XDP egress / devmap_xmit                       │  ← 较少用，多为 redirect
   └──────┬──────────────────────────────────────────────┘
          ▼
      dev_queue_xmit → qdisc → ndo_start_xmit → NIC
```

读图要点：

- **编号即执行顺序**：RX 方向 ①→⑦ 越来越晚、越来越接近应用；编号带 `'` 的是 TX 方向对应点。
- **★ 标 Netfilter**：①②③⑥⑦ 都不是 Netfilter，只有 ④⑤（及 TX 侧 ④'）是。Netfilter 占据的是 **IP 层这一段**，既不是最早也不是最晚。
- **两套数据结构边界**：①XDP native 工作在 `xdp_buff`（驱动私有、未分配 sk_buff），②及之后都工作在 `sk_buff`。这是 XDP native 性能碾压的根本原因——它跳过了 sk_buff 分配。
- **cBPF vs eBPF**：⑦ Socket filter 是 cBPF 的原生场景（1992 年 McCanne 发明 BPF 就是为它），2014 年 eBPF 扩展了它；①②③⑥ 的底层虚拟机统一在 `bpf_prog_run()`，eBPF verifier 是公共关卡。

### 17.2 逐挂载点补充

上图的程序类型、入口文件:行、能做什么已写明，此处只补图里放不下的信息：

| # | 挂载点 | 能力边界（图里没说的） | 谁调用它（典型调用者） |
|---|--------|----------------------|----------------------|
| ① | XDP native | 只能看 L2~L3 头，无 socket 上下文，无 conntrack | NAPI poll 内、驱动代码 |
| ② | XDP generic | 比 native 晚，性能优势大打折扣 | `netif_receive_skb` 路径 |
| ③ | TC ingress/egress | 在 qdisc 层，能看完整 L2~L4，但晚于 XDP | `sch_handle_ingress/egress` |
| ④ | **Netfilter 5 hooks** ★ | 只在 IP 层，ARP/bridge/三层以下够不着；规则线性扫描 | `NF_HOOK` 宏散布在 IP 收发路径 |
| ⑥ | cgroup skb | 只能 pass/drop，绑定到 cgroup，作用域受 cgroup 树限制 | `BPF_CGROUP_RUN_PROG_INET_INGRESS/EGRESS` |
| ⑦ | Socket filter | 只能 pass/drop，每个 socket 独立，无全局状态（除非用 map） | `tcp_filter`(tcp.h:1688)、`udp.c:2403`、`sock.c:551` 等 |

补充说明：

- **④ 与 Netfilter 的关系**：这一行就是本文前 16 节的全部内容。`BPF_PROG_TYPE_NETFILTER`（见 §18 TODO）是较新的融合点——让 eBPF 程序也能注册成 `nf_hook_ops`，本质仍是 Netfilter hook，只是后端从 iptables 字节码换成 eBPF 字节码。
- **⑦ 的两副面孔**：`sk_filter_trim_cap` 是**汇聚点**——无论 cBPF 还是 eBPF socket filter，最终都汇到这里。差异只在 `bpf_prog` 的来源（cBPF 经 `sk_unattached_filter_create` 转译，eBPF 经 `bpf_prog_get_type` 取 fd）。所以"Socket 滤包"的源码精读只需看这一个函数 + verdict 语义，不必单独成篇。
- **cBPF 是历史，eBPF 是现在**：①②③⑥⑦ 的 cBPF 路径在 v7.1 里大多经 `bpf_migrate_filter()` 转成 eBPF 再跑，cBPF 指令集已退化为"入口格式"。真正值得精读的是统一的 `bpf_prog_run()` 和 verifier。

### 17.3 为什么会有这么多挂载点：作用域与能力的差异

执行顺序与性能梯度上图已标清（①最早/最快 → ⑦最晚/最慢）。挂载点之所以这么多，是因为**作用域**和**能力**两个维度上各有取舍，没有任何一个能覆盖所有需求：

**作用域（越晚越聚焦到单个连接）**

| 挂载点 | 作用域 | 能看到什么 |
|--------|--------|-----------|
| ①XDP / ③TC | 网卡 / qdisc | L2~L4 头，看不到 socket、看不到进程 |
| ④Netfilter | 整个 netns | 5 元组，看不到进程（除非用 owner match） |
| ⑥cgroup | 一个 cgroup 树 | 关联到容器/进程组 |
| ⑦socket filter | 单个 socket | 唯一能精确到"某个连接"的挂载点 |

**能力（能否改包 / 送用户态 / 有连接状态）**

| 能力 | ①XDP | ③TC | ④Netfilter | ⑥cgroup | ⑦socket |
|------|------|-----|-----------|---------|---------|
| 改包内容 | ✅(L2/L3头) | ✅(L2~L4) | ✅(mangle/NAT) | ❌(只 verdict) | ❌(只 verdict) |
| 重定向到另一网卡 | ✅(DEVMAP) | ✅(redirect) | ⚠️(需 mark+route) | ❌ | ❌ |
| 送用户态裁决 | ❌ | ❌ | ✅(nfqueue) | ❌ | ❌ |
| 连接跟踪状态 | ❌ | ❌ | ✅(conntrack) | ❌ | ❌ |

按需求选挂载点：drop 大流量 DDoS → ①XDP（sk_buff 分配前就 drop）；精细策略且要性能 → ③TC BPF；复用 iptables 生态/做 NAT → ④Netfilter（生态最全但最慢）；容器级 ACL → ⑥cgroup；应用自己做协议过滤 → ⑦socket filter。

Netfilter 慢在哪：conntrack 表查找、iptables 规则线性扫描、每包 `nf_hook_state` 初始化——XDP/TC BPF 用 map + verifier 直连，绕开这些。但 XDP 能力太弱（无 conntrack、不能送用户态），适合做 Netfilter 前面的"挡箭牌"而非替代品。Cilium 的架构正是：①XDP 做 L4 LB + early drop，③TC BPF 做策略，④Netfilter 逐渐退役。

### 17.4 与本文的衔接：Netfilter 在谱系里的定位

Netfilter 的独特价值来自"中间位置 + 能力最全"：

1. **它是最早"可改包 + 有连接状态"的挂载点**。①②③ 都早于它，但要么无 conntrack（XDP）、要么 conntrack 要自己用 map 实现（TC BPF）。Netfilter 的 conntrack/NAT 生态（§7、§8）是它至今无法被完全替代的根本原因。

2. **它是唯一能送用户态裁决的挂载点**。nfqueue（§13）让 IDS/IPS 能在内核里"挂起"一个包等用户态判决，XDP/TC/socket filter 都做不到。这是 suricata 等工具的根基。

3. **它的"慢"是设计选择，不是缺陷**。`nf_hook_slow` + conntrack 查找的开销，换来的是"任意优先级 hook + 完整连接状态 + 用户态回注"的能力组合。当你不需要这些能力时，才该把过滤前移到 ①③。

4. **`BPF_PROG_TYPE_NETFILTER` 是融合而非替代**。§18 TODO 里这条，本质是让 eBPF 程序作为 `nf_hook_ops` 注册进 Netfilter 的 5 个 hook——位置没变、能力没变，只是后端从 iptables/nftables 字节码换成 eBPF 字节码。所以学完本文再学它，几乎没有新概念，只是换一种"写 hook"的语言。

### 17.5 后续学习的对位建议

按这张全景图，`docs/ebpf-xdp-study.md` 的建议结构（与本文呼应）：

- **先精读 ①XDP**：它是 eBPF 性能护城河的源头。重点 `netif_receive_generic_xdp`（generic 路径，已有 sk_buff 易懂）、`bpf_prog_run`、verifier 对 XDP 程序的限制。
- **再讲 ③TC BPF**：`cls_bpf_classify` 与 qdisc 的衔接，正好复用本文 §11 可观测性里 `tc filter` 的命令对照。
- **然后讲 ⑥cgroup + ⑦socket filter**：这两个共享 `bpf_prog_run` 的 skb verdict 路径，`sk_filter_trim_cap` 是汇聚点，cgroup 的 ingress/egress 宏在本文已列出调用点。
- **最后讲 ④`BPF_PROG_TYPE_NETFILTER`**：作为"eBPF 回到 Netfilter"的收尾，对位本文 §4 `nf_hook_slow`——同一套 hook，换字节码引擎。

验收自检：

- [ ] 能不看图说出 ①~⑦ 七个 RX 挂载点的执行顺序与各自工作在 `xdp_buff` 还是 `sk_buff`
- [ ] 能解释为什么 XDP native 比 Netfilter 快一个数量级（sk_buff 分配 + conntrack + 线性规则）
- [ ] 能说清 `sk_filter_trim_cap` 为何是 cBPF/eBPF socket filter 的共同汇聚点
- [ ] 能判断一个需求该用哪个挂载点（DDoS drop → XDP；容器 ACL → cgroup；NAT → Netfilter；tcpdump → socket filter）

---

## 18. 后续可深入（TODO）

> 核心源码章节（§4~§9 共 6 节 + §13~§15 共 3 节扩展专题）已全部填充完成，对照 v7.1-rc7 源码逐行注解。以下为后续可深入专题：

- [x] §4.3 `nf_hook_slow` 完整源码 + 逐行注解（含 §4.4 批量路径、§4.5 静态键）
- [x] §5 `nf_register_net_hook` 完整源码（含 §5.1.3 `nf_hook_entries_grow`、§5.2 注销机制、§5.3 RCU 替换模型）
- [x] §6.2 `ipt_do_table` 完整匹配循环源码（含 verdict 编码、规则布局、性能特征）
- [x] §7 conntrack 完整状态机图 + `resolve_normal_ct` 源码（含 4 hook 注册点、`IPS_*` 位、ctinfo 决策树）
- [x] §8 `nf_nat_setup_info` + `nf_nat_packet` + `nf_nat_manip_pkt` 源码（含两阶段设计、完整 NAT 流程图）
- [x] §9.3 `nft_do_chain` 字节码解释器源码（含 §9.4 快速路径优化、§9.5 字节码布局、§9.7 性能对比）
- [x] §13 nfqueue 用户态收包路径（含 `__nf_queue`、`nf_reinject`、`nf_iterate` 断点续跑）
- [x] §14 `nf_flowtable` 流卸载（含 `flow_offload_add`、`nf_flow_offload_ip_hook` 快路径、与 conntrack 协作）
- [x] §15 `nf_conntrack_helper` + expectation（含 `nf_ct_expect_related`、`nf_ct_find_expectation`、FTP 完整流程）
- [ ] `nf_synproxy`：SYN flood 防御，在 Netfilter 层代理握手
- [ ] BPF hook (`NF_HOOK_OP_BPF` / `BPF_PROG_TYPE_NETFILTER`)：Netfilter 与 eBPF 的融合点，位置/能力不变、只换字节码后端（见 §17.4，衔接下一阶段 eBPF/XDP 学习）
- [ ] `nf_tables_offload`：nftables 规则卸载到网卡硬件（flow table 的规则级版本）
- [ ] `nf_conntrack_acct` / `nf_conntrack_timestamp`：conntrack 扩展区的计费与时间戳
