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

この「複数モデルの共存」を可能にした主要な architecture mechanism を、本書では **explicit resource / control contracts** と捉える。driver-private、implicit、global だった仕組みを kernel 共通の object、API、accounting、assignment、lifetime、synchronization contract として明示することで、従来の skb/system-RAM path を残したまま programmable / zero-copy / offloaded / device-memory path を選択的に接続できるようになった、というのが本書の中心命題である。

### 本書で固定する語彙

- **中心命題:** 複数の execution / memory-ownership model の共存
- **それを可能にした mechanism:** explicit resource / control contracts
- **contract の具体形:** object / accounting / API / assignment / lifetime / synchronization
- **重要な subtheme:** ownership（特に queue / packet memory）

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
| CONTROL PLANE     | embedded/global state → reusable objects / schema / narrower lock scope                    | object / API / synchronization       | Part III「Routing / TC / offload」「Routing / Netlink / RTNL」「netfilter」             |
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

この表は release attribution の正本ではない。version と Verification status は Part VI に従う。

### この thesis が説明しないもの

`explicit resource / control contracts` は本書が観察する強い architecture tendency であり、 Linux networking の全変更を説明する万能則ではない。本書は15年間の変化を、少なくとも **(1) resource / control contract の明示化、(2) transport algorithm / protocol semantics の進化、(3) packet-processing unit / batching の拡大、(4) implementation scalability の改善**という並行する系列として読む。TFO / DCTCP / BBR / MPTCP / PLB / AccECN は (2)、GRO/GSO / BIG TCP は (3)、`dev_queue_xmit()` llist 化は (4) の代表例であり、(1) へ還元しない。中心 thesis はこれらを排除するのではなく、**どの系列を説明しているかを限定した上で、並行進化として同じ重みで扱う**。

Observability も7番目の contract form とはしない。typed identity、drop reason、tracepoint、 timestamp などは、resource/control contract と runtime behavior を理解・検証する **evidence layer** と位置付ける。

Part II–V はこの thesis を architecture の観点から検証し、Part VI–VII が release attribution と provenance を再監査可能な形で保持する。

# Part II — Four architecture eras

ここでの era は kernel major version の境界ではなく、**外部の圧力に対して networking architecture が主に何を解こうとしたか**で区切る。境界は重なり、ある era で始まった仕組みは後続 era でも継続して発展する。

| Era | おおよその期間 | 外部の圧力 | 主問題 | 代表的な milestone |
|---|---|---|---|---|
| **Era 1 — Queue & scalability foundation** | ～2014頃 | 10/40GbE、multicore server、virtualization/container、datacenter latency | queue backlog、multicore、namespace、overlay、filtering の基礎 | BQL, CoDel/fq_codel, TSQ, pacing, namespace/setns, VXLAN, nftables, eBPF foundation |
| **Era 2 — Programmable datapath** | 2015～2018頃 | cloud networking、high packet rate、SmartNIC/offload、kernel-bypass pressure | packet path を安全に拡張し、hardware/offload と接続する | TC BPF, XDP, cgroup/LWT BPF, SOCKMAP, switchdev, devlink, AF_XDP, page_pool |
| **Era 3 — Programmability becomes infrastructure** | 2019～2022頃 | hyperscale operations、protocol experimentation、100GbE級、typed tooling | programmable mechanism を protocol / operations / memory infrastructure に広げる | BTF ecosystem, struct_ops, SK_LOOKUP, MPTCP, BIG TCP, io_uring networking, drop reason, devlink health |
| **Era 4 — Explicit placement & scoped control** | 2023～ | accelerator/device memory、400GbE級、heterogeneous execution、global-lock scalability | queue・memory・device・locking scope を明示的 object/contract として配置・制御する | netmem, memory providers, Device Memory TCP, netdev-genl queue/NAPI objects, queue leasing, per-netns RTNL, BPF qdisc, YNL |

Era 1 の foundation は4つに要約できる。**queue/scalability**（BQL・TSQ・fq/pacing）、**virtualization/control**（namespace/setns・VXLAN・route-cache removal・nftables）、**programmability**（eBPF ISA → `bpf()`/maps/verifier）、**server/transport**（SO_REUSEPORT・SO_BUSY_POLL・TFO・DCTCP）である。これは新しい taxonomy ではなく、後続 lineage の出発条件を示す要約である。

### 圧力から architecture response へ

- **Era 1:** 10/40GbE と multicore 化は「ring に積めるだけ積む」設計の latency / fairness 問題を露呈し、BQL・TSQ・pacing・busy polling のように queue と CPU locality を明示的に制御する方向を促した。同時に namespace / VXLAN は virtualization と multi-tenant networking の土台になった。
- **Era 2:** cloud datapath の多様化と packet-rate pressure は、固定された kernel path を増築し続ける代わりに、eBPF/TC/XDP の verified extension points と AF_XDP の queue-bound userspace path を発達させた。SmartNIC/switch ASIC の普及は switchdev/devlink/TC offload という hardware mapping の共通化も要求した。
- **Era 3:** programmable path が production infrastructure になると、program を「置ける」だけでは不十分になり、BTF/CO-RE、struct_ops、typed observability、BIG TCP、MPTCP、io_uring networking のように reuse・operations・processing-unit の拡張が進んだ。
- **Era 4:** 400GbE級、accelerator、device memory、heterogeneous execution は「どの queue / memory / device / lock scope を誰が使うか」を暗黙の driver convention のまま扱いにくくし、netmem / memory providers / queue objects / queue leasing / narrower RTNL scope のような explicit placement/control を押し出した。

## Era 間で変わった設計上の問い

``` text
How much should we queue?
        ↓
Where can we program the datapath?
        ↓
How do those mechanisms become reusable infrastructure?
        ↓
How explicitly can placement, lifetime and synchronization be controlled?
        queue / memory / device / locking scope
```

この矢印は feature dependency ではなく、時代ごとの**支配的な設計課題の変化**を表す。Part VI の release chronology はこの periodization とは独立に検証する。

------------------------------------------------------------------------

# Part III — Long-term lineages

Part III では release 順ではなく、同じ設計課題が長期間にどう変化したかを追う。図の矢印は、明示しない限り直接の親子関係を意味せず、dependency / extension / parallel development / shared design problem を区別する。各 milestone の release は Part VI の chronology に対応する。

## Queueing / latency / pacing — multi-layer control evolution

PERFORMANCE 軸の queueing lineage は、単に qdisc algorithm が増えた歴史ではない。socket が作る burst、qdisc が保持する backlog、driver/NIC ring に積まれた outstanding bytes、さらに multiqueue NIC での queue 選択を、それぞれ別レイヤーで制御できるようになった歴史である。

``` text
socket / transport: TSQ · TCP pacing · BBR
 ∥
qdisc / scheduler: CoDel · fq_codel · sch_fq · EDT · CAKE / cake_mq
 ∥
driver / NIC: DQL/BQL · multiqueue scaling · dev_queue_xmit() llist
```

DQL/BQL は driver が NIC へ渡した byte と完了した byte を accounting し、hardware queue に過剰な backlog を作らないための backpressure contract を与える。TSQ は同じ問題を TCP socket 側から制限し、 `sch_fq` と TCP pacing は「何個 queue に入れるか」だけでなく「いつ送るか」を扱う。

Era 1 では SO_REUSEPORT（3.9）と SO_BUSY_POLL（3.11）も multicore server の receive placement / latency を改善した。これらは qdisc の派生ではないが、work placement と待ち時間を制御する同時期の server-side foundation である。IPv4 route-cache removal（3.6）は CONTROL PLANE scalability の変更なので Routing 節で扱う。

CoDel/fq_codel と CAKE は qdisc 層で delay / fairness / shaping を扱う。EDT と `SO_TXTIME` は time-based transmission を明示し、taprio は time-aware scheduling を hardware/offload と接続する。 これらは同じ API の世代交代ではなく、queueing problem を複数層へ分解した結果である。

Wi-Fi/mac80211 では airtime accounting/scheduling と AQL が、byte queue だけでは表現しにくい wireless medium の占有時間を明示的に扱う。これは wired BQL の単純な派生ではないが、 **hidden queue pressure を accounting 可能な量へ変換する**という shared design problem を持つ。

6.19 の `dev_queue_xmit()` llist 化は contract 明示化ではなく、shared-qdisc / multiqueue TX の implementation scalability を改善する milestone である。7.0 の `cake_mq` は CAKE を multi-queue-aware に拡張し、modern NIC の queue topology と qdisc control の接点を強める。

**Takeaway:** queue occupancy と completion を accounting/API として明示し、各 layer が backpressure と scheduling を独立に制御できるようになった。


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
 → IPv4 BIG TCP (6.3)
 → IPv6 BIG TCP without synthetic HBH jumbo header (Linux 7.0)
 → BIG TCP over VXLAN / GENEVE (7.3-rc / mainline)
