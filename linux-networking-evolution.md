# Linux Networking Change Log --- LWN / Upstream Cross-Reference

**対象期間:** 2019-05-07 ～ 2026-10-02\
**対象:** Linux networking
のアーキテクチャ、API、性能、protocol、BPF/XDP、netfilter、virtual
networking、zero-copy、device-memory など\
**主な情報源:** LWN.net Kernel / merge-window coverage、Linux kernel
documentation、upstream patch/commit history

> **調査方針**
>
> -   個別 NIC ドライバの通常の追加・bug fix は原則除外する。
> -   networking stack/API/architecture
>     の変化として後続開発につながる変更を優先する。
> -   LWN の独立記事だけでなく merge-window coverage も対象にする。
> -   patch series は全 revision
>     を機械的に列挙せず、初版・重要な設計変更・merge 直前版・mainline
>     merge を追う。
> -   commit hash は一次情報で確認できたものだけを記載する。未確認の
>     hash は推測しない。
> -   `Kernel` は原則として最初に mainline に入った release を示す。

------------------------------------------------------------------------

## 1. Executive timeline

Linux networking の 2019～2026
年の変化は、大きく次の流れとして読むことができる。

``` text
高速 NIC に対する packet processing / memory allocation の最適化
                         │
                         ▼
                 XDP / BPF programmability
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       packet 数を減らす         copy を減らす
       GRO/GSO → BIG TCP        zero-copy / io_uring
             │                       │
             ▼                       ▼
       AF_XDP multi-buffer      page_pool / netmem
             │                       │
             └───────────┬───────────┘
                         ▼
               netkit / Device Memory TCP
                         │
                         ▼
          host stack / host RAM を極力通さない
```

------------------------------------------------------------------------

# 2. Release chronology

## 2019

### Linux 5.2

**Tags:** `netdev`, `XDP`, `memory-management`, `high-speed-networking`

2019 年時点ですでに 100～400Gb/s 級 NIC では packet processing
そのものだけでなく、RX/TX に伴う page allocation/recycling
が重要なボトルネックになっていた。この系列は後の
`page_pool`、`netmem`、Device Memory TCP へつながる。

