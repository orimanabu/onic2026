---
title: Linux Networking Evolution
---

# Linux Networking Evolution — Linux v3.0 から 7.x まで

> **Clean canonical edition.** Part I presents the thesis, Part II the architecture eras, Parts III–V the lineages / driver-framework / observability story. Part VI is the only normative release chronology, Part VII the provenance ledger, and Appendix material is supporting evidence rather than an alternate release map.

**調査基準日:** 2026-10-02  
**構成改訂:** 2026-10-04（r24）

この文書は、Linux networking の変化を「調査した順」ではなく、 **kernel networking がどのように進化したかを読む順序**に再構成した版である。

**対象範囲:** wired / host networking を中心に扱う。RDMA、Wi-Fi 全般、QUIC や security protocol の網羅的な歴史は対象外とし、mac80211 は queue management （airtime / AQL）の観点から選択的に扱う。

## この文書の読み方

本編は **thesis → architecture map → era → lineage → driver framework / observability → conclusion** の順で読む。release 番号を調べる場合は Part VI、 commit-level provenance を再監査する場合は Part VII へ直接進んでよい。

Part I–V の version 表記は説明上の参照であり、release attribution を独立に定義しない。 その正本は Part VI、exact anchor の台帳は Part VII である。詳細な evidence 用語と canonicalization rule は Part VI 冒頭で定義する。

### 図の記法

- `→` / `↓` — 時系列または shared design problem に沿う **lineage continuation**。直接依存を意味しない。
- `⇒` — 本文で明示した **direct extension / integration**。
- `∥` — 同時期の **parallel evolution**。
- `↔` — 相互作用するが親子関係ではない。

------------------------------------------------------------------------

# Part I — Executive thesis: Linux networking は何が変わったのか

Linux networking の15年間の変化は、単純な「高速化」でも、従来の `skb` path の置き換えでもない。 従来の **CPU が system RAM 上の skb を処理するモデルを現在も広く維持しながら**、 AF_XDP、device memory、io_uring zero-copy、programmable/offloaded path など、 複数の execution / memory-ownership model を workload と hardware capability に応じて **共存させられる architecture へ拡張した**ことが大きな変化である。

その変化は6つの軸で並行して進んだ。

``` text
PERFORMANCE       queueing / pacing / aggregation / zero-copy
PROGRAMMABILITY   BPF / XDP / socket / protocol / qdisc extension points
MEMORY            page_pool / netmem / memory providers / device memory
CONTROL PLANE     Netlink object model / YNL / RTNL scope reduction
OBSERVABILITY     typed tracing / drop reason / queue-NAPI-memory identity
DRIVER FRAMEWORK  switchdev / devlink / phylink / DIM / common driver contracts
```

これらのさらに上位にある横断的な傾向を、本書では **explicit resource / control contracts** と捉える。 すなわち driver-private、implicit、global だった仕組みを、kernel 共通の object、API、accounting、 assignment、lifetime、synchronization contract として明示する方向である。

### 本書で固定する語彙

- **上位 thesis:** explicit resource / control contracts
- **contract の具体形:** object / accounting / API / assignment / lifetime / synchronization
- **重要な subtheme:** ownership（特に queue / packet memory）
- **帰結:** reusable / programmable / observable な resource control

`ownership` はこの流れの重要な一部だが、すべてを ownership だけで説明しない。たとえば BQL は 主に byte accounting / backpressure、per-netns RTNL は synchronization scope の問題である。 一方 page_pool / AF_XDP / memory providers / queue leasing では、buffer や queue を誰が管理・割り当てるかという ownership が中心問題になる。

## Architecture evolution map

``` text
                    private / implicit / global
                              │
          ┌───────────────────┼────────────────────┐
          │                   │                    │
      queue state         packet execution      memory lifetime
      BQL / pacing        BPF / XDP / TC        page_pool / netmem
          │                   │                    │
          ├──────────────┬────┴──────────────┬─────┤
          │              │                   │     │
     device control   queue/NAPI objects   memory providers
   switchdev/devlink     netdev-genl       AF_XDP / devmem
          │              │                   │
          └──────────────┼───────────────────┘
                         ▼
              explicit common contracts
                         │
          object · accounting · API · lifetime · assignment · synchronization
          （各 contract が提供する capability は API ごとに異なる）
```

この図の矢印は個々の機能の直接的な派生関係を表さない。異なる subsystem が、 **暗黙的な状態を明示的な contract にする**という共通方向へ進んだことを示す。

## Snapshot: v3.0 / v5.0 / 2026

本書の起点は v3.0 である。v5.0 は第2の起点ではなく、XDP/AF_XDP、switchdev/devlink、 page_pool など programmable/offload/memory infrastructure がすでに成立した **中間観測点**として置く。

| 観点             | v3.0 前後                         | v5.0 前後                 | 2026                                           |
|:-----------------|:----------------------------------|:--------------------------|:-----------------------------------------------|
| PERFORMANCE      | driver-local tuning が大きい      | BQL/TSQ/fq/pacing が定着  | multi-layer control + cake_mq / scalable TX    |
| PROGRAMMABILITY  | classic BPF 中心                  | XDP/AF_XDP・TC BPF        | struct_ops / TCX / BPF qdisc / netkit          |
| MEMORY           | page-centric driver-local recycle | page_pool が利用可能      | netmem / memory providers / device memory      |
| DRIVER FRAMEWORK | net_device 中心                   | switchdev/devlink/phylink | queue/NAPI objects・YNL-described APIs         |
| CONTROL PLANE    | global RTNL 依存が大きい          | global RTNL が依然中心    | per-netns / fine-grained / RTNL-less paths     |
| OBSERVABILITY    | logs/counters/tracing の個別利用  | BPF/BTF ecosystem が拡大  | typed drop reason + queue/NAPI/memory identity |

## 6軸・contract・cross-cutting domain の対応

6軸は **architecture analysis の主軸**であり、全 milestone の排他的 taxonomy ではない。 TRANSPORT、VIRTUAL / OVERLAY、SECURITY は Axis と独立した **cross-cutting domain** である。 Domain は単に「その protocol のコードを通る」という意味ではなく、protocol semantics / path management / authentication / virtual-overlay behavior 自体が milestone の主題である場合に付与する。 したがって BIG TCP や Device Memory TCP は TCP path に存在しても、それだけを理由に TRANSPORT とはしない。

| Architecture axis | 15年間の変化                                                                               | 主な contract form                   | 本文の主な扱い                                                |
|-------------------|:-------------------------------------------------------------------------------------------|:-------------------------------------|:--------------------------------------------------------------|
| PERFORMANCE       | implicit queueing → explicit accounting / multi-layer scheduling / larger processing units | accounting / API                     | Part III「Queueing / latency / pacing」「Packet aggregation」 |
| PROGRAMMABILITY   | fixed path → verified extension points → reusable programmable infrastructure              | API / object                         | Part III「BPF」「XDP / AF_XDP」                               |
| MEMORY            | driver-local page handling → common lifetime / provider / assignment model                 | lifetime / assignment / object       | Part III「Packet memory」「io_uring networking」              |
| CONTROL PLANE     | embedded/global state → reusable objects / schema / narrower lock scope                    | object / API / synchronization       | Part III「Routing / Netlink / RTNL」「netfilter」             |
| OBSERVABILITY     | opaque outcome → typed reason / identity / correlation                                     | contract を検証する evidence layer   | Part V                                                        |
| DRIVER FRAMEWORK  | driver-local convention → reusable kernel framework / device-wide object                   | object / API / accounting / lifetime | Part IV                                                       |

**Cross-cutting domain:** TRANSPORT は Part III「TCP / UDP / transport」、 VIRTUAL / OVERLAY は Part III「Virtual networking」、SECURITY は本書の主対象ではなく、 kTLS / WireGuard / TCP-AO / XFRM offload のように他の lineage と交差する場合だけ補助ラベルとして扱う。

### thesis を検証する代表例

| Milestone                            | 以前の状態                                                 | 主に明示化したもの                               | Contract form         | Axis                             | Era   |
|--------------------------------------|------------------------------------------------------------|--------------------------------------------------|-----------------------|----------------------------------|-------|
| DQL/BQL                              | driver-local TX ring tuning                                | queued/completed byte accounting                 | accounting            | PERFORMANCE / DRIVER FRAMEWORK   | Era 1 |
| `bpf()` / maps / verifier            | fixed kernel behavior / ad-hoc attachment                  | verified program + map interface                 | API                   | PROGRAMMABILITY                  | Era 1 |
| switchdev / devlink                  | vendor-local forwarding / device state                     | forwarding/device/resource objects               | object                | DRIVER FRAMEWORK                 | Era 2 |
| page_pool                            | driver-local page recycle / DMA lifetime                   | allocation / recycle / DMA lifetime              | lifetime              | MEMORY                           | Era 2 |
| AF_XDP                               | kernel-centric queue/buffer handling                       | queue / UMEM binding                             | assignment + lifetime | MEMORY / PROGRAMMABILITY         | Era 2 |
| nexthop objects                      | route に埋め込まれた nexthop state                         | reusable nexthop object / group                  | object                | CONTROL PLANE                    | Era 3 |
| BPF `struct_ops`                     | protocol algorithm が built-in implementation 中心         | typed operations implemented by verified BPF     | API + object          | PROGRAMMABILITY                  | Era 3 |
| YNL                                  | handwritten Netlink family definition / client boilerplate | machine-readable Netlink schema                  | API                   | CONTROL PLANE                    | Era 4 |
| queue/NAPI netdev-genl               | opaque driver rings / NAPI instances                       | stable queue/NAPI identity and relations         | object                | DRIVER FRAMEWORK / OBSERVABILITY | Era 4 |
| Device Memory TCP                    | packet payload は system RAM 前提                          | non-system-RAM memory bound into networking path | assignment + lifetime | MEMORY                           | Era 4 |
| io_uring ZCRX / memory-provider path | socket receive と userspace buffer lifecycle の分離        | queue-bound receive memory provider              | assignment + lifetime | MEMORY                           | Era 4 |
| per-netns RTNL                       | global RTNL scope                                          | namespace-scoped synchronization                 | synchronization       | CONTROL PLANE                    | Era 4 |
| RX HW queue leasing                  | hardware queue が physical netdev に固定                   | hardware queue assignment / leasing              | assignment            | MEMORY / DRIVER FRAMEWORK        | Era 4 |

この分布は Era の違いも示す。Era 1 では accounting / API の基礎、Era 2–3 では reusable object / programmability、Era 4 では assignment / lifetime / synchronization の明示化が目立つ。ただし 各 Era は重なりを許し、この表は厳密な periodization ではない。

この表は release attribution の正本ではない。version と Evidence class は Part VI に従う。

### この thesis が説明しないもの

`explicit resource / control contracts` は本書が観察する強い architecture tendency であり、 Linux networking の全変更を説明する万能則ではない。TFO / DCTCP / BBR / MPTCP / PLB / AccECN のような **transport algorithm / protocol evolution**、GRO/GSO / BIG TCP のような **処理単位の拡大**、`dev_queue_xmit()` llist 化のような **implementation scalability** は、それ自体を contract 化として説明しない。これらは対応する architecture axis または cross-cutting domain の lineage で扱う。

