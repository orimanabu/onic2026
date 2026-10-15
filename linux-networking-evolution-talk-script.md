# Linux Networking Evolution 講演スクリプト

Oct 7, 2026 · @Manabu

`linux-networking-evolution-talk.pptx`（34 枚）用の読み上げ原稿です。1 枚あたり 1.5〜2 分、全体で 50〜55 分、質疑 5〜10 分を想定しています。

| 区間 | スライド | 目安 |
| --- | --- | --- |
| 導入 | 1–2 | 3 分 |
| Part I 中心命題 | 3–7 | 9 分 |
| Part II Era と年表 | 8–10 | 6 分 |
| Part III 前半 | 11–17 | 12 分 |
| Part III 後半 | 18–23 | 10 分 |
| Part IV / Part V | 24–28 | 7 分 |
| Synthesis・まとめ | 29–34 | 6 分 |

時間が足りない場合は、スライド 7（Snapshot）、13（BIG TCP）、22（Transport）、31（並行系列）を飛ばしても話の筋は保てます。各スライドの見出しは `番号. スライドタイトル` で、本文がそのまま話す内容です。

**話し方の軸**: 「速くなった」「iptables が nftables になった」のような機能の置き換えの話ではなく、「何が明示されたから、何が共存できるようになったか」という一本の問いで全スライドをつなぎます。

## スライド 1–2: 導入

### 1. Linux Networking Evolution

こんにちは。今日は Linux カーネルのネットワーク機能が、v3.0 が出た 2011 年から現在の 7.x まで、15 年間でどう変わったかをお話しします。

最初にお断りしておくと、今日の範囲は有線のホストネットワーキングが中心です。RDMA、Wi-Fi 全般、QUIC、暗号プロトコルの網羅的な歴史は扱いません。ただし kTLS、XFRM、WireGuard、TCP-AO のように、ネットワークアーキテクチャと交差するセキュリティ機能は横断テーマとして取り上げます。

今日の話の元になっている調査は 2026 年 10 月 2 日を基準日にしています。7.3 についてはまだ rc 段階のものがあり、その部分は「mainline に merge 済み、final は未公開」という扱いで話します。

### 2. 本講演の構成

構成は 6 つです。まず Part I で中心命題、つまり「15 年間で何が変わったのか」を一言で言うとこうだ、という主張を先に出します。Part II ではその 15 年を 4 つの時代に区切って、それぞれの時代に外部からどんな圧力がかかり、カーネルが何を解こうとしたかを見ます。

Part III が本体で、PERFORMANCE、PROGRAMMABILITY、MEMORY、CONTROL PLANE の 4 つの軸について、同じ設計課題が長期間にどう変化したかを系譜として追います。加えて transport、virtual networking、security という横断ドメインも扱います。

Part IV はドライバフレームワーク、つまり個々のドライバの中に閉じていた知識がどうやって共通の仕組みに引き上げられたか。Part V は Observability です。最後に Synthesis として、全体を一つの表にまとめ、まだ終わっていない仕事を挙げて締めます。

順番に release 番号を並べる話ではないので、「この機能は何バージョンで入ったか」は年表スライドと、元ドキュメントの Part VI を参照してください。

## スライド 3–7: Part I 中心命題

### 3. Part I — 中心命題（セクション扉）

では Part I です。「Linux networking は何が変わったのか」。先に答えを言ってしまうと、変わったのは「速さ」ではありませんし、古い skb のパスが新しい何かに置き換わったわけでもありません。

### 4. skb path は残り、複数の execution / memory model が共存する

2011 年頃、つまり v3.0 の時代の典型像を左に描いています。NIC の RX queue に入ったパケットは system RAM 上の page や skb に置かれ、NAPI から通常のカーネルネットワークスタックへ渡される。RX queue と system RAM と skb とスタックが、ほぼ暗黙に一対一で対応していました。

右が 2026 年です。同じ「RX queue」という箱から、矢印が 5 本出ています。通常の kernel stack、XDP から AF\_XDP やリダイレクト、io\_uring の zero-copy receive、device memory に直接着地する受信、そして仮想デバイスへ貸し出された queue。同じ一本の RX queue が、どの consumer のどのメモリに向かうかを選べるようになった。

大事なのは、左の古いパスが消えていないことです。CPU が system RAM 上の skb を処理するモデルは今も圧倒的多数のワークロードで使われています。変わったのは、queue の identity、memory provider、consumer、lifetime、synchronization を明示的に扱うことで、複数の実行モデルとメモリ所有モデルを同じデバイス上で選択・共存させられるようになったこと。これが今日の中心命題です。

### 5. 変化は 6 つの軸で並行して進んだ

この変化を 6 つの軸で見ます。PERFORMANCE は queueing、pacing、aggregation、zero-copy。暗黙の queueing から明示的な accounting や多層スケジューリングへ、という変化です。PROGRAMMABILITY は BPF、XDP、socket、protocol、qdisc の拡張点。固定されたパスから、検証付きの拡張点へ、さらにそれが再利用可能なインフラになる流れです。

MEMORY は page\_pool から netmem、memory provider、device memory へ。ドライバごとの page 処理が共通の lifetime、provider、assignment モデルになりました。CONTROL PLANE は Netlink の object model、YNL、RTNL の scope 縮小。OBSERVABILITY は typed tracing、drop reason、queue や NAPI やメモリの identity。DRIVER FRAMEWORK は switchdev、devlink、phylink、DIM といった共通のドライバ契約です。

