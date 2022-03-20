親コンポーネント`default.vue`のクリックイベントで切り替わる真偽値を、`v-navigation-drawer`をラップした子コンポーネントへ渡したくなりました。

```dafault.vue
<template>
  <v-app dark>
    <LayoutsNavigationDrawer v-model="drawer" />

    <v-app-bar
      app
    >
      <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
    </v-app-bar>

    <v-main>
      <v-container>
        <Nuxt />
      </v-container>
    </v-main>

  </v-app>
</template>

<script>
export default {
  data () {
    return {
      drawer: false,
    }
  }
}
</script>
```

親コンポーネントが持つ`drawer`が、`v-model`で子コンポーネントと紐付けられています。
つまり、親の`drawer`に格納される`true` `false`によって、子のUIが変更されてビューの再レンダリングが行われるということです。同時に、子からの入力によって親の`drawer`が書き換わるということでもあります。（双方向データバインディング）

初期状態として、親の`drawer`には`false`が格納されています。

## クリックで値が書き換わり、子コンポーネントへ値が渡る流れ

```vue
<v-app-bar
  app
>
  <v-app-bar-nav-icon @click.stop="drawer = !drawer" />
</v-app-bar>
```

ヘッダーに設置されているボタンをクリックすると、`drawer`に格納されている`true`か`false`が逆の値に切り替わります。
するとこの値が紐付けられている子コンポーネントへ値が渡ります。

```vue
<LayoutsNavigationDrawer v-model="drawer" />
```

子コンポーネントのUIはこのようになっています。

```vue
<template>
  <v-navigation-drawer
    app
    v-model="drawerStatus"
    right
    fixed
    floating
    temporary
  >
    <slot />
  </v-navigation-drawer>
</template>
```

`v-navigation-drawer`がラップされており、算出プロパティ`drawerStatus`が返す値によって、開いたり閉じたりするようになっています。`script`は下記のとおりです。

```vue
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

親から渡ってきた値を、子コンポーネント内で任意の用途にて使用したいという場合があります。例えば、`<input>`要素に与えられた入力は本来`input`イベントを発火させますが、それとは違う自前のメソッドを発火させて、そちらで入力値を受け取ってよしなに処理を行いたい場合です。

その場合はこのように`model`オプションでプロパティとイベントを結びつけることで解決します。
```vue
<script>

  model: {
    prop: "drawer",
    event: "click"
  },

</script>
```
要するに今回のケースであれば、`drawer`の値は`click`イベントで操作するよ、という宣言です。

子コンポーネントに配置されている`v-navigation-drawer`は、算出プロパティの`getter`関数が返す値を使用します。

```vue
  <v-navigation-drawer
    app
    v-model="drawerStatus"
  >
```

```vue
<script>
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
</script>
```
親でクリックがあると、`drawerStatus`の`setter`関数がクリックイベントを検知し、親で格納された`true`を受け取ります。その値を`getter`関数が返し、`v-navigation-drawer`が展開されます。

画面のどこかをクリックすると`drawer`に`false`が入り、ドロワーが閉じられます。

今回の場合は、`temporary`を指定した`v-navigation-drawer`が閉じられる際のクリックイベントの実装が見えないところにあるために算出プロパティを用いる必要がありましたが、自分で実装する場合は単に`methods`で`this.$emit("イベント", 値);`を書けば、それだけで変更された`props`内の値を親へ渡すことができます。

つまり、今回のような実装でいうならば「親コンポーネントからは`true`、子コンポーネントからは`false`が渡されるだけ」の実装にすることができるわけです。とてもシンプルです。

---

### 参考リンク

https://qiita.com/toshifumiimanishi/items/07c8e96eb6f8e50c5e27
https://qiita.com/ume-kun1015/items/ebd447f769cd9ad1352c