---
title: Linux Networking Evolution
---

# Linux Networking Evolution — Linux v3.0 から 7.x まで

**調査基準日:** 2026-10-02

この文書は、Linux networking の変化を「調査した順」ではなく、 **kernel networking がどのように進化したかを読む順序**に再構成した版である。

**対象範囲:** wired / host networking を中心に扱う。RDMA、Wi-Fi 全般、QUIC や security protocol の網羅的な歴史は対象外とする。一方、SECURITY は kTLS / XFRM / WireGuard / TCP-AO のように networking architecture と交差する範囲を cross-cutting lineage として扱う。mac80211 は queue management（airtime / AQL）の観点から選択的に扱う。

## この文書の読み方

本編は **architecture framing → architecture map → era → lineage → driver framework / observability → conclusion** の順で読む。release 番号を調べる場合は Part VI、 commit-level provenance を再監査する場合は Part VII へ直接進んでよい。

Part I–V の version 表記は説明上の参照であり、release attribution を独立に定義しない。 その正本は Part VI、exact anchor の台帳は Part VII である。詳細な evidence 用語と canonicalization rule は Part VI 冒頭で定義する。

### 読者別の読み方

| 目的 | 推奨ルート |
|---|---|
| **10 分で全体像** | Part I → Part II の4 eras → Synthesis |
| **kernel / driver architecture** | Part I → Part III → Part IV → Part V |
| **release / provenance 監査** | Part VI → Part VII → Appendix source index |

### 図の記法

- `→` / `↓` — 時系列または shared design problem に沿う **lineage continuation**。直接依存を意味しない。
- `⇒` — 本文で明示した **direct extension / integration**。
- `∥` — 同時期の **parallel evolution**。
- `↔` — 相互作用するが親子関係ではない。

------------------------------------------------------------------------

# Part I — Executive framing: Linux networking は何が変わったのか

Linux networking の 15 年間の変化は、単純な「高速化」でも、従来の `skb` path の置き換えでもない。 従来の **CPU が system RAM 上の skb を処理するモデルを現在も広く維持しながら**、 AF_XDP、device memory、io_uring zero-copy、programmable/offloaded path など、 複数の execution / memory ownership model を workload と hardware capability に応じて **共存させられる architecture へ拡張した**ことが大きな変化である。

その変化は 6 つの軸で並行して進んだ。

```text
PERFORMANCE       queueing / pacing / aggregation / zero-copy
PROGRAMMABILITY   BPF / XDP / socket / protocol / qdisc extension points
MEMORY            page_pool / netmem / memory providers / device memory
CONTROL PLANE     Netlink object model / YNL / RTNL scope reduction
OBSERVABILITY     typed tracing / drop reason / queue-NAPI-memory identity
DRIVER FRAMEWORK  switchdev / devlink / phylink / DIM / common driver contracts
```

この「複数モデルの共存」を可能にした主要な architecture mechanism を、本書では **explicit resource / control contracts** と捉える。driver-private、implicit、global だった仕組みを kernel 共通の object、API、accounting、assignment、lifetime、synchronization contract として明示することで、従来の skb/system-RAM path を残したまま programmable / zero-copy / offloaded / device-memory path を選択的に接続できるようになった、というのが本書で追う主要な architecture trend である。


### 本書で固定する語彙

- **主要な architecture trend:** 複数の execution / memory ownership model の共存
- **それを可能にした mechanism:** explicit resource / control contracts
- **contract の具体形:** object / accounting / API / assignment / lifetime / synchronization
- **重要な subtheme:** ownership（特に queue / packet memory）

`ownership` はこの流れの重要な一部だが、すべてを ownership だけで説明しない。たとえば BQL は 主に byte accounting / backpressure、per-netns RTNL は synchronization scope の問題である。 一方 page_pool / AF_XDP / memory providers / queue leasing では、buffer や queue を誰が管理・割り当てるかという ownership が中心問題になる。

**execution placement** は contract form そのものではなく、object / assignment / lifetime / synchronization 等の contract によって software / kthread / userspace / NIC / accelerator のどこで処理するかを選択可能になった**結果**を表す横断概念である。

**synchronization** は文脈を区別する。RTNL では lock / serialization scope、switchdev 等では kernel canonical state と hardware state の同期、phylink では複数 component の state coordination を指す。

### 具体例: 1 本の RX queue を誰が使うのか

2011 年頃の典型像では、NIC RX queue の buffer は system RAM 上の page/skb へ入り、NAPI から通常の kernel network stack へ渡される、という対応を暗黙に想定しやすかった。2026 年には同じ「RX queue」が、通常 stack、XDP/AF_XDP、io_uring ZCRX、device-memory provider、あるいは virtual device へ lease された queue の入口になり得る。

```text
object model / data path:
RX queue
 ├─ kernel stack / skb
 ├─ XDP → AF_XDP / redirect
 ├─ io_uring ZCRX
 ├─ device-memory backed receive
 └─ leased queue → virtual / delegated consumer
```

重要なのは旧 path が消えたことではない。**queue identity、memory provider、consumer、lifetime、synchronization を明示し、複数の execution / memory ownership model を同じ network device 上で選択・共存させられるようになったこと**が本書で追う主要な architecture trend である。


## 接続点の予告

後半の Synthesis では、queue、packet memory、forwarding object、security state、observability evidence などを「どの object/resource を誰が所有・参照し、どこで実行し、どう同期するか」という接続点で比較する。 Part I の表はこの命題の**例示**であり、検証そのものは Part VI の chronology と Part VII の anchor/evidence で行う。


## Architecture evolution map

```text
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

| 観点 | v3.0 前後 | v5.0 前後 | 2026 |
|:-----------------|:----------------------------------|:--------------------------|:-----------------------------------------------|
| PERFORMANCE | driver-local tuning が大きい | BQL/TSQ/fq/pacing が定着 | multi-layer control + cake_mq / scalable TX |
| PROGRAMMABILITY | classic BPF 中心 | XDP/AF_XDP・TC BPF | struct_ops / TCX / BPF qdisc / netkit |
| MEMORY | page-centric driver-local recycle | page_pool が利用可能 | netmem / memory providers / device memory |
| DRIVER FRAMEWORK | net_device 中心 | switchdev/devlink/phylink | queue/NAPI objects・YNL-described APIs |
| CONTROL PLANE | global RTNL 依存が大きい | global RTNL が依然中心 | per-netns / fine-grained / RTNL-less paths |
| OBSERVABILITY | logs/counters/tracing の個別利用 | BPF/BTF ecosystem が拡大 | typed drop reason + queue/NAPI/memory identity |

## 6 軸・contract・cross-cutting domain の対応

6 軸は **architecture analysis の主軸**であり、全 milestone の排他的 taxonomy ではない。 TRANSPORT、VIRTUAL / OVERLAY、SECURITY は Axis と独立した **cross-cutting domain** である。 Domain は単に「その protocol のコードを通る」という意味ではなく、protocol semantics / path management / authentication / virtual-overlay behavior 自体が milestone の主題である場合に付与する。 したがって BIG TCP や Device Memory TCP は TCP path に存在しても、それだけを理由に TRANSPORT とはしない。

| Architecture axis | 15 年間の変化 | 主な contract form | 本文の主な扱い |
|-------------------|:-------------------------------------------------------------------------------------------|:-------------------------------------|:--------------------------------------------------------------|
| PERFORMANCE | implicit queueing → explicit accounting / multi-layer scheduling / larger processing units | accounting / API | Part III「Queueing / latency / pacing」「Packet aggregation」 |
| PROGRAMMABILITY | fixed path → verified extension points → reusable programmable infrastructure | API / object | Part III「BPF」「XDP / AF_XDP」 |
| MEMORY | driver-local page handling → common lifetime / provider / assignment model | lifetime / assignment / object | Part III「Packet memory」「io_uring networking」 |
| CONTROL PLANE | embedded/global state → reusable objects / schema / narrower lock scope | object / API / synchronization | Part III「Routing / TC / offload」「Routing / Netlink / RTNL」「netfilter」 |
| OBSERVABILITY | opaque outcome → typed reason / identity / correlation | contract を検証する evidence layer | Part V |
| DRIVER FRAMEWORK | driver-local convention → reusable kernel framework / device-wide object | object / API / accounting / lifetime | Part IV |

**Cross-cutting domain:** TRANSPORT は Part III「TCP / UDP / transport」、VIRTUAL / OVERLAY は Part III「Virtual networking」、SECURITY は Part III「SECURITY / kTLS / XFRM / WireGuard」で扱う。これらは複数の architecture axis を横断する domain として扱う。

### 主要 architecture trend を例示する代表例

| Milestone | 以前の状態 | 主に明示化したもの | Contract form | Axis | Era |
|--------------------------------------|------------------------------------------------------------|--------------------------------------------------|-----------------------|----------------------------------|-------|
| DQL/BQL | driver-local TX ring tuning | queued/completed byte accounting | accounting | PERFORMANCE / DRIVER FRAMEWORK | Era 1 |
| `bpf()` / maps / verifier | fixed kernel behavior / ad-hoc attachment | verified program + map interface | API | PROGRAMMABILITY | Era 1 |
| switchdev / devlink | vendor-local forwarding / device state | forwarding/device/resource objects | object | DRIVER FRAMEWORK | Era 2 |
| page_pool | driver-local page recycle / DMA lifetime | allocation / recycle / DMA lifetime | lifetime | MEMORY | Era 2 |
| AF_XDP | kernel-centric queue/buffer handling | queue / UMEM binding | assignment + lifetime | MEMORY / PROGRAMMABILITY | Era 2 |
| nexthop objects | route に埋め込まれた nexthop state | reusable nexthop object / group | object | CONTROL PLANE | Era 3 |
| BPF `struct_ops` | protocol algorithm が built-in implementation 中心 | typed operations implemented by verified BPF | API + object | PROGRAMMABILITY | Era 3 |
| YNL | handwritten Netlink family definition / client boilerplate | machine-readable Netlink schema | API | CONTROL PLANE | Era 4 |
| queue/NAPI netdev-genl | opaque driver rings / NAPI instances | stable queue/NAPI identity and relations | object | DRIVER FRAMEWORK / OBSERVABILITY | Era 4 |
| Device Memory TCP | packet payload は system RAM 前提 | non-system-RAM memory bound into networking path | assignment + lifetime | MEMORY | Era 4 |
| io_uring ZCRX / memory-provider path | socket receive と userspace buffer lifecycle の分離 | queue-bound receive memory provider | assignment + lifetime | MEMORY | Era 4 |
| per-netns RTNL | global RTNL scope | namespace-scoped synchronization | synchronization | CONTROL PLANE | Era 4 |
| RX HW queue leasing | hardware queue が physical netdev に固定 | hardware queue assignment / leasing | assignment | MEMORY / DRIVER FRAMEWORK | Era 4 |

この分布は Era の違いも示す。Era 1 では accounting / API の基礎、Era 2–3 では reusable object / programmability、Era 4 では assignment / lifetime / synchronization の明示化が目立つ。ただし 各 Era は重なりを許し、この表は厳密な periodization ではない。

この表は release attribution の正本ではない。version と Verification status は Part VI に従う。

### RDMA をどう位置づけるか

RDMA は本書の 6 軸に追加する第 7 の axis ではない。InfiniBand / RoCE の verbs path は通常の socket / `skb` datapath と異なる execution / memory model を長く持っており、**MEMORY・CONTROL PLANE・DRIVER FRAMEWORK を横断する比較対象**として扱う。

特に modern AI / HPC workload では GPU / accelerator memory と NIC の間で copy と CPU involvement を減らすことが重要になるが、RDMA 自体は AI のために生まれた技術ではない。本書では「AI が RDMA を生んだ」とはせず、**AI workload が、registered memory・queue pair・completion queue・ODP・dma-buf/P2P といった既存 Linux RDMA contract の価値を強く可視化した**と位置づける。

### 並行して進んだ別の系列

`explicit resource / control contracts` は本書が観察する強い architecture tendency であり、 Linux networking の全変更を説明する万能則ではない。本書は 15 年間の変化を、少なくとも **(1) resource / control contract の明示化、(2) transport algorithm / protocol semantics の進化、(3) packet-processing unit / batching の拡大、(4) implementation scalability の改善**という並行する系列として読む。TFO / DCTCP / BBR / MPTCP / PLB / AccECN は (2)、GRO/GSO / BIG TCP は (3)、`dev_queue_xmit()` llist 化は (4) の代表例であり、(1) へ還元しない。この trend は (1) を説明し、(2)〜(4) は別の並行系列として扱う。

Observability も 7 番目の contract form とはしない。typed identity、drop reason、tracepoint、 timestamp などは、resource/control contract と runtime behavior を理解・検証する **evidence layer** と位置付ける。

Part II–V はこの architecture trend を architecture の観点から説明し、 Part VI–VII が release attribution と provenance を検証・再監査可能な形で保持する。

# Part II — Four architecture eras

Part II の4 erasは、主要 architecture trendを証明するための分類ではなく、各時期に前面化した **主要な engineering pressure** を時系列で整理する地図である。 Part III では、この時代区分を横切って継続したarchitecture seriesを追う。

ここでの era は kernel major version の境界ではなく、**外部の圧力に対して networking architecture が主に何を解こうとしたか**で区切る。境界は重なり、ある era で始まった仕組みは後続 era でも継続して発展する。

| Era | おおよその期間 | 外部の圧力 | 主問題 | 代表的な milestone |
|---|---|---|---|---|
| **Era 1 — Queue & scalability foundation** | ～2014 頃 | 10/40GbE、multicore server、virtualization/container、datacenter latency | queue backlog、multicore、namespace、overlay、filtering の基礎 | BQL, CoDel/fq_codel, TSQ, pacing, namespace/setns, VXLAN, nftables, eBPF foundation |
| **Era 2 — Programmable datapath** | 2015～2018 頃 | cloud networking、high packet rate、SmartNIC/offload、kernel-bypass pressure | packet path を安全に拡張し、hardware/offload と接続する | TC BPF, XDP, cgroup/LWT BPF, SOCKMAP, switchdev, devlink, AF_XDP, page_pool |
| **Era 3 — Programmability becomes infrastructure** | 2019～2022 頃 | hyperscale operations、protocol experimentation、100GbE 級、typed tooling | programmable mechanism を protocol / operations / memory infrastructure に広げる | BTF ecosystem, struct_ops, SK_LOOKUP, MPTCP, BIG TCP, io_uring networking, drop reason, devlink health |
| **Era 4 — Heterogeneous I/O & control scalability** | 2023～ | accelerator/device memory、400GbE 級、heterogeneous execution、global-lock scalability | queue・memory・device・locking scope を明示的 object/contract として配置・制御する | netmem, memory providers, Device Memory TCP, netdev-genl queue/NAPI objects, queue leasing, per-netns RTNL, BPF qdisc, YNL |

Era 1 の foundation は 4 つに要約できる。**queue/scalability**（BQL・TSQ・fq/pacing）、**virtualization/control**（namespace/setns・VXLAN・route-cache removal・nftables）、**programmability**（eBPF ISA → `bpf()`/maps/verifier）、**server/transport**（SO_REUSEPORT・SO_BUSY_POLL・TFO・DCTCP）である。これは新しい taxonomy ではなく、後続 lineage の出発条件を示す要約である。

### 圧力から architecture response へ

- **Era 1:** 10/40GbE と multicore 化は「ring に積めるだけ積む」設計の latency / fairness 問題を露呈し、BQL・TSQ・pacing・busy polling のように queue と CPU locality を明示的に制御する方向を促した。同時に namespace / VXLAN は virtualization と multi-tenant networking の土台になった。
- **Era 2:** cloud datapath の多様化と packet-rate pressure は、固定された kernel path を増築し続ける代わりに、eBPF/TC/XDP の verified extension points と AF_XDP の queue-bound userspace path を発達させた。SmartNIC/switch ASIC の普及は switchdev/devlink/TC offload という hardware mapping の共通化も要求した。
- **Era 3:** programmable path が production infrastructure になると、program を「置ける」だけでは不十分になり、BTF/CO-RE、struct_ops、typed observability、BIG TCP、MPTCP、io_uring networking のように reuse・operations・processing-unit の拡張が進んだ。
- **Era 4:** 400GbE 級、accelerator、device memory、heterogeneous execution は「どの queue / memory / device / lock scope を誰が使うか」を暗黙の driver convention のまま扱いにくくし、netmem / memory providers / queue objects / queue leasing / narrower RTNL scope のような explicit placement/control を押し出した。

## Era 間で変わった設計上の問い

```text
How much should we queue?
        ↓