一番下に書いてあるとおり、TRANSPORT、VIRTUAL/OVERLAY、SECURITY はこの 6 軸に並ぶ 7 番目以降の軸ではなく、複数の軸を横断するドメインとして扱います。この区別は後で効いてきます。

### 6. 共存を可能にした mechanism: explicit contracts

共存を可能にした仕組みを、この講演では「explicit resource / control contracts」と呼びます。具体的な形は上の 6 つです。object は名前付きで再利用できる対象、accounting は queue に積んだバイト数のような計量、API は検証済みあるいはスキーマ駆動のインターフェース、assignment は queue やメモリを誰が使うかの割り当て、lifetime は確保・再利用・解放、synchronization はロックの範囲やハードウェア状態との同期です。

下の表が代表例です。DQL/BQL はドライバ任せだった TX ring の tuning を、queued と completed のバイト数の accounting にしました。bpf() syscall と map と verifier は、固定されたカーネル動作を検証済みプログラムと map の API にした。switchdev と devlink はベンダ固有だった forwarding やデバイス状態を object にした。AF\_XDP は queue と UMEM の binding、つまり assignment と lifetime。YNL は手書きだった Netlink family 定義を機械可読なスキーマに。per-netns RTNL は global だった同期の範囲を namespace 単位に。RX HW queue leasing は、物理 netdev に固定されていたハードウェア queue を貸し出せる assignment にした。

表の右端の Era を見ると傾向があります。Era 1 は accounting と API の基礎、Era 2 から 3 は object と programmability、Era 4 は assignment、lifetime、synchronization の明示化が目立ちます。ただし時代は重なっていて、厳密な時代区分ではありません。

### 7. Snapshot: v3.0 / v5.0 / 2026

3 つの時点で切った表です。v3.0 の列は「driver-local」「classic BPF」「global RTNL」と、暗黙で局所的なものが並びます。v5.0 の列には BQL、XDP、page\_pool、switchdev、devlink が揃っていて、programmable で offload でメモリ基盤のある姿がすでにできている。2026 年の列は netmem、memory provider、queue/NAPI object、per-netns RTNL、typed drop reason と、identity と assignment の言葉が増えています。

ここで注意していただきたいのは、v5.0 を第 2 の起点としては扱わないことです。v5.0 は、v3.0 から始まった変化の中間観測点として置いています。XDP や page\_pool は v5.0 で突然現れたのではなく、その前の 4.x でできあがっていたものが v5.0 の時点で見えている、という読み方です。

## スライド 8–10: Part II Era と年表

### 8. Part II — 4 つの architecture era（セクション扉）

Part II では 15 年を 4 つの時代に分けます。ただし区切りは kernel の major version ではありません。外部からどんな圧力がかかり、networking architecture が主に何を解こうとしたか、で区切ります。

### 9. 外部の圧力から architecture response へ

Era 1 は 2014 年頃まで。10GbE や 40GbE の NIC、マルチコアサーバ、コンテナ、データセンターのレイテンシ要求が圧力です。「ring に積めるだけ積む」設計のレイテンシと公平性の問題が露呈して、BQL、CoDel、TSQ、pacing のように queue と CPU locality を明示的に制御する方向へ進みました。同時に namespace と VXLAN が仮想化とマルチテナントの土台になり、nftables と eBPF の基礎もこの時代です。

Era 2 は 2015 から 2018 年。クラウドの datapath が多様化し、packet rate の圧力と kernel bypass への誘惑が強まった時代です。固定されたカーネルパスを増築し続ける代わりに、TC BPF や XDP という検証付きの拡張点と、AF\_XDP という queue に紐付いた userspace パスが発達しました。SmartNIC や switch ASIC の普及が、switchdev、devlink、TC offload というハードウェアマッピングの共通化を要求しました。

Era 3 は 2019 から 2022 年。programmable なパスが本番のインフラになると、「プログラムを置ける」だけでは足りなくなります。BTF と CO-RE、struct\_ops、typed な observability、BIG TCP、MPTCP、io\_uring networking のように、再利用、運用、処理単位の拡大が進みました。

Era 4 は 2023 年以降、現在進行中です。400GbE 級、アクセラレータ、device memory、heterogeneous な実行環境。「どの queue、どのメモリ、どのデバイス、どのロック範囲を誰が使うか」をドライバの暗黙の慣習のままにしておけなくなり、netmem、memory provider、queue object、queue leasing、per-netns RTNL のような明示的な placement と control が押し出されています。

下の 4 つの問いが、時代ごとの支配的な設計課題です。「どれだけ queue すべきか」、「どこで datapath をプログラムできるか」、「それをどう再利用可能なインフラにするか」、「placement、lifetime、synchronization をどこまで明示的に制御できるか」。この矢印は機能の依存関係ではなく、問いの変化を表しています。ある時代に始まった仕組みは次の時代でも発展し続けます。

### 10. 年表（挿入スライド）

これが今話した 4 つの時代を横軸に、6 つの軸を縦に並べた年表です。上に X.0 リリースと年、背景の帯が Era です。塗りつぶしの点が最初の canonical なリリース、白抜きの点と破線がその前の開発期間のおおよその始まりです。

