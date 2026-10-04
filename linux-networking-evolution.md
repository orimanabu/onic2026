# Linux Networking Evolution --- Linux v3.0 から 7.x まで

> **Clean canonical edition.** Part II is the only normative release
> chronology. Part III explains feature lineages, Part IV the
> driver-framework axis, Part V observability, Part VI synthesis, and
> Part VII unresolved provenance. Appendix material is supporting
> evidence, not an alternate release map.

**調査基準日:** 2026-10-02\
**構成改訂:** 2026-10-04（r13: relation/evidence semantics audit）

この文書は、Linux networking の変化を「調査した順」ではなく、 **kernel
networking がどのように進化したかを読む順序**に再構成した版である。

**対象範囲:** wired / host networking を中心に扱う。RDMA、Wi-Fi 全般、QUIC や
security protocol の網羅的な歴史は対象外とし、mac80211 は queue management
（airtime / AQL）の観点から選択的に扱う。

## この文書の読み方

> **Canonicalization rule:** release attribution と evidence grade を決定する唯一の正本は
> **Part II** とする。Part III--VI は Part II の release 番号を説明・lineage のために
> 参照してよいが、独立した attribution や grade を定義しない。Part VII は exact
> commit / unresolved boundary、Appendix は supporting evidence を保持する。

本文は次の流れで構成する。

``` text
Part I    foundations
          Linux v3.xを中心に、modern networkingの前提を形成

Part II   canonical release chronology
          v3.0 → current mainline / release-candidate state

Part III  thematic feature lineages
          aggregation / memory / XDP / BPF /
          virtual networking / transport / control plane

Part IV   Network Device Driver Framework Evolution
          BQL / switchdev / devlink / phylink / page_pool /
          netdev-genl / Rust driver abstractions

Part V    observability / explainability
          eBPF + BTF → skb_drop_reason → queue/NAPI/page_pool observability

Part VI   synthesis
          release間比較と全体architecture map

Part VII  canonical provenance ledger
          未解決のrelease/commit attribution

Appendix  evidence catalog
          LWN / upstream commits / conference provenance
```

重要な方針は、release chronology と feature lineage を分離すること。
まず年代順に「何がいつ形成されたか」を読み、その後で同じ技術を
長期lineageとして横断的に追う。

------------------------------------------------------------------------

## 編集上の用語と規約

本書では以下の用語を一貫して次の意味で使用する。

| Term | Meaning |
|---|---|
| **design / RFC** | proposal or architecture under discussion; no merge is implied |
| **series** | a posted patch series; `final series` means the latest merge-near revision identified by this research |
| **landing** | acceptance into the relevant subsystem tree or `net-next`; this is not automatically a released kernel |
| **mainline anchor** | a verified commit in Linus's mainline history that anchors part of a feature |
| **release** | a feature is present in a released mainline kernel version |
| **generation** | a release-era milestone spanning multiple commits or incremental follow-ups; not necessarily a single origin commit |
| **development** | accepted, merged into a development tree, or posted for a future cycle, but not treated here as a released baseline |

A mainline anchor can be a core/origin commit, an integration commit, or
a protocol-specific enablement commit. The text names the anchor type
when that distinction matters.

### Provenance の段階

``` text
design / RFC
      ↓
patch series
      ↓
subsystem-tree / net-next landing
      ↓
mainline commit
      ↓
released kernel
```

The arrows describe the usual path, not a guarantee that every project
passes through each stage in exactly this form.

# Part I --- Foundations: Linux v3.x

## Linux v3.x --- scalability, virtualization, programmability の誕生

``` text
3.0   setns() / namespace FD
 │
3.3   Byte Queue Limits (BQL)
 │
3.3   team driver / net_prio cgroup / TCP buffer cgroup
 │
3.5   CoDel / fq_codel
 │
3.6   TCP Small Queues / TCP Fast Open client /
 │     IPv4 route-cache removal / netfilter namespace work
 │
3.7   VXLAN / TCP Fast Open server / IPv6 NAT
 │
3.9   SO_REUSEPORT / conntrack labels / VM sockets
 │
3.11  SO_BUSY_POLL / low-latency busy polling
 │
3.12  TCP pacing + FQ-era TCP scheduling
 │
3.13  nftables
 │
3.14  TCP autocorking
 │
3.15–3.17
 │     classic BPF → extended/generalized BPF architecture
 │
3.18  bpf() syscall / maps / verifier
 │     DCTCP / Geneve / Foo-over-UDP
 │
3.19  eBPF socket attachment / ipvlan
 │
4.x   XDP / cgroup BPF / kTLS / AF_XDP ...
```

------------------------------------------------------------------------

### 1.1 v3.0 --- namespace control becomes practical

v3.0でnamespace file descriptorと`setns()`がmainline化した。

``` text
process
  ↓ open namespace FD
setns()
  ↓
another namespace
```

network namespace自体は以前から存在したが、namespaceをFDとして扱い
processを既存namespaceへ移動できることは、後のcontainer runtimeにとって
非常に重要なcontrol primitiveである。

この資料では:

``` text
network namespace
      ↓
namespace FD / setns()
      ↓
container runtime
      ↓
CNI / Kubernetes networking
```

というcontainer-networking lineageの初期milestoneとして扱う。

------------------------------------------------------------------------

### 1.2 2011--2012 DQL/BQL development → Linux v3.3 mainline

BQL (Byte Queue Limits) はNIC TX queueへ過剰なdataを押し込まないよう、
driver queueの適正なbyte量を動的に制御する。

``` text
before:

TCP/qdisc → huge NIC TX ring backlog
                 ↓
             latency

BQL:

TCP/qdisc → bounded NIC queue → NIC
                ↑
         completion feedback
```

これは非常に重要で、後の:

``` text
BQL
 ↓
CoDel / fq_codel
 ↓
TCP Small Queues
 ↓
FQ / pacing
 ↓
BBR / EDT
```

というlatency-aware Linux TX architectureの最初期の柱になる。

------------------------------------------------------------------------

### 1.3 v3.3 --- networking cgroups and interface aggregation

v3.3では:

-   `team` network driver
-   `net_prio` cgroup controller
-   TCP buffer memory cgroup controller

が入った。

``` text
process/cgroup
     ↓
network priority / TCP memory accounting
```

という、workload identityとnetwork resource controlを結びつける流れが
すでに始まっている。

`team`はbondingとは別のuserspace-controlled link aggregation
modelを提供した。

------------------------------------------------------------------------

### 1.4 v3.5 --- CoDel / fq_codel

bufferbloat対策はdevice queueだけでは不十分。

``` text
BQL
 = NIC driver queue

CoDel/fq_codel
 = qdisc queue
```

CoDelはqueue delayを観測してAQMを行い、fq_codelはflow queueingとCoDelを
組み合わせる。

``` text
flows
 ├─ flow A queue ─┐
 ├─ flow B queue ─┼─ fair scheduling + CoDel → NIC
 └─ flow C queue ─┘
```

後のLinux/router/container hostにおけるlow-latency queue managementの
重要な基礎。

------------------------------------------------------------------------

### 1.5 v3.6 --- TCP Small Queues, TFO, route-cache removal

v3.6はv3.x networkingの大きなmilestone。

### TCP Small Queues (TSQ)

BQLがdevice queueを制御するのに対してTSQは**per TCP socket**で
qdisc/deviceへ溜められるdata量を抑える。

``` text
TCP socket
   ↓
 TSQ limit
   ↓
 qdisc
   ↓
 BQL
   ↓
 NIC
```

したがってbufferbloat対策は:

``` text
TCP socket : TSQ
qdisc      : CoDel/fq_codel
driver/NIC : BQL
```

という複数layerへ広がった。

### TCP Fast Open client

TCP handshake中にapplication dataを送るTFO client supportが入った。
server supportは次のv3.7。

### IPv4 route cache removal

従来のIPv4 route cacheはtraffic
pattern依存のperformanceとDoS問題を持ち、 v3.6で長年のroute-cache
removal workが完了した。

``` text
route cache
   ↓ removed
FIB lookup + scalable caching strategies
```

現在のLinux routing scalabilityを理解する重要なarchitecture change。

### netfilter namespace work

多数のnetfilter modulesがnetwork namespaceに対応し、 container
isolationへの準備が進んだ。

------------------------------------------------------------------------

### 1.6 v3.7 --- VXLAN

