# 对话上下文续接文件

> 最近更新：2026-06-22（新增第 9 节：从"读懂源码"到"高手"的完整学习路线）  
> 用途：下次开启新对话时，将此文件内容粘贴给 AI，即可恢复当前讨论上下文。

---

## 1. 项目背景

- **项目路径**：`/home/summus/linux`
- **项目类型**：Linux Kernel 源码（版本约 v7.1-rc7）
- **已处理问题**：工作区曾出现 93562 个文件的换行符被批量污染为 CRLF，已通过 `git checkout -- .` 完全恢复，当前工作区干净。

## 2. 用户背景

- 具备**鸿蒙内核协议栈开发经验**
- 熟悉内核编程、并发控制、协议状态机
- 曾负责开发：**ARP 协议、DHCP 协议、ICMP 协议、TCP 协议**
- 当前目标：将已有经验平移到 **Linux 内核网络协议栈**，系统学习其源码实现

## 3. Linux 网络协议栈源码对位表

### ARP
| 项目 | 内容 |
|------|------|
| 文件 | `net/ipv4/arp.c` |
| 收包 | `arp_rcv()` |
| 发包 | `arp_xmit()` → `arp_xmit_finish()` |
| 邻居子系统 | `net/core/neighbour.c`（Linux 将 ARP 缓存/状态机统一到邻居层） |
| 关键结构 | `struct neighbour`、`struct arphdr` |

### ICMP
| 项目 | 内容 |
|------|------|
| 文件 | `net/ipv4/icmp.c` |
| 收包 | `icmp_rcv()` |
| 主动发送 | `__icmp_send()`（被 IP/路由/NAT 等调用） |
| 响应 | `icmp_reply()` |
| 特殊注意 | 有复杂的 rate limit 机制（`icmp_ratelimit`） |

### DHCP
- **Linux 内核没有完整 DHCP 协议栈**，常规 DHCP 在用户态实现（`dhcpcd`、`systemd-networkd`）。
- 内核仅有一个极简的 BOOTP/DHCP 客户端：`net/ipv4/ipconfig.c`（用于 initrd 无盘启动）。
- 若需了解内核如何支撑用户态 DHCP，关注：
  - `net/packet/af_packet.c`（raw socket 收发包）
  - `net/ipv4/udp.c`（UDP 67/68 端口绑定）
  - `net/core/rtnetlink.c`（网卡/地址配置通知）

### TCP（核心且最庞大）
| 功能 | 文件 | 关键函数 |
|------|------|---------|
| Socket 调用入口 | `net/ipv4/tcp.c` | `tcp_sendmsg()`、`tcp_recvmsg()`、`tcp_connect()`、`tcp_close()` |
| 接收 + 状态机 | `net/ipv4/tcp_input.c` | `tcp_rcv_established()`、`tcp_rcv_state_process()` **（最核心）** |
| 发送 | `net/ipv4/tcp_output.c` | `tcp_write_xmit()`、`__tcp_transmit_skb()` |
| IPv4 收包入口 | `net/ipv4/tcp_ipv4.c` | `tcp_v4_rcv()`、`tcp_v4_do_rcv()` |
| 定时器 | `net/ipv4/tcp_timer.c` | `tcp_retransmit_timer()`、`tcp_delack_timer()` |
| TimeWait | `net/ipv4/tcp_minisocks.c` | `tcp_timewait_state_process()` |
| FastOpen | `net/ipv4/tcp_fastopen.c` | — |
| 拥塞控制框架 | `net/ipv4/tcp_cong.c` | `tcp_cong_avoid()`，算法通过 `struct tcp_congestion_ops` 注册 |
| BBR 算法 | `net/ipv4/tcp_bbr.c` | — |
| Cubic 算法 | `net/ipv4/tcp_cubic.c` | — |

## 4. 建议学习顺序（针对该用户背景）