細かく読む必要はありませんが、2 つだけ見てください。一つは、PERFORMANCE の点が Era 1 に密集し、MEMORY と CONTROL PLANE の点が Era 4 に密集していること。さっきの「問いの変化」がそのまま点の分布に出ています。もう一つは、灰色の細い線です。たとえば XDP から AF\_XDP、page\_pool から netmem へ線が伸びていますが、これは「後の機能が前の機能の上に建っている」という系譜を示しています。Part III ではこの線をたどっていきます。

なお、BBR、MPTCP、VXLAN、kTLS のような横断ドメインの機能はこの年表には行がありません。そちらは Part III の後半で別途扱います。

## スライド 11–17: Part III 前半（PERFORMANCE / PROGRAMMABILITY / MEMORY）

### 11. Part III — 長期 lineage（セクション扉）

ここからが本体です。リリース順ではなく、同じ設計課題が長期間にどう変化したかを追います。各スライドの下に Takeaway を置いているので、細部を聞き逃してもそこだけ持ち帰っていただければ大丈夫です。

### 12. Queueing: 単一 qdisc から multi-layer control へ

PERFORMANCE 軸の最初は queueing です。この系譜は「qdisc のアルゴリズムが増えた歴史」ではありません。socket が作る burst、qdisc が持つ backlog、ドライバや NIC ring に積まれた未完了バイトを、それぞれ別の層で制御できるようになった歴史です。

一番下のドライバ層から話します。3.3 の DQL/BQL は、ドライバが NIC に渡したバイト数と完了したバイト数を数えて、ハードウェア queue に過剰な backlog を作らない backpressure を与えます。同じ時代の SO\_REUSEPORT と SO\_BUSY\_POLL は、マルチコアサーバで受信処理をどの CPU に置くかという placement の話です。ずっと後の 6.19 の dev\_queue\_xmit() の llist 化は、複数 CPU が同じ qdisc lock を取り合うところを、一つの CPU だけが lock を進めて他は lockless list に積む形に変えた、実装スケーラビリティの改善です。

真ん中の qdisc 層。CoDel は queue の長さではなく、パケットが queue に滞在した時間、sojourn time を観測して drop を決めます。これは「delay を直接 feedback 信号にした」転換点で、fq\_codel が flow ごとの分離を足しました。sch\_fq と TCP pacing は「何個 queue に入れるか」ではなく「いつ送るか」を扱い、ETF や SO\_TXTIME、taprio は送信時刻そのものを明示するところまで広げました。7.0 の cake\_mq は CAKE をマルチ queue 対応にしたものです。

一番上の socket 層。TSQ は一つの TCP socket が qdisc に押し込めるバイト数を制限し、BQL と同じ問題を socket 側から解きます。BBR や EDT は rate ベースの送信を pacing の上に建てています。

一つ補足すると、Wi-Fi の mac80211 には airtime scheduling と AQL があります。これはバイトではなく無線 medium の占有時間を資源として数えます。BQL の派生ではありませんが、「見えない queue の圧力を数えられる量に変える」という同じ設計問題を持っています。

Takeaway です。queue の占有と完了を accounting と API として明示したことで、各層が独立に backpressure と scheduling を制御できるようになりました。

### 13. Packet aggregation: GRO/GSO から BIG TCP へ

同じ PERFORMANCE 軸でも、こちらは別の性格の系譜です。左の図のとおり、GRO は受信側で同じ flow のパケットをまとめて大きな skb にし、GSO や TSO は送信側で大きな skb を wire サイズに分割します。ポイントは、スタック内部の処理単位と wire 上のパケットサイズを分離することです。

右が BIG TCP の系譜です。5.19 の IPv6 BIG TCP で gro\_max\_size と gso\_max\_size の 64KiB 制限が外れ、6.3 で IPv4、7.0 で IPv6 の synthetic な Hop-by-Hop header が不要になり、7.3-rc で VXLAN や GENEVE の tunnel 越しにも効くようになっています。

よくある誤解の訂正を一つ。BIG TCP は wire MTU を巨大化する機能ではありません。jumbo frame の別名でもありません。カーネル内部の packet-processing unit を大きくして、host 内の per-packet overhead を減らす機能です。

Takeaway。aggregation は explicit contract の系譜ではなく、一回のスタック通過で扱う単位を拡大する並行系列です。今日の中心命題で説明しないものがある、ということをここで最初に言っておきます。

### 14. BPF: packet filter から stack extension へ

PROGRAMMABILITY 軸です。BPF の起点は XDP ではありません。classic BPF は socket filter 用の小さな VM でしたが、3.15 で内部命令セットが eBPF に書き換えられ、3.18 で bpf() syscall、map、verifier が入って、userspace が検証済みプログラムと永続的な map をカーネルにロードする共通の実行モデルになりました。program と map が file descriptor で参照できるカーネル object になった、というのが転換点です。

真ん中の木が、その後の展開です。一本道ではなく、attach できる場所が三方向に広がりました。packet と datapath には XDP と TC。socket と lookup には cgroup hook、SOCK\_OPS、SOCKMAP、SK\_LOOKUP。そして protocol algorithm には struct\_ops。struct\_ops は TCP の輻輳制御のような型付きの operation を検証済み BPF で実装できるようにしたものです。

下の段の TCX、netkit、BPF qdisc は、仮想デバイスと queue 制御まで programmability の対象が広がったことを示しています。これは上の段の後継ではなく、5.x から 6.x にかけて独立に並行して追加されたものです。

