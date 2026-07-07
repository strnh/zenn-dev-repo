---
title: "VMware Workstation Pro の vTPM partial encryption の罠"
emoji: "🛟"
type: "tech"
topics:
  - vTPM
  - VMWare 
  - ubuntu26.04
  - Windows11
  - virtualbox
published: false
---


:::message
** Windows 11 VM が起動不能になった話と、間違えた原因推定 **
この記事は実際のトラブルシューティングをもとにした記録です。同じ罠を踏む方の参考になれば。
:::

## 環境

| 項目 | 内容 |
|------|------|
| ホスト OS | Ubuntu 26.04 LTS（Resolute Raccoon）、ホスト名 `ryzen5700-pc` |
| ハイパーバイザー | VMware Workstation Pro 25.0.0 |
| ゲスト | Windows 11 x64（vTPM 有効） |
| サブ環境 | VirtualBox（共存、Kernel 7.0 では競合なし） |

---

## 発端

長期間（数ヶ月）起動していなかった Windows 11 の VM を久々に起動したところ、VMware のダイアログが表示され先に進めなくなった。

```
Enter Password
🔒 This virtual machine is encrypted.
   You must enter its password to continue.

Password: [          ]
□ Remember password
```

Windows のログイン画面より手前、**VMware のレイヤーで止まっている**。

「暗号化した記憶がない」——これが最初の違和感だった。

---

## 切り分け：何が起きているのか

### BitLocker ではない

BitLocker であれば Windows の起動プロセスに入ってから青い回復画面が出る。今回は **VMware のダイアログ**で止まっているため、ゲスト OS より手前の問題と判断。

### `.vmx` を確認

```bash
grep -i "encrypt\|keySafe\|keystore" "Windows 11 x64.vmx"
```

結果：

```ini
encryption.type = "partial"
cryptedVM.guid = "3791989229576228952"
encryption.keySafe = "vmware:key/list/(pair/(phrase/20pAjIc6MGM...
    cipher%3dXTS%2dAES%2d256%3arounds%3d10000%3asalt%3d..."
encryption.data = "1HGPysT6WWNrD5kK3nirWOzFswa1cH4JQ2fZI/..."
```

`encryption.keySafe` が存在し、**AES-256・10000ラウンドのパスワードベース鍵導出**が設定されていた。

しかし `encryption.type = "partial"` ——この `partial` という値が今回のキーワードになる。

---

## partial encryption とは何か

VMware における暗号化には 2 種類ある。

| タイプ | 対象 | vmdk の中身 |
|--------|------|-------------|
| `full` | VM 全体（vmdk 含む） | 暗号化される |
| `partial` | vTPM 周辺・設定ファイル・nvram のみ | **平文のまま** |

`partial` は **vTPM を追加した際に VMware が自動で設定するもの**で、ユーザーが明示的に「暗号化する」と操作した結果ではない。見かけ上は厳格な暗号化管理（AES-256、keySafe）が存在しながら、**ディスクの中身は保護されていない**という設計になっている。

---

<<<<<<< Updated upstream
## 当初の仮説（後に棄却）
=======
## 根本原因：Kernel 更新による vTPM 封印値の変化??
>>>>>>> Stashed changes

最初に立てた仮説は「**Kernel 更新による PCR 測定値の変化**」だった。

vTPM は起動チェーンの測定値（PCR: Platform Configuration Register）に鍵を封印する仕組みがある。ホスト Kernel が 6.8 から 7.0 へ更新されたことで `vmmon`/`vmnet` 等の VMware カーネルモジュールが差し替わり、その測定値のズレが自動リリース失敗につながった——という筋書きだ。

```
<<<<<<< Updated upstream
Kernel 更新 → VMware モジュール再ビルド → PCR 値変化
    ↓
keySafe の自動リリース失敗
    ↓
「Enter Password」ダイアログ
=======

VMwareがホストのキーリング等に保存していた自動アンロック（Auto-Unlock）用のキャッシュにアクセスできなくなった、またはキャッシュが消失したため

    ↓
keySafe が鍵を自動リリースできない
    ↓
「Enter Password」ダイアログ表示
>>>>>>> Stashed changes
```

ただし調査を進めると、この説には矛盾が生じた。

---

## ログと実測値による原因調査