Where can we program the datapath?
        ↓
How do those mechanisms become reusable infrastructure?
        ↓
How explicitly can placement, lifetime and synchronization be controlled?
        queue / memory / device / locking scope
```

この矢印は feature dependency ではなく、時代ごとの**支配的な設計課題の変化**を表す。 Part VI の release chronology はこの periodization とは独立に検証する。

------------------------------------------------------------------------

### Part II から Part III へ

Part I の主要 architecture trend の観点で4 erasを読み直すと、Era 1はqueue/accountingとmulticore scalabilityの基礎、Era 2はprogrammable execution point、Era 3はprogrammabilityを支えるcommon infrastructure、Era 4はheterogeneous I/Oとcontrol scalabilityを前面化した。これらは置換関係ではなく、利用可能なexecution/memory/control modelと、それらを接続するcontractが累積してきた。Part IIIでは時代の境界を外し、queueing、aggregation、BPF/XDP、packet memory、control plane、virtualization、transport、security、RDMAなどのseriesごとに、その変化を追う。

# Part III — Long-term lineages

Part IIIはPart IIの時代区分を横切る **architecture series** を追う。release順の列挙ではなく、同じ設計課題が複数erasを通じてどう変化したかを **位置づけ → lineage / mechanism → Takeaway** の順で読む。各seriesはSynthesisで再び接続し、explicit-contract architecture trendがどこに現れ、どこでは別の説明軸が必要な箇所を区別する。driver-internal mechanismはPart IV、observability primitiveはPart V、release/provenanceはPart VI–VIIを正本とする。

## PERFORMANCE — queueing, aggregation, pacing, CPU/queue placement

<a id="detail-queueing"></a>
### Queueing / latency / pacing — backlogからplacement / time controlへ

queueingの15年は、単一qdiscの世代交代ではなく、**どこにbacklogがあり、何をfeedback signalとし、どのCPU/queueへ置き、いつ送るか**を層ごとに制御できるようになった過程である。

```text
socket / transport   TSQ · pacing · BBR
        │
qdisc                CoDel/fq_codel · CAKE · ETF/TAPRIO
        │
driver / NIC         DQL/BQL · multiqueue
        │
CPU / queue mapping  RPS/RFS/XPS
```

<a id="detail-codel"></a>
<a id="detail-aqm-pacing"></a>
#### CoDel / AQM / pacing / time scheduling

DQL/BQLはNICへ渡したbytesとcompletionをaccountし、driver/hardware側のhidden backlogを抑える。TSQはsocket側、CoDel/fq_codelはsojourn timeとflow isolation、`sch_fq`/ETF/TAPRIOはpacingとexplicit transmission timeを扱う。つまりqueue lengthだけでなく **completion・delay・time** が別々のcontrol signalになった。

<a id="detail-rps-rfs"></a>
#### CPU / queue placement

RPS/RFS/XPSはhardware queueの外側でCPU/queue placementを制御する前史であり、後年のqueue leasingとは異なる粒度で同じplacement problemを扱う。

<a id="detail-wifi-airtime"></a>
#### Wi-Fi airtime / AQL

Wi-Fiではairtime scheduling/AQLがbytesではなくmedium占有時間をaccountし、resourceに適した単位でhidden queue pressureを可視化する。

<a id="detail-tx-llist"></a>
#### TX shared-qdisc scalability

6.19の`dev_queue_xmit()` llistはshared-qdisc serializationを縮小するimplementation-scalability milestone、7.0の`cake_mq`はmodern multiqueue topologyへqdisc controlを広げるmilestoneである。これらを含め、queueingは **backpressure → delay/fairness → pacing/time scheduling → multiqueue scalability** と層を増やした。

**Takeaway:** queueing evolutionはbacklogを一か所で管理する方向ではなく、socket/qdisc/CPU/driver/NICごとに適切なaccounting・placement・time contractを持たせる方向へ進んだ。

------------------------------------------------------------------------

<a id="detail-aggregation"></a>
### Packet aggregation — GRO/GSO → BIG TCP

GRO/GSOは複数のpacketを大きなprocessing unitとして扱い、stack traversalとper-packet costをamortizeする。BIG TCPはこのmodelを64 KiB超のinternal aggregation unitへ拡張した。wire MTUを変更せず、stack内部では大きな`skb`/GSO unitを保持し、segmentation boundaryで通常のpacketへ戻す。

UDP GRO/GSOも同じprocessing-granularity modelをdatagram workloadへ広げる。`gso_size`/`gso_type`、checksum state、segment metadata、device feature advertisementが、stackとNICの間で「どこまで大きなunitを保持できるか」を接続する。

**Takeaway:** aggregationはownershipではなく **processing granularity** を拡張する並行系列であり、1回の処理で扱うpacket workを増やして高速linkのper-packet overheadを抑える。

------------------------------------------------------------------------

<a id="detail-bpf"></a>
<a id="detail-sockops"></a>
### BPF — filterからverified stack extensionへ

BPFはfilter expressionから、**verified program + typed attach point + persistent map/object**を持つkernel extension infrastructureへ拡張した。

```text
classic BPF
 → eBPF ISA
 → bpf() / maps / verifier
 → TC / cgroup / LWT / socket / XDP hooks
 → struct_ops / TCX / BPF qdisc
```

`bpf()`、maps、verifierによってprogram lifecycleとshared stateがuserspace-visible object/APIになり、その後はhookごとにcontextとhelper/kfunc capabilityを定義する方式で適用範囲が広がった。SOCK_OPS/sockmapはpacket hookからsocket/transport lifecycleへ、struct_opsはkernel algorithm implementationそのものをprogrammable boundaryへ広げる。

**Takeaway:** BPFはprogram type、attach point、map、verification boundaryを明示し、built-in kernel behaviorとprogrammable behaviorを同じsystemで組み合わせるinfrastructureになった。

------------------------------------------------------------------------

<a id="detail-xdp-afxdp"></a>
<a id="detail-xdp-extensions"></a>
### XDP / AF_XDP — early execution pointからqueue / buffer bindingへ

XDPは4.8世代にdriver RXの早い位置へverified BPF execution pointを置いた。その後DEVMAP/CPUMAPはredirect先をnamed objectへ、AF_XDPはRX queueとUMEM/userspace ringsを結び付けるqueue-bound datapathへ発展した。

```text
driver RX
  └─ XDP program
       ├─ PASS / DROP / TX
       └─ REDIRECT
            ├─ DEVMAP
            ├─ CPUMAP
            └─ XSKMAP → AF_XDP / UMEM
```

native XDPは`skb`構築前にprogramを実行し、redirect objectを通じてdevice/CPU/XSKへexecutionを分岐する。AF_XDPはRX queueとuserspace-owned UMEMを明示的にbindingし、queue assignmentとbuffer ownershipをdatapath contractへ加えた。ring/APIの実装詳細より、このbinding boundaryをarchitecture上の変化として扱う。

multi-bufferは「1 packet = 1 contiguous buffer」という初期制約を緩め、AF_XDP multi-bufferは同じ問題をXSK/UMEM側へ接続した。metadata/RX hintsはtimestamp、RSS hash等をpacket bytesとは別のtyped metadataとしてprogram/userspace pathへ渡す。

このlineageは後年のqueue leasingと共通する問題を先取りしている。kernel control/protectionを維持したまま、**execution point、redirect target、queue assignment、buffer lifetime**を明示してfast/userspace pathを選択できるようにしたからである。

**Takeaway:** XDP/AF_XDPは「早いBPF hook」から、redirect object・RX queue・UMEM ownershipを組み合わせるexplicit datapathへ発展した。

------------------------------------------------------------------------

<a id="detail-packet-memory"></a>
<a id="detail-devmem-large"></a>
### Packet memory — page_pool → netmem → memory providers

RX memoryはdriver-privateなpage allocation/recycleから、**backing typeとlifetimeをnetworking coreが表現できるmodel**へ広がった。

```text
driver-local recycle
 → page_pool
 → netmem
 → memory-provider hooks
 → system RAM / userspace / device memory
```

page_poolはRX buffer recyclingを共通化し、netmemはpacket memoryを`struct page`と同義にしないabstractionを加え、memory-provider hooksは外部backingをRX pathへ接続する。 Part IIIではこの **memory type・provider identity・queue assignment・lifetime** の分離をarchitecture storyとして扱い、page_pool/providerのdriver-facing APIとdetach/release mechanismはPart IVを正本とする。

**Takeaway:** packet memoryはallocation optimizationから、複数memory backingを明示的lifetime/assignmentの下で同じRX architectureへ接続するmodelへ進んだ。

------------------------------------------------------------------------

<a id="detail-io-uring"></a>
<a id="detail-tcp-zc-rx"></a>
<a id="detail-socket-zerocopy"></a>
### io_uring / socket zero-copy — completionとbuffer lifetimeの分離

socket zero-copyとio_uring networkingの共通点は、copy回避そのものより **submission/completionとbuffer lifetimeを明示すること**にある。

```text
application buffer
   ├─ submit/send
   ├─ kernel/NIC ownership interval
   └─ completion → application reuse

RX buffer
   ├─ kernel-selected / registered buffer
   ├─ packet delivery
   └─ recycle / return
```

`MSG_ZEROCOPY`はsend syscallの形を保ちながら、送信完了通知までapplication bufferを再利用しないlifetime modelを導入した。`IORING_OP_SEND_ZC`はこのmodelをio_uringのsubmission/completion frameworkへ統合し、送信要求とbuffer reuse可能時点をCQEで扱う。したがって **SEND_ZCはMSG_ZEROCOPYのlifetime分離モデルを継承し、completion APIをio_uringへ統合した**と整理できる。

RX側では`TCP_ZEROCOPY_RECEIVE`がmapped receive bufferをapplicationへ露出する方式を、io_uring ZCRXがregistered buffer/memory-providerとqueue pathを接続する方式を示す。後者ではpacket memory節で扱ったprovider/ownership modelとio_uring completion modelが交差する。

**Takeaway:** io_uring networkingは新しいsocket semanticsを作るより、I/O request、completion、buffer ownership/lifetimeを共通ring modelへ統合し、copy pathとzero-copy pathを同じcompletion contractで扱えるようにした。

------------------------------------------------------------------------

<a id="detail-routing-tc-offload"></a>
### Routing / TC / offload — forwarding objectとhardware mapping

routing/forwardingでは、bridge VLAN、MPLS、VRF、nexthop、TC policyを **kernel-visible object/state** として表し、必要に応じてdriver/hardwareへ写像する方向が進んだ。

```text
bridge / VLAN / FIB ── canonical state ── switchdev ── hardware
route ── nexthop object / group
TC Flower + actions ── flow rule / callbacks ── driver / hardware
```

<a id="detail-fib-nexthop"></a>
#### FIB / nexthop object

nexthop objectはgateway/device/weight等をroute entryから独立したID付きobjectへ切り出し、複数routeから再利用できるようにした。prefixとforwarding adjacencyのlifecycleを分けられるため、ECMP更新やASICのadjacency resourceとの対応も明確になる。

<a id="detail-tc"></a>
#### TC classifier / action

FlowerとTC actionsはmatch/action policyをfield-oriented representationとして表し、`ct`、redirect、VLAN、tunnel metadata等をcomposableに扱う。offloadではkernel側のpolicy representationをcanonical stateとして保ち、共通flow representation/callbackを通じてhardware executionへ配置する。

switchdevのbridge/FDB/VLAN/FIB object pathとTC offload pathは別のcontrol APIであり、driver/hardwareで交差する。この分離によって、forwarding semantics、policy representation、execution placementを独立に扱える。

**Takeaway:** routing/TC evolutionでは、forwarding adjacencyとpacket policyがnamed/reusable objectへ寄り、kernel canonical stateからsoftware/hardware executionを選べるcontrol boundaryが整備された。

------------------------------------------------------------------------

<a id="detail-routing-netlink-rtnl"></a>
<a id="detail-netdev-unreg"></a>
<a id="detail-seg6-ioam"></a>
### Routing / Netlink / RTNL — object APIとsynchronization scope

control plane scalabilityでは、**reusable routing object、machine-readable API、global lock scopeの縮小**が並行して進んだ。

```text
route → nexthop object / group

