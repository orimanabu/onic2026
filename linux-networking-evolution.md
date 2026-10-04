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

  ---------------------------------------------------------------------
  Kernel                             Networking topics to track
  ---------------------------------- ----------------------------------
  5.2                                high-speed NIC memory management,
                                     XDP

  5.3                                socket/cgroup BPF, TCP hooks

  5.4                                XDP/TC, SYN-cookie/BPF

  5.5                                virtual/socket/network-device API

  5.6                                WireGuard, BPF struct_ops, TCP CC,
                                     ethtool-netlink

  5.7                                bareudp, encapsulation/offload

  5.8                                XDP buffer API, TC/bridge

  5.9                                BPF socket lookup/iterator

  5.10                               MPTCP, BPF/TCP options

  5.11                               TCP zero-copy receive

  5.12                               MPTCP/multicast

  5.13                               MPTCP/BPF/netdev

  5.14                               routing, SO_REUSEPORT

  5.15                               IPv6 IOAM, bridge multicast

  5.16                               socket memory, IOAM

  5.17                               TC hardware offload

  5.18                               BPF/MPTCP/netdev

  **5.19**                           **BIG TCP, skb drop reasons, MPTCP
                                     API**

  6.0                                BPF/netdev continuation

  6.1                                netlink/API modernization

  6.2                                BPF/netdev

  6.3                                netlink specification / Ethernet

  6.4                                XDP/BPF

  6.5                                socket/process API

  **6.6**                            **AF_XDP multi-buffer, BPF defrag,
                                     MPTCP BPF**

  **6.7**                            **netkit, io_uring networking**

  6.8                                network-core cache/performance
                                     work

  6.9                                RTNL reduction, BPF token

  6.10                               io_uring zero-copy send
                                     improvements

  6.11                               TCP/network tuning

  **6.12**                           **Device Memory TCP RX**

  6.13                               RTNL scalability / traffic shaping

  6.14                               TCP/UDP/IPsec changes

  **6.15**                           **io_uring ZC RX, RTNL breakup,
                                     TCP_RTO_MAX_MS, BPF timestamps**

  6.16                               device-memory / DMA / networking
                                     continuation

  6.17                               TCP loss-detection cleanup

  **6.18**                           **AccECN, UDP RX optimization,
                                     DIBS, rmem default 4MB**

  6.19+                              continued TCP/netdev/BPF
                                     scalability work

  7.x                                BIG TCP/overlay,
                                     netkit/device-memory/BPF
                                     continuation
  ---------------------------------------------------------------------

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
7.  page_pool / netmem / memory-provider; DIBS (separate shared-memory
    lineage)
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

------------------------------------------------------------------------

# 13. Commit-level history --- verified entries

この章では、upstream patch archive / git history で commit ID
まで確認できた系列を記録する。 短縮 hash
だけが一次資料に掲載されている場合は短縮形をそのまま記載し、推測で full
hash に展開しない。

## 13.1 BIG TCP --- Linux 5.19

**Feature:** BIG TCP (initial IPv6 support)\
**Kernel:** Linux 5.19\
**Subsystem:** `net/core`, IPv6, TCP, GRO/GSO\
**Primary motivation:** 64KiB を超える kernel-internal GRO/GSO aggregate
を利用し、高速 TCP datapath の per-packet overhead を削減する。

### Mainline evidence

5.19 の networking pull request は、IPv6 Jumbogram extension header
を利用して 64KiB より大きな TCPv6 GSO super-segment
をサポートする機能を明示的に `BIG TCP` として記載している。

-   Networking pull request:
    https://lists.openwall.net/netdev/2022/05/24/216
-   LWN feature: https://lwn.net/Articles/884104/
-   LWN 5.19 merge-window coverage: https://lwn.net/Articles/896140/

### Verified commits

後続の upstream/stable fix history から、少なくとも以下の initial BIG
TCP commits を 確実に逆参照できる。

-   `0fe79f28bfaf73b66b7b1562d2468f94aa03bd12`
    -   Linux 5.19 で導入された BIG TCP/GRO 系列の commit。
    -   後の GRO validation fix がこの commit を introduction point
        として明示している。
-   `7c4e983c4f3cf94fcd879730c6caa877e0768a4d`
    -   Linux 5.19 の BIG TCP 関連 commit。
    -   `skb_copy_ubufs()` と BIG TCP の interaction に対する後続 fix が
        introduction point として参照している。

> BIG TCP は複数 commit からなる系列である。この2つだけを「BIG TCP の全
> commit」と 解釈してはいけない。series 全体の exact commit list
> は引き続き upstream tree と patch archive を照合する。

### Later evolution

-   IPv4 BIG TCP
-   GRO validation / HBH handling の再設計
-   AF_XDP multi-buffer との整合
-   overlay/tunnel datapath への拡張

------------------------------------------------------------------------

## 13.2 AF_XDP multi-buffer --- Linux 6.6

**Feature:** AF_XDP multi-buffer RX/TX\
**Kernel:** Linux 6.6\
**Subsystem:** `net/xdp`, AF_XDP, Intel `ice` / `i40e` initial driver
support\
**Patch series:** `[PATCH v7 bpf-next 00/24] xsk: multi-buffer support`

### Development

v7 series は 24 patches から構成され、core AF_XDP support、zero-copy、
driver support、documentation/selftests をまとめて導入した。

Patch series:

-   https://lists.openwall.net/netdev/2023/07/19/282

### Verified core commits

  ----------------------------------------------------------------------------------------------------
  Commit                                       Role
  -------------------------------------------- -------------------------------------------------------
  `804627751b42`                               `xsk: add support for AF_XDP multi-buffer on Rx path`

  `b7f72a30e9ac`                               Tx multi-buffer 用 wrappers/helpers

  `1b725b0c8163`                               core/driver が EOP bit を確認する infrastructure

  `cf24f5a5feeaae34c1a34d1e04f8ac697290427a`   `xsk: add support for AF_XDP multi-buffer on Tx path`

  `07428da9e25a`                               Tx path で zero-length descriptors を破棄

  `13ce2daa259a`                               ZC maximum fragments 用 netlink attribute
  ----------------------------------------------------------------------------------------------------

Acceptance record:

-   https://lists.openwall.net/netdev/2023/07/19/358

### Architecture

``` text
従来 AF_XDP:

one packet
   │
   └── one XDP descriptor / buffer


multi-buffer:

one packet
   │
   ├── descriptor #1
   ├── descriptor #2
   ├── ...
   └── descriptor #N (EOP)
```

これにより jumbo frame や大きな packet representation を AF_XDP
で扱いやすくなる。

### Important files

-   `net/xdp/xsk.c`
-   `net/xdp/xsk_buff_pool.c`
-   `net/xdp/xsk_queue.h`
-   `include/net/xsk_buff_pool.h`
-   `Documentation/networking/af_xdp.rst`
-   `Documentation/netlink/specs/netdev.yaml`

### Follow-up / maintenance evidence

2026 年の修正でも `Fixes: cf24f5a5feea` が使われており、TX multi-buffer
introduction point を独立に確認できる。

------------------------------------------------------------------------

## 13.3 netkit --- Linux 6.7

**Feature:** BPF-programmable `netkit` virtual network device\
**Kernel:** Linux 6.7\
**Author:** Daniel Borkmann\
**Subsystem:** BPF / virtual networking / container datapath

### Development

merge 直前の series:

-   `[PATCH bpf-next v4 1/7] netkit, bpf: Add bpf programmable net device`
-   Date: 2023-10-24
-   Patch: https://lists.openwall.net/netdev/2023/10/24/365

### Verified mainline commit

`35dfaad7188cdc043fde31709c796f5a692ba2bd`

Subject:

``` text
netkit, bpf: Add bpf programmable net device
```

この commit の説明では、BPF program を driver の `xmit` routine
内で実行し、 Pod/container egress で BPF processing を packet source
に近づけること、 さらに物理 device へ直接 redirect する場合に per-CPU
backlog queue を経由しない ことが目的として説明されている。

### Datapath implication

``` text
veth-centric:

Pod
 │
veth
 │
host backlog / host-side processing
 │
BPF/TC
 │
NIC


netkit:

Pod
 │
netkit xmit
 │
BPF
 │
direct redirect
 │
NIC
```

### References

-   LWN: https://lwn.net/Articles/949960/
-   v4 patch: https://lists.openwall.net/netdev/2023/10/24/365
-   commit mirror:
    https://git.zx2c4.com/linux-rng/commit/?id=35dfaad7188cdc043fde31709c796f5a692ba2bd

### Follow-ups

2026 年には netkit queue leasing と io_uring zero-copy RX の integration
が進み、 network namespace 内の guest/VM datapath
にまで対象が広がっている。

------------------------------------------------------------------------

## 13.4 Device Memory TCP RX --- Linux 6.12

**Feature:** Device Memory TCP receive\
**Kernel:** Linux 6.12\
**Authors:** Mina Almasry, Willem de Bruijn et al.\
**Subsystem:** TCP / netdev / DMA-BUF / page-pool / device memory

### Mainline state

Linux kernel documentation は Device Memory TCP を、TCP socket
で受信した data を DMA-BUF-backed device memory
へ直接配置する機能として説明している。

-   Kernel documentation:
    https://kernel.org/doc/html/latest/networking/devmem.html
-   LWN 6.12 merge window: https://lwn.net/Articles/990750/

### Architecture

``` text
traditional device-to-device transfer

device A
   │
   ▼
host memory
   │ network
   ▼
host memory
   │
   ▼
device B


Device Memory TCP RX

NIC
 │ DMA
 ▼
DMA-BUF / device memory
 │
 ▼
accelerator / GPU / SSD-side consumer
```

### Source areas

Current upstream tree contains Device Memory TCP infrastructure
including:

-   `net/core/devmem.h`
-   networking device-memory support
-   page-pool/netmem integration
-   TCP receive-side APIs
-   `Documentation/networking/devmem.rst`

### Commit verification status

6.12 への feature merge 自体と source/documentation は確認済み。 ただし
RX series は多数の preparatory commits に分割されているため、
**individual commit list はまだ「series 全体として確定」していない**。
単一 commit を Device Memory TCP RX の introduction commit
と誤って表記しない。

------------------------------------------------------------------------

## 13.5 Device Memory TCP TX --- Linux 6.16

**Feature:** Device Memory TCP transmit\
**Kernel:** Linux 6.16\
**Subsystem:** TCP / DMA-BUF / device-memory / zero-copy TX

RX support は 6.12 に入ったが、TX support は review を分離して後続
series となった。 2025-05 時点で TX patch set は net-next に queue
され、6.16 cycle 向けとなった。

### Significance

``` text
RX (6.12)
network → NIC → device memory

TX (6.16)
device memory → NIC → network
```

これにより Device Memory TCP は device memory を network endpoint の
data buffer として双方向に利用する方向へ進んだ。

### Verification status

-   RX が 6.12、TX が 6.16 という release separation は確認済み。
-   TX series は多数 revision を経ている。
-   exact mainline commit list は次の commit-level pass で確定する。

------------------------------------------------------------------------

# 14. Verification rules used in this document

commit-level 情報は次の優先順位で検証する。

1.  `git.kernel.org` / upstream kernel git
2.  `lore.kernel.org` または同内容の netdev mailing-list archive
3.  kernel.org documentation
4.  LWN merge-window / feature article
5.  stable-tree `Fixes:` history（introduction commit の相互検証）

特に `Fixes:` tag は、後続 bug fix から introduction commit
を逆引きするために有用だが、 それだけで patch series 全体を代表する
commit とみなさない。

------------------------------------------------------------------------

# 15. Next commit-level passes

次に同じ方法で以下を追加する。

1.  `page_pool` → `netmem` → DIBS
2.  io_uring zero-copy TX → zero-copy RX
3.  RTNL breakup / per-netns locking
4.  AccECN
5.  UDP receive-path optimization
6.  BPF `struct_ops` / TCP congestion control
7.  MPTCP + BPF
8.  nftables / flowtable / conntrack
9.  virtio-net / TAP / VM networking
10. BIG TCP IPv4 / tunnel follow-ups

------------------------------------------------------------------------

# 16. Commit-level pass 2 --- memory providers, io_uring ZC RX, RTNL, AccECN, DIBS

## 16.1 `page_pool` → `netmem` abstraction

**Subsystem:** netdev memory management / XDP / page_pool\
**Role:** Device Memory TCP と io_uring zero-copy RX の共通基盤

`page_pool` は NIC RX datapath 向けに page/page-fragment を高速 recycle
する allocator である。 従来は `struct page` が network receive buffer
の基本単位だったが、device memory のように 通常の system-RAM page
ではない memory を network stack で扱うには、この前提が障害になる。

そのため network stack の buffer representation を `struct page`
から抽象化する `netmem` 系列が進められた。

重要な patch series の一つ:

-   `Split netmem from struct page`
-   page_pool allocation / DMA sync / recycle path を `netmem` に変換
-   XDP と `skb_frag` も `netmem` を扱う方向へ変更
-   mlx5 / hns3 / mvneta など initial users を変換

LWN archive: https://lwn.net/Articles/919663/

### Device Memory TCP との接続

Device Memory TCP RFC v5 は major change として明示的に:

-   "Abstract page from net stack" series の上へ rebase
-   `struct page` の代わりに新しい `netmem` type を利用
-   page_pool の device-memory support を `netmem` ベースに再設計

としている。

LWN archive: https://lwn.net/Articles/955571/

したがって系列は、

``` text
page_pool
    │
    ▼
struct page 前提
    │
    ▼
netmem abstraction
    │
    ├───────────────┐
    ▼               ▼
Device Memory TCP   io_uring ZC RX
```

と整理できる。

------------------------------------------------------------------------

## 16.2 io_uring zero-copy RX --- Linux 6.15

**Feature:** io_uring zero-copy network receive\
**Kernel:** Linux 6.15\
**Authors:** David Wei et al.（mainline に至る系列）\
**Subsystem:** io_uring / netdev / page_pool / netmem / TCP/UDP

### Long development history

zero-copy RX + io_uring の試み自体は 2022 年以前から存在する。

-   2022-01: Hao Xu --- `[RFC 0/3] io_uring zerocopy receive`
    -   https://lwn.net/Articles/882416/
-   2022-10/11: Jonathan Lemon --- io_uring/zctap RFC v2/v3
    -   https://lwn.net/Articles/911743/
    -   https://lwn.net/Articles/913659/
-   2023-08: David Wei ---
    `[RFC PATCH 00/11] Zero copy network RX using io_uring`
    -   https://lwn.net/Articles/942809/
-   2023-12: RFC v3
    -   https://lwn.net/Articles/955805/
-   2024-03: RFC v4
    -   https://lwn.net/Articles/965214/
-   2024-10: non-RFC v1
    -   https://lwn.net/Articles/993299/
-   2024-10: v7
    -   https://lwn.net/Articles/996435/
-   2024-12: v9
    -   https://lwn.net/Articles/1002729/
-   2025-01: v10
    -   https://lwn.net/Articles/1004591/
-   2025-02: net-next v13
    -   https://lwn.net/Articles/1008076/

### Important architectural change

RFC v4 以降、Device Memory TCP と共通 infrastructure
を使う方向へ統合された。 v9 の changelog は、merged `net_iov + netmem`
infrastructure の上へ rebase したことを 明記している。

mainline 直前の v13 は、page_pool が kernel pages ではなく userspace
pages を NIC RX queue に供給する構造を説明している。

``` text
normal RX

NIC
 │ DMA
 ▼
kernel page
 │ memcpy
 ▼
userspace


io_uring ZC RX

userspace page
      ▲
      │ page_pool / memory provider
      │
NIC ──┘ DMA directly
```

socket `read` は payload copy ではなく「どの userspace memory に data
があるか」を 通知する操作へ近づく。

### Mainline

Linux 6.15 merge-window coverage は initial zero-copy reception via
io_uring の merge を 明記している。

-   https://lwn.net/Articles/1015414/
-   release: https://lwn.net/Articles/1022457/

### Relationship to Device Memory TCP

v13 patch description 自身が、

> overall approach is similar to the devmem TCP proposal

と説明し、netdev core infrastructure を Device Memory TCP
と共有している。

したがって、

``` text
                 netmem / net_iov
                       │
                 page_pool provider
                  ┌────┴────┐
                  ▼         ▼
             devmem TCP   io_uring ZCRX
                  │         │
          device memory   user memory
```

という共通 architecture として理解するのが適切。

------------------------------------------------------------------------

## 16.3 RTNL lock breakup / per-netns RTNL

**Subsystem:** network-device configuration / scalability\
**Primary issue:** global RTNL contention

RTNL は長年 networking subsystem の巨大な serialization point だった。

``` text
namespace A ─┐
namespace B ─┼── global RTNL ──► serialized
namespace C ─┘
```

container/network-namespace
数が増えるほど、互いに独立しているはずの操作まで global lock
上で競合する。

### Linux 6.13

6.13 には RTNL を per-network-namespace lock にする work が入り、
namespace-heavy workload の contention 削減を狙った。

ただし regression risk が高いため、この段階では default disabled
であり、 `DEBUG_NET_SMALL_RTNL` で有効化する形だった。

LWN: https://lwn.net/Articles/998990/

### Linux 6.15

6.15 でも RTNL breakup は継続しており、LWN はこれを "big networking
lock" の contention bottleneck 解消作業として記録している。

LWN: https://lwn.net/Articles/1015414/

### Architecture direction

``` text
global RTNL
     │
     ▼
per-netns RTNL
     │
     ▼
smaller / finer-grained locking domains
```

この系列は container/Kubernetes のように network namespace
が大量に存在する環境で 特に重要。

**Commit verification:** RTNL breakup は多数の preparatory conversion
commits に またがるため、単一 introduction commit として扱わない。series
単位で継続追跡する。

------------------------------------------------------------------------

## 16.4 AccECN --- Linux 6.18 onward

**Feature:** Accurate Explicit Congestion Notification\
**Subsystem:** TCP / congestion signaling

従来の ECN は congestion information を coarse に伝えるが、AccECN は CE
marking の 量をより正確に sender へ feedback できるようにする。

``` text
classic ECN:
 congestion occurred?  → coarse feedback

AccECN:
 amount / evolution of CE marking
          ↓
 more accurate congestion feedback
```

Linux 6.18 merge-window coverage は AccECN work が merge
されたことを明記する。

LWN: https://lwn.net/Articles/1040203/

AccECN は複数 release にまたがって deployment/default behavior
が進化するため、 6.18 の単発 feature とせず follow-up を別途追跡する。

------------------------------------------------------------------------

## 16.5 UDP receive-path optimization --- Linux 6.18

**Subsystem:** UDP / receive path / performance

6.18 merge-window coverage では Eric Dumazet の測定として UDP receive
performance 47% improvement が報告されている。

LWN: https://lwn.net/Articles/1040203/

### Interpretation warning

この `47%` は Linux UDP stack が全 workload で
47%高速化したという意味ではない。 packet size、CPU、queue
configuration、benchmark method など特定条件下の測定結果として 扱う。

commit-level change log では performance number と mechanism
を分離して記録する。

------------------------------------------------------------------------

## 16.6 Direct Internal Buffer Sharing (DIBS) --- Linux 6.18

**Feature:** Direct Internal Buffer Sharing\
**Subsystem:** networking / shared-memory transports / s390 / SMC-D

DIBS は `page_pool/netmem` の単純な後継ではない点に注意する。
これは既存の internal shared-memory transport components を generic
abstraction として 整理する別系列である。

Initial RFC:

-   `[RFC net-next 00/17] dibs - Direct Internal Buffer Sharing`
-   2025-08-06
-   https://lwn.net/Articles/1032749/

v3:

-   `[PATCH net-next v3 00/14] dibs - Direct Internal Buffer Sharing`
-   2025-09-18
-   https://lwn.net/Articles/1038688/

6.18 merge-window coverage は DIBS merge を明記している。

-   https://lwn.net/Articles/1040203/

### Important correction to earlier classification

以前の章では DIBS を `page_pool → netmem → DIBS`
のように一続きに見える形で 記載していたが、これは技術的には粗すぎる。

より正確には:

``` text
RX buffer / memory-provider lineage

page_pool
   │
 netmem
   ├── Device Memory TCP
   └── io_uring ZC RX


shared-memory transport lineage

ISM / SMC-D
   │
   ▼
 DIBS
```

であり、両者は「copy/buffer overhead
を減らす」という大きな方向性は共有するものの、 直接の継承関係ではない。

この訂正を本 change log の正式な分類とする。

------------------------------------------------------------------------

# 17. Updated architecture map

今回の upstream 照合を反映すると、memory/zero-copy
系列は以下のように整理するのが より正確。

``` text
                        Linux networking memory evolution

        ┌───────────────────────────────────────────┐
        │ RX buffer / external-memory infrastructure │
        └───────────────────────────────────────────┘

          page allocation/recycling
                    │
                page_pool
                    │
                  netmem
                    │
           ┌────────┴────────┐
           ▼                 ▼
    Device Memory TCP    io_uring ZC RX
           │                 │
     DMA-BUF/device      userspace memory
        memory


        ┌───────────────────────────────────────────┐
        │ internal shared-memory transport           │
        └───────────────────────────────────────────┘

             ISM / SMC-D mechanisms
                    │
                    ▼
                   DIBS
```

------------------------------------------------------------------------

# 18. Next commit-level pass

残る優先系列:

1.  io_uring zero-copy **TX** --- initial RFC → mainline
2.  BPF `struct_ops` / TCP congestion control
3.  MPTCP development + BPF integration
4.  nftables / flowtable / conntrack
5.  BIG TCP IPv4 + VXLAN/GENEVE follow-ups
6.  virtio-net / TAP / KubeVirt-oriented zero-copy networking
7.  AccECN individual patch/commit history
8.  UDP RX optimization individual commits
9.  RTNL breakup individual series / enabling progression

------------------------------------------------------------------------

# 19. Commit-level pass 3 --- io_uring ZC TX, BPF struct_ops, MPTCP/BPF, nftables/flowtable

## 19.1 io_uring zero-copy TX

**Subsystem:** io_uring / socket send path / `MSG_ZEROCOPY`
infrastructure\
**Mainline generation:** Linux 6.0 era\
**Primary author:** Pavel Begunkov

### Development timeline

-   2021-11-30: `[RFC 00/12] io_uring zerocopy send`
    -   https://lwn.net/Articles/877167/
-   2021-12-21: `[RFC v2 00/19] io_uring zerocopy tx`
    -   https://lwn.net/Articles/879371/
-   2021-12-30: LWN feature, *Zero-copy network transmission with
    io_uring*
    -   https://lwn.net/Articles/879724/
-   2022-06-28: RFC v3 / 29 patches
    -   https://lwn.net/Articles/899296/
-   2022-07-05: non-RFC `PATCH net-next v3 00/25`
    -   https://lwn.net/Articles/900083/
-   2022-09: API simplification: slot/flush model was replaced by
    per-request completion notifications.
    -   https://lwn.net/Articles/906803/

### Design

Normal send:

``` text
userspace buffer
       │ memcpy
       ▼
kernel-owned skb data
       │
       ▼
NIC
```

Zero-copy send:

``` text
userspace buffer
       │ pin/reference
       ▼
network stack / NIC DMA
       │
       └── completion notification ──► userspace may reuse buffer
```

The difficult part is therefore not only avoiding `memcpy()`, but
defining buffer lifetime and completion semantics. The io_uring design
delivers the "buffer may now be reused" notification through the
completion queue.

The v3 series identifies two networking-side changes in particular:

1.  passing `ubuf_info` from io_uring into the networking layer through
    the in-kernel `msghdr`;
2.  avoiding page-reference overhead for registered buffers where
    possible.

### Mainline evidence

Linux 6.0 stable history contains fixes for the newly introduced
io_uring ZC send path, including:

-   `io_uring/net: fail zc send when unsupported by socket`
-   `net: flag sockets supporting msghdr originated zerocopy`

This places the first mainline generation in the Linux 6.0 timeframe.

### Relationship to later ZC RX

TX and RX are related goals but architecturally different:

``` text
ZC TX (6.0 era)
userspace memory ──► network/NIC

ZC RX (6.15)
network/NIC ──► page_pool memory provider ──► userspace memory
```

The later RX work depends much more directly on `page_pool`, `netmem`,
and memory-provider infrastructure shared with Device Memory TCP.

------------------------------------------------------------------------

## 19.2 BPF `struct_ops` and TCP congestion control --- Linux 5.6

**Subsystem:** BPF / TCP congestion control\
**Primary author:** Martin KaFai Lau

### Development

The BPF STRUCT_OPS series explicitly states that its first use case is
implementing `struct tcp_congestion_ops` in BPF.

Important revisions:

-   2019-12-20: `[PATCH bpf-next v2 00/11] Introduce BPF STRUCT_OPS`
    -   https://lwn.net/Articles/807973/
-   2020-01-08: `[PATCH bpf-next v4 00/11] Introduce BPF STRUCT_OPS`
    -   https://lwn.net/Articles/809092/
-   LWN feature: *Kernel operations structures in BPF*
    -   https://lwn.net/Articles/811631/

### Architecture

``` text
traditional:

kernel C implementation/module
          │
          ▼
 struct tcp_congestion_ops
          │
          ▼
       TCP stack


BPF struct_ops:

BPF program
     │ verifier + BTF/trampoline infrastructure
     ▼
BPF implementation of tcp_congestion_ops
     │
     ▼
TCP stack
```

This is an important transition in BPF networking: BPF is no longer only
attached at packet/socket hooks; it can implement selected kernel
operation tables.

### Why it matters

The cover letter gives the motivation as combining faster algorithm
iteration with the kernel's existing TCP stack, rather than moving
congestion control entirely into a userspace TCP implementation.

Later `struct_ops` work extends this model beyond TCP congestion
control, including network scheduling/qdisc-related experimentation.

------------------------------------------------------------------------

## 19.3 MPTCP + BPF

**Subsystem:** MPTCP / socket selection / BPF

MPTCP entered mainline as a deliberately incremental implementation.
During the period covered by this document, the important trend is not
just "more MPTCP features", but the increasing ability to steer MPTCP
behavior with BPF.

Conceptually:

``` text
application TCP socket
        │
        ▼
       MPTCP
   ┌────┴────┐
 subflow A  subflow B
   │          │
 path A     path B
```

BPF hooks increasingly allow policy to influence socket/protocol
selection and MPTCP subflow behavior. Linux 6.6 is an important point in
this lineage, with BPF support around protocol switching/selection and
MPTCP use cases appearing in the merge-window work.

### Classification rule

MPTCP changes are split into:

1.  protocol implementation milestones;
2.  path-manager/userspace API;
3.  BPF integration.

This avoids treating every MPTCP bug fix or protocol extension as a
separate architecture-level change.

### Commit verification status

The BPF/MPTCP work spans multiple hooks and commits. Exact individual
hashes are left for a dedicated MPTCP pass rather than assigning one
commit as "the MPTCP+BPF commit".

------------------------------------------------------------------------

## 19.4 Netfilter flowtable and hardware offload --- Linux 5.3 onward

**Subsystem:** netfilter / nftables / flowtable / NIC offload\
**Primary contributor in the initial series:** Pablo Neira Ayuso

LWN's 2020 hardware-offload article states that Linux 5.3 received a
patch set adding support for offloading some netfilter packet filtering
to hardware. The work also refactored common offload paths shared with
NIC drivers.

LWN:

-   https://lwn.net/Articles/809333/
-   *Accelerating netfilter with hardware offload, part 2* --- indexed
    by LWN Kernel Index

### Architecture

``` text
without flow offload

packet
  │
  ▼
netfilter hooks / conntrack / nft rules
  │
  ▼
forwarding


flowtable fast path

first packets
  │
conntrack + nftables policy
  │
  ▼
flow established
  │
  ▼
flowtable fast path
  │
  ├── software fast path
  └── hardware offload (where supported)
```

Hardware typically supplies parser/classifier/action capabilities. The
kernel work also had to reduce duplication among previously separate TC,
ethtool, and netfilter offload paths.

### Userspace milestone

nftables 0.9.9 (2021) exposed a flowtable `offload` flag to enable the
hardware fast path.

Example from the release announcement:

``` text
flowtable f {
    hook ingress priority filter + 1
    devices = { ... }
    flags offload
}
```

Archive: https://lwn.net/Articles/857369/

------------------------------------------------------------------------

## 19.5 nftables 1.0 --- 2021

**Subsystem:** netfilter / packet filtering / userspace ABI

nftables 1.0.0 was released on 2021-08-19. LWN's retrospective
emphasizes the long transition from the multiple protocol-specific
iptables-family engines toward a more generic packet-filtering virtual
machine and rule representation.

LWN feature: https://lwn.net/Articles/867185/

### Why it belongs in a kernel-networking change log

The version number itself is userspace, but it marks the maturation of
kernel/userspace nftables interfaces that had accumulated features such
as:

-   atomic ruleset updates;
-   common IPv4/IPv6-capable rule representation;
-   sets/maps/concatenations;
-   stateful filtering/NAT integration;
-   flowtable software fast path;
-   hardware flow offload.

Therefore this entry is tracked as a **userspace/kernel-interface
milestone**, not as a single kernel commit.

------------------------------------------------------------------------

# 20. Refined networking evolution map

The additional research makes it useful to separate three kinds of "fast
path":

``` text
A. Fewer / larger packets
   GRO/GSO
      │
      └── BIG TCP


B. Avoid memory copies
   MSG_ZEROCOPY
      │
      ├── io_uring ZC TX
      │
      └── netmem/page_pool
             ├── io_uring ZC RX
             └── Device Memory TCP


C. Avoid repeated stack/policy work
   nftables/conntrack
      │
      └── flowtable
             ├── software fast path
             └── hardware offload

   BPF/XDP
      │
      ├── socket lookup
      ├── struct_ops
      └── netkit
```

These mechanisms solve different bottlenecks and should not be grouped
simply under "zero-copy" or "offload".

------------------------------------------------------------------------

# 21. Next commit-level pass

Remaining high-priority series:

1.  MPTCP --- protocol milestones, path manager, exact BPF commits
2.  nftables flowtable --- exact kernel patch/commit sequence
3.  conntrack --- scalability/GC/timeout/observability changes
4.  BIG TCP --- IPv4 and overlay/VXLAN/GENEVE follow-ups
5.  virtio-net/TAP --- XDP, multiqueue, mergeable buffers, zero-copy
    evolution
6.  KubeVirt/netkit/io_uring ZC receive developments
7.  AccECN --- exact patch and commit history
8.  UDP receive optimization --- exact mechanism and commits
9.  RTNL breakup --- exact conversion series and activation progression

------------------------------------------------------------------------

# 22. Commit-level pass 4 --- conntrack / GC / timeout / flowtable

## 22.1 Scope

Conntrack は 2019--2026 の間に「一度の大規模
rewrite」が入ったわけではない。 重要な変更は次の軸に分散している。

``` text
                         nf_conntrack
                              │
          ┌───────────────────┼────────────────────┐
          ▼                   ▼                    ▼
    lifecycle/timeout     scalability         programmability
      GC / expiry       hash / netns          BPF kfuncs
          │                   │                    │
          └──────────────┬────┴───────────────┬────┘
                         ▼                    ▼
                    flowtable             observability
                 SW/HW fast path       ctnetlink / dump
```