1. **ARP（1~2 天）**：`arp_rcv()` / `arp_xmit()` → 重点理解 `neighbour.c` 通用邻居状态机
2. **ICMP（1~2 天）**：`icmp_rcv()` → `icmp_reply()` / `__icmp_send()`，注意 rate limit
3. **TCP（2~4 周）**：
   - 第 1 周：发包流 `tcp_sendmsg()` → `tcp_write_xmit()` → `ip_queue_xmit()`
   - 第 2 周：收包流 `tcp_v4_rcv()` → `tcp_rcv_established()` / `tcp_rcv_state_process()`
   - 第 3 周：定时器 + 重传 + 拥塞控制框架
   - 第 4 周：FastOpen、BBR、或自选专题

## 5. Linux 相比鸿蒙的额外复杂度提示

- `sk_buff` 操作极复杂（GRO/GSO/TSO、frag list、shared info）
- 锁机制更复杂：`lock_sock()`、RCU、qdisc lock 等
- 大量 sysctl 参数：`/proc/sys/net/ipv4/tcp_*`
- BPF 深度集成：`tcp_bpf.c`、`net/core/filter.c`
- NAPI 收包模型：`net/core/dev.c` 中的 `netif_receive_skb()` + `NET_RX_SOFTIRQ`

---

## 6. 学习进度

- [x] 已选择从 **ARP** 开始深入
- [x] 已整理详细学习文档：`docs/arp-study.md`
  - ARP 协议基础与报文格式
  - Linux `net/ipv4/arp.c` 收发流程
  - `arp_process()` 逐行精讲
  - `net/core/neighbour.c` 邻居子系统深度解析
  - ARP 请求触发、排队、超时完整链路
  - NUD 状态机与时序图
  - **已修正 4.3 / 15.2 节 NUD 状态机图**：NONE 不会直接变 REACHABLE，必须先进入 INCOMPLETE
- [x] 已整理详细学习文档：`docs/icmp-study.md`
- [x] 已整理详细学习文档：`docs/ip-study.md`
- [x] 已整理详细学习文档：`docs/tcp-study.md`
- [x] 已整理详细学习文档：`docs/udp-study.md`
- [x] **已完成 Linux 网络基础设施阶段**：整理 `docs/network-framework-study.md`
  - 该文档**合并**了原计划 §7.1（sk_buff）、§7.2（Socket 层）、§7.3（NAPI/软中断）三部分，并额外覆盖了**协议注册与分发表**（packet_type + ptype_base / net_protocol + inet_protos）
  - 四大核心抽象：Socket 抽象层 / 网络设备层 / 协议注册与分发表层 / 报文抽象层
  - **已通过完整 review 并修复全部 P0/P1/P2 问题**（详见第 7 节）
  - 已提交：commit `7b710e503cf9`，已 push 到 master
- [ ] 下一步：进入**数据通路深潜**阶段（见第 8 节，首篇 `docs/datapath-study.md`）

---

## 7. 已完成阶段回顾：Linux 网络基础设施（sk_buff / Socket 层 / NAPI·软中断 / 协议注册）

> 本阶段原计划分三篇文档（skbuff / socket / napi-softirq），实际整合为一篇 `docs/network-framework-study.md`，并额外补入了「协议注册与分发表」。文档已通过完整 review。

### 7.1 文档内容概览

| 章节 | 主题 |
|------|------|
| §1 | Linux Socket 抽象：`struct socket` / `struct sock` / `proto_ops` / `proto`，socket 创建流程，IPv4 协议注册（`inet_family_ops` / `inetsw`），`sock_init_data()`，sk_buff 与 socket 内存归属（`skb_set_owner_w/r` / `sock_wfree`） |
| §2 | net_device 与 NAPI：`struct net_device` / `net_device_ops` / `header_ops`，`packet_type` L2 分发器，`napi_struct`，NAPI 收包流程，接收路径函数层次（`netif_receive_skb` → `__netif_receive_skb_core`），GRO，发送路径 `dev_queue_xmit` |
| §3 | 协议注册与分发表：`packet_type` + `ptype_base`（L2→L3），`net_protocol` + `inet_protos`（L3→L4），`inet_add_protocol`，`ip_local_deliver` → `ip_protocol_deliver_rcu`，`inet_init()` 整体初始化 |
| §4~§13 | `struct sk_buff` 详解：内存布局（head/data/tail/end + `skb_shared_info`），分配释放（`alloc_skb` / `build_skb` / `kfree_skb` / `consume_skb`），数据区操作（push/pull/put/trim/reserve），clone/copy/可写性，队列，校验和卸载，GSO/TSO/UFO，收发路径典型流转 |