v3.7でVXLANがmainline化。

``` text
tenant L2 frame
      ↓
VXLAN encapsulation
      ↓
UDP/IP underlay
      ↓
remote VTEP
```

現在の:

``` text
OpenStack
OVS/OVN
Kubernetes overlays
cloud virtual networking
```

につながる非常に重要なmilestone。

同時にTCP Fast Open server sideとIPv6 NATも入った。

------------------------------------------------------------------------

### 1.7 v3.9 --- multicore server socket scaling

`SO_REUSEPORT` がTCP/UDPへ導入された。

``` text
port :443
  ├─ worker/socket CPU0
  ├─ worker/socket CPU1
  ├─ worker/socket CPU2
  └─ worker/socket CPU3
```

single acceptor/dispatcher bottleneckを減らし、 multicore network server
scalabilityを改善するAPI。

後の:

``` text
SO_REUSEPORT
   ↓
reuseport BPF selection
   ↓
programmable socket dispatch
```

につながる。

同releaseにはconntrack labelsやVM socketsも含まれる。

------------------------------------------------------------------------

### 1.8 v3.12 --- TCP pacing / FQ lineage

v3.x前半の:

``` text
Linux TX latency / queue management

TCP socket layer:
  TSQ · TCP pacing · BBR
          │  packet admission / pacing / congestion control
          ▼
qdisc / scheduling layer:
  fq_codel · sch_fq · EDT · CAKE
          │  queueing / scheduling / AQM
          ▼
driver / NIC queue layer:
  BQL · TX queue management

※ 縦方向は packet が通るレイヤー配置を示す。
  「上の機構が下の機構を生み出した」という実装上の派生関係を示す矢印ではない。
```

これらは一本道の依存関係ではない。BQL は driver TX queue、fq_codel は qdisc、
TSQ は per-socket backlog を主に制御し、異なるレイヤーで同じ「過剰な滞留を避ける」
設計課題に取り組む。pacing/FQ はさらに packet を **いつ送るか** を制御する別の軸である。

BBR は congestion-control algorithm、EDT は time-based scheduling primitive であり、
後者を BBR 専用の後継機構とは扱わない。これらは異なるレイヤーの制御を組み合わせて
TX queue residency と latency を抑える、補完的な機構群として読む。

またTSO auto sizingやTCP autocorkingもこの時代のTSQ/pacing
infrastructureを 利用する。

------------------------------------------------------------------------

### 1.9 v3.13 --- nftables

v3.13でnftablesがmainline化。

従来:

``` text
iptables
ip6tables
arptables
ebtables
```

のようにprotocol/familyごとに重複したframeworkを持つ構造から、

``` text
nftables
   ↓
generic rule representation
   ↓
in-kernel virtual machine
```

へ移行する長期projectのmainline起点。

現在のLinux firewall/control
planeを考える上でv3.x最大級のmilestoneの一つ。

------------------------------------------------------------------------

### 1.10 v3.15--v3.17 --- BPF transformation

classic BPFはsocket packet filterを中心とした小さなVMだった。

この時期、Alexei StarovoitovらのworkによってBPF instruction
set/interpreterが より一般的な64-bit register machineへ再設計される。

``` text
classic BPF
 socket filter VM
      ↓
extended BPF ISA
      ↓
general in-kernel programmable VM
```

この転換がなければ後のXDP、TC BPF、cgroup
BPF、struct_ops、netkitはない。

------------------------------------------------------------------------

### 1.11 v3.18 --- eBPF core API arrives

v3.18で`bpf()` syscallがmainline化。

主要concept:

``` text
bpf()
 ├─ program load
 ├─ verifier
 ├─ maps
 └─ file-descriptor based object model
```

この時点では現在のような多数のattachment pointsはまだ存在しない。

つまりv3.18は:

``` text
"network packet filter language"
        ↓
"kernel programmable subsystem"
```

への決定的なarchitecture change。

### DCTCP

Data Center TCP congestion controlもv3.18で導入された。

ECN markingをdatacenter congestion signalとして積極的に利用する。

``` text
switch queue
   ↓ ECN mark
DCTCP sender
   ↓ estimate congestion fraction
cwnd adjustment
```

後のdatacenter transport/AccECN lineageの重要な前史。

### Geneve and Foo-over-UDP

GeneveとFoo-over-UDPも入り、overlay/tunnel infrastructureが
VXLANだけでなくより一般的に発展した。

------------------------------------------------------------------------

### 1.12 v3.19 --- eBPF meets networking again

v3.18でloadできるようになったeBPF
programを、v3.19ではsocketへattach可能に なった。

``` text
bpf()
 ↓ load
eBPF program
 ↓ SO_ATTACH_BPF
socket
```

ここから:

``` text
socket eBPF
 → TC eBPF
 → XDP
 → cgroup/socket hooks
 → SK_LOOKUP
 → struct_ops
```

へ続く。

### ipvlan

v3.19では`ipvlan`もmainline化。

``` text
physical NIC
   ↓
ipvlan
 ├─ container/netns A
 ├─ container/netns B
 └─ container/netns C
```

macvlanとは異なるmultiplexing modelを提供し、 container
networkingの重要なbuilding blockとなる。

------------------------------------------------------------------------

### 1.13 v3.xを追加したことで見える長期lineage

### Latency / bufferbloat / pacing

``` text
BQL (3.3)
 ↓
CoDel/fq_codel (3.5)
 ↓
TCP Small Queues (3.6)
 ↓
TCP pacing/FQ (3.x)
 ↓
BBR (4.9)
 ↓
time-based TX (4.19)
 ↓
EDT (4.20)
```

これはLinux TX performance史で最も重要な一本のlineage。

### Cloud virtual networking

``` text
netns + setns (3.0)
 ↓
VXLAN (3.7)
 ↓
ipvlan (3.19)
 ↓
VRF/LWT/OVS conntrack (4.3)
 ↓
BPF LWT / SRv6
 ↓
OVS/OVN / Kubernetes / UDN
```

### Programmability

``` text
classic BPF
 ↓
eBPF ISA redesign (3.x)
 ↓
bpf() + maps + verifier (3.18)
 ↓
socket eBPF (3.19)
 ↓
TC eBPF (`cls_bpf` / `act_bpf`, 4.1)
  ↓
TC direct packet access (4.7)
 ↓
XDP (4.8)
 ↓
cgroup/LWT BPF
 ↓
SOCK_OPS / SOCKMAP
 ↓
AF_XDP
 ↓
struct_ops / SK_LOOKUP / netkit
```

### Firewall

``` text
iptables family
 ↓
nftables (3.13)
 ↓
flowtable
 ↓
hardware offload
 ↓
nftables + BPF complementary model
```

### Datacenter transport

``` text
TCP Fast Open (3.6/3.7)
DCTCP (3.18)
       ↓
BBR (4.9)
       ↓
MPTCP / BIG TCP / AccECN
```

------------------------------------------------------------------------

### 1.14 世代モデルの正本

世代区分（3.x / 4.x / 5.x / 6.x--7.x）は Part II の **Dominant-theme
eras** を正本とします。 Part I では、3.x がその後の世代へ渡した
foundation の説明に限定します。

## Part I から Part II へ --- foundations から chronology へ

Part I では architecture の基礎を整理した。Part II では主要 milestone の
release attribution
を正本として固定する。後続の章で詳しく説明しても、この表の release
attribution を上書きしない。

## 以降の章の読み方

``` text
                         Part II
                  canonical chronology
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       Part III       Part IV        Part V
       lineages       drivers     observability
          └──────────────┼──────────────┘
                         ↓
                       Part VI
                       synthesis
                         ↓
                       Part VII
                      provenance
```

# Part II --- 正規 release chronology

この表に release attribution と evidence grade を集約する。

欠落している kernel release
は「重要な変更がなかった」という意味ではなく、本書が選んだ architecture
milestone を割り当てていないことだけを意味します。

RFC / review date、subsystem-tree landing、Linus mainline
merge、released tag は区別します。

``` text
RFC / review
    ≠ subsystem tree / net-next
    ≠ Linus mainline
    ≠ released tag
```

Grade の定義は次のとおりであり、この文書自体にも保持する。

