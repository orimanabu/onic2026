# Linux networking evolution — 深掘り用語集

**対応本文:** `linux-networking-evolution-v3.0-to-7.x-canonical-edition-r69.md`  
**版:** v6 / 2026-10-08

この文書は canonical edition r69 のストーリーを繰り返すものではなく、本文で圧縮した技術要素を **「何を解決するか / kernel object・data path / 何が重要か / 関連技術 / 出典」** の順で引けるようにした補助資料である。

## 読み方

- release chronology と exact mainline anchor の正本は r69 Part VI / VII とする。本用語集は mechanism と object model の理解を優先する。
- version-sensitive な API は `kernel.org/doc/html/<version>/` で対象 kernel version の documentation を確認する。`latest` は現在の設計を理解するための入口として使う。
- conference source は design motivation / implementation trade-off の理解に使い、release attribution は kernel documentation / LWN / r69 Part VI・VII と突き合わせる。
- URL はユーザー指定の source family（man7, kernel.org, LWN, NetDev, Kernel Recipes, Linux Plumbers Conference）に限定した。

---

# 1. Queueing / latency / pacing / placement

## DQL / BQL — queue に「何 byte 滞留させるか」を制御する

**DQL (Dynamic Queue Limits)** は producer が consumer に渡す outstanding data の量を動的に制限する汎用機構で、network driver での代表的利用が **BQL (Byte Queue Limits)** である。NIC TX ring に大量の packet を積み過ぎると、software qdisc が優れた scheduling/AQM を行っても、その後段の hardware queue に長い backlog が残り latency が増える。BQL は「descriptor 数」ではなく概ね byte 単位で device queue の in-flight amount を制御し、qdisc 側へ backpressure を戻す。

重要なのは BQL が packet scheduling algorithm そのものではない点である。BQL は **driver/NIC 境界の accounting contract**、qdisc は **packet をいつ・どの順序で送るかという policy** を担当する。この分離により fq_codel や pacing が意味を持つ。

**関連:** qdisc, CoDel/fq_codel, TSQ, sch_fq, driver TX ring.

**Sources**
- LWN — BQL patch series: https://lwn.net/Articles/469652/
- Kernel documentation index: https://www.kernel.org/doc/html/latest/networking/

## CoDel / fq_codel — queue length ではなく sojourn time を見る

CoDel は persistent queue が作る bufferbloat を、queue 内 packet の **sojourn time** を観測して制御する AQM である。fq_codel は flow queueing と CoDel を組み合わせ、elephant flow が他 flow の latency を押し上げにくくする。ここでの設計上の変化は「queue が一杯になったら drop」から「queueing delay と flow fairness を policy として制御」への移行である。

BQL と組み合わせると、software qdisc の後ろに巨大な driver queue を作りにくくなり、AQM の判断が実際の wire scheduling に反映されやすい。

**Sources**
- LWN — topic-specific article: https://lwn.net/Articles/507651/Articles/507651/
- Kernel documentation: https://www.kernel.org/doc/html/latest/networking/

## TSQ / sch_fq / TCP pacing — transport と qdisc の時間軸を接続する

**TCP Small Queues (TSQ)** は TCP socket が qdisc/device path に過剰な量を先行投入することを抑える。**sch_fq** は flow queueing と pacing を提供し、TCP が算出した pacing rate を packet departure time に反映できる。Linux では後に TCP 自身の pacing timer も整備され、qdisc のみに依存しない構成も可能になった。

BBR を理解するときは congestion window だけでなく pacing が重要である。BBR は bandwidth/RTT model から sending rate を決めるため、「何 packet outstanding にするか」だけではなく「いつ送るか」が data path の一級の control variable になる。

**Sources**
- LWN — topic-specific article: https://lwn.net/Articles/701165/
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/

## SO_TXTIME / EDT / taprio — packet に送信時刻を持たせる

`SO_TXTIME` は socket から packet の desired transmit time を指定する UAPI、EDT (Earliest Departure Time) は scheduling model、`taprio` は time-aware priority scheduling を Linux TC に持ち込む。これにより queueing は単なる backlog 処理ではなく **time-domain scheduling** を扱う。

用途は TSN だけではない。pacing や host-level traffic scheduling を「rate」から「departure timestamp」へ落とすと、複数の scheduler/offload layer 間で時刻を contract として共有できる。

**Sources**
- man7 socket/timestamping entry point: https://man7.org/linux/man-pages/
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/

## RPS / RFS / XPS — packet processing の CPU placement

RPS は RX packet processing を別 CPU へ software steering する。RFS は flow/application locality を考慮して CPU を選び、XPS は TX queue selection を CPU/queue topology と結びつける。RSS が NIC hardware の RX queue placement なら、RPS/RFS/XPS は host 内 execution placement を software で補完する系列である。

後の CPUMAP、queue object、queue leasing と比較すると、placement の対象が「CPU」から「明示的 queue object / execution context」へ広がったことが分かる。

**Sources**
- Kernel scaling documentation: https://www.kernel.org/doc/html/latest/networking/scaling.html
- Kernel BPF CPUMAP documentation: https://www.kernel.org/doc/html/latest/bpf/map_cpumap.html

## Wi-Fi airtime scheduling / AQL

Wi-Fi は Ethernet のように byte count だけで medium occupancy を近似しにくい。PHY rate が station ごとに異なるため、同じ byte 数でも airtime cost が異なる。airtime fairness は scheduling resource を byte ではなく **airtime** として扱い、AQL (Airtime Queue Limits) は driver/hardware 側に過剰な airtime backlog を作らないよう制御する。

BQL と同じく「下位 queue を無制限に埋めない」という思想を持つが、accounting unit が byte ではなく airtime である点が重要である。

**Sources**

- Kernel wireless/networking documentation: https://www.kernel.org/doc/html/latest/

## `dev_queue_xmit()` llist — shared qdisc の contention を減らす

近年の TX scalability 改善では、新しい UAPI を追加せず内部 data structure と batching を変えるものも重要である。`dev_queue_xmit()` 周辺の llist 化は、shared qdisc path での cross-CPU contention/cacheline bouncing を減らす implementation scalability の例である。

これは explicit resource contract の話とは別系列であり、同じ semantics をより少ない synchronization cost で実装する進化として読む。

**Sources**

- Kernel networking source documentation: https://www.kernel.org/doc/html/latest/networking/

---

# 2. Packet aggregation — GRO / GSO / BIG TCP

## GRO / GSO

GRO (Generic Receive Offload) は RX 側で複数 packet を大きな processing unit にまとめ、protocol stack の per-packet overhead を減らす。GSO (Generic Segmentation Offload) は TX 側で大きな logical packet を stack 内で扱い、NIC または software segmentation まで分割を遅延する。

重要なのは **wire MTU を変えずに kernel 内部の processing granularity を大きくする**点である。checksum、segmentation metadata、tunnel encapsulation との整合が必要になる。

**Sources**
- Kernel segmentation offloads: https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html
- LWN — topic-specific article: https://lwn.net/Articles/884104/

## BIG TCP

BIG TCP は IPv6、後に IPv4 の GRO/GSO path で 64 KiB を超える internal packet size を扱えるようにし、高速 link で per-packet processing cost をさらに下げる。これは jumbo frame とは異なり、wire packet size を巨大化する機能ではない。

BIG TCP の発展では tunnel/VXLAN/GENEVE 等との組み合わせ、GRO/GSO metadata の表現幅、driver/offload capability が重要になる。

**Sources**
- NetDev 0x15 — BIG TCP program entry: https://netdevconf.info/0x15/accepted-sessions.html
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/
- LWN — topic-specific article: https://lwn.net/Articles/884104/

---

# 3. BPF — verified kernel extension

## eBPF ISA / `bpf()` syscall / maps

eBPF は packet filter instruction set から、kernel 内の多数の hook で動く汎用 verified execution environment へ発展した。`bpf()` syscall は program/map/object の lifecycle を userspace から管理し、BPF map は program 間および kernel/userspace 間の persistent state contract になる。

program と state を分けたことが重要で、program を差し替えても map を維持する、複数 program が map を共有する、といった composition が可能になる。

**Sources**
- NetDev 0.1 — BPF In-kernel Virtual Machine: https://netdevconf.info/0.1/sessions.html
- man7 `bpf(2)`: https://man7.org/linux/man-pages/man2/bpf.2.html
- Kernel BPF docs: https://www.kernel.org/doc/html/latest/bpf/
- BPF maps: https://www.kernel.org/doc/html/latest/bpf/maps.html
- Kernel Recipes, BPF at Facebook: https://archives.kernel-recipes.org/document-library/

## BPF verifier

verifier は program load 時に control-flow と register/stack state を解析し、pointer provenance、bounds、helper/kfunc call contract 等を検証する。これにより native kernel module のように任意 pointer を操作させず、制約付きの kernel extension を成立させる。

現代 BPF の理解では「JIT が速い」こと以上に、**typed context + verifier + attach point + helper/kfunc contract** の組み合わせが重要である。

**Sources**
- Kernel verifier documentation: https://www.kernel.org/doc/html/latest/bpf/verifier.html

## TC BPF / TCX

TC BPF は classifier/action path に BPF program を配置し、skb を対象に programmable policy を実装する。TCX は BPF link ベースの attachment/lifecycle を TC ingress/egress に導入し、legacy TC BPF attachment より明示的な program ownership と update model を提供する。

XDP が driver RX の早い execution point を主戦場にするのに対し、TC/TCX は skb、qdisc、routing 等と接続しやすい位置にある。

**Sources**
- Kernel libbpf program types: https://www.kernel.org/doc/html/latest/bpf/libbpf/program_types.html
- man7 TC entry point: https://man7.org/linux/man-pages/

## cgroup BPF / socket hooks / SK_LOOKUP

cgroup BPF は process/container grouping と network policy hook を結びつける。socket address/connect/bind、sockops、SK_MSG/SOCKMAP、SK_LOOKUP 等により、packet より上位の **socket/connection object** に program を attach できる。

SK_LOOKUP は incoming packet に対応する listening socket の selection に介入でき、service steering/proxying を packet rewrite だけに依存せず実装できる。

**Sources**
- Kernel program types: https://www.kernel.org/doc/html/latest/bpf/libbpf/program_types.html
- Kernel BPF docs: https://www.kernel.org/doc/html/latest/bpf/

## BPF `struct_ops`

`struct_ops` は kernel の function-pointer based operations table を BPF implementation で提供する仕組みで、TCP congestion control など「algorithm implementation」を BPF で差し替える。packet hook への attach とは異なり、**kernel subsystem の algorithm interface** を BPF object として実装する点が特徴である。

**Sources**
- Kernel libbpf program types: https://www.kernel.org/doc/html/latest/bpf/libbpf/program_types.html
- Kernel BPF docs: https://www.kernel.org/doc/html/latest/bpf/

## BTF

BTF は BPF program/map と kernel type information を結ぶ compact type metadata である。function signature/line info へ拡張され、CO-RE、typed tracing、kfunc 等の基盤になった。

observability の観点では、raw address/offset 依存から **type-aware inspection** へ移行する鍵である。

**Sources**
- Kernel BTF documentation: https://www.kernel.org/doc/html/latest/bpf/btf.html

---

# 4. XDP / AF_XDP

## XDP

XDP は driver RX の非常に早い位置で BPF program を実行し、`XDP_PASS`, `XDP_DROP`, `XDP_TX`, `XDP_REDIRECT` 等を返す。skb allocation 前に処理できるため cost を抑えられる一方、利用できる packet metadata/context は skb path と異なる。

XDP は「kernel bypass」そのものではなく、kernel 内 fast path から通常 stack へ `PASS` できる programmable layer として設計されている。

**Sources**
- NetDev 1.2 — XDP workshop / XDP hardware offload program: https://netdevconf.info/1.2/schedule.html
- Kernel Recipes 2018: https://archives.kernel-recipes.org/document/xdp-a-new-programmable-network-layer/
- Kernel Recipes 2019: https://archives.kernel-recipes.org/document/xdp-closer-integration-with-network-stack/
- Kernel BPF program types: https://www.kernel.org/doc/html/latest/bpf/libbpf/program_types.html

## DEVMAP / CPUMAP / XSKMAP

`XDP_REDIRECT` は destination を map object として明示することで発展した。DEVMAP は netdev、CPUMAP は remote CPU execution、XSKMAP は AF_XDP socket を destination object にする。

この系列は XDP を単なる verdict hook から **redirectable object graph** へ拡張したと理解できる。

**Sources**
- DEVMAP: https://www.kernel.org/doc/html/latest/bpf/map_devmap.html
- CPUMAP: https://www.kernel.org/doc/html/latest/bpf/map_cpumap.html
- AF_XDP/XSKMAP: https://www.kernel.org/doc/html/latest/networking/af_xdp.html

## AF_XDP / UMEM

AF_XDP socket (XSK) は netdev queue と userspace memory (UMEM) を結びつける。RX/TX ring に加えて UMEM の FILL/COMPLETION ring があり、frame ownership を kernel と userspace の間で移動する。

ここで重要なのは zero-copy の有無だけではなく、**queue identity + UMEM frame + ownership transfer** が UAPI object として明示される点である。completion は frame を userspace が再利用可能になったことを示すが、それ自体が packet transmission success を保証するわけではない。

**Sources**
- Kernel AF_XDP documentation: https://www.kernel.org/doc/html/latest/networking/af_xdp.html
- XSKMAP: https://www.kernel.org/doc/html/latest/bpf/map_xskmap.html

## XDP multi-buffer / fragments

single contiguous buffer に収まらない packet を XDP で扱うため、fragment/multi-buffer model が追加された。これは jumbo/GRO、AF_XDP multi-buffer、large RX buffer と関係し、XDP program が「packet = 1 buffer」と仮定できない世界への移行を示す。

**Sources**
- Kernel AF_XDP documentation: https://www.kernel.org/doc/html/latest/networking/af_xdp.html
- Kernel XDP program types: https://www.kernel.org/doc/html/latest/bpf/libbpf/program_types.html

---

# 5. Packet memory — page_pool / netmem / memory providers

## page_pool

page_pool は network RX 用 page/page fragment の allocation/recycling に特化した allocator で、DMA mapping reuse と NAPI/queue locality を考慮する。driver-private recycling を共通 API に寄せることで、XDP/ skb path から同じ lifetime を扱いやすくする。

`page_pool_params` には device, NAPI, queue index, DMA direction 等があり、単なる slab allocator ではなく **RX queue と DMA lifecycle に結びついた memory object** である。

**Sources**
- NetDev 0x19 — Diagnosing Page Pool Leaks: https://www.netdevconf.info/0x19/sessions/tutorial/diagnosing-page-pool-leaks.html
- Kernel page_pool documentation: https://www.kernel.org/doc/html/latest/networking/page_pool.html
- netdev Generic Netlink page-pool objects: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## netmem

netmem は packet memory を `struct page` に固定しない抽象化である。driver は underlying memory type を直接仮定せず、page_pool/netmem API を通して DMA address、sync、reference を扱う。

Device Memory TCP のように CPU から読めない memory が RX payload backing になり得るため、「packet data は必ず page-backed / CPU-readable」という暗黙の前提を外す役割を持つ。

**Sources**
- Kernel netmem documentation: https://www.kernel.org/doc/html/latest/networking/netmem.html

## page_pool memory providers

memory provider は page_pool の allocation/recycle lifecycle を維持しつつ、system RAM 以外の backing や userspace-registered memory 等を供給できるようにする。page_pool を置換するのではなく、**allocator/provider choice を page_pool contract の下へ差し込む**。

io_uring ZCRX との接続では、registered userspace buffers を RX queue の provider として扱うことが重要になる。

**Sources**
- page_pool: https://www.kernel.org/doc/html/latest/networking/page_pool.html
- netdev netlink queue/provider objects: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## Device Memory TCP

Device Memory TCP は TCP payload を host RAM ではなく DMA-BUF 等の device memory へ直接配置する。header は host memory で通常の TCP/IP stack が処理し、payload は device memory へ送るため、NIC の header split 等が必要になる。

狙いは host-memory bandwidth と PCIe traffic の削減であり、GPU/accelerator/storage 間 bulk transfer が主要 motivation である。

**Sources**
- NetDev 0x17 — Device Memory TCP: https://netdevconf.info/0x17/sessions/talk/device-memory-tcp.html
- Kernel Device Memory TCP documentation: https://www.kernel.org/doc/html/latest/networking/devmem.html
- NetDev 0x17: https://netdevconf.info/0x17/sessions/talk/device-memory-tcp.html
- NetDev 0x18 AI networks: https://netdevconf.info/0x18/sessions/talk/the-future-of-ai-networks-advancing-tcp-with-device-memory-and-collective-communication.html

---

# 6. Socket zero-copy / io_uring networking

## `MSG_ZEROCOPY`

`MSG_ZEROCOPY` は send path で application buffer から kernel-owned packet buffer への copy を回避するが、application は kernel/NIC が参照を終えるまで buffer を再利用できない。そのため error queue notification を通じて completion/lifetime を管理する必要がある。

zero-copy は「copy が0回」だけを意味するのではなく、**buffer ownership と completion の管理コストを application に露出する trade-off** である。

**Sources**
- man7 `send(2)`: https://man7.org/linux/man-pages/man2/send.2.html
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/

## `TCP_ZEROCOPY_RECEIVE`

TCP RX の zero-copy は stream semantics、receive queue、page mapping/lifetime を調整する必要があり、TX zero-copy より制約が大きい。`TCP_ZEROCOPY_RECEIVE` は TCP receive data を application address space へ map する方向の API である。

**Sources**
- man7 `tcp(7)`: https://man7.org/linux/man-pages/man7/tcp.7.html
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/

## io_uring `SEND_ZC` / multishot receive / ZCRX

io_uring は submission/completion queue を共通 I/O model とし、networking では send/recv、multishot receive、provided/registered buffers、SEND_ZC、ZCRX へ拡張された。`SEND_ZC` は MSG_ZEROCOPY と同じ buffer lifetime 問題を io_uring completion model に統合する。

ZCRX はさらに RX queue と registered buffer provider を結び、page_pool/memory-provider 系列と io_uring completion 系列が交差する。

**Sources**
- NetDev 0x17 — Fast ZC Rx Data Plane using io_uring: https://netdevconf.info/0x17/sessions/talk/fast-zc-rx-data-plane-using-io-uring.html
- NetDev 0x18 — Devmem TCP & io_uring zero copy / netmem: https://netdevconf.info/0x18/sessions/bof/devmem-tcp-io-uring-zero-copy.html
- man7 io_uring entry point: https://man7.org/linux/man-pages/
- Kernel netdev netlink provider information: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html
- NetDev 0x17 zero-copy RX sessions index: https://netdevconf.info/0x17/pages/sessions.html

---

# 7. Routing / nexthop / TC / hardware offload

## FIB / route object

FIB は destination prefix 等から forwarding result を引く routing data structure である。Linux の route control plane は rtnetlink を通じて更新され、network namespace ごとに routing state を持つ。

route cache removal 後の Linux は per-flow cache に依存するより、FIB lookup 自体を scalable にする方向へ進んだ。

**Sources**
- man7 `rtnetlink(7)`: https://man7.org/linux/man-pages/man7/rtnetlink.7.html
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## nexthop object

nexthop object は route entry に埋め込まれていた gateway/device/group 情報を reusable object として分離する。複数 route が同じ nexthop/group を参照でき、ECMP group の更新を route replacement から分離できる。

control-plane object を明示することで、hardware offload、notification、resilient group 等との接続がしやすくなる。

**Sources**
- LWN — topic-specific article: https://lwn.net/Articles/790828/
- Kernel rtnetlink documentation: https://www.kernel.org/doc/html/latest/networking/

## TC classifier / action / Flower

TC は qdisc だけでなく classifier/action pipeline を持つ。Flower は L2-L4 field を構造化して match し、mirred、police、ct、tunnel key 等の action と組み合わせる。

hardware offload では Flower rule を driver が理解可能な `flow_rule`/callback representation へ変換し、kernel policy object と NIC/switch table の mapping を行う。

**Sources**
- man7 TC: https://man7.org/linux/man-pages/
- Kernel switchdev docs: https://www.kernel.org/doc/html/latest/networking/switchdev.html
- LWN — topic-specific article: https://lwn.net/Articles/809333/

## bridge VLAN / MPLS / VRF

bridge VLAN filtering は Linux bridge を VLAN-aware forwarding object にする。MPLS は label forwarding を kernel FIB/control plane に統合する。VRF device は L3 master device と routing table を結び、process/socket が routing domain を選択できる。

三者は protocol は異なるが、**Linux 内の forwarding state を netdev/FIB object として表現し、標準 control plane から扱う**点で共通する。

**Sources**
- Kernel VRF: https://www.kernel.org/doc/html/latest/networking/vrf.html
- Kernel networking index: https://www.kernel.org/doc/html/latest/networking/

---

# 8. Netlink / RTNL / YNL

## Netlink / rtnetlink / Generic Netlink

Netlink は kernel/userspace 間 message bus。rtnetlink は link/address/route/rule/neighbour 等の networking control plane、Generic Netlink は subsystem ごとの family/command/attribute model を提供する。

ioctl と比較した Netlink の強みは extensible TLV attributes、dump/notification、multipart message、namespace-aware control 等である。

**Sources**
- man7 `netlink(7)`: https://man7.org/linux/man-pages/man7/netlink.7.html
- man7 `rtnetlink(7)`: https://man7.org/linux/man-pages/man7/rtnetlink.7.html
- man7 `genetlink(7)`: https://man7.org/linux/man-pages/man7/genetlink.7.html

## RTNL lock / per-netns RTNL

RTNL は networking control path を長く大域的に serialize してきた。read/dump path の RCU 化、unlocked handlers、per-netns RTNL、subsystem-specific locking へ進むことで、巨大 host で多数 netns/device を並行構成する際の control-plane scalability を改善する。

per-netns RTNL は「RTNL がなくなった」という意味ではなく、**global serialization scope を namespace/resource scope へ縮める長期 migration** と理解する。

**Sources**
- LWN per-netns RTNL series: https://lwn.net/Articles/993037/
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## YNL

YNL (YAML Netlink) は Netlink family specification を machine-readable YAML で記述し、userspace bindings/tooling/documentation generation に利用する。従来の「header と C implementation を読んで protocol を推測」する方式から、schema を contract とする方向である。

**Sources**
- Kernel Netlink specifications: https://www.kernel.org/doc/html/latest/netlink/
- ethtool netlink: https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html

---

# 9. Netfilter / nftables / conntrack / flowtable

## nftables

nftables は ruleset を generic expression/VM model と Netlink API で表現し、sets/maps と transactional ruleset update を中心に設計された。iptables の table/chain/rule semantics を単に syntax 変更したものではなく、policy representation を再設計した framework である。

**Sources**
- Kernel Recipes 2013 “Nftables Why and how?”: https://archives.kernel-recipes.org/document/nftables-why-and-how/
- LWN Linux 3.13 coverage: https://lwn.net/Articles/573272/
- NetDev nftables ingress tutorial: https://www.netdevconf.info/1.1/tutorial-nftables-ingress-pablo-neira-ayuso.html

## conntrack

conntrack は flow の tuple、direction、state、timeout、NAT mapping、mark/label 等を保持する shared state substrate。nftables firewall/NAT だけでなく TC ct や flowtable 等もこの state を利用する。

「packet rule engine」と「flow state」を分離して理解すると、fast path/offload が state semantics を保ちながら execution path だけを変える構造が見える。

**Sources**
- Kernel Netfilter documentation: https://www.kernel.org/doc/html/latest/networking/
- LWN — topic-specific article: https://lwn.net/Articles/809333/

## Netfilter flowtable

flowtable は nftables policy により選ばれた established flow を classic forwarding path より短い software fast path へ入れる。miss/exception は通常 path に戻り、conntrack/NAT state は維持される。

これは firewall を迂回する無条件 shortcut ではなく、**policy-selected fast path** である。

**Sources**
- Kernel flowtable documentation: https://www.kernel.org/doc/html/latest/networking/nf_flowtable.html
- LWN flow offload infrastructure: https://lwn.net/Articles/738214/

## flowtable hardware offload

software flowtable entry を `flow_rule` 等へ変換し、driver callback 経由で NIC/switch hardware table に配置する。software `[OFFLOAD]` と hardware `[HW_OFFLOAD]` は execution placement が異なる。

TC Flower offload と Netfilter flowtable offload は共通の driver infrastructure を利用し得るが、policy/control-plane origin は別である。

**Sources**
- LWN hardware offload: https://lwn.net/Articles/804384/
- Kernel flowtable documentation: https://www.kernel.org/doc/html/latest/networking/nf_flowtable.html

---

# 10. Virtual networking / namespace / overlay

## network namespace / `setns()`

network namespace は netdev、routing table、firewall state、ports 等の networking resources を分離する。namespace file descriptor と `setns()` により namespace は process lifetime だけに結びつかず、明示的 object として参照・参加できる。

container networking の土台だが、namespace 自体は veth/bridge/overlay policy を提供するものではない。