Observability も7番目の contract form とはしない。typed identity、drop reason、tracepoint、 timestamp などは、resource/control contract と runtime behavior を理解・検証する **evidence layer** と位置付ける。

Part II–V はこの thesis を architecture の観点から検証し、Part VI–VII が release attribution と provenance を再監査可能な形で保持する。

# Part II — Four architecture eras

ここでの era は kernel major version の境界ではなく、**networking architecture が主に解こうとした問題**で区切る。 境界は重なり、ある era で始まった仕組みは後続 era でも継続して発展する。

| Era                                                | おおよその期間 | 主問題                                                                           | 代表的な milestone                                                                                                         |
|:---------------------------------------------------|:---------------|:---------------------------------------------------------------------------------|:---------------------------------------------------------------------------------------------------------------------------|
| **Era 1 — Queue & scalability foundation**         | ～2014頃       | queue backlog、multicore、namespace、overlay、filtering の基礎                   | BQL, CoDel/fq_codel, TSQ, pacing, namespace/setns, VXLAN, nftables, eBPF foundation                                        |
| **Era 2 — Programmable datapath**                  | 2015～2018頃   | packet path を安全に拡張し、hardware/offload と接続する                          | TC BPF, XDP, cgroup/LWT BPF, SOCKMAP, switchdev, devlink, AF_XDP, page_pool                                                |
| **Era 3 — Programmability becomes infrastructure** | 2019～2022頃   | programmable mechanism を protocol / operations / memory infrastructure に広げる | BTF ecosystem, struct_ops, SK_LOOKUP, MPTCP, BIG TCP, io_uring networking, drop reason, devlink health                     |
| **Era 4 — Explicit resource / control contracts**  | 2023～         | queue・memory・device・locking scope を明示的 object/contract として扱う         | netmem, memory providers, Device Memory TCP, netdev-genl queue/NAPI objects, queue leasing, per-netns RTNL, BPF qdisc, YNL |

この区分は release attribution の正本ではない。たとえば page_pool は Era 2 で利用可能になり、 Era 4 の memory-provider architecture の重要な基礎になる。per-netns RTNL も一度に完成した機能ではなく、 複数 release にわたる migration である。正確な release は Part VI を参照する。

## Era 間で変わった設計上の問い

``` text
How much should we queue?
        ↓
Where can we program the datapath?
        ↓
How do those mechanisms become reusable infrastructure?
        ↓
How explicitly can resources, ownership and synchronization be modeled?
        queue / memory / device / locking scope
```

この矢印も feature dependency ではなく、時代ごとの**支配的な設計課題の変化**を表す。

------------------------------------------------------------------------

# Part III — Long-term lineages

Part III では release 順ではなく、同じ設計課題が長期間にどう変化したかを追う。 図の矢印は、明示しない限り「直接の親子関係」を意味しない。dependency / extension / parallel development / shared design problem を区別する。release attribution の正本は Part VI である。

## Era 1 foundations — 6軸を初期問題へ投影した4つの見方

v3.x は release-by-release に読むより、後の Linux networking を支える4つの foundation が 並行して形成された時期として読む方が、本書の architecture-first な構成に合う。 個々の release attribution の正本は Part VI に置き、ここでは設計上の意味だけを追う。

ここでいう4 foundation は新しい分類体系ではなく、Era 1 の milestone を6軸から読み直すための編集上の束ね方である。

### Queue / scalability foundation

v3.x では「どこに、どれだけ packet / byte を滞留させるか」を複数レイヤーで制御する仕組みが整った。

この foundation は **socket / qdisc / driver-NIC の3層が同じ queue-pressure 問題を別々に扱い始めたこと**を指す。詳細な3層図と後続の BBR / EDT / CAKE / AQL / cake_mq は、直後の `Queueing / latency / pacing` に集約する。

Era 1 で重要なのは、BQL、CoDel/fq_codel、TSQ、sch_fq/TCP pacing が単一の後継系列ではなく、queueing / latency / burst control を異なる層で扱う foundation を形成したことである。

### Virtualization / control foundation

namespace を file descriptor と `setns()` で操作できるようになったことにより、 network namespace は container / orchestration から利用しやすい control primitive になった。 同時期に VXLAN、ipvlan などが加わり、host 内外の virtual network topology を構成する building block が増えた。

``` text
namespace / setns
      │
      ├── host-local isolation and placement
      │
      ├── VXLAN ── overlay reachability
      │
      └── ipvlan ─ interface multiplexing
```

IPv4 route-cache removal や nftables も、従来の global/cache-heavy または個別 subsystem 的な設計から、 より明示的な lookup / rule / object model へ向かう control-plane foundation として重要である。

### Programmability foundation

classic BPF から eBPF への変化は、一度に「modern BPF」が完成した出来事ではない。

``` text
classic BPF
    │
    ▼
eBPF ISA redesign
    │
    ▼
bpf() + maps + verifier
    │
    ▼
socket attachment
```

ここで成立したのは、後の TC BPF、XDP、cgroup/LWT hooks、SOCKMAP/SOCK_OPS、 AF_XDP、SK_LOOKUP、struct_ops、netkit、BPF qdisc へ展開できる **verified in-kernel programmability の基礎 contract** である。

以後の BPF history は一本の attachment-point lineage ではない。packet/datapath、 socket/lookup、protocol algorithms、virtual-device/queue control が独立・並行して拡大する。 詳細は後述の BPF / XDP lineage で扱う。

### Server / transport foundation

`SO_REUSEPORT`、`SO_BUSY_POLL`、TCP Fast Open、DCTCP などは、 multicore server、low latency、connection establishment、datacenter congestion control という 異なる問題に対する基礎を形成した。

Era 1 では TFO と DCTCP が、それぞれ connection establishment と datacenter congestion feedback の 基礎 milestone となった。後続の BBR / MPTCP / AccECN は `TCP / UDP / transport`、 BIG TCP は `Packet aggregation` で扱い、ここでは Era 1 を越える transport timeline を再掲しない。

### v3.x から後続 era へ

v3.x で重要なのは個々の feature 数ではなく、後の architecture を可能にする **accounting、namespace/control、verified programmability、multi-layer queue control** が kernel 共通 primitive として揃い始めたことである。

Part III の以下の節では、Part VI の milestone が**どの設計課題を共有し、 どこで dependency / extension / parallel evolution / shared design problem の関係にあるか**を説明する。 release ごとの provenance は繰り返さない。driver-specific な共通化は Part IV、 tracing/tooling の詳細は Part V で扱う。

## Queueing / latency / pacing — multi-layer control evolution

PERFORMANCE 軸の queueing lineage は、単に qdisc algorithm が増えた歴史ではない。 socket が作る burst、qdisc が保持する backlog、driver/NIC ring に積まれた outstanding bytes、 さらに multiqueue NIC での queue 選択を、それぞれ別レイヤーで制御できるようになった歴史である。

``` text
socket / transport:  TSQ · TCP pacing · BBR
                          ∥
qdisc / scheduler:   CoDel · fq_codel · sch_fq · EDT · CAKE / cake_mq
                          ∥
driver / NIC:        DQL/BQL · multiqueue scaling · dev_queue_xmit() llist
```

DQL/BQL は driver が NIC へ渡した byte と完了した byte を accounting し、hardware queue に過剰な backlog を作らないための backpressure contract を与える。TSQ は同じ問題を TCP socket 側から制限し、 `sch_fq` と TCP pacing は「何個 queue に入れるか」だけでなく「いつ送るか」を扱う。

CoDel/fq_codel と CAKE は qdisc 層で delay / fairness / shaping を扱う。EDT と `SO_TXTIME` は time-based transmission を明示し、taprio は time-aware scheduling を hardware/offload と接続する。 これらは同じ API の世代交代ではなく、queueing problem を複数層へ分解した結果である。

Wi-Fi/mac80211 では airtime accounting/scheduling と AQL が、byte queue だけでは表現しにくい wireless medium の占有時間を明示的に扱う。これは wired BQL の単純な派生ではないが、 **hidden queue pressure を accounting 可能な量へ変換する**という shared design problem を持つ。

6.19 の `dev_queue_xmit()` llist 化は contract 明示化ではなく、shared-qdisc / multiqueue TX の implementation scalability を改善する milestone である。7.0 の `cake_mq` は CAKE を multi-queue-aware に拡張し、modern NIC の queue topology と qdisc control の接点を強める。

## Packet aggregation — GRO/GSO → BIG TCP

v5.0 ですでに GRO/GSO/TSO は成熟していたが、高速 NIC では per-packet metadata processing が支配的になる。

``` text
wire packets
    ↓ GRO
large skb
    ↓ GSO/TSO
wire packets
```

v5.19 BIG TCP は kernel internal GRO/GSO aggregate の 64KiB 制約を緩和した。

``` text
GRO/GSO
  ↓
IPv6 BIG TCP (5.19)
  ⇒ IPv4 BIG TCP (6.3)
  ⇒ IPv6 BIG TCP without synthetic HBH jumbo header (Linux 7.0)
  ⇒ BIG TCP over VXLAN / GENEVE (7.3-rc/mainline)
```

PPPoE GRO/GSO のような encapsulation-specific aggregation も、同じ PERFORMANCE 軸で per-packet cost を減らすが、BIG TCP の派生機能ではない。

BIG TCP は wire MTU を巨大化する機能ではなく、 **kernel 内部の packet-processing unit を大きくする機能**として理解する。利用可否と効果は protocol path、 GRO/GSO/offload capability、driver/NIC、tunnel implementation などに依存し、すべての device / path で 一律に大きな aggregate を利用できることを意味しない。

------------------------------------------------------------------------

## Packet memory — page_pool → netmem → memory providers / device memory

packet memory の変化は「zero-copy が増えた」という一語では足りない。重要なのは、 **allocation / recycling / lifetime / DMA mapping / queue binding を誰が管理するか**が driver-private な慣習から common contract へ移ってきたことである。

### page_pool — RX memory lifecycle の共通化

従来の RX driver は page allocation、DMA mapping、recycle を個別に実装しがちだった。 page_pool は RX/XDP 向けに page の再利用と DMA lifecycle を共通化し、driver と networking core の 間に reusable lifetime contract を作る。

``` text
old RX:
NIC → allocate/map page → stack → unmap/free

page_pool:
NIC → recycled/mapped page → stack ─┐
      ↑                             │
      └──────── return/recycle ─────┘
```

page_pool は XDP 専用ではない。XDP と強く結び付いて普及したが、後の zero-copy RX、 memory-provider、device-memory path が driver ごとの独自 allocator に戻らずに済むための common infrastructure として読む。

### netmem — `struct page` と packet memory を同一視しない

system RAM の page だけを packet backing と仮定すると、device memory や userspace-owned memory を 同じ networking path で扱いにくい。`netmem` は networking が扱う memory identity を `struct page` そのものから切り離す方向を表す。