- **A**: released milestone で、release/mainline evidence が十分に確認済み。
- **A-rc**: 現在の未リリース cycle について Linus mainline への merge を確認済み。
- **B**: release/cycle attribution は強いが、exact commit/tag boundary の一部が未確定。
- **C**: development boundary または release attribution が引き続き監査対象。
- **D**: RFC/design evidence のみで、mainline/release fact として扱わない。

C/D は現在の canonical milestone table では使用していないが、将来の監査対象を表すため grade scheme として定義を残す。

**Evidence は3軸で監査する。** Grade は主として release attribution の確度を表し、機能があらゆる環境で成立することや、関連 commit を全件列挙済みであることを保証しない。これとは独立に、
(1) SHA が正しい Git object / subject を指すか、(2) その anchor が主張する機能をどの範囲まで代表するか、
(3) その変更が対象の final release tag に含まれるか、を確認する。したがって「exact SHA がある」だけで
feature series 全体や release attribution が自動的に Grade A になるわけではない。

**Part II ↔ Part VII invariant:** Part II の Evidence / status で `exact anchor` または `anchor retained in Part VII` と主張する場合、Part VII の exact-anchor inventory に対応する完全な40桁SHAが存在しなければならない。Part VII にSHAを保持しない milestone は、Part II では `release-generation`、`release`、`series` などの表現に留める。


| Release | 正規 milestone | Grade | Evidence / status |
|---|---|---|---|
| 3.0 | namespace FD / setns() (`CLONE_NEWNET`) foundation | A | `setns()` and network-namespace reassociation documented since Linux 3.0 |
| 3.3 | DQL/BQL; team; net_prio; TCP memcg | A | release; exact DQL anchor |
| 3.5 | CoDel / fq_codel | A | release; CoDel exact anchor retained; fq_codel release attribution confirmed |
| 3.6 | TSQ; TFO client; IPv4 route-cache removal | A | release-generation; detailed anchors not reproduced in Part VII |
| 3.7 | VXLAN; TFO server; IPv6 NAT | A | release verified |
| 3.9 | TCP/UDP SO_REUSEPORT; conntrack labels; VM sockets | A | release; SO_REUSEPORT infrastructure anchor retained |
| 3.11 | SO_BUSY_POLL / low-latency busy polling | A | release documented |
| 3.12 | sch_fq; sk_pacing_rate-driven pacing; TSO autosizing generation | A | qdisc-based pacing; TCP-internal fallback later |
| 3.13 | nftables | A | release; exact core anchor retained in Part VII |
| 3.14 | TCP autocorking | B | release-generation |
| 3.15 | internal BPF ISA rework toward eBPF/native-JIT-friendly format | A | release-generation |
| 3.18 | bpf() syscall/maps/verifier generation; DCTCP; Geneve/FOU | A | release-generation |
| 3.19 | ipvlan; switchdev origin; SO_ATTACH_BPF | A | release-generation |
| 4.1 | cls_bpf / act_bpf eBPF support; kprobe BPF | B | early TC/tracing eBPF milestone |
| 4.3 | VRF; LWT; OVS conntrack | A | series + release |
| 4.6 | devlink | A | release + exact origin anchor |
| 4.7 | TC BPF direct packet access | A | release milestone |
| 4.8 | XDP | A | initial series + release |
| 4.9 | BBR | A | release; exact anchor retained |
| 4.10 | cgroup BPF; BPF LWT; IPv6 Segment Routing | A | series + release |
| 4.13 | SOCK_OPS; kTLS TX; TCP-internal pacing/fallback generation | A | release |
| 4.14 | phylink; SOCKMAP; TCP MSG_ZEROCOPY; XDP devmap | A | release; exact phylink / TCP MSG_ZEROCOPY anchors retained; devmap documented since 4.14 |
| 4.15 | XDP cpumap | A | release documented |
| 4.16 | netdevsim; Net DIM generation; nftables software flowtable | A | release; DIM exact SHA pending |
| 4.17 | BPF_PROG_TYPE_SK_MSG; sockmap sendmsg/sendfile | B | final-series generation |
| 4.18 | AF_XDP; TCP_ZEROCOPY_RECEIVE; refurbished page_pool/XDP memory return; cgroup UDP sendmsg hooks | A | series + release; page_pool anchor retained |
| 4.19 | SO_TXTIME; CAKE | A | release-generation; detailed anchors not reproduced in Part VII |
| 4.20 | TCP EDT pacing; BPF flow dissector; taprio; rtnetlink strict checking | A | release |
| 5.0 | UDP GRO; UDP MSG_ZEROCOPY | A | release |
| 5.1 | devlink health; BPF spinlocks/DCE; SO_BINDTOIFINDEX; Y2038 timestamps; io_uring substrate; mac80211 airtime accounting/scheduling to TXQs | A | release; airtime commit contained in v5.1-rc1 |
| 5.3 | nexthop objects; DIM generalized into common lib/dim | A | release-generation; nexthop core anchor not reproduced in Part VII |
| 5.5 | mac80211 Airtime Queue Limits (AQL) | B | release-generation |
| 5.6 | MPTCP; WireGuard; BPF struct_ops/TCP CC; ethtool Generic Netlink | A | release; exact ethtool anchor retained |
| 5.9 | BPF_PROG_TYPE_SK_LOOKUP | A | exact anchor retained in Part VII |
| 5.11 | auxiliary bus | A | release + exact origin anchor |
| 5.12 | threaded NAPI | A | release |
| 5.15 | IPv6 IOAM core; MCTP; bridge per-VLAN multicast | A | release |
| 5.17 | kfree_skb_reason / structured drop-reason foundation | A | release |
| 5.18 | XDP multi-buffer / frags generation | B | prerequisite for later AF_XDP multi-buffer |
| 5.19 | IPv6 BIG TCP; drop-reason expansion | A | release; exact BIG TCP anchors retained |
| 6.0 | io_uring IORING_OP_SEND_ZC; multishot receive | A | release-generation |
| 6.2 | TCP PLB; XFRM/IPsec packet offload | A | release; exact XFRM anchor retained |
| 6.3 | YNL/YAML Netlink tooling; IPv4 BIG TCP | A | YNL origin contained in v6.3-rc1; IPv4 BIG TCP release attribution confirmed |
| 6.6 | AF_XDP multi-buffer; TCX / bpf_mprog multi-program attachment | A | release |
| 6.7 | netkit; initial TCP-AO | A | release; netkit anchor retained |
| 6.8 | Rust phylib + Rust Asix reference PHY; queue/NAPI netdev-genl visibility | B | Rust PHY and queue-object anchors exact; broader NAPI/object generation remains composite |
| 6.11 | virtio-net AF_XDP RX zero-copy | A | release-generation; detailed anchors not reproduced in Part VII |
| 6.12 | Device Memory TCP RX | B | final series verified; implementation SHA list not reproduced |
| 6.13 | per-netns RTNL infrastructure/migration milestone | B | milestone, not completion |
| 6.15 | io_uring ZCRX; further RTNL breakup | A | merge/series evidence |
| 6.16 | Device Memory TCP TX; BPF qdisc; DCCP removal | A | release/final-series generation; BPF-qdisc exact inventory pending |
| 6.18 | AccECN core; UDP RX evolution; DIBS separate shared-memory lineage | B | release-generation |
| 6.19 | `dev_queue_xmit()` llist TX scheduling; threaded-NAPI kthread busy-poll extension; WireGuard YNL-described Netlink | A | final `v6.19`; selected architecture milestones from the 6.19 networking cycle |
| 7.0 | cake_mq / multi-queue-aware sch_cake; IPv6 BIG TCP without synthetic HBH jumbo header; AccECN enablement; large RX buffers for memory providers/io_uring ZCRX | A | final `v7.0`; AccECN enablement and large-buffer ZCRX complete earlier lineages |
| 7.1 | RX HW queue leasing; dedicated qdisc-drop tracepoint | A | final `v7.1`; TX queue leasing remains unmerged; qdisc drop context becomes directly observable |
| 7.2 | MPTCP PM limits: subflows 8→64; accepted ADD_ADDR 8→64; endpoints 8→255; PPPoE GRO/GSO | A | final `v7.2`; MPTCP limit expansion plus aggregation support over PPPoE |
| 7.3-rc / mainline | BIG TCP over VXLAN/GENEVE; RTNL-less FIB-rule updates; devmem buffers > PAGE_SIZE; per-netns netdev-unregistration infrastructure | A-rc | net-next-7.3 merged 2026-08-20; final 7.3 pending |