```

PPPoE GRO/GSO のような encapsulation-specific aggregation も、同じ PERFORMANCE 軸で per-packet cost を減らすが、BIG TCP の派生機能ではない。

BIG TCP は wire MTU を巨大化する機能ではなく、 **kernel 内部の packet-processing unit を大きくする機能**として理解する。利用可否と効果は protocol path、 GRO/GSO/offload capability、driver/NIC、tunnel implementation などに依存し、すべての device / path で 一律に大きな aggregate を利用できることを意味しない。


**Takeaway:** aggregation は主に explicit resource contract の系列ではなく、GRO/GSO/BIG TCP により **\1 回の stack traversal で扱う processing unit を拡大する並行系列**である。

------------------------------------------------------------------------

## BPF — packet filter から stack extension へ

BPF lineage の起点は XDP ではない。classic BPF は socket/filtering 文脈の小さな packet-filter VM だったが、3.15 世代の internal eBPF ISA rework と 3.18 の `bpf()` syscall / maps / verifier により、**userspace が verified program と persistent map object を kernel にロードする共通 execution model**へ変わった。3.19 の `SO_ATTACH_BPF` は socket filtering を新しい eBPF program model へ接続し、その後 TC、cgroup/LWT、socket hooks、XDP、`struct_ops`、TCX、qdisc へ attachment point が広がった。

``` text
classic BPF packet filter
 ↓
internal eBPF ISA (3.15)
 ↓
bpf() / maps / verifier (3.18)
 ├─ SO_ATTACH_BPF (3.19) / TC BPF (4.1...)
 ├─ cgroup / LWT / XDP
 ├─ SOCK_OPS (4.13) / SOCKMAP
 └─ struct_ops / TCX / BPF qdisc
```

このため「packet filter から stack extension へ」という節題は、programming language の置換ではなく、**verified execution contract と attachment model の一般化**を表す。


BPF の発展は一本道ではなく、attachment point と適用範囲が複数方向へ増えたものとして捉える。

TC direct packet access、devmap/cpumap、SK_MSG、flow dissector、`struct_ops`、SK_LOOKUP、TCX は、 packet/datapath、socket/message、protocol algorithm、attachment/lifetime という異なる面を拡張した。 一本の attachment-point lineage としてではなく、verified programmability の適用範囲が広がったものとして読む。

``` text
 BPF core / verifier / maps
 │
 ┌─────────────────────┼─────────────────────┐
 ▼ ▼ ▼
 packet / datapath socket / lookup protocol algorithms
 XDP, TC cgroup hooks, struct_ops / TCP CC
 SK_LOOKUP
 │ │
 └──────────────┐ ┌──────────┘
 ▼ ▼
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

これらの研究は mainline milestone そのものではないが、kernel bypass と in-kernel programmability の設計上の trade-off を理解する補助線になる。


**Takeaway:** extension point、program type、map、typed operations を API/object として明示した。


------------------------------------------------------------------------

## XDP / AF_XDP

XDP は 4.8 世代に、driver RX の非常に早い位置で verified BPF program を実行する datapath として mainline に現れた。重要なのは単なる「高速 \1」ではなく、その後 redirect target と memory ownership が段階的に増えたことである。4.14 の DEVMAP、4.15 の CPUMAP は redirect を device / remote CPU という明示的 target object に広げ、4.18 の AF_XDP は RX queue と UMEM/userspace buffer を結び付ける queue-bound datapath を追加した。

``` text
XDP at driver RX (4.8)
 ├─ redirect → DEVMAP (4.14)
 ├─ redirect → CPUMAP (4.15)
 ├─ packet memory → page_pool
 └─ RX queue ↔ AF_XDP / UMEM (4.18)
 ├─ zero-copy
 ├─ multi-buffer (6.6)
 └─ queue ownership / later netkit integration
```

4.8–4.18 は、後年の queue leasing や memory-provider architecture につながる queue / redirect / buffer-binding の前史として重要である。


その後は native/generic/hardware-offload mode、page_pool、multi-buffer、AF_XDP zero-copy など周辺 infrastructure が成熟した。XDP → AF_XDP → netkit / queue leasing を単純な派生関係とはみなさない。XDP は execution placement、AF_XDP や queue leasing は queue/buffer ownership という共通課題から並行して発展した面を持つ。

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


**Takeaway:** verified execution point に加え、redirect target、RX queue、UMEM/buffer ownership を object/assignment として明示し、kernel path と userspace path の共存を可能にした。

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
 ↑ │
 └──────── return/recycle ─────┘
```

page_pool は XDP 専用ではない。XDP と強く結び付いて普及したが、後の zero-copy RX、 memory-provider、device-memory path が driver ごとの独自 allocator に戻らずに済むための common infrastructure として読む。

### netmem — `struct page` と packet memory を同一視しない

system RAM の page だけを packet backing と仮定すると、device memory や userspace-owned memory を 同じ networking path で扱いにくい。`netmem` は networking が扱う memory identity を `struct page` そのものから切り離す方向を表す。

``` text
network packet memory
 ↓
 netmem
 / \
system RAM non-page / device-backed memory
```

ここで重要なのは「RAM を device memory に置換した」ことではなく、従来 path を維持したまま 異なる backing memory を表現できる contract を追加したことである。

### memory providers / queue binding — memory lifetime と RX queue を接続する

Device Memory TCP RX は6.1\1 で先に mainline 化され、この時点では page_pool の `mp_priv` など devmem 用の専用経路を利用していた。6.1\1 で `memory_provider_ops` / page_pool custom-provider hooks が導入され、devmem TCP と io_uring ZCRX が同じ **汎用 memory-provider contract** に接続できる形へ整理された。したがって `Device Memory TCP RX (6.12) → generic memory-provider hooks (6.15)` は、**consumer-specific path の先行 → provider API の一般化**という順序である。

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


**Takeaway:** packet buffer の lifetime・provider・queue assignment を driver-private convention から共通 contract へ移した。


## io_uring networking

io_uring networking は、socket I/O を一回ごとの syscall から submission/completion と登録済み resource の model へ移し、network buffer の lifetime と queue binding を userspace-visible な contract として扱う方向へ進んだ。6.0 世代の SEND_ZC / multishot receive は TX copy avoidance と receive batching を進め、6.15 の ZCRX は RX queue と userspace-owned memory の直接的な結び付きを追加した。

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

目標は syscall reduction だけではなく、 NI\1 から \\1 までの buffer ownership/lifetime の効率化にある。


**Takeaway:** socket I/O を submission/completion と registered resources に分解し、ZCRX では queue と buffer/memory-provider の関係まで明示する方向へ進んだ。

------------------------------------------------------------------------

## Routing / TC / offload — forwarding semantics と hardware mapping の並行進化

routing / forwarding control は BPF だけでは説明できない。bridge VLAN filtering、MPLS、VRF は Linux 内部の forwarding domain / lookup semantics を明示し、Flower と TC `ct` action は packet field と conntrack state を TC pipeline の match/action model に持ち込んだ。これらは後の hardware offload と接続するが、すべてが switchdev を経由するわけではない。

``` text
bridge VLAN filtering (3.9) ─┐
MPLS routing (4.1) ├─ Linux forwarding / routing semantics
VRF device (4.3) ┘

Flower classifier (4.2) → TC match/action → ndo_setup_tc / flow-block callbacks → driver / hardware
 ↘ TC ct action (5.3): conntrack state/metadata in TC

bridge / FIB / VLAN objects → switchdev notifications / objects → switch driver / hardware
```

ここで重要なのは、**switchdev と TC offload を一本の経路として扱わない**ことである。bridge/FDB/VLAN/FIB の object/notification path と、TC classifier/action の `ndo_setup_tc` / flow-block callback path は driver/hardware で合流し得るが、kernel API としては並行する経路である。

**Takeaway:** forwarding semantics を kernel の canonical state として保ちながら、bridge/FIB/TC policy を software または hardware execution へ写像できるようになった。switchdev と TC offload は同じ直列 API ではなく、別の control path が driver/hardware で交差する。

### XFRM packet offload — security state と forwarding placement

6.2 の XFRM packet offload は、従来の crypto offload より広く packet-level IPsec processing を \\1 へ配置する仕組みである。kernel/XFRM が policy/state の \1 control plan\1 を保持しながら、executio\1 を \\1 または \\1 へ置ける点で switchdev/TC offload と共通する。security semantics は独立した cross-cutting domain であり、ここでは hardware execution placement との接点だけを扱う。

## Routing / Netlink / RTNL

Routing/control-plane scalability では、共有状態の削減、再利用可能な routing object、machine-readable API、lock scope の縮小が並行して進んだ。IPv4 route-cache removal（3.6）は per-destination shared cache を外した早い転換点であり、nexthop object や per-netns RTNL の直接の祖先ではないが、global/shared state を減らすという同じ scalability pressure に応えた。

6.19 の WireGuard YNL-described Netlink は、machine-readable Netlink schema が個別 subsystem へ広がった例である。

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
 → RTNL-less FIB rule updates (7.3-rc / mainline)
```

この `→` は direct extension を意味せず、global RTNL dependency を縮小する複数の locking / refactoring techniques が同じ設計方向に進んだことを示す。

7.\1 向け \1 pul\1 では `RTM_NEWRULE` / `RTM_DELRULE` の \1 rul\1 変更が RTNL-lock-les\1 化され、further RTNL-dependency reductio\1 や \1 GE\1 準備と 同じ「global RTN\1 依存を減らす」流れとして \\1 へ入った。