SOCK\_OPS について一言。これはパケットごとに呼ばれる hook ではなく、SYN や接続確立、RTT callback のような socket lifecycle event で呼ばれます。TC や XDP が packet の実行点を programmable にしたのに対して、SOCK\_OPS は transport object の状態遷移を programmable にしました。

Takeaway。言語の置き換えではなく、検証済み実行という contract と attachment モデルの一般化です。

### 15. XDP / AF\_XDP: placement と ownership

XDP は 4.8 で、ドライバが skb を構築する前の RX buffer に対して検証済み BPF を実行する datapath として入りました。XDP\_PASS は通常スタックへ、XDP\_DROP は早期破棄、XDP\_TX は受信デバイスから送り返し、XDP\_REDIRECT は 4.14 の DEVMAP や 4.15 の CPUMAP を介して別のデバイスや CPU へ。速い理由は「BPF だから」だけではなく、skb 割当前という placement と、redirect 先を明示する object があるからです。

右の AF\_XDP は 4.18 です。XSK socket と userspace の UMEM、そして FILL、RX、TX、COMPLETION の 4 つの ring で構成されます。FILL ring で userspace が空き frame を供給し、RX ring で受信 descriptor を受け取る。zero-copy mode では UMEM の frame と RX queue が直接結び付くので、queue の binding とメモリの ownership が正しさの一部になります。

ここが大事で、「kernel bypass ではなく、kernel が管理する資源を userspace datapath に貸す」という考え方が、後の queue leasing につながります。4.8 から 4.18 のこの時期は、queue、redirect、buffer binding の前史として重要です。

その後の拡張として、5.18 の multi-buffer で 1 packet = 1 連続 buffer という前提が外れ、jumbo frame や scatter-gather を扱える一般的な RX 表現になりました。XDP metadata や RX hints は、ハードウェアタイムスタンプや RSS hash を packet bytes とは別の contract で BPF へ渡します。

Takeaway。検証済み実行点に加えて、redirect 先、RX queue、UMEM の ownership を object と assignment として明示したことで、kernel path と userspace path が共存できるようになりました。

### 16. Packet memory: page\_pool → netmem → providers

MEMORY 軸です。この変化を「zero-copy が増えた」の一語で済ませると本質を落とします。大事なのは、allocation、recycling、lifetime、DMA mapping、queue binding を誰が管理するかが、ドライバ内の慣習から共通の contract に移ったことです。

上の流れ。4.18 の page\_pool は RX の page 再利用と DMA lifecycle を共通化しました。XDP と一緒に普及しましたが XDP 専用ではなく、後のすべての zero-copy パスがドライバごとの独自 allocator に戻らずに済むための基盤です。6.9 の netmem は、networking が扱うメモリを struct page そのものから切り離しました。system RAM の page だけを前提にすると device memory や userspace 所有のメモリを同じパスで扱えないからです。

順序に注意してください。Device Memory TCP の RX は 6.12 で先に入り、この時点では専用の経路を使っていました。6.15 で memory\_provider\_ops という汎用の provider hook が入り、devmem TCP と io\_uring ZCRX が同じ contract に接続されました。consumer 固有のパスが先行し、後から provider API が一般化された、という順です。

表は役割分担です。netmem が memory の表現、memory provider が allocation と lifetime の供給元、queue binding や leasing が「どの RX queue がそのメモリドメインを使うか」を担当します。だから Era 4 では MEMORY 軸が lifetime だけでなく assignment と強く結び付きます。

Takeaway。packet buffer の lifetime、provider、queue assignment をドライバ固有の慣習から共通 contract へ移しました。

### 17. Zero-copy と io\_uring networking

上の流れは socket API 側から見た zero-copy の系譜です。4.14 の MSG\_ZEROCOPY は送信側で userspace の page を pin して copy を避けます。4.18 の TCP\_ZEROCOPY\_RECEIVE は受信データを page-aligned な mapping として見せます。6.0 の io\_uring SEND\_ZC と multishot receive、そして 6.15 の ZCRX で RX queue と userspace 所有のメモリが直接結び付きます。

下の 2 つの図は Device Memory TCP の効果です。従来は NIC から system RAM に入れて CPU が copy や mapping をして GPU へ。device-memory path では NIC から直接 device memory へ。

共通する設計上のポイントが 3 つあります。一つ、operation の完了と memory lifetime の完了は別です。send が返っても buffer を再利用してよいとは限らず、MSG\_ZEROCOPY は error queue で、SEND\_ZC は別の completion でそれを通知します。二つ、zero-copy RX は透明な最適化ではなく、memory layout と socket consumption の semantics を API に露出する contract です。三つ、ZCRX では RX queue、page\_pool、memory provider と登録済み userspace メモリが接続されるので、queue ownership と buffer lifetime が io\_uring の object model の外部条件になります。

Takeaway。socket I/O を submission と completion、登録済み資源に分解し、ZCRX では queue と buffer と memory provider の関係まで明示する方向へ進みました。

## スライド 18–23: Part III 後半（CONTROL PLANE / cross-cutting）

### 18. Routing / TC / offload: 2 つの control path

CONTROL PLANE 軸に入ります。このスライドで一番言いたいのは、switchdev と TC offload を一本の経路として扱わない、ということです。