hand-written Netlink description
        → YAML specification / YNL
        → generated policy / docs / clients

global RTNL
 → unlocked operations / RCU readers
 → per-netns RTNL
 → subsystem locks / refcounts
 → RTNL-less FIB rule updates
```

IPv4 route-cache removal（3.6）はper-destination shared cacheを外した早いscalability milestoneである。nexthop objectはrouteとforwarding adjacencyを分離し、YNL/netdev-genlはNetlink family/attributesをmachine-readable schemaとgenerated toolingへ接続した。

RTNL側では、多数のnetwork object更新を一つのglobal mutexで直列化するmodelから、operation/namespace/subsystemごとに保護範囲を縮める方向へ進んだ。6.13のper-netns RTNL開始、6.15以降の分解、7.3-rc/mainlineのRTNL-less FIB rule updateとper-netns netdev unregistrationは、このserialization scope縮小の連続したmilestoneである。teardown/unregisterも同じ対象に含まれるため、「configuration fast pathだけのlock分割」とは異なる。

SRv6/IOAMは別のcontrol plane展開として、routing objectからpacket path intentやpath telemetryを指定/運搬する。host-local BPF observabilityとは観測placementが異なるが、routing/control objectを通じてnetwork path behaviorを明示する例である。

**Takeaway:** control planeはroute/device stateをnamed objectとschema-driven APIで表し、同時にglobal RTNL serializationをnamespace/subsystem単位へ縮小する方向へ進んだ。

------------------------------------------------------------------------

<a id="detail-netfilter"></a>
### netfilter / nftables / conntrack — policy-selected execution path

Netfilterはpacketごとのrule evaluationから、**transactional policy representation + shared flow state + selected fast/offload path**を組み合わせるarchitectureへ広がった。

```text
nftables policy
   ├─ sets/maps/transactions
   └─ conntrack state
          └─ flowtable
              ├─ software fast path
              └─ optional HW offload
```

nftablesはrulesetをexpression/set/mapとNetlink transactionで表現する。conntrackはdirection/state、NAT mapping、mark/label、timeout等を共有flow-stateとして提供する。flowtableはpolicy/conntrackが選んだestablished flowをsoftware fast pathへ移し、対応topologyではhardwareへさらにplacementする。exceptionはclassic forwarding pathへ戻るため、複数execution pathが同じpolicy/state semanticsの下で共存する。

BPF/XDPは別のprogrammable execution pointを提供し、Netfilterはtransactional policyとstateful flow modelを提供するため、両者はpolicy boundaryに応じて組み合わせられる。

**Takeaway:** Netfilterはpolicy representation、flow state、fast path eligibility、software/hardware placementを分離し、同じpolicy semanticsから複数execution pathを選択できるようにした。

------------------------------------------------------------------------

<a id="detail-virtual"></a>
<a id="detail-tunnel-offloads"></a>
<a id="detail-netns-fd"></a>
### Virtual networking — namespace / device / datapath / queueの選択肢

Virtual networkingは一つのvirtual-device modelへ収束せず、**isolation、overlay、guest datapath、hardware assignment、programmable virtual device**を別objectとして組み合わせる方向へ発展した。

**用語メモ:** SR-IOVはPF/VFによるhardware partition、VFIOはIOMMUを用いたdevice assignment、representorはhardware portをLinux `net_device`としてcontrolする表現、vDPAはguestのvirtio modelを維持しながらdatapathをacceleratorへplacementするframeworkである。

```text
isolation       netns / setns
virtual link    veth / ipvlan / netkit
overlay         VXLAN / GENEVE
guest path      virtio-net / vhost / vDPA
device path     SR-IOV / VFIO / representor
queue path      AF_XDP / queue assignment / leasing
```

3.0のnamespace FD/`setns()`はnetwork namespaceをprocess生成時だけの属性から、FDで参照・保持・再関連付けできるobjectへした。VXLAN（3.7）はL3 underlayとL2 tenant topologyを分離し、ipvlan（3.19）はlower deviceを共有する軽量virtual interfaceを追加した。

virtio/vhost/vDPAはguest-visible virtio semanticsとexecution placementを分離する。SR-IOV/VFIOはphysical function/resourceをVM/userspaceへ直接割り当て、representor/switchdevはそのhardware datapathをLinux control planeへ再び見せる。netkitはBPF-first virtual deviceとしてprogram placementをdevice modelへ組み込む。

overlayではtunnel GSO/GRO、checksum/segmentation metadata、TC tunnel offloadがinner/outer header境界とNIC capabilityを接続し、software encapsulationとhardware executionを選択できる。近年のAF_XDP/virtio-net zero-copyやRX HW queue leasingはさらに、virtual deviceとphysical queue/buffer resourceのassignmentを明示する。

**Takeaway:** Virtual networkingはnamespace、virtual device、overlay、guest ABI、hardware/queue assignmentを独立した接続点にし、kernel path・accelerated path・direct assignment・programmable pathを同じhostで選択できるようにした。

------------------------------------------------------------------------

<a id="detail-rdma"></a>
### RDMA — registered memoryとoffloaded executionの比較対象

RDMAは本書では通常のsocket/netdev pathと対照的なexecution / memory modelを示すselective cross-cutting case studyとして扱う。verbs pathでは **QP / CQ / MR / PD** が明示的resource objectとなり、applicationがwork requestをqueueへpostし、NIC/HCAがregistered memoryへDMAする。control/setupにはRDMA-CMやLinuxのaddress/route stateが関与し、data executionはverbs queueとHCAへ大きくplacementされる。

```text
Linux netdev / IP control state
            │
         RDMA-CM
            │
       QP / CQ / MR
        /       \
 execution      memory
   HCA      registered region
```

Linux 3.0の時点でRDMA subsystem自体は既に成熟していたため、本書では起源ではなく、対象期間内に **memory backing・lifetime・implementation placementが一般化された境界**を見る。

#### RoCEv2 / RXE — control stateとexecution placement

4.5のRoCEv2 supportはroutable Ethernet/IP fabricとverbs datapathを接続した。IP address、route、neighbor、netdev stateはLinux control planeと関係しながら、established data pathのtransport処理はHCAへplacementできる。4.8のSoft-RoCE/RXEは同じRDMA/RoCE semanticsをsoftwareでも実行できるようにし、**verbs semanticsとhardware implementationを分離できる**ことを示した。

RDMAは、connection/resource controlをLinux側で管理しつつ、registered memoryとqueue pairを用いるdata executionをHCAへ配置するarchitectureである。本書では、通常のsocket/network stackと対照的なexecution / memory modelを示すcross-cutting case studyとして扱う。

#### ODP → dma-buf MR — registered memoryのbackingを一般化する

traditional MRではprocess VA、access permission、DMA可能なbacking memoryをregistration時に強く結び付ける。3.19のODPはregistration contractと全pageのeager/permanent pinningを分離し、fault時のmappingとMMU invalidationをHCA translation lifecycleと協調させる方向を導入した。

5.12のRDMA dma-buf MRはさらに、ordinary userspace RAMだけでなくdma-bufで表現されたdevice/accelerator memoryをMRへimportする共通経路を導入した。

```text
process VA ── MR ── eager pinned RAM
       │
       └─ ODP ── fault / invalidation-managed backing

GPU / device memory ── dma-buf ── RDMA MR ── RNIC
```

ここでの変化は単なるzero-copy最適化とは異なる。**memory type、registration、permission、mapping、revocation/lifetimeを別々のcontractとして扱える範囲が広がった**ことが重要である。これはsocket/netdev側の`page_pool → netmem → memory providers`と異なるdatapath上で、よく似たarchitecture pressureへ応答した例になる。

#### AI / HPCとobservability

AI trainingはRDMAを生んだworkloadとは別系統だが、大規模collective communicationとaccelerator-resident dataによって、CPU involvement、system-memory copy、fabric congestionのcostを強く可視化した。RoCEv2/RDMAはその代表的transportであり、GPU/device memoryをRNICへ直接接続するdma-buf/P2P modelの価値も大きくなった。

同時にexecutionをHCAへ移すほど、通常のkernel packet traceだけではfailureを説明しにくくなる。LPC 2025のproduction事例が示すように、large-model trainingではQP/MR/device-resource stateとapplication errorを結び付けるobservabilityが必要になる。これはPart Vの「evidence layer」のRDMA版である。

**Takeaway:** RDMAはsocket/`skb` pathを置き換える系列ではなく、QP/CQ/MRとregistered memoryを中心にした並行execution modelである。RoCEv2/RXEはexecution placementの選択肢を、ODP/dma-bufはmemory backingとlifetimeの選択肢を広げた。AI/HPCはこの既存architectureの価値と、congestion/observabilityの課題を同時に前面化した。

------------------------------------------------------------------------

<a id="detail-congestion-signals"></a>
<a id="detail-transport"></a>
<a id="detail-mptcp"></a>
### TCP / UDP / transport — connection / feedback / path semantics

transportはpacket-memoryやprogrammabilityとは別に、**connection establishment、congestion feedback、multipath、authentication**そのものを更新してきた。

| Theme | Representative milestones | Semantics |
|---|---|---|
| establishment | TFO | handshake / early data |
| congestion / feedback | DCTCP · BBR · PLB · AccECN | signal と sender/path behavior |
| multipath | MPTCP | logical connection と subflow/path |
| authentication | TCP-AO | connection authentication / keying |

TFOはhandshakeとearly dataの境界を変える。DCTCPはECN marking、BBRはbandwidth/RTT model、PLBはpersistent congestionを受けたpath reselection、AccECNはより詳細なECN feedbackをtransport behaviorへ接続する。これらは同じalgorithmの世代交代ではなく、異なるsignal/modelをsender behaviorへ反映する並行進化である。

MPTCPではapplicationから見える一つのbyte streamの下に複数subflowを持ち、logical connection、subflow、endpoint、address advertisement、scheduler/path managerが別stateになる。MP_CAPABLE/MP_JOINとDSS mapping、in-kernel/userspace Path Manager、Generic Netlink APIにより、**connection identityとpath managementを分離したobject/control model**が形成された。7.2のPM limit expansionはこのmodelのoperational scaleを広げる。

```text
application byte stream
        ↓
MPTCP connection
   ├─ subflow / path A
   ├─ subflow / path B
   └─ Path Manager → endpoint / address / subflow lifecycle
```

TCP_REPAIRはlive TCP stateをcheckpoint/restore lifecycleへ、TSQ/pacingはsocket-side bufferingとrate-controlled transmissionへ接続する。UDP側でもzero-copyやGRO/GSOが性能を広げたが、そのarchitecture上の意味はzero-copy/aggregation節で扱う。

**Takeaway:** transport evolutionはconnection、feedback、path、authentication semanticsを独自に発展させ、その一部ではMPTCPのようにimplicitな「1 connection = 1 path」をexplicit object/control stateへ分解した。

------------------------------------------------------------------------

<a id="detail-security"></a>
### SECURITY / kTLS / XFRM / WireGuard — security stateとexecution placement

SECURITYはTRANSPORTやVIRTUAL/OVERLAYを横断し、**kernel-visible security state/policyとcrypto/packet executionの配置を分離する**storyとして整理できる。

```text
TLS socket state ── kTLS SW ── TLS NIC offload
XFRM SA/policy ─── software ── crypto offload ── packet offload
WireGuard peer/key/allowed-IP ── secure netdevice
```

kTLSはhandshake/key-exchange policyをuserspaceに残しつつ、established TLS record stateをTCP socketへ渡してTX/RX record processingをkernelへ配置する。generic TLS device offloadは同じsocket/security semanticsを維持したままcrypto executionをNICへ移せるようにした。

XFRMではSA/policyがcanonical kernel stateであり、4.12のcrypto offloadは暗号処理をNICへ、6.2のpacket offloadはpolicy/encapsulationを含むより広いexecutionをhardwareへ配置する。 Part IIIではこのplacement boundaryを扱い、driver-facing offload API/lifecycleの詳細はPart IV側のframework storyへ委ねる。

WireGuardはpeer/key/allowed-IP stateを持つL3 secure tunnelを通常の`net_device`として統合し、RTNL、Generic Netlink、UDP tunnel、GRO/GSO、NAPI等の既存networking infrastructureへ接続する。kTLS/XFRMがexecution placementを広げる例なら、WireGuardはsecurity semanticsをstandard network object/APIへ統合する例である。

| Mechanism | Canonical state | Placement / integration |
|---|---|---|
| kTLS | TLS record/crypto state on TCP socket | software / NIC TLS |
| XFRM/IPsec | SA + policy | software / crypto offload / packet offload |
| WireGuard | peer + key + allowed-IP + netdevice | normal Linux network object/API |

**Takeaway:** securityではprotocol state/policyをkernel-visible contractとして保ち、software/NIC executionまたはstandard netdevice integrationを選べるようになった。これはCONTROL PLANE、DRIVER FRAMEWORK、TRANSPORT、VIRTUAL/OVERLAYの接続点である。

------------------------------------------------------------------------

## Part IIIからPart IVへ — feature lineageからdriver contractへ

前章ではnetworking mechanismをend-to-endのlineageとして追った。 Part IVでは視点を変え、個々のdriverの責務のうち何が共通networking-core frameworkへ移されたかを見る。

# Part IV — Network Device Driver Framework

Part IVは、driver-privateなresource/state管理がnetworking coreから扱えるcommon contractへ昇格した過程を追う。対象はobject、accounting、API、assignment、lifetime、synchronizationであり、Part IIIのfeature lineageを再説明せず、**driver/framework側で何が共通化されたか**に焦点を置く。

```text
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

これらは別々のsubsystemだが、共通する方向は **「driver内部にしか見えなかったresource/stateをkernel共通のcontractとして表現する」**ことである。


## Queue accounting — DQL/BQL

DQL/BQLはdriver ringの大きさそのものを標準化するframeworkではない。driver/NICに渡したbytesとcompletionされたbytesをaccountingし、software queueがdeviceに過剰なbacklogを押し込むのを抑える仕組みである。

Part IIIのqueueing節ではBQLをlatency / pacingの一部として扱った。driver-frameworkの観点では、より重要なのは **各driverが経験則で決めていたoutstanding workを共通accounting modelにしたこと**である。

