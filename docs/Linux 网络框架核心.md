# Linux 网络框架核心：Socket、net_device 与 sk_buff（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `include/linux/net.h` —— BSD socket 抽象：`struct socket`、`struct proto_ops`
> - `include/net/sock.h` —— 内核协议无关 socket：`struct sock`、`struct sock_common`、`struct proto`
> - `net/socket.c` —— socket 系统调用与通用 API
> - `net/ipv4/af_inet.c` —— IPv4 socket 创建与协议注册
> - `include/linux/netdevice.h` —— 网络设备：`struct net_device`、`struct napi_struct`、`struct packet_type`
> - `net/core/dev.c` —— 网络设备核心：`netif_receive_skb()`、`__netif_receive_skb_core()`、`dev_queue_xmit()`、NAPI 调度
> - `net/core/gro.c` —— GRO 接收卸载：`gro_receive_skb()`、`napi_gro_receive()`
> - `include/linux/skbuff.h` —— `struct sk_buff` 及核心内联 API
> - `net/core/skbuff.c` —— skbuff 分配、释放、拷贝、队列等实现
> - `net/core/sock.c` —— socket 初始化、内存记账、skb 与 socket 的绑定
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、内存布局图和时序图使用 `text` 代码块表示。

---

## 1. Linux Socket 抽象

Linux 网络栈把“用户看到的 socket”和“内核实际维护的 socket”拆成两个层次：

- **`struct socket`**：BSD socket 抽象，面向 VFS 和用户态，保存文件指针、类型、等待队列等。
- **`struct sock`**：内核协议无关的 socket 表示，保存状态、队列、内存记账、路由缓存等。
- **`struct proto_ops`**：面向用户态的系统调用处理表（bind/connect/sendmsg/recvmsg 等）。
- **`struct proto`**：面向内核的协议实现表（init/close/connect/sendmsg/backlog_rcv 等），同时是 socket 对象的 kmem_cache 管理器。

### 1.1 `struct socket`：用户态视角

定义位于 `include/linux/net.h:137`。

```c
struct socket {
	socket_state		state;

	short			type;

	unsigned long		flags;

	struct file		*file;
	struct sock		*sk;
	const struct proto_ops	*ops; /* Might change with IPV6_ADDRFORM or MPTCP. */

	struct socket_wq	wq;
};
```

| 字段 | 含义 |
|------|------|
| `state` | socket 状态：`SS_FREE`、`SS_UNCONNECTED`、`SS_CONNECTED` 等 |
| `type` | socket 类型：`SOCK_STREAM`、`SOCK_DGRAM`、`SOCK_RAW` 等 |
| `flags` | 标志位，如 `SOCK_NOSPACE`、`SOCK_SUPPORT_ZC` |
| `file` | 关联的 VFS `struct file`，用户通过 fd 操作 socket |
| `sk` | 指向内核 `struct sock` |
| `ops` | 协议族特定的 `proto_ops`，如 IPv4 的 `inet_stream_ops` |
| `wq` | socket 等待队列，用于 `poll()` / `select()` / 阻塞 IO |

### 1.2 `struct sock`：内核协议无关视角

定义位于 `include/net/sock.h:365`，开头嵌入 `struct sock_common`。下面只摘录与后续章节相关的关键字段（完整定义近 200 行，按 cacheline 分组排列）：

```c
struct sock {
	struct sock_common	__sk_common;
#define sk_node			__sk_common.skc_node
#define sk_nulls_node		__sk_common.skc_nulls_node
#define sk_refcnt		__sk_common.skc_refcnt
#define sk_tx_queue_mapping	__sk_common.skc_tx_queue_mapping
#define sk_state		__sk_common.skc_state      /* via sock_common */
	...

	struct sk_buff_head	sk_receive_queue;
	...
	refcount_t		sk_wmem_alloc;             /* sock.h:485 */
	...
	struct {
		atomic_t	rmem_alloc;
		int		len;
		struct sk_buff	*head;
		struct sk_buff	*tail;
	} sk_backlog;
#define sk_rmem_alloc sk_backlog.rmem_alloc      /* sock.h:426 */
	...
	struct sk_buff_head	sk_write_queue;
	...
	int			sk_rcvbuf;                  /* sock.h:443 */
	...
	u16			sk_protocol;                /* sock.h:512 */
	...
	int			sk_sndbuf;                  /* sock.h:526 */
	...
	struct socket		*sk_socket;
	...
};
```

下表列出常用字段及其真实位置，便于对照源码：

| 字段 | 来源 | 含义 |
|------|------|------|
| `sk_receive_queue` | `struct sock` 直接成员 | 收到的报文队列（已提交给 socket） |
| `sk_write_queue` | `struct sock` 直接成员 | 发送队列（如 TCP 原始发送队列） |
| `sk_wmem_alloc` | `struct sock` 直接成员（`refcount_t`，sock.h:485） | 当前 socket 已占用的发送内存（含在途 skb），初值为 `SK_WMEM_ALLOC_BIAS`=1，见 §1.7 |
| `sk_rmem_alloc` | **宏**，实为 `sk_backlog.rmem_alloc`（sock.h:426） | 当前 socket 已占用的接收内存。注意它和 backlog 长度共用一个匿名结构体，是 `atomic_t` 而非 `refcount_t` |
| `sk_sndbuf` / `sk_rcvbuf` | `struct sock` 直接成员（sock.h:526 / 443） | 发送/接收缓冲区上限（字节） |
| `sk_socket` | `struct sock` 直接成员 | 反向指针，指向 `struct socket` |
| `sk_protocol` | `struct sock` 直接成员（sock.h:512，`u16`） | 协议号，如 `IPPROTO_TCP`、`IPPROTO_UDP` |
| `sk_state` | **宏**，实为 `__sk_common.skc_state`（sock.h:389） | 连接状态，如 `TCP_CLOSE`、`TCP_ESTABLISHED`，放在 `sock_common` 里以便 timewait sock 共享 |
| `sk_refcnt` | **宏**，实为 `__sk_common.skc_refcnt` | socket 引用计数 |

`struct sock_common` 是 `struct sock`、`struct inet_timewait_sock`、`struct inet6_timewait_sock`、`struct request_sock` 等共享的最小公共部分，包含地址、端口、状态、hash 节点、引用计数等。把它独立出来是为了让体积更小的 timewait/request sock 不必背负完整 `struct sock`：

```c
struct sock_common {
	union {
		__addrpair	skc_addrpair;
		struct {
			__be32	skc_daddr;
			__be32	skc_rcv_saddr;
		};
	};
	union {
		__portpair	skc_portpair;
		struct {
			__be16	skc_dport;
			__u16	skc_num;
		};
	};

	unsigned short		skc_family;
	volatile unsigned char	skc_state;
	...
	struct proto		*skc_prot;
	possible_net_t		skc_net;
	...
	refcount_t		skc_refcnt;
};
```

### 1.3 `struct proto_ops` 与 `struct proto` 的分工

#### `proto_ops`：用户系统调用层

```c
struct proto_ops {
	int		family;
	int		(*release)   (struct socket *sock);
	int		(*bind)      (struct socket *sock, struct sockaddr_unsized *myaddr, int sockaddr_len);
	int		(*connect)   (struct socket *sock, struct sockaddr_unsized *vaddr, int sockaddr_len, int flags);
	int		(*accept)    (struct socket *sock, struct socket *newsock, struct proto_accept_arg *arg);
	int		(*listen)    (struct socket *sock, int len);
	int		(*shutdown)  (struct socket *sock, int flags);
	int		(*setsockopt)(struct socket *sock, int level, int optname, sockptr_t optval, unsigned int optlen);
	int		(*getsockopt)(struct socket *sock, int level, int optname, char __user *optval, int __user *optlen);
	int		(*sendmsg)   (struct socket *sock, struct msghdr *m, size_t total_len);
	int		(*recvmsg)   (struct socket *sock, struct msghdr *m, size_t total_len, int flags);
	...
};
```

#### `proto`：内核协议实现层

```c
struct proto {
	void		(*close)(struct sock *sk, long timeout);
	int		(*connect)(struct sock *sk, struct sockaddr_unsized *uaddr, int addr_len);
	struct sock	*(*accept)(struct sock *sk, struct proto_accept_arg *arg);
	int		(*ioctl)(struct sock *sk, int cmd, int *karg);
	int		(*init)(struct sock *sk);
	void		(*destroy)(struct sock *sk);
	int		(*sendmsg)(struct sock *sk, struct msghdr *msg, size_t len);
	int		(*recvmsg)(struct sock *sk, struct msghdr *msg, size_t len, int flags);
	int		(*bind)(struct sock *sk, struct sockaddr_unsized *addr, int addr_len);
	int		(*backlog_rcv)(struct sock *sk, struct sk_buff *skb);
	...
	/* socket 对象池 */
	struct kmem_cache	*slab;
	unsigned int		obj_size;
	...
};
```

**关系**：`proto_ops` 处理“用户态如何与 socket 交互”，`proto` 处理“这个协议在内核里如何真正实现”。例如 IPv4 TCP 的 `proto_ops` 是 `inet_stream_ops`，而其 `proto` 是 `tcp_prot`。

### 1.4 socket 创建流程

用户态调用 `socket(AF_INET, SOCK_STREAM, 0)` 后：

```text
SYSCALL_DEFINE3(socket, family, type, protocol)
  → __sys_socket()
    → __sys_socket_create()
      → sock_create()
        → __sock_create()
          → sock_alloc()                  分配 struct socket + inode
          → 根据 family 找到 net_proto_family
          → pf->create()                  对 IPv4 即 inet_create()
            → 在 inetsw[sock->type] 链表匹配 protocol/type
            → sock->ops = answer->ops     如 inet_stream_ops
            → sk_alloc()                  从 tcp_prot->slab 分配 struct sock
            → sock_init_data(sock, sk)    初始化 sk 并与 socket 互相关联
            → 协议特定初始化（如 tcp_v4_init_sock）
      → sock_map_fd()                     分配 fd 并关联 file
```

`__sock_create()` 核心逻辑：

```c
int __sock_create(struct net *net, int family, int type, int protocol,
		  struct socket **res, int kern)
{
	struct socket *sock;
	const struct net_proto_family *pf;

	if (family < 0 || family >= NPROTO)
		return -EAFNOSUPPORT;
	if (type < 0 || type >= SOCK_MAX)
		return -EINVAL;

	sock = sock_alloc();
	if (!sock)
		return -ENFILE;

	sock->type = type;

	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);
	...
	rcu_read_unlock();

	err = pf->create(net, sock, protocol, kern);
	...
	*res = sock;
	return 0;
}
```

`sock_alloc()` 分配 socket 时顺便创建一个 sockfs inode：

```c
struct socket *sock_alloc(void)
{
	struct inode *inode;
	struct socket *sock;

	inode = new_inode_pseudo(sock_mnt->mnt_sb);
	if (!inode)
		return NULL;

	sock = SOCKET_I(inode);

	inode->i_ino = get_next_ino();
	inode->i_mode = S_IFSOCK | S_IRWXUGO;
	inode->i_uid = current_fsuid();
	inode->i_gid = current_fsgid();
	inode->i_op = &sockfs_inode_ops;

	return sock;
}
```

### 1.5 IPv4 协议注册：`inet_family_ops` 与 `inetsw`

IPv4 协议族通过 `inet_family_ops` 注册到 `net_families[PF_INET]`：

```c
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};
```

而 IPv4 内部用 `inetsw` 数组按 `type` 管理协议实现：

```c
static struct inet_protosw inetsw_array[] =
{
	{
		.type =       SOCK_STREAM,
		.protocol =   IPPROTO_TCP,
		.prot =       &tcp_prot,
		.ops =        &inet_stream_ops,
		.flags =      INET_PROTOSW_PERMANENT | INET_PROTOSW_ICSK,
	},

	{
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_UDP,
		.prot =       &udp_prot,
		.ops =        &inet_dgram_ops,
		.flags =      INET_PROTOSW_PERMANENT,
       },

       {
		.type =       SOCK_DGRAM,
		.protocol =   IPPROTO_ICMP,
		.prot =       &ping_prot,
		.ops =        &inet_sockraw_ops,
		.flags =      INET_PROTOSW_REUSE,
       }
};
```

`inet_create()` 根据 `sock->type` 和 `protocol` 在该数组中匹配，得到对应的 `ops` 和 `prot`：