さらに 7.3-rc / mainline では per-netns netdev unregistration infrastructure が入り、 per-netns RTNL の大きな blocker だった device unregistration path の分解も進んだ。 これは 6.13 以降の per-netns RTNL lineage の継続として扱う。


**Takeaway:** routing object と synchronization scope を明示し、global RTNL dependency を縮小する方向へ進んだ。


------------------------------------------------------------------------

## netfilter / nftables / conntrack — rule engine から policy-selected fast path へ

Netfilter の1\1 年間を `iptables → nftables` という userspace command の置換として見ると本質を取りこぼす。変化の中心は、既存の hook / conntrack / NAT を残しながら、**ruleset representation、state、policy placement、fast-path eligibility、hardware placement を明示化したこと**にある。

``` text
iptables / ip6tables / ebtables / arptables
 ↓
nftables core (3.13)
 ├─ expression / VM-based rules
 ├─ Netlink control plane
 ├─ sets / maps
 └─ transactional ruleset updates
 ├─ ingress hook / netdev family (4.2)
 ├─ bridge-family + conntrack
 └─ flowtable (4.16)
 ├─ software fast path
 └─ hardware offload (5.5)
```

### nftables — ruleset model の再設計

nftables は3.1\1 で mainline に入り、family ごとに重複していた xtables machinery を汎用 expression/VM と Netlink API に寄せた。重要なのは syntax ではなく、atomic/transactional update、sets/maps、monitoring/tracing を含む **ruleset object model** への移行である。4.2 の ingress hook、5.16 の netdev egress hoo\1 は、Netfilter polic\1 を適用できる datapath point を広げた。

### conntrack — shared flow-state substrate

conntrack は nftables に置換されたものではない。flow direction/state、NAT mapping、mark/label、timeout/lifetime を保持し、nftables、TC `ct`、BPF、flowtable が共有する state substrate である。

``` text
packet → Netfilter hook → conntrack
 ├─ state / direction
 ├─ NAT mapping
 ├─ mark / label
 └─ lifetime
 ├─ nftables
 ├─ TC ct
 ├─ BPF kfunc
 └─ flowtable
```

4.7 世代では conntrack hash table の scope が全 netns 共有へ寄せられ、entry accounting は netns 単位に残った。4.9 世代では GC worker による timeout entry 回収が導入された。5.18/6.0 世代には BPF から conntrack entry を lookup / allocate / insert する kfunc も加わり、state は firewall 専用ではなく複数 execution model から利用される object になった。

### flowtable — policy と fast path の分離

4.16 の flowtable では、最初の packet が classic forwarding / conntrack / nftables policy を通り、選択された established flow が fast path に登録される。hi\1 時は後段の classic forwarding hooks を bypas\1 して `neigh_xmit()` へ進み、miss、fragment、FIN/RST、MTU exception などは classic pat\1 へ戻る。

``` text
 miss / exception
ingress → flowtable lookup ─────────→ classic forwarding
 │ │
 │ hit │ policy / conntrack
 ▼ │
 fast forwarding ←── flow add ─────┘
 │
 ▼
 neigh_xmit()
```

5.5 世代では hardware offload に拡張され、flow は `flow_rule` として driver/hardware datapath に写像される。hardwar\1 投入は非同期なので、完了まで software path を通る packet もある。これは fast path が policy engine を置換したのではなく、**policy/state を共有しながら software slow path・software fast path・hardware path を共存させた**例である。

### BPF / XDP との関係

BPF/XDP も nftables の置換ではない。bpfilter は iptables-compatible ruleset を BPF へ変換する構想として4.1\1 に骨格が入ったが6.\1 で削除された。一方、定着したのは conntrack kfunc、6.4 の `BPF_PROG_TYPE_NETFILTER`、flowtable lookup kfunc のように既存 Netfilter infrastructure と相互乗り入れする方向である。

**Takeaway:** Netfilter では policy representation、transactional update、shared flow state、fast-path eligibility、software/hardware execution placement が明示化された。これは CONTROL PLANE から「explicit contracts が複数 execution model の共存を可能にする」ことを示す代表例である。

------------------------------------------------------------------------

## Virtual networking — datapath と queue assignment の並行進化

この lineage の前史は v5.0 よりかなり早い。network namespace と `setns()` は network stack instance を process/container 単位に切り替える isolation/control primitive を与え、VXLAN（3.7）は L3 underlay 上に L2 overlay を構成する一般的な tunnel device を、ipvlan（3.19）は veth/macvla\1 とは異なる lightweight virtual interface model を追加した。これらは後年の netkit、vDPA、queue leasing の直接の祖先ではないが、**一つの physical network device / host stack の上に複数の virtual networking model を共存させる foundation**である。

``` text
network namespace / setns
 ├─ veth / bridge
 ├─ VXLAN overlay (3.7)
 └─ ipvlan (3.19)
 ↓
later: virtio/vhost/vDPA, SR-IOV, netkit, queue assignment
```


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


**Takeaway:** virtual device / datapath を一つに統一するのではなく、namespace/device/queue assignment を接続点として virtio/vhost/vDPA、SR-IOV、AF_XDP、netkit などを共存させた。

------------------------------------------------------------------------

## TCP / UDP / transport

この節は **transport protocol 自体の semantics / feedback / path management** に絞る。 TCP 上で使われるという理由だけで、memory、aggregation、programmability の milestone を ここへ再収容しない。

### Transport protocol evolution

| Theme | Representative milestones | この節で追う意味 |
|-------------------------------|----------------------------|--------------------------------------------|
| connection establishment | TFO client / server | handshake latency / early data |
| congestion control / feedback | DCTCP · BBR · PLB · AccECN | congestion signal と sender behavior |
| multipath | MPTCP | 複数 path / subflow の transport semantics |
| authentication | TCP-AO | TCP connection authentication |

これらも一本の派生系列ではない。

TFO は handshake と application data の境界を変え、connection establishment latency を削減した。DCTCP は ECN marking を datacenter congestion feedback として積極的に利用し、BBR は loss を主信号とする従来型とは異なり bandwidth / RTT model に基づいて sending behavior を決める。PLB は ECMP 環境で persistent congestion を検出した flow の path を transport 側から再選択する。AccECN は CE feedback の情報量を増やし、より細かな congestion signal を transport に返す。

MPTCP は一つの論理 connection の下に複数 subflow / address / path-manager state を持つ transport へ TCP semantics を拡張した。TCP-AO は connection authentication を現在の TCP option / key-management model に更新する。これらは packet-memory や BPF execution contract とは異なり、**transport が保持する connection / congestion / path / authentication semantics 自体**を変更する lineage である。

``` text
connection establishment: TFO
congestion control / feedback: DCTCP ∥ BBR ∥ PLB ∥ AccECN
multipath: MPTCP
authentication: TCP-AO
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


**Takeaway:** transport は explicit-contract thesi\1 だけでは説明しない。TFO、DCTCP、BBR、MPTCP、PLB、AccECN は主に connection / congestion / path / feedback semantics 自体を更新する並行系列である。

------------------------------------------------------------------------

## Part III から Part IV へ — feature lineage から driver contract へ

前章では networking mechanism を end-to-end の lineage として追った。Part IV では視点を変え、個々の driver の責務のうち何が共通 networking-core framework へ移されたかを見る。

# Part IV — Network Device Driver Framework

Part III が packet path / control path の長期 lineage を追ったのに対し、Part IV は **driver-local な実装知識がどのように共通 framework / object / API へ引き上げられたか**を見る。ここで重要なのは、個々の driver の高速化ではなく、NIC が持つ queue、link、switch、health、memory、interrupt moderation などの能力を networking core から共通に扱えるようにしたことである。

この流れは一つの framework が他を置き換えた歴史ではない。

``` text
driver-local convention
   │
   ├─ queue accounting ───────→ DQL/BQL
   ├─ link topology ──────────→ phylink
   ├─ switch datapath ────────→ switchdev
   ├─ device control/health ──→ devlink
   ├─ interrupt moderation ───→ DIM
   ├─ packet memory ──────────→ page_pool → netmem / providers
   └─ queue/NAPI identity ────→ netdev-genl
```

これらは別々の subsystem だが、共通する方向は **「driver 内部にしか見えなかった resource/state を kernel 共通の contract として表現する」**ことである。

## Queue accounting — DQL/BQL

DQL/BQL は driver ring の大きさそのものを標準化する framework ではない。driver/NIC に渡した bytes と completion された bytes を accounting し、software queue が device に過剰な backlog を押し込むのを抑える仕組みである。

Part III の queueing 節では BQL を latency / pacing の一部として扱った。driver-framework の観点では、より重要なのは **各 driver が経験則で決めていた outstanding work を共通 accounting model にしたこと**である。

``` text
qdisc
  │
  ▼
driver queue
  │   enqueue bytes
  ├──────────────→ DQL/BQL accounting
  │                    ▲
  ▼                    │ completed bytes