**Sources**
- man7 `network_namespaces(7)`: https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- man2 `setns(2)`: https://man7.org/linux/man-pages/man2/setns.2.html

## veth / TUN/TAP / macvlan / ipvlan

veth は namespace 間を packet pair で接続する virtual Ethernet link。TUN/TAP は packet/frame を userspace fd に接続する。macvlan/ipvlan は parent netdev 上に複数 logical interface を構成するが L2/L3 identity model が異なる。

「virtual NIC」という一語でまとめず、**peer device / userspace endpoint / parent-device multiplexing** のどの model かを区別するとよい。

**Sources**
- man7 `veth(4)`: https://man7.org/linux/man-pages/man4/veth.4.html
- Kernel TUN/TAP: https://www.kernel.org/doc/html/latest/networking/tuntap.html
- Kernel networking index: https://www.kernel.org/doc/html/latest/networking/

## VXLAN / overlay

VXLAN は L2 frame を UDP/IP network 上へ encapsulate し、VNI で logical segment を識別する。Linux では bridge/FDB、neighbour、route、TC/switchdev offload 等と接続するため、単なる tunnel device より広い forwarding object graph の一部になる。

**Sources**
- Kernel VXLAN documentation: https://www.kernel.org/doc/html/latest/networking/vxlan.html
- Kernel networking index: https://www.kernel.org/doc/html/latest/networking/

---

# 11. Transport — TCP / UDP / MPTCP

## TCP Fast Open (TFO)

TFO は handshake と application data transmission の一部を重ね、connection establishment latency を減らす。cookie/state と replay/security trade-off を伴うため、単なる SYN optimization ではない。

**Sources**
- man7 `tcp(7)`: https://man7.org/linux/man-pages/man7/tcp.7.html
- LWN — topic-specific article: https://lwn.net/Articles/458610/

## DCTCP

DCTCP は ECN marking fraction を congestion signal として利用し、datacenter の shallow queue/low latency を狙う congestion control。loss event だけでなく **marking density** を feedback に使う点が重要である。

**Sources**
- man7 `tcp(7)`: https://man7.org/linux/man-pages/man7/tcp.7.html
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## BBR

BBR は loss を直接 congestion measure とする Reno/CUBIC 系とは異なり、bottleneck bandwidth と RTT を model 化して pacing/cwnd を制御する。したがって sch_fq/TCP pacing の歴史と合わせて理解する必要がある。

**Sources**
- LWN — BBR congestion control: https://lwn.net/Articles/701165/
- Kernel TCP docs: https://www.kernel.org/doc/html/latest/networking/

## PLB / AccECN

PLB (Protective Load Balancing) は congestion signal を利用して multipath fabric で flow の path rehash を促す host-side mechanism。AccECN は ECN feedback をより精密に伝える。両者とも「packet processing placement」ではなく **transport feedback semantics** の進化である。

**Sources**
- LWN — topic-specific article: https://lwn.net/Articles/909957/
- LWN — AccECN: https://lwn.net/Articles/1058666/
- Kernel TCP documentation: https://www.kernel.org/doc/html/latest/networking/

## MPTCP

MPTCP は1つの application-visible connection を複数 TCP subflow に分ける。meta-socket/connection level state と subflow state、path manager、scheduler が分離される。

重要なのは「TCPを複数本束ねる」だけでなく、address advertisement、subflow creation、fallback、data sequence mapping により **connection semantics を複数 path 上で維持する**点である。

**Sources**
- Kernel MPTCP documentation: https://www.kernel.org/doc/html/latest/networking/mptcp.html
- man7 TCP entry point: https://man7.org/linux/man-pages/man7/tcp.7.html

## TCP-AO

TCP Authentication Option は TCP segment authentication を提供し、key selection/lifetime 等を transport connection state と結びつける。IPsec/XFRM や WireGuard と異なり TCP semantics に直接統合された authentication mechanism である。

**Sources**
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/
- LWN — topic-specific article: https://lwn.net/Articles/957098/

## UDP GSO / GRO / zero-copy

UDP でも segmentation/aggregation により syscall/per-packet overhead を削減できる。UDP は TCP の retransmission/flow-control state を持たないため、application が batching と drop/backpressure をどう扱うかが重要になる。

**Sources**
- man7 `udp(7)`: https://man7.org/linux/man-pages/man7/udp.7.html
- Kernel segmentation offloads: https://www.kernel.org/doc/html/latest/networking/segmentation-offloads.html

---

# 12. Security — kTLS / XFRM / WireGuard

## kTLS

kTLS は TLS record encryption/decryption の一部を kernel socket path に移し、sendfile/zerocopy 等の kernel data path と TLS を組み合わせやすくする。TLS session/key negotiation は通常 userspace TLS library が担当し、kernel は record data path を担当する。

NIC TLS offload はさらに crypto execution を hardware へ移せるが、socket/TLS state と fallback semantics は kernel が管理する。

**Sources**
- Kernel TLS documentation: https://www.kernel.org/doc/html/latest/networking/tls.html
- LWN — topic-specific article: https://lwn.net/Articles/892216/

## XFRM / IPsec offload

XFRM は Security Association/policy と packet transformation を管理する Linux の IPsec framework。crypto offload では cryptographic operation、packet offload ではより広い IPsec packet processing を NIC へ配置できる。

ここでは **security state/policy ownership は kernel、execution は software/hardware から選択**という構造が明瞭である。

**Sources**
- Kernel XFRM device docs: https://www.kernel.org/doc/html/latest/networking/xfrm_device.html
- Kernel XFRM index: https://www.kernel.org/doc/html/latest/networking/index.html

## WireGuard

WireGuard は network interface として secure tunnel を提供し、peer/key/allowed-IP state と routing を接続する。XFRM policy/SA model とは異なる object model を採用し、通常の netdev として control/data path に統合される。

**Sources**
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/
- LWN — topic-specific article: https://lwn.net/Articles/816534/

---

# 13. RDMA — verbs / registered memory / offloaded execution

## QP / CQ / MR / PD

RDMA verbs の中心 object は Queue Pair (QP)、Completion Queue (CQ)、Memory Region (MR)、Protection Domain (PD) である。application は memory を MR として register し、QP に Work Request を post し、completion を CQ から回収する。

socket stack と比較すると、buffer registration、queue ownership、completion が application-visible object として強く露出する。

**Sources**
- man7 RDMA-CM: https://man7.org/linux/man-pages/man7/rdma_cm.7.html
- `ibv_create_qp`: https://man7.org/linux/man-pages/man3/ibv_create_qp.3.html
- `ibv_create_cq`: https://man7.org/linux/man-pages/man3/ibv_create_cq.3.html
- `ibv_reg_mr`: https://man7.org/linux/man-pages/man3/ibv_reg_mr.3.html
- Kernel RDMA driver API: https://www.kernel.org/doc/html/latest/driver-api/infiniband.html

## RDMA-CM / RoCEv2

RDMA-CM は address/route resolution と connection management を提供し、verbs data execution と control path を接続する。RoCEv2 は RDMA transport を UDP/IP network 上へ載せ、routing domain と HCA execution を接続する。

**Sources**
- man7 RDMA-CM: https://man7.org/linux/man-pages/man7/rdma_cm.7.html
- NetDev 0x19 — AI Networks / RoCEv2 and the role of netdev: https://netdevconf.info/0x19/sessions/workshop/ai-networks-rocev2-and-the-role-of-netdev.html
- Linux Plumbers archives/schedules: https://lpc.events/

## RXE / Soft-RoCE

RXE は RoCE/RDMA verbs semantics を software network stack 上で実装する。これにより verbs object model と hardware HCA implementation が概念上分離され、testing/development や software-defined RDMA が可能になる。

**Sources**
- Kernel RDMA documentation: https://www.kernel.org/doc/html/latest/driver-api/infiniband.html
- NetDev 0x19 — AI Networks / RoCEv2 and the role of netdev: https://netdevconf.info/0x19/sessions/workshop/ai-networks-rocev2-and-the-role-of-netdev.html

## ODP

On-Demand Paging (ODP) は MR registration 時に全 memory を長期 pin する model を緩和し、page fault/MMU notifier 等と連携して必要な page mapping を動的に扱う。registered-memory model に virtual-memory lifecycle を接続する機能である。

**Sources**
- `ibv_advise_mr`: https://man7.org/linux/man-pages/man3/ibv_advise_mr.3.html
- Kernel RDMA docs: https://www.kernel.org/doc/html/latest/driver-api/infiniband.html

## dma-buf MR / P2P

dma-buf MR は GPU 等が export した DMA-BUF を RDMA MR として register し、system RAM を経由しない device/peer-memory workflow を可能にする。P2P feasibility は PCIe topology/IOMMU/device capability に依存する。

**Sources**
- Kernel DMA-BUF: https://www.kernel.org/doc/html/latest/driver-api/dma-buf.html
- Kernel PCI P2P DMA: https://www.kernel.org/doc/html/latest/driver-api/pci/p2pdma.html
- LWN recent RDMA dma-buf example: https://lwn.net/Articles/1064824/

---

# 14. Driver framework

## switchdev

switchdev は Linux bridge/VLAN/FDB/FIB 等の kernel forwarding state を switch ASIC driver が offload する model。physical switch port を `struct net_device` として表し、Linux control plane を hardware programming の source of truth として利用する。

**Sources**
- Kernel switchdev documentation: https://www.kernel.org/doc/html/latest/networking/switchdev.html

## devlink

devlink は個々の netdev を越えた device-wide resource/control/health object を提供する。port、resource、parameter、health reporter、region、reload 等を扱い、PCI device/ASIC 全体の management plane を networking core に標準化する。

**Sources**
- Kernel devlink documentation: https://www.kernel.org/doc/html/latest/networking/devlink/
- netdevsim devlink example: https://www.kernel.org/doc/html/latest/networking/devlink/netdevsim.html

## phylink

phylink は MAC、PHY、PCS、SFP 等の link components の state coordination を共通化する。固定的な `net_device -> phy_device` だけでは表現しにくい topology で、link mode negotiation/state transition を framework に集約する。

**Sources**
- Kernel PHY link topology: https://www.kernel.org/doc/html/latest/networking/phy-link-topology.html
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## DIM

DIM (Dynamic Interrupt Moderation) は packet/byte/event rate を観測し、interrupt coalescing profile を動的に選択する policy framework。driver ごとに似た adaptive moderation algorithm を持つ代わりに、common library として policy を共有する。

**Sources**
- NetDev 0x12 — DIM / program: https://netdevconf.info/0x12/schedule.html
- Kernel networking documentation: https://www.kernel.org/doc/html/latest/networking/


## netdevsim

netdevsim は実 hardware を必要とせず networking control/offload API を test する software driver。devlink resource/health、TC offload、ethtool PHY 等の framework testbed として使われる。

**Sources**
- Kernel netdevsim/devlink: https://www.kernel.org/doc/html/latest/networking/devlink/netdevsim.html
- LWN netdevsim PHY example: https://lwn.net/Articles/1028195/

## auxiliary bus

auxiliary bus は1つの physical device/driver stack 内にある独立機能を auxiliary device/driver として分割する。networking では NIC core と RDMA 等の subfunction を分離しつつ parent resource/lifetime を共有する用途が代表的である。

**Sources**
- Kernel driver API index: https://www.kernel.org/doc/html/latest/driver-api/


## threaded NAPI

通常 NAPI poll は softirq context で実行される。threaded NAPI は NAPI instance の polling を kernel thread context で実行する選択肢を提供し、scheduler affinity/priority と execution context をより明示的に制御できる。

busy-poll 系との組み合わせでは「誰が poll loop を実行するか」が queue/NAPI 単位の execution-placement problem になる。

**Sources**
- Kernel NAPI documentation: https://www.kernel.org/doc/html/latest/networking/napi.html
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## ethtool netlink

legacy ioctl の ethtool control plane を Generic Netlink family に移し、bitset、extended ACK、dump/notification、structured attributes を利用可能にした。link modes、rings、channels、coalesce、FEC、RSS 等が machine-readable API になる。

**Sources**
- Kernel ethtool netlink: https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html
- Kernel interface statistics: https://www.kernel.org/doc/html/latest/networking/statistics.html

## netdev Generic Netlink / queue objects

netdev Generic Netlink family は NAPI、queue、page_pool、statistics、XDP/AF_XDP capability 等を明示的 object として userspace に公開する。近年は queue lease、DMA-BUF、io_uring memory-provider information も queue object に結びつく。

これは「driver の内部 queue number」から **control-plane-visible resource identity** への変化である。

**Sources**
- Kernel netdev Netlink spec: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## Rust PHY abstractions

Rust-for-Linux の networking 導入では PHY driver abstraction が初期の主要対象となった。重要なのは言語変更だけでなく、kernel framework が lifetime/ownership/error contract を Rust type system に表現できる API boundary を持つ必要がある点である。

**Sources**
- Kernel Rust documentation: https://www.kernel.org/doc/html/latest/rust/
- LWN — Rust abstractions for network PHY drivers: https://lwn.net/Articles/948895/

---

# 15. Observability

## counter / statistics

counter は低 overhead で aggregate symptom を示すが、「どの packet がどこで、なぜ drop されたか」の causality は持たない。`rtnl_link_stats64`、ethtool stats、per-queue stats、protocol stats 等は scope と reset/lifetime semantics を確認して読む必要がある。

**Sources**
- Kernel interface statistics: https://www.kernel.org/doc/html/latest/networking/statistics.html

## tracepoint / kprobe / fentry

tracepoint は比較的 stable な named event、kprobe はほぼ任意 kernel function への dynamic probe、BPF fentry/fexit は BTF を利用した function tracing を提供する。stable contract と observability coverage の trade-off が異なる。

**Sources**
- Kernel BPF docs: https://www.kernel.org/doc/html/latest/bpf/
- Kernel BTF: https://www.kernel.org/doc/html/latest/bpf/btf.html

## `skb_drop_reason`

structured drop reason は drop call-site で reason enum を伝播し、単なる drop counter より「なぜ」を machine-readable にする。すべての drop が完全に reason 化されるわけではないため、counter/tracepoint/call-site tracing と併用する。

**Sources**
- LWN — `skb` drop reasons in `net/core/dev.c`: https://lwn.net/Articles/886798/
- LWN — TCP state-path drop reasons: https://lwn.net/Articles/913104/
- Kernel networking docs: https://www.kernel.org/doc/html/latest/networking/

## timestamping

software/hardware timestamp、RX/TX timestamp、packet lifecycle event を組み合わせると latency を区間分解できる。timestamp の clock domain と capture point を区別しないと誤解しやすい。

**Sources**
- Kernel timestamping documentation: https://www.kernel.org/doc/html/latest/networking/timestamping.html
- man7 socket: https://man7.org/linux/man-pages/man7/socket.7.html

## page_pool / queue / NAPI observability

netdev Generic Netlink により page_pool ID、inflight references/memory、NAPI ID、queue statistics 等を userspace から関連付けられる。これは packet-memory lifetime や queue ownership を debugging する際に、driver-private debugfs/statistics だけに依存しない方向である。

**Sources**
- netdev Netlink spec: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html
- interface statistics: https://www.kernel.org/doc/html/latest/networking/statistics.html

## pwru

pwru (“packet, where are you?”) は eBPF/kprobe 等を利用して packet が kernel network stack のどの function を通るかを追跡する debugger。tcpdump が wire/interface observation を得意とするのに対し、pwru は kernel call path の可視化を狙う。

**Sources**
- Linux Plumbers Conference 2024: https://lpc.events/event/18/contributions/1942/
- LPC 2024 schedule: https://lpc.events/event/18/timetable/?view=lpc

## Retis

Retis は kernel networking tracing を higher-level event/collector model で扱い、skb tracking、drop、OVS/netfilter 等の複数 subsystem の event を相関することを狙う。pwru と同じく BPF/BTF ecosystem を利用するが、raw function trace より structured analysis pipeline を重視する。

**Sources**
- Linux Plumbers Conference schedules/index: https://lpc.events/
- NetDev 0x19 — Retis tutorial: https://netdevconf.info/0x19/sessions/tutorial/challenges-and-limitations-of-debugging-increasingly-complex-virtualized-networks-with-retis.html

---

# 16. Cross-topic concepts

## execution placement

execution placement は独立した「第7の contract」ではなく、object/API/assignment/lifetime/synchronization contract の結果として **処理をどこで実行するか**を選べることを指す。

例:
- XDP: driver RX 前段
- CPUMAP: remote CPU
- AF_XDP: userspace
- switchdev/TC/flowtable: NIC/switch hardware
- kTLS/XFRM: CPU vs crypto/NIC offload
- RDMA: HCA queue execution
- threaded NAPI: softirq vs kernel thread

## memory ownership / lifetime

zero-copy、高速 RX、device memory では「copy 回数」より **誰が buffer を所有し、いつ再利用できるか**が本質になる。AF_XDP UMEM completion、MSG_ZEROCOPY completion、page_pool recycling、RDMA CQ/MR、io_uring CQE はそれぞれ異なる API だが、この lifetime 問題に答える。

## synchronization scope

global lock から per-object/per-netns lock、RCU reader、refcount/lifetime management へ scope を縮めることで control-plane concurrency を上げる。RTNL は代表例だが、driver framework や object lifecycle にも同じ問題が現れる。

## control plane object

route/nexthop、qdisc/classifier/action、nftables ruleset/flowtable、devlink resource、netdev queue、RDMA QP/MR のように resource/state を explicit object として扱うと、更新・参照・offload・observability の境界を定義しやすい。

---

# 18. Source index

## Linux man-pages

- Main index: https://man7.org/linux/man-pages/
- `bpf(2)`: https://man7.org/linux/man-pages/man2/bpf.2.html
- `netlink(7)`: https://man7.org/linux/man-pages/man7/netlink.7.html
- `rtnetlink(7)`: https://man7.org/linux/man-pages/man7/rtnetlink.7.html
- `network_namespaces(7)`: https://man7.org/linux/man-pages/man7/network_namespaces.7.html
- `setns(2)`: https://man7.org/linux/man-pages/man2/setns.2.html
- `tcp(7)`: https://man7.org/linux/man-pages/man7/tcp.7.html
- `udp(7)`: https://man7.org/linux/man-pages/man7/udp.7.html
- RDMA-CM: https://man7.org/linux/man-pages/man7/rdma_cm.7.html

## Kernel documentation

- Version selector / index: https://www.kernel.org/doc/html/
- Latest networking: https://www.kernel.org/doc/html/latest/networking/
- Latest BPF: https://www.kernel.org/doc/html/latest/bpf/
- Latest Netlink specs: https://www.kernel.org/doc/html/latest/netlink/
- Latest RDMA driver API: https://www.kernel.org/doc/html/latest/driver-api/infiniband.html

## LWN

- BQL: https://lwn.net/Articles/469652/
- TCP Fast Open / LPC: https://lwn.net/Articles/458610/
- BBR: https://lwn.net/Articles/701165/
- BTF: https://lwn.net/Articles/752047/
- MPTCP upstreaming / LPC: https://lwn.net/Articles/800501/
- Netfilter hardware offload: https://lwn.net/Articles/809333/
- BIG TCP: https://lwn.net/Articles/884104/
- io_uring zero-copy TX: https://lwn.net/Articles/879724/
- io_uring zero-copy RX series: https://lwn.net/Articles/1004591/
- Rust PHY abstractions: https://lwn.net/Articles/948895/
- AccECN: https://lwn.net/Articles/1058666/
- MPTCP path-manager limit expansion: https://lwn.net/Articles/1072108/
- nftables 3.13: https://lwn.net/Articles/573272/
- Netfilter flow offload: https://lwn.net/Articles/738214/
- Netfilter hardware flowtable offload: https://lwn.net/Articles/804384/
- per-netns RTNL series: https://lwn.net/Articles/993037/

## NetDev conference

- 0.1 (2015) BPF In-kernel Virtual Machine / switchdev material: https://netdevconf.info/0.1/sessions.html
- 1.1 (2016) nftables ingress: https://www.netdevconf.info/1.1/tutorial-nftables-ingress-pablo-neira-ayuso.html
- 1.1 (2016) bridge filtering with nftables: https://www.netdevconf.info/1.1/talk-bridge-filtering-nftables-florian-westphal.html
- 1.2 (2016) XDP workshop / cls_bpf / XDP HW offload program: https://netdevconf.info/1.2/schedule.html
- 0x12 (2018) DIM / UDP GSO / MPTCP program: https://netdevconf.info/0x12/schedule.html
- 0x15 (2021) BIG TCP / XDP / TC / Netfilter program: https://netdevconf.info/0x15/accepted-sessions.html
- 0x17 (2023) Device Memory TCP: https://netdevconf.info/0x17/sessions/talk/device-memory-tcp.html
- 0x17 (2023) io_uring zero-copy RX: https://netdevconf.info/0x17/sessions/talk/fast-zc-rx-data-plane-using-io-uring.html
- 0x17 (2023) netdev object Netlink APIs (program): https://netdevconf.info/0x17/pages/sessions.html
- 0x18 (2024) devmem TCP + io_uring ZC / netmem BoF: https://netdevconf.info/0x18/sessions/bof/devmem-tcp-io-uring-zero-copy.html
- 0x19 (2025) page_pool leak diagnosis: https://www.netdevconf.info/0x19/sessions/tutorial/diagnosing-page-pool-leaks.html
- 0x19 (2025) Retis tutorial: https://netdevconf.info/0x19/sessions/tutorial/challenges-and-limitations-of-debugging-increasingly-complex-virtualized-networks-with-retis.html
- 0x19 (2025) AI Networks / RoCEv2: https://netdevconf.info/0x19/sessions/workshop/ai-networks-rocev2-and-the-role-of-netdev.html
- Device Memory TCP (0x17): https://netdevconf.info/0x17/sessions/talk/device-memory-tcp.html
- 0x17 sessions: https://netdevconf.info/0x17/pages/sessions.html
- AI Networks / Device Memory TCP (0x18): https://netdevconf.info/0x18/sessions/talk/the-future-of-ai-networks-advancing-tcp-with-device-memory-and-collective-communication.html
- nftables ingress (1.1): https://www.netdevconf.info/1.1/tutorial-nftables-ingress-pablo-neira-ayuso.html

## Kernel Recipes

- Document library: https://archives.kernel-recipes.org/document-library/
- XDP programmable network layer: https://archives.kernel-recipes.org/document/xdp-a-new-programmable-network-layer/
- XDP closer integration: https://archives.kernel-recipes.org/document/xdp-closer-integration-with-network-stack/
- nftables — Why and how?: https://archives.kernel-recipes.org/document/nftables-why-and-how/

## Linux Plumbers Conference

- 2025: https://lpc.events/event/19/ — https://lpc.events/event/19/timetable/?view=lpc
- 2024: https://lpc.events/event/18/ — https://lpc.events/event/18/timetable/?view=lpc
- 2023: https://lpc.events/event/17/
- 2022: https://lpc.events/event/16/
- 2021: https://lpc.events/event/11/
- 2020: https://lpc.events/event/7/
- 2019: https://lpc.events/event/4/
- 2018: https://lpc.events/event/2/
- pwru at LPC 2024: https://lpc.events/event/18/contributions/1942/

---

# 19. v2で追加した実装レベルの読み方

v2では、下記候補のうち **主要 object/function/UAPI、data-path図、調査コマンド、failure mode、r69 Part VI/VIIへの逆引き方針** を後続章として実装した。

## 将来さらに追加できる項目

この v1 は r69 の主要な architecture topic を一通り引けることを優先した。次の版では、各項目に以下を追加すると「用語集」から「実装リファレンス」へ発展できる。

1. **主要 `struct` / function / UAPI attribute** — 例: `struct xdp_buff`, `struct page_pool`, `struct nf_conn`, `struct nexthop`, `struct napi_struct`.
2. **data-path diagram** — RX/TX の function-level path と ownership transition。
3. **release-by-release evolution** — r69 Part VI/VII の milestone を各用語へ逆引き。
4. **debugging commands** — `ip`, `tc`, `nft`, `bpftool`, `ethtool`, `ss`, `devlink`, `rdma`, tracefs/BPF tools。
5. **failure modes / pitfalls** — queue buildup、incorrect NUMA/CPU placement、zero-copy lifetime、offload fallback、RTNL contention、provider mismatch 等。
6. **conference source expansion** — NetDev / Kernel Recipes / LPC の関連発表を各用語に複数紐付ける。


---

# 20. 実装リファレンス — object / function / UAPI / command

この章は「名前を知る」ための用語集から一段下がり、**どの kernel object・callback・UAPI を見ると実装を追えるか**をまとめる。kernel internal symbol は release により変わり得るため、ここでは安定した概念上の入口を優先する。

## 19.1 RX queue の end-to-end object map

```text
NIC RX queue / descriptor ring
        │ DMA
        ▼
driver RX buffer
        │
        ├─ page_pool / netmem
        │      └─ backing: system page / provider memory / device memory
        │
        ▼
NAPI instance
        │
        ├─ XDP program
        │    ├─ PASS ───────────────► skb ─► protocol stack
        │    ├─ REDIRECT ─► DEVMAP / CPUMAP / XSKMAP
        │    ├─ TX
        │    └─ DROP
        │
        └─ normal RX path ──────────► skb
```