```text
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

したがってBQLのcontract formはownershipではなく **accounting / backpressure** である。


<a id="detail-switchdev"></a>
## switchdev — kernel forwarding objectとswitch ASIC

ここではswitchdevの **driver-facing object/state synchronization contract** に限定する。
switchdevはLinux bridge/FDB/VLANなどのkernel forwarding stateをswitch ASICに同期するためのframeworkとして発展した。目的は「hardware switchを特別な別世界として管理する」ことではなく、Linux networking objectをcanonical control stateとして維持しながらforwarding executionをhardwareに配置できるようにすることである。

```text
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

ただしTC hardware offloadはこの直列pathの一部ではない。TC classifier/actionのoffloadは`ndo_setup_tc`やflow-block callbackなど別のkernel API pathを通り、driver/hardware側でswitchdev系のforwarding objectと交差し得る。

```text
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

この区別はPart IIIのRouting / TC / offload節で説明したcontrol-path separationを、driver API側から見たものである。


port は通常の `net_device` として既存 Linux control plane に残るため、bridge への enslave、FDB learning notification、VLAN filtering、STP state、unsupported operation の扱いを kernel canonical state との整合性として議論できる。現在の architecture/API semantics は kernel switchdev documentation（<https://docs.kernel.org/networking/switchdev.html>）を参照。

<a id="detail-devlink"></a>
## devlink — device-wide control object

ここでは **device-wide resource / configuration / health lifecycle** をdriver frameworkとして扱う。
devlinkはport/netdev単位だけでは表しにくい **device-wide resource / parameter / port / health state** を扱うobjectとして導入された。特にswitch ASICやSmartNICのように、一つのdeviceが複数port、representor、resource、firmware stateを持つ場合、netdevだけではdevice全体のcontrol planeを表現しにくい。

```text
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

devlink healthはこのobject modelをobservability/recoveryに広げた。driver-specific debug commandの集合ではなく、reporter、diagnose、dump、recoverという共通interfaceを通してfailure stateを扱う点に意味がある。

devlinkはdatapath自体を提供しない。switchdev/TC offloadが「どこでforwardingを実行するか」を扱うのに対し、devlinkはそのdeviceの **resource / configuration / health lifecycle** を扱う。


`devlink resource` は TCAM 等の有限 hardware resource を階層 object として公開し、size 変更や occupancy を control plane から扱える。trap、rate、line card 等も device-wide management model に入り、netdev がまだ存在しない初期化段階や複数 port で共有する ASIC resource を表現できる。health reporter は error detection、dump、recover を driver-specific debugfs から共通 lifecycle へ持ち上げる。詳細は devlink docs（<https://docs.kernel.org/networking/devlink/>）、resource（<https://docs.kernel.org/networking/devlink/devlink-resource.html>）、health（<https://docs.kernel.org/networking/devlink/devlink-health.html>）。

<a id="detail-phylink"></a>
## phylink — link topologyとMAC/PHY/PCS coordination

phylinkはMAC driverがPHY、fixed-link、SFP、PCSなどのlink topologyを個別に扱う重複を減らすframeworkである。4.14世代に導入され、link negotiation / mode change / carrier stateに関する共通orchestrationをnetworking core側へ引き上げた。

```text
MAC driver
    │
  phylink
 ┌──┼──────────────┐
 ▼  ▼              ▼
PHY PCS        fixed-link / SFP
```

重要なのは「PHY APIを置き換えた」ことではない。MACとlink-side componentの関係をframeworkが仲介し、driverがtopologyごとのstate machineを重複実装する必要を減らしたことである。

Rust PHY abstractionもこのlineage上に置ける。Rust networking supportの初期段階では、high-performance NIC driver全体を書き換えるより、PHY abstractionのような比較的明確なinterface boundaryから型安全なbinding/abstractionを作る方向が先行した。


### phylinkのstate machine

phylinkはMAC driver、PHY、PCS、SFP cage、fixed-link、in-band autonegotiationの組合せで生じるlink state coordinationを共通state machineへ移す。MAC driverは`phylink_mac_ops`等を実装し、link mode validation/config、MAC prepare/config/finish、link up/downをphylink lifecycleへ接続する。

価値は「PHY を簡単にする」ことではなく、MAC/PCS/PHY の責務境界を明確にし、SFP hotplug や in-band negotiation を含む複雑な topology を driver ごとの ad-hoc state machine から外すことにある。Kernel documentation: <https://docs.kernel.org/networking/sfp-phylink.html>

<a id="detail-dim"></a>
## DIM — interrupt moderation policyの共通化

PERFORMANCE側ではadaptive moderationの効果が主題になる。ここでは **measurement → policy decision → driver setting** を共通library contractにした点だけを扱う。

Dynamic Interrupt Moderation (DIM) はpacket/byte/event rateを観測し、interrupt coalescing parameterをworkloadに応じて調整する仕組みである。複数driverが似たadaptive moderation logicを独自実装する代わりに、測定とprofile selectionを共通library/frameworkに寄せた。

```text
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

これはdatapath ownershipのcontractではなく、**measurement → policy decision → driver setting** を再利用可能にしたframework化の例である。


## page_pool — packet-memory lifecycleの共通化

page_poolの詳細な発展史はPart III「Packet memory」で扱う。ここではdriver frameworkとしての意味だけに絞る。

従来、RX driverはpage allocation、DMA mapping、recycling、fragment handlingをdriverごとに組み合わせていた。page_poolはRX packet memoryのallocation/recycling lifecycleとDMA-aware handlingを共通化し、XDPを含む高速RX pathから利用できるframeworkを提供した。

```text
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

後年のnetmem / memory providers / Device Memory TCPは、page_poolが確立したpacket-memory lifetime / recyclingの共通基盤を土台に、memory objectの表現、backing type、provider selectionを明示する方向へ拡張した。page_poolはdriverとnetworking coreの間でRX bufferのallocation / recycle / lifetimeを担い、後続の仕組みはその上で利用可能なmemory backingとconsumerを広げる。

page_poolは通常RX queue / NAPIに近い単位で配置され、packetがstackやredirect先へ渡された後もbufferがin-flightになり得るため、pool detachと最終releaseは別イベントになる。fragment support、page-pool introspection、custom memory providerもこのlifetime modelの上に積み上がる。したがって本質は「高速allocator」だけではなく、**RX packet memoryのownership / recycling / DMA lifetimeをdriver共通contractにしたこと**にある。

ここではPart IIIと同じrelease chronologyを繰り返さず、driver-frameworkとしての役割だけを保持する。

<a id="detail-netdevsim"></a>
## netdevsim — framework APIをhardwareなしで検証する

netdevsim（4.16）は実NICを模倣するための一般的なemulatorというより、networking core / driver APIを **hardware independentにselftestできるtest device** として重要である。devlink resource、FIB offload、rate objectなど、driver-facing contractが増えるほど「特定vendor hardwareなしにAPI semanticsを検証できること」がframework evolutionの一部になる。

Kernel documentation: <https://docs.kernel.org/networking/devlink/netdevsim.html>


<a id="detail-auxiliary-bus"></a>
## auxiliary bus — device内部機能の分割

auxiliary busは、一つのphysical device/PCI functionに含まれる複数の機能を、親driverと補助driverの間で分離して扱うための共通infrastructureである。networking専用ではないが、SmartNIC/RDMA/network driverのように一つのdeviceが複数subsystemへ機能を公開する構成で重要になった。

これはdevlinkのようなuserspace-visible control objectとは役割が異なる。auxiliary busは **driver composition / binding boundary** を提供する。


### auxiliary busのcomposition contract

auxiliary busは1つのphysical PCI device/driverが内部に複数の機能単位を持ち、それぞれを別driver/moduleへbindしたい場合のcomposition mechanismである。network driverではRDMA、crypto、subfunction等とcore device stateを共有しながら、巨大なmonolithic driverに統合し続けるのを避ける用途がある。

ここで共有されるのはhardwareそのものだが、lifetimeとprobe/remove boundaryはauxiliary device/driver objectとして明示される。このためPart IVではperformance featureではなく **driver composition / lifetime contract** として扱う。

<a id="detail-threaded-napi"></a>
## threaded NAPI — execution contextの選択肢

NAPIは通常softirq contextでpollされるが、threaded NAPIはpoll処理をkernel threadで実行できる選択肢を追加した。これはXDPやAF_XDPのような新しいpacket pathではなく、既存NAPI processingの **execution context / scheduling placement** を変える仕組みである。

real-time性、CPU scheduling、isolationの要件によってsoftirqとthreaded executionを選択できる点が重要であり、「一つの最適なNAPI execution model」へ統一する変更ではない。6.19世代ではthreaded NAPIにbusy-poll modeが加わり、threaded executionの中でもIRQ-drivenとpolling-orientedな配置を選べる方向へ進んだ。

Kernel netdev specification: <https://docs.kernel.org/7.1/netlink/specs/netdev.html>


<a id="detail-threaded-busypoll"></a>
### Threaded NAPI busy-poll — NAPIごとのdedicated polling execution

threaded NAPIはsoftirqではなくkernel threadでNAPI pollを実行する選択肢を導入した。後のthreaded busy-poll拡張は、そのthreadをcontinuous pollingさせ、RX/TX descriptorを低jitterで回収する用途を追加する。NetlinkからNAPI単位でthreaded/busy-pollを選択し、userspaceがthread PIDを取得してaffinity、priority、scheduler policyを設定できる設計である。

これは単なる busy loop optimization ではなく、**NAPI object ごとに execution context と CPU scheduling policy を割り当てる contract**への進化である。AF_XDP の hard low-latency use case が series の motivation として明示されている。LWN patch series: <https://lwn.net/Articles/1030809/>

<a id="detail-ethtool-genl"></a>
## ethtool netlinkとYNL — driver control APIの構造化

CONTROL PLANEの時系列を繰り返さず、ここではethtool familyの **structured API / schema evolution** とdriver-facing control contractに焦点を置く。

従来のethtool ioctl interfaceは長年利用されてきたが、機能追加、dump、notification、extensible attributeという面ではNetlinkの方が扱いやすい。ethtool netlinkはlink modes、coalescing、channelsなどのdevice configurationをstructured Netlink APIへ移す方向を示した。

さらにYNLはNetlink familyのschemaをmachine-readableに記述し、policy/documentation/userspace helper生成へ接続する。ethtool netlinkとYNLは同じfeatureではないが、

```text
ad-hoc ioctl / hand-written Netlink
              ↓
structured Netlink objects
              ↓
machine-readable schema / generated tooling
```

というcontrol APIの長期的な方向を示す。


Generic Netlink family `ethtool`はlink modes、features、rings、channels、coalescing、pause、EEE、FEC、module、stats等をnested attributesで表し、GET / SET / ACT、dump、notification、extended ACK、bitset representationを利用できる。移行はioctlを一度に廃止する方式ではなく機能単位でNetlink commandを追加する形で進み、後のYNLはstructured APIをYAML schemaから扱う方向をさらに明確にした。

<a id="detail-netdev-genl"></a>
## netdev-genl — queue / NAPI identityをcontrol plane objectへ

Observability側ではqueue/NAPI/page_pool visibilityが主題になる。ここでは後続のqueue-bound APIが参照できる **stable object identity / schema contract** に焦点を置く。

netdev generic Netlink familyでは、NAPI instanceやRX/TX queueのidentityをuserspaceから列挙・参照できる方向が進んだ。これは単なるobservability enhancementではない。AF_XDP、io_uring ZCRX、memory providers、queue leasingのように **特定queueへresourceをbindingするAPI** が増えると、queueそのものを安定して指し示すcontrol plane identityが必要になるためである。

```text
physical netdev
   ├─ NAPI object
   ├─ RX queue object
   └─ TX queue object
          │
          ├─ statistics / introspection
          └─ later queue-bound APIs
```

ここでqueue identityはOBSERVABILITYとCONTROL PLANEの接点になる。


<a id="detail-driver-modern"></a>
### Modern driver contracts — XDP features / queue objects / Rust PHY

近年のdriver frameworkでは「このdriverがXDPを実装しているか」というbinaryな判定から、redirect、ndo_xmit、zero-copy等のfeature capabilityをmachine-readableに公開する方向へ進んだ。netdev-genlのqueue/NAPI/page_pool objectと組み合わせると、control planeは **capability + resource identity + assignment** を別々に問い合わせられる。

Rust PHY supportはnetwork driver全体をRustへ置き換えるものではなく、PHY abstractionという比較的明確なdriver contractからRust binding/implementationを導入した例である。本書では言語移行そのものより、既存C subsystem APIのownership/lifetime ruleをRust type systemへどう写像するかというDRIVER FRAMEWORKの変化として位置付ける。

## Driver frameworkのsynthesis

15年間を通して見ると、driver frameworkの変化は次のように要約できる。

| 以前 driver 内に埋もれていたもの | 共通化した代表 framework | 代表 release | 主な contract | Evidence / source |
|---|---|---:|---|---|
| outstanding TX work | DQL/BQL | 3.3 | accounting (backpressure) | LWN/netdev patch series: <https://lwn.net/Articles/469652/> |
| switch forwarding state | switchdev | 3.19 | object / synchronization (notification) | Kernel documentation: <https://docs.kernel.org/networking/switchdev.html> |
| device-wide resource | devlink | 4.6 | object / API | Kernel documentation: <https://docs.kernel.org/networking/devlink/> |
| MAC–PHY/PCS topology | phylink | 4.14 | API / synchronization | Kernel documentation: <https://docs.kernel.org/networking/sfp-phylink.html> ; origin anchor: Part VII |
| hardware-independent API test | netdevsim | 4.16 | API (contract validation) | Kernel documentation: <https://docs.kernel.org/networking/devlink/netdevsim.html> |
| interrupt moderation logic | DIM | 4.16→5.3 | accounting / API | Kernel documentation: <https://docs.kernel.org/networking/net_dim.html> ; 5.3 integration anchor: Part VII |
| RX packet-memory recycling | page_pool | 4.18 | lifetime / object | Kernel documentation: <https://docs.kernel.org/networking/page_pool.html> ; origin anchor: Part VII |
| device health/recovery | devlink health | 5.1 | lifetime / API | Kernel documentation: <https://docs.kernel.org/networking/devlink/devlink-health.html> |
| NIC configuration | ethtool netlink | 5.6 | structured API | LWN 5.6 merge window: <https://lwn.net/Articles/810780/> |
| multi-function driver binding | auxiliary bus | 5.11 | object / lifetime | origin anchor: Part VII |
| NAPI execution placement | threaded NAPI | 5.12; busy-poll 6.19 | assignment / synchronization (execution placement) | release attribution: Part VI; exact anchor がある項目は Part VII |
| queue / NAPI identity | netdev-genl | 6.8+ | object / assignment | kernel Netlink spec: <https://docs.kernel.org/networking/netlink_spec/netdev.html> ; queue-object anchor: Part VII |