### 7.2 Review 修复记录

文档经过一轮完整 review，对照 v7.1-rc7 源码逐项校验，共修复 9 类问题：

| 级别 | 问题 | 修复要点 |
|------|------|----------|
| 🔴 P0 | §1.7 `sock_wfree` 简化过度，丢失关键设计意图 | 贴完整源码，讲清三条路径（TCP 快速 / 通用快速 / 通用慢速），解释「先减 `len-1` 再 `len=1`」防 use-after-free 的设计 |
| 🔴 P0 | §1.7 `ooo_okay` 解释错误 | 修正为 `netdev_pick_tx()` 的提示位；解释 `SK_WMEM_ALLOC_BIAS=1` 是 socket 自身引用，`old_wmem==1` 表示无在途 skb |
| 🔴 P0 | §2.6/§2.7 `netif_receive_skb` 调用链层次混乱 | 重写为四层调用链，明确 `netif_receive_skb` 非 NAPI 热路径、`pt_prev` 攒着最后调用的优化、`INDIRECT_CALL_INET` |
| 🟡 P1 | §1.2 `struct sock` 字段表与代码片段不对应 | 加「来源」列，标注 `sk_state`/`sk_refcnt`/`sk_rmem_alloc` 是宏别名，补行号 |
| 🟡 P1 | `ptype_base`/`ptype_specific` 的 per-net/per-dev 关系没讲清 | 补 `ptype_head()` 完整源码 + 三类链表作用域表 + 分流规则 |
| 🟡 P1 | `kfree_skb` vs `consume_skb` 对比过度简化 | 贴归一路径 `kfree_skb_reason` → `sk_skb_reason_drop`，强调 `skb_unref` 先减引用、归零才释放/trace |
| 🟢 P2 | sk_buff 部分二级章节号全错（2.1/3.1/4.1...） | 修正 19 个子章节号对齐父章节 |
| 🟢 P2 | `net_device` 代码片段 `features` 重复声明 | 加注释说明为按热路径分组的教学性摘录 |
| 🟢 P2 | 多处代码片段简化但未标注 | §2.7 `__netif_receive_skb_core`、§3.6 `inet_init` proto_register 加简化标注 |

**校验结果**：15 个顶层 `##` 章节连续，48 个 `###` 子章节全部与父章节匹配；18 处源码行号引用逐一对照 v7.1-rc7 全部一致。

### 7.3 验收标准达成情况

- [x] 能独立画出 `sk_buff` 从分配到释放的完整生命周期图（§7~§9）
- [x] 能解释 `skb_clone()` 与 `skb_copy()` 的使用场景差异（§9.1/§9.2）
- [x] 能 trace 一次 `sendmsg()` 从用户态到 `tcp_sendmsg()` / `udp_sendmsg()` 的完整调用链（§1.8 + §13.2）
- [x] 能 trace 一次网卡收包从硬中断到 `ip_rcv()` / `tcp_v4_rcv()` 的完整调用链（§2.5 + §3.8）
- [x] 能解释 NAPI 相比传统中断收包的优势，以及 budget/weight 的作用（§2.4/§2.5）
- [x] 产出学习文档：`docs/network-framework-study.md`（合并版，覆盖原计划三篇 + 协议注册）

---

## 8. 下一学习阶段目标：数据通路深潜 + 路由子系统