**主要 object / field**

| Layer | 代表 object / API | 見るポイント |
|---|---|---|
| netdev | `struct net_device` | device identity、queue 数、`net_device_ops` |
| RX queue | `struct netdev_rx_queue` | queue identity と XDP RXQ 情報 |
| NAPI | `struct napi_struct` | poll instance、weight、threaded state、NAPI ID |
| packet memory | `struct page_pool` / `netmem_ref` | allocation、recycle、DMA mapping、provider |
| XDP | `struct xdp_buff`, `struct xdp_frame` | RX buffer view と redirect/TX 用 frame |
| skb | `struct sk_buff` | protocol-stack packet representation |
| AF_XDP | `struct xdp_desc`, UMEM rings | userspace/kernel 間の frame ownership |

page_pool は RX queue/NAPI locality を利用した recycling を前提とし、最新 documentation は `page_pool_params.napi` を設定した場合、その NAPI が pool の consumer-side resource を単独で扱えることを明示している。これは「page_pool は単なる高速 allocator」という理解より、**queue/NAPI に結び付いた lifetime object** と見る方が正確である。

**Source**
- https://www.kernel.org/doc/html/latest/networking/page_pool.html
- https://www.kernel.org/doc/html/latest/networking/napi.html
- https://www.kernel.org/doc/html/latest/networking/af_xdp.html

## 19.2 TX queue / qdisc / BQL

```text
socket / protocol
       │
       ▼
   skb enqueue
       │
       ▼
 qdisc / pacing
       │ dequeue
       ▼
dev_queue_xmit()
       │
       ▼
driver ndo_start_xmit()
       │
       ├─ BQL/DQL accounting
       ▼
NIC TX descriptor ring
       │
       ▼
wire
```

**主要入口**

- `struct Qdisc` — scheduler state。
- `struct netdev_queue` — netdev TX queue。
- `netdev_tx_sent_queue()` / `netdev_tx_completed_queue()` — BQL accounting の代表的 driver-side interface。
- `ndo_start_xmit` — network driver TX callback。
- `skb->tstamp` — time-based scheduling/pacing と関係する timestamp。

調査では「qdisc backlog が小さいのに latency が大きい」場合、qdisc だけでなく driver/NIC ring の outstanding bytes を見る。BQL の意義はこの downstream backlog を制限して qdisc の policy を実効的にする点にある。

**Source**
- https://www.kernel.org/doc/html/latest/networking/
- https://man7.org/linux/man-pages/

## 19.3 XDP / AF_XDP object model

```text
XDP program
   │ XDP_REDIRECT
   ▼
BPF map
 ├─ DEVMAP  ─► another netdev
 ├─ CPUMAP  ─► remote CPU processing
 └─ XSKMAP  ─► AF_XDP socket
                  │
                  ▼
               RX ring
                  │
                UMEM
             ▲         │
        FILL ring   TX ring
             │         ▼
             └── COMPLETION ring
```

AF_XDP の4 ring は同じ役割ではない。

| Ring | Producer → Consumer | Ownership の意味 |
|---|---|---|
| FILL | userspace → kernel | RX に使ってよい UMEM frame を渡す |
| RX | kernel → userspace | packet が入った frame descriptor を渡す |
| TX | userspace → kernel | 送信対象 frame を渡す |
| COMPLETION | kernel → userspace | kernel が frame の利用を終え、再利用可能になったことを返す |

COMPLETION は **frame ownership の返却**であり、それ単独では packet が wire 上で正常送信された保証ではない。

**Source**
- https://www.kernel.org/doc/html/latest/networking/af_xdp.html
- https://www.kernel.org/doc/html/latest/bpf/map_devmap.html
- https://www.kernel.org/doc/html/latest/bpf/map_cpumap.html
- https://www.kernel.org/doc/html/latest/bpf/map_xskmap.html

## 19.4 page_pool → netmem → memory provider

```text
                 page_pool lifecycle
              alloc / recycle / DMA sync
                       │
                       ▼
                    netmem
          ┌────────────┼────────────┐
          │            │            │
     system page   DMA-BUF      provider memory
          │            │            │
          └────────────┴────────────┘
                       │
               RX payload backing
```

実装を読むときは3層を混同しない。

1. **page_pool** — RX memory の allocation/recycle/DMA lifecycle。
2. **netmem** — backing memory type を `struct page` から抽象化する packet-memory reference。
3. **memory provider** — page_pool lifecycle に backing/provider choice を接続する。

netmem 対応 driver は page_pool を使い、payload memory を netmem API で扱い、CPU から読めない netmem を許容する必要がある。driver-private `struct page *` recycling を残すと abstraction を破る。

**Source**
- https://www.kernel.org/doc/html/latest/networking/page_pool.html
- https://www.kernel.org/doc/html/latest/networking/netmem.html
- https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 19.5 io_uring ZCRX と memory provider の接続

```text
userspace registered buffers
          │
          ▼
 io_uring memory provider
          │
          ▼
       page_pool
          │
          ▼
     NIC RX queue
          │
          ▼
 completion / CQE
          │
          ▼
      userspace
```

ここでの核心は「recv syscall を io_uring にした」ことではない。RX queue が使用する memory provider と userspace buffer registration、completion を一つの lifetime model に接続した点にある。最新 netdev Generic Netlink specification には `io-uring-provider-info` と page-pool の `io-uring` provider information が公開されている。

**Source**
- https://www.kernel.org/doc/html/latest/networking/
- https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 19.6 Netfilter object map

```text
nftables ruleset
      │
      ├─ expression / set / map
      │
      ▼
 policy decision
      │
      ├──────────────► classic forwarding path
      │
      ▼
 conntrack state
      │
      ▼
 flowtable entry
      │
      ├─ software fast path
      └─ flow_rule ─► driver ─► NIC / switch HW
```

**主要 object/concept**

- nftables table / chain / rule / expression。
- set / map — lookup state。
- `struct nf_conn` — conntrack flow state。
- flowtable / flow offload tuple — accelerated forwarding state。
- `flow_rule` — driver/offload 側へ渡す match/action representation。

調査では「nft rule があるか」だけでなく、対象 packet が classic hook path、flowtable software path、hardware-offloaded path のどこで実行されているかを区別する。

**Source**
- https://www.kernel.org/doc/html/latest/networking/nf_flowtable.html
- https://archives.kernel-recipes.org/document/nftables-why-and-how/
- https://lwn.net/Articles/804384/

## 19.7 Routing / nexthop / TC object map

```text
userspace
 ip / tc / routing daemon
        │ Netlink
        ▼
  FIB / nexthop objects
        │
        ├─ software forwarding
        └─ switchdev / driver offload

TC:
 classifier (Flower/BPF)
        │
        ▼
     actions
 ct / mirred / police / tunnel ...
        │
        ▼
    flow_rule
        │
        ▼
 driver / hardware
```

**主要 userspace commands**

```bash
ip -details route show
ip nexthop show
ip -details link show
tc -s qdisc show dev DEV
tc -s filter show dev DEV ingress
bridge -d vlan show
bridge fdb show
```

`ip route` と `ip nexthop` を分けて観察することで、route entry と reusable forwarding object の lifecycle を区別できる。

**Source**
- https://man7.org/linux/man-pages/man7/rtnetlink.7.html
- https://www.kernel.org/doc/html/latest/networking/switchdev.html

## 19.8 RTNL / Netlink debugging map

```text
userspace request
      │
      ▼
Netlink family / command
      │
      ├─ attribute validation
      ├─ dump / doit handler
      └─ notification
      │
      ▼
network object update
      │
      ├─ RTNL
      ├─ per-netns RTNL
      ├─ RCU
      └─ subsystem-specific lock/refcount
```

**確認コマンド**

```bash
# rtnetlink object を詳細表示
ip -details -statistics link show

# Generic Netlink family
genl ctrl list

# ethtool Netlink-backed state
ethtool --show-features DEV
ethtool -g DEV
ethtool -l DEV
ethtool -c DEV
```

YNL を調べる場合は kernel tree の `Documentation/netlink/specs/*.yaml` と generated documentation を対応付ける。

**Source**
- https://man7.org/linux/man-pages/man7/netlink.7.html
- https://man7.org/linux/man-pages/man7/rtnetlink.7.html
- https://www.kernel.org/doc/html/latest/netlink/
- https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html

## 19.9 Driver framework object map

```text
                       net_device
                           │
          ┌────────────────┼────────────────┐
          │                │                │
       phylink         switchdev        ethtool
 MAC/PCS/PHY/SFP    FDB/VLAN/FIB      link/ring/RSS
          │                │                │
          └────────────────┼────────────────┘
                           │
                        driver
                           │
                  ┌────────┴────────┐
                  │                 │
               devlink         auxiliary bus
       device-wide mgmt     subfunction split
```

この図で `net_device` は packet I/O interface、devlink は device-wide management、phylink は link component coordination、switchdev は forwarding-object offload を担当する。似た「driver framework」でも責務が違う。

**Source**
- https://www.kernel.org/doc/html/latest/networking/devlink/
- https://www.kernel.org/doc/html/latest/networking/switchdev.html
- https://www.kernel.org/doc/html/latest/networking/phy-link-topology.html
- https://www.kernel.org/doc/html/latest/networking/ethtool-netlink.html

## 19.10 RDMA object / lifetime map

```text
PD
├─ MR ── registered/pinned/on-demand/device memory
│
└─ QP
   ├─ Send Queue
   └─ Receive Queue
          │
          ▼
         HCA
          │
          ▼
         CQ
          │ completion
          ▼
     application
```

**代表 API**

```text
ibv_alloc_pd()
ibv_reg_mr()
ibv_create_qp()
ibv_post_send()
ibv_post_recv()
ibv_create_cq()
ibv_poll_cq()
```

RDMA は buffer registration、queue submission、completion が application-visible であるため、Linux networking の memory ownership / queue placement の比較対象として有用である。

**Source**
- https://man7.org/linux/man-pages/man3/ibv_reg_mr.3.html
- https://man7.org/linux/man-pages/man3/ibv_create_qp.3.html
- https://man7.org/linux/man-pages/man3/ibv_create_cq.3.html
- https://man7.org/linux/man-pages/man3/ibv_post_send.3.html
- https://www.kernel.org/doc/html/latest/driver-api/infiniband.html

---

# 21. 調査コマンド集 — 「どこを見るか」から引く

## 20.1 Queue / qdisc / NIC

```bash
tc -s qdisc show dev DEV
ip -s -s link show dev DEV
ethtool -S DEV
ethtool -g DEV
ethtool -l DEV
ethtool -c DEV
ethtool -k DEV
```

見る順序は **qdisc backlog → netdev stats → per-queue/driver stats → ring/channel/coalesce** が分かりやすい。software queue が空でも NIC ring に backlog があれば latency は残る。

## 20.2 CPU / IRQ / NAPI placement

```bash
grep -i DEV /proc/interrupts
cat /proc/irq/IRQ/smp_affinity_list
ethtool -x DEV
grep . /sys/class/net/DEV/queues/rx-*/rps_cpus
grep . /sys/class/net/DEV/queues/tx-*/xps_cpus
```

RSS、IRQ affinity、RPS/RFS、XPS は別レイヤーなので、同じ「CPU affinity」として一括りにしない。

## 20.3 BPF / XDP

```bash
bpftool prog show
bpftool map show
bpftool link show
bpftool net show
ip -details link show dev DEV
```

TCX/BPF link を調べるときは program だけでなく **link object** の lifetime を見る。

## 20.4 AF_XDP

```bash
ip -details link show dev DEV
ethtool -l DEV
ethtool -x DEV
bpftool map show
bpftool prog show
```

AF_XDP の問題は XDP program、XSKMAP、queue id、UMEM、driver zero-copy capability のどこでも起こり得る。

## 20.5 page_pool / NAPI / queue objects

kernel/iproute2 の対応状況に応じて netdev Generic Netlink の queue/NAPI/page-pool dump を利用する。重要なのは `ifindex`、queue id、NAPI ID、page-pool ID、`inflight`、`inflight-mem`、provider 情報を相関することである。

**Source**
- https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 20.6 Routing / neighbour / nexthop

```bash
ip -details route show
ip rule show
ip nexthop show
ip neigh show
ip -details link show
```

route が正しくても neighbour resolution や nexthop object が期待と違えば forwarding は成立しない。

## 20.7 TC / offload

```bash
tc -s qdisc show dev DEV
tc -s filter show dev DEV ingress
tc -s filter show dev DEV egress
devlink dev show
devlink port show
devlink resource show
```

`in_hw` / `not_in_hw` 等の表示が利用できる場合は、rule の存在と execution placement を分けて確認する。

## 20.8 nftables / conntrack

```bash
nft list ruleset
conntrack -L
nft list flowtables
```

flowtable/offload の問題では「rule hit」だけでなく conntrack state と flowtable entry の lifecycle を確認する。

## 20.9 Socket / TCP

```bash
ss -tinm
ss -uapnm
nstat
ip -s link
```

`ss -tinm` では congestion control、RTT、cwnd、pacing rate、socket memory 等を同時に観察できる環境がある。

## 20.10 devlink / health

```bash
devlink dev show
devlink port show
devlink param show
devlink resource show
devlink health show
devlink region show
```

netdev が正常でも device-wide firmware/resource/health problem がある場合があるため、devlink は `ip link` の代替ではなく別 scope の observability である。

## 20.11 RDMA

```bash
rdma link show
rdma dev show
rdma resource show
rdma statistic show
```

verbs application 側では CQ completion、QP state、MR registration/lifetime と Linux netdev/RDMA-CM state を分けて追う。

---

# 22. Failure-mode glossary — 症状から architecture を逆引きする

## 21.1 latency が高いが qdisc backlog は小さい

候補:

- NIC TX ring に backlog。
- BQL が未使用/不適切。
- interrupt moderation/coalescing。
- CPU/NUMA placement。
- hardware offload queue。

次に見る: `ethtool -S`, `-g`, `-c`, IRQ affinity。

## 21.2 XDP program は動くが AF_XDP に packet が来ない

確認順:

1. XDP program が期待 interface/attach mode に存在するか。
2. XSKMAP entry が存在するか。
3. XSK が同じ netdev/queue id に bind されているか。
4. FILL ring に frame を供給しているか。
5. driver の XDP/AF_XDP zero-copy capability。
6. multi-buffer packet と application capability。

## 21.3 page_pool が消えない

page_pool は detach 後も outstanding memory が返るまで生存し得る。netdev Generic Netlink の `inflight`, `inflight-mem`, `detach-time` を見ると、「driver は detach 済みだが skb/socket/queue/userspace が reference を保持している」状態を区別できる。

**Source**
- https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 21.4 zero-copy が期待ほど速くない

zero-copy は copy cost を lifetime/accounting/completion cost と交換する。small I/O では pin/register/completion overhead が勝つことがある。NUMA locality、batch size、buffer reuse、NIC DMA mapping、completion frequency を確認する。

## 21.5 TC/nft rule は存在するのに packet path が期待と違う

rule representation と execution placement を分ける。

- classic software path
- XDP/TC BPF path
- Netfilter flowtable
- TC hardware offload
- Netfilter hardware flowtable
- switchdev bridge/FDB/FIB offload

同じ policy intent でも観測点が変わる。

## 21.6 RTNL contention

多数 netns/netdev を並列更新する workload では、global RTNL、per-netns RTNL、subsystem lock のどこで serialize されるかを kernel version と対象 operation ごとに確認する。per-netns RTNL は一括移行ではない。

## 21.7 Device Memory TCP で driver が落ちる/データを読めない

netmem は CPU-unreadable backing を許容する。driver が `struct page *` や direct virtual address を暗黙に仮定していないか、page_pool/netmem helper を経由しているかを確認する。

**Source**
- https://www.kernel.org/doc/html/latest/networking/netmem.html

---

# 23. r69 Part VI / VII への逆引き方針

本用語集では release/commit の正本を複製しない。詳細を調べるときは次の順で読む。

```text
用語集の mechanism / object
          │
          ▼
r69 Part III / IV / V
architecture lineage / driver contract / evidence
          │
          ▼
r69 Part VI
canonical release chronology
          │
          ▼
r69 Part VII
exact mainline anchor / provenance
```

この分離により、用語集側で release attribution を更新するたびに chronology と不整合になることを避ける。特に最近の 7.x 系、queue leasing、Device Memory TCP、io_uring ZCRX、RTNL 分割のように複数 release にまたがる系列は Part VI/VII を正本とする。

---

# 24. 実装を読むための代表 struct / concept 索引

| Topic | 最初に探す symbol / object | 次に見るもの |
|---|---|---|
| netdev | `struct net_device` | `net_device_ops`, queues |
| NAPI | `struct napi_struct` | poll callback, NAPI ID |
| skb | `struct sk_buff` | head/data/frags, metadata |
| XDP | `struct xdp_buff` | `xdp_frame`, RXQ info |
| AF_XDP | `struct xdp_desc` | UMEM, XSKMAP, four rings |
| page_pool | `struct page_pool` | params, inflight, DMA sync |
| netmem | `netmem_ref` | backing/provider helpers |
| BPF | `struct bpf_prog` | map, link, verifier state |
| qdisc | `struct Qdisc` | enqueue/dequeue, watchdog |
| route | FIB objects | nexthop, rules, neighbour |
| TC | Flower / actions | `flow_rule`, driver callbacks |
| conntrack | `struct nf_conn` | tuple, timeout, NAT state |
| nftables | ruleset objects | expression, set/map |
| NAPI/queue netlink | netdev family | queue/NAPI/page-pool IDs |
| devlink | devlink instance | port/resource/health/region |
| phylink | phylink instance | MAC/PCS/PHY state |
| RDMA | QP/CQ/MR/PD | work request/completion |
| io_uring | SQ/CQ + registered buffers | CQE, provider lifetime |

---

# 25. Source-use policy for this glossary

本用語集では source を次の役割で使い分ける。

- **Linux man-pages** — userspace-visible syscall/socket/RDMA API semantics。
- **kernel.org documentation** — current object model、driver API、UAPI/Netlink specification。
- **LWN** — merge history、design discussion、review context。
- **NetDev** — netdev subsystem の design motivation、performance data、work-in-progress architecture。
- **Kernel Recipes** — subsystem author/maintainer による architecture explanation。
- **Linux Plumbers Conference** — cross-subsystem design discussion、prototype、将来方向。

`latest` kernel documentation は current design の説明に使う。特定 release の semantics を述べる場合は `https://www.kernel.org/doc/html/vX.Y/` の versioned documentation と r69 Part VI/VII を優先する。


# 26. Linux Plumbers Conference — 年度別プログラム監査（2008–2025）

**監査区分:** 「個別発表を確認」は発表タイトル・概要が確認できたもの。「トラック確認」はプログラム/CFPまで確認できたもの。「未検証」は今回の監査で個別発表を確認できていないもの。古い年度の全セッションを網羅したという意味ではない。発表は設計議論の証拠であり、mainline release の証拠にはしない。

| 年 | 確認結果 | r69への適用 | 出典 |
|---|---|---|---|
| 2008 | 未検証 | 初期networking議論の追加調査候補 | https://blog.linuxplumbersconf.org/2008/program/schedule/index.html |
| 2009 | 未検証 | 同上 | https://blog.linuxplumbersconf.org/2009/program/index.html |
| 2010 | 未検証 | 同上 | https://blog.linuxplumbersconf.org/2010/program/index.html |
| 2011 | 未検証 | queueing、virtualization前史の候補 | https://blog.linuxplumbersconf.org/2011/ocw/events/LPC2011/schedule |
| 2012 | 未検証 | netns/virtualization、BPF前史の候補 | https://blog.linuxplumbersconf.org/2012/resources-slides-and-videos/index.html |
| 2013 | **個別発表を確認** | OVS hardware-assisted filtering、NIC embedded switches、DPDK、network virtualization | https://blog.linuxplumbersconf.org/2013/ocw/events/LPC2013/proposals/ |
| 2014 | 未検証 | switchdev、BPF前史の候補 | https://blog.linuxplumbersconf.org/2014/ocw/events/LPC2014/schedule |
| 2015 | 未検証 | BPF/XDP前史、driver offload候補 | https://blog.linuxplumbersconf.org/2015/ocw/events/LPC2015/schedule/ |
| 2016 | 未検証 | RDMA、BPF/tracing候補 | https://blog.linuxplumbersconf.org/2016/ocw/events/LPC2016/schedule.html |
| 2017 | **個別発表を確認** | RDMA kABI、paravirtual RDMA、Tracing/BPF、container networking | https://blog.linuxplumbersconf.org/2017/ocw/events/LPC2017/schedule.html |
| 2018 | プログラム入口 | BPF/Networkingの追加確認候補 | https://lpc.events/event/2/ |
| 2019 | プログラム入口 | networking/BPFの追加確認候補 | https://lpc.events/event/4/ |
| 2020 | プログラム入口 | networking/BPFの追加確認候補 | https://lpc.events/event/7/ |
| 2021 | **個別発表を確認** | BPF subsystem audit / program signing、networking/BPF summit | https://lpc.events/event/11/contributions/947/ |
| 2022 | **個別発表を確認** | 400Gbps級IP/TCP stack scalability、FIB lookupとpacket-rate budget | https://lpc.events/event/16/contributions/1345/ |
| 2023 | プログラム/トピック一覧を確認 | networking/BPF、driver/observability関連発表の継続調査 | https://lpc.events/event/17/contributions/ |
| 2024 | **個別発表を確認** | UDP GSO、BPF SK_LOOKUP、TCP timestampなどの実運用上の課題 | https://lpc.events/event/18/contributions/1962/ |
| 2025 | **Networking Trackを確認** | netdev maintainer updates、networking proposals、control/data-path進化 | https://lpc.events/event/19/sessions/236/ |

## 個別に採用する発表

### LPC 2013 — Network Virtualization I / II

- **Open vSwitch: Hardware assisted filtering** — OVS software policyとNIC offloadの接続。r69のswitchdev / TC offloadの歴史的背景として有用。
- **Integrating NICs with embedded switches in a Linux ecosystem** — NIC embedded switchをLinux control planeで扱う課題。switchdev/devlink系列の前史。
- **Intel DPDK: Data Plane Development Kit** — kernel networkingとuserspace datapathを比較する資料。
- 出典（発表タイトルと概要）: https://blog.linuxplumbersconf.org/2013/ocw/events/LPC2013/proposals/

### LPC 2017 — RDMA / Tracing-BPF

- **Paravirtual RDMA device** — QEMU/virtio RDMAを含むdevice abstractionの議論。RDMAを単なるNIC機能ではなくvirtual device/verbs modelとして扱う補足。
- **Backporting issues with multi-subsystem device (RDMA + netdev + more)** — NIC core、RDMA、その他subsystem間の責務とABIの問題。auxiliary busの設計背景との比較に有用だが、auxiliary busの直接の起源と断定しない。
- **Debugging with BPF sucks if you aren't a kernel developer** — BPF tracingの利用可能性とobservability toolingの課題。
- 出典: https://blog.linuxplumbersconf.org/2017/ocw/events/LPC2017/schedule.html

### LPC 2021 — BPF trust / observability

- **Watching the super powers** — BPF programのauditとsecurity/signing。BPF program lifecycleをverifierの安全性だけに還元しないための補足。
- 出典: https://lpc.events/event/11/contributions/947/

### LPC 2022 — High-speed IP/TCP scalability

- **Can the Linux networking stack be used with very high speed applications?** — 400Gbpsでのpps budget、TCP sequence wrap、FIB lookup costを定量的に提示。BIG TCP、GRO/GSO、zero-copy、CPU/queue placementの設計圧力を説明する導入として有用。
- 出典: https://lpc.events/event/16/contributions/1345/

### LPC 2024 — Linux networkingの現実的な制約

- **What makes the panda sad in the Linux network stack today?** — UDP GSOの制約、BPF socket lookupを用いたUDP return path、TCP timestampとport exhaustionなど。APIを追加するだけでは解決しないsemantic/fallbackの問題を説明する補足。
- 出典: https://lpc.events/event/18/contributions/1962/

### LPC 2025 — Networking Track

- **Networking track kick-off** — netdev maintainerによる近況とnetdev foundationの進捗。個別featureのrelease証拠ではなく、2025年時点の課題・方向性の背景として利用。
- 出典: https://lpc.events/event/19/sessions/236/

## 既存用語集への反映箇所

| 用語集の節 | LPCから得られる追加の説明 |
|---|---|
| Queueing / GRO / BIG TCP | 2022年の400Gbps performance budgetを背景として追加 |
| BPF / observability | 2017年のBPF usability、2021年のaudit/trust、2024年のSK_LOOKUP operational issues |
| TC / switchdev / offload | 2013年のOVS hardware filteringとembedded NIC switchの設計背景 |
| Virtual networking | 2013年のnetwork virtualization、2017年のcontainer networking |
| RDMA / auxiliary bus | 2017年のRDMA+netdev multi-subsystem deviceとparavirtual RDMA |
| Transport | 2024年のUDP GSO/TCP timestampの制約 |

