linux-networking-evolution-v3.0-to-7.x-canonical-edition-r24.md に載っていないネットワーク関連の主な更新

## v3.x

| 初収録 | 未掲載の機能 | 導入コミット |
|---|---|---|
| v3.3 | Ethernet _team_ デバイス。複数ポートをまとめ、Generic Netlink で制御 | `3d249d4ca7d0ed6629a135ea1ea21c72286c0d80` |
| v3.4 | `SO_PEEK_OFF`。ソケットの `MSG_PEEK` 開始位置を指定 | `ef64a54f6e558155b4f149bb10666b9e914b6c54` |
| v3.5 | `TCP_REPAIR`。TCP ソケットの状態を復元するための repair mode | `ee9952831cfd0bbe834f4a26489d7dce74582e37` |
| v3.7 | netfilter の IPv6 NAT | `58a317f1061c894d2344c0b6a18ab4a64b69b815` |
| v3.8 | TUN/TAP の複数キュー。複数のファイル記述子をキューとして使用 | `c8d68e6be1c3b242f1c598595830890b65cea64a` |
| v3.9 | Linux bridge の VLAN フィルタリング基盤 | `243a2e63f5f47763b802e9dee8dbf1611a1c1322` |
| v3.10 | TCP Tail Loss Probe（TLP）。末尾のパケット損失による再送タイムアウトを減らす | `6ba8a3b19e764b6a65e4030ab0999be50c291e6c` |
| v3.12 | `TCP_NOTSENT_LOWAT`。送信待ちの未送出データ量を制御 | `c9bee3b7fdecb0c1d070c7b54113b3bdfb9a3d36` |
| v3.14 | TCP autocorking。小さな書き込みを送信前にまとめる | `f54b311142a92ea2e42598e347b84e1655caf8e3` |
| v3.16 | bridge の 802.1ad VLAN フィルタリング | `204177f3f30c2dbd2db0aa62b5e9cf9029786450` |
| v3.17 | SCTP の `SCTP_SNDINFO` 制御メッセージ（RFC 6458） | `63b949382c5f263746b1c177f6ff84de2201ae9d` |
| v3.18 | FOU（Foo-over-UDP）の受信処理 | `23461551c00628c3f3fe9cf837bf53cf8f212b63` |
| v3.18 | GUE（Generic UDP Encapsulation）の受信処理 | `37dd0247797b168ad1cc7f5dbec825a1ee66535b` |
| v4.0 | peer network namespace ID を登録・取得する RTNL API | `0c7aecd4bde4b7302cd41986d3a29e4f0b0ed218` |

## v4.x

| 初収録 | 機能 | 導入コミット |
|---|---|---|
| v4.1 | MPLS パケットのラベルによるルーティング | `0189197f441602acdca3f97750d392a895b778fd` |
| v4.2 | `tc` の Flower 分類器。パケットのフィールドに基づくフローフィルタ | `77b9900ef53ae047e36a37d13a2aa33bb2d60641` |
| v4.2 | nftables の netdev ingress テーブルとチェーン | `ed6c4136f1571bd6ab362afc3410905a8a69ca42` |
| v4.3 | VRF デバイスによるルーティング領域の分離 | `193125dbd8eb292d88feb201f030889b488b0a02` |
| v4.4 | TCP RACK による、送信時刻を使った損失検出 | `4f41b1c58a32537542f14c1150099131613a5e8a` |
| v4.4 | `SO_INCOMING_CPU` の設定機能。受信 CPU に応じた `SO_REUSEPORT` ソケット選択 | `70da268b569d32a9fddeea85dc18043de9d89f89` |
| v4.5 | `SO_ATTACH_REUSEPORT_CBPF/EBPF` による受信ソケット選択 | `538950a1b7527a0a52ccd9337e3fcd304f027f13` |
| v4.6 | MACsec（IEEE 802.1AE）の暗号化ネットワークデバイス | `c09440f7dcb304002dfced8c0fea289eb25f2da0` |
| v4.7 | GTP-U トンネルのカーネル内データパス | `459aa660eb1d8ce67080da1983bb81d716aa5a69` |
| v4.10 | IPv6 Segment Routing Header のデータパス処理 | `1ababeba4a21f3dba3da3523c670b207fb2feb62` |
| v4.14 | SRv6 の `seg6local` lightweight tunnel アクション | `d1df6fd8a1d22d37cffa0075ab8ad423ce656777` |
| v4.14 | ERSPAN Type II のネイティブトンネル | `84e54fe0a5eaed696dee4019c396f8396f5a908b` |
| v4.15 | Credit Based Shaper（CBS）qdisc | `585d763af09cc21daf48ecc873604ccdb70f6014` |
| v4.17 | kTLS のソフトウェア受信・復号経路。r24 は kTLS TX を掲載 | `c46234ebb4d1eee5e09819f49169e51cfc6eb909` |
| v4.19 | Earliest TxTime First（ETF）qdisc。r24 掲載の `SO_TXTIME` を使って送信時刻順に処理 | `25db26a91364db00f5a30da2fea8e9afe14a163c` |
| v4.19 | TLS 受信処理のネットワークデバイスへのオフロード機能 | `14136564c8ee94566945e85014019cbdb1716dca` |
| v4.19 | `BPF_PROG_TYPE_SK_REUSEPORT`。reuseport 配列からソケットを選ぶ専用 BPF プログラム | `2dbb9b9e6df67d444fbe425c7f6014858d337adf` |


