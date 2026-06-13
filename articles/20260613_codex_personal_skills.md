---
title: Codex の personal skill を作り、配布可能な skill-export まで整備する
description: Codex の personal skill を SKILL.md 形式で整備し、agents/openai.yaml を追加し、さらに他プロジェクトや別マシンへ安全に配布するための skill-export を作る流れを公開用にまとめます
date: 2026-06-13T00:00:00.000Z
draft: false
tags:
  - Codex
  - OpenAI
  - AI
  - Zenn
categories:
  - Technology
---

# Codex の personal skill を作り、配布可能な skill-export まで整備する

## きっかけ

- Claude Code で handover という skillがあるのは知っていた。
- 最近のゴタゴタで claude-codeだけに頼るのは危険だと思った。
- codex に仕事を分けてみたが、GPT-5.5のコード品質は Opus4.7と遜色ないか、別の視点でreview出来る。
- codex でも skill を作らせた(handover.md を拝借して、porting）
- skillを作る方法が判った。
- skillを他の repo に移す、公開するだけのスキルは？（なかったので）　⇒　この記事を書かせるきっかけ。
- この記事の大半は codex との対話で生まれた。

## この記事でやること

Codex で使う personal skill を、単なる Markdown メモから実際に再利用できる skill に育てる手順をまとめます。

この記事では次を扱います。

- `SKILL.md` 形式の personal skill を作る
- `agents/openai.yaml` を追加して UI 向けメタデータを持たせる
- `handover` のような運用メモを skill 化する
- 作った skill を他プロジェクトや別マシンへ配布するための `skill-export` を作る
- 公開前に機微情報を除去し、公開 repo へ追加する前提を整理する

## 前提

Codex の skill は、単体の Markdown ファイルではなく、基本的に次のようなディレクトリ構成で管理すると扱いやすいです。

```text
~/.codex/skills/
  my-skill/
    SKILL.md
    agents/
      openai.yaml
```

最低限必要なのは `SKILL.md` です。UI 連携も意識するなら `agents/openai.yaml` も付けます。

## 1. Markdown メモを skill にする

最初にやったのは、既存の Markdown をそのまま読むのではなく、Codex が使いやすい skill 形式に変換することでした。

たとえば PR 作成手順をまとめたメモがあるなら、次のように再配置します。

```text
~/.codex/skills/pr/
  SKILL.md
```

`SKILL.md` には frontmatter を付けます。

```md
---
name: pr
description: Create and update GitHub pull requests from the current branch by analyzing git changes, generating a title and Markdown body, pushing the branch if needed, creating the PR with gh, and optionally processing review feedback.
---
```

重要なのは `description` です。これは単なる説明ではなく、Codex に「どういう依頼でこの skill を使うべきか」を伝えるトリガーになります。

## 2. handover メモも skill 化する

同じ要領で、セッション終わりの引き継ぎメモも skill にできます。

たとえば `handover` なら、内容はシンプルです。

- `prompt/handovers` を作る
- タイムスタンプ付きファイル名で保存する
- 今回やったこと、決定事項、捨てた選択肢、ハマりどころ、学び、次にやること、関連ファイルを書く

この種の手順は短くても skill 化する価値があります。毎回同じ粒度で出力させやすくなるからです。

## 3. UI 向けに agents/openai.yaml を追加する

skill 本体だけでも使えますが、UI で扱いやすくするには `agents/openai.yaml` を追加します。

たとえば `handover` の場合はこうなります。

```yaml
interface:
  display_name: "Handover"
  short_description: "Generate concise session handover notes"
  default_prompt: "Use $handover to generate a concise session handover note."
```

ポイントは 3 つです。

- `display_name`: UI 上の表示名
- `short_description`: 一覧用の短い説明
- `default_prompt`: skill 呼び出し時の既定プロンプト

`default_prompt` には `$skill-name` を明示しておくと分かりやすくなります。

## 4. 配布専用の skill-export を作る

skill が増えると、「この skill を他プロジェクトや別マシンでも使いたい」という話が出てきます。

そこで配布手順自体を `skill-export` という skill にまとめました。

`skill-export` の役割は次です。

- どのファイルを配ればよいか判断する
- global 配置か project-local 配置かを整理する
- Git 管理、アーカイブ配布、repo 追加のどれが適切か決める
- 公開前チェックを行う

最小構成の考え方は明快です。

```text
<skill-name>/
  SKILL.md
  agents/openai.yaml
```

さらに `assets/`、`references/`、`scripts/` がある skill なら、それもディレクトリごとまとめて持ち出します。

## 5. 標準的な配布方法

`skill-export` では、配布方法を次の優先度で整理しました。

### ディレクトリごとそのまま配る

最も壊れにくい方法です。skill はファイル単体ではなくディレクトリ単位で配るのが基本です。

### Git リポジトリとして管理する

複数マシンや複数ユーザーで継続的に更新するなら、skill ディレクトリを Git 管理するのが実用的です。

### アーカイブとして渡す

一時的な共有なら `tar` や `zip` でも十分です。ただし、継続運用には向きません。

## 6. 公開前に機微情報を除去する

配布を考え始めると、すぐに問題になるのが機微情報です。

`skill-export` には、機微情報の除去を標準工程として組み込みました。対象として想定したのは次のような情報です。

- API キー、トークン、認証情報
- 社内 URL、内部ホスト名
- 個人名、メールアドレス、チャットハンドル
- 顧客名、案件名、社内コードネーム
- 内部手順や公開不要な判断基準
- ローカル絶対パス

ここで重要なのは、無言で削るのではなく、次を必ず利用者に伝えることです。

- 何を除去したか
- どの程度一般化したか
- どこに残留リスクがあり得るか

たとえば削除だけで意味が崩れるなら、`<REDACTED>` のような置換を使います。

## 7. 公開 repo URL を引数で受ける

さらに `skill-export` には、公開先 repo を指定する運用も追加しました。

入力形式は次です。

```text
コマンド名 公開するスキル名 公開repo-URL
```

例:

```text
$skill-export handover https://github.com/example/codex-skills
```

この仕様にしておくと、skill 側で次を一貫して判断できます。

- repo URL が与えられているか
- public repo と private repo のどちらを想定するか
- 現在の利用者権限で追加できるか

ここでの方針は保守的です。

- repo 追加は利用者の権限内でのみ行う
- 権限不足や保護設定があるなら停止する
- 無理に回避しない
- 追加した場合は、追加先パスと除去内容を明示する

## 8. 実際に得られた知見

今回の作業で、Codex の skill 運用について次の知見が得られました。

- 単なる Markdown より `SKILL.md` の方が再利用しやすい
- `description` の質が skill の使われやすさを大きく左右する
- `agents/openai.yaml` があると UI 上での扱いが整理される
- 配布用 skill は、公開方法と機微情報除去まで含めて初めて実用になる
- 「運用メモ」も skill 化すると再現性が上がる

## 9. 公開用テンプレートとしてのまとめ

これから Codex 用の personal skill を作るなら、最初から次の 3 つをセットで考えると運用しやすいです。

1. `SKILL.md` で手順を定義する
2. `agents/openai.yaml` で UI メタデータを付ける
3. `skill-export` のような配布ルールを別 skill として持つ

skill を「その場限りのメモ」ではなく「他の Codex でも再利用できる作業単位」として扱うのが肝です。

## おわりに

今回の流れは、PR 作成、handover、skill 配布という一見別々の題材を扱っていますが、実際には同じテーマに収束しています。

それは、「手順を Codex に渡せる形に構造化する」ということです。

一度 `SKILL.md` と `openai.yaml` の型に乗せてしまえば、あとは改善も配布もやりやすくなります。最初の 1 つを丁寧に作る価値は大きいです。

## owener の結びの言葉

- 記事を書かせる形まで構造化してしまう可能性が出てきた（笑)
‐ この手の「自動でやらせちゃった」系の量産記事の一つになっているかも？
- strnh は リスクヘッジで claude-code/codexの二本立てですが、[FreeBSD](https://www.freebsd.org) に対応している codex で　作ったのも書いておく。
- サーバで codex IaC なんての ネタとして抱えてるので、そのうち出ると思います。

