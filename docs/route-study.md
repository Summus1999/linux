# Linux IPv4 路由子系统详解（结合 Linux v7.1-rc7 源码）

>
> 重点文件：
> - `net/ipv4/route.c` —— 高层路由 API、dst cache、输入/输出路由构造
> - `net/ipv4/fib_trie.c` —— FIB 最长前缀匹配（LC-trie）
> - `net/ipv4/fib_semantics.c` —— 路由语义：`fib_info`、`fib_nh`、multipath
> - `net/ipv4/fib_rules.c` —— 策略路由（`ip rule`）
> - `net/ipv4/fib_frontend.c` —— netlink 路由配置接口
> - `include/net/route.h`、`include/net/ip_fib.h`、`include/net/flow.h` —— 核心结构定义
>
> 约定：本文中的 `c` 代码块只引用内核原始源码片段；教学性的调用链、数据关系图使用 `text` 代码块表示。

---

## 1. 路由子系统要解决的问题

IPv4 协议栈在收到或发送一个报文时，都要回答几个核心问题：

1. 这个目的 IP 是不是本机？ 是则上交本机协议栈（`RTN_LOCAL`）。
2. 是不是广播/组播？ 广播要泛发到链路；组播要交给本机 IGMP/MC 层或按组播路由转发。
3. 需要转发吗？ 如果需要，应该从哪个网卡出去、下一跳 MAC 是什么、TTL 怎么处理？
4. 源地址选哪个？ 多地址接口上，内核要根据目的地址/出接口推断最合适的源地址。

Linux 把这些判断统一放在路由子系统里。它由两层组成：

- FIB（Forwarding Information Base）：真正的路由表，按目的前缀做最长匹配，存于 `fib_trie`。
- 路由缓存/结果对象：每次查找成功会生成一个 `struct rtable`，内嵌 `struct dst_entry`，供协议栈后续直接复用（输出缓存、输入缓存、例外缓存等）。

> 注意：老版本 Linux 有一个全局的 "route cache"，现代内核已经没有全局路由 cache，改为按下一跳分布的 dst cache。

---

## 2. 核心数据结构

### 2.1 `struct flowi4` —— 路由查找的"查询键"

文件：`include/net/flow.h:68-94`

```c
struct flowi4 {
    struct flowi_common __fl_common;
#define flowi4_oif      __fl_common.flowic_oif
#define flowi4_iif      __fl_common.flowic_iif
#define flowi4_mark     __fl_common.flowic_mark
#define flowi4_dscp     __fl_common.flowic_dscp
#define flowi4_scope    __fl_common.flowic_scope
#define flowi4_proto    __fl_common.flowic_proto
#define flowi4_flags    __fl_common.flowic_flags
    __be32          saddr;
    __be32          daddr;
    union flowi_uli uli;  /* sport/dport, icmp type/code 等 */
} __attribute__((__aligned__(BITS_PER_LONG/8)));
```

`flowi4` 是一次路由查询的输入参数集合，主要字段含义：

| 字段 | 含义 |
|------|------|
| `daddr` | 目的 IPv4 地址 |
| `saddr` | 源 IPv4 地址（可空，由路由结果推断） |
| `flowi4_oif` | 期望的出接口索引（可空） |
| `flowi4_iif` | 入接口索引（输入路由用） |
| `flowi4_mark` | skb 的 fwmark，策略路由常用 |
| `flowi4_dscp` | DSCP（由 TOS 转换） |
| `flowi4_scope` | 路由范围： `RT_SCOPE_UNIVERSE`、`RT_SCOPE_LINK`、`RT_SCOPE_HOST` 等 |
| `flowi4_proto` | 四层协议（TCP/UDP/ICMP...） |
| `fl4_sport / fl4_dport` | 四层端口（multipath hash 用） |

初始化输出查询键的辅助函数：

```c
static inline void flowi4_init_output(struct flowi4 *fl4, int oif,
                      __u32 mark, __u8 tos, __u8 scope,
                      __u8 proto, __u8 flags,
                      __be32 daddr, __be32 saddr,
                      __be16 dport, __be16 sport,
                      kuid_t uid)
{
    fl4->flowi4_oif = oif;
    fl4->flowi4_iif = LOOPBACK_IFINDEX;
    fl4->flowi4_mark = mark;
    fl4->flowi4_dscp = inet_dsfield_to_dscp(tos);
    fl4->flowi4_scope = scope;
    fl4->flowi4_proto = proto;
    fl4->flowi4_flags = flags;
    fl4->daddr = daddr;
    fl4->saddr = saddr;
    fl4->fl4_dport = dport;
    fl4->fl4_sport = sport;
    fl4->flowi4_multipath_hash = 0;
}
```

### 2.2 `struct rtable` —— IPv4 路由结果对象

文件：`include/net/route.h:57-78`

```c
struct rtable {
    struct dst_entry    dst;

    int         rt_genid;
    unsigned int        rt_flags;
    __u16           rt_type;
    __u8            rt_is_input;
    __u8            rt_uses_gateway;

    int         rt_iif;

    u8          rt_gw_family;
    /* Info on neighbour */
    union {
        __be32      rt_gw4;
        struct in6_addr rt_gw6;
    };

    /* Miscellaneous cached information */
    u32         rt_mtu_locked:1,
                rt_pmtu:31;
};
```

`struct rtable` 是 IPv4 专用的路由结果对象，内嵌通用 `struct dst_entry`（第一个字段），因此可以通过 `dst_rtable()` 宏从 `dst_entry` 反推：

```c
#define dst_rtable(_ptr) container_of_const(_ptr, struct rtable, dst)
```

关键字段：