## 残る監査課題

2008–2012、2014–2016、2018–2020、2023の各年度は、今回個別のnetworking関連セッションを網羅的に照合できていない。特に2018–2020と2023はprogramから個別contribution URLを抽出し、発表資料・動画の有無を確認することを優先する。2025もNetworking Track全件の個別発表監査は未完了。未検証年度を「関連発表なし」とは解釈しない。

---



---

# LPC 2018–2020 個別発表の補足（v5）

## 2018–2020 個別セッション追加監査

| 年 | 発表 | r69への有用性 | 発表URL |
|---|---|---|---|
| 2018 | XDP firewall / BPF maps | XDP policyとmap分離 | https://lpc.events/event/2/contributions/100/ |
| 2018 | Kernel FIB lookup from XDP | FIBをBPF側で重複管理しない設計 | https://lpc.events/event/2/contributions/93/ |
| 2018 | OVS with TC BPF/XDP/AF_XDP | 各execution placementの機能制約 | https://lpc.events/event/2/contributions/107/ |
| 2018 | XDP practical introduction | driver RX hook とuserspace control plane | https://lpc.events/event/2/contributions/71/ |
| 2018 | BPF network resource management | TC shaping scalabilityとBPF | https://lpc.events/event/2/contributions/113/ |
| 2019 | Android networking with eBPF | 統計とxlat464 offload | https://lpc.events/event/4/contributions/411/ |
| 2020 | Resizable BPF maps | map capacity/memory accounting | https://lpc.events/event/7/contributions/681/ |
| 2020 | Traceloop and BPF | cgroup単位のflight-recorder tracing | https://lpc.events/event/7/contributions/667/ |
| 2020 | Kubernetes service LB with BPF/XDP | socket layerとdriver XDPの役割分担 | https://lpc.events/event/7/contributions/674/ |

