## 
結論としては、
レコードの特定にはユニークIDを使う
中間テーブルでは自動連番を使う

## 方法
$fillableに`uuid`を追加する
```php
public function getRouteKeyName()
{
    return 'uuid';
}
```
ルートモデル結合を行えるように対策する

withを使って積極的ロードを行うなど、単なるモデルの取得以上にクエリを叩くなら別途RouteServiceProvider内で対策の必要あり
```php
Route::bind('library', function ($id) {
  return Library::with('books')->where('uuid', $id)->first();
});
```

連番は連番で残すので、マイグレーションファイルは`$table->uuid('uuid')->unique();`を追記するのみ。
ただレスポンスでは連番を知られないようにするため、モデルに`$hidden=['id']`を追加する。プロパティへの直アクセスでは取得できるが、配列等の形になったときにはIDが表示されなくなる。

テストコードやフロントエンドでも、直接連番のIDを使用することはなくす。
リクエストボディからIDを受け取ってDBを叩くようなコードは全て修正が必要になる。