```c
static int inet_create(struct net *net, struct socket *sock, int protocol,
		       int kern)
{
	struct sock *sk;
	struct inet_protosw *answer;
	struct proto *answer_prot;
	...
	sock->state = SS_UNCONNECTED;

lookup_protocol:
	rcu_read_lock();
	list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
		if (protocol == answer->protocol) {
			if (protocol != IPPROTO_IP)
				break;
		} else {
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
	}
	...
	sock->ops = answer->ops;
	answer_prot = answer->prot;
	...
	sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern);
	...
	sock_init_data(sock, sk);
	...
	sk->sk_protocol = protocol;
	sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;
	...
}
```

### 1.6 `sock_init_data()`：绑定 socket 与 sock

```c
void sock_init_data_uid(struct socket *sock, struct sock *sk, kuid_t uid)
{
	sk_init_common(sk);
	...
	sk->sk_allocation	= GFP_KERNEL;
	sk->sk_rcvbuf		= READ_ONCE(sysctl_rmem_default);
	sk->sk_sndbuf		= READ_ONCE(sysctl_wmem_default);
	sk->sk_state		= TCP_CLOSE;
	sk_set_socket(sk, sock);

	if (sock) {
		sk->sk_type	= sock->type;
		RCU_INIT_POINTER(sk->sk_wq, &sock->wq);
		sock->sk	= sk;
	}
	...
	sk->sk_state_change	= sock_def_wakeup;
	sk->sk_data_ready	= sock_def_readable;
	sk->sk_write_space	= sock_def_write_space;
	sk->sk_error_report	= sock_def_error_report;
	sk->sk_destruct		= sock_def_destruct;
	...
	refcount_set(&sk->sk_refcnt, 1);
}
```

该函数完成 `struct socket` 与 `struct sock` 的双向关联，并设置默认回调（如收到数据时唤醒等待进程）。

### 1.7 sk_buff 与 socket 的关系：内存归属

sk_buff 通过 `skb->sk` 和 `skb->destructor` 关联到 socket，实现发送/接收内存记账。

#### 发送路径：`skb_set_owner_w()`

```c
void skb_set_owner_w(struct sk_buff *skb, struct sock *sk)
{
	int old_wmem;

	skb_orphan(skb);
#ifdef CONFIG_INET
	if (unlikely(!sk_fullsock(sk)))
		return skb_set_owner_edemux(skb, sk);
#endif
	skb->sk = sk;
	skb->destructor = sock_wfree;
	skb_set_hash_from_sk(skb, sk);

	__refcount_add(skb->truesize, &sk->sk_wmem_alloc, &old_wmem);

	skb->ooo_okay = (old_wmem == SK_WMEM_ALLOC_BIAS);
}
```

- 把 skb 挂到 socket 发送内存配额 `sk_wmem_alloc`：`__refcount_add(skb->truesize, &sk->sk_wmem_alloc, &old_wmem)` 同时返回加之前的旧值 `old_wmem`，供下面 `ooo_okay` 判断使用。
- 设置 destructor 为 `sock_wfree`，发送完成释放 skb 时由它递减 `sk_wmem_alloc`。
- `ooo_okay` 是给多队列网卡的 TX queue 选择函数 `netdev_pick_tx()` 看的提示位，**不是直接控制网卡硬件**。

#### `ooo_okay` 与 `SK_WMEM_ALLOC_BIAS`

```c
	skb->ooo_okay = (old_wmem == SK_WMEM_ALLOC_BIAS);
```

`SK_WMEM_ALLOC_BIAS` 定义为 1（`include/net/sock.h:2340`）。socket 创建时 `sk_wmem_alloc` 被初始化为 `SK_WMEM_ALLOC_BIAS`（`net/core/sock.c:2329`）：

```c
	refcount_set(&sk->sk_wmem_alloc, SK_WMEM_ALLOC_BIAS);
```

这个 1 是**留给 socket 自身的引用**，不归任何 skb 所有。因此：

- `old_wmem == SK_WMEM_ALLOC_BIAS`（即 == 1）说明扣减前没有任何在途发送 skb，**当前 skb 是该 socket 的第一个待发送包**。此时 `netdev_pick_tx()` 可以按 XPS 任意选 TX queue，不存在乱序风险，置 `ooo_okay = 1`。
- 否则说明已有在途 skb（在 qdisc 或 NIC 队列里），若换 queue 并行发送可能导致 TCP 乱序，置 `ooo_okay = 0`，`netdev_pick_tx()` 会保持原 queue 选择。

注释原文（`net/core/sock.c:2742-2747`）：

```c
	/* (old_wmem == SK_WMEM_ALLOC_BIAS) if no other TX packet for this socket
	 * is in a host queue (qdisc, NIC queue).
	 * Set skb->ooo_okay so that netdev_pick_tx() can choose a TX queue
	 * based on XPS for better performance.
	 * Otherwise clear ooo_okay to not risk Out Of Order delivery.
	 */
	skb->ooo_okay = (old_wmem == SK_WMEM_ALLOC_BIAS);
```

#### `sock_wfree()`：发送完成时的内存归还与 socket 释放

`sock_wfree()` 在 skb 释放时被调用，完整源码（`net/core/sock.c:2671`）：

```c
void sock_wfree(struct sk_buff *skb)
{
	unsigned int len = skb->truesize;
	struct sock *sk = skb->sk;
	bool free;
	int old;

	if (!sock_flag(sk, SOCK_USE_WRITE_QUEUE)) {
		void (*sk_write_space)(struct sock *sk);

		sk_write_space = READ_ONCE(sk->sk_write_space);

		if (sock_flag(sk, SOCK_RCU_FREE) &&
		    sk_write_space == sock_def_write_space) {
			rcu_read_lock();
			free = __refcount_sub_and_test(len, &sk->sk_wmem_alloc,
						       &old);
			sock_def_write_space_wfree(sk, old - len);
			rcu_read_unlock();
			if (unlikely(free))
				__sk_free(sk);
			return;
		}

		/*
		 * Keep a reference on sk_wmem_alloc, this will be released
		 * after sk_write_space() call
		 */
		WARN_ON(refcount_sub_and_test(len - 1, &sk->sk_wmem_alloc));
		sk_write_space(sk);
		len = 1;
	}
	/*
	 * if sk_wmem_alloc reaches 0, we must finish what sk_free()
	 * could not do because of in-flight packets
	 */
	if (refcount_sub_and_test(len, &sk->sk_wmem_alloc))
		__sk_free(sk);
}
```

这里有三条路径，体现了一套精心设计的「原子递减 + 唤醒 + 延迟释放」机制：

**路径 A：TCP 快速路径（`SOCK_USE_WRITE_QUEUE` 置位）**

TCP socket 在 `tcp_v4_init_sock()` 里会置 `SOCK_USE_WRITE_QUEUE`，直接跳过整个 `if` 块，只做最后的 `refcount_sub_and_test(len, ...)`。TCP 自己通过 `sk_stream_wait_memory()` / `tcp_check_space()` 管理发送内存唤醒，不需要 `sock_wfree` 重复唤醒。注意内核还有个 `__sock_wfree()`（`net/core/sock.c:2715`）专门用于这种场景，注释明确写「This variant of sock_wfree() is used by TCP, since it sets SOCK_USE_WRITE_QUEUE」。

**路径 B：通用快速路径（`SOCK_RCU_FREE` + 默认 `sk_write_space`）**

针对最常见的非 TCP socket 且用默认回调的场景，用 `__refcount_sub_and_test(len, ...)` **一次性原子地**完成「递减 `sk_wmem_alloc` + 拿到递减后的旧值 `old` + 判断是否归零」三件事，避免多步操作间的 race。然后用 `old - len`（即递减后的值）调用专用唤醒函数 `sock_def_write_space_wfree()`。这条路径直接 `return`，不走通用路径。

**路径 C：通用慢速路径（其它非 TCP socket）**

这是笔记必须重点理解的设计。代码先减 `len - 1`，调用 `sk_write_space(sk)` 唤醒等待发送的进程，然后把 `len` 置为 1，由末尾的 `refcount_sub_and_test(len, ...)` 减掉最后这 1。

**为什么先减 `len - 1` 再 `len = 1`？** 注释写得很清楚：「Keep a reference on sk_wmem_alloc, this will be released after sk_write_space() call」。如果在调用回调 `sk_write_space()` 之前就把 `len` 全减完，`sk_wmem_alloc` 可能先归零，触发 socket 被 `__sk_free()` 异步释放；而 `sk_write_space()` 内部还要访问 `sk`，就会 use-after-free。故意留 1 不减，等回调返回后再减最后这 1，保证回调执行期间 socket 一定活着。`WARN_ON(refcount_sub_and_test(...))` 还顺带断言「减 `len-1` 不该让计数归零」——若归零说明 `sk_wmem_alloc` 记账出了 bug。

**末尾的最终释放**：三条路径最终都可能走到 `if (refcount_sub_and_test(len, &sk->sk_wmem_alloc)) __sk_free(sk);`。只有 `sk_wmem_alloc` 真正归零（即 socket 自身引用的那 1 也被减掉）才调用 `__sk_free(sk)` 完成 socket 释放。这承接了 `sk_free()` 的「延迟释放」约定：`sk_free()` 不会立即释放 socket，而是等所有在途 skb 都通过 `sock_wfree()` 归还内存后，由最后一个 `sock_wfree()` 触发真正的 `__sk_free()`。

#### 接收路径：`skb_set_owner_r()`

```c
static inline void skb_set_owner_r(struct sk_buff *skb, struct sock *sk)
{
	skb_orphan(skb);
	skb->sk = sk;
	skb->destructor = sock_rfree;
	atomic_add(skb->truesize, &sk->sk_rmem_alloc);
	sk_mem_charge(sk, skb->truesize);
}
```

- 把 skb 计入 `sk_rmem_alloc` 和 `sk_forward_alloc`。
- 释放时 `sock_rfree()` 自动递减，避免 socket 内存泄漏。

#### 内存记账：`sk_mem_charge()` / `sk_mem_uncharge()`

```c
static inline void sk_mem_charge(struct sock *sk, int size)
{
	if (!sk_has_account(sk))
		return;
	sk_forward_alloc_add(sk, -size);
}

static inline void sk_mem_uncharge(struct sock *sk, int size)
{
	if (!sk_has_account(sk))
		return;
	sk_forward_alloc_add(sk, size);
	sk_mem_reclaim(sk);
}
```

`sk_forward_alloc` 是 socket 预先分配的一段配额，避免每次分配 skb 都触碰全局计数器。`sk_mem_charge()` 从预分配中扣减，不足时再向协议全局内存申请。

### 1.8 socket 系统调用到 sk_buff 的完整映射

```text
用户态
  socket(AF_INET, SOCK_STREAM, 0)
    → 分配 struct socket + struct sock，建立 socket ↔ sk 关联

  send(fd, buf, len, 0)
    → sock_sendmsg() → inet_sendmsg() → tcp_sendmsg()
      → 分配 skb，skb_reserve() / skb_put() 填充数据
      → skb_set_owner_w(skb, sk)        计入发送内存
      → tcp_transmit_skb() → ip_queue_xmit() → dev_queue_xmit()
        → 网卡发送完成 → consume_skb() → sock_wfree() 释放发送内存

  recv(fd, buf, len, 0)
    → sock_recvmsg() → inet_recvmsg() → tcp_recvmsg()
      → 从 sk_receive_queue 取出 skb
      → 拷贝数据到用户态
      → consume_skb() → sock_rfree()    释放接收内存
```

---

## 2. net_device 与 NAPI：L2 与 L3 的交接

如果说 `struct sk_buff` 是报文容器、`struct socket`/`struct sock` 是端到端传输的端点抽象，那么 **`struct net_device`** 就是 Linux 网络栈中**网络设备的内核表示**，负责把skb 从网卡硬件搬运到协议栈（RX），或从协议栈搬运到网卡硬件（TX）。

**NAPI**（New API）是 Linux 用于高吞吐收包的一套机制：用**轮询（polling）**代替传统每个包都触发中断，减少中断风暴。

### 2.1 `struct net_device`：网络设备的内核表示

定义位于 `include/linux/netdevice.h:2124`，字段极多，可按热路径分组理解。下面的代码块为**按热路径分组的教学性摘录**：真实结构体中各字段唯一，此处为体现「哪些字段落在哪条 cacheline 分组」而把同一字段（如 `features`）在不同分组里重复展示，并省略大量成员：

