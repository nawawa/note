Readouble.comもそろそろ読み慣れたと思っていたのですが、意外にAPIリソースのページについてはやや読みづらさを感じるところがあったので、ポートフォリオ作成を通して学んだLaravelにおけるAPIリソースのTipsをまとめてみようと思います。

## APIリソースで解決できる課題
- **フロントで使いやすい形に整形した上でデータを返却したい**
- **他のデータと抱き合わせて返却する際、同じ整形方法を流用した上で抱き合わせたい**

例えば、個別の「記事」ではなく「記事一覧」を取得してくる場合には、いちいち`created_at`などまで含め全てのカラムを返却する必要はないこともあるでしょう。あるいは、同じテーブル内の情報であったとしても「タイトル」と「それ以外」のように、ある程度情報をカテゴリー分けして配列を入れ子させたいこともあるかもしれません。

そういった様々な面で考えられる「フロント側で使いやすい配列の形式」を定義できるのが、LaravelのAPIリソースです。

## Resourceクラスにモデルインスタンスをぶっこむ

一番シンプルな用途はこれです。

```
php artisan make:resource UserResource
```

コマンドでリソースクラスを追加し、配列の雛形を作ります。

```php
class UserResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array|\Illuminate\Contracts\Support\Arrayable|\JsonSerializable
     */
    public function toArray($request)
    {
        return [
            'data' => [
                'login_id' => $this->login_id,
                'attribute' => [
                    'name' => $this->name,
                    'icon' => $this->icon_path,
                ],
            ],
        ];
    }
}
```

例えばですが、こんなふうに。
フロント側が受け取るUserモデルのプロパティは`login_id`と名前とアイコン画像のパスだけでとりあえず十分だという場合のリソースです。

まず便利なのは、このように自由に配列を入れ子できるという点です。
オブジェクトをそのまま配列化しただけでは、こんなふうに、ただ単に連想配列が一列に並ぶだけになります。

```php
id: 1,
uuid: "210b4581-c769-4c7c-b38c-39b727071b90",
name: "宇野 花子",
login_id: "hamada.asuka",
email_verified_at: "2022-03-02 14:17:15",
icon_name: "default_icon.png",
icon_path: "storage/default_icon.jpg",
created_at: "2022-03-02 14:17:15",
updated_at: "2022-03-02 14:17:15",
deleted_at: null,
```

そのままでは、例えばフロント側には名前だけ返却されればいい場面であったとしても、無意味に全てのカラムが返却されてしまいます。
Eloquentで名前だけ取得することもできるかもしれませんが、リソースクラスで定義された雛形のほうが、ある要件にのみ対応するクエリよりも再利用性が高いケースは考えられるでしょう。

もしすべてのカラムの値が必要なこともある場合でも、それはそれで別のリソースクラスを定義すればいいわけです。このように一つのモデルに対していくつも雛形を作れ、様々な場所から使い回すことができるという点が便利です。

使う時はnewして引数にオブジェクトを渡し、そのまま返却します。

```php
use App\Http\Resources\UserResource;

$user = User::find($id);

return new UserResource($user);
```

## モデルのコレクションをリソースクラスへぶっこむ

やたらと「コレクション」という言葉が飛び交うのがドキュメントのわかりづらいところでもあるのですが、「モデルのコレクション」というのは、データベースから`get()`や`all()`などでデータを複数取得した場合に返却される`EloquentCollection`であったり、`collect`メソッドで作成する`Collection`インスタンスのことです。

先ほどのリソースクラスは、単体のオブジェクトを引数に渡すことで配列化できるものでした。
リソースクラスで定義した雛形をそのままに、複数のオブジェクトを配列化するときに用いられるのが`collection`メソッドです。

```php
use App\Http\Resources\UserResource;

$users = User::where('created_at', '>=', '2020-01-01')->get();

return UserResource::collection($users);
```

`where`句で絞り込んだレコードを`get`した場合、`first`や`find`で取得した際とは異なり、フロントへ返却するオブジェクトは複数である場合も考えられます。
そうした場合に、全てのオブジェクトをリソースクラスで定義した雛形通りに整形するために用いるのが`collection`メソッドです。

## モデルのコレクションをリソースコレクションへ渡し、独自に配列を定義する

配列化されたオブジェクトといっしょに任意の値（レスポンスメッセージとか）を返却する場合や、異なるモデルのオブジェクトをまとめて返却したい場合に用いるのがリソースコレクションです。

モデルのコレクションとその他の値を返却する場合は、まずこのようなコレクションを作成し、

```php
$response = collect([
  'users' => User::all(),
  'message' => '正常に取得されました。'
]);
```

このコレクションインスタンスを、リソースコレクションへ渡します。

```php
return new UserCollection($response);
```

リソースコレクションの中で、渡されたコレクションインスタンスから値を取り出して配列を作ります。

```php
use App\Http\Resources\UserResource;

class UserCollection extends ResourceCollection
{
    /**
     * Transform the resource collection into an array.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array|\Illuminate\Contracts\Support\Arrayable|\JsonSerializable
     */
    public function toArray($request)
    {
        return [
          'users' => UserResource::collection($this->collection['users']),
          'message' => $this->collection['message'],
        ];
    }
}
```

リソースクラスの場合は、`$this`で直接オブジェクトのプロパティにアクセスしていましたが、リソースコレクション内では`$this`のプロパティに渡されたコレクションが入るという形になります。

もし配列化するコレクションがモデルのコレクションだけなのであれば、直接`collection`メソッドの引数に`$this->collection`を渡すだけでもOKです。

```php
return new UserCollection(User::all());
```

```php
use App\Http\Resources\UserResource;

class UserCollection extends ResourceCollection
{
    /**
     * Transform the resource collection into an array.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return array|\Illuminate\Contracts\Support\Arrayable|\JsonSerializable
     */
    public function toArray($request)
    {
        return UserResource::collection($this->collection);
    }
}
```

## まとめ

最初に触った際にはなんとなく掴みかねたのですが、実際の理解としてはこのようなものなのだと思います。

- 返却するモデルインスタンスが単数 ▶︎ リソースクラス
- 返却するモデルインスタンスが複数
  - 返却するのはモデルインスタンスのみ ▶︎ リソースクラスの`collection`メソッド
  - メッセージなどその他の要素もまとめて返す ▶︎ リソースコレクション
- 返却するモデルが複数種類（リレーション元・リレーション先をまとめて返却するなど）▶︎ リソースコレクション

`mergeWhen`など、条件によってプロパティを入れたり入れなかったりできるメソッドも実装されています。
詳しくは[Readouble.com](https://readouble.com/laravel/8.x/ja/eloquent-resources.html)を御覧ください。

また今回の内容はあくまで正常系のレスポンスのみを考慮しており、エラー発生時のレスポンスなどは別途例外処理にてアレコレする必要があるかと思います。
筆者にエラーハンドリング周りの知識が足りていないので、そのへんは別途記事を書くか、こちらに後日加筆いたします。





















