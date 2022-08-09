---
title: 'microCMS JavaScript SDK でGET APIのレートリミットによる 429(Too Many Requests) を回避する'
emoji: '🈵'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [microcms, jaavscript, typescript, headlessCMS, api]
published: false
---

# 制限事項

https://document.microcms.io/manual/limitations#h1eb9467502

> GET APIの呼び出し回数：1秒あたり60回

> 制限事項を超えた場合は、ステータスコード429(Too Many Requests)が返却されます。

# 秒間リクエスト数を制限する

一応下記で回避はできるので、SSGのリクエスト詰まりにどうぞ。

@[gist](https://gist.github.com/the-fukui/2fbcf3476d3ab7073da08ea0c29be828)

:::message
エッジキャッシュ済みのAPIはレートリミット対象ではないので、上記の単純な秒間リクエスト数制限では非効率的...だが一応429は回避できる。
:::