NIC ring ───────────────┘
```

したがって BQL の contract form は ownership ではなく **accounting / backpressure** である。

## switchdev — kernel forwarding object と switch ASIC

switchdev は Linux bridge/FDB/VLAN などの kernel forwarding state を switch ASIC に同期するための framework として発展した。目的は「hardware switch を特別な別世界として管理する」ことではなく、Linux networking object を canonical control state として維持しながら forwarding execution を hardware に配置できるようにすることである。

``` text
Linux bridge / FDB / VLAN / FIB objects
                  │
                  ▼
        switchdev notifications
                  │
                  ▼
             switch driver
                  │
                  ▼
                 ASIC
```

ただし TC hardware offload はこの直列 path の一部ではない。TC classifier/action の offload は `ndo_setup_tc` や flow-block callback など別の kernel API path を通り、driver/hardware 側で switchdev 系の forwarding object と交差し得る。

``` text
bridge / FIB / VLAN objects                 TC classifier / actions
          │                                          │
          ▼                                          ▼
switchdev notifications / objects          ndo_setup_tc / flow-block callbacks
          │                                          │
          └───────────────┐          ┌───────────────┘
                          ▼          ▼
                      switch / NIC driver
                              │
                              ▼
                           hardware
```

この区別は Part III の Routing / TC / offload 節で説明した control-path separation を、driver API 側から見たものである。

## devlink — device-wide control object

devlink は port/netdev 単位だけでは表しにくい **device-wide resource / parameter / port / health state** を扱う object として導入された。特に switch ASIC や SmartNIC のように、一つの device が複数 port、representor、resource、firmware state を持つ場合、netdev だけでは device 全体の control plane を表現しにくい。

``` text
PCI / physical device
        │
        ▼
     devlink
   ├─ ports
   ├─ resources
   ├─ parameters
   ├─ regions
   └─ health reporters
        │
        ├─ netdev / representor
        └─ driver / firmware / ASIC
```

devlink health はこの object model を observability/recovery に広げた。driver-specific debug command の集合ではなく、reporter、diagnose、dump、recover という共通 interface を通して failure state を扱う点に意味がある。

devlink は datapath 自体を提供しない。switchdev/TC offload が「どこで forwarding を実行するか」を扱うのに対し、devlink はその device の **resource / configuration / health lifecycle** を扱う。

## phylink — link topology と MAC/PHY/PCS coordination

phylink は MAC driver が PHY、fixed-link、SFP、PCS などの link topology を個別に扱う重複を減らす framework である。4.14 世代に導入され、link negotiation / mode change / carrier state に関する共通 orchestration を networking core 側へ引き上げた。

``` text
MAC driver
    │
  phylink
 ┌──┼──────────────┐
 ▼  ▼              ▼
PHY PCS        fixed-link / SFP
```

重要なのは「PHY APIを置き換えた」ことではない。MAC と link-side component の関係を framework が仲介し、driver が topology ごとの state machine を重複実装する必要を減らしたことである。

Rust PHY abstraction もこの lineage 上に置ける。Rust networking support の初期段階では、high-performance NIC driver 全体を書き換えるより、PHY abstraction のような比較的明確な interface boundary から型安全な binding/abstraction を作る方向が先行した。

## DIM — interrupt moderation policy の共通化

Dynamic Interrupt Moderation (DIM) は packet/byte/event rate を観測し、interrupt coalescing parameter を workload に応じて調整する仕組みである。複数 driver が似た adaptive moderation logic を独自実装する代わりに、測定と profile selection を共通 library/framework に寄せた。

``` text
packet / byte / event samples
          │
          ▼
        DIM
          │
   profile decision
          │
          ▼
driver coalescing parameters
```

これは datapath ownership の contract ではなく、**measurement → policy decision → driver setting** を再利用可能にした framework 化の例である。

## page_pool — packet-memory lifecycle の共通化

page_pool の詳細な発展史は Part III「Packet memory」で扱う。ここでは driver framework としての意味だけに絞る。

従来、RX driver は page allocation、DMA mapping、recycling、fragment handling を driver ごとに組み合わせていた。page_pool は RX packet memory の allocation/recycling lifecycle と DMA-aware handling を共通化し、XDPを含む高速RX pathから利用できる framework を提供した。

``` text
RX queue
   │
   ▼
page_pool
   ├─ allocate
   ├─ DMA-aware lifecycle
   └─ recycle
        │
        ▼
      driver
```

後年の netmem / memory providers / Device Memory TCP は page_pool の単純な後継ではない。しかし page_pool が **packet memory lifetime を driver-private convention から共通 objectへ移した**ことが、system RAM以外のmemory providerを扱うための重要な前提になった。

ここではPart IIIと同じrelease chronologyを繰り返さず、driver-frameworkとしての役割だけを保持する。

## netdevsim — framework API を hardware なしで検証する

netdevsim（4.16）は実NICを模倣するための一般的な emulator というより、networking core / driver API を **hardware independent にselftestできる test device** として重要である。devlink resource、FIB offload、rate object など、driver-facing contract が増えるほど「特定vendor hardwareなしにAPI semanticsを検証できること」がframework evolutionの一部になる。

Kernel documentation: https://docs.kernel.org/networking/devlink/netdevsim.html

## auxiliary bus — device 内部機能の分割

auxiliary bus は、一つのphysical device/PCI functionに含まれる複数の機能を、親driverと補助driverの間で分離して扱うための共通 infrastructure である。networking専用ではないが、SmartNIC/RDMA/network driverのように一つのdeviceが複数subsystemへ機能を公開する構成で重要になった。

これは devlink のような userspace-visible control object とは役割が異なる。auxiliary bus は **driver composition / binding boundary** を提供する。

## threaded NAPI — execution context の選択肢

NAPI は通常 softirq context でpollされるが、threaded NAPI はpoll処理をkernel threadで実行できる選択肢を追加した。これはXDPやAF_XDPのような新しいpacket pathではなく、既存NAPI processingの **execution context / scheduling placement** を変える仕組みである。

real-time性、CPU scheduling、isolationの要件によってsoftirqとthreaded executionを選択できる点が重要であり、「一つの最適なNAPI execution model」へ統一する変更ではない。6.19 世代では threaded NAPI に busy-poll mode が加わり、threaded execution の中でも IRQ-driven と polling-oriented な配置を選べる方向へ進んだ。

Kernel netdev specification: https://docs.kernel.org/7.1/netlink/specs/netdev.html

## ethtool netlink と YNL — driver control API の構造化

従来のethtool ioctl interfaceは長年利用されてきたが、機能追加、dump、notification、extensible attributeという面ではNetlinkの方が扱いやすい。ethtool netlinkはlink modes、coalescing、channelsなどのdevice configurationをstructured Netlink APIへ移す方向を示した。

さらにYNLはNetlink familyのschemaをmachine-readableに記述し、policy/documentation/userspace helper生成へ接続する。ethtool netlinkとYNLは同じfeatureではないが、

``` text
ad-hoc ioctl / hand-written Netlink
              ↓
structured Netlink objects
              ↓
machine-readable schema / generated tooling
```

というcontrol APIの長期的な方向を示す。

## netdev-genl — queue / NAPI identity を control-plane object へ

netdev generic Netlink familyでは、NAPI instanceやRX/TX queueのidentityをuserspaceから列挙・参照できる方向が進んだ。これは単なるobservability enhancementではない。AF_XDP、io_uring ZCRX、memory providers、queue leasingのように **特定queueへresourceをbindingするAPI** が増えると、queueそのものを安定して指し示すcontrol-plane identityが必要になるためである。

``` text
physical netdev
   ├─ NAPI object
   ├─ RX queue object
   └─ TX queue object
          │
          ├─ statistics / introspection
          └─ later queue-bound APIs