この表は各frameworkが同じ問題を解くという意味ではない。共通するのは、**driver-privateだったstate/resource/algorithm boundaryをkernel共通のcontractへ引き上げ、複数driver・複数hardware・複数execution modelから再利用可能にしたこと**である。

Part IIIのMEMORY/PROGRAMMABILITY/CONTROL PLANEが「datapath側で何が共存できるようになったか」を説明するのに対し、 Part IVはその共存を支える **driver-facing contract surface** がどう増えたかを説明する。

------------------------------------------------------------------------

<a id="detail-observability"></a>
# Part V — Observability / Explainability

Part Vは **kernel observability primitiveと、それを相関するtool-side case study** に責務を限定する。execution pointが分散するほど、packet traceだけでなくtyped reason、object identity、timestamp、resource stateを横断するevidence layerが必要になる。各primitiveの「いつ入ったか」はPart VI、datapath自体のmechanismはPart III/IVを正本とし、ここでは「どのevidenceから挙動を説明できるか」を扱う。Retis / pwruはその相関例である。

## Kernel observability primitive

Linux networkingのobservabilityは、単純なinterface counter / packet captureから、 kernel内部のtyped eventとpacket-lifecycle metadataを相関できる方向へ進化した。

```text
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

### Primitiveが提供する情報と限界

| Primitive | 主に提供するもの | 境界 / 注意点 |
|:------------------------|:-------------------------------------------------|:----------------------------------------------------------------|
| BTF | kernel type / field metadata | runtime event や packet trajectory 自体は記録しない |
| eBPF tracing | attach point での event / state | coverage、attach point、権限、overhead に依存 |
| `skb_drop_reason` | 対応箇所での structured drop reason | 全 drop path が必ず reason を付与するわけではない |
| timestamping | 特定地点での時刻情報 | clock、取得地点、HW/SW timestamp semantics に依存 |
| netdev-genl | queue / NAPI 等の identity、state、configuration | 可視化できることと自由に ownership/configuration できることは別 |

したがってcross-layer packet journeyは単一primitiveの機能ではなく、toolが複数のidentifier、event、 timestamp、metadataを相関して **推論する目標**として扱う。

<a id="detail-btf"></a>
### BTFとeBPF tracing

BTFによりrunning kernelの型情報を利用できるため、observability toolはprivate kernel structureの固定offsetに依存する必要を減らせる。これはnetworkingのように内部構造の変化が 速い領域で特に重要である。

BTFは単なるdebug symbolの圧縮ではなく、struct / union / enum / function prototype等をtype IDで参照できるmachine-readable schemaとして働く。libbpf/CO-REはこの型情報を使ってcompile時とruntime kernelのlayout差をrelocationでき、tracing toolは`struct sk_buff`や`struct sock`のfieldをversionごとのhard-coded offsetに依存せず参照しやすくなる。

したがって BTF は PROGRAMMABILITY と OBSERVABILITY の接続点である。ただし BTF 自体を本書の contract form に追加するのではなく、観測側が利用できる **typed evidence** と位置付ける。Kernel documentation: <https://docs.kernel.org/bpf/btf.html>

<a id="detail-drop-reason"></a>
### Structured drop reason

Linux 5.17の`kfree_skb_reason()` / `skb_drop_reason`世代は、「packetが消えた」という観測を 「どの理由でdropされたか」というstructured metadataへ変えた。その後、coverageはnetworking stackの各所へ拡張されている。

7.1のdedicated qdisc-drop tracepointは、qdisc内のdrop contextをgeneric tracingから直接観測しやすくした例であり、structured drop reasonと補完関係にある。

`kfree_skb_reason()` / `enum skb_drop_reason`は、toolがfree siteから理由を推測するだけでなく、drop producer側がsemantic reasonをeventに付与できるようにする。IP / neighbour / core receive path / TCP等へcoverageが拡大し、subsystem固有reasonも扱えるようになった。

ここでの要点は、counter、tracepoint、call-site、structured reasonが併存し、問いに応じて異なるevidenceを提供することである。

### Timestampingとpacket lifecycle

`SO_TIMESTAMPING`、driver/hardware timestamp、BPFから取得できる時刻・contextは、 単一地点のpacket captureでは見えないqueueing / scheduling / offloadの時間軸を補う。

software timestampとNIC hardware timestampはpacket eventをclock domainに結び付ける。PHC (PTP Hardware Clock) を含むtimestampingは、pwru/Retisが扱うpacket provenanceとは別種類のevidenceであり、「どこを通ったか」に対して「いつ起きたか」を与える。

### Queue / NAPI / page_pool observability

Part IVで説明したqueue、NAPI、page_poolのobject化は、control planeだけでなくobservabilityにも 寄与する。packet memory、polling context、queue identityをuserspace-visible objectとして関連付ける ことで、zero-copy / memory-provider時代の問題を説明しやすくなる。ただし、readableなidentity / statsと、 userspaceがqueueやNAPIのownership/configurationを変更できるcontrol plane capabilityは区別する。

この軸をPart VIの確定milestoneに対応させると、代表的なobservability milestoneの**時系列**は次のようになる。これは機能間の依存関係を示す図ではない。

```text
4.1   kprobe-attached eBPF / tracing foundation
5.17  kfree_skb_reason / structured drop-reason foundation
5.19  drop-reason coverage expansion
6.8   queue/NAPI netdev-genl visibility
7.1   dedicated qdisc-drop tracepoint
```

`netdev-genl`によるqueue / NAPI identityとstatsは、従来のinterface-level counterだけでは説明しにくかった「どのqueue / pollerで問題が起きているか」をuserspaceから関連付ける基盤になる。page_pool側でもpool identity、allocation / recycle information、leak diagnosticsが整備され、packet-memory ownership自体が 観測対象へ移っている。ここでのrelease attributionはPart VIの値を参照する。

## Toolのcase study

Retisとpwruは上記primitiveを利用する代表例だが、本書ではkernel evolutionそのものと区別する。

- **pwru**: 広いkernel function trajectoryから「packetがどこを通ったか」を探索する。
- **Retis**: networking event、skb metadata、OVS/OVN contextなどを意味的にenrichして相関する。

詳細なconference/source provenanceはAppendixのcase-study indexに集約する。

pwruは **function-level trajectory** を広く探索するdebuggerとして強く、Retisはnetworking subsystemのstructured event / metadataをpacket journeyへ相関する方向に強い。両者は単純な代替関係ではない。kernel primitive側のBTF、BPF tracing、drop reason、object identityがricherになるほど、tool側はraw traceの羅列からsemantic correlationへ進める。


# Synthesis — architecture framingへの回帰と未完の仕事

Linux networkingの15年は、一つのmechanismへ収束した歴史ではなく、queueing、transport、programmability、memory、control plane、virtualization、security、driver frameworkなどが異なるpressureに応答して並行進化した歴史である。その中で複数seriesを横断して繰り返し現れたのが、**resource / state / lifetime / assignment / synchronizationを明示的contractとして表し、複数のexecution / memory modelを同じsystemで共存させる**方向である。本書ではこれをLinux networking全体を還元する法則ではなく、15年間を理解するための主要architecture trendとして位置づける。

Part Iの「1本のRX queueを誰が使うのか」に戻ると、 Part IIのscalability pressure、 Part IIIのXDP/AF_XDP・packet memory・io_uring・virtual networking、 Part IVのdriver contractが一つの場所で交差する。2011年頃にはdriver/NAPI/kernel stackに強く結び付いていたqueueが、2026年にはAF_XDP、io_uring ZCRX、device-memory path、virtual deviceなど複数consumerとの間でidentity/assignment/lifetimeを明示して扱えるresourceへ変わっている。重要なのは全packetが新しいpathへ移ったことではなく、**同じNIC resourceへ異なるmodelを安全に接続するための境界が増えたこと**である。

## Contractが共存を可能にした接続点

| 明示化された対象 | Contract の lineage | その接続点で共存できるようになったもの |
|---|---|---|
| queue occupancy | DQL/BQL → TSQ / `sch_fq` / pacing | socket・qdisc・driver が backlog を別々の層で制御する multi-layer control |
| execution point | `bpf()` / maps / verifier → XDP・TC・cgroup・socket hooks → `struct_ops` / TCX / BPF qdisc | built-in stack と verified program が同じ packet・socket・algorithm を扱う |
| packet memory | page_pool → netmem → memory providers | system-RAM page、userspace-owned buffer、device memory が同じ RX architecture に接続される |
| registered / peer memory | RDMA MR → ODP → dma-buf MR | pinned userspace RAM、on-demand resident memory、accelerator/device memory を同じ verbs object model から扱う |
| queue identity | AF_XDP queue / UMEM binding → netdev-genl queue/NAPI objects → RX HW queue leasing | NIC queue を kernel stack、AF_XDP、io_uring ZCRX、device-memory path、virtual device が用途に応じて使い分ける |
| forwarding / device state | switchdev・devlink → representor / TC offload → XFRM packet offload | kernel が canonical semantics/control state を保持しながら software path と hardware path を選べる |
| control plane state | nexthop objects → YNL → per-netns RTNL | reusable object / machine-readable API / narrower synchronization scope の下で configuration を並行化する |
| policy / flow state | nftables → conntrack consumers → flowtable / HW offload | transactional policy と shared state を保ちながら slow / fast / hardware path を選択する |
| security state / policy | kTLS / XFRM / WireGuard | kernel-visible security state を保ちつつ software / NIC execution または通常の netdevice integration を選択 |

表中の`→`は図の記法どおりlineage continuationであり、直接依存を意味しない。また各行は「一つのcontractが後続機能を直接生んだ」という因果を主張するものではなく、**共存を可能にしたarchitecture上の接続点**を要約している。

この表の各行はPart Iの6軸と一対一には対応しない。6軸は「何が変わったか」の分類であり、この表は「何が明示されたから、何が共存できたか」という読み方である。OBSERVABILITYは独立した行ではなく、typed drop reason、queue/NAPI identity、page_pool introspection、RDMA QP/MR/device-state tracingなどを通じ、これらの接続点で実際に何が起きているかを検証するevidence layerとして全行にかかる。

## 並行して進んだarchitecture series

explicit-contract architecture trendと並行して、別の設計軸も継続的に進化した。

- **Transport semantics / algorithms:** TFO、DCTCP、BBR、MPTCP、PLB、AccECNは、接続確立、輻輳制御、multipath、loss/ECN signalの扱いを発展させた。
- **Processing granularity:** GRO/GSOからBIG TCPへの系列は、packet processingの単位を拡大してper-packet costを下げた。
- **Implementation scalability:** `dev_queue_xmit()`のllist化などは、共有data structureのcontentionを減らす実装上のscalability改善である。
- **CPU / queue placement:** RPS/RFS/XPSは、packet processingをどのCPU / queueに配置するかという系列であり、後のqueue assignment / leasingを理解する前史になる。

これらをexplicit contractsの派生として扱うのではなく、Linux networkingを構成する並行seriesとして重ねて読む。explicit-contract architecture trendは、その中でもexecution / memory / control modelの共存を説明する横断軸である。

## 6軸への回帰

- **PERFORMANCE:** queueingはdriver-local tuningからaccounting / pacing / multi-layer schedulingへ進み、同時にBIG TCPやscalable TXのようなcontract外の高速化も進んだ。
- **PROGRAMMABILITY:** BPF/XDPは単一fast pathからsocket / protocol / qdisc / virtual-deviceへ広がり、再利用可能なinfrastructureになった。
- **MEMORY:** page_poolからnetmem / memory providers / device memoryへ、allocation・lifetime・assignmentが明示的になった。
- **CONTROL PLANE:** reusable objects、transactional policy、YNL schema、per-netns / fine-grained lockingによりembedded / global stateの範囲が縮小した。
- **OBSERVABILITY:** typed reason / identity / tracingは、複雑化したprogrammable / offloaded / zero-copy pathをoperationally explainableにするevidence layerになった。
- **DRIVER FRAMEWORK:** BQL、switchdev、devlink、phylink、DIM、page_pool、netdev-genlはdriver-local conventionをcommon contractへ引き上げた。

## 読者にとっての含意

- **driver / kernel developer:** fast pathそのものだけでなく、resource identity、ownership/lifetime、fallback、synchronization、observabilityをuAPI/driver contractの一部として設計する。
- **operator / SRE:** `skb` packet traceだけを前提にせず、queue/NAPI、page_pool、offload state、RDMA QP/MRなど「どのobjectがどこで実行されているか」を観測対象にする。
- **researcher / architect:** kernel bypass対kernel stackという二択より、control semanticsをどこに残し、data executionとmemory ownershipをどこへplacementするかというdesign spaceとして比較する。

## 未完の仕事 — contractがまだ届いていない箇所

未完点は「次に入る機能」の一覧ではなく、**explicit contractsの適用範囲がまだ届いていない箇所**として読む。

- **TX側のqueue assignment** — RX HW queue leasingは7.1に入ったが、TX queue leasingは未mergeである。assignment contractはRX側に偏っている。
- **global synchronizationの縮小** — per-netns RTNLはglobal RTNLの一括置換ではなく、複数releaseにわたるmigrationである。7.3-rc / mainlineのRTNL-less FIB-rule更新とper-netns netdev unregistrationは、その途中経過にあたる。
- **異種memoryの一般化** — netmem（6.9）と汎用page_pool memory-provider hooks（6.15）はcanonical boundaryを確定した。残る課題は、より多くのdriver / memory typeがこのprovider-aware modelを利用できるようにすることである。
- **execution placementの共通化** — switchdev pathとTC offload pathは並行経路のままであり、XDPもnative / generic / offloadで実行位置と必要なdriver supportが異なる。softwareとhardwareのどちらで実行するかを扱う単一のcontractはない。
- **observabilityのcoverage** — drop reasonはすべてのdrop pathに付くわけではなく、cross-layerのpacket journeyは今もtool側の推論に依存する。
- **Rust** — networking側はPHY / core abstractionが中心で、一般的なhigh-performance NIC driverがRustへ移行した段階ではない。

これらは将来のreleaseで評価が変わり得るため、確定史ではなく本書のopen questionsとして扱う。


------------------------------------------------------------------------
# Part VI — Canonical release chronology

ここだけが **release attribution と Verification status の正本**である。 Part I–V の version 表記は story/lineage の参照であり、この表を上書きしない。

## 採用基準

Part VI は release note の網羅表ではない。採用するのは、本文で追う 6 軸の長期 lineage、 または TRANSPORT / VIRTUAL-OVERLAY / SECURITY の cross-cutting domain を理解するために必要な **origin / integration / enablement milestone** と、architecture 上の operational range を大きく変える milestone である。

単発の機能追加、局所的な数値上限変更、後続 lineage を変えない個別 protocol/offload enhancement は 原則として採用しない。

本文に version が現れても、それが補足的な evolution / scale change / removal であり上記の採用基準を満たさない場合、 Part VI の独立 milestone には正規化しない。本文の version 表記はその説明範囲に限定する。「Axis を付けられる」だけでは採用理由にならず、 Part I–V の story で役割を 説明できることを要求する。

**Axis と Domain は独立である。** Axis は architecture 上の主問題を示す。Domain は単なる実装場所ではなく、 protocol semantics / path management / authentication / virtual-overlay behavior 自体が主題となる場合に付与する。 Axis は分類軸であり、すべての変化が explicit-contract architecture trend に属するわけではない。たとえば BIG TCP は PERFORMANCE 軸だが、contract 化ではなく processing-unit expansion として説明する。

## Verification status

Part VI は chronology であり、この列は **参照先ではなく attribution の検証状態**を示す。外部 source は Part VII の exact anchor と Appendix の release から辿る。

- **release** — final release containment を確認済み。
- **anchor: Part VII** — final release に加え、 Part VII に 40 桁 representative mainline SHA がある。
- **series + release** — patch series/pull と final release の双方で確認。
- **mainline; final pending** — Linus mainline merge 済みで final release は未公開。

## Canonical milestone table


### 年表から architecture detail への読み方

`Architecture detail` 列は、各 milestone を Part III / IV の説明へ逆リンクする。ここでは release attribution を説明し直さず、**年表 → mechanism / object model → Part VII exact anchor または source audit** の順に辿れるようにする。`—` は milestone が独立した詳細節を必要としないか、現時点で release chronology のみを保持していることを意味する。


`Architecture detail` は年表の各 milestone と本文を1対1で再説明する列ではない。複数 milestone が同じ architecture series に属する場合は、共通の節へリンクする。たとえば bridge VLAN / MPLS / VRF は Routing / TC / offload 節、TCP zero-copy RX / socket zero-copy は io_uring / socket zero-copy 節で系列として読む。

### 3.0–3.18

| Release | Milestone | Axis | Domain | Verification status | Architecture detail |
| :-------- | :------------------------------------------------ | :------------------------------- | :------------------ | ---------------- | --- |
| 3.0 | namespace FD / setns() | CONTROL PLANE | — | release | [namespace FD / setns](#detail-netns-fd) |
| 3.3 | DQL/BQL | PERFORMANCE / DRIVER FRAMEWORK | — | anchor: Part VII | [queueing](#detail-queueing) |
| 3.5 | CoDel | PERFORMANCE | — | anchor: Part VII | [CoDel / AQM / pacing](#detail-codel) |
| 3.5 | fq_codel | PERFORMANCE | — | release | [AQM / scheduling](#detail-aqm-pacing) |
| 3.6 | TSQ | PERFORMANCE | — | release | [queueing](#detail-queueing) |
| 3.6 | TFO client | — | TRANSPORT | release | [transport](#detail-transport) |
| 3.6 | IPv4 route-cache removal | CONTROL PLANE | — | release | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 3.7 | VXLAN | — | VIRTUAL / OVERLAY | release | [virtual networking](#detail-virtual) |
| 3.7 | TFO server | — | TRANSPORT | release | [transport](#detail-transport) |
| 3.9 | bridge VLAN filtering infrastructure | CONTROL PLANE | VIRTUAL / OVERLAY | anchor: Part VII | [routing / TC / offload](#detail-routing-tc-offload) |
| 3.9 | TCP/UDP SO_REUSEPORT | PERFORMANCE | — | anchor: Part VII | [queueing](#detail-queueing) |
| 3.11 | SO_BUSY_POLL | PERFORMANCE | — | release | [queueing](#detail-queueing) |
| 3.12 | sch_fq pacing | PERFORMANCE | TRANSPORT | release | [queueing](#detail-queueing) |
| 3.13 | nftables | CONTROL PLANE | — | anchor: Part VII | [Netfilter](#detail-netfilter) |
| 3.15 | internal BPF ISA rework | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 3.18 | bpf() / maps / verifier generation | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 3.18 | DCTCP | — | TRANSPORT | release | [transport](#detail-transport) |
| 3.18 | Geneve | — | VIRTUAL / OVERLAY | release | [virtual networking](#detail-virtual) |


### 3.19–4.20

| Release | Milestone | Axis | Domain | Verification status | Architecture detail |
| :-------- | :------------------------------------- | :------------------------- | :--------------------- | ---------------- | --- |
| 3.19 | RDMA On-Demand Paging core | MEMORY / DRIVER FRAMEWORK | — | anchor: Part VII | [RDMA](#detail-rdma) |
| 3.19 | switchdev origin | DRIVER FRAMEWORK | — | release | [switchdev](#detail-switchdev) |
| 3.19 | ipvlan | — | VIRTUAL / OVERLAY | release | [virtual networking](#detail-virtual) |
| 3.19 | SO_ATTACH_BPF | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 4.1 | MPLS routing / AF_MPLS | CONTROL PLANE | VIRTUAL / OVERLAY | anchor: Part VII | [routing / TC / offload](#detail-routing-tc-offload) |
| 4.1 | cls_bpf / act_bpf eBPF support | PROGRAMMABILITY | — | series + release | [BPF](#detail-bpf) |
| 4.1 | kprobe BPF milestone | OBSERVABILITY | — | series + release | [observability](#detail-observability) |
| 4.2 | Flower classifier | PROGRAMMABILITY | — | anchor: Part VII | [TC / Flower](#detail-tc) |
| 4.3 | VRF device | CONTROL PLANE | VIRTUAL / OVERLAY | anchor: Part VII | [routing / TC / offload](#detail-routing-tc-offload) |
| 4.5 | RoCEv2 core + mlx4/mlx5 support | CONTROL PLANE / DRIVER FRAMEWORK | — | anchor: Part VII | [RDMA / RoCE](#detail-rdma) |
| 4.6 | devlink | DRIVER FRAMEWORK | — | anchor: Part VII | [devlink](#detail-devlink) |
| 4.7 | TC BPF direct packet access | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 4.8 | Soft-RoCE / RXE | DRIVER FRAMEWORK | — | anchor: Part VII | [RDMA](#detail-rdma) |
| 4.8 | XDP | PROGRAMMABILITY | — | series + release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 4.9 | BBR | — | TRANSPORT | anchor: Part VII | [transport](#detail-transport) |
| 4.10 | cgroup BPF | PROGRAMMABILITY | — | series + release | [BPF](#detail-bpf) |
| 4.10 | BPF LWT | PROGRAMMABILITY | — | series + release | [BPF](#detail-bpf) |
| 4.12 | XFRM crypto offload API | DRIVER FRAMEWORK / CONTROL PLANE | SECURITY | anchor: Part VII | [security](#detail-security) |
| 4.13 | SOCK_OPS | PROGRAMMABILITY | TRANSPORT | release | [SOCK_OPS](#detail-sockops) |
| 4.13 | kTLS TX / kernel TLS record datapath | PERFORMANCE / CONTROL PLANE | SECURITY / TRANSPORT | anchor: Part VII | [security](#detail-security) |
| 4.14 | phylink | DRIVER FRAMEWORK | — | anchor: Part VII | [phylink](#detail-phylink) |
| 4.14 | SOCKMAP | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 4.14 | XDP devmap | PROGRAMMABILITY | — | release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 4.14 | TCP MSG_ZEROCOPY | PERFORMANCE | — | anchor: Part VII | [socket zero-copy](#detail-socket-zerocopy) |
| 4.15 | XDP cpumap | PROGRAMMABILITY | — | release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 4.16 | netdevsim | DRIVER FRAMEWORK | — | anchor: Part VII | [netdevsim](#detail-netdevsim) |
| 4.16 | Net DIM initial generation | DRIVER FRAMEWORK | — | series + release | [DIM](#detail-dim) |
| 4.16 | nftables software flowtable | PERFORMANCE | — | release | [Netfilter](#detail-netfilter) |
| 4.17 | BPF_PROG_TYPE_SK_MSG | PROGRAMMABILITY | — | anchor: Part VII | [BPF](#detail-bpf) |
| 4.17 | kTLS RX | PERFORMANCE / CONTROL PLANE | SECURITY / TRANSPORT | anchor: Part VII | [security](#detail-security) |
| 4.18 | AF_XDP | MEMORY / PROGRAMMABILITY | — | series + release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 4.18 | page_pool origin / XDP memory return | MEMORY | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 4.18 | TCP_ZEROCOPY_RECEIVE | PERFORMANCE | — | release | [Socket / TCP zero-copy RX](#detail-tcp-zc-rx) |
| 4.18 | BTF origin / typed BPF metadata | PROGRAMMABILITY / OBSERVABILITY | — | anchor: Part VII | [BTF](#detail-btf) |
| 4.18 | generic TLS device offload TX | DRIVER FRAMEWORK / PERFORMANCE | SECURITY / TRANSPORT | anchor: Part VII | [security](#detail-security) |
| 4.19 | SO_TXTIME | PERFORMANCE | — | release | [queueing](#detail-queueing) |
| 4.19 | CAKE | PERFORMANCE | — | release | [AQM / scheduling](#detail-aqm-pacing) |
| 4.19 | generic TLS device offload RX | DRIVER FRAMEWORK / PERFORMANCE | SECURITY / TRANSPORT | anchor: Part VII | [security](#detail-security) |
| 4.20 | TCP EDT | PERFORMANCE | — | release | [queueing](#detail-queueing) |
| 4.20 | taprio | PERFORMANCE | — | release | [AQM / scheduling](#detail-aqm-pacing) |
| 4.20 | BPF flow dissector | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |

### 5.0–6.1

| Release | Milestone | Axis | Domain | Verification status | Architecture detail |
| :-------- | :--------------------------------------- | :----------------- | :--------------------- | ---------------- | --- |
| 5.0 | UDP GRO | PERFORMANCE | — | release | [packet aggregation](#detail-aggregation) |
| 5.0 | UDP MSG_ZEROCOPY | PERFORMANCE | — | release | [socket zero-copy](#detail-socket-zerocopy) |
| 5.1 | devlink health | DRIVER FRAMEWORK | — | release | [devlink](#detail-devlink) |
| 5.1 | mac80211 airtime accounting/scheduling | PERFORMANCE | — | release | [Wi-Fi airtime](#detail-wifi-airtime) |
| 5.3 | nexthop objects | CONTROL PLANE | — | release | [FIB / nexthop](#detail-fib-nexthop) |
| 5.3 | TC ct action | PROGRAMMABILITY | — | anchor: Part VII | [routing / TC / offload](#detail-routing-tc-offload) |
| 5.3 | Net DIM common-library integration | DRIVER FRAMEWORK | — | anchor: Part VII | [DIM](#detail-dim) |
| 5.5 | nftables flowtable hardware offload | PERFORMANCE / DRIVER FRAMEWORK | — | release | [Netfilter](#detail-netfilter) |
| 5.5 | mac80211 AQL | PERFORMANCE | — | release | [Wi-Fi airtime](#detail-wifi-airtime) |
| 5.6 | MPTCP | — | TRANSPORT | release | [MPTCP](#detail-mptcp) |
| 5.6 | BPF struct_ops / TCP CC | PROGRAMMABILITY | TRANSPORT | release | [BPF](#detail-bpf) |
| 5.6 | ethtool Generic Netlink | DRIVER FRAMEWORK | — | anchor: Part VII | [ethtool Netlink](#detail-ethtool-genl) |
| 5.6 | WireGuard mainline secure L3 netdevice | DRIVER FRAMEWORK / CONTROL PLANE | SECURITY / VIRTUAL / OVERLAY | anchor: Part VII | [security](#detail-security) |
| 5.7 | vDPA bus | DRIVER FRAMEWORK | VIRTUAL / OVERLAY | anchor: Part VII | [virtual networking](#detail-virtual) |
| 5.9 | BPF_PROG_TYPE_SK_LOOKUP | PROGRAMMABILITY | — | anchor: Part VII | [BPF](#detail-bpf) |
| 5.11 | auxiliary bus | DRIVER FRAMEWORK | — | anchor: Part VII | [auxiliary bus](#detail-auxiliary-bus) |
| 5.12 | RDMA dma-buf memory region | MEMORY / DRIVER FRAMEWORK | — | anchor: Part VII | [RDMA](#detail-rdma) |
| 5.12 | threaded NAPI | DRIVER FRAMEWORK | — | release | [threaded NAPI](#detail-threaded-napi) |
| 5.17 | structured drop-reason foundation | OBSERVABILITY | — | release | [drop reason](#detail-drop-reason) |
| 5.18 | XDP multi-buffer / frags generation | PROGRAMMABILITY | — | series + release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 5.19 | IPv6 BIG TCP | PERFORMANCE | — | anchor: Part VII | [packet aggregation](#detail-aggregation) |
| 5.19 | drop-reason expansion | OBSERVABILITY | — | release | [drop reason](#detail-drop-reason) |
| 6.0 | io_uring SEND_ZC | PERFORMANCE | — | release | [io_uring](#detail-io-uring) |
| 6.0 | io_uring multishot receive | PERFORMANCE | — | release | [io_uring](#detail-io-uring) |

### 6.2–7.3-rc

| Release | Milestone | Axis | Domain | Verification status | Architecture detail |
| :-------- | :------------------------------------------------------ | :--------------------------------- | :--------------------- | ---------------- | --- |
| 6.2 | TCP PLB | — | TRANSPORT | release | [congestion signals](#detail-congestion-signals) |
| 6.2 | XFRM/IPsec packet offload | DRIVER FRAMEWORK | SECURITY | anchor: Part VII | [security](#detail-security) |
| 6.3 | YNL / YAML Netlink tooling | CONTROL PLANE | — | release | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 6.3 | IPv4 BIG TCP | PERFORMANCE | — | release | [packet aggregation](#detail-aggregation) |
| 6.4 | BPF netfilter programs (`BPF_PROG_TYPE_NETFILTER`) | PROGRAMMABILITY | — | release | [Netfilter](#detail-netfilter) |
| 6.6 | AF_XDP multi-buffer | MEMORY / PROGRAMMABILITY | — | release | [XDP / AF_XDP](#detail-xdp-afxdp) |
| 6.6 | TCX / bpf_mprog | PROGRAMMABILITY | — | release | [BPF](#detail-bpf) |
| 6.7 | netkit | PROGRAMMABILITY | VIRTUAL / OVERLAY | anchor: Part VII | [virtual networking](#detail-virtual) |
| 6.7 | TCP-AO | — | SECURITY / TRANSPORT | release | [transport](#detail-transport) |
| 6.8 | Rust phylib / Asix reference PHY | DRIVER FRAMEWORK | — | anchor: Part VII | [modern driver contracts](#detail-driver-modern) |
| 6.8 | queue/NAPI netdev-genl visibility | DRIVER FRAMEWORK / OBSERVABILITY | — | anchor: Part VII | [netdev-genl / queue](#detail-netdev-genl) |
| 6.8 | page_pool identity / Netlink introspection | OBSERVABILITY / MEMORY | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 6.9 | netmem abstraction | MEMORY | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 6.11 | virtio-net AF_XDP RX zero-copy | MEMORY | VIRTUAL / OVERLAY | release | [virtual networking](#detail-virtual) |
| 6.12 | Device Memory TCP RX | MEMORY | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 6.13 | per-netns RTNL infrastructure milestone | CONTROL PLANE | — | anchor: Part VII | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 6.15 | io_uring ZCRX | MEMORY | — | anchor: Part VII | [io_uring](#detail-io-uring) |
| 6.15 | further RTNL breakup | CONTROL PLANE | — | series + release | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 6.15 | page_pool custom memory-provider hooks | MEMORY / DRIVER FRAMEWORK | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 6.16 | Device Memory TCP TX | MEMORY | — | anchor: Part VII | [packet memory](#detail-packet-memory) |
| 6.16 | BPF qdisc | PROGRAMMABILITY | — | series + release | [BPF](#detail-bpf) |
| 6.18 | AccECN core | — | TRANSPORT | series + release | [congestion signals](#detail-congestion-signals) |
| 6.18 | UDP RX evolution | PERFORMANCE | — | series + release | [transport](#detail-transport) |
| 6.19 | `dev_queue_xmit()` llist TX scheduling | PERFORMANCE | — | release | [TX shared-qdisc scalability](#detail-tx-llist) |
| 6.19 | threaded-NAPI kthread busy-poll extension | DRIVER FRAMEWORK | — | release | [threaded busy-poll](#detail-threaded-busypoll) |
| 6.19 | WireGuard YNL-described Netlink | CONTROL PLANE | SECURITY | release | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 7.0 | cake_mq | PERFORMANCE | — | release | [AQM / scheduling](#detail-aqm-pacing) |
| 7.0 | IPv6 BIG TCP without synthetic HBH jumbo header | PERFORMANCE | — | release | [packet aggregation](#detail-aggregation) |
| 7.0 | AccECN enablement | — | TRANSPORT | release | [congestion signals](#detail-congestion-signals) |
| 7.0 | large RX buffers for memory providers / io_uring ZCRX | MEMORY | — | release | [packet memory](#detail-packet-memory) |
| 7.1 | RX HW queue leasing | MEMORY / DRIVER FRAMEWORK | — | anchor: Part VII | [netdev-genl / queue](#detail-netdev-genl) |
| 7.1 | dedicated qdisc-drop tracepoint | OBSERVABILITY | — | release | [observability](#detail-observability) |
| 7.3-rc | BIG TCP over VXLAN/GENEVE | PERFORMANCE | VIRTUAL / OVERLAY | mainline; final pending | [packet aggregation](#detail-aggregation) |
| 7.3-rc | RTNL-less FIB-rule updates | CONTROL PLANE | — | mainline; final pending | [routing / Netlink](#detail-routing-netlink-rtnl) |
| 7.3-rc | devmem buffers \>PAGE_SIZE | MEMORY | — | mainline; final pending | [devmem buffer geometry](#detail-devmem-large) |
| 7.3-rc | per-netns netdev-unregistration infrastructure | CONTROL PLANE | — | mainline; final pending | [per-netns unregister](#detail-netdev-unreg) |


## Part VI から Part VII へ — chronology から provenance へ

Part VI は「いつ」を正規化し、 Part VII はその attribution を再監査できる exact anchor を保持する。


## Coverage rule

Part VI は release chronology の正本である。各 milestone の `Architecture detail` は mechanism の説明先を示し、`Verification status` は release/anchor evidence の強さを示す。detail link の存在は exact commit の確定を意味しない。重複 milestone は置かず、同一機能の origin / integration / generation を分ける必要がある場合だけ別行とする。

# Part VII — 正規 provenance ledger

**Evidence model:** Part VII の SHA は feature series の「代表 anchor」であり、anchor の存在だけで series 全体を証明しない。 各項目は **SHA identity / feature correspondence / release containment** を別々に監査する。

**Evidence status:** `mainline; final pending` は authoritative な pull/merge evidence により Linus mainline への merge を確認済みだが、final release tag が未公開の状態を示す。 Part VI の Verification status と一致させる。

## Exact mainline anchor inventory

ここに示すのは feature series の全 commit ではなく、再監査可能な代表 anchor である。`Anchor type` は次の意味で使う。

- `origin` — object / mechanism の最初の mainline anchor
- `enablement` — feature を実際に利用可能にした代表 commit
- `integration` — subsystem 間を接続した代表 commit
- `merge` — multi-commit series / generation を Linus mainline に統合した canonical merge

巨大な series を恣意的な 1 patch で代表させるより、pull/merge message が feature generation を明示する場合は `merge` を優先する。特に per-netns RTNL のような複数 release にまたがる migration では、`merge` は「完成 commit」ではなく **その generation の開始/ 統合点**を意味する。本文の台帳では representative SHA と first final release に絞り、first-containing rc tag の機械監査は作業ログへ分離する。

| Item | Anchor type | Exact mainline anchor | Exact subject | Architectural role | First final release |
|---|---|---|---|---|---|
| DQL | origin | `75957ba36c05b979701e9ec64b37819adc12f830` | `dql: Dynamic queue limits` | dynamic queue-limit accounting primitive | v3.3 |
| CoDel | origin | `76e3cc126bb223013a6b9a0e2a51238d1ef2e409` | `codel: Controlled Delay AQM` | CoDel qdisc core anchor | v3.5 |
| SO_REUSEPORT infrastructure | origin | `055dc21a1d1d219608cd4baac7d0683fb2cbbe8a` | `soreuseport: infrastructure` | shared reuseport selection infrastructure | v3.9 |
| nftables core | origin | `96518518cc417bb0a8c80b9fb736202e28acdf96` | `netfilter: add nftables` | nftables core rule engine | v3.13 |
| nftables set API | integration | `20a69341f2d00cd042e81c82289fba8a13c05a25` | `netfilter: nf_tables: add netlink set API` | set-API anchor; not core origin | v3.13 |
| bridge VLAN filtering | origin | `243a2e63f5f47763b802e9dee8dbf1611a1c1322` | `bridge: Add vlan filtering infrastructure` | bridge VLAN filtering infrastructure | v3.9 |
| MPLS routing / AF_MPLS | origin | `0189197f441602acdca3f97750d392a895b778fd` | `mpls: Basic routing support` | MPLS label-based routing / AF_MPLS | v4.1 |
| Flower classifier | origin | `77b9900ef53ae047e36a37d13a2aa33bb2d60641` | `tc: introduce Flower classifier` | initial `cls_flower` classifier | v4.2 |
| VRF device | origin | `193125dbd8eb292d88feb201f030889b488b0a02` | `net: Introduce VRF device driver` | VRF device / routing-domain separation | v4.3 |
| TC ct action | integration | `b57dc7c13ea90e09ae15f821d2583fa0231b4935` | `net/sched: Introduce action ct` | conntrack state/metadata as TC action | v5.3 |
| devlink | origin | `bfcd3a46617209454cfc0947ab093e37fd1e84ef` | `Introduce devlink infrastructure` | driver/device management infrastructure | v4.6 |
| BBR | origin | `0f8782ea14974ce992618b55f0c041ef43ed0b78` | `tcp_bbr: add BBR congestion control` | initial BBR mainline anchor | v4.9 |
| phylink | origin | `9525ae83959b60c6061fe2f2caabdc8f69a48bc6` | `phylink: add phylink infrastructure` | MAC/PCS/PHY coordination framework | v4.14 |
| netdevsim | origin | `83c9e13aa39aed5cf9a2f8dd69770b7c35ba1281` | `netdevsim: add software driver for testing offloads` | hardware-independent offload test device | v4.16 |
| SK_MSG | enablement | `4f738adba30a7cfc006f605707e7aee847ffefa0` | `bpf: create tcp_bpf_ulp allowing BPF to monitor socket TX/RX data` | socket-message verdict / `BPF_PROG_TYPE_SK_MSG` | v4.17 |
| TCP MSG_ZEROCOPY | enablement | `f214f915e7db99091f1312c48b30928c1e0c90b7` | `tcp: enable MSG_ZEROCOPY` | socket TX zero-copy API enablement | v4.14 |
| page_pool origin | origin | `ff7d6b27f894f1469dc51ccb828b7363ccd9799f` | `page_pool: refurbish version of page_pool code` | page_pool core origin anchor | v4.18 |
| page_pool/XDP integration | integration | `60bbf7eeef10dc647430646d7fe5e3d8d132dbec` | `mlx5: use page_pool for xdp_return_frame call` | mlx5 page_pool/XDP integration | v4.18 |
| ethtool Generic Netlink | origin | `2b4a8990b7df55875745a80a609a1ceaaf51f322` | `ethtool: introduce ethtool netlink interface` | Netlink-based ethtool control plane | v5.6 |
| SK_LOOKUP | origin | `e9ddbb7707ff5891616240026062b8c1e29864ca` | `bpf: Introduce SK_LOOKUP program type with a dedicated attach point` | dedicated SK_LOOKUP program type / attach point | v5.9 |
| auxiliary bus | origin | `7de3697e9cbd4bd3d62bafa249d57990e1b8f294` | `Add auxiliary bus support` | common auxiliary-device bus | v5.11 |
| IPv6 BIG TCP / GRO | enablement | `0fe79f28bfaf73b66b7b1562d2468f94aa03bd12` | `net: allow gro_max_size to exceed 65536` | allow `gro_max_size` > 65536 | v5.19 |
| IPv6 BIG TCP / GSO | enablement | `7c4e983c4f3cf94fcd879730c6caa877e0768a4d` | `net: allow gso_max_size to exceed 65536` | allow `gso_max_size` > 65536 | v5.19 |
| XFRM/IPsec crypto offload API | origin | `d77e38e612a017480157fe6d2c1422f42cb5b7e3` | `xfrm: Add an IPsec hardware offloading API` | introduces `xfrmdev_ops` for SA/ESP hardware crypto offload | v4.12 |
| kTLS TX / kernel TLS | origin | `3c4d7559159bfe1e3b94df3a657b2cda3a34e218` | `tls: kernel TLS support` | TLS ULP + software TX record datapath | v4.13 |
| kTLS RX | enablement | `c46234ebb4d1eee5e09819f49169e51cfc6eb909` | `tls: RX path for ktls` | `TLS_RX`, recvmsg/splice/poll | v4.17 |
| TLS device offload TX | integration | `e8f69799810c32dd40c6724d829eccc70baad07f` | `net/tls: Add generic NIC offload infrastructure` | same kTLS API/state with NIC TX crypto | v4.18 |
| TLS device offload RX | integration | `4799ac81e52a72a6404827bf2738337bb581a174` | `tls: Add rx inline crypto offload` | completes generic RX device-offload infrastructure | v4.19 |
| WireGuard | origin | `e7096c131e5161fa3b8e52a650d7719d2857adfd` | `net: WireGuard secure network tunnel` | L3 secure tunnel as normal netdevice | v5.6 |
| XFRM packet offload | integration | `d14f28b8c1de668bab863bf5892a49c824cb110d` | `xfrm: add new packet offload flag` | packet-level IPsec offload; policy/SA synchronization with NIC | v6.2 |
| RoCEv2 generation merge | merge | `048ccca8c1c8f583deec3367d7df521bb1f542ae` | `Merge tag 'for-linus' of git://git.kernel.org/pub/scm/linux/kernel/git/dledford/rdma` | RDMA 4.5 pull containing core, mlx4 and mlx5 RoCEv2 support | v4.5 |
| RDMA On-Demand Paging core | enablement | `8ada2c1c0c1d75a60723cd2ca7d49c594a146af6` | `IB/core: Add support for on demand paging regions` | separates MR registration from eager/permanent page pinning and establishes ODP memory lifecycle | v3.19 |
| Soft-RoCE / RXE | origin | `8700e3e7c4857d28ebaa824509934556da0b3e76` | `Soft RoCE driver` | software implementation of RoCE/RDMA verbs semantics over the Linux networking stack | v4.8 |
| RDMA dma-buf memory region | integration | `368c0159d492d7fbdb5791b40c9263ec4e97a10f` | `RDMA/umem: Support importing dma-buf as user memory region` | imports dma-buf-backed memory into RDMA umem/MR lifecycle for device/peer memory | v5.12 |
| BTF origin | origin | `69b693f0aefa0ed521e8bd02260523b5ae446ad7` | `bpf: btf: Introduce BPF Type Format (BTF)` | typed metadata for BPF program/map | v4.18 |
| Net DIM common-library integration | integration | `4f75da3666c0c572967729a2401ac650be5581b6` | `linux/dim: Move implementation to .c files` | driver-local/header logic becomes common `lib/dim` implementation | v5.3 |
| vDPA bus | origin | `961e9c84077f6c8579d7a628cbe94a675cb67ae4` | `vDPA: introduce vDPA bus` | common virtio datapath / vendor-control abstraction | v5.7 |
| netmem abstraction | origin | `18ddbf5cf0e7553fd05c3e1a02d740514ee3f0a6` | `net: introduce abstraction for network memory` | `netmem_ref` decouples network memory references from `struct page` | v6.9 |
| page_pool userspace identity | enablement | `f17c69649c698e4df3cfe0010b7bbf142dec3e40` | `net: page_pool: id the page pools` | creates stable IDs for uAPI references | v6.8 |
| page_pool netlink GET | enablement | `950ab53b77ab829defeb22bc98d40a5e926ae018` | `net: page_pool: implement GET in the netlink API` | exposes pool identity / ifindex / NAPI ID | v6.8 |
| page_pool netlink statistics | integration | `d49010adae737638447369a4eff8f1aab736b076` | `net: page_pool: expose page pool stats via netlink` | driver-independent observability | v6.8 |
| page_pool custom memory providers | integration | `57afb483015768903029c8336ee287f4b03c1235` | `net: page_pool: create hooks for custom memory providers` | shared allocator hook for devmem TCP and io_uring ZCRX | v6.15 |
| Device Memory TCP RX | merge | `9410645520e9b820069761f3450ef6661418e279` | `Merge tag 'net-next-6.12' of git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next` | Device Memory TCP RX generation integration merge | v6.12 |
| per-netns RTNL start | merge | `fcc79e1714e8c2b8e216dc3149812edd37884eef` | `Merge tag 'net-next-6.13' of git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next` | initial per-netns RTNL migration wave merge | v6.13 |
| io_uring ZCRX | merge | `71f0dd5a3293d75d26d405ffbaedfdda4836af32` | `Merge branch 'io_uring-zero-copy-rx'` | io_uring zero-copy receive generation merge | v6.15 |
| Device Memory TCP TX | merge | `1b98f357dadd6ea613a435fbaef1a5dd7b35fd21` | `Merge tag 'net-next-6.16' of git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next` | Device Memory TCP TX generation integration merge | v6.16 |
| netkit | origin | `35dfaad7188cdc043fde31709c796f5a692ba2bd` | `netkit, bpf: Add bpf programmable net device` | netkit core anchor | v6.7 |
| Rust PHY abstractions | integration | `f20fd5449ada3872dcd67aca397f0e27ca2e8ad6` | `rust: core abstractions for network PHY drivers` | Rust core abstractions for network PHY drivers | v6.8 |
| netdev-genl queue object | integration | `bc877956272f0521fef107838555817112a450dc` | `netdev-genl: spec: Extend netdev netlink spec in YAML for queue` | YAML spec for queue object | v6.8 |
| RX HW queue leasing / queue-create | enablement | `7789c6bb76acf21539c2c74b0cc869bb57de99e6` | `net: Add queue-create operation` | virtual RX queue may lease a physical RX queue | v7.1 |
| RX HW queue leasing generation | merge | `91a4855d6c03e770e42f17c798a36a3c46e63de2` | `Merge tag 'net-next-7.1' of git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next` | RX queue leasing generation merge | v7.1 |
| net-next 7.3 merge | merge | `91ec2035134982b98fab0609a9fd8480e8217dc1` | `Merge tag 'net-next-7.3' of git://git.kernel.org/pub/scm/linux/kernel/git/netdev/net-next` | net-next 7.3 networking generation merge; final release pending | 7.3 final pending |