2023年は [eBPF & Networking 詳細時間割](https://lpc.events/event/17/timetable/?view=indico_weeks_view) から、BTF evolution、BPF exceptions、verifier range analysis、BPF memory model 等の発表タイトルを確認した。個別発表の概要・資料をまだ照合していないため、今回の確定済み個別URLには加えていない。

**監査上の注意:** 2018年のOVS発表は当時の制約を説明する資料であり、現在のTC/XDP/AF_XDPの能力やmainline採用状況をそのまま示すものではない。2019年Androidのxlat464、2020年CiliumのKubernetes LBも特定の実装・導入例であって、一般的なkernel API仕様とは区別する。



## LPC 2015–2016: 個別セッション確認済みの補足

2015年は Networking microconference がそれまでの Network Virtualization から IPv6、security へ対象を広げた。以下は **プログラムだけでなく個別セッションの概要まで確認できたもの**である。

| 年 | 発表 / セッション | 技術的な意義 | 個別URL |
|---|---|---|---|
| 2015 | network virtualization with BPF (Alexei Starovoitov) | 既存 bridge/router/tunnel を TC+BPF で組み合わせる。BPF を独立 datapath ではなく kernel networking components の glue とする設計 | https://blog.linuxplumbersconf.org/2015/ocw/sessions/3255.html |
| 2015 | Pushing the Limits of Kernel Networking (Alexander Duyck) | 10→40→100Gbps 時代の small-packet PPS bottleneck。queue/aggregation/scaling 系列の背景 | https://blog.linuxplumbersconf.org/2015/ocw/sessions/2799.html |
| 2015 | Kernel networking and TCP's (ir)relevance (Tom Herbert) | QUIC/SPUD 等の userspace-over-UDP transport と kernel API の役割分担を議論。後年の io_uring/UDP GSO の直接の起源と断定しない | https://blog.linuxplumbersconf.org/2015/ocw/sessions/3309.html |
| 2016 | Container Networking with BPF & XDP (Thomas Graf) | container ごとの動的 BPF policy と driver RX XDP を組み合わせた in-kernel container networking | https://blog.linuxplumbersconf.org/2016/ocw/events/LPC2016/schedule.html |

2015年の [Networking track](https://blog.linuxplumbersconf.org/2015/ocw/events/LPC2015/tracks/447.html) は、単なる講演会ではなく短い導入の後に実装上の未解決点を議論する microconference である。そのためセッション概要は設計課題の根拠として扱い、機能の採用・release attribution の根拠にはしない。

2016年の Container Networking with BPF & XDP は [refereed presentation track](https://blog.linuxplumbersconf.org/2016/ocw/events/LPC2016TALKS/tracks/543.html) に掲載されている。詳細スライドの実査は未了であり、現時点ではプログラムに書かれた概要までを根拠とする。


## LPC 2023 — eBPF & Networking 個別発表の一次資料照合

2023年は、これまで時間割から題名だけ確認していたものについて、Indico の個別 contribution または公式 PDF まで照合した。以下の「発表概要で確認」は発表者の説明に基づくもので、mainline の採用・release を意味しない。

| 発表 | r69対応 | 具体的な位置付け | 一次資料 |
|---|---|---|---|
| Evolving the BPF Type Format | BTF / PROGRAMMABILITY | BTFがftrace等にも使われることに伴う仕様上の拡張課題 | https://lpc.events/event/17/timetable/?view=standard |
| Exceptions in BPF / early termination | BPF verifier / resource lifetime | BPFプログラムの異常終了とruntime resource cleanupの設計上の課題 | https://lpc.events/event/17/contributions/1610/ |
| BPF Memory Model, Two Years On | BPF / synchronization | BPF memory modelの議論が継続していたことの証拠。具体的な保証内容はこの時間割だけから断定しない | https://lpc.events/event/17/timetable/?view=indico_weeks_view |
| BPF struct_ops — current status | BPF struct_ops / TCP congestion control | BPF実装のTCP congestion-control callback登録など、subsystem method contract | https://lpc.events/event/17/contributions/1607/contribution.pdf |
| Container Networking: BPF and network namespaces | VIRTUAL / PROGRAMMABILITY | cgroup BPFとnetns、veth/netkit/IPVLAN/SR-IOVの分担とlatency tradeoff | https://lpc.events/event/17/timetable/?print=1&view=standard_numbered |
| Application network security and observability in an encrypted future | OBSERVABILITY / SECURITY | TLS 1.3、encrypted DNS/HTTPS下でL7可観測性をどう保つか | https://lpc.events/event/17/contributions/1584/ |
| SYN Proxy at Scale with BPF | TCP / BPF | SYN cookieとISN/SEQ/ACK変換の状態保持問題、stateless SYN proxyとkfuncの検討 | https://lpc.events/event/17/contributions/1645/ |
| Modernizing Android BPF and the Android Security Model | BPF / ecosystem | CO-RE/iterator導入時のsecurity policyとdevice lifecycle制約 | https://lpc.events/event/17/contributions/1599/ |

**重要な区別:** `struct_ops` のようなsubsystem callback contractは kernel networkingの拡張点そのものだが、発表に出てくるBPF exceptions、stateless SYN proxy、Android CO-RE展開などは議論・提案・deploymentの話を含む。実装済みの一般機能と一括りにしない。

### 2023のsource granularity

- 個別 contribution: `.../contributions/<ID>/` を優先。
- 発表の存在のみ確認できたもの: 年度時間割への参照とし、**session description/slides未確認**と記す。
- 2023年全252 contributionのうち、ネットワークに関連する全件を完全照合したとは主張しない。

---

# LPC 2023追加監査: netkit / verifier / eBPF ISA / userspace runtime

- **BPF programmable netdevice / netkit**: TCXとnetkitの連携を具体的に説明する公式スライド。`bpf_redirect_peer()`によるhost-side観測点の迂回と、`dev_queue_xmit_nit`相当のBPF kfunc案を区別する。後者は発表時点のfuture workであり、mainline実装済みとは扱わない。Source: https://lpc.events/event/17/contributions/1581/attachments/1292/2602/lpc_netkit_devs.pdf
- **BPF verifierとkernel stack**: verifierの静的解析が成立してもruntime environmentのstack assumptionsが安全性の前提となる。Source: https://lpc.events/event/17/contributions/1595/
- **eBPF ISA conformance**: Linux kernelのverifier/JITだけでなく、BPF ISA specificationに対する複数runtimeの適合性という別のcontract。Source: https://lpc.events/event/17/contributions/1606/
- **bpftime**: userspace BPF runtimeでuprobe等のoverheadを下げる試み。kernel networkingの機能追加や一般的な性能保証と混同しない。Source: https://lpc.events/event/17/contributions/1639/

2023年全体のprogram: https://lpc.events/event/17/sessions/155/ 。2018以前の未確認年度と2025年の全セッションについては監査未完了。


## LPC 2025 Networking Track — 個別プログラムの再照合

2025年のNetworking Trackは12月13日に開催された。以下は公式 timetable/session に掲載された発表であり、**発表題目・概要が確認できる**という意味での監査である。実装のmainline収録は別途r69 Part VI/VIIで確認する。

| 発表 | 用語集との接点 | 公式出典 |
|---|---|---|
| Congestion Signaling (CSIG) for Linux TCP Data Center Networking | TCP congestion feedback / transport | https://lpc.events/event/19/sessions/236/ |
| Packet Metadata — Where Are Thee? | skb metadata / programmable datapath | https://lpc.events/event/19/sessions/236/ |
| The case for zero-copy in containers: Accelerating KubeVirt and enabling memory providers | zero-copy / page_pool provider / virtualization | https://lpc.events/event/19/sessions/236/ |
| Kernel-Native Packet Processing on AMD GPUs: XDP Offload without Userspace Runtimes | XDP / heterogeneous execution | https://lpc.events/event/19/sessions/236/ |
| Linux Networking with MANA: RX Path Optimization and Netshaper Bandwidth Control | page_pool fragments / RX buffer sizing / netshaper | https://lpc.events/event/19/contributions/2276/ |
| Locality-Aware, Self-Adjusting Data Structures: A Case Study in Linux Firewalls | Netfilter / scalability | https://lpc.events/event/19/sessions/236/ |

MANAの発表概要は、64 KiB base pageを用いる環境で1500-byte packetごとにpageを確保するとmemory wasteが生じること、RX queue単位のpre-DMA-mapped page poolとfragment allocationによる改善を説明している。これはpage_poolを単なるallocatorではなく、RX queueに紐づくmemory lifecycle contractとして読む具体例である。

**出典:** https://lpc.events/event/19/contributions/2276/

## LPC 2012 — 一次資料が残る歴史的セッション

LPC 2012の公式resourcesページでは、以下のNetworking発表のスライド/ノートへのリンクが公開されている。古いネットワーク機能の導入動機を調べる際は、近年の回顧より当時の設計議論を優先する。

- Linux Traffic classification and Shaping / Qdisc Experiments at 10Gbps — qdisc/scalability。
- Linux Kernel Implementation of Multipath TCP — MPTCPの初期実装とmainline前の文脈。
- TCP Loss Probe (TLP): fast recovery for tail losses — TCP loss recovery。
- tc interface to hardware transmit rate limiting — TC/hardware offload。
- Byte Queue Limits revisited — BQL。
- KVM Network performance and scalability — virtualization/network path。

**出典:** https://blog.linuxplumbersconf.org/2012/resources-slides-and-videos/index.html

## LPC 2014 — Wireless microconferenceの議事録

2014年のWireless Networking Microconferenceには、当時の発表と議論を記録したetherpad notesがある。これはWi-Fi airtime/AQLを直接裏付ける資料とはまだ断定せず、wireless subsystemの当時の課題を調べる補助出典として区別する。

**出典:** https://wireless.docs.kernel.org/en/latest/en/developers/summits/duesseldorf-2014.html

## 監査上の制限

- 2025年はNetworking Trackの**プログラム**を再照合した。全発表のスライド本文・動画・コードを精査したとは扱わない。
- 2012年は公式resourcesで発表と資料リンクの存在を確認したが、各スライドの本文監査は別作業である。
- 2014年Wirelessは議事録の存在を確認した。AQLの導入時期など、後年の機能を遡及して結び付けない。
- 2008–2011年、2013年、および他年度の全トラック網羅は未完了。


---

# LPC追加監査：残存年度と2025年の出典精密化

監査日: 2026-10-08。対象: 既存v9の未確認年度（特に2008–2011、2013）と2025年Networking Trackの発表単位URL。

## 2025年：公式プログラムから個別発表へ

以下はLPC 2025公式Networking Trackに掲載されている発表。プログラムに載った事実と、mainline実装・採用の事実は区別する。

| 発表 | 対応トピック | 出典 | 確認範囲 |
|---|---|---|---|
| Packet Metadata — Where Are Thee? | skb metadata、bpf_dynptr、RX/TX metadata transport | https://lpc.events/event/19/contributions/2269/ | 個別概要 |
| Running PTP at scale | hardware timestamping、driver API、observability | https://lpc.events/event/19/contributions/2268/ | 個別概要 |
| Networking track kick-off | maintainer update | https://lpc.events/event/19/contributions/2289/ | 個別概要 |
| Kernel-Native Packet Processing on AMD GPUs: XDP Offload without Userspace Runtimes | GPU offload、XDP、DMA-BUF/P2PDMA、Device Memory TCP | https://lpc.events/event/19/contributions/2267/contribution.pdf | 個別発表概要PDF |
| Methodology and Practice in Observing Kernel Networking | production observability、drop/stall、tracing overhead | https://lpc.events/event/19/contributions/2055/ | 個別概要 |

全Networking Trackの一次プログラム: https://lpc.events/event/19/sessions/236/

### 特に重要な技術的区別

- **Packet Metadata**: `skb` metadataをBPF/user-spaceへ運ぶ設計課題とロードマップを扱う。`bpf_dynptr`が登場しても、RX/TX path全体のmetadata transportが完成済みとは断定しない。
- **XDP GPU offload**: 発表はproject/proposalの説明であり、一般的なmainline XDP GPU offloadが完成済みである根拠にはしない。
- **PTP at scale**: timestamping APIとdriverのscale/reliabilityを扱い、通常のpacket timestampingとPTP operational scalabilityを分ける。

## 2008–2011、2013年：監査判定

今回の公開Web検索では、これらの各年度のNetworking関連セッションを**完全な発表単位リストとして再構成できるだけの一次資料を確認できなかった**。したがって、推測した講演タイトルや年次を新たな出典として採用しない。旧ブログ・archiveのprogramと各発表PDFを照合する必要がある。

- 状態: 未完了（資料不存在の意味ではない）
- 次の調査対象: 2008–2011の旧LPC archiveと2013のNetworking Microconference program、個別スライド・動画
- 注意: 古いセッションの開催年と機能のmainline merge年は一致するとは限らない

## Source-quality policy

1. 個別発表URL > 個別セッションPDF > 年度program > トラック一覧 > 大会トップページ。
2. `program-confirmed`、`abstract-reviewed`、`slides-reviewed`、`video-reviewed`、`upstream-verified`を混同しない。
3. kernel実装時期はr69 Part VI/VIIとkernel gitによって別途検証する。


## 2008–2013年の追加確認（v11）

この節は**公式プログラムで発表題目を確認できたもの**と、メーリングリストの**予定・採択通知のみ**を区別する。スライド本文の精査や、当該機能のmainline merge確認は別作業である。

### 2009 — 公式Programで発表題目確認

- David S. Miller, **Networking Futures: Multiqueue Networking**（keynote）。r69のmultiqueue、RSS/RPS/RFS、driver scalabilityの前史。https://blog.linuxplumbersconf.org/2009/program/index.html
- Tom Herbert, **Receive Packet Steering**。RPSの設計経緯を調べる優先資料。https://blog.linuxplumbersconf.org/2009/program/index.html
- Pavel Emelyanov, **Why network namespace sucks and how to make it suck faster**。netns isolation/scalabilityの当時の課題。https://blog.linuxplumbersconf.org/2009/program/index.html
- Steven Rostedt, **Threaded Network Device Interrupts**。threaded NAPIと同一実装ではないが、ネットワーク処理のexecution placement/latencyの歴史的比較対象。https://blog.linuxplumbersconf.org/2009/program/index.html
- John Linville, **Modern Configuration API for Wireless Networking**。wireless control-plane APIの歴史的資料。https://blog.linuxplumbersconf.org/2009/program/index.html

公式Programには複数発表のslides/videoへのリンクが存在する。個々の資料内容はこの追加監査では未読のため、講演題目以上の技術的主張は行わない。

### 2010 — 当時の採択通知を確認

- **User-visible Network Issues** mini-conferenceの採択題目として、**Network Device Naming**、**Scaling techniques in the stack for servers with high connection rates**、**Challenges in Mobile Networking**を確認。https://lists.openwall.net/netdev/2010/10/15/118
- **Achieving Better Out-of-the-box Network Scaling**、**Data Center Bridging**等は2010-09時点の**proposal**として確認。採択済みと断定しない。https://lists.openwall.net/netdev/2010/09/17/90
- 2010年公式サイトにschedule PDFへの案内は存在するが、この追加監査ではPDF本文と各講演資料を照合していない。https://blog.linuxplumbersconf.org/2010/

### 2013 — 公式会期・プログラム案内を確認

- 公式アーカイブには2013年のconference programへの案内があるが、旧scheduleの個別session内容を今回再構成できなかった。**2013年の全Networking発表を確認済みとはしない**。https://blog.linuxplumbersconf.org/2013/

### 2008、2011 — 確認状況

- 2008: 公式archiveの存在は確認したが、networking関連の個別講演を特定する根拠としては不十分。https://blog.linuxplumbersconf.org/2008/index.html
- 2011: この追加監査では確実な個別講演プログラムを取得できなかったため、未確認として残す。

**評価:** 2009年の公式Programはr69のqueue/placement/netnsの前史を補強する有力な一次資料。2010年はproposalと採択済み題目を区別する。古い年度の網羅性は未達。


---

# LPC early-year verification addendum (2026-10-08)

This addendum separates *officially published programme/abstract evidence* from inferred implementation history. It is not a claim of exhaustive session-by-session review.

## 2008 — official schedule and microconference list

- Programme: https://blog.linuxplumbersconf.org/2008/program/schedule/index.html
- Microconferences: https://blog.linuxplumbersconf.org/2008/program/microconfs/index.html
- **Confirmed**: the official microconference list includes debugging/tracing/profiling and kernel/userspace interfaces, but no dedicated networking microconference. Do not infer that there were no networking-related BoFs.
- **Confirmed (proposed topic, not accepted networking talk)**: the server management microconference page mentions networking at 10Gb and beyond, polling/RDMA APIs, and latency among *Ideas*: https://blog.linuxplumbersconf.org/2008/program/microconfs/getmc.php-mc%3Dwong08 . These are not evidence that a corresponding presentation took place.

## 2011 — bufferbloat and queue-management problem statement

- Track proposal/abstract: https://blog.linuxplumbersconf.org/2011/ocw/sessions/81
- **Confirmed abstract**: Bufferbloat and Networking aimed to address excessive buffering, queue latency, congestion control, queue management and network design. This is historical context for the later BQL/CoDel/fq_codel developments; it does **not** prove those later features were presented or merged in 2011.
- 2011 LPC programme structure / retrospectively enumerated tracks: https://blog.linuxplumbersconf.org/2011/

## 2013 — official confirmed proposals, OVS and NIC offload

- Official proposal list: https://blog.linuxplumbersconf.org/2013/ocw/events/LPC2013/proposals/
- **Confirmed presentation titles and abstracts**: *Open vSwitch: Hardware assisted filtering* (Thomas Graf), *Integrating NICs with embedded switches in a Linux ecosystem* (John Fastabend), *Improving Open vSwitch & Network Stack Integration* (Jesse Gross), *OVS: from Open vSwitch to Open network Virtualization System* (Alexei Starovoitov), *Intel DPDK: Data Plane Development Kit* (John Fastabend), *Network Service Headers in Linux* (Kyle Mestery).
- These are valuable pre-switchdev/TC offload and software-vs-hardware datapath architecture references. The official proposal list establishes confirmed session status, **not** the eventual upstream merge or detailed code semantics.

## 2014 — transition toward switchdev architecture

- *Network switching offload architecture for the Linux kernel (Part I)*: https://blog.linuxplumbersconf.org/2014/ocw/proposals/2463
- The proposal page also lists a Part II BoF focused on multi-layer network switching chips and existing offload APIs.
- This provides a more direct conceptual bridge from the 2013 OVS/NIC-offload discussions to the switchdev framework than merely citing a conference homepage.

## Source-verification policy

- **Programme verified**: title, speaker, track and abstract visible in official programme/proposal page.
- **Slides verified**: actual linked presentation inspected, not merely inferred from programme.
- **Mainline verified**: kernel commit and release ancestry separately audited against the canonical r69 Part VI/VII.
- Never turn a programme proposal, abstract or idea list into a claim that a feature was implemented or released.


---

# LPC 2011・2016 一次資料追加確認（v13）

## 2011 — Bufferbloat and Networking microconference

2011年の公式proposal一覧には、次の講演が個別題目として記録されている。これらは**講演の存在とabstractの証拠**であり、機能のmainline導入年の証拠ではない。

| 発表 | 技術的な位置付け | 公式出典 |
|---|---|---|
| Byte Queue Limits (Tom Herbert) | NIC TX ringに蓄積するbytesを制限し、bufferbloatを抑える。qdiscそのものとは異なるdriver-side accounting | https://blog.linuxplumbersconf.org/2011/ocw/events/LPC2011MC/proposals?sort=track |
| Scalable packet processing with many queues and cores (Willem de Bruijn) | NIC multiqueueとCPU数増加に伴うRX/TX placement・scaling | 同上 |
| Optimizing igb and ixgbe network driver scaling performance (Alexander Duyck) | driverのqueue/IRQ/scalability改善 | 同上 |
| TCP Fast Open (Yuchung Cheng) | TCP handshake中のdata exchange、latency削減 | 同上 |
| Proportional Rate Reduction for TCP (Nandita Dukkipati) | loss recovery中のsending-rate制御 | 同上 |
| State of the BufferFish (James Gettys / Dave Täht) | bufferbloatの問題設定と対策 | 同上 |
| Dynamic Firewall Management and Network Zones (Thomas Wörner) | firewall control-plane管理 | 同上 |

**設計上の接続:** BQLは「NICの下位queueに何bytes置けるか」、qdiscは「どのpacketをいつ送るか」、TCP congestion controlは「どのrate/volumeを送るか」を扱う。三者は連動するが同一機能ではない。

公式プログラム: https://blog.linuxplumbersconf.org/2011/ocw/events/LPC2011MC/proposals?sort=track

## 2011 — Unified Memory Management（関連する前史）

同年のUnified Memory Management発表は、allocation、DMA mapping、buffer sharingを別々の課題として整理し、device間zero-copy pipelineを例示している。これは後年のnetmemやDevice Memory TCPと**問題領域が関連する**が、同一APIの直接の起源であることを示す資料ではない。

- https://blog.linuxplumbersconf.org/2011/ocw/sessions/567
- https://blog.linuxplumbersconf.org/2011/ocw/events/LPC2011MC/tracks/117

## 2016 — Linux Bufferbloat fixes: 2011–2016

2016年の発表スライドは2011–2016のqueue/latency改善を振り返り、BQL (3.3)、CoDel/fq_codel (3.5)、TSQ (3.7)、host FQ/pacing (3.13) 等を一覧化している。**歴史を振り返る発表者の整理**として有用だが、正確なmainline releaseの最終根拠はr69 Part VI/VIIとgit tag/commitを優先する。

- 公式スライドPDF: https://blog.linuxplumbersconf.org/2016/ocw/system/presentations/3963/original/linuxplumbers_wifi_latency-3Nov.pdf

## 確認レベルと制約

- 2011年: 公式プログラムの個別題目・abstractを確認。全講演のスライド本文は未確認。
- 2016年: 上記PDFに含まれる年表・説明を確認。全年度の全セッションを監査したという意味ではない。
- release attribution: 講演日・発表スライドだけから推定しない。


---

# LPC一次資料追加監査（v14）：2023・2024・2025

## 2023: netkit / TCX — 発表資料本文

LPC 2023 の netkit 発表スライドは、veth / ipvlan / netkit の device topology、BPF attach位置、routing model、運用上の制約を比較している。特にnetkitはBPF logicをpeer deviceの送信処理へ組み込む設計で、veth上のTC BPFとはprogramの実行位置が異なる。スライド内の *wip* は当時の未完成事項として扱い、現在のmainline機能として読み替えない。

- 一次資料（PDF）: https://lpc.events/event/17/contributions/1581/attachments/1292/2602/lpc_netkit_devs.pdf
- 当日のプログラム・概要: https://lpc.events/event/17/timetable/?print=1&view=standard_numbered

## 2024: packet metadataの問題設定

2024年のmetadata発表スライドは、NIC→XDP→routing→iptables→`sk_lookup`、tunnel decapsulationをまたぐpacket metadataの保持・伝搬を具体的に図示している。`skb` metadataやBPF側のmetadata representationは単独のhookだけで完結しない。2025年の発表へ継続する設計課題として位置付ける。

- 一次資料（PDF）: https://lpc.events/event/18/contributions/1935/attachments/1669/3453/LPC%202024%20Metadata-4.pdf
- 年度別プログラム: https://lpc.events/event/18/contributions/

## 2025: RX/TX metadataの残課題

*Packet Metadata - Where Are Thee?* の公式概要は、古いskb traits案から既存skb metadataの再利用へ方針変更したこと、`bpf_dynptr` の役割、`skb->data_meta` の課題、RX側の伝搬障害、TX側のroadmapを列挙する。これらは**2025年時点の課題・提案**であり、mainlineにすべて実装済みとは記述しない。

- 個別発表: https://lpc.events/event/19/contributions/2269/

## 2025: container zero-copy / hardware RX queue leasing

*The case for zero-copy in containers* は、netkit等のvirtual deviceへphysical NICのhardware queueをleaseする構想を、io_uring/Device Memory TCPとAF_XDP/KubeVirtという異なるconsumerの共通基盤として説明している。さらにAF_XDP TX hook、BPF multi-attach、queue-range attachの拡張を提案する。**発表時点の提案とmainline採用済み機能は別**であり、採用時期はr69 Part VI/VIIで照合する。

- 個別発表・資料リンク: https://lpc.events/event/19/contributions/2275/

## 年度別資料の扱い

- **資料本文確認**: 上記2023年PDF・2024年PDF。
- **公式概要確認**: 上記2025年の2発表。
- **program確認**: https://lpc.events/event/17/timetable/?print=1&view=standard_numbered ; https://lpc.events/event/18/contributions/ ; https://lpc.events/event/19/timetable/?view=indico_weeks_view
- 2013年以前の歴史的発表については、以前の監査結果を維持し、今回追加検証したとは扱わない。

---

# LPC 2024–2025 一次資料の追加確認（v15）

この追補ではLPC公式Indicoの**個別発表ページの概要**を確認した。PDF/動画の全編検証、mainline mergeの確認とは区別する。

## LPC 2024 — eBPFによるpacket追跡（pwru）

- **発表:** pwru - Linux kernel and BPF-based networking debugger
- **発表者:** Gray Liang、Martynas Pumputis
- **確認レベル:** 公式発表ページの概要および添付資料の存在
- **技術的意義:** 既存のhost networking function tracingだけでなく、BPF program内のpacket処理を追跡する必要が生じたことを明示。`tcpdump`では追えないkernel function間のpacket pathが主題。
- **用語集との接続:** `pwru`、Retis、BPF observability、TCX/netkit。
- **出典:** https://lpc.events/event/18/contributions/1942/

## LPC 2024 — Marking Packets With Rich Metadata

- **発表者:** Arthur Fabre、Jakub Sitnicki
- **確認レベル:** 公式発表概要・添付資料の存在
- **技術的意義:** `skb->mark` 32 bitを複数subsystemが共有する限界を指摘。XDP metadataを後段の`sk_lookup`、`reuseport`、socket filter、userspaceの`cmsg`まで運ぶ設計を議論。**提案・課題**であり、全機能のmainline実装を意味しない。
- **用語集との接続:** XDP metadata、skb、BPF program chaining、socket API。
- **出典:** https://lpc.events/event/18/contributions/1935/

## LPC 2024 — Netdev CI

- **発表:** Netdev CI: what is being validated, and what Network subsystems can do
- **発表者:** Matthieu Baerts
- **確認レベル:** 公式発表概要
- **技術的意義:** build/static check中心だったNetdev CIにnetworking selftestsとunit testsを導入する段階を説明。feature chronologyだけでなく**変更を検証するインフラ**の進化として重要。
- **用語集との接続:** observability/evidence、selftests、release validation。
- **出典:** https://lpc.events/event/18/contributions/1961/

## LPC 2024 — Networking APIの実運用上の制約

- **発表:** What makes the panda sad in the Linux network stack today?
- **発表者:** Jakub Sitnicki
- **確認レベル:** 公式発表概要
- **技術的意義:** UDP GSOの制約、BPF socket lookup時のUDP返信source selection、TCP timestampに起因するport exhaustion等。新機能の存在と実際の利用可能性を分けて評価する例。
- **出典:** https://lpc.events/event/18/contributions/1962/

## LPC 2025 — Container zero-copyとRX queue leasing

- **発表:** The case for zero-copy in containers: Accelerating KubeVirt and enabling memory providers
- **発表者:** Anton Protopopov、Daniel Borkmann、David Wei
- **確認レベル:** 公式発表概要・PDF/動画の存在
- **技術的意義:** physical NICのHW queueをnetkitなどのvirtual deviceへleaseし、container内のio_uring、Device Memory TCP、AF_XDPを共通基盤で高速化する構想。KubeVirt/QEMU向けにはAF_XDP TX-side BPF hook、`bpf_mprog`によるmulti-attach、queue-range attachなども議論。
- **重要な注意:** 発表で述べるTX-side API revampは**提案**として扱い、RX queue leasingのmerge statusと混同しない。
- **出典:** https://lpc.events/event/19/contributions/2275/

## LPC 2025 — MANA driverのRX page_pool最適化

- **発表:** Linux Networking with MANA: RX Path Optimization and Netshaper Bandwidth Control
- **確認レベル:** 公式発表概要
- **技術的意義:** 64 KiB base page環境で1500-byte packetごとに1 pageを使う無駄を、page_pool fragmentとRX queue単位のpre-DMA-mapped page poolで改善する説明。page_poolを単なる汎用allocatorではなく、DMA lifecycleとfragment reuseの仕組みとして読む具体例。
- **出典:** https://lpc.events/event/19/contributions/2276/

## 監査方法と未確認範囲

- **確認済み:** 上記6件の公式発表ページ、2024 Networking Track一覧、2025 Networking Track一覧。
- **未実施:** 全PDFの本文精読、動画の視聴、全年度の全セッション網羅、各発表に対応するkernel commit/tag照合。
- **基準:** 「programで採択を確認」「abstract確認」「slides本文確認」「mainline実装確認」を別の証拠レベルとして扱う。
- **2024公式一覧:** https://lpc.events/event/18/sessions/179/
- **2025公式一覧:** https://lpc.events/event/19/sessions/236/


---

# LPC 2024–2025 スライド本文の深掘り（v16）

## LPC 2024: Marking Packets With Rich Metadata

- 発表資料（70ページ）: https://lpc.events/event/18/contributions/1935/attachments/1669/3453/LPC%202024%20Metadata-4.pdf
- 確認レベル: **PDF本文**（特にp.3–4, p.8–9, p.13–14, p.19–27, p.29–37）。

### 背景と実装上の制約

`skb->mark` は32-bit fieldで、routing/firewall/NAT/XFRM/SO_MARKなど複数consumerが意味を共有する。このため、サービス識別、packet provenance、trace correlationを単一markへ押し込む設計は衝突しやすい（p.3–5）。

5-tupleによる追跡はencapsulation/decapsulationやuserspaceをまたぐと不安定であり、`sk_buff` identityも`skb_clone()`やnetns境界で限界がある（p.13–14）。発表はpacket IDなどをrich metadataとして運ぶ案を提示しているが、これは**完成したmainline APIの説明ではなく設計提案**である。

### data_meta / skb representation

XDPには`struct xdp_md.data_meta`と`bpf_xdp_adjust_meta()`によるmetadata headroomがある（p.20–21）。スライドは`sk_buff`内でのmetadata領域、TCの`__sk_buff.data_meta`、socket filterへの拡張、`sk_lookup`へmetadata accessを追加する可能性を検討している（p.22–27）。この範囲の一部は**proposal**であり、全BPF program typeで利用可能と解釈してはいけない。

### userspace境界とlifetime

スライドp.29–37は、packet metadataをsocket/userspaceへ届ける方法と、socket storage、`sockops`、`getsockopt`の組み合わせを比較している。packetごとのmetadataとconnection lifetimeのmetadataを混同しないことが重要である。図に出てくる`skb->meta`はconceptual notationであり、当該構造体fieldがmainlineに存在することを示すものではない。

**r69対応:** PROGRAMMABILITY / CONTROL PLANE / OBSERVABILITY。特にmetadata lifetimeとcross-hook semanticsの未解決点を補足する。

## LPC 2025: The case for zero-copy in containers

- 発表ページ: https://lpc.events/event/19/contributions/2275/
- 発表PDF（64ページ）: https://lpc.events/event/19/contributions/2275/attachments/1946/4156/zero_copy.pdf
- 確認レベル: **64ページのPDF本文を確認**。詳細は末尾のv17監査章を参照。

発表概要は、physical NIC hardware queueをnetkit等のvirtual deviceへleaseし、container/Podのio_uring/Device Memory TCPとKubeVirt向けAF_XDPを共通基盤で扱うことを目的としている。AF_XDP TX hook、BPF multi-attach (`bpf_mprog`)、queue-range attachも議論される。これらを「2025年時点ですべてmainline済み」と扱ってはいけない。個別機能のmerge/revert/re-mergeはr69 Part VI/VIIで検証する。

**r69対応:** MEMORY / PROGRAMMABILITY / VIRTUAL / DRIVER FRAMEWORK。特にqueue ownershipのAPI contract。

## LPC 2025: Linux Networking with MANA

- 発表ページ: https://lpc.events/event/19/contributions/2276/
- 確認レベル: **公式概要**（Google Slides本文は未監査）。

64KiB base pageのARM64環境で、1500-byte RX packetにpage全体を割り当てる方式の無駄を減らすため、MANA RXにpage_pool fragmentsとper-RX-queue pre-DMA-mapped page poolsを導入した事例。発表概要の測定値は64KiB ARM64でper-packet memory overhead 97%以上削減、throughput最大約200%改善、x86-64でmemory約50%削減、throughput約15%改善とされる。**講演者が報告した測定結果**であり、一般のNICに保証される値ではない。

後半はnetshaperによるdriver-level bandwidth controlを扱う。`tc`のqdiscとnetshaperは抽象化レベルが異なり、後者はdevice/driver側のshaping resource管理を意図する。

**r69対応:** PERFORMANCE / MEMORY / CONTROL PLANE / DRIVER FRAMEWORK。

## Provenance注意事項

- PDF本文確認と発表概要確認を区別した。
- PDF内のdesign sketchをmainline APIとして記述しない。
- benchmarkは対象ハードウェア・構成付きの発表結果として扱う。
- commit/releaseはLPC資料ではなくr69 Part VI/VIIおよびupstream gitで判定する。


---

# LPC 2025 zero-copy in containers — 64-page slide audit

Primary PDF: https://lpc.events/event/19/contributions/2275/attachments/1946/4156/zero_copy.pdf
Talk: https://lpc.events/event/19/contributions/2275/

## Architecture (slides 4–29)

- Slides 4–9: conventional KubeVirt Pod bridge/vtap/virtio-net datapath compared with Cilium netkit/TCX/FIB. Slide 7 reports +13% throughput, slide 8 12% lower latency under the stated ConnectX-6 Dx TCP_RR test configuration; these are presentation benchmarks, not general performance claims.
- Slides 10–14: SR-IOV VF assignment trades host network-stack visibility/policy and unified Pod networking for hardware virtualization; the authors propose netkit-based queue leasing rather than simply placing a VF into the Pod.
- Slides 15–22: host namespace physical NIC RX queues are mapped to virtual netkit queues. Netkit peer with multiple RX queues moves to Pod netns; the application binds the netkit ifindex and virtual queue ID. This is **queue resource delegation**, not a migration of the physical NIC into the Pod.
- Slide 23: AF_XDP bypasses the upper kernel networking stack, while io_uring/TCP zero-copy retains the TCP/IP stack and uses header/data split. The two have different packet-processing semantics despite sharing queue leasing.
- Slides 25–29: proposed KubeVirt datapath uses RSS context based on VM MAC, netkit-to-physical-queue binding, QEMU AF_XDP backend. Authors report +37% TCP_RR throughput and 30–35% lower latency in their preliminary benchmark, but explicitly say QEMU's AF_XDP backend was not yet fully zero-copy (slide 26).

## Implementation and lifecycle (slides 30–36)

- Slides 30–32: proposed netdev Generic Netlink (`netdev.yaml`) extensions for queue operations. This is a **v5 patch-series snapshot**, not a stable UAPI guarantee.
- Slide 33: virtual netdev locked before physical netdev; virtual netdev retains reference to physical; lease lasts until virtual-device destruction; resizing affecting leased queues is blocked; physical-device removal notifier tears down associated virtual devices. These are key deadlock, reference-lifetime, and hot-unplug contracts.
- Slide 34: new virtual-device `ndo` for queue creation; netkit allocates a real virtual queue and sysfs entries and returns a queue ID to be used for subsequent leasing.
- Slide 35: existing memory-provider user API remains unchanged, with netkit peer ifindex + virtual queue ID translated to physical queue identity.

## XDP API proposals (slides 38–52)

- Slides 38–39: AF_XDP TX bypass creates a policy/observability gap; authors propose an XDP TX hook and per-queue/RSS-context attach.
- Slides 40–44: reuse `bpf_mprog` (inspired by TCX) for XDP multi-attach, proposing `BPF_XDP_INGRESS` and `BPF_XDP_EGRESS`, with relative program placement and expected revision.
- Slides 45–52: a straightforward fast-path `bpf_mprog` loop would require changes in XDP drivers. Slides explore an alternative generated BPF wrapper with direct calls to reduce driver changes. **Do not treat `BPF_DIRECT_CALL`, new attach types, or wrapper scheme as merged mainline APIs on the basis of these slides.**

## Status and provenance

The slides describe a mixture of pre-existing netkit/TCX/XDP infrastructure, a queue-leasing v5 patch series, benchmark prototypes, and forward-looking XDP API proposals. The canonical r69 Part VI/VII is the separate release/commit authority; the LPC presentation alone does not prove upstream merge status.

## Follow-up source checks

- Check the `netdev` Generic Netlink specification for actually exported queue create/lease commands in the relevant release: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html
- Check AF_XDP's supported rings, bind/queue semantics, and driver zero-copy limitations: https://www.kernel.org/doc/html/latest/networking/af_xdp.html
- Check XDP attach types against the corresponding kernel's `include/uapi/linux/bpf.h` rather than assuming proposed names were accepted.

---

# v18追補 — LPC 2025 queue leasing と2026年patch seriesの設計変遷

## 1. LPC発表時の問題設定

Pod内の仮想netdevはhost namespace内の物理NIC RX queueを直接構成できない。このためAF_XDPやpage_pool memory providerを必要とするio_uring ZCRX/Device Memory TCPを、コンテナからnative-speedで利用することが難しかった。queue leasingは仮想queueを物理RX queueへproxyすることでこの境界を接続する。

- LPC公式講演: https://lpc.events/event/19/contributions/2275/
- 公式スライド: https://lpc.events/event/19/contributions/2275/attachments/1946/4156/zero_copy.pdf

## 2. queue leasing の主要実装ポイント

| Interface / object | 役割 | 調査先 |
|---|---|---|
| `ndo_queue_create` | 仮想netdev側queue生成callback | `include/linux/netdevice.h`, `drivers/net/netkit.c` |
| `netdev_nl_queue_create_doit` | netdev Generic Netlink経由のqueue生成 | `net/core/netdev-genl.c`周辺、netdev YNL spec |
| RX queue proxy | 仮想queueから物理queueへの対応付け | `include/net/netdev_rx_queue.h` |
| memory-provider open/close proxy | leased queueから実queueへのprovider操作 | `include/net/page_pool/memory_provider.h` |
| AF_XDP pool proxy | leased queueへのXSK bindとpool lifecycle | `include/net/xdp_sock_drv.h`, netkit driver |
| DMA device lookup | memory provider/UMEMで使うDMA deviceの解決 | `include/net/netdev_queues.h` |
| unregister / resize handling | 物理NICの消滅やqueue再構成時の参照安全性 | netkit notifier, ethtool queue management |

上記は2026年3月のv10 patch seriesで示された変更ファイルとseries説明をもとにした実装調査の入口である。実際のfunction signatureはkernel revisionに依存する。

- v10 patch series: https://lwn.net/Articles/1065043/
- netdev Generic Netlink spec: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 3. 2025→2026 API変遷

1. 2025年9月の初期seriesでは *queue peering* と呼ばれ、virtual queueからreal queueへのmappingを提案。
2. 2025年10月のv4ではbind時のABBA deadlock防止、物理device消滅時のhandling、コンテナへの物理device情報露出防止を強化。
3. 2026年1月のv5では *queue leasing* に改称し、queue-create APIへ再構成。selftests、複数queue binding、DMA device lookup整理が進む。
4. 2026年3月のv10では`ndo_queue_create`、AF_XDP pool proxy、memory-provider proxy、netkit selftestsを含むseriesとして提示。

- 初期series: https://lwn.net/Articles/1038910/
- v4: https://lwn.net/Articles/1044813/
- v5: https://lwn.net/Articles/1053710/
- v10: https://lwn.net/Articles/1065043/

## 4. zero-copy RXとTXを混同しない

LPCのAF_XDP use caseはkernel TCP stackをbypassするが、io_uring ZCRXとDevice Memory TCPはTCP stackを利用する。RX queue leasing自体はRX側のhardware queueを仮想netdevへproxyする基盤であり、TX側のXDP policy hookやDevice Memory TCP TX転送を自動的に完成させるものではない。

2026年4〜5月にはnetkitを経由するDevice Memory TCP TXについて、CPU-unreadable DMA-BUF backed skbのforwarding validationとDMA mappingを扱う追加patch seriesが提案された。これはqueue leasing本体とは別の実装系列として扱う。

- devmem/netkit TX initial series: https://lwn.net/Articles/1070256/
- devmem/netkit TX v3: https://lwn.net/Articles/1071898/

## 5. 監査上の注意

LPCスライドの設計案、patch seriesの提案、mainlineで実際に採用された実装は異なる。release attributionとmerge/revert履歴についてはr69 Part VI/VIIを正本とする。特にXDP egress attach、`bpf_mprog`によるmulti-attach、queue-range attachの提案をqueue leasing RX本体のmergeと同一視しない。


---

# Queue leasing 実装追跡 — v19

## 1. Kernel control path

```text
userspace YNL netdev queue-create
  -> net/core/netdev-genl.c:netdev_nl_queue_create_doit()
     -> virtual netdev を lock
     -> lease元の physical netdev を取得して lock
     -> physical RX queue ID / busy state を検証
     -> virtual netdev queue_mgmt_ops->ndo_queue_create()
        -> drivers/net/netkit.c:netkit_queue_create()
           -> netif_set_real_num_rx_queues()
     -> netdev_rx_queue_lease(virtual_rxq, physical_rxq)
```

`queue-create` は既存queueの属性変更ではなく、仮想netdevにRX queueを追加して物理RX queueと対応付ける操作である。virtual → physical のlock順序はABBA deadlockを避けるための明示的なcontractである。

### YNL API例（ifindexは環境依存）

```bash
ynl --family netdev --output-json --do queue-create \
  --json '{"ifindex":8,"type":"rx","lease":{"ifindex":4,"queue":{"type":"rx","id":15}}}'
```

実装例: https://git.zx2c4.com/wireguard-linux/commit/include/net?h=devel&id=d04686d9bc86432ea3008d5f358373d8466d1943

## 2. netkit側のqueue生成

`netkit_queue_create()` は`dev->real_num_rx_queues`を読み、`netif_set_real_num_rx_queues()`で1 queue追加し、元のqueue数を新規queue IDとして返す。paired modeでは非primary peer側のみがlease可能で、single-device modeではlease可能。作成時のqueue capacityは`alloc_netdev_mqs()`で確保する。capacityと現在のreal queue数は異なる。

実装: https://kernel.googlesource.com/pub/scm/linux/kernel/git/netdev/net/%2B/6b5a2b7d9bc156e505f09e698d85d6a1547c1206/drivers/net/netkit.c

## 3. 代表的なvalidationと制約

- `queue_id_lease >= real_num_rx_queues` は範囲外として拒否。
- physical queueがすでにbusyの場合は拒否。
- 異なるphysical deviceから同じvirtual deviceへのqueue leaseは、capability/DMA deviceの共通部分を安全に計算できないため拒否する実装が確認できる。
- 現時点でqueue leasingはRX側のみ。`netdev_queue_get_dma_dev()`もRX leased queueに限ってphysical deviceのDMA deviceへproxyする。
- netkitのqueue countは作成時の上限があり、ethtool -Lによる後からのresizeを禁止する設計。

Sources:
- https://code.googlesource.com/linux/torvalds/linux/%2B/68ff4a3ccda9f98c74f23c70c8c7c581f9eee931/net/core/netdev-genl.c
- https://kernel.googlesource.com/pub/scm/linux/kernel/git/torvalds/linux.git/%2B/7dd38d9dd7a05329825fe2324d4d8e27ad4b3cec/net/core/netdev_queues.c
- https://lists.openwall.net/netdev/2026/01/29/421

## 4. Runtime observation

`queue-get` は `lease` 情報と`napi-id`、`dmabuf`、`io-uring`、`xsk`等を返す。したがってvirtual ifindex/queue idを物理queueと結び付け、provider bindingがどちらにあるかを調べられる。netdev YNL仕様では`lease.netns-id`も定義される。

Source: https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 5. 2025 LPCとmainlineの区別

LPC 2025スライドに含まれるAF_XDP TX hook、XDP EGRESS/multi-attach等の将来案を、RX queue leasingのmainline実装と混同しない。queue leasingのmerge/rollback/remergeはr69 Part VIIのcommit anchorを正本とする。

### 調査レベル

本章は公開されたkernel関数本文とYNL仕様、およびpatch cover letterを照合したもの。全エラー経路の動的実行確認、selftestsの実行、kernel tag containmentの再監査は未実施。


---

# v20 — RX queue leasing lifecycle / locking / error path 監査

調査日: 2026-10-08。対象: upstream Linux の公開ソース（GitHub master の閲覧時点）。**実機・selftest実行なし**。v19のcall pathを前提とするが、未確認の遷移を実装済みと断定しない。

## 1. 操作の境界: `queue-create` は存在するが、対称な `queue-delete` は確認できない

`net/core/netdev-genl-gen.h` のnetdev generic netlink操作一覧には `netdev_nl_queue_create_doit` がある一方、`netdev_nl_queue_delete_doit` はない。従って、`create → bind → unbind → delete` という独立したNetlink API列を前提とした状態遷移図は現状の仕様として不正確である。`queue-create` で生成された仮想queueの後始末はdevice teardown/queue cleanup側のソースで確認する必要がある。**単独queue削除APIがあるとは記述しない。**

- https://github.com/torvalds/linux/blob/master/net/core/netdev-genl-gen.h
- https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html

## 2. queue-createで確認できる正常系・拒否経路

```text
netdev_nl_queue_create_doit()
  required attrs/type RX / nested lease attrs検証
  → [cross-netns] CAP_NET_ADMIN / netns-id lookup
  → virtual netdev lock
  → netdev_can_create_queue()
  → physical netdev reference取得・lock
  → netdev_can_lease_queue()
  → physical queue index range check
  → netdev_queue_busy()
  → same physical NIC restriction
  → ndo_queue_create()  [virtual queue増設]
  → netdev_rx_queue_lease() [virtual↔physical関連付け]
  → unlock/reply
```

公開コードから確認できる拒否条件:

| 条件 | 観測される処理 | 注意 |
|---|---|---|
| 必須attribute欠落 / queue type != RX | `-EINVAL` | TX leasingではない |
| 別netns指定かつCAP_NET_ADMINなし | `-EPERM` | namespace境界での権限検証 |
| netns-id不正 | `-ENONET` | source namespace取得失敗 |
| virtual netdevなし | `-ENODEV` | 参照取得に失敗 |
| physical queue id >= real_num_rx_queues | `-ERANGE` | v19記述の `-EINVAL` と区別 |
| physical queue busy | `-EBUSY` | AF_XDP / lease / memory provider |
| 既存leaseが別physical NIC由来 | `-EOPNOTSUPP` | capabilities/DMA deviceの交差を未実装 |
| `ndo_queue_create()`失敗 | callbackの負値を返す | ここまではleaseを確立しない |

Sources: https://code.googlesource.com/linux/torvalds/linux/+/3a2976df778a9af95e91f7ff88008b4517ddc658/net/core/netdev-genl.c ; https://github.com/torvalds/linux/blob/master/net/core/netdev_queues.c

### 重要な境界: callback成功後

`ndo_queue_create()` の成功後に `netdev_rx_queue_lease()` を呼ぶ。**この間に別のエラー戻りや明示的rollbackがあるか**を、固定commitの関数本文で最終確認すべき。`genlmsg_reply()` の送信結果とqueue生成の原子性も区別する必要がある（送信失敗はqueue作成の取り消しを意味しない）。これはコードからの監査論点であり、動的検証結果ではない。

## 3. lock順序とDMA lookup

`queue-create` のソースコメントは virtual → physical の順序を明示する。`netdev_queue_get_dma_dev()` は、leased RX queueの場合にphysical RX queueへproxyし、physical deviceのlockを取得してDMA deviceを返す。**RXのみ**で、非leaseのqueueには通常のdriver callback/parent DMA deviceが使われる。

```text
virtual netdev lock
  └── physical netdev lock
        └── physical RX queue / DMA device
```

`netdev_queue_get_dma_dev()` はDMA device pointerを返すが、それだけで DMA-BUF lifetime / mapping lifetime が保証されるとはいえない。mapping ownershipはmemory provider側で別途追跡する。

Source: https://github.com/torvalds/linux/blob/master/net/core/netdev_queues.c

## 4. busy判定: 3系統を区別

`netdev_queue_busy()` は、AF_XDP pool (`xsk_get_pool_from_qid`)、既存queue lease (`netif_rxq_is_leased`)、memory provider (`netif_rxq_has_mp`) を区別して検査する。いずれかが存在するとqueue leasingを拒否する。これは**create時の排他**であり、unregisterとunbindの並行時の全安全性を証明するものではない。

Source: https://github.com/torvalds/linux/blob/master/net/core/netdev_queues.c

## 5. 未解決のlifecycle / concurrency監査項目

以下は**未検証**であり、安全・危険いずれとも断定しない。

| シナリオ | 追跡する対象 | 確認したい不変条件 |
|---|---|---|
| physical netdev unregisterとvirtual netdev削除の競合 | `netdev_rx_queue_lease`, netdev unregister notifier, queue release | physical queueを参照中にfreeしない |
| namespace削除とqueue-create競合 | netns reference / device tracker / lock | netnsとdeviceのlifetimeが操作終了まで有効 |
| AF_XDP bindとqueue-create競合 | XSK pool binding / `netdev_queue_busy()` | 同一hardware RX queueを二重に占有しない |
| memory provider bindとqueue-create競合 | `mp_params`, RX queue open/close | providerの状態を破壊しない |
| ethtool -Lとlease競合 | real queue count / queue reallocation | leased RX queueのindex/pointerが不意に変わらない |
| queue-createのreply delivery失敗 | Netlink reply / userspace retry | 成功した副作用を重複生成しない |

### 調査手順（ソース確認用）

```bash
# mainlineの固定tag/commitをcheckoutしてから実行
rg -n 'netdev_rx_queue_lease|netif_rxq_is_leased|__netif_get_rx_queue_lease' net include drivers
rg -n 'queue_create|queue_delete|queue_busy|ndo_queue_create' net/core drivers/net include/net Documentation/netlink
rg -n 'NETDEV_UNREGISTER|unregister_netdevice|netdev_unregister' net/core drivers/net
rg -n 'netif_set_real_num_rx_queues|ndo_queue_create|ethtool.*channels' drivers/net/netkit.c net/core
rg -n 'queue-create|lease|netkit|xsk' tools/testing/selftests/net tools/testing/selftests/bpf
```

上記コマンドは**調査計画**であり、実行済みselftestではない。

## 6. v19に対する監査上の修正

- v19の `queue_id_lease >= real_num_rx_queues` の説明を、`queue-create`で返す実際のerror `-ERANGE` に修正。
- `queue-delete`/`unbind` を独立したNetlink操作として図示しない。
- AF_XDPのpool、memory provider、既存leaseはbusy判定の異なる原因として扱う。
- sourceをmaster閲覧時点のものと固定SHAのものに分ける。将来のmaster更新により内容が変わり得る。

## 7. Evidence / limitations

- **関数本文を確認:** `netdev_nl_queue_create_doit()`、`netdev_queue_busy()`、`netdev_queue_get_dma_dev()`、`netdev_can_create_queue()`、`netdev_can_lease_queue()`。
- **操作宣言を確認:** `netdev_nl_queue_create_doit()`があり、対称な`netdev_nl_queue_delete_doit()`は公開された生成headerにない。
- **未確認:** 全unregister notifierの呼出順、release時の参照カウント、全エラー経路の動的試験、selftest実行、race reproducer、固定tagに対する全関数の再検証。

## Sources

1. https://code.googlesource.com/linux/torvalds/linux/+/3a2976df778a9af95e91f7ff88008b4517ddc658/net/core/netdev-genl.c
2. https://github.com/torvalds/linux/blob/master/net/core/netdev_queues.c
3. https://github.com/torvalds/linux/blob/master/net/core/netdev-genl-gen.h
4. https://github.com/torvalds/linux/blob/master/drivers/net/netkit.c
5. https://lists.openwall.net/netdev/2026/03/20/371
6. https://www.kernel.org/doc/html/latest/netlink/specs/netdev.html


---

# v21 — RX queue leasing: unlease・参照管理・memory provider cleanup

調査日: 2026-10-08。基準ソース: Linux v7.1-rc1 `net/core/netdev_rx_queue.c`。公開ソースの静的確認であり、selftest・動的競合検証は未実施。

## 1. 対称な参照操作

`netdev_rx_queue_lease(dst, src)` は両netdevのlock保持をassertし、`netdev_hold(src->dev, &src->lease_tracker, GFP_KERNEL)` に続いて `src->lease=dst` と `dst->lease=src` を書き込む。`src` はphysical RX queue、`dst` はvirtual RX queueに対応する。

`netdev_rx_queue_unlease(dst, src)` は同じlock前提で `netif_rxq_cleanup_unlease(src,dst)` → 双方のlease pointerをNULL → `netdev_put(src->dev,&src->lease_tracker)` の順に処理する。したがって、物理netdevの参照を保持したままprovider cleanupを行う。

根拠: https://kernel.googlesource.com/pub/scm/linux/kernel/git/mic/linux/+/refs/tags/v7.1-rc1/net/core/netdev_rx_queue.c （関数冒頭）

## 2. Memory provider cleanup

`netif_rxq_cleanup_unlease(phys_rxq, virt_rxq)` はphysical RX queueの `mp_params.mp_ops` がNULLなら何もしない。providerが存在する場合、`__netif_mp_uninstall_rxq(virt_rxq,p)` を呼び、`__netif_mp_close_rxq(phys_rxq->dev,idx,p)` でphysical queue側のprovider設定を消去・再構成する。

このcleanupが必要な理由は、**physical RX queueはvirtual queueより長生きできる**ため。virtual queueのlease解除後もphysical queueに古いprovider状態が残ることを防ぐ。

## 3. RX queue reconfiguration失敗時

`netdev_rx_queue_reconfig()` はnew/old queue memoryを確保し、新queue memoryをallocateし、必要なら旧queue stop → 新queue startを行う。新queue startが失敗すると旧queue startで復旧を試み、旧queueの再起動にも失敗した場合はWARNを出す。これは**常に完全復旧する保証ではない**。

`__netif_mp_close_rxq()` は再構成エラーについて `-ENETDOWN` 以外をWARNする。provider cleanupは無条件の成功を保証する操作ではない。

## 4. 未解決の競合条件と次の確認先

- physical netdev unregisterが始まる場合、lease参照によりnetdevのfreeが遅延することは示せるが、**unregister自体を阻止する**とは断定できない。
- virtual netdev削除・netns teardown・physical unregisterでどのcallerが `netdev_rx_queue_unlease()` を実行するか、順序・lock acquisitionを追跡する必要がある。
- AF_XDP poolのdetach/unbindとprovider uninstallの並列処理は、`net/xdp/xsk.c` およびdriver側と合わせて調べる必要がある。
- `netif_mp_close_rxq()` はleased queueの場合physical netdevへ転送してlockする。virtual→physical lock順序をcaller含めて監査する。

関連ソース:
- https://github.com/torvalds/linux/blob/master/net/xdp/xsk.c
- https://github.com/torvalds/linux/blob/master/net/core/netdev-genl.c
- https://github.com/torvalds/linux/blob/master/net/core/dev.h

## 5. 実装に基づくテスト案（未実行）

1. queue-create成功→virtual netdev削除後にphysical queueのlease/provider情報が消えること。
2. providerをvirtual leased queueへattach→virtual削除→physical queueへ再attachできること。
3. AF_XDP pool bind済みphysical queueをleaseしようとしてbusy拒否されること。
4. netns teardownとphysical unregisterを同時に行い、ref trackerとlockdep/KASANの警告を確認すること。
5. driverのqueue restart失敗注入時に旧queue復旧とWARNの発生を観測すること。

上記は試験仕様案であり、結果を示すものではない。


---

# v22 — RX queue leasing: physical unregister → netkit notifier → unlease

調査日: 2026-10-08。静的ソース・patch履歴監査。実機競合試験は未実施。

## 1. 主要な発見

物理netdevが `NETREG_UNREGISTERING` となったとき、netkitのnotifierが `netkit_check_lease_unregister()` を通じてそのRX queueのlease先を走査し、netkit deviceをunregisterする。これにより、`netdev_hold()` の参照が残っているだけで物理netdevの撤去が停滞する状況を解消する意図がある。

- patch v10: https://lists.openwall.net/netdev/2026/03/27/240
- v11シリーズ適用報告: https://www.spinics.net/lists/netdev/msg1176249.html
- v11 notifier commit: https://git.kernel.org/netdev/net-next/c/25444470570b
- 実装スナップショット: https://kernel.googlesource.com/pub/scm/linux/kernel/git/netdev/net/+/6b5a2b7d9bc156e505f09e698d85d6a1547c1206/drivers/net/netkit.c

## 2. physical unregister の経路

```text
NETDEV_UNREGISTER event (notifier)
  -> netkit_check_lease_unregister(physical_dev)
      guard: reg_state == NETREG_UNREGISTERING && dev->dev.parent != NULL
      netdev_lock_ops(physical_dev)
      for q_idx in [0, real_num_rx_queues):
        __netif_get_rx_queue_lease(..., NETIF_PHYS_TO_VIRT)
        if owner is netkit and not already queued:
          netkit_del_link(owner, &list_kill)
      netdev_unlock_ops(physical_dev)
      unregister_netdevice_many(&list_kill)
```

**注意:** 上記はnotifierを起点とした概念的call graph。notifier登録・callback分岐とnetdevice coreのunregister全体における正確な同期位置は別途追跡が必要。

## 3. netkit削除側

2026-03-27のpatch v10 (11/14)では以下の実装が示される。

```text
netkit_del_link()
  -> unregister_netdevice_queue(netkit, head)
  -> peerも対象（ただし既にqueuedなら二重追加しない）
...
netkit_uninit()
  -> netkit_release_all()
  -> netkit_queue_unlease()
      -> netdev_lock(virtual)
      -> 各 leased queue: netdev_lock(physical)
      -> netdev_rx_queue_unlease(virtual_rxq, physical_rxq)
      -> netdev_unlock(physical)
      -> netdev_unlock(virtual)
```

`netdev_rx_queue_unlease()`はmemory provider cleanup、双方向lease pointer解除、`netdev_put(physical)` の順。出典: https://kernel.googlesource.com/pub/scm/linux/kernel/git/mic/linux/+/refs/tags/v7.1-rc1/net/core/netdev_rx_queue.c

## 4. 競合条件の整理

| 事象 | 静的に確認できる防御 | 残る検証 |
|---|---|---|
| physical unregister | notifierがlease先netkitを削除対象へ追加 | notifierとnetdevice coreの呼出し・lock順序の完全追跡 |
| 同じnetkitに複数queueをlease | `unregister_netdevice_queued()` で重複処理回避 | 同時netns teardown時の全ケース |
| peerが既にunregister queueに存在 | peer二重追加を防ぐguard | 実際のlist整合性試験 |
| virtual netkit削除 | `netkit_uninit()` → `netkit_queue_unlease()` | AF_XDP socket teardownと同時進行するケース |
| memory providerあり | `netif_rxq_cleanup_unlease()` | reconfiguration失敗後のdriver状態 |

## 5. 関連selftestの位置付け

v11シリーズに `selftests/net: Add queue leasing tests with netkit` が含まれ、commit `65d657d80684` として取り込まれた。**この監査ではテスト本文のassertや実行結果は未確認**。

https://git.kernel.org/netdev/net-next/c/65d657d80684

## 6. 今後の確認ポイント

1. `netkit_notifier()` のイベント分岐、登録と解除、`unregister_netdevice_many()` の同期位置。
2. `netkit_queue_unlease()` の全分岐、`netdev_lock` と `netdev_lock_ops` の取得順序、lockdep。
3. AF_XDP `xsk` bind/unbindと物理queue teardownの競合。
4. selftest `65d657d80684` の具体的なテスト項目、hot-unplug・netns teardown coverage。
5. `netif_rxq_cleanup_unlease()` のerror propagation（またはWARNのみ）とDMA buffer lifetime。

## 7. 確認レベル

- **直接確認:** v10 patchに記載されたnotifier、queue走査、重複回避、uninit/unlease呼び出し。v11シリーズ適用報告と個別commit URL。
- **補助確認:** 公開netkit.cのソーススニペットとv7.1-rc1 `netdev_rx_queue.c`。
- **未確認:** 完全なunregister call graph、lockdep/実行結果、selftest本文、すべての失敗経路。


---

# v23 — AF_XDP / io_uring ZCRX / Device Memory TCP 比較監査

作成日: 2026-10-08。対象: Linux networking の RX data path。kernel.org の現行ドキュメントを主要資料とする。mainline導入版は canonical r69 Part VI/VII を参照し、本稿の `latest` 記述と混同しない。

## 1. まず分類を正す

- **AF_XDP**: XDP program + XSKMAP でpacketをXSKへredirectするkernel-bypass寄りのpacket I/O。通常のTCP socket stackは通らない。**AF_XDPというだけでzero-copyとは限らない**。XDP_SKB、XDP_DRVのcopy modeと、NIC/driverが対応するXDP_ZEROCOPYを区別する。
- **io_uring ZCRX**: TCP stackでheaderを処理し、payloadを登録済みuserspace領域へ直接DMAするsocket receive API。`IORING_OP_RECV_ZC`、`io_uring_zcrx_cqe`、refill ringを使う。`TCP_ZEROCOPY_RECEIVE`とは別物。
- **Device Memory TCP (devmem TCP)**: TCP stackでheaderを処理し、payloadをDMA-BUFに受信するmemory-placement mechanism。`MSG_SOCK_DEVMEM`、`SCM_DEVMEM_DMABUF`、`SO_DEVMEM_DONTNEED`を用いる受信インターフェースを持つ。TXは別のAPI/条件を持つ。

**関係性:** AF_XDP と TCP socket pathは通常異なる受信経路。ZCRX と devmem TCPは概念上排他的なカテゴリではないが、**任意のDMA-BUFを現行のZCRX登録領域としてそのまま使えると断言してはいけない**。組合せの成立は対象versionのregistration API、memory provider、driverの対応で確認する。

## 2. RX data path

```text
(A) AF_XDP (zero-copy-capable driverの場合)
NIC RX DMA -> UMEM frame -> XDP driver program -> XSKMAP redirect
    -> AF_XDP RX ring descriptor -> userspace packet processing
    -> userspace recycles frame to FILL ring

(B) io_uring ZCRX
NIC RX header split -> header in host/kernel memory -> TCP/IP stack
                    -> payload DMA to registered userspace region
    -> IORING_OP_RECV_ZC CQE (offset/length) -> userspace processing
    -> ZCRX refill ring -> NIC RX buffer reuse

(C) Device Memory TCP RX
NIC RX header split -> header in host/kernel memory -> TCP/IP stack
                    -> payload DMA into DMA-BUF / device memory
    -> recvmsg(MSG_SOCK_DEVMEM) + SCM_DEVMEM_DMABUF token/offset
    -> device-side consumption -> SO_DEVMEM_DONTNEED(token)
```

## 3. Contract比較

| 観点 | AF_XDP | io_uring ZCRX | Device Memory TCP RX |
|---|---|---|---|
| API/対象 | AF_XDP socket、XSKMAP、packet | TCP socket、io_uring recv | TCP socket、DMA-BUF |
| TCP stack | 通常は迂回 | 通過 | 通過 |
| RX buffer | UMEM frames | 登録済みuserspace memory area | RX queueにbindされたDMA-BUF |
| アプリが受け取るもの | RX ring `xdp_desc` | CQEのoffset等 | `recvmsg` control messageとfrag token |
| buffer返却 | FILL ringへframe address | ZCRX refill ringへ返却 | `SO_DEVMEM_DONTNEED` |
| TX completionとの混同注意 | COMPLETION ringは**TX** frame再利用通知 | CQEはreceive completion、refillは別 | RX token返却とTX `MSG_ERRQUEUE`は別 |
| NIC/driver要件 | zero-copyには対応driver/NIC; copy fallbackあり | header/data split、flow steering等 | header split、flow steering、RSS、対応memory provider |
| skb | native XDP redirect pathでは通常作らない | TCP/IP stackで利用 | TCP/IP stackで利用（payloadはCPU不可読の場合あり） |
| 主な失敗・制約 | XSKMAP未設定、queue不一致、FILL枯渇、copy mode | 未対応NIC、steering不備、refill枯渇、登録失敗 | queue/buffer枯渇、header split例外、CPU不可読payload |

## 4. ownershipを正確に扱う

### AF_XDP

1. Userspaceは利用可能なUMEM frameのoffsetをFILL ringへ積む。ここで受信バッファの利用権をkernel側へ渡す。
2. NIC/driverが受信し、XDP programがXSKMAPへredirectすると、RX ringに`xdp_desc`が現れる。
3. Userspaceは受信データの処理後に同じframeを再びFILL ringへ渡せる。TXに回す場合はTX ringに渡し、**TX COMPLETION ring**で再利用可能となったことを確認する。
4. RX ring descriptorの受領だけで自動的にframeが再利用されるわけではない。

### io_uring ZCRX

1. アプリが受信メモリ領域とrefill ringを登録する。kernel/NICは登録領域のbufferを受信に使う。
2. `IORING_OP_RECV_ZC`のcompletionでデータ位置を通知する。TCP headerはkernelが処理する。
3. アプリは消費後に**refill ring**へbufferを戻す。CQEのackだけをbuffer返却とみなさない。
4. メモリ登録、DMA pinning、completionの実際の粒度はkernel versionとproviderで検証する。

### Device Memory TCP

1. アプリはnetdev Generic Netlink `bind-rx`でDMA-BUFをNIC RX queueへbindする。bindしたnetlink socketをcloseするとbindingが解除される。
2. TCP受信時、payloadがDMA-BUFへ入った場合、`SCM_DEVMEM_DMABUF`に`dmabuf_id`、offset、size、frag tokenが入る。host memoryへ入った部分は`SCM_DEVMEM_LINEAR`になる場合がある。
3. `recvmsg(MSG_SOCK_DEVMEM)`後、アプリはdevice側でpayloadを消費する。kernelはfragmentをpinしており、消費後`setsockopt(SO_DEVMEM_DONTNEED)`でtokenを返却する。
4. tokenを返さなければ限られたDMA-BUF領域が枯渇しpacket dropを招く。**CPUからpayloadを読めない**ためsoftware checksum、tcpdump、BPFのpayload観測等に制限がある。

## 5. 「zero-copy」の範囲

- **AF_XDP**: `XDP_ZEROCOPY`成立時のNIC↔UMEM copy回避。XDP_SKBやdriver copy modeはzero-copyではない。
- **ZCRX**: kernel TCP socket receiveでkernel→userspace copyを回避。header processingやTCP state処理まで省略するわけではない。
- **devmem TCP**: NIC→device memoryへのpayload直接配置。headerはhost memoryで通常処理し、software pathでpayloadを読む操作に制限が生じる。
- 3方式を一律に「コピー0回」と表現しない。header/payload分離、driver fallback、scatter-gather、TX/RX方向、DMA mappingの有無を区別する。

## 6. 検証用チェックリスト

1. `ethtool -i`, `ethtool -k`, `ethtool -g`, `ethtool -n/-u`でNIC/driver、header split、flow steering、queue構成を確認。
2. AF_XDP: `XDP_ZEROCOPY`要求時のbind成功/失敗と`XDP_COPY`の挙動を区別し、FILL不足、XSKMAPミスマッチ、TX completionをテスト。
3. ZCRX: io_uring setup flags、registration、`IORING_OP_RECV_ZC`、CQE、refill ringをそれぞれ追跡。buffer未返却時の挙動を観察。
4. devmem TCP: `netdev bind-rx`、steering、`MSG_SOCK_DEVMEM`、DMABUF/LINEAR cmsg、`SO_DEVMEM_DONTNEED`を確認。token返却停止時のdropとnetlink socket close時のunbindingをテスト。
5. すべてで、CPU使用率、host memory bandwidth、PCIe traffic、throughput、latencyを別々に測る。zero-copy化で性能が必ず向上するとは限らない。

## 7. 一次資料

- AF_XDP: https://www.kernel.org/doc/html/latest/networking/af_xdp.html
- io_uring zero copy Rx: https://www.kernel.org/doc/html/latest/networking/iou-zcrx.html
- Device Memory TCP: https://www.kernel.org/doc/html/latest/networking/devmem.html
- Device Memory TCP (NetDev 0x17): https://netdevconf.info/0x17/sessions/talk/device-memory-tcp.html
- io_uring ZCRX (NetDev 0x17): https://netdevconf.info/0x17/sessions/talk/zero-copy-receive-using-io_uring.html
- Linux 6.15 release coverage: https://lwn.net/Articles/1022457/

## 8. 確認済みと未確認

**確認済み:** kernel.org公開ドキュメントのAPI、ownership説明、3方式の基本データ経路。**未確認:** 現行7.xでの全関数call graph、実NIC上のzero-copy成否、driverごとのfallback条件、3方式の実測性能、ZCRXとDMA-BUFの具体的相互運用条件。これらは推測で埋めない。


---

# v24 — page_pool / netmem / memory provider / DMA ownership 監査

調査日: 2026-10-08。対象: AF_XDP、io_uring ZCRX、Device Memory TCP のRX側buffer ownership。

## バージョンと証拠レベル

- **Linux v6.15**: [Netmem Support for Network Drivers](https://www.kernel.org/doc/html/v6.15/networking/netmem.html) — **バージョン固定の公式仕様文書**。コードそのものではない。
- **Linux source commit `f24ba334afafc70c3149e9db9c0cf8ecc6d52a09`**: [net/core/page_pool.c](https://code.googlesource.com/linux/torvalds/linux/+/f24ba334afafc70c3149e9db9c0cf8ecc6d52a09/net/core/page_pool.c) — **commit固定のコード**。このcommitがどのリリースタグに含まれるかは今回未確認。以下の関数のコード説明はこのcommitを基準とする。
- **Linux latest (2026-10-08閲覧)**: [Page Pool API](https://www.kernel.org/doc/html/latest/networking/page_pool.html)、[Device Memory TCP](https://kernel.org/doc/html/latest/networking/devmem.html)、[io_uring zero copy Rx](https://kernel.org/doc/html/latest/networking/iou-zcrx.html) — **可変の公式仕様文書**。リリースを固定した証拠には使わない。
- **Linux master (2026-10-08閲覧)**: [net/core/page_pool.c](https://github.com/torvalds/linux/blob/master/net/core/page_pool.c) — **可変のコード**。関数名の確認に利用し、v6.15のコードだとは主張しない。

## 1. コードのalloc / recycle / release経路

固定commit `f24ba334...` の `net/core/page_pool.c` にある主要な関数名を調査対象とする。

```text
page_pool_create()
  -> page_pool_create_percpu()
      -> page_pool_init()
          -> memory provider init (providerを設定した場合)

page_pool_alloc_netmems()
  -> cached netmem
  -> memory provider alloc_netmems() [providerあり]
  -> ordinary page allocation [providerなし]

page_pool_put_unrefed_netmem()
  -> page_pool recycle cache / ptr_ring
  -> page_pool_return_netmem() [recycle不可]
      -> memory provider release_netmem() [providerあり]
      -> DMA unmap + put_page() [ordinary page]
```

上図は**実装の概念的な分岐**であり、すべての例外条件・参照カウント経路を省略している。特に`release_netmem()`の返り値によって通常の`put_page()`を実行するかが変わる。provider-backed bufferを常に通常のpage allocatorへ返すわけではない。

## 2. DMA mappingとCPU同期を混同しない

| 概念 | 主体・意味 | 注意点 |
|---|---|---|
| Buffer allocation | page_pool / provider | `netmem_ref`は表現であってalloc処理そのものではない |
| DMA address | NIC driver / DMA API / provider | DMAアドレスはCPU仮想アドレスと同義ではない |
| DMA sync | DMA API + driver/page_pool | streaming DMAでCPU/NIC間の可視性を調整する。CPU不可読device memoryでは通常pageと同じ前提を置けない |
| Recycling | page_pool cache / ptr_ring | 利用中の参照が残るbufferは即時再利用できない |
| Final release | page_pool/provider | provider固有のreleaseと普通の`put_page()`は区別 |

## 3. 3方式のbuffer返却を同じ言葉でまとめない

| 方式 | Buffer種別 | ユーザーから見た返却 | Kernel側の意味 |
|---|---|---|---|
| AF_XDP RX | XSK UMEM frame | FILL ringへ空きframeを供給 | アプリが再利用可能としたframeをNIC RXへ再供給。RX用COMPLETION ringは存在しない |
| io_uring ZCRX | 登録したユーザー領域 | ZCRX refill ringのRQE | CQEが示す受信データを処理した後、該当chunkを再利用可能にする |
| Device Memory TCP | DMA-BUF-backed device memory fragment | `SO_DEVMEM_DONTNEED`でtoken返却 | kernelが保持しているfragment pinを解放し、有限のdevice memoryを再利用可能にする |

**注意**: io_uring ZCRXはsocket API/ユーザーメモリの登録機構、Device Memory TCPはTCP受信のdevice memory配置機構であり、概念的に排他的な三者択一ではない。現行APIが同一buffer上でどう組み合わさるかは実装・kernel version・driverによるため、この資料では統合構成の動作を断定しない。

## 4. バッファ枯渇とlifetime

- AF_XDP: RXで消費したframeをFILL ringへ再投入しなければ、NICへ渡せる空きframeが不足する。
- io_uring ZCRX: CQEの処理だけではbuffer返却にならず、refill ringへRQEを投入する必要がある。登録解除時のin-flight処理は別途監査対象。
- Device Memory TCP: `SCM_DEVMEM_DMABUF`で通知されたfragmentはkernelにpinされ、`SO_DEVMEM_DONTNEED`のtoken返却を怠るとDMA-BUF枯渇とdropにつながる（公式文書に明記）。
- page_pool: `pages_state_hold_cnt`と`pages_state_release_cnt`の差でin-flightを追跡する実装がある。これは各アプリケーションのtoken数と同義ではない。

## 5. 今回未確認の範囲

1. `f24ba334...`のリリースタグ包含と、v6.15・v7.1における関数本文のdiff。
2. `mp_dmabuf_devmem.c`のprovider固有のalloc/release実装とDMA-BUF attach/map/unmapの完全なcall graph。
3. `io_uring/zcrx.c`の登録解除とCQE/refill間のrace・参照解放順序。
4. mlx5等の実driverにおけるDMA direction・page_pool recycle・header splitの実装差。
5. 実機selftest、perf、メモリリーク・競合再現試験。

上記は**未検証**であり、正常性・競合安全性を証明したとは扱わない。


---

# v25: DMA-BUF / io_uring ZCRX memory lifecycle audit

## Scope and version discipline

- **Confirmed source version:** Linux **v6.16**, `net/core/devmem.c`: https://github.com/torvalds/linux/blob/v6.16/net/core/devmem.c
- **Not version-pinned, reference only:** `io_uring/zcrx.c` master: https://github.com/torvalds/linux/blob/master/io_uring/zcrx.c . Do not attribute these details to v6.16 until the tag-specific file is verified.
- **Not version-pinned, reference only:** `include/uapi/linux/io_uring/zcrx.h` master: https://github.com/torvalds/linux/blob/master/include/uapi/linux/io_uring/zcrx.h
- Kernel documentation (rolling): https://docs.kernel.org/networking/devmem.html
- No dynamic tests performed. This is a static source review.

## 1. Device Memory TCP DMA-BUF binding (Linux v6.16)

`net_devmem_bind_dmabuf()` (`net/core/devmem.c`):

1. `dma_buf_get(dmabuf_fd)` acquires the DMA-BUF reference.
2. `dma_buf_attach(binding->dmabuf, dev->dev.parent)` creates the device attachment.
3. `dma_buf_map_attachment_unlocked(binding->attachment, direction)` obtains the DMA-mapped scatter-gather table.
4. The SG DMA addresses are recorded in `gen_pool` chunks; each chunk owns `net_iov` entries (`NET_IOV_DMABUF`).
5. `page_pool_set_dma_addr_netmem()` associates each `net_iov`'s `netmem_ref` with its DMA address.
6. An XArray binding ID and socket-associated binding list entry are installed.

`net_devmem_bind_dmabuf_to_queue()` opens the RX queue with `__net_mp_open_rxq()` and records its association. If recording fails, `__net_mp_close_rxq()` rolls back the queue change.

`mp_dmabuf_devmem_init()` disables ordinary page-pool DMA synchronization for DMA-BUF memory (`pool->dma_sync = false; pool->dma_sync_for_cpu = false`) and rejects nonzero page-pool order.

## 2. Unbind vs final release (Linux v6.16)

`net_devmem_unbind_dmabuf()`:

- Erases the global binding ID, then `synchronize_net()` to exclude in-flight lookups.
- Closes each associated RX queue through `__net_mp_close_rxq()`.
- Drops the binding reference with `net_devmem_dmabuf_binding_put()`.

`__net_devmem_dmabuf_binding_free()` (deferred work):

- Cleans up chunk-owner metadata and `gen_pool` (subject to pool availability check).
- Calls `dma_buf_unmap_attachment_unlocked()`, `dma_buf_detach()`, `dma_buf_put()`.
- Destroys the queue-association XArray and frees binding metadata.

**Conclusion:** Unbinding a queue is not synonymous with immediately unmapping the DMA-BUF. Mapping lifetime follows binding references and delayed cleanup.

## 3. Source-audit caveat: DMA direction (Linux v6.16)

The mapping call takes a `direction` parameter; the observed `__net_devmem_dmabuf_binding_free()` and error rollback unmap calls use `DMA_FROM_DEVICE`. This deserves focused review for TX bindings. The source inspection alone is insufficient to label it a confirmed defect or establish whether it has since been corrected.

## 4. io_uring ZCRX: provisional comparison (master, NOT v6.16)

The *unversioned* `io_uring/zcrx.c` exposes `io_import_dmabuf()`:

`dma_buf_get(dmabuf_fd)` → `dma_buf_attach(mem->dmabuf, ifq->dev)` → `dma_buf_map_attachment_unlocked(..., DMA_FROM_DEVICE)` → SG-length validation.

Failure cleanup calls `io_release_dmabuf()`, which unmaps, detaches, and puts the DMA-BUF. This describes the observed master source, **not a verified v6.16 implementation**.

The master UAPI shows `IORING_REGISTER_ZCRX_IFQ`, `io_uring_zcrx_area_reg`, `io_uring_zcrx_rqe`, `ZCRX_REG_IMPORT` and `IORING_ZCRX_AREA_DMABUF`. These symbols may not all exist in older releases. A version-tag comparison is required before writing a release-specific call graph.

## 5. Ownership comparison

| Dimension | Device Memory TCP (v6.16 confirmed) | io_uring ZCRX (master provisional) |
|---|---|---|
| DMA-BUF attach target | Physical device parent (`dev->dev.parent`) | IFQ's DMA device (`ifq->dev`) |
| DMA direction at map | Supplied `direction` | `DMA_FROM_DEVICE` |
| RX queue association | Memory-provider binding per RX queue | IFQ registration / refill ring |
| Return mechanism | `SO_DEVMEM_DONTNEED` tokens (rolling documentation) | ZCRX refill queue entries (master UAPI) |
| DMA unmap | Deferred binding free | Area/IFQ release (full teardown path not audited) |
| Version certainty | Confirmed v6.16 | **Not pinned** |

## Next verification tasks

1. Retrieve `io_uring/zcrx.c` at an immutable Linux tag or SHA and identify registration/teardown functions with exact source lines.
2. Compare DMA-BUF importer and user-memory ZCRX paths, including pin/unpin and partial-registration rollback.
3. Trace `net_devmem_dmabuf_binding_put()` and its deferred work scheduling to verify all references and release ordering.
4. Inspect `DMA_TO_DEVICE` TX unmap direction in later tags and patch history.
5. Execute relevant selftests on supported NIC hardware; no test results are claimed here.


---

# v26 — io_uring ZCRX 固定バージョン調査（Linux v6.16）

調査日: 2026-10-08。**コード参照は Linux v6.16 固定タグ**。以下は静的コード調査であり、実機試験は未実施。

## 一次ソース

- Linux v6.16 `io_uring/zcrx.c`: https://kernel.googlesource.com/pub/scm/linux/kernel/git/stable/linux-stable.git/+/refs/tags/v6.16/io_uring/zcrx.c
- Linux v6.16 `io_uring/zcrx.h`: https://gbmc.googlesource.com/linux/+/refs/heads/linux-6.16.y/io_uring/zcrx.h （**stable branchは可変**、固定タグではないため補助資料）
- Linux v6.17 ZCRX documentation: https://www.kernel.org/doc/html/v6.17/networking/iou-zcrx.html （説明用。v6.16コードと同一とは限らない）
- 比較対象 Linux v6.16 Device Memory TCP: https://github.com/torvalds/linux/blob/v6.16/net/core/devmem.c

## 1. 登録経路：Linux v6.16

`io_register_zcrx_ifq()` がIFQとrefill ringを登録する入口。ソースに現れる `io_zcrx_ifq_alloc()` はIFQを確保し、`if_rxq = -1`、spinlock、DMA用mutexを初期化する。

`io_close_queue()` は、登録されたRX queueを `net_mp_close_rxq()` で閉じ、netdev参照を `netdev_put()` で返却して `if_rxq = -1` にする。`io_zcrx_ifq_free()` は `io_close_queue()` と `io_zcrx_drop_netdev()` を経由し、area・device参照・refill ringを解放する。**これらの関数名・呼出関係はv6.16固定タグのソースで確認**。

### 概念的な所有権遷移

```text
ユーザー: IFQ register
    ↓
IFQ / refill ring の初期化
    ↓
NIC RX queue に ZCRX memory provider を設定
    ↓
RX packet の payload を登録bufferに格納
    ↓
アプリ: CQEからoffset等を取得してbufferを読む
    ↓
アプリ: 使用済みbufferをrefill ringに返却
    ↓
NIC がbufferを再利用可能

終了: io_close_queue() → net_mp_close_rxq() → netdev_put()
      io_zcrx_ifq_free() → area/device/ring解放
```

上図は**所有権説明の概念図**であり、各矢印が単一関数呼出しを意味するものではない。

## 2. 返却と終了の違い

- CQEは受信完了を通知するが、受信bufferをNICに返却する操作そのものではない。
- refill ringに使用済みbufferを登録して、kernel側がそれを回収してから再利用する。
- IFQのunregister/closeはqueueを閉じ、memory providerとDMA/メモリ参照を片付ける**別のlifecycle**。
- RX bufferの返却をAF_XDP TX COMPLETION ringと混同しない。

## 3. v6.16とmasterの差分に関する注意

`master`の `io_register_zcrx()`、`ZCRX_REG_IMPORT`、`IORING_ZCRX_AREA_DMABUF` などを、v6.16で確認した関数名・UAPIとして引用しない。`master`は新機能追加で大きく変わっており、リリースタグのソースと混在させるとcall graphを誤る。

## 4. 比較表（コードバージョン明示）

| 観点 | ZCRX（Linux v6.16） | Device Memory TCP（Linux v6.16） |
|---|---|---|
| queueとの接続 | IFQ register、`net_mp_close_rxq()`で解除 | DMA-BUF bindingをRX queueのmemory providerへ関連付け |
| 受信通知 | io_uring CQE | `recvmsg()`とancillary data |
| buffer返却 | refill ring | `SO_DEVMEM_DONTNEED` token |
| 最終解放 | IFQ closeとarea/ring/device解放 | binding参照が尽きてから遅延DMA-BUF unmap/detach |
| CPU可視性 | 登録したmemory areaの種類に依存 | device memoryはCPUから直接読めない場合がある |

## 5. 確認できていない点

- v6.16の `io_zcrx_recv()` の全分岐、refill consumerの細部、エラーごとの参照カウント収支は未精査。
- DMA-BUFを使う新しいZCRX拡張については、v6.16の機能であると断定しない。
- memory provider closeと未返却bufferの競合安全性、DMA unmap完了タイミングは動的試験で未検証。
- 参照ページが部分取得だったため、上記を全関数網羅のソース監査と称さない。

## 次の作業

Linux v6.16タグの `io_uring/zcrx.c` 全文を取得可能な環境で、`io_zcrx_recv()`、refill ring consumer、`io_unregister_zcrx_ifqs()`、`io_shutdown_zcrx_ifqs()` の各分岐を行番号付きで精査し、error path別のreference balanceを監査する。


---

# v27 — Linux networking全体への横断調査（4領域）

> 調査基準日: 2026-10-08。v26までのqueue leasing/zero-copy個別調査を終了し、r69の六軸に戻す。以下の「確認」は明記した**バージョンの公式文書**に対するもの。関数本文を直接取得・行単位照合したとの意味ではない。

## 1. Packet metadata：情報は自動的にはend-to-endで保持されない

### バージョンを固定した確認

- **Linux v6.3**: XDP RX Metadata仕様。`bpf_xdp_metadata_rx_timestamp()`、`bpf_xdp_metadata_rx_hash()`を提供し、driver未対応は `-EOPNOTSUPP`、packetに情報がない場合は `-ENODATA`。XDPプログラムは `bpf_xdp_adjust_meta()` で `xdp_buff->data_meta` に独自レイアウトを作れる。AF_XDP側のレイアウト解釈はアプリとの契約であり、RX descriptorにmetadata長はない。v6.3文書は `XDP_PASS` での `xdp_buff → skb` 変換時、XDP metadataが自動的にskb metadataへ転記されないことを明記する。
  - https://docs.kernel.org/6.3/networking/xdp-rx-metadata.html
- **Linux v6.11**: `netdev` netlink specificationでXDP RX metadata capability（timestamp/hash/VLAN tag）を列挙。**capabilityの公開**と**各packetでのmetadata保持**は別問題。
  - https://docs.kernel.org/6.11/networking/netlink_spec/netdev.html
- **LPC 2024発表（実装済みの証拠ではない）**: Rich Packet Metadata。`skb->mark`等の既存フィールドに情報を詰め込む限界、XDP/TC/routing/socketを横断するmetadata contractの課題。
  - https://lpc.events/event/18/contributions/1935/
  - https://lpc.events/event/18/contributions/1935/attachments/1669/3453/LPC%202024%20Metadata-4.pdf

### 実装調査の次の焦点

| 境界 | 確認する対象 | 主要な破綻条件 |
|---|---|---|
| NIC → XDP | driver metadata kfunc、`xdp_buff` | driver非対応、情報欠落 |
| XDP → AF_XDP | `data_meta`、UMEM headroom | producer/consumerのlayout不一致 |
| XDP_PASS → skb | driverのskb構築、metadata変換 | 独自metadataが自動では保持されない |
| TC → routing | `skb` metadata、clone/encap | clone、再生成、encapsulationでの意味の変化 |
| routing → socket | mark、priority、timestamp、cmsg | 全フィールドがsocket APIへ公開されるわけではない |

**解釈**: XDP/TC/socketをまたぐ共通の汎用metadata objectが既に存在する、という表現は避ける。転送契約はmetadataの種類・datapath・driver・consumerごとに確認する。

## 2. Control plane scalability：lockの粒度とAPI生成は異なる課題

- **Linux v6.15公式文書** `networking/netdevices.html`: 従来のglobal RTNLからper-netns RTNLへ移行中と説明し、netdev instance lockの利用を明記。これは**全rtnetlink操作が完全にper-netns化済み**という意味ではない。
  - https://docs.kernel.org/6.15/networking/netdevices.html
- **Linux v6.13公式文書** `networking/devlink/index.html`: devlink instance lockの取得順はdevlink → RTNL。`devl_*` APIは呼び出し側がlockを保持する契約。global RTNLの縮小は、driver独自のlock inversion問題を消さない。
  - https://docs.kernel.org/6.13/networking/devlink/index.html
- **Linux v6.13公式文書** `networking/ethtool-netlink.html`: ethtoolのGeneric Netlink family、attributeベースのAPI、extended ACK。YNLはnetlink familyの仕様を元にユーザー側のコード生成・利用を支援するものであり、YNL自体がkernel側のRTNL競合を解決するわけではない。
  - https://docs.kernel.org/6.13/networking/ethtool-netlink.html
- **r69既存のmainline anchor（本調査で再検証していない）**: per-netns RTNL移行開始 `fcc79e1714e8c2b8e216dc3149812edd37884eef`、v6.13。

### 監査軸

1. command入口でglobal RTNL / per-netns RTNL / instance lockのどれを取得するか。
2. 複数netns・複数netdevを扱う操作のlock ordering。
3. dumpとmutationの整合性、netdev unregister時の参照。
4. YNL specとkernel handlerのattribute policy/ExtAckの一致。

## 3. Driver framework：共通化する責務は別々

| framework | 共通化する主な責務 | driver固有に残る責務 | 固定版の確認資料 |
|---|---|---|---|
| page_pool / netmem | RX buffer再利用とbuffer表現 | DMA条件、RX descriptor、NAPI統合 | v6.15 `networking/page_pool.html` / `networking/netmem.html`（追加精査対象） |
| phylink | PHY/MAC/link mode・SFP等の調停 | MAC register/PCS設定、hardware capability | **v5.3** `networking/sfp-phylink.html` |
| DIM | interrupt moderationの測定・制御の共通ロジック | hardware moderation register、driver integration | r69に既存anchor、今回は関数本文未確認 |
| devlink | chip-wide resources/health/portsの管理API | firmware/hardware固有の実装 | **v6.13** `networking/devlink/index.html` |
| Rust PHY abstraction | PHY driver登録・操作の型付き抽象 | PHY固有の制御・reg操作 | **v6.12** `rust.docs.kernel.org/6.12/kernel/net/phy/index.html` |

- phylink v5.3文書: https://docs.kernel.org/5.3/networking/sfp-phylink.html
- devlink v6.13文書: https://docs.kernel.org/6.13/networking/devlink/index.html
- Rust PHY v6.12文書: https://rust.docs.kernel.org/6.12/kernel/net/phy/index.html
- **歴史上の導入版と、仕様を読んだ版を混同しない**。例: phylinkの導入はr69ではv4.14と記録、上記のAPI説明はv5.3。

## 4. Observability：dropの理由とpacketの履歴は異なる

| 仕組み | 何を観測するか | 主な限界 | 追加確認する固定版コード |
|---|---|---|---|
| `skb_drop_reason` | skb解放時のreason分類 | XDP_DROP、NIC hardware drop、reason設定されないpathを完全には説明できない | v6.16 `include/net/dropreason-core.h`, `net/core/skbuff.c` |
| drop monitor / `kfree_skb` tracepoint | skb dropイベント・位置・理由（構成依存） | packetがskbになる前、hardware内のdropは不可視 | v6.16 `net/core/drop_monitor.c` |
| pwru | eBPFによるpacket関連関数・skbの通過点追跡 | kernel/BTF/attach可能点、skbがない区間 | kernel v6.16 + pwru releaseを別々に固定 |
| Retis | eBPFによるpacket tracingと相関 | tracerのprobe対応とkernel機能に依存 | kernel v6.16 + Retis releaseを別々に固定 |

- **導入時期**: `skb_drop_reason`はv5.17（r69の既存整理）。本節のv6.16は**次に関数本文を検証する対象版**であり、導入版を示さない。
- **検証すべき境界**: NIC hardware → XDP → skb生成 → TC/OVS → IP/conntrack → socket。各地点でskbの存在、pointerの同一性、clone/GRO/GSOによる相関破綻を区別する。
- LPC発表資料を使う場合は「実装済み」ではなく「発表で提起された問題」と明記する。

## 5. 次の具体的なソース監査計画

1. **Packet metadata**を最優先: Linux **v6.16固定タグ**の `net/core/xdp.c`、`include/net/xdp.h`、`net/core/skbuff.c`、代表driverのXDP_PASS処理を取得し、`data_meta`の受け渡しを追う。LPC 2024の提案と混同しない。
2. **Control plane**: Linux **v6.13とv6.16の差分**でRTNL lock API、`rtnetlink_rcv_msg()`、`devl_*`経路を比較する。YNLはuserspace generatorとして別章。
3. **Driver framework**: 各frameworkの導入版、共通化されたcallback、driver固有部分、lock順を表にする。
4. **Observability**: v5.17とv6.16の`skb_drop_reason`のAPI差分、drop monitorのtracepoint、pwru/Retisの観測可能範囲をテスト例付きで整理する。

### 検証レベル

- **確認済み**: リンク先のバージョン固定公式ドキュメントの記述と、その概念的な比較。
- **未確認**: v6.16の関数本文・全error path、r69の各SHAのタグ包含、pwru/Retisの特定releaseでのprobe coverage。未確認項目は断定しない。

---

# v28追補 — Packet metadata scrub / skb extension の一次資料監査（2026-10-08）

## 対象と証拠レベル

本追補は v27 の四軸監査を継承し、2026-09-10 の net-next **提案パッチ**を一次資料として確認したもの。`skb_metadata_clear()` の全 callsite の固定タグソース監査はまだ実施していない。パッチ差分の存在を mainline merge と読み替えない。

## 1. `skb_scrub_packet()` と `skb_ext_reset()` の境界

Jakub Sitnicki の `[PATCH net-next v2 03/14] bpf: Make BPF skb extension survive packet scrubbing` によると、従来の `skb_scrub_packet()` は `skb_ext_reset(skb)` により skb extensions を一律解除する。呼び出し経路には tunnel encap/decap（`ip_tunnel_rcv`, `vxlan_rcv` 等）と cross-netns forwarding（`dev_forward_skb`）がある。この性質は **skb extensions** についてのもので、XDP custom `data_meta` と同義ではない。

提案差分では `skb_ext_reset(skb)` を `skb_ext_scrub(skb)` に変更し、`SKB_EXT_BPF` だけを残して他の extensions を削除する。clone と共有する extension slab の実体を即時破棄せず、skb 単位の `active_extensions` bit を落とす設計である。

## 2. UDP receive の追加境界

同パッチは `udp_try_make_stateless()` も `skb_ext_scrub()` に変更する。BPF extension が残る場合は stateless fast path を使わない。提案の説明によれば、そうしないと `__consume_stateless_skb()` が `skb_release_head_state()` を省略し、extension slab のリークにつながる。この修正は tunnel 境界だけでなく **UDP socket receive queue と ownership cleanup** に関係する。

## 3. 2026年9月の議論

9月23日の返信では、PSP skb extension も namespace forwarding をまたいで保持したいという別の要望が出ている。したがって、`SKB_EXT_BPF` のみを残す提案は、すべての extension の保持要件を満たすとは限らない。9月24日にも議論が継続している。現時点で merge commit/tag containment は未検証。

## 4. Metadata種別の区別

| 対象 | 格納・参照 | scrubでの扱い（確認範囲） |
|---|---|---|
| XDP custom metadata | `data_meta` と packet headroom | このパッチは直接扱わない。`skb_metadata_clear()` callsite 別途監査 |
| skb extensions | `skb->extensions` / active bit | 既存 `skb_ext_reset()` で解除。提案は BPF extension を選択保持 |
| `skb->mark` | skb field | cross-netns の `xnet` によるリセットを別途検証 |
| tunnel metadata | tunnel dst / tunnel info | skb extension / XDP custom bytes と混同しない |

## 5. 次に確定する項目

1. **v6.16固定タグ**の `skb_metadata_clear()` 定義・全呼び出し箇所と call path をコード行付きで抽出する。
2. `skb_scrub_packet()` の v6.16 と 2026-09提案を差分比較し、 `xnet` 条件と `skb_ext_reset` を分離する。
3. VXLAN/Geneve、GRO、UDP/TCP socket について **bytes preserved / semantic validity / userspace exposure** を別判定する。
4. BPF skb extension series の後続revision、mainline SHA、tag包含、selftest coverageを追跡する。

### 一次資料

- 2026-09-10 v2 03/14: https://lists.openwall.net/netdev/2026/09/10/367
- 2026-09-23 PSP extension関連の返信: https://lists.openwall.net/netdev/2026/09/23/467
- 2026-09-24 継続議論: https://lists.openwall.net/netdev/2026/09/24/499

**証拠区分:** 提案パッチの本文・差分とメールでの議論を確認。固定タグの全 callsite、merge status、実行テストは未確認。

---

# v29 追補：Packet metadata callsite監査の検証境界（2026-10-08）

> **監査状態：部分完了・ソース取得待ち。** v6.16固定タグのkernel tree全文を今回の環境で取得できなかったため、`skb_metadata_clear()`の「全callsite確認済み」とは記載しない。v28までの確定事項と、次回の再現可能な抽出手順を分離する。

## 1. 確定事項と未確定事項

| 項目 | 判定 | 理由・必要な検証 |
|---|---|---|
| `skb_scrub_packet()`の`skb_ext_reset()`とXDP `data_meta`は別機構 | 確定（v28で根拠整理済み） | skb extensionの解除とmetadata headroom操作を混同しない |
| `skb_scrub_packet()`単体でXDP custom metadataを直接clearする | 否定（v28監査範囲） | ただし呼び出し元・後続処理による無効化は別問題 |
| `skb_metadata_clear()`のv6.16全callsite | **未検証** | 固定タグの全ソース検索が必要 |
| GRO後の各入力packet metadataの一対一対応 | 保証を置かない | aggregation経路別に追跡する必要あり |
| 通常VXLAN/Geneve decap後のXDP metadata有効性 | **未検証** | `iptunnel_pull_header()`、`__skb_tunnel_rx()`、再投入経路の順に追跡 |
| 2026-09 BPF skb extension seriesのmainline merge | **未検証** | 提案パッチとmerge済みcommitを区別 |

## 2. 固定タグで実行するソース監査コマンド

以下は**未実行の再現手順**であり、実行結果を意味しない。

```bash
git clone --filter=blob:none https://github.com/torvalds/linux.git
cd linux
git checkout v6.16
# 定義・呼び出し元を全ソースから抽出
git grep -n -F 'skb_metadata_clear(' -- ':!Documentation'
git grep -n -E 'skb_metadata_(set|differs|clear)\(' -- include net drivers
# 通常受信経路とGRO経路を別々に追跡
git grep -n -E 'skb_scrub_packet\(|__skb_tunnel_rx\(|iptunnel_pull_header\(' -- include net
git grep -n -E 'vxlan_gro_receive\(|geneve_gro_receive\(|skb_gro_receive\(' -- net
# 実装位置とgit履歴の照合
git log --oneline --all -S'skb_metadata_clear' -- include net drivers
```

抽出後は各callsiteについて、呼び出し関数、呼び出し条件、`skb->data_meta`と`skb->data`の前後関係、GRO/decap/再投入のどの境界か、`v6.16`以降の差分を表形式で記録する。

## 3. 検証で特に注意する点

1. `skb_metadata_clear()`と`skb_ext_reset()`を同一視しない。
2. `skb->mark`等の標準フィールドに転記された情報は、元のcustom metadataとは別の寿命を持つ。
3. GROでskbを集約することと、通常のVXLAN/Geneve decapsulationを混同しない。
4. cross-netnsでの`xnet=true`とsame-netnsでの`xnet=false`を区別する。
5. proposal（net-next patch series）をmainline release済みと扱わない。

## 4. 一次資料の参照先

- Linux v6.16固定タグ： https://github.com/torvalds/linux/tree/v6.16
- skbヘッダ： https://github.com/torvalds/linux/blob/v6.16/include/linux/skbuff.h
- skb実装： https://github.com/torvalds/linux/blob/v6.16/net/core/skbuff.c
- XDP RX metadataドキュメント： https://www.kernel.org/doc/html/latest/networking/xdp-rx-metadata.html
- 2026-09-10 net-next提案： https://lists.openwall.net/netdev/2026/09/10/367

**r69 canonical ledgerへのrelease attributionは、固定タグのcallsite確認後に行う。**

## 5. 追加監査（2026-10-08）：`skb_metadata_clear()`とLWT境界

### 5.1 確認できた実装上の意味

`include/linux/skbuff.h`の実装では、`skb_metadata_clear(skb)`は`skb_metadata_set(skb, 0)`を呼び、`skb_shinfo(skb)->meta_len`を0に設定する。metadataバイト列の消去ではなく、長さの無効化である。**参照した公開ソースは固定タグv6.16ではなく、mainlineの特定commitのスナップショット**であるため、v6.16の確定事項とはしない。

- ソース：https://code.googlesource.com/linux/torvalds/linux/+/f24ba334afafc70c3149e9db9c0cf8ecc6d52a09/include/linux/skbuff.h

### 5.2 LWTにおける明示的なmetadata無効化

`net/core/lwtunnel.c`の公開ソースでは、`lwtunnel_xmit()`と`lwtunnel_input()`が、対応するencapsulation operationの呼び出し前に`skb_metadata_clear(skb)`を実行する。コメントは「Encap pushes outer headers over the metadata; drop it.」であり、outer headerのpushによりmetadata領域が重なるため、無効化するという意図が明記されている。これは通常のVXLAN/Geneve UDP受信経路やGRO経路をそのまま説明するものではない。

- ソース：https://code.googlesource.com/linux/torvalds/linux/+/4bf54e47525dcacd4c6cdd97fb5902592414dd7a/net/core/lwtunnel.c
- 関連する修正提案：https://www.spinics.net/lists/netdev/msg1181105.html （2026-04-28）

### 5.3 `skb_data_move()`の例外的な無効化

公開ソースの`skb_data_move()`は、metadataが期待した位置にない、またはheadroomが不足する条件で`skb_metadata_clear()`を実行する。逆に条件が満たされれば、metadataをpacket dataとともに移動させる設計である。このため「skbのpush/pullで必ずmetadataが消える」という説明は不正確。

- ソース：https://code.googlesource.com/linux/torvalds/linux/+/f24ba334afafc70c3149e9db9c0cf8ecc6d52a09/include/linux/skbuff.h

### 5.4 証拠の限界と次の実行項目

今回、作業環境からGitHubへのDNS解決に失敗し、`v6.16`ソース全体の取得・`git grep`は実行できなかった。従って、**全callsiteを列挙した、v6.16に存在する、という主張は未検証**である。上記は公開された個別ソースの実装確認に限定する。v6.16の取得が可能になった時点で、Section 2のコマンドを実行し、ファイル名・行番号・条件分岐・git commitを確定すること。

| 項目 | 今回の判定 |
|---|---|
| `skb_metadata_clear`が`meta_len=0`にする | 公開ソースで確認、v6.16固定未検証 |
| `lwtunnel_input/xmit`の明示的clear | 公開ソースで確認、v6.16固定未検証 |
| `skb_data_move`の失敗時clear | 公開ソースで確認、v6.16固定未検証 |
| 全callsite列挙 | 未完了 |
| VXLAN/Geneve通常decapでのmetadata最終状態 | 未完了 |
| GRO metadata差異の全分岐 | 未完了 |


## v31 追補：通常のVXLAN受信とLWT metadata境界（2026-10-08）

### 監査対象と証拠水準

本追補は通常のVXLAN UDP受信経路と、2026年のLWT metadata修正系列を切り分ける。**v6.16 tagの全ファイルを取得して横断検索した結果ではない**ため、全callsite網羅とは記さない。

### VXLAN通常受信（GRO callbackとは別）

`vxlan_rcv()` / `vxlan_udp_encap_recv()` は通常UDP tunnel受信の入口であり、VXLAN header処理後に `iptunnel_pull_header()` と `skb_scrub_packet()` の経路を経てinner Ethernet frameを処理する。2016年のpatchではscrubを `vxlan_rcv()` から `iptunnel_pull_header(..., xnet)` へ移す差分が示されている。**古いコード断片のscrub位置を現在のコードへそのまま当てはめてはいけない**。

- https://lists.openwall.net/netdev/2016/02/18/94
- https://lists.openwall.net/netdev/2016/02/18/95
- https://code.googlesource.com/linux/torvalds/linux/+/f155f3b3ed1af23884ffaffe8a669722b87ac9d6/drivers/net/vxlan/vxlan_core.c

### LWT修正のrelease attributionに関する注意

2026-04-28の最初の案は `ip_finish_output2()` と `ip6_finish_output2()` で `skb_metadata_clear()` を呼ぶ構成。2026-06-19のv3は、代わりに `net/core/lwtunnel.c` の `lwtunnel_output()`、`lwtunnel_xmit()`、`lwtunnel_input()` に移している。**この二つを同時に採用された実装として扱わない**。提示されたmainlineスナップショットでは後者のコードが確認できるが、v6.16への帰属は未検証。

- https://lists.openwall.net/netdev/2026/04/28/506
- https://lists.openwall.net/netdev/2026/06/19/187
- https://code.googlesource.com/linux/torvalds/linux/+/4bf54e47525dcacd4c6cdd97fb5902592414dd7a/net/core/lwtunnel.c

### XDP metadataの意味論

`skb_metadata_clear()` は `meta_len` を無効化する操作であり、領域のゼロ埋めを意味しない。`skb_scrub_packet()` は別の目的（routing, conntrack, skb extension等のリセット）を持つ。従って、通常のVXLAN/Geneve受信でXDP custom metadataが**必ず保持される／必ず消去される**という両極端な記述を避け、headroom配置・meta_len・後続のpush/pull・GRO・BPFの参照可能性を別々に監査する。

### 未解決（v32候補）

1. Linux v6.16固定tagで `skb_metadata_clear(` の全callsiteをソース全体から列挙し、呼び出し元の条件と根拠commitを記録。
2. `iptunnel_pull_header()`、`vxlan_rcv()`、`geneve_udp_encap_recv()` を同一tagで突き合わせ、xnetの算出位置を比較。
3. 通常受信・GRO・forwarding・LWT encapの4経路を同一パケットで追い、`meta_len` の変化をselftestで実証。
4. LWT修正の最初のmerge commitとstable backport状況を特定。提案メールの投稿時期とmainline導入versionを混同しない。

## v32 追補：固定tag取得の再試行と監査可能性（2026-10-08）

### 今回の取得結果

Linux v6.16の `net/core/skbuff.c` を `raw.githubusercontent.com/torvalds/linux/v6.16/net/core/skbuff.c` から取得する操作を再試行したが、実行環境ではHTTP status `000`、転送0バイトで失敗した。Web経由の同tagの `lwtunnel.c`、`geneve.c`、`vxlan_core.c` もcache missとなった。**この環境でv6.16固定tagのcallsite網羅を実証したとは扱わない。**

### 根拠の区分と訂正

- `skb_metadata_clear()` が `meta_len` を無効化する、という既存の記述は、以前に確認した公開ソースの範囲に留まる。今回v6.16で再確認できていない。
- 2026年のLWT修正は、v6.16（2025年リリース）より後の提案である。**2026年の修正がv6.16に存在すると仮定してはならない。** v6.16 baselineと2026年の変更後コードを別表に分けて扱う。
- `skb_scrub_packet()` の `xnet` 差異、`skb_metadata_clear()`、`skb_ext_reset()` は別々の操作であり、ある関数の呼び出しだけからcustom XDP metadataの後段での可視性を断定しない。
- v31に記したVXLANのscrub位置は2016年のpatchと個別ソースsnapshotに基づくため、**v6.16の確定call graphではない**。

### 次の検証手順（実行待ち）

```bash
# Linux kernel sourceを取得できる環境で実施
 git clone --filter=blob:none --depth=1 --branch v6.16 \
   https://github.com/torvalds/linux.git linux-v6.16
 cd linux-v6.16
 git grep -n -F 'skb_metadata_clear(' -- ':!Documentation'
 git grep -n -E 'skb_scrub_packet\(|iptunnel_pull_header\(' -- \
   net/core net/ipv4 net/ipv6 drivers/net
 git grep -n -E 'geneve_udp_encap_recv|vxlan_udp_encap_recv|vxlan_rcv' -- \
   drivers/net net
```

callsite表には「定義・呼び出し・macro展開の別」「実際の条件分岐」「同一tagのcommit hash」「XDP `data_meta`への影響」を列記する。2026年LWT修正は別の導入commit/tagで再検証する。

### Phase 1の現在の判定

**保留**：v6.16全callsite、VXLAN/Geneve同一tagの通常decapsulation比較、LWT修正の導入commit。未取得のソースについて網羅性・release attributionを主張しない。Phase 2へ移行する場合も、これらを未解決項目として維持する。

## Phase 2 / Control-plane scalability: RTNL lock migration (2026-10-08 audit)

### Scope and status

This section separates three mechanisms: (1) global RTNL serialization, (2) per-network-namespace RTNL mutexes, and (3) per-netdev instance locks. The introduction of (2) is not evidence that operations in separate namespaces are already parallel: during migration `rtnl_net_lock(net)` still acquires global `rtnl_lock()` before `__rtnl_net_lock(net)` (source checked in upstream `net/core/rtnetlink.c` master; exact release attribution pending).

| Layer | Mechanism | Locking / compatibility point | Evidence |
|---|---|---|---|
| Global | `rtnl_lock()` | Historical global serialization; remains relevant to most NDO paths | https://www.kernel.org/doc/html/v6.13/networking/netdevices.html |
| Per-netns | `rtnl_net_lock(net)`, `__rtnl_net_lock(net)` | Nested migration under global RTNL; multiple netns may need deterministic lock ordering | https://github.com/torvalds/linux/blob/master/net/core/rtnetlink.c ; https://netdev.bots.linux.dev/netconf/2024/notes.html |
| Per-device | `netdev_lock()`, `netdev_unlock()`, `request_ops_lock` | Driver callback synchronization; do not equate with per-netns RTNL | https://www.kernel.org/doc/html/v7.2/networking/netdevices.html |
| ethtool | `ethtool_ops::op_needs_rtnl` | For ops-locked drivers, selected operations may still explicitly need RTNL | https://www.kernel.org/doc/html/v7.2/networking/netdevices.html |
| Queue leasing | Two instance locks | Virtual device lock must precede physical device lock | https://www.kernel.org/doc/html/v7.2/networking/netdevices.html |

### Verified transition design (not a completed migration)

Netconf 2024 notes describe a phased transition: first nest per-netns locking under the global lock, then remove the old lock only once call sites have migrated. The notes explicitly flag operations spanning multiple namespaces. Upstream `rtnetlink.c` provides `rtnl_net_cmp_locks()` and a namespace lock ordering helper, with `init_net` ordered first. This is a lock-ordering/deadlock-avoidance problem, not simply an API rename.

The v7.2 netdevice documentation says ops-locked drivers may have `ethtool_ops` executed only under instance lock, but callbacks that need RTNL must opt in using `op_needs_rtnl`. `NETDEV_UNREGISTER` is documented as occurring without the instance lock, so notifier lifetime rules must be audited separately.

### Pending exact-source audit

- Compare `net/core/rtnetlink.c`, `include/linux/rtnetlink.h`, `include/linux/netdevice.h`, and `Documentation/networking/netdevices.rst` at v6.13, v6.16, v7.1, v7.2 and current mainline.
- Extract the first mainline commits introducing per-netns RTNL fields and helper functions; do not infer from documentation publication dates.
- Trace `RTM_NEWLINK`/`RTM_DELLINK` across namespaces, netdev unregister, ethtool GET/SET and netdev generic-netlink queue management with exact lock acquisition order.
- Record `CONFIG_DEBUG_NET_SMALL_RTNL` implications and avoid assuming configuration-independent parallelism.
- Study YNL independently: generated Netlink bindings are not, by themselves, a locking or concurrency change.

**Evidence grade:** Documentation and current upstream code support the mechanism and transition design. Per-release landing commits and full function call graphs are **pending**, so no new exact-version claims should be inserted into the canonical Part VII ledger yet.


## Phase 2 / RTNL operation call-path audit (v34, 2026-10-08)

### Source scope and evidence discipline

This appendix compares the **current upstream `master`** implementations visible in the linked source pages, rather than treating them as v6.16 or v7.2 implementations. The earlier historical `rtnetlink.c` snapshot in the second link demonstrates why the dispatch and lock model must not be backported by assumption. Exact introduction commits and per-release diff remain pending.

- Current upstream: https://github.com/torvalds/linux/blob/master/net/core/rtnetlink.c
- Older immutable source snapshot: https://code.googlesource.com/linux/torvalds/linux/+/e557793799c5a8406afb08aa170509619f7eac36/net/core/rtnetlink.c
- ethtool implementation snapshot: https://code.googlesource.com/linux/torvalds/linux/+/2646fb032f511862312ec8eb7f774aaededf310d/net/ethtool/netlink.c
- RTNL declarations: https://github.com/torvalds/linux/blob/master/include/net/rtnetlink.h

### RTM_NEWLINK and RTM_DELLINK (current master)

The current handler table registers `RTM_NEWLINK` with `.doit = rtnl_newlink` and `RTNL_FLAG_DOIT_PERNET`, while `RTM_DELLINK` has `.doit = rtnl_dellink` and `RTNL_FLAG_DOIT_PERNET_WIP`. **These distinct flags are direct evidence that the migration status is not uniform across link operations.** They do not alone prove that two namespaces can execute those handlers simultaneously; dispatch and underlying global locking must also be audited.

`RTM_NEWLINK` (simplified):

```text
NETLINK_ROUTE request
  -> rtnetlink_rcv / rtnetlink_rcv_msg (dispatcher; lock context to verify per tag)
  -> rtnl_newlink
     -> parse IFLA_* / link kind / target, link and peer netns
     -> rtnl_nets_lock(...) (namespace set; branch-dependent)
     -> existing link: rtnl_changelink -> do_setlink
     -> new link: rtnl_newlink_create -> rtnl_link_ops::newlink
     -> unlock / release namespace references
```

`RTM_DELLINK` (source-confirmed inner path):

```text
NETLINK_ROUTE request
  -> rtnl_dellink
     -> parse attributes / optional IFLA_TARGET_NETNSID
     -> rtnl_net_lock(tgt_net)
     -> find device by ifindex or name
     -> rtnl_delete_link
        -> rtnl_link_ops::dellink(dev, &list_kill)
        -> unregister_netdevice_many_notify(...)
     -> rtnl_net_unlock(tgt_net)
```

A group delete instead calls `rtnl_group_dellink`, which iterates devices, invokes each link kind's `dellink`, and unregisters a batch. The **device lifecycle/unregister notifier** is a separate concern from simply selecting the per-netns mutex.

### RTM_SETLINK and netns lock set

Current `rtnl_setlink` parses the target namespace, constructs `rtnl_nets`, adds the source and target namespace, acquires `rtnl_nets_lock`, resolves the device, and calls `do_setlink`. Thus a cross-namespace operation cannot be represented accurately as taking only a single target-netns mutex. Lock order and any compatibility global lock require per-tag verification.

### ethtool netlink GET and SET

The ethtool generic-netlink implementation has `ethnl_default_doit` for default GET operations. It selects a request descriptor, parses the header and device, prepares reply data, and calls request-specific handlers. This is **not** the same dispatch path as `NETLINK_ROUTE`'s `RTM_*LINK` handlers. The locking contract is influenced by `request_ops_lock` and operation-specific `op_needs_rtnl`, but one cannot infer that all ethtool GET/SET operations are RTNL-free from the presence of a device instance lock. A complete SET path needs the specific `ethnl_set_*` family operation and fixed-tag source audit.

### netdev generic-netlink / YNL boundary

`netdev` generic-netlink queue configuration belongs to a different Generic Netlink family and dispatch path. The YNL generated client/specification layer defines encoding, parsing and API exposure; it does **not** itself eliminate global RTNL or establish a lock-order guarantee. Source verification of queue-specific `doit` handlers and lock acquisition remains open.

### Open items (not canonical ledger claims)

1. Extract dispatcher-level `rtnetlink_rcv_msg` locking conditions and precise meaning of `RTNL_FLAG_DOIT_PERNET` versus `RTNL_FLAG_DOIT_PERNET_WIP` at fixed tags.
2. Compare `v6.13`, `v6.16`, `v7.1`, `v7.2` with source hashes and identify first landing commits for each flag and lock helper.
3. Trace ethtool GET/SET separately, including request-specific locking and driver callback execution.
4. Audit netdev Generic Netlink queue operations and YNL specs independently of RTNL.
5. Validate netdev unregister notifier context, multi-netns lock ordering, and `CONFIG_DEBUG_NET_SMALL_RTNL` impact.

**Assessment:** Current-upstream inner call paths are supported by source; dispatcher locking and historical release attribution remain **pending**. Do not promote these entries to Part VII as version-certified facts yet.

## Phase 2 supplemental audit: rtnetlink dispatcher flags (2026-10-08)

### Confirmed from current upstream source (not version-attributed)

In `include/net/rtnetlink.h`, `RTNL_FLAG_DOIT_PERNET` and `RTNL_FLAG_DOIT_PERNET_WIP` are both aliases of `RTNL_FLAG_DOIT_UNLOCKED` (`BIT(0)`). They do **not** themselves instruct the dispatcher to acquire a per-netns mutex.

In `net/core/rtnetlink.c`, `rtnetlink_rcv_msg()` checks `flags & RTNL_FLAG_DOIT_UNLOCKED`. For this branch it drops the RCU read lock and invokes the handler without acquiring global `rtnl_lock()`. In the other branch, it acquires global RTNL before invoking the handler. Per-netns synchronization, where required, is performed by the individual handler. Therefore the presence of `DOIT_PERNET` / `DOIT_PERNET_WIP` on a handler registration is not proof of complete per-netns concurrency or global RTNL removal.

`RTM_NEWLINK` is registered with `RTNL_FLAG_DOIT_PERNET`; `RTM_DELLINK` and `RTM_SETLINK` use `RTNL_FLAG_DOIT_PERNET_WIP` in the inspected current tree. These are descriptions of registration, not release-attribution claims.

### Source anchors

- https://github.com/torvalds/linux/blob/master/include/net/rtnetlink.h
- https://github.com/torvalds/linux/blob/master/net/core/rtnetlink.c
- Immutable source snapshot of the header: https://code.googlesource.com/linux/torvalds/linux/+/4edcdefd4083ae04b1a5656f4be6cd83ae919ef4/include/net/rtnetlink.h

### Outstanding

- Verify handler-local locking for `rtnl_newlink()`, `rtnl_dellink()`, `rtnl_setlink()` with pinned release tags.
- Attribute the flag aliasing and handler transitions to exact mainline commits and releases before changing the canonical Part VII ledger.
- Compare `CONFIG_DEBUG_NET_SMALL_RTNL` enabled/disabled builds, since `rtnl_net_lock()` can still take global RTNL during migration.

## Phase 2 addendum: v6.13 merge-window attribution audit (2026-10-08)

### Confirmed release boundary

The networking pull request for Linux v6.13 (2024-11-19) explicitly lists:

- `rtnetlink: Convert RTM_NEWLINK to per-netns RTNL.`
- `rtnetlink: Register rtnl_dellink() and rtnl_setlink() with RTNL_FLAG_DOIT_PERNET_WIP.`
- `rtnetlink: Introduce struct rtnl_nets and helpers.`
- `rtnetlink: Define RTNL_FLAG_DOIT_PERNET for per-netns RTNL doit().`

Evidence: https://lkml.iu.edu/2411.2/03002.html and https://lists.openwall.net/linux-kernel/2024/11/19/972

**Interpretation:** v6.13 is a substantiated integration boundary for these changes, *not* a completion date for per-netns locking. `RTNL_FLAG_DOIT_PERNET` and `RTNL_FLAG_DOIT_PERNET_WIP` are aliases of `RTNL_FLAG_DOIT_UNLOCKED`, so the dispatcher skips its usual global lock and the handler must implement appropriate locking.

### Why `WIP` was retained

The November 2024 patch discussion explains that `rtnl_setlink()` may need to propagate operations to upper devices in other network namespaces and that `rtnl_dellink()` may remove related devices. The upper bound on involved namespaces is not known in advance. Therefore a simple single-netns locking scheme is insufficient.

Primary evidence: https://lists.openwall.net/netdev/2024/11/06/193 and patch diff https://lists.openwall.net/netdev/2024/11/05/33

### Call-path interpretation and limitations

```text
NETLINK_ROUTE request
  -> rtnetlink_rcv()
  -> netlink_rcv_skb(..., rtnetlink_rcv_msg)
  -> rtnetlink_rcv_msg()
      -> RTNL_FLAG_DOIT_UNLOCKED set: invoke handler without dispatcher global RTNL
      -> otherwise: acquire global rtnl_lock(), invoke handler, unlock
  -> rtnl_newlink() / rtnl_dellink() / rtnl_setlink()
      -> handler-local rtnl_nets_lock() or rtnl_net_lock() as appropriate
```

This is a **conceptual** graph for the examined implementations; the exact handler-local branches must be verified for each release tag. In particular, a per-netns API call does not itself prove global-lock-free parallel execution because transition-era helpers may also acquire global RTNL.

Upstream reference: https://github.com/torvalds/linux/blob/master/net/core/rtnetlink.c and https://github.com/torvalds/linux/blob/master/include/net/rtnetlink.h

### Version evidence matrix

| Kernel | Supported conclusion | Confidence / outstanding |
| --- | --- | --- |
| v6.13 | v6.13 networking merge window included RTM_NEWLINK per-netns conversion and SETLINK/DELLINK WIP registration | A for merge-window attribution; exact individual commit IDs not yet extracted |
| v6.16 | Expected to contain v6.13 merged changes, subject to any later revert | B; tag-specific inspection pending |
| v7.1 | Do not infer complete per-netns migration from handler flag names | B; tag-specific locking audit pending |
| v7.2 | Do not infer complete per-netns migration from handler flag names | B; tag-specific locking audit pending |

### Reproducible verification commands

```bash
for tag in v6.13 v6.16 v7.1 v7.2; do
  echo "=== $tag ==="
  git -C linux show "$tag:include/net/rtnetlink.h" | grep -A8 'enum rtnl_link_flags'
  git -C linux show "$tag:net/core/rtnetlink.c" | grep -nE 'rtnetlink_rcv_msg|RTNL_FLAG_DOIT_PERNET|rtnl_nets_lock|rtnl_net_lock'
done
# Resolve exact landing commits in an upstream Linux checkout:
git -C linux log --all --oneline --grep='Convert RTM_NEWLINK to per-netns RTNL'
git -C linux log --all --oneline --grep='Register rtnl_dellink() and rtnl_setlink() with RTNL_FLAG_DOIT_PERNET_WIP'
```

**Canonical ledger status:** Do not add exact SHAs or v7.x completion claims until verified against the upstream release tags. This addendum corrects the historical interpretation but does not change r69 Part VII itself.