| 字段 | 含义 |
|------|------|
| `dst` | 通用目的缓存，包含 `dev`、`input`、`output`、`expires`、`metrics` 等 |
| `rt_type` | 路由类型：`RTN_UNICAST`、`RTN_LOCAL`、`RTN_BROADCAST`、`RTN_MULTICAST` 等 |
| `rt_is_input` | 是否为输入路由 |
| `rt_uses_gateway` | 是否需要经过网关 |
| `rt_gw4` / `rt_gw6` | 下一跳网关地址 |
| `rt_iif` | 原始入接口 |
| `rt_pmtu` / `rt_mtu_locked` | PMTU 信息 |

`rtable.dst` 由 `ipv4_dst_ops` 管理，生命周期遵循 `dst_entry` 的引用计数规则。

### 2.3 FIB 核心结构

#### `struct fib_info` —— 一条路由的"下一跳集合"

文件：`include/net/ip_fib.h:136-163`

```c
struct fib_info {
    struct hlist_node   fib_hash;
    struct hlist_node   fib_lhash;
    struct list_head    nh_list;
    struct net      *fib_net;
    refcount_t      fib_treeref;
    refcount_t      fib_clntref;
    unsigned int        fib_flags;
    unsigned char       fib_dead;
    unsigned char       fib_protocol;
    unsigned char       fib_scope;
    unsigned char       fib_type;
    __be32          fib_prefsrc;
    u32         fib_tb_id;
    u32         fib_priority;
    struct dst_metrics  *fib_metrics;
    int         fib_nhs;
    struct nexthop      *nh;
    struct rcu_head     rcu;
    struct fib_nh       fib_nh[] __counted_by(fib_nhs);
};
```

`fib_info` 是路由语义的核心：
- 它描述一条路由的"怎么走"：下一跳是谁、出接口、优先级、metrics、源地址等。
- 多个前缀（`fib_alias`）可以共享同一个 `fib_info`，因此 `fib_info` 单独做引用计数。
- `fib_nh[]` 是下一跳数组；`fib_nhs` 表示下一跳数量。单路径路由时 `fib_nhs == 1`。

#### `struct fib_nh` / `struct fib_nh_common` —— 单个下一跳

文件：`include/net/ip_fib.h:83-128`

```c
struct fib_nh_common {
    struct net_device   *nhc_dev;
    netdevice_tracker   nhc_dev_tracker;
    int         nhc_oif;
    unsigned char       nhc_scope;
    u8          nhc_family;
    u8          nhc_gw_family;
    unsigned char       nhc_flags;
    struct lwtunnel_state   *nhc_lwtstate;

    union {
        __be32          ipv4;
        struct in6_addr ipv6;
    } nhc_gw;

    int         nhc_weight;
    atomic_t        nhc_upper_bound;

    /* v4 specific, but allows fib6_nh with v4 routes */
    struct rtable __rcu * __percpu *nhc_pcpu_rth_output;
    struct rtable __rcu     *nhc_rth_input;
    struct fnhe_hash_bucket __rcu *nhc_exceptions;
};

struct fib_nh {
    struct fib_nh_common    nh_common;
    struct hlist_node   nh_hash;
    struct fib_info     *nh_parent;
#ifdef CONFIG_IP_ROUTE_CLASSID
    __u32          nh_tclassid;
#endif
    __be32          nh_saddr;
    int         nh_saddr_genid;
#define fib_nh_family      nh_common.nhc_family
#define fib_nh_dev     nh_common.nhc_dev
#define fib_nh_oif     nh_common.nhc_oif
#define fib_nh_flags       nh_common.nhc_flags
#define fib_nh_lws     nh_common.nhc_lwtstate
#define fib_nh_scope       nh_common.nhc_scope
#define fib_nh_gw_family   nh_common.nhc_gw_family
#define fib_nh_gw4     nh_common.nhc_gw.ipv4
#define fib_nh_gw6     nh_common.nhc_gw.ipv6
#define fib_nh_weight      nh_common.nhc_weight
#define fib_nh_upper_bound nh_common.nhc_upper_bound
};
```

`fib_nh_common` 是下一跳公共部分，`fib_nh` 是 IPv4 专用封装。注意其中的缓存指针：

- `nhc_pcpu_rth_output`：每个 CPU 一个输出 `rtable` 缓存。
- `nhc_rth_input`：单个输入 `rtable` 缓存。
- `nhc_exceptions`：针对特定目的地址的例外缓存（重定向、PMTU 等）。

#### `struct fib_alias` —— Trie 叶子中的别名

文件：`net/ipv4/fib_lookup.h:11-24`

```c
struct fib_alias {
    struct hlist_node   fa_list;
    struct fib_info     *fa_info;
    dscp_t          fa_dscp;
    u8          fa_type;
    u8          fa_state;
    u8          fa_slen;
    u32         tb_id;
    s16         fa_default;
    ...
    struct rcu_head     rcu;
};
```

`fib_alias` 把"一个前缀 + DSCP + 类型"映射到一个 `fib_info`。Trie 的叶子节点上挂的是一个 `fib_alias` 链表。

#### `struct fib_result` —— FIB 查找结果

文件：`include/net/ip_fib.h:173-185`

```c
struct fib_result {
    __be32          prefix;
    unsigned char       prefixlen;
    unsigned char       nh_sel;
    unsigned char       type;
    unsigned char       scope;
    u32         tclassid;
    dscp_t          dscp;
    struct fib_nh_common    *nhc;
    struct fib_info     *fi;
    struct fib_table    *table;
    struct hlist_head   *fa_head;
};
```

`fib_table_lookup()` 返回这个结构，后续由 `fib_select_path()` 选择具体下一跳，再由 `__mkroute_*()` 生成 `rtable`。

---

## 3. FIB 的实现：为什么用 LC-trie 而不是 hash

传统教科书上常讲路由用 "hash" 或 " Patricia tree"。Linux IPv4 在 2.6 以后采用 LC-trie（Level-Compressed trie） 存路由，主要优势：

1. 最长前缀匹配天然适合 trie：从根按位往下走，最深的匹配叶子就是最长前缀。
2. Level compression：把连续单分支的节点压缩，减少内存访问次数。
3. 查找时间稳定，最坏情况可控。
4. 更新方便：插入/删除只影响相关路径，不需要像 hash 那样重新哈希整张表。