### Upstream anchor audit status

Part VII の 52 anchor は upstream Git object と照合済みである。52/52 で SHA と exact commit subject を確認し、final release が確定している 51 anchor は **記載 final tag が commit を含み、直前 final tag が含まない**ことを確認したため、first final release も51/51で再検証済みである。`net-next-7.3` merge (`91ec2035...`) は mainline merge と exact subject まで確認済みだが、この版の基準時点では final release 未確定なので `7.3 final pending` を維持する。

このため台帳では、機械監査できる **Exact subject** と、文書上の意味づけである **Architectural role** を別列にする。前者は Git object の属性、後者は本書の architecture synthesis における解釈であり、混同しない。

### Exact-anchor interpretation notes

RDMA の canonical rows は RDMA subsystem の起源を示すものではない。RDMA は v3.0 より前から Linux に存在する。4.5 RoCEv2 は core + mlx4 + mlx5 の複数 commit から成るため、 Part VII では `048ccca8...` の RDMA pull merge を generation-level representative anchor とする。対象期間内の architecture boundary として、3.19 ODP を memory-registration lifecycle の一般化、4.5 RoCEv2 を Ethernet/IP control plane と verbs datapath の接続、4.8 RXE を execution implementation の分離、5.12 dma-buf MR を device/peer memory integration の代表点として採用する。

