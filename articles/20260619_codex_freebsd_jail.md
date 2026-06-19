---
title: "FreeBSD jail で Codex CLI の device-auth が失敗する問題と暫定回避策"
emoji: "🛠️"
type: "tech"
topics:
  - freebsd
  - jail
  - codex
  - nodejs
  - openai
published: false
---

## 何をしていたか

FreeBSD/codex 環境 をいくつか管理している jail 展開時、device-auth ブロックを経験した。　一旦諦めかけていたものの、状況切り分けと暫定解決を報告する。

## 概要

FreeBSD jail 上で Codex CLI を利用していたところ、`device-auth` による認証が失敗する問題に遭遇した。

調査の結果、IPv6 到達性の有無と症状に相関が見られた。また、Node.js の DNS 解決順序を変更することで回避できたため、その内容を記録しておく。

現時点では根本原因は未確定であるため、本記事は **調査メモ兼暫定運用記録** である。

## 発生した症状

Codex CLI の認証時に以下のようなエラーが発生した。

```text
Error logging in with device code:
error sending request for url
https://auth.openai.com/api/accounts/deviceauth/usercode
```

また、MCP 利用時には以下のようなエラーも発生した。

```text
MCP startup failed:
handshaking with MCP server failed

HTTP request failed:
https://chatgpt.com/backend-api/ps/mcp
```

## 環境

- FreeBSD
- jail 環境
- Codex CLI
- IPv6 非到達

## 切り分け

まず OpenAI 認証サーバへの疎通を確認した。

```sh
curl -4 -v https://auth.openai.com/
```

これは正常に成功した。

一方で Codex CLI の `device-auth` は失敗する。

そのため、

- DNS 解決
- Node.js の通信処理
- IPv4 / IPv6 の優先順位

のいずれかが関係していると考えた。

## 比較結果

| 環境 | device-auth |
|--------|--------|
| IPv6 到達可能 jail | 成功 |
| IPv6 非到達 jail | 失敗 |
| IPv6 非到達 jail + IPv4 優先設定 | 成功 |

IPv6 が利用可能な jail では認証に成功した。

一方、IPv6 が利用できない jail では失敗した。

## 暫定回避策

Node.js の DNS 解決順序を IPv4 優先へ変更する。

```sh
export NODE_OPTIONS="--dns-result-order=ipv4first"
```

その後に認証を実行する。

```sh
codex login --device-auth
```

この設定により、IPv6 が利用できない jail 環境でも `device-auth` が正常に成功した。

## 恒久設定例

ユーザー単位で設定する場合。

```sh
# ~/.profile

export NODE_OPTIONS="--dns-result-order=ipv4first"
```

あるいは shell 設定ファイル。

```sh
# ~/.shrc

setenv NODE_OPTIONS "--dns-result-order=ipv4first"
```

## 考察

現時点では、

> Codex CLI が IPv6 を必須としている

可能性は低く

> Node.js の DNS 解決結果として AAAA レコードが優先され、到達不能な IPv6 経路へ接続しようとして失敗している

という、この手の問題にありがちなパターンに落ちたと見られる。

今回の結果から分かるのは、

- IPv6 到達可能環境では成功する
- IPv6 非到達環境では失敗する
- IPv4 優先設定で成功する

という事実までである。

## 今後の調査項目

以下については未調査である。

- `tcpdump` による接続先確認
- MCP 初期化失敗との関連
- Node.js バージョン差異の影響
- jail ネットワーク設定との関連

## まとめ

FreeBSD jail 上で Codex CLI の `device-auth` が失敗する場合、

```sh
export NODE_OPTIONS="--dns-result-order=ipv4first"
```

を設定することで回避できる可能性がある。
少なくとも筆者の環境では、この設定によって IPv6 非到達 jail 上でも正常に認証できるようになった。
暫定運用としては十分実用的な回避策と考えている。

## IPv6 に対する　余談

ちなみに "2026/06 現在"でも github.com の AAAA はなくIPv6 接続性が必須でない豊かな世界と、
IPv6普及のせめぎ合いに巻き込まれている感はある。
