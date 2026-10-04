# Linux Networking Evolution --- Linux v3.0 から 7.x まで

> **Clean canonical edition.** Part II is the only normative release
> chronology. Parts III--V explain long-term lineages; Part VII records
> unresolved provenance. Appendix material is supporting evidence, not
> an alternate release map.

**調査基準日:** 2026-10-02\
**構成改訂:** 2026-10-03

この文書は、Linux networking の変化を「調査した順」ではなく、 **kernel
networking がどのように進化したかを読む順序**に再構成した版である。

## この文書の読み方

> **Canonicalization note (2026-10-03):** Release chronology is
> authoritative only in **Part II**. Development-series dates and
> mainline release dates are intentionally separated. The document uses
> one six-axis architecture model. Appendix material is
> provenance/research history and is non-normative.

本文は次の流れで構成する。

``` text
Part I    v3.x → v5.1
          modern Linux networking の基礎形成

Part II   v5.2 → 7.x
          expansion / scale / memory ownership への発展

Part III  thematic lineages
          packet aggregation / memory / XDP / BPF /
          virtual networking / transport / control plane

Part IV   observability / explainability
          eBPF + BTF → skb_drop_reason → Retis / pwru

Part V    synthesis
          release間比較と全体architecture map

Appendix  LWN / upstream commits / conference provenance /
          completeness・verification audit
```

重要な方針は、release chronology と feature lineage を分離すること。
まず年代順に「何がいつ形成されたか」を読み、その後で同じ技術を
長期lineageとして横断的に追う。

------------------------------------------------------------------------

# Part I --- Foundations: Linux v3.x → v5.1

## Linux v3.x --- scalability, virtualization, programmability の誕生

``` text
3.0   setns() / namespace FD
 │
3.2   Byte Queue Limits (BQL)
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
3.10  networking scalability/offload continuation
 │
3.12  TCP pacing + FQ-era TCP scheduling
 │
3.13  nftables
 │
3.14–3.17
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
BQL → fq_codel → TSQ
```

でqueueを短くする仕組みが整った後、TCP packetを**いつ送るか**を制御する
pacing/FQ方向が発達する。

この流れは後に:

``` text
TCP pacing
  ↓
BBR
  ↓
time-based TX
  ↓
EDT
```

へつながる。

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

### 1.10 v3.14--v3.17 --- BPF transformation

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
BQL (3.2)
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
EDT (5.0)
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
TC direct access (4.7)
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

### 1.14 Revised generation model

v3.xを含めると、Linux networking
evolutionは4世代に分けると分かりやすい。

``` text
Linux 3.x — SCALABILITY + VIRTUALIZATION + PROGRAMMABILITY BIRTH
BQL / fq_codel / TSQ
VXLAN / SO_REUSEPORT / nftables
eBPF / DCTCP / ipvlan

Linux 4.x — PROGRAMMABLE FAST PATH
XDP / cgroup BPF / SOCK_OPS
BBR / kTLS / AF_XDP
VRF / LWT / devlink

Linux 5.x — EXPANSION
MPTCP / struct_ops / SK_LOOKUP
page_pool / BIG TCP

Linux 6.x–7.x — MEMORY + QUEUE OWNERSHIP
netmem / Device Memory TCP
io_uring ZCRX / netkit / queue leasing
per-netns RTNL
```

したがって、この資料の大きな結論は:

> **modern Linux networkingはv4.xから突然始まったのではない。v3.xで
> scalability、buffer management、overlay virtualization、firewall VM、
> eBPFという基礎が形成され、v4.xでそれらがprogrammable fast
> pathへ統合された。**

------------------------------------------------------------------------

## Linux v4.x --- programmable fast path の形成

**調査時点:** 2026-10-02

この版では v4.x を追加監査した。結論として modern Linux networking
は単一の release から始まったのではなく、v4.x の複数の milestone
で形成された。

### v4.x milestone map

``` text
4.3   VRF / lightweight tunnel / OVS conntrack
4.6   devlink / per-netns TCP knobs
4.7   TC BPF direct packet access / BPF tracepoints
4.8   XDP
4.9   BBR / NIC BPF offload
4.10  cgroup BPF / BPF LWT / IPv6 Segment Routing
4.12  generic XDP
4.13  BPF SOCK_OPS / kTLS TX
4.14  SOCKMAP
4.15  bpftool / netns-aware TCP / CBS qdisc
4.16  BPF-to-BPF calls / netdevsim
4.17  bind/connect/sendmsg BPF / kTLS RX / RDS zero-copy
4.18  AF_XDP / TCP zero-copy receive
4.19  time-based packet transmission / CAKE
5.0   EDT pacing / BPF flow dissector / taprio /
      UDP GRO / UDP MSG_ZEROCOPY
```

#### v4.3 --- cloud/virtual networking foundation

VRF、lightweight tunnel、OVS conntrack が同じ release に入った。

``` text
VRF → multiple routing domains
LWT → lightweight programmable encapsulation foundation
OVS + conntrack → switching + stateful flow processing
```

現在の Linux routing / OVN-OVS / cloud networking の重要な前史である。

#### v4.6 --- devlink and namespace scalability

devlink が導入され、netdevとは別にdevice-wide networking
resourceを管理する control plane が形成され始めた。多数のTCP/network
sysctlもnetns-awareになった。

#### v4.7 --- BPF becomes a practical datapath language

TC cls_bpf/act_bpf がpacket dataを直接参照可能になり、BPF tracepoint
attachmentも mainline化した。

#### v4.8 --- XDP

``` text
NIC → driver RX → XDP/BPF → skb allocation → normal stack
```

skb生成前のprogrammable hookという現在まで続く大きなarchitecture
change。

#### v4.9 --- BBR

delivery rateとRTpropを利用するmodel-based congestion
controlがmainline化。 後のpacing/EDT/BPF congestion-control
lineageの重要な節目。

#### v4.10 --- cgroup BPF / BPF LWT / SRv6

container identity、programmable policy、programmable
routing/tunnelが接続される。

#### v4.12 --- generic XDP

driver-native XDPを持たないdeviceにもXDP
semanticsを広げ、XDPを一般的なLinux networking APIへ近づけた。

#### v4.13 --- socket BPF and kTLS

BPF_PROG_TYPE_SOCK_OPSによりBPFがsocket lifecycleへ入り、kTLS
TXもmainline化。

``` text
packet programmability → socket/protocol programmability
```

#### v4.14--4.17 --- socket datapath and BPF ecosystem

SOCKMAP、bpftool、BPF-to-BPF calls、netdevsim、cgroup bind/connect
hooks、 sendmsg filtering、kTLS RXなどが続く。

#### v4.18 --- AF_XDP and TCP zero-copy RX

``` text
NIC queue → XDP → XSKMAP → AF_XDP → shared UMEM → userspace
```

AF_XDP multi-buffer、virtio-net AF_XDP、netkit queue
leasingへ続く直接的な祖先。 同時にTCP zero-copy receiveも入った。

#### v4.19 --- time-aware transmission

time-based packet transmissionとCAKEが入り、packet
schedulingは単純なqueue managementからtime-aware schedulingへ拡大した。

#### v4.20 --- EDT, BPF flow dissector, taprio, strict rtnetlink generation

4.20 is the landing release for TCP EDT pacing, BPF programmable flow
dissector, taprio and strict rtnetlink checking. Linux 4.20 was released
on 2018-12-23; it is not skipped in the kernel version sequence.

#### v5.0 --- foundationからbaselineへ

v5.0はmodern networkingの開始点ではなく、v4.xで形成された技術が成熟した
baselineと位置付ける。

### Long-term lineage

``` text
Programmability:
TC BPF → XDP → cgroup/LWT BPF → SOCK_OPS → SOCKMAP
→ AF_XDP → struct_ops → SK_LOOKUP → netkit → BPF qdisc

Cloud/routing:
VRF/LWT/OVS-CT → devlink → BPF LWT/SRv6
→ nexthop objects → YNL → OVN/Kubernetes

Zero-copy:
AF_XDP + TCP ZC RX → UDP MSG_ZEROCOPY
→ page_pool/netmem → io_uring ZCRX → Device Memory TCP

TCP:
BBR → time-based TX → EDT → BPF struct_ops CC
→ BIG TCP → AccECN
```

### Three-generation interpretation

``` text
Linux 4.x — FOUNDATIONS
XDP / BPF / VRF / LWT / devlink / BBR / kTLS / AF_XDP

Linux 5.x — EXPANSION
MPTCP / struct_ops / SK_LOOKUP / page_pool / BIG TCP

Linux 6.x–7.x — MEMORY + OWNERSHIP
netmem / Device Memory TCP / io_uring ZCRX /
netkit / queue leasing / per-netns RTNL
```

------------------------------------------------------------------------

## Linux v4.20--v5.1 --- modern baseline の形成

## Linux v4.20 / v5.0 を連続した baseline として扱う

Linux v5.0 時点ですでに
TCP/UDP、GRO/GSO、qdisc/TC、netfilter/conntrack、 rtnetlink、network
namespace、veth/bridge/tunnel、virtio-net/TAP/vhost、 XDP/eBPF/AF_XDP
という現在の主要構成要素は存在していた。

networking の観点で特に重要な v5.0 の節目は:

-   TCP pacing の EDT (Earliest Departure Time) model
-   UDP `MSG_ZEROCOPY`
-   UDP GRO
-   XDP / AF_XDP が既に利用可能

である。

``` text
Linux v5.0 baseline
│
├─ TCP / UDP / socket
│   ├─ EDT pacing
│   ├─ UDP MSG_ZEROCOPY
│   └─ UDP GRO
├─ skb + struct page
├─ GRO / GSO / TSO
├─ qdisc / TC
├─ netfilter / conntrack / nftables
├─ rtnetlink + global RTNL
├─ netns / veth / bridge / tunnel
├─ virtio-net / TAP / vhost
└─ XDP / eBPF / AF_XDP
```

ここから 2026 年までの data-plane evolution は概ね:

``` text
packet を速く処理する
        ↓
packet 数を減らす
        ↓
copy / allocation を減らす
        ↓
host RAM を経由しない
        ↓
queue / datapath を適切な consumer に委譲する
```

control plane では:

``` text
global / hand-written API
        ↓
machine-readable Netlink API
        ↓
per-netns / fine-grained locking
        ↓
large-scale container / VM networking
```

へ進んだ。

------------------------------------------------------------------------

## Linux v5.0 / v5.1 deep audit

この節は、当初の監査範囲（2019-05-07以降）より前に位置する Linux v5.0 と
v5.1 を、後続 release と同じ観点で追加監査した結果である。

### Linux v4.20 → v5.0 --- modern high-speed networking baseline

#### 1A.1 TCP EDT pacing --- Linux 4.20

v5.0 の重要な networking change の一つが TCP pacing の **Earliest
Departure Time (EDT)** model への移行である。

従来の考え方を単純化すると:

``` text
TCP
 ↓
packet queue
 ↓
qdisc pacing
```

EDT model では各 packet
に「いつ送信可能か」という時刻を持たせる方向になる。

``` text
TCP computes pacing schedule
        ↓
skb carries earliest departure time
        ↓
FQ/qdisc schedules transmission
```

この変更は後の BPF/TC based pacing、large-scale host networking、
container networking に重要な baseline となる。

**LWN:** `4.20/5.0 Merge window part 1`

#### 1A.2 BPF programmable flow dissector --- Linux 4.20

v5.0 では network flow dissector を BPF program
として実装できるようになった。

``` text
packet
  ↓
flow dissector
  ↓
flow keys
  ├─ hash
  ├─ classifier
  └─ policy
```

を:

``` text
packet
  ↓
BPF flow dissector
  ↓
custom flow keys / parsing policy
```

へ拡張する。

これは後の:

``` text
BPF packet parsing
 → socket lookup
 → protocol hooks
 → netkit/BPF qdisc
```

という programmable-network-stack lineage の初期段階として位置付ける。

#### 1A.3 rtnetlink strict checking --- Linux 4.20

rtnetlink に strict checking option が追加された。

これは YNL ほど大きな architecture change ではないが、

``` text
loosely validated Netlink messages
        ↓
stricter request validation
        ↓
better specified Netlink APIs
        ↓
YAML/YNL
```

という control-plane API modernization の前史として記録する。

#### 1A.4 UDP GRO --- Linux 5.0

plain UDP socket に GRO が導入された。

``` text
UDP datagram
UDP datagram
UDP datagram
     │
     ▼ GRO
larger receive aggregate
     │
     ▼
fewer receive operations / lower per-packet cost
```

これは後年の UDP receive optimization、QUIC/high-rate UDP、 tunnel
aggregation を理解する重要な baseline である。

#### 1A.5 UDP MSG_ZEROCOPY --- Linux 5.0

`MSG_ZEROCOPY` が UDP socket でも利用可能になった。

したがって copy-reduction lineage は:

``` text
v4.14 TCP MSG_ZEROCOPY → v5.0 UDP MSG_ZEROCOPY
        ↓
TCP/io_uring zero-copy work
        ↓
io_uring ZC TX
        ↓
io_uring ZCRX
        ↓
device-memory networking
```

と長い時間軸で見るべきである。

#### 1A.6 taprio --- Linux 4.20

Time-Aware Priority Scheduler (`taprio`) も v5.0 の重要な qdisc change。

これは TSN/time-aware networking の branch であり、BIG TCP や XDP とは
異なるが、TC/qdisc が単なる best-effort queue management から
time-sensitive scheduling へ広がった節目として残す。

#### v4.20 / v5.0 canonical summary

  Area        v5.0 change                   Later lineage
  ----------- ----------------------------- -------------------------------
  TCP         EDT pacing                    FQ/BPF pacing, scalable hosts
  UDP         GRO                           high-rate UDP, QUIC/tunnels
  zero-copy   UDP MSG_ZEROCOPY              io_uring ZC, device-memory
  BPF         programmable flow dissector   deeper stack programmability
  TC          taprio                        TSN/time-aware scheduling
  Netlink     strict checking               API formalization/YNL

------------------------------------------------------------------------

### Linux v5.1 --- observability, BPF state and the io_uring substrate

#### 1A.7 BPF spinlocks

BPF map values gained spinlock-based concurrency control.

``` text
multiple BPF programs / CPUs / userspace
             ↓
          BPF map
             ↓
      shared mutable state
             ↓
        bpf_spin_lock
```

Networking BPF programs increasingly maintain flow/state information, so
this is an important enabling primitive even though it is not
network-specific.

#### 1A.8 BPF verifier dead-code elimination

The verifier gained dead-code detection/removal. This belongs to the BPF
execution infrastructure lineage that made increasingly complex
networking programs practical.

#### 1A.9 SO_BINDTOIFINDEX

`SO_BINDTOIFINDEX` provides interface binding by ifindex rather than
interface name.

``` text
SO_BINDTODEVICE → name based
SO_BINDTOIFINDEX → stable kernel interface identifier
```

It is a relatively small socket API change, but useful in
namespace-heavy and programmatic network management environments.

#### 1A.10 Y2038-safe socket timestamps

socket timestamp APIs gained Y2038-safe variants.

This is primarily ABI maintenance, but timestamping is fundamental to
packet capture, latency measurement, pacing and observability, so it
belongs in the networking API history.

#### 1A.11 devlink health

devlink gained a health-reporting mechanism for network devices.

This marks a broader evolution:

``` text
driver-specific diagnostics
       ↓
devlink health reporter
       ↓
standardized device health / recovery / observability
```

and later devlink became a major NIC/switch management interface.

#### 1A.12 Wi-Fi airtime fairness chronology --- correction

The previous draft incorrectly used Linux 5.1 as the airtime-fairness
milestone. mac80211 airtime-fairness work predates that release (4.17
generation), while AQL is a later 5.5-generation milestone. 5.1 is
therefore not used as the canonical origin here.

mac80211 gained airtime-aware fairness support. Unlike byte/packet
fairness, wireless capacity is fundamentally constrained by airtime:

``` text
equal packets/bytes ≠ equal radio resource

fairness unit → airtime
```

This is an important networking scheduler concept even though it is
Wi-Fi-specific.

#### 1A.13 io_uring appears

Linux v5.1 introduced io_uring.

At this point it was not yet the zero-copy networking mechanism seen in
later kernels, but it is the substrate from which the later lineage
grows:

``` text
5.1 io_uring
   ↓
sendmsg / recvmsg support
   ↓
multishot / registered buffers
   ↓
zero-copy TX
   ↓
6.15 zero-copy RX
   ↓
memory-provider / device-memory integration
```

Therefore the networking evolution timeline should mark **5.1 as the
origin of the io_uring branch**, while distinguishing that from the
later networking-specific features.

#### v5.1 canonical summary

  ----------------------------------------------------------------------
  Area                  v5.1 change           Later lineage
  --------------------- --------------------- --------------------------
  BPF                   spinlocks             stateful/concurrent BPF
                                              networking

  BPF                   verifier dead-code    larger/more sophisticated
                        elimination           programs

  socket API            SO_BINDTOIFINDEX      programmatic/netns-aware
                                              socket control

  timestamping          Y2038-safe APIs       long-lived timestamp ABI

  device management     devlink health        standardized NIC
                                              health/recovery

  Wi-Fi                 airtime fairness      airtime-aware scheduling

  async I/O             io_uring introduced   networking ZC TX/RX
  ----------------------------------------------------------------------

------------------------------------------------------------------------

### 1A.14 Corrected starting graph

v5.0/v5.1 を追加すると、この文書の evolution graph
は次のように補正できる。

``` text
Linux v5.0
│
├─ EDT TCP pacing ─────────────→ FQ / BPF pacing / scalable TCP
│
├─ UDP GRO ────────────────────→ high-rate UDP / QUIC / tunnels
│
├─ UDP MSG_ZEROCOPY ───────────→ zero-copy networking
│                                      │
│                                      ├→ io_uring ZC TX/RX
│                                      └→ device-memory networking
│
├─ BPF flow dissector
│       ↓
│   socket/cgroup hooks
│       ↓
│   struct_ops / SK_LOOKUP
│       ↓
│   netkit / BPF qdisc
│
└─ rtnetlink strict checking ──→ API formalization → YNL

Linux v5.1
│
├─ BPF spinlocks ──────────────→ stateful/concurrent BPF programs
├─ devlink health ─────────────→ modern NIC health/management
└─ io_uring ───────────────────→ networking async/ZC branch
```

これにより「v5.0 は単なる開始番号」という扱いではなく、 **現在の Linux
networking の複数の主要 lineage がすでに分岐し始めていた 技術的
baseline** として扱える。

------------------------------------------------------------------------

## v3.x / v4.x provenance re-audit --- LWN + upstream commit anchors

> **Audit policy:** v5以降と同じく、(1)
> LWNによるrelease/architecture確認、(2) patch series、 (3) mainline
> introduction commit、(4)
> 後続の`Fixes:`参照による独立確認、を分離する。 exact
> SHAを今回確認できなかった項目は推測せず`pending exact-SHA audit`とする。