したがって本資料では、単に `nf_conntrack` に触れた全 commit
を列挙するのではなく、 entry の寿命・lookup/GC・offload・BPF/API
に意味のある系列を追う。

------------------------------------------------------------------------

## 22.2 2019 --- bridge conntrack

2019 年には bridge datapath に connection tracking support
を追加する系列が投稿された。

主要 patch:

-   `netfilter: nf_conntrack: allow to register bridge support`
-   `netfilter: bridge: add connection tracking system`
-   `netfilter: nf_conntrack_bridge: add support for IPv6`
-   `netfilter: nf_conntrack_bridge: register inet conntrack for bridge`

Archive: https://lwn.net/Articles/787195/

主要 source area:

-   `net/bridge/netfilter/nf_conntrack_bridge.c`
-   `include/net/netfilter/nf_conntrack_bridge.h`
-   `net/netfilter/nf_conntrack_proto.c`

これは routed IPv4/IPv6 だけでなく bridge datapath でも conntrack を共通
infrastructure として 使う方向を示す。

------------------------------------------------------------------------

## 22.3 Conntrack timeout model

Current kernel documentation exposes protocol-specific defaults
including:

``` text
nf_conntrack_udp_timeout         = 30 seconds
nf_conntrack_udp_timeout_stream  = 120 seconds

nf_conntrack_tcp_timeout_established = 432000 seconds
```

また flowtable は独立した aging timeout を持つ。

``` text
nf_flowtable_tcp_timeout = 30 seconds
nf_flowtable_udp_timeout = 30 seconds
```

flowtable entry が age out すると connection は classic conntrack path
に戻る。

Kernel documentation:
https://static.lwn.net/kerneldoc/networking/nf_conntrack-sysctl.html

### Important distinction

``` text
conntrack timeout
     │
     └── struct nf_conn の寿命


flowtable timeout
     │
     └── fast-path/offload entry の寿命
              │
              └── age out 後は conntrack に戻る
```

この二つを混同しない。

------------------------------------------------------------------------

## 22.4 2022 --- BPF can manipulate conntrack lifecycle

2022-07 の v7 series は BPF/XDP/TC 側から conntrack を操作するための
kfunc を拡張した。

Patch series: https://lwn.net/Articles/902023/

追加対象:

``` text
bpf_{xdp,skb}_ct_alloc()
bpf_ct_insert_entry()
bpf_ct_{set,change}_timeout()
bpf_ct_{set,change}_status()
```

### Significance

従来:

``` text
packet
  │
netfilter conntrack
  │
nf_conn entry
```

BPF integration 後:

``` text
XDP / TC BPF
      │
      ├── lookup
      ├── allocate
      ├── insert
      ├── timeout manipulation
      └── status manipulation
              │
              ▼
         nf_conntrack
```

つまり BPF datapath が conntrack を単に参照するだけでなく、entry
lifecycle の一部を programmable にできる方向へ進んだ。

------------------------------------------------------------------------

## 22.5 2022 --- delayed TCP packets and timeout refresh semantics

Florian Westphal の 2022 series は、すでに ACK 済みの非常に遅れた TCP
packet を conntrack がどう扱うべきかを修正した。

Archive: https://lwn.net/Articles/906188/

重要な設計点は、そのような packet を即 INVALID/drop 対象にするのではなく
**valid として通す一方、conntrack timeout の延長や state transition
には使わない** という点。

概念的には:

``` text
normal valid TCP packet
        │
        ├── accept
        ├── state update
        └── timeout refresh


overly delayed / already-ACKed packet
        │
        ├── accept as valid
        ├── no state transition
        └── no timeout extension
```

これは「packet が conntrack entry に match した」ことと 「その packet が
entry の expiry を延長する」ことが同義ではない好例。

------------------------------------------------------------------------

## 22.6 UDP NEW offload and early-drop interaction

UDP NEW connection を `act_ct`/flow offload へ載せる work では、
conntrack table pressure 時の **early drop** と offloaded entry
の関係が問題になった。

Patch series: https://lwn.net/Articles/921995/

系列には以下が含まれる。

-   flowtable: UDP state に応じた timeout fix
-   unidirectional flowtable rule
-   `act_ct`: UDP NEW connection offload
-   `nf_conntrack`: offloaded UDP conn も early drop の候補にする

理由は、offloaded UDP NEW entry を early-drop 対象外のままにすると、
一方向 UDP packet を大量に送ることで table
を埋められる可能性があるため。

``` text
UDP first packet
      │
      ▼
   NEW conntrack
      │
      ▼
   flow offload

table pressure
      │
      └── early_drop must still be able to reclaim it
```

------------------------------------------------------------------------

## 22.7 2024 --- conntrack userspace observability

`libnetfilter_conntrack 1.1.0` では conntrack dump/flush filtering
が改善され、 ctnetlink event BPF filtering も IPv6/zone matching
を含めて強化された。

Release: https://lwn.net/Articles/991808/

これは kernel datapath の変更ではないが、大規模 conntrack table を
userspace から 観測・操作する API evolution として記録する。

------------------------------------------------------------------------

## 22.8 2025 --- per-network-namespace conntrack hash-table RFC

2025-11 の RFC は conntrack hash table を global から per-netns
に移すことを提案した。

Archive: https://lwn.net/Articles/1045157/

主な系列:

-   empty table では GC worker を schedule しない
-   hash table auto-sizing を helper 化
-   `nf_conntrack_hash` を `struct net` 側へ移す
-   netns ごとに GC worker を起動
-   conntrack hash table を per-netns 化
-   lazy allocation
-   non-init netns から resize を可能にする
-   NAT bysource hash も per-netns 化

### Motivation

従来:

``` text
netns A ─┐
netns B ─┼── global conntrack hash
netns C ─┘
```

提案:

``` text
netns A ── conntrack hash A
netns B ── conntrack hash B
netns C ── conntrack hash C
```

ただし RFC 時点では未解決事項も明記されている。

-   lock array はまだ global
-   hash secret も global
-   per-netns memory accounting / memcg integration 未解決
-   OVS/TC/BPF CT helper testing 不十分

したがって、この文書では **2025 時点では RFC / not yet merged**
と明示する。

------------------------------------------------------------------------

## 22.9 2026 --- custom conntrack timeout policy lifetime

2026-06 の `cttimeout` series は custom timeout policy の
lifetime/refcount handling を 整理する。

Patch: https://lwn.net/Articles/1076158/

重要な変更:

-   `struct nf_ct_timeout` に dataplane use を追跡する refcount
-   conntrack entry が timeout policy を参照している間は object を保持
-   control-plane refcount を整理
-   policy object の削除と既存 conntrack 参照の lifetime を分離

元 infrastructure の introduction commit として patch は:

``` text
50978462300f
"netfilter: add cttimeout infrastructure for fine timeout tuning"
```

を `Fixes:` で参照している。

------------------------------------------------------------------------

## 22.10 2026 --- flowtable GC partial-state race

2026-08 の fix は、flow entry が hash table に publish された後、
hardware-offload setup が完全に終わる前に GC が entry を観測できる狭い
window を扱う。

Patch: https://lwn.net/Articles/1089361/

新しい `NF_FLOW_CONFIRMED` bit を導入し、

``` text
flow allocation
     │
hash insertion
     │
HW setup
     │
memory barrier
     │
NF_FLOW_CONFIRMED
     │
     └── only now GC may act on entry
```

とする。

これは一般的な conntrack GC そのものではなく **flowtable GC** の race
だが、 conntrack-backed fast path の lifetime/aging
を理解するうえで重要。

------------------------------------------------------------------------

## 22.11 2026 --- stale `skb->_nfct` revalidation

TC/clsact/pedit など kernel 内で packet header が変更された場合、 skb
にすでに付いている conntrack reference と現在の L3/L4 header
が一致しない可能性がある。

2026-09 series: https://lwn.net/Articles/1094326/

新しい careful lookup は protocol/header を再検証し、不一致なら stale
reference を捨てて `nf_conntrack_in()` に再 lookup させる。

``` text
skb has TCP conntrack
       │
       ▼
TC/pedit changes packet → UDP
       │
       ▼
old behavior:
stale TCP nf_conn may remain attached

new behavior:
revalidate tuple/protocol
       │
       ├── matches → reuse
       └── mismatch → drop reference + relookup
```

------------------------------------------------------------------------

# 23. Conntrack GC / expiry model --- conceptual notes

conntrack expiry を考えるとき、少なくとも次を分ける必要がある。

``` text
1. packet lookup
       │
       ▼
2. existing nf_conn reference acquired
       │
       ▼
3. protocol/state validation
       │
       ├── timeout refresh may occur
       └── refreshしない packet class もある
       │
       ▼
4. expiry/GC machinery
       │
       ▼
5. hash removal / dying state
       │
       ▼
6. final object release after references disappear
```

したがって、

> `struct nf_conn` への reference を持っている

ことと、

> entry が conntrack hash table から削除されない

ことは同じ保証ではない。

また flowtable を使う場合はさらに:

``` text
conntrack lifetime
        │
        └── flowtable lifetime / aging
```

という別 layer が加わる。

極端に短い UDP conntrack timeout を設定した検証では、この lifecycle
boundary を 通常 workload よりはるかに高頻度で踏むため、GC/refresh race
の観測確率も高くなる。

------------------------------------------------------------------------

# 24. Conntrack-related source map

主な追跡対象:

``` text
net/netfilter/nf_conntrack_core.c
net/netfilter/nf_conntrack_proto_tcp.c
net/netfilter/nf_conntrack_proto_udp.c
net/netfilter/nf_conntrack_netlink.c
net/netfilter/nf_conntrack_bpf.c

net/netfilter/nf_flow_table_core.c
net/netfilter/nf_flow_table_offload.c

include/net/netfilter/nf_conntrack.h
include/net/netfilter/nf_conntrack_core.h
include/net/netfilter/nf_flow_table.h
```

関連 datapath:

``` text
net/sched/act_ct.c
net/openvswitch/conntrack.c
net/bridge/netfilter/nf_conntrack_bridge.c
```

------------------------------------------------------------------------

# 25. Next pass

次の commit-level pass は以下を優先する。

1.  **BIG TCP follow-up**
    -   IPv4 BIG TCP
    -   GRO/GSO metadata evolution
    -   VXLAN / GENEVE
2.  **virtio-net / TAP / VM networking**
    -   XDP
    -   mergeable buffers
    -   multiqueue
    -   zero-copy
3.  **netkit + KubeVirt/io_uring ZC RX**
4.  **AccECN exact patch/commit history**
5.  **UDP receive optimization exact commits**
6.  **RTNL breakup exact series**
7.  MPTCP exact milestone/commit map

------------------------------------------------------------------------

# 26. Commit-level pass 5 --- BIG TCP evolution, virtio-net/AF_XDP, netkit/KubeVirt

## 26.1 BIG TCP: IPv6 → IPv4 → tunnel/overlay

BIG TCP の発展は次の3段階に分けると理解しやすい。

``` text
Linux 5.19
IPv6 BIG TCP
   │
   ▼
Linux 6.3
IPv4 BIG TCP
   │
   ▼
2026 / Linux 7.3
BIG TCP through VXLAN / GENEVE
```

### Linux 5.19 --- initial IPv6 BIG TCP

初期 BIG TCP は IPv6 jumbogram の仕組みを利用し、64KiB を超える
kernel-internal TCP/GSO packet を扱えるようにした。

この時点では IPv6-specific な仕組み、特に Hop-by-Hop (HBH) header
を利用する 実装上の制約が残っていた。

### Linux 6.3 --- IPv4 BIG TCP

Xin Long の IPv4 series は v2/v3 で 10 patches。

Important series:

-   `[PATCHv2 net-next 00/10] net: support ipv4 big tcp`
-   2023-01-23
-   LWN archive: https://lwn.net/Articles/921060/

v3:

-   2023-01-27
-   https://lwn.net/Articles/921509/

Linux 6.3 release note は IPv4 BIG TCP support を significant change
として明記する。

-   https://lwn.net/Articles/929851/

### IPv4 BIG TCP patch structure

系列には以下が含まれる。

``` text
helpers for IPv4 total length
       │
       ├── bridge netfilter
       ├── OVS conntrack
       ├── TC
       ├── netfilter
       ├── ipvlan
       └── AF_PACKET
       │
       ▼
IPv4 BIG TCP core support
       │
       ▼
selftest
```

つまり IPv4 BIG TCP は `net/ipv4` だけの変更ではない。 従来「IPv4 total
length は16bitに収まる」と仮定していた複数 subsystem を BIG TCP-aware
にする必要があった。

主要 affected files:

-   `include/linux/ip.h`
-   `net/core/gro.c`
-   `net/core/sock.c`
-   `net/ipv4/af_inet.c`
-   `net/ipv4/ip_input.c`
-   `net/ipv4/ip_output.c`
-   `net/bridge/netfilter/nf_conntrack_bridge.c`
-   `net/openvswitch/conntrack.c`
-   `net/sched/act_ct.c`
-   `net/sched/sch_cake.c`
-   `tools/testing/selftests/net/big_tcp.sh`

------------------------------------------------------------------------

## 26.2 2026 --- BIG TCP without IPv6 HBH

Alice Mikityanska の 2026 series は IPv6 BIG TCP の設計を IPv4
に近づけるため、 BIG TCP 用 Hop-by-Hop header を除去する。

Important revision:

-   `[PATCH net-next v3 00/11] BIG TCP without HBH in IPv6`
-   2026-02-02
-   https://lwn.net/Articles/1057080/

### Why remove HBH?

従来:

``` text
IPv6 BIG TCP
     │
     └── HBH header carries 32-bit packet length
```

しかし tunnel を組み合わせると、

``` text
outer IPv6
   │ HBH?
   ▼
VXLAN/GENEVE
   │
inner IPv6
   │ HBH?
   ▼
BIG TCP packet
```

のように inner/outer の組み合わせごとに HBH stripping/validation
が必要になる。

さらに NIC driver 側にも BIG TCP HBH handling が必要だった。

新しい方向:

``` text
IPv6 BIG TCP
     │
     └── restore/derive length from skb metadata
             │
             └── align with IPv4 BIG TCP
```

これにより IPv4/IPv6 BIG TCP implementation を揃え、 UDP tunnel support
を追加しやすくした。

この series では mlx5, mlx4, ice, bnxt_en, gve, mana などから
BIG-TCP-specific `jumbo_remove` processing を除去する。

------------------------------------------------------------------------

## 26.3 Linux 7.3 --- BIG TCP over VXLAN / GENEVE

2026 年の follow-up series:

-   `[PATCH net-next v9 0/9] BIG TCP for UDP tunnels`
-   2026-07-10
-   https://lwn.net/Articles/1082319/

Linux 7.3 merge-window coverage は、

> big TCP packets in UDP tunnels managed with VXLAN and GENEVE

が可能になったことを明記する。

-   https://lwn.net/Articles/1089791/

### Patch architecture

v9 series:

1.  UDP length-field accessor
2.  UDP tunnel code の BIG TCP gaps を修正
3.  `udp_gro_receive()` で malformed `len=0` packet を排除
4.  tcpdump formatting 用に overflow 時の UDP length を0として扱う
5.  VXLAN `tso_max_size` を拡大
6.  GENEVE `tso_max_size` を拡大
7.  selftests

BIG TCP UDP tunnel では oversized internal UDP packet の length field に
通常の16bit表現をそのまま使えないため、`len=0` handling と validation
が重要になる。

### Evolution

``` text
IPv6 BIG TCP (5.19)
       │
       ▼
IPv4 BIG TCP (6.3)
       │
       ▼
remove IPv6 HBH dependency
       │
       ▼
BIG TCP through UDP tunnel
       │
       ├── VXLAN
       └── GENEVE
             │
             ▼
        Linux 7.3
```

これは Kubernetes/OVN/Cilium など overlay-heavy
な環境にも直接関係する変更である。

------------------------------------------------------------------------

# 27. virtio-net and AF_XDP zero-copy

## 27.1 Why virtio-net is special

物理 NIC driver の AF_XDP zero-copy は比較的直接的である。

``` text
NIC RX queue
    │
    ▼
UMEM
    │
    ▼
AF_XDP application
```

virtio-net では間に virtqueue / virtual device / host backend がある。

``` text
guest
 virtio-net
    │
 virtqueue
    │
 host backend
    │
 physical networking
```

そのため AF_XDP zero-copy 対応には、

-   virtqueue reset
-   premapped DMA
-   XDP refactoring
-   queue ownership/sharing
-   NAPI wakeup
-   mergeable receive buffer

などの前提作業が必要になる。

------------------------------------------------------------------------

## 27.2 2023 --- initial large AF_XDP zero-copy series

Xuan Zhuo の初期 series:

-   `[PATCH 00/33] virtio-net: support AF_XDP zero copy`
-   2023-02-02
-   https://lwn.net/Articles/921993/

2023-10:

-   `[PATCH net-next v1 00/19] virtio-net: support AF_XDP zero copy`
-   https://lwn.net/Articles/947912/

この段階から、AF_XDP zero-copy を virtio-net driver に持ち込むための
大規模 refactor が継続した。

------------------------------------------------------------------------

## 27.3 2024 --- series decomposition

2024 年には series を分割し、preparatory work と TX zero-copy を段階的に
review する形へ移行した。

Preparation:

-   `[PATCH net-next 0/7] virtnet_net: prepare for af-xdp`
-   2024-05-08
-   https://lwn.net/Articles/972853/

v5:

-   `[PATCH net-next v5 00/15] virtio-net: support AF_XDP zero copy`
-   2024-06-14
-   https://lwn.net/Articles/978434/

TX-specific series:

-   `[PATCH net-next 00/13] virtio-net: support AF_XDP zero copy (tx)`
-   2024-08-20
-   https://lwn.net/Articles/986516/

### Important design constraint

virtio-net は AF_XDP 専用に queue 数を自由に増やせないため、 AF_XDP と
kernel networking が queue を共有する設計が必要になる。

また TX NAPI が別 CPU 上で動いていた場合の wakeup など、 physical NIC
driver にはない virtio-specific な問題もある。

------------------------------------------------------------------------

## 27.4 2025 --- zero-copy multi-buffer XDP with mergeable buffers

RFC v2:

-   `[RFC PATCH net-next v2 0/2] virtio-net: support zerocopy multi buffer XDP in mergeable`
-   2025-05-27
-   https://lwn.net/Articles/1022732/

従来、virtio-net の zero-copy + mergeable receive-buffer mode では XDP
packet が single buffer に制限されていた。

新しい series は XDP frags を利用して、

``` text
large packet / jumbo MTU
        │
        ▼
virtio mergeable buffers
   ┌────┼────┐
   ▼    ▼    ▼
 buf1  buf2  buf3
   └────┼────┘
        ▼
multi-buffer XDP
```

を zero-copy path でも扱えるようにする。

これは前章の AF_XDP multi-buffer と BIG TCP/jumbo packet evolution
に対応する VM-side の重要な work とみなせる。

------------------------------------------------------------------------

# 28. netkit queue leasing --- physical queue into a network namespace

## 28.1 Problem

container/VM が network namespace 内にある場合、通常は host physical NIC
の queue を直接 reconfigure できない。

しかし io_uring zero-copy RX memory provider や AF_XDP zero-copy は、
physical RX queue に memory provider/UMEM を bind する必要がある。

``` text
host namespace

physical NIC
 RXQ0 RXQ1 RXQ2
      │
      X  namespace boundary
      │
container / VM netns
```

------------------------------------------------------------------------

## 28.2 Queue leasing

Daniel Borkmann の 2026 netkit series は **queue leasing** を導入する。

Important revision:

-   `[PATCH net-next v8 00/16] netkit: Support for io_uring zero-copy and AF_XDP`
-   2026-01-29
-   https://lwn.net/Articles/1056727/

Concept:

``` text
host namespace

physical NIC
 RXQ0  RXQ1  RXQ2
        │
        │ lease
        ▼
+--------------------------+
| container / VM netns     |
|                          |
| netkit leased queue      |
|        │                 |
|        ├─ io_uring ZC RX |
|        └─ AF_XDP         |
+--------------------------+
```

leased queue は physical netdev の real queue に binding される proxy
として動作する。

userspace は network namespace 内の virtual netdev の:

``` text
ifindex + queue_id
```

を指定し、operation は underlying physical queue へ proxy される。

### Tested hardware

series description では少なくとも以下で testing したと記載されている。

-   NVIDIA ConnectX-6 / mlx5
-   Broadcom BCM957504 / bnxt_en 100G

------------------------------------------------------------------------

# 29. Why this matters to KubeVirt

2026 LSFMM+BPF summit の LWN report は KubeVirt を具体的な use case
として挙げている。

-   https://lwn.net/Articles/1083418/

KubeVirt では概念的に:

``` text
Kubernetes Pod / network namespace
             │
             ▼
        QEMU / KubeVirt VM
             │
          virtio-net
             │
             ▼
       virtual datapath
             │
             ▼
         host NIC
```

という namespace isolation と VM networking を同時に必要とする。

従来の高速 VM networking:

``` text
SR-IOV / device passthrough
        │
        └── fast
             but
        host/network-namespace policy integration が難しい
```

netkit + queue leasing:

``` text
physical NIC queue
       │
       ▼
netkit queue lease
       │
       ▼
network namespace
       │
       ├── io_uring zero-copy RX
       └── AF_XDP
       │
       ▼
VM / KubeVirt
```

という方向で、network namespace semantics を保ちながら physical queue と
zero-copy datapath を結びつける。

LWN の 2026 report は netkit が network namespace 内の VM へ zero-copy
packet reception を提供できる段階まで進んだと報告している。

------------------------------------------------------------------------

# 30. Combined VM/networking evolution

今回までの調査を統合すると、VM/container datapath
は次のような流れになる。

``` text
                     traditional virtual networking

physical NIC
     │
 host kernel
     │
 veth/TAP
     │
 QEMU
     │
virtio-net guest


                  packet processing optimization

physical NIC
     │
 XDP / AF_XDP
     │
 TAP / virtio-net
     │
 guest


                    copy reduction

physical NIC
     │
 AF_XDP / io_uring ZC
     │
     │ namespace boundary problem
     ▼
 virtual workload


                  netkit queue leasing

physical NIC RX queue
     │
     ▼
 leased queue / proxy
     │
     ▼
 network namespace
     │
 io_uring ZC / AF_XDP
     │
     ▼
 VM / KubeVirt
```

これは、

> 「VM に NIC を passthrough して host stack を bypass する」

以外の高速化ルートとして重要である。

------------------------------------------------------------------------

# 31. Cross-series relationship

``` text
GRO/GSO
   │
   └── BIG TCP
          │
          ├── IPv4 BIG TCP
          │
          └── VXLAN/GENEVE BIG TCP


XDP
 │
 └── AF_XDP
       │
       └── multi-buffer
              │
              └── virtio-net multi-buffer ZC


page_pool
   │
 netmem / memory provider
   │
   ├── Device Memory TCP
   └── io_uring ZC RX
             │
             ▼
       netkit queue leasing
             │
             ▼
        container / VM netns
             │
             ▼
          KubeVirt
```

これらは別々の機能に見えるが、

1.  packet aggregation を大きくする
2.  copy を減らす
3.  memory ownership を抽象化する
4.  physical queue を namespace 境界越しに利用可能にする

という連続した設計課題として読むことができる。

------------------------------------------------------------------------

# 32. Verification status and next pass

今回確認できた release-level facts:

-   IPv4 BIG TCP: Linux 6.3
-   BIG TCP over VXLAN/GENEVE: Linux 7.3
-   virtio-net AF_XDP zero-copy: 2023--2025 に複数 revision /
    preparatory series
-   netkit queue leasing + io_uring ZC/AF_XDP: 2026 series
-   KubeVirt/network-namespace VM: netkit queue-leasing work の明示的
    use case

次の pass:

1.  AccECN exact series / commits
2.  UDP receive optimization exact commits
3.  RTNL breakup exact series / commits
4.  MPTCP exact milestone map
5.  BIG TCP follow-up の exact mainline commits
6.  virtio-net AF_XDP work の merged-vs-RFC status を release 単位で確定
7.  netkit queue leasing の individual mainline commits

------------------------------------------------------------------------

# 33. Commit-level pass 6 --- AccECN, UDP RX, RTNL, MPTCP/BPF

## 33.1 AccECN core --- Linux 6.18

**Feature:** Accurate Explicit Congestion Notification (AccECN)\
**Core merge:** Linux 6.18\
**Protocol:** RFC 9768\
**Subsystem:** TCP / ECN / congestion-control feedback\
**Primary development lineage:** Ilpo Järvinen's earlier AccECN work,
later upstreamed/reworked by Chia-Yu Chang and reviewers.

### Why AccECN exists

Classic RFC3168 ECN mainly tells the sender that congestion was
encountered. AccECN carries more detailed congestion-marking feedback,
allowing the sender/congestion-control algorithm to estimate how much CE
marking occurred.

``` text
RFC3168 ECN

network CE marks
      │
      ▼
receiver
      │ ECE/CWR semantics
      ▼
sender
      │
      └── congestion happened


AccECN

network CE marks
      │
      ▼
receiver
      │
      ├── ACE field
      └── AccECN option / counters
              │
              ▼
sender obtains richer CE feedback
```

This matters particularly for modern congestion-control/AQM work where
the *amount* of congestion signaling is useful, not merely a binary
indication.

### Long review history

The upstream protocol series went through unusually many revisions.

Selected milestones:

-   2025-03: v2 --- missing preparation patch restored
-   2025-04: v4/v5 --- 32-bit ARM alignment fixes
-   2025-05-14: v7 / 15 patches
    -   https://lwn.net/Articles/1021315/
-   2025-06-10: v8
    -   https://lwn.net/Articles/1024740/
-   2025-07-03: v10
    -   introduced separate `include/net/tcp_ecn.h`
    -   added sysctl documentation and additional negotiation cleanup
    -   https://lwn.net/Articles/1028208/
-   2025-07-18: v13
    -   lookup/table and option-processing refinements
    -   https://lwn.net/Articles/1030805/
-   2025-09-06: v16 / 14 patches
    -   https://lwn.net/Articles/1037055/

### v16 core contents

The v16 cover letter describes the series as covering:

-   Accurate ECN core
-   AccECN negotiation
-   AccECN TCP options
-   failure handling

Important patch subjects in the development series include:

``` text
tcp: reorganize SYN ECN code
tcp: AccECN core
tcp: accecn: AccECN negotiation
tcp: accecn: add AccECN rx byte counters
tcp: accecn: AccECN needs to know delivered bytes
tcp: sack option handling improvements
tcp: accecn: AccECN option
tcp: accecn: AccECN option send control
tcp: accecn: AccECN option failure handling
```

Affected areas include:

-   `include/linux/tcp.h`
-   `include/net/tcp.h`
-   `include/net/tcp_ecn.h`
-   `include/uapi/linux/tcp.h`
-   `net/ipv4/tcp.c`
-   `net/ipv4/tcp_input.c`
-   `net/ipv4/tcp_output.c`
-   `net/ipv4/tcp_minisocks.c`
-   syncookies
-   TCP sysctls/documentation

### Merge

LWN's Linux 6.18 merge-window coverage explicitly records that AccECN
was merged.

-   https://lwn.net/Articles/1040203/

### Important: 6.18 is not the end of the series

AccECN core merge did not mean all integration work was complete.

2025-09 onward, a separate **case handling** series added:

-   exceptional RFC9768 handling
-   identifiers for congestion-control modules
-   `ecn_delta` in `rate_sample`
-   ACE counter preservation
-   fallback/persistence behavior

Example:

-   2025-10 v4: https://lwn.net/Articles/1041621/
-   2026-01 v7: https://lwn.net/Articles/1052707/

### 2026 --- GRO/GSO / virtio offload integration

AccECN changes the meaning of TCP flag bits used in ACE signaling.
Existing RFC3168 ECN offload assumptions can therefore corrupt AccECN
signaling if applied blindly.

2026 series:

-   `[PATCH ...] ECN offload handling for AccECN`
-   https://lwn.net/Articles/1056942/

Affected code includes:

-   `drivers/net/virtio_net.c`
-   mlx5 RX
-   hns3 RX
-   `include/linux/skbuff.h`
-   `include/linux/virtio_net.h`
-   UAPI virtio-net header

This is particularly relevant to VM networking: AccECN must survive
GRO/GSO and virtio metadata transport without CWR/ACE corruption.

### Correct classification

``` text
6.18
AccECN protocol core
       │
       ▼
case handling / CC integration
       │
       ▼
GRO/GSO + driver/virtio offload correctness
       │
       ▼
continued 2026 integration
```

Therefore this document does **not** label Linux 6.18 as "AccECN
completed"; it is the mainline core milestone.

------------------------------------------------------------------------

# 34. UDP receive performance --- Linux 6.18

LWN's 6.18 merge-window summary reports a **47% UDP receive performance
improvement** according to Eric Dumazet's measurements.

-   https://lwn.net/Articles/1040203/

A follow-up comment on the LWN article provides an important benchmark
qualification: the measurement used **120-byte packets under high
network load**.

-   https://lwn.net/Articles/1041072/

Therefore the change log records this as:

``` text
reported improvement: ~47%
workload: high network load
packet size: 120 bytes
scope: benchmark result, not universal UDP throughput improvement
```

### Why this qualification matters

Small-packet UDP receive is primarily a packet-rate/CPU-cost workload:

``` text
large packets:
bandwidth / memory movement often dominates

small 120-byte packets:
packet rate
   │
   ├── skb allocation/free
   ├── socket lookup
   ├── queueing
   ├── locking
   └── per-packet accounting
          │
          ▼
       CPU cost dominates
```

Thus a large percentage improvement in this test should not be
extrapolated to jumbo frames, low-rate UDP, or application-limited
workloads.

### Commit verification status

The 6.18 release-level performance claim is verified. Search did not
produce a sufficiently authoritative mapping from the 47% figure to one
single commit; this document therefore does not invent an "UDP 47%
commit". The optimization may span multiple receive-path changes and is
kept at release/series level until the exact net-next pull/commits are
identified.

------------------------------------------------------------------------

# 35. RTNL scalability --- exact series landmarks

## 35.1 Problem

Classic RTNL is a broad global serialization mechanism.

``` text
netns A operation ─┐
netns B operation ─┼── rtnl_lock()
netns C operation ─┘
                         │
                         ▼
                    serialization
```

The modern work proceeds along **two complementary directions**:

1.  make RTNL smaller/per-netns;
2.  remove RTNL from read/dump paths where a narrower synchronization
    mechanism is enough.

------------------------------------------------------------------------

## 35.2 RTNL-less qdisc dumps --- 2024

Eric Dumazet posted:

-   `[PATCH net-next 00/14] net_sched: first series for RTNL-less qdisc dumps`
-   2024-04-15
-   https://lwn.net/Articles/969889/

The cover letter states the medium-term goal directly:

``` text
tc qdisc show
```

should no longer need to acquire RTNL.

The first series converted 14 qdisc dump implementations to
lockless/narrower-lock operation, including `fq`, `cake`, `cbs`, and
others.

This illustrates that RTNL breakup is not just:

