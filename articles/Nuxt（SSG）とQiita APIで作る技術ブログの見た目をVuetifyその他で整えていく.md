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

### 開閉ボタン

`default.vue`が長ったらしくなってしまうので、ナビゲーションドロワーをコンポーネントへ切り出します。

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

上記のソースでは、共通レイアウト内でドロワーを呼び出し、ヘッダーには開閉ボタンだけをつけた状態です。

`v-app-bar-nav-icon`のクリックイベントで`drawer`の真偽値が切り替わり、ドロワーの`props`が変更された値を受け取ります。
ドロワー側は算出プロパティでそれをセットする他、今回は画面のどこかをクリックすることで閉じられるようにするために`temporary`を指定しているので、ドロワー側にクリックイベントを付けて、親コンポーネント側へ変更した`drawer`プロパティを戻します。