**7.3 status note:** 調査基準日時点では final 7.3 は未リリースのため `A-rc` とする。正式リリースを確認した時点で `A` へ更新する。

**MPTCP endpoint-limit note:** endpoint の実効上限は **255** である。endpoint ID は8-bitで、ID 0が予約されているためである。

## 各世代を支配したテーマ

``` text
Linux 3.x
  scalability + virtualization + programmability foundations

Linux 4.x
  programmable fast-path formation
  + hardware-control models
  + early zero-copy / packet-memory foundations

Linux 5.x
  programmability expansion
  + operationalization
  + packet-memory infrastructure maturation

Linux 6.x–7.x
  memory-provider / queue-ownership architecture
  + finer-grained control and locking
```

これは各時代を特徴づける説明上の区分であり、機能の起源を示す境界ではない。たとえば
queue control は Linux 4.x より前から始まり、packet-memory の取り組みも
6.x の memory-provider 世代より前から存在する。

## 6つの architecture 軸

以降では、この chronology を相互に関係する6つの architecture
軸から読む。

``` text
PERFORMANCE
PROGRAMMABILITY
MEMORY
CONTROL PLANE
OBSERVABILITY
DRIVER FRAMEWORK
```

1つの機能が複数の軸に属することもある。たとえば `page_pool` は memory
mechanism であると同時に driver-framework contract
でもあり、`netdev-genl` は control plane と observability の双方の
infrastructure である。

## Part II から Part III へ --- chronology から lineage へ

release table
が答えるのは「いつか」である。次章以降では「ある仕組みがどのように次の仕組みへつながったか」を扱うため、release
table を繰り返さず architecture ごとに milestone をまとめる。

# Part III --- Long-term feature lineages

Part III では、Part II の milestone が**どの設計課題を共有し、どこで依存・拡張・並行発展するか**を説明する。release ごとの provenance
は繰り返さない。driver-specific な仕組みは Part IV、tracing/tooling
の詳細は Part V で扱う。

## Packet aggregation --- GRO/GSO → BIG TCP

v5.0 ですでに GRO/GSO/TSO は成熟していたが、高速 NIC では per-packet
metadata processing が支配的になる。

``` text
wire packets
    ↓ GRO
large skb
    ↓ GSO/TSO
wire packets
```

v5.19 BIG TCP は kernel internal GRO/GSO aggregate の 64KiB
制約を緩和した。

``` text
GRO/GSO
  ↓
IPv6 BIG TCP (5.19)
  ↓
IPv4 BIG TCP (6.3)
  ↓
IPv6 BIG TCP without synthetic HBH jumbo header (Linux 7.0)
  ↓
BIG TCP over VXLAN / GENEVE (7.3-rc/mainline)
```

BIG TCP は wire MTU を巨大化する機能ではなく、 **kernel 内部の
packet-processing unit を大きくする機能**として理解する。利用可否と効果は protocol path、
GRO/GSO/offload capability、driver/NIC、tunnel implementation などに依存し、すべての device / path で
一律に大きな aggregate を利用できることを意味しない。

------------------------------------------------------------------------

## Packet memory --- page_pool → netmem → Device Memory TCP

``` text
old RX:
NIC → allocate page → stack → free page

page_pool:
NIC → recycled page → stack ─┐
      ↑                      │
      └──────────────────────┘
```

その後、device memory を扱うため `struct page`
前提を弱める必要が生じる。

``` text
network memory
      ↓
    netmem
   /      \
RAM      device memory
```

v6.12 Device Memory TCP RX:

``` text
traditional:
NIC → system RAM → CPU/copy/mapping → GPU

devmem:
NIC ─────────────→ device memory → GPU/accelerator
```

v6.16 では TX 側も mainline に入る。

**DIBS はこの直系ではない。**

``` text
page_pool → netmem → Device Memory TCP / memory providers
```

はRX/TX packet-memory
ownershipのlineageである。一方、DIBSはshared-memory
transport側の別lineageとして扱い、この矢印へ直接接続しない。

------------------------------------------------------------------------

## XDP / AF_XDP

v5.0 時点で XDP/AF_XDP は存在した。その後の本質は周辺 infrastructure
の成熟。ここで XDP → AF_XDP → netkit / queue leasing を単純な派生関係とはみなさない。
XDP は native driver mode、generic/SKB mode、hardware offload で実行位置・性能特性・必要な driver support が異なり、
AF_XDP や queue leasing は queue ownership / zero-copy という共通課題から並行して発展した面を持つ。

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

------------------------------------------------------------------------

## BPF --- packet filter から stack extension へ

BPF の発展は一本道ではなく、attachment point と適用範囲が複数方向へ増えたものとして捉える。

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

ここで下段は上段の単純な後継ではない。5.x～6.x にかけて **適用範囲と attachment point が
独立・並行して追加された**結果として、packet、socket、protocol algorithm、virtual device、
queue/datapath まで programmability の対象が広がった、と読む。

------------------------------------------------------------------------

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

これらは upstream Linux release milestone ではないため Part II には追加せず、architecture の動機を説明する research evidence として扱う。

## Virtual networking --- veth/virtio → netkit/queue ownership

v5.0 の典型:

``` text
container → veth → host stack → NIC

guest virtio-net → QEMU/vhost/TAP → bridge/OVS → NIC
```

現在は複数の branch がある。

``` text
virtio-net
├─ vhost
├─ vDPA
├─ SR-IOV/VFIO
└─ AF_XDP zero-copy

container:
veth → netkit → BPF-native datapath → RX queue leasing
```

v6.7 netkit、v6.11 virtio-net AF_XDP RX ZC、v7.1 RX HW queue leasing
は 「full kernel bypass」よりも、

**kernel が ownership/control を保持し、data movement を最小化する**

方向として読むと理解しやすい。

------------------------------------------------------------------------

## TCP / UDP / transport

### TCP

``` text
TCP zero-copy RX (`TCP_ZEROCOPY_RECEIVE`, 4.18)
  ↓
v5.0 era (EDT pacing already present since 4.20)
  ├─ BPF congestion control
  ├─ MPTCP (5.6)
  ├─ BIG TCP (5.19)
  ├─ TCP-AO/security
  ├─ Device Memory TCP
  └─ AccECN
```

MPTCP は initial upstream から multi-subflow、userspace path manager、
Generic Netlink、BPF integration へ進化した。

### Linux 6.0 --- io_uring networking inflection

Linux 6.0 is the selected networking milestone for io_uring zero-copy
send (`IORING_OP_SEND_ZC`) and multishot receive. This closes the
chronology between the earlier io_uring substrate and the later 6.15
ZCRX memory-provider architecture.

### UDP

v5.0 自体が `MSG_ZEROCOPY` と GRO の節目。その後は GRO/GSO、
tunnel/encapsulation、high packet-rate RX、receive-buffer scaling
が進む。 QUIC や overlay networking の基盤として UDP の重要性も増した。

------------------------------------------------------------------------

## Routing / Netlink / RTNL

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

7.3向けnetworking pullでは `RTM_NEWRULE` / `RTM_DELRULE` のFIB
rule変更が RTNL-lock-less化され、further RTNL-dependency
reductionやlock-less GET準備と 同じ「global
RTNL依存を減らす」流れとしてmainlineへ入った。

さらに 7.3 cycle では per-netns netdev unregistration infrastructure が入り、
per-netns RTNL の大きな blocker だった device unregistration path の分解も進んだ。
これは 6.13 以降の per-netns RTNL lineage の継続として扱う。

------------------------------------------------------------------------

## Linux 6.19 --- TX scheduling と structured Netlink の継続

6.19 では `dev_queue_xmit()` の deferred TX path が lockless list (`llist`) を使う形へ
再構成され、shared qdisc / multiqueue TX scalability の改善が進んだ。ここでは特定 benchmark の
「4倍」という値を一般化せず、**TX scheduling architecture の変更**として扱う。

同じ release では threaded NAPI の kthread-based busy polling 拡張と、WireGuard Netlink の
YAML/YNL specification 化も入り、execution model と machine-readable Netlink API の両方で
既存 lineage が前進した。

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

conntrack では performance だけでなく lifetime/GC、per-netns
scalability、 hardware flow offload race、BPF kfunc access
が重要なテーマとなった。

------------------------------------------------------------------------