## v5.x

| 初収録 | 機能 | 根拠となるコミット |
|---|---|---|
| v5.3 | TC での MPLS ヘッダーの追加・削除・変更 | `2a2ea50870baa3fb4de0872c5b60828138654ca7` |
| v5.3 | TC の `ct` action。パケットを conntrack に渡し、その状態を分類に利用 | `b57dc7c13ea90e09ae15f821d2583fa0231b4935` |
| v5.3 | TC の `ctinfo` action。conntrack mark から DSCP や skb mark を復元 | `24ec483cec981618f8a4782a36d1e3f319d42cad` |
| v5.3 | nftables の SYN proxy | `ad49d86e07a497e834cb06f2b151dccd75f8e148` |
| v5.6 | Flow Queue PIE qdisc | `ec97ecf1ebe485a17cd8395a5f35e6b80b57665a` |
| v5.6 | Enhanced Transmission Selection（ETS）qdisc | `dcc68b4d8084e1ac9af0d4022d6b1aff6a139a33` |
| v5.6 | ESP-in-TCP。IKE/IPsec メッセージを TCP でカプセル化 | `e27cca96cd68fa2c6814c90f9a1cfd36bb68c593` |
| v5.7 | Bareudp。MPLS・IP・NSH などを運ぶ汎用 UDP トンネル | `571912c69f0ed731bd1e071ade9dc7ca4aa52065` |
| v5.7 | IPv6 RPL Source Routing Header の受信処理 | `8610c7c6e3bd647ff98d21c8bc0580e77bc2f8b3` |
| v5.7 | UDP ソケットに対する BPF sockmap の基本フック | `edc6741cc66059532ba621928e3f1b02a53a2f39` |
| v5.8 | TC の gate action。指定した時間枠ごとにフレームの通過を制御 | `a51c328df3106663879645680609eb49b3ff6444` |
| v5.8 | Linux bridge の Media Redundancy Protocol（MRP）統合 | `6536993371fab3de4e8379649b60e94d03e6ff37` |
| v5.9 | Parallel Redundancy Protocol（PRP）のパケット処理 | `451d8123f89791bb628277c0bdb4cae34a3563e6` |
| v5.10 | TC BPF の `redirect_peer`。ネットワーク名前空間をまたぐ veth への転送 | `9aa1206e8f48222f35a0c809f33b2f4aaa1e2661` |
| v5.11 | preferred busy polling。アプリケーション側での NAPI 処理を優先 | `7fd3253a7de6a317a0683f83739479fb880bffc8` |
| v5.15 | IPv6 IOAM Pre-allocated Trace のデータ処理と、経路設定による挿入 | `9ee11f0fff205b4b3df9750bff5e94f97c71b6a0`、`3edede08ff37c6a9370510508d5eeb54890baf47` |

## v6.x