``` text
one global lock → one lock per netns
```

but also:

``` text
operation previously under RTNL
          │
          ▼
identify actual protected state
          │
          ▼
use local locking / RCU / lockless dump
          │
          ▼
remove RTNL dependency
```

------------------------------------------------------------------------

## 35.3 Linux 6.13 --- per-network-namespace RTNL

LWN confirms that Linux 6.13 contains work turning RTNL into a
per-network-namespace lock.

-   https://lwn.net/Articles/998990/

Important limitation:

-   disabled by default at this stage;
-   enabled through `DEBUG_NET_SMALL_RTNL`;
-   considered regression-prone;
-   explicitly described as one step in a longer process.

Architecture:

``` text
before

netns A ─┐
netns B ─┼── global RTNL
netns C ─┘


direction in 6.13

netns A ─── RTNL(A)
netns B ─── RTNL(B)
netns C ─── RTNL(C)
```

This is particularly relevant to Kubernetes/OpenShift hosts where
independent netns operations are common.

------------------------------------------------------------------------

## 35.4 Link creation and namespace semantics

Xiao Liang's 2024 v5 series:

-   `[PATCH net-next v5 0/5] net: Improve netns handling in RTNL and ip_tunnel`
-   https://lwn.net/Articles/1001477/

changes link creation so that a device intended for another namespace is
created directly in the target namespace rather than created in one
namespace and moved afterward.

Example motivation:

``` text
ip link add netns ns1 link-netns ns2 tun0 type gre ...
```

The new design passes both source and link namespaces into `newlink()`
callbacks.

This matters for finer-grained RTNL because "which namespace's lock
protects creation?" must be well-defined.

------------------------------------------------------------------------

# 36. MPTCP + BPF --- exact development landmarks

## 36.1 Problem: enabling MPTCP for unmodified applications

An application normally opts into MPTCP with:

``` c
socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP)
```

`mptcpize` used `LD_PRELOAD` to make legacy applications use MPTCP, but
that approach has limitations:

-   applications not using libc, such as some Go binaries;
-   environments where changing launch environment is difficult;
-   per-cgroup/per-netns policy is awkward.

Geliang Tang's BPF series therefore moved protocol selection into the
kernel/BPF policy path.

------------------------------------------------------------------------

## 36.2 2023 --- `update_socket_protocol()` / "Force to MPTCP"

Important revision:

-   `[PATCH bpf-next v8 0/4] bpf: Force to MPTCP`
-   2023-08-03
-   https://lwn.net/Articles/940312/

Later revision:

-   v14, 2023-08-16
-   https://lwn.net/Articles/941738/

The key design change arrived around v6:

``` text
update_socket_protocol()
```

allowing a BPF hook during socket creation to change:

``` text
IPPROTO_TCP / protocol 0
          │
          ▼
      IPPROTO_MPTCP
```

Conceptually:

``` text
application
 socket(AF_INET, SOCK_STREAM, 0)
          │
          ▼
BPF socket-create policy
          │
          ├── keep TCP
          └── switch to MPTCP
                    │
                    ▼
                MPTCP socket
```

This avoids requiring the application binary to know about MPTCP.

------------------------------------------------------------------------

## 36.3 MPTCP subflow visibility from BPF

Later work expands BPF from protocol selection to **inspection/iteration
of MPTCP subflows**.

Series:

-   `bpf: Add mptcp_subflow bpf_iter support`
-   https://lwn.net/Articles/997541/

The series adds:

-   common MPTCP kfunc registration;
-   `mptcp_subflow` BPF iterator;
-   acquire/release helpers for MPTCP socket lifetime;
-   multi-endpoint selftests.

Affected code:

-   `net/mptcp/bpf.c`
-   BPF MPTCP selftests

A later 2025 iteration similarly describes registering basic MPTCP
kfuncs and adding the subflow iterator:

-   https://lwn.net/Articles/1014990/

### Evolution

``` text
MPTCP protocol implementation
          │
          ▼
userspace path-manager/control APIs
          │
          ▼
BPF: choose TCP vs MPTCP
          │
          ▼
BPF: inspect/iterate MPTCP subflows
          │
          ▼
richer programmable MPTCP policy
```

This is the more useful way to classify "MPTCP+BPF" than assigning the
whole feature to a single release.

------------------------------------------------------------------------

# 37. Cross-feature interaction: AccECN × virtio × BPF × MPTCP

The separate series increasingly meet in common metadata and
virtual-network paths.

``` text
TCP connection
     │
     ├── MPTCP?
     │      └── BPF policy / subflows
     │
     ├── AccECN?
     │      └── ACE / TCP option / CC feedback
     │
     ▼
GRO/GSO
     │
virtio-net metadata
     │
     ▼
VM / container datapath
```

A modern virtual networking stack therefore cannot treat:

-   TCP flags,
-   ECN metadata,
-   GSO metadata,
-   MPTCP protocol choice,
-   BPF policy

as unrelated concerns.

The 2026 AccECN virtio/offload series is a concrete example: existing
RFC3168-oriented offload metadata had to be updated because the same TCP
flag bits participate in AccECN signaling.

------------------------------------------------------------------------

# 38. Next pass

Remaining high-priority items:

1.  identify exact mainline commits for BIG TCP IPv4 / tunnel
    follow-ups;
2.  resolve virtio-net AF_XDP series into **merged vs RFC/not-merged**
    release status;
3.  netkit queue-leasing individual commits;
4.  RTNL per-netns individual commits and when the feature becomes
    generally enabled;
5.  AccECN individual mainline hashes;
6.  exact UDP 6.18 receive-path commit set;
7.  complete MPTCP release-by-release milestone table;
8.  audit the 2019--2026 LWN article list for omissions after these
    thematic passes.

------------------------------------------------------------------------

# 39. Completeness audit --- LWN networking topics missing from the first thematic passes

この章は「mainline に入った大機能」だけを追う前章までの方式を補完する。
LWN の networking coverage
を年別に再走査し、次の3種類を区別して記録する。

``` text
A. merged architecture/API changes
B. important development series / design discussions
C. significant failed, stalled, or removed approaches
```

Linux networking の技術史を理解するには C も重要である。 例えば bpfilter
や P4TC は、mainline に定着した機能だけを見ていると見落とす。

------------------------------------------------------------------------

## 39.1 2020 --- IPv6 extension-header processing

LWN feature:

-   *The trouble with IPv6 extension headers*
-   2020-01-07
-   https://lwn.net/Articles/808896/

IPv6 extension headers は IPv4 option より柔軟な protocol extension
mechanism だが、 kernel fast path、middlebox
behavior、security、offloadability との間に tension がある。

この議論は後の BIG TCP で特に興味深い。

``` text
2020:
IPv6 extension headers の generic processing をどう扱うか

2019–2022:
IPv6 BIG TCP が HBH/Jumbogram mechanism を利用

2026:
BIG TCP が HBH dependency を取り除く方向へ
```

したがって BIG TCP の HBH removal は単なる code cleanup ではなく、
長年の IPv6 extension-header processing/offload complexity
の文脈でも読める。

------------------------------------------------------------------------

## 39.2 2020 --- threaded NAPI

LWN feature:

-   *NAPI polling in kernel threads*
-   2020-10-09
-   https://lwn.net/Articles/833840/

Traditional NAPI:

``` text
NIC IRQ
  │
  ▼
schedule NAPI
  │
  ▼
NET_RX softirq
  │
  ▼
driver poll()
```

Threaded NAPI:

``` text
NIC IRQ
  │
  ▼
schedule NAPI
  │
  ▼
dedicated kernel thread
  │
  ▼
driver poll()
```

Motivation:

-   softirq context から network work を切り離す;
-   CPU affinity / scheduling priority を管理しやすくする;
-   heavily loaded CPUs と idle CPUs の imbalance を改善する;
-   latency-sensitive workloads で network processing
    を制御しやすくする。

Initial patch evidence:

-   `net: add support for threaded NAPI polling`
-   2020-08
-   https://lwn.net/Articles/828372/

この系列は page_pool/BIG TCP
のような「packet当たりのcostを減らす」変更とは異なり、 **network
processingをどのexecution contextで実行するか**を変える。

------------------------------------------------------------------------

## 39.3 2020 → 2025 --- bpfilter: failed in-kernel experiment and later userspace revival

### 2020

LWN:

-   *Rethinking bpfilter and user-mode helpers*
-   2020-06-12
-   https://lwn.net/Articles/822744/

初期 bpfilter は kernel 内から user-mode helper/blob を起動し、iptables
compatibility を BPF-based firewallへ変換する構想だった。

しかし開発停滞と user-mode helper infrastructure 自体への懸念から、
kernel-side experiment は失敗した方向として扱われた。

### 2025

LWN:

-   *Faster firewalls with bpfilter*
-   2025-05-14
-   https://lwn.net/Articles/1017705/

2025年の bpfilter は、古い in-kernel user-mode-blob design
と同一視してはいけない。 userspace daemon / BPF-based packet filtering
として再構成された別世代の取り組みである。

Change-log classification:

``` text
bpfilter v1
  kernel user-mode helper architecture
       │
       └── stalled / removed direction

bpfilter later project
  userspace control plane
       │
       └── BPF datapath/firewall acceleration
```

------------------------------------------------------------------------

## 39.4 2021 --- `SO_REUSEPORT` connection-failure semantics

LWN feature:

-   *Avoiding unintended connection failures with SO_REUSEPORT*
-   2021-04-23
-   https://lwn.net/Articles/853637/

`SO_REUSEPORT` allows multiple sockets/processes to bind the same
listening endpoint and distribute incoming connections.

Problem discussed by LWN:

``` text
incoming SYN
     │
reuseport group
 ┌───┼───┐
 ▼   ▼   ▼
S1  S2  S3

selected listener closes / becomes unavailable
     │
     └── connection may be lost unexpectedly
```

This belongs in the change log because it concerns **listener selection
and failover semantics at high connection rates**, not merely a
socket-option bug.

It also forms part of the broader trend toward scalable listener/socket
selection, together with BPF `SK_REUSEPORT` and socket-lookup hooks.

------------------------------------------------------------------------

## 39.5 2022 --- `skb_drop_reason`: packet-drop observability

LWN coverage and patch archives show a broad effort to replace opaque
`kfree_skb()` sites with explicit drop reasons.

TCP state-transition series:

-   initial: https://lwn.net/Articles/895346/
-   v3: https://lwn.net/Articles/897523/

Concept:

``` text
old:

packet
  │
validation/state processing
  │
  └── kfree_skb()
         │
         └── "packet disappeared"


new:

packet
  │
validation/state processing
  │
  └── SKB_DROP_REASON_*
         │
         ▼
trace / drop monitor / debugging
```

The important architectural change is that **drop cause becomes
structured kernel data** instead of requiring inference from packet
traces or scattered tracepoints.

This is highly relevant to production debugging of:

-   TCP state-machine drops;
-   routing/input validation;
-   firewall/stack behavior;
-   performance loss.

It should therefore be treated as a networking observability milestone.

------------------------------------------------------------------------

## 39.6 2022 --- in-kernel TLS handshake

LWN:

-   *Extending in-kernel TLS support*
-   2022-04-25
-   https://lwn.net/Articles/892216/
-   *Adding an in-kernel TLS handshake*
-   2022-06-01
-   https://lwn.net/Articles/896746/

Linux already had KTLS record-layer support, but initiating TLS from
kernel consumers such as NFS/NVMe was difficult because handshake logic
remained in userspace.

Existing model:

``` text
userspace
  │ TLS handshake
  ▼
established TLS socket
  │
  └── hand socket to kernel consumer
```

Desired model:

``` text
kernel consumer (NFS/NVMe/...)
       │
       ▼
kernel requests handshake
       │
       ├── kernel TLS integration
       └── userspace assistance where needed
```

This is a significant socket/security API evolution even though the
cryptographic handshake is not simply "move all TLS code into kernel".

The LSFMM discussion also noted possible relevance to future
QUIC-related kernel work.

------------------------------------------------------------------------

## 39.7 2024 --- P4TC as an important *non-merged* architecture proposal

LWN:

-   *P4TC hits a brick wall*
-   2024-06-10
-   https://lwn.net/Articles/977310/

P4TC proposed integrating P4-programmable packet processing with Linux
traffic control.

Conceptually:

``` text
P4 description
     │
     ▼
P4TC objects / pipeline
     │
     ▼
Linux TC datapath
```

The proposal had been under review since early 2023, but LWN reported
substantial maintainer objections and a stalled merge path.

This entry is deliberately classified:

``` text
status: important design effort / not mainline milestone
```

Why include it?

Because later BPF qdisc and BPF/TC programmability discussions make more
sense when viewed alongside the kernel community's concerns around
introducing another programmable network pipeline/API.

------------------------------------------------------------------------

## 39.8 2025 --- BPF qdisc with `struct_ops`

Patch series:

-   `[PATCH bpf-next v6 00/11] bpf qdisc`
-   2025-03-19
-   https://lwn.net/Articles/1014971/

The design uses BPF `struct_ops` to implement traffic-control queueing
disciplines.

``` text
traditional qdisc:

kernel C qdisc
   │
enqueue/dequeue
   │
scheduler


BPF qdisc:

BPF struct_ops
   │
enqueue/dequeue policy
   │
TC qdisc framework
```

v6 intentionally kept the first version minimal:

-   attach only at root or `mq`;
-   classful qdisc support deferred;
-   direct `bpf_list` / `bpf_rbtree` skb support deferred.

This directly extends the idea first demonstrated by BPF
`tcp_congestion_ops` in Linux 5.6:

``` text
BPF struct_ops
    ├── TCP congestion control
    └── qdisc / packet scheduling
```

LWN's 2025 merge-window coverage records BPF-implemented qdisc support
among networking changes:

-   https://lwn.net/Articles/1023924/

------------------------------------------------------------------------

## 39.9 2025 --- DCCP removal

Patch series:

-   `[PATCH v1 net-next 0/4] net: Retire DCCP.`
-   2025-04-07
-   https://lwn.net/Articles/1016830/

LWN's merge-window coverage later records DCCP removal:

-   https://lwn.net/Articles/1023924/

The series removes:

-   `net/dccp`;
-   associated netfilter/LSM integration;
-   documentation;
-   DCCP-specific shared networking code.

UAPI headers were deliberately retained.

The removal is architecturally interesting because TCP and DCCP shared
infrastructure. Once DCCP is gone, code such as:

``` text
tcp_or_dccp_get_hashinfo()
```

can become TCP-specific, enabling further cleanup.

This is a useful example of **network-stack simplification by protocol
removal**, not just feature addition.

------------------------------------------------------------------------

# 40. Completeness audit: classification table

  ------------------------------------------------------------------------
  Topic                      Year Classification       Why it matters
  ------------------ ------------ -------------------- -------------------
  IPv6                       2020 design/API           protocol
  extension-header                discussion           extensibility vs
  processing                                           fast path/offload

  Threaded NAPI              2020 merged architecture  moves RX polling
                                  direction            out of softirq
                                                       context

  bpfilter rethink           2020 failed/stalled       BPF firewall
                                  design               architecture
                                                       history

  SO_REUSEPORT               2021 socket semantics     scalable listener
  failover                                             behavior

  skb drop reasons           2022 observability        structured
                                                       packet-drop
                                                       diagnostics

  in-kernel TLS              2022 API/architecture     kernel-originated
  handshake                                            secure transports

  P4TC                   2023--24 significant          programmable TC
                                  non-merged proposal  pipeline debate

  BPF qdisc                  2025 merged/programming   `struct_ops`
                                  model                reaches packet
                                                       scheduling

  DCCP removal               2025 removal/cleanup      simplifies shared
                                                       transport code
  ------------------------------------------------------------------------

------------------------------------------------------------------------

# 41. Audit finding: article list vs kernel-change list

The document now intentionally contains **two overlapping indexes**.

## Kernel-change index

Use this when the question is:

> "What changed in Linux networking?"

It prioritizes merged changes and release mapping.

## LWN-history index

Use this when the question is:

> "What networking developments did LWN cover?"

It additionally includes:

-   RFCs;
-   stalled proposals;
-   architectural debates;
-   removals;
-   user-space/kernel-interface milestones.

This distinction prevents a common completeness error:

``` text
LWN networking article
       ≠
mainline kernel feature
```

Examples:

``` text
P4TC article
    → important LWN networking history
    → not a mainline feature

Device Memory TCP
    → LWN development history
    → later mainline feature

DCCP removal
    → mainline change
    → removal rather than feature addition
```

------------------------------------------------------------------------

# 42. Next completeness-audit pass

The next audit should focus on categories that broad year searches can
miss:

1.  routing / FIB / nexthop API;
2.  netlink/YNL specification work;
3.  TC actions / qdisc / hardware offload;
4.  bridge / VLAN / multicast;
5.  IPv6 IOAM / SRv6;
6.  socket APIs and `SO_*` / `TCP_*` additions;
7.  network namespaces;
8.  MCTP and other new protocol families;
9.  TLS / QUIC / encrypted transports;
10. removal/deprecation stories;
11. Wi-Fi/mac80211 only where core networking architecture is affected;
12. Rust networking abstractions only where they affect netdev core.

After that category audit, the article inventory can be compared against
the release matrix to produce a final "coverage gaps" section.

------------------------------------------------------------------------

# 43. Completeness audit pass 2 --- routing, Netlink/YNL, bridge, MCTP, SRv6, socket APIs, QUIC

## 43.1 2019 --- nexthop objects and FIB scalability

David Ahern's nexthop-object work separates the nexthop lifecycle from
route-prefix objects.

Final initial series:

-   `[PATCH v3 net-next 00/20] net: Enable nexthop objects with IPv4 and IPv6 routes`
-   2019-06-07
-   https://lwn.net/Articles/790828/

The earlier RFC explains the model succinctly:

``` text
traditional route

prefix
  └── gateway + device + encapsulation inline


nexthop-object model

prefix
  │
  └── nexthop ID
          │
          └── gateway + device + encapsulation
```

Motivations:

-   avoid repeatedly validating identical nexthop information;
-   reduce excessive RCU synchronization during large route installs;
-   align the kernel model with routing daemons and switch ASICs;
-   allow independent nexthop lifecycle and groups.

The series cites a full IPv4 feed of roughly 700k routes as a motivating
scalability case.

This is a major FIB/control-plane API milestone and is now tracked
separately from generic "routing improvements".

------------------------------------------------------------------------

## 43.2 2022--2024 --- Netlink specifications and YNL

Netlink historically has a large amount of manually maintained:

-   UAPI definitions;
-   attribute policies;
-   kernel operation tables;
-   userspace encoders/decoders;
-   documentation.

Jakub Kicinski's YAML specification work changes this model.

### 2022 documentation groundwork

-   *docs: netlink: basic introduction to Netlink*
-   https://lwn.net/Articles/905079/

### 2023 mergeable protocol-spec series

-   `[PATCH net-next v3 0/8] Netlink protocol specs`
-   https://lwn.net/Articles/920499/

The series adds:

-   YAML schemas for Netlink specifications;
-   kernel C code generators;
-   FOU protocol specification;
-   generated UAPI/policy/operation tables;
-   generic YNL userspace client;
-   documentation.

Architecture:

``` text
before:

UAPI headers ─┐
policy tables ─┼── manually synchronized
docs ─────────┤
userspace ────┘


YNL/spec model:

          YAML protocol spec
           /      |       \
          ▼       ▼        ▼
      kernel C   docs    userspace client
```

This is one of the most important networking-UAPI maintainability
changes in the period.

### Follow-up adoption

MPTCP conversion:

-   https://lwn.net/Articles/947314/

nftables specification + transactional multi-message support:

-   https://lwn.net/Articles/969496/

The kernel Netlink Handbook now documents the specification format, code
generation, and YNL library/client.

------------------------------------------------------------------------

## 43.3 2021 --- bridge per-VLAN multicast snooping

Nikolay Aleksandrov's series adds per-VLAN multicast contexts to Linux
bridge.

Kernel series:

-   `[PATCH net-next 00/15] net: bridge: multicast: add vlan support`
-   https://lwn.net/Articles/863487/

Userspace/global-options series:

-   `[PATCH iproute2-next v2 00/19] bridge: vlan: add global multicast options`
-   https://lwn.net/Articles/867804/

Before:

``` text
bridge
  │
  └── one multicast context
```

After:

``` text
bridge
  │
  ├── VLAN 10 multicast context
  ├── VLAN 20 multicast context
  └── VLAN 30 multicast context
```

The implementation introduced context pointers so packet processing can
switch between bridge-wide and per-VLAN multicast state when VLAN
snooping is enabled.

This is relevant to Kubernetes/OpenShift L2 networking and
virtual-switch behavior because multicast policy/state can be isolated
by VLAN rather than globally across the bridge.

------------------------------------------------------------------------

## 43.4 2021 --- Management Component Transport Protocol (MCTP)

Initial MCTP series:

-   https://lwn.net/Articles/858176/
-   later revision: https://lwn.net/Articles/864175/

The series introduces a complete new protocol family for
platform-management traffic.

Key components:

``` text
AF_MCTP
SOCK_DGRAM
    │
    ├── sockaddr_mctp
    ├── routing
    ├── neighbour table
    ├── fragmentation/reassembly
    ├── netlink management
    └── physical transport bindings
```

Kernel documentation describes MCTP interfaces as `struct netdevice`
instances and MCTP networks as separate endpoint-ID address spaces.

This is not Internet packet forwarding; it is
management-controller/device communication, but it is a genuine Linux
networking protocol stack and therefore belongs in a broad networking
history.

### Follow-ups

2022:

-   tag-control API: https://lwn.net/Articles/884100/

2025:

-   gateway routing: https://lwn.net/Articles/1026183/

2026:

-   MCTP over Platform Communication Channel (PCC):
    https://lwn.net/Articles/1061131/

------------------------------------------------------------------------

## 43.5 SRv6 evolution after initial mainline support

SRv6 itself predates this document's start date (initial Linux support
appeared in Linux 4.10), but substantial capability growth occurs inside
the audit period.

### 2022 --- Headend Reduced

-   `[net-next v5 0/4] seg6: add support for SRv6 Headend Reduced`
-   https://lwn.net/Articles/902806/

Reduced encapsulation avoids carrying an unnecessary first segment in
the SRH in cases where the IPv6 destination already represents it.

### 2023 --- PSP flavor

-   `seg6: add PSP flavor support for SRv6 End behavior`
-   https://lwn.net/Articles/923380/

### 2023 --- NEXT-C-SID

-   `seg6: add NEXT-C-SID support for SRv6 End.X behavior`
-   https://lwn.net/Articles/939830/

Compressed SID mechanisms address the overhead of carrying many 128-bit
SIDs in an SRH.

### 2026 --- Mobile User Plane

-   `seg6: add SRv6 Mobile User Plane (RFC 9433) behaviors`
-   https://lwn.net/Articles/1070981/

### 2026 --- L2 VPN RFC

-   End.DT2U + `srl2` Ethernet pseudowire device
-   https://lwn.net/Articles/1064184/

Status distinction is important: the 2026 L2 VPN item is an RFC series,
not automatically a merged feature.

------------------------------------------------------------------------

## 43.6 2023 --- `SCM_PIDFD` and `SO_PEERPIDFD`

Alexander Mikhalitsyn's series adds pidfd-based peer/process
identification to Unix/socket APIs.

Selected revisions:

-   v1: https://lwn.net/Articles/926312/
-   v3: https://lwn.net/Articles/928752/
-   v7: https://lwn.net/Articles/934278/

APIs:

``` text
SCM_CREDENTIALS
      │
      └── plain PID
              │
              └── PID reuse ambiguity


SCM_PIDFD
      │
      └── pidfd


SO_PEERCRED
      │
      └── peer PID/credentials

SO_PEERPIDFD
      │
      └── peer pidfd
```

This is especially useful for long-lived service/socket relationships
where PID reuse makes a numeric PID a weak identity token.

------------------------------------------------------------------------

## 43.7 2024--2026 --- in-kernel QUIC

### 2024 initial implementation proposal

-   `[PATCH net-next 0/5] net: implement the QUIC protocol in linux kernel`
-   2024-09-09
-   https://lwn.net/Articles/989623/

### 2025 LWN feature

-   *QUIC for the kernel*
-   2025-07-22
-   https://lwn.net/Articles/1029851/

The motivation is not primarily to replace userspace QUIC
implementations used by browsers. Kernel consumers such as SMB/NFS need
an in-kernel secure, multiplexed transport API.

QUIC provides:

-   UDP-based transport;
-   integrated encryption;
-   stream multiplexing;
-   low-latency connection establishment;
-   path/connection migration.

### 2025 redesign

The later series splits out core infrastructure/subcomponents:

-   v1: https://lwn.net/Articles/1028932/
-   v3: https://lwn.net/Articles/1038836/
-   v5: https://lwn.net/Articles/1047727/

The v3/v5 design integrates with `net/handshake` and exposes familiar
socket semantics such as:

``` text
listen()
accept()
connect()
sendmsg()
recvmsg()
getsockopt()
setsockopt()
```

Classification in this document:

``` text
2024–2026:
important active development series

NOT assumed merged merely because LWN covered it
```

This distinction is essential for the final completeness table.

------------------------------------------------------------------------

## 43.8 2026 --- BPF access across network namespaces

LWN:

-   *Examining other network namespaces using BPF*
-   2026-08-05
-   https://lwn.net/Articles/1085896/

The motivating Cilium use case is socket-level load balancing where a
sufficiently privileged BPF program needs to inspect sockets belonging
to another network namespace.

Conceptual issue:

``` text
BPF program in context A
        │
        X traditional namespace boundary
        │
        ▼
sockets in netns B
```

The discussion explores mechanisms for controlled cross-netns inspection
rather than simply weakening network-namespace isolation.

This belongs alongside the netkit/KubeVirt work because both expose a
broader 2026 theme:

``` text
retain namespace isolation
        +
allow explicitly delegated high-performance / observability operations
```

------------------------------------------------------------------------

# 44. Updated completeness matrix

  Area                 Important audit additions
  -------------------- -----------------------------------------------
  Routing/FIB          nexthop objects
  Netlink/UAPI         YAML protocol specs, YNL, generated APIs/docs
  Bridge               per-VLAN multicast snooping
  Platform protocols   MCTP / AF_MCTP
  IPv6/SRv6            Headend Reduced, PSP, NEXT-C-SID, MUP
  Socket API           SCM_PIDFD, SO_PEERPIDFD
  Secure transports    KTLS handshake, in-kernel QUIC development
  Namespaces/BPF       cross-netns socket inspection discussion
  Packet scheduling    BPF qdisc, P4TC
  Observability        skb_drop_reason
  RX execution         threaded NAPI

------------------------------------------------------------------------

# 45. Coverage status

After this pass, the largest remaining audit gaps are narrower:

1.  IPv6 IOAM;
2.  TC action/offload evolution beyond BPF qdisc/P4TC;
3.  socket-memory/default-buffer and socket API additions;
4.  routing-policy and FIB changes after nexthop objects;
5.  network-device configuration API (`ethtool` netlink, devlink where
    networking-core relevant);
6.  TLS/KTLS follow-ups;
7.  protocol removals/deprecations beyond DCCP;
8.  2025--2026 article-by-article final sweep;
9.  exact mapping of every retained LWN article to
    merged/RFC/stalled/removed status.

The final audit should then generate a canonical article inventory with
fields:

``` text
date
title
LWN URL
category
kernel release
status
upstream series
mainline commit(s)
tags
```

and compare that inventory against the thematic chapters above.

------------------------------------------------------------------------

# 46. Completeness audit pass 3 --- IOAM, ethtool-netlink, TC offload, socket memory, KTLS, removals

## 46.1 2021 --- IPv6 IOAM

Initial upstream series:

-   `[PATCH net-next v4 0/5] Support for the IOAM Pre-allocated Trace with IPv6`
-   https://lwn.net/Articles/857497/
-   v5: https://lwn.net/Articles/863746/

IOAM (In-situ Operations, Administration, and Maintenance) carries
telemetry inside packets as they traverse the network.

Conceptually:

``` text
packet
  │
  ├── node A appends telemetry
  ├── node B appends telemetry
  ├── node C appends telemetry
  ▼
receiver / collector
```

The Linux implementation includes IPv6 IOAM trace handling,
namespace/schema configuration, Generic Netlink control, sysctls, and
selftests.

Linux 5.16 then enhanced IOAM with encapsulation support for in-transit
packets.

LWN merge-window: https://lwn.net/Articles/874683/

This is different from SRv6: SRv6 primarily encodes
forwarding/service-path instructions, whereas IOAM carries
operational/telemetry information. They may coexist in IPv6 networks but
solve different problems.

------------------------------------------------------------------------

## 46.2 2019--2025 --- ethtool ioctl → Generic Netlink

The ethtool userspace/kernel interface historically used ioctl
structures.

Problems identified by the netlink series:

-   poor extensibility;
-   GET-modify-SET races;
-   limited error reporting;
-   no multicast notifications;
-   difficulty dumping state for many devices.

Important in-scope series:

-   2019-07 v6: https://lwn.net/Articles/792611/
-   2019-12 v8: https://lwn.net/Articles/808028/

New architecture:

``` text
old:

ethtool
   │ ioctl
   ▼
fixed UAPI structs


new:

ethtool / NetworkManager / systemd-networkd / ...
   │
Generic Netlink family "ethtool"
   │
   ├── GET
   ├── SET
   ├── ACT
   ├── dumps
   ├── extended ACK
   └── multicast notifications
```

Kernel code was split into `net/ethtool/`, with dedicated netlink
handlers and documentation.

### Continued migration

The conversion was intentionally incremental.

Linux 5.16-era work added transceiver-module control through ethtool
netlink.

By 2025 RSS configuration was being completed over Netlink:

-   RSS_SET: https://lwn.net/Articles/1029617/
-   RSS context create/remove: https://lwn.net/Articles/1030503/

The latter series states that, for RSS configuration, all functionality
available via ioctl had then become available through Netlink.

### Relationship to YNL

The later YNL/specification work changes the implementation again:

``` text
ethtool Generic Netlink
       │
       ▼
YAML protocol specification
       │
       ├── generated UAPI/policy
       ├── generated tooling
       └── documentation
```

Recent ethtool patches modify:

``` text
Documentation/netlink/specs/ethtool.yaml
```

rather than treating UAPI definitions and docs as unrelated
hand-maintained artifacts.

------------------------------------------------------------------------

# 47. TC / hardware-offload evolution

## 47.1 Shared `flow_rule` / `flow_action` representation

A key precursor predates the start date but is essential context:
drivers were moved away from parsing TC-native action layouts directly
toward common `flow_rule` / `flow_action` structures.

The architectural result is:

``` text
TC flower ───────┐
                 │
ethtool RX NFC ──┼──► flow_rule / flow_action ──► NIC driver
                 │
netfilter ───────┘       (later integration)
```

This common representation reduces duplicate rule parsers in drivers and
enables multiple kernel subsystems to share hardware-offload
infrastructure.

Archive: https://lwn.net/Articles/775046/

------------------------------------------------------------------------

## 47.2 2021 --- standalone TC action hardware offload

Series:

-   `[PATCH v7 net-next 00/12] allow user to offload tc action to net device`
-   2021-12-17
-   https://lwn.net/Articles/879034/

Before, hardware actions were primarily offloaded as part of a
flow/filter.

The new model allows an action instance to have an independent
lifecycle:

``` text
TC action instance
      │
      ├── flow A references it
      ├── flow B references it
      └── flow C references it
      │
      ▼
hardware action object
```

The motivating example was OVS metering using a shared police action.

The series also adds:

-   `skip_hw` / `skip_sw`;
-   hardware action statistics;
-   `in_hw_count`;
-   reoffload when drivers appear/disappear;
-   selftests.

------------------------------------------------------------------------

## 47.3 2023 --- partial hardware offload and software continuation

Series:

-   `net/sched: cls_api: Support hardware miss to tc action`
-   https://lwn.net/Articles/920501/

This handles rules where only part of an action list can execute in
hardware.

``` text
TC rule

action A ──► action B ──► action C
   │ HW         │ HW          │ unsupported
   └────────────┴───── miss ───┘
                         │
                         ▼
                  continue in software
                  at specific action
```

This is important because real TC/OVS pipelines are rarely
all-or-nothing offloadable.

------------------------------------------------------------------------

# 48. Socket memory and receive-buffer evolution

## 48.1 Linux 5.16 --- `SO_RESERVE_MEM`

LWN 5.16 merge-window:

https://lwn.net/Articles/874683/

`SO_RESERVE_MEM` lets users reserve kernel memory for a socket.

Purpose:

``` text
normal socket
    │
memory pressure
    │
allocation may become difficult


SO_RESERVE_MEM socket
    │
reserved memory
    │
network operation can proceed more predictably
```

The feature is tied to memory cgroups; reserved memory is charged
against the cgroup quota.

This belongs in the networking change log because it changes
socket-memory guarantees, rather than simply tuning a sysctl.

------------------------------------------------------------------------

## 48.2 2025 --- TCP receive-side autotuning work

Eric Dumazet's 2025 series:

-   `[PATCH net-next 00/11] tcp: receive side improvements`
-   https://lwn.net/Articles/1021321/

The cover letter notes that Google had used a 15MB `tcp_rmem[2]` for
years but that high-speed/small-RTT flows exposed overestimation in TCP
RX autotuning.

The work addresses receive-side sizing/accounting rather than simply
"make buffers larger".

This is relevant to the conceptual distinction:

``` text
net.core.rmem_default / rmem_max
        │
        └── generic socket limits/defaults

tcp_rmem
        │
        └── TCP autotuning parameters

sk_rcvbuf / rcvq_space
        │
        └── per-socket runtime state
```

These should not be treated as interchangeable knobs.

------------------------------------------------------------------------

## 48.3 Linux 6.18 --- default socket receive buffer raised to 4MB

LWN's 6.18 merge-window explicitly records:

> the default socket receive buffer size has been raised to 4MB

https://lwn.net/Articles/1040203/

This is recorded separately from TCP autotuning because the generic
socket default and TCP's dynamic receive-window/buffer logic are
distinct layers.

------------------------------------------------------------------------

# 49. KTLS / kernel-handshake follow-up

Earlier chapters covered the 2022 effort to let kernel socket consumers
request TLS handshakes.

The generic handshake upcall mechanism reached a mature v8 series in
2023:

-   `[PATCH v8 0/4] Another crack at a handshake upcall mechanism`
-   https://lwn.net/Articles/928240/

Purpose:

``` text
kernel socket consumer
(NFS / NVMe / etc.)
        │
        ▼
generic handshake request
        │
        ▼
userspace TLS policy/handshake helper
        │
        ▼
KTLS-enabled kernel socket
```

This avoids requiring each kernel consumer to invent a separate
userspace upcall protocol.

### NVMe/TCP TLS

After the handshake upcall and `tls_read_sock()` work landed, NVMe/TCP
could build on the common infrastructure.

-   `[PATCHv7 00/17] nvme: In-kernel TLS support for TCP`
-   2023-08
-   https://lwn.net/Articles/941139/

### 2026 continuation

Kernel consumers of `read_sock` still had limitations around TLS control
records.

2026 series:

-   `Deliver TLS control records to kernel read_sock consumers`
-   https://lwn.net/Articles/1083795/

This demonstrates that KTLS evolution during the period is not a single
feature merge but an ongoing effort to make encrypted sockets behave
like ordinary kernel-consumable transport streams.

------------------------------------------------------------------------

# 50. Protocol retirement as network-stack optimization

## 50.1 DECnet --- Linux 6.1

LWN 6.1 merge-window:

https://lwn.net/Articles/910312/

DECnet protocol support was removed while UAPI definitions were retained
so existing source code can still compile.

This establishes a pattern later seen with other retired protocols:

``` text
remove implementation
       │
       ├── eliminate maintenance/security surface
       ├── simplify shared fast-path code
       └── sometimes retain UAPI for build compatibility
```

------------------------------------------------------------------------

## 50.2 DCCP --- 2025

Covered earlier.

DCCP removal is particularly useful because TCP/DCCP shared code could
become TCP-specific after the protocol disappeared.

------------------------------------------------------------------------

## 50.3 UDP-Lite --- Linux 7.1

Removal series:

-   `[PATCH v2 net-next 00/15] udp: Retire UDP-Lite`
-   2026-03-05
-   https://lwn.net/Articles/1061588/

LWN 7.1 merge-window confirms removal:

https://lwn.net/Articles/1067250/

The cover letter gives unusually useful performance data.

Without FDO:

``` text
13.3 Mpps → 14.7 Mpps
≈ 10% increase
```

With FDO:

``` text
20.1 Mpps → 20.7 Mpps
≈ 3% increase
```

for the author's `udp_rr` workload with 20,000 flows.

Why can removing an unused protocol improve UDP?

UDP-Lite shared many conditionals and helper paths with normal UDP:

``` text
UDP RX/TX fast path
      │
      ├── UDP?
      └── UDP-Lite?
             │
             └── checksum-coverage special cases
```

Removing UDP-Lite eliminates branches, separate tables/helpers,
partial-checksum logic, and code footprint from the common UDP path.

This is a strong example of **negative code as networking performance
work**.

------------------------------------------------------------------------

# 51. Audit synthesis --- three recurring modernization patterns

The newly audited items reveal three broad patterns.

## 51.1 Fixed UAPI → extensible, generated Netlink

``` text
ioctl structs
    │
    ▼
Generic Netlink
    │
    ▼
YAML/YNL generated specification
```

Examples:

-   ethtool;
-   MPTCP;
-   nftables;
-   other netdev APIs.

## 51.2 Software-only pipeline → common representation → partial hardware offload

``` text
TC-specific representation
      │
      ▼
flow_rule / flow_action
      │
      ├── TC
      ├── ethtool
      └── netfilter
      │
      ▼
NIC hardware
      │
      └── software continuation on miss
```

## 51.3 Add protocols → later prune unused protocol complexity

``` text
large monolithic network stack
      │
      ├── DECnet removed
      ├── DCCP removed
      └── UDP-Lite removed
             │
             ▼
smaller shared fast paths
```

The UDP-Lite case demonstrates that removal can produce measurable
packet-rate gains.

------------------------------------------------------------------------

# 52. Remaining audit before canonical inventory

At this point the broad architectural categories are substantially
covered.

One final sweep remains for:

1.  `devlink` changes that affect netdev architecture rather than
    individual drivers;
2.  route/FIB policy changes after nexthop objects;
3.  IPv6 and TCP sysctl/socket-option additions;
4.  major GRO/GSO/NAPI/XDP changes not already attached to BIG
    TCP/page_pool;
5.  removals/deprecations announced but not yet represented;
6.  2026 articles through 2026-10-02;
7.  duplicate and status reconciliation.

After that, generate the canonical inventory:

  -----------------------------------------------------
  Date LWN Category Kernel Status Series Commits Tags
  title
  -----------------------------------------------------

------------------------------------------------------------------------

Status values will be normalized to:

``` text
merged
merged-follow-up
RFC
under-review
stalled
removed
userspace-milestone
design-discussion
```

This table will become the basis for the final completeness check.

------------------------------------------------------------------------

# 53. Final sweep --- late socket, multipath, tunnel, and removal items

## 53.1 Linux 6.15 --- `TCP_RTO_MAX_MS`

Eric Dumazet's series:

-   `[PATCH net-next 0/5] tcp: allow to reduce max RTO`
-   https://lwn.net/Articles/1008854/

adds:

``` text
TCP_RTO_MAX_MS          per-socket option
tcp_rto_max_ms          per-network-namespace sysctl
```

This addresses applications that need a smaller maximum retransmission
timeout than the traditional TCP ceiling. `TCP_KEEPINTVL` and
`TCP_KEEPCNT` do not solve this problem because keepalive timers and
retransmission backoff are different mechanisms.

Affected areas include:

-   `include/uapi/linux/tcp.h`
-   `net/ipv4/tcp.c`
-   `net/ipv4/tcp_timer.c`
-   `net/ipv4/tcp_output.c`
-   IPv4 sysctl documentation

LWN's Linux 6.15 merge-window summary records the feature.

-   https://lwn.net/Articles/1015414/

------------------------------------------------------------------------

## 53.2 Linux 6.15 --- BPF network timestamp callbacks

The 6.15 merge window also added BPF callbacks for obtaining timestamps
at multiple points in the network stack.

LWN: https://lwn.net/Articles/1015414/

Primary use case:

``` text
packet enters networking path
        │
        ├── timestamp A
        ▼
processing stage
        │
        ├── timestamp B
        ▼
later stage
        │
        ├── timestamp C
        ▼
packet exits
```

This makes it possible to measure latency *inside* the network stack
rather than only observing ingress/egress timestamps.

It belongs beside `skb_drop_reason` as an observability milestone:

``` text
skb_drop_reason
    → why was the packet dropped?

BPF network timestamps
    → where did the packet spend time?
```

------------------------------------------------------------------------

## 53.3 2025 --- local TCP multipath route selection

Willem de Bruijn's series:

-   `[PATCH net-next 0/3] ip: improve tcp sock multipath routing`
-   https://lwn.net/Articles/1018316/
-   v2: https://lwn.net/Articles/1019096/

addresses local TCP connections under layer-4 multipath hash policies.

The series fixes/changes three aspects:

1.  source-address selection should match the selected nexthop device;
2.  multiple local TCP connections should use all available paths;
3.  connected sockets should not be rehashed unexpectedly by unrelated
    socket-state changes.

This is a useful follow-up to the earlier nexthop-object/FIB work:

``` text
nexthop objects
      │
      └── scalable representation/control plane

multipath hash policy
      │
      └── which nexthop a flow actually selects
```

------------------------------------------------------------------------

## 53.4 2026 --- double UDP tunnel GRO/GSO

Paolo Abeni's series:

-   `[PATCH v5 net-next 00/10] geneve: introduce double tunnel GSO/GRO support`
-   2026-01-21
-   https://lwn.net/Articles/1055518/

explicitly targets container orchestration running inside virtual
environments, where double UDP encapsulation --- particularly GENEVE ---
is common.

Typical topology:

``` text
workload/container
       │
inner overlay tunnel
       │ GENEVE/VXLAN
       ▼
VM virtual NIC
       │
outer virtual/cloud tunnel
       │ GENEVE
       ▼
physical network
```

Without support for nested encapsulation, GRO/GSO aggregation is lost
for inter-VM traffic and packets are segmented too early.

The series adds:

-   generalized device GSO admission features;
-   GSO-partial support for GENEVE/VXLAN;
-   GENEVE option/hint for double-tunnel GRO;
-   selftests.

Both GSO partial and double-encapsulation GRO are disabled by default
and require explicit configuration.

This is separate from, but closely related to, BIG TCP over UDP tunnels:

``` text
double-tunnel GSO/GRO
       │
       └── preserve ordinary segmentation aggregation across nested tunnels

BIG TCP over VXLAN/GENEVE
       │
       └── allow >64KiB BIG-TCP aggregates through UDP tunnels
```

For OpenShift/OVN/KubeVirt this is one of the most directly relevant
late-period changes.

------------------------------------------------------------------------

# 54. 2026 merge-window reconciliation

A final pass over LWN's 2026 merge-window summaries found additional
items that need to appear in the canonical inventory even when they do
not require full thematic chapters.

## Linux 7.0

LWN records:

-   AccECN enabled for general use;
-   CAKE multi-queue support;
-   VSOCK network-namespace support.

Merge-window: https://lwn.net/Articles/1058664/

### CAKE multiqueue

The CAKE qdisc can distribute shaping work across multiple queues/CPUs.
This is a scalability evolution of an existing sophisticated qdisc
rather than a new qdisc API.

### VSOCK network namespaces

VSOCK is commonly used for host/guest communication. Adding
network-namespace awareness makes it fit containerized virtualization
environments more naturally.

------------------------------------------------------------------------

## Linux 7.1

LWN records:

-   Unix-domain socket `user.*` xattrs;
-   UDP-Lite removal;
-   IPv6 can no longer be built as a module;
-   large removal of legacy networking subsystems/drivers in the latter
    half.

First half: https://lwn.net/Articles/1067250/

Rest: https://lwn.net/Articles/1067785/

The large removal wave includes ATM, AX.25/amateur-radio pieces, ISDN,
Bluetooth CMTP, CAIF, and numerous old drivers.

This is kept as a **removal/maintenance milestone**, not as one
networking feature.

------------------------------------------------------------------------

## Linux 7.2

LWN records:

-   TCP Authentication Option implementation moved to libcrypto;
-   MPTCP maximum subflows increased from 8 to 64;
-   continued RTNL-lock reduction;
-   AppleTalk and additional legacy network support removed.

Merge-window: https://lwn.net/Articles/1078068/

### TCP-AO → libcrypto

TCP-AO itself entered the kernel earlier; the 7.2 change is an
implementation modernization:

``` text
TCP-AO
   │
older crypto integration
   ▼
kernel libcrypto
   │
   ├── simpler code
   └── fewer supported/unused algorithms
```

### MPTCP 8 → 64 subflows

This is a useful scalability milestone and belongs in the
release-by-release MPTCP map.

------------------------------------------------------------------------

## Linux 7.3

As of 2026-10-02 Linux 7.3 is still in the release-candidate phase; the
merge window is complete but the final release has not yet occurred.

LWN's merge-window summary records:

-   BIG TCP over VXLAN and GENEVE.

https://lwn.net/Articles/1089791/

Therefore the canonical inventory status should say:

``` text
kernel: 7.3
status: merged in 7.3 development tree / final release pending as of 2026-10-02
```

rather than simply "Linux 7.3 released".

------------------------------------------------------------------------

# 55. Canonical LWN networking inventory

The following table is the normalized inventory assembled from the
thematic research and the completeness audits. It is intentionally
focused on **architectural/core networking coverage** rather than every
individual NIC-driver posting.

Legend:

``` text
M   merged/mainline milestone
F   merged follow-up
R   RFC / under review
D   design discussion
S   stalled/non-merged direction
X   removal/deprecation
U   userspace/kernel-interface milestone
```

  ----------------------------------------------------------------------------------------------------------
  Date /       LWN topic          Category             Kernel / era    Status              Tags
  period                                                                                   
  ------------ ------------------ -------------------- --------------- ------------------- -----------------
  2019-05      Memory management  RX memory            5.x             D/F                 page_pool, RX,
               for 400Gb/s                                                                 memory
               interfaces                                                                  

  2019-05      BPF: what's good,  BPF                  5.x             D                   BPF, XDP
               what's coming, and                                                          
               what's needed                                                               

  2019-06      The TCP SACK panic TCP/security         5.2             M/fix               TCP, SACK

  2019-06      Providing wider    BPF/security         5.x             D                   BPF, privilege
               access to `bpf()`                                                           

  2019-06      nexthop objects    routing/FIB          5.x             M                   FIB, nexthop
               with IPv4/IPv6                                                              
               routes                                                                      

  2019-09      Upstreaming        transport            5.6-era         D/M                 MPTCP
               multipath TCP                                                               

  2019-10      WireGuard and the  VPN/crypto           5.6             D/M                 WireGuard
               crypto API                                                                  

  2019-12      ethtool Netlink v8 UAPI                 5.x             M                   ethtool, netlink

  2020-01      The trouble with   IPv6                 5.x             D                   IPv6, EH
               IPv6 extension                                                              
               headers                                                                     

  2020-01      Accelerating       netfilter/offload    5.3+            M                   nftables,
               netfilter with                                                              flowtable
               hardware offload,                                                           
               parts 1/2                                                                   

  2020-02      Kernel operations  BPF                  5.6             M                   struct_ops, TCP
               structures in BPF                                                           CC

  2020-06      Rethinking         firewall/BPF         ---             S/D                 bpfilter
               bpfilter and                                                                
               user-mode helpers                                                           

  2020-08/10   NAPI polling in    RX execution         5.x             M                   NAPI, softirq
               kernel threads                                                              

  2020-08      End-to-end network BPF/P4               5.x             D                   BPF,
               programmability                                                             programmability

  2021-03      BPF meets io_uring BPF/io_uring         5.x             D                   BPF, io_uring

  2021-04      Avoiding           sockets              5.x             M/F                 SO_REUSEPORT
               unintended                                                                  
               connection                                                                  
               failures with                                                               
               SO_REUSEPORT                                                                

  2021-05      Calling kernel     BPF                  5.x             M                   kfunc
               functions from BPF                                                          

  2021         bridge per-VLAN    bridge               5.x             M                   bridge, VLAN,
               multicast                                                                   multicast

  2021         MCTP core protocol protocol             5.x             M                   AF_MCTP
               stack                                                                       

  2021-08      Nftables reaches   firewall             5.x             U                   nftables
               1.0                                                                         

  2021-12      Zero-copy network  zero-copy            6.0-era         D/M                 io_uring, ZC TX
               transmission with                                                           
               io_uring                                                                    

  2021-12      standalone TC      TC/offload           5.x             M                   TC, HW offload
               action hardware                                                             
               offload                                                                     

  2021         IPv6 IOAM          telemetry            5.15/5.16       M                   IOAM, IPv6

  2022-02      Going big with TCP TCP/performance      5.19            M                   BIG TCP
               packets                                                                     

  2022-02      Better visibility  observability        5.x             M                   skb_drop_reason
               into                                                                        
               packet-dropping                                                             
               decisions                                                                   

  2022-04      Extending          TLS                  5.x             D/M                 KTLS
               in-kernel TLS                                                               
               support                                                                     

  2022-06      Adding an          TLS                  5.x             D                   net/handshake
               in-kernel TLS                                                               
               handshake                                                                   

  2022         BPF conntrack      conntrack/BPF        6.x             M                   conntrack, BPF
               lifecycle kfuncs                                                            

  2022         delayed TCP        conntrack            6.x             F                   conntrack, TCP
               packets and CT                                                              
               timeout refresh                                                             

  2022         SRv6 Headend       SRv6                 6.x             M                   SRv6
               Reduced                                                                     

  2022-10      TCP source-port    TCP                  6.x             D                   TCP, privacy
               fingerprinting                                                              

  2022-11      Moving past TCP in transport            6.x             D                   datacenter
               the data center,                                                            
               part 2                                                                      

  2023         IPv4 BIG TCP       TCP/performance      6.3             M                   BIG TCP, IPv4

  2023         Netlink protocol   UAPI                 6.x             M                   Netlink, YAML,
               specs / YNL                                                                 YNL

  2023         SCM_PIDFD /        sockets              6.x             M                   pidfd, Unix
               SO_PEERPIDFD                                                                sockets

  2023         MPTCP protocol     MPTCP/BPF            6.x             M                   MPTCP, BPF
               switching with BPF                                                          

  2023         SRv6               SRv6                 6.x             M/F                 SRv6
               PSP/NEXT-C-SID                                                              

  2023         partial TC HW      TC/offload           6.x             M                   TC, offload
               offload                                                                     
               continuation                                                                

  2023-11      The                virtual netdev       6.7             M                   netkit, BPF
               BPF-programmable                                                            
               network device                                                              

  2023--24     virtio-net AF_XDP  VM networking        6.x             R/M-parts           virtio-net,
               zero-copy series                                                            AF_XDP

  2024         Device Memory TCP  zero-copy/RX         6.12            M                   devmem, netmem
               development                                                                 

  2024-06      P4TC hits a brick  TC/P4                ---             S/D                 P4TC
               wall                                                                        

  2024         RTNL-less qdisc    scalability          6.x             M/F                 RTNL, qdisc
               dumps                                                                       

  2024-09      in-kernel QUIC     transport/security   ---             R                   QUIC
               initial series                                                              

  2024-11      struct sockaddr    sockets/UAPI         6.x             D/F                 sockaddr
               flexible-array                                                              
               issue                                                                       

  2024-12      The Homa network   transport            ---             R/D                 Homa
               protocol                                                                    

  2025-03      Warming up to      memory/RX            6.x             D/M                 netmem, pages
               frozen pages for                                                            
               networking                                                                  

  2025         BPF qdisc          TC/BPF               6.x             M                   qdisc, struct_ops

  2025         io_uring zero-copy zero-copy/RX         6.15            M                   io_uring,
               RX                                                                          page_pool

  2025         TCP_RTO_MAX_MS     TCP API              6.15            M                   TCP, RTO

  2025         BPF network        observability        6.15            M                   BPF, latency
               timestamp                                                                   
               callbacks                                                                   

  2025         Device Memory TCP  zero-copy/TX         6.16            M                   devmem
               TX                                                                          

  2025         DCCP removal       protocol removal     6.x             X                   DCCP

  2025-05      Faster firewalls   firewall/BPF         userspace/BPF   U/D                 bpfilter
               with bpfilter                                                               

  2025         local TCP          routing              6.x             M/F                 ECMP, TCP
               multipath routing                                                           
               improvements                                                                

  2025-07      QUIC for the       transport/security   ---             R/D                 QUIC
               kernel                                                                      

  2025         AccECN core series TCP/ECN              6.18            M                   AccECN

  2025         UDP RX             UDP/performance      6.18            M                   UDP
               optimization                                                                

  2025         DIBS               local transport      6.18            M                   DIBS

  2025         default socket     socket memory        6.18            M                   rmem
               rmem 4MB                                                                    

  2025         per-netns          conntrack            ---             R                   conntrack, netns
               conntrack hash RFC                                                          

  2025-11      A struct sockaddr  sockets/UAPI         6.x             D/F                 sockaddr
               sequel                                                                      

  2026-01      netkit queue       VM/container         7.x-era         R/M                 netkit, AF_XDP
               leasing +          networking                                               
               io_uring/AF_XDP                                                             

  2026-01      double             tunnel/performance   7.x-era         R/M                 GENEVE, VXLAN
               GENEVE/VXLAN                                                                
               tunnel GRO/GSO                                                              

  2026-02      More accurate      TCP/ECN              7.0 follow-up   F/D                 AccECN
               congestion                                                                  
               notification for                                                            
               TCP                                                                         

  2026         CAKE multiqueue    qdisc                7.0             M                   CAKE

  2026         VSOCK network      virtualization       7.0             M                   VSOCK, netns
               namespaces                                                                  

  2026         UDP-Lite removal   protocol removal     7.1             X                   UDP-Lite

  2026         legacy network     removal              7.1             X                   ATM, AX25, ISDN,
               subsystem removal                                                           CAIF
               wave                                                                        

  2026         TCP-AO → libcrypto TCP/security         7.2             F                   TCP-AO

  2026         MPTCP max subflows MPTCP                7.2             F                   MPTCP
               8→64                                                                        

  2026-07      netkit/BPF         VM networking        7.x             D/F                 netkit
               user-space update                                                           

  2026-08      Examining other    BPF/netns            ---             D/R                 BPF, netns
               network namespaces                                                          
               using BPF                                                                   

  2026-08      BIG TCP over       tunnel/TCP           7.3 dev         M-pending-release   BIG TCP
               VXLAN/GENEVE                                                                

  2026         conntrack          conntrack            7.x             F                   conntrack
               timeout-policy                                                              
               lifetime                                                                    

  2026         flowtable GC       flowtable            7.x             F                   flowtable, GC
               confirmation race                                                           
               fix                                                                         
  ----------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 56. Canonical status rules

To avoid overstating upstream state, the inventory uses the following
rules.

### `merged`

Verified as present in a mainline release or mainline development tree.

### `merged-follow-up`

Not the introduction of a feature; modifies/scales/fixes an already
merged architecture.

### `RFC` / `under-review`

LWN or mailing-list coverage exists, but this document has not
established that the complete feature was merged.

### `stalled`

A significant design effort whose proposed architecture did not become
the expected mainline solution.

### `removed`

Code/protocol/subsystem removed from mainline.

### `userspace-milestone`

Primarily a userspace release or userspace/kernel-interface maturity
milestone.

### `design-discussion`

Important for understanding Linux networking direction but not itself a
kernel feature.

------------------------------------------------------------------------

# 57. Coverage conclusion

The audit now covers the major Linux networking architecture lines
between 2019-05-07 and 2026-10-02:

``` text
packet memory:
page_pool → netmem → devmem / io_uring ZC

packet size:
GRO/GSO → BIG TCP → tunnel BIG TCP

programmability:
XDP/BPF → struct_ops → socket lookup → netkit → BPF qdisc

virtual networking:
virtio-net → AF_XDP ZC → multi-buffer → netkit queue leasing

control plane:
ioctl → Generic Netlink → YAML/YNL

routing:
FIB → nexthop objects → multipath refinements

firewall:
nftables → flowtable → HW offload

observability:
skb_drop_reason → BPF network timestamps

locking/scalability:
global RTNL → narrower locking → per-netns RTNL work

transport:
TCP → MPTCP / AccECN / TCP-AO
      + QUIC/Homa development

protocol maintenance:
DECnet → DCCP → UDP-Lite and other legacy removals
```

The remaining work is no longer broad discovery. It is **verification
and normalization**:

1.  replace remaining approximate dates with exact LWN publication
    dates;
2.  attach exact mainline commit hashes where verified;
3.  resolve every `R/M-parts` item into per-patch merged status;
4.  deduplicate articles that appear both as patch postings and feature
    articles;
5.  verify that every canonical inventory row has at least one primary
    upstream source;
6.  generate a compact release-by-release appendix from the canonical
    inventory.

This distinction is important: the thematic search is substantially
complete, while commit-level provenance is intentionally still stricter
and incomplete where evidence has not yet been verified.

------------------------------------------------------------------------

# 58. Netdev conference cross-reference (2019-05-07--2026-10-02)

Netdev Society の archive を 0x13--0x1A まで確認し、LWN/kernel change
log と 技術的に対応する講演を cross-reference する。

注意点:

-   Netdev 0x13 は 2019-03-20--22 開催のため、本資料の開始日 2019-05-07
    より前。 したがって **本期間の正式 inventory
    からは除外**する。ただし、後の機能の前史として
    有用な講演（conntrack, XDP/virtio-net, BPF TCP extensibility
    等）は存在する。
-   0x14 (2020) 以降は対象期間内。
-   conference talk は upstream merge を意味しない。`proposal`,
    `development`, `deployment`, `research` を区別して読む。
-   Netdev の講演資料は、LWN記事より前に proposal/design
    を説明しているケースが多く、
    「機能がどこから来たか」を追う一次資料として有用。

Netdev archive: https://netdevconf.info/

------------------------------------------------------------------------

## 58.1 Netdev 0x14 --- 2020

Conference: https://netdevconf.info/0x14/

既存 change log と特に関連する講演:

  -----------------------------------------------------------------------
  Netdev 0x14 session     Change-log lineage      Relevance
  ----------------------- ----------------------- -----------------------
  The Path To TCP 4K MTU  TCP RX zero-copy →      early receive-side
  and RX ZeroCopy         io_uring ZCRX / devmem  zero-copy work

  Implementation of IPv6  IPv6 IOAM               direct upstream
  IOAM in Linux Kernel                            implementation topic

  Using Upstream MPTCP in MPTCP                   follows initial MPTCP
  Linux Systems                                   mainlining

  TC Connection tracking  conntrack / TC HW       flow offload lineage
  hardware offload -      offload                 
  upstream work                                   

  Fast OVS data path with XDP / virtual switching XDP fast-path work
  XDP                                             

  Issuing SYN cookies in  XDP/BPF TCP             SYN-cookie fast path
  XDP                                             

  Replacing HTB with EDT  BPF / traffic control   programmable scheduling
  and BPF                                         lineage

  Hierarchical QoS        TC HW offload           qdisc/offload
  Hardware Offload (HTB)                          

  kTLS HW offload -       KTLS                    TLS record/offload
  implementation and                              lineage
  performance gains                               

  Performance study of    KTLS / handshake        precursor to later
  kernel TLS handshakes                           kernel-handshake work

  devlink enhancements    devlink/SR-IOV          netdev
  for sub functions                               device-management API
  management                                      

  Hardware offload for    container/offload       Kubernetes datapath
  K8s container                                   
  networking                                      

  Hardware Acceleration   container/offload       virtual networking
  of Container Networking                         
  Interfaces                                      
  -----------------------------------------------------------------------

Netdev 0x14 is particularly useful because several topics later
appearing as mature kernel features were already being discussed
together:

``` text
TCP zero-copy
MPTCP
IOAM
XDP
TC/conntrack offload
KTLS
container HW offload
```

------------------------------------------------------------------------

## 58.2 Netdev 0x15 --- 2021

Accepted sessions: https://netdevconf.info/0x15/accepted-sessions.html

### BIG TCP --- Eric Dumazet

One of the strongest cross-references in this audit.

Netdev 0x15 accepted **BIG TCP** in June 2021, before the LWN feature
article and before the Linux 5.19 merge.

This gives the lineage:

``` text
Netdev 0x15 (2021)
BIG TCP talk
      │
      ▼
LWN "Going big with TCP packets" (2022)
      │
      ▼
Linux 5.19
IPv6 BIG TCP
      │
      ▼
Linux 6.3
IPv4 BIG TCP
      │
      ▼
Linux 7.3 development
VXLAN/GENEVE BIG TCP
```

### Accelerating synproxy with XDP

Session:
https://netdevconf.info/0x15/loadsessions/Accelerating-synproxy-with-XDP.html

The talk explicitly describes extending BPF helpers so XDP can:

-   query conntrack information;
-   generate/check SYN cookies without a local listening socket.

This connects:

``` text
XDP
  ├── SYN-cookie helpers
  └── conntrack BPF integration
```

and is a useful precursor to the later BPF conntrack lifecycle kfunc
work.

### Resilient nexthop groups

This directly follows the 2019 nexthop-object work and belongs in the
routing/FIB lineage.

``` text
nexthop objects
      │
      ▼
nexthop groups
      │
      ▼
resilient groups
```

### Rethinking Zero-Copy Networking with MAIO

This is relevant as an alternative high-performance networking design in
the same period that eventually produced stronger kernel-side zero-copy
efforts around io_uring, page_pool/netmem and Device Memory TCP.

### TC / ACL / BPF

Relevant sessions include:

-   Recent Enhancements to the TC Police Action
-   Where turbo boosting TC flower control path had led us to
-   Linux ACL Performance Analysis
-   Introducing Ptables
-   XDP General Workshop
-   Switchdev Offload Workshop