> 前置阶段已完成"框架骨架"（socket / net_device / sk_buff / 协议分发表）。下一阶段目标：把这些骨架串成完整的数据通路，并补上路由子系统。
>
> **总体判断**：当前 5 篇协议笔记（ARP/IP/ICMP/TCP/UDP）是"单点知识"，framework 笔记是"骨架"，但骨架之间的"数据流"还没亲手追过一遍。最高 ROI 的下一步是 **数据通路深潜**——追一个包的完整生命周期，这会让前面所有笔记"激活"。

### 8.1 数据通路深潜：RX/TX 完整生命周期 ⭐ 首选

**为什么先做这个**：framework 文档里 §2.5/§3.8/§13 已经画了 RX/TX 的概览调用链，但只是"地图"，没有逐函数深入。亲手追一遍后，IP/TCP/UDP 笔记里那些孤立的函数会自动连成通路，理解会产生质变。

**学习目标**：
- 亲手追一个包的完整生命周期，画两张完整调用链图（逐函数，含 `skb_push`/`skb_pull` 头部增删时机）
- **RX 路径**：NIC IRQ → NAPI poll → `net_rx_action` → `__netif_receive_skb_core` → `ip_rcv` → `ip_rcv_core`/`ip_rcv_finish` → `ip_local_deliver` → `ip_local_deliver_finish` → `ip_protocol_deliver_rcu` → `tcp_v4_rcv` → `tcp_v4_do_rcv` → `tcp_rcv_established` → socket 接收队列 → 唤醒 `recvmsg`
- **TX 路径**：`tcp_sendmsg` → `tcp_sendmsg_locked`（构造 skb 入发送队列）→ `tcp_write_xmit`（拥塞/窗口判断）→ `tcp_transmit_skb` → `ip_queue_xmit` → `ip_local_out` → `ip_output` → `ip_finish_output`（分片/iptables POSTROUTING）→ `ip_finish_output2`（查 neighbour）→ `neigh_output` → `dev_queue_xmit` → `__dev_queue_xmit`（qdisc）→ `dev_hard_start_xmit` → `ndo_start_xmit` → NIC
- 重点理解每一层的 `skb_push`/`skb_pull`/`skb_reserve` 时机，以及 `skb->protocol`、各 header offset 的设置点

**重点文件**：
- `net/ipv4/tcp_input.c`（接收 fast path / slow path）
- `net/ipv4/tcp_output.c`（发送路径）
- `net/ipv4/ip_output.c`、`net/ipv4/ip_input.c`
- 复用 `net/core/dev.c`（framework 已覆盖，此处聚焦与 IP 层的衔接）

**输出物**：`docs/datapath-study.md`（建议结构：RX 全链路逐函数 / TX 全链路逐函数 / 两张完整调用链图 / 头部增删时序表）

**验收标准**：
- [ ] 能不看文档画出 RX 从硬中断到 `tcp_rcv_established` 的完整调用链
- [ ] 能不看文档画出 TX 从 `tcp_sendmsg` 到 `ndo_start_xmit` 的完整调用链
- [ ] 能说清每一层在哪里 `skb_push` 加头、哪里 `skb_pull` 剥头

### 8.2 路由子系统

**为什么放在数据通路之后**：TX 路径里 `ip_queue_xmit` → `ip_route_output_flow` 是必经节点，RX 路径里 `ip_rcv_finish` 的 `dst.input` 路由判定也是核心。先追完数据通路会发现路由是绕不开的瓶颈，这时再学路由子系统目标最明确。

**学习目标**：
- `struct rtable`、`ip_route_output_flow`、`ip_route_input_slow`
- FIB：`net/ipv4/fib_*.c`、`fib_trie`（Linux 用 trie 而非 hash 存路由）
- `struct flowi4` —— 路由查找的"查询键"
- 路由 cache（`rt_cache`）与 FIB 的关系、`dst_entry` 的 role
- 策略路由（`fib_rules`）、multipath

**重点文件**：
- `net/ipv4/route.c`、`net/ipv4/fib_frontend.c`、`net/ipv4/fib_trie.c`、`net/ipv4/fib_rules.c`