### vmware.log が示した一次証拠

`~/vmware/Windows 11 x64/vmware.log` を検索すると、決定的な行が見つかった。

```
2026-05-10T13:43:56.785Z Host is Linux 7.0.0-14-generic Ubuntu 26.04 LTS
...
2026-05-10T13:44:01.256Z DecryptWithPadding: Failed because padding is too short or too long
```

**ポイントは「ユーザーがパスワードを入力するより前」にこのエラーが出ていること**。パスワードダイアログを表示する前に、VMware が自動アンロックを試みて失敗している。その後、ユーザーがパスワードを入力した形跡も残っていた（14:07 に再度同エラー）——つまりパスワードを間違えた。

`DecryptWithPadding: Failed` は PKCS パディングエラーで、「復号に使った鍵が間違っている」ことを意味する。正しい鍵であれば必ずパディングは正常になる。

### Kernel 更新単独説は棄却

| 根拠 | 内容 |
|------|------|
| 障害時の Kernel | `7.0.0-14`。すでに 7.0 系を複数回更新済みで他の VM は正常動作 |
| 過去の更新歴 | Ubuntu 25.10 → 26.04 のアップグレードを含む複数回の更新を経ても未発症 |
| vmmon ビルド日 | ビルド日と障害日に相関なし |

Kernel 更新のたびに再現するなら、過去の更新時にも問題が出ていたはず。→ **Kernel 更新単独説は棄却。**

### keySafe の仕組みと TPM の関係

`.vmx` の `encryption.keySafe` を見ると：

```ini
vmware:key/list/(pair/(phrase/...cipher%3dXTS-AES-256%3arounds%3d10000%3asalt%3d...
```

これは **パスフレーズベースの鍵導出**（PBKDF）方式であり、TPM の PCR 値に鍵を「封印」（sealing）する方式ではない。

実際にホストの TPM デバイスを確認すると物理 TPM2 は存在する：

```bash
$ ls /dev/tpm*
/dev/tpm0   /dev/tpmrm0
```

しかし VMware Workstation の `partial` 暗号化は、Enterprise 向けの vSphere Key Provider や KMS とは異なり、**ホスト TPM を keySafe の保護に使用しない**。PCR 値が変わっても keySafe の復号には影響しない。

→ **「PCR 値変化→自動アンロック失敗」という当初の機序説明は誤りだった。**

### fwupd による UEFI 更新との時系列

```bash
$ fwupdmgr get-history
```

| 更新内容 | 日付 |
|---------|------|
| Secure Boot KEK CA 更新（ver.2011 → 2023） | **2026-02-09** |
| UEFI dbx 更新（20241101 → 20250902） | 2026-06-03 |

dbx 更新は障害（2026-05-10）より後なので除外。**KEK 更新（2026-02-09）だけが障害より前の新変数**。

ただし VMware Workstation の passphrase-based keySafe は PCR 封印を使わないため、KEK 更新が DecryptWithPadding を直接引き起こす機序は確認できなかった。「KEK 更新によるリブートがトリガーになって別の何かが起きた」という間接経路は否定できないが、断定できる証拠はない。

### 資格情報ストアの確認

```bash
$ secret-tool search --all 2>/dev/null | grep -iA3 vmware
（出力なし）
```

GNOME Keyring に VMware の認証情報は存在しない。`~/.vmware/` 内にも credential ファイルは見当たらなかった。

**VMware が自動アンロックに使えるものが何もない状態**だった。

### 時系列の整理

| 日付 | イベント |
|------|---------|
| 2026-02-09 | fwupdmgr：Secure Boot KEK CA 更新（要リブート） |
| ~2026年4月 | Ubuntu 25.10 → 26.04 ディストロアップグレード |
| 2026-05-04 | Kernel 7.0.0-14 インストール |
| **2026-05-10** | Win11 VM 起動試行（kernel=7.0.0-14）→ 失敗 |
| 　13:44:01 | `DecryptWithPadding: Failed`（自動アンロック失敗） |
| 　14:07:14 | `DecryptWithPadding: Failed`（パスワード入力→誤り） |

---

## 根本原因：自動アンロックの喪失 ＋ パスワード失念

調査から言えることをまとめると：