The ACL analysis compares:

-   iptables;
-   iptables + IPSet;
-   XDP/eBPF;
-   TC/eBPF;
-   TC flower.

Ptables was implemented over eBPF at TC/XDP and provides useful
historical context for the later bpfilter/BPF-firewall discussions.

------------------------------------------------------------------------

## 58.3 Netdev 0x16 --- 2022

Conference: https://netdevconf.info/0x16/

### Merging the Networking Worlds --- David Ahern, Shrijeet Mukherjee

Session:
https://netdevconf.info/0x16/sessions/talk/merging-the-networking-worlds.html

This is a particularly important architectural precursor to Device
Memory TCP and modern io_uring zero-copy receive.

The talk compares:

``` text
traditional socket API
    │
    ├── syscalls
    ├── memcpy
    └── page refcount overhead

io_uring
    │
    └── fewer syscalls + TX zero-copy
        but no direct hardware queue ownership

AF_XDP
    │
    └── hardware queues + registered userspace memory
        but bypasses the kernel TCP stack

RDMA
    │
    └── direct memory/hardware access
        but separate networking ecosystem
```

The proposal attempts to combine:

``` text
kernel TCP/IP stack
       +
pre-registered application memory
       +
hardware queues
       +
zero-copy RX/TX
```

This is almost exactly the architectural problem later attacked by:

``` text
page_pool/netmem
     │
Device Memory TCP
     │
io_uring ZCRX
     │
netkit queue leasing
```

### Historical significance

For the change log this gives a useful design lineage:

``` text
Netdev 0x16 (2022)
"Merging the Networking Worlds"
          │
          ▼
Netdev 0x17 (2023)
Device Memory TCP
Zero Copy Receive using io_uring
          │
          ▼
Netdev 0x18 (2024)
Devmem TCP & io_uring zero copy BoF
          │
          ▼
Linux 6.12
Device Memory TCP RX
          │
          ▼
Linux 6.15
io_uring ZC RX
```

------------------------------------------------------------------------

## 58.4 Netdev 0x17 --- 2023

Sessions: https://netdevconf.info/0x17/pages/sessions.html

This edition has unusually strong overlap with the mainline networking
developments covered in this document.

### Device Memory TCP

Speakers:

-   Mina Almasry
-   Willem de Bruijn
-   Eric Dumazet
-   Kaiyuan Zhang

This is the direct conference counterpart to the Device Memory TCP
RFC/mainline lineage.

### Fast ZC Rx Data Plane using io_uring / Zero Copy Receive using io_uring

Session:
https://netdevconf.info/0x17/sessions/talk/zero-copy-receive-using-io_uring.html

Speakers:

-   David Wei
-   Pavel Begunkov

The description explicitly frames memory bandwidth as a bottleneck and
compares the kernel socket copy path with kernel bypass and RDMA.

This is one of the most useful primary sources for the later Linux 6.15
io_uring ZCRX feature.

### eBPF Qdisc: a generic building block for traffic control

Speakers:

-   Hsin-Wei (Amery) Hung
-   Cong Wang

This is a direct precursor/context source for the later BPF qdisc /
`struct_ops` development.

### Integrating eBPF Into The P4TC Datapath

This provides the conference-side history for the P4TC proposal that
later encountered maintainer resistance and was covered by LWN in 2024.

### Netlink APIs to Expose/Configure Netdev Objects

This belongs alongside:

``` text
Generic Netlink
      │
YNL specifications
      │
modern netdev configuration APIs
```

### Other relevant sessions

-   NIC offloads at Hyperscale: experience, new offloads and validation
-   SO_TIMESTAMPING: powering fleetwide RPC monitoring
-   TCP Offload via AF_XDP sockets --- Not your grandmother's TCP
    Offload!
-   Using eBPF to inject IPv6 Extension Headers
-   Multi-core IPsec tunnels
-   TLS handshake for in-kernel consumers (BoF)
-   XDP Workshop
-   TC Workshop
-   Netfilter Mini Workshop

The TLS handshake BoF is especially useful alongside the
`net/handshake`/KTLS chapters.

------------------------------------------------------------------------

## 58.5 Netdev 0x18 --- 2024

Schedule: https://netdevconf.info/0x18/pages/schedule.html

### Devmem TCP & io uring zero copy --- BoF

Speakers include Willem de Bruijn et al.

This conference occurred while both Device Memory TCP and io_uring ZC RX
were moving toward mainline maturity.

The sequence is therefore:

``` text
2022  architecture proposal
          │
2023  implementation talks
          │
2024  joint devmem/io_uring BoF
          │
2024  Linux 6.12 devmem TCP RX
          │
2025  Linux 6.15 io_uring ZC RX
```

### The Future of AI Networks: Advancing TCP with Device Memory and Collective Communication

This extends Device Memory TCP from a pure networking optimization into
AI/GPU/collective communication use cases.

### A new lightweight Zero-Copy Notification Mechanism in Linux

Relevant to the broader zero-copy TX/RX completion/lifetime problem.

### Characterizing IOTLB Wall for Multi-100-Gbps Linux-based Networking

Relevant to DMA/IOMMU overhead in high-speed networking and therefore
complementary to page_pool/netmem/device-memory work.

### Fine-grained TCP Tuning

Relevant to the growing set of per-socket TCP controls and later
RTO/receive-side tuning changes.

### Workshops/BoFs

-   Extension Headers Workshop
-   TC Workshop
-   IPsec Workshop

------------------------------------------------------------------------

## 58.6 Netdev 0x19 --- 2025

Schedule: https://netdevconf.info/0x19/pages/schedule.html

Sessions: https://netdevconf.info/0x19/pages/sessions.html

This edition maps closely to the 2025 chapters in the change log.

### Diagnosing Page Pool Leaks

Directly relevant to:

``` text
page_pool
   │
   ▼
netmem
   │
   ├── devmem TCP
   └── io_uring ZCRX
```

It is useful operational material because page_pool recycling/lifetime
bugs are a key failure mode in modern RX-memory infrastructure.

### MPTCP: present, future, and its development workflow

Directly complements the MPTCP release/patch history.

### SRv6 in Linux Kernel, FRR and eBPF

Directly complements the SRv6 Headend Reduced / PSP / NEXT-C-SID
evolution.

### Communication via Internal Shared Memory (ISM) - Time to open up

This is especially interesting in light of the later DIBS/shared-memory
transport work.

The conceptual lineage is not `netmem → DIBS`; instead:

``` text
ISM / SMC-D / local shared-memory transports
              │
              ▼
DIBS / later shared-memory socket ideas
```

### mq-cake: Scaling software rate limiting across CPU cores

This is the conference-side precursor/context for CAKE multiqueue work
that later appears in the Linux 7.0-era change log.

### Linux Kernel Support for IOAM Direct Exporting

Follow-up to the earlier IPv6 IOAM implementation.

### The Battle Of The ZCs: Who is the prettiest of them all?

Useful comparison material for the multiple Linux zero-copy mechanisms
that this document otherwise follows separately.

### IRQ Suspension: a new, efficient mechanism for packet delivery

Relevant to the NAPI/softirq/busy-poll execution-model evolution.

### The future of SO_TIMESTAMPING

Complements the packet/network timestamp observability lineage.

### State of the union in TCP land --- Eric Dumazet

Useful cross-reference for contemporary TCP performance/API work,
including the period around AccECN, receive-side optimization, and timer
tuning.

------------------------------------------------------------------------

## 58.7 Netdev 0x1A --- 2026

Sessions: https://netdevconf.info/0x1A/pages/sessions.html

Netdev 0x1A is inside the requested time window and is especially
valuable because its materials were available by August 2026.

### io_uring ZCRX: Progress and Next Steps --- Pavel Begunkov

The session covers the post-merge ZCRX problems:

-   refill-queue exhaustion;
-   buffers that cannot immediately be recycled;
-   sharing NIC queues between processes;
-   memory-pressure detection;
-   API/performance follow-ups.

This is the natural follow-up to Linux 6.15 ZCRX.

### Accelerating Software RDMA (RXE) with Netkit and Devmem

Directly intersects two major late-period themes:

``` text
netkit
   +
Device Memory / devmem
   +
software RDMA
```

and reinforces that netkit is becoming a mechanism for exposing modern
memory/queue facilities beyond ordinary container networking.

### Kernel shared memory socket transport --- David Wei

Session:
https://netdevconf.info/0x1A/sessions/talk/kernel-shared-memory-socket-transport.html

Proposal:

``` text
AF_UNIX SOCK_SEQPACKET
        +
io_uring registered buffers
        +
shared memory
        │
        ▼
zero-copy sender + receiver IPC
```

This is a new RFC/proposal lineage and should not be confused with DIBS,
but it belongs in the same broad "avoid local-host copies" design space.

### Linux QUIC: Bringing a Modern Secure Transport into the Kernel

Direct conference counterpart to the 2024--2026 kernel QUIC RFC series.

### TCP State of the union (2026) --- Eric Dumazet

Session:
https://netdevconf.info/0x1A/sessions/talk/tcp-state-of-the-union-2026.html

The talk explicitly focuses on recent/upcoming TCP changes and
performance on modern platforms, making it a useful companion source for
the late TCP chapters.

### Thrice the charm: an skb extension for BPF metadata

Relevant to the long-running question of how metadata is carried with
skb/BPF datapaths.

### Securing IOAM in the Linux Kernel

Follow-up to IOAM deployment: telemetry integrity/trust rather than
merely carrying trace data.

### Network Observability BoF

Complements:

-   `skb_drop_reason`;
-   SO_TIMESTAMPING;
-   BPF network timestamp callbacks;
-   modern tracing/Retis work.

### Other relevant 0x1A sessions

-   AF_XDP copy mode needs more love
-   XDP Workshop
-   SRv6 Workshop
-   What's next for the PSP Security Protocol
-   Can Homa and TCP Get Along?
-   Rakaia: Scalable In-Kernel Scheduling for TCP-Based RPCs
-   Scripting Netfilter with Lua
-   Could an IPv6-Only Kernel Be a Reality?
-   Chat with the Maintainers / Netconf Update

------------------------------------------------------------------------

# 59. Netdev ↔ LWN ↔ mainline lineage map

The conference material makes several feature histories much clearer.

## BIG TCP

``` text
Netdev 0x15 (2021)
BIG TCP — Eric Dumazet
        │
        ▼
LWN 2022
Going big with TCP packets
        │
        ▼
Linux 5.19
IPv6 BIG TCP
        │
        ▼
Linux 6.3
IPv4 BIG TCP
        │
        ▼
2026
HBH removal + UDP tunnel work
        │
        ▼
Linux 7.3 development
VXLAN/GENEVE BIG TCP
```

## Zero-copy receive / Device Memory TCP

``` text
Netdev 0x14 (2020)
TCP RX ZeroCopy discussion
        │
        ▼
Netdev 0x16 (2022)
Merging the Networking Worlds
        │
        ├──────────────┐
        ▼              ▼
Netdev 0x17        Netdev 0x17
Device Memory TCP  io_uring ZC RX
        │              │
        └──────┬───────┘
               ▼
Netdev 0x18 (2024)
Devmem TCP & io_uring ZC BoF
        │
        ├── Linux 6.12: Device Memory TCP RX
        └── Linux 6.15: io_uring ZC RX
               │
               ▼
Netdev 0x1A (2026)
io_uring ZCRX: Progress and Next Steps
```

## BPF traffic control

``` text
BPF struct_ops / TCP CC
        │
        ▼
Netdev 0x17 (2023)
eBPF Qdisc
        │
        ▼
2025 BPF qdisc series
        │
        ▼
mainline traffic-control programmability
```

## P4TC

``` text
Netdev 0x13 (pre-window)
P4 compiler backend for TC
        │
        ▼
Netdev 0x17 (2023)
eBPF in P4TC datapath
        │
        ▼
LWN 2024
P4TC hits a brick wall
```

## Shared-memory networking

``` text
ISM / SMC-D
      │
Netdev 0x19 (2025)
Communication via ISM
      │
      ├── DIBS development
      │
      └── Netdev 0x1A (2026)
          kernel shared-memory socket transport
```

These are related design spaces, not necessarily direct implementation
ancestry.

------------------------------------------------------------------------

# 60. Recommended Netdev talks for this change log

For understanding the architectural evolution rather than simply
collecting conference talks, the highest-value sessions are:

1.  **BIG TCP** --- Netdev 0x15 (2021)
2.  **Merging the Networking Worlds** --- Netdev 0x16 (2022)
3.  **Device Memory TCP** --- Netdev 0x17 (2023)
4.  **Zero Copy Receive using io_uring** --- Netdev 0x17 (2023)
5.  **eBPF Qdisc: a generic building block for traffic control** ---
    Netdev 0x17 (2023)
6.  **Devmem TCP & io uring zero copy** --- Netdev 0x18 (2024)
7.  **Diagnosing Page Pool Leaks** --- Netdev 0x19 (2025)
8.  **Communication via Internal Shared Memory (ISM) - Time to open up**
    --- Netdev 0x19 (2025)
9.  **State of the union in TCP land** --- Netdev 0x19 (2025)
10. **io_uring ZCRX: Progress and Next Steps** --- Netdev 0x1A (2026)
11. **Linux QUIC: Bringing a Modern Secure Transport into the Kernel**
    --- Netdev 0x1A (2026)
12. **TCP State of the union (2026)** --- Netdev 0x1A (2026)

These talks are especially useful because they form a bridge between:

``` text
design motivation
      ↓
prototype / RFC
      ↓
upstream patch series
      ↓
LWN analysis
      ↓
mainline kernel
      ↓
post-merge operational lessons
```

which is the central goal of this change log.

------------------------------------------------------------------------

# 61. Netdev coverage note

The Netdev site confirms the relevant editions in the requested period:

``` text
0x14 — 2020
0x15 — 2021
0x16 — 2022
0x17 — 2023
0x18 — 2024
0x19 — 2025
0x1A — 2026
```

0x13 took place in March 2019 and is outside the 2019-05-07 start
boundary, so its sessions are used only as historical context and are
not counted as in-window conference entries.

Future provenance work should add a `Netdev` column to the canonical LWN
inventory where a direct conference counterpart exists.

------------------------------------------------------------------------

# 62. Provenance index --- Netdev → LWN → patch series → mainline

この章は canonical inventory の主要項目について、conference talk と
upstream development の位置関係を横断的に追えるようにした索引である。

`Netdev phase`:

-   **pre-merge** --- mainline merge 前の設計・prototype・proposal
-   **merge-era** --- patch review / merge と同時期
-   **post-merge** --- mainline 後の運用・performance・次世代設計
-   **parallel** --- 同じ問題領域だが直接の実装系列とは断定しない

  -------------------------------------------------------------------------------------------------------------------------------
  Feature           Netdev counterpart   Phase                   LWN / upstream       Kernel milestone  Provenance interpretation
                                                                 milestone                              
  ----------------- -------------------- ----------------------- -------------------- ----------------- -------------------------
  BIG TCP           0x15 (2021) ---      pre-merge               LWN *Going big with  5.19              conference
                    **BIG TCP**, Eric                            TCP packets*; IPv6                     prototype/design → LWN
                    Dumazet                                      BIG TCP series                         analysis → merge

  IPv4 BIG TCP      BIG TCP 0x15 as      pre-merge               2023 IPv4 BIG TCP    6.3               original design
                    origin                                       series                                 generalized to IPv4

  BIG TCP over      BIG TCP 0x15 as      historical origin       BIG TCP for UDP      7.3 development   BIG TCP extended to
  tunnels           origin; later tunnel                         tunnels series                         VXLAN/GENEVE
                    work                                                                                

  TCP RX zero-copy  0x14 --- **The Path  pre-merge               multiple later ZC RX multi-release     early socket/TCP
                    To TCP 4K MTU and RX                         RFCs                                   receive-copy reduction
                    ZeroCopy**                                                                          work

  Device Memory TCP 0x16 **Merging the   pre-merge → merge-era   Device Memory TCP    6.12 RX           unusually clear
                    Networking Worlds**                          RFC series                             conference→RFC→mainline
                    → 0x17 **Device                                                                     lineage
                    Memory TCP** → 0x18                                                                 
                    devmem/io_uring BoF                                                                 

  io_uring ZC RX    0x16 **Merging the   pre-merge → post-merge  2022--25 RFC/patch   6.15              design → implementation →
                    Networking Worlds**                          series                                 joint review → post-merge
                    → 0x17 **Zero Copy                                                                  refinement
                    Receive using                                                                       
                    io_uring** → 0x18                                                                   
                    BoF → 0x1A **ZCRX:                                                                  
                    Progress and Next                                                                   
                    Steps**                                                                             

  page_pool         0x19 **Diagnosing    post-merge              page_pool/netmem     established infra operational/lifetime
                    Page Pool Leaks**                            development                            debugging after broad
                                                                                                        adoption

  netmem            0x17/0x18 devmem +   merge-era/parallel      netmem abstraction   6.x               enabling memory
                    ZCRX talks                                   series                                 abstraction used by
                                                                                                        devmem/ZCRX

  BPF qdisc         0x17 **eBPF Qdisc: a pre-merge               BPF qdisc            2025-era          research/prototype
                    generic building                             `struct_ops` series                    direction → upstream BPF
                    block for traffic                                                                   qdisc
                    control**                                                                           

  P4TC              0x17 **Integrating   pre-merge               LWN *P4TC hits a     not established   conference design →
                    eBPF Into The P4TC                           brick wall*          as mainline       upstream review
                    Datapath**                                                        feature           resistance/stall

  XDP SYN proxy /   0x15 **Accelerating  pre-merge               BPF conntrack        6.x               XDP needs CT/SYN-cookie
  conntrack         synproxy with XDP**                          kfunc/helper work                      primitives → richer BPF
                                                                                                        networking API

  nexthop           0x15 **Resilient     post-origin             2019 nexthop-object  5.x onward        object model → resilient
  objects/groups    nexthop groups**                             series                                 grouping/selection

  MPTCP             0x14 **Using         merge-era → post-merge  LWN *Upstreaming     5.6 onward        initial deployment →
                    Upstream MPTCP**;                            multipath TCP*;                        mature workflow/scaling
                    0x19 **MPTCP:                                later BPF/subflow                      
                    present, future...**                         work                                   

  IPv6 IOAM         0x14                 pre/merge → post        IPv6 IOAM patch      5.15/5.16 onward  implementation → export →
                    **Implementation of                          series                                 security
                    IPv6 IOAM in Linux                                                                  
                    Kernel**; 0x19                                                                      
                    **IOAM Direct                                                                       
                    Exporting**; 0x1A                                                                   
                    **Securing IOAM**                                                                   

  KTLS / kernel     0x14 KTLS offload +  pre/merge               LWN kernel-TLS       6.x               KTLS record path →
  handshake         handshake                                    handshake articles;                    generic kernel-consumer
                    performance; 0x17                            `net/handshake`                        handshake
                    TLS handshake BoF                                                                   

  TC HW offload     0x14 conntrack HW    merge-era               standalone TC action 5.x/6.x           common flow/action
                    offload / HTB HW                             HW-offload series                      representation → richer
                    offload; 0x15 TC                                                                    HW lifecycle
                    flower/police talks                                                                 

  SO_TIMESTAMPING / 0x17                 post/parallel           `skb_drop_reason`,   6.x               packet causality + timing
  observability     SO_TIMESTAMPING;                             BPF timestamp                          observability
                    0x19 **future of                             callbacks                              
                    SO_TIMESTAMPING**;                                                                  
                    0x1A observability                                                                  
                    BoF                                                                                 

  SRv6              0x19 SRv6            post/ongoing            Headend Reduced,     6.x/7.x           mainline behavior
                    Linux/FRR/eBPF BoF;                          PSP, NEXT-C-SID, MUP                   growth +
                    0x1A SRv6 workshop                                                                  operational/planning
                                                                                                        feedback

  shared-memory     0x19 **Communication parallel / new proposal DIBS and local       6.18+ / RFC       common goal: remove
  networking        via ISM**; 0x1A                              shared-memory work                     same-host copies; not one
                    **kernel                                                                            direct code lineage
                    shared-memory socket                                                                
                    transport**                                                                         

  QUIC              0x1A **Linux QUIC:   merge-era/development   2024--26 kernel QUIC RFC/development   conference presentation
                    Bringing a Modern                            RFC series; LWN                        of active upstream
                    Secure Transport                             *QUIC for the                          architecture
                    into the Kernel**                            kernel*                                

  TCP contemporary  0x19 **State of the  post/ongoing            AccECN, RX tuning,   6.15--7.x         useful umbrella talks for
  evolution         union in TCP land**;                         RTO API, performance                   late-period TCP changes
                    0x1A **TCP State of                          work                                   
                    the union (2026)**                                                                  

  netkit / devmem   0x1A **Accelerating  post/ongoing            netkit + queue       6.7 onward        netkit expands from
                    Software RDMA (RXE)                          leasing/devmem                         virtual netdev to
                    with Netkit and                              development                            queue/memory
                    Devmem**                                                                            infrastructure
  -------------------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 63. Detailed provenance chains

## 63.1 BIG TCP

``` text
Netdev 0x15 (2021)
"BIG TCP"
Eric Dumazet
        │
        │ design/prototype
        ▼
upstream BIG TCP patch work
        │
        ▼
LWN (2022)
"Going big with TCP packets"
        │
        ▼
Linux 5.19
IPv6 BIG TCP
        │
        ├── Linux 6.3: IPv4 BIG TCP
        │
        ├── 2026: IPv6 HBH dependency removal
        │
        └── Linux 7.3 development:
            BIG TCP over VXLAN/GENEVE
```

**Interpretation:** Netdev is not merely a later presentation of the
merged feature. In this case it records the design before mainline
adoption.

Netdev 0x15 materials describe the motivation as reducing TCP/IP stack
overhead at 200/400-Gbit speeds by allowing much larger TSO/GRO
aggregates.

------------------------------------------------------------------------

## 63.2 Device Memory TCP + io_uring ZCRX

``` text
Netdev 0x14 (2020)
TCP RX zero-copy discussion
        │
        ▼
Netdev 0x16 (2022)
"Merging the Networking Worlds"
        │
        │ asks how to combine:
        │ sockets + io_uring + AF_XDP/RDMA-like memory/queue control
        ▼
┌──────────────────────────┐
│ Netdev 0x17 (2023)       │
│                          │
│ Device Memory TCP        │
│ Zero Copy RX w/ io_uring │
└────────────┬─────────────┘
             │
             ▼
Netdev 0x18 (2024)
Devmem TCP & io_uring ZC BoF
             │
       ┌─────┴──────┐
       ▼            ▼
Linux 6.12      Linux 6.15
Devmem TCP RX   io_uring ZCRX
                    │
                    ▼
Netdev 0x1A (2026)
"ZCRX: Progress and Next Steps"
```

**Interpretation:** this is the strongest provenance chain in the
document. Netdev captures the problem definition, concrete
implementations, integration discussion, and post-merge operational/API
issues.

The 0x17 io_uring talk explicitly describes preserving the Linux TCP
stack while DMAing payload data into userspace-visible memory, rather
than replacing the stack with a kernel-bypass datapath.

------------------------------------------------------------------------

## 63.3 BPF qdisc

``` text
Linux 5.6
BPF struct_ops for TCP congestion control
        │
        ▼
Netdev 0x17 (2023)
"eBPF Qdisc: a generic building block for traffic control"
        │
        ▼
BPF qdisc upstream series
        │
        ▼
2025 networking merge work
```

**Interpretation:** `struct_ops` evolves from replacing TCP operation
tables to replacing or implementing scheduling operations in the
traffic-control layer.

This is more informative than treating BPF qdisc as an isolated 2025
feature.

------------------------------------------------------------------------

## 63.4 MPTCP

``` text
2019 LWN
"Upstreaming multipath TCP"
        │
        ▼
Linux 5.6
initial upstream MPTCP
        │
        ▼
Netdev 0x14 (2020)
"Using Upstream MPTCP in Linux Systems"
        │
        ▼
userspace path-manager / multiple subflows
        │
        ▼
BPF protocol selection + subflow visibility
        │
        ▼
Netdev 0x19 (2025)
"MPTCP: present, future, and its development workflow"
        │
        ▼
Linux 7.2
max subflows 8 → 64
```

**Interpretation:** Netdev 0x14 is deployment-oriented shortly after
initial upstreaming; 0x19 is a mature-project/status discussion.

------------------------------------------------------------------------

## 63.5 IOAM

``` text
Netdev 0x14 (2020)
"Implementation of IPv6 IOAM in Linux Kernel"
        │
        ▼
upstream IPv6 IOAM series
        │
        ▼
Linux 5.15 / 5.16 era
IOAM + encapsulation
        │
        ▼
Netdev 0x19 (2025)
IOAM Direct Exporting
        │
        ▼
Netdev 0x1A (2026)
Securing IOAM in the Linux Kernel
```

**Interpretation:** the topic evolves from implementing telemetry
carriage to exporting the telemetry efficiently and then protecting its
integrity/trust.

------------------------------------------------------------------------

## 63.6 KTLS / kernel handshake

``` text
KTLS record layer
       │
       ▼
Netdev 0x14
KTLS HW offload
TLS handshake performance
       │
       ▼
LWN 2022
Extending in-kernel TLS support
Adding an in-kernel TLS handshake
       │
       ▼
net/handshake generic upcall
       │
       ▼
Netdev 0x17
TLS handshake for in-kernel consumers BoF
       │
       ▼
NVMe/TCP TLS and later kernel consumers
```

The important distinction remains:

``` text
KTLS
    ≠ "all TLS runs inside the kernel"

net/handshake
    = common mechanism allowing kernel socket consumers
      to coordinate handshake work, including userspace assistance
```

------------------------------------------------------------------------

## 63.7 P4TC vs BPF qdisc

Netdev helps separate two superficially similar programmable-TC
directions.

``` text
P4TC
  │
  ├── P4 pipeline/object model
  ├── Netdev 0x17 P4TC+BPF
  └── substantial upstream API/design resistance
          │
          └── LWN: "P4TC hits a brick wall"


BPF qdisc
  │
  ├── existing TC qdisc framework
  ├── BPF struct_ops
  ├── Netdev 0x17 eBPF qdisc
  └── later upstream merge work
```

They should therefore remain separate rows in the canonical inventory.

------------------------------------------------------------------------

# 64. Canonical inventory --- Netdev counterpart field

For the final normalized inventory, add these fields:

``` text
Date
LWN title/topic
Category
Kernel
Status
Patch series
Mainline commit(s)
Netdev edition
Netdev session
Netdev phase
Tags
```

Example:

  ------------------------------------------------------------------------------------------------------
  LWN /      Kernel            Status                 Netdev   Session           Phase
  feature                                                                        
  ---------- ----------------- ---------------------- -------- ----------------- -----------------------
  BIG TCP    5.19              merged                 0x15     BIG TCP           pre-merge

  Device     6.12              merged                 0x17 /   Device Memory TCP pre/merge-era
  Memory TCP                                          0x18     / Devmem+io_uring 
  RX                                                           BoF               

  io_uring   6.15              merged                 0x17 /   Zero Copy RX /    pre→post
  ZCRX                                                0x18 /   BoF / Progress    
                                                      0x1A     and Next Steps    

  BPF qdisc  2025-era          merged                 0x17     eBPF Qdisc        pre-merge

  P4TC       ---               stalled/under-review   0x17     eBPF Into P4TC    pre-merge
                               lineage                         Datapath          

  MPTCP      5.6+              merged/evolving        0x14 /   Using Upstream    merge→post
                                                      0x19     MPTCP / present & 
                                                               future            

  IPv6 IOAM  5.15/5.16+        merged/evolving        0x14 /   implementation /  pre→post
                                                      0x19 /   export / security 
                                                      0x1A                       

  QUIC       RFC/development   under review           0x1A     Linux QUIC        merge-era/development
  ------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 65. How to use the provenance index

The document can now be read in three directions.

### Start from a kernel release

``` text
Linux 6.15
   │
   └── io_uring ZCRX
          │
          ├── LWN merge-window
          ├── lore patch series
          └── Netdev 0x17 → 0x18 → 0x1A
```

### Start from an LWN article

``` text
LWN "Going big with TCP packets"
   │
   ├── Netdev 0x15 design talk
   ├── Linux 5.19 merge
   ├── IPv4 follow-up
   └── tunnel follow-up
```

### Start from a Netdev talk

``` text
Netdev 0x17 Device Memory TCP
   │
   ├── RFC revisions
   ├── netmem/page_pool prerequisites
   ├── LWN coverage
   └── Linux 6.12 merge
```

This makes Netdev a provenance source rather than a detached conference
bibliography.

------------------------------------------------------------------------

# 66. Remaining provenance work

The next verification pass should fill the remaining empty cells rather
than add more themes:

1.  exact LWN publication date and article ID for every canonical row;
2.  exact lore/netdev patch-series URL;
3.  mainline commit hash(es) from `git.kernel.org`;
4.  exact first kernel release;
5.  Netdev slide/paper/video URL where available;
6.  `pre-merge`, `merge-era`, `post-merge`, or `parallel`
    classification;
7.  explicit `RFC/not merged` markers where appropriate.

Priority order for exact commit provenance:

``` text
BIG TCP
Device Memory TCP
io_uring ZCRX
BPF qdisc
netkit
MPTCP/BPF
AccECN
RTNL breakup
nexthop objects
IOAM
```

These have the strongest Netdev↔LWN↔upstream relationships and will
provide the highest value if fully verified first.

------------------------------------------------------------------------

# 67. Provenance verification pass 1 --- exact upstream anchors

This pass verifies representative/mainline commits for the first five
high-priority lineages. A **merge commit** and a **feature commit** are
recorded separately.

## 67.1 BIG TCP

### IPv4 BIG TCP --- Linux 6.3

Final v4 series was accepted into net-next with ten commits.

Key final commits:

``` text
a13fbf5ed5b4  netfilter: use skb_ip_totlen and iph_totlen
7eb072be41ba  cipso_ipv4: use iph_set_totlen in skbuff_setattr
50e6fb5c6efb  ipvlan: use skb_ip_totlen in ipvlan_get_L3_hdr
8e08bb75b60f  packet: add TP_STATUS_GSO_TCP for tp_status
9eefedd58ae1  net: add gso_ipv4_max_size and gro_ipv4_max_size per device
b1a78b9b9886  net: add support for ipv4 big tcp
```

The last commit is the principal feature anchor:

``` text
b1a78b9b9886 ("net: add support for ipv4 big tcp")
```

It must not be described as the *entire* IPv4 BIG TCP implementation; it
is patch 10/10 of the accepted series.

A later 2026 GRO fix explicitly references `b1a78b9b9886` as the
introduction point for IPv4 BIG TCP behavior, independently confirming
this provenance.

### Provenance

``` text
Netdev 0x15 BIG TCP
       │
       ▼
IPv6 BIG TCP / Linux 5.19
       │
       ▼
[PATCHv4 net-next 00/10]
IPv4 BIG TCP
       │
       ├── 9eefedd58ae1 per-device IPv4 GSO/GRO maximum
       └── b1a78b9b9886 IPv4 BIG TCP core support
               │
               ▼
Linux 6.3
```

------------------------------------------------------------------------