```c
struct net_device {
	/* TX read-mostly hotpath */
	const struct net_device_ops *netdev_ops;
	const struct header_ops *header_ops;
	struct netdev_queue	*_tx;
	netdev_features_t	features;
	unsigned int		mtu;
	unsigned int		gso_max_size;
	unsigned short		needed_headroom;

	/* TXRX read-mostly hotpath */
	unsigned long		state;
	unsigned int		flags;
	unsigned short		hard_header_len;
	netdev_features_t	features;	/* 同上，按分组重复展示 */

	/* RX read-mostly hotpath */
	struct bpf_prog __rcu	*xdp_prog;
	struct list_head	ptype_specific;
	int			ifindex;
	unsigned int		real_num_rx_queues;
	struct netdev_rx_queue	*_rx;
	rx_handler_func_t __rcu	*rx_handler;
	possible_net_t		nd_net;

	char			name[IFNAMSIZ];
	struct list_head	dev_list;
	struct list_head	napi_list;
	struct list_head	ptype_all;

	unsigned short		type;
	unsigned char		addr_len;
	unsigned char		perm_addr[MAX_ADDR_LEN];
	unsigned char		dev_addr[MAX_ADDR_LEN];

	/* 协议特定指针 */
	struct in_device __rcu	*ip_ptr;
	struct inet6_dev __rcu	*ip6_ptr;
	...
};
```

| 字段 | 含义 |
|------|------|
| `netdev_ops` | 设备操作集：`ndo_open()`、`ndo_stop()`、`ndo_start_xmit()` 等 |
| `header_ops` | 二层头操作：`create()`、`parse()`、`parse_protocol()` 等 |
| `features` | 设备 offload 能力：TSO/GSO/CSUM/VLAN 等 |
| `mtu` | 最大传输单元 |
| `ifindex` | 接口索引，全局唯一 |
| `name` | 接口名，如 `eth0`、`lo` |
| `ptype_all` / `ptype_specific` | 注册在该设备上的报文类型处理函数列表 |
| `napi_list` | 设备关联的 NAPI 实例列表 |
| `state` | 设备状态：`__LINK_STATE_START` 等 |
| `flags` | 设备标志：`IFF_UP`、`IFF_BROADCAST`、`IFF_NOARP` 等 |
| `type` | ARPHRD 类型，如 `ARPHRD_ETHER` |
| `addr_len` | MAC 地址长度 |
| `ip_ptr` / `ip6_ptr` | 指向 IPv4/IPv6 在该设备上的配置块 |

### 2.2 `struct net_device_ops` 与 `struct header_ops`

每个网卡驱动都会实例化一个 `net_device_ops`：

```c
struct net_device_ops {
	int			(*ndo_init)(struct net_device *dev);
	void			(*ndo_uninit)(struct net_device *dev);
	int			(*ndo_open)(struct net_device *dev);
	int			(*ndo_stop)(struct net_device *dev);
	netdev_tx_t		(*ndo_start_xmit)(struct sk_buff *skb,
						  struct net_device *dev);
	netdev_features_t	(*ndo_features_check)(struct sk_buff *skb,
						      struct net_device *dev,
						      netdev_features_t features);
	u16			(*ndo_select_queue)(struct net_device *dev,
						    struct sk_buff *skb,
						    struct net_device *sb_dev);
	int			(*ndo_set_mac_address)(struct net_device *dev,
						       void *addr);
	int			(*ndo_validate_addr)(struct net_device *dev);
	int			(*ndo_change_mtu)(struct net_device *dev,
						  int new_mtu);
	void			(*ndo_tx_timeout)(struct net_device *dev,
						  unsigned int txqueue);
	...
};
```

`header_ops` 处理二层头：

```c
struct header_ops {
	int	(*create)(struct sk_buff *skb, struct net_device *dev,
			  unsigned short type, const void *daddr,
			  const void *saddr, unsigned int len);
	int	(*parse)(const struct sk_buff *skb,
			 const struct net_device *dev,
			 unsigned char *haddr);
	__be16	(*parse_protocol)(const struct sk_buff *skb);
	bool	(*validate)(const char *ll_header, unsigned int len);
	...
};
```

### 2.3 `struct packet_type`：二层到三层的分发器

协议栈在三类链表上注册 `packet_type`，内核根据 `skb->protocol`（即以太网类型字段，如 `ETH_P_IP`、`ETH_P_ARP`）找到对应的处理函数。这三类链表分属不同作用域：

| 链表 | 作用域 | 定义位置 | 用途 |
|------|--------|----------|------|
| `ptype_all` | per-net（`net->ptype_all`）和 per-device（`dev->ptype_all`） | `include/net/net_namespace.h` / `include/linux/netdevice.h` | 抓包：监听所有 ethertype（`ETH_P_ALL`），如 tcpdump / AF_PACKET |
| `ptype_specific` | per-net（`net->ptype_specific`）和 per-device（`dev->ptype_specific`） | 同上 | 绑定到具体 net 或 dev 的特定 ethertype handler |
| `ptype_base[]` | **全局**（hash 表，`PTYPE_HASH_SIZE` 桶） | `net/core/dev.c:172` | 未绑定 dev/net 的特定 ethertype handler，按 `ntohs(type) & PTYPE_HASH_MASK` 散列 |

```c
struct packet_type {
	__be16			type;	/* htons(ether_type) */
	bool			ignore_outgoing;
	struct net_device	*dev;	/* NULL 表示通配 */
	int			(*func)(struct sk_buff *skb,
					struct net_device *dev,
					struct packet_type *pt,
					struct net_device *orig_dev);
	struct list_head	list;
};
```

IPv4 注册示例（`net/ipv4/af_inet.c`）。注意 `dev` 和 `af_packet_net` 都没填，因此会被归到全局 `ptype_base`：

```c
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
	.list_func = ip_list_rcv,
};
```

注册与注销的入口都是 `dev_add_pack()` / `__dev_remove_pack()`，真正决定挂到哪条链表的是 `ptype_head()`（`net/core/dev.c:594`）：

```c
static inline struct list_head *ptype_head(const struct packet_type *pt)
{
	if (pt->type == htons(ETH_P_ALL)) {
		if (!pt->af_packet_net && !pt->dev)
			return NULL;

		return pt->dev ? &pt->dev->ptype_all :
				 &pt->af_packet_net->ptype_all;
	}

	if (pt->dev)
		return &pt->dev->ptype_specific;

	return pt->af_packet_net ? &pt->af_packet_net->ptype_specific :
				 &ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];
}
```

分流规则：

- **`type == ETH_P_ALL`**（抓包）：必须显式指定 `dev` 或 `af_packet_net`，否则返回 NULL 拒绝注册。指定 `dev` 挂 per-device `dev->ptype_all`，否则挂 per-net `af_packet_net->ptype_all`。
- **`type != ETH_P_ALL` 且 `dev` 非空**：挂 per-device `dev->ptype_specific`。AF_PACKET socket 绑定到具体网卡（`SO_BINDTODEVICE`）时走这条。
- **`type != ETH_P_ALL` 且 `dev` 为空**：若填了 `af_packet_net` 挂 per-net `af_packet_net->ptype_specific`，否则挂**全局** `ptype_base[hash]`。IPv4/ARP 的协议 handler 走最后这条。

```c
void dev_add_pack(struct packet_type *pt)
{
	struct list_head *head = ptype_head(pt);

	if (WARN_ON_ONCE(!head))
		return;

	spin_lock(&ptype_lock);
	list_add_rcu(&pt->list, head);
	spin_unlock(&ptype_lock);
}
```


### 2.4 NAPI 机制：`struct napi_struct`

```c
struct napi_struct {
	unsigned long		state;
	struct list_head	poll_list;
	int			weight;
	int			(*poll)(struct napi_struct *, int);
	struct net_device	*dev;
	struct sk_buff		*skb;
	struct gro_node		gro;
	...
};
```

| 字段 | 含义 |
|------|------|
| `state` | NAPI 状态：`NAPI_STATE_SCHED` 表示已调度 |
| `poll_list` | 挂到 per-CPU `softnet_data.poll_list` |
| `weight` / `budget` | 本轮轮询最多处理的包数 |
| `poll` | 驱动提供的轮询函数 |
| `dev` | 关联的 net_device |
| `gro` | GRO 聚合状态 |

NAPI 状态常量：

```c
enum {
	NAPI_STATE_SCHED,		/* Poll is scheduled */
	NAPI_STATE_MISSED,		/* reschedule a napi */
	NAPI_STATE_DISABLE,		/* Disable pending */
	NAPI_STATE_NPSVC,		/* Netpoll - don't dequeue from poll_list */
	NAPI_STATE_LISTED,		/* NAPI added to system lists */
	...
};
```

### 2.5 典型 NAPI 驱动收包流程

```text
网卡收到报文 → 触发硬中断
  → 驱动中断处理函数
    → napi_schedule(napi)          关闭硬中断，把 napi 加入 poll_list
    → raise NET_RX_SOFTIRQ

NET_RX_SOFTIRQ 处理（net_rx_action）
  → 遍历 poll_list
    → napi->poll(napi, budget)     驱动轮询函数
      → 从 RX ring 取描述符
      → build_skb() / napi_alloc_skb() 构造 skb
      → eth_type_trans(skb, dev)   设置 skb->protocol 和 pkt_type
      → napi_gro_receive(napi, skb) 或 netif_receive_skb(skb)
        → __netif_receive_skb_core()
          → VLAN 解标签
          → TC ingress / netfilter
          → rx_handler（桥接、 bonding、macvlan 等）
          → deliver_ptype_list_skb() 按 ethertype 分发
            → ip_rcv() / arp_rcv() / ipv6_rcv()
      → 处理完 budget 个包后
        → napi_complete_done(napi, work_done)  重新开硬中断
```

### 2.6 接收路径的函数层次：谁才是「真正的入口」

先说结论：**v7.x 中 `netif_receive_skb()` 并不是 NAPI 驱动的热路径入口**。NAPI 驱动走的是 `napi_gro_receive()` → `gro_normal_one()` → `netif_receive_skb_list()` 这条批量路径（见 §2.8）；`netif_receive_skb()` 只是单个 skb 投递的便捷封装，以及非 NAPI 场景（如 loopback、tunnel）的入口。

真正完成 L2→L3 分发的核心是 `__netif_receive_skb_core()`。理解下面这条层次链是阅读 `net/core/dev.c` 收包代码的先决条件：

```text
netif_receive_skb(skb)                      单个 skb 的便捷入口（非 NAPI 热路径）
  └─ netif_receive_skb_internal(skb)        时间戳预处理 + RPS 判定
       ├─ [RPS 命中] enqueue_to_backlog()   跨 CPU 转发到目标 CPU 的 backlog
       └─ __netif_receive_skb(skb)          PFMEMALLOC 判定
            └─ __netif_receive_skb_one_core(skb, pfmemalloc)
                 ├─ __netif_receive_skb_core(&skb, pfmemalloc, &pt_prev)
                 │       ↑ 真正的 L2 分发核心：ptype_all / TC / VLAN /
                 │         rx_handler / ptype_base，返回最后一个待调用的 pt_prev
                 └─ pt_prev->func(skb, ...)  调用如 ip_rcv / arp_rcv / ipv6_rcv

netif_receive_skb_list(head)                批量入口（NAPI + GRO 的 GRO_NORMAL 路径）
  └─ netif_receive_skb_list_internal(head)  同样做 RPS，但按 sublist 批量处理
       └─ __netif_receive_skb_list(head)
            └─ （对每个 skb）__netif_receive_skb_one_core(...)
```

关键设计点：`__netif_receive_skb_core()` 不直接调用最后一个匹配的 `packet_type->func`，而是通过输出参数 `pt_prev` 把它「攒」着返回给调用方，由调用方 `__netif_receive_skb_one_core()` 统一调用。这样除了最后一个 handler，前面的 handler 都在 `deliver_skb()` 里额外 `refcount_inc(&skb->users)`；最后一个不增引用直接调用，省掉一次原子操作。这是收包热路径上的经典优化。

下面按这条层次链自顶向下展开。

#### 2.6.1 `netif_receive_skb()`：单个 skb 入口

```c
int netif_receive_skb(struct sk_buff *skb)
{
	int ret;

	trace_netif_receive_skb_entry(skb);

	ret = netif_receive_skb_internal(skb);
	trace_netif_receive_skb_exit(ret);

	return ret;
}
```

只是包了 tracepoint，真正逻辑在 `netif_receive_skb_internal()`。

#### 2.6.2 `netif_receive_skb_internal()`：时间戳 + RPS

```c
static int netif_receive_skb_internal(struct sk_buff *skb)
{
	int ret;

	net_timestamp_check(READ_ONCE(net_hotdata.tstamp_prequeue), skb);

	if (skb_defer_rx_timestamp(skb))
		return NET_RX_SUCCESS;

	rcu_read_lock();
#ifdef CONFIG_RPS
	if (static_branch_unlikely(&rps_needed)) {
		struct rps_dev_flow voidflow, *rflow = &voidflow;
		int cpu = get_rps_cpu(skb->dev, skb, &rflow);

		if (cpu >= 0) {
			ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
			rcu_read_unlock();
			return ret;
		}
	}
#endif
	ret = __netif_receive_skb(skb);
	rcu_read_unlock();
	return ret;
}
```

