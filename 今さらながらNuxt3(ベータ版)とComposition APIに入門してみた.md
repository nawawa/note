環境構築はこちらを真似しました。

https://zenn.dev/szn/articles/nuxt-3-with-docker-compose

行ったことや感じたことの備忘録です。

## Vuetify3を入れる

こちらを真似。

https://zenn.dev/winteryukky/articles/87a40b60fddb96#vuetify3%E3%81%AE%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB

## Volarが思いの外使いにくかった

型情報の表示やpropsの型チェックなどをしてくれる分には良さそうなのですが、現状ではVeturを使ってたときのような感覚で`<script setup lang="ts">`なり`<style lang="scss" scoped>`なりを入力補完してもらえるわけではなかったので、まだTypeScriptの勉強を始めていない自分としてはそこまでの恩恵を感じられてません。とはいえちゃんとしたアプリを作っていればきっと活躍してくれるでしょう。

特にわからなかったのは、設定をミスっていただけなのかもしれませんが、nuxt.config.tsで「モジュール 'nuxt' またはそれに対応する型宣言が見つかりません。」警告が出続けていたことです。VSCode上だけで警告されているだけでビルドも実行も問題なくできていましたが、なんとなく気になりました。

結局安定して使えるVeturを今でも使っています。

## Google Fontを使うために行ったこと

標準モジュールがまだNuxt3に対応していないようで、こちらのissueからコピったソースで動かすようにしました。

https://github.com/nuxt-community/google-fonts-module/issues/67#issuecomment-978852151

```ts
buildModules: [
  ["./modules/fonts/index.ts", {
    families: {
      Montserrat: [400],
    },
  }]
],
```

これでスタイルから指定すればMontserrat使えました。わーい





## 