-   LWN: [The first half of the 5.2 merge
    window](https://lwn.net/Articles/787963/)
-   LWN: [The rest of the 5.2 merge
    window](https://lwn.net/Articles/788532/)
-   LWN feature: *Memory management for 400Gb/s interfaces*（LWN Kernel
    Index から参照）

### Linux 5.3

**Tags:** `BPF`, `socket`, `TCP`, `IPv4`

主な流れは socket/networking への BPF hook 拡大。BPF が単なる packet
filter/XDP program ではなく、socket behavior や TCP stack を
programmable にする方向へ進む。

### Linux 5.4

**Tags:** `BPF`, `XDP`, `TCP`, `SYN-cookie`

XDP/TC datapath と TCP SYN-cookie 処理の接続が進む。高負荷時の L4
protection/load-balancing を BPF datapath 側へ寄せる流れの一部。

------------------------------------------------------------------------

## 2020

### Linux 5.6 --- BPF `struct_ops` / WireGuard

**Tags:** `BPF`, `TCP`, `congestion-control`, `WireGuard`,
`ethtool-netlink`

`BPF_PROG_TYPE_STRUCT_OPS` が入り、BPF program が kernel 内部の
operations structure を実装できるようになった。最初の主要ユースケースが
`tcp_congestion_ops` であり、TCP congestion-control algorithm を BPF
で実装可能になった。

``` text
従来:
kernel built-in/module
        │
        ▼
tcp_congestion_ops

5.6:
BPF program
        │
        ▼
BPF struct_ops
        │
        ▼
tcp_congestion_ops
```

同 release では WireGuard が mainline 化され、ethtool の ioctl → netlink
移行も大きく進んだ。

-   LWN: [The 5.6 merge window opens](https://lwn.net/Articles/810780/)
-   LWN: [The rest of the 5.6 merge
    window](https://lwn.net/Articles/811230/)
-   LWN feature: [Kernel operations structures in
    BPF](https://lwn.net/Articles/811631/)
-   LWN: [The 5.6 kernel has been
    released](https://lwn.net/Articles/816213/)

### Linux 5.7--5.8

**Tags:** `XDP`, `TC`, `tunnel`, `bridge`, `offload`

この時期は XDP buffer handling、TC hardware offload、bridge/tunnel
周辺の infrastructure が継続的に拡張された。後の AF_XDP multi-buffer や
BIG TCP と合わせて読むとよい。

### Linux 5.9 --- BPF socket lookup

**Tags:** `BPF`, `socket`, `TCP`, `UDP`

BPF が TCP/UDP socket lookup に介入できる方向へ進展。container/service
datapath を iptables/nftables の NAT
だけに依存せず実装する基礎の一つとなる。

### Linux 5.10 --- MPTCP / BPF TCP options

**Tags:** `MPTCP`, `BPF`, `TCP-options`

MPTCP の mainline implementation が実用機能を増やし、BPF と TCP option
processing の接点も増加した。

------------------------------------------------------------------------

# 3. 2021--2022: zero-copy と BIG TCP

## Linux 5.11 --- TCP zero-copy receive

**Tags:** `TCP`, `zero-copy`, `performance`

TCP receive path で user-space への不要な copy
を削減する系列が進む。このテーマは後に io_uring zero-copy RX と Device
Memory TCP に発展する。

## Linux 5.14--5.18

**Tags:** `MPTCP`, `routing`, `TC`, `offload`, `BPF`

MPTCP、routing、TC offload、BPF integration が継続的に改善された時期。

## Linux 5.19 --- BIG TCP

**Tags:** `TCP`, `IPv6`, `GRO`, `GSO`, `performance`, `BIG-TCP`

### Motivation

高速 datapath では wire 上の MTU よりも、kernel 内部で一度に処理できる
packet aggregate の大きさが CPU overhead に強く影響する。

``` text
wire packets
 1500 B
 1500 B     GRO
 1500 B  ───────► large skb
 1500 B

              従来の上限 ≒ 64 KiB
```

BIG TCP は IPv6 jumbogram infrastructure を利用して、kernel 内部の
TCP/GRO/GSO packet を 64KiB より大きく扱えるようにした。

``` text
               BIG TCP

many wire packets
       │
       ▼
  >64 KiB GRO/GSO aggregate
       │
       ▼
per-packet CPU overhead ↓
```

### Mainline

Linux 5.19 merge window で BIG TCP patch set が mainline 化されたことを
LWN が明記している。

### References

-   LWN: [5.19 Merge window, part 1](https://lwn.net/Articles/896140/)
-   LWN feature: [Going big with TCP
    packets](https://lwn.net/Articles/884104/)
-   LWN: [Linux 5.19 release status](https://lwn.net/Articles/903696/)

### Follow-ups

BIG TCP
は単独の機能ではなく、後の以下の変更と組み合わせて見るべきである。

``` text
BIG TCP
   │
   ├── AF_XDP multi-buffer (6.6)
   │
   ├── page_pool / netmem improvements
   │
   └── VXLAN / GENEVE 対応（7.x 系列）
```

------------------------------------------------------------------------

# 4. 2023: AF_XDP multi-buffer と netkit

## Linux 6.6 --- AF_XDP multi-buffer

**Tags:** `AF_XDP`, `XDP`, `multi-buffer`, `MPTCP`, `BPF`

AF_XDP が複数 buffer にまたがる packet を扱えるようになった。

これは jumbo packet や巨大な internal packet representation と AF_XDP
を組み合わせるうえで重要である。

同 release では以下も入った。

-   BPF packet-defragmentation hook
-   `update_socket_protocol` BPF hook
-   TCP request を MPTCP に切り替える用途
-   MPTCP subflow routing の BPF support

**References**

-   LWN: [The first half of the 6.6 merge
    window](https://lwn.net/Articles/942954/)

## Linux 6.7 --- netkit

**Tags:** `BPF`, `netkit`, `container`, `virtual-networking`

netkit は BPF-programmable な virtual network device として導入された。

概念的には、

``` text
traditional

container
   │
  veth
   │
host networking processing
   │
 physical NIC
```

に対して、

``` text
netkit / BPF

container / VM
      │
      ▼
    netkit
      │
     BPF
      │
      ▼
 physical NIC
```

のように datapath を BPF 中心に構築しやすくする。

**References**

-   LWN feature: [The BPF-programmable network
    device](https://lwn.net/Articles/949960/)
-   LWN: [The first half of the 6.7 merge
    window](https://lwn.net/Articles/949294/)
-   LWN: [Linux 6.7 release status](https://lwn.net/Articles/957389/)

------------------------------------------------------------------------

# 5. 2024: Device Memory TCP

## Linux 6.12 --- Device Memory TCP RX

**Tags:** `TCP`, `zero-copy`, `DMA-BUF`, `page_pool`, `device-memory`,
`GPU`

### Problem

通常の receive path では、device が最終 consumer であっても system RAM
が中継点になりやすい。

``` text
NIC
 │ DMA
 ▼
system RAM
 │
 │ copy / mapping / CPU processing
 ▼
GPU / accelerator
```

Device Memory TCP は、TCP payload を DMA-BUF-backed device memory
に直接配置するための infrastructure を導入する。

``` text
NIC
 │
 │ DMA
 ▼
device memory
 │
 ▼
GPU / accelerator
```

### Mainline

LWN の 6.12 merge-window coverage は Device Memory TCP patch set の
merge を明記している。

-   LWN: [The 6.12 merge window
    begins](https://lwn.net/Articles/990750/)
-   LWN: [The rest of the 6.12 merge
    window](https://lwn.net/Articles/991301/)
-   LWN: [Linux 6.12 release](https://lwn.net/Articles/997958/)
-   Kernel docs: [Device Memory
    TCP](https://docs.kernel.org/networking/devmem.html)

### Development history

Device Memory TCP は一回の patch submission で完成した機能ではない。2023
年から RFC/patch series が繰り返され、page-pool、netmem、DMA-BUF、TCP
receive API との integration が整理された後に mainline 化された。

change log では revision をすべて列挙するのではなく、

1.  initial RFC
2.  architecture/API が大きく変わった revision
3.  merge 前の最終系列
4.  mainline merge
5.  TX-side follow-up

を追跡対象とする。

------------------------------------------------------------------------

# 6. 2025: zero-copy RX / RTNL / AccECN / DIBS

## Linux 6.15 --- io_uring zero-copy RX

**Tags:** `io_uring`, `zero-copy`, `RX`, `TCP`, `RTNL`, `BPF`

Linux 6.15 では initial zero-copy receive support が io_uring
に追加された。

同時期の networking change:

-   RTNL ("big networking lock") 分割の継続
-   `TCP_RTO_MAX_MS`
-   networking stack 内の複数地点から timestamp を得る BPF callback

**References**

-   LWN: [The first part of the 6.15 merge
    window](https://lwn.net/Articles/1015414/)
-   LWN: [Linux 6.15 released](https://lwn.net/Articles/1022457/)

### 系譜

``` text
TCP zero-copy RX
       │
       ├──────────────┐
       ▼              ▼
io_uring ZC RX     page_pool/netmem
       │              │
       └──────┬───────┘
              ▼
      device-memory networking
```

## Linux 6.18 --- AccECN / UDP RX / DIBS

**Tags:** `TCP`, `AccECN`, `UDP`, `DIBS`, `socket-buffer`, `PSP`

LWN が networking section で挙げている主要変更:

-   Accurate ECN (AccECN)
-   UDP receive performance improvement
-   Direct Internal Buffer Sharing (DIBS)
-   default socket receive buffer size を 4MB に増加
-   TCP + PSP encryption

**Reference**

-   LWN: [6.18 merge window, part 1](https://lwn.net/Articles/1040203/)
-   LWN: [Linux 6.18 released](https://lwn.net/Articles/1048703/)

### UDP performance の注意

LWN の「47%」という数字は、あらゆる UDP workload が一律
47%高速化するという意味ではない。packet size、CPU、load、benchmark
method に依存する測定値として扱う必要がある。

### DIBS

DIBS は networking buffer ownership/sharing の overhead
を削減する系列として、page_pool/netmem と合わせて追う価値がある。

------------------------------------------------------------------------

# 7. 主要技術系列

## 7.1 GRO / GSO / BIG TCP

``` text
GRO/GSO
   │
   ▼
高速NICで packet-rate overhead が顕在化
   │
   ▼
BIG TCP (5.19)
   │
   ├── AF_XDP multi-buffer (6.6)
   │
   └── tunnel/overlay への拡張
```

**Primary tags:** `TCP`, `GRO`, `GSO`, `BIG-TCP`, `AF_XDP`

## 7.2 XDP / AF_XDP

``` text
XDP
 │
 ├── driver/native XDP
 ├── redirect
 ├── AF_XDP
 │     └── multi-buffer
 └── BPF-based virtual datapath
       └── netkit
```

## 7.3 BPF networking

``` text
packet filter
    │
    ▼
XDP / TC
    │
    ▼
socket hooks
    │
    ▼
struct_ops → TCP congestion control
    │
    ▼
socket lookup / MPTCP steering
    │
    ▼
netkit / qdisc / richer network-stack programmability
```

## 7.4 zero-copy / memory

``` text
skb/page allocation optimization
       │
       ▼
    page_pool
       │
       ▼
     netmem
       │
       ├── io_uring ZC RX/TX
       ├── DIBS
       └── Device Memory TCP
```

## 7.5 TCP evolution

主な追跡対象:

-   BPF congestion control (`struct_ops`)
-   BIG TCP
-   MPTCP
-   retransmission control API
-   AccECN
-   TCP authentication/encryption extensions

## 7.6 UDP

主な追跡対象:

-   GRO/GSO interaction
-   high packet-rate receive optimization
-   socket receive-buffer defaults
-   UDP-based transports
-   tunnel/overlay datapath

## 7.7 RTNL scalability

古典的な RTNL は networking configuration path の広範囲を serialize
する。

``` text
             global RTNL
                 │
                 ▼
           contention point
                 │
                 ▼
      smaller locking domains
                 │
                 ▼
 per-netns / finer-grained locking
```

大規模 container/network-namespace 環境ほど影響が大きい。

## 7.8 netfilter / nftables / conntrack

この系列では以下を別途 commit-level で追跡する。

-   nftables 1.0 前後
-   flowtable / hardware offload
-   bpfilter の初期案と再設計
-   conntrack scalability / observability
-   BPF と traditional netfilter datapath の役割分担

## 7.9 virtual networking / VM

追跡対象:

-   veth
-   TAP/TUN
-   virtio-net
-   XDP/AF_XDP
-   netkit
-   zero-copy receive
-   KubeVirt/VM datapath

------------------------------------------------------------------------

# 8. Release matrix

  -----------------------------------------------------------------------
  Kernel                              Networking topics to track
  ----------------------------------- -----------------------------------
  5.2                                 high-speed NIC memory management,
                                      XDP

  5.3                                 socket/cgroup BPF, TCP hooks

  5.4                                 XDP/TC, SYN-cookie/BPF

  5.5                                 virtual/socket/network-device API

  5.6                                 WireGuard, BPF struct_ops, TCP CC,
                                      ethtool-netlink

  5.7                                 bareudp, encapsulation/offload

  5.8                                 XDP buffer API, TC/bridge

  5.9                                 BPF socket lookup/iterator

  5.10                                MPTCP, BPF/TCP options

  5.11                                TCP zero-copy receive

  5.12                                MPTCP/multicast

  5.13                                MPTCP/BPF/netdev

  5.14                                routing, SO_REUSEPORT

  5.15                                IPv6 IOAM, bridge multicast

  5.16                                socket memory, IOAM

  5.17                                TC hardware offload

  5.18                                BPF/MPTCP/netdev

  **5.19**                            **BIG TCP, skb drop reasons, MPTCP
                                      API**

  6.0                                 BPF/netdev continuation

  6.1                                 netlink/API modernization

  6.2                                 BPF/netdev

  6.3                                 netlink specification / Ethernet

  6.4                                 XDP/BPF

  6.5                                 socket/process API

  **6.6**                             **AF_XDP multi-buffer, BPF defrag,
                                      MPTCP BPF**

  **6.7**                             **netkit, io_uring networking**

  6.8                                 network-core cache/performance work

  6.9                                 RTNL reduction, BPF token

  6.10                                io_uring zero-copy send
                                      improvements

  6.11                                TCP/network tuning

  **6.12**                            **Device Memory TCP RX**

  6.13                                RTNL scalability / traffic shaping

  6.14                                TCP/UDP/IPsec changes

  **6.15**                            **io_uring ZC RX, RTNL breakup,
                                      TCP_RTO_MAX_MS, BPF timestamps**

  6.16                                device-memory / DMA / networking
                                      continuation

  6.17                                TCP loss-detection cleanup

  **6.18**                            **AccECN, UDP RX optimization,
                                      DIBS, rmem default 4MB**

  6.19+                               continued TCP/netdev/BPF
                                      scalability work

  7.x                                 BIG TCP/overlay,
                                      netkit/device-memory/BPF
                                      continuation
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 9. LWN reading list --- core articles

以下は、この期間の Linux networking
の技術史を追う「幹」として特に有用な記事群である。順位付けではなく、時系列の学習経路として並べている。

1.  *Memory management for 400Gb/s interfaces* --- high-speed networking
    と memory management
2.  [Kernel operations structures in
    BPF](https://lwn.net/Articles/811631/) --- BPF `struct_ops`
3.  *Zero-copy network transmission with io_uring* --- zero-copy TX
4.  [Going big with TCP packets](https://lwn.net/Articles/884104/) ---
    BIG TCP
5.  [5.19 Merge window, part 1](https://lwn.net/Articles/896140/) ---
    BIG TCP mainline
6.  [The first half of the 6.6 merge
    window](https://lwn.net/Articles/942954/) --- AF_XDP multi-buffer
7.  [The BPF-programmable network
    device](https://lwn.net/Articles/949960/) --- netkit
8.  [The 6.12 merge window begins](https://lwn.net/Articles/990750/) ---
    Device Memory TCP
9.  [The first part of the 6.15 merge
    window](https://lwn.net/Articles/1015414/) --- io_uring ZC RX / RTNL
10. [6.18 merge window, part 1](https://lwn.net/Articles/1040203/) ---
    AccECN / UDP / DIBS

------------------------------------------------------------------------

# 10. Tag index

## `BPF`

-   5.3 socket/cgroup hooks
-   5.4 XDP/TC integration
-   5.6 `struct_ops`
-   5.9 socket lookup
-   5.10 TCP options
-   6.6 MPTCP/defrag hooks
-   6.7 netkit
-   6.15 network timestamp callbacks

## `TCP`

-   zero-copy RX
-   BPF congestion control
-   BIG TCP
-   MPTCP interaction
-   Device Memory TCP
-   `TCP_RTO_MAX_MS`
-   AccECN

## `UDP`

-   GRO/GSO
-   high packet-rate receive optimization
-   receive-buffer sizing
-   UDP-based tunnel/transport work

## `XDP` / `AF_XDP`

-   XDP buffer infrastructure
-   AF_XDP
-   AF_XDP multi-buffer
-   netkit/BPF datapath

## `zero-copy`

-   TCP zero-copy
-   io_uring zero-copy TX
-   io_uring zero-copy RX
-   Device Memory TCP
-   DIBS

## `virtual-networking`

-   veth
-   tunnel/overlay
-   netkit
-   virtio-net/TAP
-   KubeVirt-oriented zero-copy work

------------------------------------------------------------------------

# 11. Commit-level research status

この文書では commit hash を「それらしい hash」で埋めず、upstream tree /
lore で照合できたものだけを確定情報として追加する。

優先して commit-level history を展開する系列:

1.  BIG TCP
2.  AF_XDP multi-buffer
3.  netkit
4.  Device Memory TCP RX
5.  Device Memory TCP TX
6.  io_uring zero-copy TX/RX
7.  page_pool / netmem / DIBS
8.  RTNL breakup
9.  AccECN
10. UDP receive optimization
11. BPF `struct_ops`
12. MPTCP BPF integration
13. nftables / flowtable / conntrack
14. virtio-net / VM networking

各系列は最終的に以下の形式にする。

``` text
Feature:
Kernel:
Subsystem:
Authors:

Motivation:
Architecture:

Initial RFC:
Important revisions:
Final patch series:

Merge commit:
Key commits:

Changed files:
Documentation:

LWN:
Lore:
git.kernel.org:

Follow-ups:
Tags:
```

------------------------------------------------------------------------

# 12. Sources / entry points

-   [LWN Kernel Index](https://lwn.net/Kernel/Index/)
-   [LWN 5.2 first-half merge window](https://lwn.net/Articles/787963/)
-   [LWN 5.6 merge window](https://lwn.net/Articles/810780/)
-   [LWN BPF struct_ops](https://lwn.net/Articles/811631/)
-   [LWN BIG TCP / 5.19](https://lwn.net/Articles/896140/)
-   [LWN 6.6 merge window](https://lwn.net/Articles/942954/)
-   [LWN netkit](https://lwn.net/Articles/949960/)
-   [LWN 6.12 / Device Memory TCP](https://lwn.net/Articles/990750/)
-   [LWN 6.15 merge window](https://lwn.net/Articles/1015414/)
-   [LWN 6.18 merge window](https://lwn.net/Articles/1040203/)
-   [Linux kernel networking
    documentation](https://docs.kernel.org/networking/)
-   [Linux Device Memory TCP
    documentation](https://docs.kernel.org/networking/devmem.html)
-   [Linux networking patch archive
    (lore)](https://lore.kernel.org/netdev/)
-   [Linux kernel
    git](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/)

------------------------------------------------------------------------

## Notes on completeness

この change log は「LWN の記事タイトル一覧」ではなく、2019-05-07 以降の
Linux networking
の重要な変更を技術系列として再構成することを目的としている。そのため、個別
driver の通常更新は除外する一方、独立した LWN feature がなくても
merge-window coverage に現れる architecture-level change は含める。

commit-level の項目は upstream で確認できたものから順次追加し、LWN
の記述だけから hash を逆算・推測しない。