- 先做接收时间戳的延迟预处理（`net_timestamp_check`）和 deferred timestamp 判定。
- 若开启 RPS 且 `get_rps_cpu()` 给出目标 CPU，则 `enqueue_to_backlog()` 把 skb 跨 CPU 转发到目标 CPU 的 `softnet_data->input_pkt_queue`，由目标 CPU 的 `NET_RX_SOFTIRQ` 接着处理（最终也会回到 `__netif_receive_skb_core`）。
- 否则就地调用 `__netif_receive_skb()`。

#### 2.6.3 `__netif_receive_skb()`：PFMEMALLOC 判定

```c
static int __netif_receive_skb(struct sk_buff *skb)
{
	int ret;

	if (sk_memalloc_socks() && skb_pfmemalloc(skb)) {
		unsigned int noreclaim_flag;

		/*
		 * PFMEMALLOC skbs are special, they should
		 * - be delivered to SOCK_MEMALLOC sockets only
		 * - stay away from userspace
		 * - have bounded memory usage
		 */
		noreclaim_flag = memalloc_noreclaim_save();
		ret = __netif_receive_skb_one_core(skb, true);
		memalloc_noreclaim_restore(noreclaim_flag);
	} else
		ret = __netif_receive_skb_one_core(skb, false);

	return ret;
}
```

这一层只做一件事：识别 `PFMEMALLOC` skb（内存紧张时从 reserve 分配出来的包），用 `memalloc_noreclaim_save()` 临时进入非回收模式再进入核心，保证这类 skb 只投递给 `SOCK_MEMALLOC` socket，不泄露给用户态。普通 skb 直接走 `pfmemalloc=false` 分支。

#### 2.6.4 `__netif_receive_skb_one_core()`：核心调度 + 末尾 handler 调用

```c
static int __netif_receive_skb_one_core(struct sk_buff *skb, bool pfmemalloc)
{
	struct net_device *orig_dev = skb->dev;
	struct packet_type *pt_prev = NULL;
	int ret;

	ret = __netif_receive_skb_core(&skb, pfmemalloc, &pt_prev);
	if (pt_prev)
		ret = INDIRECT_CALL_INET(pt_prev->func, ipv6_rcv, ip_rcv, skb,
					 skb->dev, pt_prev, orig_dev);
	return ret;
}
```

- 调用 `__netif_receive_skb_core()` 完成全部 L2 分发逻辑，通过 `pt_prev` 拿回「最后一个匹配但还没调用」的 handler。
- 若存在 `pt_prev`，用 `INDIRECT_CALL_INET` 直接调用——这是一个编译期优化的间接调用宏，对 IPv4/IPv6 这两个最常见 handler 做成直接调用，避免间接跳转的分支预测开销。
- 注意 `skb` 是 `struct sk_buff **` 传入：`__netif_receive_skb_core()` 可能替换 skb（如 rx_handler 把 skb 转成 bridge 的另一个 skb），所以调用方读的是更新后的 `skb`。

> 还有一个导出的 `netif_receive_skb_core()`（dev.c:6222），注释说明它「skip RPS and Generic XDP」，仅供需要绕过 RPS/Generic-XDP 的特殊调用方使用，不在常规收包路径上。

### 2.7 `__netif_receive_skb_core()`：二层分发核心

```c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
	struct packet_type *ptype, *pt_prev;
	rx_handler_func_t *rx_handler;
	struct sk_buff *skb = *pskb;
	struct net_device *orig_dev;
	bool deliver_exact = false;
	int ret = NET_RX_DROP;
	__be16 type;
	/* 注：以下省略 orig_dev 赋值、pfmemalloc 过滤、drop_reason 局部变量、
	 * skip_defrag / deliver_no_ptype 等细节，保留主干分发逻辑。 */

	skb_reset_network_header(skb);
	skb_reset_mac_len(skb);

	pt_prev = NULL;

another_round:
	skb->skb_iif = skb->dev->ifindex;
	__this_cpu_inc(softnet_data.processed);

	/* 1. 先投递给 ptype_all（tcpdump / AF_PACKET 等监听所有报文） */
	list_for_each_entry_rcu(ptype, &dev_net_rcu(skb->dev)->ptype_all, list) {
		if (unlikely(pt_prev))
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}

	list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list) {
		if (unlikely(pt_prev))
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}

	/* 2. TC ingress / netfilter */
#ifdef CONFIG_NET_INGRESS
	if (static_branch_unlikely(&ingress_needed_key)) {
		/* ... sch_handle_ingress() 处理 TC ingress，可能改 skb 并触发 another_round */
		skb = sch_handle_ingress(skb, &pt_prev, &ret, orig_dev, &another);
		if (another)
			goto another_round;
		if (!skb)
			goto out;
	}
#endif

	/* 3. VLAN 处理 */
	if (skb_vlan_tag_present(skb)) {
		if (unlikely(pt_prev)) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		if (vlan_do_receive(&skb))
			goto another_round;
	}

	/* 4. rx_handler：桥接、bonding、macvlan 等 */
	rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (unlikely(pt_prev)) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out;
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
			break;
		case RX_HANDLER_PASS:
			break;
		}
	}

	/* 5. 按 ethertype 分发给具体协议 */
	type = skb->protocol;
	if (likely(!deliver_exact)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &ptype_base[ntohs(type) & PTYPE_HASH_MASK]);
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &dev_net_rcu(skb->dev)->ptype_specific);
	}
	deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
			       &orig_dev->ptype_specific);

	if (pt_prev) {
		*ppt_prev = pt_prev;
	} else {
		kfree_skb_reason(skb, drop_reason);
		ret = NET_RX_DROP;
	}

out:
	*pskb = skb;
	return ret;
}
```

> 上面的代码片段为保留主干分发的简化版，省略了 `pfmemalloc` 过滤、`orig_dev`/`drop_reason` 局部变量的赋值、`deliver_no_ptype` 等分支，真实实现见 `net/core/dev.c:5972`。

`deliver_skb()` 会递增 skb 引用计数并调用 `ptype->func()`：

```c
static int deliver_skb(struct sk_buff *skb,
		       struct packet_type *pt_prev,
		       struct net_device *orig_dev)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
		return -ENOMEM;
	refcount_inc(&skb->users);
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

> `__netif_receive_skb_one_core()` 如何使用 `pt_prev` 调用最后一个 handler，见 §2.6.4，此处不再重复。

### 2.8 `napi_gro_receive()` 与 GRO

NAPI 驱动通常把 skb 交给 `napi_gro_receive()`，先尝试 GRO 合并，再决定是上送协议栈还是继续聚合：

```c
static inline gro_result_t napi_gro_receive(struct napi_struct *napi,
					    struct sk_buff *skb)
{
	return gro_receive_skb(&napi->gro, skb);
}
```

`gro_receive_skb()` 流程：

```text
napi_gro_receive(napi, skb)
  → gro_receive_skb(&napi->gro, skb)
    → dev_gro_receive(gro, skb)
      ├─ 找到匹配 flow 的 GRO 包 → GRO_MERGED / GRO_HELD（聚合）
      └─ 没有匹配 → GRO_NORMAL（直接上送）
    → gro_skb_finish()
      ├─ GRO_NORMAL → gro_normal_one() → 最终 netif_receive_skb()
      └─ GRO_MERGED_FREE → 释放 skb
```

GRO 能显著降低小包场景下的协议栈处理开销。

### 2.9 发送路径：`dev_queue_xmit()` → 驱动

```c
int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
{
	struct net_device *dev = skb->dev;
	struct netdev_queue *txq = NULL;
	struct Qdisc *q;

	skb_reset_mac_header(skb);
	skb_assert_len(skb);

	rcu_read_lock_bh();

	/* 选择发送队列 */
	txq = netdev_core_pick_tx(dev, skb, sb_dev);
	q = rcu_dereference_bh(txq->qdisc);

	trace_net_dev_queue(skb);
	if (q->enqueue) {
		/* 有 qdisc：入队，由 qdisc 择机发送 */
		rc = __dev_xmit_skb(skb, q, dev, txq);
		goto out;
	}

	/* 无 qdisc 的设备（如 loopback、隧道）直接发 */
	if (unlikely(!(dev->flags & IFF_UP))) {
		...
		goto drop;
	}

	HARD_TX_LOCK(dev, txq, cpu);
	if (!netif_xmit_stopped(txq)) {
		skb = dev_hard_start_xmit(skb, dev, txq, &rc);
	}
	HARD_TX_UNLOCK(dev, txq);
	...
}
```

发送流程概览：

```text
协议栈（如 ip_queue_xmit）
  → 构造 skb，设置 skb->dev
  → dev_queue_xmit(skb)
    → 选择 tx queue
    → 若设备有 qdisc：__dev_xmit_skb() 入队
      → qdisc 的 dequeue / ndo_start_xmit()
    → 若设备无 qdisc：直接 dev_hard_start_xmit()
      → dev->netdev_ops->ndo_start_xmit(skb, dev)
        → 驱动把 skb 写入 TX ring / DMA
        → 发送完成中断中释放 skb（consume_skb / napi_consume_skb）
```

### 2.10 net_device、NAPI 与 sk_buff 的关系

| 场景 | 谁持有/产生 skb | 关键函数 |
|------|----------------|----------|
| 驱动收包 | 驱动分配 `napi_alloc_skb()`，填充 DMA 数据 | `napi_gro_receive()` / `netif_receive_skb()` |
| 协议栈 RX | `__netif_receive_skb_core()` 按 ethertype 分发给协议 handler | `deliver_skb()` → `ip_rcv()` / `arp_rcv()` |
| 协议栈 TX | IP/TCP/UDP 构造 skb，设置 `skb->dev` | `dev_queue_xmit()` |
| 驱动发送 | `ndo_start_xmit()` 消费 skb | `dev_hard_start_xmit()` |
| 发送完成 | 驱动/网卡释放 skb | `consume_skb()` / `napi_consume_skb()` |

一句话：**net_device 是 sk_buff 跨越软硬件边界的“港口”，NAPI 是港口的高效卸货机制。**

---

## 3. 协议注册与分发表

前两章分别讲了 **Socket 端点抽象** 和 **net_device / NAPI 二层接入**。报文从网卡进入内核后，需要经过两次关键分发才能到达最终协议处理函数：

1. **L2 → L3 分发**：根据以太网类型（`ethertype`，如 `ETH_P_IP`、`ETH_P_ARP`）把二层帧交给 IP、ARP 等协议入口。
2. **L3 → L4 分发**：根据 IP 头中的 `protocol` 字段（如 `IPPROTO_TCP`、`IPPROTO_UDP`、`IPPROTO_ICMP`）把 IP 包交给 TCP、UDP、ICMP 等处理函数。

Linux 用**两张表**完成这两次分发：
- `ptype_base[]` / `ptype_specific` / `ptype_all`：由 `struct packet_type` 组成，负责 L2→L3。
- `inet_protos[]`：由 `struct net_protocol` 组成，负责 IPv4 的 L3→L4。

### 3.1 `struct packet_type`：L2 → L3 的分发表项

```c
struct packet_type {
	__be16			type;	/* htons(ether_type) */
	bool			ignore_outgoing;
	struct net_device	*dev;	/* NULL 表示通配 */
	netdevice_tracker	dev_tracker;
	int			(*func)(struct sk_buff *skb,
					 struct net_device *dev,
					 struct packet_type *pt,
					 struct net_device *orig_dev);
	void			(*list_func)(struct list_head *head,
					     struct packet_type *pt,
					     struct net_device *orig_dev);
	bool			(*id_match)(struct packet_type *ptype,
					    struct sock *sk);
	struct net		*af_packet_net;
	void			*af_packet_priv;
	struct list_head	list;
};
```

| 字段 | 含义 |
|------|------|
| `type` | 以太网协议类型，如 `ETH_P_IP`（0x0800）、`ETH_P_ARP`（0x0806）、`ETH_P_IPV6`（0x86DD） |
| `dev` | 绑定到具体设备。非空时挂 per-device `dev->ptype_specific`（`ETH_P_ALL` 时挂 `dev->ptype_all`）；为空时配合 `af_packet_net` 决定挂 per-net 链表或全局 `ptype_base[hash]`，详见 §2.3 的 `ptype_head()` |
| `af_packet_net` | AF_PACKET 用的 per-net 归属，`dev` 为空时据此挂 `net->ptype_all` / `net->ptype_specific` |
| `func` | 收到匹配帧时的处理函数，签名固定为 `int func(skb, dev, pt, orig_dev)` |
| `list_func` | 列表接收处理函数，用于 GRO list 路径 |
| `list` | 链表节点，挂到 `ptype_base[]` / `ptype_all` / `ptype_specific`（由 `ptype_head()` 决定） |

IPv4 与 ARP 的 `packet_type` 注册：

```c
/* net/ipv4/af_inet.c */
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
	.list_func = ip_list_rcv,
};