`netmem` と memory providers は別の boundary である。6.9 の `netmem_ref` は **memory type を `struct page` から抽象化する型/API contract**、6.15 の memory-provider hooks は **page_pool が custom allocator/provider を選択する lifecycle/integration contract** である。Device Memory TCP RX は6.12 に専用経路で先行し、6.15 で devmem TCP と io_uring ZCRX が共通 provider interface へ整理された。

page_pool introspection は単一 commit へ縮約しない。`f17c69649...` が pool ID、`950ab53b...` が Netlink GET、`d49010ad...` が stats exposure を導入するため、この 3 段階を6.8 generation の representative anchors とする。

per-netns RTNL は6.13 で完成した機能ではない。`fcc79e17...` は複数 release にまたがる migration の開始 wave をまとめた merge anchor である。RX HW queue leasing も単一 commit ではなく、`7789c6bb...` を queue-create/lease API の enablement、`91a4855d...` を7.1 generation の merge anchor として扱う。


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

### RDMA / accelerator-memory sources

**API / object model**

- man7 `rdma_cm(7)` — socket-like connection management adapted to QP semantics: <https://man7.org/linux/man-pages/man7/rdma_cm.7.html>
- man7 `ibv_create_qp(3)` — QP, SQ/RQ, CQ association and queue capacity: <https://man7.org/linux/man-pages/man3/ibv_create_qp.3.html>
- man7 `ibv_create_cq(3)` — completion-queue object: <https://man7.org/linux/man-pages/man3/ibv_create_cq.3.html>
- man7 `ibv_post_send(3)` — work-request posting and completion lifetime: <https://man7.org/linux/man-pages/man3/ibv_post_send.3.html>
- man7 `ibv_reg_mr(3)` — MR, access flags, ODP and `ibv_reg_dmabuf_mr()`: <https://man7.org/linux/man-pages/man3/ibv_reg_mr.3.html>
- man7 `ibv_advise_mr(3)` — ODP prefetch/advice semantics: <https://man7.org/linux/man-pages/man3/ibv_advise_mr.3.html>
- Linux kernel RDMA/InfiniBand driver API — `ib_umem`, ODP fault/map/invalidation machinery: <https://kernel.org/doc/html/latest/driver-api/infiniband.html>