``` text
network packet memory
          ↓
        netmem
       /      \
system RAM   non-page / device-backed memory
```

ここで重要なのは「RAM を device memory に置換した」ことではなく、従来 path を維持したまま 異なる backing memory を表現できる contract を追加したことである。

### memory providers / queue binding — memory lifetime と RX queue を接続する

io_uring ZCRX や Device Memory TCP では、memory pool を用意するだけでは不十分である。 どの RX queue がどの memory provider を使うか、buffer がいつ application/device から返却されるか、 queue reconfiguration 中に lifetime をどう保つかを kernel/driver/userspace 間で合意する必要がある。

``` text
RX queue
   ↔ memory provider
   ↔ page_pool / netmem-backed buffers
   ↔ userspace or device consumer
```

このため Era 4 では MEMORY 軸が `lifetime` だけでなく `assignment` と強く結び付く。 netdev-genl queue objects、io_uring ZCRX、Device Memory TCP、RX HW queue leasing は 同一機能ではないが、**queue identity と memory ownership/lifetime を明示的に結び付ける** という architecture problem を共有する。

Device Memory TCP は system RAM を経由しない RX/TX path を可能にする代表例である。

``` text
traditional:
NIC → system RAM → CPU/copy/mapping → GPU/accelerator

device-memory path:
NIC ─────────────→ device memory → GPU/accelerator
```

large RX buffers や `>PAGE_SIZE` devmem buffer の拡張は、この model が fixed-size page assumption から 離れていく実装上の進展として位置付ける。

DIBS は shared-memory communication 側の別 lineage であり、この packet-memory contract の 直接の predecessor/successor ではない。本書の canonical packet-networking story には昇格させない。

## io_uring networking

6.0 の SEND_ZC / multishot receive と 6.15 の ZCRX を同じ lineage で扱い、TCP 節には重複記載しない。

TCP `MSG_ZEROCOPY` は send-side copy avoidance の先行 milestone であり、後の io_uring `SEND_ZC` と 同一 API ではないが、userspace↔kernel の data movement cost を減らす PERFORMANCE lineage の 重要な前段として扱う。

``` text
sendmsg / recvmsg
      ↓
zero-copy TX
      ↓
multishot / registered buffers
      ↓
zero-copy RX (6.15)
      ↓
memory-provider / device-memory integration
```

目標は syscall reduction だけではなく、 NICからuserspaceまでの buffer ownership/lifetime の効率化にある。

------------------------------------------------------------------------

## XDP / AF_XDP

v5.0 時点で XDP/AF_XDP は存在した。その後の本質は周辺 infrastructure の成熟。ここで XDP → AF_XDP → netkit / queue leasing を単純な派生関係とはみなさない。 XDP は native driver mode、generic/SKB mode、hardware offload で実行位置・性能特性・必要な driver support が異なり、 AF_XDP や queue leasing は queue ownership / zero-copy という共通課題から並行して発展した面を持つ。

``` text
XDP
├─ redirect
├─ page_pool
├─ link/lifecycle
└─ AF_XDP
    ├─ zero-copy
    ├─ multi-buffer (6.6)
    ├─ virtio-net ZC
    └─ queue ownership / netkit integration
```

v6.6 AF_XDP multi-buffer は:

``` text
one packet = one buffer
```

から:

``` text
one packet
├─ buffer 1
├─ buffer 2
└─ buffer N (EOP)
```

への重要な変更である。

XDP multi-buffer / frags は、single contiguous buffer を暗黙の前提にしていた XDP packet model を multi-buffer packet へ拡張した。6.6 の AF_XDP multi-buffer は同じ multi-buffer problem を userspace zero-copy path へ接続するが、native/generic/offload XDP mode の差を消すものではない。

------------------------------------------------------------------------

## BPF — packet filter から stack extension へ

BPF の発展は一本道ではなく、attachment point と適用範囲が複数方向へ増えたものとして捉える。

TC direct packet access、devmap/cpumap、SK_MSG、flow dissector、`struct_ops`、SK_LOOKUP、TCX は、 packet/datapath、socket/message、protocol algorithm、attachment/lifetime という異なる面を拡張した。 一本の attachment-point lineage としてではなく、verified programmability の適用範囲が広がったものとして読む。

``` text
                     BPF core / verifier / maps
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
 packet / datapath        socket / lookup       protocol algorithms
 XDP, TC                  cgroup hooks,         struct_ops / TCP CC
                         SK_LOOKUP
        │                                           │
        └──────────────┐                 ┌──────────┘
                       ▼                 ▼
                 virtual devices / queue control
                 netkit · BPF qdisc · related hooks
```

ここで下段は上段の単純な後継ではない。5.x～6.x にかけて **適用範囲と attachment point が 独立・並行して追加された**結果として、packet、socket、protocol algorithm、virtual device、 queue/datapath まで programmability の対象が広がった、と読む。

### USENIX research から見える「kernel bypass → in-kernel extensibility」の流れ

USENIX の研究を併せて読むと、Linux networking の programmability が解こうとしてきた問題を別の角度から確認できる。

``` text
mTCP (NSDI 2014): user-level TCP / kernel overhead avoidance
        ↓
XDP/eBPF: early in-kernel programmable hook
        ↓
Electrode (NSDI 2023) / DINT (NSDI 2024):
  kernel の protection / isolation を維持し frequent path を eBPF 化
        ↓
eTran (NSDI 2025):
  transport 自体を eBPF で extensible にする
```

mTCP は Linux kernel TCP processing の CPU cost に対して user-level TCP stack を採った。一方 Electrode と DINT は、kernel bypass の security / isolation / maintainability 上の trade-off を避けながら、XDP/eBPF により frequent path を kernel 内で処理する。eTran はさらに TCP/DCTCP や Homa のような transport design を eBPF ベースの extensible kernel transport として扱う。この研究史は、**kernel を単に迂回するのではなく、kernel の ownership / protection model を保ったまま fast path や protocol behavior を programmable にする**という方向を補強する。

これらは upstream Linux release milestone ではないため Part VI には追加せず、architecture の動機を説明する research evidence として扱う。

------------------------------------------------------------------------

## Virtual networking — datapath と queue assignment の並行進化

v5.0 前後の典型的な virtual datapath は次のような形だった。

``` text
container → veth → host stack → NIC

guest virtio-net → QEMU/vhost/TAP → bridge/OVS → NIC
```

その後は一つの後継系列ではなく、複数の branch が並行して発展した。

``` text
VM datapath:
  virtio-net ∥ vhost ∥ vDPA ∥ SR-IOV/VFIO
                    ↔
              AF_XDP zero-copy

container datapath:
  veth ∥ netkit / BPF-native path

queue / zero-copy assignment:
  AF_XDP / io_uring ↔ RX HW queue leasing
```

netkit は veth の単純な後継ではなく、RX HW queue leasing も netkit から派生した機能ではない。queue lease は virtual-device queue と physical-device queue の対応を明示し、AF_XDP や memory-provider 型の queue-bound API を container/virtual-device 側から利用可能にする **assignment mechanism** として位置付ける。

v6.7 netkit、v6.11 virtio-net AF_XDP RX ZC、v7.1 RX HW queue leasing に共通して見える方向性は、 full kernel bypass そのものではなく、**kernel の control / protection model を維持しながら data movement と datapath overhead を減らす選択肢を増やしたこと**である。

------------------------------------------------------------------------

## TCP / UDP / transport

この節は **transport protocol 自体の semantics / feedback / path management** に絞る。 TCP 上で使われるという理由だけで、memory、aggregation、programmability の milestone を ここへ再収容しない。

### Transport protocol evolution

| Theme                         | Representative milestones  | この節で追う意味                           |
|-------------------------------|----------------------------|--------------------------------------------|
| connection establishment      | TFO client / server        | handshake latency / early data             |
| congestion control / feedback | DCTCP · BBR · PLB · AccECN | congestion signal と sender behavior       |
| multipath                     | MPTCP                      | 複数 path / subflow の transport semantics |
| authentication                | TCP-AO                     | TCP connection authentication              |

これらも一本の派生系列ではない。

``` text
connection establishment:       TFO
congestion control / feedback:  DCTCP ∥ BBR ∥ PLB ∥ AccECN
multipath:                       MPTCP
authentication:                  TCP-AO
```

MPTCP は initial upstream から multi-subflow、userspace path manager、Generic Netlink、BPF integration へ発展し、7.2 の PM limit expansion は operational scale を広げたが、BBR や BIG TCP の後継ではない。

### 他の architecture lineage へ置く TCP 関連機能

- **BIG TCP** — `Packet aggregation` を参照。
- **Device Memory TCP** — `Packet memory` を参照。
- **BPF congestion control / struct_ops** — `BPF` を参照。
- **`TCP_ZEROCOPY_RECEIVE` / io_uring ZCRX** — zero-copy / memory-delivery mechanism として `io_uring networking` および `Packet memory` を参照。

### UDP

UDP でも `MSG_ZEROCOPY` と GRO/GSO は copy cost と per-packet processing を減らす重要な PERFORMANCE milestone だが、protocol semantics の変更ではない。前者は socket zero-copy、 後者は aggregation の lineage として位置付ける。

UDP は GRO/GSO、tunnel/encapsulation、high packet-rate RX、receive-buffer scaling の 基盤として発展してきた。ここでは UDP 固有の protocol lineage と、PERFORMANCE / VIRTUAL-OVERLAY 側で扱う packet aggregation・tunnel acceleration を混同しない。

------------------------------------------------------------------------

## Routing / Netlink / RTNL

6.19 の WireGuard YNL-described Netlink は、個別 security protocol の説明ではなく、machine-readable Netlink schema が実 subsystem へ広がる CONTROL PLANE の例としてここに置く。

routing は nexthop object により:

``` text
route → embedded nexthop
```

だけでなく:

``` text
route → reusable nexthop object → group / resilient group
```

へ進んだ。

Netlink は YNL により:

``` text
YAML specification
├─ UAPI
├─ policy
├─ generated helper
├─ documentation
└─ userspace client
```

という machine-readable API の方向へ進む。

RTNL は:

``` text
global RTNL
 → unlocked operations
 → RCU readers
 → per-netns RTNL
 → subsystem locks/refcounts
 → RTNL-less FIB rule updates (7.3-rc/mainline)
```

この `→` は direct extension を意味せず、global RTNL dependency を縮小する複数の locking / refactoring techniques が同じ設計方向に進んだことを示す。

7.3向けnetworking pullでは `RTM_NEWRULE` / `RTM_DELRULE` のFIB rule変更が RTNL-lock-less化され、further RTNL-dependency reductionやlock-less GET準備と 同じ「global RTNL依存を減らす」流れとしてmainlineへ入った。

さらに 7.3 cycle では per-netns netdev unregistration infrastructure が入り、 per-netns RTNL の大きな blocker だった device unregistration path の分解も進んだ。 これは 6.13 以降の per-netns RTNL lineage の継続として扱う。

------------------------------------------------------------------------

## netfilter / nftables / conntrack