**输出物**：`docs/routing-study.md`

### 8.3 后续可选专题（按需推进，优先级排序）

| 优先级 | 专题 | 关键点 | 何时做 |
|--------|------|--------|--------|
| 高 | Netfilter + eBPF/XDP | 5 个 hook 点、conntrack、XDP early drop、`BPF_PROG_TYPE_XDP` | 数据通路 + 路由后，理解 hook 点位置后学最顺 |
| 中 | 性能与内存机制 | NAPI 深入、GRO/GSO/TSO 细节、socket 内存压力反压（`sk_mem_schedule`）、qdisc/TC | framework 已涉猎，可随时深挖 |
| 中 | TCP 拥塞控制模块化 | `tcp_cong.c` 框架 + `tcp_cubic.c`/`tcp_bbr.c` | TCP 笔记讲的是通用层，这里才是算法内核 |
| 中 | IPv6 | `net/ipv6/` 整套（`ip6_rcv`/`ip6_route_output`/`tcp_v6_*`） | 与 IPv4 平行，生产不可回避 |
| 低 | 虚拟化与容器网络 | netns、bridge/veth/macvlan、VXLAN/GRE、tun/tap、virtio-net | 偏运维/云原生，按工作需要推进 |

### 8.4 建议的近期行动

1. **立即**：开 `docs/datapath-study.md`，先画 RX/TX 两张概览图（复用 framework §2.5/§3.8/§13 的内容作为起点）
2. **第 1 周**：逐函数追 TX 路径（从 `tcp_sendmsg` 到 NIC），因为 TX 比 RX 线性、好追
3. **第 2 周**：逐函数追 RX 路径（从 NIC 到 `tcp_rcv_established`），RX 分支多、需配合 NAPI 理解
4. **第 3 周**：进入路由子系统 `docs/routing-study.md`

---

## 9. 从"读懂源码"到"Linux 网络高手"：完整学习路线

> 本节是对前 8 节学习成果的阶段性复盘，并给出从"协议单点知识已齐 + 骨架已立 + 调用链已串通"到"能改、能调、能定位线上问题"的完整路线。判定依据：现有 8 篇笔记覆盖了 L2 框架 / L2.5 邻居 / L3 IPv4+路由 / L3.5 ICMP / L4 TCP+UDP / RX·TX 调用链，但**几乎不碰 Netfilter/eBPF/XDP、性能与可观测性、TCP 拥塞控制算法本体**——这三块是从"读"到"用"的关键距离。

### 9.1 当前水平评估

| 层 | 已覆盖 | 深度 |
|----|--------|------|
| L2 框架 | socket / net_device / sk_buff / NAPI / 协议分发表 | 深（2590 行） |
| L2.5 | ARP + neighbour 子系统 | 深（2070 行） |
| L3 | IPv4 收发/分片/转发 | 中深 |
| L3 | 路由子系统（FIB/LC-trie/策略路由/multipath） | 中深 |
| L3.5 | ICMP | 中深 |
| L4 | TCP 状态机/收发/拥塞/定时器 | 中（1253 行，单篇） |
| L4 | UDP | 中 |
| 横向 | RX/TX 完整调用链 | 中（686 行概览级） |

**瓶颈诊断**：再往协议细节里堆笔记边际收益已低。真正缺的是：① Netfilter/eBPF/XDP（生产网络的实际控制面）；② 性能与可观测性工具链（从"懂"到"能定位"）；③ TCP 拥塞控制算法本体与内部数据结构（从"通用层"到"能调优"）。

### 9.2 学习路线（按 ROI 排序）

#### 第一梯队（必做，决定能否从"读"到"用"）

