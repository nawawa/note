久しぶりなもので、ちょっと忘れちゃいまして。
いろいろ調べながらユーザー登録・認証周りの処理を実装します。

フロントエンド側はSPAを想定しており、認証にはSanctumを用いますが、トークンではなくセッションによる認証を行いますので、普通にLaravel側で実装されてる認証機能を使うのとほとんど変わりありません。ただログアウト処理に関しては。SPA側でセッションを削除する処理を実装する形にしようと考えているので、Laravel側ではなにもしません。今回はフロント側の実装には触れませんので、今後フロントエンド領域のタスクに手を出した際に別途記事を書く予定です。

なお、内容のほとんどは[[@ucan-lab](https://qiita.com/ucan-lab)様の「Laravel Sanctum でSPA(クッキー)認証する」という記事](https://qiita.com/ucan-lab/items/3e7045e49658763a9566)の受け売りとなっておりますが、筆者がLaravelにてすでに書いているソースとの兼ね合いや、Postmanでの動作確認までを踏まえたやや簡易な内容にしようと思ってます。

## ユーザー登録

新規登録に関しては、単にUserが一人増えるだけなので、普通にCRUDでいうCを行えばいいのですが。

```php
  /**
   * 新規ユーザー作成
   *
   * @param User $user
   * @param array $param
   * @return User
   */
  public function createUser(User $user, array $input): User
  {
    $user->fill([
      'name' => $input['name'],
      'login_id' => $input['login_id'],
      'password' => Hash::make($input['password']),
      'icon_name' => ($input['icon_name']) ? $input['icon_name']: DefaultUserIcon::NAME,
      'icon_path' => ($input['icon_path']) ? $input['icon_path']: DefaultUserIcon::PATH,
    ]);
    $user->save();

    return $user; 
  }
```
### デフォルトのユーザーアイコンを用意する

任意のユーザーアイコンを設定できるが特に必須でもない、という仕様にしたかったので、入力値にアイコンの指定がなかった場合は`/storage/app/public/images`に配置してあるデフォルトアイコンを使用するようにしました。

アイコンのパスは`App`配下に`Consts`ディレクトリを作成し、その中に定数クラスを作成して管理しています。

### 登録したらすぐにログイン状態にする

ここをどうするかはサービス設計次第なんでしょうけど。

認証周りを厳格に行いたい場合は一回ログインページにリダイレクトさせるのがいいと思うのですが、今回に関しては登録が完了したらすぐに使い始められるようにしようと思ったので、こんなふうにしました。

```php
  /**
   * 新規ユーザー作成
   *
   * @param User $user
   * @param array $param
   * @return User
   */
  public function createUser(User $user, array $input): User
  {
    $user->fill([
      'name' => $input['name'],
      'login_id' => $input['login_id'],
      'password' => Hash::make($input['password']),
      'icon_name' => ($input['icon_name']) ? $input['icon_name']: DefaultUserIcon::NAME,
      'icon_path' => ($input['icon_path']) ? $input['icon_path']: DefaultUserIcon::PATH,
    ]);
    $user->save();
    Auth::login($user, $remember = true);

    return $user;
  }
```

AuthファサードのloginメソッドにUserモデルのオブジェクトを渡せばログイン済みにしてくれて、かつ第二引数にtrueを渡すと`remember me`にチェックが入る。とても楽ですね。

なお実際のソースでは、「データベースへの永続化」と、「Authファサードを利用した認証・Hashファサードによるパスワードのハッシュ化」についてはそれぞれ別のリポジトリクラスを作成し、実装を分離しています。

ここまでのテストコードはこんな感じで。

```php
    private array $user_param;
    
    public function setUp() :void
    {
        parent::setUp();

        $this->user_param = $this->createUserParam();
    }
    
    /**
     * @test
     */
    public function ユーザー追加()
    {
        $response = $this->postJson('/api/user/register', $this->user_param);
        $response
            ->assertStatus(201)
            ->assertExactJson([
                'data' => [
                    'login_id' => $this->user_param['login_id'],
                    'attribute' => [
                        'name' => $this->user_param['name'],
                        'icon' => DefaultUserIcon::PATH,
                    ],
                ],
            ]);
        $this->assertAuthenticatedAs(
          User::where('name', $this->user_param['name'])
          ->where('login_id', $this->user_param['login_id'])
          ->first()
        );
    }

    /**
     * 入力値生成
     *
     * @return array
     */
    private function createUserParam(): array
    {
        return [
            'name' => '結月ゆかり',
            'login_id' => 'yuduki_yukari',
            'password' => 'yukarisan_saiko',
            'icon_name' => null,
            'icon_path' => null,
        ];
    }
```

## ログイン

Sanctumを使います。

```php
    'paths' => ['api/*', 'login', 'sanctum/csrf-cookie'],
```

今回は（慣習上？）`web.php`にログイン処理のエンドポイントを書いており、`config\cors.php`のこの部分に、`login`を追加します。

```php
    /**
     * ログイン
     *
     * @param array $credentials
     * @param boolean $remember
     * @throws AuthorizationException
     * @return User
     */
    public function login(array $credentials, bool $remember=false): User
    {
        return (Auth::attempt($credentials, $remember)) ?
            $this->get_auth_user():
            throw new AuthorizationException(UserAuthentication::LOGIN_FAILURE);
    }

		/**
     * ログインしたユーザーを取得
     *
     * @return User
     */
    public function get_auth_user(): User
    {
        $this->request->session()->regenerate();
        return Auth::user();
    }
```

最低限の実装となっています。

Authファサードの`attempt`メソッドへ渡す引数は、第一引数が「`users`テーブルの何かのカラム」と「パスワード」の2つで構成された配列と、第二引数が`remember me`にチェックするかどうかの真偽値です。

今回渡している配列はこのようなもの。

```php
[
  'login_id' => $request->input('login_id'),
  'password' => $request->input('password'),
],
```

第一引数の「`users`テーブルの何かのカラム」については、デフォルトではメールアドレスになると思いますし、自分で追加したカラムでも何でもいいです。今回は独自に`login_id`というカラムを作っておりまして、「`login_id`カラムが入力値と等しいユーザーを検索し、パスワードが合致したら`true`を返す」という流れで処理が行われます。とてもシンプルですね。

これが`false`であれば`AuthorizationException`という例外を投げるようにはしていますが、全然使いこなせていません（レスポンスを考えられてないので）。とりあえず、定数クラスで定義した固定メッセージを引数に渡すだけになってます。現状そんなに意味はない。

```php
		private User $user;
    private int $user_id;
    private string $password;

    public function setUp() :void
    {
        parent::setUp();

        $this->user = User::factory()->create();
        $this->user_id = $this->user->id;
        $this->password = '$2y$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi';
    }

    /**
     * @test
     */
    public function ログイン()
    {
        $response = $this->post('/login', [
            'login_id' => $this->user->login_id,
            'password' => $this->password,
            'remember' => true,
        ]);

        $response->assertStatus(200);
      	$this->assertAuthenticatedAs($this->user);
    }

    /**
     * @test
     */
    public function 間違ったパスワード()
    {
        $this->withoutExceptionHandling();
        $this->expectExceptionMessage(UserAuthentication::LOGIN_FAILURE);

        $response = $this->post('/login', [
            'login_id' => $this->user->login_id,
            'password' => 'password',
            'remember' => true,
        ]);
        $response->assertStatus(403);
        $this->assertGuest($guard = null);
    }
```

テストコードは、最低限ですがこんな感じ。パスワードは、適当に生成して`Factory`に書いているものです。

二番目の以上系テストで書いている一行目・二行目は、それぞれ「例外処理を行わない」宣言と「例外発生時のエラーメッセージが引数に渡したものであること」をチェックするものです。これを書かないとただ単に自分で設定した例外がスローされてテストが止まってしまうので、意図的に例外を発生させるテストをする際には必要です。

## 動作確認

ここまでの内容は、単にSanctumをインストールして（最初から入ってたので筆者はやらなかったですが）、ログイン処理とテストを書いただけでできる内容です。

こうして実装したログイン処理の動作確認を外部から行うためには、いくつか設定を追加する必要があります。
今回はPostmanを使用して、「ログイン」と「認証ガード付きルートへのアクセス」を検証してみようと思います。

SeederでもTinkerでもいいので、とりあえずユーザーを一人追加しておきましょう。

```
Psy Shell v0.10.12 (PHP 8.1.0 — cli) by Justin Hileman
>>> $user = new User;
[!] Aliasing 'User' to 'App\Models\User' for this Tinker session.
=> App\Models\User {#3636}
>>> $user->create(['login_id' => 'yuduki_yukari', 'password' => Hash::make('yukarisan_saiko'), 'name' => 'yuduki',  'icon_name' => DefaultUserIcon::NAME, 'icon_path' => DefaultUserIcon::PATH]);
=> App\Models\User {#4451
     login_id: "yuduki_yukari",
     #password: "$2y$10$266CRguI.vov/ec5S6m0.e.h80m2cRzuXfiE5L9oLPqDHc/5zPyh6",
     name: "yuduki",
     icon_name: "default_icon.png",
     icon_path: "storage/default_icon.jpg",
     updated_at: "2022-03-04 13:31:24",
     created_at: "2022-03-04 13:31:24",
     id: 3,
   }
>>> 
```

### 設定をいろいろ追加

Laravel側の設定から。`.env`にて項目を変更・追加します。

```
SESSION_DRIVER=cookie

SANCTUM_STATEFUL_DOMAINS=localhost:ポート番号,localhost
```

`SESSION_DRIVER`を`file`から`cookie`へ変更するほか、`SANCTUM_STATEFUL_DOMAINS`を追加して`localhost:ポート番号` と `localhost`からのアクセスを許可します。

あとは`config/cors.php`もいじっておきます。今回は`web.php`にログイン処理へのルートを記述したので、そのパスを追加します。

```php
'paths' => ['api/*', 'login', 'sanctum/csrf-cookie'],
```

### Postman内に環境変数を設定

Postman右上の目のようなアイコンを押して、新しくEnviromentを追加。
そしたらこんな感じで`localhost:ポート番号`を`APP_URL`として定義しておきまして、

## ![スクリーンショット 2022-03-04 15.17.22](/Users/nawaryoga/Library/Application Support/typora-user-images/スクリーンショット 2022-03-04 15.17.22.png)

Postmanで追加したログインAPIの画面で、このようなスクリプトを入力。

```JavaScript
let csrfRequestUrl = pm.environment.get('APP_URL') + '/sanctum/csrf-cookie';
pm.sendRequest(csrfRequestUrl, function(err, res, {cookies}) {
    let xsrfCookie = cookies.one('XSRF-TOKEN');
    if (xsrfCookie) {
        let xsrfToken = decodeURIComponent(xsrfCookie['value']);
        pm.request.headers.upsert({
            key: 'X-XSRF-TOKEN',
            value: xsrfToken,
        });                
        pm.environment.set('XSRF-TOKEN', xsrfToken);
    }
});
```

![スクリーンショット 2022-03-04 15.40.57](/Users/nawaryoga/Desktop/スクリーンショット 2022-03-04 15.40.57.png)

これが、例えばVue.jsでAxiosなどを使った際に、Axios側で勝手に行ってもらえる処理だそうです。

一行目は、環境変数を取り出して`http:/localhost:ポート番号/sanctum/csrf-cookie`というエンドポイントへのURLを生成してます。
二行目ではそのURLへリクエストを送信し、`get`以外のメソッドでエンドポイントへリクエストを送信する場合に必ず含めなければならない`XSRF-TOKEN`の値を取得したあと、そのトークンを環境変数へぶち込んで`envioment`全体で共有できるようにしています。

本来はトークンが変わる度に手動でヘッダのトークンを書き換えなければなりませんが、このスクリプトはその作業を自動化してくれるわけですね。筆者もモバイルトークンを必要とするAPIをLaravelで構築し、Postmanから動作確認していたことがあるので、そのめんどくささは体験したことがあります。その手作業を省けるのは助かります。

そんな感じでヘッダを整えて、最終的にこの状態にします。

![スクリーンショット 2022-03-04 15.42.11](/Users/nawaryoga/Library/Application Support/typora-user-images/スクリーンショット 2022-03-04 15.42.11.png)

あとは`Body`にJsonを入力。
筆者の場合は、`login_id` `password` `remember`の3つの要素を持つ配列を期待するようにコントローラーにて処理を記述したので、配列はこのようになります。

```json
{
    "login_id": "yuduki_yukari",
    "password": "yukarisan_saiko",
    "remember": true
}
```

そうして状態を整え、リクエストを送信。

![スクリーンショット 2022-03-04 15.46.49](/Users/nawaryoga/Library/Application Support/typora-user-images/スクリーンショット 2022-03-04 15.46.49.png)

ステータス200でちゃんと帰ってきました。
`UserResource`で整形した通りにログインしたユーザーの情報が返ってきています。

ログインできたので、ログインしていないとアクセスできないルートにもリクエストしてみましょう。
ユーザー情報を取得するAPIである`localhost:8080/api/user/3`にアクセスすると、ちゃんとIDが3である`yuduki`さんが取得できました。

![スクリーンショット 2022-03-04 16.12.38](/Users/nawaryoga/Library/Application Support/typora-user-images/スクリーンショット 2022-03-04 16.12.38.png)

このAPIはGETメソッドでリクエストしますが、その場合は`Origin`に`APP_URL`環境変数を入れる必要があります。

### 出がちなエラー

```
Symfony\Component\Routing\Exception\RouteNotFoundException: Route [login] not defined. in file /var/www/html/vendor/laravel/framework/src/Illuminate/Routing/UrlGenerator.php on line 444
```

動作確認をしていて`login`なんていうルートはないぜ！というエラーが出る場合、ヘッダに`Accept` `application/json`を入れ忘れています。

Laravelはこれがないとリクエスト元がAPIクライアントであることを理解してくれないので、「誰だこいつは？ログインさせな！」と、ログインページへリダイレクトさせようとします。でも今回はログインページを作っていないので、無いページへアクセスしようとしてエラーになるということになります。

## まとめ

- ログイン処理は`Attempt`に配列と真偽値を渡すだけ。
- ログアウトはサーバーサイドでもできるけど、フロント側でセッションを破棄すればいいだけなので実装しなくていい場合もある

デプロイした後は`.env`にSPAのドメインを入力する必要があったりもするのですが、設定ファイルをどうするかはSPA側がどのように運用されるか（別ドメインか、Laravel内蔵か、サブドメインか）で変わってくる部分もあるようです。まだフロントエンドには手を付けてない（全然考えてない）ので、実際のSPAで運用しようとした際にどんな設定が足りてないかは、現時点ではわからない状態です。

筆者自身は別リポジトリで、Laravelアプリケーションからは完全に分離したSPAを構築しようと考えてますので、そこでどのようになるかは着手してから検証しようと思います。