/* net/ipv4/arp.c */
static struct packet_type arp_packet_type __read_mostly = {
	.type =	cpu_to_be16(ETH_P_ARP),
	.func =	arp_rcv,
};
```

注册入口 `dev_add_pack()` 把 `packet_type` 挂到对应的链表：

```c
void dev_add_pack(struct packet_type *pt)
{
	struct list_head *head = ptype_head(pt);

	if (WARN_ON_ONCE(!head))
		return;

	spin_lock(&ptype_lock);
	list_add_rcu(&pt->list, head);
	spin_unlock(&ptype_lock);
}
```

`ptype_head()` 根据 `pt->type` 和 `pt->dev` 选择挂到全局 `ptype_base[hash]`、设备级 `ptype_specific` 或 `ptype_all`。

### 3.2 `__netif_receive_skb_core()` 中的 L2→L3 分发

当 `__netif_receive_skb_core()` 处理完 VLAN、TC、rx_handler 后，会执行以下分发：

```c
	type = skb->protocol;

	if (likely(!deliver_exact)) {
		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &ptype_base[ntohs(type) & PTYPE_HASH_MASK]);

		deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
				       &dev_net_rcu(skb->dev)->ptype_specific);
	}

	deliver_ptype_list_skb(skb, &pt_prev, orig_dev, type,
			       &orig_dev->ptype_specific);
```

`deliver_ptype_list_skb()` 遍历链表，匹配 `ptype->type == skb->protocol`，最终调用 `deliver_skb()`：

```c
static int deliver_skb(struct sk_buff *skb,
		       struct packet_type *pt_prev,
		       struct net_device *orig_dev)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
		return -ENOMEM;
	refcount_inc(&skb->users);
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

注意这里递增了 `skb->users`，因为同一个 skb 可能需要投递给多个 `packet_type`（例如 `ptype_all` 上的 tcpdump 和 `ptype_base` 上的 `ip_rcv`）。最后一个 handler 由 `__netif_receive_skb_one_core()` 直接调用：

```c
static int __netif_receive_skb_one_core(struct sk_buff *skb, bool pfmemalloc)
{
	struct net_device *orig_dev = skb->dev;
	struct packet_type *pt_prev = NULL;
	int ret;

	ret = __netif_receive_skb_core(&skb, pfmemalloc, &pt_prev);
	if (pt_prev)
		ret = INDIRECT_CALL_INET(pt_prev->func, ipv6_rcv, ip_rcv, skb,
					 skb->dev, pt_prev, orig_dev);
	return ret;
}
```

### 3.3 `struct net_protocol`：IPv4 的 L3 → L4 分发表项

IPv4 报文被 `ip_rcv()` 接收后，经过路由、分片重组，最终到达 `ip_local_deliver()`，再由 IP 头中的 `protocol` 字段决定交给谁。

```c
/* include/net/protocol.h */
struct net_protocol {
	int		(*handler)(struct sk_buff *skb);
	int		(*err_handler)(struct sk_buff *skb, u32 info);

	unsigned int	no_policy:1,
			icmp_strict_tag_validation:1;
	u32		secret;
};
```

| 字段 | 含义 |
|------|------|
| `handler` | 协议主处理函数，如 `tcp_v4_rcv()`、`udp_rcv()`、`icmp_rcv()` |
| `err_handler` | ICMP 错误处理函数，如 `tcp_v4_err()`、`udp_err()` |
| `no_policy` | 是否跳过 XFRM IPsec 策略检查 |
| `icmp_strict_tag_validation` | 是否对 ICMP 做更严格的 tag 校验 |

IPv4 核心协议初始化时注册到 `inet_protos[]`：

```c
/* net/ipv4/af_inet.c */
static const struct net_protocol icmp_protocol = {
	.handler =	icmp_rcv,
	.err_handler =	icmp_err,
	.no_policy =	1,
};

static int __init inet_init(void)
{
	...
	if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
		pr_crit("%s: Cannot add ICMP protocol\n", __func__);

	net_hotdata.udp_protocol = (struct net_protocol) {
		.handler =	udp_rcv,
		.err_handler =	udp_err,
		.no_policy =	1,
	};
	if (inet_add_protocol(&net_hotdata.udp_protocol, IPPROTO_UDP) < 0)
		pr_crit("%s: Cannot add UDP protocol\n", __func__);

	net_hotdata.tcp_protocol = (struct net_protocol) {
		.handler	=	tcp_v4_rcv,
		.err_handler	=	tcp_v4_err,
		.no_policy	=	1,
		.icmp_strict_tag_validation = 1,
	};
	if (inet_add_protocol(&net_hotdata.tcp_protocol, IPPROTO_TCP) < 0)
		pr_crit("%s: Cannot add TCP protocol\n", __func__);
#ifdef CONFIG_IP_MULTICAST
	if (inet_add_protocol(&igmp_protocol, IPPROTO_IGMP) < 0)
		pr_crit("%s: Cannot add IGMP protocol\n", __func__);
#endif
	...
}
```

### 3.4 `inet_add_protocol()` / `inet_protos[]`

`inet_protos[]` 是一个长度为 256 的指针数组，下标就是 IP 头中的 `protocol` 字段：

```c
/* net/ipv4/protocol.c */
struct net_protocol __rcu *inet_protos[MAX_INET_PROTOS] __read_mostly;

int inet_add_protocol(const struct net_protocol *prot, unsigned char protocol)
{
	return !cmpxchg((const struct net_protocol **)&inet_protos[protocol],
			NULL, prot) ? 0 : -1;
}

int inet_del_protocol(const struct net_protocol *prot, unsigned char protocol)
{
	int ret;

	ret = (cmpxchg((const struct net_protocol **)&inet_protos[protocol],
		       prot, NULL) == prot) ? 0 : -1;

	synchronize_net();

	return ret;
}
```

- `inet_add_protocol()` 用 `cmpxchg` 把 `inet_protos[protocol]` 从 NULL 设为 `prot`。
- `inet_del_protocol()` 删除时调用 `synchronize_net()` 等待所有 RCU 读者退出。

### 3.5 L3 → L4 分发流程：`ip_local_deliver()` → `ip_protocol_deliver_rcu()`

IP 包到达本机后，路由 `dst.input` 指向 `ip_local_deliver()`：

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
```

先进行 IP 分片重组，再经过 Netfilter `NF_INET_LOCAL_IN` 链，最后调用 `ip_local_deliver_finish()`：

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

`__skb_pull()` 剥掉 IP 头，然后 `ip_protocol_deliver_rcu()` 根据 `ip_hdr(skb)->protocol` 查表：

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
				kfree_skb_reason(skb, SKB_DROP_REASON_XFRM_POLICY);
				return;
			}
			nf_reset_ct(skb);
		}
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
		} else {
			__IP_INC_STATS(net, IPSTATS_MIB_INDELIVERS);
			consume_skb(skb);
		}
	}
}
```

分发逻辑：

1. 先尝试 `raw_local_deliver()`，把包发给匹配的 RAW socket。
2. 用 `rcu_dereference(inet_protos[protocol])` 取得 `struct net_protocol`。
3. 若存在，调用 `ipprot->handler(skb)`，通常是 `tcp_v4_rcv()`、`udp_rcv()`、`icmp_rcv()`。
4. 若 handler 返回负值 `-protocol`，表示需要把包重新投递到另一个协议（如 GRE/IPIP 解封装后重新进入 IP 栈）。
5. 若没有注册 handler 且没有 RAW socket，发送 `ICMP_DEST_UNREACH / ICMP_PROT_UNREACH` 并丢弃。

### 3.6 `inet_init()`：IPv4 协议栈的整体初始化

`inet_init()` 是 IPv4 子系统的入口，在 `net/ipv4/af_inet.c` 中由 `fs_initcall(inet_init)` 注册。它完成了 socket 协议、网络层协议、二层报文类型的全部注册：

```c
static int __init inet_init(void)
{
	struct inet_protosw *q;
	struct list_head *r;
	int rc;

	/* 1. 注册 socket 层协议实现（分配 kmem_cache）
	 *    真实源码每个调用后都有 if (rc) goto out;，此处省略错误处理 */
	rc = proto_register(&tcp_prot, 1);
	rc = proto_register(&udp_prot, 1);
	rc = proto_register(&raw_prot, 1);
	rc = proto_register(&ping_prot, 1);

	/* 2. 向 socket 子系统注册地址族 PF_INET */
	(void)sock_register(&inet_family_ops);

	/* 3. 注册 L3 → L4 协议处理函数 */
	if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
		pr_crit("%s: Cannot add ICMP protocol\n", __func__);

	net_hotdata.udp_protocol = (struct net_protocol) {
		.handler =	udp_rcv,
		.err_handler =	udp_err,
		.no_policy =	1,
	};
	if (inet_add_protocol(&net_hotdata.udp_protocol, IPPROTO_UDP) < 0)
		pr_crit("%s: Cannot add UDP protocol\n", __func__);

	net_hotdata.tcp_protocol = (struct net_protocol) {
		.handler	=	tcp_v4_rcv,
		.err_handler	=	tcp_v4_err,
		.no_policy	=	1,
		.icmp_strict_tag_validation = 1,
	};
	if (inet_add_protocol(&net_hotdata.tcp_protocol, IPPROTO_TCP) < 0)
		pr_crit("%s: Cannot add TCP protocol\n", __func__);

#ifdef CONFIG_IP_MULTICAST
	if (inet_add_protocol(&igmp_protocol, IPPROTO_IGMP) < 0)
		pr_crit("%s: Cannot add IGMP protocol\n", __func__);
#endif

	/* 4. 初始化 inetsw[]，供 inet_create() 查找 socket 类型/协议对应关系 */
	for (r = &inetsw[0]; r < &inetsw[SOCK_MAX]; ++r)
		INIT_LIST_HEAD(r);

	for (q = inetsw_array; q < &inetsw_array[INETSW_ARRAY_LEN]; ++q)
		inet_register_protosw(q);

	/* 5. 初始化 ARP、IP、TCP、UDP、ICMP 等子系统 */
	arp_init();
	ip_init();
	...
	tcp_init();
	udp_init();
	raw_init();
	ping_init();
	icmp_init();
	...

	/* 6. 注册 L2 → L3 报文类型处理函数 */
	dev_add_pack(&ip_packet_type);

	...
}
```

### 3.7 两张表的关系：socket 层 `inetsw` vs 网络层 `inet_protos`

容易混淆的两张表：

| 表 | 结构 | 索引 | 作用 | 使用场景 |
|----|------|------|------|----------|
| `inetsw[]` | `struct inet_protosw` | `socket type`（SOCK_STREAM/DGRAM/RAW） | socket 创建时匹配 `type + protocol` → `proto_ops` + `proto` | `inet_create()` |
| `inet_protos[]` | `struct net_protocol` | IP 头 `protocol` 字段 | IP 包接收时匹配 protocol → handler | `ip_protocol_deliver_rcu()` |

- `inetsw` 解决的是 **"用户要创建什么 socket"**：`socket(AF_INET, SOCK_STREAM, 0)` 对应 TCP。
- `inet_protos` 解决的是 **"收到的 IP 包该交给哪个 L4 协议"**：IP 头 protocol=6 交给 TCP。

### 3.8 完整接收分发路径

```text
网卡收到帧
  → 驱动构造 skb，eth_type_trans() 设置 skb->protocol
  → napi_gro_receive() / netif_receive_skb()
    → __netif_receive_skb_core()
      → ptype_all 投递（tcpdump / AF_PACKET）
      → TC ingress / netfilter
      → VLAN 处理
      → rx_handler（bridge / bonding / macvlan）
      → deliver_ptype_list_skb() 按 ethertype 分发
        ├─ ETH_P_IP  → ip_rcv()
        ├─ ETH_P_ARP → arp_rcv()
        └─ ETH_P_IPV6→ ipv6_rcv()

ip_rcv()
  → ip_rcv_core() 基础校验
  → NF_INET_PRE_ROUTING
  → ip_rcv_finish() 路由
    → dst_input(skb)
      ├─ 转发：ip_forward()
      └─ 本机：ip_local_deliver()
        → IP 分片重组
        → NF_INET_LOCAL_IN
        → ip_local_deliver_finish()
          → __skb_pull(IP 头)
          → ip_protocol_deliver_rcu(ip_hdr(skb)->protocol)
            → inet_protos[protocol]->handler(skb)
              ├─ IPPROTO_TCP → tcp_v4_rcv()
              ├─ IPPROTO_UDP → udp_rcv()
              └─ IPPROTO_ICMP→ icmp_rcv()
```