``` text
iptables/netfilter
      ↓
nftables maturation
      ↓
flowtable
      ↓
hardware offload
```

一方で BPF と nftables は単純な新旧置換ではない。

conntrack では performance だけでなく lifetime/GC、per-netns scalability、 hardware flow offload race、BPF kfunc access が重要なテーマとなった。

------------------------------------------------------------------------

## Part III から Part IV へ — feature lineage から driver contract へ

前章では networking mechanism を end-to-end の lineage として追った。Part IV では視点を変え、個々の driver の責務のうち何が共通 networking-core framework へ移されたかを見る。

# Part IV — Network Device Driver Framework の進化

本章では networking core と driver の間の contract に焦点を当てる。`page_pool`、AF_XDP、Device Memory TCP、io_uring は Part III にも登場するが、ここでは feature 全体の歴史ではなく、**driver から見た queue / memory ownership** の役割を扱う。

## Driver-framework projection

Part VI で DRIVER FRAMEWORK を主軸または副軸に持つ milestone を architecture story の観点から投影する。 release attribution / Evidence class はここでは再定義しない。

| Release     | Driver-framework milestone                  | Architectural effect                                                |
|-------------|---------------------------------------------|---------------------------------------------------------------------|
| 3.3         | DQL/BQL                                     | common TX queue accounting / backpressure                           |
| 3.19        | switchdev origin                            | kernel forwarding objects と switch-ASIC offload の接続             |
| 4.6         | devlink                                     | device/ASIC-wide resource/control object                            |
| 4.14        | phylink                                     | common MAC/PHY/PCS/SFP state machine                                |
| 4.16 / 5.3  | Net DIM → `lib/dim`                         | adaptive interrupt moderation の common library 化                  |
| 4.16        | netdevsim                                   | physical hardware なしで common offload API を test                 |
| 5.1         | devlink health                              | common health reporting/recovery                                    |
| 5.6         | ethtool Generic Netlink                     | structured/extensible driver management ABI                         |
| 5.11        | auxiliary bus                               | complex device subfunction の独立 binding                           |
| 5.12 / 6.19 | threaded NAPI → kthread busy-poll extension | NAPI execution model の明示化/拡張                                  |
| 6.2         | XFRM packet offload                         | packet-level IPsec offload contract                                 |
| 6.8         | Rust phylib                                 | safe driver abstraction の networking-side representative milestone |
| 6.8         | queue/NAPI netdev-genl visibility           | queue/NAPI identity を generic object として公開                    |
| 7.1         | RX HW queue leasing                         | physical queue assignment を explicit contract 化                   |

page_pool / memory providers は MEMORY 軸を主とするため、この projection table ではなく Part III `Packet memory` と本章後半の memory-contract 説明で扱う。

### 長期的な architecture の変化

``` text
driver-private mechanisms
        ↓
common queue/backpressure · link · offload · device-control contracts
        ↓
structured/testable userspace-visible APIs
        ↓
queue / NAPI / page-pool objects
        ↓
queue-bound memory providers

parallel language-safety track:
C driver APIs ⇒ Rust-safe abstractions ⇒ Rust PHY / driver-core expansion
```

後半の変化は特に重要である。modern netdev では、従来 driver 内部の不透明な実装詳細だったものを、ID と関係性を持つ generic object として公開する方向が強まっている。

本章では個々の NIC driver を網羅的には列挙しない。Linux network driver に要求される実装を変化させ、driver ごとに重複していた仕組みを再利用可能な kernel framework へ移した common infrastructure を追う。

## Linux 3.0 時点の出発点

Linux 3.0 の時点で NAPI と multiqueue networking はすでに確立していた。RPS/RFS は 2.6.35、XPS は 2.6.38 で導入済みであり、v3.x の driver model はすでに次の要素を中心としていた。

``` text
RX/TX descriptor rings
IRQ / MSI-X
NAPI poll contexts
multiple RX/TX queues
RSS in hardware
RPS/RFS/XPS in the stack
ethtool + net_device_ops
```

したがって v3.x における重要な変化は NAPI の発明ではなく、driver と共通 networking-core algorithm の協調が強まったことである。

## Linux 3.3 — DQL/BQL: queue control が共通 core へ移る

DQL/BQL は、この変化を示す初期の代表例である。

``` text
old:
  each driver/hardware queue can accumulate excessive TX backlog

DQL/BQL:
  common dynamic queue-limit algorithm
  + driver reports queued/completed bytes
  + core dynamically controls outstanding data
```

Exact commit provenance は Part VII にのみ保持する。

LWN series: https://lwn.net/Articles/469651/ https://lwn.net/Articles/469652/

Driver Framework の観点では、performance policy が個々の driver から再利用可能な net core infrastructure へ移り始めたことに意味がある。

## Offload / device-object contracts — hardware が Linux networking の first-class object になる

### switchdev — Linux 3.19 で始まり 4.x で拡大

The initial switchdev infrastructure belongs to the Linux 3.19 generation. The 4.x era is where the model expands into the broader hardware-offload architecture discussed below.

switchdev は switch ASIC forwarding を proprietary SDK だけが制御する孤立した仕組みではなく、Linux driver model の一部として扱えるようにする。

LWN: https://lwn.net/Articles/675826/ https://lwn.net/Articles/676096/

The model represents physical switch ports as normal netdevices and lets bridge, VLAN, routing and related kernel objects drive hardware offload.

``` text
Linux bridge / FIB / VLAN / TC
          ↓
     switchdev model
          ↓
     switch ASIC driver
          ↓
        hardware
```

これは driver の責務における大きな変化であり、driver は Linux networking semantics を hardware 上へ実装する役割を担うようになる。

### devlink — device 全体を扱う control plane

devlink series (v3): https://lwn.net/Articles/677967/

devlink fills a gap left by `net_device`: many settings belong to the whole ASIC/device, not one network interface.

Examples include:

``` text
device resources
port splitting
shared buffers
eswitch mode
device parameters
firmware / health
```

Linux 5.1 later adds devlink health reporting/recovery as a generic mechanism.

This establishes a useful split:

``` text
net_device / ethtool
  interface-facing behavior

devlink
  device / ASIC / resource / health behavior
```

### phylink と SFP

phylink/SFP の初期 RFC は design provenance として扱い、mainline anchor と release containment は Part VII / Part VI に委ねる。

The 2015 phylink/SFP RFC addresses a recurring driver problem: MAC, PHY, PCS/SerDes and hot-pluggable SFP combinations could not be modeled cleanly by simple PHY attachment.

RFC: https://lwn.net/Articles/667055/

Conceptually:

``` text
MAC driver
   │
 phylink
   ├── PHY
   ├── PCS / SerDes
   └── SFP module
```

This progressively removes link-mode state-machine duplication from Ethernet MAC drivers.

### VF representor と SmartNIC/DPU control

Representors extend the switchdev idea to SR-IOV embedded switches and later SmartNIC/DPU architectures.

LWN: https://lwn.net/Articles/692942/

A representor is both a control-plane representation of a VF/SF and a netdevice endpoint through which the normal Linux stack can control the virtual switch.

This is the foundation for the now-familiar:

``` text
PF / uplink
   │
embedded switch
 ├─ VF representor
 ├─ VF representor
 └─ SF / other representors
       ↓
bridge / TC / OVS / routing
       ↓
hardware offload
```

### XDP が driver fast path を変える

Linux 4.8 introduces first-generation XDP. From the driver’s point of view the important change is that the RX path gains a programmable hook before skb allocation / normal stack processing.

Kernel Recipes 2018 explicitly describes XDP as a programmable layer running in device driver context: https://archives.kernel-recipes.org/document/xdp-a-new-programmable-network-layer/

This creates new common driver responsibilities:

``` text
construct xdp_buff
run XDP program
handle PASS / DROP / TX / REDIRECT
support ndo_xdp_xmit
manage RX memory so buffers can move between RX/XDP/TX
```

The last point is one of the pressures that increased the value of a common RX-memory recycling infrastructure such as page_pool; page_pool was not created solely for XDP.

### USENIX research に見る XDP / SmartNIC offload

OSDI 2020 の **hXDP** は、Linux XDP/eBPF の program model、map、helper semantics を FPGA NIC 上へ持ち込み、unmodified eBPF program を NIC 側で実行する研究である。これは upstream XDP hardware-offload の release provenance ではないが、XDP が **Linux の programmable-datapath semantics を hardware execution target へ投影できる abstraction** として研究されたことを示す。USENIX ATC 2022 の program-warping work はこの方向をさらに最適化した。

NSDI 2023 の **IO-TCP** は TCP stack の control plane を CPU 側に保持しつつ、disk I/O と TCP packet-transfer data plane を SmartNIC へ offload する split-stack design を示した。upstream feature そのものではないが、「canonical semantics/control は host に保持し、data movement / fast path を device へ移す」という driver/offload architecture の比較材料になる。

### DIM — interrupt moderation の共通 library 化

DIM の release attribution と evidence class は Part VI、未解決 boundary は Part VII にのみ置く。

Netdev 0x12 (2018) presented DIM as a driver-independent Dynamic Interrupt Moderation library.

https://www.netdevconf.info/0x12/

Rather than every driver inventing adaptive interrupt/coalescing algorithms:

``` text
driver samples events/bytes/packets
        ↓
common DIM algorithm
        ↓
profile decision
        ↓
driver programs hardware moderation
```

This is another example of extracting policy from drivers into common netdev infrastructure.

### page_pool — RX memory management の共通 infrastructure 化

The page_pool work was motivated by drivers independently reinventing high-speed DMA page recycling. A 2016 RFC explicitly described it as a generic API for streaming-DMA page pools, and the refurbished implementation appears in the 2018 XDP-era work.

Key late series: https://lists.openwall.net/netdev/2018/03/31/91

Modern page_pool provides a common allocation/recycling/DMA model for skb and XDP buffers.

``` text
before:
 driver-specific RX allocator
 driver-specific recycling
 driver-specific DMA lifetime tricks

after:
        page_pool
       /         \
     skb       xdp_frame
       \         /
     common recycling
```

Its importance grows well beyond the original XDP motivation. By Netdev 0x19, page_pool is described as the standard RX datapath memory-management mechanism, and newer zero-copy features require drivers to integrate with it.

## Operational driver infrastructure — management API・health・共通 library

### devlink health

Linux 5.1 adds generic devlink health reporting and recovery.

Netdev 0x13 describes the goals as:

``` text
real-time alerting
driver debug information
self-healing / recovery
vendor-support data collection
```

Conference provenance は Appendix に集約する。

This changes hardware error handling from driver-specific logs/private tools toward a common operational model.

### ethtool ioctl → Generic Netlink

The ethtool netlink work addresses limitations of the old ioctl ABI: extensibility, races, error reporting and lack of notifications.

Series: https://lwn.net/Articles/808028/ https://lwn.net/Articles/810618/

Architecturally this is not just a userspace-tool rewrite. It creates a structured, extensible management API between userspace, networking core and drivers.

### netdevsim と selftest-driven driver API design