### 3.1 LC-trie 在 Linux 中的映射

```text
fib_table
   └── tb_data[] 指向 struct trie
        └── kv (root key_vector)
             ├── 内部节点 key_vector：pos/bits/slen + tnode[] 子节点指针数组
             └── 叶子节点 key_vector：pos=KEYLENGTH，leaf 链表挂 fib_alias
```

文件：`net/ipv4/fib_trie.c:121-172`

```c
struct key_vector {
    t_key key;
    unsigned char pos;      /* 节点在 key 中的位置 */
    unsigned char bits;     /* 子节点索引位数 */
    unsigned char slen;
    union {
        struct hlist_head leaf;         /* 叶子：挂 fib_alias 链表 */
        DECLARE_FLEX_ARRAY(struct key_vector __rcu *, tnode); /* 内部节点 */
    };
};
```

### 3.2 查找：`fib_find_node` + `fib_table_lookup`

文件：`net/ipv4/fib_trie.c:930-970`

```c
static struct key_vector *fib_find_node(struct trie *t,
                    struct key_vector **tp, u32 key)
{
    struct key_vector *pn, *n = t->kv;
    unsigned long index = 0;

    do {
        pn = n;
        n = get_child_rcu(n, index);

        if (!n)
            break;

        index = get_cindex(key, n);

        if (index >= (1ul << n->bits)) {
            n = NULL;
            break;
        }
    } while (IS_TNODE(n));

    *tp = pn;
    return n;
}
```

`fib_find_node` 负责在 trie 中定位一个叶子（leaf）。真正的最长前缀匹配和回退逻辑在 `fib_table_lookup` 中。

文件：`net/ipv4/fib_trie.c:1420-1624`

```c
int fib_table_lookup(struct fib_table *tb, const struct flowi4 *flp,
             struct fib_result *res, int fib_flags)
{
    struct trie *t = (struct trie *) tb->tb_data;
    const t_key key = ntohl(flp->daddr);
    struct key_vector *n, *pn;
    struct fib_alias *fa;
    ...

    /* Step 1: Travel to the longest prefix match in the trie */
    for (;;) {
        index = get_cindex(key, n);

        if (index >= (1ul << n->bits))
            break;

        /* we have found a leaf. Prefixes have already been compared */
        if (IS_LEAF(n))
            goto found;

        if (n->slen > n->pos) {
            pn = n;
            cindex = index;
        }

        n = get_child_rcu(n, index);
        if (unlikely(!n))
            goto backtrace;
    }

    /* Step 2: Sort out leaves and begin backtracing for longest prefix */
    for (;;) {
        ...
backtrace:
        while (!cindex) {
            ...
            if (IS_TRIE(pn)) {
                trace_fib_table_lookup(tb->tb_id, flp, NULL, -EAGAIN);
                return -EAGAIN;
            }
            ...
        }
        ...
    }
found:
    fa = fib_find_alias(&n->leaf, ...);
    ...
}
```

查找分为两步：

1. Step 1：从根向下走，按 key 的位选择子节点，直到叶子或 NULL。
2. Step 2：如果当前节点前缀不匹配，沿父指针回退，寻找次长前缀的叶子。
3. 找到叶子后，遍历 `fib_alias` 链表，匹配 DSCP、scope、priority、table id。

> 回退（backtrace）是 trie 实现最长前缀匹配的关键：当 key 的某段与节点前缀不一致时，需要回到父节点，看父节点或更短前缀是否仍有匹配项。

### 3.3 插入：`fib_table_insert`

文件：`net/ipv4/fib_trie.c:1194-1380`

插入一条路由时：

1. `fib_create_info()` 创建或复用 `fib_info`（下一跳、metrics、权重）。
2. `fib_find_node()` 在 trie 中定位对应前缀的叶子。
3. `fib_find_alias()` 在叶子链表中找合适的插入位置。
4. `fib_insert_alias()` 把新的 `fib_alias` 挂到叶子链表上。

```text
inet_rtm_newroute()
  → rtm_to_fib_config()          解析 nlattr
  → fib_create_info()            创建/复用 fib_info
      → fib_nh_init()            初始化单个下一跳
      → fib_rebalance()          多路径权重计算
  → fib_table_insert()
      → fib_find_node()          定位 trie 叶子
      → fib_find_alias()         在 fib_alias 链表中定位
      → fib_insert_alias()       插入别名
```

---

## 4. 输出路由查找流程

协议栈要发送一个报文时，典型的上层入口是 `ip_queue_xmit()`，它会调用路由 API 获取 `rtable`。

### 4.1 调用链

```text
ip_queue_xmit(skb)
  → ip_route_output_ports(net, fl4, sk, daddr, saddr, dport, sport, proto, tos, oif)
        include/net/route.h:201
    → ip_route_output_flow(net, fl4, sk)
          net/ipv4/route.c:2929
        → __ip_route_output_key(net, fl4)
              include/net/route.h:166
          → ip_route_output_key_hash(net, fl4, skb)
                net/ipv4/route.c:2691
            → ip_route_output_key_hash_rcu(net, fl4, &res, skb)
                  net/ipv4/route.c:2712
              │
              ├── 特殊处理：本地回环、广播/组播、指定 oif
              │
              → fib_lookup(net, fl4, res, 0)
                    include/net/ip_fib.h:316
                │
                ├── 无策略路由：fib_get_table(RT_TABLE_MAIN) → fib_table_lookup()
                │
                └── 有策略路由：__fib_lookup() → fib_rules_lookup()
                                                  → fib4_rule_action()
                                                    → fib_get_table(rule->table)
                                                    → fib_table_lookup()
              │
              → fib_select_path(net, res, fl4, skb)
                    net/ipv4/fib_semantics.c:2208
                ├── multipath: fib_multipath_hash() → fib_select_multipath()
                └── 单路径默认路由: fib_select_default()
              │
              → __mkroute_output(res, fl4, orig_oif, dev_out, flags)
                    net/ipv4/route.c:2562
                ├── 检查 nhc->nhc_pcpu_rth_output / fnhe 缓存
                → rt_dst_alloc(dev_out, flags, type, noxfrm)
                → rt_set_nexthop(rt, daddr, res, fnhe, fi, type, 0, do_cache)
                      net/ipv4/route.c:1586
                    ├── 成功：rt_cache_route() 存入 percpu 下一跳缓存
                    └── 失败：rt_add_uncached_list()
```

