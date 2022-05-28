ローカルのDocker環境で、Laravel SanctumとNuxtのAuthモジュールを使って認証を行うまでの手順をまとめます。

今回初めてNuxt・Laravel間の認証を実装しようといろいろ調べましたが、RESTfulにしようという意識もあってなのか、サーバーとクライアント間で状態を維持しない、JWTトークンを使用した方法を解説する記事がほとんどでした。あるいはCSR前提でSSRについては考慮されていない内容であったり、そもそも認証方法もAxiosで`/api/login`ルートへPOSTするとか、はたまたAuthモジュールを使うとか、記事によってマチマチだったり…

かなりググりまくってようやく実装できたので、本記事にてまとめようと思います。すぐ古くなるでしょうけど…

## 実際の設定内容

```.env
NODE_ENV='development'
API_URL=http://<Nginxコンテナのコンテナ名>:<Nginxコンテナのポート>
SERVER_URL=http://0.0.0.0:<Laravelコンテナのポート>
NODE_URL=http://0.0.0.0:<Nuxtコンテナのポート>
```

```nuxt.config.js
axios: {
    prefix: process.env.API_URL,
    proxy: true,
    credentials: true,
  },

  proxy: {
    '/laravel': {
      target: process.env.NODE_URL,
      pathRewrite: { '^/laravel': '/' }
    },
    '/api/': process.env.SERVER_URL,
  },

  auth: {
    redirect: {
      login: '/login',
      logout: '/', 
      callback: '/',
      home: '/'
    },
    strategies: {
      laravelSanctum: {
        provider: 'laravel/sanctum',
        url: process.env.SERVER_URL,
        cookie: {
          name: 'XSRF-TOKEN',
        },
        endpoints: {
          csrf: {
            url: '/sanctum/csrf-cookie'
          },
          login: { url: '/auth/login', method: 'post' },
          logout: { url: '/auth/logout', method: 'post' },
        }
      }
    }
  },

  router: {
    middleware: ['auth']
  },

  serverMidlleware: [
    { path: '/login', handler: '~/middleware/login_page' }
  ],

  privateRuntimeConfig: {
    axios: {
      prefix: process.env.API_URL,
    }
  },
  publicRuntimeConfig: {
    axios: {
      browserBaseURL: process.env.NODE_ENV !== 'production' ? process.env.SERVER_URL : '',
    }
  },
```

```index.js
export const actions = {
  async nuxtServerInit({ commit }, { app }) {
    
    await app.$axios
      .$get('/api/user') // ログインしてるユーザーのオブジェクトを返すだけのAPI(auth:sanctumガード付きルート)
      .then((authUser) => {
        app.$auth.setUser(authUser)
      })
      .catch((err) => {
        app.$auth.setUser(null)
      })

  }
}
```
```/middleware/login_page.js
// NuxtLinkからではなく http://0.0.0.0:4001/login へ直接アクセスされた場合にリダイレクトする
export default function ({ store, redirect }) {
  if (store.state.auth.user) {
    return redirect('/')
  }
}
```

### 認証方法の違いについて

JWTトークン認証は、クライアントごとにユニークなトークン文字列を持っておくことでそれを鍵にしようという方法です。REST原則においてはサーバーとクライアントは状態を持ちませんので、サーバーも認証においてセッション情報を保持したりしません。

しかしながらセッション認証と比較した場合にJWTトークンはあまりセキュアではないという懸念があることで、認証状態についてはステートフルでいいんでねえの、という雰囲気になってもいるようです。Laravel Sanctumはどちらの認証方法もサポートしているものの、公式にはセッション認証を推奨しています。

### Authモジュールについて

既存の記事では、

- axiosでPOSTしてログイン
- nuxtServerInitメソッド内でユーザー情報を取得(認証ガード付きAPIへGET)してstoreへ格納
- middlewareでstoreを参照
- storeにユーザー情報があれば認証済み、なければ未認証としてログインページへリダイレクト

という流れを自前実装しているものも多いです。

しかし、これらの大部分はNuxtが公式にサポートしているAuthモジュール側で実装してくれているようだったので、とりあえずそれをインストールしてあとは細かい部分を自分で入れればどうにかなりました。「これさえ入れとけば」的なものがあってくれるのは助かりますね。