`netdevsim` becomes an important test vehicle for driver-facing APIs. Current netdev maintainer documentation explicitly encourages new driver configuration APIs to have netdevsim/selftest coverage, while also requiring a real driver use case.

This changes the development model:

``` text
new driver API
   ↓
generic implementation
   ↓
netdevsim model + selftests
   ↓
real hardware driver
```

Driver frameworks are increasingly expected to be testable without the physical NIC.

### auxiliary bus — 1つの PCI device と複数 subsystem driver

Merged for Linux 5.11, the auxiliary bus addresses complex devices exposing Ethernet, RDMA, vDPA and related functions from shared hardware.

Instead of ad-hoc cross-driver glue:

``` text
              PCI function
                   │
              parent/core
             /      |      \
        netdev     RDMA    vDPA
       auxiliary drivers / devices
```

This becomes increasingly important for SmartNIC/IPU/DPU architectures.

## Explicit queue / memory contracts — queue・NAPI・memory provider の object 化

### page_pool の可観測化

2023 page_pool netlink introspection associates pools with netdevices and NAPI IDs and exports allocation/recycling/memory information.

LWN: https://lwn.net/Articles/948718/

This is an important architectural transition:

``` text
page_pool as hidden driver implementation detail
                 ↓
page_pool as identifiable / observable netdev resource
```

### queue / NAPI object の generic netdev API 化

Netdev 0x17 discusses exposing queues and NAPI instances through `netdev-genl`.

https://netdevconf.info/0x17/sessions/talk/netlink-apis-to-exposeconfigure-netdev-objects.html

The proposed/ongoing model makes properties such as these explicit:

``` text
queue
 ├─ NAPI instance
 ├─ stats
 ├─ memory model
 └─ XDP / zero-copy capabilities

NAPI
 ├─ NAPI ID
 ├─ device
 ├─ IRQ
 └─ thread / CPU relationship
```

This is a conceptual shift from “the driver owns opaque rings” toward explicit queue / NAPI / memory objects with stable identities and API-defined properties.

ただし **可視化・設定・割り当て・ownership は同義ではない**。API ごとに許される操作は異なり、 ある object を userspace から列挙・参照できても、その lifetime や ownership を userspace が自由に 変更できるとは限らない。netdev-genl の read/configuration API、memory-provider registration、 queue-leasing のような assignment mechanism は、それぞれ capability と permission boundary を個別に読む必要がある。

This connects conceptually to Device Memory TCP, io_uring ZCRX and queue-leasing work elsewhere in this document, but does not imply that one generic API grants all of those control operations.

### page_pool → netmem → memory providers

The driver-framework view of the memory lineage is:

``` text
driver-private RX recycling
        ↓
page_pool
        ↓
page_pool as common driver contract
        ↓
netmem abstraction
        ↓
memory providers
        ↓
host pages / userspace memory / device memory
```

Kernel Recipes 2024 の公開 abstract が直接述べるのは、kernel network stack を利用し、vanilla TCP と互換性のある zero-copy receive の設計である。NIC / firmware / driver support、page_pool、netmem、queue configuration という具体的な実装依存関係は abstract 自体ではなく、同 conference の live blog と後続 upstream implementation から確認する。

つまり modern high-speed network driver は、単に `struct page` を allocate するだけではなく、次第に **memory-provider-aware** であることを求められている。

### XFRM device / IPsec packet offload

XFRM device offload is another common driver-framework contract. The important 6.2 generation extends the model from crypto acceleration to packet offload, where the NIC can own SA/policy processing as well as encryption/decryption.

``` text
XFRM core
   ↓ state + policy synchronization
xfrmdev_ops
   ↓
NIC driver
   ↓
IPsec hardware pipeline
```

This is the same architectural pattern seen in switchdev and TC offload: the kernel keeps the canonical networking semantics while the driver maps those semantics onto hardware.

## Rust — common driver contracts を safe abstraction として表現する parallel track

Rust は networking architecture の主 lineage ではなく、common C driver contracts が明確になるほど、 その ownership / lifetime / state-machine boundary を型安全な abstraction として表現できる、という **parallel language-safety track** として扱う。

Linux 6.1 の Rust-for-Linux 導入は kernel-wide context であり、networking の canonical milestone ではない。本書での networking milestone は、6.8 の Rust PHY abstraction / Asix reference PHY を代表点とする。release attribution と exact anchor は Part VI / VII にのみ置く。

``` text
common C driver contracts
  phylib · NAPI · devlink · page_pool · DMA / device lifecycle
                         ⇒
             explicit lifetime / state boundaries
                         ⇒
                safe Rust abstractions
```

重要なのは C implementation を Rust に置換したことではなく、driver framework が持つ lifetime、ownership、state transition、resource cleanup の contract を `unsafe` boundary の内側へ 閉じ込め、safe code から利用できる API として表現できることである。

real NIC driver には PHY だけでなく PCI/platform probing、DMA、MMIO、IRQ、device removal など kernel-wide driver-core abstraction も必要である。したがって Rust networking の成熟は networking subsystem 単独では完結しない。ただし conference/review の詳細年表は本文の主題ではない。Appendix には architecture を補助する代表例だけを置く。

## Conference に見る lineage

### Netdev

Netdev is the strongest conference source for driver-framework implementation:

``` text
2016  switchdev / hardware-offload model (conference provenance not independently re-verified here)
2018  DIM, switchdev/NOS, offload and driver API work
2019  devlink health
2023  Rust networking tutorial
      queue/NAPI netdev-genl objects
2024  Driver and H/W APIs workshop
      memory pools / queues / devlink / fwctl
2025  page_pool leak diagnostics
2026  dedicated Device Driver Workshop
```

### Kernel Recipes

Kernel Recipes は zero-copy networking と Rust/C driver abstraction の architecture context を補う。 代表的な講演 provenance は Appendix に置く。網羅的な Rust networking conference history は本書の対象外とする。

## Part IV から Part V へ — common contracts から operability へ

fast path、memory ownership、offload は、operator/developer が kernel の動作を理解できて初めて運用可能になる。Part V では、それと並行して進化した visibility、tracing、explainability を追う。

# Part V — Observability / Explainability

Part V は **kernel 側の observability primitive の進化**に限定します。Observability は独立した軸であると同時に、programmable / offloaded / zero-copy path が増えて複雑化した networking を **operationally explainable にする evidence layer** と位置付けます。 Retis / pwru はこれらを利用する case study であり、kernel release chronology そのものではないため Appendix に移します。

## Kernel observability primitive

Linux networking の observability は、単純な interface counter / packet capture から、 kernel 内部の typed event と packet-lifecycle metadata を相関できる方向へ進化しました。

``` text
Observability primitives (parallel / complementary)

  tracepoints / perf / kprobes ─┐
  eBPF tracing + BTF ────────────┼─► tool-side correlation ─► higher-level inference
  structured drop reasons ───────┤
  timestamping ──────────────────┤
  queue / NAPI identity ─────────┤
  page_pool stats/diagnostics ───┘

※ 左側の primitive は一本道に「進化」した関係ではない。異なる種類の evidence を並行して提供する。
  packet-journey reconstruction は、それらを相関する tool / analysis 側の目標である。
```

### Primitive が提供する情報と限界

| Primitive               | 主に提供するもの                                 | 境界 / 注意点                                                   |
|:------------------------|:-------------------------------------------------|:----------------------------------------------------------------|
| BTF                     | kernel type / field metadata                     | runtime event や packet trajectory 自体は記録しない             |
| eBPF tracing            | attach point での event / state                  | coverage、attach point、権限、overhead に依存                   |
| `skb_drop_reason`       | 対応箇所での structured drop reason              | 全 drop path が必ず reason を付与するわけではない               |
| timestamping            | 特定地点での時刻情報                             | clock、取得地点、HW/SW timestamp semantics に依存               |
| netdev-genl             | queue / NAPI 等の identity、state、configuration | 可視化できることと自由に ownership/configuration できることは別 |
| page_pool introspection | pool identity / stats / diagnostics              | packet 単位の end-to-end trajectory ではない                    |

したがって cross-layer packet journey は単一 primitive の機能ではなく、tool が複数の identifier、event、 timestamp、metadata を相関して **推論する目標**として扱う。

### BTF と eBPF tracing

BTF により running kernel の型情報を利用できるため、observability tool は private kernel structure の固定offsetに依存する必要を減らせます。これは networking のように内部構造の変化が 速い領域で特に重要です。

### Structured drop reason

Linux 5.17 の `kfree_skb_reason()` / `skb_drop_reason` 世代は、「packet が消えた」という観測を 「どの理由でdropされたか」という structured metadata へ変えました。その後、coverage は networking stack の各所へ拡張されています。

7.1 の dedicated qdisc-drop tracepoint は、qdisc 内の drop context を generic tracing から直接観測しやすくした例であり、structured drop reason と補完関係にある。

### Timestamping と packet lifecycle

`SO_TIMESTAMPING`、driver/hardware timestamp、BPFから取得できる時刻・contextは、 単一地点のpacket captureでは見えない queueing / scheduling / offload の時間軸を補います。

### Queue / NAPI / page_pool observability

Part IVで説明した queue、NAPI、page_pool のobject化は、control planeだけでなくobservabilityにも 効きます。packet memory、polling context、queue identityをuserspace-visible objectとして関連付ける ことで、zero-copy / memory-provider時代の問題を説明しやすくなります。ただし、readable な identity / stats と、 userspace が queue や NAPI の ownership/configuration を変更できる control-plane capability は区別する。

この軸を Part VI の確定 milestone に対応させると、代表的な observability milestone の**時系列**は次のようになる。これは機能間の依存関係を示す図ではない。

``` text
4.1   kprobe-attached eBPF / tracing foundation
  ∥
5.17  kfree_skb_reason / structured drop-reason foundation
  ⇒
5.19  drop-reason coverage expansion
  ∥
6.8   queue/NAPI netdev-genl visibility
  ∥
7.1   dedicated qdisc-drop tracepoint
```

`netdev-genl` による queue / NAPI identity と stats は、従来の interface-level counter だけでは説明しにくかった「どの queue / poller で問題が起きているか」を userspace から関連付ける基盤になる。page_pool 側でも pool identity、allocation / recycle information、leak diagnostics が整備され、packet-memory ownership 自体が 観測対象へ移っている。ここでの release attribution は Part VI の値を参照する。

## Tool の case study

Retis と pwru は上記primitiveを利用する代表例ですが、本書ではkernel evolutionそのものと区別します。

- **pwru**: 広いkernel function trajectoryから「packetがどこを通ったか」を探索する。
- **Retis**: networking event、skb metadata、OVS/OVN contextなどを意味的にenrichして相関する。

詳細なconference/source provenanceはAppendixのcase-study indexに集約します。

# Synthesis — thesis への回帰と未完の仕事

15年間を通じて一貫して観察できるのは、単なる packet/s の増加ではない。 driver-private / implicit / global だった state と convention の一部が、 **object / accounting / API / assignment / lifetime / synchronization** という kernel 共通 contract として明示されてきたことである。