### 4.2 高层入口：`ip_route_output_flow`

文件：`net/ipv4/route.c:2929-2946`

```c
struct rtable *ip_route_output_flow(struct net *net, struct flowi4 *flp4,
                    const struct sock *sk)
{
    struct rtable *rt = __ip_route_output_key(net, flp4);

    if (IS_ERR(rt))
        return rt;

    if (flp4->flowi4_proto) {
        flp4->flowi4_oif = rt->dst.dev->ifindex;
        rt = dst_rtable(xfrm_lookup_route(net, &rt->dst,
                          flowi4_to_flowi(flp4),
                          sk, 0));
    }

    return rt;
}
```

`ip_route_output_flow()` 是 socket 层最常用的输出路由入口：
- 先做一次基础路由查找 `__ip_route_output_key()`。
- 如果涉及 IPsec/XFRM，再走 `xfrm_lookup_route()` 做安全策略路由。

### 4.3 核心查找：`ip_route_output_key_hash_rcu`

文件：`net/ipv4/route.c:2712-2880`

这个函数处理各种边界情况（源地址非法、组播、广播、指定 oif、本地回环），然后进入 `fib_lookup()`。

```c
struct rtable *ip_route_output_key_hash_rcu(struct net *net, struct flowi4 *fl4,
                        struct fib_result *res,
                        const struct sk_buff *skb)
{
    ...
    if (!fl4->daddr) {
        fl4->daddr = fl4->saddr;
        if (!fl4->daddr)
            fl4->daddr = fl4->saddr = htonl(INADDR_LOOPBACK);
        dev_out = net->loopback_dev;
        fl4->flowi4_oif = LOOPBACK_IFINDEX;
        res->type = RTN_LOCAL;
        flags |= RTCF_LOCAL;
        goto make_route;
    }

    err = fib_lookup(net, fl4, res, 0);
    ...
make_route:
    rth = __mkroute_output(res, fl4, orig_oif, dev_out, flags);
    ...
}
```

### 4.4 构造路由对象：`__mkroute_output` + `rt_set_nexthop`

文件：`net/ipv4/route.c:2562-2685`

`__mkroute_output()` 根据 `fib_result` 决定报文类型（单播/广播/组播/本地），然后检查缓存：

```c
static struct rtable *__mkroute_output(const struct fib_result *res,
                       const struct flowi4 *fl4, int orig_oif,
                       struct net_device *dev_out,
                       unsigned int flags)
{
    struct fib_info *fi = res->fi;
    struct fib_nh_exception *fnhe;
    u16 type = res->type;
    struct rtable *rth;
    bool do_cache;
    ...
    if (ipv4_is_lbcast(fl4->daddr)) {
        type = RTN_BROADCAST;
        /* reset fi to prevent gateway resolution */
        fi = NULL;
    } else if (ipv4_is_multicast(fl4->daddr)) {
        type = RTN_MULTICAST;
    } else if (ipv4_is_zeronet(fl4->daddr)) {
        return ERR_PTR(-EINVAL);
    }
    ...
    do_cache = true;
    if (type == RTN_BROADCAST) {
        flags |= RTCF_BROADCAST | RTCF_LOCAL;
    } else if (type == RTN_MULTICAST) {
        flags |= RTCF_MULTICAST | RTCF_LOCAL;
        if (!ip_check_mc_rcu(in_dev, fl4->daddr, fl4->saddr,
                     fl4->flowi4_proto))
            flags &= ~RTCF_LOCAL;
        else
            do_cache = false;
        /* If multicast route do not exist use
         * default one, but do not gateway in this case.
         * Yes, it is hack.
         */
        if (fi && res->prefixlen < 4)
            fi = NULL;
    } else if ((type == RTN_LOCAL) && (orig_oif != 0) &&
           (orig_oif != dev_out->ifindex)) {
        /* For local routes that require a particular output interface
         * we do not want to cache the result.
         */
        do_cache = false;
    }

    fnhe = NULL;
    do_cache &= fi != NULL;
    if (fi) {
        struct fib_nh_common *nhc = FIB_RES_NHC(*res);
        struct rtable __rcu **prth;

        fnhe = find_exception(nhc, fl4->daddr);
        if (!do_cache)
            goto add;
        if (fnhe) {
            prth = &fnhe->fnhe_rth_output;
        } else {
            if (unlikely(fl4->flowi4_flags &
                     FLOWI_FLAG_KNOWN_NH &&
                     !(nhc->nhc_gw_family &&
                       nhc->nhc_scope == RT_SCOPE_LINK))) {
                do_cache = false;
                goto add;
            }
            prth = raw_cpu_ptr(nhc->nhc_pcpu_rth_output);
        }
        rth = rcu_dereference(*prth);
        if (rt_cache_valid(rth) && dst_hold_safe(&rth->dst))
            return rth;
    }

add:
    rth = rt_dst_alloc(dev_out, flags, type,
               IN_DEV_ORCONF(in_dev, NOXFRM));
    if (!rth)
        return ERR_PTR(-ENOBUFS);

    rth->rt_iif = orig_oif;
    ...
    rt_set_nexthop(rth, fl4->daddr, res, fnhe, fi, type, 0, do_cache);
    lwtunnel_set_redirect(&rth->dst);

    return rth;
}
```