## io_uring networking

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

目標は syscall reduction だけではなく、 NICからuserspaceまでの buffer
ownership/lifetime の効率化にある。

------------------------------------------------------------------------

## Part III から Part IV へ --- feature lineage から driver contract へ

前章では networking mechanism を end-to-end の lineage
として追った。Part IV では視点を変え、個々の driver の責務のうち何が共通
networking-core framework へ移されたかを見る。

# Part IV --- Network Device Driver Framework の進化

本章では networking core と driver の間の contract
に焦点を当てる。`page_pool`、AF_XDP、Device Memory TCP、io_uring は Part
III にも登場するが、ここでは feature 全体の歴史ではなく、**driver
から見た queue / memory ownership** の役割を扱う。

## Driver Framework の正規 chronology

これは Part II の release map を Driver Framework
の観点から投影したものであり、独立した第2の chronology ではない。以下の
release attribution はすべて Part II と一致させる。

| Release | Driver-framework milestone | Architectural effect |
|---|---|---|
| 3.3 | DQL/BQL | common queue-pressure control replaces driver-local queue sizing policy |
| 3.19 | switchdev origin | Linux forwarding objects begin to drive switch-ASIC offload |
| 4.6 | devlink | device/ASIC-wide resources and control separated from one `net_device` |
| 4.8 | XDP | driver RX path gains a programmable pre-skb execution point |
| 4.14 | phylink | common MAC/PHY/PCS/SFP link-management state machine |
| 4.16 | netdevsim | common offload APIs become testable without physical hardware |
| 4.18 | refurbished page_pool/XDP memory return | RX allocation/recycling begins moving into common memory infrastructure |
| 5.1 | devlink health | common reporting/recovery model for device health |
| 5.6 | ethtool Generic Netlink | driver management ABI becomes structured/extensible |
| 5.11 | auxiliary bus | complex devices can expose independently bound subfunctions |
| 5.12 | threaded NAPI | NAPI execution model becomes more explicitly configurable |
| 6.8 | Rust phylib + queue/NAPI netdev-genl objects | safe driver abstraction and explicit netdev objects develop in parallel |
| 6.x→7.x | page_pool introspection → queue/NAPI configuration → memory providers | queue, poller and memory ownership become first-class driver/core contracts |

### 長期的な architecture の変化

``` text
driver-private mechanisms
        ↓
common queue/backpressure primitives
        ↓
common hardware-offload and device-control models
        ↓
common link, interrupt and RX-memory frameworks
        ↓
testable + structured userspace-visible driver APIs
        ↓
queue / NAPI / page-pool objects
        ↓
queue-bound memory providers
        ↓
typed / memory-safe Rust driver abstractions
```

後半の変化は特に重要である。modern netdev では、従来 driver
内部の不透明な実装詳細だったものを、ID と関係性を持つ generic object
として公開する方向が強まっている。

本章では個々の NIC driver を網羅的には列挙しない。Linux network driver
に要求される実装を変化させ、driver
ごとに重複していた仕組みを再利用可能な kernel framework へ移した common
infrastructure を追う。

## Linux 3.0 時点の出発点

Linux 3.0 の時点で NAPI と multiqueue networking
はすでに確立していた。RPS/RFS は 2.6.35、XPS は 2.6.38
で導入済みであり、v3.x の driver model
はすでに次の要素を中心としていた。

``` text
RX/TX descriptor rings
IRQ / MSI-X
NAPI poll contexts
multiple RX/TX queues
RSS in hardware
RPS/RFS/XPS in the stack
ethtool + net_device_ops
```

したがって v3.x における重要な変化は NAPI の発明ではなく、driver と共通
networking-core algorithm の協調が強まったことである。

## Linux 3.3 --- DQL/BQL: queue control が共通 core へ移る

DQL/BQL は、この変化を示す初期の代表例である。

``` text
old:
  each driver/hardware queue can accumulate excessive TX backlog

DQL/BQL:
  common dynamic queue-limit algorithm
  + driver reports queued/completed bytes
  + core dynamically controls outstanding data
```

Exact DQL core anchor already audited elsewhere in this document:

``` text
75957ba36c05b979701e9ec64b37819adc12f830
dql: Dynamic queue limits
```

LWN series: https://lwn.net/Articles/469651/
https://lwn.net/Articles/469652/

Driver Framework の観点では、performance policy が個々の driver
から再利用可能な net core infrastructure へ移り始めたことに意味がある。

## Linux 4.x --- hardware が Linux networking の first-class object になる

### switchdev --- Linux 3.19 で始まり 4.x で拡大

The initial switchdev infrastructure belongs to the Linux 3.19
generation. The 4.x era is where the model expands into the broader
hardware-offload architecture discussed below.

switchdev は switch ASIC forwarding を proprietary SDK
だけが制御する孤立した仕組みではなく、Linux driver model
の一部として扱えるようにする。

LWN: https://lwn.net/Articles/675826/ https://lwn.net/Articles/676096/

The model represents physical switch ports as normal netdevices and lets
bridge, VLAN, routing and related kernel objects drive hardware offload.

``` text
Linux bridge / FIB / VLAN / TC
          ↓
     switchdev model
          ↓
     switch ASIC driver
          ↓
        hardware
```

これは driver の責務における大きな変化であり、driver は Linux networking
semantics を hardware 上へ実装する役割を担うようになる。

### devlink --- device 全体を扱う control plane

devlink series (v3): https://lwn.net/Articles/677967/

devlink fills a gap left by `net_device`: many settings belong to the
whole ASIC/device, not one network interface.

Examples include:

``` text
device resources
port splitting
shared buffers
eswitch mode
device parameters
firmware / health
```

Linux 5.1 later adds devlink health reporting/recovery as a generic
mechanism.

This establishes a useful split:

``` text
net_device / ethtool
  interface-facing behavior

devlink
  device / ASIC / resource / health behavior
```

### phylink と SFP

The 2015 26-patch phylink/SFP RFC is **design provenance**, not the
mainline landing. The canonical infrastructure anchor is:

``` text
9525ae83959b60c6061fe2f2caabdc8f69a48bc6
phylink: add phylink infrastructure
Russell King
authored 2017-07-25; committed 2017-08-06
```

Thus the canonical history is
`2015 RFC → Linux 4.14 mainline infrastructure → later PCS/SFP/MAC API expansion`.

The 2015 phylink/SFP RFC addresses a recurring driver problem: MAC, PHY,
PCS/SerDes and hot-pluggable SFP combinations could not be modeled
cleanly by simple PHY attachment.

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

This progressively removes link-mode state-machine duplication from
Ethernet MAC drivers.

### VF representor と SmartNIC/DPU control

Representors extend the switchdev idea to SR-IOV embedded switches and
later SmartNIC/DPU architectures.

LWN: https://lwn.net/Articles/692942/

A representor is both a control-plane representation of a VF/SF and a
netdevice endpoint through which the normal Linux stack can control the
virtual switch.

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

Linux 4.8 introduces first-generation XDP. From the driver's point of
view the important change is that the RX path gains a programmable hook
before skb allocation / normal stack processing.

Kernel Recipes 2018 explicitly describes XDP as a programmable layer
running in device driver context:
https://archives.kernel-recipes.org/document/xdp-a-new-programmable-network-layer/

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

### DIM --- interrupt moderation の共通 library 化

**Release attribution:** Net DIM is treated as a Linux 4.16 generation;
the later DIM refactor/generalization into common `lib/dim`
infrastructure is a Linux 5.3 generation. Exact origin/generalization
SHAs remain to be pinned in this edition.

Netdev 0x12 (2018) presented DIM as a driver-independent Dynamic
Interrupt Moderation library.

https://www.netdevconf.info/0x12/

Rather than every driver inventing adaptive interrupt/coalescing
algorithms:

``` text
driver samples events/bytes/packets
        ↓
common DIM algorithm
        ↓
profile decision
        ↓
driver programs hardware moderation
```

This is another example of extracting policy from drivers into common
netdev infrastructure.

### page_pool --- RX memory management の共通 infrastructure 化

The page_pool work was motivated by drivers independently reinventing
high-speed DMA page recycling. A 2016 RFC explicitly described it as a
generic API for streaming-DMA page pools, and the refurbished
implementation appears in the 2018 XDP-era work.

Key late series: https://lists.openwall.net/netdev/2018/03/31/91