左の経路。bridge、FIB、VLAN といったカーネルの forwarding object があり、3.9 の bridge VLAN filtering、4.1 の MPLS、4.3 の VRF、5.3 の nexthop object がここに並びます。これらは switchdev の notification を通して switch driver に同期されます。右の経路。TC の classifier と action、4.2 の Flower や 5.3 の ct action があり、こちらは ndo\_setup\_tc や flow-block callback を通して flow\_rule という共通表現でドライバへ渡されます。二つはドライバとハードウェアで合流し得ますが、カーネル API としては並行した別の経路です。

nexthop object について。従来の route は gateway や output device を route entry の中に埋め込んでいました。5.3 の nexthop object はそれを ID 付きの独立 object に切り出し、複数の route から同じ nexthop や group を参照できるようにしました。ECMP group の変更が、大量の route を書き換える操作から、共有 object を更新する操作に変わります。ASIC 側でも route table と adjacency table は別の資源なので、mapping しやすくなる。これが「reusable object」という contract の代表例です。

offload の読み方も一つ。TC rule が hardware-native な rule に置き換わるのではありません。カーネルが canonical な状態を保持し、ハードウェア実行は optional です。だから「何が offload できるか」「一部の action だけ非対応のときどうするか」「stats をどこから読むか」が contract になります。

Takeaway。forwarding semantics をカーネルの canonical state として保ちながら、software か hardware かに写像できるようになりました。

### 19. Routing object・YNL・RTNL scope 縮小

同じ CONTROL PLANE で、3 つの流れを並べています。左は routing object。3.6 の IPv4 route-cache removal は per-destination の共有 cache を外した早い転換点で、nexthop object の直接の祖先ではありませんが、global な共有状態を減らすという同じ圧力に応えています。

真ん中は YNL。6.3 から、Generic Netlink family を YAML で記述し、UAPI、policy、helper、ドキュメント、userspace client を生成できます。attribute 番号を手書きする API から、schema 駆動の control plane へ。netdev family が queue、NAPI、page\_pool を object として公開しているのが代表例で、6.19 では WireGuard の Netlink も YNL 記述になりました。

右が RTNL です。rtnetlink の多くの更新パスは global な RTNL mutex に依存していたので、namespace やデバイスが増えると無関係な操作まで直列化されます。unlocked operation や RCU 読み出し、6.13 からの per-netns RTNL、6.15 の further breakup、7.3-rc の RTNL-less な FIB rule 更新と per-netns の netdev unregistration。この矢印は直接の拡張関係ではなく、global RTNL 依存を縮小する複数の技法が同じ方向に進んだことを示します。

補足として、per-netns RTNL は 6.13 で完成した機能ではなく、複数リリースにまたがる migration です。7.3 を「完了」と呼ぶ根拠もありません。

Takeaway。Netlink の message format より、shared mutable state の保護範囲を縮小することが主眼です。

### 20. netfilter: rule engine から fast path へ

Netfilter の 15 年を「iptables から nftables へ」というコマンドの置き換えとして見ると本質を取りこぼします。上の流れ。3.13 の nftables は、family ごとに重複していた xtables を汎用の expression VM と Netlink API に寄せ、atomic な transactional update、sets と maps を持つ ruleset object model にしました。4.2 の ingress hook と 5.16 の egress hook で、policy を適用できる datapath 上の位置が広がりました。

左下の図が 4.16 の flowtable です。最初のパケットは従来の forwarding、conntrack、nftables policy を通ります。そこで選ばれた established flow が fast path に登録され、次からは hit すれば classic hook を bypass して neigh\_xmit() へ。miss や fragment、FIN/RST は classic path へ戻ります。5.5 でこれが hardware offload に拡張され、flow が flow\_rule としてドライバへ写像されます。投入は非同期なので、完了までは software path を通るパケットもある。つまり policy engine が置き換わったのではなく、policy と state を共有しながら slow、fast、hardware の 3 つの path が共存しています。

右の箇条書き。conntrack は置換されていません。flow の state、NAT mapping、mark、lifetime を保持し、nftables、TC の ct action、BPF の kfunc、flowtable が共有する基盤です。BPF や XDP も nftables の置換ではありません。iptables 互換の ruleset を BPF に変換する bpfilter は 6.8 で削除され、定着したのは conntrack kfunc や 6.4 の BPF\_PROG\_TYPE\_NETFILTER のように既存の Netfilter と相互乗り入れする方向でした。

Takeaway。policy の表現、transactional update、共有 flow state、fast-path の適格性、実行位置が明示化され、複数の path が共存します。今日の中心命題を CONTROL PLANE 側から見た代表例です。

### 21. Virtual networking: datapath と queue assignment

ここから 3 枚は横断ドメインです。まず virtual networking。上の流れの前半は v5.0 よりずっと前で、3.0 の setns() は namespace を FD で参照し、保持し、別 process へ渡せる object にしました。3.7 の VXLAN は L3 underlay 上の L2 overlay、3.19 の ipvlan は軽量な仮想 interface。これらが、一つの物理デバイスの上に複数の仮想ネットワークモデルを共存させる土台です。