### 3.9 扩展协议：如何注册新的 L3/L4 协议

- **新增 L2→L3 协议**：定义 `struct packet_type`，填 `type` 和 `func`，调用 `dev_add_pack()`。
- **新增 IPv4 L4 协议**：定义 `struct net_protocol`，填 `handler`，调用 `inet_add_protocol()`。
- **新增 socket 类型/协议组合**：定义 `struct inet_protosw`，加入 `inetsw_array[]` 或在模块中调用 `inet_register_protosw()`。

例如 IPIP 隧道在 `net/ipv4/ipip.c` 中同时注册了 `net_protocol`（protocol = IPPROTO_IPIP）和 `packet_type`（可选，用于接收 GRE/MPLS 封装包）。

---

## 4. sk_buff 是什么

**`struct sk_buff`**（常简称为 **skb**）是 Linux 内核网络栈中**最重要的数据结构**，它代表一个网络数据包。所有网络层（L2 ~ L4）之间的报文传递，都通过 sk_buff 完成。

一句话概括：**skb 是网络报文在内核中的“容器”**，它既保存报文数据，也保存协议栈处理过程中需要的各种元数据。skb 在收发路径中的典型流转见 §13。

---

## 5. 核心数据结构：`struct sk_buff`

定义位于 `include/linux/skbuff.h:886`。

```c
struct sk_buff {
	union {
		struct {
			/* These two members must be first to match sk_buff_head. */
			struct sk_buff		*next;
			struct sk_buff		*prev;

			union {
				struct net_device	*dev;
				unsigned long		dev_scratch;
			};
		};
		struct rb_node		rbnode;
		struct list_head	list;
		struct llist_node	ll_node;
	};

	struct sock		*sk;

	union {
		ktime_t		tstamp;
		u64		skb_mstamp_ns;
	};

	/* 控制缓冲区，各层自由使用 */
	char			cb[48] __aligned(8);

	union {
		struct {
			unsigned long	_skb_refdst;
			void		(*destructor)(struct sk_buff *skb);
		};
		struct list_head	tcp_tsorted_anchor;
	};

	unsigned int		len,
				data_len;
	__u16			mac_len,
				hdr_len;

	__u8			cloned:1,
				nohdr:1,
				fclone:2,
				peeked:1,
				head_frag:1,
				pfmemalloc:1,
				pp_recycle:1;

	/* 报文类型、校验和、VLAN、timestamp 等大量标志位 ... */

	__be16			protocol;
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;

	/* 这些字段必须在结构体末尾，详见 alloc_skb() */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,
				*data;
	unsigned int		truesize;
	refcount_t		users;
};
```

### 5.1 关键字段速查

| 字段 | 含义 |
|------|------|
| `next` / `prev` | 队列指针，挂到 `sk_buff_head` 或 frag_list |
| `dev` | 关联的网络设备 |
| `sk` | 关联的 socket，用于内存记账和 destructor |
| `cb[48]` | 控制缓冲区，各协议层自由使用（如 TCP 保存控制信息） |
| `len` | 报文总长度（head 数据 + 分片数据） |
| `data_len` | 非线性数据长度（即分片/frags 中的数据量） |
| `mac_len` | MAC 头长度 |
| `hdr_len` | 可写头部长度（用于 headerless / nohdr skb） |
| `cloned` | 该 skb 是否为 clone 产物 |
| `nohdr` | 表示 head 数据不可写，只引用 payload |
| `head` / `data` / `tail` / `end` | 数据缓冲区的四个边界指针 |
| `truesize` | skb 实际占用的总内存（含头、分片、元数据） |
| `users` | 引用计数 |
| `protocol` | 三层协议类型，如 `ETH_P_IP` |
| `mac_header` / `network_header` / `transport_header` | 各层头部偏移 |

---

## 6. 内存布局：head、data、tail、end

skb 的数据区由一段线性缓冲区（`skb->head` 指向）和可选的**分页分片**组成。

```text
低地址
├─ headroom ─┤──────── data ────────┤─ tailroom ─┤ skb_shared_info ┤
   ↑            ↑                      ↑            ↑
 skb->head   skb->data              skb->tail    skb->end

headroom：可用于 skb_push() 向前扩展数据（如加 MAC 头）
data：     当前有效数据起点
tail：     当前有效数据终点（线性区）
end：      线性区末尾，之后是 struct skb_shared_info
tailroom： 可用于 skb_put() 向后扩展数据
```

### 6.1 `skb_shared_info`：非线性数据与 GSO

`skb->end` 之后紧跟 `struct skb_shared_info`，它保存分片页、GSO 信息、时间戳等。

```c
struct skb_shared_info {
	__u8		flags;
	__u8		meta_len;
	__u8		nr_frags;
	__u8		tx_flags;
	unsigned short	gso_size;
	unsigned short	gso_segs;
	struct sk_buff	*frag_list;
	union {
		struct skb_shared_hwtstamps hwtstamps;
		struct xsk_tx_metadata_compl xsk_meta;
	};
	unsigned int	gso_type;
	u32		tskey;

	atomic_t	dataref;        /* 数据区引用计数 */

	union {
		struct {
			u32	xdp_frags_size;
			u32	xdp_frags_truesize;
		};
		void	*destructor_arg;
	};

	/* must be last field, see pskb_expand_head() */
	skb_frag_t	frags[MAX_SKB_FRAGS];
};
```

| 字段 | 含义 |
|------|------|
| `nr_frags` | 分页分片数组 `frags[]` 中有效项数 |
| `frags[]` | 每个元素指向一个 page + offset + len |
| `frag_list` | 子 skb 链表，用于 GRO/GSO 或 fraglist GSO |
| `gso_size` / `gso_segs` / `gso_type` | GSO/TSO 分片信息 |
| `dataref` | 数据区引用计数，clone 时共享 |
| `hwtstamps` | 硬件时间戳 |

### 6.2 线性区 vs 非线性区

```c
static inline int skb_is_nonlinear(const struct sk_buff *skb)
{
	return skb->data_len;
}

static inline unsigned int skb_headlen(const struct sk_buff *skb)
{
	return skb->len - skb->data_len;
}
```

- **`skb_headlen(skb)`**：线性区数据长度，即 `data` 到 `tail` 的距离。
- **`skb->data_len`**：非线性区数据总长度（所有 frags + frag_list）。
- **`skb->len`**：报文总长度。

---

## 7. sk_buff 的分配与释放

### 7.1 分配：`alloc_skb()` / `__alloc_skb()`

```c
struct sk_buff *__alloc_skb(unsigned int size, gfp_t gfp_mask,
			    int flags, int node)
{
	struct sk_buff *skb = NULL;
	struct kmem_cache *cache;
	u8 *data;

	if (sk_memalloc_socks() && (flags & SKB_ALLOC_RX))
		gfp_mask |= __GFP_MEMALLOC;

	if (flags & SKB_ALLOC_FCLONE) {
		cache = net_hotdata.skbuff_fclone_cache;
		goto fallback;
	}
	cache = net_hotdata.skbuff_cache;
	...
	if (flags & SKB_ALLOC_NAPI) {
		skb = napi_skb_cache_get(true);
		...
	}

	if (!skb) {
fallback:
		skb = kmem_cache_alloc_node(cache, gfp_mask & ~GFP_DMA, node);
		if (unlikely(!skb))
			return NULL;
	}
	skbuff_clear(skb);

	data = kmalloc_reserve(&size, gfp_mask, node, skb);
	if (unlikely(!data))
		goto nodata;

	__finalize_skb_around(skb, data, size);

	if (flags & SKB_ALLOC_FCLONE) {
		struct sk_buff_fclones *fclones;

		fclones = container_of(skb, struct sk_buff_fclones, skb1);
		skb->fclone |= SKB_FCLONE_ORIG;
		refcount_set(&fclones->fclone_ref, 1);
	}

	return skb;
}
```

分配流程：

```text
__alloc_skb(size, gfp, flags, node)
    │
    ├─ 从 kmem_cache / NAPI percpu cache 分配 struct sk_buff 壳
    ├─ kmalloc_reserve() 分配数据缓冲区（含尾部 skb_shared_info 空间）
    ├─ __finalize_skb_around() 初始化 head/data/tail/end/users/shinfo
    │
    └─ 若 flags 含 SKB_ALLOC_FCLONE，再分配一个“兄弟”skb 用于快速 clone
```

### 7.2 `__finalize_skb_around()`：初始化数据区

```c
static inline void __finalize_skb_around(struct sk_buff *skb, void *data,
					 unsigned int size)
{
	struct skb_shared_info *shinfo;

	size -= SKB_DATA_ALIGN(sizeof(struct skb_shared_info));

	skb->truesize = SKB_TRUESIZE(size);
	refcount_set(&skb->users, 1);
	skb->head = data;
	skb->data = data;
	skb_reset_tail_pointer(skb);
	skb_set_end_offset(skb, size);
	skb->mac_header = (typeof(skb->mac_header))~0U;
	skb->transport_header = (typeof(skb->transport_header))~0U;
	skb->alloc_cpu = raw_smp_processor_id();

	shinfo = skb_shinfo(skb);
	memset(shinfo, 0, offsetof(struct skb_shared_info, dataref));
	atomic_set(&shinfo->dataref, 1);
}
```

初始化后：`head == data == tail`，`end` 指向线性区末尾，`mac_header` 和 `transport_header` 被设为无效值 `~0U`。

### 7.3 接收路径常用：`__netdev_alloc_skb()` / `build_skb()`

网卡驱动通常先分配原始 DMA 缓冲区，收包完成后再用 `build_skb()` 套壳：

```c
struct sk_buff *__netdev_alloc_skb(struct net_device *dev, unsigned int len,
				   gfp_t gfp_mask)
{
	struct page_frag_cache *nc;
	struct sk_buff *skb;
	...
	len += NET_SKB_PAD;

	if (len <= SKB_WITH_OVERHEAD(SKB_SMALL_HEAD_CACHE_SIZE) ||
	    len > SKB_WITH_OVERHEAD(PAGE_SIZE) ||
	    (gfp_mask & (__GFP_DIRECT_RECLAIM | GFP_DMA))) {
		skb = __alloc_skb(len, gfp_mask, SKB_ALLOC_RX, NUMA_NO_NODE);
		...
	}

	/* 否则从 page fragment cache 分配 */
	data = page_frag_alloc(nc, len, gfp_mask);
	...
	skb = __build_skb(data, len);
	skb->head_frag = 1;

skb_success:
	skb_reserve(skb, NET_SKB_PAD);
	skb->dev = dev;
	...
}
```

`NET_SKB_PAD` 默认至少 32 字节，为内核提前预留 headroom，减少后续重新分配。

### 7.4 释放：`kfree_skb()` vs `consume_skb()`

两者最终都走 `__kfree_skb()`，差别只在调用前的 tracepoint 和 drop reason。先看 `__kfree_skb()`（`net/core/skbuff.c:1201`），它假定调用方已确认 `users` 归零，直接释放：

```c
void __kfree_skb(struct sk_buff *skb)
{
	skb_release_all(skb, SKB_DROP_REASON_NOT_SPECIFIED);
	kfree_skbmem(skb);
}
```

`consume_skb()`（`net/core/skbuff.c:1430`）做「减引用 + 归零才释放」：

```c
void consume_skb(struct sk_buff *skb)
{
	if (!skb_unref(skb))
		return;

	trace_consume_skb(skb, __builtin_return_address(0));
	__kfree_skb(skb);
}
```

`kfree_skb()` 在 v7.1-rc7 是 inline，经 `kfree_skb_reason()` 归一到 `sk_skb_reason_drop()`（`net/core/skbuff.c:1208`）：

```c
static inline void kfree_skb(struct sk_buff *skb)
{
	kfree_skb_reason(skb, SKB_DROP_REASON_NOT_SPECIFIED);
}

static inline void
kfree_skb_reason(struct sk_buff *skb, enum skb_drop_reason reason)
{
	sk_skb_reason_drop(NULL, skb, reason);
}

static __always_inline
bool __sk_skb_reason_drop(struct sock *sk, struct sk_buff *skb,
			  enum skb_drop_reason reason)
{
	if (unlikely(!skb_unref(skb)))
		return false;
	...
	if (reason == SKB_CONSUMED)
		trace_consume_skb(skb, __builtin_return_address(0));
	else
		trace_kfree_skb(skb, __builtin_return_address(0), reason, sk);
	return true;
}
```