Modern page_pool provides a common allocation/recycling/DMA model for
skb and XDP buffers.

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

Its importance grows well beyond the original XDP motivation. By Netdev
0x19, page_pool is described as the standard RX datapath
memory-management mechanism, and newer zero-copy features require
drivers to integrate with it.

## Linux 5.x --- driver management API の構造化と可観測性

### devlink health

Linux 5.1 adds generic devlink health reporting and recovery.

Netdev 0x13 describes the goals as:

``` text
real-time alerting
driver debug information
self-healing / recovery
vendor-support data collection
```

Conference: Netdev 0x13, "devlink health reporting and recovery system".
以前記載していた `loadsessions/...` URL
は実在を確認できないため削除し、conference archive/session index
を参照する。

This changes hardware error handling from driver-specific logs/private
tools toward a common operational model.

### ethtool ioctl → Generic Netlink

The ethtool netlink work addresses limitations of the old ioctl ABI:
extensibility, races, error reporting and lack of notifications.

Series: https://lwn.net/Articles/808028/
https://lwn.net/Articles/810618/

Architecturally this is not just a userspace-tool rewrite. It creates a
structured, extensible management API between userspace, networking core
and drivers.

### netdevsim と selftest-driven driver API design

`netdevsim` becomes an important test vehicle for driver-facing APIs.
Current netdev maintainer documentation explicitly encourages new driver
configuration APIs to have netdevsim/selftest coverage, while also
requiring a real driver use case.

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

Driver frameworks are increasingly expected to be testable without the
physical NIC.

### auxiliary bus --- 1つの PCI device と複数 subsystem driver

Merged for Linux 5.11, the auxiliary bus addresses complex devices
exposing Ethernet, RDMA, vDPA and related functions from shared
hardware.

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

## Linux 6.x--7.x --- queue と memory が明示的な framework object になる

### page_pool の可観測化

2023 page_pool netlink introspection associates pools with netdevices
and NAPI IDs and exports allocation/recycling/memory information.

LWN: https://lwn.net/Articles/948718/

This is an important architectural transition:

``` text
page_pool as hidden driver implementation detail
                 ↓
page_pool as identifiable / observable netdev resource
```

### queue / NAPI object の generic netdev API 化

Netdev 0x17 discusses exposing queues and NAPI instances through
`netdev-genl`.

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

This is a conceptual shift from "the driver owns opaque rings" toward explicit
queue / NAPI / memory objects with stable identities and API-defined properties.

ただし **可視化・設定・割り当て・ownership は同義ではない**。API ごとに許される操作は異なり、
ある object を userspace から列挙・参照できても、その lifetime や ownership を userspace が自由に
変更できるとは限らない。netdev-genl の read/configuration API、memory-provider registration、
queue-leasing のような assignment mechanism は、それぞれ capability と permission boundary を個別に読む必要がある。

This connects conceptually to Device Memory TCP, io_uring ZCRX and queue-leasing work elsewhere in this document,
but does not imply that one generic API grants all of those control operations.

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

Kernel Recipes 2024 の公開 abstract が直接述べるのは、kernel network
stack を利用し、vanilla TCP と互換性のある zero-copy receive
の設計である。NIC / firmware / driver support、page_pool、netmem、queue
configuration という具体的な実装依存関係は abstract 自体ではなく、同
conference の live blog と後続 upstream implementation から確認する。

つまり modern high-speed network driver は、単に `struct page` を
allocate するだけではなく、次第に **memory-provider-aware**
であることを求められている。

### XFRM device / IPsec packet offload

XFRM device offload is another common driver-framework contract. The
important 6.2 generation extends the model from crypto acceleration to
packet offload, where the NIC can own SA/policy processing as well as
encryption/decryption.

``` text
XFRM core
   ↓ state + policy synchronization
xfrmdev_ops
   ↓
NIC driver
   ↓
IPsec hardware pipeline
```

This is the same architectural pattern seen in switchdev and TC offload:
the kernel keeps the canonical networking semantics while the driver
maps those semantics onto hardware.

## Queue / NAPI / page_pool object model

この節の正本は直前の「queue and NAPI objects move toward a generic
netdev API」と 「page_pool → netmem → memory
providers」です。ここでは重複した図を再掲しません。

## Rust --- language support から safe driver model へ

### Context: Linux 6.1 --- Rust が kernel に入る

> これは networking milestone として Part II に採用した項目ではなく、6.8 の Rust PHY
> milestone を理解するための kernel-wide contextual reference である。

Linux 6.1 introduces the initial Rust-for-Linux support. This does not
yet mean that network drivers can generally be written in Rust;
driver-facing abstractions must be built subsystem by subsystem.

### 2023 --- 初期 network-device / PHY abstraction

A June 2023 proposal adds minimum Rust abstractions for `net_device`
drivers and a Rust dummy driver:

https://lwn.net/Articles/934517/

In parallel, PHY abstractions mature through repeated review.

### Linux 6.8 --- Rust PHY support が mainline へ

Linux 6.8 is the first major networking-driver milestone. LWN's
merge-window coverage says Rust support for creating network PHY drivers
was added, including abstractions and an Asix reference PHY driver.

https://lwn.net/Articles/957188/

This is more significant architecturally than the size of the example
driver:

``` text
C phylib API
     ↓
sound Rust abstraction boundary
     ↓
safe Rust PHY driver
```

The abstraction decides where `unsafe` is contained and what
lifetime/state guarantees can be expressed in the type system.

### Driver core / PCI / platform / DMA / MMIO / IRQ abstraction

Writing a real high-performance NIC driver needs much more than
`net_device_ops`:

``` text
PCI / platform probing
MMIO
DMA mapping
IRQ
device resources
lifetime / removal handling
networking abstractions
```

The 2024--2026 Rust driver-core work therefore matters directly to
future network drivers, even when developed outside `net/`. なお Kernel
Recipes 2024 の Andreas Hindborg の講演は block-device driver API
を具体例とした講演であり、networking talk ではない。本書では Rust/C API
boundary の一般的な driver-framework lesson としてのみ参照する。

Kernel Recipes:
https://kernel-recipes.org/en/2024/schedule/interfacing-kernel-c-apis-from-rust/
https://kernel-recipes.org/en/2025/schedule/so-you-want-to-write-a-driver-in-rust/
https://kernel-recipes.org/en/2026/schedule/enforcing-device-driver-lifecycle-rules-at-compile-time/

Kernel Recipes 2026 の公式 schedule と講演ページで Danilo Krummrich の
"Enforcing Device Driver Lifecycle Rules at Compile Time"
の実在を確認できる。この講演は networking-specific ではなく Rust
driver-core / Linux device model の講演である。その driver-model work は
lifecycle conventions を compile-time invariants
に変換することを主要な目標としている:

``` text
C driver:
 conventions + documentation + review

Rust driver:
 lifetime + ownership + type state
             ↓
 compile-time lifecycle constraints
```

### Rust は単なる C から Rust への書き換えではない

The deeper significance is that common driver frameworks become **safe
abstraction boundaries**.

The long-term lineage is therefore:

``` text
common C driver framework
  NAPI / phylib / devlink / page_pool / DMA / PCI
                   ↓
well-defined ownership and lifecycle contracts
                   ↓
Rust abstractions around those contracts
                   ↓
more driver logic can live in safe Rust
```

Netdev 0x17's Rust networking tutorial emphasized memory safety and
prevention of use-after-free, double-free and data-race classes, while
Kernel Recipes 2026 extends this idea to device lifecycle rules
themselves.

## Conference に見る lineage

### Netdev

Netdev is the strongest conference source for driver-framework
implementation:

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

Kernel Recipes is especially useful for architecture:

``` text
2018  XDP as a programmable layer in driver context
2019  XDP integration and generic packet-buffer ideas
2024  zero-copy networking (abstract: kernel-stack / vanilla-TCP compatibility; implementation details: live blog + upstream evidence)
      Rust/C API abstraction discussion (block-device example; not a networking talk)
2025  practical Rust driver development
2026  compile-time enforcement of driver lifecycle rules (Rust driver-core/device model; not networking-specific)
```

## Part IV から Part V へ --- mechanism から observability へ

fast path、memory ownership、offload は、operator/developer が kernel
の動作を理解できて初めて運用可能になる。Part V
では、それと並行して進化した visibility、tracing、explainability
を追う。

