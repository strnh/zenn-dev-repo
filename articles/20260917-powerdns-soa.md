---
title: "PowerDNS で native ゾーンを AXFR の primary/secondary に仕立て直す（FreeBSD jail 編）"
emoji: "🔄"
type: "tech"
topics: ["powerdns", "freebsd", "dns", "axfr", "letsencrypt"]
published: false
---

前回、certbot + PowerDNS API で `*.hoge.dom` のワイルドカード証明書を DNS-01 で取得するところまで書いた。

- [Lets'encrypt - DNS-01チャレンジ](https://zenn.dev/hikosakasohtaro/articles/8b00079732197a)

その `hoge.dom` ゾーンを、今度は素の **AXFR / NOTIFY** で primary/secondary 同期させようとしたのだが、一見単純に見える手順のあちこちで止まった。備忘を兼ねて詰まった箇所を並べておく。

なお、IP・ドメインは前回同様 `hoge.dom` / `203.0.113.x`（公開側）/ `10.0.0.5x`（jail 内部側）に読み替えている。SOA-EDIT-API の項は現在も検証中の内容を含む。

## 発端 — なぜ native のままだと困ったか

普段、権威サーバは gpgsql バックエンドを共有した **native ゾーン**で束ねている。この構成だと全サーバが同じ DB を見るので、そもそも「ゾーン転送で同期する」という発想自体が不要で、DNS-01 の `_acme-challenge` TXT を API で書けば、どのサーバに引かれても同じ内容が返る。前回 credentials で `dns_powerdns_disable_notify = true` としていても平気だったのはこのためである。

ところが今回、事情があって `hoge.dom` だけは DB 共有ではなく**素の AXFR で secondary に配りたい**、となった。ここで初めて、

- `hoge.dom` ゾーンを作ったときに kind を指定しておらず `native` のままだった
- native ゾーンは DNS プロトコルレベルの AXFR に**一切参加しない**

という事実に突き当たった、という次第。DNSSEC も Dynamic Update も使わない、ただの AXFR 同期のはずが、ここから芋づる式に止まる。

## 前提 — PowerDNS のレプリケーションは 3 種類

先に整理しておかないと最初の一歩で迷う。PowerDNS のゾーンには kind があり、複製のされ方が変わる。

| kind | 複製の方法 |
| --- | --- |
| `native` | DB バックエンド側で複製する（gpgsql レプリケーション等）。**AXFR には参加しない** |
| `primary` / `secondary` | AXFR + NOTIFY で明示的に転送する。ゾーンは各サーバに個別に持たせる |
| autoprimary / autosecondary | secondary が NOTIFY を受けたゾーンを自動追加する |

`primary` / `secondary` は以前の `master` / `slave` の新名称で、`pdnsutil` は互換のため古い名前も受け付ける。今回やりたいのは真ん中。まず `native` から抜けるところからになる。

## 1. ゾーンの kind を primary / secondary にする

```
# 現状確認（kind を見る）
$ pdnsutil list-all-zones
$ pdnsutil show-zone hoge.dom

# primary 側
$ pdnsutil set-kind hoge.dom primary

# secondary 側
$ pdnsutil set-kind hoge.dom secondary
```

`set-kind` に渡すのは `primary` / `secondary` / `native`（旧 `master` / `slave` も可）。ここを変えない限り、以降の設定は全部空振りする。native のまま NOTIFY を投げても AXFR は起きない。

## 2. pdns.conf の ACL（primary / secondary で別物）

プロトコルとしての許可を出す。**primary は「誰に AXFR させるか」、secondary は「誰からの NOTIFY を受けるか」**を書く。方向が逆なので混乱しやすい。

primary 側 `pdns.conf`

```
# 転送を許可する secondary の IP
allow-axfr-ips=203.0.113.34

# secondary が ゾーンの NS に載っていない（隠し secondary 等）なら明示 NOTIFY
# also-notify=203.0.113.34
```

secondary 側 `pdns.conf`

```
# NOTIFY を受け付ける primary の IP
allow-notify-from=203.0.113.33
```

- 忘れがちなのが、secondary として実際に取りに行く動作の有効化。`secondary=yes`（旧 `slave=yes`）が要る。primary から NOTIFY を送るなら `primary=yes`（旧 `master=yes`）。ACL だけ入れて有効化を入れ忘れると、NOTIFY を受けても何もしない。
- `allow-axfr-ips` はグローバル設定だが、ゾーン単位で `ALLOW-AXFR-FROM` メタデータでも指定できる（後述の metadata コマンド）。

## 3. secondary に primary のアドレスを登録する

kind を変えるだけでなく、secondary には「どこが primary か」を教える必要がある。

```
# 既存の secondary ゾーンの primary を設定・変更する
$ pdnsutil change-secondary-zone-primary hoge.dom 203.0.113.33

# ゼロから secondary ゾーンを作るなら
$ pdnsutil create-secondary-zone hoge.dom 203.0.113.33
```

primary は空白区切りで複数、`IP:port` 形式も指定可。

- メモでは `change-primary` と略していたが、正式名は `change-secondary-zone-primary`（5.0 より前は `change-slave-zone-master`）。手元のバージョンで `pdnsutil` を引数なし実行し、使えるサブコマンド名を確認するのが確実。

## 4. SOA-EDIT-API — serial が上がらないと AXFR は起きない（検証中）

ここが一番「動いているように見えて動かない」ところ。AXFR は **secondary が SOA の serial を比較し、primary の方が新しいときだけ**転送する。つまり primary 側でレコードを変えても serial が増えないと、NOTIFY は飛ぶのに secondary は「変わっていない」と判断して AXFR しない。

`pdnsutil` で作ったゾーンは、環境によっては `SOA-EDIT-API` が未設定（＝ `NONE` 相当）で、API や `pdnsutil` 経由の変更で serial が自動増加しないことがある。前回の DNS-01 は `_acme-challenge` TXT を **API で書き換える**手順だったので、まさにこの serial 自動増加が効いていないと、TXT を書いても secondary に伝わらない、という形で顕在化する。

明示的にセットする。

```
# 5.0 以降
$ pdnsutil metadata set hoge.dom SOA-EDIT-API INCEPTION-INCREMENT

# 5.0 より前
$ pdnsutil set-meta hoge.dom SOA-EDIT-API INCEPTION-INCREMENT

# 確認（5.0 以降 / それ以前）
$ pdnsutil metadata get hoge.dom
$ pdnsutil get-meta hoge.dom
```

`INCEPTION-INCREMENT` は `YYYYMMDDnn` 形式でその日の連番を回してくれる、よく使われる値。手動運用中心なら `pdnsutil increase-serial hoge.dom` で明示的に上げる手もある。

> この項目は「serial が上がらないと AXFR が起きない」という切り分けまでは取れているが、`SOA-EDIT-API` の最適値（`DEFAULT` / `INCEPTION-INCREMENT` / `EPOCH` 等）と、前回プラグインの `dns_powerdns_disable_notify` との兼ね合いは引き続き検証中。

## 5. FreeBSD の jail に置くと送信元 IP がずれる

今回いちばんハマったのがこれ。jail に DNS を置き、外からは RDR で公開 IP → jail 内部 IP に転送している構成だと、**NOTIFY と AXFR で「相手から見える送信元アドレス」が一致しない**ことがある。

- primary(`203.0.113.33` / 内部 `10.0.0.53`) から secondary の**公開 IP** `203.0.113.34` に NOTIFY を送る
- ところが secondary（jail 内 pdns）が受け取った NOTIFY の**送信元**は、経路（同一ホスト内のヘアピン等）によっては primary の**内部 IP** `10.0.0.53` に見える
- 逆に secondary が公開 IP `203.0.113.33` 宛てに AXFR を張り返すと、primary(jail 内) が観測する AXFR の**送信元**も、NAT/RDR を経て公開側とは別のアドレスになりうる

`allow-notify-from` も `allow-axfr-ips` も「**受信プロセスが実際に観測した送信元 IP**」で判定するので、想定していた「公開 IP」だけで ACL を書くと弾かれる。前回 API を `10.0.0.53:8081` の内部側で叩いていた感覚のまま公開 IP を書くと、ここでズレる。

対処は単純で、**実際に観測される送信元アドレスを確認してから ACL に入れる**。

```
# 受信側で実際の送信元を見る
$ tcpdump -ni <if> port 53
```

pdns 側のログ（`loglevel` を上げる、`log-dns-queries=yes`）でも拒否された AXFR/NOTIFY の送信元が出るので突き合わせられる。公開・内部の両方を入れておくのが安全なこともある。

## 動作確認 — NOTIFY → AXFR を手で叩く

自動 NOTIFY を待たず、経路とレプリケーションが生きているかを手動で確認できる。

```
# primary から secondary へ NOTIFY を明示送信
$ pdns_notify 203.0.113.34 hoge.dom
# あるいは稼働中の pdns 経由で
$ pdns_control notify hoge.dom

# secondary 側で強制的に取りに行かせる
$ pdns_control retrieve hoge.dom

# serial を突き合わせる
$ dig +short SOA hoge.dom @203.0.113.33   # primary
$ dig +short SOA hoge.dom @203.0.113.34   # secondary
```

`pdns_notify` を手で叩いて NOTIFY → AXFR が走り、両者の serial が揃うところまでは確認できた。これで「経路と ACL は通っている」「serial さえ上がれば転送される」の 2 点が切り分けられる。あとは 4 の SOA-EDIT-API を詰めれば、API 経由の TXT 更新（＝前回の DNS-01 自動化）でも secondary まで自動で回るはず、というのが現状。

## まとめ — ハマりどころチェックリスト

- ゾーンの kind が `native` のままになっていないか（`pdnsutil show-zone`）
- primary/secondary を `pdnsutil set-kind` で明示したか
- secondary に primary アドレスを登録したか（`change-secondary-zone-primary`）
- `pdns.conf` に primary `allow-axfr-ips` / secondary `allow-notify-from`
- `primary=yes` / `secondary=yes`（旧 `master`/`slave`）を入れ忘れていないか
- serial が上がる仕組みがあるか（`SOA-EDIT-API` もしくは手動 `increase-serial`）
- jail/NAT/RDR 環境で、ACL に入れる IP が「実際に観測される送信元」になっているか
- `pdns_notify` / `pdns_control retrieve` + `dig SOA` で切り分けたか

native + 共有 DB で回してきた人ほど、AXFR 特有の「kind」「serial 比較」「送信元 ACL」の 3 点でつまずきやすい、というのが今回の教訓。前回の DNS-01 自動化と組み合わせると、証明書の challenge TXT が secondary までちゃんと伝播する、というところまで繋がる。

## 参考

- PowerDNS 公式ドキュメント: Modes of Operation / Domain Metadata / `pdnsutil` manpage
- 前回記事: [Lets'encrypt - DNS-01チャレンジ](https://zenn.dev/hikosakasohtaro/articles/8b00079732197a)
