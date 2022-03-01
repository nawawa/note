久しぶりなもので、ちょっと忘れちゃいまして。
いろいろ調べながらユーザー登録・認証周りの処理を実装します。

なおフロントエンド側はSPA(ないし別でインフラを持つNuxtアプリケーション)を想定していますが、そっち側で行う動作については~~無計画~~あんまり考慮してません。JsonResponseにリンクを含めるとかすればいいのかな。各自お願いします。

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

任意のユーザーアイコンを設定できるが、特に必須でもないという仕様にしたかったので、入力値にアイコンの指定がなかったら`/storage/app/public/images`に配置してあるデフォルトアイコンを使用するようにしています。

アイコンのパスは定数クラスを作成して管理しています。

### 登録したらすぐにログイン状態にする

ここをどうするかはサービス設計次第という感じもします。
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

Authファサードのloginメソッドにユーザーオブジェクトを渡せばログイン済みにしてくれて、かつ第二引数にtrueを渡すと`remember me`にチェックが入る。らくっすね。

fillからのsaveを行う箇所は、後でRepositoryクラスでも作ってそちらへ移植しようと思います。
エラーハンドリングに関してはまとも意識したことがなく、どんなふうにしたらいいのか全然わかりません。超なんとなくのトライキャッチです。いかにも初心者っぽいですね。一度には勉強できないので後回ししますけどね。

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
        $this->assertAuthenticatedAs(User::where('name', $this->user_param['name'])->where('login_id', $this->user_param['login_id'])->first());
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

## ログイン・ログアウト

これ、Sanctumで独自ログインにするか、Google等のソーシャルログインにするか一考の余地がありますね。Google認証のほうがセキュリティ的にこちらが負うべき責任が減るのでうれしいのですが、どこの馬の骨かわからんやつのサービスなんかに大切なGoogleIDを渡してたまるかい、という人情もあるわけで。

ここはおとなしくSanctumにしておきますかね。