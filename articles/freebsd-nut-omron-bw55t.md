---
title: "FreeBSD + NUT で Omron BW55T が認識されない — 犯人は uhid だった"
emoji: "🔋"
type: "tech"
topics: ["freebsd", "nut", "ups", "omron"]
published: false
---

## はじめに

FreeBSD 上の NUT (Network UPS Tools) で Omron BW55T を監視しようとしたところ、`blazer_usb` がデバイスを掴めず `No supported devices found` で落ち続けた。最終的な原因はカーネルの `uhid` ドライバがデバイスを専有していたことで、`hw.usb.quirk` に `UQ_HID_IGNORE` を追加して解決した。

`No appropriate HID device found` という NUT のメッセージが「HID デバイスが見つからない」ではなく「HID として **カーネルに取られていて libusb から開けない**」を意味していた、という読み違えが遠回りの元だったので、その顛末と検証手順を残しておく。

:::message
検証対象がリモートにあり、UPS 認識後にシャットダウンを走らせてしまった都合で追跡ログを残せていない。本記事のコマンド出力は再現用の代表例（こう見えるはず、という期待値）であり、実キャプチャそのものではない点に注意してほしい。
:::

## 環境

- FreeBSD 14.x（PRIMERGY）
- NUT 2.8.5 系（ports / pkg）
- Omron BW55T（USB VID:PID = `0590:00d0`）

## 症状

`blazer_usb` を起動すると、内蔵の対応デバイステーブルを一通り舐めた末にスキップされる。

```text
[D2] Trying to match device
[D2] Device does not match - skipping
[D2] libusb1: No appropriate HID device found
No supported devices found. Please check your device availability with 'lsusb'
```

`vendorid` / `productid` を明示しても変わらない。Omron の VID (`0590`) は `blazer_usb` の内蔵テーブルに無く、明示指定は「既にテーブルにある候補を絞り込む」だけなので、テーブルに無いデバイスは追加されない。ここで `subdriver = ippon` を足すのが BW シリーズの定石なのだが、**それでもテーブル走査が走ってスキップされる**、という状態に陥った。

## 遠回りした誤診

最初は権限問題を疑った。`blazer_usb` は起動時に `become_user(nut)` で `nut` ユーザに降格してから USB を列挙するため、root shell から叩いても実際には `nut` 権限で見に行く。ここが原因なら root では通るはず——だが root でも掴めない。この時点で権限説は消える。

次に `subdriver=ippon` の指定漏れを疑ったが、コマンドラインで明示しても症状は同じ。ここでようやく「libusb 以前の問題では?」と気づいた。

## 真因: uhid がデバイスを専有していた

FreeBSD では HID クラスの USB デバイスにカーネルの `uhid`（あるいは他の HID ドライバ）が先にアタッチする。Omron UPS は HID デバイスとして振る舞うため `uhid` に掴まれ、`ugen` の生 USB ノードとして libusb から開けなくなる。`blazer_usb` / `nutdrv_qx` は libusb で直接デバイスを叩くので、`uhid` に専有されていると `No appropriate HID device found` を返す。

つまりこのメッセージは「HID が無い」ではなく「HID として **既に取られている**」のサインだった。

## 対処: UQ_HID_IGNORE

`uhid` にこのデバイスを無視させ、`ugen` ノードとして残す。VID/PID を指定した quirk を入れる。

`/boot/loader.conf`:

```conf
hw.usb.quirk.0="0x0590 0x00d0 0x0000 0xffff UQ_HID_IGNORE"
```

quirk は列挙（アタッチ）時に評価されるため、`loader.conf` に書いた場合は反映に **再起動** が要る。複数デバイスに付けるときはスロット番号を `.0` `.1` `.2` … と連番にしてインデックス衝突を避ける。

再起動せず試したい場合は、揮発的に投入して再プローブする手もある（バス.アドレスは `usbconfig list` で確認）。

```sh
# usbconfig -d ugen0.2 add_quirk UQ_HID_IGNORE
# usbconfig -d ugen0.2 reset
```

## 検証手順（再現）

### 1. デバイスが列挙されているか

FreeBSD では `lsusb` ではなく `usbconfig`。NUT が出す「`lsusb` を使え」は Linux 前提の定型文なので読み替える。

```sh
# usbconfig list
ugen0.2: <OMRON BW55T> at usbus0, cfg=0 md=HOST spd=FULL (12Mbps) pwr=ON (2mA)
```