| 初収録 | 機能 | 根拠となるコミット |
|---|---|---|
| v6.1 | CAN XL のフレーム定義と CAN RAW ソケットでの送受信 | `1a3e3034c049503ec6992a4a7d573e7fff31fac4`、`626332696d7506e8f844a564277bdba2dc78fcb5` |
| v6.2 | bridge の MAC Authentication Bypass（MAB） | `a35ec8e38cdd1766f29924ca391a01de20163931` |
| v6.2 | nftables でのトンネル内側ヘッダーの照合 | `3a07327d10a09379315c844c63f27941f5081e0a` |
| v6.3 | XDP から受信メタデータを取得する kfunc | `3d76a4d3d4e591af3e789698affaad88a5a8e8ab` |
| v6.4 | netfilter フックで実行する BPF プログラム | `fd9c663b9ad67dedfc9a3fd3429ddd3e83782b4d` |
| v6.4 | vsock ソケットの BPF sockmap 対応 | `634f1a7110b439c65fd8a809171c1d2d28bcea6f` |
| v6.6 | MPTCP のパケットスケジューラ登録 API | `740ebe35bd3f5c4ff8ec60e5e521e47ea8f5492c` |
| v6.8 | AF\_XDP 送信メタデータによるタイムスタンプ・チェックサムオフロード要求 | `48eb03dd26304c24f03bdbb9382e89c8564e71df` |
| v6.11 | XDP プログラムから netfilter flowtable を検索する kfunc | `391bb6594fd3a567efb1cd3efc8136c78c4c9e31` |
| v6.14 | IP-TFS の XFRM モード。IPsec トンネル内で小パケットの集約と断片化を扱う | `4b3faf610cc63bfac972711635eafbca5e7d7117` |
| v6.14 | `SO_RCVPRIORITY`。受信パケットの優先度を補助データとして取得 | `e45469e594b255ef8d750ed5576698743450d2ac` |
| v6.15 | `TCP_RTO_MIN_US`。ソケット単位で最小再送タイムアウトを設定・取得 | `f38805c5d26fe4af97837c10d58074a7496638bf` |
| v6.15 | Netlink による IPv4 マルチキャスト参加アドレスの一覧取得 | `eb4e17a1d915d3c550e5a58c4bf370659dbe0dc8` |
| v6.16 | OpenVPN Data Channel Offload（`ovpn`）のカーネル内データ経路 | `9f23d943eb6b55990acc45ea1e130b20a44c76ce` |
| v6.17 | DualPI2 qdisc。L4S と従来型トラフィックを扱うキュー制御 | `8f9516daedd67097a0c6e463fcb7a42b5ee9d477` |
| v7.0 | mac80211 の初期 UHR 接続・AP ステーション対応 | `a1085114715ee9980405d6856276c5e88339cee7` |

## v7.x

| 初収録 | 機能 | 根拠となるコミット |
|---|---|---|
| v7.1 | ドライバがスリープ可能な文脈で受信モードを更新する `ndo_set_rx_mode_async` | `3554b4345d855089ab7af5e3557f5dc3262d14c9` |
| v7.1 | bridge ごとの STP モード選択。別のネットワーク名前空間でも userspace STP を選択可能に | `54fc83a1728535831df0f251e155d05574918115` |
| v7.1 | nftables で二重 VLAN タグと PPPoE の内側のプロトコルを照合 | `3785091c6c16a1ce4a5e0460881fc81ed8d2c8a1` |
| v7.1 | Wi-Fi NAN のデータ通信用インターフェース | `0e8ec738a71ee4e8da7c56d21dd7bb54f954c38b` |
| v7.1 | SRv6 トンネルの送信元アドレスを経路単位で設定 | `78723a62b969af404fde2468bb9782519d5f8ba7` |
| v7.2 | bridge で、近隣探索を抑制しながら gratuitous ARP・非要請 Neighbor Advertisement は転送 | `27c082c600b1df749cafa57edf5815e951655a3c` |
| v7.2 | `XFRM_MSG_MIGRATE_STATE` による単一 IPsec SA の移行 | `a9d155ea9b44d9b979796506bec518222f10b9e6` |
| v7.2 | IPVS の接続数上限を調整する `conn_max` | `4a15044a2b06748c99a8c8c3c6b3ee0a01f8004d` |
| v7.2 | devlink パラメータの可変長 `u64` 配列値 | `eb7b4d458e0d6833ffbb717edf4282f5ca6a7b57` |
| v7.2 | ネットワークデバイス用 work の共通基盤 `netdev_work` | `12c765be84d28f22deca10e775889f54bd571a85` |
| v7.3-rc | BPF LSM プログラムから UDP ソケットを作成・使用する `ksock` kfunc | `7ae4eb14c5f9d9bf0e0feabeab206151b1280512` |
| v7.3-rc | BPF の FIB 検索で VLAN 送信先・入力 VLAN を解決 | `35dac1daeb3c8208515047d32f22c4a162e8de5f`、`217828aad80d091fa1d840587a3d9b6187ee170f` |
| v7.3-rc | BPF プログラムから ICMP 応答を送る `bpf_icmp_send` | `f3603df9aebb2a2fe2f745bd71ca38aeca60e6e7` |
| v7.3-rc | `bpf_redirect_peer` の egress 方向への拡張 | `509ca545d425512f83ca70093f6d836ec8ab5bd1` |
| v7.3-rc | VXLAN の FDB nexthop ごとに UDP 宛先ポートを指定 | `53531e6a644a48c2d5a9423f084ae4e91ec6019a` |
| v7.3-rc | PRP RedBox の interlink 転送と重複フレーム除去 | `841afc9143ee340818a4a9cf191f4e894a43c750` |
| v7.3-rc | nftables で NAT されたフローの conntrack expectation を作成。ただしホスト起点などの条件あり | `d4beefc90a66672e43fdf82b43e4b3c0b1b18c5e` |