**① Netfilter + conntrack + nftables**（约 1~2 周）
- 理由：TX/RX 笔记里 6 处出现 `NF_INET_*` hook 但都跳过，这是"债"；Cilium/Calico/iptables/防火墙/NAT 全靠这层。
- 重点：`struct nf_hook_ops` 注册机制、5 hook 点与调用时机、conntrack 状态机、NAT 的 `nf_nat` 模块、nftables 替代 iptables 的原因。
- 输出：`docs/netfilter-study.md`（**骨架已建，待逐节填充**，见第 10 节）
- 验收：能画出包流经 5 个 hook 的完整图；能解释 `iptables -t nat -A POSTROUTING -j MASQUERADE` 在内核里改了什么。

**② eBPF + XDP**（约 2~3 周）⭐ 2026 年最值得投入的方向
- 理由：现代 Linux 网络高手的"新护城河"。XDP 在驱动 RX 软中断之前就 drop/redirect/修改包，性能比 iptables 高一个数量级；Cilium、Katran、Cloudflare 都靠它。
- 重点：`BPF_PROG_TYPE_XDP`/`BPF_PROG_TYPE_SCHED_CLS`、`bpf_redirect`/`bpf_xdp_adjust_meta`、XDP 4 种 action（PASS/DROP/REDIRECT/ABORTED）、AF_XDP 零拷贝、TC BPF。
- 输出：`docs/ebpf-xdp-study.md`
- 验收：能写一个 XDP 程序按 IP 黑名单 drop 包；能解释 XDP 与 TC BPF 的挂载层次差异。

**③ TCP 拥塞控制深潜 + `tcp_sock` 内部数据结构**（约 2 周）
- 理由：TCP 笔记停在"通用框架"。真正的高手能讲清发送队列组织（`sk_write_queue` vs `tcp_rtx_queue` rbtree vs scoreboard）、Cubic 三次曲线、BBR 的 `BtlBw`/`RTprop` 估计、RACK（RFC 8985）丢包检测。
- 重点：`net/ipv4/tcp_cong.c` 框架、`tcp_cubic.c`、`tcp_bbr.c`、`tcp_rate.c`、`tcp_rack.c`、`tcp_sock` 内部 rbtree。
- 输出：`docs/tcp-congestion-deepdive.md`
- 验收：能解释 BBR 为何在长肥管道上打败 Cubic；能看懂 `ss -i` 输出里 `cwnd`/`ssthresh`/`bbr`/`rto` 的含义。

#### 第二梯队（构建系统能力，2~3 个月内逐步推进）

**④ 性能与可观测性工具链**（持续穿插，不单独成篇也要会）
- `perf trace`/`perf record` 抓网络热路径；`bpftrace` 一行脚本追 `tcp_sendmsg` 延迟分布；`dropwatch -l kas` 定位丢包点；`ss -tin` 看 TCP 内部状态；`nstat -az | grep -i tcp` 看 SNMP 计数。
- 建议做法：每学一个新子系统，配套写一段"如何用工具观测它"的附录。

**⑤ qdisc / TC / GRO·GSO·TSO 深度**
- 重点：`sch_fq_codel`/`sch_fq`（BBR 配套）、`htb`/`tbf` 限速、`mq` 多队列、TC class/filter 体系、GSO 在 `__dev_queue_xmit` 的分段时机、TSO 与 `tcp_tso_autosize` 的关系。
- 输出：`docs/qdisc-tc-study.md`

**⑥ socket 内存管理与反压**（`sk_mem_schedule` / `__sk_mem_raise_allocated`）
- 重点：`sk_wmem_alloc`/`sk_rmem_alloc`、`tcp_memory_allocated` 全局账本、`tcp_memory_pressure` 反压、`SO_SNDBUF`/`SO_RCVBUF` 与 `sysctl_tcp_mem` 的关系。
- 这是"高并发场景下卡住 90% 人"的细节，建议单独成篇。
- 输出：`docs/socket-memory-study.md`

#### 第三梯队（扩展广度，按工作需要推进）

