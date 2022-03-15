## 環境構築
https://zenn.dev/szn/articles/nuxt-2-with-docker-compose

こちらの記事を見ながら構築しましたが、`digital envelope routines::unsupported`というwebpack由来らしいエラーが発生し、コンテナのビルドに失敗。
`config`に設定を差し込んでどうにかしようとするもうまくいかず、これ以上は今の自分の努力ではどうにもならんわと感じたため、雑に`node`のバージョンを現時点で最新の17系から16系へ下げてビルドし直す形で対処しました。

```Dockerfile
#修正
FROM node:16-alpine 

RUN apk update && \
    yarn global add create-nuxt-app

ENV HOST 0.0.0.0
EXPOSE 3001
```

言語はJavaScript、テストはJest、UIはVuetify、モードはユニバーサルに設定。今回はSSGで静的ファイル生成を行いたいので、デプロイターゲットは`Static`です。

「なんか見様見真似で構築してしまったけど、どんな設定かようわからんわ」という場合は、とりあえず`nuxt.config.js`に`target:'static',`の一行があればNetlifyへのデプロイはできます。

`yarn dev`でこんな画面が出たのでとりあえず構築完了です。

![スクリーンショット 2022-03-13 20.24.51.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/0d73c8a6-a844-0736-1bae-3a5d45ccbf95.png)

## いったんデプロイしてみる

この状態で一度デプロイしてみます。
本番環境には、ホスティングプラットフォームのNetlifyを使います。

https://netlify.com/

https://docs.netlify.com/configure-builds/common-configurations/nuxt/

Githubのmainブランチへプッシュがあると、自動的に再ビルド・デプロイしてくれるすんばらしいサービスです。
初回デプロイは正直案内手順どおりにやる、以外に書くことがないので、公開ディレクトリの設定だけ少しメモしておきます。

```
Base directory: src
Build command: nuxt generate
Publish directory: src/dist/
```

Nuxtアプリケーションだけプッシュしているのであれば`dist`を公開ディレクトリに設定するだけでいいのですが、今回はDockerでローカル環境を構築しているので、Nuxtのソース自体は`main`ブランチの`src`ディレクトリ以下に配置されています。なので`src`をベースに、それ以下に生成される`dist`を公開ディレクトリに指定する必要があります。

初期ページがなかなか表示できずに詰まりかけたのですが、`src/dist/`と、末尾にスラッシュを付ける必要があっただけでした。

![スクリーンショット 2022-03-13 21.11.12.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/b4611f59-8215-9781-4f2d-e5b7e1df11dc.png)

無事に同じページが表示され、`.netlify.app`ドメインにてデプロイが完了しました。

## 環境変数の設定

今回はQiita APIから自分の書いた記事を取得してくるサイトを作ろうと思っているので、取得したアクセストークンを環境ごとに管理します。

`src`以下に`.env`を追加し、環境を表す`NODE_ENV='development'`とAPIアクセストークンを格納する`API_SECRET=アクセスキー`などの変数を格納。デフォルトで`.gitignore`に入っているのでプッシュされません。

あとは`nuxt.config.js`にて設定を記述します。

```js
export default {

  privateRuntimeConfig: {
    apiURL: process.env.API_URL,
    apiSecret: process.env.API_SECRET
  },
  publicRuntimeConfig: {
    apiURL: process.env.NODE_ENV !== 'production' ? process.env.API_URL : '',
    apiSecret: process.env.NODE_ENV !== 'production' ? process.env.API_SECRET : ''
  },
・
・
}
```

これら2つの便利なプロパティのおかげで、特に`.env`を`require`することなどもなく、安全に環境変数をフロントエンドから隠蔽できるようになったみたいです。うれしい。

本番環境の環境変数も、`.env`に対応する形で定義します。

![スクリーンショット 2022-03-15 14.57.10.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/f16b81cf-7595-9725-12f1-c24bdad93f4e.png)

こういうのもパッと見でわかりやすくていいですね。

もし本番環境とステージング環境を分ける場合は、Netlifyでもう一度サイトを作り直して、デプロイ対象ブランチを`main`ではなく開発ブランチに設定すれば実現可能です。有料プランに入ればBasic認証もかけられますし、`microCMS`のプレビュー用エンドポイントとかそういう感じのやつと組み合わせれば、JAMstacでも十分な制作体制を無料で整えられますね。すばらしい……WordPressとレンタルサーバーなんていらんかったんや……

よっしゃ、あとはゴリゴリ実装するだけ💪

---

参考リンク：

https://zenn.dev/szn/articles/nuxt-2-with-docker-compose

https://qiita.com/cnloni/items/1c83cac956599fb24158

https://cly7796.net/blog/javascript/create-a-jamstack-site-with-nuxt-js-and-microcms/