> 注意：per-cpu 输出缓存命中判断只用 `rt_cache_valid(rth) && dst_hold_safe(&rth->dst)`，不再比较 `daddr`/`saddr`。因为缓存是按下一跳组织的，每条缓存对应固定的 (下一跳, 出接口) 组合，daddr/saddr 已隐含在 fib_result 中。`struct rtable` 里也没有 `rt_dst`/`rt_src` 字段（老版本路由 cache 时代的残留，现代内核已移除）。

文件：`net/ipv4/route.c:1586-1643`

`rt_set_nexthop()` 设置网关、metrics、lwtunnel，并尝试缓存：

```c
static void rt_set_nexthop(struct rtable *rt, __be32 daddr,
               const struct fib_result *res,
               struct fib_nh_exception *fnhe,
               struct fib_info *fi, u16 type, u32 itag,
               const bool do_cache)
{
    bool cached = false;

    if (fi) {
        struct fib_nh_common *nhc = FIB_RES_NHC(*res);

        if (nhc->nhc_gw_family && nhc->nhc_scope == RT_SCOPE_LINK) {
            rt->rt_uses_gateway = 1;
            rt->rt_gw_family = nhc->nhc_gw_family;
            if (likely(nhc->nhc_gw_family == AF_INET))
                rt->rt_gw4 = nhc->nhc_gw.ipv4;
            else
                rt->rt_gw6 = nhc->nhc_gw.ipv6;
        }

        ip_dst_init_metrics(&rt->dst, fi->fib_metrics);
        rt->dst.lwtstate = lwtstate_get(nhc->nhc_lwtstate);

        if (unlikely(fnhe))
            cached = rt_bind_exception(rt, fnhe, daddr, do_cache);
        else if (do_cache)
            cached = rt_cache_route(nhc, rt);
        if (unlikely(!cached))
            rt_add_uncached_list(rt);
    } else
        rt_add_uncached_list(rt);
    ...
}
```

缓存策略：
- 如果存在 `fib_nh_exception`（如 ICMP 重定向、PMTU 例外），优先绑定到例外缓存。
- 否则尝试存入 `nhc_pcpu_rth_output`（per-cpu 输出缓存）。
- 如果缓存失败，放入 `rt_uncached_list`，由 GC 回收。

---

## 5. 输入路由查找流程

网卡收到一个 IP 报文后，协议栈需要判断：是上交本机、转发，还是丢弃。

### 5.1 调用链

```text
网卡驱动收包
  → __netif_receive_skb_core()
    → 根据 ethertype = 0x0800 进入 ip_rcv()
      → ip_rcv_finish()
        → ip_route_input_noref(skb, daddr, saddr, dscp, dev)
              net/ipv4/route.c:2546
          → ip_route_input_rcu(skb, daddr, saddr, dscp, dev, &res)
                net/ipv4/route.c:2493
            │
            ├── 组播 → ip_route_input_mc()
            └── 其它 → ip_route_input_slow()
                          net/ipv4/route.c:2263
                │
                → 构造 flowi4 → fib_lookup() → fib_table_lookup()
                → ip_mkroute_input(skb, res, in_dev, daddr, saddr, dscp, flkeys)
                      net/ipv4/route.c:2169
                    ├── multipath: fib_multipath_hash() → fib_select_multipath()
                    │
                    → __mkroute_input(skb, res, in_dev, daddr, saddr, dscp)
                          net/ipv4/route.c:1812
                        ├── 检查 nhc->nhc_rth_input / fnhe->fnhe_rth_input 缓存
                        → rt_dst_alloc() → rt_set_nexthop()
```

### 5.2 入口：`ip_route_input_noref`

文件：`net/ipv4/route.c:2546-2559`

```c
enum skb_drop_reason ip_route_input_noref(struct sk_buff *skb, __be32 daddr,
                      __be32 saddr, dscp_t dscp,
                      struct net_device *dev)
{
    enum skb_drop_reason reason;
    struct fib_result res;

    rcu_read_lock();
    reason = ip_route_input_rcu(skb, daddr, saddr, dscp, dev, &res);
    rcu_read_unlock();

    return reason;
}
```

### 5.3 `ip_route_input_rcu`

文件：`net/ipv4/route.c:2493-2544`

```c
static enum skb_drop_reason
ip_route_input_rcu(struct sk_buff *skb, __be32 daddr, __be32 saddr,
           dscp_t dscp, struct net_device *dev,
           struct fib_result *res)
{
    if (ipv4_is_multicast(daddr)) {
        ...
        reason = ip_route_input_mc(skb, daddr, saddr, dscp, dev, our);
        return reason;
    }

    return ip_route_input_slow(skb, daddr, saddr, dscp, dev, res);
}
```

输入路由的特殊点：
- 组播单独处理，判断是否是本机订阅的组播组。
- 非组播走 `ip_route_input_slow()`，构造 `flowi4` 后查 FIB。
- 需要额外做反向路径过滤（RPF / Martian source）检查，防止源地址欺骗。

---

## 6. 策略路由（Policy Routing）

Linux 默认使用单一路由表（`RT_TABLE_MAIN`）。开启 `CONFIG_IP_MULTIPLE_TABLES` 后，支持策略路由，即先按规则（`ip rule`）选择路由表，再在该表内做最长前缀匹配。

### 6.1 规则匹配：`fib4_rule_match`

文件：`net/ipv4/fib_rules.c:180-215`

```c
INDIRECT_CALLABLE_SCOPE int fib4_rule_match(struct fib_rule *rule,
                        struct flowi *fl, int flags)
{
    struct fib4_rule *r = (struct fib4_rule *) rule;
    struct flowi4 *fl4 = &fl->u.ip4;
    __be32 daddr = fl4->daddr;
    __be32 saddr = fl4->saddr;

    if (((saddr ^ r->src) & r->srcmask) ||
        ((daddr ^ r->dst) & r->dstmask))
        return 0;

    /* When DSCP selector is used we need to match on the entire DSCP field
     * in the flow information structure. When TOS selector is used we need
     * to mask the upper three DSCP bits prior to matching to maintain
     * legacy behavior.
     */
    if (r->dscp_full && (r->dscp ^ fl4->flowi4_dscp) & r->dscp_mask)
        return 0;
    else if (!r->dscp_full && r->dscp &&
         !fib_dscp_masked_match(r->dscp, fl4))
        return 0;

    if (rule->ip_proto && (rule->ip_proto != fl4->flowi4_proto))
        return 0;

    if (!fib_rule_port_match(&rule->sport_range, rule->sport_mask,
                 fl4->fl4_sport))
        return 0;

    if (!fib_rule_port_match(&rule->dport_range, rule->dport_mask,
                 fl4->fl4_dport))
        return 0;

    return 1;
}
```

