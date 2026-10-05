# Linux Networking Evolution — r28 audit / editorial worklog

このファイルは読者向け本編ではなく、canonical edition を更新するための作業ログである。
監査手順、候補、改訂履歴を本編から分離する。

## rc-tag containment audit policy

`First containing rc tag` は final release から推測しない。必要な場合は Torvalds tree で ancestry を確認する。

```bash
git merge-base --is-ancestor <sha> vX.Y-rc1 && echo contained
git tag --contains <sha> --sort=version:refname | grep -E '^v[0-9]+\.[0-9]+(-rc[0-9]+)?$' | head
```

r27 時点では rc-tag 列の大半が `open` だったため、r28 では本文の provenance ledger から列自体を外した。
監査が完了した値は将来必要なら machine-readable ledger に保持する。

## r28 editorial decisions

- Part I の「共存」と `explicit resource / control contracts` を一本化し、**明示的 contract が複数 execution / memory model の共存を可能にした**という関係にした。
- Era 4 は全体 thesis と名前が衝突しないよう **Explicit placement & scoped control** に改称した。
- Era 1 foundations は Part II に吸収し、Part III は6軸に近い順序へ再配置した。
- Part III の routing/TC/offload にあった r27 audit table と revision-context prose を削除した。
- Part V を「である」体へ寄せた。
- Part VI の Evidence class は reader-facing な `Ref / status` に簡素化した。
- selection rule と衝突していた `MPTCP PM limit expansion` と `PPPoE GRO/GSO` は canonical rows から外した。重要性の否定ではない。
- `netmem`, memory providers, BTF, vDPA, page_pool introspection は release を推測せず、Part VI の boundary-open register に可視化した。
- 7.3 表記は `7.3-rc / mainline`（final release 未公開）へ寄せた。
- BIG TCP の世代間関係は direct extension を意味する `⇒` ではなく、successive capability expansion の `→` とした。
- changelog / candidate promotion history は本ファイルへ移した。

## Candidate set carried from r27

## Unpromoted milestone candidate set — r25 review

添付の `exclusions.md` は canonical table の「漏れ一覧」ではなく、**次の再監査候補集**として扱う。r25 では reviewer が指摘した routing / TC / offload の空白を埋めるため、bridge VLAN filtering (3.9)、MPLS routing (4.1)、Flower (4.2)、VRF (4.3)、TC `ct` action (5.3) の5件だけを Part VI / VII へ昇格した。

その他の TUN/TAP multiqueue、TLP/RACK、ETF、CBS、GTP-U、SRv6、preferred busy polling、XDP RX/TX metadata、DualPI2、`netdev_work` 等は重要な機能だが、現行の採用基準では「本文の長期 lineage を代表する canonical point」とする追加理由を個別に監査してから昇格する。**未掲載 = 重要でない**ではない。

## Previous changelog carried from r27

# Changelog / Errata（非正規）
- **r27:** routing / TC / offload の5 milestone に primary-source audit status を追加。bridge VLAN filtering のorigin commit、AF_MPLS の4.1導入、Flower origin SHA、TC `ct` のconntrack metadata semantics を再確認。`First containing rc tag` は検索や日付から推定せず、git object graph の ancestry 確認が終わるまで `open` を維持。`git merge-base --is-ancestor` / `git tag --contains` による再現可能な監査手順を明記。
- **r26:** canonical verification procedure を強化。`First containing rc tag` は git ancestry または同等の一次資料で containment を確認した場合だけ記入することを明文化。routing / TC / offload の5件について final release + 40桁 anchor の再監査表を本文に追加し、switchdev object path と TC `ndo_setup_tc` / flow-block path が並行経路であることを明示。一次資料への verification entry points も追加。
- **r25:** routing / TC / offload の短い lineage を追加し、bridge VLAN filtering (3.9)、MPLS routing (4.1)、Flower (4.2)、VRF (4.3)、TC `ct` action (5.3) を canonical milestone / exact-anchor inventory に追加。
- **r25:** switchdev 図を bridge/FIB/VLAN の switchdev path と TC の `ndo_setup_tc` / flow-block path の並行経路へ修正。
- **r25:** Part VII の `First containing tag` を `First containing rc tag` と `First final release` に分離し、未監査 rc containment は `open` のまま維持。Part VI に canonical verification bundle を追加。
- **r25:** synthesis で explicit-contract thesis と transport/protocol、processing-unit/batching、implementation-scalability、routing/forwarding の並行進化を同格に明記。
- **r24:** RTNL の progression を `⇒` から `→` に変更。一つの feature の direct extension ではなく、global RTNL dependency を縮小する複数の locking/refactoring techniques の lineage として扱う。BIG TCP など direct extension が明確な系列の `⇒` は維持。

- **r23:** WireGuard 5.6 を `SECURITY / VIRTUAL / OVERLAY` とし、`evidence grade` の残語を `Ref / status` へ修正。netmem / memory providers の Part VI 未収載が意図的な open provenance であることを明記。
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

- **r18:** Part VI を Axis / Domain の独立列へ変更し、Era/Grade 列を削除。束ね行を分割し、Ref / status を唯一の evidence-status 列とした。

- **r18:** Part I の baseline を v3.0 / v5.0 / 2026 に整理し、軸→本文節の地図と図記法を追加。

- **r18:** PERFORMANCE の queueing lineage を独立化し、io_uring / 6.19 cross-lineage の重複を整理。

- **r18:** Part IV から raw SHA / release-attribution / 編集経緯を除去し、Part VII の open items を未解決 provenance に限定。

この節は canonical chronology / provenance の一部ではない。版間の編集上の変更だけを記録する。

- **r17:** Part VI を1行1 milestoneへ正規化し、Axis/domain・Era・Ref / status を追加。 evidence 語彙を統制し、selection policy を Part VI 冒頭へ移動。
- **r17:** Part I に axis/contract 対応表と thesis の非適用領域を追加。
- **r17:** Part IV の major-version 見出しを architecture-oriented headings に変更。
- **r17:** conclusion/synthesis を chronology の前に追加。
- **r17:** Part VII から旧版監査経緯を分離し、短縮 SHA を exact inventory として扱わない方針を明記。

## r28

- 読み物と監査ログを分離。
- thesis / era / Part III ordering / Part VI evidence presentation を再構成。
- SECURITY/kTLS の追加検討はこの構造整理の後に行う。