### 4.1 Linux v3.x

  -----------------------------------------------------------------------------------------------------------------------------------------------------------
  Kernel       Feature        LWN / upstream evidence                             Mainline anchor                                      Audit
  ------------ -------------- --------------------------------------------------- ---------------------------------------------------- ----------------------
  3.2          Byte Queue     [BQL v3 series](https://lwn.net/Articles/469652/),  exact SHA: pending                                   release/series
               Limits         [DQL 1/10](https://lwn.net/Articles/469651/)                                                             verified
               (BQL/DQL)                                                                                                               

  3.5          CoDel /        [iproute2 3.5.0:                                    exact SHA: pending                                   release/tooling
               fq_codel       codel/fq_codel](https://lwn.net/Articles/509446/)                                                        verified; commit audit
                                                                                                                                       pending

  3.6          TCP Small      [3.6 merge                                          exact SHA: pending                                   merge verified
               Queues         window](https://lwn.net/Articles/507852/)                                                                

  3.6          TCP Fast Open  [3.6 merge                                          exact SHA: pending                                   merge verified
               client / IPv4  window](https://lwn.net/Articles/507852/)                                                                
               route-cache                                                                                                             
               removal                                                                                                                 

  3.7          VXLAN          [3.7 merge                                          `d342894c5d2f8c7df194c793ec4059656e09ca31`           **A**
                              window](https://lwn.net/Articles/518275/)                                                                

  3.7          TCP Fast Open  [3.7 merge                                          exact SHA: pending                                   merge verified
               server / IPv6  window](https://lwn.net/Articles/518275/)                                                                
               NAT                                                                                                                     

  3.13         nftables       [3.13 merge                                         exact SHA: pending                                   merge verified
                              window](https://lwn.net/Articles/573272/)                                                                

  3.14--3.17   BPF core       [split BPF out of                                   `f5bffecda951b59d0d3cdd616d68952abc52bc40` is one    **B** (multi-commit
               separation /   networking](https://lwn.net/Articles/600989/), [BPF verified core-separation anchor                      evolution)
               eBPF VM        tracing filters](https://lwn.net/Articles/575531/)                                                       
               evolution                                                                                                               

  3.18         `bpf()`        [RFC series](https://lwn.net/Articles/603816/), [A  full final-series SHA enumeration: pending           landing/release
               syscall / maps reworked BPF API](https://lwn.net/Articles/606089/)                                                      verified, exact set
               / verifier                                                                                                              pending

  3.18         DCTCP          [DCTCP v3                                           `e3118e8359bb7c59555aca60c725106e6d78c5ce` (DCTCP    **A**
                              series](https://lwn.net/Articles/614000/), [3.18    algorithm)                                           
                              merge window](https://lwn.net/Articles/615825/)                                                          

  3.18         Geneve /       [3.18 merge                                         exact SHA: pending                                   merge verified
               Foo-over-UDP   window](https://lwn.net/Articles/615825/)                                                                

  3.19         eBPF socket    [Attaching eBPF programs to                         exact SHA: pending                                   release/architecture
               attachment     sockets](https://lwn.net/Articles/625224/), [3.19                                                        verified
                              merge window](https://lwn.net/Articles/626150/)                                                          

  3.19         ipvlan         [initial ipvlan                                     `2ad7bf363841`                                       **A**
                              patch](https://lwn.net/Articles/620087/), [3.19     (`ipvlan: Initial check-in of the IPVLAN driver.`)   
                              merge window](https://lwn.net/Articles/626150/)                                                          
  -----------------------------------------------------------------------------------------------------------------------------------------------------------

#### VXLAN independent `Fixes:` confirmation

The original VXLAN commit is not inferred from an old release summary
only. Later fixes explicitly carry:

``` text
Fixes: d342894c5d2f ("vxlan: virtual extensible lan")
```

This independently anchors the original implementation to
`d342894c5d2f8c7df194c793ec4059656e09ca31`.

#### ipvlan independent `Fixes:` confirmation

Later ipvlan fixes repeatedly reference:

``` text
Fixes: 2ad7bf363841 ("ipvlan: Initial check-in of the IPVLAN driver.")
```

Therefore the initial v3.19 ipvlan anchor is considered strong.

#### DCTCP independent `Fixes:` confirmation

The v3.18 DCTCP algorithm commit is:

``` text
e3118e8359bb7c59555aca60c725106e6d78c5ce
net: tcp: add DCTCP congestion control algorithm
```

Later fixes explicitly use `Fixes: e3118e8359bb`, providing independent
confirmation.

### 4.2 Important correction to the eBPF narrative

The v3.x BPF history should not be written as:

``` text
3.18: eBPF appeared
```

A more accurate lineage is:

``` text
classic BPF
   ↓
BPF VM redesign / tracing-oriented extension
   ↓
generic BPF core separation
   ↓
3.18: bpf() syscall + maps + verifier + userspace loading API
   ↓
3.19: attach eBPF programs to sockets
   ↓
4.x: TC/XDP/cgroup/LWT/socket programmability
```

LWN's 2014 articles explicitly show that the VM/API redesign and
subsystem separation were already underway before the 3.18 userspace API
landed.

### 4.3 Linux v4.x

  ------------------------------------------------------------------------------------------------------------------------------
  Kernel   Feature        LWN evidence                                  Mainline anchor                              Audit
  -------- -------------- --------------------------------------------- -------------------------------------------- -----------
  4.3      VRF /          [networking                                   exact per-feature SHA: pending               merge
           lightweight    pull](https://lwn.net/Articles/657074/), [4.3                                              verified
           tunnels / OVS  merge                                                                                      
           conntrack      window](https://lwn.net/Articles/656731/)                                                  

  4.6      devlink /      [4.6 merge                                    exact SHA: pending                           merge
           per-netns TCP  window](https://lwn.net/Articles/680566/)                                                  verified
           knobs                                                                                                     

  4.7      TC BPF direct  LWN/upstream exact mapping: audit pending     exact SHA: pending                           **C until
           packet access                                                                                             exact
           / BPF tracing                                                                                             mapping**
           expansion                                                                                                 

  4.8      XDP initial    release/patch lineage audit still required    exact SHA set: pending                       **B/C**
           mainline                                                                                                  
           generation                                                                                                

  4.9      BBR / BPF      exact LWN + commit mapping still required     exact SHA: pending                           **C until
           NIC-offload                                                                                               audited**
           generation                                                                                                

  4.10     cgroup BPF /   exact per-feature audit required              exact SHA: pending                           **C until
           BPF LWT / IPv6                                                                                            audited**
           Segment                                                                                                   
           Routing                                                                                                   

  4.13     BPF `SOCK_OPS` [4.13 merge                                   exact SHA: pending                           merge
           / kTLS TX      window](https://lwn.net/Articles/727385/)                                                  verified

  4.14     `SOCKMAP`      kernel docs confirm `BPF_MAP_TYPE_SOCKMAP`    exact SHA: pending                           release
                          introduced in 4.14                                                                         verified

  4.18     AF_XDP         [initial AF_XDP                               final exact series SHA enumeration: pending  merge
                          series](https://lwn.net/Articles/752546/),                                                 verified
                          [4.18 merge                                                                                
                          window](https://lwn.net/Articles/756898/)                                                  

  4.18     TCP zero-copy  [initial                                      exact SHA: pending                           merge/API
           receive        article](https://lwn.net/Articles/752188/),                                                evolution
                          [reworked                                                                                  verified
                          API](https://lwn.net/Articles/754681/), [4.18                                              
                          merge                                                                                      
                          window](https://lwn.net/Articles/756898/)                                                  

  4.19     time-based     [RFC v3](https://lwn.net/Articles/748744/),   exact final SHA set: pending                 merge
           packet         [4.19 merge                                                                                verified
           transmission   window](https://lwn.net/Articles/762566/)                                                  

  4.19     CAKE           [CAKE patch                                   `046f6fd5daefac7f5abdafb436b30f63bc7c602b`   **A**
                          series](https://lwn.net/Articles/752777/),                                                 
                          [4.19 merge                                                                                
                          window](https://lwn.net/Articles/762566/)                                                  
  ------------------------------------------------------------------------------------------------------------------------------

#### CAKE independent `Fixes:` confirmation

The original CAKE commit is:

``` text
046f6fd5daefac7f5abdafb436b30f63bc7c602b
sched: Add Common Applications Kept Enhanced (cake) qdisc
```

Later fixes explicitly reference `Fixes: 046f6fd5daef`, making this a
strong introduction anchor.

### 4.4 Audit status and next exact-SHA pass

This pass deliberately separates three states:

``` text
A  release + LWN/series + exact mainline SHA + independent later evidence
B  release/landing is strong, but the feature spans multiple commits or
   exact final-series enumeration is incomplete
C  release-level statement is plausible/known, but LWN ↔ final mainline
   mapping still needs exact verification
```

After this pass, representative **A-grade** pre-v5 anchors include:

``` text
Linux 3.7   VXLAN
  d342894c5d2f8c7df194c793ec4059656e09ca31

Linux 3.18  DCTCP algorithm
  e3118e8359bb7c59555aca60c725106e6d78c5ce

Linux 3.19  ipvlan
  2ad7bf363841...

Linux 4.19  CAKE
  046f6fd5daefac7f5abdafb436b30f63bc7c602b
```

The next pass should prioritize exact SHA enumeration for:

``` text
BQL/DQL
CoDel/fq_codel
TCP Small Queues
TCP Fast Open
IPv4 route-cache removal
nftables
bpf() syscall/maps/verifier
Geneve/Fou
SO_ATTACH_BPF
VRF/LWT/OVS conntrack
devlink
TC direct-access BPF
XDP
BBR
cgroup/LWT BPF
SOCK_OPS/kTLS
SOCKMAP
AF_XDP
TCP zero-copy receive
SO_TXTIME / ETF
```

This list is intentionally explicit so that no release-level statement
is silently promoted to exact provenance without verification.

------------------------------------------------------------------------

## v4.x provenance re-audit pass 2 --- programmable fast path

This pass focuses on v4.7--v4.19, where modern programmable networking
became a coherent architecture.

### 5.1 Linux 4.7 --- TC BPF direct packet access

LWN's 4.7 merge-window summary explicitly records that `cls_bpf` and
`act_bpf` gained direct packet access through `skb->data` /
`skb->data_end`, replacing special packet-load helpers in this path.

-   LWN: https://lwn.net/Articles/686943/
-   Current verifier documentation preserves the same
    direct-packet-access model.

``` text
helper-based packet loads
 → TC direct packet access
 → verifier range tracking
 → XDP direct packet model
```

**Quality B:** release/semantics verified; exact final applied SHA still
pending.

### 5.2 Linux 4.8 --- first-generation XDP

Late review lineage contains
`[PATCH v8 01/11] bpf: add XDP prog type for early driver filter`,
introducing `BPF_PROG_TYPE_XDP`, `struct xdp_md`, packet start/end
pointers and XDP actions.

-   v6: https://lists.openwall.net/netdev/2016/07/08/8
-   v8: https://lists.openwall.net/netdev/2016/07/12/36
-   LWN architecture retrospective: https://lwn.net/Articles/707844/

Do not turn a review Message-ID into a mainline SHA. **Quality B** until
canonical landing SHA enumeration is complete.

### 5.3 Linux 4.9 --- BBR

LWN patch/release provenance: - https://lwn.net/Articles/701149/ -
https://lwn.net/Articles/701165/ - https://lwn.net/Articles/703110/

Final v4 series: https://lists.openwall.net/netdev/2016/09/20/50

The series first adds supporting delivery-rate/pacing/congestion-control
infrastructure and finally the BBR module.

``` text
0f8782ea14974ce992618b55f0c041ef43ed0b78
tcp_bbr: add BBR congestion control
```

Later fixes and 2026 upstream work independently identify `0f8782ea1497`
as the BBR introduction commit.

**Quality A.**

### 5.4 Linux 4.10 --- cgroup BPF, BPF LWT, IPv6 SRv6

LWN 4.10 merge window explicitly lists all three:
https://lwn.net/Articles/709017/

cgroup-BPF ABI discussion: https://lwn.net/Articles/711234/

BPF LWT series: https://lwn.net/Articles/705609/

The LWT series attaches BPF to `lwtunnel_input()`, `lwtunnel_output()`
and `lwtunnel_xmit()`, with later revisions introducing
`BPF_PROG_TYPE_LWT_IN`, `BPF_PROG_TYPE_LWT_OUT` and
`LWTUNNEL_ENCAP_BPF`.

``` text
packet classifier BPF
 → route/dst-entry BPF
 → programmable routing/encapsulation
 → later SRv6/BPF route behaviors
```

**Quality B** for each until final multi-commit SHA sets are enumerated.

### 5.5 Linux 4.13 --- SOCK_OPS

LWN archives the v5 net-next series: https://lwn.net/Articles/727189/

It introduces `BPF_PROG_TYPE_SOCK_OPS` / `struct bpf_sock_ops` and uses
cgroup-BPF attachment.

``` text
packet BPF → cgroup networking → SOCK_OPS
 → TCP connection-parameter programmability
 → later struct_ops/TCP-CC programmability
```

**Quality B:** final-series/release provenance strong; exact commit
enumeration pending.

### 5.6 Linux 4.14 --- SOCKMAP

Kernel documentation explicitly records: `BPF_MAP_TYPE_SOCKMAP`
introduced in Linux 4.14; `SOCKHASH` in 4.18.

https://static.lwn.net/kerneldoc/bpf/map_sockmap.html

**Quality B:** release documented; exact introduction SHA pending.

### 5.7 Linux 4.18 --- AF_XDP

Review lineage: - RFC: https://lwn.net/Articles/745934/ - v2:
https://lwn.net/Articles/750293/ - merge-near:
https://lwn.net/Articles/752546/ - 4.18 merge window:
https://lwn.net/Articles/756898/

Strong foundational anchor:

``` text
c0c77d8fb787cfe0c3fca689c2a30d1dad4eaba7
xsk: add user memory registration support sockopt
```

The commit explicitly says it sets up the base structure of AF_XDP.
Multiple later fixes use `Fixes: c0c77d8fb787`.

AF_XDP is nevertheless a series; this commit is a foundational anchor,
not the entire feature.

**Quality A for anchor / B for full series.**

### 5.8 Linux 4.18 --- TCP zero-copy receive

LWN's 4.18 merge window explicitly confirms TCP zero-copy receive.
Existing initial/reworked API references should be kept as an evolution
rather than collapsed to one proposal.

**Quality B:** landing/release verified; exact final SHA set pending.

### 5.9 Linux 4.19 --- time-based packet transmission

LWN RFC v3: https://lwn.net/Articles/748744/

It describes `SO_TXTIME`, time-based qdisc, hardware offload and
software fallback. LWN 4.19 merge window confirms the series merged:
https://lwn.net/Articles/762566/

Recovered socket-option anchor:

``` text
80b14dee2bea...
net: Add a new socket option for a future transmit time
```

This is one anchor in a multi-commit feature.

**Quality B/A-anchor; full series enumeration pending.**

### 5.10 Linux 4.19 --- CAKE

Pass-1 anchor remains:

``` text
046f6fd5daefac7f5abdafb436b30f63bc7c602b
sched: Add Common Applications Kept Enhanced (cake) qdisc
```

LWN confirms the 4.19 merge; later fixes independently reference the
introduction commit.

**Quality A.**

### 5.11 Revised v4.x lineage

``` text
4.7  TC BPF direct packet access
 ↓
4.8  XDP
 ↓
4.9  BBR + BPF HW-offload generation
 ↓
4.10 cgroup BPF + BPF LWT + IPv6 SRv6
 ↓
4.13 SOCK_OPS
 ↓
4.14 SOCKMAP
 ↓
4.18 AF_XDP + TCP zero-copy receive
 ↓
4.19 time-based TX + CAKE
 ↓
5.x+ struct_ops / SK_LOOKUP / netkit /
     Device Memory TCP / io_uring ZCRX
```

Linux 4.x is therefore best described as the **programmable fast-path
formation era**.

### 5.12 Quality update after pass 2

  Feature                            Release Quality
  -------------------------------- --------- ---------
  TC BPF direct packet access            4.7 B
  XDP first generation                   4.8 B
  BBR                                    4.9 A
  cgroup BPF ingress/egress             4.10 B
  BPF LWT                               4.10 B
  IPv6 Segment Routing                  4.10 B
  SOCK_OPS                              4.13 B
  SOCKMAP                               4.14 B
  AF_XDP foundational anchor            4.18 A
  AF_XDP full series                    4.18 B
  TCP zero-copy receive                 4.18 B
  time-based packet transmission        4.19 B
  CAKE                                  4.19 A

Remaining work is primarily exact final-series commit enumeration, not
release identification.

------------------------------------------------------------------------

## v3.x / v4.x exact-anchor pass 3

This pass upgrades previously release-only or series-only entries where
an exact mainline introduction anchor can be independently verified.

### 6.1 Linux 3.2 --- Dynamic Queue Limits / BQL foundation

The core DQL implementation is:

``` text
75957ba36c05b979701e9ec64b37819adc12f830
dql: Dynamic queue limits
```

This is the reusable queue-limit library underneath BQL. The original
LWN series remains:

-   https://lwn.net/Articles/469651/
-   https://lwn.net/Articles/469652/

Important distinction:

``` text
DQL = generic dynamic queue-limit algorithm/library
BQL = networking driver use of DQL through netdev TX queue accounting
```

Thus `75957ba...` is a strong DQL foundation anchor, not a claim that
one commit alone converted every NIC driver to BQL.

**Quality A for DQL core / B for complete BQL rollout.**

### 6.2 Linux 3.5 --- CoDel and fq_codel

Exact introduction anchors:

``` text
76e3cc126bb223013a6b9a0e2a51238d1ef2e409
codel: Controlled Delay AQM

4b549a2ef4bef9965d97cbd992ba67930cd3e0fe
fq_codel: Fair Queue Codel AQM
```

LWN preserves the late CoDel patch: https://lwn.net/Articles/496502/

A 2025 upstream fix independently references both original commits:

``` text
Fixes: 4b549a2ef4be ("fq_codel: Fair Queue Codel AQM")
Fixes: 76e3cc126bb2 ("codel: Controlled Delay AQM")
```

This is unusually strong provenance.

**Quality A.**

### 6.3 Linux 3.6 --- TCP Small Queues

Exact introduction:

``` text
46d3ceabd8d98ed0ad10f20c595ca784e34786c5
tcp: TCP Small Queues
```

LWN: https://lwn.net/Articles/506237/

The commit and review text explicitly state the design goal: reduce TCP
packets queued in qdisc/device queues, reducing RTT and cwnd bias caused
by bufferbloat.

**Quality A.**

### 6.4 Linux 3.6 --- TCP Fast Open client

The client feature is a series. One strong userspace/API anchor is:

``` text
cf60af03ca4e71134206809ea892e49b92a88896
net-tcp: Fast Open client - sendmsg(MSG_FASTOPEN)
```

The patch makes `MSG_FASTOPEN` a combined connect+write operation and
documents the `tcp_fastopen` client bit.

The original v3 series is visible on netdev:
https://lists.openwall.net/netdev/2012/07/19/99

A 2026 security fix independently references:

``` text
Fixes: cf60af03ca4e ("net-tcp: Fast Open client - sendmsg(MSG_FASTOPEN)")
```

This provides particularly strong long-term confirmation.

**Quality A for the client API anchor / B for full TFO series.**

### 6.5 Linux 3.13 --- nftables

The initial nftables core introduction is anchored by:

``` text
96518518...
netfilter: add nftables
```

The commit describes nftables as the intended successor to iptables and
introduces the register-based pseudo-machine/expression framework.

A neighboring foundational set-API commit is:

``` text
20a69341f2d00cd042e81c82289fba8a13c05a25
netfilter: nf_tables: add netlink set API
```

The initial feature is clearly multi-commit, so `96518518...` is the
core introduction anchor, not the entire nftables implementation.

**Quality A for core anchor / B for full initial series.**

### 6.6 Linux 3.18 --- bpf() syscall, maps, program load and verifier

This history can now be represented with exact anchors rather than one
generic "eBPF" entry.

``` text
99c55f7d47c0dc6fc64729f37bf435abf43f4c60
bpf: introduce BPF syscall and maps
```

This commit introduces the multiplexed BPF syscall and `BPF_MAP_CREATE`.

Then:

``` text
09756af46893c18839062976c3252e93a1beeba7
bpf: expand BPF syscall with program load/unload
```

The latter explicitly describes verifier-based safety checking for
loaded eBPF programs.

LWN RFC: https://lwn.net/Articles/603816/

The original author later identified `99c55f7d47c0` as the BPF syscall
introduction point when marking BPF's seventh birthday.

The corrected lineage is therefore:

``` text
BPF VM redesign / generic core
       ↓
99c55f7d...
bpf() syscall + maps
       ↓
09756af...
program load + verifier-facing API
       ↓
3.19 socket attachment
       ↓
4.x TC/XDP/cgroup/LWT/SOCK_OPS
```

**Quality A for these two anchors.**

### 6.7 Linux 4.18 --- AF_XDP independent confirmation

The pass-2 foundational anchor is now independently confirmed:

``` text
c0c77d8fb787cfe0c3fca689c2a30d1dad4eaba7
xsk: add user memory registration support sockopt
```

Its commit message states that it sets up the base structure of the
AF_XDP address family. A subsequent 2018 fix carries:

``` text
Fixes: c0c77d8fb787 ("xsk: add user memory registration support sockopt")
```

This upgrades confidence in the anchor itself to **A** while retaining
**B** for complete AF_XDP series enumeration.

### 6.8 Updated exact-anchor table

  ------------------------------------------------------------------------------------------------------
  Kernel   Feature         Exact mainline anchor                        Scope                 Quality
  -------- --------------- -------------------------------------------- --------------------- ----------
  3.2      DQL/BQL         `75957ba36c05b979701e9ec64b37819adc12f830`   DQL core              A
           foundation                                                                         

  3.5      CoDel           `76e3cc126bb223013a6b9a0e2a51238d1ef2e409`   qdisc/core algorithm  A

  3.5      fq_codel        `4b549a2ef4bef9965d97cbd992ba67930cd3e0fe`   fq_codel qdisc        A

  3.6      TCP Small       `46d3ceabd8d98ed0ad10f20c595ca784e34786c5`   TSQ introduction      A
           Queues                                                                             

  3.6      TCP Fast Open   `cf60af03ca4e71134206809ea892e49b92a88896`   MSG_FASTOPEN client   A
           client                                                       API anchor            

  3.7      VXLAN           `d342894c5d2f8c7df194c793ec4059656e09ca31`   initial VXLAN         A

  3.13     nftables        `96518518...`                                core introduction     A-anchor

  3.13     nftables sets   `20a69341f2d00cd042e81c82289fba8a13c05a25`   netlink set API       A

  3.18     bpf() + maps    `99c55f7d47c0dc6fc64729f37bf435abf43f4c60`   syscall/maps          A

  3.18     BPF program     `09756af46893c18839062976c3252e93a1beeba7`   program load          A
           load/verifier                                                                      
           API                                                                                

  3.18     DCTCP           `e3118e8359bb7c59555aca60c725106e6d78c5ce`   CC algorithm          A

  3.19     ipvlan          `2ad7bf363841...`                            initial driver        A

  4.9      BBR             `0f8782ea14974ce992618b55f0c041ef43ed0b78`   BBR algorithm         A

  4.18     AF_XDP          `c0c77d8fb787cfe0c3fca689c2a30d1dad4eaba7`   foundational          A
                                                                        UMEM/address-family   
                                                                        anchor                

  4.19     CAKE            `046f6fd5daefac7f5abdafb436b30f63bc7c602b`   qdisc introduction    A
  ------------------------------------------------------------------------------------------------------

### 6.9 What remains intentionally unresolved

Exact SHA work should continue only where it adds provenance value.
Remaining high-value targets are:

``` text
IPv4 route-cache removal
TFO server side
nftables complete initial series
SO_ATTACH_BPF
VRF
LWT core
OVS conntrack
devlink
TC direct packet access
XDP complete initial series
cgroup BPF complete series
BPF LWT complete series
SOCK_OPS
SOCKMAP
TCP zero-copy receive
SO_TXTIME/ETF complete series
```

For these, release assignment is already strong. The remaining task is
exact **series boundary** identification, not rediscovering which kernel
release contained the feature.

------------------------------------------------------------------------

## v4.x provenance re-audit pass 4 --- series boundaries and design changes

This pass closes the remaining high-value gaps by preserving
multi-commit and API-rework history.

### 7.1 Linux 4.3 --- VRF, lightweight tunnels, OVS conntrack

David Miller's networking pull for 4.3 explicitly groups these three
major changes:

-   OVS conntrack support
-   initial VRF support
-   lightweight tunnel infrastructure

LWN: https://lwn.net/Articles/657074/

The lightweight-tunnel work is itself a 22-patch series:
https://lwn.net/Articles/651497/

The cover letter explains that it consolidates OVS/native tunnel
infrastructure and adds encapsulation-independent, flow-based
lightweight tunnels.

OVS conntrack is likewise a real series rather than one isolated patch:
https://lwn.net/Articles/652967/

The initial series contains the CT action plus
state/zone/mark/label/helper integration. Later revisions reached at
least v6 before merge.

Correct representation:

``` text
Linux 4.3
 ├─ VRF initial foundation
 ├─ LWT infrastructure (multi-patch)
 └─ OVS conntrack (multi-patch)
      ├─ CT action
      ├─ ct_state / zone
      ├─ ct_mark
      ├─ ct_label
      └─ helpers
```

**Quality A for release/series provenance; B for complete per-patch SHA
enumeration.**

### 7.2 Linux 4.7 --- TC direct packet access

The verifier documentation and 4.7 merge evidence agree on the semantic
boundary: `cls_bpf` / `act_bpf` programs gained direct access through
`skb->data` and `skb->data_end`.

This is more important historically than forcing one "TC-BPF commit"
label:

``` text
old:
  helper-mediated packet loads

4.7:
  skb->data / skb->data_end
  + verifier bounds proof

4.8:
  same basic verifier model applied to xdp_md packet pointers
```

**Quality A for the architectural/release statement; exact patch SHA
remains B-level provenance.**

### 7.3 Linux 4.8 --- XDP initial series boundary

The late XDP review history is now pinned to v8:

https://lists.openwall.net/netdev/2016/07/12/36

Patch 01/11 adds:

``` text
BPF_PROG_TYPE_XDP
struct xdp_md
packet start/end direct access
XDP action return model
```

The series then wires this model into early driver receive paths.

The historical unit should therefore be:

``` text
XDP core program model
       +
driver integration
       =
first-generation XDP
```

rather than treating `BPF_PROG_TYPE_XDP` alone as all of XDP.

**Quality A for final-review-series identity / B for canonical SHA
enumeration.**

### 7.4 Linux 4.10 --- cgroup BPF is a family, not one hook

The 4.10 generation should distinguish:

``` text
BPF_PROG_TYPE_CGROUP_SKB
  ingress / egress packet hooks

BPF_PROG_TYPE_CGROUP_SOCK
  socket-create context
```

A strong exact socket-side anchor is:

``` text
610236587600...
bpf: Add new cgroup attach type to enable sock modifications
```

The final v7 review patch is:
https://lists.openwall.net/netdev/2016/12/01/166

The commit explicitly says `BPF_PROG_TYPE_CGROUP_SOCK` is similar to
`BPF_PROG_TYPE_CGROUP_SKB`, but runs when a process in the cgroup opens
an AF_INET/AF_INET6 socket.

This corrects an overly compressed "4.10 cgroup BPF" label into multiple
attachment contexts.

### 7.5 Linux 4.10 --- BPF LWT series boundary

The LWT-BPF series: https://lwn.net/Articles/705609/

establishes BPF execution at lightweight-tunnel input/output/xmit paths.

The correct provenance unit is the series because it includes both:

``` text
BPF program types / verifier context
        +
LWT route attachment and execution
```

This is a direct ancestor of later programmable route encapsulation and
SRv6/BPF work.

**Quality A for review/landing identity / B for exact SHA set.**

### 7.6 Linux 4.13 / 4.14 --- SOCK_OPS → SOCKMAP

These should be read as two consecutive architectural steps:

``` text
4.13 SOCK_OPS
  observe/control TCP connection events and parameters
        ↓
4.14 SOCKMAP
  store sockets in a BPF map and redirect data between sockets
```

SOCK_OPS final-series evidence remains: https://lwn.net/Articles/727189/

Kernel documentation independently states SOCKMAP was introduced in
4.14: https://static.lwn.net/kerneldoc/bpf/map_sockmap.html

This sequence is the immediate foundation for later SK_MSG and
socket-level BPF data paths.

### 7.7 Linux 4.18 --- AF_XDP is two related landing stories

The history should explicitly separate address-family introduction from
zero-copy enablement.

AF_XDP address-family review lineage:

-   RFC: https://lwn.net/Articles/745934/
-   RFC v2: https://lwn.net/Articles/750293/
-   15-patch merge-near series: https://lwn.net/Articles/752546/

The v2/merge-near series deliberately removed zero-copy code to make the
AF_XDP socket model reviewable first.

Then zero-copy support followed as a separate series:

-   RFC 12 patches: https://lwn.net/Articles/754659/
-   11-patch series: https://lwn.net/Articles/756549/

The 4.18 merge-window confirms AF_XDP landed:
https://lwn.net/Articles/756898/

Therefore:

``` text
AF_XDP socket model
 RX/TX rings + UMEM + XSKMAP
          ↓
separate ZC series
 driver queue / DMA integration
          ↓
AF_XDP zero-copy fast path
```

This is a more accurate lineage than saying "4.18 introduced AF_XDP
zero-copy" as if it were one atomic patch.

### 7.8 Linux 4.18 --- TCP zero-copy receive API changed before release

This feature has an important API-design correction that should remain
visible.

Initial model: https://lwn.net/Articles/752188/

The first design used `mmap()` itself to consume/map socket data.
Locking/API concerns led to a rework.

v3 rework series: https://lwn.net/Articles/752938/

Final model:

``` text
mmap()
  reserve/setup userspace mapping
        +
getsockopt(TCP_ZEROCOPY_RECEIVE)
  request/consume TCP data into mapping
```

LWN's detailed rework article: https://lwn.net/Articles/754681/

And 4.18 merge-window confirmation: https://lwn.net/Articles/756898/

This is exactly the kind of case where preserving RFC → API rework →
landing is more informative than recording only a final SHA.

**Quality A for design/landing provenance.**

### 7.9 Linux 4.19 --- SO_TXTIME / time-based transmission is a series

The early RFC v3 includes 18 patches and introduces `SO_TXTIME`:
https://lists.openwall.net/netdev/2018/03/07/24

A later v2 net-next series has 14 patches; its socket-option patch is:
https://lists.openwall.net/netdev/2018/07/03/136

The feature spans:

``` text
SO_TXTIME userspace API
      ↓
SCM_TXTIME ancillary data
      ↓
skb transmit timestamp
      ↓
time-aware qdisc scheduling
      ↓
hardware/software scheduling paths
```

LWN 4.19 merge-window confirms time-based packet transmission landed.

The previously recorded `80b14dee2bea...` should therefore be labeled an
API anchor, not "the SO_TXTIME feature commit".

**Quality A for final-series/release provenance / B for full SHA
enumeration.**

### 7.10 Revised provenance rule for multi-commit networking features

For pre-v5 history the document now uses this rule:

``` text
single coherent introduction commit
  → exact SHA can be the feature anchor

multi-patch subsystem feature
  → identify final/merge-near series
  → list exact SHAs only when verified
  → never pretend one patch is the whole feature

API redesigned before release
  → preserve RFC/design history
  → preserve rework
  → identify landed API separately
```

This prevents false precision while still providing stronger provenance
than a release-note-only history.

### 7.11 Pass-4 status

  ---------------------------------------------------------------------
  Feature                                  Release Provenance after
                                                   pass 4
  -------------------- --------------------------- --------------------
  VRF                                          4.3 release/merge
                                                   strong; exact full
                                                   SHA audit optional

  Lightweight tunnels                          4.3 22-patch series +
                                                   merge verified

  OVS conntrack                                4.3 multi-revision
                                                   series + merge
                                                   verified

  TC direct packet                             4.7 semantics/release
  access                                           verified

  XDP                                          4.8 late v8 series
                                                   boundary verified

  cgroup BPF                                  4.10 attachment-family
                                                   model clarified;
                                                   exact socket anchor
                                                   added

  BPF LWT                                     4.10 series boundary
                                                   verified

  SOCK_OPS                                    4.13 final-series
                                                   provenance

  SOCKMAP                                     4.14 kernel-doc release
                                                   provenance

  AF_XDP                                      4.18 address-family
                                                   series separated
                                                   from ZC series

  TCP ZC RX                                   4.18 initial API →
                                                   reworked landed API
                                                   verified

  SO_TXTIME                                   4.19 RFC/final-series
                                                   architecture
                                                   verified
  ---------------------------------------------------------------------

At this point the remaining exact-SHA work is mainly archival
completeness rather than a material uncertainty about the historical
evolution.

------------------------------------------------------------------------

## Cross-document factual corrections after release-map audit

This section records corrections that are also applied to the
interpretation used by the release map and synthesis sections. Where an
older research-pass appendix still quotes an earlier shorthand, this
section is authoritative.

### 8.1 `skb_drop_reason`: Linux 5.17, not 5.19

The initial structured drop-reason infrastructure (`kfree_skb_reason()`
and the `skb_drop_reason` model) belongs to **Linux 5.17**. Linux
5.18/5.19 and later releases expanded coverage and subsystem-specific
reasons.

Therefore:

``` text
5.17  structured skb drop-reason foundation
5.18+ broader call-site/subsystem coverage
5.19  BIG TCP and other networking work — not the origin of skb_drop_reason
```

### 8.2 There is no Linux 4.20

Linux versioning jumps from **4.19 to 5.0**. Consequently EDT pacing,
BPF flow dissector, taprio and strict rtnetlink checking must not be
moved into a fictitious 4.20 release. The existing Linux 5.0 baseline
remains the correct release-generation bucket for those features. UDP
GRO and UDP `MSG_ZEROCOPY` are also part of the 5.0 networking
generation.

### 8.3 BQL remains a Linux 3.2-generation feature

The DQL/BQL work was rebased for the 3.2 cycle. It must not be moved to
3.3 merely because team, net_prio and TCP memory-cgroup work are
associated with the following release.

### 8.4 BPF sendmsg hook remains Linux 4.17

The 4.17 merge-window material explicitly records BPF filtering/hooks
for `sendmsg()` / `sendfile()`-related socket paths. Do not move this
item to 4.18.

### 8.5 TCP autocorking: add Linux 3.14 milestone

TCP autocorking is a Linux **3.14** networking milestone and should be
represented in the 3.x transport/scalability lineage.

### 8.6 per-netns RTNL: Linux 6.13 is a milestone, not completion

Linux 6.13 introduced important per-network-namespace RTNL
infrastructure and conversions. It did **not** mean that the global RTNL
lock had been completely replaced.

Use:

``` text
global RTNL
 → unlocked/finer-grained paths
 → per-netns RTNL infrastructure (6.13 milestone)
 → subsystem-by-subsystem conversion
 → continuing lock decomposition
```

### 8.7 BPF qdisc and DCCP removal: Linux 6.16

Both are now release-resolved:

``` text
6.16
 ├─ BPF struct_ops qdisc
 └─ DCCP removal
```

The older "2025 merge-window" wording should be read as development
chronology; the release assignment is Linux 6.16.

### 8.8 TCP-AO: initial mainline support is Linux 6.7

TCP Authentication Option support first entered mainline in **Linux
6.7**. The later 7.2 work is a separate cryptographic
implementation/integration step and must not be presented as the origin
of TCP-AO.

``` text
6.7  initial TCP-AO mainline support
7.2  later libcrypto-oriented evolution
```

### 8.9 Linux 5.12: threaded NAPI belongs in the release timeline

Threaded NAPI was merged for Linux 5.12 and should appear not only in
the thematic body but also in the chronological timeline.

### 8.10 AccECN: distinguish protocol landing from default policy

AccECN arrived incrementally before 7.0. Linux 7.0 changes the default
policy to `tcp_ecn=5`.

That means:

``` text
incoming connections:
  AccECN enabled by default

outgoing connections:
  ECN/AccECN initiation remains disabled by default
```

Therefore "AccECN default enabled in 7.0" is too broad without this
qualification.

### 8.11 Linux 6.2: add TCP PLB and IPsec packet offload

The 6.2 networking chronology should include:

``` text
TCP PLB
IPsec packet-offload generation
```

These belong in the performance/offload lineage and should not be hidden
under a generic "BPF/netdev/API continuation" label.

### 8.12 Revised era model: eras describe dominant themes, not hard beginnings

The earlier four-era model is retained only as a dominant-theme model.
Architectural lineages overlap release boundaries.

In particular:

``` text
Linux 4.x
  programmable fast-path formation
  + early zero-copy / packet-memory foundations

Linux 5.x
  programmability expansion
  + packet-memory infrastructure maturation

Linux 6.x–7.x
  memory-provider and queue-ownership architecture
  becomes a first-class design axis
```

AF_XDP and TCP zero-copy RX already appear in 4.18, so the
memory/zero-copy lineage must not be described as starting only in 6.x.

### 8.13 Evolution includes retreat and removal

BPF/network programmability is not a monotonic "more hooks forever"
story.

The historical model must include:

``` text
experimentation
 → review
 → adoption / integration
 → operational feedback
 → retention, redesign, or removal
```

Examples worth tracking include unsuccessful or abandoned directions
such as bpfilter, P4TC proposals that did not become the dominant
upstream TC model, and later removal of problematic integrations such as
TLS+sockmap. Protocol removal (for example DCCP) is the same kind of
architectural pruning on a different axis.

This changes the synthesis from "Linux networking continuously
accumulated features" to "Linux networking repeatedly explored,
selected, redesigned, and removed mechanisms while moving architectural
boundaries."

### 8.14 Items deliberately left pending exact release/commit audit

The following claims are useful leads but are not promoted to
authoritative release-map facts in this revision until exact landing
evidence is attached:

``` text
page_pool exact initial release
netkit queue leasing exact release assignment
IPv6 BIG TCP HBH-removal exact release assignment
large (>4K) memory-provider RX buffers exact release assignment
TLS + sockmap removal exact 7.2 landing
FIB-rule per-netns mutex exact 7.3-development landing/benchmark provenance
```

This is deliberate: a plausible or remembered release number is not
sufficient for the provenance standard used by this document.

------------------------------------------------------------------------

# Part II --- Canonical release chronology

This is the **single normative release map**. Appendix material records
provenance and research history but does not define release attribution.

The audit separates three different facts:

``` text
RFC / review date
        ≠
net-next / subsystem-tree landing
        ≠
released mainline version
```

`Verified` below means that the release assignment has been
cross-checked against a release/merge-window source or an equivalent
upstream landing anchor. `Inherited` means the assignment is retained
from the earlier exact-provenance passes and remains subject to the
continuing full sweep.

  ----------------------------------------------------------------------
  Release             Canonical networking      Audit
                      milestones                
  ------------------- ------------------------- ------------------------
  3.0                 namespace/setns-era       inherited
                      foundation                

  3.3                 BQL/DQL mainline          **corrected**
                      generation; team;         
                      net_prio; TCP memcg       

  3.5                 CoDel / fq_codel          verified earlier

  3.6                 TSQ; TFO client; IPv4     verified earlier
                      route-cache removal       

  3.7                 VXLAN; TFO server; IPv6   inherited
                      NAT                       

  3.9                 SO_REUSEPORT and          inherited
                      socket-scaling work       

  3.13                nftables                  verified earlier

  3.14                TCP autocorking;          **corrected**
                      continuing BPF transition 

  3.18                bpf()                     verified earlier
                      syscall/maps/verifier     
                      generation; DCTCP;        
                      Geneve/FOU                

  3.19                ipvlan; initial switchdev **corrected**
                      generation                

  4.3                 VRF; LWT; OVS conntrack   verified earlier

  4.6                 devlink                   inherited

  4.7                 TC BPF direct packet      verified earlier
                      access                    

  4.8                 XDP                       verified earlier

  4.9                 BBR                       verified

  4.10                cgroup BPF; BPF LWT; IPv6 verified earlier
                      Segment Routing           

  4.13                SOCK_OPS; kTLS TX         inherited
                      generation                

  4.14                SOCKMAP; TCP              **verified/corrected**
                      `MSG_ZEROCOPY`            

  4.17                `SK_MSG`/sockmap sendmsg  **split/corrected**
                      path; mac80211            
                      airtime-fairness          
                      generation                

  4.18                AF_XDP;                   **verified/corrected**
                      `TCP_ZEROCOPY_RECEIVE`;   
                      page_pool refurbished/XDP 
                      memory-model generation;  
                      cgroup UDP sendmsg hooks  

  4.19                SO_TXTIME/time-based TX;  verified earlier
                      CAKE                      

  4.20                TCP EDT pacing; BPF flow  **verified/corrected**
                      dissector; taprio;        
                      rtnetlink strict checking 

  5.0                 UDP GRO; UDP              corrected
                      `MSG_ZEROCOPY`            

  5.1                 devlink health; BPF       inherited
                      spinlocks/DCE;            
                      SO_BINDTOIFINDEX; Y2038   
                      timestamps; io_uring      

  5.3                 nexthop objects           **corrected**

  5.5                 Wi-Fi AQL generation      corrected

  5.6                 MPTCP; WireGuard; BPF     verified earlier
                      struct_ops/TCP CC;        
                      ethtool-netlink           

  5.9                 SK_LOOKUP                 inherited

  5.12                threaded NAPI             **verified**

  5.15                IPv6 IOAM; MCTP; bridge   inherited/partly
                      per-VLAN multicast        verified

  5.17                `kfree_skb_reason()` /    **verified**
                      structured skb            
                      drop-reason foundation    

  5.19                IPv6 BIG TCP; later       verified earlier
                      drop-reason expansion;    
                      MPTCP evolution           

  6.2                 TCP PLB; IPsec            **series/release
                      packet-offload generation generation verified**

  6.3                 IPv4 BIG TCP; YNL         verified earlier
                      generation                

  6.6                 AF_XDP multi-buffer       verified earlier

  6.7                 netkit; initial TCP-AO    **verified**
                      mainline support          

  6.11                virtio-net AF_XDP RX      verified earlier
                      zero-copy                 

  6.12                Device Memory TCP RX      verified earlier

  6.13                per-netns RTNL            **corrected**
                      infrastructure and        
                      migration milestone ---   
                      not completion            

  6.15                io_uring ZCRX; further    verified earlier
                      RTNL breakup              

  6.16                Device Memory TCP TX; BPF verified earlier
                      qdisc; DCCP removal       

  6.18                AccECN core; UDP RX work; verified earlier
                      DIBS --- separate         
                      shared-memory lineage     

  7.x                 later                     continuing audit
                      AccECN/default-policy,    
                      queue/memory/offload and  
                      protocol evolution        
  ----------------------------------------------------------------------

## Corrected boundary notes

### BQL: development generation vs release generation

The late 2011 BQL v3 series explicitly says that it was rebased to Linux
3.2. That is a **development-tree base**, not sufficient evidence to
label BQL a released 3.2 feature. The canonical chronology therefore
records the BQL mainline generation at 3.3 while the provenance appendix
retains the 3.2-rebase history.

### Linux 4.20 is a real release

Linux 4.20 was released in December 2018. Its merge-window networking
highlights include:

``` text
TCP earliest-departure-time pacing
BPF programmable flow dissector
taprio
rtnetlink strict checking
```

These must not be assigned to 5.0 merely because the next development
cycle was renamed 5.0.

### Two different BPF "sendmsg" lineages

Do not collapse these into one feature:

``` text
4.17 generation
  SOCKMAP / SK_MSG
  message data-path processing

4.18 generation
  cgroup UDP sendmsg hooks
  BPF_CGROUP_UDP4_SENDMSG / BPF_CGROUP_UDP6_SENDMSG
  socket address / policy control
```

### Zero-copy chronology

``` text
4.14  MSG_ZEROCOPY transmit foundation
4.18  TCP_ZEROCOPY_RECEIVE
5.0   UDP MSG_ZEROCOPY release generation
5.11  later TCP zero-copy receive API/implementation expansion
```

Thus 5.11 must not be described as the introduction of TCP zero-copy
receive.

### page_pool chronology

The XDP redirect memory-return series in the 4.18 development cycle
introduced the refurbished page_pool as the first allocator type for the
new XDP memory-return model and explicitly identified AF_XDP zero-copy
as an integration target. Later page_pool work is maturation/redesign,
not the origin of the lineage.

## Canonical architecture model --- six axes

``` text
PERFORMANCE
PROGRAMMABILITY
MEMORY
CONTROL PLANE
OBSERVABILITY
DRIVER FRAMEWORK
```

## Era model

Era labels describe dominant themes, not hard technology boundaries:

``` text
3.x       scalability + virtualization + programmability foundations
4.x       programmable fast path + early zero-copy / packet-memory foundations
5.x       expansion + operationalization + memory-infrastructure maturation
6.x–7.x   memory providers + queue ownership + finer-grained control/locking
```

## Release-attribution re-audit --- pass 2

The second sweep resolves several entries that were still marked
inherited.

### Linux 3.7

Release-level LWN coverage explicitly lists all three relevant
networking changes:

``` text
server-side TCP Fast Open
IPv6 NAT
VXLAN
```

The earlier 3.6 TFO article also distinguishes the already-merged client
support in 3.6 from server support expected in 3.7. The canonical split
is therefore:

``` text
3.6 TFO client
3.7 TFO server
```

### Linux 4.6 --- devlink

The 4.6 merge-window report explicitly describes devlink as a new
network-control API for parameters not tied to a particular device
class. This release assignment is now verified.

### Linux 4.13 --- SOCK_OPS and kTLS

The 4.13 merge-window report explicitly lists both
`BPF_PROG_TYPE_SOCK_OPS` and the initial kernel TLS implementation. Both
entries are promoted from inherited to verified.

### Linux 5.1 and 5.5 --- airtime fairness vs AQL

The previous pass over-corrected this chronology. Direct release-level
evidence shows:

``` text
5.1  mac80211 airtime fairness
5.5  Airtime Queue Limits (AQL)
```

AQL patch history in late 2019 describes it as queue limiting based on
expected airtime; it is a distinct follow-on mechanism, not another name
for the 5.1 fairness scheduler.

### Linux 5.3 --- nexthop objects

The 2018 RFC introduced nexthops as independent objects to improve route
scalability and align the kernel with routing daemons and switch ASICs.
The final functionality is present in the Linux 5.3 generation. This
also matches the existing exact core anchor `ab84be7e54fc` retained in
the provenance appendix.

### Linux 5.9 --- SK_LOOKUP

The May 2020 bpf-next series introduces `BPF_PROG_TYPE_SK_LOOKUP`,
allowing BPF to run during transport-layer socket lookup and select the
receiving socket. The canonical 5.9 assignment is retained, with the
series provenance now explicitly attached.

### Linux 5.15 --- IOAM, MCTP, per-VLAN multicast

The 5.15 merge-window report directly lists:

``` text
IPv6 IOAM pre-allocated trace
MCTP
bridge per-VLAN multicast
```

All three release assignments are now verified at release level.

### Linux 6.2 --- TCP PLB and IPsec packet offload

The 6.2 merge-window report explicitly includes TCP Protective Load
Balancing (PLB). The IPsec/XFRM packet-offload series belongs to the
same 6.2 development generation and provides full packet-policy/state
hardware offload rather than only crypto offload.

## Release-attribution re-audit --- pass 3: exact-series boundary

This pass moves from release headlines to the actual series boundary for
ambiguous APIs.

### Linux 3.9 --- `SO_REUSEPORT`

LWN states explicitly that TCP and UDP `SO_REUSEPORT` support was merged
in the Linux 3.9 development cycle. The 3.9 merge-window coverage
independently lists the same socket option. The patch series was rebased
to 3.8 during review, which is another example of why a series' base
kernel must not be mistaken for the release containing the feature.

Canonical interpretation:

``` text
review/rebase base: 3.8
mainline release generation: 3.9
```

### Linux 4.14 --- `MSG_ZEROCOPY`

The send-side zero-copy series defines the socket opt-in plus
`MSG_ZEROCOPY` send semantics and completion notifications. Contemporary
documentation initially described the feature as implemented for TCP.
UDP support is a later extension, so the canonical lineage remains:

``` text
4.14  TCP MSG_ZEROCOPY foundation
5.0   UDP MSG_ZEROCOPY
```

This transmit API is distinct from `TCP_ZEROCOPY_RECEIVE`.

### Linux 4.17 --- `BPF_PROG_TYPE_SK_MSG`

The March 2018 bpf-next v2 series is explicitly titled
`bpf,sockmap: sendmsg/sendfile ULP` and introduces/uses
`BPF_PROG_TYPE_SK_MSG`, `BPF_SK_MSG_VERDICT`, and message helpers.

Architecturally this is:

``` text
socket in SOCKMAP
       ↓
sendmsg / sendfile
       ↓
SK_MSG BPF verdict
       ↓
pass / drop / redirect / message manipulation
```

It is not the same mechanism as cgroup UDP sendmsg address hooks.

### Linux 4.18 --- refurbished page_pool in the XDP memory-return model

The XDP redirect memory-return series says explicitly that a
**refurbished version of page_pool** is the first return-allocator type
and an integration point for AF_XDP zero-copy.

The wording matters. This document therefore distinguishes:

``` text
earlier page-pool ideas / RFC history
             ↓
4.18-generation refurbished page_pool
as part of XDP memory-return infrastructure
             ↓
later generic RX recycling / DMA helpers
             ↓
netlink introspection
             ↓
netmem / memory-provider work
```

The document no longer claims that every concept named page_pool first
appeared in 4.18.

### Linux 5.9 --- `BPF_PROG_TYPE_SK_LOOKUP`

The May 2020 bpf-next series defines a new `BPF_PROG_TYPE_SK_LOOKUP`
program type and a per-network-namespace attach point. It runs at
transport socket lookup and can select a receiving TCP/UDP socket with
`bpf_sk_assign()`.

This is an important continuation of the socket-programmability lineage:

``` text
SO_REUSEPORT
 → SOCKMAP / SK_MSG
 → cgroup socket/address hooks
 → SK_LOOKUP
```

but SK_LOOKUP is a distinct receive-side socket-selection hook, not
another sockmap verdict.

### Linux 6.2 --- XFRM/IPsec packet offload

The final XFRM series distinguishes two hardware models:

``` text
crypto offload:
  NIC encrypts/decrypts
  kernel retains packet-policy processing

packet offload:
  NIC encrypts/decrypts
  NIC handles packet encapsulation
  NIC handles SA/policy state
  kernel synchronizes XFRM state/policy with hardware
```

The driver-facing contract is exposed through `xfrmdev_ops`, with
packet-offload-specific policy callbacks in addition to state callbacks.
This is therefore relevant not only to the protocol timeline but also to
the Driver Framework axis.

## Pass-3 confidence rule

A feature is marked `release-level verified` when a release/merge-window
source explicitly places it in that cycle.
`final-series generation verified` additionally means that the reviewed
final/merge-near series and its API boundary have been identified.
`exact SHA` is reserved for a canonical mainline commit that has itself
been checked; a series article alone is not promoted to exact-SHA
status.

## Release-attribution re-audit --- pass 4: exact mainline anchors

This pass records only commit IDs that can be independently
corroborated. A feature can span several commits; an "anchor" is not
automatically the entire feature.

### Linux 3.9 --- `SO_REUSEPORT`

Exact infrastructure anchor:

``` text
055dc21a1d1d219608cd4baac7d0683fb2cbbe8a
soreuseport: infrastructure
Tom Herbert
2013-01-23
```

This commit adds the common socket-level infrastructure. TCP/IPv4,
TCP/IPv6 and UDP support are additional commits in the same series, so
`055dc21a...` is deliberately labeled an **infrastructure anchor**, not
"the complete SO_REUSEPORT feature in one commit".

### Linux 4.14 --- TCP `MSG_ZEROCOPY`

Exact TCP enablement anchor:

``` text
f214f915e7db99091f1312c48b30928c1e0c90b7
tcp: enable MSG_ZEROCOPY
Willem de Bruijn
2017-08-03
```

The full feature is a multi-commit branch including the generic
`MSG_ZEROCOPY` definition, socket opt-in, completion
notification/coalescing, accounting and TCP enablement. The anchor above
is the protocol-specific TCP enablement commit.

### Linux 4.17 --- `SK_MSG`

The final v3 series and the bpf-next pull request are verified,
including the new sendmsg / sendfile BPF hook and helpers. An exact
canonical implementation SHA is **not inserted yet** because the
currently recovered SHA `82a8616889d5` is a selftest commit, not the
core program-type/ULP implementation. This is intentionally left at
final-series confidence.

### Linux 4.18 --- page_pool / XDP memory return

A strong exact integration anchor is:

``` text
60bbf7eeef10dc647430646d7fe5e3d8d132dbec
mlx5: use page_pool for xdp_return_frame call
Jesper Dangaard Brouer
2018-04-17
```

Later fixes explicitly use this commit in `Fixes:` tags when referring
to where page_pool was added to mlx5 XDP handling. It is therefore a
strong integration anchor, but the document does **not** label it as the
single origin commit of page_pool itself. The series contains separate
commits for refurbishing page_pool and registering it as an XDP memory
allocator type.

### Linux 5.9 --- `SK_LOOKUP`

Exact program-type anchor:

``` text
e9ddbb7707ff
bpf: Introduce SK_LOOKUP program type with a dedicated attach point
Jakub Sitnicki
2020-07-17
```

This hash is independently referenced by later fixes, and the original
author later stated that SK_LOOKUP first appeared in Linux 5.9. This is
A-grade provenance for the program-type origin.

### Linux 6.2 --- XFRM packet offload

Exact UAPI/core anchor:

``` text
d14f28b8c1de668bab863bf5892a49c824cb110d
xfrm: add new packet offload flag
Leon Romanovsky
2022-12-02
```

This introduces `XFRM_OFFLOAD_PACKET`. It is the cleanest anchor for
distinguishing the new packet-offload mode from the older crypto-offload
model. The complete functionality is again a series, not one commit.

## Exact-anchor policy after pass 4

The canonical inventory now distinguishes:

``` text
origin/core anchor
integration anchor
protocol-specific enablement anchor
series merge/pull
release attribution
```

This prevents a common historical error: finding one convenient commit
in a feature series and presenting it as if the entire architecture
landed atomically.

## Release-attribution re-audit --- pass 5: Driver Framework

### switchdev --- 3.19 origin, 4.x model expansion

Kernel documentation defines switchdev as the in-kernel driver model for
switch devices that offload the Linux forwarding data plane. The history
must distinguish the initial 3.19-generation switchdev work from the
substantial 2015/2016 object/attribute, transaction, bridge/FIB and
documentation expansion.

The 2015 v3 series introduced generic switchdev get/set attribute
operations and the prepare/commit model; subsequent commits generalized
switchdev objects for VLAN/FIB/FDB. These are **expansion anchors**, not
evidence that switchdev itself first appeared in 4.x.

### devlink --- Linux 4.6

The Linux 4.6 merge-window report explicitly calls devlink a new
network-control API for parameters not tied to a specific device class.
This is the canonical release origin. The exact initial commit set
remains a multi-commit item in the provenance backlog; no guessed SHA is
inserted.

### phylink --- RFC 2015 → mainline 4.13

Exact infrastructure anchor:

``` text
9525ae83959b60c6061fe2f2caabdc8f69a48bc6
phylink: add phylink infrastructure
```

This resolves a previous chronology ambiguity: 2015 is the RFC/design
point, while the mainline infrastructure arrives in the Linux 4.13
generation.

### DIM / net_dim --- common driver algorithm

Netdev 0x12 (2018) describes DIM as a **network-driver-independent
framework** for dynamic interrupt moderation and states that it was
already upstream in mlx5 and as the net_dim library. This is strong
architectural/conference provenance. The exact initial generic library
SHA is left pending rather than inferred from a later driver
integration.

### netdevsim --- Linux 4.16

The Linux 4.16 merge-window report explicitly introduces netdevsim as a
virtual device for testing networking hardware-offload features,
initially BPF offload. Its architectural importance is testability:
common driver/offload APIs can be exercised without requiring the
corresponding NIC or switch ASIC.

### ethtool Generic Netlink --- Linux 5.6 groundwork

The 5.6 merge-window report states that much of the long-running ethtool
ioctl→Netlink groundwork was merged for 5.6. The matching userspace
series says that the kernel interface is available since 5.6-rc1. This
is therefore the canonical start of the ethtool Generic Netlink
generation, while individual command coverage continues over later
releases.

### auxiliary bus --- Linux 5.11

The auxiliary-bus history is now release-level verified. The driver-core
signed tag was prepared specifically for the 5.11-rc1 merge and reaches:

``` text
0d2bf11a6b3e275a526b8d42d8d4a3a6067cf953
driver core: auxiliary bus: minor coding style tweaks
```

That SHA is the tip of the auxiliary-bus support tag, **not** the origin
commit. The underlying feature commit is Dave Ertman's
`Add auxiliary bus support`; its exact SHA is kept pending until
independently verified.

### Rust PHY --- RFC 2023 → Linux 6.8

The lineage is now explicit:

``` text
2023-09 RFC v1
   ↓
2023-12 final/near-final v10/v11
   ↓
Linux 6.8
Rust phylib abstractions
module_phy_driver macro
Rust Asix reference PHY driver
```

LWN's 6.8 merge-window coverage calls this the first user-visible Rust
code in the kernel. The final v11 series contains four patches and
creates `rust/kernel/net/phy.rs` plus the Rust Asix driver. Exact
per-patch mainline SHAs remain pending; the document does not invent
them.

### Driver-framework confidence table

  -----------------------------------------------------------------------------
  Framework         Design/RFC              Release      Exact anchor status
                                            landing      
  ----------------- ----------------------- ------------ ----------------------
  switchdev         2014/2015 evolution     3.19 origin; multi-commit;
                                            4.x          expansion commits
                                            expansion    identified

  devlink           2016 series             4.6          release verified;
                                                         exact origin pending

  phylink           2015 RFC                4.13         `9525ae83959b...`
                                            generation   exact

  DIM/net_dim       pre-2018 work           upstream by  generic-library SHA
                                            Netdev 0x12  pending
                                            / 2018       

  netdevsim         offload test framework  4.16         release verified

  ethtool-netlink   2018 RFC lineage        5.6          release verified;
                                            groundwork   exact core set pending

  auxiliary bus     ancillary/virtual-bus   5.11         signed tag/tip
                    predecessors                         verified; origin SHA
                                                         pending

  Rust phylib       2023 RFC→v11            6.8          release/final-series
                                                         verified; exact SHAs
                                                         pending
  -----------------------------------------------------------------------------

## Release-attribution re-audit --- pass 6: Driver Framework exact anchors

### ethtool Generic Netlink --- exact core anchor

The core Generic Netlink interface has an exact mainline anchor:

``` text
2b4a8990b7df55875745a80a609a1ceaaf51f322
ethtool: introduce ethtool netlink interface
Michal Kubecek
2019-12-27
```

This commit creates the `ethtool` Generic Netlink family, UAPI header,
kernel header, `net/ethtool/netlink.c`, and the initial documentation.
It is therefore an appropriate **core-interface anchor** for the 5.6
generation.

The feature remains intentionally described as a generation rather than
a one-commit conversion: individual ethtool commands were migrated
incrementally after the family itself appeared.

### auxiliary bus --- final series is known, exact origin SHA remains conservative

The final standalone v4 patch states the architecture directly:

``` text
auxiliary_device
auxiliary_driver
probe/remove
shutdown
suspend/resume
string-based matching on the auxiliary bus
```

It is authored by Dave Ertman with multiple co-developers and was posted
in December 2020. Together with the driver-core 5.11 signed tag, this is
enough to establish design and release provenance. The document still
does not assign an exact origin SHA until that commit object is
independently recovered.

### Rust PHY --- exact final series shape

The final v11 series contains exactly four patches:

``` text
1. rust: core abstractions for network PHY drivers
2. rust: net::phy add module_phy_driver macro
3. MAINTAINERS: add Rust PHY abstractions for ETHERNET PHY LIBRARY
4. net: phy: add Rust Asix PHY driver
```

The first patch adds the phylib abstractions and
`CONFIG_RUST_PHYLIB_ABSTRACTIONS`; the fourth adds
`drivers/net/phy/ax88796b_rust.rs`. This provides a clean
design→final-series boundary even while exact mainline SHAs remain
pending.

### What remains deliberately unresolved

The following items are **not** assigned guessed hashes:

``` text
switchdev initial origin commit set
devlink initial origin commit set
generic DIM/net_dim initial library commit
netdevsim initial core commit
auxiliary-bus feature-origin commit
Rust PHY four mainline SHAs
```

For these, release-level or final-series provenance is stronger than an
unverified hash. They remain explicitly pending.

## Release-attribution re-audit --- pass 7: Driver Framework integration

This pass does not create a competing timeline. It projects the
canonical release map onto the Driver Framework axis and connects older
driver APIs to the newer queue/memory object model.

### netdev-genl queue/NAPI object model

Netdev 0x17 describes extending the YAML-defined `netdev-genl` family
with queue and NAPI objects. Queue attributes include the servicing NAPI
instance, statistics and memory model; NAPI attributes include NAPI ID,
device, IRQ vector and threaded-poller PID.

The Linux 6.8 netdev Netlink documentation already contains `napi` and
`queue` attribute sets with these relationships. Therefore this document
uses **6.8 as a documentation-level milestone**, not as a claim that
every modern queue operation landed in 6.8.

### page_pool introspection

The 2023 RFC→v2 series adds:

``` text
page-pool IDs
per-netdev association
NAPI ID association
Netlink GET
state-change notifications
memory accounting
recycling statistics
```

This is the point where page_pool evolves from a common internal
allocator/recycler into an identifiable and observable networking
object.

### Modern state must not be backdated

Current `netdev-genl` documentation includes later capabilities such as:

``` text
napi-set
queue-create
queue statistics
DMA-BUF binding
io_uring provider information
XSK information
```

These current capabilities are shown as the **direction of evolution**
and are not all retroactively assigned to Linux 6.8. Exact release
attribution for queue creation, queue leasing and each memory-provider
operation remains in the 7.x audit backlog.

## Release-attribution re-audit --- pass 8: 7.x and development boundary

The 7.x audit uses a stricter rule than the earlier draft: a net-next
series is not called a released 7.x feature merely because its likely
target cycle is known.

As of this audit, Linux 7.2 has a stable series. Work targeting the next
cycle is labeled **7.3-development** until a released mainline baseline
exists.

### Queue leasing --- accepted/reworked development lineage

The January 2026 netkit v7 series defines queue leasing as a way for a
virtual netdev in a container namespace to proxy a physical NIC queue,
so that AF_XDP and memory providers can be configured through the
virtual interface at native speed.

The architectural relationship is:

``` text
physical NIC RX queue
        ↓ leased/proxied
netkit guest queue
        ↓
container namespace
        ├── AF_XDP
        └── memory provider / zero-copy RX
```

The history includes an early merge, revert and later remerge/rework
already recorded in the exact-provenance appendix. Because this lineage
crossed development-cycle boundaries, the canonical table does not
assign it to a released 7.1 merely from series dates.

### Large RX buffers for memory providers

The large-buffer work is not just an io_uring optimization. The series
explicitly builds on per-queue configuration and says the mechanism can
be used by other memory providers.

The key design change is:

``` text
old assumption:
  one provider fragment ≈ PAGE_SIZE / 4K on x86

new model:
  queue-specific provider buffer length
  e.g. 32K buffers
        ↓
  hardware GRO can coalesce into larger contiguous regions
        ↓
  fewer fragments through the stack
```

Reported development benchmarks on a 200-Gbit NIC show substantial
CPU-utilization gains for 32K versus 4K receive buffers. These numbers
are development-series measurements, not universal performance
guarantees.

A later series explicitly describes support for memory providers with
buffers larger than 4K/PAGE_SIZE. Exact released-version attribution
remains separate from the architectural lineage.

### IPv6 BIG TCP without Hop-by-Hop header

The v3 series dated 2026-02-02 removes the synthetic IPv6 Hop-by-Hop
jumbo header used by the original BIG TCP implementation.

Motivation:

``` text
original IPv6 BIG TCP
  temporary HBH carries 32-bit packet length
        ↓
tunnels create difficult inner/outer HBH handling
        ↓
drivers would need tunnel-specific stripping logic
        ↓
remove HBH and recover length from skb->len
        ↓
align IPv6 BIG TCP with IPv4 BIG TCP
```

This is a prerequisite for the later BIG TCP tunnel work. The document
therefore treats "BIG TCP without HBH" and "BIG TCP for UDP tunnels" as
two separate series.

### BIG TCP for VXLAN / GENEVE --- 7.3-development

The final v9 series was posted on 2026-07-10. It is explicitly a
follow-up to the IPv6 HBH-removal work and adds BIG TCP support for
IPv4/IPv6 workloads over VXLAN and GENEVE.

This is recorded as **7.3-development**, not as a released 7.3 feature
in the canonical release map.

### TLS + sockmap cleanup

The 7.x cleanup lineage must distinguish two different statements:

``` text
historical:
  TLS and sockmap had integration paths and many interaction fixes

current cleanup:
  kernel rejects unsupported TLS + sockmap combinations
```

Stable trees in 2026 contain the change
`tls: reject the combination of TLS and sockmap`. This confirms the
cleanup direction, but it is not sufficient by itself to label a
particular mainline release as "TLS+sockmap removal". The old wording is
therefore downgraded until the originating mainline commit/tag is
audited.

### FIB-rule per-netns locking

Search results for the exact 7.3-development landing/benchmark are not
yet strong enough for A-grade provenance. The document keeps this item
in the development audit backlog rather than presenting an unverified
release assignment.

## 7.x confidence rule

``` text
stable release evidence
    > mainline merge/tag containment
    > accepted/pulled net-next series
    > final patch series
    > RFC
```

A feature is never promoted upward merely because the calendar makes a
target release plausible.

# Part III --- Long-term feature lineages

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
BIG TCP (5.19 IPv6)
  ↓
IPv4 BIG TCP (6.3)
  ↓
AF_XDP multi-buffer (6.6)
  ↓
VXLAN / GENEVE BIG TCP (7.x)
```

BIG TCP は wire MTU を巨大化する機能ではなく、 **kernel 内部の
packet-processing unit を大きくする機能**として理解する。

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
`page_pool → netmem → device-memory/memory-provider lineage; DIBS is separate`
ではなく、 shared-memory transport 側の別 lineage として扱う。

------------------------------------------------------------------------

## XDP / AF_XDP

v5.0 時点で XDP/AF_XDP は存在した。その後の本質は周辺 infrastructure
の成熟。

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

``` text
v5.0: XDP / TC / cgroup BPF
          ↓
5.3–5.4: socket hooks / SYN-cookie integration
          ↓
5.6: struct_ops → tcp_congestion_ops
          ↓
5.9: SK_LOOKUP
          ↓
6.x: MPTCP / defrag / timestamp / netkit / BPF qdisc
```

つまり:

``` text
packet programmability
 → socket programmability
 → protocol algorithm programmability
 → virtual-device programmability
 → queue/datapath programmability
```

へ拡大した。

------------------------------------------------------------------------

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
veth → netkit → BPF-native datapath → queue leasing
```

v6.7 netkit、v6.11 virtio-net AF_XDP RX ZC、2026 の netkit queue leasing
は 「full kernel bypass」よりも、

**kernel が ownership/control を保持し、data movement を最小化する**

方向として読むと理解しやすい。

------------------------------------------------------------------------

## TCP / UDP / transport

### TCP

``` text
v5.0 EDT pacing
  ├─ BPF congestion control
  ├─ MPTCP (5.6)
  ├─ TCP zero-copy RX
  ├─ BIG TCP (5.19)
  ├─ TCP-AO/security
  ├─ Device Memory TCP
  ├─ TCP_RTO_MAX_MS
  └─ AccECN
```

MPTCP は initial upstream から multi-subflow、userspace path manager、
Generic Netlink、BPF integration へ進化した。

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
```

という長期的な scalability 改善が続く。

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

# Part IV --- Network Device Driver Framework Evolution

## Canonical Driver Framework chronology

This is the Driver Framework projection of the Part II release map. It
is not a second independent chronology; every release below must agree
with Part II.

  -----------------------------------------------------------------------
  Release                 Driver-framework        Architectural effect
                          milestone               
  ----------------------- ----------------------- -----------------------
  3.3                     DQL/BQL                 common queue-pressure
                                                  control replaces
                                                  driver-local queue
                                                  sizing policy

  3.19                    switchdev origin        Linux forwarding
                                                  objects begin to drive
                                                  switch-ASIC offload

  4.6                     devlink                 device/ASIC-wide
                                                  resources and control
                                                  separated from one
                                                  `net_device`

  4.8                     XDP                     driver RX path gains a
                                                  programmable pre-skb
                                                  execution point

  4.13                    phylink                 common MAC/PHY/PCS/SFP
                                                  link-management state
                                                  machine

  4.16                    netdevsim               common offload APIs
                                                  become testable without
                                                  physical hardware

  4.18                    refurbished             RX allocation/recycling
                          page_pool/XDP memory    begins moving into
                          return                  common memory
                                                  infrastructure

  5.1                     devlink health          common
                                                  reporting/recovery
                                                  model for device health

  5.6                     ethtool Generic Netlink driver management ABI
                                                  becomes
                                                  structured/extensible

  5.11                    auxiliary bus           complex devices can
                                                  expose independently
                                                  bound subfunctions

  5.12                    threaded NAPI           NAPI execution model
                                                  becomes more explicitly
                                                  configurable

  6.8                     Rust phylib +           safe driver abstraction
                          queue/NAPI netdev-genl  and explicit netdev
                          objects                 objects develop in
                                                  parallel

  6.x→7.x                 page_pool introspection queue, poller and
                          → queue/NAPI            memory ownership become
                          configuration → memory  first-class driver/core
                          providers               contracts
  -----------------------------------------------------------------------

### The long architectural transition

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

The last steps are important: modern netdev work increasingly exposes
what used to be opaque driver implementation details as generic objects
with IDs and relationships.

This section deliberately does **not** enumerate individual NIC drivers.
It follows the common infrastructure which changed what a Linux network
driver is expected to implement, and which moved repeated driver-local
mechanisms into reusable kernel frameworks.

## Architectural thesis

The driver-side evolution can be summarized as:

``` text
driver-local mechanisms
        ↓
common queue / CPU scaling primitives
        ↓
common hardware-control and offload models
        ↓
common RX-memory / interrupt / link-management frameworks
        ↓
introspectable netdev objects and queue/memory ownership
        ↓
typed / memory-safe driver abstractions in Rust
```

This is a sixth axis crossing the performance, programmability, memory,
control-plane and observability axes already used elsewhere in this
document.

## Starting point at Linux 3.0

By the Linux 3.0 era, NAPI and multiqueue networking were already
established. RPS/RFS had arrived in 2.6.35 and XPS in 2.6.38, so the
v3.x story begins with a driver model already centered on:

``` text
RX/TX descriptor rings
IRQ / MSI-X
NAPI poll contexts
multiple RX/TX queues
RSS in hardware
RPS/RFS/XPS in the stack
ethtool + net_device_ops
```

The important v3.x change is therefore not the invention of NAPI, but
increasing coordination between drivers and common networking-core
algorithms.

## Linux 3.2 --- DQL/BQL: queue control moves into common core

DQL/BQL is one of the clearest early examples.

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

Driver-framework significance: performance policy begins moving out of
individual drivers into reusable net core infrastructure.

## Linux 4.x --- hardware becomes a first-class Linux networking object

### phylink provenance correction: RFC in 2015, mainline infrastructure in 4.13

The 2015 26-patch RFC established the architecture for coordinating MAC,
PHY, PCS/SerDes and hot-pluggable SFP modules. It was **not** yet the
mainline landing.

The canonical infrastructure anchor is:

``` text
9525ae83959b60c6061fe2f2caabdc8f69a48bc6
phylink: add phylink infrastructure
Russell King
authored 2017-07-25; committed 2017-08-06
```

Later stable fixes explicitly cite this SHA in `Fixes:` tags, providing
independent confirmation of its role. The history is therefore:

``` text
2015 RFC architecture
   ↓
2017 / Linux 4.13 generation
   phylink infrastructure mainline
   ↓
later PCS/SFP/MAC API expansion
```

### 4.1 switchdev --- origin in Linux 3.19, expansion through 4.x

The initial switchdev infrastructure belongs to the Linux 3.19
generation. The 4.x era is where the model expands into the broader
hardware-offload architecture discussed below.

switchdev turns switch ASIC forwarding into a Linux driver model rather
than a proprietary SDK-controlled island.

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

This is a major change in driver responsibility: a driver becomes an
implementation of Linux networking semantics in hardware.

### 4.2 devlink --- device-wide control plane

Initial devlink series: https://lwn.net/Articles/677967/

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

### 4.3 phylink and SFP

The The 2015 phylink/SFP RFC addresses a recurring driver problem: MAC,
PHY, PCS/SerDes and hot-pluggable SFP combinations could not be modeled
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

### 4.4 VF representors and SmartNIC/DPU control

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

### 4.5 XDP changes the driver fast path

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

The last point directly motivates page_pool.

### 4.6 DIM --- interrupt moderation becomes a common library

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

### 4.7 page_pool --- RX memory management becomes shared infrastructure

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

## Linux 5.x --- driver management APIs become structured and observable

### 5.1 devlink health

Linux 5.1 adds generic devlink health reporting and recovery.

Netdev 0x13 describes the goals as:

``` text
real-time alerting
driver debug information
self-healing / recovery
vendor-support data collection
```

Conference:
https://netdevconf.org/0x13/loadsessions/devlink-health-reporting-and-recovery-system.html

This changes hardware error handling from driver-specific logs/private
tools toward a common operational model.

### 5.2 ethtool ioctl → Generic Netlink

The ethtool netlink work addresses limitations of the old ioctl ABI:
extensibility, races, error reporting and lack of notifications.

Series: https://lwn.net/Articles/808028/
https://lwn.net/Articles/810618/

Architecturally this is not just a userspace-tool rewrite. It creates a
structured, extensible management API between userspace, networking core
and drivers.

### 5.3 netdevsim and selftest-driven driver API design

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

### 5.4 auxiliary bus --- one PCI device, multiple subsystem drivers

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

## Linux 6.x--7.x --- queues and memory become explicit framework objects

### 6.1 page_pool becomes observable

2023 page_pool netlink introspection associates pools with netdevices
and NAPI IDs and exports allocation/recycling/memory information.

LWN: https://lwn.net/Articles/948718/

This is an important architectural transition:

``` text
page_pool as hidden driver implementation detail
                 ↓
page_pool as identifiable / observable netdev resource
```

### 6.2 queue and NAPI objects move toward a generic netdev API

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

This is a conceptual shift from "the driver owns opaque rings" toward
"queues and memory are objects negotiated among driver, kernel and
userspace".

It directly connects to Device Memory TCP, io_uring ZCRX and
queue-leasing work elsewhere in this document.

### 6.3 page_pool → netmem → memory providers

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

Kernel Recipes 2024's io_uring zero-copy discussion makes the dependency
explicit: zero-copy RX requires support from NIC hardware, firmware and
driver, and uses page_pool / netmem plus queue configuration.

This means modern high-speed network drivers are increasingly
**memory-provider-aware** rather than simply allocating `struct page`
objects.

### 6.4 XFRM device / IPsec packet offload

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

## Queue / NAPI / page_pool become first-class netdev objects

The 2023 Netdev 0x17 design work is an important turning point. Rather
than treating an RX queue, its NAPI poller and its memory allocator as
opaque details inside each driver, `netdev-genl` exposes generic
relationships such as:

``` text
net_device
   ├── RX queue ID
   │      ├── NAPI ID
   │      ├── queue type
   │      ├── memory model/provider
   │      └── per-queue statistics
   │
   ├── NAPI ID
   │      ├── IRQ vector
   │      ├── threaded-NAPI PID
   │      └── per-NAPI configuration
   │
   └── page_pool ID
          ├── NAPI ID
          ├── net_device
          ├── allocation/recycling statistics
          └── outstanding memory
```

By Linux 6.8 documentation, the netdev Netlink specification already
exposes queue and NAPI objects, including the NAPI ID servicing a queue,
interrupt-vector information and threaded-NAPI PID. This turns topology
that previously required driver-specific inspection into a generic
kernel ABI.

### page_pool introspection is the memory side of the same transition

The 2023 page_pool introspection series gives page pools IDs, associates
them with netdev and NAPI IDs, and exports memory/recycling statistics
over the same netdev Netlink family.

This changes page_pool's architectural role:

``` text
fast RX allocation helper
        ↓
shared driver memory contract
        ↓
identifiable kernel object
        ↓
observable memory owner
        ↓
basis for configurable queue-bound memory providers
```

The important point is not merely observability. Once queue, NAPI and
page-pool identity exists in a common ABI, later zero-copy/device-memory
APIs can express **which queue owns which memory model** without
inventing a new driver-specific control plane.

### Current netdev-genl completes the direction

Current kernel documentation goes beyond read-only topology. It contains
operations such as per-NAPI configuration, queue creation, queue
statistics, and binding DMA-BUF memory to queues. Queue attributes can
also carry provider-specific information such as io_uring
memory-provider or XSK state.

This provides the missing bridge between the Driver Framework and Memory
axes:

``` text
Driver Framework                     Memory
----------------                     ------
queue object        ───────────────→ memory-provider attachment
NAPI object         ───────────────→ polling / ownership context
page_pool object    ───────────────→ recyclable RX memory
queue statistics    ───────────────→ provider/offload observability
```

Thus Device Memory TCP, io_uring ZCRX and queue leasing should not be
described as isolated zero-copy features. They are consumers of a
broader transition in which queue and memory ownership become explicit
networking-core concepts.

## Rust --- from language support to a safe driver model

### 7.1 Linux 6.1: Rust enters the kernel

Linux 6.1 introduces the initial Rust-for-Linux support. This does not
yet mean that network drivers can generally be written in Rust;
driver-facing abstractions must be built subsystem by subsystem.

### 7.2 2023: first network-device and PHY abstraction work

A June 2023 proposal adds minimum Rust abstractions for `net_device`
drivers and a Rust dummy driver:

https://lwn.net/Articles/934517/

In parallel, PHY abstractions mature through repeated review.

### 7.3 Linux 6.8: Rust PHY support reaches mainline

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

### 7.4 Driver core, PCI, platform, DMA, MMIO and IRQ abstractions

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
future network drivers, even when developed outside `net/`.

Kernel Recipes:
https://kernel-recipes.org/en/2024/schedule/interfacing-kernel-c-apis-from-rust/
https://kernel-recipes.org/en/2025/schedule/so-you-want-to-write-a-driver-in-rust/
https://kernel-recipes.org/en/2026/schedule/enforcing-device-driver-lifecycle-rules-at-compile-time/

The 2026 driver-model work frames a major goal as converting lifecycle
conventions into compile-time invariants:

``` text
C driver:
 conventions + documentation + review

Rust driver:
 lifetime + ownership + type state
             ↓
 compile-time lifecycle constraints
```

### 7.5 Rust is not merely a C-to-Rust rewrite

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

## Conference lineage

### Netdev

Netdev is the strongest conference source for driver-framework
implementation:

``` text
2016  switchdev / hardware-offload model
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
2024  zero-copy networking + page_pool/netmem/memory providers
      Rust/C API abstraction discussion
2025  practical Rust driver development
2026  compile-time enforcement of driver lifecycle rules
```

## Revised six-axis model

The complete document can now use six interacting axes:

``` text
PERFORMANCE
  BQL → TSQ → pacing → BIG TCP

PROGRAMMABILITY
  BPF → TC → XDP → AF_XDP → struct_ops → netkit

MEMORY
  page_pool → netmem → memory providers → devmem/io_uring ZCRX

CONTROL PLANE
  rtnetlink → devlink/ethtool-netlink/YNL → fine-grained RTNL

OBSERVABILITY
  tracepoints/BTF → drop reasons → Retis/pwru → resource introspection

DRIVER FRAMEWORK
  NAPI/multiqueue
   → DQL/BQL
   → switchdev/devlink/phylink
   → XDP/DIM/page_pool
   → netdevsim/ethtool-netlink/auxiliary bus
   → queue+NAPI+memory objects
   → Rust safe driver abstractions
```

The key insight is that the driver-framework axis is not independent. It
is the layer that makes the other six axes implementable across
heterogeneous hardware without every driver reinventing the same
mechanisms.

## Topics worth a second exact-commit audit

Before assigning every item a precise kernel release/commit, a follow-up
provenance pass should enumerate:

``` text
switchdev initial core series
phylink first mainline landing
devlink initial mainline commit set
VF representor generic model
DIM / net_dim introduction
page_pool initial landing and subsequent DMA/recycle redesign
netdevsim initial landing
ethtool-netlink merge boundary
auxiliary bus exact merge
Rust PHY 6.8 exact commit set
Rust net_device / PCI / DMA / IRQ abstraction landing status
netdev-genl queue/NAPI object landing boundaries
```

As with the rest of this document, review proposals should not be
promoted to mainline facts until landing is verified.

------------------------------------------------------------------------

# Part V --- Observability / Explainability

## B. Observability / Explainability evolution

Linux networkingの進化には、performance、programmability、memory/control
planeに加えて **observability / explainability**
という独立した軸がある。

``` text
"What happened?"
      ↓
"Where did it happen?"
      ↓
"Why did this packet drop?"
      ↓
"What path did this packet take through the kernel?"
```

Retisはkernel datapathそのものではなく、eBPF、tracepoints、BTF、
`skb_drop_reason`、`struct sk_buff` metadataなど、kernel側で発達した
observability
primitivesを統合する代表的なconsumer/toolとして位置付ける。

### B.1 Counters/interface capture → kernel-internal observation

従来はinterface statistics、MIB/SNMP counters、ethtool statistics、
tcpdump/AF_PACKETなどが中心だった。これらではcounter増加と特定packetを
対応付けたり、kernel内部のどのfunctionをどう通ったかを追うのが難しい。

tracepoint、kprobe/fentry、eBPFにより:

``` text
NIC
 ↓
netif_receive_skb()  ← probe
 ↓
IP                   ← probe
 ↓
netfilter / OVS      ← probe
 ↓
TCP/UDP              ← probe
 ↓
socket
```

のようなkernel datapath内部の観測が可能になった。

### B.2 BTF --- running kernel as typed data

BTFはrunning kernelのtype informationをmachine-readableにする。

``` text
kernel types/enums/layout
          ↓
         BTF
          ↓
eBPF observability tools
```

CO-REだけでなく、runtime kernel introspectionという意味でも重要である。

### B.3 Linux 5.17 --- `skb_drop_reason`

v5.17のcommit `c504e5c2f964`で`kfree_skb_reason()`が導入された。

``` text
before:
tcp_v4_rcv → kfree_skb
             "where"は分かっても"why"が弱い

after:
tcp_v4_rcv
  → kfree_skb_reason(skb, SKB_DROP_REASON_NO_SOCKET)
  → skb:kfree_skb tracepoint
       location = tcp_v4_rcv
       reason   = NO_SOCKET
```

重要なのはnetwork stack自身がdrop decisionの意味をstructured
metadataとして trace infrastructureへ渡すようになった点である。

### B.4 Coverage expands beyond core networking

drop-reason
coverageは段階的にIP、neighbour、TCP、qdisc/device/XDP関連pathへ
拡大した。さらにnon-core reasonのruntime registrationにより、mac80211や
Open vSwitchのようなsubsystem固有reasonも表現可能になった。

``` text
core skb reasons
      ↓
protocol/path-specific reasons
      ↓
subsystem-specific semantic reasons
```

### B.5 Raw enum values are not stable ABI

`skb_drop_reason`はkernel internal enumであり、numeric valueをstable
userspace ABI として扱うべきではない。

``` text
raw integer
   ↓
running-kernel definition required
   ↓
BTF-aware decoding
```

ここがRetisとBTFが強く結び付く理由の一つ。

### B.6 Retis --- observability primitivesのintegrator

概念的には:

``` text
eBPF --------------------------┐
tracepoints/kprobes/fentry ----┤
BTF ---------------------------┤
skb_drop_reason ---------------┤
struct sk_buff metadata -------┤
conntrack / OVS state ---------┤
                               ▼
                             Retis
                               ├─ packet inspection
                               ├─ drop monitoring
                               ├─ stack/context
                               ├─ metadata filtering
                               └─ packet tracking
```

RetisはBTFを使ってrunning kernelのdrop-reason definitionを解釈するため、
kernel versionごとのraw enum値を固定tableとして仮定しない。

### B.7 Drop monitoring → packet journey

Retisの本質はdropwatchの高機能版だけではない。複数のskb-aware
function/tracepointを同時に観測し、tracking logicによって:

``` text
event A
event B
event C
event D
   ↓
same logical packet:
A → B → C → D
```

へ再構成する方向にある。

tracking IDはkernelのuniversal ABIではなくRetis側のtracking
mechanismである、 という区別は重要。

### B.8 Header filtering → kernel-metadata filtering

pcap-style packet filterに加え、BTFを利用して:

``` text
skb->dev->name
skb->mark
network namespace
nested skb/kernel metadata
```

などでfilterできる。

``` text
packet header filter
       +
kernel metadata filter
       ↓
first matching probe
       ↓
start tracking
       ↓
follow packet through later probes
```

となり、interface packet captureとは異なる観測modelになる。

### B.9 cBPF → eBPFという歴史の再接続

Retisのpcap-style filteringは、classic BPF由来のpacket-filter modelを
modern eBPF probeへ橋渡しする。

``` text
pcap-filter syntax
      ↓
classic BPF representation
      ↓
eBPF
      ↓
kernel-internal probes
```

Linux networking史の:

``` text
classic BPF → eBPF → TC/XDP/socket BPF → BPF tracing/BTF
```

がobservability tool内で再接続されている例と見ることができる。

### B.10 Why this matters for OVN/OVS/container networking

v3.x〜v4.xでnetwork datapathは:

``` text
netns / veth / tap
      ↓
OVS / OVN
      ↓
conntrack / netfilter
      ↓
routing / VRF
      ↓
VXLAN/Geneve
      ↓
physical NIC
```

のように複雑化した。

programmability、virtualization、offloadがnetworkingを強力にした一方で、
packetが「どこを通り、なぜdropされたか」を理解する難易度も上がった。
Retis型observabilityはこの複雑化への回答と位置付けられる。

### B.11 Performance evolution creates an observability requirement

``` text
virtualization / overlays
        ↓
OVS / conntrack / namespaces
        ↓
XDP / programmable BPF paths
        ↓
offload / zero-copy / device memory
        ↓
faster but more complex datapath
        ↓
eBPF tracing + BTF
        ↓
structured drop reasons
        ↓
Retis-style packet journey tracing
```

observabilityは付加的なdebug機能ではなく、programmable/heterogeneousな
network datapathを運用するためのarchitecture
capabilityへ発展したと考えられる。

### B.12 Updated six-axis model

``` text
PERFORMANCE
BQL → TSQ → pacing → BIG TCP → device memory

PROGRAMMABILITY
eBPF → XDP → AF_XDP → struct_ops → netkit

MEMORY
page_pool → netmem → Device Memory TCP / io_uring ZCRX

CONTROL PLANE
rtnetlink → YNL → per-netns/fine-grained RTNL

OBSERVABILITY / EXPLAINABILITY
tracepoints + eBPF
 → BTF
 → skb_drop_reason
 → subsystem-specific reasons
 → arbitrary-point packet inspection
 → metadata filtering + packet journey reconstruction
```

### B.14 2023 OVSCon --- Retis as an OVS kernel/userspace correlator

OVSCon 2023のRetis発表は、Retisを単なるgeneric Linux tracing
toolとしてではなく、 OVS datapath
troubleshootingのための統合observability toolとして理解する重要資料。

発表ではnetwork tracingの問題を三つに整理している。

``` text
packet mutates
    → tracking is required

many places/components
    → modular collectors are required

many packets
    → filtering is required
```

2023時点でRetisはOVS kernel datapathをfirst-class targetとして扱い、
`openvswitch:ovs_dp_upcall`などのkernel tracepointに加え、
`ovs-vswitchd`側のUSDT probesを利用してupcallを追跡していた。

``` text
OVS kernel datapath
       │
       │ flow miss
       ▼
ovs_dp_upcall                ← kernel tracepoint
       │
       ▼
netlink socket
       │
       ▼
ovs-vswitchd handler
       │
       ▼
upcall_recv                  ← USDT
       │
       ▼
classification / translation
       │
       ▼
flow_put / flow_exec         ← USDT
       │
       ▼
kernel datapath
       │
       ▼
ovs_execute_actions
```

この点は重要で、eBPF/kprobe/tracepointだけではkernel→userspace→kernelという
OVS slow path全体を一つのpacket journeyとして相関しにくい。
Retisはkernel probesとUSDTを組み合わせてこの境界を越える。

2023資料ではcollector modelも明確になっている。

``` text
skb          packet/skb information
skb-tracking logical packet ID, clone/modification tracking
skb-drop     drop reason
ovs          OVS datapath + upcall tracking
ct           conntrack state
nft          nftables table/chain/verdict
```

したがってRetisは単なるpacket dumperではなく、 **multiple networking
subsystemsのcontextを同じevent streamへ載せる** architectureを持つ。

------------------------------------------------------------------------

### B.15 OVS tracking --- packet identity is harder than skb identity

OVSCon
2023資料ではRetisのtrackingが`struct sk_buff *`だけに依存しないことも
重要である。

packetは:

``` text
NAT
clone
encapsulation
userspace upcall
```

などでrepresentationやidentityが変化する。

Retisはskb tracking ID、packet contentのhash、thread/event
orderingなどを 状況に応じて利用してOVS upcallを相関する。

これは:

``` text
same skb pointer
      ≠
same logical network packet
```

というnetwork observability上の本質的問題への対応である。

したがってpacket trackingはkernel ABIではなくtool-side
heuristic/correlation logicであり、各tracking
boundaryにはassumptionがあることを明示する。

------------------------------------------------------------------------

### B.16 2024 OVSCon --- from upcall tracing to flow enrichment

OVSCon 2024では2023年のarchitectureを維持しつつ、Retisはさらに:

``` text
skb
ct
ovs
nft
skb-drop
packet filters
metadata filters
stack traces
pcap
Python bindings
```

を統合する方向へ進んでいる。

OVS-specificな重要な進展が **OVS flow enrichment**。

Retisはkernel datapathの`ovs_flow_tbl_lookup_stats`を(kret)probeし、

``` text
UFID
flow pointer
actions pointer
lookup result
```

を取得する。

さらにruntimeでOVS unixctlへ問い合わせ:

``` text
dpctl/get-flow
ofproto/detrace
```

などを使ってkernel datapath flowをOVS/OpenFlow
representationへ関連付ける。

概念的には:

``` text
actual packet
    ↓
kernel networking event
    ↓
OVS datapath lookup
    ↓
UFID / sw_flow / actions
    ↓
Retis flow enrichment
    ↓
ODP flow/actions
    ↓
OpenFlow representation
```

となる。

これは単なる「packetはOVSを通った」という観測から、

**そのpacketがどのdatapath flow/actionと対応したか**

を説明する方向への進化。

ただし2024資料自身がflow deletion/update
trackingやupcall時のflow表示などに
制約があることを明示しており、完全なOVS state reconstructionではない。

------------------------------------------------------------------------

### B.17 `ofproto/trace` and Retis --- simulation vs live observation

OVSには以前から`ofproto/trace`という強力なtroubleshooting
mechanismがある。

役割を単純化すると:

``` text
ofproto/trace
      │
      ▼
given/synthetic packet
      │
      ▼
simulate OVS/OpenFlow processing
      │
      ▼
"OVS pipeline should do this"
```

Retisは:

``` text
actual packet
      │
      ▼
live kernel/userspace probes
      │
      ▼
observe real execution
      │
      ▼
"this packet actually did this"
```

という役割。

両者は競合するというより補完的。

``` text
               OVS troubleshooting

             ┌──────────────────┐
             │ expected behavior │
             │   ofproto/trace   │
             └────────┬─────────┘
                      │ compare
             ┌────────▼─────────┐
             │ actual behavior   │
             │      Retis        │
             └──────────────────┘
```

特にconntrack state、kernel datapath behavior、upcall、runtime
stateなどを含む 問題ではlive observationが重要になる。

一方、OpenFlow pipeline logicそのものを理解するにはsimulation-based
traceも 依然として有用。

------------------------------------------------------------------------

### B.18 2025 --- arbitrary-point packet dumping becomes user-facing

2025年1月のRed Hat Developer記事は、Retisのgeneric Linux
networking側の価値を 明確に説明している。

traditional capture:

``` text
NIC driver
    │
    ├── PF_PACKET / tcpdump
    │
network stack
```

では基本的にdriverとnetwork stackの境界付近のpacket stateを観測する。

Retis:

``` text
netif_receive_skb()   ← dump
       ↓
IP                    ← dump
       ↓
netfilter             ← dump
       ↓
OVS                   ← dump
       ↓
TCP/UDP               ← dump
       ↓
net_dev_start_xmit    ← dump
```

ではskb-aware kernel function/tracepointをcapture
pointとして選択できる。

さらに複数probeを同時に使い、packet trackingによってflowを再構成できる。

Retisで収集したpacketを`pcap`へ変換しtcpdump/Wiresharkへ渡せる点も重要。

``` text
kernel-internal capture
        ↓
      Retis
        ↓
       pcap
        ↓
tcpdump / Wireshark
```

つまり新しいkernel observabilityを既存packet-analysis
ecosystemへ橋渡ししている。

------------------------------------------------------------------------

### B.19 Revised Retis evolution

追加資料を踏まえるとRetisの発展は次のように整理できる。

``` text
Linux kernel foundations

tracepoints / kprobes
        +
       eBPF
        +
       BTF
        +
skb_drop_reason
        │
        ▼
2023 Retis / OVSCon
        │
        ├─ arbitrary kernel probes
        ├─ skb tracking
        ├─ skb-drop
        ├─ conntrack / nftables
        └─ OVS upcall tracking
             kernel ↔ ovs-vswitchd
        │
        ▼
2024 Retis / OVSCon
        │
        ├─ metadata filtering
        ├─ richer post-processing
        ├─ Python integration
        └─ OVS flow enrichment
             actual packet
                ↔ datapath flow
                ↔ OpenFlow
        │
        ▼
2025 Retis
        │
        ├─ arbitrary-point packet dumping
        ├─ pcap export
        └─ kernel metadata filtering
        │
        ▼
network-stack journey analysis
```

------------------------------------------------------------------------

### B.20 Observability evolution --- final interpretation

この資料全体ではRetisを次の位置に置く。

``` text
COUNTERS
"something happened"
      ↓
PACKET CAPTURE
"this packet crossed this interface"
      ↓
KERNEL TRACING
"this code path handled this packet"
      ↓
STRUCTURED REASONS
"this is why it was dropped"
      ↓
PACKET TRACKING
"these events belong to the same logical packet"
      ↓
CROSS-SUBSYSTEM CORRELATION
"the packet crossed IP/NF/CT/OVS..."
      ↓
KERNEL ↔ USERSPACE CORRELATION
"the OVS upcall went to ovs-vswitchd and came back"
      ↓
FLOW ENRICHMENT
"this actual packet corresponds to this OVS/OpenFlow state"
```

この意味でRetisはLinux kernel networkingの新しいforwarding
architectureではない。

**Linux networking stackが長年かけて獲得したprogrammability、typed
metadata、 tracepoints、structured drop
semanticsを統合し、複雑化したdatapathを explainableにするtooling layer**

として扱うのが最も正確。

------------------------------------------------------------------------

### B.21 Retis source trail

この系譜のRetis側の主要資料として以下を扱う。

-   Red Hat Developer (2023-07-19): *How to retrieve packet drop reasons
    in the Linux kernel*
-   Red Hat Developer (2024-01-04): *An update on packet drop reasons in
    Linux*
-   Red Hat Developer (2025-01-09): *Dumping packets from anywhere in
    the networking stack*
-   Red Hat Developer (2025-10-02): *Filtering packets from anywhere in
    the networking stack*

kernel側の中心anchorはv5.17の`kfree_skb_reason()` /
`skb_drop_reason`であり、 その後もsubsystem
coverageが継続して拡張される。

------------------------------------------------------------------------

## C. pwru and Retis --- two packet-journey observability models

pwruとRetisは競合する部分を持つが、単純な「軽量版/高機能版」という関係ではない。
両者は同じmodern Linux kernel
observability基盤から異なる探索戦略を取る。

``` text
                 eBPF + BTF
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
        pwru                   Retis
          │                     │
 broad automatic          selected probes
 function tracing         + semantic collectors
          │                     │
          ▼                     ▼
 kernel function          enriched/correlated
 trajectory               packet journey
```

### C.1 pwru --- start broad when you do not know where to look

pwruの中心的アイデアは、kernel
BTFから`struct sk_buff *`をargumentとして取る
functionsを発見し、それらへeBPF probeを広くattachすること。

``` text
kernel BTF
    ↓
find skb-accepting functions
    ↓
kprobe / kprobe-multi
    ↓
filter target packet
    ↓
print functions actually traversed
```

このmodelは、

``` text
"packetはinterfaceまで来ている"
        ↓
"でもstackのどこで問題が起きたか見当がつかない"
```

という初動調査に特に強い。

FOSDEM 2024では約1,500個規模のfunctionsへprobeをattachする例が示され、
pcap filterで対象packetだけを選択してtrajectoryを表示している。

### C.2 pwru is more than a simple skb-pointer tracer

pwruはskb pointerを表示するだけではない。

current architectureには:

-   skb clone/copy tracking
-   skb lifetime termination handling
-   stack-based tracking
-   veth/XDP→skb tracking
-   skb/netns/interface/mark filtering
-   BTF-based full skb output
-   skb metadata expressions
-   call stack/caller output
-   tunnel tuple output
-   TC/XDP BPF program visibility
-   kprobe-multi backend

などが含まれる。

したがって:

``` text
pwru = "all skb functionsにprobeするだけ"
```

と表現するのは不正確。

本質は **broad discovery-oriented tracing** にある。

### C.3 Retis --- start from events and enrich their meaning

Retisはcollector architectureを使ってeventへsubsystem
contextを追加する。

``` text
probe/event
    │
    ├─ skb
    ├─ skb tracking
    ├─ drop reason
    ├─ conntrack
    ├─ nftables
    ├─ OVS
    ├─ stack
    └─ packet
          ↓
     enriched event
          ↓
 sort/correlate/reconstruct
```

特にOVSではkernel datapathだけでなくuserspace upcallとの相関を行うため、
単一kernel function trajectoryを越えたsubsystem-specific
semanticsを扱う。

### C.4 Same foundation, different use of BTF

両者ともBTFが重要だが、使い方の重点が異なる。

``` text
pwru
BTF
 ├─ discover functions accepting skb
 ├─ understand kernel types
 └─ print skb/kernel state

Retis
BTF
 ├─ understand kernel types/enums
 ├─ decode version-dependent metadata
 └─ metadata filtering/introspection
```

この違いはBTFが単なるCO-RE portability mechanismではなく、 network
observability infrastructureへ発展したことを示す。

### C.5 pcap filter: classic BPF history reconnects to eBPF tracing

pwruのFOSDEM 2024資料はfilter compilation pathを明示している。

``` text
pcap-filter syntax
       ↓
     libpcap
       ↓
 cBPF bytecode
       ↓
     cbpfc
       ↓
 eBPF bytecode
       ↓
 tracing program
```

Retisにもpcap-style packet filteringがあり、両者は歴史的なclassic BPFの
packet-filter modelをmodern eBPF observabilityへ再接続している。

``` text
classic BPF
  │
  ├─ tcpdump / libpcap
  │
  ▼
eBPF
  │
  ├─ TC/XDP datapath programmability
  │
  └─ tracing
       │
       ├─ pwru
       └─ Retis
```

BPFは「packetをfilterする技術」から「network
datapathをprogramする技術」へ進化し、 さらにそのprogrammable
datapath自身を観測する技術としても使われるようになった。

### C.6 Tracking comparison

両者ともtrackingを行えるため、

``` text
pwru = trackingなし
Retis = trackingあり
```

という比較は誤り。

違いはtrackingの目的とcorrelation scopeにある。

``` text
pwru
packet/skbを追いながら
kernel function trajectoryを明らかにする
          │
          ▼
"where in the kernel?"

Retis
packet eventを追いながら
subsystem metadata/stateをcorrelateする
          │
          ▼
"what happened, where, and in which subsystem context?"
```

どちらもclone、representation change、XDP↔skbなどpacket
identityが変化する 境界にはtool-side logic/assumptionが必要になる。

### C.7 OVS is the clearest architectural difference

OVS troubleshootingでは違いが分かりやすい。

``` text
                pwru

packet
  ↓
OVS kernel functions
  ↓
ovs_flow_tbl_lookup...
  ↓
ovs_execute_actions...
  ↓
kernel function trajectory
```

Retis:

``` text
packet
  ↓
OVS kernel datapath
  ↓
flow miss
  ↓
ovs_dp_upcall
  ↓
       kernel/userspace boundary
  ↓
ovs-vswitchd USDT
  ↓
translation / flow install
  ↓
kernel datapath
  ↓
OVS flow enrichment
```

pwruでもOVS kernel functionsを観測できるが、 RetisのOVS
collectorはOVS固有のupcall semanticsやuserspace correlationを
明示的にmodel化する。

ここが「generic broad tracing」と「subsystem-aware
correlation」の典型的な差。

### C.8 Comparison matrix

  -----------------------------------------------------------------------
  Aspect                  pwru                    Retis
  ----------------------- ----------------------- -----------------------
  Primary model           broad kernel-function   event/collector-based
                          tracing                 correlation

  Starting point          target packet, location probes/events +
                          unknown                 collectors

  Probe discovery         BTFからskb              configured/selected
                          functionsを広く発見     probes and profiles

  kprobe-multi            supported               different probe
                                                  architecture

  Packet filter           pcap-style              pcap-style

  Kernel metadata         skb/BTF/expressions     skb/BTF metadata
                                                  filters

  skb tracking            yes                     yes

  clone handling          yes                     tracking/correlation
                                                  logic

  XDP tracking            supported               XDP/kernel probes
                                                  depending on collection

  Drop reason             can observe/output      dedicated skb-drop
                          relevant path/state     semantics

  Conntrack               generic tracing/state   dedicated collector
                          inspection possible     

  nftables                generic kernel          dedicated semantic
                          trajectory              collector

  OVS kernel path         visible                 dedicated OVS collector

  OVS userspace upcall    not the central model   explicit
                                                  kernel↔ovs-vswitchd
                                                  correlation

  Post-processing         trajectory-oriented     collect → store →
                          output/JSON             sort/reconstruct

  Best first question     "where did this packet  "what happened in this
                          go?"                    subsystem context?"
  -----------------------------------------------------------------------

この表は絶対的な機能境界ではない。両projectとも進化しており、
overlapする機能は多い。比較軸は「できる/できない」より**design
center**。

### C.9 Practical troubleshooting model

典型的には次のような使い分けが理解しやすい。

``` text
connectivity failure
       ↓
location completely unknown
       ↓
      pwru
       ↓
discover suspicious region:
 nf_hook_slow?
 ovs_execute_actions?
 routing?
 kfree_skb_reason?
       ↓
subsystem identified
       ↓
Retis or subsystem-specific tools
       ↓
enrich with:
 skb/drop reason
 conntrack
 nftables
 OVS/upcall
 netns/device
```

ただしこれは必須workflowではない。
Retisだけで最初から追跡することも、pwruだけでroot
causeへ到達することもある。

### C.10 Evolutionary interpretation

Linux networking historyの観点では両者を同じbranchに置く。

``` text
classic BPF
    ↓
eBPF
    ↓
BTF + CO-RE + tracing infrastructure
    ↓
kernel becomes dynamically introspectable
    │
    ├─────────────────────┐
    ▼                     ▼
  pwru                  Retis
    │                     │
discover broadly      correlate semantically
    │                     │
    └──────────┬──────────┘
               ▼
       packet journey debugging
```

これはmodern Linux networkingの重要な変化。

``` text
programmable datapath
        ↓
datapath complexity increases
        ↓
BPF/BTF-based observability
        ↓
tools can discover and explain
the running kernel dynamically
```

つまりpwru/Retisはkernel networking featureそのものではないが、
**eBPF/BTFによってLinux kernelが「実行中に探索可能なnetwork platform」へ
変化したことを象徴するtools** と位置付けられる。

### C.11 Conference provenance

pwruについては特に以下をdesign/architecture evidenceとして扱う。

-   FOSDEM 2024, Quentin Monnet: *Packet, where are you? Track in the
    stack with pwru*
-   Linux Plumbers Conference 2024: pwru architecture/evolution
    presentation

Retisについては前章の:

-   OVSCon 2023
-   OVS/OVN Conf 2024
-   Red Hat Developer 2023--2025

と対にして扱う。

この組み合わせによりconference provenanceも:

``` text
FOSDEM/LPC
   pwru generic kernel tracing
          │
          ├── eBPF/BTF foundation ──┐
          │                         │
OVSCon                              │
   Retis OVS-aware correlation ─────┘
          │
          ▼
Linux networking observability evolution
```

として整理できる。

------------------------------------------------------------------------

# Part VI --- Synthesis: v3.x → 7.x を一つの進化として見る

## v5.0 と 2026 を比較する

``` text
                     v5.0                    2026

packet memory        struct page      →      page_pool / netmem / providers
fast path            XDP/AF_XDP       →      AF_XDP MB / netkit / queue lease
aggregation          GRO/GSO          →      BIG TCP / tunnel BIG TCP
BPF                   packet/socket    →      struct_ops/netkit/BPF qdisc
virtual networking   veth/virtio      →      vDPA/AF_XDP ZC/netkit
control locking      global RTNL      →      per-netns/fine-grained
Netlink API           hand-written     →      YNL-described
memory path          NIC→RAM          →      NIC→RAM or device memory
async I/O             conventional     →      io_uring ZC TX/RX
```

最大の変化は、network stack が **「CPU が system RAM 上の skb
を逐次処理する単一モデル」から離れたこと** にある。

------------------------------------------------------------------------

## Evolution map

``` text
                    Linux v5.0
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
      XDP/BPF         GRO/GSO          skb/page
        │               │                │
        ▼               ▼                ▼
   struct_ops        BIG TCP          page_pool
   SK_LOOKUP             │                │
        │                │             netmem
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

この後には、元の
`linux-networking-lwn-change-log-2019-2026-unified-audited.md` を
**Research Appendix** としてそのまま保持する。

推奨順序:

1.  Sections 1--14 で v5.0 → 7.x の進化を把握
2.  Appendix の release chronology で release landing を確認
3.  commit-level dossier で exact SHA / patch series を確認
4.  LWN / conference provenance で設計意図と後続 evolution を確認

------------------------------------------------------------------------

# Part VII --- Canonical provenance ledger

This section records only unresolved or especially important attribution
boundaries. The detailed research diary from the re-audit passes is
intentionally omitted from the clean edition.

  -----------------------------------------------------------------------
  Topic                               Canonical status
  ----------------------------------- -----------------------------------
  BQL/DQL                             Linux 3.3 mainline generation; DQL
                                      exact anchor retained

  SO_REUSEPORT                        Linux 3.9; infrastructure exact
                                      anchor retained

  switchdev                           3.19 origin; later 4.x work treated
                                      as expansion

  devlink                             Linux 4.6 release origin; exact
                                      initial commit set still pending

  phylink                             2015 RFC → Linux 4.13
                                      infrastructure; exact anchor
                                      retained

  MSG_ZEROCOPY                        TCP foundation 4.14; UDP extension
                                      5.0

  SK_MSG                              4.17 generation; final series
                                      verified, core SHA still pending

  page_pool                           4.18 refurbished/XDP-memory-return
                                      generation; do not claim all
                                      page_pool ideas originated there

  ethtool Generic Netlink             5.6 generation; exact
                                      core-interface anchor retained

  SK_LOOKUP                           5.9; exact program-type anchor
                                      retained

  auxiliary bus                       5.11; final series/tag verified,
                                      exact feature-origin SHA pending

  XFRM packet offload                 6.2; exact `XFRM_OFFLOAD_PACKET`
                                      anchor retained

  Rust PHY                            2023 RFC/final v11 → Linux 6.8;
                                      per-patch mainline SHA set pending

  queue/NAPI objects                  6.8 documentation-level milestone;
                                      later write/configuration APIs must
                                      not be backdated

  7.x development                     released 7.2 separated from
                                      7.3-development/net-next work
  -----------------------------------------------------------------------

## Evidence grades

``` text
A  exact mainline SHA and/or tag containment + release evidence
B  final/accepted series + release evidence
C  development status or architecture verified; exact landing incomplete
D  RFC/design/proposal only
```

The canonical chronology in Part II takes precedence over all provenance
notes.

# Appendix — Evidence catalog and research notes (non-normative)

This appendix is a compact evidence catalog. It intentionally omits the old duplicate release chronology, intermediate audit passes, completeness matrices, and “next pass” notes. Part II is the only normative release map.

The retained material serves four purposes: source navigation, exact-commit dossiers, conference provenance, and cross-source lineage reconstruction. RFC/review dates in this appendix must not be read as released-kernel attribution.

## LWN reading list --- core articles

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


## Tag index

### `BPF`

-   5.3 socket/cgroup hooks
-   5.4 XDP/TC integration
-   5.6 `struct_ops`
-   5.9 socket lookup
-   5.10 TCP options
-   6.6 MPTCP/defrag hooks
-   6.7 netkit
-   6.15 network timestamp callbacks

### `TCP`

-   zero-copy RX
-   BPF congestion control
-   BIG TCP
-   MPTCP interaction
-   Device Memory TCP
-   `TCP_RTO_MAX_MS`
-   AccECN

### `UDP`

-   GRO/GSO
-   high packet-rate receive optimization
-   receive-buffer sizing
-   UDP-based tunnel/transport work

### `XDP` / `AF_XDP`

-   XDP buffer infrastructure
-   AF_XDP
-   AF_XDP multi-buffer
-   netkit/BPF datapath

### `zero-copy`

-   TCP zero-copy
-   io_uring zero-copy TX
-   io_uring zero-copy RX
-   Device Memory TCP
-   DIBS

### `virtual-networking`

-   veth
-   tunnel/overlay
-   netkit
-   virtio-net/TAP
-   KubeVirt-oriented zero-copy work

------------------------------------------------------------------------


## Commit-level research status

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


## Sources / entry points

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

### Notes on completeness

この change log は「LWN の記事タイトル一覧」ではなく、2019-05-07 以降の
Linux networking
の重要な変更を技術系列として再構成することを目的としている。そのため、個別
driver の通常更新は除外する一方、独立した LWN feature がなくても
merge-window coverage に現れる architecture-level change は含める。

commit-level の項目は upstream で確認できたものから順次追加し、LWN
の記述だけから hash を逆算・推測しない。

------------------------------------------------------------------------


## Commit-level history --- verified entries

この章では、upstream patch archive / git history で commit ID
まで確認できた系列を記録する。 短縮 hash
だけが一次資料に掲載されている場合は短縮形をそのまま記載し、推測で full
hash に展開しない。

### 13.1 BIG TCP --- Linux 5.19

**Feature:** BIG TCP (initial IPv6 support)\
**Kernel:** Linux 5.19\
**Subsystem:** `net/core`, IPv6, TCP, GRO/GSO\
**Primary motivation:** 64KiB を超える kernel-internal GRO/GSO aggregate
を利用し、高速 TCP datapath の per-packet overhead を削減する。

#### Mainline evidence

5.19 の networking pull request は、IPv6 Jumbogram extension header
を利用して 64KiB より大きな TCPv6 GSO super-segment
をサポートする機能を明示的に `BIG TCP` として記載している。

-   Networking pull request:
    https://lists.openwall.net/netdev/2022/05/24/216
-   LWN feature: https://lwn.net/Articles/884104/
-   LWN 5.19 merge-window coverage: https://lwn.net/Articles/896140/

#### Verified commits

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

#### Later evolution

-   IPv4 BIG TCP
-   GRO validation / HBH handling の再設計
-   AF_XDP multi-buffer との整合
-   overlay/tunnel datapath への拡張

------------------------------------------------------------------------

### 13.2 AF_XDP multi-buffer --- Linux 6.6

**Feature:** AF_XDP multi-buffer RX/TX\
**Kernel:** Linux 6.6\
**Subsystem:** `net/xdp`, AF_XDP, Intel `ice` / `i40e` initial driver
support\
**Patch series:** `[PATCH v7 bpf-next 00/24] xsk: multi-buffer support`

#### Development

v7 series は 24 patches から構成され、core AF_XDP support、zero-copy、
driver support、documentation/selftests をまとめて導入した。

Patch series:

-   https://lists.openwall.net/netdev/2023/07/19/282

#### Verified core commits

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

#### Architecture

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

#### Important files

-   `net/xdp/xsk.c`
-   `net/xdp/xsk_buff_pool.c`
-   `net/xdp/xsk_queue.h`
-   `include/net/xsk_buff_pool.h`
-   `Documentation/networking/af_xdp.rst`
-   `Documentation/netlink/specs/netdev.yaml`

#### Follow-up / maintenance evidence

2026 年の修正でも `Fixes: cf24f5a5feea` が使われており、TX multi-buffer
introduction point を独立に確認できる。

------------------------------------------------------------------------

### 13.3 netkit --- Linux 6.7

**Feature:** BPF-programmable `netkit` virtual network device\
**Kernel:** Linux 6.7\
**Author:** Daniel Borkmann\
**Subsystem:** BPF / virtual networking / container datapath

#### Development

merge 直前の series:

-   `[PATCH bpf-next v4 1/7] netkit, bpf: Add bpf programmable net device`
-   Date: 2023-10-24
-   Patch: https://lists.openwall.net/netdev/2023/10/24/365

#### Verified mainline commit

`35dfaad7188cdc043fde31709c796f5a692ba2bd`

Subject:

``` text
netkit, bpf: Add bpf programmable net device
```

この commit の説明では、BPF program を driver の `xmit` routine
内で実行し、 Pod/container egress で BPF processing を packet source
に近づけること、 さらに物理 device へ直接 redirect する場合に per-CPU
backlog queue を経由しない ことが目的として説明されている。

#### Datapath implication

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

#### References

-   LWN: https://lwn.net/Articles/949960/
-   v4 patch: https://lists.openwall.net/netdev/2023/10/24/365
-   commit mirror:
    https://git.zx2c4.com/linux-rng/commit/?id=35dfaad7188cdc043fde31709c796f5a692ba2bd

#### Follow-ups

2026 年には netkit queue leasing と io_uring zero-copy RX の integration
が進み、 network namespace 内の guest/VM datapath
にまで対象が広がっている。

------------------------------------------------------------------------

### 13.4 Device Memory TCP RX --- Linux 6.12

**Feature:** Device Memory TCP receive\
**Kernel:** Linux 6.12\
**Authors:** Mina Almasry, Willem de Bruijn et al.\
**Subsystem:** TCP / netdev / DMA-BUF / page-pool / device memory

#### Mainline state

Linux kernel documentation は Device Memory TCP を、TCP socket
で受信した data を DMA-BUF-backed device memory
へ直接配置する機能として説明している。

-   Kernel documentation:
    https://kernel.org/doc/html/latest/networking/devmem.html
-   LWN 6.12 merge window: https://lwn.net/Articles/990750/

#### Architecture

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

#### Source areas

Current upstream tree contains Device Memory TCP infrastructure
including:

-   `net/core/devmem.h`
-   networking device-memory support
-   page-pool/netmem integration
-   TCP receive-side APIs
-   `Documentation/networking/devmem.rst`

#### Commit verification status

6.12 への feature merge 自体と source/documentation は確認済み。 ただし
RX series は多数の preparatory commits に分割されているため、
**individual commit list はまだ「series 全体として確定」していない**。
単一 commit を Device Memory TCP RX の introduction commit
と誤って表記しない。

------------------------------------------------------------------------

### 13.5 Device Memory TCP TX --- Linux 6.16

**Feature:** Device Memory TCP transmit\
**Kernel:** Linux 6.16\
**Subsystem:** TCP / DMA-BUF / device-memory / zero-copy TX

RX support は 6.12 に入ったが、TX support は review を分離して後続
series となった。 2025-05 時点で TX patch set は net-next に queue
され、6.16 cycle 向けとなった。

#### Significance

``` text
RX (6.12)
network → NIC → device memory

TX (6.16)
device memory → NIC → network
```

これにより Device Memory TCP は device memory を network endpoint の
data buffer として双方向に利用する方向へ進んだ。

#### Verification status

-   RX が 6.12、TX が 6.16 という release separation は確認済み。
-   TX series は多数 revision を経ている。
-   exact mainline commit list は次の commit-level pass で確定する。

------------------------------------------------------------------------


## Verification rules used in this document

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


## Netdev conference cross-reference (2019-05-07--2026-10-02)

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

### 58.1 Netdev 0x14 --- 2020

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

### 58.2 Netdev 0x15 --- 2021

Accepted sessions: https://netdevconf.info/0x15/accepted-sessions.html

#### BIG TCP --- Eric Dumazet

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

#### Accelerating synproxy with XDP

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

#### Resilient nexthop groups

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

#### Rethinking Zero-Copy Networking with MAIO

This is relevant as an alternative high-performance networking design in
the same period that eventually produced stronger kernel-side zero-copy
efforts around io_uring, page_pool/netmem and Device Memory TCP.

#### TC / ACL / BPF

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

### 58.3 Netdev 0x16 --- 2022

Conference: https://netdevconf.info/0x16/

#### Merging the Networking Worlds --- David Ahern, Shrijeet Mukherjee

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

#### Historical significance

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

### 58.4 Netdev 0x17 --- 2023

Sessions: https://netdevconf.info/0x17/pages/sessions.html

This edition has unusually strong overlap with the mainline networking
developments covered in this document.

#### Device Memory TCP

Speakers:

-   Mina Almasry
-   Willem de Bruijn
-   Eric Dumazet
-   Kaiyuan Zhang

This is the direct conference counterpart to the Device Memory TCP
RFC/mainline lineage.

#### Fast ZC Rx Data Plane using io_uring / Zero Copy Receive using io_uring

Session:
https://netdevconf.info/0x17/sessions/talk/zero-copy-receive-using-io_uring.html

Speakers:

-   David Wei
-   Pavel Begunkov

The description explicitly frames memory bandwidth as a bottleneck and
compares the kernel socket copy path with kernel bypass and RDMA.

This is one of the most useful primary sources for the later Linux 6.15
io_uring ZCRX feature.

#### eBPF Qdisc: a generic building block for traffic control

Speakers:

-   Hsin-Wei (Amery) Hung
-   Cong Wang

This is a direct precursor/context source for the later BPF qdisc /
`struct_ops` development.

#### Integrating eBPF Into The P4TC Datapath

This provides the conference-side history for the P4TC proposal that
later encountered maintainer resistance and was covered by LWN in 2024.

#### Netlink APIs to Expose/Configure Netdev Objects

This belongs alongside:

``` text
Generic Netlink
      │
YNL specifications
      │
modern netdev configuration APIs
```

#### Other relevant sessions

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

### 58.5 Netdev 0x18 --- 2024

Schedule: https://netdevconf.info/0x18/pages/schedule.html

#### Devmem TCP & io uring zero copy --- BoF

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

#### The Future of AI Networks: Advancing TCP with Device Memory and Collective Communication

This extends Device Memory TCP from a pure networking optimization into
AI/GPU/collective communication use cases.

#### A new lightweight Zero-Copy Notification Mechanism in Linux

Relevant to the broader zero-copy TX/RX completion/lifetime problem.

#### Characterizing IOTLB Wall for Multi-100-Gbps Linux-based Networking

Relevant to DMA/IOMMU overhead in high-speed networking and therefore
complementary to page_pool/netmem/device-memory work.

#### Fine-grained TCP Tuning

Relevant to the growing set of per-socket TCP controls and later
RTO/receive-side tuning changes.

#### Workshops/BoFs

-   Extension Headers Workshop
-   TC Workshop
-   IPsec Workshop

------------------------------------------------------------------------

### 58.6 Netdev 0x19 --- 2025

Schedule: https://netdevconf.info/0x19/pages/schedule.html

Sessions: https://netdevconf.info/0x19/pages/sessions.html

This edition maps closely to the 2025 chapters in the change log.

#### Diagnosing Page Pool Leaks

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

#### MPTCP: present, future, and its development workflow

Directly complements the MPTCP release/patch history.

#### SRv6 in Linux Kernel, FRR and eBPF

Directly complements the SRv6 Headend Reduced / PSP / NEXT-C-SID
evolution.

#### Communication via Internal Shared Memory (ISM) - Time to open up

This is especially interesting in light of the later DIBS/shared-memory
transport work.

The conceptual lineage is not `netmem → DIBS`; instead:

``` text
ISM / SMC-D / local shared-memory transports
              │
              ▼
DIBS / later shared-memory socket ideas
```

#### mq-cake: Scaling software rate limiting across CPU cores

This is the conference-side precursor/context for CAKE multiqueue work
that later appears in the Linux 7.0-era change log.

#### Linux Kernel Support for IOAM Direct Exporting

Follow-up to the earlier IPv6 IOAM implementation.

#### The Battle Of The ZCs: Who is the prettiest of them all?

Useful comparison material for the multiple Linux zero-copy mechanisms
that this document otherwise follows separately.

#### IRQ Suspension: a new, efficient mechanism for packet delivery

Relevant to the NAPI/softirq/busy-poll execution-model evolution.

#### The future of SO_TIMESTAMPING

Complements the packet/network timestamp observability lineage.

#### State of the union in TCP land --- Eric Dumazet

Useful cross-reference for contemporary TCP performance/API work,
including the period around AccECN, receive-side optimization, and timer
tuning.

------------------------------------------------------------------------

### 58.7 Netdev 0x1A --- 2026

Sessions: https://netdevconf.info/0x1A/pages/sessions.html

Netdev 0x1A is inside the requested time window and is especially
valuable because its materials were available by August 2026.

#### io_uring ZCRX: Progress and Next Steps --- Pavel Begunkov

The session covers the post-merge ZCRX problems:

-   refill-queue exhaustion;
-   buffers that cannot immediately be recycled;
-   sharing NIC queues between processes;
-   memory-pressure detection;
-   API/performance follow-ups.

This is the natural follow-up to Linux 6.15 ZCRX.

#### Accelerating Software RDMA (RXE) with Netkit and Devmem

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

#### Kernel shared memory socket transport --- David Wei

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

#### Linux QUIC: Bringing a Modern Secure Transport into the Kernel

Direct conference counterpart to the 2024--2026 kernel QUIC RFC series.

#### TCP State of the union (2026) --- Eric Dumazet

Session:
https://netdevconf.info/0x1A/sessions/talk/tcp-state-of-the-union-2026.html

The talk explicitly focuses on recent/upcoming TCP changes and
performance on modern platforms, making it a useful companion source for
the late TCP chapters.

#### Thrice the charm: an skb extension for BPF metadata

Relevant to the long-running question of how metadata is carried with
skb/BPF datapaths.

#### Securing IOAM in the Linux Kernel

Follow-up to IOAM deployment: telemetry integrity/trust rather than
merely carrying trace data.

#### Network Observability BoF

Complements:

-   `skb_drop_reason`;
-   SO_TIMESTAMPING;
-   BPF network timestamp callbacks;
-   modern tracing/Retis work.

#### Other relevant 0x1A sessions

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


## Netdev ↔ LWN ↔ mainline lineage map

The conference material makes several feature histories much clearer.

### BIG TCP

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

### Zero-copy receive / Device Memory TCP

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

### BPF traffic control

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

### P4TC

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

### Shared-memory networking

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


## Recommended Netdev talks for this change log

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


## Netdev coverage note

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


## Provenance index --- Netdev → LWN → patch series → mainline

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


## Detailed provenance chains

### 63.1 BIG TCP

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

### 63.2 Device Memory TCP + io_uring ZCRX

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

### 63.3 BPF qdisc

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

### 63.4 MPTCP

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

### 63.5 IOAM

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

### 63.6 KTLS / kernel handshake

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

### 63.7 P4TC vs BPF qdisc

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


## Canonical inventory --- Netdev counterpart field

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


## How to use the provenance index

The document can now be read in three directions.

#### Start from a kernel release

``` text
Linux 6.15
   │
   └── io_uring ZCRX
          │
          ├── LWN merge-window
          ├── lore patch series
          └── Netdev 0x17 → 0x18 → 0x1A
```

#### Start from an LWN article

``` text
LWN "Going big with TCP packets"
   │
   ├── Netdev 0x15 design talk
   ├── Linux 5.19 merge
   ├── IPv4 follow-up
   └── tunnel follow-up
```

#### Start from a Netdev talk

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


## Kernel Recipes cross-reference

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

### 104.1 In-scope conference talks found so far

The document's time range begins on 2019-05-07. Therefore Kernel Recipes
2019 is in scope.

#### Kernel Recipes 2019 --- XDP closer integration with network stack

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

#### Relation to the change log

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

#### Kernel Recipes 2019 --- BPF at Facebook

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

#### Relation to the change log

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

### 104.2 Kernel Recipes 2023 --- Netconf 2023 Workshop

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

#### Willem de Bruijn

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

#### Daniel Borkmann

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

#### Eric Dumazet

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

#### David Ahern

The workshop summary lists:

``` text
Linux TCP for machine-learning use cases
BIG TCP for IPv6
zero-cost counters for userspace monitoring
```

The ML/TCP discussion is particularly relevant to the later Device
Memory TCP and AI-networking work.

#### Florian Westphal

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

#### Other networking topics

The slides also mention:

``` text
SYN proxy at scale with BPF
TCP extended data offset
software simulation of hardware offloads
netdev feature expansion
driver review / devlink device orchestration
DPU/IPU modeling
```

#### Provenance role

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


## Cross-category Kernel Recipes talks relevant to networking

Category filtering alone is not sufficient.

### 105.1 Kernel Recipes 2019 --- Faster IO through io_uring

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

### 105.2 Kernel Recipes 2022 --- What's new with io_uring

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

### 105.3 Kernel Recipes 2023 --- On the way to io_uring networking

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


## Kernel Recipes → LWN → mainline mapping

  -----------------------------------------------------------------------------
  Kernel           Year Archive      Change-log lineage  Phase
  Recipes               category                         
  ------------- ------- ------------ ------------------- ----------------------
  XDP closer       2019 networking   XDP →               design/architecture
  integration                        page_pool/AF_XDP →  
  with network                       netkit              
  stack                                                  

  BPF at           2019 networking   BPF networking →    deployment/design
  Facebook                           struct_ops/socket   
                                     hooks/netkit        

  Faster IO        2019 storage      io_uring →          pre-networking
  through                            networking TX/RX    foundation
  io_uring                                               

  What's new       2022 storage      io_uring → ZC       foundation/merge-era
  with io_uring                      networking          

  On the way to    2023 storage      io_uring ZC TX/RX   design/merge-era
  io_uring                                               
  networking                                             

  Netconf 2023     2023 networking   SO_DEVMEM, BIG TCP, multi-lineage workshop
  Workshop                           XDP/BPF, IPsec,     
                                     nftables, TCP/ML    
  -----------------------------------------------------------------------------

------------------------------------------------------------------------


## Kernel Recipes and Netdev serve different provenance roles

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

------------------------------------------------------------------------


## Linux Plumbers Conference cross-reference --- 2019--2025

The Linux Plumbers Conference (LPC) is now added as a fourth
conference/review layer.

Unlike Netdev, LPC often combines networking with BPF and
cross-subsystem discussions, and many sessions are explicitly intended
to discuss work that is not yet finished.

### 113.1 LPC 2019 --- Networking Summit

The 2019 Networking Summit already contains several lineages that later
become major parts of this change log:

``` text
Multipath TCP Upstreaming
Programmable socket lookup with BPF
XDP bulk packet processing
netfilter hardware offloads
SwitchDev offload optimizations
Linux kernel VXLAN + multicast routing
```

#### MPTCP

``` text
LPC 2019
Multipath TCP Upstreaming
        │
        ▼
LWN upstreaming coverage
        │
        ▼
initial upstream series
        │
        ▼
Linux 5.6 native MPTCP
```

This is strong pre-merge provenance for the initial MPTCP generation.

#### BPF socket lookup

The "Programmable socket lookup with BPF" session predates the later
`BPF_PROG_TYPE_SK_LOOKUP` mainline API and belongs directly in the
socket-programmability lineage.

#### Netfilter hardware offload

This provides conference provenance for the same offload direction
covered by LWN's 2020 two-part netfilter hardware-offload series.

------------------------------------------------------------------------

### 113.2 LPC 2020 --- Networking and BPF Summit

Important networking sessions include:

``` text
Multiple XDP programs on a single interface
A programmable Qdisc with eBPF
Kubernetes service load-balancing at scale with BPF & XDP
BPF extensible network: TCP header option, CC, and socket local storage
Userspace OVS with HW Offload and AF_XDP
```

#### BPF qdisc

The 2020 "A programmable Qdisc with eBPF" talk is a particularly useful
early ancestor of the later BPF-qdisc work.

It should be classified as:

``` text
LPC 2020
programmable qdisc design
        │
        ▼
later BPF/net_sched iterations
        │
        ▼
Netdev 0x17 eBPF Qdisc
        │
        ▼
mainline BPF Qdisc
```

It is **design lineage**, not evidence that the later `Qdisc_ops`
struct_ops design had already landed in 2020.

#### BPF TCP extensibility

The TCP header option / congestion-control / socket-local-storage
session connects to:

``` text
BPF struct_ops TCP congestion control
BPF TCP options
socket local storage
later MPTCP/BPF extensions
```

------------------------------------------------------------------------

### 113.3 LPC 2021 --- BPF & Networking Summit

Important sessions include:

``` text
Socket migration for SO_REUSEPORT
BPF-datapath extensions for Kubernetes workloads
bpfilter - BPF based firewall
Dynamic Encapsulation Using eBPF
From XDP to Socket
Bringing TSO/GRO and Jumbo frames to XDP
```

#### SO_REUSEPORT socket migration

The talk explicitly describes the Linux 5.14 socket-migration feature
and its BPF extension.

This maps directly to the LWN `SO_REUSEPORT` failover coverage.

Phase:

``` text
merge/post-merge explanation
```

#### XDP multi-buffer precursor

"Bringing TSO/GRO and Jumbo frames to XDP" states the core limitation
clearly:

``` text
XDP single-buffer packet model
        │
        ├── simple memory model
        └── fast direct packet access
```

but it prevents straightforward support for:

``` text
jumbo frames
TSO/GRO-style packets
header/data split
non-linear frames
```

This is direct design provenance for the later:

``` text
XDP non-linear frames
        ↓
AF_XDP multi-buffer
        ↓
virtio-net multi-buffer / zero-copy interactions
```

------------------------------------------------------------------------


## LPC 2022 --- eBPF & Networking

LPC 2022 is one of the richest years for the change-log lineages.

Sessions include:

``` text
Tuning Linux TCP for data-center networks
Can the Linux networking stack be used with very high speed applications?
Machine readable description for netlink protocols (YAML?)
Cilium's BPF kernel datapath revamped
A BPF map for online packet classification
Bringing packet queueing to XDP
XDP gaining access to NIC hardware hints via BTF
MPTCP: Extending kernel functionality with eBPF and Netlink
```

### 114.1 High-speed Linux TCP

David Ahern's talk quantifies the scaling problem at 400 Gb/s:

``` text
1500-byte MTU:
~33 million packets/sec
~one packet every 30 ns
```

and frames the key question:

``` text
Can the normal Linux IP/TCP stack scale to modern line rates?
```

This belongs in the same architectural family as:

``` text
GRO/GSO
    ↓
BIG TCP
    ↓
zero-copy
    ↓
Device Memory TCP
```

rather than treating those later mechanisms as isolated optimizations.

### 114.2 Machine-readable Netlink protocols

The LPC session:

``` text
Machine readable description for netlink protocols (YAML?)
```

is direct pre-merge/design provenance for the Netlink YAML/YNL work.

Lineage:

``` text
LPC 2022
machine-readable Netlink proposal
       │
       ▼
YAML protocol specs
       │
       ▼
generated policy/UAPI/docs
       │
       ▼
YNL userspace tooling
       │
       ▼
MPTCP / nftables and other conversions
```

### 114.3 MPTCP + BPF/Netlink

The MPTCP session explicitly says:

``` text
MPTCP initial support: Linux 5.6
        │
        ▼
core functionality grows
        │
        ▼
runtime extensibility
        ├── Generic Netlink
        └── BPF
```

This aligns almost exactly with the commit provenance already
reconstructed in this document:

``` text
5.6 initial MPTCP
       ↓
2022 mptcp_sock BPF support
       ↓
update_socket_protocol()
       ↓
later iterator/kfunc work
```

### 114.4 XDP hardware hints / metadata

The BTF hardware-hints session identifies several consumers:

``` text
BPF programs
XDP→skb conversion
AF_XDP userspace
chained BPF programs
```

This is early provenance for the packet-metadata problem that remains
active in later LPC sessions.

------------------------------------------------------------------------


## LPC 2023 --- eBPF & Networking

A particularly important session is:

``` text
Zero Copy Receive using io_uring
David Wei / Pavel Begunkov
```

The talk states the core problem:

``` text
NIC DMA
  ↓
kernel memory
  ↓ copy
userspace memory
```

and contrasts kernel-socket networking with kernel-bypass approaches
such as:

``` text
DPDK
PF_RING
AF_XDP
RDMA / RoCE / InfiniBand
```

The design goal is to preserve the generic kernel socket/networking
stack while removing the extra data copy.

#### Provenance chain

``` text
Kernel Recipes 2022
future ZC RX design
        │
        ▼
Kernel Recipes 2023
io_uring networking
        │
        ▼
LPC 2023
Zero Copy Receive using io_uring
        │
        ▼
Netdev discussions
        │
        ▼
LWN RFC → v13 history
        │
        ▼
Linux 6.15
io_uring ZCRX
```

This gives unusually strong multi-conference provenance for the feature.

------------------------------------------------------------------------


## LPC 2024 --- Networking Track

LPC 2024 has a dedicated Networking Track described by the organizers as
an in-person manifestation of the netdev mailing list.

Important sessions include:

``` text
Per Netns RTNL
What makes the panda sad in the Linux network stack today?
Reducing the Overhead of Network Virtualization
SMC-ERM
Automatically reasoning about the cache usage of network stacks
Netdev CI
WireGuard & GRO?
State of the Bloat
```

### 116.1 Per Netns RTNL

This is one of the strongest LPC matches to the existing change log.

The session explicitly describes `rtnl_lock()` as the networking
subsystem's "Big Kernel Lock" and notes that work to avoid it for some
requests dates back to Linux 4.14.

Lineage:

``` text
global RTNL
    │
    ├── unlocked request flags
    ├── RCU reader conversion
    │
    ▼
LPC 2024
Per Netns RTNL
    │
    ▼
Linux 6.13
per-netns RTNL infrastructure
    │
    ▼
6.15+ continued RTNL removal
    │
    ▼
subsystem-specific lock/refcount conversion
```

This directly validates the document's decision to treat RTNL breakup as
a **multi-year migration**, not a single-release feature.

### 116.2 Network virtualization overhead

The network-virtualization session discusses reducing the cost of
traversing both guest and host networking stacks.

It belongs in the broader lineage:

``` text
virtio-net
   ↓
XDP / AF_XDP
   ↓
virtio-net AF_XDP zero-copy
   ↓
netkit
   ↓
queue leasing / KubeVirt zero-copy
```

------------------------------------------------------------------------


## LPC 2025 --- Networking Track

The 2025 Networking Track is exceptionally relevant to the late part of
this change log.

Sessions include:

``` text
Networking track kick-off
SUPERp: Scale UP EtherRnet Protocol
Linux firewall self-adjusting data structures
Congestion Signaling (CSIG) for Linux TCP Data Center Networking
Running PTP at scale
Packet Metadata - Where Are Thee?
The case for zero-copy in containers
Kernel-Native Packet Processing on AMD GPUs
Linux Networking with MANA: RX Path Optimization and Netshaper
```

### 117.1 Zero-copy in containers / netkit queue leasing

This is the most important LPC 2025 session for the current document.

The talk proposes leasing a physical NIC hardware queue to a virtual
device such as `netkit`, so applications inside network namespaces/Pods
can use:

``` text
io_uring zero-copy
Device Memory TCP
AF_XDP
```

For KubeVirt, the intended path is:

``` text
physical NIC queue
       │ lease
       ▼
netkit
       │
       ▼
AF_XDP
       │
       ▼
QEMU/KVM
```

The talk also identifies the need for:

``` text
AF_XDP TX policy hook
bpf_mprog conversion
multi-attach
queue-range BPF attachment
TX-side BPF attachment
```

#### Exact historical relationship

``` text
LPC 2025
zero-copy in containers design
        │
        ▼
netkit queue leasing series
        │
        ├── first merge
        ├── revert
        ├── redesign
        └── re-merge
        │
        ▼
2026 Netdev / LWN
netkit + KubeVirt zero-copy follow-up
```

This is a high-value **design-to-landing** provenance source.

### 117.2 Packet metadata

The 2025 packet-metadata talk covers:

``` text
skb metadata
bpf_dynptr
skb->data_meta limitations
RX metadata propagation
future TX metadata
```

This is a later continuation of the XDP hardware-hints/metadata
discussions seen at LPC 2022.

Lineage:

``` text
LPC 2022
XDP hardware hints / BTF
        │
        ▼
metadata APIs evolve
        │
        ▼
LPC 2025
packet metadata RX/TX roadmap
```

### 117.3 Kernel-native XDP on AMD GPUs

This session connects several otherwise separate lineages:

``` text
XDP
DMA-BUF
P2PDMA
Device Memory TCP
AMDGPU
BPF JIT
```

The stated goal is kernel-native packet processing on an AMD GPU without
a userspace GPU runtime.

Architecturally:

``` text
NIC / host memory
       │
       ├── DMA-BUF / P2PDMA
       ▼
GPU memory
       │
       ▼
GPU-executed packet processing
```

Device Memory TCP is explicitly part of the data-transfer discussion.

This should be treated as **post-merge application/extension
provenance** for the device memory infrastructure, not as part of the
initial Device Memory TCP implementation.

### 117.4 page_pool in MANA

The MANA session describes replacing per-page RX-buffer use with:

``` text
page_pool fragments
+
pre-DMA-mapped page pools per RX queue
```

to avoid wasting a full 64-KiB page for a small packet on large-page
ARM64 systems.

This is a useful post-merge production example of why `page_pool` became
a core networking memory-management primitive.

------------------------------------------------------------------------


## LPC timeline mapped to the major change-log lineages

  ---------------------------------------------------------------------------
      Year LPC topic          Change-log lineage           Phase
  -------- ------------------ ---------------------------- ------------------
      2019 Multipath TCP      MPTCP 5.6                    pre-merge
           Upstreaming                                     

      2019 Programmable       SK_LOOKUP                    pre-merge
           socket lookup with                              
           BPF                                             

      2019 netfilter hardware nftables/flowtable/offload   design/merge-era
           offloads                                        

      2020 Programmable Qdisc BPF qdisc                    early design
           with eBPF                                       

      2020 BPF TCP header     BPF TCP programmability      design/merge-era
           option/CC/socket                                
           storage                                         

      2020 OVS + AF_XDP       AF_XDP/virtual networking    application

      2021 SO_REUSEPORT       SO_REUSEPORT failover        merge/post-merge
           socket migration                                

      2021 TSO/GRO/Jumbo for  XDP multi-buffer             pre-merge
           XDP                                             

      2021 bpfilter           BPF firewall                 design

      2022 high-speed Linux   BIG TCP/ZC/devmem            architecture
           networking                                      

      2022 machine-readable   YNL                          pre-merge
           Netlink YAML                                    

      2022 MPTCP BPF +        MPTCP extensibility          merge/design
           Netlink                                         

      2022 XDP hardware hints packet metadata              design

      2022 packet queueing in programmable queueing        RFC
           XDP                                             

      2023 io_uring ZC        io_uring ZCRX                pre-merge
           receive                                         

      2024 Per Netns RTNL     RTNL breakup                 pre/merge-era

      2024 network            virtio/AF_XDP/netkit         architecture
           virtualization                                  
           overhead                                        

      2025 zero-copy in       netkit queue leasing         design/merge-era
           containers                                      

      2025 packet metadata    XDP/BPF metadata             ongoing

      2025 XDP on AMD GPU     devmem/P2PDMA/XDP            post-merge
                                                           extension

      2025 MANA RX page_pool  page_pool                    post-merge
                                                           application
  ---------------------------------------------------------------------------

------------------------------------------------------------------------


## Conference provenance stack after LPC integration

The full provenance stack is now:

``` text
Kernel Recipes
  architecture / production experience / maintainer workshops
        │
        ▼
Linux Plumbers Conference
  cross-subsystem design discussion / unfinished work / BoFs
        │
        ▼
Netdev
  focused Linux-networking implementation and design
        │
        ▼
LWN
  independent technical explanation / patch-series / merge reporting
        │
        ▼
lore + patchwork
  review history / accepted series
        │
        ▼
git.kernel.org
  canonical commits
        │
        ▼
kernel release
```

The ordering is conceptual, not strictly chronological: a feature may
appear at Netdev before LPC, or return to LPC after merging.

The important property is that each source answers a different question:

``` text
Kernel Recipes:
why is this architectural direction useful?

LPC:
what design/API problems are developers actively trying to solve?

Netdev:
how is the networking-specific implementation supposed to work?

LWN:
what changed, what was controversial, and what landed?

lore/patchwork:
what exact series was reviewed and accepted?

git:
what is actually in mainline?
```

This makes it possible to distinguish **idea provenance** from **landing
provenance** and from **mainline provenance**.

------------------------------------------------------------------------


## LPC cross-track sweep --- DMA, RDMA, virtualization and io_uring

The LPC search was expanded beyond the Networking/eBPF tracks. This
matters because several prerequisites for modern zero-copy networking
live in RDMA, VFIO/IOMMU/PCI, virtualization, and generic io_uring
discussions rather than in the networking track itself.

### 120.1 LPC 2019 --- Challenges of the RDMA subsystem

``` text
Speaker:
Jason Gunthorpe

Track:
Refereed / RDMA-related

Year:
2019
```

The session discusses several cross-subsystem problems:

``` text
userspace DMA programming model
long-term DMA and get_user_pages
DAX / page cache interaction
GPU + DMA-BUF + VFIO interoperability
PCI peer-to-peer DMA
overlap with netdev / virtio / NVMe
```

The accompanying RDMA microconference description is even more explicit
about:

``` text
RDMA ↔ GPU / NVMe P2P
HMM
DMA-BUF
DAX
ODP
container support
```

#### Historical relevance

This is not direct Device Memory TCP provenance. The networking feature
did not yet exist in its later form.

It is instead **infrastructure/problem-space provenance** for a
recurring kernel question:

``` text
How can a high-speed I/O device DMA directly to memory that is not ordinary
kernel-owned system RAM while preserving Linux memory-management, isolation,
lifetime and device-ownership rules?
```

That question later appears in several networking lineages:

``` text
RDMA / P2P / HMM / DMA-BUF
             │
             ├──────────────┐
             ▼              ▼
        GPU memory      userspace memory
             │              │
             ▼              ▼
       Device Memory     io_uring ZCRX
           TCP
```

The important distinction is that Device Memory TCP solves the
networking-stack side of the problem through
page_pool/netmem/memory-provider integration rather than simply reusing
the RDMA subsystem.

------------------------------------------------------------------------

### 120.2 LPC 2020 --- xen-netfront and virtio_net XDP offloading

``` text
Track:
Networking and BPF Summit

Year:
2020
```

The proposal discusses moving/offloading guest XDP programs toward the
host for `virtio-net` and `xen-netfront`.

It predates the later AF_XDP zero-copy work, but belongs in the
virtual-networking performance lineage:

``` text
guest virtual NIC
      │
      ▼
XDP processing
      │
      ▼
host/guest XDP offload discussion
      │
      ▼
virtio-net XDP refactoring
      │
      ▼
virtio-net AF_XDP RX zero-copy
      │
      ▼
netkit / KubeVirt queue-leasing direction
```

This is **design lineage**, not a claim that the later virtio-net AF_XDP
API originated from this exact proposal.

------------------------------------------------------------------------

### 120.3 LPC 2020 --- Userspace OVS with HW Offload and AF_XDP

The session proposes a three-level processing hierarchy:

``` text
1. hardware
   tc-flower offload
       ↓ fallback
2. XDP
       ↓ fallback
3. OVS userspace datapath via AF_XDP
```

A key property is that AF_XDP retains a kernel network driver, unlike an
OVS-DPDK style full userspace-driver deployment. That allows hardware
offload through `tc-flower` while also providing a high-performance
userspace fallback.

#### Relevance

This is useful historical evidence for the recurring Linux networking
design pattern:

``` text
do not choose only between
"normal kernel stack"
and
"complete kernel bypass"

instead combine:
hardware offload
+ programmable in-kernel fast path
+ selective zero-copy userspace path
```

The same architectural philosophy later appears in:

``` text
Device Memory TCP
io_uring ZCRX
netkit queue leasing
```

although the mechanisms differ.

------------------------------------------------------------------------

### 120.4 LPC 2021 --- io_uring: BPF controlled I/O

``` text
Speaker:
Pavel Begunkov

Track:
LPC Refereed Track

Year:
2021
```

This is not itself a networking feature. It explores allowing BPF to
influence/control io_uring submission in order to reduce context
switches and scheduling overhead.

For this networking history its value is as an early cross-subsystem
connection:

``` text
BPF programmability
        +
io_uring asynchronous I/O
```

The later networking history increasingly combines these two ecosystems,
but the 2021 proposal should remain classified as **generic io_uring/BPF
design context**, not as a direct ancestor of a specific networking API.

------------------------------------------------------------------------


## LPC 2022 --- P2P inside virtual machines

### Exposing PCIe topology to Guest OS for peer-to-peer

``` text
Speaker:
Oded Gabbay

Track:
VFIO/IOMMU/PCI

Year:
2022
```

The talk identifies a concrete virtualization problem for peer-to-peer
DMA.

Typical P2P consumers include:

``` text
GPU ↔ RDMA NIC
AI accelerator ↔ NVMe
```

and kernel implementations may use:

``` text
p2pdma
DMA-BUF
```

The host kernel can validate PCI topology/distance, but a guest may not
see enough of the physical PCI topology to make the same decision.

#### Relevance to the networking history

This is especially useful when interpreting later GPU/device-memory
networking work.

``` text
P2P DMA on bare metal
       │
       ▼
virtualization hides physical topology
       │
       ▼
guest cannot safely determine P2P feasibility
```

That is a different problem from the later KubeVirt/netkit queue-leasing
path, but both show why zero-copy networking inside
virtualized/containerized environments requires more than merely
exposing a fast packet API.

------------------------------------------------------------------------


## LPC 2023 --- ZCRX as a hybrid rather than kernel-bypass design

The cross-track sweep reinforces the significance of the 2023:

``` text
Zero Copy Receive using io_uring
```

session.

Its description explicitly positions the proposal between:

``` text
full kernel copy
        and
full kernel bypass
```

using:

``` text
flow steering
header splitting
page_pool memory providers
io_uring
```

The control plane/stateful networking remains in the normal kernel stack
while payload DMA can go directly to userspace memory.

Conceptually:

``` text
                packet
                  │
           header splitting
             ┌────┴────┐
             ▼         ▼
          header     payload
             │         │
             ▼         ▼
       kernel stack   DMA
       TCP state       │
                       ▼
                  userspace memory
```

The LPC material also explicitly mentions future extension toward GPU
device memory.

This provides an important bridge:

``` text
io_uring ZCRX
       │
       ├── userspace memory provider
       │
       └── future device/GPU memory
                    │
                    ▼
              Device Memory TCP
              adjacent design space
```

The two features should still be kept distinct in the canonical
inventory.

------------------------------------------------------------------------


## VFIO/IOMMU/PCI as supporting provenance

The 2023--2025 VFIO/IOMMU/PCI microconference descriptions repeatedly
list:

``` text
ATS / PRI
SR-IOV / PASID
SVA
RDMA
P2PDMA
CXL
IOMMU
DMA ownership
```

These sessions are too broad to map to a single networking feature and
should not be promoted into the main networking timeline individually.

They are nevertheless useful as **supporting infrastructure provenance**
for topics such as:

``` text
Device Memory TCP
GPU packet processing
RDMA/GPU P2P
zero-copy virtual machines
confidential-device networking
```

The rule used in this document is therefore:

``` text
specific cross-track talk
    → canonical conference inventory entry

generic microconference proposal
    → supporting context only
```

------------------------------------------------------------------------


## Cross-track findings that materially change the networking narrative

The additional LPC material strengthens three long-term architectural
themes.

### Theme 1 --- zero copy is also a memory-ownership problem

``` text
2019 RDMA / DMA-BUF / HMM / P2P discussions
                 │
                 ▼
       memory ownership / pinning /
       DMA lifetime / isolation
                 │
                 ▼
2022 VM P2P topology problem
                 │
                 ▼
2023 page_pool memory-provider ZCRX
                 │
                 ▼
2024–2025 netmem / Device Memory TCP
```

Thus modern zero-copy networking cannot be explained only as:

``` text
remove memcpy()
```

It also requires solving:

``` text
who owns the memory?
who may DMA to it?
how is lifetime tracked?
how is it isolated?
how is it exposed across namespaces/VMs?
```

### Theme 2 --- kernel bypass is not the only high-performance model

Several LPC talks independently converge on:

``` text
hardware offload
       +
kernel programmable fast path
       +
selective zero-copy
       +
normal kernel control plane
```

Examples include:

``` text
OVS + tc-flower + XDP + AF_XDP
io_uring ZCRX
Device Memory TCP
netkit queue leasing
```

### Theme 3 --- virtualization makes direct-I/O topology harder

``` text
physical topology
    ├── NIC
    ├── GPU
    └── accelerator
          │
          ▼
      hypervisor
          │
          ▼
guest sees virtual topology
```

This affects:

``` text
P2PDMA validation
device-memory access
queue ownership
AF_XDP placement
KubeVirt zero-copy
```

The later netkit queue-leasing work can therefore be understood as part
of a broader attempt to preserve kernel-managed isolation while exposing
enough of the physical data path to containers and VMs.

------------------------------------------------------------------------


## Updated LPC canonical inventory

  -------------------------------------------------------------------------
            Year Session          LPC area          Canonical role
  -------------- ---------------- ----------------- -----------------------
            2019 Multipath TCP    Networking        MPTCP pre-merge
                 Upstreaming                        

            2019 Programmable     Networking        SK_LOOKUP pre-merge
                 socket lookup                      
                 with BPF                           

            2019 Challenges of    Refereed/RDMA     DMA/P2P/device-memory
                 the RDMA                           problem-space
                 subsystem                          

            2019 RDMA MC          RDMA              HMM/DMA-BUF/P2P
                                                    supporting context

            2020 xen-netfront and Networking+BPF    virtual-NIC XDP design
                 virtio_net XDP                     
                 offloading                         

            2020 Userspace OVS    Networking+BPF    hybrid HW/XDP/AF_XDP
                 with HW Offload                    datapath
                 and AF_XDP                         

            2020 A programmable   Networking+BPF    BPF-qdisc early design
                 Qdisc with eBPF                    

            2021 TSO/GRO/Jumbo    BPF+Networking    XDP multi-buffer
                 frames for XDP                     precursor

            2021 io_uring: BPF    Refereed          io_uring/BPF context
                 controlled I/O                     

            2022 High-speed Linux eBPF+Networking   BIG-TCP/ZC architecture
                 TCP                                

            2022 Netlink YAML     eBPF+Networking   YNL pre-merge

            2022 MPTCP BPF +      eBPF+Networking   MPTCP extensibility
                 Netlink                            

            2022 PCIe topology to VFIO/IOMMU/PCI    virtualized P2P
                 guest for P2P                      constraint

            2023 Zero Copy        eBPF+Networking   io_uring ZCRX pre-merge
                 Receive using                      
                 io_uring                           

            2024 Per Netns RTNL   Networking        RTNL breakup

            2024 Network          Networking        virtual-network
                 virtualization                     optimization
                 overhead                           

            2025 Zero-copy in     Networking        netkit queue leasing
                 containers                         

            2025 Packet Metadata  Networking        metadata API evolution

            2025 XDP on AMD GPUs  Networking        devmem/P2PDMA
                                                    post-merge extension
  -------------------------------------------------------------------------

------------------------------------------------------------------------


## Provenance graph with cross-subsystem LPC material

The networking-memory branch can now be represented more accurately as:

``` text
                 ┌──────────────────────────┐
                 │  RDMA / MM / DMA world  │
                 │  GUP, HMM, DMA-BUF, P2P │
                 └────────────┬─────────────┘
                              │
                              ▼
                       DMA ownership /
                       memory lifetime
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
   page_pool               AF_XDP               P2PDMA
       │                      │                      │
       ▼                      │                      ▼
    netmem                    │                 GPU memory
       │                      │                      │
   ┌───┴───────────┐          │                      │
   ▼               ▼          ▼                      ▼
Device Memory   io_uring   virtio/netkit      GPU packet
TCP             ZCRX       queue leasing      processing
```

This is intentionally a dependency/problem-space graph, not a claim that
each box was implemented directly from the box above it.

It captures a major conclusion of the multi-source research:

> Linux networking's recent zero-copy evolution is inseparable from the
> kernel's broader DMA, memory-ownership, device-memory and
> virtualization work.

------------------------------------------------------------------------


## Unified chronological timeline --- conferences → LWN → patches → mainline

Legend: `[KR]` Kernel Recipes, `[LPC]` Linux Plumbers Conference,
`[Netdev]` Netdev, `[LWN]` LWN, `[Patch]` upstream series, `[Mainline]`
canonical landing, `[Release]` kernel milestone, `[RFC]` design only,
`[X]` removal.

### 2019 --- upstreaming and programmable networking

``` text
[KR] XDP closer integration → high-speed RX memory pressure
[Mainline] ab84be7e54fc Initial nexthop code → [Patch] route integration v4/20
[LWN/Patch] MPTCP RFC → [LPC] MPTCP Upstreaming → [LWN] upstreaming coverage
[KR] BPF at Facebook / Suricata+XDP / David Miller eBPF
[LPC] Programmable socket lookup with BPF
```

### 2020 --- native MPTCP and BPF extensibility

``` text
f870fa0b5768 MPTCP socket stubs
648ef4b88673 MPTCP receive path
048d19d444be MPTCP kselftest
        ↓
Linux 5.6

[LWN] BPF struct_ops
[LPC] programmable Qdisc with eBPF
[LPC] BPF-extensible TCP
[LPC] OVS + HW offload + AF_XDP
[LWN] threaded NAPI / netfilter HW offload
[Netdev 0x14] IOAM / MPTCP / TC conntrack offload
```

### 2021 --- multi-buffer pressure, IOAM, zero-copy

``` text
[LPC] TSO/GRO/Jumbo for XDP → non-linear XDP → AF_XDP multi-buffer

[Netdev] IOAM
  ↓
[Patch] v5/6
  ├─ 9ee11f0fff20 data plane
  ├─ 3edede08ff37 lwtunnel
  ├─ de8e80a54c96 docs
  └─ 968691c777af selftest
  ↓
Linux 5.15

[Netdev] resilient nexthops
[LWN/LPC] SO_REUSEPORT failover/migration
[LWN] BPF meets io_uring / zero-copy TX
```

### 2022 --- BIG TCP, MPTCP extensibility, YNL, ZC architecture

``` text
[Netdev] BIG TCP → [LWN] Going big with TCP packets → Linux 5.19

[Patch] MPTCP/BPF v5/7
  └─ 3bc253c2e652 implementation
  ↓
[LPC] MPTCP with eBPF and Netlink
  ├─ BPF scheduler direction
  └─ userspace path manager (5.19)

[LPC] machine-readable Netlink (YAML?)
  ↓
YAML specs → YNL → protocol conversions

[KR] io_uring path to zero-copy
  ├─ registered buffers
  ├─ future ZC RX
  └─ DMA-BUF/P2P
```

### 2023 --- netkit, Device Memory TCP and ZCRX

``` text
[Patch] IPv4 BIG TCP v4/10 → Linux 6.3

[KR/Netconf] SO_DEVMEM/direct GPU placement
  ↓
[Netdev 0x17] Device Memory TCP
  ↓
RFC/revision cycle

[KR] io_uring networking
  ↓
[LPC] io_uring Zero Copy Receive
  ↓
[Netdev] ZCRX
  ↓
RFC/revision cycle

[LWN] BPF-programmable network device
  ↓
35dfaad7188c netkit → Linux 6.7

AF_XDP multi-buffer → Linux 6.6
```

### 2024 --- devmem RX, virtio AF_XDP and RTNL

``` text
Device Memory TCP v26/13 → Linux 6.12

e9f3962441c0 virtio-net XSK fill
a4e7ba702701 small RX
99c861b44eb1 mergeable RX
        ↓
Linux 6.11

RCU/unlocked RTNL readers
  ↓
[LPC] Per Netns RTNL
  ↓
per-netns RTNL infrastructure → Linux 6.13
  ↓
continued subsystem RTNL removal
```

### 2025 --- ZCRX landing and container zero-copy

``` text
io_uring ZCRX revisions
  ↓
ca0b04ba0b35... merge
  ↓
Linux 6.15

[LPC 2020] programmable qdisc
  ↓
[Netdev 0x17] eBPF Qdisc
  ↓
c8240344956e... BPF Qdisc_ops

[LPC] zero-copy in containers
  ↓
physical NIC queue → netkit lease
  ├─ io_uring ZC in Pods
  ├─ devmem TCP in Pods
  └─ AF_XDP → QEMU/KVM
```

### 2026 --- queue leasing, devmem TX, AccECN, tunnel BIG TCP

``` text
77b9c4a438fc... first queue-leasing merge
  ↓
8766d61a1d33... revert
  ↓
redesign
  ↓
15089225889b... re-merge

Device Memory TCP TX v14/9 → Linux 6.16

[LWN] AccECN
  ↓
542a495cbaa6 core
3cae34274c79 negotiation
  ↓
Linux 6.18 generation

IPv6 BIG TCP → IPv4 BIG TCP
  ↓
2026 v9/9 BIG TCP for UDP tunnels
  ↓
VXLAN / GENEVE → Linux 7.3 development
```


## Unified feature lineage index

### Packet memory / zero copy

``` text
RX memory pressure → page_pool → DMA-BUF/P2P + memory providers → netmem
                                                        ├→ Device Memory TCP
                                                        └→ io_uring ZCRX
                                                             ↓
                                                     queue leasing/netns
```

### Packet size

``` text
GRO/GSO → IPv6 BIG TCP 5.19 → IPv4 BIG TCP 6.3 → tunnel BIG TCP
```

### BPF networking

``` text
XDP → struct_ops → SK_LOOKUP → MPTCP BPF → netkit → BPF qdisc → queue/TX policy
```

### MPTCP

``` text
2019 LPC/LWN → 5.6 native MPTCP → path management → 2022 BPF → userspace PM → extensions
```

### Control plane

``` text
hand-written Netlink → 2022 YAML proposal → YAML specs → YNL → protocol conversions
```

### Locking

``` text
global RTNL → unlocked flags → RCU readers → per-netns RTNL → subsystem lock removal
```


## Source-role matrix

  ---------------------------------------------------------------------
  Source                             Best used for
  ---------------------------------- ----------------------------------
  Kernel Recipes                     architectural motivation and
                                     production experience

  Linux Plumbers Conference          cross-subsystem API/design
                                     discussion and unresolved problems

  Netdev                             networking-specific implementation
                                     and datapath design

  LWN                                independent explanation, review
                                     history and merge interpretation

  lore / patchwork                   exact revision and accepted-series
                                     evidence

  git.kernel.org                     canonical mainline commits

  release pulls/tags                 release-level confirmation
  ---------------------------------------------------------------------

``` text
Why?            → KR / LPC / Netdev
API debate?     → LPC / Netdev / lore
Implementation? → lore / LWN
Accepted?       → patchwork / maintainer pull
Mainline?       → git
Which release?  → release pull/tag
```


## Recommended reading order

``` text
1. Unified chronological timeline
2. Unified feature lineage index
3. Individual feature dossiers
4. Conference sections
5. LWN inventory
6. Provenance/commit tables
```

The document is intended to function as both a Linux networking history
and an auditable source map.

------------------------------------------------------------------------