**RoCE / execution placement**

- NetDev 1.1 — Ethernet HA/LAG and RDMA; useful architecture summary of RDMA-CM control path vs verbs data path and RoCEv2 4.5: <https://www.netdevconf.info/1.1/talk-hw-high-availabity-and-link-aggregation-ethernet-switch-and-nic-rdma-using-linux-bondingteam.html>
- Linux 4.5 RDMA pull merge containing core + mlx4 + mlx5 RoCEv2 support: <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=048ccca8c1c8f583deec3367d7df521bb1f542ae>
- NetDev 0x16 — RDMA programming tutorial (QP/CQ/MR, kernel bypass, transport offload, accelerator memory): <https://netdevconf.info/0x16/sessions/tutorial/rdma-programming-tutorial.html>
- LPC 2017 RDMA microconference — system boot, OFA direction, paravirtual RDMA and subsystem integration context: <https://blog.linuxplumbersconf.org/2017/ocw/events/LPC2017/schedule.html>

**Memory / peer-to-peer evolution**

- ODP exact anchor: <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8ada2c1c0c1d75a60723cd2ca7d49c594a146af6>
- Soft-RoCE exact anchor: <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8700e3e7c4857d28ebaa824509934556da0b3e76>
- RDMA dma-buf exact anchor: <https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=368c0159d492d7fbdb5791b40c9263ec4e97a10f>
- LWN / linux-rdma — RDMA dma-buf design series; GPU VRAM, dynamic attach and ODP motivation: <https://lwn.net/Articles/839314/>
- Linux dma-buf documentation — exporter/importer, attachment, P2P and synchronization/lifetime contract: <https://docs.kernel.org/driver-api/dma-buf.html>
- Linux PCI P2PDMA documentation — RNIC as P2P client and direct DMA to peer-device memory: <https://docs.kernel.org/driver-api/pci/p2pdma.html>

**AI / operations / current direction**

- NetDev 0x19 — AI Networks: RoCEv2 and the role of netdev: <https://netdevconf.info/0x19/sessions/workshop/ai-networks-rocev2-and-the-role-of-netdev.html>
- NetDev 0x19 — Congestion-control in AI/ML networks at datacenter scale: <https://netdevconf.info/0x19/sessions/talk/congestion-control-in-aiml-networks-at-datacenter-scale.html>
- NetDev 0x17 — io_uring ZC RX comparison with RDMA/kernel-bypass trade-offs: <https://netdevconf.info/0x17/sessions/talk/fast-zc-rx-data-plane-using-io-uring.html>
- LPC 2025 — BPF-based production observability for RDMA driver/resource failures in large-model training: <https://lpc.events/event/19/contributions/2177/>
- 2026 RDMA dma-buf export series (non-canonical / proposed): <https://lwn.net/Articles/1056826/>
- 2026 Soft-RoCE dma-buf series (non-canonical / proposed): <https://lwn.net/Articles/1064824/>
- 2026 generic RDMA UMEM buffer-descriptor series (non-canonical / proposed): <https://lwn.net/Articles/1073183/>

The searched Kernel Recipes archive did not yield a comparably direct RDMA architecture talk; therefore Kernel Recipes material is not used merely to fill the bibliography. The useful conference evidence for this topic came primarily from NetDev and Linux Plumbers.

## Kernel documentation / primary-source index

| Source topic | Canonical destination | What it supports |
|:-------------------------------------|:--------------------------------------------------------|:----------------------------------------------------------------|
| NAPI / `SO_BUSY_POLL` | Part VI 3.11; Part IV | busy polling and NAPI execution model |
| BPF DEVMAP | Part VI 4.14; Part III XDP / AF_XDP | XDP redirect via devmap |
| BPF CPUMAP | Part VI 4.15; Part III XDP / AF_XDP | XDP redirect to remote CPU |
| nftables core / architecture | Part VI 3.13; Part III netfilter / nftables / conntrack | VM/expression model, Netlink control, transactional ruleset evolution |
| nftables ingress / bridge | Part III netfilter / nftables / conntrack | earlier hook placement and L2/L3 policy integration |
| nftables flowtable | Part VI 4.16; Part III netfilter / nftables / conntrack | policy-selected software fast path; later HW-offload evolution |
| TCX / `bpf_mprog` | Part VI 6.6; Part III BPF | link-based TC attachment and multi-program ordering |
| Device Memory TCP / memory providers | Part VI 6.12/6.16; Part III packet memory | RX/TX device-memory zero-copy evolution |
| io_uring ZCRX | Part VI 6.15; Part III io_uring networking | queue-bound zero-copy receive |
| `net-next-7.1` | Part VI 7.1 | RX HW queue leasing |
| `net-next-7.3` | Part VI 7.3-rc | tunnel BIG TCP, RTNL-less FIB rules, \>PAGE_SIZE devmem buffers |

## Exact anchor index

Exact SHA の正本は Part VII に置く。Appendix では同じ commit list を複製しない。

## Conference provenance index

Conference talk は設計意図や当時の problem statement を補足する資料として扱い、 release attribution には使用しない。

| Conference lineage | Canonical destination |
|:----------------------------------------|:----------------------------|
| Netdev: XDP / AF_XDP / TC / BPF | Part III BPF / XDP |
| Netdev: BIG TCP | Part III packet aggregation |
| Netdev: Device Memory TCP / zero-copy | Part III packet memory |
| Netdev 0x19: Diagnosing Page Pool Leaks | Part IV page_pool |
| Netdev: queue/NAPI/netdev-genl | Part IV driver framework |
| Netdev: MPTCP / TCP state-of-the-union | Part III transport |
| Netdev 0.1/1.1: nftables architecture / ingress / bridge | Part III netfilter |
| Netdev: Netfilter flowtable / TC conntrack offload | Part III netfilter / Routing-TC-offload |
| Kernel Recipes: nftables Why and how / What's new | Part III netfilter |
| Kernel Recipes: XDP / BPF / io_uring | Parts III–IV |
| OVS/OVN Conf: Retis | Part V case-study note |
| LPC / FOSDEM: pwru and tracing | Part V case-study note |

### Additional architecture / implementation conference references

- **Netdev 0x13 — devlink health reporting and recovery system:** <https://netdevconf.info/wiki/doku.php?id=0x13:reports:d2t3t07-devlink-health-reporting-and-recovery-system>
- **Kernel Recipes 2024 — Efficient zero-copy networking using io_uring:** <https://kernel-recipes.org/en/2024/schedule/efficient-zero-copy-networking-using-io_uring/>
- **Kernel Recipes 2024 — live blog (io_uring zero-copy / netmem / queue API context):** <https://kernel-recipes.org/en/2024/2024/09/23/live-blog-day-1-afternoon/>
- **Kernel Recipes 2024 — Interfacing Kernel C APIs from Rust:** <https://kernel-recipes.org/en/2024/schedule/interfacing-kernel-c-apis-from-rust/>

これらは design/context provenance であり、 Part VI の release attribution には使用しない。

## Retis / pwru source index

Retis と pwru の位置づけと機構説明は Part V を正本とする。ここでは関連資料への索引だけを保持する。

```text
pwru  = broad kernel-function packet trajectory
Retis = networking-event-centric semantic enrichment / correlation
```

これらは BTF、eBPF tracing、drop reason、timestamp、OVS/OVN metadata などの kernel primitive を利用する **consumer/tooling examples** であり、独立した kernel evolution axis ではない。

- pwru — LPC 2024: <https://lpc.events/event/18/contributions/1942/>
- Retis — project repository: <https://github.com/retis-org/retis>
- Retis の architecture / usage context — Red Hat Developer の解説: <https://developers.redhat.com/articles/2024/01/04/retis-new-tool-network-observability-linux>
 （補助資料として扱い、mainline release boundary の正本は Part VI / VII とする。）


## Provenance policy

```text
conference / RFC
      ↓ design intent
patch series
      ↓ landing evidence
subsystem pull / Linus mainline
      ↓ canonical integration
released tag
```