关键点：`skb_unref()` 先把 `users` 减 1，**若 `users` 未归零（说明还有其它持有者）就直接返回 false，既不释放也不触发 tracepoint**。只有归零才会触发 tracepoint 并真正释放。所以「触发 tracepoint」的前提是引用计数归零。

| 函数 | 用途 | tracepoint | 触发条件 |
|------|------|------------|----------|
| `kfree_skb()` | 因错误/丢包释放，reason=`NOT_SPECIFIED` | `trace_kfree_skb` | `users` 归零时 |
| `kfree_skb_reason()` | 带 drop reason 的释放（如 `SKB_DROP_REASON_*`） | `trace_kfree_skb`（reason==`SKB_CONSUMED` 时为 `trace_consume_skb`） | `users` 归零时 |
| `consume_skb()` | 正常消费释放（如成功发送后），内部 reason=`SKB_CONSUMED` | `trace_consume_skb` | `users` 归零时 |
| `__kfree_skb()` | 内部函数，**不做** `skb_unref`，直接释放 | 无 | 调用方须保证 `users` 已归零 |

**共同行为**：`kfree_skb` / `kfree_skb_reason` / `consume_skb` 都先 `skb_unref()` 减引用计数，仅当归零才真正释放；`users > 1` 时只减计数即返回，不释放、不 trace。这样同一个 skb 被 clone/共享时，最后一次 unref 才触发释放。

释放流程：

```text
kfree_skb / consume_skb
    → skb_unref() 减 users 计数
    → 若 users != 0，直接返回（不释放、不 trace）
    → 触发 tracepoint（kfree_skb / consume_skb）
    → __kfree_skb()
        → skb_release_all()
            → skb_release_head_state()   释放 dst、调用 destructor
            → skb_release_data()         释放/递减数据区引用
        → kfree_skbmem()                 释放 struct sk_buff 壳
```

### 7.5 `skb_release_data()`：释放共享数据

```c
static void skb_release_data(struct sk_buff *skb, enum skb_drop_reason reason)
{
	struct skb_shared_info *shinfo = skb_shinfo(skb);
	int i;

	if (!skb_data_unref(skb, shinfo))
		goto exit;

	if (skb_zcopy(skb)) {
		...
		skb_zcopy_clear(skb, true);
		...
	}

	for (i = 0; i < shinfo->nr_frags; i++)
		__skb_frag_unref(&shinfo->frags[i], skb->pp_recycle);

	if (shinfo->frag_list)
		kfree_skb_list_reason(shinfo->frag_list, reason);

	skb_free_head(skb);
exit:
	skb->pp_recycle = 0;
}
```

通过 `dataref` 实现引用计数：
- `dataref == 1`：自己是最后持有者，真正释放数据区和 frags。
- `dataref > 1`：仅递减计数，数据仍被其他 clone 共享。

---

## 8. 数据区操作：push / pull / put / trim / reserve

### 8.1 预留 headroom：`skb_reserve()`

```c
static inline void skb_reserve(struct sk_buff *skb, int len)
{
	skb->data += len;
	skb->tail += len;
}
```

分配空 skb 后，先 `skb_reserve(headroom)` 把 `data` 和 `tail` 同时后移，腾出前端空间供后续 `skb_push()` 使用。

### 8.2 向后扩展数据：`skb_put()`

```c
void *skb_put(struct sk_buff *skb, unsigned int len)
{
	void *tmp = skb_tail_pointer(skb);
	SKB_LINEAR_ASSERT(skb);
	skb->tail += len;
	skb->len  += len;
	if (unlikely(skb->tail > skb->end))
		skb_over_panic(skb, len, __builtin_return_address(0));
	return tmp;
}
```

在尾部增加 `len` 字节，返回旧 tail 指针（即新增数据的起始地址）。如果超出 `end` 会 panic。

### 8.3 向前扩展数据：`skb_push()`

```c
void *skb_push(struct sk_buff *skb, unsigned int len)
{
	skb->data -= len;
	skb->len  += len;
	if (unlikely(skb->data < skb->head))
		skb_under_panic(skb, len, __builtin_return_address(0));
	return skb->data;
}
```

在头部增加 `len` 字节，返回新的 data 指针。常用于协议栈向下层封装时添加头部。

### 8.4 从头部移除数据：`skb_pull()` / `__skb_pull()`

```c
static __always_inline void *__skb_pull(struct sk_buff *skb, unsigned int len)
{
	skb->len -= len;
	if (unlikely(skb->len < skb->data_len))
		BUG();
	return skb->data += len;
}

void *skb_pull(struct sk_buff *skb, unsigned int len)
{
	return skb_pull_inline(skb, len);
}
```

`skb_pull()` 从头部“吃掉”`len` 字节，data 前移，len 减小。常用于协议栈向上层递交时剥掉已处理头部。

### 8.5 截断尾部：`skb_trim()`

```c
void skb_trim(struct sk_buff *skb, unsigned int len)
{
	if (skb->len > len)
		__skb_trim(skb, len);
}
```

把报文截断到指定长度。

### 8.6 确保线性头可读：`pskb_may_pull()`

```c
static __always_inline enum skb_drop_reason
pskb_may_pull_reason(struct sk_buff *skb, unsigned int len)
{
	if (likely(len <= skb_headlen(skb)))
		return SKB_NOT_DROPPED_YET;

	if (unlikely(len > skb->len))
		return SKB_DROP_REASON_PKT_TOO_SMALL;

	if (unlikely(!__pskb_pull_tail(skb, len - skb_headlen(skb))))
		return SKB_DROP_REASON_NOMEM;

	return SKB_NOT_DROPPED_YET;
}
```

协议栈处理头部前常用它保证 `skb->data` 起始的 `len` 字节都在线性区；如果不够，会把分页数据拉取到线性头部。

---

## 9. Clone、Copy 与可写性

### 9.1 `skb_clone()`：浅拷贝，共享数据

```c
struct sk_buff *skb_clone(struct sk_buff *skb, gfp_t gfp_mask)
{
	struct sk_buff_fclones *fclones = container_of(skb,
						       struct sk_buff_fclones,
						       skb1);
	struct sk_buff *n;

	if (skb_orphan_frags(skb, gfp_mask))
		return NULL;

	if (skb->fclone == SKB_FCLONE_ORIG &&
	    refcount_read(&fclones->fclone_ref) == 1) {
		n = &fclones->skb2;
		refcount_set(&fclones->fclone_ref, 2);
		n->fclone = SKB_FCLONE_CLONE;
	} else {
		n = kmem_cache_alloc(net_hotdata.skbuff_cache, gfp_mask);
		...
		n->fclone = SKB_FCLONE_UNAVAILABLE;
	}

	return __skb_clone(n, skb);
}
```

`skb_clone()` 只复制 `struct sk_buff` 壳，**共享数据区**。它递增 `skb_shared_info->dataref`。

```c
static struct sk_buff *__skb_clone(struct sk_buff *n, struct sk_buff *skb)
{
	n->next = n->prev = NULL;
	n->sk = NULL;
	__copy_skb_header(n, skb);

	C(len); C(data_len); C(mac_len);
	n->hdr_len = skb->nohdr ? skb_headroom(skb) : skb->hdr_len;
	n->cloned = 1;
	n->nohdr = 0;
	n->destructor = NULL;
	C(tail); C(end); C(head); C(head_frag); C(data); C(truesize);
	refcount_set(&n->users, 1);

	atomic_inc(&(skb_shinfo(skb)->dataref));
	skb->cloned = 1;

	return n;
}
```

### 9.2 `skb_copy()`：深拷贝，完全私有

```c
struct sk_buff *skb_copy(const struct sk_buff *skb, gfp_t gfp_mask)
{
	struct sk_buff *n;
	unsigned int size;
	int headerlen;

	headerlen = skb_headroom(skb);
	size = skb_end_offset(skb) + skb->data_len;
	n = __alloc_skb(size, gfp_mask, skb_alloc_rx_flag(skb), NUMA_NO_NODE);
	if (!n)
		return NULL;

	skb_reserve(n, headerlen);
	skb_put(n, skb->len);

	BUG_ON(skb_copy_bits(skb, -headerlen, n->head, headerlen + skb->len));

	skb_copy_header(n, skb);
	return n;
}
```

`skb_copy()` 复制整个数据区，返回一个完全可写的 skb。适合需要修改数据的场景，但开销较大。

### 9.3 `__pskb_copy_fclone()`：只拷贝头部，共享分片

```c
struct sk_buff *__pskb_copy_fclone(struct sk_buff *skb, int headroom,
				   gfp_t gfp_mask, bool fclone)
{
	unsigned int size = skb_headlen(skb) + headroom;
	int flags = skb_alloc_rx_flag(skb) | (fclone ? SKB_ALLOC_FCLONE : 0);
	struct sk_buff *n = __alloc_skb(size, gfp_mask, flags, NUMA_NO_NODE);

	if (!n)
		goto out;

	skb_reserve(n, headroom);
	skb_put(n, skb_headlen(skb));
	skb_copy_from_linear_data(skb, n->data, n->len);

	n->truesize += skb->data_len;
	n->data_len  = skb->data_len;
	n->len       = skb->len;

	if (skb_shinfo(skb)->nr_frags) {
		int i;

		if (skb_orphan_frags(skb, gfp_mask) ||
		    skb_zerocopy_clone(n, skb, gfp_mask)) {
			kfree_skb(n);
			n = NULL;
			goto out;
		}
		for (i = 0; i < skb_shinfo(skb)->nr_frags; i++) {
			skb_shinfo(n)->frags[i] = skb_shinfo(skb)->frags[i];
			skb_frag_ref(skb, i);
		}
		skb_shinfo(n)->nr_frags = i;
		skb_shinfo(n)->flags |= skb_shinfo(skb)->flags & SKBFL_SHARED_FRAG;
	}

	if (skb_has_frag_list(skb)) {
		skb_shinfo(n)->frag_list = skb_shinfo(skb)->frag_list;
		skb_clone_fraglist(n);
	}

	skb_copy_header(n, skb);
out:
	return n;
}
```

只拷贝线性头部，分页分片通过引用共享。适合只需要修改头部的场景（如 NAT、封装）。

### 9.4 `pskb_expand_head()`：扩大 headroom

```c
int pskb_expand_head(struct sk_buff *skb, int nhead, int ntail,
		     gfp_t gfp_mask)
{
	unsigned int osize = skb_end_offset(skb);
	unsigned int size = osize + nhead + ntail;
	long off;
	u8 *data;
	...
	BUG_ON(skb_shared(skb));

	data = kmalloc_reserve(&size, gfp_mask, NUMA_NO_NODE, NULL);
	if (!data)
		goto nodata;
	size = SKB_WITH_OVERHEAD(size);

	memcpy(data + nhead, skb->head, skb_tail_pointer(skb) - skb->head);
	memcpy((struct skb_shared_info *)(data + size),
	       skb_shinfo(skb),
	       offsetof(struct skb_shared_info, frags[skb_shinfo(skb)->nr_frags]));
	...
	skb->head = data;
	skb->data += off;
	skb_set_end_offset(skb, size);
	skb->tail += off;
	skb_headers_offset_update(skb, nhead);
	skb->cloned = 0;
	skb->nohdr = 0;
	atomic_set(&skb_shinfo(skb)->dataref, 1);
	...
}
```

当 `skb_push()` 发现 headroom 不够时，调用 `pskb_expand_head()` 重新分配更大的缓冲区，复制数据，并修正各头部偏移。

### 9.5 可写性判断：`skb_shared()` / `skb_cloned()` / `skb_header_cloned()`

```c
static inline int skb_shared(const struct sk_buff *skb)
{
	return refcount_read(&skb->users) != 1;
}

static inline int skb_cloned(const struct sk_buff *skb)
{
	return skb->cloned &&
	       (atomic_read(&skb_shinfo(skb)->dataref) & SKB_DATAREF_MASK) != 1;
}

static inline int skb_header_cloned(const struct sk_buff *skb)
{
	int dataref;

	if (!skb->cloned)
		return 0;

	dataref = atomic_read(&skb_shinfo(skb)->dataref);
	dataref = (dataref & SKB_DATAREF_MASK) - (dataref >> SKB_DATAREF_SHIFT);
	return dataref != 1;
}
```