同時に、従来の system-RAM + `skb` path は消えていない。XDP/AF_XDP、offload、 Device Memory TCP、io_uring zero-copy などはそれを全面置換するのではなく、 用途と hardware capability に応じて **複数の execution / memory-ownership model を共存** させる方向へ Linux networking を拡張した。

この変化には未完の領域もある。TX-side queue leasing は RX-side と同じ意味で完成しておらず、 per-netns RTNL は global RTNL を一括置換したものではない。Rust networking も現時点では PHY/core abstraction を中心とする段階で、一般的な high-performance NIC driver ecosystem が Rust に移行したことを意味しない。これらは「次の release の機能」ではなく、 **explicit contracts の適用範囲が今後どこまで広がるか**という継続課題である。

## 6軸への回帰

- **PERFORMANCE:** queueing は driver-local tuning から accounting / pacing / multi-layer scheduling へ進み、同時に BIG TCP や scalable TX のような contract 外の高速化も進んだ。
- **PROGRAMMABILITY:** BPF/XDP は単一 fast path から socket / protocol / qdisc / virtual-device へ広がり、再利用可能な infrastructure になった。
- **MEMORY:** page_pool から netmem / memory providers / device memory へ、allocation・lifetime・assignment が明示的になった。
- **CONTROL PLANE:** reusable objects、YNL schema、per-netns/fine-grained locking により embedded/global state の範囲が縮小した。
- **OBSERVABILITY:** typed reason / identity / tracing は、複雑化した programmable/offloaded/zero-copy path を operationally explainable にする evidence layer になった。
- **DRIVER FRAMEWORK:** BQL、switchdev、devlink、phylink、DIM、page_pool、netdev-genl は driver-local convention を common contract へ引き上げた。

したがって本書の thesis は未来予測ではなく、既存 milestone を横断した説明モデルである。 次の Part VI では、この story からいったん離れ、各 milestone の release attribution を 1行1項目で正規化する。

# Part VI — Canonical release chronology

ここだけが **release attribution と Evidence class の正本**である。Part I–V の version 表記は story/lineage の参照であり、この表を上書きしない。

## 採用基準

Part VI は release note の網羅表ではない。採用するのは、本文で追う6軸の長期 lineage、 または TRANSPORT / VIRTUAL-OVERLAY / SECURITY の cross-cutting domain を理解するために必要な **origin / integration / enablement milestone** と、architecture 上の operational range を大きく変える milestone である。

単発の機能追加、局所的な数値上限変更、後続 lineage を変えない個別 protocol/offload enhancement は 原則として採用しない。「Axis を付けられる」だけでは採用理由にならず、Part I–V の story で役割を 説明できることを要求する。

**Axis と Domain は独立である。** Axis は architecture 上の主問題を示す。Domain は単なる実装場所ではなく、 protocol semantics / path management / authentication / virtual-overlay behavior 自体が主題となる場合に付与する。 Axis があることは `explicit contracts` thesis で説明可能であることを意味しない。たとえば BIG TCP は PERFORMANCE 軸だが、contract 化ではなく processing-unit expansion として説明する。

## Evidence model

development stage は混同しない。

``` text
RFC / review
    ≠ subsystem tree / net-next
    ≠ Linus mainline
    ≠ released tag
```

Evidence class は次の語彙だけを用いる。

- **TAG** — final release containment / authoritative release evidence を確認。
- **TAG+ANCHOR** — TAG に加え、Part VII に完全な40桁 mainline anchor を保持。
- **SERIES+TAG** — relevant series/pull と final release containment を確認。
- **SERIES** — series/pull evidence は強いが exact release boundary の監査を残す。
- **GENERATION** — release-generation attribution は強いが exact origin/integration boundary を残す。
- **MAINLINE** — Linus mainline merge 済み、final tag 未公開。

旧 Grade は独立列として持たない。必要なら `TAG / TAG+ANCHOR / SERIES+TAG → A`、 `GENERATION / SERIES → B`、`MAINLINE → A-rc` と機械的に導出できる。

**Part VI ↔ Part VII invariant:** `TAG+ANCHOR` を使う milestone には Part VII に完全な40桁 SHA が 存在しなければならない。

## Canonical milestone table

### 3.0–3.18

| Release | Milestone                                       | Axis                           | Domain            | Evidence class |
|:--------|:------------------------------------------------|:-------------------------------|:------------------|----------------|
| 3.0     | namespace FD / setns()                          | CONTROL PLANE                  | —                 | TAG            |
| 3.3     | DQL/BQL                                         | PERFORMANCE / DRIVER FRAMEWORK | —                 | TAG+ANCHOR     |
| 3.5     | CoDel                                           | PERFORMANCE                    | —                 | TAG+ANCHOR     |
| 3.5     | fq_codel                                        | PERFORMANCE                    | —                 | TAG            |
| 3.6     | TSQ                                             | PERFORMANCE                    | —                 | TAG            |
| 3.6     | TFO client                                      | —                              | TRANSPORT         | TAG            |
| 3.6     | IPv4 route-cache removal                        | CONTROL PLANE                  | —                 | TAG            |
| 3.7     | VXLAN                                           | —                              | VIRTUAL / OVERLAY | TAG            |
| 3.7     | TFO server                                      | —                              | TRANSPORT         | TAG            |
| 3.9     | TCP/UDP SO_REUSEPORT                            | PERFORMANCE                    | —                 | TAG+ANCHOR     |
| 3.11    | SO_BUSY_POLL                                    | PERFORMANCE                    | —                 | TAG            |
| 3.12    | sch_fq / TCP pacing / TSO autosizing generation | PERFORMANCE                    | —                 | TAG            |
| 3.13    | nftables                                        | CONTROL PLANE                  | —                 | TAG+ANCHOR     |
| 3.15    | internal BPF ISA rework                         | PROGRAMMABILITY                | —                 | TAG            |
| 3.18    | bpf() / maps / verifier generation              | PROGRAMMABILITY                | —                 | TAG            |
| 3.18    | DCTCP                                           | —                              | TRANSPORT         | TAG            |
| 3.18    | Geneve                                          | —                              | VIRTUAL / OVERLAY | TAG            |

### 3.19–4.20

| Release | Milestone                            | Axis                     | Domain               | Evidence class |
|:--------|:-------------------------------------|:-------------------------|:---------------------|----------------|
| 3.19    | switchdev origin                     | DRIVER FRAMEWORK         | —                    | TAG            |
| 3.19    | ipvlan                               | —                        | VIRTUAL / OVERLAY    | TAG            |
| 3.19    | SO_ATTACH_BPF                        | PROGRAMMABILITY          | —                    | TAG            |
| 4.1     | cls_bpf / act_bpf eBPF support       | PROGRAMMABILITY          | —                    | GENERATION     |
| 4.1     | kprobe BPF milestone                 | OBSERVABILITY            | —                    | GENERATION     |
| 4.6     | devlink                              | DRIVER FRAMEWORK         | —                    | TAG+ANCHOR     |
| 4.7     | TC BPF direct packet access          | PROGRAMMABILITY          | —                    | TAG            |
| 4.8     | XDP                                  | PROGRAMMABILITY          | —                    | SERIES+TAG     |
| 4.9     | BBR                                  | —                        | TRANSPORT            | TAG+ANCHOR     |
| 4.10    | cgroup BPF                           | PROGRAMMABILITY          | —                    | SERIES+TAG     |
| 4.10    | BPF LWT                              | PROGRAMMABILITY          | —                    | SERIES+TAG     |
| 4.13    | SOCK_OPS                             | PROGRAMMABILITY          | TRANSPORT            | TAG            |
| 4.13    | kTLS TX                              | —                        | SECURITY / TRANSPORT | TAG            |
| 4.14    | phylink                              | DRIVER FRAMEWORK         | —                    | TAG+ANCHOR     |
| 4.14    | SOCKMAP                              | PROGRAMMABILITY          | —                    | TAG            |
| 4.14    | XDP devmap                           | PROGRAMMABILITY          | —                    | TAG            |
| 4.14    | TCP MSG_ZEROCOPY                     | PERFORMANCE              | —                    | TAG+ANCHOR     |
| 4.15    | XDP cpumap                           | PROGRAMMABILITY          | —                    | TAG            |
| 4.16    | netdevsim                            | DRIVER FRAMEWORK         | —                    | TAG+ANCHOR     |
| 4.16    | Net DIM generation                   | DRIVER FRAMEWORK         | —                    | GENERATION     |
| 4.16    | nftables software flowtable          | PERFORMANCE              | —                    | TAG            |
| 4.17    | BPF_PROG_TYPE_SK_MSG                 | PROGRAMMABILITY          | —                    | TAG+ANCHOR     |
| 4.18    | AF_XDP                               | MEMORY / PROGRAMMABILITY | —                    | SERIES+TAG     |
| 4.18    | page_pool origin / XDP memory return | MEMORY                   | —                    | TAG+ANCHOR     |
| 4.18    | TCP_ZEROCOPY_RECEIVE                 | PERFORMANCE              | —                    | TAG            |
| 4.19    | SO_TXTIME                            | PERFORMANCE              | —                    | TAG            |
| 4.19    | CAKE                                 | PERFORMANCE              | —                    | TAG            |
| 4.20    | TCP EDT                              | PERFORMANCE              | —                    | TAG            |
| 4.20    | taprio                               | PERFORMANCE              | —                    | TAG            |
| 4.20    | BPF flow dissector                   | PROGRAMMABILITY          | —                    | TAG            |

### 5.0–6.1

| Release | Milestone                              | Axis             | Domain               | Evidence class |
|:--------|:---------------------------------------|:-----------------|:---------------------|----------------|
| 5.0     | UDP GRO                                | PERFORMANCE      | —                    | TAG            |
| 5.0     | UDP MSG_ZEROCOPY                       | PERFORMANCE      | —                    | TAG            |
| 5.1     | devlink health                         | DRIVER FRAMEWORK | —                    | TAG            |
| 5.1     | mac80211 airtime accounting/scheduling | PERFORMANCE      | —                    | TAG            |
| 5.3     | nexthop objects                        | CONTROL PLANE    | —                    | TAG            |
| 5.3     | DIM generalized into lib/dim           | DRIVER FRAMEWORK | —                    | GENERATION     |
| 5.5     | mac80211 AQL                           | PERFORMANCE      | —                    | TAG            |
| 5.6     | MPTCP                                  | —                | TRANSPORT            | TAG            |
| 5.6     | WireGuard                              | —                | SECURITY / VIRTUAL / OVERLAY | TAG            |
| 5.6     | BPF struct_ops / TCP CC                | PROGRAMMABILITY  | TRANSPORT            | TAG            |
| 5.6     | ethtool Generic Netlink                | DRIVER FRAMEWORK | —                    | TAG+ANCHOR     |
| 5.9     | BPF_PROG_TYPE_SK_LOOKUP                | PROGRAMMABILITY  | —                    | TAG+ANCHOR     |
| 5.11    | auxiliary bus                          | DRIVER FRAMEWORK | —                    | TAG+ANCHOR     |
| 5.12    | threaded NAPI                          | DRIVER FRAMEWORK | —                    | TAG            |
| 5.17    | structured drop-reason foundation      | OBSERVABILITY    | —                    | TAG            |
| 5.18    | XDP multi-buffer / frags generation    | PROGRAMMABILITY  | —                    | GENERATION     |
| 5.19    | IPv6 BIG TCP                           | PERFORMANCE      | —                    | TAG+ANCHOR     |
| 5.19    | drop-reason expansion                  | OBSERVABILITY    | —                    | TAG            |
| 6.0     | io_uring SEND_ZC                       | PERFORMANCE      | —                    | TAG            |
| 6.0     | io_uring multishot receive             | PERFORMANCE      | —                    | TAG            |