可匹配的维度：源/目的 IP、DSCP/TOS、四层协议、源/目的端口、入接口、fwmark 等。

### 6.2 规则动作：`fib4_rule_action`

文件：`net/ipv4/fib_rules.c:111-145`

```c
INDIRECT_CALLABLE_SCOPE int fib4_rule_action(struct fib_rule *rule,
                         struct flowi *flp, int flags,
                         struct fib_lookup_arg *arg)
{
    int err = -EAGAIN;
    struct fib_table *tbl;
    u32 tb_id;

    switch (rule->action) {
    case FR_ACT_TO_TBL:
        break;

    case FR_ACT_UNREACHABLE:
        return -ENETUNREACH;

    case FR_ACT_PROHIBIT:
        return -EACCES;

    case FR_ACT_BLACKHOLE:
    default:
        return -EINVAL;
    }

    rcu_read_lock();

    tb_id = fib_rule_get_table(rule, arg);
    tbl = fib_get_table(rule->fr_net, tb_id);
    if (tbl)
        err = fib_table_lookup(tbl, &flp->u.ip4,
                       (struct fib_result *)arg->result,
                       arg->flags);

    rcu_read_unlock();
    return err;
}
```

规则动作包括：
- `FR_ACT_TO_TBL`：跳到指定路由表继续查找。
- `FR_ACT_UNREACHABLE`：返回不可达。
- `FR_ACT_PROHIBIT`：返回禁止访问（`EACCES`）。
- `FR_ACT_BLACKHOLE`：静默丢弃。

### 6.3 `__fib_lookup` 调用规则引擎

文件：`net/ipv4/fib_rules.c:84-108`

```c
int __fib_lookup(struct net *net, struct flowi4 *flp,
         struct fib_result *res, unsigned int flags)
{
    struct fib_lookup_arg arg = {
        .result = res,
        .flags = flags,
    };
    int err;

    l3mdev_update_flow(net, flowi4_to_flowi(flp));

    err = fib_rules_lookup(net->ipv4.rules_ops, flowi4_to_flowi(flp), 0, &arg);
    ...

    if (err == -ESRCH)
        err = -ENETUNREACH;

    return err;
}
```

策略路由 vs 普通路由的入口差异只在 `fib_lookup()` 这个 inline 函数里：
- 无策略路由（未开 `CONFIG_IP_MULTIPLE_TABLES`）：直接查 `RT_TABLE_MAIN`。
- 有策略路由：先 `fib_rules_lookup()` 选规则，再 `fib_table_lookup()` 查表。

> 注意：这两个 `fib_lookup()` 是同一个符号的两种编译期实现，由 `CONFIG_IP_MULTIPLE_TABLES` 在头文件里选择，不是运行时分支。无策略路由版本见 `include/net/ip_fib.h:316`，有策略路由版本见 `include/net/ip_fib.h:374`。

---

## 7. Multipath（多路径路由）

### 7.1 等价路由的下一跳组织

`fib_info` 内部包含 `fib_nh[]` 数组，`fib_nhs` 表示下一跳数量。每个 `fib_nh` 有 `nhc_weight` 权重。

### 7.2 权重计算：`fib_rebalance`

文件：`net/ipv4/fib_semantics.c:824-861`

```c
static void fib_rebalance(struct fib_info *fi)
{
    int total;
    int w;

    if (fib_info_num_path(fi) < 2)
        return;

    total = 0;
    for_nexthops(fi) {
        if (nh->fib_nh_flags & RTNH_F_DEAD)
            continue;

        if (ip_ignore_linkdown(nh->fib_nh_dev) &&
            nh->fib_nh_flags & RTNH_F_LINKDOWN)
            continue;

        total += nh->fib_nh_weight;
    } endfor_nexthops(fi);

    w = 0;
    change_nexthops(fi) {
        int upper_bound;

        if (nexthop_nh->fib_nh_flags & RTNH_F_DEAD) {
            upper_bound = -1;
        } else if (ip_ignore_linkdown(nexthop_nh->fib_nh_dev) &&
               nexthop_nh->fib_nh_flags & RTNH_F_LINKDOWN) {
            upper_bound = -1;
        } else {
            w += nexthop_nh->fib_nh_weight;
            upper_bound = DIV_ROUND_CLOSEST_ULL((u64)w << 31,
                                total) - 1;
        }

        atomic_set(&nexthop_nh->fib_nh_upper_bound, upper_bound);
    } endfor_nexthops(fi);
}
```

将 `[0, 2^31-1]` 区间按权重比例划分给每个可用下一跳，`fib_nh_upper_bound` 是该下一跳的上界。两个关键点：
1. 第一个循环用 `for_nexthops`（只读）累加存活下一跳的权重和 `total`，跳过 DEAD/LINKDOWN。
2. 第二个循环用 `change_nexthops`（可写）写入 `upper_bound`：DEAD/LINKDOWN 的下一跳不跳过，而是显式置为 `-1`——这正是 §7.3 选路时 `nh_upper_bound == -1` 跳过该下一跳的来源。

### 7.3 选路：`fib_select_path` + `fib_select_multipath`

文件：`net/ipv4/fib_semantics.c:2208-2239`

