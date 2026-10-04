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