### 6.2–7.3-rc

| Release | Milestone                                             | Axis                             | Domain               | Evidence class |
|:--------|:------------------------------------------------------|:---------------------------------|:---------------------|----------------|
| 6.2     | TCP PLB                                               | —                                | TRANSPORT            | TAG            |
| 6.2     | XFRM/IPsec packet offload                             | DRIVER FRAMEWORK                 | SECURITY             | TAG+ANCHOR     |
| 6.3     | YNL / YAML Netlink tooling                            | CONTROL PLANE                    | —                    | TAG            |
| 6.3     | IPv4 BIG TCP                                          | PERFORMANCE                      | —                    | TAG            |
| 6.6     | AF_XDP multi-buffer                                   | MEMORY / PROGRAMMABILITY         | —                    | TAG            |
| 6.6     | TCX / bpf_mprog                                       | PROGRAMMABILITY                  | —                    | TAG            |
| 6.7     | netkit                                                | PROGRAMMABILITY                  | VIRTUAL / OVERLAY    | TAG+ANCHOR     |
| 6.7     | TCP-AO                                                | —                                | SECURITY / TRANSPORT | TAG            |
| 6.8     | Rust phylib / Asix reference PHY                      | DRIVER FRAMEWORK                 | —                    | TAG+ANCHOR     |
| 6.8     | queue/NAPI netdev-genl visibility                     | DRIVER FRAMEWORK / OBSERVABILITY | —                    | GENERATION     |
| 6.11    | virtio-net AF_XDP RX zero-copy                        | MEMORY                           | VIRTUAL / OVERLAY    | TAG            |
| 6.12    | Device Memory TCP RX                                  | MEMORY                           | —                    | SERIES         |
| 6.13    | per-netns RTNL infrastructure milestone               | CONTROL PLANE                    | —                    | SERIES         |
| 6.15    | io_uring ZCRX                                         | MEMORY                           | —                    | SERIES+TAG     |
| 6.15    | further RTNL breakup                                  | CONTROL PLANE                    | —                    | SERIES+TAG     |
| 6.16    | Device Memory TCP TX                                  | MEMORY                           | —                    | SERIES+TAG     |
| 6.16    | BPF qdisc                                             | PROGRAMMABILITY                  | —                    | SERIES+TAG     |
| 6.18    | AccECN core                                           | —                                | TRANSPORT            | GENERATION     |
| 6.18    | UDP RX evolution                                      | PERFORMANCE                      | —                    | GENERATION     |
| 6.19    | `dev_queue_xmit()` llist TX scheduling                | PERFORMANCE                      | —                    | TAG            |
| 6.19    | threaded-NAPI kthread busy-poll extension             | DRIVER FRAMEWORK                 | —                    | TAG            |
| 6.19    | WireGuard YNL-described Netlink                       | CONTROL PLANE                    | SECURITY             | TAG            |
| 7.0     | cake_mq                                               | PERFORMANCE                      | —                    | TAG            |
| 7.0     | IPv6 BIG TCP without synthetic HBH jumbo header       | PERFORMANCE                      | —                    | TAG            |
| 7.0     | AccECN enablement                                     | —                                | TRANSPORT            | TAG            |
| 7.0     | large RX buffers for memory providers / io_uring ZCRX | MEMORY                           | —                    | TAG            |
| 7.1     | RX HW queue leasing                                   | MEMORY / DRIVER FRAMEWORK        | —                    | TAG            |
| 7.1     | dedicated qdisc-drop tracepoint                       | OBSERVABILITY                    | —                    | TAG            |
| 7.2     | MPTCP PM limit expansion                              | —                                | TRANSPORT            | TAG            |
| 7.2     | PPPoE GRO/GSO                                         | PERFORMANCE                      | —                    | TAG            |
| 7.3-rc  | BIG TCP over VXLAN/GENEVE                             | PERFORMANCE                      | VIRTUAL / OVERLAY    | MAINLINE       |
| 7.3-rc  | RTNL-less FIB-rule updates                            | CONTROL PLANE                    | —                    | MAINLINE       |
| 7.3-rc  | devmem buffers \>PAGE_SIZE                            | MEMORY                           | —                    | MAINLINE       |
| 7.3-rc  | per-netns netdev-unregistration infrastructure        | CONTROL PLANE                    | —                    | MAINLINE       |

### provenance 正規化を残す architecture-critical items

`netmem`、memory providers、page_pool introspection、BTF、vDPA は本文の architecture story 上重要だが、 この版では origin / integration / enablement のどれを1行の canonical milestone とするかを 一次 evidence で再正規化できていない。**したがって Part VI に行がないことは意図的であり、重要度が低いことを意味しない。** release を推測して `GENERATION` 行を作らず、Part VII の **Open provenance items** に明示的に残す。

## Part VI から Part VII へ — chronology から provenance へ

Part VI は「いつ」を正規化し、Part VII はその attribution を再監査できる exact anchor と 未解決 boundary を保持する。

# Part VII — 正規 provenance ledger

**Evidence model:** Part VII の SHA は feature series の「代表 anchor」であり、anchor の存在だけで series 全体を証明しない。 各項目は **SHA identity / feature correspondence / release containment** を別々に監査する。

**Evidence status:** `MAINLINE` は authoritative な pull/merge evidence により Linus mainline への merge を確認済みだが、final release tag が未公開の状態を示す。Part VI の Evidence class と一致させる。

## Exact mainline anchor inventory

ここに示すのは feature series の全 commit ではなく、再監査可能な代表 anchor である。`Anchor type` は `origin / integration / enablement / merge` の役割を示す。`First containing tag` を本版で再正規化していない項目は `open` とし、推測で補わない。

| Item | Anchor type | Exact mainline anchor | Subject / role | First containing tag |
|---|---|---|---|---|
| DQL | origin | `75957ba36c05b979701e9ec64b37819adc12f830` | `dql: Dynamic queue limits` | open |
| CoDel | origin | `76e3cc126bb223013a6b9a0e2a51238d1ef2e409` | CoDel qdisc core anchor | open |
| SO_REUSEPORT infrastructure | origin | `055dc21a1d1d219608cd4baac7d0683fb2cbbe8a` | `soreuseport: infrastructure` | open |
| nftables core | origin | `96518518cc417bb0a8c80b9fb736202e28acdf96` | `netfilter: add nftables` | open |
| nftables set API | integration | `20a69341f2d00cd042e81c82289fba8a13c05a25` | set-API anchor; not core origin | open |
| devlink | origin | `bfcd3a46617209454cfc0947ab093e37fd1e84ef` | `Introduce devlink infrastructure` | v4.6-rc1 |
| BBR | origin | `0f8782ea14974ce992618b55f0c041ef43ed0b78` | initial BBR mainline anchor | open |
| phylink | origin | `9525ae83959b60c6061fe2f2caabdc8f69a48bc6` | `phylink: add phylink infrastructure` | open |
| netdevsim | origin | `83c9e13aa39aed5cf9a2f8dd69770b7c35ba1281` | hardware-independent offload test device | v4.16-rc1 |
| SK_MSG | enablement | `4f738adba30a7cfc006f605707e7aee847ffefa0` | socket-message verdict / `BPF_PROG_TYPE_SK_MSG` | open |
| TCP MSG_ZEROCOPY | enablement | `f214f915e7db99091f1312c48b30928c1e0c90b7` | `tcp: enable MSG_ZEROCOPY` | open |
| page_pool origin | origin | `ff7d6b27f894f1469dc51ccb828b7363ccd9799f` | page_pool core origin anchor | open |
| page_pool/XDP integration | integration | `60bbf7eeef10dc647430646d7fe5e3d8d132dbec` | mlx5 page_pool/XDP integration | open |
| ethtool Generic Netlink | origin | `2b4a8990b7df55875745a80a609a1ceaaf51f322` | `ethtool: introduce ethtool netlink interface` | open |
| SK_LOOKUP | origin | `e9ddbb7707ff5891616240026062b8c1e29864ca` | dedicated SK_LOOKUP program type / attach point | open |
| auxiliary bus | origin | `7de3697e9cbd4bd3d62bafa249d57990e1b8f294` | `Add auxiliary bus support` | v5.11-rc1 |
| IPv6 BIG TCP / GRO | enablement | `0fe79f28bfaf73b66b7b1562d2468f94aa03bd12` | allow `gro_max_size` > 65536 | open |
| IPv6 BIG TCP / GSO | enablement | `7c4e983c4f3cf94fcd879730c6caa877e0768a4d` | allow `gso_max_size` > 65536 | open |
| XFRM packet offload | enablement | `d14f28b8c1de668bab863bf5892a49c824cb110d` | add packet offload flag | open |
| netkit | origin | `35dfaad7188cdc043fde31709c796f5a692ba2bd` | netkit core anchor | open |
| Rust PHY abstractions | integration | `f20fd5449ada3872dcd67aca397f0e27ca2e8ad6` | Rust core abstractions for network PHY drivers | v6.8-rc1 |
| netdev-genl queue object | integration | `bc877956272f0521fef107838555817112a450dc` | YAML spec for queue object | v6.8-rc1 |
| MPTCP subflow / accepted ADD_ADDR limits | enablement | `c8646664fbf1c0beb0990cef391cb52d3c909e78` | limits expanded to 64 | open |
| MPTCP endpoint limit | enablement | `e845e6397d78bf6b842cfa8b5818ca8189f7e22e` | endpoint limit expanded to 255 | open |
| net-next 7.3 merge | merge | `91ec2035134982b98fab0609a9fd8480e8217dc1` | merge tag `net-next-7.3` | mainline snapshot |

Net DIM / `lib/dim` は algorithm の driver-local origin と common-library generalization の exact boundary を本版で再確認できていないため、この exact inventory には追加せず Open provenance items に残す。

## Open provenance items

Part VI の `GENERATION` / `SERIES` 全行を複製する表ではない。Part VI にまだ独立 milestone を立てられていない architecture-critical item、または exact-anchor inventory と canonical row の対応に追加監査が必要な item だけを置く。各 `GENERATION` / `SERIES` 行の evidence status 自体は Part VI が正本である。