```c
void fib_select_path(struct net *net, struct fib_result *res,
             struct flowi4 *fl4, const struct sk_buff *skb)
{
    if (fl4->flowi4_oif)
        goto check_saddr;

#ifdef CONFIG_IP_ROUTE_MULTIPATH
    if (fib_info_num_path(res->fi) > 1) {
        int h = fib_multipath_hash(net, fl4, skb, NULL);

        fib_select_multipath(res, h, fl4);
    }
    else
#endif
    if (!res->prefixlen &&
        res->table->tb_num_default > 1 &&
        res->type == RTN_UNICAST)
        fib_select_default(fl4, res);

check_saddr:
    if (!fl4->saddr) {
        struct net_device *l3mdev;

        l3mdev = dev_get_by_index_rcu(net, fl4->flowi4_l3mdev);

        if (!l3mdev ||
            l3mdev_master_dev_rcu(FIB_RES_DEV(*res)) == l3mdev)
            fl4->saddr = fib_result_prefsrc(net, res);
        else
            fl4->saddr = inet_select_addr(l3mdev, 0, RT_SCOPE_LINK);
    }
}
```

文件：`net/ipv4/fib_semantics.c:2164-2206`

```c
void fib_select_multipath(struct fib_result *res, int hash,
              const struct flowi4 *fl4)
{
    struct fib_info *fi = res->fi;
    struct net *net = fi->fib_net;
    bool use_neigh;
    int score = -1;
    __be32 saddr;

    if (unlikely(res->fi->nh)) {
        nexthop_path_fib_result(res, hash);
        return;
    }

    use_neigh = READ_ONCE(net->ipv4.sysctl_fib_multipath_use_neigh);
    saddr = fl4 ? fl4->saddr : 0;

    change_nexthops(fi) {
        int nh_upper_bound, nh_score = 0;

        /* Nexthops without a carrier are assigned an upper bound of
         * minus one when "ignore_routes_with_linkdown" is set.
         */
        nh_upper_bound = atomic_read(&nexthop_nh->fib_nh_upper_bound);
        if (nh_upper_bound == -1 ||
            (use_neigh && !fib_good_nh(nexthop_nh)))
            continue;

        if (saddr && nexthop_nh->nh_saddr == saddr)
            nh_score += 2;
        if (hash <= nh_upper_bound)
            nh_score++;
        if (score < nh_score) {
            res->nh_sel = nhsel;
            res->nhc = &nexthop_nh->nh_common;
            if (nh_score == 3 || (!saddr && nh_score == 1))
                return;
            score = nh_score;
        }

    } endfor_nexthops(fi);
}
```

选路逻辑：
1. 如果路由用的是 nexthop 对象（`fib_info.nh != NULL`，即 `ip route add ... nhid N` 配置的多路径），直接走 `nexthop_path_fib_result()` 按 hash 选下一跳并返回——不进入下面的逐跳打分。
2. 否则用 `fib_multipath_hash()` 计算一个 hash 值，遍历下一跳逐个打分：
   - 跳过失效（`upper_bound == -1`，由 §7.2 `fib_rebalance` 设置）或邻居不可达（`use_neigh=1`）的下一跳。
   - 如果源地址匹配该下一跳的 `nh_saddr`，加分（+2）。
   - 如果 hash 落在该下一跳的 upper_bound 范围内，加分（+1）。
   - 选择得分最高的下一跳，写入 `res->nhc`；满分（3 分，或无 saddr 时 1 分）即提前返回。

### 7.4 多路径哈希策略

文件：`net/ipv4/route.c:2066-2166`

`sysctl_fib_multipath_hash_policy` 控制哈希输入：

| policy | 含义 |
|--------|------|
| 0 | 仅 L3 地址（src/dst IP） |
| 1 | L3 + L4（端口、协议） |
| 2 | 内层 IP（用于隧道/GRE 等封装） |
| 3 | 自定义字段哈希，由 `sysctl_fib_multipath_hash_fields` 位掩码选择参与哈希的字段 |

> 注意：policy 3 不是 eBPF。它调用 `fib_multipath_custom_hash_skb/fl4()`，读取 `sysctl_fib_multipath_hash_fields` 位掩码（`FIB_MULTIPATH_HASH_FIELD_*`，如 `SRC_IP`/`DST_IP`/`IP_PROTO`/`SRC_PORT`/`DST_PORT` 及对应的 inner 字段），按用户勾选的字段做哈希。

最终通过 `fib_multipath_hash_from_keys()`（内部 SipHash）计算哈希值，保证流量分布均匀且同一条流固定到同一下一跳。

---

## 8. 路由 cache 与 FIB 的关系

### 8.1 现代内核中 route cache 已不存在

传统 Linux 2.2/2.4 有一个全局的 "routing cache"，按 (src, dst, tos, oif) 做哈希。这个全局 cache 在 3.x 时代被移除，原因是：

1. 全局 cache 容易成为性能瓶颈和锁竞争热点。
2. DoS 攻击可以轻易把 cache 打满（如随机源地址扫描）。
3. FIB 本身用 LC-trie 已经够快，不需要一层大而全的 cache。

现代内核采用分布式、按下一跳组织的 dst cache：

| 缓存级别 | 位置 | 说明 |
|---|---|---|
| Per-nexthop 输出缓存 | `fib_nh_common.nhc_pcpu_rth_output` | 每个 CPU 一个 `rtable`，对应某个下一跳的通用出路由 |
| Per-nexthop 输入缓存 | `fib_nh_common.nhc_rth_input` | 单个 `rtable`，对应某个下一跳的通用入路由 |
| Per-exception 缓存 | `fib_nh_exception.fnhe_rth_input/output` | 针对特定目的地址的例外缓存（重定向、PMTU） |
| Uncached list | `rt_uncached_list` | 无法被缓存的 `rtable` 临时列表 |

### 8.2 `rtable` 与 `dst_entry` 的关系

```c
struct rtable {
    struct dst_entry dst;   /* 第一个字段 */
    ...
};

#define dst_rtable(_ptr) container_of_const(_ptr, struct rtable, dst)
```

