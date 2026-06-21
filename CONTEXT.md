# 对话上下文续接文件

> 最近更新：2026-06-21  
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
- [ ] 下一步：进入下一学习阶段（见第 8 节）

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

> 前置阶段已完成”框架骨架”（socket / net_device / sk_buff / 协议分发表）。下一阶段目标：把这些骨架串成完整的数据通路，并补上路由子系统。

### 8.1 数据通路深潜：RX/TX 完整生命周期

**学习目标**：
- 亲手追一个包的完整生命周期，画两张完整调用链图
- **RX 路径**：NIC IRQ → NAPI poll → `net_rx_action` → `__netif_receive_skb_core` → `ip_rcv` → `ip_local_deliver` → `tcp_v4_rcv` → socket 接收队列 → 唤醒 `recvmsg`
- **TX 路径**：`tcp_sendmsg` → `tcp_write_xmit` → `ip_queue_xmit` → `ip_output` → `ip_finish_output2`（查 neighbour）→ `dev_queue_xmit` → qdisc → `dev_hard_start_xmit` → NIC
- 重点理解各层的 `skb_push`/`skb_pull` 头部增删时机

**重点文件**：
- `net/ipv4/tcp_input.c`（接收 fast path / slow path）
- `net/ipv4/tcp_output.c`（发送路径）
- `net/ipv4/ip_output.c`、`net/ipv4/ip_input.c`

**输出物**：`docs/datapath-study.md`

### 8.2 路由子系统

**学习目标**：
- `struct rtable`、`ip_route_output_flow`
- FIB：`net/ipv4/fib_*.c`、`fib_trie`（Linux 用 trie 而非 hash 存路由）
- `struct flowi4` —— 路由查找的”查询键”
- 策略路由、multipath、路由 cache 与 FIB 的关系

**重点文件**：
- `net/ipv4/route.c`、`net/ipv4/fib_*.c`

**输出物**：`docs/routing-study.md`

### 8.3 后续可选专题（按需推进）

- Netfilter + eBPF/XDP（现代 Linux 网络：5 个 hook 点、conntrack、XDP early drop）
- 性能与内存机制（NAPI 深入、GRO/GSO/TSO、socket 内存压力反压、qdisc/TC）
- TCP 拥塞控制模块化（`tcp_cong.c` + cubic/bbr）
- IPv6（`net/ipv6/` 整套）
- 虚拟化与容器网络（netns、bridge/veth/macvlan、VXLAN/GRE、tun/tap、virtio-net）