| 函数 | 含义 |
|------|------|
| `skb_shared()` | `users` 引用计数 > 1，不能释放/修改 skb 壳 |
| `skb_cloned()` | 数据区被多个 skb 共享，不能写数据 |
| `skb_header_cloned()` | 头部分被多个 skb 共享，不能写头部 |

### 9.6 引用计数与 `skb_share_check()`

```c
static inline struct sk_buff *skb_share_check(struct sk_buff *skb, gfp_t pri)
{
	might_sleep_if(gfpflags_allow_blocking(pri));
	if (skb_shared(skb)) {
		struct sk_buff *nskb = skb_clone(skb, pri);

		if (likely(nskb))
			consume_skb(skb);
		else
			kfree_skb(skb);
		skb = nskb;
	}
	return skb;
}
```

如果 skb 被共享，先 clone 一份再释放旧的。这是协议栈“写前复制”的常见入口。

### 9.7 `skb_orphan()`：解除与 socket 的关联

```c
static __always_inline void skb_orphan(struct sk_buff *skb)
{
	if (skb->destructor) {
		skb->destructor(skb);
		skb->destructor = NULL;
		skb->sk		= NULL;
	} else {
		BUG_ON(skb->sk);
	}
}
```

网卡驱动发送完成或转发路径常调用 `skb_orphan()`，让 skb 不再占用 socket 发送内存配额。

---

## 10. sk_buff 队列

### 10.1 `struct sk_buff_head`

```c
struct sk_buff_head {
	/* These two members must be first to match sk_buff. */
	struct_group_tagged(sk_buff_list, list,
		struct sk_buff	*next;
		struct sk_buff	*prev;
	);

	__u32		qlen;
	spinlock_t	lock;
};
```

队列头与 `struct sk_buff` 前两个字段兼容，因此 skb 可以直接挂入队列。

### 10.2 常用队列 API

```c
void skb_queue_head(struct sk_buff_head *list, struct sk_buff *newsk);
void skb_queue_tail(struct sk_buff_head *list, struct sk_buff *newsk);
struct sk_buff *skb_dequeue(struct sk_buff_head *list);
struct sk_buff *skb_dequeue_tail(struct sk_buff_head *list);
void skb_unlink(struct sk_buff *skb, struct sk_buff_head *list);
void skb_queue_purge(struct sk_buff_head *list);
```

```c
void skb_queue_purge_reason(struct sk_buff_head *list,
			    enum skb_drop_reason reason)
{
	struct sk_buff *skb;

	while ((skb = skb_dequeue(list)) != NULL)
		kfree_skb_reason(skb, reason);
}
```

### 10.3 锁版本与非锁版本

- `skb_queue_head()` / `skb_queue_tail()`：带锁。
- `__skb_queue_head()` / `__skb_queue_tail()`：不带锁，调用者已持有 `list->lock`。
- `skb_dequeue()` / `__skb_dequeue()`：同理。

---

## 11. 校验和卸载（Checksum Offload）

`sk_buff` 通过 `ip_summed`、`csum_start`、`csum_offset`、`csum` 等字段与网卡协商校验和卸载。

```c
#define CHECKSUM_NONE		0
#define CHECKSUM_UNNECESSARY	1
#define CHECKSUM_COMPLETE	2
#define CHECKSUM_PARTIAL	3
```

| 值 | 含义 |
|----|------|
| `CHECKSUM_NONE` | 没有校验和 offload，由协议栈软件计算 |
| `CHECKSUM_UNNECESSARY` | 硬件已验证，无需再算 |
| `CHECKSUM_COMPLETE` | 硬件给出整个报文校验和，存于 `skb->csum` |
| `CHECKSUM_PARTIAL` | 硬件只需计算 `csum_start` 到结尾，结果写入 `csum_offset` |

发送路径示例：

```text
TCP 构造 skb
  → 设置 ip_summed = CHECKSUM_PARTIAL
  → 设置 csum_start  = transport_header 相对 data 的偏移
  → 设置 csum_offset = TCP checksum 字段在 L4 头中的偏移
  → 网卡驱动 ndo_start_xmit()
    → 若支持 HW_CSUM，直接 DMA 发送
    → 若不支持，调用 skb_checksum_help() 软件补算
```

---

## 12. GSO / TSO / UFO

`skb_shared_info` 中的 GSO 字段让协议栈可以一次性构造一个“超级 skb”，由后续层（或网卡）分段。

```c
struct skb_shared_info {
	unsigned short	gso_size;    /* 每个分片的数据长度 */
	unsigned short	gso_segs;    /* 预计分成的段数 */
	unsigned int	gso_type;    /* SKB_GSO_TCPV4 / TCPV6 / UDP_L4 等 */
	...
};
```

常见 GSO 类型：

```c
#define SKB_GSO_TCPV4		0x01
#define SKB_GSO_UDP_L4		0x02
#define SKB_GSO_DODGY		0x04
#define SKB_GSO_TCP_ECN		0x08
#define SKB_GSO_TCPV6		0x80
#define SKB_GSO_FRAGLIST	0x100
#define SKB_GSO_TUNNEL_REMCSUM	0x04
```

分段入口 `skb_segment()` 位于 `net/core/skbuff.c:4768`，它会根据 `gso_size` 把一个 GSO skb 拆成多个普通 skb。

---

## 13. sk_buff 在收发路径中的典型流转

### 13.1 接收路径

```text
网卡中断 / NAPI
  → 驱动 alloc skb / build_skb
  → skb_reserve(NET_SKB_PAD)
  → DMA 完成，填充数据
  → napi_gro_receive() / netif_receive_skb()
    → __netif_receive_skb_core()
      → 根据 ethertype 分发到 packet_type handler
        → ip_rcv() / arp_rcv() / ipv6_rcv()
          → skb_pull(ethhdr_len)
          → pskb_may_pull(ip_header_len)
          → ip_rcv_finish() → dst_input()
            → tcp_v4_rcv() / udp_rcv()
              → 放入 socket receive queue
                → 用户态 recv()
```

### 13.2 发送路径

```text
用户态 send()
  → sock_sendmsg()
    → tcp_sendmsg() / udp_sendmsg()
      → alloc_skb(headroom + payload)
      → skb_reserve(headroom)
      → skb_put(payload) 并拷贝用户数据
      → 构造 L4 / L3 / L2 头（skb_push 添加头部）
      → ip_queue_xmit() / ip_local_out()
        → netfilter (NF_INET_LOCAL_OUT / POST_ROUTING)
        → dst_neigh_output()
          → dev_queue_xmit()
            → 网卡驱动 ndo_start_xmit()
              → DMA 发送
              → 发送完成中断中 consume_skb()
```

---

## 14. 常见陷阱与注意事项

1. **`skb_put()` / `skb_push()` 越界会 panic**  
   调用前必须确保 headroom / tailroom 足够，否则应先用 `pskb_expand_head()` 扩容。

2. **clone 后不要写共享数据**  
   `skb_clone()` 只拷贝壳，数据共享。写数据前先用 `skb_copy()` 或 `pskb_copy()`。

3. **`skb_pull()` 不会释放内存**  
   只是移动 `data` 指针并减小 `len`，数据仍在线性区中。要真正回收头部空间需重新分配或 `skb_condense()`。

4. **`pskb_may_pull()` 可能失败并返回 drop reason**  
   在需要读取连续头部的位置（如 IP 选项、TCP 选项）必须检查返回值。

5. **`skb->data` 与头部偏移字段的关系**  
   `mac_header`、`network_header`、`transport_header` 是相对于 `skb->head` 的偏移（或指针，取决于 `NET_SKBUFF_DATA_USES_OFFSET`）。`skb_push()` / `skb_pull()` 后要用 `skb_reset_*_header()` 或 `skb_set_*_header()` 更新。

6. **`kfree_skb()` 与 `consume_skb()` 的区别主要在 tracepoint**  
   功能相同，但错误路径用 `kfree_skb()`，正常路径用 `consume_skb()`。

---

## 15. 小结

| 主题 | 核心文件 | 关键函数/结构 |
|------|----------|---------------|
| BSD socket 抽象 | `include/linux/net.h` | `struct socket`、`struct proto_ops`、`struct net_proto_family` |
| 内核 socket 表示 | `include/net/sock.h` | `struct sock`、`struct sock_common`、`struct proto` |
| socket 创建 | `net/socket.c` / `net/ipv4/af_inet.c` | `__sock_create()`、`inet_create()`、`sock_init_data()` |
| IPv4 协议注册 | `net/ipv4/af_inet.c` | `inet_family_ops`、`inetsw_array[]`、`inet_register_protosw()` |
| skb 与 socket 内存绑定 | `net/core/sock.c` / `include/net/sock.h` | `skb_set_owner_w()`、`skb_set_owner_r()`、`sock_wfree()`、`sock_rfree()` |
| 网络设备 | `include/linux/netdevice.h` | `struct net_device`、`struct net_device_ops`、`struct header_ops` |
| 二层到三层分发 | `net/core/dev.c` / `include/linux/netdevice.h` | `struct packet_type`、`dev_add_pack()`、`deliver_skb()` |
| NAPI 轮询 | `include/linux/netdevice.h` / `net/core/dev.c` | `struct napi_struct`、`napi_schedule()`、`napi_complete_done()` |
| 接收总入口 | `net/core/dev.c` | `netif_receive_skb()`、`__netif_receive_skb_core()` |
| GRO 接收卸载 | `net/core/gro.c` / `include/linux/netdevice.h` | `napi_gro_receive()`、`gro_receive_skb()` |
| 发送总入口 | `net/core/dev.c` | `dev_queue_xmit()`、`__dev_queue_xmit()`、`ndo_start_xmit()` |
| L3 → L4 分发表 | `include/net/protocol.h` / `net/ipv4/protocol.c` | `struct net_protocol`、`inet_protos[]`、`inet_add_protocol()` |
| IPv4 协议初始化 | `net/ipv4/af_inet.c` | `inet_init()`、`icmp_protocol`、`tcp_protocol`、`udp_protocol` |
| IP 包本机投递 | `net/ipv4/ip_input.c` | `ip_local_deliver()`、`ip_local_deliver_finish()`、`ip_protocol_deliver_rcu()` |
| 结构定义 | `include/linux/skbuff.h` | `struct sk_buff`、`struct skb_shared_info` |
| 分配释放 | `net/core/skbuff.c` | `__alloc_skb()`、`build_skb()`、`kfree_skb()`、`consume_skb()` |
| 数据操作 | `include/linux/skbuff.h` / `net/core/skbuff.c` | `skb_reserve()`、`skb_put()`、`skb_push()`、`skb_pull()`、`skb_trim()` |
| 拷贝 | `net/core/skbuff.c` | `skb_clone()`、`skb_copy()`、`__pskb_copy_fclone()`、`pskb_expand_head()` |
| 队列 | `include/linux/skbuff.h` / `net/core/skbuff.c` | `struct sk_buff_head`、`skb_queue_tail()`、`skb_dequeue()` |
| 校验和 | `include/linux/skbuff.h` | `ip_summed`、`csum_start`、`csum_offset` |
| GSO | `net/core/skbuff.c` | `skb_segment()`、`struct skb_shared_info` GSO 字段 |

Linux 网络框架的核心可以概括为**四层抽象**：

1. **Socket 抽象层**：通过 `struct socket` / `struct sock` / `struct proto_ops` / `struct proto` 把用户态 API 与内核协议实现解耦，支持同一地址族下多种协议、多种 socket 类型。
2. **网络设备层**：`struct net_device` 与 NAPI 把硬件报文接收/发送和上层协议栈衔接起来，`netif_receive_skb()` 是数据从 L2 进入 L3 的总入口，`dev_queue_xmit()` 是数据从 L3 下放到 L2 的总入口。
3. **协议注册与分发表层**：`struct packet_type` + `ptype_base[]` 完成 L2→L3 分发，`struct net_protocol` + `inet_protos[]` 完成 IPv4 的 L3→L4 分发；`inet_init()` 在启动时统一注册所有 IPv4 协议。
4. **报文抽象层**：`struct sk_buff` 作为统一的报文描述符，在不同协议层之间高效传递数据，同时支持共享、clone、分页、offload 等高级特性。

理解 socket、net_device/NAPI、协议分发表与 sk_buff 四者的协作——尤其是 `skb_set_owner_w/r()` 如何将报文生命周期与 socket 内存配额绑定、`netif_receive_skb_core()` 如何按 ethertype 把报文分发给协议 handler、`ip_protocol_deliver_rcu()` 如何按 IP protocol 字段把包交给 TCP/UDP/ICMP、`dev_queue_xmit()` 如何通过 qdisc 与驱动 ndo_start_xmit 完成发送——是阅读 Linux 网络源码的关键起点。