协议栈中 `skb->dst` 指向通用 `dst_entry`，通过 `skb_rtable(skb)` 得到 IPv4 专用的 `rtable`。

### 8.3 FIB 查找结果如何变成路由对象

```text
fib_table_lookup() 返回 fib_result
        │
        ▼
fib_select_path() 选择具体下一跳 → res->nhc
        │
        ▼
__mkroute_output() / __mkroute_input()
        │
        ├── rt_dst_alloc() 分配 rtable
        │
        └── rt_set_nexthop()
              ├── 设置 rt_gw4/6、rt_uses_gateway、metrics、lwtunnel
              ├── 尝试 rt_bind_exception() → fnhe 缓存
              ├── 尝试 rt_cache_route() → nhc_pcpu_rth_output / nhc_rth_input
              └── 失败则 rt_add_uncached_list()
```

### 8.4 缓存失效

当路由表、接口状态、邻居状态、PMTU 等发生变化时，内核会：
- 增加 `rt_genid`，使旧缓存 `rt_cache_valid()` 返回 false。
- 调用 `rt_cache_flush()` 或按设备 flush。
- 被释放的 `rtable` 走 `dst_destroy()` 清理资源。

---

## 9. 用户态配置接口

用户空间通过 netlink（`ip route`、`ip rule`）或 ioctl 配置路由。

### 9.1 新增路由：`inet_rtm_newroute`

文件：`net/ipv4/fib_frontend.c:910`

```text
用户空间: ip route add 192.168.2.0/24 via 10.0.0.1 dev eth0
  │
  ▼
RTM_NEWROUTE netlink 消息
  │
  ▼
inet_rtm_newroute(skb, nlh, extack)
  │   net/ipv4/fib_frontend.c:910
  ▼
rtm_to_fib_config(net, skb, nlh, &cfg, extack)
  │   net/ipv4/fib_frontend.c:734
  │   解析 RTA_DST/OIF/GATEWAY/PRIORITY/MULTIPATH/NH_ID 等属性
  ▼
fib_new_table(net, cfg.fc_table) / fib_get_table()
  │   net/ipv4/fib_frontend.c:77/113
  ▼
fib_table_insert(net, tb, &cfg, extack)
  │   net/ipv4/fib_trie.c:1194
  │
  ├── fib_create_info(cfg, extack)   ← 创建/复用 fib_info
  │       net/ipv4/fib_semantics.c:1347
  │
  ├── fib_find_node(t, &tp, key)     ← 在 trie 中找叶子
  │       net/ipv4/fib_trie.c:930
  │
  ├── fib_find_alias()               ← 在叶子链表中定位插入点
  │       net/ipv4/fib_trie.c:977
  │
  └── fib_insert_alias()             ← 插入新 fib_alias
```

### 9.2 删除路由：`inet_rtm_delroute`

删除流程类似：
1. `rtm_to_fib_config()` 解析配置。
2. `fib_table_delete()` 在 trie 中定位并删除对应的 `fib_alias`。
3. 如果 `fib_info` 的引用计数归零，释放 `fib_info` 和下一跳资源。

---

## 10. 路由子系统整体架构图

```text
┌─────────────────────────────────────────────────────────────┐
│                    用户空间配置接口                           │
│   ip route / ip rule / netlink / ioctl                      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 fib_frontend.c                              │
│  - netlink 入口: inet_rtm_newroute/delroute                 │
│  - rtm_to_fib_config 解析 nlattr                            │
│  - fib_new_table / fib_get_table                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 fib_rules.c (CONFIG_IP_MULTIPLE_TABLES)     │
│  - 策略路由 fib4_rule                                       │
│  - __fib_lookup → fib_rules_lookup → fib4_rule_action       │
│  - 匹配 src/dst/dscp/mark/iif/oif/ports 等                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 fib_trie.c                                  │
│  - LC-trie 数据结构: key_vector / tnode / trie              │
│  - fib_table_insert / fib_table_delete / fib_table_lookup   │
│  - fib_find_node / fib_find_alias / fib_insert_alias        │
│  - leaf 节点是 fib_alias 链表                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 fib_semantics.c                             │
│  - fib_info / fib_nh / fib_nh_common 创建与销毁             │
│  - fib_create_info / fib_find_info / fib_rebalance          │
│  - 多路径选路: fib_select_multipath / fib_select_path       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 route.c                                     │
│  - 高层路由 API: ip_route_output_* / ip_route_input_*       │
│  - dst cache 管理: rt_dst_alloc / rt_dst_clone              │
│  - 下一跳缓存: nhc_pcpu_rth_output / nhc_rth_input          │
│  - 例外缓存: fib_nh_exception (fnhe)                        │
└─────────────────────────────────────────────────────────────┘
```

---

## 11. 总结

Linux v7.1-rc7 的 IPv4 路由子系统可以概括为：

1. FIB 用 LC-trie 存路由，天然支持最长前缀匹配；叶子节点挂 `fib_alias`，再由 `fib_alias` 指向 `fib_info`（下一跳集合）。
2. `struct flowi4` 是查询键，包含目的/源地址、接口、mark、DSCP、scope、四层端口等，决定一次路由查询的行为。
3. `struct rtable` 是查询结果，内嵌 `dst_entry`，供协议栈后续收发报文使用。
4. 策略路由通过 `fib_rules_lookup()` 先匹配规则、再选路由表，支持 src/dst/mark/dscp/ports/iif/oif 等多维度。
5. Multipath 在 `fib_info` 中维护多个 `fib_nh`，按权重分配 hash 区间，由 `fib_select_multipath()` 选路。
6. 全局 route cache 已不存在，现代内核使用按下一跳分布的 dst cache（percpu output、input、exception、uncached list），避免全局锁和 DoS 问题。

一句话：Linux IPv4 路由 = LC-trie FIB 做最长前缀匹配 + 策略路由选表 + 按下一跳分布的 dst cache。