| Item                         | Unresolved boundary                                                                             |
|:-----------------------------|:------------------------------------------------------------------------------------------------|
| Net DIM / `lib/dim`          | driver-local origin と common-library generalization の exact anchors                           |
| queue/NAPI netdev-genl       | queue-object anchor は保持済みだが、composite milestone 全体の boundary                         |
| `netmem`                     | canonical origin / first containing tag                                                         |
| memory providers             | provider model の canonical integration / enablement boundary                                   |
| page_pool introspection      | observability milestone として採用する canonical boundary                                       |
| BTF networking observability | networking history 上どの milestone を代表点とするか                                            |
| vDPA                         | 本書の virtual-networking story で採用する origin/integration boundary                          |
| RX HW queue leasing          | v7.1 final containment は確定。exact 40-digit representative anchor を inventory に追加する余地 |

**Queue-leasing release boundary:** v7.0 merge window の initial merge `77b9c4a438fc66e2ab004c411056b3fb71a54f2c` は `8766d61a1d33cb5f15bfdd6ce9832bbe1fc649c2` で revert され、v7.0 final には含まれない。 v7.1 final には revised RX queue-leasing series が含まれる。短縮 SHA `7789c6bb76ac` は exact 40-digit inventory には登録しない。TX queue leasing は未 merge である。

# Appendix — Source index（非正規）

Appendix は supporting evidence / research provenance の索引である。本文の release attribution や lineage を再定義しない。

## USENIX research index — architecture motivation / design-space evidence

以下は upstream release attribution の根拠ではなく、Linux networking architecture の design space と、その機能が解こうとする問題を説明する research evidence である。

- **NSDI 2011 — Multipath TCP congestion control** — Linux implementation を用いた multipath congestion control の研究。MPTCP design lineage の前史として参照する。
- **NSDI 2014 — mTCP** — kernel TCP/syscall/packet-I/O overhead に対する user-level TCP stack という設計点。
- **OSDI 2020 — hXDP** — Linux XDP/eBPF semantics を FPGA NIC execution target へ展開。
- **USENIX ATC 2022 — eBPF Program Warping** — hXDP を発展させ、eBPF program の一部を FPGA pipeline へ変換。
- **NSDI 2023 — Electrode** — XDP/eBPF により kernel-stack traversal と user/kernel crossing を削減。
- **NSDI 2023 — IO-TCP** — TCP control plane を CPU 側に保持し、packet-transfer/data plane を SmartNIC へ offload。
- **NSDI 2023 — Valinor** — eBPF を利用して qdisc、CC、scheduler、NIC、hardware offload を複数 vantage point から観測。
- **NSDI 2024 — DINT** — kernel-bypass-like performance と kernel stack の security/isolation/maintainability の両立を狙う。
- **NSDI 2025 — eTran** — eBPF によって kernel transport を extensible にし、TCP/DCTCP と Homa を実装。

これらは Part VI / Part VII の canonical release/commit evidence と混同しない。

## Kernel documentation / primary-source index

| Source topic                         | Canonical destination                                   | What it supports                                                |
|:-------------------------------------|:--------------------------------------------------------|:----------------------------------------------------------------|
| NAPI / `SO_BUSY_POLL`                | Part VI 3.11; Part IV                                   | busy polling and NAPI execution model                           |
| BPF DEVMAP                           | Part VI 4.14; Part III XDP / AF_XDP                     | XDP redirect via devmap                                         |
| BPF CPUMAP                           | Part VI 4.15; Part III XDP / AF_XDP                     | XDP redirect to remote CPU                                      |
| nftables flowtable                   | Part VI 4.16; Part III netfilter / nftables / conntrack | software flowtable fast path; later HW-offload evolution        |
| TCX / `bpf_mprog`                    | Part VI 6.6; Part III BPF                               | link-based TC attachment and multi-program ordering             |
| Device Memory TCP / memory providers | Part VI 6.12/6.16; Part III packet memory               | RX/TX device-memory zero-copy evolution                         |
| io_uring ZCRX                        | Part VI 6.15; Part III io_uring networking              | queue-bound zero-copy receive                                   |
| `net-next-7.1`                       | Part VI 7.1                                             | RX HW queue leasing                                             |
| `net-next-7.3`                       | Part VI 7.3-rc                                          | tunnel BIG TCP, RTNL-less FIB rules, \>PAGE_SIZE devmem buffers |

## Exact anchor index

Exact SHA の正本は Part VII に置きます。Appendix では同じcommit listを複製しません。

## Conference provenance index

Conference talk は設計意図や当時のproblem statementを補足する資料として扱い、 release attributionには使用しません。

| Conference lineage                      | Canonical destination       |
|:----------------------------------------|:----------------------------|
| Netdev: XDP / AF_XDP / TC / BPF         | Part III BPF / XDP          |
| Netdev: BIG TCP                         | Part III packet aggregation |
| Netdev: Device Memory TCP / zero-copy   | Part III packet memory      |
| Netdev 0x19: Diagnosing Page Pool Leaks | Part IV page_pool           |
| Netdev: queue/NAPI/netdev-genl          | Part IV driver framework    |
| Netdev: MPTCP / TCP state-of-the-union  | Part III transport          |
| Kernel Recipes: XDP / BPF / io_uring    | Parts III–IV                |
| OVS/OVN Conf: Retis                     | Part V case-study note      |
| LPC / FOSDEM: pwru and tracing          | Part V case-study note      |

### Additional architecture / implementation conference references

- **Netdev 0x13 — devlink health reporting and recovery system:** https://netdevconf.info/wiki/doku.php?id=0x13:reports:d2t3t07-devlink-health-reporting-and-recovery-system
- **Kernel Recipes 2024 — Efficient zero-copy networking using io_uring:** https://kernel-recipes.org/en/2024/schedule/efficient-zero-copy-networking-using-io_uring/
- **Kernel Recipes 2024 — live blog (io_uring zero-copy / netmem / queue API context):** https://kernel-recipes.org/en/2024/2024/09/23/live-blog-day-1-afternoon/
- **Kernel Recipes 2024 — Interfacing Kernel C APIs from Rust:** https://kernel-recipes.org/en/2024/schedule/interfacing-kernel-c-apis-from-rust/

これらは design/context provenance であり、Part VI の release attribution には使用しない。

## Retis / pwru case-study index

Retis と pwru の詳細説明は本文から外しました。両者の位置づけは次の一文で十分です。

``` text
pwru  = broad kernel-function packet trajectory
Retis = networking-event-centric semantic enrichment / correlation
```

これらは BTF、eBPF tracing、drop reason、timestamp、OVS/OVN metadata などの kernel primitiveを利用する **consumer/tooling examples** であり、独立したkernel evolution axisではありません。

## Provenance policy

``` text
conference / RFC
      ↓ design intent
patch series
      ↓ landing evidence
subsystem pull / Linus mainline
      ↓ canonical integration
released tag
```

# Changelog / Errata（非正規）
- **r24:** RTNL の progression を `⇒` から `→` に変更。一つの feature の direct extension ではなく、global RTNL dependency を縮小する複数の locking/refactoring techniques の lineage として扱う。BIG TCP など direct extension が明確な系列の `⇒` は維持。

- **r23:** WireGuard 5.6 を `SECURITY / VIRTUAL / OVERLAY` とし、`evidence grade` の残語を `Evidence class` へ修正。netmem / memory providers の Part VI 未収載が意図的な open provenance であることを明記。
- **r23:** XDP frags と USENIX subsection の配置を修正し、io_uring networking を Packet memory の直後へ移動。Era 1 の重複 queueing 図を削減。
- **r23:** BIG TCP / RTNL / drop-reason の直接 extension に `⇒` を適用し、Virtual networking の凡例重複を削除。
- **r23:** Part VII の2つの anchor 表と MPTCP anchor を単一 inventory に統合。未監査の first-containing tag は `open` として推測を避けた。

- **r22:** r21 の編集事故で欠落した USENIX research / Virtual networking / TCP-UDP transport の3ブロックを r20 から復元。

- **r22:** r20 から落ちていた PPPoE GRO/GSO、qdisc-drop tracepoint、conference provenance、Open provenance 説明を復元。

- **r22:** Part VI の Era 見出しを release range のみに戻し、WireGuard の cross-cutting domain を VIRTUAL / OVERLAY に整理。Part V の observability timeline を canonical milestone に合わせて更新。

- **r21:** r20 の編集事故で欠落した Part I 前半と Part III `XDP / AF_XDP` を r19 から復元し、r20 の Queueing / Packet memory 改善だけを選択的に再適用。

- **r21:** Part VI の採用基準を一本化し、長期 architecture story に役割を持たない10 milestone を canonical table から除外。TCP MSG_ZEROCOPY と XDP frags は本文 lineage に明示して維持。

- **r21:** Domain 定義、Era-label 残骸、Programmability foundation の重複、Part V qdisc-drop 配置、Appendix の編集履歴見出しを整理。


- **r20:** Packet memory と Queueing を thesis の中核 lineage として拡充。netmem / memory providers / AQL / taprio / cake_mq / large-buffer devmem を本文へ統合。

- **r20:** Part VI の除外基準を復活させ、本文の長期 lineage を変えない局所 milestone を整理。

- **r20:** Part I の検証表に previous state と Era 3 例を追加し、Snapshot 行名を6軸へ統一。

- **r20:** 宙に浮いていた Netdev 0x13 / Kernel Recipes 2024 provenance を Appendix に復元。

- **r20:** Part IV projection と Part V observability projection を Part VI の分類に合わせて再整理。

- **r19:** TCP/UDP 節を transport semantics に限定し、BIG TCP / Device Memory TCP / BPF CC / zero-copy を各 architecture lineage へ戻した。

- **r19:** Virtual networking を datapath と queue-assignment の parallel lineage に再描画し、`netkit → queue leasing` の派生表現を廃止。

- **r19:** Part IV の Rust 本文を architecture 上の意味に圧縮し、conference/review の細部を Appendix 側へ寄せた。

- **r19:** Part VI の range 見出しから Era 名を除去し、Era=Part II / chronology=Part VI の役割分担を明確化。

- **r18:** Part VI を Axis / Domain の独立列へ変更し、Era/Grade 列を削除。束ね行を分割し、Evidence class を唯一の evidence-status 列とした。

- **r18:** Part I の baseline を v3.0 / v5.0 / 2026 に整理し、軸→本文節の地図と図記法を追加。

- **r18:** PERFORMANCE の queueing lineage を独立化し、io_uring / 6.19 cross-lineage の重複を整理。

- **r18:** Part IV から raw SHA / release-attribution / 編集経緯を除去し、Part VII の open items を未解決 provenance に限定。

この節は canonical chronology / provenance の一部ではない。版間の編集上の変更だけを記録する。

- **r17:** Part VI を1行1 milestoneへ正規化し、Axis/domain・Era・Evidence class を追加。 evidence 語彙を統制し、selection policy を Part VI 冒頭へ移動。
- **r17:** Part I に axis/contract 対応表と thesis の非適用領域を追加。
- **r17:** Part IV の major-version 見出しを architecture-oriented headings に変更。
- **r17:** conclusion/synthesis を chronology の前に追加。
- **r17:** Part VII から旧版監査経緯を分離し、短縮 SHA を exact inventory として扱わない方針を明記。
