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

## — Kernel 更新で Windows 11 VM が起動不能になった話 —

:::message
この記事は実際のトラブルシューティングをもとにした記録です。同じ罠を踏む方の参考になれば。
:::

## 環境

| 項目 | 内容 |
|------|------|
| ホスト OS | Ubuntu（Kernel 6.8 → 7.0 へ更新） |
| ハイパーバイザー | VMware Workstation Pro |
| ゲスト | Windows 11 x64（vTPM 有効） |
| サブ環境 | VirtualBox（共存、ただし 6.8 では競合あり） |

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

```
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
| `partial` | vTPM 周辺・設定ファイルのみ | **平文のまま** |

`partial` は **vTPM を追加した際に VMware が自動で設定するもの**で、ユーザーが明示的に「暗号化する」と操作した結果ではない。見かけ上は厳格な暗号化管理（AES-256、keySafe）が存在しながら、**ディスクの中身は保護されていない**という設計になっている。

---

## 根本原因：Kernel 更新による vTPM 封印値の変化

ホストの Kernel が 6.8 から 7.0 へ更新されたことで、`vmmon` / `vmnet` 等の VMware カーネルモジュールが差し替わった。

vTPM は起動チェーンの測定値（PCR: Platform Configuration Register）に鍵を封印する仕組みになっており、ホスト側のドライバ・ファームウェアが変わると**測定値がズレて自動リリースに失敗**する。これが今回の「パスワード要求」の正体だった。

```
Kernel 6.8 → 7.0 更新
    ↓
vmmon/vmnet ドライバ差し替え
    ↓
vTPM の PCR 測定値が変化
    ↓
keySafe が鍵を自動リリースできない
    ↓
「Enter Password」ダイアログ表示
```

加えて、VM を長期間起動していなかったため `Remember password` のキャッシュも残っていなかった。

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

### 2. パスワードは必ず記録する

`Remember password` にチェックを入れていれば普段は意識しないが、**Kernel 更新・ドライバ差し替え・長期未起動**などのタイミングで要求される。パスワードマネージャーへの登録は必須。

### 3. Kernel 更新後は早めに VM 起動を確認する

ドライバ差し替えのタイミングでキャッシュが有効なうちに一度起動しておけば、今回のような「久々に起動したらパスワードが分からない」事態を防げる。

### 4. partial encryption なら VirtualBox でのサルベージが効く

万が一同じ状況に陥ったら、vmdk を VirtualBox に直接インポートすることでデータを救出できる可能性がある。

---

## まとめ

| 項目 | 内容 |
|------|------|
| 直接の原因 | Kernel 更新による vTPM PCR 測定値の変化 |
| なぜパスワードが分からなかったか | 自動設定 + Remember password 頼み + 長期未起動 |
| なぜ救出できたか | `partial` encryption で vmdk が平文だったため |
| 教訓 | partial ≠ データ保護、パスワード管理と定期起動確認を |

VMware の vTPM 管理は見かけ上は厳格だが、`partial` encryption の実態は「データを守っていない」という点で設計の曖昧さが残る。Windows 11 の要件として vTPM が必須になった現在、同様のトラブルは増えると思われる。

---

*動作環境・バージョン情報は記事執筆時点のものです。*