| 优先级 | 专题 | 一句话理由 |
|--------|------|-----------|
| 中 | IPv6 全栈 | 与 IPv4 平行，生产不可回避；学到时只需对位差异 |
| 中 | Netlink 协议族 | `rtnetlink`/`genetlink` 是用户态配网络的唯一正道，写过协议栈的人必学 |
| 中 | netns + veth/bridge/macvlan | 理解容器网络的基础，Cilium/Calico 都建在这上面 |
| 低 | tun/tap、VXLAN/GRE、virtio-net | 云原生/虚拟化方向才深挖 |
| 低 | DCCP/SCTP/MPTCP | MPTCP 已进主线，有研究价值 |

### 9.3 建议的近期行动

1. **立即**：填充 `docs/netfilter-study.md`（骨架已建）——优先 §4 `nf_hook_slow` 完整源码、§5 `nf_register_net_hook` 完整源码、§7 conntrack 状态机图，这是理解后续 eBPF/XDP 的对照基线。
2. **第 1~2 周**：完成 Netfilter 笔记，配套用 `iptables -t nat -L -v` / `conntrack -L` / `perf trace -e 'net:*nf*'` 观测一次真实 NAT 流量。
3. **第 3~5 周**：进入 eBPF/XDP，先写一个 XDP drop 黑名单小程序跑通工具链（`clang -target bpf` + `ip link set dev x dp obj`），再深入源码。
4. **第 6~7 周**：TCP 拥塞控制深潜，配合 `ss -i` 与 `tcp_probe` tracepoint 观察一次 BBR 与 Cubic 的 cwnd 变化。
5. **持续**：qdisc/socket 内存/可观测性按需穿插。

### 9.4 验收里程碑（"高手"自检清单）

- [ ] 能不看文档画出 RX/TX 从 syscall 到 NIC 的完整调用链（含 5 个 Netfilter hook 位置）
- [ ] 能用 `bpftrace`/`perf` 一行命令定位"某个连接为什么慢/为什么丢包"
- [ ] 能解释 BBR 与 Cubic 在长肥管道上的行为差异，并会调 `sysctl` 切换
- [ ] 能读懂 Cilium 一个 `NetworkPolicy` CR 最终生成了哪些 eBPF 程序、挂在哪些 hook
- [ ] 能解释 `iptables -t nat -A POSTROUTING -j MASQUERADE` 在 conntrack/NAT 层面改了哪些字段
- [ ] 能写出 socket 内存反压触发的完整路径（`sk_wmem_alloc` → `tcp_memory_pressure` → 应用 `EAGAIN`）

---

## 10. 文档状态索引

| 文档 | 状态 | 行数 | 最近更新 |
|------|------|------|----------|
| `docs/arp-study.md` | ✅ 完成 | 2070 | 2026-06-14 |
| `docs/icmp-study.md` | ✅ 完成 | 1295 | 2026-06-20 |
| `docs/ip-study.md` | ✅ 完成 | 1415 | 2026-06-20 |
| `docs/tcp-study.md` | ✅ 完成 | 1253 | 2026-06-20 |
| `docs/udp-study.md` | ✅ 完成 | 939 | 2026-06-20 |
| `docs/network-framework-study.md` | ✅ 完成 | 2590 | 2026-06-21 |
| `docs/network-send-recv-callchain.md` | ✅ 完成 | 686 | 2026-06-21 |
| `docs/route-study.md` | ✅ 完成 | 1233 | 2026-06-22 |
| `docs/netfilter-study.md` | ✅ 核心章节 + 扩展专题完成（2906 行，9 节源码深潜） | 2906 | 2026-06-22 |
| `docs/datapath-study.md` | ⏳ 待开（第 8.1 节） | — | — |
| `docs/ebpf-xdp-study.md` | ⏳ 待开（第 9.2 ②） | — | — |
| `docs/tcp-congestion-deepdive.md` | ⏳ 待开（第 9.2 ③） | — | — |
| `docs/qdisc-tc-study.md` | ⏳ 待开（第 9.2 ⑤） | — | — |
| `docs/socket-memory-study.md` | ⏳ 待开（第 9.2 ⑥） | — | — |