### 2. uhid が掴んでいないか（＝真因の確認）

これが今回の肝。`dmesg` に `uhid` が UPS へアタッチした形跡があれば当たり。

```sh
# dmesg | grep -i uhid
uhid0: <OMRON BW55T, class 0/0, rev 1.10/1.00, addr 2> on usbus0
```

quirk を入れて再起動（または再プローブ）した後は、`uhid` が消えて `ugen` のみになっていることを確認する。

```sh
# dmesg | grep -iE 'ugen|uhid'
ugen0.2: <OMRON OMRON BW55T> at usbus0
```

### 3. blazer_usb で認識確認

quirk が効いていれば、テーブル走査を飛ばして対象デバイスへ直行する。

```sh
# /usr/local/libexec/nut/blazer_usb -a BW55T -DDDD \
    -x subdriver=ippon -x vendorid=0590 -x productid=00d0
...
[D2] Checking device 1 of 1 (0590/00D0)
[D2] - VendorID: 0590
[D2] - ProductID: 00d0
[D2] - Manufacturer: OMRON
[D2] - Product: BW55T
```

`Checking device 1 of 1 (0590/00D0)` まで来れば成功。ここで再び内蔵テーブルの走査（`0x05B8`, `0x06DA` … を順に照合）が見えるなら、quirk か `-x` オプションが効いていない。

### 4. devd で権限を恒久化

`uhid` が外れて `ugen` ノードになった今こそ、そのノードを group `nut` に渡す devd ルールが効く。`ugen` は再アタッチのたびに作り直されるので、手動 chown ではなく devd で自動化する。ports は雛形を同梱している。

```sh
# ls /usr/local/etc/devd/nut*
# cp nut-usb.conf.sample nut-usb.conf  # 無ければ雛形から
# service devd restart
```

雛形が合わなければ最小ルールを置く。

```conf
notify 100 {
    match "system"    "USB";
    match "subsystem" "DEVICE";
    match "type"      "ATTACH";
    match "vendor"    "0x0590";
    match "product"   "0x00d0";
    action "chgrp nut /dev/$cdev; chmod g+rw /dev/$cdev";
};
```

quirk（uhid を外す）と devd（`nut` に権限を渡す）は **別レイヤーの別問題**なので、両方入れないと「root では通るが `nut` 起動では開けない」に逆戻りする。

### 5. 値の取得

```sh
# upsc BW55T
battery.charge: 100
battery.voltage: 13.7
device.mfr: OMRON
device.model: BW55T
driver.name: blazer_usb
input.voltage: 101.2
output.voltage: 101.2
ups.load: 12
ups.status: OL
```

## ups.conf（最終形）

```conf
[BW55T]
    driver = blazer_usb
    port = auto
    subdriver = ippon
    vendorid = 0590
    productid = 00d0
    default.battery.voltage.high = 13.6
    default.battery.voltage.low = 11.0
    desc = "Omron BW55T"
```

## 認識後のトラブル: 商用電源喪失の誤検知でサーバが落ちた

デバイスを認識できて一段落——と思ったところで実際にやられた。**商用電源（東電）は正常に来ているのに NUT が「電源喪失（OB / On Battery）」と誤検知し、`upsmon` が保護シャットダウンを実行してサーバが落ちた。**

### 何が起きたか

- 商用電源は生きているのに `ups.status` が `OB` と報告された
- `upsmon` は `OB` + `LB`（Low Battery）を検知すると FSD（Forced Shutdown）を発行する
- 結果、無停電のはずが保護シャットダウンで停止した

### 原因の見立て

generic Q1 / ippon の状態デコードが Omron の状態バイト配置と噛み合っていない可能性が高い。NUT の omron subdriver 移植メモにも「base Q1 マッピングは q1 準拠だが状態文字を 2 つ省く」「あるインデックスは商用運転中にセットされるが、どちらの OMRON ドライバ版も扱わない」とあり、Omron の状態フィールドは generic q1 と配置が異なることが示唆されている。つまり ippon が「商用電源あり」ビットを読み違え、OB を誤って立てうる。

加えて `battery.voltage` の high/low 閾値が実機とずれていると `battery.charge` が過小に算出され、LB が早期に立って FSD を後押しする。誤 OB と誤 LB が重なると、無停電でも一発でシャットダウンまで行ってしまう。