### 環境について

TypeScriptの学習を後回しにしていることもあって、Nuxtはバージョン2系で構築しています。
また、CSRだけではなくSSRも行う前提です。SPA+JWTトークンという構成の記事が多く、案外情報がなかったので、役に立てば幸いです。

## Laravel側の設定

CORSを許可しないといけません。

例えば筆者の環境では、Laravelコンテナは`http://localhost:9000`で動作し、Nuxtコンテナは`http://localhost:4001`で動作しています。なので、この場合は`locaihost:4001`が`localhost:9000`が作成・返却したレスポンスを参照・使用することを許可する必要があります。

```php
<?php

return [

    /*
    |--------------------------------------------------------------------------
    | Cross-Origin Resource Sharing (CORS) Configuration
    |--------------------------------------------------------------------------
    |
    | Here you may configure your settings for cross-origin resource sharing
    | or "CORS". This determines what cross-origin operations may execute
    | in web browsers. You are free to adjust these settings as needed.
    |
    | To learn more: https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
    |
    */

    'paths' => [
        'api/*', 
        'auth/login',
        'auth/logout',
        'sanctum/csrf-cookie',
    ],

    'allowed_methods' => ['*'],

    'allowed_origins' => [env('FRONT_ORIGIN')],

    'allowed_origins_patterns' => [],

    'allowed_headers' => ['*'],

    'exposed_headers' => [],

    'max_age' => 0,

    'supports_credentials' => true,

];
```

`.env`内に`FRONT_ORIGIN=http://0.0.0.0:4001`を追加し、その環境変数を`config/cors.php`で呼び出します。


### Sanctumの設定

普通に公式ページや他の記事でも書いてあることをするだけなので、読み飛ばしてもいいところです。

```app/Http/Kernel.php
         'api' => [
             \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
             'throttle:api',
             \Illuminate\Routing\Middleware\SubstituteBindings::class,
         ],
```

```.env
SESSION_DRIVER=cookie
SESSION_DOMAIN=0.0.0.0

SANCTUM_STATEFUL_DOMAINS=0.0.0.0:4001
```

```config/cors.php
    'paths' => [
        'api/*', 
        'auth/login',
        'auth/logout',
        'sanctum/csrf-cookie',
    ],

    //略

    'supports_credentials' => true,
```

### ログイン処理を追加

ここは各自いろいろ書き方のある部分かと思います。(メールアドレスの代わりにユーザーIDを使いたいとか)
とりあえずやるべきことは、ログインの場合は`attempt`メソッドで入力値をデータベースと照合することとセッションを生成すること、ログアウトの場合はセッションを破棄することです。

```LoginController.php

<?php

// 略

// とりあえず動作させるためhttps://qiita.com/ucan-lab/items/3e7045e49658763a9566からコピペしたソースです。
// 実際はAuthファサードは直に呼ばず、リポジトリクラスを追加してそこにAuthを使いたいメソッドを切り出したりします。
// バリデーションもRequestクラスを別途作成して行うなどする予定です。

class LoginController extends Controller
{
    /**
     * @param AuthManager $auth
     */
    public function __construct(
        private AuthManager $auth,
    ) {
    }

    /**
     * @Method POST
     * @param  Request  $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required'
        ]);
        
        if (auth()->attempt($credentials)) {
            $request->session()->regenerate();

            return response()->json(Auth::user());
        }

        return response()->json(['message' => 'ユーザーが見つかりません。'], 422);
    }

    /**
     * Undocumented function
     *
     * @param Request $request
     * @return JsonResponse
     */
    public function logout(Request $request): JsonResponse
    {
        if ($this->auth->guard()->guest()) {
            return new JsonResponse([
                'message' => 'Already Unauthenticated.',
            ]);
        }
        
        $this->auth->guard()->logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return new JsonResponse([
            'message' => 'Unauthenticated.',
        ]);
    }
}
```

ログインルートは、基本的には`web.php`内に書くようです。はっきりとした理由はドキュメントを読んでもよくわかりませんでしたけど、

> 通常、SanctumはLaravelの「web」認証ガードを利用してこれを実現します。

