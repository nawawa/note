Qiitaから自分の記事を取ってきて表示する技術ブログを作るシリーズです。
環境構築と初回デプロイを行なった記事は[こちら](https://qiita.com/inarikawa/items/1f7684ee918f8908f8c9)。

## 今回作るページ

- 記事を個別に表示するページ
- 記事一覧（ページネーションできるようにする）

### 要件

- パスは`ドメイン/articles/Qiitaで設定された記事ID`にする
- 自分のものではない記事のIDを入力されたら、404ページへリダイレクトさせる
- 記事は1ページにつき10件表示する

## 個別の記事ページ

### APIへリクエスト

`pages`配下に、`articles`ディレクトリと`_id.vue`というコンポーネントを追加。

その中で、Axiosを使ってAPIへリクエストを送るスクリプトを書く。

```vue
<template>
</template>

<script>
import axios from 'axios'

export default {
  async asyncData({ params, $config: { apiSecret, apiURL }, redirect }) {
    const {data} = await axios.get(
      `${apiURL}/items/${params.id}`,
      {
        headers: { Authorization: `Bearer ${apiSecret}` }
      }
    )
    return (data.user.id === 'inarikawa') ?
      { article: data }:
      redirect({ path: '/404'});
  }
}
</script>
```

環境変数から、APIのURL内の共通部分やアクセストークンを取り出して渡す。設定周りについては環境構築の記事で書いてるので割愛します。
`params`でこのページのURL（ドメイン/articles/記事ID）を受け取り、`params.id`で記事IDを取得。Qiitaから該当する記事を取得します。

`data`にAPIからの戻り値を受けると、[こんな感じの配列](https://qiita.com/api/v2/docs#%E6%8A%95%E7%A8%BF)が返ってくるので、まずはユーザーIDを確認。僕のIDは`inarikawa`なので、そうじゃなかったらコンポーネントの`data.article`へ戻り値をバインドせずに、404ページへリダイレクトさせています。[リダイレクト周りの公式ドキュメント](https://nuxtjs.org/ja/docs/internals-glossary/context/#redirect)

404ページはとりあえず空っぽのコンポーネントを`pages`に入れておいて、あとで好きなように作り込むつもりです。

![スクリーンショット 2022-03-31 12.48.47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/c75e2a20-1118-d9d6-afa8-d050885747c3.png)


### 記事の表示

タイトルと本文は別で値が入っているので、それぞれをテンプレート内に呼び出し。

```vue
<template>
  <div>
    <article>
      <CommonArticleHeader>
        <template #title>{{article.title}}</template>
      </CommonArticleHeader>
    </article>
    <CommonArticleAside />
  </div>
</template>
```

[こういうページ](https://devlog.atlas.jp/2017/10/30/1662)を参考にしながら最適なマークアップを行おうと思いましたので、記事は`article`要素の中に展開することにします。
その中でもタイトルや投稿日などの情報は`header`に書くものらしいので、`header`要素をラップしたコンポーネントを追加。自動インポートを使えるので、`components/Common/Article/Header.vue`をこの記述で呼び出せます。

```vue
<CommonArticleTitle>
  <slot name="title" />
</CommonArticleTitle>
```

`Header`コンポーネント内でこのように値を受け取り、さらに`Title`コンポーネントへパス。

```vue
<template>
  <v-container>
    <h1>
      <slot />
    </h1>
  </v-container>
</template>

<script>
export default {
}
</script>

<style lang="scss" scoped>
  h1 {
    font-size: 32px;
    font-weight: bold;
    line-height: 1.4;
    margin-top: 8px;
    word-break: break-all;
  }
</style>
```

スクリプトを書くコンポーネントとスタイルを書くコンポーネントを分けて管理したいので、こんな感じにしています。
`Title`コンポーネントでは、とりあえずQiitaと同じCSSを当てました。あとでいい感じにします。

あとは記事の本文ですが、マークダウンとレンダリング済みHTMLの両方で送ってもらえるので、手っ取り早くHTMLの方を`v-html`で展開することにします。
XSSに対する脆弱性が心配になりますが、少なくともQiita APIの投稿は十分に入力値検証なりエスケープなりされてると思うので、とりあえず信用して展開します。何より不特定多数のユーザーの入力値をそのまま受け取るわけではないですし。

```vue
<CommonArticleSection v-html='article.rendered_body' />
```

`renderd_body`というキーで格納されているので、それを`section`要素をラップしたコンポーネントへ渡します。
`Section.vue`はこんなとりあえずこんな感じです。

```vue
<template>
  <v-container>
    <section>
      <div v-bind="$attrs" />
    </section>
  </v-container>
</template>
```

これで、とりあえず記事を表示できるようになりました。

### スタイルを当てる

`v-html`で展開したHTMLに対してはディープセレクタを使うことでスタイルを当てられるようなのですが、やってみてもよくわからなかったので、スクリプトでごりごりDOM操作を行うことにしました。noteのフロントエンド改善の発表でも「document APIを使ってるところが云々かんぬん」という話をしていたので、似たようなことをやってるのかもしれません。

暫定的に、スタイルが当たるべき要素をクラス名ごとに取ってきて、スタイル属性に対してQiitaと同じCSSをぶちこむという力技的な方法を取りました。

```vue
<script>
export default {
  computed: {
    styling() {
      return (elements, styles) => {
          Array.prototype.forEach.call(elements, function(el) {
          el.style = styles;
        });
      };
    },
  },
  mounted() {
    /**
     * 
     * Qiitaからコピってきたスタイルを当てる
     * 
     */
    this.styling(
      document.getElementsByClassName('code-frame'), 
      `
        background-color: #364549;
        color: #e3e3e3;
        margin: 1.5em -32px;
        padding: 1em 32px;
        font-size: .9em;
        position: relative;
      `
    );

    // ▲ こんなのが延々続きます

}
</script>
```

やたらといっぱいあるのですが、ほとんどはコードブロック内に書かれるコードの色を指定するCSSです。あれ、色が違うところは全部`span`タグで分けて、`color`ごとにクラス属性を指定してたんですね。

とりあえず必要な部分のスタイルは全部この方法で当ててしまい、Qiitaで見た時と同じような見た目にしました。
スタイルは全て、あとから自分なりに考えます。完パクもどうかと思うし。

## 本番環境の設定を整える

ここまでの手順を踏んでも、`NuxtLink`がないとSSGにページを生成してもらえないので本番環境では記事を見ることができません。どこかから記事の個別ページへ内部リンクがはられている必要があります。

また、Netlifyの404を出さずに自前の404ページを表示するための設定も必要です。

```js
export default {
  generate: {
    fallback: true,
  },
```

`nuxt.config.js`に`generate`という項目を付けると、不明なリンク（SSGが生成したページ以外へのリンク）を叩かれてもNuxt内で解決し、生成済みの404ページへ飛ばしてくれます。（[参考記事](https://zenn.dev/shlia/articles/a6c2fb22ab7c6e)）

あとは、どこでもいいので`/articles/記事ID`へ通じる`nuxt-link`を設置します。

```vue
<NuxtLink to="/articles/1f7684ee918f8908f8c9">
  記事
</NuxtLink>
```

すると、ちゃんと`ドメイン/articles/1f7684ee918f8908f8c9`のページをレンダリングしてくれて、アクセスすれば記事を表示できるようになります。めちゃめちゃ楽です。
リンクを付けるという点では`a`タグも`NuxtLink`もできることは変わりませんが、アプリケーション内部の「自分で生成して欲しい」リンクは`NuxtLink`で通すというイメージです。

## 記事一覧ページ（ページネーション）

今回ブログを作るに当たっては、microCMSの公式ブログなども参考にはしていたのですが、microCMS + Nuxt で作る場合と比べ、Qiita API + Nuxt で作る場合は少し工夫が必要そうでした。(いまデプロイしているソースも大いに改善の余地があります)

とりあえず作ります。

### 設定を追加する

```js
router: {
  extendRoutes(routes, resolve) {
    routes.push({
      path: '/articles/page/:p',
      component: resolve(__dirname, 'pages/articles/index.vue'),
      name: 'page',
    })
  },
},
```

`/articles/page/ページ番号`にアクセスがあったら、`resolve`の第二引数に渡したコンポーネントを使ってね、という設定です。

### コンポーネントを作る

なんとなく[milieu](https://milieu.ink)のような構造にしたかったので、ページネーション時に使用するコンポーネントを作ります。

結果から書くとこんなテンプレートになっています。

```vue
<template>
  <v-container>
    <CommonArticleListContainer :articles="articles" />
    <div v-if="is_paginated">
      <NuxtLink v-for="number in length" :key="number" :to="`/articles/page/${number}`">
        {{number}}
      </NuxtLink>
    </div>
  </v-container>
</template>
```

1ページあたり10件表示したいのですが、仮に記事数が10件を下回る場合はページネーションボタンそのものを出すべきではありません。なので`v-if`で真偽値を受け取り、条件付きレンダリングを行うことにしました。

スクリプトはこんな感じです。

```vue
<script>
  computed: {
    article_total_count() {
      return this.user_data.items_count;
    },
    is_paginated() {
      return this.article_total_count > this.page_articles_count;
    },
    length() {
      return Math.ceil(((this.article_total_count) / this.page_articles_count));
    }
  },
  async asyncData({ params, $config: { apiSecret, apiURL } }) {
    const page_number = params.p || 1;
    const page_articles_count = 10;
    const token = { Authorization: `Bearer ${apiSecret}` };

    const response = await Promise.all([
      axios.get(`${apiURL}/users/inarikawa/items?page=${page_number}&per_page=${page_articles_count}`, { headers: token     	}),
      axios.get(`${apiURL}/users/inarikawa`, { headers: token }),
    ]);
    return { articles: response[0].data, user_data: response[1].data, page_articles_count: page_articles_count };
  },
</script>
```

`/articles/page/ページ番号`へアクセスがあると、`params`から取り出したページ番号を`page`パラメータへ渡し、Qiita APIへリクエストします。
記事だけではなくユーザー情報を取得するAPIも一緒に叩いていますが、これは1ページあたり10件表示する場合に何ページできるかを計算するため、投稿した記事の総数が欲しいからです。

QIita APIは、どうもmicroCMSのようなすべての記事を一気に取得するURIがないようで、一度に最大20件しか取ってこれません。
具体的には、URIのクエリで`page`（ページネーションした場合のページ番号）と`per_page`（一ページあたりの取得数）を渡さないといけません。このパラメータを省略すると、勝手に1ページ目が指定されて最初の20件を取得してきます。なので、事前に記事の総数を10で割ったら何ページできるかを計算するためには、ユーザー情報の`items_count`を取得する必要があるわけです。
複数APIを一緒に叩くので`Promise.all`を使ってます。

```js
const response = await Promise.all([
      axios.get(`${apiURL}/users/inarikawa/items?page=${page_number}&per_page=${page_articles_count}`, { headers: token     	}),
      axios.get(`${apiURL}/users/inarikawa`, { headers: token }),
    ]);
    return { articles: response[0].data, user_data: response[1].data, page_articles_count: page_articles_count };
```

そうやって記事情報を`articles`、ユーザー情報を`user_data`に入れ、一ページあたりの件数である`10`という数字も持っておきます。これはあとで定数化とかしたほうがいいかも。（読む人が任意で変えられるようにしたければコンポーネント内で持たせておけばいいけど）

```js
computed: {
    article_total_count() {
      return this.user_data.items_count;
    },
    is_paginated() {
      return this.article_total_count > this.page_articles_count;
    },
    length() {
      return Math.ceil(((this.article_total_count) / this.page_articles_count));
    }
  },
```

仮に記事数が0〜10件であれば、ページネーション自体できないのでボタンも必要ありません。なので算出プロパティ`is_paginated`で、`article_total_count`が返す記事の総数を`10`と比較して、ボタンを出すべきかどうか判定します。`length`が記事一覧ページの総数を計算し、3ページなら3つページネーションボタンを作り、それぞれに`/articles/page/ページ番号`のリンクを付けます。

ほんとはVuetifyの`v-pagination`コンポーネントを使ってサクッと実装できたらよかったのですけど、SSGでよしなにやってもらいたい場合は`NuxtLink`にする以外のやり方がよくわからなかったので、自分で作ってしまうことにしました。Vuetifyはほんとにブラックボックス的で、初めて使うコンポーネントは勝手が全くわからんです。

ちなみに記事の配列を受け取るコンポーネントは、とりあえずこんな感じになってます。

```vue
<template>
  <div>
    <div v-for="article in articles" :key="article.id">
      <NuxtLink :to="`/articles/${article.id}`">
        <CommonArticleListCard>
          {{ article.title }}
        </CommonArticleListCard>
      </NuxtLink>
    </div>
  </div>
</template>

<script>
export default{
  props: {
    articles: {
      type: Array
    },
  },
}
</script>
```

```vue
<template>
  <v-card>
    <v-card-title>
      <slot />
    </v-card-title>
  </v-card>
</template>

<script>
export default {
}
</script>
```

シンプルにぶん回して、`v-card`に渡したタイトルにリンクを付けています。
あとでQiitaやZennみたいな「記事カード」風のデザイン作り込もうと思っています。

ちなみにmilieuみたいな構造というのは、

- トップページに最新記事10件
- 記事一覧ページが1〜nページ

という感じで、つまり最新の記事10件がトップページと`articles/page/1`の両方に表示されるということです。

ここが最大の要改善ポイントなのですが、いまのところは全く同じコードを`index.vue`と`/articles/index.vue`の両方に書いています。
トップページと記事一覧ページのマークアップを明確に分けたかったためにこういう構造にしたわけですが、Nuxtの`middleware`機能や`Vuex`を使うような形で一箇所に統一する方法を後で模索する予定です。

## 完成&mainへマージしてデプロイ

というわけで、個別の記事ページと記事一覧ページ（ページネーション対応）の基本的な実装がひとまず完了しました。
本番環境でもご覧の通り。

![スクリーンショット 2022-03-31 13.08.45.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/ae9fbb60-b7f8-08af-c6df-307b14d1b1be.png)
![スクリーンショット 2022-03-31 13.09.01.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/87042958-922e-3e91-71a0-22aa40b12f2e.png)
![スクリーンショット 2022-03-31 13.09.15.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/651036/b3715e4b-8010-8b18-96e8-d53a463fa816.png)

25記事しかないので3ページだけ。

ここが重要ですが、記事一覧ページのコンポーネント内で値を持っているので、リロードしても同じページを表示することができています。最初に値を取ってきてVuexに入れる方法だと、リロードしたらそれらが消えてしまったりするので、このあとからはその課題に向き合う予定。


でもいったんスタイルをゴリゴリ書いていく方に浮気したいと思います。