Qiitaに投稿した自分の記事を取得して表示する技術ブログを、Nuxt.jsで作ろうよシリーズです。

環境構築編：https://qiita.com/inarikawa/items/1f7684ee918f8908f8c9
Qiita APIとのつなぎ込み編：https://qiita.com/inarikawa/items/c0266529ada6a4282a0c

前回の補足的な内容から始めたいと思います。
あの状態では、Qiitaに記事を投稿しても、その内容をすぐさまNuxtアプリケーションに反映させることはできません。

おそらく「Qiitaに投稿する」ことをトリガーにすることはできないのですけど、僕はローカルで書いたマークダウン記事をコピペしてQiitaに投稿しており、原文についてはQiitaのサーバーとは別にGitHub上で管理しています（『勉強』で草が生えてもええやないか…と思ったのと、いずれZennにリモートリポジトリから投稿したり、マークダウンからレンダリングする形のJAMstacもやってみたいので）。

なので僕の場合は、GitHubへ記事がプッシュされたことをトリガーとするWebhookを設定することで、Netlify側で自動的に再ビルドを行なってもらうことができます。その設定方法についてメモします（超簡単）。

### NetlifyのBuild hooksを作成する

`settings`の`deploys`にある`Buile hooks`の項目へ移動し、`add build hooks`で新規作成。
するとビルド対象のブランチがデフォルトで入っていると思いますので、今回は`main`しかないためこのままでOK。

新規作成されたことで発行されたURIをコピって、それをGitHubの記事用リポジトリの`Settings`へ持っていきます。

### GitHubでWebhooksを作成する

あとは`Add webhooks`から新規に作成し、`Payload URL`にペーストしたら`Add Webhook`押下で保存。
トリガーとなる操作はデフォルトで`push`になってますので、やることはこれで全部です。

---

## 色を設定する

Vuetifyを採用したNuxtアプリケーションですと、CSS変数よろしく、各色が`nuxt.config.js`にて設定されています。
デフォルトだとダークモードの色だけ設定されているようです。

```js
    themes: {
      dark: {
        background: `#353D58`,
        primary: `#6383A9`,
        accent: `#586694`,
        secondary: `#9D7D66`,
        info: `#56A597`,
        warning: `#B6B480`,
      }
    }
```

今回はダークモードの色も調整しつつ、ライトモードのマテリアルカラーも考えてみました。
全体はこんな感じです。

```js
    theme: {
      options: {
        customProperties: true
      },
      dark: true,
      themes: {
        dark: {
          font: '#F5F5F5',
          background: `#282F43`,
          primary: `#6C8CB3`,
          accent: `#353B51`,
          secondary: `#9D7D66`,
          info: `#56A597`,
          warning: `#B6B480`,
          error: `#AB7878`,
          success: `#539164`
        },
        light: {
          font: '#473838',
          background: `#EEEEEE`,
          primary: `#7399C5`,
          accent: `#4E4747`,
          secondary: `#EBDABF`,
          info: `#BDE4DB`,
          warning: `#EBEAC3`,
          error: `#EACDC3`,
          success: `#C0ECCA`
        }
      }
    }
```

フォントの色、背景色を加え、あとはデフォルトの各要素に対応した色を設定しました。
（こんなのを作りつつ）

画像

画面上のものが、ローカル環境で各マテリアルカラーのついた文字をブラウザに出してみた時に撮ったスクショ。
下の2つが、それを参考にライト・ダークモードそれぞれで色を調整してみたやつです。実際にブラウザで表示しながら調整していたので、デザインツールで完結するわけではないのですが。（結局コーディングしながら作ることになる）

背景色は、いくらダークモードでも真っ黒ではつまらない。ライトモードも真っ白ではつまらない。ということで、深いネイビーやうっっっっっすいグレーといった形でひねりを入れてみました。ささいなこだわりですけど。

結果的にはこんな感じになっています。

画像 

画像

## ヘッダーを整える

特にデザインを考えずに作っているので行き当たりばったりですが、まずはヘッダーから。
最低限以下の3つを実装します。

- ナビゲーションドロワーの開閉ボタン
- サイトタイトル（トップページへのリンク付き）
- ダーク・ライトの切り替えスイッチ

### コンポーネント分割

`default.vue`が長ったらしくなってしまうので、各パーツごとにコンポーネント分割します。

```vue
<template>
  <v-app-bar
    :style="{background: $vuetify.theme.themes[theme].background}" 
    :clipped-left="clipped"
    fixed 
    flat 
    app
  >
    <slot />
  </v-app-bar>
</template>

<script>
export default {
  data() {
    return {
      clipped: false,
    }
  },
  computed: {
    theme() {
      return this.$store.state.theme.theme;
    },
  },
}
</script>
```



### ナビゲーションドロワーの開閉ボタン

```vue
<template>
  <v-navigation-drawer
    color="background" 
    app
    v-model="drawerStatus"
    right
    fixed
    temporary 
  >
    
  </v-navigation-drawer>
</template>

<script>
export default {
  model: {
    prop: "drawer",
    event: "click"
  },
  props: {
    drawer: {
      type: Boolean
    }
  },
  computed: {
    drawerStatus: {
      get() {
        return this.drawer;
      },
      set(changedStatus) {
        this.$emit("click", changedStatus);
      },
    },
  },
}
</script>
```

`default.vue`で呼び出します。

```vue
<template>
  <v-app :style="{background: $vuetify.theme.themes[theme].background}">

    <LayoutsNavigationDrawer v-model="drawer" />

    <LayoutsHeaderBody>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
    </LayoutsHeaderBody>

    <v-main>
      <Nuxt />
    </v-main>

    <LayoutsFooter />

  </v-app>