# Part V --- Observability / Explainability

Part V は **kernel 側の observability primitive の進化**に限定します。
Retis / pwru はこれらを利用する case study であり、kernel release
chronology そのものではないため Appendix に移します。

## Kernel observability primitive

Linux networking の observability は、単純な interface counter / packet
capture から、 kernel 内部の typed event と packet-lifecycle metadata
を相関できる方向へ進化しました。

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

| Primitive | 主に提供するもの | 境界 / 注意点 |
|---|---|---|
| BTF | kernel type / field metadata | runtime event や packet trajectory 自体は記録しない |
| eBPF tracing | attach point での event / state | coverage、attach point、権限、overhead に依存 |
| `skb_drop_reason` | 対応箇所での structured drop reason | 全 drop path が必ず reason を付与するわけではない |
| timestamping | 特定地点での時刻情報 | clock、取得地点、HW/SW timestamp semantics に依存 |
| netdev-genl | queue / NAPI 等の identity、state、configuration | 可視化できることと自由に ownership/configuration できることは別 |
| page_pool introspection | pool identity / stats / diagnostics | packet 単位の end-to-end trajectory ではない |

したがって cross-layer packet journey は単一 primitive の機能ではなく、tool が複数の identifier、event、
timestamp、metadata を相関して **推論する目標**として扱う。

### BTF と eBPF tracing

BTF により running kernel の型情報を利用できるため、observability tool
は private kernel structure
の固定offsetに依存する必要を減らせます。これは networking
のように内部構造の変化が 速い領域で特に重要です。

### Structured drop reason

Linux 5.17 の `kfree_skb_reason()` / `skb_drop_reason` 世代は、「packet
が消えた」という観測を 「どの理由でdropされたか」という structured
metadata へ変えました。その後、coverage は networking stack
の各所へ拡張されています。

### Timestamping と packet lifecycle

`SO_TIMESTAMPING`、driver/hardware
timestamp、BPFから取得できる時刻・contextは、 単一地点のpacket
captureでは見えない queueing / scheduling / offload の時間軸を補います。

### Queue / NAPI / page_pool observability

Part IVで説明した queue、NAPI、page_pool のobject化は、control
planeだけでなくobservabilityにも 効きます。packet memory、polling
context、queue identityをuserspace-visible objectとして関連付ける
ことで、zero-copy / memory-provider時代の問題を説明しやすくなります。ただし、readable な identity / stats と、
userspace が queue や NAPI の ownership/configuration を変更できる control-plane capability は区別する。

この軸を Part II の確定 milestone に対応させると、次の流れになる。

``` text
5.17  kfree_skb_reason / structured drop-reason foundation
  ↓
5.19  drop-reason coverage expansion
  ↓
6.8   queue/NAPI netdev-genl visibility
  ↓
6.x–7.x  page_pool / queue / memory-provider diagnostics
```

`netdev-genl` による queue / NAPI identity と stats は、従来の interface-level
counter だけでは説明しにくかった「どの queue / poller で問題が起きているか」を
userspace から関連付ける基盤になる。page_pool 側でも pool identity、allocation /
recycle information、leak diagnostics が整備され、packet-memory ownership 自体が
観測対象へ移っている。ここでの release attribution は Part II の値を参照する。

## Tool の case study

Retis と pwru は上記primitiveを利用する代表例ですが、本書ではkernel
evolutionそのものと区別します。

-   **pwru**: 広いkernel function
    trajectoryから「packetがどこを通ったか」を探索する。
-   **Retis**: networking event、skb metadata、OVS/OVN
    contextなどを意味的にenrichして相関する。

詳細なconference/source provenanceはAppendixのcase-study
indexに集約します。

## Part V から Part VI へ --- evidence から synthesis へ

# Part VI --- Synthesis: v3.x → 7.x を一つの進化として見る

## v5.0 と 2026 を比較する

``` text
                     v5.0                    2026

packet memory        early page_pool  →      page_pool maturation / memory providers
fast path            XDP/AF_XDP       →      AF_XDP MB / netkit / queue lease
aggregation          GRO/GSO          →      BIG TCP / tunnel BIG TCP
BPF                   packet/socket    →      struct_ops/netkit/BPF qdisc
virtual networking   veth/virtio      →      vDPA/AF_XDP ZC/netkit
control locking      global RTNL      →      per-netns/fine-grained
Netlink API           hand-written     →      YNL-described
memory path          NIC→RAM          →      NIC→RAM or device memory
async I/O             conventional     →      io_uring ZC TX/RX
```

最大の変化は、従来の **「CPU が system RAM 上の skb を処理するモデル」自体を置き換えたことではなく**、
そのモデルを現在も広く維持しながら、AF_XDP、device memory、io_uring zero-copy、programmable/offloaded path など
**複数の処理・memory-ownership model を用途と hardware capability に応じて共存させる方向へ拡張したこと**にある。

------------------------------------------------------------------------

## Evolution map

``` text
                    Linux 5.x
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
      XDP/BPF         GRO/GSO       page_pool (already present)
        │               │                │
        ▼               ▼                ▼
   struct_ops        BIG TCP       netmem / memory providers
   SK_LOOKUP             │                │
        │                │                │
        ▼                │          ┌─────┴─────┐
     netkit              │          ▼           ▼
        │                │      Devmem TCP   io_uring ZCRX
  queue leasing          │          │           │
        └────────────┬───┴──────────┴───────────┘
                     ▼
              hybrid / zero-copy
                     │
             ┌───────┴────────┐
             ▼                ▼
          container           VM
       BPF/netkit/CNI   virtio/vDPA/AF_XDP
```

control plane:

``` text
global RTNL ─────────────→ per-netns / fine-grained locking
ad-hoc Netlink ──────────→ YNL-described APIs
single network model ────→ multi-network / VM-aware networking
```

------------------------------------------------------------------------

## 読み方

推奨順序:

1.  Part IIでrelease attributionを確認する。
2.  Part III--Vでfeature / driver / observabilityのlineageを読む。
3.  Part VIIで未解決のattribution境界を確認する。
4.  Appendixは一次資料・conference・commit
    provenanceを検証するときに参照する。

Appendixは旧release chronologyのコピーではなく、non-normativeなevidence
catalogである。

------------------------------------------------------------------------

## Transition: synthesis → provenance

The synthesis is intentionally compact. Part VII records the attribution
boundaries that remain important for verification or future re-audit.

# Part VII --- 正規 provenance ledger

**Evidence model:** Part VII の SHA は feature series の「代表 anchor」であり、anchor の存在だけで series 全体を証明しない。
各項目は **SHA identity / feature correspondence / release containment** を別々に監査する。

**Evidence status:** `A-rc` は authoritative な pull/merge evidence により Linus mainline への merge を確認済みだが、final release が未公開の状態を示す。released milestone の Grade A とは区別する。

## 本版で保持する exact mainline anchor inventory

ここに示すのは代表的な **anchor commit** であり、feature series の全commitを列挙するものではない。

### Part IV TODO から確定できた origin / landing anchors

以下は exact commit と release containment を確認できたため、従来の「SHA pending」から確定扱いへ移す。

| Item | Exact mainline anchor | First containing tag |
|---|---|---|
| devlink | `bfcd3a46617209454cfc0947ab093e37fd1e84ef` — `Introduce devlink infrastructure` | v4.6-rc1 |
| netdevsim | `83c9e13aa39aed5cf9a2f8dd69770b7c35ba1281` | v4.16-rc1 |
| auxiliary bus | `7de3697e9cbd4bd3d62bafa249d57990e1b8f294` — `Add auxiliary bus support` | v5.11-rc1 |
| Rust PHY abstractions | `f20fd5449ada3872dcd67aca397f0e27ca2e8ad6` — `rust: core abstractions for network PHY drivers` | v6.8-rc1 |
| netdev-genl queue object | `bc877956272f0521fef107838555817112a450dc` — `netdev-genl: spec: Extend netdev netlink spec in YAML for queue` | v6.8-rc1 |

Net DIMについては、algorithm自体は4.16以前からmlx5e内に存在しており、4.16は「共通libraryへの切り出し」という表現を維持する。本版では、提示されたDIM SHAを一次Git objectとして再確認できていないため exact inventory への追加は保留する。


