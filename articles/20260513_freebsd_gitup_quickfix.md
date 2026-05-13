---
title: FreeBSD\ gitup\ のトラブルシュート
description: FreeBSD-ports の更新に使う gitup の default設定で起きている障害回避策
tags:
  - GitHub
  - FreeBSD
  - ports
  - gitup
categories:
  - Technologie
private: false
updated_at: '2026-05-12T16:58:09+09:00'
id: f410c03d23058c6ae844
organization_url_name: null
slide: false
ignorePublish: false
---
## FreeBSD gitup のトラブルシュート

- gitup とはなんですか ⇒ [便利そうなGUI](https://github.com/git-up/gitup)ではありません。　⇒ [portsnap とか svnlite とかの代わりは gitup がいいらしいですよ](https://qiita.com/s_mitu/items/73e19035bec72bf59c71)
- portsnap はもう 終わりました (2026/04/30) [Recommended usage of git instead of portsnap](https://forums.freebsd.org/threads/recommended-usage-of-git-instead-of-portsnap.93822/)
- 手元の機材を 全部 gitup ports に変更(そういえば、cvsup から portsnap に入れ替えたのいつだか覚えてない大昔）
- 2026/05/08 頃から、 gitup ports で ports を拾っていたのがエラーになっています。
- 最近 pkg ではセキュリティアップデートが間に合ってない(python3.11は放置な）のでports でコンパイルして間に合わせています
- nginx-full いれると色々入りすぎるから ports がいいのだが 手元で ports更新が出来ていません
- portsnap が未だ生きているので、戻したところもあり。

## 何が起きてるのか

```console

# Scanning local repository...
# Host: git.freebsd.org
# Port: 443
# Repository Path: /ports.git
# Target Directory: /usr/ports
# Have: ae83ddf02de2c9adf8d2765a6dcb1b43d490a0af
# Want: 8e2222c5a1e9046687b6a2b7f631b9b94a25084b
# Branch: main
gitup: process_command: read failure:

HTTP/1.1 200 OK
Server: nginx/1.28.3
Date: Tue, 12 May 2026 07:28:28 GMT
Content-Type: application/x-git-upload-pack-result
Transfer-Encoding: chunked
Connection: keep-alive
Expires: Fri, 01 Jan 1980 00:00:00 GMT
Pragma: no-cache
Cache-Control: no-cache, max-age=0, must-revalidate
0011shallow-info
0001000dpackfile
2004PACK
: Invalid argument
```

- なにかが壊れた？
- "Invalid argument" これだけじゃ分からない。

## とりあえず足掻く

- /usr/ports 全消し ⇒　効果なし
- /var/db/gitup/ 以下全消し　⇒　効果なし

## 検索してみる。

 ×  "足掻く”　のと同じことを進められる (by ChatGPT)

## メーリングリストで

- git コマンドで github repo. で拾ったら問題なかった
- github repo から取ってくれば動くようになるんじゃね？

## 解決patch 

- FreeBSD-ports の mirror が他にあればそこにしてもいいのだが、取り敢えず githubにしといた。

```
-- gitup.conf.sample	2026-04-24 23:35:41.000000000 +0900
+++ gitup.conf	2026-05-12 15:39:31.815924000 +0900
@@ -17,7 +17,8 @@
 	},
 
 	"ports" : {
-		"repository_path"  : "/ports.git",
+                "host"             : "github.com",
+		"repository_path"  : "/freebsd/freebsd-ports.git",
 		"branch"           : "main",
 		"target_directory" : "/usr/ports",
 		"ignores"          : [],
```
## 暫定開通・結果

- 2026/05/12 現在、この方法で expat-2.8.0 (CVE-2026-41080) 対応した 2.8.1 を取得できている
- そのうち FreeBSD-ports 公式 repoが治ることを期待するか、このままで行くか

## 参考

- [Bug 295065 - Error running gitup ports](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=295065)