1. VMware は起動時に GNOME Keyring 経由でパスワードの自動アンロックを試みる
2. そのエントリが存在しないため `DecryptWithPadding` が発生し、パスワードダイアログが表示された
3. ユーザーはパスワードを覚えておらず、入力しても失敗した

**なぜキャッシュが消えたか**については断定できなかった。最も有力な経路は「Ubuntu 25.10 → 26.04 のディストロアップグレード時に GNOME Keyring が再初期化またはマイグレーションされ、VMware のエントリが失われた」というもの。あるいは「そもそも Remember password が未設定で、昔は手動入力していたが長期未起動で忘れた」だけという可能性も排除できない。

KEK 更新（2026-02-09）は**時系列的に最も近い新変数**ではあるが、keySafe が passphrase-based である以上、直接の因果関係を実証するには至らなかった。

:::message
当初「Kernel 更新 → PCR 変化 → 自動リリース失敗」と説明したが、これは VMware の keySafe 実装を誤解した説明だった。VMware Workstation の `partial` 暗号化は TPM PCR sealing を使わない。
:::

---

## サルベージ：partial だから助かった

パスワードが不明なまま VMware での復旧を断念。ダメ元で **VirtualBox に vmdk を直接インポート**してみたところ——あっさり起動した。

`partial` 暗号化のため **vmdk 自体は平文**であり、別のハイパーバイザーからは制約なく読み込めたのだ。

:::message alert
逆に言えば、`partial` encryption は「データを守る暗号化」ではない。  
vTPM のための鍵管理が目的であり、vmdk の保護にはなっていない。
:::

---

## VirtualBox と VMware の共存問題（余談）

Kernel 6.8 の時点では VirtualBox と VMware Workstation を同一ホストで共存させることができなかった（カーネルモジュールが競合）。Kernel 7.0 では干渉が解消されており、今回のサルベージ作業で両者を同時に使うことができた。

USB 周りは未検証のため、完全な共存確認ではないが、以前より状況は改善している模様。

---

## 教訓

### 1. vTPM を使うなら「partial か full か」を意識する

VMware で vTPM を追加すると自動的に `partial` 暗号化が設定される。これはデータ保護ではなく vTPM の鍵管理のための仕組み。データを守りたいなら明示的に `full` encryption を選ぶ必要がある。

### 2. パスワードは必ずパスワードマネージャに記録する

GNOME Keyring の「Remember password」は**ディストロアップグレードや Keyring 再初期化で失われることがある**。OS 側の認証ストアに依存せず、Bitwarden や KeePass 等で明示的に管理すること。

### 3. UEFI 更新・ディストロアップグレード後は早めに VM 起動を確認する

暗号化 VM は「動く状態のうちに」起動確認しておくことで、パスワードが正しいか・自動アンロックが機能するかを早期に検証できる。長期未起動との組み合わせが被害を拡大する。

### 4. partial encryption なら VirtualBox でのサルベージが効く

万が一同じ状況に陥ったら、vmdk を VirtualBox に直接インポートすることでデータを救出できる可能性がある。

### 5. ログを先に見る

`vmware.log` の `DecryptWithPadding` エラーはパスワード入力前から記録されていた。「何が起きているか」を推測より先に**ログで確認**することで、的外れな仮説（PCR 変化説）に時間を使わずに済んだかもしれない。

---

## まとめ

| 項目 | 内容 |
|------|------|
| 当初の誤った仮説 | Kernel 更新 → PCR 変化 → vTPM 自動リリース失敗 |
| 実際の直接原因 | GNOME Keyring に VMware 認証情報なし → 自動アンロック不可 ＋ パスワード失念 |
| 最有力トリガー | ディストロアップグレード（25.10→26.04）による Keyring 消失、または長期未起動による記憶喪失 |
| なぜパスワードが分からなかったか | 自動設定 + Remember password 頼み + 長期未起動 |
| なぜ救出できたか | `partial` encryption で vmdk が平文だったため |
| 教訓 | partial ≠ データ保護、パスワードは OS 外で管理、定期起動確認を |

VMware の vTPM 管理は見かけ上は厳格だが、`partial` encryption の実態は「データを守っていない」という点で設計の曖昧さが残る。Windows 11 の要件として vTPM が必須になった現在、同様のトラブルは増えると思われる。

---

*動作環境・バージョン情報は記事執筆時点のものです。*