</template>

<script>
export default {
  data () {
    return {
      drawer: false,
    }
  },
  computed: {
    theme() {
      return this.$store.state.theme.theme;
    },
  },
}
</script>
```

共通レイアウト内でドロワーを呼び出し、ヘッダーには開閉ボタンだけをつけた状態です。

`v-app-bar-nav-icon`のクリックイベントで`drawer`の真偽値が切り替わり、ドロワーの`props`が変更された値を受け取ります。
ドロワー側は算出プロパティでそれをセットする他、今回は画面のどこかをクリックすることで閉じられるようにするために`temporary`を指定しているので、ドロワー側にクリックイベントを付けて、親コンポーネント側へ変更した`drawer`プロパティを戻します。

各所に記載した`theme`という算出プロパティは何かというと、とある処理でVuexストアに格納している文字列（`dark`あるいは`light`）を返却しています。
Vuetifyのコンポーネントにスタイルを付与するにあたって`:style="{background: $vuetify.theme.themes[theme].background}"`という記述をするわけですが、この際に現在設定されている方のテーマが当たるようにするためのプロパティです。
Vuexストアを使ってみたかったあまりこんな書き方をしたものの、後々詳しく書きますが、結果的には「現在のテーマ」という情報を`nuxt.config.js`とVuexストアの両方で持つことになってしまっているので、実際には`default.vue`で取り出した`nuxt.config.js`の値を`props`で伝えるだけでいいと思います。

ダーク・ライトの切り替え機能をつけるまでは、この算出プロパティはこんな感じにしておけばいいと思います。

```js
computed: {
  theme() {
    return this.$vuetify.theme.dark ? "dark" : "light";
  },
},
```

`dark`はデフォルトで`true`なので、こう書いているうちは必ずダークテーマが当たります。

### サイトタイトル

タイトルは`v-toolbar-title`コンポーネントで作ります。

```vue
<template>
  <v-toolbar-title 
    align-content="center"
    class="mx-auto"
  >
    <h1><NuxtLink to="/" class="text">{{ title }}</NuxtLink></h1>
  </v-toolbar-title>
</template>

<script>
export default {
  data() {
    return {
      title: `nawamemo`
    }
  },
}
</script>

<style lang="scss" scoped>
  h1 {
    margin: 0 0 10px 0;
    border-bottom: none;
  }

  a {
    color: var(--v-font-base) !important;
  }

  .text {
    font-family: 'Raleway', sans-serif !important;
    font-size: 18px !important;
    font-weight: bold;
    letter-spacing: 0.1em !important;
  }
</style>
```

Vuetifyによる装飾を行う場合、`a`タグには標準で`config`で設定されている`primary`のマテリアルカラーが当たります。

タイトル文字にはトップページへのリンクを付けているわけですが、この文字色に関しては本文の色と合わせたいので、このコンポーネント内に限定してスタイルを上書きしています。

```vue
<template>
  <v-app :style="{background: $vuetify.theme.themes[theme].background}">

    <LayoutsNavigationDrawer v-model="drawer" />

    <LayoutsHeaderBody>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
      <LayoutsHeaderTitle />
    </LayoutsHeaderBody>

    <v-main>
      <Nuxt />
    </v-main>

    <LayoutsFooter />

  </v-app>
</template>
```

サイトタイトルの情報は、いまのところタイトルコンポーネントでしか使う気がないのでコンポーネント内で値を保持していますが、もしフッターなど違う部分でも使いたいということがあれば、Vuexストアに入れておくなりしたほうがいいと思います。特にアプリケーションなどの場合は、サービス規約や説明文などで多用するでしょうから、サイト名を表記する全ての部分を算出プロパティにするなどして共通化するのがいいと思います。

### ダークモード・ライトモードの切り替えスイッチ

`v-switch`コンポーネントを使えば簡単にできると思うのですが、なんとなく自分で作ってみたくなったのでマークアップからやりました。

```vue
<template>
  <div id="switch" class="d-flex align-center">
    <div id="frame">
      <div />
    </div>
  </div>
</template>

<script>
export default {
}
</script>

<style lang="scss" scoped>
  #switch {
    height: 100%;

    #frame {
      width: 60px;
      height: 30px;
      border: 1px solid var(--v-font-base);
      border-radius: 50px;

      div {
        height: 100%;
        width: 50%;
        background: var(--v-font-base);
        border-radius: 50%;
        border: 4px solid var(--v-background-base);
      }
    }
  }
</style>
```

とりあえずこんなふうに要素とスタイルを書いて、ヘッダー内に配置します。

```vue
<template>
  <v-app :style="{background: $vuetify.theme.themes[theme].background}">

    <LayoutsNavigationDrawer v-model="drawer" />

    <LayoutsHeaderBody>
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
      <LayoutsHeaderTitle />
      <LayoutsHeaderToggleThemeSwitch />
    </LayoutsHeaderBody>

    <v-main>
      <Nuxt />
    </v-main>

    <LayoutsFooter />

  </v-app>
</template>
```

ここから、スクリプトを書いて機能を実装していきます。

### テーマの設定・切り替え

ダークモード・ライトモードの設定については、このような要件で作ることにしました。

- アクセス時、ユーザーの設定に合わせてダークモード・ライトモードいずれかを設定する
- テーマの状態は、`nuxt.config.js`の`$vuetify.theme.dark`の真偽値によって管理する
- その値は`default.vue`が一元管理し、各コンポーネントへは`props`で伝達する

