とあるので、webルート用に実装されてるミドルウェアを呼ぶためなのかな？とは思います。

実際Sanctumが何をするかというと、`SANCTUM_STATEFULL_DOMAIN`に書かれたドメインに限ってセッションをステートフルにすること、CSRFトークンを使ってCORSを許可しているサイトかどうかを識別すること、セッションに認証情報がなければモバイルトークンの有無を見るなどで、ルートに対するガードを行う機能はありません。

```routes/web.php
  Route::prefix('auth')->group(function() {
      Route::post('/login', [LoginController::class, 'login']);
      Route::post('/logout', [LoginController::class, 'logout']);
  });
```

## Nuxt側の設定

ふたつのモジュールをインストールする必要があります。

- nuxt/axios([ドキュメント](https://axios.nuxtjs.org/))
- nuxt/auth([ドキュメント](https://auth.nuxtjs.org/))

### axios

```js
  modules: [
    '@nuxtjs/axios',
  ],
```

インストールしたこれらを`modules`にて呼び出します。
これによりNuxtアプリケーション自体に彼らが組み込まれるため、インポートせずともコンポーネント内であれば`this.$axios`とか、`nuxtServerInit({ commit }, { app })`の中であれば`app.$auth`とかの記述で使えるようになります。

```js
  axios: {
    prefix: process.env.API_URL,
    proxy: true,
    credentials: true,
  },

  proxy: {
    '/laravel': {
      target: process.env.NODE_URL,
      pathRewrite: { '^/laravel': '/' }
    },
    '/api/': process.env.SERVER_URL,
  },
```

axiosの設定はこんな感じにしてます。`SERVER_URL`はLaravelコンテナ(nginxコンテナではなく)の、`NODE_URL`はNuxtコンテナのオリジンを格納している環境変数です。

設定値の解説(axiosのドキュメントをかいつまんで引用)はこんな感じ。

- `prefix`…`proxy`を`true`にする場合の`baseURL`の代わり。後述する`RuntimeConfig`が読み込まれていない場合のデフォルト値。
- `credentials`…`true`にする。異なるドメイン間でCookie等の資格情報をやりとりするために必要。

他には下にある配列でプロキシ設定を行うわけですが、これは、ブラウザからXMLHttpRequestによって送信されたリクエストをNuxt(Node.js)側で受けたあと、外部サーバーへそのリクエストを転送する際にリクエスト先を指定するための設定です。(…と思ってます、この辺あんまり理解してません)

たとえば上記の例では`$axios.<なんらかのHTTPメソッド>('/api')`は、この設定により$axios.<なんらかのHTTPメソッド>(`http://0.0.0.0:9000/api)`へのリクエストに書き換わります。 

```nuxt.config.js
  privateRuntimeConfig: {
    axios: {
      prefix: process.env.API_URL,
    }
  },
  publicRuntimeConfig: {
    axios: {
      browserBaseURL: process.env.NODE_ENV !== 'production' ? process.env.SERVER_URL : '',
    }
  },
```

`privateRuntimeConfig`はサーバー専用、`publicRuntimeConfig`はサーバーとクライアント(ブラウザとか)共通で使用されるオブジェクトです。まあ、ざっくりと`privateRuntimeConfig`がSSR用、`publicRuntimeConfig`がCSR用と考えればいいんじゃないでしょうか。

また、`privateRuntimeConfig`はフロントエンドで表示されるべきでない環境変数(APIキーとか)を扱うものでもあります。

今回の場合は両者ともにaxiosの設定のみ格納しており、`prefix`はデフォルトのリクエスト先、`browserBaseURL`はクライアントサイドからのリクエスト先を指定します。

上記の設定値についてですが、CORSエラーと戦いながらこの形に落ち着いたものです。LaravelとNuxtそれぞれで環境を構築した当初は、とりあえずつなぎこみをしようと思い、認証のない簡単なAPIとGETリクエスト(`aryncData`)を書いていました。

どんな値を指定してもなかなかリクエストを届けられなかったのですが、なにかの記事で「オリジンはコンテナ名で指定する必要がある」ということを知ったため、`API_URL`には`http://<Nginxコンテナの名前>:80`を入力。`http://<Nginxコンテナの名前>:80/api/<ルート名>`へリクエストするようにしたらうまくいきました。

コンテナ名は`docker compose ps`を実行すると確認できます。

### auth

```nuxt.config.js
  modules: [
    '@nuxtjs/axios',
    '@nuxtjs/auth-next', // 同様に追加
  ],
```

authモジュールがやってくれるのは、例えばこんなことです。

- ログイン中かどうかを真偽値で返してくれる(`$auth.loggedIn`)
- ログインしたユーザーをストア(`auth.js`がエディタからは見えないけどある)へ格納してくれる
- 認証方法のパターンをいくつか指定できる(Laravel Sanctum とか Google とか)

axios単体で認証機能を実装しようとしたら色々と自前で実装する必要が出てきますが、authモジュールはそのへんの需要を諸々満たしてくれてます。すごい。

```nuxt.config.js
  auth: {
    redirect: {
      login: '/login', 
      logout: '/', 
      callback: '/',
      home: '/'
    },
    strategies: {
      laravelSanctum: {
        provider: 'laravel/sanctum',
        url: process.env.SERVER_URL,
        cookie: {
          name: 'XSRF-TOKEN',
        },
        endpoints: {
          csrf: {
            url: '/sanctum/csrf-cookie'
          },
          login: { url: '/auth/login', method: 'post' },
          logout: { url: '/auth/logout', method: 'post' },
        }
      }
    }
  },
```

authモジュールは内部的にaxiosモジュールを使用してリクエストを飛ばすため先程の設定が必要でした。
それに加え、`auth: {}`の中にauthモジュール側の設定を色々書いていきます。

`redirect`の各設定値はこんな感じです。
- login (ログインが必要な場合のリダイレクト先URL)
- logout (ログアウトしたあとのリダイレクト先URL)
- callback (ログイン終了後、GoogleなどのIDプロバイダがリダイレクトするURL)
- home (ログイン終了後、NuxtがリダイレクトするURL)

ログインを求めたいページへ未認証でアクセスされた場合にログインページへリダイレクトする、ログインできたらトップページへリダイレクトする…といった処理は、全てこのオブジェクトを配置すれば実現できます。楽。

`strategies`は、認証方法それぞれで個別に使用する設定をごにょごにょ書く場所です。今回はLaravel Sanctumを使ってセッションに状態をもたせる形の認証を行うため、`/sanctum/csrf-cookie`へGETリクエストを送ってXSRFトークンをもらう設定とか、`/auth/login`や`/auth/logout`へPOSTする際にCookie内のXSRFトークンを参照する設定とかを書きます。

あとはこんなふうにログインページのコンポーネントを作れば…

```vue
<template>
  <section>
    <h1>ログイン</h1>
    <form @submit.prevent="submit">
      <div>
        <label for="email">email</label>
        <input type="text" id="email" v-model="email" />
      </div>
      <div>
        <label for="password">password</label>
        <input type="password" id="password" v-model="password" />
      </div>
      <button type="submit">login</button>
    </form>
  </section>
</template>

<script>
export default {
  data() {
    return {
      email: '',
      password: '',
    }
  },
  methods: {
    async submit() {
      await this.$auth.loginWith('laravelSanctum', {
        data: {
          email: this.email,
          password: this.password
        }
      })
        .then((res) => {
        })
        .catch(err => {

        });
    },
  },
}
</script>
```

ログイン機能は出来上がりです。

ログアウトも、Laravel側でログアウトの処理を書いていれば、あとはこんなふうに作れます。

```vue
<template>
  <button @click="logout">
    ログアウト
  </button>
</template>

<script>
export default {
  methods: {
    async logout() {
      await this.$auth.logout();
    }
  }
}
</script>
```

ありがたいことに、ログインした後はログアウトするまでログインページへアクセスできなくなっています。

とはいえ普通に考えたら、

```vue
<Nuxtlink to="/login">ログイン</NuxtLink>
```

みたいなリンクは、認証状態を見て出したり出さなかったりしたほうがいいでしょう。

### ログインしても、ログインページ入れるが？

と思ったかもしれませんね。無対策だと、ブラウザのアドレスバーへURLを直に入力すれば、ログインしていようがログインページは開けます。めっちゃ不具合起きそうだし、もといこの挙動自体が不具合ですよね。

これ、NuxtのNode.jsサーバーへリクエストされた時点では認証状態を参照できないがために起こります。storeの中身もページが再レンダリングされる時点で消えてしまうので、単に`middleware`でstoreを参照するようなことをしても無意味です。

### serverMiddlewareとnuxtServerInitで解決する

とりま`nuxtServerInit`に処理を書きます。

```index.js
export const actions = {
  async nuxtServerInit({ commit }, { app }) {
    
    await app.$axios
      .$get('/api/user')
      .then((authUser) => {
        app.$auth.setUser(authUser)
      })
      .catch((err) => {
        app.$auth.setUser(null)
      })

  }
}
```

セッションが継続している(ログイン中である)場合は、`authuser`にユーザーの情報が入ってくるので、この時点でstoreに格納してしまいます。もしセッションが切れている場合は`401 Unauthorized`が返ってくるので、そのエラーをキャッチしてstoreの中身をリセットします。

```nuxt.config.js
  serverMidlleware: [
    { path: '/login', handler: '~/middleware/login_page' }
  ],
```

あとはこんなふうに、「'/login'へのリクエストをサーバーが受け取ったら、まずこのミドルウェアを呼びます」の記述を加えて…

```js
export default function ({ store, redirect }) {
  if (store.state.auth.user) {
    return redirect('/')
  }
}
```

storeの中身を見て、ユーザー情報があればトップへリダイレクトするミドルウェアを実装。
以上で、直にURLを入力された場合でもログインページを開かせないようにできました。


## まとめ

GWの10連休をまるまる使っても実装できなかったのは、まあセッションやCookieについてあまりに無理解だったことや、ちょいちょい`cors.php`や`SANCTUM_STATEFULL_DOMAIN`等の設定値をミスしてたこともありましたが、一番は「ログインページへの直アクセス対策」に時間を取られていたのでした。(むしろそれ以外はスタンダードなことしかしていない)

どう時間を取られたかといえば、記事を漁る中で「Cookieにセッション情報が書き込まれてるわけだから、それを参照するミドルウェアが必要なのか？」とか、「結局必要なライブラリはなに？`nuxt-universal-cookie`？`cookie-parser`？`js-cookie`?`express-session`？」などと迷走していたという具合でして……

最終的には「そっか、普通にサーバーサイドでstoreを更新すればミドルウェアで見れるわ」ということに気づき、@ucan-labさんと@h23kさんの記事を足して2で割ったくらいの結論に落ち着きました。

とはいえ、知識不足の中バリエーションに富んだ実装手順の中から適切な情報を探し出すのはなかなかキツかったです。


---


## 参考記事リンク
- nuxt/axios
https://axios.nuxtjs.org/

- nuxt/auth
https://auth.nuxtjs.org/

- NuxtJS
https://nuxtjs.org/docs/directory-structure/nuxt-config/ 
https://nuxtjs.org/docs/configuration-glossary/configuration-runtime-config/

- Nuxtのライフサイクル
https://qiita.com/yosuke_takeuchi/items/059b2d33fa9f46e7eb07

- Readouble.com
https://readouble.com/laravel/8.x/ja/sanctum.html
https://readouble.com/laravel/8.x/ja/authentication.html

- Laravel Sanctumとログイン処理
https://qiita.com/ucan-lab/items/3e7045e49658763a9566

- nuxt/authによる実装方法やCORS許可の設定など
https://zenn.dev/nagi125/articles/78d931f9b72fff1f1a7d
https://zenn.dev/nagi125/articles/7d01336868c79654b4af

- serverMiddleware周り
https://develop365.gitlab.io/nuxtjs-2.8.X-doc/ja/api/configuration-servermiddleware/
https://blog.cloud-acct.com/posts/nuxt3-server-middleware/

- nuxtServerInitでstoreに値をぶちこむやり方
https://qiita.com/h23k/items/f70a6d208fc4e720e11c
https://qiita.com/kiyoshi999/items/9ef7c89e3eaff444286d