### 切り分け

```sh
# 商用電源が生きている状態で確認
# upsc BW55T ups.status
OL        # 期待値。ここが OB なら状態デコードがおかしい
```

`OB` が出るなら、デバッグ出力で Q1 応答の生の状態ビットと実状態を突き合わせると確実。

### 対処（正攻法から順に）

1. **デコードを正す = native omron を試す。** `nutdrv_qx` の `protocol=omron` は OMRON の状態／シャットダウン形式を使うので、ippon の誤読を根本回避できる可能性がある（後述。ただし BW55T は自動検出外なので明示指定）。
2. **ippon 継続なら閾値を正す。** `default.battery.voltage.high/low` を実測に合わせ、`runtimecal` で妥当な runtime を与えて `battery.charge` / LB の誤発火を抑える。
3. **upsmon 側で瞬間的な誤検知に耐性を持たせる。** `POLLFREQ` / `POLLFREQALERT` / `DEADTIME` を調整し、一発の OB で即 FSD にならないようにする。
4. **調査中の暫定退避**として、`MONITOR` のシャットダウン動作を外して「通知のみ」にする手もある。

:::message alert
2〜4 は誤検知を「握りつぶす」方向にも働く。閾値や upsmon 猶予を緩めすぎると、**本物の停電で落ちるべき時に落ちなくなる。** 最終的には 1（正しいデコード）で直すのが本筋で、2〜4 は原因究明までの一時措置と割り切る。
:::

### 検証（安全な実地確認）

設定変更後は、**非本番機**で UPS の入力プラグを意図的に抜き、`ups.status` が `OB` → 復電で `OL` に正しく追従するか、そしてタイマ通りにシャットダウンが走るかを確認する。誤検知が消え、かつ本物の停電では正しく落ちる——両方を確かめて初めて本番投入する。

## native omron ドライバについて

NUT 2.8.5 系では OMRON 純正 GPL ドライバ（Synology / QNAP 由来）を移植した omron subdriver / protocol が入った。ただし現状 USB ID として登録されているのは **BN150T (`0590:00b7`) のみ**で、検証も無負荷の BN150T だけ。BW55T (`0590:00d0`) は自動検出の対象外なので、試すなら `nutdrv_qx` で明示指定になる。

```sh
# /usr/local/libexec/nut/nutdrv_qx -a BW55T -DDDD \
    -x vendorid=0590 -x productid=00d0 -x protocol=omron
```

ここで用語の罠に注意。`nutdrv_qx` では `subdriver` が **USB-シリアル変換の方式**、`protocol` が **Qx 方言**を指す（ドキュメントでは方言側を subdriver と呼ぶ箇所があり紛らわしい）。認識できれば `battery.temperature` や `ups.firmware` まで拾えるので、監視スタックに載せるなら omron 方言の方が情報量は多い。ビルドが認識するトークンは `nutdrv_qx --help` で確認する。

## ハマりどころまとめ

- FreeBSD の USB 列挙は `lsusb` ではなく `usbconfig`。
- `No appropriate HID device found` は「HID が無い」ではなく「HID として掴まれていて libusb から開けない」。まず `dmesg | grep uhid` を見る。
- 対処は `hw.usb.quirk` + `UQ_HID_IGNORE`。反映には再起動（または揮発投入 + 再プローブ）。
- quirk（uhid 剥がし）と devd（権限付与）は別問題。両方必要。
- Omron VID `0590` は `blazer_usb` 内蔵テーブルに無いので `subdriver=ippon` + `vendorid`/`productid` の明示が要る。
- native omron subdriver は今のところ BN150T のみ登録。BW55T は明示指定で試す。
- generic Q1 / ippon は Omron の状態バイトを読み違え、**商用電源ありでも OB を誤検知しうる**。誤検知シャットダウンを踏んだら、upsmon を緩める前にまず状態デコード（omron protocol）を疑う。

:::message alert
シャットダウン試験は文字どおり給電を落とす。`upsmon` の FSD / `upsdrvctl shutdown` を叩くと接続機器ごと落ちるので、本番機では意図して実施すること。加えて本記事の「認識後のトラブル」のとおり、**設定を詰める前に本番投入すると誤検知で意図せず落ちる**。筆者はまさにこれを踏み、認識直後の追跡ログを失った。
:::