## 67.2 Device Memory TCP RX --- Linux 6.12

This lineage requires special care because earlier patchwork-bot
messages could look as if large early series had been accepted. In
December 2023, for example, only the first page-pool preparation patch
was actually taken; Mina Almasry explicitly clarified that the bot was
overly optimistic.

The definitive core series was:

``` text
[PATCH net-next v26 00/13] Device Memory TCP
```

Applied to `netdev/net-next.git` on 2024-09-12.

Verified commits:

``` text
7c88f86576f3  netdev: add netdev_rx_queue_restart()
3efd7ab46d0a  net: netdev netlink api to bind dma-buf to a net device
170aafe35cb9  netdev: support binding dma-buf to netdevice
28c5c74eeaa0  netdev: netdevice devmem allocator
8ab79ed50cf1  page_pool: devmem support
0f9214046893  memory-provider: dmabuf devmem memory provider
9f6b619edf2e  net: support non paged skb frags
65249feb6b3d  net: add support for skbs with unreadable frags
8f0b3cc9a4c1  tcp: RX path for devmem TCP
678f6e28b5f6  net: add SO_DEVMEM_DONTNEED setsockopt to release RX frags
09d1db26b5e5  net: add devmem TCP documentation
85585b4bc8d8  selftests: add ncdevmem, netcat for devmem TCP
d0caf9876a1c  netdev: add dmabuf introspection
```

Main feature anchors:

``` text
8ab79ed50cf1  page_pool devmem support
0f9214046893  dmabuf memory provider
8f0b3cc9a4c1  TCP RX path
678f6e28b5f6  userspace buffer-release API
```

The Linux 6.12 networking pull describes the result as receiving TCP
payloads directly into a DMA-BUF/device-memory region while packet
headers remain in ordinary kernel buffers so the normal TCP stack can
process them.

### Provenance

``` text
Netdev 0x16
Merging the Networking Worlds
        │
        ▼
Netdev 0x17
Device Memory TCP
        │
        ▼
many RFC / PATCH revisions
        │
        ▼
v26 / 13 patches
        │
        ▼
8f0b3cc9a4c1
TCP RX path
        │
        ▼
Linux 6.12
```

------------------------------------------------------------------------

## 67.3 Device Memory TCP TX --- Linux 6.16

The TX direction was merged separately after RX.

Final accepted series:

``` text
[PATCH net-next v14 0/9] Device memory TCP TX
```

Verified commits:

``` text
03e96b8c11d1  netmem: add niov->type attribute
e9f3d61db5cb  net: add get_netmem/put_netmem support
8802087d20c0  net: devmem: TCP tx netlink api
bd61848900bf  net: devmem: Implement TX path
17af8cc06a5a  net: add devmem TCP TX documentation
383faec0fd64  net: enable driver support for netmem TX
c32532670cec  gve: add netmem TX support to GVE DQO-RDA mode
ae28cb114727  net: check for driver support in netmem TX
2f1a805f32ba  selftests: ncdevmem: Implement devmem TCP TX
```

Primary TX feature anchor:

``` text
bd61848900bf ("net: devmem: Implement TX path")
```

Again, this commit is not the whole feature; the netlink API, netmem
lifetime support, driver capability, GVE implementation and selftests
are separate commits.

------------------------------------------------------------------------

## 67.4 io_uring zero-copy RX --- Linux 6.15

The feature was merged through Jens Axboe's pull:

``` text
for-6.15/io_uring-rx-zc-20250325
```

Mainline merge commit:

``` text
ca0b04ba0b35d48e1473a280c2e8905e7f80e906
Merge tag 'for-6.15/io_uring-rx-zc-20250325'
```

The pull explicitly describes:

-   zero-copy bulk receive directly into application memory;
-   a new io_uring receive request;
-   a shared refill queue;
-   a zero-copy page_pool feeding a hardware RX queue;
-   buffer lifetime controlled by io_uring rather than normal networking
    refcount rules.

Representative series commits include:

``` text
io_uring/zcrx: add interface queue and refill queue
io_uring/zcrx: add io_zcrx_area
io_uring/zcrx: add io_recvzc request
io_uring/zcrx: set pp memory provider for an rx queue
net: add documentation for io_uring zcrx
```

The final pre-merge tip named in the pull was:

``` text
89baa22d75278b69d3a30f86c3f47ac3a3a659e9
io_uring/zcrx: add selftest case for recvzc with read limit
```

### Important distinction

``` text
ca0b04ba...
```

is a **mainline merge commit**, not the single implementation commit.
The implementation is a multi-commit series beneath that merge.

### Provenance

``` text
Netdev 0x16 architecture
       │
Netdev 0x17 implementation
       │
Netdev 0x18 BoF
       │
v13 net-next / io_uring series
       │
ca0b04ba... mainline merge
       │
Linux 6.15
       │
Netdev 0x1A post-merge follow-up
```

------------------------------------------------------------------------

## 67.5 BPF qdisc --- 2025

The accepted BPF pull identifies ten Amery Hung commits implementing the
initial BPF qdisc support.

Principal feature commit:

``` text
c8240344956e3f0b4e8f1d40ec3435e47040cacb
bpf: net_sched: Support implementation of Qdisc_ops in bpf
```

This allows BPF `struct_ops` programs to implement:

``` text
Qdisc_ops.enqueue
Qdisc_ops.dequeue
Qdisc_ops.init
Qdisc_ops.reset
Qdisc_ops.destroy
```

Additional commits in the accepted series add:

-   qdisc skb kfuncs;
-   watchdog timer;
-   bstats update;
-   root/mq attachment restrictions;
-   libbpf create/destroy support;
-   FIFO/FQ selftests.

BPF-side merge tip:

``` text
fd23ce3eb4a1005bd109977856d12ec0fde7ef75
Merge branch 'bpf-qdisc'
```

Merge into net-next:

``` text
07e32237ed9d3f5815fb900dee9458b5f115a678
Merge tag 'for-netdev' ...
```

The code was followed almost immediately by fixes for using BPF qdisc as
the default qdisc and by making the currently supported `Qdisc_ops`
callbacks mandatory. This is worth recording because the initial merge
was functional but the API/validation rules were still settling.

### Provenance

``` text
BPF struct_ops / TCP CC
       │
       ▼
Netdev 0x17 eBPF Qdisc
       │
       ▼
2025 BPF qdisc series
       │
       ├── c8240344956e feature anchor
       ├── fd23ce3eb4a1 BPF branch merge
       └── 07e32237ed9d net-next merge
```

------------------------------------------------------------------------

## 67.6 netkit core --- Linux 6.7

The original netkit device has a clean feature anchor:

``` text
35dfaad7188cdc043fde31709c796f5a692ba2bd
netkit, bpf: Add bpf programmable net device
```

The accepted v4 series consisted of seven commits:

``` text
35dfaad7188c  netkit, bpf: Add bpf programmable net device
5c1b994de4be  tools: Sync if_link uapi header
05c31b4ab205  libbpf: Add link-based API for netkit
92a85e18ad47  bpftool: Implement link show support for netkit
bec981a4add6  bpftool: Extend net dump with netkit progs
51f1892b5289  selftests/bpf: Add netlink helper library
ace15f91e569  selftests/bpf: Add selftests for netkit
```

The feature commit explicitly states that BPF runs inside the driver's
xmit routine so container/Pod egress can be processed earlier and can
redirect directly to a physical device without traversing the per-CPU
backlog queue.

------------------------------------------------------------------------

## 67.7 netkit queue leasing --- 2026: merge, revert, rework, re-merge

This history corrects an earlier oversimplification in this document.

### First merge

A v7-era queue-leasing series was merged into net-next in January 2026:

``` text
77b9c4a438fc66e2ab004c411056b3fb71a54f2c
Merge branch 'netkit-support-for-io_uring-zero-copy-and-af_xdp'
```

Representative commits included:

``` text
a5546e18f77c  net: Add queue-create operation
31127deddef4  net: Implement netdev_nl_queue_create_doit
9e2103f36110  net: Add lease info to queue-get response
ff8889ff9107  net, ethtool: Disallow leased real rxqs to be resized
...
920da3634194  netkit: Add xsk support for af_xdp applications
```

### Immediate revert

The merge was then reverted:

``` text
8766d61a1d33cb5f15bfdd6ce9832bbe1fc649c2
Revert "Merge branch 'netkit-support-for-io_uring-zero-copy-and-af_xdp'"
```

The revert message states:

``` text
The series will conflict with io_uring work,
and the code needs more polish.
```

Therefore the January 2026 merge must **not** be treated as the final
upstream landing.

### Rework

The series continued through later revisions, including:

``` text
v8  2026-01-29
v10 2026-03-27
```

The design remained queue leasing:

``` text
container netns
    │
netkit virtual queue
    │ lease/proxy
    ▼
physical NIC queue
    │
    ├── io_uring memory provider
    └── AF_XDP
```

### Final re-merge

A reworked series was merged again in April 2026.

Merge commit:

``` text
15089225889ba4b29f0263757cd66932fa676cb0
Merge branch 'netkit-support-for-io_uring-zero-copy-and-af_xdp'
```

The accepted selftest tip includes:

``` text
65d657d806848add1e1f0632562d7f47d5d5c188
selftests/net: Add queue leasing tests with netkit
```

This tests io_uring zero-copy from a network namespace through netkit
leased queues backed by a physical netdev.

### Correct status timeline

``` text
2026-01
queue leasing merged
      │
      ▼
immediately reverted
      │
      ▼
v8 → v10 → later rework
      │
      ▼
2026-04
re-merged
      │
      ▼
post-merge fixes / Netdev 0x1A use cases
```

This is now the canonical history used by this document.

------------------------------------------------------------------------

# 68. Provenance quality levels

To make the final inventory auditable, each commit mapping should carry
one of these quality levels.

``` text
A — exact accepted series + exact mainline commit(s) verified
B — merge commit / pull request verified, individual commits partially enumerated
C — patch series verified, merge status/release verified, hashes incomplete
D — design/RFC only; no mainline commit expected
```

Current status:

  Feature                Quality
  ---------------------- --------------------------------------
  IPv4 BIG TCP           A
  Device Memory TCP RX   A
  Device Memory TCP TX   A
  io_uring ZCRX          B
  BPF qdisc              A
  netkit core            A
  netkit queue leasing   A, including revert/re-merge history
  AccECN                 C
  RTNL breakup           C
  MPTCP/BPF              C
  IOAM                   C
  nexthop objects        C

------------------------------------------------------------------------

# 69. Next provenance batch

Next verification order:

1.  AccECN;
2.  per-netns/fine-grained RTNL;
3.  MPTCP initial merge + BPF protocol switching + subflow iterator;
4.  nexthop objects and resilient nexthop groups;
5.  IPv6 IOAM;
6.  AF_XDP multi-buffer;
7.  virtio-net AF_XDP zero-copy status;
8.  BIG TCP IPv6 initial commit series and 2026 tunnel series.

The goal remains to upgrade each row to quality **A** where the upstream
history permits it, without inventing a single "feature commit" for
genuinely multi-commit features.

------------------------------------------------------------------------

# 70. Provenance verification pass 2 --- AccECN, RTNL, MPTCP/BPF, nexthop, AF_XDP

## 70.1 AccECN --- correction and exact core anchors

Earlier revisions of this document stopped the review history too early
at v16. The protocol series continued:

``` text
v16  2025-09-06
v17  2025-09-08
...
v19  2025-09-16
```

The v17 cover letter explicitly disabled AccECN enablement until the
complete feature had been accepted. v19 had been reduced to the ten core
protocol patches.

Verified mainline feature anchors:

``` text
542a495cbaa6dc57a310da62b501fdf318657cad
tcp: AccECN core

3cae34274c79e0c60ccd1c10516973af1aed2a7c
tcp: accecn: AccECN negotiation
```

The core commit implements Accurate ECN accounting but deliberately does
not by itself provide negotiation or the AccECN option; those arrive in
subsequent commits.

Therefore:

``` text
542a495cbaa6
    = core accounting anchor

NOT
    = complete AccECN feature
```

The accepted lineage must be treated as a series containing:

-   core Accurate ECN state/accounting;
-   negotiation;
-   receive byte counters;
-   delivered-byte accounting;
-   SACK option preparation;
-   AccECN TCP option;
-   option send control/failure handling.

The core series entered the Linux 6.18 development cycle. Later releases
continue deployment/CC/offload integration.

**Quality: A for core + negotiation anchors; B for the complete later
integration chain.**

------------------------------------------------------------------------

## 70.2 RTNL breakup --- correction: migration, not a one-release switch

The Linux 6.13 networking pull describes per-netns RTNL as:

> a very large, in-progress effort

and says the new behavior was disabled by default.

The key debug/migration mechanism is:

``` text
CONFIG_DEBUG_NET_SMALL_RTNL
```

Its help text is unusually explicit:

``` text
rtnl_lock()
    │
    ▼
rtnl_net_lock()
    │
    ├── global RTNL
    └── small per-netns RTNL mutex

during conversion:
    both locks are taken

after conversion goal:
    global rtnl_lock() can disappear
    and rtnetlink gains per-netns scalability
```

A representative series patch is:

``` text
rtnetlink: Add per-netns RTNL.
```

which adds the per-netns mutex, locking helpers, lock ordering rules and
the `DEBUG_NET_SMALL_RTNL` knob.

### 6.13 scope

The 6.13 pull lists work including:

-   RCU-ifying FIB control-path pieces;
-   per-netns locking helpers;
-   namespacing the IPv4 address hash;
-   `rtnl_register_many()`;
-   moving validation out of RTNL;
-   converting phonet handlers to RCU;
-   converting IPv4 address manipulation to per-netns RTNL.

### Important current-state correction

The debug knob remains present in later kernel source, so this document
must not describe 6.13 as "global RTNL replaced by per-netns RTNL".

Correct timeline:

``` text
pre-6.13
global RTNL
    │
    ▼
6.13
per-netns lock infrastructure +
dual-lock debug conversion
    │
    ▼
6.15 ... 7.x
operation-by-operation RTNL dependency reduction
    │
    ▼
target architecture
fine/per-netns locking where possible
```

**Quality: A for the 6.13 migration architecture; per-operation
conversion remains a multi-release provenance set rather than one
feature commit.**

------------------------------------------------------------------------

## 70.3 MPTCP + BPF protocol switching

The BPF MPTCP series evolved substantially before merge.

A key design milestone was v6/v7:

``` text
update_socket_protocol()
```

hooked from `__sys_socket()`.

Purpose:

``` text
legacy application

socket(AF_INET, SOCK_STREAM, 0)
              │
              ▼
BPF / security hook:
update_socket_protocol()
              │
              ├── leave IPPROTO_TCP
              └── return IPPROTO_MPTCP
```

This avoids the limitations of `mptcpize`/`LD_PRELOAD`, including
applications that do not use libc and environments where launch-time
environment changes are difficult.

The series went through at least:

``` text
v6  introduces update_socket_protocol
v7  hook details
v8/v9 cleanup and validation
later revisions before integration
```

A 2026 selftest fix independently confirms that the `mptcpify` BPF
program in current kernels hooks `update_socket_protocol()` to rewrite
eligible TCP socket creation into `IPPROTO_MPTCP`.

### Important semantic detail

The hook sees the raw socket `type` before normal masking of:

``` text
SOCK_CLOEXEC
SOCK_NONBLOCK
```

which caused the 2026 `mptcpify` selftest/application mismatch and
required masking `SOCK_TYPE_MASK` in the BPF program.

This is a useful post-merge provenance point: it demonstrates that the
API is not merely an RFC artifact.

**Quality: B** --- merged/current behavior is independently verified,
but the exact introduction hash should still be taken from the final
accepted series rather than guessed from an RFC revision.

------------------------------------------------------------------------

## 70.4 Nexthop objects --- 2019 initial architecture

Final initial series:

``` text
[PATCH v3 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
2019-06-07
```

The author describes it as the **final set of the initial nexthop object
work**.

Motivating measurement:

``` text
~700k IPv4 routes
1 hop       ~18 sec
4 paths     ~28 sec
```

at the start of the work, with major costs attributed to repeated
`synchronize_rcu()` and repeated validation of
device/gateway/encapsulation information.

The object model changes:

``` text
route
  └── embedded/repeated nexthop information

to

route
  └── nexthop ID
          │
          └── independently managed nexthop object/group
```

This aligns Linux better with routing-daemon and hardware models and
enables later resilient groups.

**Quality: C→B** --- final series and merge-era provenance verified;
exact individual initial commit hashes remain to be enumerated.

------------------------------------------------------------------------

## 70.5 Resilient nexthop groups --- 2021

LWN/netdev archives show the kernel series implementing resilient
nexthop groups.

The design introduces a bucket table:

``` text
flow hash
   │
   ▼
bucket
   │
   ▼
nexthop
```

When group membership or weight changes, buckets can migrate in a
controlled way rather than remapping essentially every flow.

Kernel-side series components include:

-   resilient NH-group UAPI;
-   group data structures;
-   bucket implementation;
-   notifications;
-   netlink handlers;
-   bucket get/dump;
-   activity reporting;
-   final enablement.

The iproute2 support explicitly references kernel commit:

``` text
2a0186a37700b0d5b8cc40be202a62af44f02fa2
```

as the kernel-side resilient-nexthop-group implementation baseline.

Example:

``` text
ip nexthop add id 10 group 1/2 type resilient \
    buckets 8 idle_timer 60 unbalanced_timer 300
```

This gives a clean lineage:

``` text
2019 nexthop objects
       │
       ▼
2021 resilient nexthop groups
       │
       ▼
later local TCP / multipath selection refinements
```

**Quality: A/B** --- kernel baseline hash and final userspace API
verified; the complete kernel multi-commit series can still be
enumerated.

------------------------------------------------------------------------

## 70.6 AF_XDP multi-buffer --- Linux 6.6

Final accepted development series:

``` text
[PATCH v7 bpf-next 00/24] xsk: multi-buffer support
```

Core verified RX commit:

``` text
804627751b4281dd95148e7564759145da67855e
xsk: add support for AF_XDP multi-buffer on Rx path
```

The implementation maps one packet across multiple AF_XDP descriptors
and uses:

``` text
XDP_PKT_CONTD
```

to tell userspace that the packet continues in the next descriptor.

TX preparation commit:

``` text
b7f72a30e9ac2555b05afc6cfddc9dbc98e1eb8d
xsk: introduce wrappers and helpers for supporting multi-buffer in Tx path
```

The accepted series also contains:

-   `XSK_USE_SG`;
-   EOP/continuation handling;
-   RX multi-buffer;
-   TX multi-buffer;
-   zero-length descriptor handling;
-   ZC maximum-fragment Netlink attribute;
-   i40e/ice driver support;
-   documentation;
-   selftests.

A 2026 bug fix carries:

``` text
Fixes: 804627751b42
("xsk: add support for AF_XDP multi-buffer on Rx path")
```

which independently confirms the introduction anchor.

### Why this matters

``` text
single-buffer AF_XDP
     │
packet must fit one UMEM frame
     ▼
multi-buffer AF_XDP
     │
packet spans descriptors
     ▼
jumbo / fragmented XDP representation
```

This is an important prerequisite for combining AF_XDP zero-copy with
larger packets and modern multi-buffer drivers.

**Quality: A.**

------------------------------------------------------------------------

# 71. virtio-net AF_XDP --- status must be split by capability

The long virtio-net AF_XDP effort should not be represented as one
binary "merged/not merged" feature.

The series evolved through:

``` text
2023  large initial AF_XDP zero-copy series
2023  net-next v1 19-patch series
2024  virtnet preparation/refactoring
2024  v5 15-patch zero-copy series
2024  TX-focused series
2025  zero-copy multi-buffer mergeable-RX RFC
```

This history contains three different questions:

``` text
A. Does virtio-net have the core/refactoring needed for AF_XDP?
B. Does it support AF_XDP zero-copy for a given RX/TX mode?
C. Does it support mergeable multi-buffer zero-copy?
```

The 2025 RFC for:

``` text
virtio-net: support zerocopy multi buffer XDP in mergeable
```

is explicit evidence that multi-buffer zero-copy in mergeable receive
mode was still being developed separately.

Therefore the canonical inventory should avoid a row such as:

``` text
virtio-net AF_XDP zero-copy | merged
```

without naming the mode/capability.

Instead use capability rows:

``` text
virtio-net AF_XDP preparation       merged pieces
virtio-net AF_XDP ZC RX/TX          verify per series/release
mergeable multi-buffer AF_XDP ZC    RFC/development at 2025 point
```

**Quality: C pending a capability-by-capability mainline diff audit.**

------------------------------------------------------------------------

# 72. IOAM provenance --- current verified level

The initial IPv6 IOAM series and current kernel implementation are both
verified.

Initial series:

``` text
Support for the IOAM Pre-allocated Trace with IPv6
```

Current code still implements:

``` text
IOAM6_TYPE_PREALLOC
```

in IPv6 hop-by-hop option processing.

Later evolution includes a 2024 Generic Netlink multicast-event series
for exporting IOAM trace data to userspace.

This supports the conceptual evolution already documented:

``` text
IOAM packet trace implementation
       │
       ▼
configuration / encapsulation
       │
       ▼
trace export to userspace
       │
       ▼
direct export / security work
```

**Quality: B/C** --- behavior and series lineage are verified; exact
initial merge hashes remain for the next commit-enumeration pass.

------------------------------------------------------------------------

# 73. Quality table after pass 2

  Feature                                     Previous                       Now
  ------------------------------ --------------------- -------------------------
  AccECN core                                        C                   **A/B**
  RTNL migration architecture                        C                     **A**
  MPTCP BPF protocol switching                       C                     **B**
  nexthop objects                                    C                     **B**
  resilient nexthop groups               not separated                   **A/B**
  AF_XDP multi-buffer              previously verified                     **A**
  virtio-net AF_XDP                          ambiguous   **C, capability-split**
  IPv6 IOAM                                          C                   **B/C**

Corrections introduced in this pass:

1.  AccECN review history extends beyond v16 to v19.
2.  Linux 6.13 is the start/major milestone of per-netns RTNL
    conversion, not completion.
3.  virtio-net AF_XDP must be described per RX/TX/mergeable/multi-buffer
    capability.
4.  AF_XDP multi-buffer RX introduction is exactly anchored at
    `804627751b42`.

------------------------------------------------------------------------

# 74. Next exact-commit pass

Remaining high-value commit enumeration:

1.  initial MPTCP merge into Linux 5.6;
2.  final `update_socket_protocol()` introduction commit;
3.  MPTCP subflow BPF iterator/kfunc commits;
4.  all 2019 initial nexthop-object commits;
5.  all resilient-nexthop kernel commits around `2a0186a37700`;
6.  initial IPv6 IOAM commit series;
7.  virtio-net AF_XDP mainline capability matrix;
8.  initial IPv6 BIG TCP series;
9.  BIG TCP UDP-tunnel commits in 7.3 development;
10. per-operation RTNL conversions in 6.13→7.3 where they represent
    major API/locking milestones.

The next pass should also start replacing mirror commit links with
canonical `git.kernel.org` links wherever the exact object is known.

------------------------------------------------------------------------

# 75. Provenance verification pass 3 --- MPTCP, IOAM, BIG TCP tunnels

## 75.1 Initial upstream MPTCP --- Linux 5.6

The upstream MPTCP project documents Linux 5.6 as the first kernel
release exposing the native MPTCP socket API:

``` c
socket(AF_INET6, SOCK_STREAM, IPPROTO_MPTCP)
```

with:

``` text
IPPROTO_MPTCP = 262
```

The upstreaming work was deliberately split into prerequisite and
protocol series. An October 2019 prerequisite RFC states that the
following series would:

-   add `CONFIG_MPTCP`;
-   introduce the MPTCP socket type;
-   implement the basic protocol;
-   add selftests.

A useful exact anchor for the initial 5.6 implementation is the original
MPTCP selftest:

``` text
048d19d444be1e42abca19a6b969343954ae4e17
mptcp: add basic kselftest for mptcp
```

Later selftest fixes continue to carry:

``` text
Fixes: 048d19d444be ("mptcp: add basic kselftest for mptcp")
```

which independently verifies the initial-test provenance.

The canonical inventory should still treat the initial MPTCP
implementation as a **multi-commit feature**, not equate `048d19d444be`
with the whole protocol.

### Lineage

``` text
2019 LWN
Upstreaming multipath TCP
        │
        ▼
prerequisite series
        │
        ▼
MPTCP socket/protocol series
        │
        ├── native IPPROTO_MPTCP socket
        └── 048d19d444be initial kselftest
        │
        ▼
Linux 5.6
```

**Quality: B** for the initial feature series; **A** for the initial
selftest anchor and first-release identification.

------------------------------------------------------------------------

## 75.2 `update_socket_protocol()` --- exact BPF/MPTCP anchor

The BPF hook used to transparently convert eligible TCP socket creation
to MPTCP has an exact upstream anchor:

``` text
0dd061a6a115
bpf: Add update_socket_protocol hook
```

The earlier review lineage includes:

``` text
RFC bpf-next v7 1/6
PATCH bpf-next v8 1/4
```

The hook is placed in the socket-creation path and allows a BPF program
to alter the protocol before the socket is created.

Primary MPTCP use case:

``` text
application:
socket(AF_INET, SOCK_STREAM, 0)
          │
          ▼
update_socket_protocol()
          │
          ▼
BPF return:
IPPROTO_MPTCP
```

A later SMC design discussion explicitly cites:

``` text
commit 0dd061a6a115 ("bpf: Add update_socket_protocol hook")
```

and describes it as allowing protocol modification dynamically through
eBPF.

This independently confirms that the hook is merged infrastructure, not
merely an RFC.

**Quality: A.**

------------------------------------------------------------------------

## 75.3 IPv6 IOAM Pre-allocated Trace --- exact data-plane anchor

Final development series:

``` text
[PATCH net-next v5 0/6]
Support for the IOAM Pre-allocated Trace with IPv6
2021-07-20
```

The series implements the Pre-allocated Trace carried in an IPv6
Hop-by-Hop option and includes UAPI, data plane, configuration and
tests.

Exact mainline data-plane anchor:

``` text
9ee11f0fff205b4b3df9750bff5e94f97c71b6a0
ipv6: ioam: Data plane support for Pre-allocated Trace
```

This commit adds processing for the IOAM IPv6 Hop-by-Hop TLV and
associated per-interface and per-netns configuration such as:

``` text
net.ipv6.conf.<if>.ioam6_enabled
net.ipv6.ioam6_id
net.ipv6.conf.<if>.ioam6_id
```

Later 2026 fixes repeatedly use:

``` text
Fixes: 9ee11f0fff20
```

which independently validates it as the data-plane introduction point.

### Scope warning

`9ee11f0fff20` is the data-plane anchor, not the complete IOAM series.
The final series also contains:

-   IPv6 IOAM UAPI header definitions;
-   namespace/schema configuration;
-   lightweight tunnel/output support;
-   selftests.

### Netdev relationship

``` text
Netdev 0x14
Implementation of IPv6 IOAM
        │
        ▼
v4 / v5 upstream series
        │
        ▼
9ee11f0fff20
IPv6 IOAM data plane
        │
        ▼
Linux 5.15/5.16-era evolution
        │
        ├── encapsulation
        ├── direct export
        └── security work
```

**Quality: A for the data-plane introduction; B for complete-series
commit enumeration.**

------------------------------------------------------------------------

## 75.4 Initial IPv6 BIG TCP --- Linux 5.19 exact introduction anchor

The Linux 5.19 networking pull describes the new feature as:

``` text
TCPv6 segmentation offload with super-segments > 64KiB
using IPv6 Jumbogram support
```

and explicitly names it **BIG TCP**.

A later kernel CVE/stable history independently identifies the
introduction point:

``` text
0fe79f28bfaf73b66b7b1562d2468f94aa03bd12
```

as introducing the affected BIG TCP behavior in Linux 5.19.

The earlier audit also identified another introduction-related commit:

``` text
7c4e983c4f3cf94fcd879730c6caa877e0768a4d
```

The canonical interpretation remains:

``` text
0fe79f28bfaf...
7c4e983c4f3c...
    = introduction-related BIG TCP commits

NOT
    = proof that either single commit equals the whole feature
```

The 5.19 networking pull is the release-level authoritative anchor.

### Evolution

``` text
Netdev 0x15 (2021)
BIG TCP
       │
       ▼
Linux 5.19
IPv6 BIG TCP / >64KiB super-segments
       │
       ▼
Linux 6.3
IPv4 BIG TCP
       │
       ▼
2026
IPv6 BIG TCP without HBH
       │
       ▼
Linux 7.3 development
BIG TCP over VXLAN/GENEVE
```

**Quality: A/B** --- exact introduction anchor and release verified;
complete initial multi-commit series can still be enumerated.

------------------------------------------------------------------------

## 75.5 BIG TCP over UDP tunnels --- Linux 7.3 development tree

Final accepted series:

``` text
[PATCH net-next v9 0/9] BIG TCP for UDP tunnels
2026-07-10
```

The series is explicitly a follow-up to **BIG TCP without HBH in IPv6**
and enables IPv4/IPv6 BIG TCP workloads over VXLAN and GENEVE.

Patchwork/netdev bot confirms that all nine patches were applied to
`netdev/net-next.git`.

Exact commits:

``` text
5329647dad1b  net: Use helpers to get/set UDP len tree-wide
842870cdfa33  net: Enable BIG TCP with partial GSO
47282504ad21  udp: Support BIG TCP GSO packets where they can occur
efbc1aa8ed54  udp: Support gro_ipv4_max_size > 65536
8475a3efe6e6  udp: Validate UDP length in udp_gro_receive
99ad24516295  udp: Set length in UDP header to 0 for big GSO packets
f3d0f753f066  vxlan: Enable BIG TCP packets
03ebe91b0f61  geneve: Enable BIG TCP packets
5cb53743e1ff  selftests: net: Add a test for BIG TCP in UDP tunnels
```

The two most visible feature anchors are therefore:

``` text
f3d0f753f066  VXLAN
03ebe91b0f61  GENEVE
```

but they rely on the preceding generic UDP/GSO/GRO work in the same
series.

### Packet-format detail

The series uses UDP length `0` for BIG UDP/GSO packets where the true
length cannot be represented in the 16-bit UDP length field, adds
receive-side validation, and raises `tso_max_size` for the VXLAN/GENEVE
devices.

### Release status

As of the document cutoff (2026-10-02):

``` text
merged into net-next / Linux 7.3 development
final 7.3 release pending
```

so the status remains **merged-development**, not "released".

**Quality: A.**

------------------------------------------------------------------------