その後は一つの後継系列ではなく、3 つの branch が並行して発展しました。VM datapath では virtio-net、vhost、5.7 の vDPA、SR-IOV と VFIO が並び、6.11 では virtio-net が AF\_XDP の RX zero-copy に対応しました。container datapath では veth と、6.7 の netkit。netkit は veth の単純な後継ではなく、peer device の内部に BPF の実行点を持つ BPF-first な設計です。そして queue assignment。7.1 の RX HW queue leasing は、仮想デバイスの queue と物理デバイスの queue の対応を明示し、AF\_XDP や memory provider 型の API をコンテナ側から使えるようにする割り当ての仕組みです。TX 側の leasing はまだ merge されていません。

Takeaway。netkit、virtio-net の zero-copy、queue leasing に共通するのは、full kernel bypass ではなく、カーネルの control と protection モデルを維持しながら data movement と overhead を減らす選択肢を増やしたことです。

### 22. TCP / UDP / transport: protocol semantics 自体の並行進化

transport は、この講演の中心命題だけでは説明しない領域です。表の 4 つのテーマを見てください。TFO は handshake と application data の境界を変えました。DCTCP は ECN を datacenter の輻輳 feedback として使い、BBR は loss ではなく bandwidth と RTT のモデルで送信を決め、PLB は ECMP 環境で path を transport 側から再選択し、AccECN は CE feedback の情報量を増やします。MPTCP は一つの論理 connection の下に複数の subflow を持ち、TCP-AO は MD5 signature の後継として認証を更新します。

これらは packet memory や BPF の実行 contract とは違って、transport が保持する connection、congestion、path、authentication の semantics そのものを変える系譜です。

左下の MPTCP の object model だけ補足します。「TCP connection = 1 path」という暗黙の対応が崩れ、論理 connection、subflow、endpoint、address advertisement、path manager が別々の状態になります。in-kernel の PM と userspace の PM があり、Netlink で endpoint と path の状態を制御します。ここだけは「暗黙の状態を名前付きの object と control API に出す」という今日の流れと強く接続しています。

右の注意点。TCP の上で使われるという理由だけで、BIG TCP や Device Memory TCP や struct\_ops の CC をここに再収容しません。それらは aggregation、memory、programmability の系譜です。

### 23. SECURITY: semantics / state と execution placement の分離

最後の横断ドメインは SECURITY です。ここで追うのは暗号方式の変遷ではなく、security の state や policy をカーネルが保持したまま、その実行を software、accelerator、NIC のどこで行うかを選べるようになった、という共通テーマです。

kTLS。4.13 の kTLS TX は、handshake と証明書・鍵交換の policy を userspace に残し、完了後の対称鍵の状態を setsockopt で TCP socket に渡して、TLS record の datapath をカーネルに移しました。4.17 で RX が加わり双方向に。4.18 と 4.19 の device offload で、同じ socket と API のまま crypto の実行を NIC に置けるようになりました。TLS\_SW と TLS\_HW は別のプロトコルではなく、実行位置の違いです。

XFRM。カーネルが SA と policy を canonical state として持ちます。4.12 の crypto offload では NIC が encrypt と decrypt を担当し、packet 構築や XFRM 処理の多くはカーネルに残ります。6.2 の packet offload では SA だけでなく policy も NIC と同期し、encapsulation と policy の実行まで NIC が担当できます。ここでは rekey、lifetime、policy の同期が contract の中心です。

WireGuard は違う方向です。5.6 で、L3 の secure tunnel を通常の network device driver として実装し、RTNL、genl、UDP tunnel API、GRO/GSO、NAPI といった既存のインフラに統合しました。XFRM の次世代でも kTLS と同じ offload 系譜でもなく、SECURITY と VIRTUAL/OVERLAY の交差点に置きます。

Takeaway。security の進化も専用 fast path への置換ではなく、semantics と state をカーネルが見える contract として保ち、execution placement と network integration を明示する方向です。

## スライド 24–26: Part IV Driver framework

### 24. Part IV — Network device driver framework（セクション扉）

Part III は packet path と control path を end-to-end の系譜として追いました。Part IV では視点を変えて、個々のドライバの責務のうち何が共通の networking-core framework に移されたかを見ます。個々のドライバが速くなった話ではなく、NIC が持つ queue、link、switch、health、memory、interrupt moderation をカーネル側から共通に扱えるようにした話です。

### 25. driver-local convention から共通 contract へ

左の「driver-local convention」から 8 本の矢印が出ています。これらは別々の subsystem で、一つの framework が他を置き換えた歴史ではありません。共通するのは「ドライバ内部にしか見えなかった資源や状態をカーネル共通の contract として表現する」ことです。

上から順に。3.3 の DQL/BQL は、各ドライバが経験則で決めていた outstanding な TX の量を共通の accounting model にしました。4.14 の phylink は、MAC driver が PHY、PCS、SFP、fixed-link を個別に扱う重複を減らし、link state の調整を共通の state machine に移しました。3.19 の switchdev は、bridge や FDB や VLAN のカーネル forwarding state を switch ASIC に同期する framework で、4.6 の devlink は port 単位では表せない device 全体の resource、parameter、health を扱う object です。

DIM は、packet や byte の rate を観測して interrupt coalescing を調整するロジックを、4.16 の初期実装から 5.3 で共通ライブラリにしたものです。measurement、policy decision、driver setting という流れが再利用可能になりました。page\_pool は Part III で話したとおり。6.8 以降の netdev-genl は queue と NAPI の identity を、5.6 の ethtool netlink は NIC configuration を structured API にしました。

