---
title: "Fujitsu PRIMERGY TX1310 M5でHDDのATAエラーを追跡し、iRMC・Redfish・BIOSで冷却を改善した話"
emoji: "🌡️"
type: "tech"
topics: ["freebsd", "zfs", "fujitsu", "server", "hardware"]
published: true
---

## はじめに

FreeBSDで運用しているPRIMERGY TX1310 M5で、ZFSミラーの再構築中に気になる現象に遭遇しました。

SMARTは正常にもかかわらず、

```
CAM status: ATA Status Error
ATA status: 51 (DRDY SERV ERR)
Retrying command
FLUSHCACHE48
```

が数秒おきに繰り返されていました。ディスク故障判断から解決を探った記録です。

## 発生した現象

syslogには以下のログが大量に記録されていました。

```
(ada1:ahcich3:0:0:0)
CAM status: ATA Status Error
ATA status: 51 (DRDY SERV ERR)
Retrying command
```

発生時刻を見ると `Jul 4 08:57` ごろから `Jul 4 12:51` ごろまで、ほぼ数秒周期で継続しました。
その日はちょうど停電作業にあたる土曜日で、出社している人がいないオフィスにある機材でした。

## SMARTは正常だった

まず `smartctl` を確認しました。　しかし・・・

```
Reallocated Sector Count      0
Current Pending Sector        0
Offline Uncorrectable         0
UDMA CRC Error Count          0
SMART Error Log               No Errors Logged
```

自己診断では異常なし。つまり媒体不良やケーブル不良を示す証拠は見つかりません。

## 温度を確認する

iRMCのセンサーを見ると以下の通り・・・

```
CPU      35℃
MEM      40℃
PCH      57℃
```

CPUは十分低温である一方、**PCHだけがかなり高いです。**

## HDD温度の推移

過去1年間のRRDを見ると、通常35℃前後、最大56℃　という記録です。

## 冷却を疑う

TX1310 M5は前面吸気・背面排気という構造になっている。CPUは十分冷えているのですが、PCHやSATAコントローラ周辺は風量不足ではないでしょうか？？

## Redfishを調べる

まずiRMCのRedfish APIを調査しました。

```
/redfish/v1/Chassis/0
```

を見ると `Thermal` や `Fans` などの情報が取得できます。さらに

```
/redfish/v1/Chassis/0/Thermal
```

では `FAN CPU`、`FAN SYS`、`FAN OEM` の回転数が確認できました。

## ファン制御APIを探す

Redfishをかなり調べたが、見つかったのは `Execute fan test` のみです。DellやSupermicroのような `Set Fan Speed` に相当するOEM APIは見当たりません。`ipmitool raw` も試しましたが、`Invalid command` となりました。

## BIOSを調査

BIOSを確認すると、`Server Mgmt` の中に `Fan Control` を発見。初期値は `Auto` でした。

## Fullに変更

試しに `Fan Control` を `Auto` から `Full` へ変更。起動するとすぐに**ファン音が一気に増大。**　静かなオフィスなので若干サーバ音が高く響きます。

## センサー値の変化

変更前:

```
CPU FAN  約1560 RPM
SYS FAN   約720 RPM
OEM FAN  約1030 RPM
```

変更後:

```
CPU FAN 5760 RPM
SYS FAN 3552 RPM
OEM FAN 5352 RPM
```

約4〜5倍まで回転数が上昇しました。

## 温度も大きく変化

変更後は以下の通りです。

```
Ambient 19℃
CPU     29℃
MEM     32〜34℃
PCH     34℃
```

特にPCHは57℃から34℃へ、約23℃低下しました。

## ATAエラーとの関係

今回の調査では、SMART正常・CRCエラーなし・セクタ異常なしである一方、PCH温度だけが高いという状況です。そのため、AHCIコントローラやPCH周辺の熱が影響していた可能性も十分考えられます。

もちろん、今回だけでは因果関係までは断定できません。今後、リシルバ完了後もしばらく監視を続けます。同様のATAエラーが再発するか確認したいと思います。


## 今回分かったこと

- TX1310 M5にはBIOSのFan Control設定がある
- AutoとFullを切り替えられる
- Redfishではファン状態は取得できる
- しかし回転数変更APIは見当たらなかった
- FullではSYSファンもCPUファンも大幅に高速化される
- PCH温度が20℃以上下がった
- SATAエラー対策として有効である可能性がある

## まとめ

今回の調査では、「HDD故障」と思われた現象を、SMART・iRMC・Redfish・BIOSを組み合わせて切り分けて行いました。結果として、**原因候補はHDDそのものではなく、PCHやSATAコントローラ周辺の熱環境にある可能性**が浮上しています。

今後は以下を進める予定です。

- Fan ControlをAutoへ戻した場合との比較
- PCH温度とATAエラーの相関
- Redfish OEM APIのさらなる解析

## 最後に

- HGST の 古い4TB から Seagate の最新 8TB (IronWolf Pro) に換えたのもあって、性能が向上しています。
- smartctl で S.M.A.R.T. 情報を確認するときにカクついてたのが改善しました。
- 最初からファン速度を full に 上げておけばよかったのですが、こうなると富士通製のサーバとしてはラックタイプに近い騒音発生量です。サーバルームであれば問題ないのですが・・・