| Feature | Mainline anchor | Subject / role |
|---|---|---|
| DQL | `75957ba36c05b979701e9ec64b37819adc12f830` | `dql: Dynamic queue limits` |
| CoDel | `76e3cc126bb223013a6b9a0e2a51238d1ef2e409` | CoDel qdisc core anchor |
| SO_REUSEPORT infrastructure | `055dc21a1d1d219608cd4baac7d0683fb2cbbe8a` | `soreuseport: infrastructure` |
| nftables core | `96518518cc417bb0a8c80b9fb736202e28acdf96` | `netfilter: add nftables` — core origin anchor |
| nftables set API | `20a69341f2d00cd042e81c82289fba8a13c05a25` | set-API anchor; not the core origin |
| BBR | `0f8782ea14974ce992618b55f0c041ef43ed0b78` | initial BBR mainline anchor |
| phylink | `9525ae83959b60c6061fe2f2caabdc8f69a48bc6` | `phylink: add phylink infrastructure`; Linux 4.14 |
| TCP MSG_ZEROCOPY | `f214f915e7db99091f1312c48b30928c1e0c90b7` | `tcp: enable MSG_ZEROCOPY` |
| page_pool origin | `ff7d6b27f894f1469dc51ccb828b7363ccd9799f` | `page_pool: refurbish version of page_pool code` |
| page_pool/XDP integration | `60bbf7eeef10dc647430646d7fe5e3d8d132dbec` | mlx5 page_pool/XDP integration anchor |
| ethtool Generic Netlink | `2b4a8990b7df55875745a80a609a1ceaaf51f322` | `ethtool: introduce ethtool netlink interface` |
| SK_LOOKUP | `e9ddbb7707ff5891616240026062b8c1e29864ca` | `bpf: Introduce SK_LOOKUP program type with a dedicated attach point` |
| XFRM packet offload | `d14f28b8c1de668bab863bf5892a49c824cb110d` | `xfrm: add new packet offload flag` |
| IPv6 BIG TCP / GRO | `0fe79f28bfaf73b66b7b1562d2468f94aa03bd12` | `net: allow gro_max_size to exceed 65536` |
| IPv6 BIG TCP / GSO | `7c4e983c4f3cf94fcd879730c6caa877e0768a4d` | `net: allow gso_max_size to exceed 65536` |
| netkit | `35dfaad7188cdc043fde31709c796f5a692ba2bd` | netkit core anchor |
| net-next 7.3 merge | `91ec2035134982b98fab0609a9fd8480e8217dc1` | `Merge tag 'net-next-7.3' ...` |

**MPTCP 7.2 limit expansion anchors:** 8-patch series のうち上限値を直接引き上げる commit は2つである。

- `c8646664fbf1c0beb0990cef391cb52d3c909e78` — subflows と accepted `ADD_ADDR` の上限をともに 64 へ拡大
- `e845e6397d78bf6b842cfa8b5818ca8189f7e22e` — endpoint 上限を 255 へ拡大

残りは preparation / selftest であり、「three limit-expansion commits」とは数えない。

## Evidence grades

Grade の正本は Part II と canonical dataset です。Part VII は
commit-level anchor と、 まだ解消していない attribution boundary
のみを記録します。

Appendix は evidence catalog である。そこに現れる日付は RFC date、posting date、review base、conference date の場合があり、Part II に明記されない限り release attribution として読まない。

## Open attribution items and evidence grade

以下は Part II の Grade を再定義する表ではなく、未解決の attribution boundary を記録する補助表である。Grade を記載する場合は Part II と完全に一致させる。

| Item | Canonical Grade | Current treatment |
|---|---|---|
| RX HW queue leasing | A | revised RX implementation is contained in final `v7.1`; TX queue leasing remains unmerged |
| devmem buffers > `PAGE_SIZE` | A-rc | explicitly listed in the `net-next-7.3` pull merged to Linus mainline; final 7.3 pending |
| MPTCP PM limit expansion | A | limit-expansion commits are contained in final `v7.2`; endpoint maximum is 255 because ID 0 is reserved |
| DIM / `net_dim` | B | Linux 4.16 Net DIM generation; Linux 5.3 common `lib/dim` generalization; exact SHAs pending |
| `cake_mq` | A | `net-next-7.0` pull explicitly lists multi-queue-aware `sch_cake`; canonical generation Linux 7.0 |

**Queue-leasing release boundary:** initial merge `77b9c4a438fc66e2ab004c411056b3fb71a54f2c` と revert `8766d61a1d33cb5f15bfdd6ce9832bbe1fc649c2` はともに v7.0 merge window 内で相殺され、v7.0 release には含まれない。v7.1 の revised RX series では `7789c6bb76ac` (`net: Add queue-create operation`) が queue-leasing infrastructure の代表的な series anchor であり、final `v7.1` に含まれる。以前ここに記載していた `15089225889ba4b29f0263757cd66932fa676cb0` は `Merge branch 'netkit-support-for-io_uring-zero-copy-and-af_xdp'` であり、queue-leasing implementation commit として扱うのは誤りだったため撤回した。TX queue leasing は未 merge である。

### 2026 networking-cycle audit note

6.19--7.3 の再監査では、全 networking commit を chronology に列挙するのではなく、
本書の6軸（performance / programmability / memory / control plane / observability / driver framework）
の長期 lineage を変える項目だけを Part II に採用した。したがって TLS RFC 8449、ICMP RFC 5837、
UDP-Lite removal、個別 XFRM message、TCP-AO crypto-library refactor、個別 tunnel/offload enhancement などは
有用な変更だが、本版では canonical architecture milestone には昇格させていない。

# Appendix --- Source index（非正規）

Appendix は supporting evidence / research provenance の索引である。本文の release
attribution や lineage を再定義しない。

## USENIX research index --- architecture motivation / design-space evidence

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

これらは Part II / Part VII の canonical release/commit evidence と混同しない。


## Kernel documentation / primary-source index

| Source topic | Canonical destination | What it supports |
|---|---|---|
| NAPI / `SO_BUSY_POLL` | Part II 3.11; Part IV | busy polling and NAPI execution model |
| BPF DEVMAP | Part II 4.14; Part III XDP | XDP redirect via devmap |
| BPF CPUMAP | Part II 4.15; Part III XDP | XDP redirect to remote CPU |
| nftables flowtable | Part II 4.16; Part III netfilter | software flowtable fast path; later HW-offload evolution |
| TCX / `bpf_mprog` | Part II 6.6; Part III BPF | link-based TC attachment and multi-program ordering |
| Device Memory TCP / memory providers | Part II 6.12/6.16; Part III memory | RX/TX device-memory zero-copy evolution |
| io_uring ZCRX | Part II 6.15; Part III io_uring | queue-bound zero-copy receive |
| `net-next-7.1` | Part II 7.1 | RX HW queue leasing |
| `net-next-7.3` | Part II 7.3-rc | tunnel BIG TCP, RTNL-less FIB rules, >PAGE_SIZE devmem buffers |

## Exact anchor index

Exact SHA の正本は Part VII に置きます。Appendix では同じcommit
listを複製しません。

## Conference provenance index

Conference talk は設計意図や当時のproblem
statementを補足する資料として扱い、 release
attributionには使用しません。

  Conference lineage                        Canonical destination
| Conference lineage | Canonical destination |
|---|---|
| Netdev: XDP / AF_XDP / TC / BPF | Part III programmability |
| Netdev: BIG TCP | Part III packet aggregation |
| Netdev: Device Memory TCP / zero-copy | Part III packet memory |
| Netdev 0x19: Diagnosing Page Pool Leaks | Part IV page_pool |
| Netdev: queue/NAPI/netdev-genl | Part IV driver framework |
| Netdev: MPTCP / TCP state-of-the-union | Part III transport |
| Kernel Recipes: XDP / BPF / io_uring | Parts III--IV |
| OVS/OVN Conf: Retis | Part V case-study note |
| LPC / FOSDEM: pwru and tracing | Part V case-study note |

## Retis / pwru case-study index

Retis と pwru
の詳細説明は本文から外しました。両者の位置づけは次の一文で十分です。

``` text
pwru  = broad kernel-function packet trajectory
Retis = networking-event-centric semantic enrichment / correlation
```

これらは BTF、eBPF tracing、drop reason、timestamp、OVS/OVN metadata
などの kernel primitiveを利用する **consumer/tooling examples**
であり、独立したkernel evolution axisではありません。

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
