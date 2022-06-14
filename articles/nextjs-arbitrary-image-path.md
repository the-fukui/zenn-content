---
title: 'next/imageのimages.domainsに共用ドメインを使用すると危ないかも'
emoji: '⚠'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: [nextjs]
published: false
---

next/imageは画像を最適化してくれるNext.jsビルトインの機能です。  
サイト内外を問わず圧縮リサイズなどがお手軽に出来て便利ですが、注意が必要なパターンもあります。

# TL;DR

- 外部画像を最適化する場合、最適化を許可する画像URLのドメインを`next.config.js`の`images.domains`に指定する
- クラウド系ストレージ(Storage as a Service)等を利用していて、**サブディレクトリ**でテナントを切り分ける方式の場合、**任意のテナントの画像を表示・圧縮し放題**
  - 例：Google Cloud Storage（storage.googleapis.com）
- もちろんCPUリソースは自分のところのものが使われる
- しかも自分のサイトのドメインのコンテンツとして発信される
- 対策はストレージに独自ドメイン割り当てるか画像URLをプロキシするかぐらい?

# next/image?

next/imageは画像の各種最適化を行ってくれるコンポーネント、API、Functionを提供してくれるNext.jsビルトインの機能です。  
デフォルトではNext.jsサーバーで最適化を行いますが、設定によってimgixなどの画像処理系サービスの利用も可能です。

https://nextjs.org/docs/basic-features/image-optimization
https://nextjs.org/docs/api-reference/next/image
https://zenn.dev/catnose99/articles/883f7dbbe21632a5254e

# 前提：外部画像の利用

サンプルを用意しました。  
https://nextjs-arbitrary-image-path.vercel.app/

かわいいワンちゃんですね。

create-next-appしたNext.jsプロジェクトに、Google Cloud Storageに置いた自分の画像を表示しています。  
元画像: https://storage.googleapis.com/nextjs-arbitrary-image-path/wanchan.jpg

外部URLの画像をnext/imageで使う場合、画像URLのドメインをホワイトリスト登録する必要があるので、下記のようにnext.config.jsを作成しました。

```js:next.config.js
module.exports = {
  images: {
    domains: ['storage.googleapis.com'],
  },
}
```

```jsx:pages/index.js
<Image src="https://storage.googleapis.com/nextjs-arbitrary-image-path/wanchan.jpg" alt="ワンちゃんが海辺を走っている画像" width={640} height={427} />
```

# 問題：第三者の画像を最適化できる

ここで、サンプルで表示されている画像（next/imageで最適化している画像）のURLは下記です。  
https://nextjs-arbitrary-image-path.vercel.app/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fnextjs-arbitrary-image-path%2Fwanchan.jpg&w=1920&q=75

末尾に`url`というクエリで画像のパスが指定されています。  
それでは、**別のバケット**においた下記画像を`url`に指定してみます。  
　https://storage.googleapis.com/nextjs-arbitrary-image-path-2/nekochan.jpg

https://nextjs-arbitrary-image-path.vercel.app/_next/image?url=https://storage.googleapis.com/nextjs-arbitrary-image-path-2/nekochan.jpg&w=1920&q=75

なんと、猫ちゃんが表示されました。悪そうですね。

このように、第三者が自由な画像を表示させる事ができてしまいます。  
しかも表示されているのは自分のサイトのドメインで、圧縮などの最適化に使われるCPUリソースも自分持ちです。

今回は猫ちゃんだからよかったものの、💀や🔞な画像だったら大変なことになるケースもありますね。

## 考えうる悪用ケース

- 信頼あるドメインの場合、フィッシング詐欺の入り口として
- 画像最適化サーバーとしてリソースのタダ乗り

# 原因:サブディレクトリ方式によるテナントわけ

本来表示していた画像の元のURLは https://storage.googleapis.com/nextjs-arbitrary-image-path/wanchan.jpg でした。  
Google Cloud Storageの場合、https://storage.googleapis.com/{バケットID}/{パス・ファイル名} というURL構成になっており、**サブディレクトリ**でバケットを切り分けています。  
Next.js側でホワイトリストとして登録できる情報は今のところドメインのみなので、`storage.googleapis.com` を指定するしかありません。  
つまりサブディレクトリ以下は、**自他に関わらず、全てのGoogle Cloud Storageバケットが許可されている**ことになります。

# 解決策

先述の通り、next/imageのホワイトリストはドメイン単位なので、画像側のURLをなんとかする必要があります。

## 独自ドメインを割り当てる

Google Cloud Storageの場合、ロードバランサーのバックエンドバケットとして指定することで独自ドメインの割当が可能です。

https://cloud.google.com/storage/docs/hosting-static-website?hl=ja

<!-- textlint-disable ja-technical-writing/ja-no-weak-phrase -->

（ただし、このためだけにロードバランサーまで用意するのはちょっと大げさかもしれません）。

<!-- textlint-enable ja-technical-writing/ja-no-weak-phrase -->

## 画像URLをプロキシする

Next.jsのrewrite機能を使い、外部画像のURLをプロキシすることも有効です。  
テナントを分けているサブディレクトリまで含めたパスを`destination`と指定することで、相対URLのまま外部画像の表示が可能です。  
（相対URLなので、next.config.jsのimages.domainsに登録は必要ありません）。
お手軽ですが、CMS等外部APIから返ってきた画像パスがフルパスの場合、別途相対パスに書き換える処理が必要になります。

```js:next.config.js
module.exports = {
  async rewrites() {
    return [
      {
        source: '/proxy/:path*',
        destination:
          'https://storage.googleapis.com/nextjs-arbitrary-image-path/:path*',
      },
    ]
  },
}
```

```js:pages/index.js
<Image src="/proxy/wanchan.jpg" alt="ワンちゃんが海辺を走っている画像" width={640} height={427} />
```
