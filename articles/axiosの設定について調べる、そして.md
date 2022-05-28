## インストール

```bash
yarn add @nuxtjs/axios
```

そして、nuxt.config.jsのmodulesセクションに追加してください。

```js
export default {
  modules: ['@nuxtjs/axios']
}
```

以上です。これで、Nuxtアプリで`$axios`（グローバル変数）が使えるようになりました。 ✨

## 設定

nuxt.config.js に axios オブジェクトを追加して、すべてのリクエストに適用されるグローバルなオプションを設定します。

```js
export default {
  modules: [
    '@nuxtjs/axios',
  ],

  axios: {
    // proxy: true
  }
}
```

---

## オプション

nuxt.config.js の axios プロパティを使って、異なるオプションを渡すことができます。

```javascript
export default {
  axios: {
    // Axios options here
  }
}
```

## Runtime Config

実運用で環境変数を使用する場合は、実行時設定の使用が必須です。
そうしないと、ビルド時に値がハードコーディングされ、変更されなくなります。

```
ハードコーディング：ソース中に固定の値が埋め込まれること

そういう値は環境変数なり定数なりから実行時に呼び出されるのが望ましい(ソフトコーディング)
```

サポートされているオプション

- `baseURL`
- `browserBaseURL`

### nuxt.config.js

```js
export default {
  modules: [
    '@nuxtjs/axios'
  ],

  axios: {
    baseURL: 'http://localhost:4000', // ランタイムコンフィグが提供されない場合のフォールバックとして使用されます。
  },

  publicRuntimeConfig: {
    axios: {
      browserBaseURL: process.env.BROWSER_BASE_URL
    }
  },

  privateRuntimeConfig: {
    axios: {
      baseURL: process.env.BASE_URL
    }
  },
}
```

## prefix, host and port

これらのオプションは、`baseURL` と `browserBaseURL` のデフォルト値として使用されます。

これらは環境変数 API_PREFIX, API_HOST (または HOST), API_PORT (または PORT) でカスタマイズすることができます。

`prefix` のデフォルト値は / です。

## baseURL

- Default: `http://[HOST]:[PORT][PREFIX]`

サーバーサイドのリクエストを行う際に使用されるベース URL を定義する。

環境変数 `API_URL` は、 `baseURL` を上書きするために使用することができる。

**警告: ** `baseURL` と `proxy` は同時に使用することができないため、 `proxy` オプションを使用している場合は、 `baseURL` の代わりに `prefix` を定義する必要がある。

## browserBaseURL

- Default: `baseURL`

クライアントサイドのリクエストを行う際に使用するベース URL を定義し、その先頭に追加します。

環境変数 `API_URL_BROWSER` は、 `browserBaseURL` を上書きするために使用することができます。

**警告:** `proxy` オプションが有効な場合、 `browserBaseURL` のデフォルトは `baseURL` の代わりに `prefix` になります。

## https

- Default: `false`

true` に設定すると、 `baseURL` と `browserBaseURL` の両方にある `http://` は `https://` に変更されます。

## proxy

- Default: `false`

Axios と Proxy モジュールを簡単に統合することができます。これは、CORS と生産/配備の問題を防ぐために強く推奨されます。

```js
{
  modules: [
    '@nuxtjs/axios'
  ],

  axios: {
    proxy: true // デフォルトのオプションを持つオブジェクトも可能です。
  },

  proxy: {
    '/api/': 'http://api.example.com',
    '/api2/': 'http://api.another-website.com'
  }
}
```

Note: 手動で @nuxtjs/proxy モジュールを登録する必要はありませんが、依存関係にあるモジュールである必要があります。

Note: proxy モジュールでは、API エンドポイントへのすべてのリクエストに /api/ が追加されます。これを削除したい場合は、pathRewrite オプションを使用します。

```js
proxy: {
  '/api/': { target: 'http://api.example.com', pathRewrite: {'^/api/': ''} }
}
```

## credentials

- Default: `false`

バックエンドに認証ヘッダを渡す必要があるリクエストを baseURL に発行する際に、withCredentials axios 設定を自動的に設定するインターセプターを追加します。