Takeaway。driver-private だった state、resource、algorithm の境界をカーネル共通の contract に引き上げ、複数のドライバ、ハードウェア、実行モデルから再利用できるようにしました。

### 26. Modern driver contracts

3 つだけ掘り下げます。左の devlink。PCI device の下に ports、resources、parameters、regions、health reporters がぶら下がります。resources は TCAM のような有限なハードウェア資源を階層 object として公開し、occupancy やサイズ変更を control plane から扱えます。5.1 の devlink health は、diagnose、dump、recover という共通インターフェースで障害状態を扱います。devlink は datapath を提供しません。switchdev や TC offload が「どこで forwarding を実行するか」を扱うのに対し、devlink は device の resource、configuration、health の lifecycle を扱います。

真ん中の netdev-genl。物理 netdev の下に NAPI object、RX queue object、TX queue object、そして page\_pool の identity と stats。これは単なる可視化ではありません。AF\_XDP、ZCRX、memory provider、queue leasing のように特定の queue へ資源を bind する API が増えると、queue を安定して指し示す control-plane の identity が必要になるからです。queue identity は OBSERVABILITY と CONTROL PLANE の接点です。

右は execution context と capability。5.12 の threaded NAPI は、softirq ではなく kernel thread で poll を実行する選択肢を加えました。6.19 の busy-poll 拡張では、NAPI 単位で thread と affinity、scheduler policy を割り当てられます。「一つの最適な NAPI 実行モデル」に統一するのではなく、要件に応じて選べる。XDP features は「XDP 対応か否か」の二値から、redirect や zero-copy の capability を機械可読に公開する形へ。6.8 の Rust PHY abstraction は、ドライバ全体を Rust にするのではなく、PHY という明確な境界から C の ownership と lifetime のルールを Rust の型システムに写像した例です。netdevsim はハードウェアなしにこうした API を検証する test device で、auxiliary bus は一つの PCI device が持つ複数機能を別 driver へ bind する composition の仕組みです。

Takeaway。control plane は capability、resource identity、assignment を別々に問い合わせられるようになりました。

## スライド 27–28: Part V Observability

### 27. Part V — Observability / Explainability（セクション扉）

Part V はカーネル側の observability primitive の進化に限定します。programmable、offloaded、zero-copy の path が増えて複雑化した networking を、運用上説明可能にする evidence layer、という位置付けです。

### 28. Observability: 並行する evidence を相関する

表の 5 つの primitive は、一本道で進化した関係ではなく、異なる種類の evidence を並行して提供します。4.18 の BTF はカーネルの型と field の metadata を機械可読にし、CO-RE で layout の違いを relocation できます。内部構造の変化が速い networking では特に重要ですが、runtime の event や packet の軌跡そのものは記録しません。eBPF tracing は attach point での event と state を与えますが、coverage や権限、overhead に依存します。

5.17 の kfree\_skb\_reason と skb\_drop\_reason は、「パケットが消えた」という観測を「どの理由で drop されたか」という structured metadata に変えました。drop を発生させる側が semantic な reason を付与できる。ただし全 drop path が reason を付けるわけではありません。timestamping は「どこを通ったか」に対して「いつ起きたか」を与えます。そして 6.8 の netdev-genl と 7.1 の qdisc-drop tracepoint。queue や NAPI や page\_pool の identity と stats が見えるようになりましたが、見えることと、ownership や configuration を変更できることは別です。

左下は tool の case study。pwru は広い kernel function の軌跡から「packet がどこを通ったか」を探索する debugger、Retis は networking event や skb metadata、OVS/OVN の context を意味的に enrich して相関するツールです。両者は代替関係ではなく、カーネルの primitive が richer になるほど、raw trace の羅列から semantic な correlation へ進めます。

右の 3 点。cross-layer の packet journey は単一 primitive の機能ではなく、tool が複数の identifier、event、timestamp を相関して推論する目標です。Observability は 7 番目の contract form ではなく、contract と runtime behavior を検証する evidence layer です。IOAM はカーネル内部を外から観測するのではなく、network path 上で telemetry を packet 自身に運ぶので、観測対象と placement が異なります。

Takeaway。「counter から drop reason へ置き換わった」という一本道ではなく、counter、tracepoint、structured reason、identity が併存し、問いに応じて異なる evidence を提供します。

## スライド 29–34: Synthesis・まとめ・締め

### 29. Synthesis — thesis への回帰と未完の仕事（セクション扉）

最後に Part I の命題に戻ります。命題は 2 段でした。Linux networking は skb path を置き換えたのではなく、複数のモデルを共存させる architecture に拡張した。そしてそれを可能にしたのが explicit contracts だ。Part II から V を通過した今、この 2 段のつながりを具体的に言えます。新しい path は既存 path の隣に独立して追加されたのではなく、暗黙だった状態が contract として明示された箇所を接続点として、そこに追加された、ということです。

### 30. contract が共存を可能にした接続点

この表が今日の話の要約です。左が明示化された対象、真ん中が contract の系譜、右がその接続点で共存できるようになったものです。

queue occupancy が BQL から TSQ と pacing へ明示化されたから、socket と qdisc と driver が backlog を別々の層で制御できる。execution point が bpf() から XDP、TC、struct\_ops へ明示化されたから、built-in のスタックと検証済みプログラムが同じ packet や socket を扱える。packet memory が page\_pool から memory provider へ明示化されたから、system RAM と userspace buffer と device memory が同じ RX architecture に接続される。queue identity が AF\_XDP から netdev-genl、queue leasing へ明示化されたから、NIC queue を kernel stack、AF\_XDP、ZCRX、device-memory path、virtual device が使い分けられる。

