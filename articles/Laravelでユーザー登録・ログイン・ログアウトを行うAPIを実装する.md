久しぶりなもので、ちょっと忘れちゃいまして。
いろいろ調べながらユーザー登録・認証周りの処理を実装します。

フロントエンド側はSPAを想定しており、認証にはSanctumを用いますが、トークンではなくセッションによる認証を行いますので、普通にLaravel側で実装されてる認証機能を使うのとほとんど変わりありません。ただログアウト処理に関しては。SPA側でセッションを削除する処理を実装する形にしようと考えているので、Laravel側ではなにもしません。今回はフロント側の実装には触れませんので、今後フロントエンド領域のタスクに手を出した際に別途記事を書く予定です。

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

任意のユーザーアイコンを設定できるが、特に必須でもないという仕様にしたかったので、入力値にアイコンの指定がなかったら`/storage/app/public/images`に配置してあるデフォルトアイコンを使用するようにしました。

アイコンのパスは`App`配下に`Consts`ディレクトリを作成し、その中に定数クラスを作成して管理しています。

### 登録したらすぐにログイン状態にする

ここをどうするかはサービス設計次第なんでしょうけど。

認証周りを厳格に行いたい場合は一回ログインページにリダイレクトさせるのがいいと思うのですが、今回に関しては可能な限りゆるふわな作りにしたかったので、登録即認証って感じにしようかなと。

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

    try {
      Auth::login($user, $remember = true);
    } catch (Throwable $e) {
      report($e);
      return false;
    }

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
     * ログイン中ユーザー取得
     *
     * @return User
     */
    public function get_auth_user(): User
    {
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
    }
```

