```

ここでqueue identityはOBSERVABILITYとCONTROL PLANEの接点になる。


## Driver framework の synthesis

15年間を通して見ると、driver framework の変化は次のように要約できる。

| 以前 driver 内に埋もれていたもの | 共通化した代表 framework | 代表 release | 主な contract | Source |
|---|---|---:|---|---|
| outstanding TX work | DQL/BQL | 3.3 | accounting / backpressure | Part VII / source index |
| switch forwarding state | switchdev | 3.19 | object / notification | Part VII / source index |
| device-wide resource | devlink | 4.6 | object / API | kernel devlink docs |
| MAC–PHY/PCS topology | phylink | 4.14 | API / state coordination | Part VII |
| hardware-independent API test | netdevsim | 4.16 | selftest / contract validation | kernel netdevsim docs |
| interrupt moderation logic | DIM | 4.16→5.3 | measurement / policy | Part VII |
| RX packet-memory recycling | page_pool | 4.18 | lifetime / object | Part VII |
| device health/recovery | devlink health | 5.1 | lifecycle / observability | kernel devlink-health docs |
| NIC configuration | ethtool netlink | 5.6 | structured API | source index |
| multi-function driver binding | auxiliary bus | 5.11 | composition / lifetime | source index |
| NAPI execution placement | threaded NAPI | 5.12; busy-poll 6.19 | execution choice | source index |
| queue / NAPI identity | netdev-genl | 6.8+ | object / assignment | kernel netdev spec |


この表は各frameworkが同じ問題を解くという意味ではない。共通するのは、**driver-privateだったstate/resource/algorithm boundaryをkernel共通のcontractへ引き上げ、複数driver・複数hardware・複数execution modelから再利用可能にしたこと**である。

Part IIIのMEMORY/PROGRAMMABILITY/CONTROL PLANEが「datapath側で何が共存できるようになったか」を説明するのに対し、Part IVはその共存を支える **driver-facing contract surface** がどう増えたかを説明する。

------------------------------------------------------------------------

# Part V — Observability / Explainability

Part V は **kernel 側の observability primitive の進化**に限定する。Observability は独立した軸であると同時に、programmable / offloaded / zero-copy path が増えて複雑化した networking を **operationally explainable にする evidence layer** と位置付ける。 Retis / pwru はこれらを利用する case study であり、kernel release chronology そのものではないため Appendix に置く。

## Kernel observability primitive

Linux networking の observability は、単純な interface counter / packet capture から、 kernel 内部の typed event と packet-lifecycle metadata を相関できる方向へ進化した。

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

したがって cross-layer packet journey は単一 primitive の機能ではなく、tool が複数の identifier、event、 timestamp、metadata を相関して **推論する目標**として扱う。

### BTF と eBPF tracing

BTF により running kernel の型情報を利用できるため、observability tool は private kernel structure の固定offsetに依存する必要を減らせる。これは networking のように内部構造の変化が 速い領域で特に重要である。

### Structured drop reason

Linux 5.17 の `kfree_skb_reason()` / `skb_drop_reason` 世代は、「packet が消えた」という観測を 「どの理由でdropされたか」という structured metadata へ変えた。その後、coverage は networking stack の各所へ拡張されている。

7.1 の dedicated qdisc-drop tracepoint は、qdisc 内の drop context を generic tracing から直接観測しやすくした例であり、structured drop reason と補完関係にある。

### Timestamping と packet lifecycle

`SO_TIMESTAMPING`、driver/hardware timestamp、BPFから取得できる時刻・contextは、 単一地点のpacket captureでは見えない queueing / scheduling / offload の時間軸を補う。

### Queue / NAPI / page_pool observability

Part IVで説明した queue、NAPI、page_pool のobject化は、control planeだけでなくobservabilityにも 寄与する。packet memory、polling context、queue identityをuserspace-visible objectとして関連付ける ことで、zero-copy / memory-provider時代の問題を説明しやすくなる。ただし、readable な identity / stats と、 userspace が queue や NAPI の ownership/configuration を変更できる control-plane capability は区別する。

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

Retis と pwru は上記primitiveを利用する代表例だが、本書ではkernel evolutionそのものと区別する。

- **pwru**: 広いkernel function trajectoryから「packetがどこを通ったか」を探索する。
- **Retis**: networking event、skb metadata、OVS/OVN contextなどを意味的にenrichして相関する。

詳細なconference/source provenanceはAppendixのcase-study indexに集約する。

# Synthesis — thesis への回帰と未完の仕事

Part I の中心命題は2段からなる。Linux networking は従来の system-RAM + `skb` path を置き換えたのではなく、複数の execution / memory-ownership model を**共存**させる architecture へ拡張した。そして、その共存を可能にした主要な mechanism が **explicit resource / control contracts** である。

Part II–V を通過すると、この2段のつながりを具体的に言える。新しい path は既存 path の隣に独立して追加されたのではない。driver-private / implicit / global だった state が共通 contract として明示された箇所を**接続点**として、そこに追加された。

## Contract が共存を可能にした接続点

| 明示化された対象 | Contract の lineage | その接続点で共存できるようになったもの |
|---|---|---|
| queue occupancy | DQL/BQL → TSQ / `sch_fq` / pacing | socket・qdisc・driver が backlog を別々の層で制御する multi-layer control |
| execution point | `bpf()` / maps / verifier → XDP・TC・cgroup・socket hooks → `struct_ops` / TCX / BPF qdisc | built-in stack と verified program が同じ packet・socket・algorithm を扱う |
| packet memory | page_pool → netmem → memory providers | system-RAM page、userspace-owned buffer、device memory が同じ RX architecture に接続される |
| queue identity | AF_XDP queue / UMEM binding → netdev-genl queue/NAPI objects → RX HW queue leasing | NIC queue を kernel stack、AF_XDP、io_uring ZCRX、device-memory path、virtual device が用途に応じて使い分ける |
| forwarding / device state | switchdev・devlink → representor / TC offload → XFRM packet offload | kernel が canonical semantics/control state を保持しながら software path と hardware path を選べる |
| control-plane state | nexthop objects → YNL → per-netns RTNL | reusable object / machine-readable API / narrower synchronization scope の下で configuration を並行化する |
| policy / flow state | nftables → conntrack consumers → flowtable / HW offload | transactional policy と shared state を保ちながら slow / fast / hardware path を選択する |

表中の `→` は図の記法どおり lineage continuation であり、直接依存を意味しない。また各行は「一つの contract が後続機能を直接生んだ」という因果を主張するものではなく、**共存を可能にした architecture 上の接続点**を要約している。

この表の各行は Part I の6軸と一対一には対応しない。6軸は「何が変わったか」の分類であり、この表は「何が明示されたから、何が共存できたか」という読み方である。OBSERVABILITY は独立した行ではなく、typed drop reason、queue/NAPI identity、page_pool introspection などを通じ、これらの接続点で実際に何が起きているかを検証する evidence layer として全行にかかる。

## Contract に還元しない並行系列

15年間の変化のすべてが上の表に収まるわけではない。**transport / protocol** では TFO・DCTCP・BBR・MPTCP・PLB・AccECN が接続確立、輻輳制御、multipath、feedback semantics を更新した。**processing unit** では GRO/GSO と BIG TCP が1回の処理で扱う単位を拡大した。**implementation scalability** では `dev_queue_xmit()` の llist 化のように、contract を変えずに実装を速くする変更が続いた。**routing / forwarding** では bridge VLAN filtering、MPLS、VRF、Flower、TC `ct` action が software forwarding semantics と hardware offload の接点を広げた。

これらは thesis の例外ではなく、別の説明を要する並行系列である。本書の thesis は、どの系列を説明しているかを限定した上で成り立つ。

## 6軸への回帰

- **PERFORMANCE:** queueing は driver-local tuning から accounting / pacing / multi-layer scheduling へ進み、同時に BIG TCP や scalable TX のような contract 外の高速化も進んだ。
- **PROGRAMMABILITY:** BPF/XDP は単一 fast path から socket / protocol / qdisc / virtual-device へ広がり、再利用可能な infrastructure になった。
- **MEMORY:** page_pool から netmem / memory providers / device memory へ、allocation・lifetime・assignment が明示的になった。
- **CONTROL PLANE:** reusable objects、transactional policy、YNL schema、per-netns / fine-grained locking により embedded / global state の範囲が縮小した。
- **OBSERVABILITY:** typed reason / identity / tracing は、複雑化した programmable / offloaded / zero-copy path を operationally explainable にする evidence layer になった。
- **DRIVER FRAMEWORK:** BQL、switchdev、devlink、phylink、DIM、page_pool、netdev-genl は driver-local convention を common contract へ引き上げた。

## 未完の仕事 — contract がまだ届いていない箇所

未完点は「次に入る機能」の一覧ではなく、**explicit contracts の適用範囲がまだ届いていない箇所**として読む。

- **TX 側の queue assignment** — RX HW queue leasing は 7.1 に入ったが、TX queue leasing は未 merge である。assignment contract は RX 側に偏っている。
- **global synchronization の縮小** — per-netns RTNL は global RTNL の一括置換ではなく、複数 release にわたる migration である。7.3-rc / mainline の RTNL-less FIB-rule 更新と per-netns netdev unregistration は、その途中経過にあたる。
- **異種 memory の一般化** — netmem（6.9）と汎用 page_pool memory-provider hooks（6.15）は canonical boundary を確定した。残る課題は、より多くの driver / memory type がこの provider-aware model を利用できるようにすることである。
- **execution placement の共通化** — switchdev path と TC offload path は並行経路のままであり、XDP も native / generic / offload で実行位置と必要な driver support が異なる。software と hardware のどちらで実行するかを扱う単一の contract はない。
- **observability の coverage** — drop reason はすべての drop path に付くわけではなく、cross-layer の packet journey は今も tool 側の推論に依存する。
- **Rust** — networking 側は PHY / core abstraction が中心で、一般的な high-performance NIC driver が Rust へ移行した段階ではない。

これらは将来の release で評価が変わり得るため、確定史ではなく本書の open questions として扱う。

したがって本書の thesis は未来予測ではなく、既存 milestone を横断した説明モデルである。次の Part VI では、この story からいったん離れ、各 milestone の release attribution を1行1項目で正規化する。

------------------------------------------------------------------------

# Part VI — Canonical release chronology

ここだけが **release attribution と Verification status の正本**である。Part I–V の version 表記は story/lineage の参照であり、この表を上書きしない。

## 採用基準

Part VI は release note の網羅表ではない。採用するのは、本文で追う6軸の長期 lineage、 または TRANSPORT / VIRTUAL-OVERLAY / SECURITY の cross-cutting domain を理解するために必要な **origin / integration / enablement milestone** と、architecture 上の operational range を大きく変える milestone である。

単発の機能追加、局所的な数値上限変更、後続 lineage を変えない個別 protocol/offload enhancement は 原則として採用しない。「Axis を付けられる」だけでは採用理由にならず、Part I–V の story で役割を 説明できることを要求する。

**Axis と Domain は独立である。** Axis は architecture 上の主問題を示す。Domain は単なる実装場所ではなく、 protocol semantics / path management / authentication / virtual-overlay behavior 自体が主題となる場合に付与する。 Axis があることは `explicit contracts` thesis で説明可能であることを意味しない。たとえば BIG TCP は PERFORMANCE 軸だが、contract 化ではなく processing-unit expansion として説明する。

## Verification status

Part VI は chronology であり、この列は **参照先ではなく attribution の検証状態**を示す。外部 source は Part VII の exact anchor と Appendix の source index から辿る。`Verification status` は次の語彙を用いる。

- **release** — final release containment を確認済み。
- **anchor: Part VII** — final release に加え、Part VII に40桁 representative mainline SHA がある。
- **series + release** — patch series/pull と final release の双方で確認。
- **generation** — release 世代は確認できるが、origin / integration / enablement boundary は未正規化。
- **series** — series evidence はあるが canonical boundary の追加監査を残す。
- **mainline; final pending** — Linus mainline merge 済みで final release は未公開。

監査手続き、rc-tag containment、昇格候補、改訂履歴は本文から分離し、別ファイル `linux-networking-evolution-audit-worklog.md` に置く。

## Canonical milestone table

### 3.0–3.18

| Release | Milestone                                       | Axis                           | Domain            | Verification status |
|:--------|:------------------------------------------------|:-------------------------------|:------------------|----------------|
| 3.0     | namespace FD / setns()                          | CONTROL PLANE                  | —                 | release            |
| 3.3     | DQL/BQL                                         | PERFORMANCE / DRIVER FRAMEWORK | —                 | anchor: Part VII     |
| 3.5     | CoDel                                           | PERFORMANCE                    | —                 | anchor: Part VII     |
| 3.5     | fq_codel                                        | PERFORMANCE                    | —                 | release            |
| 3.6     | TSQ                                             | PERFORMANCE                    | —                 | release            |
| 3.6     | TFO client                                      | —                              | TRANSPORT         | release            |
| 3.6     | IPv4 route-cache removal                        | CONTROL PLANE                  | —                 | release            |
| 3.7     | VXLAN                                           | —                              | VIRTUAL / OVERLAY | release            |
| 3.7     | TFO server                                      | —                              | TRANSPORT         | release            |
| 3.9     | bridge VLAN filtering infrastructure              | CONTROL PLANE                  | VIRTUAL / OVERLAY | anchor: Part VII     |
| 3.9     | TCP/UDP SO_REUSEPORT                            | PERFORMANCE                    | —                 | anchor: Part VII     |
| 3.11    | SO_BUSY_POLL                                    | PERFORMANCE                    | —                 | release            |
| 3.12    | sch_fq / TCP pacing / TSO autosizing generation | PERFORMANCE                    | —                 | release            |
| 3.13    | nftables                                        | CONTROL PLANE                  | —                 | anchor: Part VII     |
| 3.15    | internal BPF ISA rework                         | PROGRAMMABILITY                | —                 | release            |
| 3.18    | bpf() / maps / verifier generation              | PROGRAMMABILITY                | —                 | release            |
| 3.18    | DCTCP                                           | —                              | TRANSPORT         | release            |
| 3.18    | Geneve                                          | —                              | VIRTUAL / OVERLAY | release            |

### 3.19–4.20

| Release | Milestone                            | Axis                     | Domain               | Verification status |
|:--------|:-------------------------------------|:-------------------------|:---------------------|----------------|
| 3.19    | switchdev origin                     | DRIVER FRAMEWORK         | —                    | release            |
| 3.19    | ipvlan                               | —                        | VIRTUAL / OVERLAY    | release            |
| 3.19    | SO_ATTACH_BPF                        | PROGRAMMABILITY          | —                    | release            |
| 4.1     | MPLS routing / AF_MPLS              | CONTROL PLANE            | VIRTUAL / OVERLAY    | anchor: Part VII     |
| 4.1     | cls_bpf / act_bpf eBPF support       | PROGRAMMABILITY          | —                    | generation     |
| 4.1     | kprobe BPF milestone                 | OBSERVABILITY            | —                    | generation     |
| 4.2     | Flower classifier                    | PROGRAMMABILITY          | —                    | anchor: Part VII     |
| 4.3     | VRF device                            | CONTROL PLANE            | VIRTUAL / OVERLAY    | anchor: Part VII     |
| 4.6     | devlink                              | DRIVER FRAMEWORK         | —                    | anchor: Part VII     |
| 4.7     | TC BPF direct packet access          | PROGRAMMABILITY          | —                    | release            |
| 4.8     | XDP                                  | PROGRAMMABILITY          | —                    | series + release     |
| 4.9     | BBR                                  | —                        | TRANSPORT            | anchor: Part VII     |
| 4.10    | cgroup BPF                           | PROGRAMMABILITY          | —                    | series + release     |
| 4.10    | BPF LWT                              | PROGRAMMABILITY          | —                    | series + release     |
| 4.13    | SOCK_OPS                             | PROGRAMMABILITY          | TRANSPORT            | release            |
| 4.13    | kTLS TX                              | —                        | SECURITY / TRANSPORT | release            |
| 4.14    | phylink                              | DRIVER FRAMEWORK         | —                    | anchor: Part VII     |
| 4.14    | SOCKMAP                              | PROGRAMMABILITY          | —                    | release            |
| 4.14    | XDP devmap                           | PROGRAMMABILITY          | —                    | release            |
| 4.14    | TCP MSG_ZEROCOPY                     | PERFORMANCE              | —                    | anchor: Part VII     |
| 4.15    | XDP cpumap                           | PROGRAMMABILITY          | —                    | release            |
| 4.16    | netdevsim                            | DRIVER FRAMEWORK         | —                    | anchor: Part VII     |
| 4.16    | Net DIM initial generation                   | DRIVER FRAMEWORK         | —                    | generation     |
| 4.16    | nftables software flowtable          | PERFORMANCE              | —                    | release            |
| 4.17    | BPF_PROG_TYPE_SK_MSG                 | PROGRAMMABILITY          | —                    | anchor: Part VII     |
| 4.18    | AF_XDP                               | MEMORY / PROGRAMMABILITY | —                    | series + release     |
| 4.18    | page_pool origin / XDP memory return | MEMORY                   | —                    | anchor: Part VII     |
| 4.18    | TCP_ZEROCOPY_RECEIVE                 | PERFORMANCE              | —                    | release            |
| 4.18    | BTF origin / typed BPF metadata                           | PROGRAMMABILITY / OBSERVABILITY | —                    | anchor: Part VII |
| 4.19    | SO_TXTIME                            | PERFORMANCE              | —                    | release            |
| 4.19    | CAKE                                 | PERFORMANCE              | —                    | release            |
| 4.20    | TCP EDT                              | PERFORMANCE              | —                    | release            |
| 4.20    | taprio                               | PERFORMANCE              | —                    | release            |
| 4.20    | BPF flow dissector                   | PROGRAMMABILITY          | —                    | release            |

### 5.0–6.1

| Release | Milestone                              | Axis             | Domain               | Verification status |
|:--------|:---------------------------------------|:-----------------|:---------------------|----------------|
| 5.0     | UDP GRO                                | PERFORMANCE      | —                    | release            |
| 5.0     | UDP MSG_ZEROCOPY                       | PERFORMANCE      | —                    | release            |
| 5.1     | devlink health                         | DRIVER FRAMEWORK | —                    | release            |
| 5.1     | mac80211 airtime accounting/scheduling | PERFORMANCE      | —                    | release            |
| 5.3     | nexthop objects                        | CONTROL PLANE    | —                    | release            |
| 5.3     | TC ct action                            | PROGRAMMABILITY  | —                    | anchor: Part VII     |
| 5.3     | DIM generalized into lib/dim           | DRIVER FRAMEWORK | —                    | generation     |
| 5.3     | Net DIM common-library integration                      | DRIVER FRAMEWORK                | —                    | anchor: Part VII |
| 5.5     | nftables flowtable hardware offload          | PERFORMANCE / DRIVER FRAMEWORK | —          | release            |
| 5.5     | mac80211 AQL                           | PERFORMANCE      | —                    | release            |
| 5.6     | MPTCP                                  | —                | TRANSPORT            | release            |
| 5.6     | WireGuard                              | —                | SECURITY / VIRTUAL / OVERLAY | release            |
| 5.6     | BPF struct_ops / TCP CC                | PROGRAMMABILITY  | TRANSPORT            | release            |
| 5.6     | ethtool Generic Netlink                | DRIVER FRAMEWORK | —                    | anchor: Part VII     |
| 5.9     | BPF_PROG_TYPE_SK_LOOKUP                | PROGRAMMABILITY  | —                    | anchor: Part VII     |
| 5.11    | auxiliary bus                          | DRIVER FRAMEWORK | —                    | anchor: Part VII     |
| 5.12    | threaded NAPI                          | DRIVER FRAMEWORK | —                    | release            |
| 5.17    | structured drop-reason foundation      | OBSERVABILITY    | —                    | release            |
| 5.18    | XDP multi-buffer / frags generation    | PROGRAMMABILITY  | —                    | generation     |
| 5.19    | IPv6 BIG TCP                           | PERFORMANCE      | —                    | anchor: Part VII     |
| 5.19    | drop-reason expansion                  | OBSERVABILITY    | —                    | release            |
| 6.0     | io_uring SEND_ZC                       | PERFORMANCE      | —                    | release            |
| 6.0     | io_uring multishot receive             | PERFORMANCE      | —                    | release            |

### 6.2–7.3-rc

| Release | Milestone                                             | Axis                             | Domain               | Verification status |
|:--------|:------------------------------------------------------|:---------------------------------|:---------------------|----------------|
| 6.2     | TCP PLB                                               | —                                | TRANSPORT            | release            |
| 6.2     | XFRM/IPsec packet offload                             | DRIVER FRAMEWORK                 | SECURITY             | anchor: Part VII     |
| 6.3     | YNL / YAML Netlink tooling                            | CONTROL PLANE                    | —                    | release            |
| 6.3     | IPv4 BIG TCP                                          | PERFORMANCE                      | —                    | release            |
| 6.4     | BPF netfilter programs (`BPF_PROG_TYPE_NETFILTER`) | PROGRAMMABILITY | —             | release            |
| 6.6     | AF_XDP multi-buffer                                   | MEMORY / PROGRAMMABILITY         | —                    | release            |
| 6.6     | TCX / bpf_mprog                                       | PROGRAMMABILITY                  | —                    | release            |
| 6.7     | netkit                                                | PROGRAMMABILITY                  | VIRTUAL / OVERLAY    | anchor: Part VII     |
| 6.7     | TCP-AO                                                | —                                | SECURITY / TRANSPORT | release            |
| 6.8     | Rust phylib / Asix reference PHY                      | DRIVER FRAMEWORK                 | —                    | anchor: Part VII     |
| 6.8     | queue/NAPI netdev-genl visibility                     | DRIVER FRAMEWORK / OBSERVABILITY | —                    | generation     |
| 6.8     | page_pool identity / Netlink introspection              | OBSERVABILITY / MEMORY          | —                    | anchor: Part VII |
| 6.11    | virtio-net AF_XDP RX zero-copy                        | MEMORY                           | VIRTUAL / OVERLAY    | release            |
| 6.12    | Device Memory TCP RX                                  | MEMORY                           | —                    | anchor: Part VII |
| 6.13    | per-netns RTNL infrastructure milestone               | CONTROL PLANE                    | —                    | anchor: Part VII |
| 6.15    | io_uring ZCRX                                         | MEMORY                           | —                    | anchor: Part VII |
| 6.15    | further RTNL breakup                                  | CONTROL PLANE                    | —                    | series + release     |
| 6.15    | page_pool custom memory-provider hooks                  | MEMORY / DRIVER FRAMEWORK       | —                    | anchor: Part VII |
| 6.16    | Device Memory TCP TX                                  | MEMORY                           | —                    | anchor: Part VII |
| 6.16    | BPF qdisc                                             | PROGRAMMABILITY                  | —                    | series + release     |
| 6.18    | AccECN core                                           | —                                | TRANSPORT            | generation     |
| 6.18    | UDP RX evolution                                      | PERFORMANCE                      | —                    | generation     |
| 6.19    | `dev_queue_xmit()` llist TX scheduling                | PERFORMANCE                      | —                    | release            |
| 6.19    | threaded-NAPI kthread busy-poll extension             | DRIVER FRAMEWORK                 | —                    | release            |
| 6.19    | WireGuard YNL-described Netlink                       | CONTROL PLANE                    | SECURITY             | release            |
| 7.0     | cake_mq                                               | PERFORMANCE                      | —                    | release            |
| 7.0     | IPv6 BIG TCP without synthetic HBH jumbo header       | PERFORMANCE                      | —                    | release            |
| 7.0     | AccECN enablement                                     | —                                | TRANSPORT            | release            |
| 7.0     | large RX buffers for memory providers / io_uring ZCRX | MEMORY                           | —                    | release            |
| 7.1     | RX HW queue leasing                                   | MEMORY / DRIVER FRAMEWORK        | —                    | anchor: Part VII |
| 7.1     | dedicated qdisc-drop tracepoint                       | OBSERVABILITY                    | —                    | release            |
| 7.3-rc  | BIG TCP over VXLAN/GENEVE                             | PERFORMANCE                      | VIRTUAL / OVERLAY    | mainline; final pending       |
| 7.3-rc  | RTNL-less FIB-rule updates                            | CONTROL PLANE                    | —                    | mainline; final pending       |
| 7.3-rc  | devmem buffers \>PAGE_SIZE                            | MEMORY                           | —                    | mainline; final pending       |
| 7.3-rc  | per-netns netdev-unregistration infrastructure        | CONTROL PLANE                    | —                    | mainline; final pending       |

### Architecture-critical items with open canonical boundary

次の項目は本文の中心 lineage に属するため chronology から消さず、**release attribution を断定しない boundary-open register** としてここに残す。これは canonical release row ではない。

| Item | Axis | なぜ本編に必要か | Open boundary |
|---|---|---|---|
| `netmem` | MEMORY | `struct page` と packet memory identity の分離 | canonical origin / first final release |

## Part VI から Part VII へ — chronology から provenance へ

Part VI は「いつ」を正規化し、Part VII はその attribution を再監査できる exact anchor を保持する。canonical boundary が未確定の項目は Part VI の boundary-open register に集約する。
# Part VII — 正規 provenance ledger

**Evidence model:** Part VII の SHA は feature series の「代表 anchor」であり、anchor の存在だけで series 全体を証明しない。 各項目は **SHA identity / feature correspondence / release containment** を別々に監査する。

**Evidence status:** `mainline; final pending` は authoritative な pull/merge evidence により Linus mainline への merge を確認済みだが、final release tag が未公開の状態を示す。Part VI の Verification status と一致させる。

## Exact mainline anchor inventory

ここに示すのは feature series の全 commit ではなく、再監査可能な代表 anchor である。`Anchor type` は次の意味で使う。

- `origin` — object / mechanism の最初の mainline anchor
- `enablement` — feature を実際に利用可能にした代表 commit
- `integration` — subsystem 間を接続した代表 commit
- `merge` — multi-commit series / generation を Linus mainline に統合した canonical merge

巨大な series を恣意的な1 patchで代表させるより、pull/merge message が feature generation を明示する場合は `merge` を優先する。特に per-netns RTNL のような複数 release にまたがる migration では、`merge` は「完成 commit」ではなく **その generation の開始/統合点**を意味する。本文の台帳では representative SHA と first final release に絞り、first-containing rc tag の機械監査は作業ログへ分離する。

| Item | Anchor type | Exact mainline anchor | Subject / role | First final release |
|---|---|---|---|---|
| DQL | origin | `75957ba36c05b979701e9ec64b37819adc12f830` | `dql: Dynamic queue limits` | v3.3 |
| CoDel | origin | `76e3cc126bb223013a6b9a0e2a51238d1ef2e409` | CoDel qdisc core anchor | v3.5 |
| SO_REUSEPORT infrastructure | origin | `055dc21a1d1d219608cd4baac7d0683fb2cbbe8a` | `soreuseport: infrastructure` | v3.9 |
| nftables core | origin | `96518518cc417bb0a8c80b9fb736202e28acdf96` | `netfilter: add nftables` | v3.13 |
| nftables set API | integration | `20a69341f2d00cd042e81c82289fba8a13c05a25` | set-API anchor; not core origin | v3.13 |
| bridge VLAN filtering | origin | `243a2e63f5f47763b802e9dee8dbf1611a1c1322` | bridge VLAN filtering infrastructure | v3.9 |
| MPLS routing / AF_MPLS | origin | `0189197f441602acdca3f97750d392a895b778fd` | MPLS label-based routing / AF_MPLS | v4.1 |
| Flower classifier | origin | `77b9900ef53ae047e36a37d13a2aa33bb2d60641` | initial `cls_flower` classifier | v4.2 |
| VRF device | origin | `193125dbd8eb292d88feb201f030889b488b0a02` | VRF device / routing-domain separation | v4.3 |
| TC ct action | integration | `b57dc7c13ea90e09ae15f821d2583fa0231b4935` | conntrack state/metadata as TC action | v5.3 |
| devlink | origin | `bfcd3a46617209454cfc0947ab093e37fd1e84ef` | `Introduce devlink infrastructure` | v4.6 |
| BBR | origin | `0f8782ea14974ce992618b55f0c041ef43ed0b78` | initial BBR mainline anchor | v4.9 |
| phylink | origin | `9525ae83959b60c6061fe2f2caabdc8f69a48bc6` | `phylink: add phylink infrastructure` | v4.14 |
| netdevsim | origin | `83c9e13aa39aed5cf9a2f8dd69770b7c35ba1281` | hardware-independent offload test device | v4.16 |
| SK_MSG | enablement | `4f738adba30a7cfc006f605707e7aee847ffefa0` | socket-message verdict / `BPF_PROG_TYPE_SK_MSG` | v4.17 |
| TCP MSG_ZEROCOPY | enablement | `f214f915e7db99091f1312c48b30928c1e0c90b7` | `tcp: enable MSG_ZEROCOPY` | v4.14 |
| page_pool origin | origin | `ff7d6b27f894f1469dc51ccb828b7363ccd9799f` | page_pool core origin anchor | v4.18 |
| page_pool/XDP integration | integration | `60bbf7eeef10dc647430646d7fe5e3d8d132dbec` | mlx5 page_pool/XDP integration | v4.18 |
| ethtool Generic Netlink | origin | `2b4a8990b7df55875745a80a609a1ceaaf51f322` | `ethtool: introduce ethtool netlink interface` | v5.6 |
| SK_LOOKUP | origin | `e9ddbb7707ff5891616240026062b8c1e29864ca` | dedicated SK_LOOKUP program type / attach point | v5.9 |
| auxiliary bus | origin | `7de3697e9cbd4bd3d62bafa249d57990e1b8f294` | `Add auxiliary bus support` | v5.11 |
| IPv6 BIG TCP / GRO | enablement | `0fe79f28bfaf73b66b7b1562d2468f94aa03bd12` | allow `gro_max_size` > 65536 | v5.19 |
| IPv6 BIG TCP / GSO | enablement | `7c4e983c4f3cf94fcd879730c6caa877e0768a4d` | allow `gso_max_size` > 65536 | v5.19 |
| BTF origin | origin | `69b693f0aefa0ed521e8bd02260523b5ae446ad7` | `bpf: btf: Introduce BPF Type Format (BTF)`; typed metadata for BPF program/map | v4.18 |
| Net DIM common-library integration | integration | `4f75da3666c0c572967729a2401ac650be5581b6` | `linux/dim: Move implementation to .c files`; driver-local/header logic becomes common `lib/dim` implementation | v5.3 |
| vDPA bus | origin | `961e9c84077f6c8579d7a628cbe94a675cb67ae4` | `vDPA: introduce vDPA bus`; common virtio datapath / vendor-control abstraction | v5.7 |
| netmem abstraction | origin | `18ddbf5cf0e7553fd05c3e1a02d740514ee3f0a6` | `net: introduce abstraction for network memory`; `netmem_ref` decouples network memory references from `struct page` | v6.9 |
| page_pool userspace identity | enablement | `f17c69649c698e4df3cfe0010b7bbf142dec3e40` | `net: page_pool: id the page pools`; creates stable IDs for uAPI references | v6.8 |
| page_pool netlink GET | enablement | `950ab53b77ab829defeb22bc98d40a5e926ae018` | `net: page_pool: implement GET in the netlink API`; exposes pool identity / ifindex / NAPI ID | v6.8 |
| page_pool netlink statistics | integration | `d49010adae737638447369a4eff8f1aab736b076` | `net: page_pool: expose page pool stats via netlink`; driver-independent observability | v6.8 |
| page_pool custom memory providers | integration | `57afb483015768903029c8336ee287f4b03c1235` | `net: page_pool: create hooks for custom memory providers`; shared allocator hook for devmem TCP and io_uring ZCRX | v6.15 |
| Device Memory TCP RX | merge | `9410645520e9b820069761f3450ef6661418e279` | merge tag `net-next-6.12`; Device Memory TCP RX generation | v6.12 |
| per-netns RTNL start | merge | `fcc79e1714e8c2b8e216dc3149812edd37884eef` | merge tag `net-next-6.13`; initial per-netns RTNL conversion wave, explicitly in-progress | v6.13 |
| io_uring ZCRX | merge | `71f0dd5a3293d75d26d405ffbaedfdda4836af32` | merge branch `io_uring-zero-copy-rx`; queue-bound userspace-page RX / memory-provider integration | v6.15 |
| Device Memory TCP TX | merge | `1b98f357dadd6ea613a435fbaef1a5dd7b35fd21` | merge tag `net-next-6.16`; Device Memory TCP transmit path | v6.16 |
| XFRM packet offload | enablement | `d14f28b8c1de668bab863bf5892a49c824cb110d` | add packet offload flag | v6.2 |
| netkit | origin | `35dfaad7188cdc043fde31709c796f5a692ba2bd` | netkit core anchor | v6.7 |
| Rust PHY abstractions | integration | `f20fd5449ada3872dcd67aca397f0e27ca2e8ad6` | Rust core abstractions for network PHY drivers | v6.8 |
| netdev-genl queue object | integration | `bc877956272f0521fef107838555817112a450dc` | YAML spec for queue object | v6.8 |
| RX HW queue leasing / queue-create | enablement | `7789c6bb76acf21539c2c74b0cc869bb57de99e6` | `net: Add queue-create operation`; virtual RX queue may lease a physical RX queue | v7.1 |
| RX HW queue leasing generation | merge | `91a4855d6c03e770e42f17c798a36a3c46e63de2` | merge tag `net-next-7.1`; pull message explicitly lists HW queue leasing | v7.1 |
| net-next 7.3 merge | merge | `91ec2035134982b98fab0609a9fd8480e8217dc1` | merge tag `net-next-7.3` | 7.3 final pending |

### Exact-anchor interpretation notes

`netmem` と memory providers は別の boundary である。6.9 の `netmem_ref` は **memory type を `struct page` から抽象化する型/API contract**、6.15 の memory-provider hooks は **page_pool が custom allocator/provider を選択する lifecycle/integration contract** である。Device Memory TCP RX は6.12に専用経路で先行し、6.15でdevmem TCPとio_uring ZCRXが共通provider interfaceへ整理された。

page_pool introspection は単一commitへ縮約しない。`f17c69649...` が pool ID、`950ab53b...` がNetlink GET、`d49010ad...` がstats exposureを導入するため、この3段階を6.8 generationのrepresentative anchorsとする。

per-netns RTNLは6.13で完成した機能ではない。`fcc79e17...` は複数releaseにまたがるmigrationの開始waveをまとめたmerge anchorである。RX HW queue leasingも単一commitではなく、`7789c6bb...`をqueue-create/lease APIのenablement、`91a4855d...`を7.1 generationのmerge anchorとして扱う。


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
| nftables core / architecture            | Part VI 3.13; Part III netfilter / nftables / conntrack | VM/expression model, Netlink control, transactional ruleset evolution |
| nftables ingress / bridge                | Part III netfilter / nftables / conntrack                | earlier hook placement and L2/L3 policy integration             |
| nftables flowtable                       | Part VI 4.16; Part III netfilter / nftables / conntrack | policy-selected software fast path; later HW-offload evolution  |
| TCX / `bpf_mprog`                    | Part VI 6.6; Part III BPF                               | link-based TC attachment and multi-program ordering             |
| Device Memory TCP / memory providers | Part VI 6.12/6.16; Part III packet memory               | RX/TX device-memory zero-copy evolution                         |
| io_uring ZCRX                        | Part VI 6.15; Part III io_uring networking              | queue-bound zero-copy receive                                   |
| `net-next-7.1`                       | Part VI 7.1                                             | RX HW queue leasing                                             |
| `net-next-7.3`                       | Part VI 7.3-rc                                          | tunnel BIG TCP, RTNL-less FIB rules, \>PAGE_SIZE devmem buffers |

## Exact anchor index

Exact SHA の正本は Part VII に置く。Appendix では同じcommit listを複製しない。

## Conference provenance index

Conference talk は設計意図や当時のproblem statementを補足する資料として扱い、 release attributionには使用しない。

| Conference lineage                      | Canonical destination       |
|:----------------------------------------|:----------------------------|
| Netdev: XDP / AF_XDP / TC / BPF         | Part III BPF / XDP          |
| Netdev: BIG TCP                         | Part III packet aggregation |
| Netdev: Device Memory TCP / zero-copy   | Part III packet memory      |
| Netdev 0x19: Diagnosing Page Pool Leaks | Part IV page_pool           |
| Netdev: queue/NAPI/netdev-genl          | Part IV driver framework    |
| Netdev: MPTCP / TCP state-of-the-union  | Part III transport          |
| Netdev 0.1/1.1: nftables architecture / ingress / bridge | Part III netfilter |
| Netdev: Netfilter flowtable / TC conntrack offload | Part III netfilter / Routing-TC-offload |
| Kernel Recipes: nftables Why and how / What's new | Part III netfilter |
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

Retis と pwru は kernel primitive を利用する tool-side case study である。両者の位置づけは次の一文で十分である。

``` text
pwru  = broad kernel-function packet trajectory
Retis = networking-event-centric semantic enrichment / correlation
```

これらは BTF、eBPF tracing、drop reason、timestamp、OVS/OVN metadata などの kernel primitiveを利用する **consumer/tooling examples** であり、独立したkernel evolution axisではない。

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