# 76. Updated provenance matrix

  ---------------------------------------------------------------------------------
  Feature                    Kernel      Exact anchor(s)                  Quality
  -------------------------- ----------- -------------------------------- ---------
  initial MPTCP              5.6         `048d19d444be` selftest + native B
                                         API/release verification         

  BPF                        later       `0dd061a6a115`                   **A**
  `update_socket_protocol`   MPTCP/BPF                                    
                             era                                          

  IPv6 IOAM data plane       5.x         `9ee11f0fff20`                   **A**

  IPv6 BIG TCP               5.19        `0fe79f28bfaf...` + release pull A/B

  IPv4 BIG TCP               6.3         `9eefedd58ae1`, `b1a78b9b9886`   **A**

  BIG TCP VXLAN              7.3 dev     `f3d0f753f066`                   **A**

  BIG TCP GENEVE             7.3 dev     `03ebe91b0f61`                   **A**

  AF_XDP multi-buffer        6.6         `804627751b42` et al.            **A**

  Device Memory TCP RX       6.12        `8f0b3cc9a4c1` + series          **A**

  io_uring ZCRX              6.15        merge `ca0b04ba0b35...`          B

  BPF qdisc                  2025-era    `c8240344956e...`                **A**

  netkit core                6.7         `35dfaad7188c...`                **A**

  netkit queue leasing       2026        merge→revert→`15089225889b...`   **A**
                                         remerge                          
  ---------------------------------------------------------------------------------

------------------------------------------------------------------------

# 77. Remaining capability audit

The provenance backlog is now narrower:

1.  enumerate all initial MPTCP 5.6 protocol commits rather than using
    the selftest as the easiest exact anchor;
2.  enumerate final 2019 nexthop-object commits;
3.  enumerate complete IOAM v5 accepted commit set around
    `9ee11f0fff20`;
4.  produce the virtio-net AF_XDP capability matrix:
    -   XDP refactoring;
    -   AF_XDP copy mode;
    -   zero-copy RX;
    -   zero-copy TX;
    -   mergeable-buffer support;
    -   multi-buffer support;
5.  enumerate MPTCP subflow BPF iterator/kfunc commits;
6.  track the major RTNL conversion commits by release rather than
    trying to invent a single RTNL "feature commit".

The tunnel BIG TCP branch no longer needs status research: the v9 series
and all nine net-next commits are now exactly identified.

------------------------------------------------------------------------

# 78. Provenance verification pass 4 --- virtio-net AF_XDP capability matrix

The virtio-net AF_XDP history is now split by capability rather than
described as one feature.

## 78.1 Preparation phase --- 2023--2024

The early series repeatedly identifies three prerequisites:

``` text
1. virtqueue per-queue reset
2. virtio-core premapped DMA
3. virtio-net XDP refactoring
```

Representative LWN/netdev series:

``` text
2023-02  [PATCH 00/33] virtio-net: support AF_XDP zero copy
2023-10  [PATCH net-next v1 00/19]
2024-01  split/refactored series
2024-06  v5/v6
2024-07  final RX series
```

The initial 2023 series was therefore development/proposal history, not
evidence that virtio-net already supported AF_XDP zero-copy in a
released kernel.

------------------------------------------------------------------------

## 78.2 AF_XDP RX zero-copy --- Linux 6.11

The Linux 6.11 networking pull explicitly lists:

``` text
VirtIO net:
  - support for AF_XDP Rx zero-copy
```

This is the release-level authoritative milestone.

Important RX commits include:

``` text
e9f3962441c0a4d6f16c656e6c8aa02a3ccdd568
virtio_net: xsk: rx: support fill with xsk buffer

a4e7ba7027012f009f22a68bcfde670f9298d3a4
virtio_net: xsk: rx: support recv small mode

99c861b44eb1fb9dfe8776854116a6a9064c19bb
virtio_net: xsk: rx: support recv merge mode
```

### Small receive mode

`a4e7ba702701` implements the AF_XDP zero-copy receive handling for the
virtio-net small buffer mode.

### Mergeable receive mode

`99c861b44eb1` adds AF_XDP zero-copy receive support to mergeable
receive buffers.

This does **not** mean arbitrary multi-buffer XDP programs are supported
in zero-copy mode; see below.

------------------------------------------------------------------------

## 78.3 Important correction: proposed revert was not the final status

In September 2024 a seven-patch **RFC** proposed reverting the RX
zero-copy work because of crashes when `VIRTIO_F_ACCESS_PLATFORM` was
absent and:

``` text
net.core.high_order_alloc_disable=1
```

The RFC explicitly proposed reverting, among others:

``` text
99c861b44eb1  recv merge mode
a4e7ba702701  recv small mode
e9f3962441c0  fill with xsk buffer
```

However, this must not be recorded as "virtio-net RX ZC was reverted
from mainline".

Later fixes in 2025 still use these commits as active `Fixes:` targets,
and the current driver retains the XSK pool setup path.

Therefore:

``` text
2024-07
RX ZC merged for Linux 6.11
      │
      ▼
2024-09
RFC to revert due to crash
      │
      └── proposal / review event,
          not the canonical final state
      │
      ▼
2025
fixes against the retained RX implementation
```

------------------------------------------------------------------------

## 78.4 AF_XDP TX zero-copy --- accepted in November 2024

TX was separated from the RX work.

Development:

``` text
2024-07  RFC TX series
2024-08  [PATCH net-next 00/13]
2024-09  RFC v1 after merge window closed
2024-11  [PATCH net-next v4 00/13]
```

The v4 series was applied to `netdev/net-next.git` on 2024-11-16.

Verified series anchors include:

``` text
9f19c084057a
virtio_ring: introduce vring_need_unmap_buffer

21a4e3ce6dc7b0a3bc882ebe1cb921a40235ddb0
virtio_net: xsk: bind/unbind xsk for tx

37e0ca657a3d
virtio_net: xdp_features add NETDEV_XDP_ACT_XSK_ZEROCOPY
```

The final feature-advertisement commit is especially useful:

``` text
NETDEV_XDP_ACT_XSK_ZEROCOPY
```

is advertised after the TX series lands.

### Capability timeline

``` text
Linux 6.11 era
    AF_XDP RX zero-copy
          │
          ▼
late 2024 net-next
    AF_XDP TX zero-copy
          │
          ▼
combined driver XSK pool binding
    RX + TX queue handling
```

------------------------------------------------------------------------

## 78.5 Mergeable RX is not the same as multi-buffer XDP support

This distinction is essential.

`99c861b44eb1` allows AF_XDP zero-copy receive in virtio-net's
**mergeable receive-buffer mode**.

But when a packet itself spans multiple XDP buffers and an XDP program
is attached, the driver did not yet support running that program over
the multi-buffer zero-copy packet.

2025 RFC:

``` text
[RFC PATCH net-next v2 0/2]
virtio-net: support zerocopy multi buffer XDP in mergeable
```

states explicitly:

``` text
currently:
zerocopy + mergeable receive mode
        │
        ├── single-buffer XDP: supported
        └── multi-buffer XDP: not supported

proposal:
use XDP frags for multi-buffer zero-copy XDP
```

The RFC uses a jumbo-MTU example where one packet exceeds one XDP buffer
and therefore must span multiple buffers.

### 2025 correctness fix

A later fix targets:

``` text
Fixes: 99c861b44eb1
("virtio_net: xsk: rx: support recv merge mode")
```

because the unsupported multi-buffer case could bypass the attached XDP
program and incorrectly pass the packet to the normal stack.

The accepted fix eventually became:

``` text
1ab665817448c31f4758dce43c455bd4c5e460aa
virtio-net: drop the multi-buffer XDP packet in zerocopy
```

and returns an abort/drop result rather than silently bypassing the XDP
program.

This is strong evidence for the capability boundary:

``` text
mergeable AF_XDP ZC RX
        ≠
multi-buffer XDP program support
```

------------------------------------------------------------------------

# 79. virtio-net AF_XDP capability matrix

  ----------------------------------------------------------------------------------
  Capability      Status         Mainline / series   Interpretation
                                 anchor              
  --------------- -------------- ------------------- -------------------------------
  virtqueue reset merged         pre-AF_XDP work     enables queue reconfiguration
  prerequisite    prerequisite                       

  premapped DMA   merged         virtio-core work    avoids ordinary per-buffer
  prerequisite    prerequisite                       mapping path

  XDP refactoring merged         pre-6.11 work       prepares common RX/XDP path
                  prerequisite                       

  AF_XDP RX       **merged**     `a4e7ba702701`      Linux 6.11 networking milestone
  zero-copy,                                         
  small mode                                         

  AF_XDP RX       **merged**     `99c861b44eb1`      mergeable receive mode
  zero-copy,                                         supported
  mergeable mode                                     

  AF_XDP TX       **merged**     v4/13 applied Nov   separate later series
  zero-copy                      2024;               
                                 `21a4e3ce6dc7...`   

  XSK ZC feature  **merged**     `37e0ca657a3d`      `NETDEV_XDP_ACT_XSK_ZEROCOPY`
  advertisement                                      

  zero-copy       **not          2025 RFC v2         separate XDP-frags problem
  multi-buffer    established by                     
  XDP in          base 6.11                          
  mergeable RX    work**                             

  unsupported     **fixed**      `1ab665817448...`   prevents silent XDP bypass
  multi-buffer                                       
  handling                                           
  ----------------------------------------------------------------------------------

### Canonical wording

Do not write:

``` text
Linux 6.11: virtio-net gained full AF_XDP zero-copy support
```

Prefer:

``` text
Linux 6.11 added virtio-net AF_XDP RX zero-copy support, including small and
mergeable receive modes. TX zero-copy was merged separately later in 2024.
Zero-copy multi-buffer XDP processing in mergeable mode remained a separate
problem and was still the subject of RFC work in 2025.
```

This is the capability-accurate description.

------------------------------------------------------------------------

# 80. MPTCP BPF provenance --- subflow support predates protocol switching

An important chronology point:

``` text
2020
BPF gains awareness of MPTCP subflows
       │
       ▼
later
BPF update_socket_protocol()
       │
       ▼
BPF can transparently select MPTCP
       │
       ▼
later iterator/kfunc work
richer subflow inspection/control
```

A 2020 series:

``` text
[PATCH bpf-next 0/3] bpf: add MPTCP subflow support
```

was motivated by the fact that BPF previously could not distinguish
ordinary TCP sockets from TCP sockets acting as MPTCP subflows.

This means the canonical MPTCP+BPF history should not begin with
`update_socket_protocol()`.

Instead separate:

1.  **subflow identification/visibility**;
2.  **protocol selection (`update_socket_protocol`)**;
3.  **subflow iterators/kfuncs and richer control**.

This better reflects how BPF integration expanded from observation to
selection and then to management.

------------------------------------------------------------------------

# 81. Nexthop series --- exact final-series structure

The final 2019 initial series is confirmed as:

``` text
[PATCH v3 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
2019-06-07
```

The archive exposes all 20 patch subjects. They fall into four
conceptual groups:

``` text
A. IPv6 fib6_nh preparation
   - make existing IPv6 paths able to deal with nexthop-contained fib6_nh

B. nexthop-object route integration
   - IPv4
   - IPv6

C. object lifecycle/API
   - replace
   - delete/use/reference handling

D. selftests
   - PMTU
   - multiple prefixes / shared nexthop
   - router multipath using nexthop objects
```

The key architectural fact is that the series is explicitly described by
David Ahern as the **final set of the initial nexthop object work**.

For provenance purposes, the 20-patch series itself is therefore a
stronger initial feature anchor than choosing one arbitrary commit and
labeling it "the nexthop commit".

The next pass should attach the exact mainline hash to each of these 20
subjects from `git.kernel.org`.

------------------------------------------------------------------------

# 82. Quality update after capability audit

  ---------------------------------------------------------------------
  Feature                            Quality
  ---------------------------------- ----------------------------------
  virtio-net AF_XDP RX ZC small mode **A**

  virtio-net AF_XDP RX ZC mergeable  **A**
  mode                               

  virtio-net AF_XDP TX ZC            **A**

  virtio-net multi-buffer ZC XDP     **D/RFC for 2025 proposal;
                                     boundary verified A**

  MPTCP early BPF subflow visibility **C/B**

  MPTCP `update_socket_protocol`     **A**

  2019 nexthop initial series        **B**, exact 20-hash enumeration
                                     pending
  ---------------------------------------------------------------------

The most important result of this pass is that virtio-net can now be
represented without the misleading binary label "AF_XDP zero-copy
supported".

------------------------------------------------------------------------

# 83. Provenance verification pass 5 --- RTNL migration map and MPTCP lineage

## 83.1 RTNL: release-by-release migration, not a single feature

The RTNL work is now represented as a migration program.

### Before Linux 6.13: reader-side preparation

By the first per-netns RTNL proposal in September 2024:

``` text
dumpit handlers:
15 / 44 already converted to RCU

doit handlers:
12 / 87 converted
mostly reader-side RTM_GET*
```

The proposal explicitly notes two earlier infrastructure milestones:

``` text
Linux 4.14
RTNL_FLAG_DOIT_UNLOCKED

Linux 6.9
RTNL_FLAG_DUMP_UNLOCKED
```

These allow individual rtnetlink handlers to declare that they do not
require the global RTNL lock.

Jakub Kicinski's 2024 networking retrospective summarizes the work as
two tracks:

``` text
Track A
read operations
global RTNL
    ↓
RCU / fine-grained locking
    ↓
6.9 / 6.10-era progress

Track B
writer/configuration operations
global RTNL
    ↓
per-network-namespace RTNL
    ↓
preparations entering 6.13
```

### Linux 6.13: per-netns RTNL infrastructure

Final early series:

``` text
[PATCH v3 net-next 0/4]
rtnetlink: Per-netns RTNL.
2024-10-04
```

It adds:

``` text
rtnl_net_lock(net)
rtnl_net_unlock(net)
per-netns RTNL mutex
assertion/debug helpers
CONFIG_DEBUG_NET_SMALL_RTNL
```

The series describes itself as:

``` text
the first step of the per-netns RTNL conversion
```

and the 6.13 LWN merge-window coverage calls it only one step in a long
process, with the per-namespace behavior disabled by default because of
regression risk.

Therefore:

``` text
Linux 6.13
≠ "RTNL became per-netns"

Linux 6.13
= infrastructure + first conversions + debug migration mode
```

### Linux 6.15: breakup continues

The 6.15 merge-window coverage still says:

``` text
Work continues toward the breaking up of the RTNL lock
```

This is important evidence that 6.13 was not the completion point.

### 2025: subsystem-by-subsystem removal

Example:

``` text
[PATCH v2 net-next 00/13]
mpls: Remove RTNL dependency.
2025-10-29
```

The series replaces RTNL with:

-   device reference counting;
-   RCU for dump/read paths;
-   a dedicated per-netns mutex for MPLS platform labels.

This illustrates the mature migration pattern:

``` text
global RTNL
   │
   ├── lifetime protection → refcount / RCU
   ├── read serialization → RCU
   └── configuration state → dedicated/per-netns mutex
```

### 2026: multicast routing and neighbour work

IPv4 multicast routing:

``` text
[PATCH net-next 00/15]
ipmr: No RTNL for RTNL_FAMILY_IPMR rtnetlink.
2026-02
```

IPv6 multicast routing:

``` text
[PATCH ... 00/15]
ip6mr: No RTNL for RTNL_FAMILY_IP6MR rtnetlink.
2026-04
```

The IPv6 series is explicitly described as the IPv6 version of the IPv4
work.

By August/September 2026 the neighbour subsystem was described as:

``` text
almost ready to drop RTNL
```

but still had another global scalability problem: `arp_tbl` and `nd_tbl`
themselves were global per-table objects. The proposed solution was to
make them per-netns.

This demonstrates the deeper point:

``` text
removing rtnl_lock()
does not automatically mean
the subsystem has no global serialization bottleneck
```

### Canonical RTNL timeline

``` text
pre-6.9
global RTNL dominates rtnetlink slow path
        │
        ▼
6.9 / 6.10
more dump/read paths become RTNL-less via RCU
        │
        ▼
6.13
per-netns RTNL infrastructure
DEBUG_NET_SMALL_RTNL
first writer-side conversion framework
        │
        ▼
6.15
continued breakup
        │
        ▼
2025
MPLS and other subsystem-specific conversions
        │
        ▼
2026
IPMR / IP6MR conversions
neighbour subsystem namespacing work
        │
        ▼
target:
RCU + refcounts + dedicated/per-netns locks
instead of one networking "big lock"
```

**Quality: A for the migration architecture and release chronology.**

------------------------------------------------------------------------

## 83.2 MPTCP initial upstreaming --- precise interpretation

The upstream MPTCP implementation guide confirms:

``` text
kernel < 5.6:
IPPROTO_MPTCP unavailable

kernel >= 5.6:
native MPTCP socket protocol exists
```

Native API:

``` c
socket(AF_INET,  SOCK_STREAM, IPPROTO_MPTCP);
socket(AF_INET6, SOCK_STREAM, IPPROTO_MPTCP);
```

with:

``` text
IPPROTO_MPTCP = 262
```

The October 2019 prerequisite RFC explains the upstreaming split:

``` text
series 1:
TCP/socket prerequisites

series 2:
CONFIG_MPTCP
MPTCP socket type
basic protocol
selftests
```

The prerequisite series itself contained work such as:

-   widening `sk_protocol`;
-   defining `IPPROTO_MPTCP`;
-   MPTCP TCP option definitions;
-   skb MPTCP extensions;
-   preventing TCP coalesce/collapse where MPTCP metadata must survive;
-   exporting/reusing TCP helpers.

This is useful because it shows that MPTCP was deliberately integrated
into the existing TCP machinery rather than introduced as an isolated
parallel stack.

Exact stable provenance anchor already verified:

``` text
048d19d444be1e42abca19a6b969343954ae4e17
mptcp: add basic kselftest for mptcp
```

Later fixes repeatedly reference it through `Fixes:`.

### Canonical initial-MPTCP interpretation

``` text
TCP infrastructure changes
        │
        ▼
MPTCP skb/options/socket prerequisites
        │
        ▼
native IPPROTO_MPTCP socket
        │
        ▼
basic MPTCP protocol
        │
        ▼
048d19d444be initial kselftest
        │
        ▼
Linux 5.6
```

The selftest commit remains an **exact anchor**, not the whole feature.

------------------------------------------------------------------------

## 83.3 MPTCP+BPF --- 2020 subflow-control series

Final reviewed development series found:

``` text
[PATCH bpf-next v3 0/5]
bpf: add MPTCP subflow support
2020-09-18
```

Before this work, `BPF_PROG_TYPE_SOCK_OPS` could not distinguish a
normal TCP socket from a TCP socket used as an MPTCP subflow.

The series adds:

``` text
bpf_tcp_sock.is_mptcp
```

and a MPTCP-specific BPF socket representation/helper.

The practical purpose is not just identification. A BPF program can
apply different per-subflow settings, for example:

``` text
socket mark
TCP congestion-control algorithm
other socket options
```

to different TCP subflows belonging to one MPTCP connection.

### Correct BPF evolution

``` text
2020
subflow visibility / per-subflow policy
        │
        ▼
later
update_socket_protocol()
        │
        └── transparently select MPTCP at socket creation
        │
        ▼
later
iterators / kfuncs
        │
        └── richer inspection and management
```

This is more accurate than treating BPF-MPTCP integration as beginning
with `update_socket_protocol()`.

**Quality: B for the 2020 series; A for its final v3 cover-letter
provenance.**

------------------------------------------------------------------------

# 84. Nexthop-object provenance --- what is verified and what is not

The final initial nexthop-object series is:

``` text
[PATCH v3 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
David Ahern
2019-06-07
```

The author explicitly calls it:

``` text
the final set of the initial nexthop object work
```

The original performance motivation is also unusually concrete:

``` text
700k+ IPv4 routes

1-hop:
~18 seconds

4-path:
~28 seconds
```

with excessive `synchronize_rcu()` and repeated validation of
device/gateway/encapsulation information identified as major kernel-side
costs.

### Provenance policy

Search/index results do not yet give a trustworthy one-to-one mapping of
all 20 patch subjects to final mainline SHA-1s. Therefore this document
deliberately does **not** invent or infer the missing hashes.

Current confidence:

``` text
final series identity       A
date/author                  A
20-patch structure          A
motivation/design           A
all 20 final mainline SHA   pending
```

This is preferable to presenting mirror/rebase hashes as if they were
canonical mainline objects.

------------------------------------------------------------------------

# 85. IOAM and MPTCP remaining exact-hash policy

The same rule now applies to the remaining multi-commit sets.

### IOAM

Already exact:

``` text
9ee11f0fff205b4b3df9750bff5e94f97c71b6a0
ipv6: ioam: Data plane support for Pre-allocated Trace
```

Still to enumerate:

``` text
UAPI
namespace/schema configuration
output/tunnel path
selftests
```

### MPTCP 5.6

Already exact:

``` text
048d19d444be...
initial MPTCP selftest
```

Still to enumerate from canonical mainline history:

``` text
socket type
protocol registration
basic send/receive
DSS/options
initial protocol state
```

No SHA will be added merely because a similarly named commit appears in
an old MPTCP development branch.

------------------------------------------------------------------------

# 86. RTNL architectural conclusion

The 2019--2026 networking history shows a broader locking transition:

``` text
one global lock
      │
      ▼
mark selected operations unlocked
      │
      ▼
RCU for readers
      │
      ▼
per-netns lock for namespace-local writers
      │
      ▼
dedicated locks/refcounts for subsystem state
      │
      ▼
remove remaining hidden global tables/serialization
```

RTNL is therefore best treated in the canonical inventory as a
**multi-year architectural program**, not as a feature introduced by
Linux 6.13.

This is analogous to the way the document treats:

``` text
page_pool → netmem → devmem
```

and:

``` text
GRO/GSO → BIG TCP → tunnel BIG TCP
```

as evolving architectural lineages rather than isolated release notes.

------------------------------------------------------------------------

# 87. Provenance backlog after pass 5

Highest-value unresolved exact mappings are now:

1.  all 20 final mainline SHA-1s for the 2019 nexthop-object series;
2.  complete initial MPTCP 5.6 SHA set;
3.  complete IOAM v5 SHA set;
4.  exact 2020 MPTCP+BPF five-commit SHA set;
5.  later MPTCP iterator/kfunc SHA set;
6.  selected RTNL conversion SHA sets where a subsystem reaches a clear
    lock-removal milestone.

The discovery/architecture phase for these topics is effectively
complete; remaining work is now canonical Git provenance normalization.

------------------------------------------------------------------------

# 88. Provenance verification pass 6 --- review lineage vs actual landing

This pass tightens an important rule: an old patch series may explain
the design without being the series that actually landed upstream.

## 88.1 MPTCP+BPF: 2020 series is design/review provenance, not final landing

The 2020 final review series was:

``` text
[PATCH bpf-next v3 0/5] bpf: add MPTCP subflow support
2020-09-18
```

with five patches:

``` text
1. bpf: expose is_mptcp flag to bpf_tcp_sock
2. mptcp: attach subflow socket to parent cgroup
3. bpf: add 'bpf_mptcp_sock' structure and helper
4. bpf: selftests: add MPTCP test base
5. bpf: selftests: add bpf_mptcp_sock() verifier tests
```

Its motivation is well established: `BPF_PROG_TYPE_SOCK_OPS` could not
distinguish ordinary TCP sockets from MPTCP subflow TCP sockets,
preventing per-subflow BPF policy.

However, this exact five-patch series must **not** be labeled as the
final merged series.

### Evidence: cgroup part landed separately

Later kernel fixes identify the mainline introduction of the cgroup
behavior as:

``` text
3764b0c5651e3
mptcp: attach subflow socket to parent cgroup
```

and a December 2020 MPTCP net-next series contains that patch
independently.

### Evidence: BPF mptcp_sock work was substantially reworked

In 2022 the BPF side returned as:

``` text
[PATCH bpf-next ...] bpf: mptcp: Support for mptcp_sock
```

The cover letter explicitly says that code from the 2020 series was
recognizable but had been **reworked quite a bit**.

The series continued through v5:

``` text
[PATCH bpf-next v5 0/7]
bpf: mptcp: Support for mptcp_sock
2022-05-19
```

and patchwork-bot confirms on 2022-05-20:

``` text
This series was applied to bpf/bpf-next.git (master)
by Andrii Nakryiko
```

The accepted v5 consists of:

``` text
bpf: add bpf_skc_to_mptcp_sock_proto
selftests/bpf: Enable CONFIG_IKCONFIG_PROC in config
selftests/bpf: test bpf_skc_to_mptcp_sock
selftests/bpf: add MPTCP test base
selftests/bpf: verify token of struct mptcp_sock
selftests/bpf: verify ca_name of struct mptcp_sock
selftests/bpf: verify first of struct mptcp_sock
```

Notably, by v4 the special-case kernel handling for `tcp_sock.is_mptcp`
had been dropped in favor of existing BPF TCP helpers.

### Correct canonical lineage

``` text
2020 v1→v3
MPTCP subflow BPF design/review
        │
        ├── cgroup issue
        │       └── 3764b0c5651e3 lands separately
        │
        ▼
2022 reworked BPF series
mptcp_sock / BTF access
        │
        ▼
v5 / 7 patches
        │
        ▼
applied to bpf-next
        │
        ▼
later update_socket_protocol()
and newer MPTCP BPF facilities
```

This replaces the earlier oversimplified interpretation that the 2020 v3
series itself was the final mainline implementation.

**Quality: A for review/landing chronology; exact SHA enumeration of the
seven accepted 2022 commits remains pending.**

------------------------------------------------------------------------

## 88.2 MPTCP 5.6 initial selftest anchor remains independently strong

The initial test anchor remains:

``` text
048d19d444be
mptcp: add basic kselftest for mptcp
```

Multiple later net/stable fixes carry:

``` text
Fixes: 048d19d444be
```

including 2023, 2025 and 2026 fixes.

Therefore it remains a safe exact anchor for the initial MPTCP
generation, while not being misrepresented as the whole MPTCP
implementation.

------------------------------------------------------------------------

## 88.3 IOAM data-plane anchor independently reconfirmed

The exact IOAM data-plane introduction remains:

``` text
9ee11f0fff20
ipv6: ioam: Data plane support for Pre-allocated Trace
```

A 2026 receive-path overflow fix again uses:

``` text
Fixes: 9ee11f0fff20
```

and modifies the same IOAM6 Pre-allocated Trace receive/send validation
path.

This raises confidence in the existing IOAM mapping without requiring
inference from an old development branch.

------------------------------------------------------------------------

# 89. Canonical provenance rule

For every remaining multi-year feature, the inventory now distinguishes
three layers:

``` text
DESIGN PROVENANCE
    RFC / Netdev talk / early patch series
             │
             ▼
LANDING PROVENANCE
    exact accepted series + patchwork-bot / pull request
             │
             ▼
MAINLINE PROVENANCE
    canonical SHA + later Fixes: references
```

An early series is no longer promoted to "merged" merely because later
code resembles it.

This rule is particularly important for:

``` text
MPTCP+BPF
virtio-net AF_XDP
Device Memory TCP
netkit queue leasing
RTNL conversion
QUIC
P4TC
```

all of which underwent substantial redesign, partial landing, revert, or
multi-stage integration.

------------------------------------------------------------------------

# 90. Updated MPTCP provenance quality

  ----------------------------------------------------------------------
  MPTCP item                   Status               Quality
  ---------------------------- -------------------- --------------------
  native MPTCP API / Linux 5.6 merged               A/B

  initial kselftest            merged               **A**
  `048d19d444be`                                    

  2020 BPF subflow v3 design   review/design        **A**
                               provenance           

  subflow parent-cgroup        merged separately    **A**
  `3764b0c5651e3`                                   

  2022 BPF `mptcp_sock` v5     applied to bpf-next  **A landing
                                                    provenance**

  `update_socket_protocol()`   merged               **A**

  later iterator/kfunc work    evolving             pending exact
                                                    enumeration
  ----------------------------------------------------------------------

The remaining MPTCP task is now mostly mechanical SHA enumeration rather
than historical interpretation.

------------------------------------------------------------------------

# 91. Provenance verification pass 7 --- exact MPTCP/BPF landing and nexthop correction

## 91.1 2022 MPTCP/BPF `mptcp_sock` --- all seven accepted commits

The final accepted series is:

``` text
[PATCH bpf-next v5 0/7]
bpf: mptcp: Support for mptcp_sock
2022-05-19
```

Patchwork-bot reported the series applied to `bpf/bpf-next.git` and
provided the exact commit object for every patch:

``` text
3bc253c2e652
bpf: add bpf_skc_to_mptcp_sock_proto

d3294cb1e06d
selftests/bpf: Enable CONFIG_IKCONFIG_PROC in config

8039d353217c
selftests/bpf: add MPTCP test base

3bc48b56e345
selftests/bpf: test bpf_skc_to_mptcp_sock

026622346772
selftests/bpf: verify token of struct mptcp_sock

ccc090f46900
selftests/bpf: verify ca_name of struct mptcp_sock

4f90d034bba9
selftests/bpf: verify first of struct mptcp_sock
```

The BPF-next pull subsequently lists the same MPTCP selftests,
independently confirming that the accepted branch entered the BPF pull
stream.

The primary implementation anchor is therefore:

``` text
3bc253c2e652
bpf: add bpf_skc_to_mptcp_sock_proto
```

while the remaining six commits establish test/config coverage.

### Canonical MPTCP+BPF timeline

``` text
2020 v3 design series
        │
        ├── parent-cgroup part later lands separately
        │       3764b0c5651e3
        │
        ▼
2022 reworked mptcp_sock series
        │
        ▼
v5 / 7 patches
        │
        ├── 3bc253c2e652 implementation
        └── six test/config commits
        │
        ▼
bpf-next pull
        │
        ▼
later update_socket_protocol()
0dd061a6a115
```

**Quality: A.**

------------------------------------------------------------------------

## 91.2 Nexthop-object correction --- v4, not v3, was the applied final series

An earlier section called v3 the final initial series. That is
incomplete.

Timeline:

``` text
2019-06-07
v3 / 20 patches

2019-06-08
v4 / 20 patches

2019-06-10
David Miller:
"Series applied, thanks."
```

Therefore the canonical landing series is:

``` text
[PATCH v4 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
```

The design text remains the same: David Ahern calls this the final set
of the initial nexthop-object work and gives the original \~700k-route
scalability motivation.

The canonical inventory is corrected from:

``` text
v3 final series
```

to:

``` text
v3 late review revision
v4 accepted/applied final series
```

### Exact anchors recovered so far

Mainline history gives:

``` text
493ced1a
ipv4: Allow routes to use nexthop objects
```

A later selftest fix identifies:

``` text
cab14d1087d9
selftests: Add version of router_multipath.sh using nexthop objects
```

and uses it as a `Fixes:` target.

The v4 series contains, among others:

``` text
01/20 nexthops: Add ipv6 helper to walk all fib6_nh in a nexthop struct
...
11/20 ipv4: Allow routes to use nexthop objects
12/20 ipv4: Optimization for fib_info lookup with nexthops
13/20 ipv6: Allow routes to use nexthop objects
14/20 nexthops: add support for replace
...
17/20 selftests: pmtu: Add support for routing via nexthop objects
19/20 selftests: Add test with multiple prefixes using single nexthop
20/20 selftests: Add version of router_multipath.sh using nexthop objects
```

The full 20-SHA table remains incomplete, but the **accepted revision
itself is now unambiguous**.

**Quality: A for final-series identity/acceptance; B for complete commit
enumeration.**

------------------------------------------------------------------------

## 91.3 Nexthop history is larger than the final 20-patch route-integration series

The June 2019 networking pull shows that nexthop support arrived as a
broader sequence than only the final 20 patches. Earlier commits already
established:

``` text
net: nexthop uapi
net: Initial nexthop code
nexthop: Add support for IPv4 nexthops
nexthop: Add support for IPv6 gateways
nexthop: Add support for lwt encaps
nexthop: Add support for nexthop groups
selftests: Add test cases for nexthop objects
nexthop: Add entry to MAINTAINERS
```

The final v4/20 series then connects those objects comprehensively into
IPv4/IPv6 route handling and adds replacement/testing.

So the architecture should be pictured as:

``` text
nexthop UAPI/object core
        │
        ├── IPv4 / IPv6 gateway representation
        ├── lwt encap
        └── groups
        │
        ▼
v4 / 20 final initial-integration series
        │
        ├── IPv4 routes reference objects
        ├── IPv6 routes reference objects
        ├── replace
        └── route/selftests
        │
        ▼
later resilient groups
```

This is more accurate than assigning the entire nexthop-object feature
to one of the final 20 commits.

------------------------------------------------------------------------

# 92. IOAM accepted-series interpretation

The v5 series is:

``` text
[PATCH net-next v5 0/6]
Support for the IOAM Pre-allocated Trace with IPv6
2021-07-20
```

and Linux networking release history confirms IPv6 IOAM Pre-allocated
Trace support in the resulting kernel cycle.

The series has six logical pieces:

``` text
1. IPv6 IOAM UAPI/header definitions
2. data-plane Pre-allocated Trace processing
3. Generic Netlink configuration API
4. IOAM injection via lightweight tunnels
5. sysctl/documentation
6. selftests
```

Exact data-plane commit:

``` text
9ee11f0fff205b4b3df9750bff5e94f97c71b6a0
ipv6: ioam: Data plane support for Pre-allocated Trace
```

Later fixes repeatedly cite this exact object in `Fixes:`.

### Review-history nuance

The earlier v4 series was explicitly challenged as premature because the
relevant IETF documents were still drafts. v5 later added stronger
validation, RCU handling, refined sysctls and tests before the July 2021
landing generation.

This is useful provenance because it shows:

``` text
Netdev 0x14 / early implementation
        │
        ▼
v4
technical implementation largely present
but standards-maturity concern
        │
        ▼
v5
validation + RCU + API refinement + tests
        │
        ▼
mainline IOAM
```

The exact remaining five SHA-1s around `9ee11f0fff20` are still being
treated as pending rather than inferred from tree adjacency.

**Quality: A for data-plane anchor and accepted-generation identity; B
for complete six-commit enumeration.**

------------------------------------------------------------------------

# 93. Provenance table after pass 7

  -----------------------------------------------------------------------
  Feature           Landing series    Exact             Quality
                                      implementation    
                                      anchor            
  ----------------- ----------------- ----------------- -----------------
  MPTCP BPF         2022 v5/7,        `3bc253c2e652`    **A**
  `mptcp_sock`      applied to                          
                    bpf-next                            

  MPTCP parent      separate landing  `3764b0c5651e3`   **A**
  cgroup                                                

  MPTCP protocol    later BPF hook    `0dd061a6a115`    **A**
  switching                                             

  nexthop route     **v4/20 applied** IPv4 `493ced1a`;  A/B
  integration                         full set pending  

  nexthop multipath v4/20             `cab14d1087d9`    **A**
  selftest                                              

  IPv6 IOAM data    v5/6 generation   `9ee11f0fff20`    **A**
  plane                                                 

  complete IOAM v5  v5/6              five additional   B
  set                                 SHA pending       
  -----------------------------------------------------------------------

Corrections from this pass:

``` text
nexthop:
v3 final  →  v4 accepted final revision

MPTCP/BPF:
2020 v3 merge assumption
    → 2020 design lineage
    → 2022 reworked v5/7 actual BPF landing
```

These corrections are now the canonical interpretation used by the
document.

------------------------------------------------------------------------

# 94. Provenance verification pass 8 --- canonical landing confidence

## 94.1 MPTCP/BPF v5 is now fully canonical

The seven-commit 2022 `mptcp_sock` series is no longer merely an
accepted-series reference. Patchwork-bot returned direct
`git.kernel.org/bpf/bpf-next/c/<SHA>` objects for every patch:

``` text
3bc253c2e652  bpf: add bpf_skc_to_mptcp_sock_proto
d3294cb1e06d  selftests/bpf: Enable CONFIG_IKCONFIG_PROC in config
8039d353217c  selftests/bpf: add MPTCP test base
3bc48b56e345  selftests/bpf: test bpf_skc_to_mptcp_sock
026622346772  selftests/bpf: verify token of struct mptcp_sock
ccc090f46900  selftests/bpf: verify ca_name of struct mptcp_sock
4f90d034bba9  selftests/bpf: verify first of struct mptcp_sock
```

The implementation anchor `3bc253c2e652` is also independently
referenced by later stable/fix work, including a 5.19 stable fix and
2026 MPTCP/BPF correctness fixes.

This satisfies all three provenance layers:

``` text
review/design
    2020 subflow-BPF series
        ↓
landing
    2022 v5/7 + patchwork applied confirmation
        ↓
mainline durability
    later Fixes: 3bc253c2e652
```

**Quality: A, high confidence.**

------------------------------------------------------------------------

## 94.2 MPTCP parent-cgroup landing is independently canonical

The separately landed cgroup behavior is:

``` text
3764b0c5651e
mptcp: attach subflow socket to parent cgroup
```

Patchwork-bot for the December 2020 MPTCP net-next series provides the
canonical `git.kernel.org` object directly.

This confirms the historical split:

``` text
2020 BPF proposal
    included parent-cgroup concept
          │
          ├── parent-cgroup fix lands in MPTCP net-next
          │     3764b0c5651e
          │
          └── BPF mptcp_sock API is redesigned and lands in 2022
                3bc253c2e652...
```

So these should remain separate canonical inventory entries rather than
be attributed to one BPF series.

------------------------------------------------------------------------

## 94.3 Nexthop final landing revision is confirmed, hashes remain conservative

The accepted nexthop route-integration revision is conclusively:

``` text
[PATCH v4 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
```

David Miller's response on 2019-06-10 is explicit:

``` text
Series applied, thanks.
```

The June 8 archive also exposes the v4 subjects, including:

``` text
11/20 ipv4: Allow routes to use nexthop objects
12/20 ipv4: Optimization for fib_info lookup with nexthops
13/20 ipv6: Allow routes to use nexthop objects
14/20 nexthops: add support for replace
17/20 selftests: pmtu: Add support for routing via nexthop objects
19/20 selftests: Add test with multiple prefixes using single nexthop
20/20 selftests: Add version of router_multipath.sh using nexthop objects
```

Known exact anchors remain:

``` text
493ced1a...    ipv4: Allow routes to use nexthop objects
cab14d1087d9   router_multipath nexthop-object selftest
```

Search indexing did not expose a trustworthy canonical mapping for every
one of the twenty commits. They therefore remain **pending rather than
inferred**.

This is intentional: series acceptance is Quality A, while complete SHA
enumeration remains Quality B.

------------------------------------------------------------------------

## 94.4 IOAM v5: exact series date and revision changes

The final v5 series was posted:

``` text
2021-07-20 21:42:55 +0200
[PATCH net-next v5 0/6]
Support for the IOAM Pre-allocated Trace with IPv6
```

v5 explicitly added/refined:

``` text
sysctl types/min/max/defaults
wide IOAM-ID sysctls
stronger header validation
RCU for schema↔namespace pointers
per-operation Generic Netlink policies
selftests
removal of virtual/anonymous tunnel decapsulation
```

This explains why the v5 generation, rather than the earlier v4, is the
correct provenance anchor.

The exact data-plane object remains:

``` text
9ee11f0fff205b4b3df9750bff5e94f97c71b6a0
ipv6: ioam: Data plane support for Pre-allocated Trace
```

and a February 2026 receive-path overflow fix again carries:

``` text
Fixes: 9ee11f0fff20
```

Thus the IOAM data-plane provenance is now doubly anchored by:

``` text
v5 accepted-generation history
        +
later mainline Fixes history
```

The other five v5 objects remain pending exact canonical enumeration.

------------------------------------------------------------------------

# 95. Provenance-confidence convention

The document now uses a stricter interpretation of Quality A:

``` text
Quality A
    accepted/pulled series identity
        +
    exact canonical SHA
        +
    preferably independent later mainline evidence
    (Fixes:, stable backport, pull log, etc.)

Quality B
    accepted series/release is certain
        +
    only part of the exact SHA set is enumerated

Quality C
    development series/status is certain
        +
    exact landing still incomplete

Quality D
    RFC/design/proposal; no merge assumed
```

Under this definition:

  Feature                              Confidence
  ------------------------------------ -------------------------------------
  MPTCP/BPF `mptcp_sock` 2022          **A**
  MPTCP parent-cgroup                  **A**
  `update_socket_protocol()`           **A**
  AF_XDP multi-buffer                  **A**
  Device Memory TCP RX/TX              **A**
  BPF qdisc                            **A**
  netkit core / queue leasing          **A**
  BIG TCP IPv4 / tunnel series         **A**
  IOAM data-plane                      **A**
  IOAM complete v5 series              **B**
  nexthop v4/20 landing                **A for series / B for all hashes**
  initial MPTCP 5.6 complete SHA set   **B**

The remaining work is therefore concentrated in complete multi-commit
enumeration, not in determining whether these features actually landed.

------------------------------------------------------------------------

# 96. Provenance verification pass 9 --- MPTCP 5.6 and IOAM commit expansion

## 96.1 MPTCP 5.6: core implementation anchors

The initial native MPTCP implementation can now be anchored more
precisely than by the selftest alone.

### Socket infrastructure

``` text
f870fa0b5768842cb4690c1c11f19f28b731ae6d
mptcp: Add MPTCP socket stubs
```

This commit creates the MPTCP socket infrastructure and allows userspace
to call:

``` c
socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP)
```

At this point the socket is still essentially a wrapper around one
regular TCP subflow; the full on-wire MPTCP protocol is added by the
subsequent commits in the initial series.

### Receive path

``` text
648ef4b88673
mptcp: Implement MPTCP receive path
```

This commit parses incoming DSS information, carries MPTCP metadata
through an skb extension, validates mappings and exposes in-sequence
data to the MPTCP socket layer.

Its status as an introduction anchor is independently confirmed by 2026
fixes using:

``` text
Fixes: 648ef4b88673
("mptcp: Implement MPTCP receive path")
```

### Initial selftest

``` text
048d19d444be
mptcp: add basic kselftest for mptcp
```

remains the test-suite anchor.

### More accurate initial lineage

``` text
TCP/socket prerequisites
        │
        ▼
f870fa0b5768
MPTCP socket stubs
        │
        ▼
initial option / handshake / send-receive commits
        │
        ├── 648ef4b88673 receive path
        │
        ▼
048d19d444be
basic kselftest
        │
        ▼
Linux 5.6
native MPTCP generation
```

This is significantly stronger provenance than treating the selftest as
the only exact 5.6 anchor.

**Quality: A for socket-stub and receive-path anchors; B for complete
initial-series SHA enumeration.**

------------------------------------------------------------------------

## 96.2 IOAM v5: four of six commits now exactly anchored

The final v5 series contains:

``` text
1. uapi: IPv6 IOAM headers definition
2. ipv6: ioam: Data plane support for Pre-allocated Trace
3. ipv6: ioam: IOAM Generic Netlink API
4. ipv6: ioam: Support for IOAM injection with lwtunnels
5. ipv6: ioam: Documentation for new IOAM sysctls
6. selftests: net: Test for the IOAM insertion with IPv6
```

Exact canonical objects now independently verified:

``` text
9ee11f0fff205b4b3df9750bff5e94f97c71b6a0
ipv6: ioam: Data plane support for Pre-allocated Trace

3edede08ff37c6a9370510508d5eeb54890baf47
ipv6: ioam: Support for IOAM injection with lwtunnels

de8e80a54c96d2b75377e0e5319a64d32c88c690
ipv6: ioam: Documentation for new IOAM sysctls

968691c777af78d2daa2ee87cfaeeae825255a58
selftests: net: Test for the IOAM insertion with IPv6
```

The networking pull for Linux 5.15 lists the complete IOAM feature set,
including the above six feature commits plus a later selftest
improvement.

### Remaining exact objects

Still pending canonical SHA verification:

``` text
uapi: IPv6 IOAM headers definition
ipv6: ioam: IOAM Generic Netlink API
```

These are known to be present in the final v5 series and the Linux 5.15
networking pull, but their SHA values are not inferred merely from
parent adjacency.

### IOAM provenance chain

``` text
Netdev 0x14
implementation discussion
       │
       ▼
v3 / v4 review
       │
       ▼
v5 / 6 final generation
       │
       ├── UAPI
       ├── 9ee11f0fff20 data plane
       ├── Generic Netlink
       ├── 3edede08ff37 lwtunnel injection
       ├── de8e80a54c96 documentation
       └── 968691c777af selftest
       │
       ▼
Linux 5.15
       │
       ▼
5.16+ encapsulation / later export and security work
```

**Quality: A for 4/6 exact commits; B only for the two still-unresolved
hashes.**

------------------------------------------------------------------------

# 97. Nexthop core: exact initial-object anchor

The initial nexthop infrastructure has an independently verified core
commit:

``` text
ab84be7e54fc
net: Initial nexthop code
```

Later 2025/2026 bug and memory-accounting discussions explicitly use:

``` text
Fixes: ab84be7e54fc
("net: Initial nexthop code")
```

This is an important distinction from the later route-integration
series.

The architecture is now anchored as:

``` text
ab84be7e54fc
Initial nexthop object infrastructure
        │
        ├── IPv4 nexthops
        ├── IPv6 gateways
        ├── lwt encapsulation
        └── nexthop groups
        │
        ▼
2019 v4 / 20
IPv4/IPv6 route integration
        │
        ├── 493ced1a... IPv4 route use
        └── cab14d1087d9 multipath selftest
        │
        ▼
2021
resilient nexthop groups
```

This resolves an ambiguity in earlier versions of the document: the
20-patch v4 series is the final **route-integration** stage of the
initial work, not the first creation of the nexthop object
infrastructure.

**Quality: A for the object-core anchor and final-series identity; B for
full route-series SHA enumeration.**

------------------------------------------------------------------------

# 98. Release-level cross-check

The exact commit work is also checked against release-level pull
requests rather than treated in isolation.

### MPTCP

The initial MPTCP socket and protocol commits are part of the native
MPTCP generation landing for Linux 5.6.

### IOAM

The Linux 5.15 networking pull explicitly lists:

``` text
uapi: IPv6 IOAM headers definition
ipv6: ioam: Data plane support for Pre-allocated Trace
ipv6: ioam: IOAM Generic Netlink API
ipv6: ioam: Support for IOAM injection with lwtunnels
ipv6: ioam: Documentation for new IOAM sysctls
selftests: net: Test for the IOAM insertion with IPv6
```

This release-level cross-check protects against confusing a commit from
a development branch with an actually pulled feature.

------------------------------------------------------------------------

# 99. Provenance status after pass 9

  ---------------------------------------------------------------------
  Lineage                            Exact coverage
  ---------------------------------- ----------------------------------
  MPTCP 5.6 socket API               `f870fa0b5768` verified

  MPTCP 5.6 RX path                  `648ef4b88673` verified

  MPTCP initial selftest             `048d19d444be` verified

  MPTCP/BPF 2022                     all 7 commits verified

  MPTCP protocol switching           `0dd061a6a115` verified

  IOAM v5                            **4/6 exact commits verified**

  nexthop object core                `ab84be7e54fc` verified

  nexthop v4 route integration       accepted series verified; partial
                                     SHA set

  resilient nexthop                  baseline previously verified
  ---------------------------------------------------------------------

The remaining SHA gaps are now small enough that they should be treated
as an appendix normalization task rather than blocking the architectural
history.

------------------------------------------------------------------------

# 100. Provenance verification pass 10 --- conservative close-out of exact-SHA gaps

## 100.1 IOAM: series/release mapping is complete; SHA mapping remains 4/6

The Linux 5.15 networking pull independently confirms the exact six
feature subjects:

``` text
uapi: IPv6 IOAM headers definition
ipv6: ioam: Data plane support for Pre-allocated Trace
ipv6: ioam: IOAM Generic Netlink API
ipv6: ioam: Support for IOAM injection with lwtunnels
ipv6: ioam: Documentation for new IOAM sysctls
selftests: net: Test for the IOAM insertion with IPv6
```

Exact canonical objects already verified:

``` text
9ee11f0fff20  Data plane support for Pre-allocated Trace
3edede08ff37  IOAM injection with lwtunnels
de8e80a54c96  Documentation for new IOAM sysctls
968691c777af  IOAM IPv6 insertion selftest
```

The two remaining subjects are known to be in the pulled feature set:

``` text
uapi: IPv6 IOAM headers definition
ipv6: ioam: IOAM Generic Netlink API
```

but this audit does not assign SHA values without an independently
verifiable canonical object.

Thus:

``` text
series identity       complete
release mapping       complete
subject enumeration  complete
SHA enumeration       4 / 6
```

This is a **B only for hash completeness**, not uncertainty about
whether the feature landed.

------------------------------------------------------------------------

## 100.2 Nexthop: split the provenance into object-core and route-integration generations

The June 2019 networking pull exposes the initial object-core generation
as a distinct sequence before the final route-integration v4/20 series.

Object-core subjects include:

``` text
net: nexthop uapi
net: Initial nexthop code
nexthop: Add support for IPv4 nexthops
nexthop: Add support for IPv6 gateways
nexthop: Add support for lwt encaps
nexthop: Add support for nexthop groups
selftests: Add test cases for nexthop objects
nexthop: Add entry to MAINTAINERS
```

Exact core anchor:

``` text
ab84be7e54fc
net: Initial nexthop code
```

The second generation is the accepted v4/20 route-integration series:

``` text
[PATCH v4 net-next 00/20]
net: Enable nexthop objects with IPv4 and IPv6 routes
```

with explicit maintainer response:

``` text
Series applied, thanks.
```

Known exact route-integration anchors:

``` text
493ced1a...
ipv4: Allow routes to use nexthop objects

cab14d1087d9
selftests: Add version of router_multipath.sh using nexthop objects
```

### Why this split matters

A single "nexthop objects commit" would hide two architectural steps:

``` text
step 1:
create reusable nexthop objects/groups
        │
        ▼
step 2:
teach IPv4/IPv6 FIB routes to reference those objects
```

The later resilient-nexthop work then builds on both.

------------------------------------------------------------------------

## 100.3 MPTCP 5.6: exact anchors versus complete protocol series

The initial MPTCP generation has three independently strong exact
anchors:

``` text
f870fa0b5768
mptcp: Add MPTCP socket stubs

648ef4b88673
mptcp: Implement MPTCP receive path

048d19d444be
mptcp: add basic kselftest for mptcp
```

These correspond to three different layers:

``` text
socket/API infrastructure
        ↓
protocol data path
        ↓
end-to-end selftest
```

This is enough to establish the Linux 5.6 native-MPTCP generation
without pretending that one SHA represents the complete protocol.

The initial series also contains option parsing/creation, handshake and
send-side work, but exact SHA values for every one of those commits
remain outside the verified set.

### Canonical wording

Prefer:

``` text
Linux 5.6 introduced the first native upstream MPTCP implementation through a
multi-commit series. Exact anchors include f870fa0b5768 for the socket stubs,
648ef4b88673 for the receive path, and 048d19d444be for the initial kselftest.
```

Avoid:

``` text
MPTCP was introduced by commit 048d19d444be
```

because that commit is only the selftest.

------------------------------------------------------------------------

# 101. What "complete" means for this change log

For a feature to be useful in this historical document, it is not
necessary to list every mechanical commit.

The audit now distinguishes:

``` text
architecture completeness
    Can we explain what changed and why?

landing completeness
    Can we identify the accepted series/pull and release?

anchor completeness
    Do we have exact representative mainline commits?

enumeration completeness
    Do we have every commit in the multi-commit feature?
```

For the major networking lineages:

``` text
architecture completeness   high
landing completeness        high
anchor completeness         high
enumeration completeness    intentionally mixed
```

This avoids spending disproportionate effort reconstructing dozens of
mechanical refactoring commits while still making every major claim
auditable.

------------------------------------------------------------------------

# 102. Recommended canonical inventory schema

The final normalized table should use:

``` text
Date
LWN article/topic
Subsystem
Kernel release
Status
Design/RFC provenance
Accepted series / pull
Representative mainline commit(s)
Netdev conference session
Phase
Tags
```

and **not** a single `Commit` column implying that every feature has one
defining SHA.

Example:

``` text
Feature:
IPv6 IOAM Pre-allocated Trace

Accepted series:
v5 / 6

Representative commits:
9ee11f0fff20  data plane
3edede08ff37  lwtunnel
de8e80a54c96  docs
968691c777af  selftest

Completeness:
4/6 exact SHA; 6/6 subjects/release verified
```

This schema better represents how Linux networking development actually
lands.

------------------------------------------------------------------------

# 103. Audit state at the end of exact-provenance passes

### Fully strong / Quality A representative provenance

``` text
MPTCP/BPF mptcp_sock
update_socket_protocol
AF_XDP multi-buffer
Device Memory TCP RX
Device Memory TCP TX
BPF qdisc
netkit core
netkit queue leasing/remerge
IPv4 BIG TCP
BIG TCP over VXLAN/GENEVE
virtio-net AF_XDP RX small
virtio-net AF_XDP RX mergeable
virtio-net AF_XDP TX
IOAM data plane
nexthop object core
AccECN core/negotiation anchors
```

### Strong landing + partial full enumeration

``` text
initial MPTCP 5.6
IPv6 IOAM complete v5 series
2019 nexthop route integration
RTNL multi-release conversion
initial IPv6 BIG TCP
```

### Intentionally retained as development/RFC status

``` text
P4TC stalled direction
kernel QUIC development
some virtio-net multi-buffer ZC work
newer SRv6 proposal work
per-netns conntrack hash RFC
```

The document is therefore now much closer to a **historical networking
change database** than a simple LWN reading list.

------------------------------------------------------------------------

# 104. Kernel Recipes cross-reference

Source library:

``` text
Kernel Recipe Archives
Document Library
https://archives.kernel-recipes.org/document-library/
```

The first pass uses:

``` text
Category = networking
```

and then supplements it with talks filed under other categories whose
contents directly intersect the networking change log.

## 104.1 In-scope conference talks found so far

The document's time range begins on 2019-05-07. Therefore Kernel Recipes
2019 is in scope.

### Kernel Recipes 2019 --- XDP closer integration with network stack

``` text
Speaker:
Jesper Dangaard Brouer

Category:
networking

Year:
2019
```

Main idea:

``` text
XDP
 │
 ├── fast programmable layer before skb/netstack
 │
 ├── may bypass much of the normal stack
 │
 └── but should integrate with existing kernel facilities
       ├── routing
       ├── ARP/neighbour
       └── other in-kernel tables
```

A particularly interesting proposal in the abstract is moving skb
allocation out of drivers by using XDP frames.

### Relation to the change log

This is useful architectural provenance for:

``` text
XDP
 ↓
XDP redirect / frame handling
 ↓
page_pool and RX-memory work
 ↓
AF_XDP
 ↓
AF_XDP multi-buffer
 ↓
netkit / queue leasing
```

It also reinforces an important theme of the later networking work:

``` text
high performance
does not necessarily mean
kernel bypass
```

That same idea later appears in Device Memory TCP, io_uring zero-copy RX
and netkit: retain kernel TCP/networking semantics while removing
unnecessary allocation/copy/queue costs.

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/xdp-closer-integration-with-network-stack/`

------------------------------------------------------------------------

### Kernel Recipes 2019 --- BPF at Facebook

``` text
Speaker:
Alexei Starovoitov

Category:
networking

Year:
2019
```

The archive describes production BPF uses including:

-   scaling networking;
-   denial-of-service protection;
-   container security;
-   performance analysis.

### Relation to the change log

This is broad architectural context for the subsequent progression:

``` text
BPF networking deployment
        │
        ▼
BPF struct_ops
        │
        ▼
SK_LOOKUP / socket hooks
        │
        ▼
MPTCP protocol selection
        │
        ▼
netkit
        │
        ▼
BPF qdisc
```

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/bpf-at-facebook/`

------------------------------------------------------------------------

## 104.2 Kernel Recipes 2023 --- Netconf 2023 Workshop

``` text
Speaker:
David Miller

Category:
networking

Year:
2023
```

This is the most important Kernel Recipes networking artifact found in
the first pass.

The slides summarize the Netconf workshop held at Kernel Recipes and
name the major discussion areas.

### Willem de Bruijn

``` text
complex-code refactoring
SO_DEVMEM direct GPU data placement
```

This maps directly onto:

``` text
page_pool / memory-provider work
        ↓
netmem
        ↓
Device Memory TCP
```

and is especially valuable because it captures SO_DEVMEM while the
architecture was still being designed.

### Daniel Borkmann

The workshop summary lists:

``` text
header/data split
BIG TCP and zero copy
XDP + bpf_mprog
per-queue XDP programs
```

These connect to several later lines:

``` text
BIG TCP
AF_XDP
netkit / queue-oriented datapaths
multi-program BPF networking
```

### Eric Dumazet

Topics include:

``` text
struct file reorganization
avoiding skb clone in NIT tap
deferred wakeups
three-band FQ with WRR
UDP accept()
```

These belong to the broader socket/skb/queue scalability work
surrounding the release timeline.

### David Ahern

The workshop summary lists:

``` text
Linux TCP for machine-learning use cases
BIG TCP for IPv6
zero-cost counters for userspace monitoring
```

The ML/TCP discussion is particularly relevant to the later Device
Memory TCP and AI-networking work.

### Florian Westphal

Topics include:

``` text
IPsec workshop
multi-CPU spreading for single-tunnel acceleration
iptables support for offload
IPtap traffic-flow security
PF_KEY deprecation
nftables CVE fixes
```

This provides conference context for the document's netfilter/offload
and IPsec tracks.

### Other networking topics

The slides also mention:

``` text
SYN proxy at scale with BPF
TCP extended data offset
software simulation of hardware offloads
netdev feature expansion
driver review / devlink device orchestration
DPU/IPU modeling
```

### Provenance role

Unlike a single feature talk, Netconf 2023 is best represented as:

``` text
Kernel Recipes / Netconf workshop
        │
        ├── SO_DEVMEM ───────────→ Device Memory TCP
        ├── BIG TCP ─────────────→ later BIG TCP work
        ├── XDP/bpf_mprog ───────→ programmable datapath evolution
        ├── per-queue XDP ───────→ queue-level APIs / netkit context
        ├── TCP for ML ──────────→ AI-networking / devmem direction
        ├── IPsec scaling ───────→ networking offload/scalability
        └── nftables ────────────→ netfilter maintenance
```

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/netconf-2023-workshop/`

Slides:
`https://archives.kernel-recipes.org/wp-content/uploads/2025/01/netconf_2023.pdf`

------------------------------------------------------------------------

# 105. Cross-category Kernel Recipes talks relevant to networking

Category filtering alone is not sufficient.

## 105.1 Kernel Recipes 2019 --- Faster IO through io_uring

``` text
Speaker:
Jens Axboe

Category:
storage

Year:
2019
```

This predates the networking-specific io_uring work, but is important
architectural provenance for the interface itself.

Lineage:

``` text
2019 Kernel Recipes
io_uring high-performance asynchronous I/O model
        │
        ▼
2021/2022
io_uring zero-copy network TX work
        │
        ▼
2023
On the way to io_uring networking
        │
        ▼
2025
io_uring zero-copy RX
```

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/faster-io-through-io_uring/`

------------------------------------------------------------------------

## 105.2 Kernel Recipes 2022 --- What's new with io_uring

``` text
Speaker:
Jens Axboe

Category:
storage

Year:
2022
```

Although filed as storage, the archive describes io_uring as a
consistent high-performance I/O model, and it sits chronologically in
the same period as the first serious io_uring zero-copy networking work.

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/whats-new-with-io_uring/`

------------------------------------------------------------------------

## 105.3 Kernel Recipes 2023 --- On the way to io_uring networking

``` text
Speaker:
Pavel Begunkov

Category:
storage

Year:
2023
```

The archive explicitly describes networking as io_uring's next
performance frontier.

This is therefore a direct networking talk despite its archive category.

Lineage:

``` text
io_uring storage-oriented foundation
        │
        ▼
network send/recv APIs
        │
        ▼
zero-copy TX
        │
        ▼
zero-copy RX design
        │
        ▼
page_pool / netmem integration
        │
        ▼
Linux 6.15 io_uring ZCRX
```

This should be treated as a **merge-era/design-era conference
counterpart** to the io_uring networking LWN series.

Kernel Recipes archive:
`https://archives.kernel-recipes.org/document/on-the-way-to-io_uring-networking/`

------------------------------------------------------------------------

# 106. Kernel Recipes → LWN → mainline mapping

  ----------------------------------------------------------------------------------------
  Kernel                     Year Archive       Change-log lineage  Phase
  Recipes                         category                          
  ------------- ----------------- ------------- ------------------- ----------------------
  XDP closer                 2019 networking    XDP →               design/architecture
  integration                                   page_pool/AF_XDP →  
  with network                                  netkit              
  stack                                                             

  BPF at                     2019 networking    BPF networking →    deployment/design
  Facebook                                      struct_ops/socket   
                                                hooks/netkit        

  Faster IO                  2019 storage       io_uring →          pre-networking
  through                                       networking TX/RX    foundation
  io_uring                                                          

  What's new                 2022 storage       io_uring → ZC       foundation/merge-era
  with io_uring                                 networking          

  On the way to              2023 storage       io_uring ZC TX/RX   design/merge-era
  io_uring                                                          
  networking                                                        

  Netconf 2023               2023 networking    SO_DEVMEM, BIG TCP, multi-lineage workshop
  Workshop                                      XDP/BPF, IPsec,     
                                                nftables, TCP/ML    
  ----------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 107. Kernel Recipes and Netdev serve different provenance roles

The two conference archives complement each other.

``` text
Netdev
  tends to provide:
  - focused networking implementation talks
  - protocol/datapath design
  - pre-merge and merge-era detail

Kernel Recipes
  tends to provide:
  - broader kernel architecture context
  - maintainer/workshop summaries
  - production-use perspective
  - cross-subsystem links such as io_uring
```

For example:

``` text
Device Memory TCP

Kernel Recipes 2023 / Netconf
SO_DEVMEM + direct GPU placement
            │
            ▼
Netdev 0x16 / 0x17 / 0x18
architecture → implementation → BoF
            │
            ▼
LWN patch-series coverage
            │
            ▼
Linux 6.12 mainline
```

and:

``` text
io_uring networking

Kernel Recipes 2019
io_uring foundation
       │
Kernel Recipes 2022
io_uring evolution
       │
Kernel Recipes 2023
"On the way to io_uring networking"
       │
Netdev 0x17 / 0x18
zero-copy RX implementation / BoF
       │
LWN series
       │
Linux 6.15 ZCRX
```

This additional conference layer makes the change log useful not only
for answering "when was it merged?" but also "where did the
architectural direction come from?"