下半分も同じ読み方です。forwarding と device state、control-plane state、policy と flow state、security state。いずれも、カーネルが canonical な状態を保持しながら、software と hardware、slow と fast、あるいは別の実行位置を選べるようになっています。

注意点を 2 つ。表の各行は「一つの contract が後続機能を直接生んだ」という因果ではなく、共存を可能にした architecture 上の接続点の要約です。そして OBSERVABILITY は独立した行ではなく、typed drop reason や queue identity を通じて、これらの接続点で実際に何が起きているかを検証する evidence layer として全行にかかります。

### 31. contract に還元しない並行系列

15 年の変化のすべてが前の表に収まるわけではありません。この講演は 15 年を少なくとも 4 つの並行する系列として読んでいます。一つ目が resource / control contract の明示化で、これが中心命題です。二つ目が transport と protocol semantics。TFO、DCTCP、BBR、MPTCP、PLB、AccECN、TCP-AO は、接続確立や輻輳制御、multipath、feedback、認証の semantics そのものを更新しました。三つ目が processing unit の拡大で、GRO/GSO と BIG TCP。四つ目が implementation scalability で、dev\_queue\_xmit() の llist 化のように contract を変えずに実装を速くする変更です。

routing と forwarding は第 5 の系列ではなく、CONTROL PLANE と VIRTUAL/OVERLAY、PROGRAMMABILITY と DRIVER FRAMEWORK が交差する接続点として扱います。

これらは thesis の例外ではなく、別の説明を要する並行系列です。今日の thesis は、どの系列を説明しているかを限定した上で成り立ちます。未来予測ではなく、既存の milestone を横断した説明モデルです。

### 32. 未完の仕事 — explicit contracts がまだ届いていない箇所

このスライドは「次に入る機能の一覧」ではなく、contract の適用範囲がまだ届いていない箇所として読んでください。

TX 側の queue assignment。RX HW queue leasing は 7.1 に入りましたが TX queue leasing は未 merge で、assignment contract は RX に偏っています。global synchronization の縮小。per-netns RTNL は一括置換ではなく複数リリースにわたる migration で、7.3-rc の RTNL-less FIB rule や netdev unregistration はその途中経過です。異種 memory の一般化。netmem と provider hook で境界は確定しましたが、より多くのドライバと memory type がこの model を使えるようにするのはこれからです。

execution placement の共通化。switchdev path と TC offload path は並行経路のままで、XDP も native、generic、offload で実行位置と必要な driver support が異なります。software と hardware のどちらで実行するかを扱う単一の contract はありません。observability の coverage。drop reason はすべての drop path に付くわけではなく、cross-layer の packet journey は今も tool 側の推論に依存します。Rust。PHY と core abstraction が中心で、一般的な high-performance NIC driver が Rust に移行した段階ではありません。

これらは将来のリリースで評価が変わり得るので、確定した歴史ではなく open questions として扱います。

### 33. まとめ

3 点でまとめます。一つ、置き換えではなく共存です。system RAM と skb の path は残り、AF\_XDP、device memory、io\_uring ZCRX、programmable や offloaded の path が同じ device 上で選択・共存できる architecture に拡張されました。

二つ、それを可能にしたのは explicit contracts です。driver-private、implicit、global だった状態を、object、accounting、API、assignment、lifetime、synchronization として明示した箇所が、新しい path の接続点になりました。

三つ、並行系列と evidence layer。transport semantics、processing unit、implementation scalability は別の系列です。observability は contract と runtime behavior を検証する evidence layer です。

一番下の一行が、今日の話を一本につなぐ問いの変化です。Era 1 の問いは「どれだけ queue すべきか」でした。Era 4 の問いは「placement、lifetime、synchronization をどこまで明示的に制御できるか」です。

### 34. Thank you

以上です。今日話したバージョン番号は、元ドキュメントの Part VI を release attribution の正本とし、Part VII で 48 個の mainline commit anchor を upstream の Git と照合しています。個々の機構の詳細は docs.kernel.org の netlink spec、page\_pool、devlink、switchdev、mptcp、msg\_zerocopy のドキュメントを参照してください。質問をお受けします。

**想定される質問と答えの軸**

- 「kernel bypass（DPDK 等）との比較は？」→ スライド 15 と 21 の論点。AF\_XDP、ZCRX、queue leasing は bypass ではなく kernel 管理の資源を貸す設計。Electrode / DINT / eTran などの研究も「protection model を保ったまま fast path を programmable にする」方向を補強。
- 「BIG TCP を有効にすればどの NIC でも速くなる？」→ スライド 13。protocol path、GRO/GSO/offload capability、driver、tunnel 実装に依存し、一律ではない。
- 「per-netns RTNL はもう完成？」→ スライド 19 / 32。複数リリースにわたる migration で、7.3 時点でも途中。
- 「nftables は BPF に置き換わる？」→ スライド 20。bpfilter は 6.8 で削除。定着したのは相互乗り入れ。
- 「この thesis で説明できないものは？」→ スライド 31。transport semantics、processing unit、implementation scalability は別系列として明示的に外している。
