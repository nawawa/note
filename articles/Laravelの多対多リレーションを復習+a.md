久しぶりにLaravelで多対多リレーションをやったので、復習＋ついでに学習したことをまとめます。

## 一般的なモデルの設定
2つのモデルと、関連付け用の中間モデル。普通の形です。

- Library（図書館）
  - id

- Book（書籍情報マスタ）
  - id

- LibraryBook（蔵書）
  - library_id
  - book_id

各モデルは`php artisan make:model`で作成されています。

### 関連付け

図書館モデルからそこの蔵書リストを作りたい、という場合は、LibraryモデルにbelongsToManyメソッドを記述するだけで可能になります。

```Library.php
 /**
  * 蔵書一覧を取得
  *
  * @return void
  */
  public function books()
  {
     return $this->belongsToMany(Book::class);
  }
```

```php
$library->books; // 全取得
$library->books()->where('title', $request->input('title'))->get(); // 絞り込み
```

特定の本が所蔵されている図書館のリストを作る場合でも全く同様です。

```Book.php
 /**
  * 蔵書一覧を取得
  *
  * @return void
  */
  public function libraries()
  {
      return $this->belongsToMany(Library::class);
  }
```

```php
$book->libraries; // 全取得
$book->libraries()->where('name', $request->input('name'))->get(); // 絞り込み
```

## Pivotでもうちょい便利にする

`php artisan make:model`で生成されたモデルクラスは、必ず`Model`クラスを継承したものとなります。

多対多リレーションの中間モデルに、ModelではなくPivotを継承したPivotモデルクラスを使用すると、より簡単に応用的なリレーションを組めるようになります。

`php artisan make:model LibraryBook --pivot`で生成します。

### Pivotモデルクラスにするとどう便利なのか
今回の例では「図書館」と「蔵書」の関係ですが、例えば中間モデルが「貸出中」という状態を持つとします。

- LibraryBook
  - library_id
  - book_id
  - lending （trueが入ると「貸出中」）

先程の蔵書一覧取得のような形で「貸出中の蔵書一覧」を取得したい、という場合は、このようなリレーションを組むことができます。

```Library.php
 /**
  * 貸出中の蔵書一覧を取得
  *
  * @return void
  */
  public function lendingBooks()
  {
     return $this->belongsToMany(Book::class)
      ->wherePivot('lending', true);
  }
```
`where`メソッド同様、カラムと値を指定して絞り込むことができます。

```php
$library->lendingBooks; // 貸出中の蔵書を取得
```
他にどんなメソッドが使えるかは[こちら](https://readouble.com/laravel/8.x/ja/eloquent-relationships.html#filtering-queries-via-intermediate-table-columns)から。

### これをやるのに必要な作業

### 補足：ルートモデル結合で親子モデル両方を取得するとき

ルートモデル結合でこのように書いたとき、

```php
Route::get('/library/{library}/book/{book}/'....
```
指定のlibraryの蔵書内にbook_idがあるかどうかを自動的に参照してもらえますが、もしこのときテーブル名を`library_books`にしているとエラーになります。ルートモデル結合の処理時には、2つのモデル名がアルファベット順に連結されたテーブル名を参照しようとするためです。

今回の例だと`book_libraries`のようなテーブル名なんだな、と判断されてしまうため、「そんなテーブルないじゃねーか」というエラーが出ます。

もちろんこれは簡単に回避できます。belongsToManyメソッドの第二引数にテーブル名を入れるだけで参照先が上書きされます。

```Library.php
 /**
  * 貸出中の蔵書一覧を取得
  *
  * @return void
  */
  public function lendingBooks()
  {
     return $this->belongsToMany(Book::class, 'library_books')
       ->wherePivot('lending', true);
  }
```

## リレーションについて理解しておくべきこと
中間テーブルの値を使ってリレーション先の状態を定義する場合、テストコード等でちょっとハマりました。ちょっと考えればわかったことかもしれませんが、まあ筆者はアホなんだなあと

### 遅延読み込み
さて筆者は、貸出中の書籍だけを取り出すためのこんなリレーションを定義しました。
```Library.php
    /**
     * 貸出中の蔵書を取得
     *
     * @return void
     */
    public function checkedOutBooks()
    {
        return $this->belongsToMany(Book::class, 'library_books')
            ->using(LibraryBook::class)
            ->withPivot('checked_out')
            ->wherePivot('checked_out', true);
    }
```
中間テーブルの`checked_out`カラムが`true`なら貸出中という（最低限すぎる）仕様です。

この記述によって、こう書くことで貸出中の蔵書だけ取得できるようになったわけですが、
```php
$library->checkedOutBooks;
```
返却処理のテストの中で、返却処理の前後で貸出中の蔵書を数えるアサーションをいれました。

```php
$this->assertCount(3, $library->checkedOutBooks); // 事前処理で3冊の貸出を行っている

// ~~~~~~返却処理を実行~~~~~~

$this->assertCount(0, $library->checkedOutBooks);
```
筆者は、こう書けばいい感じにテーブルの状態を比較できると思っちゃったわけです。<br>
でも違います。処理の前後で`$library->checkedOutBooks`の中身はなにも変わっていません。

### 一度取ってきたらもう取らない
クエリログを出力するメソッドを使って実験したところ、こんな結果でした。

**処理前に出力**
```php
\DB::enableQueryLog();

$library->checkedOutBooks;

dump(\DB::getQueryLog());

// 出力されたクエリ

/**
 * array:1 [
 *   0 => array:3 [
 *     "query" => "select `books`.*, `library_books`.`library_id` as `pivot_library_id`, `library_books`.`book_id` as `pivot_book_id`, `library_books`.`checked_out` as `pivot_checked_out` from `books` inner join `library_books` on `books`.`id` = `library_books`.`book_id` where `library_books`.`library_id` = ? and `library_books`.`checked_out` = ?"
 *     "bindings" => array:2 [
 *       0 => 6
 *       1 => true
 *     ]
 *     "time" => 1.3
 *   ]
 * ]
 */

$this->createCheckedOutBooks($request_books); // 3冊の貸出を行う
$this->assertCount(3, $library->checkedOutBooks); // 3ではなく0。通らない
```
**処理後に出力**
```php
$this->createCheckedOutBooks($request_books); // 3冊の貸出を行う
$this->assertCount(3, $library->checkedOutBooks); // 通る

\DB::enableQueryLog();

$library->checkedOutBooks;

dump(\DB::getQueryLog());

// 出力されたクエリ

/**
 * []
*/
```
このような結果に。<br>
つまり、初回はDBを参照して値を取得するが、その後は取得済みの値を参照する仕様になっているということのようです。

一度取得した値は、単にDBから取り出した情報がメモリ上に格納されているに過ぎないので、DBの状態が変わっても取得済みの値が変わってるはずありません。当たり前ですね。

ちなみに、Readouble.comのリレーションのページには、`プロパティとしてEloquentリレーションへアクセスすると、関連するモデルは「遅延読み込み」されます。`と記述されています。つまり、
```php
$library = Library::find(1);
```
が実行された時点では、Libraryのリレーション先は読み込まれていません。
```php
$library = Library::find(1);
$library->checkedOutBooks;
```
2行目まで来て、初めてリレーション先を読み込むクエリが発行されます。<br>
つまりデフォルトでは、
```php
$library = Library::find(1); // 読み込まない
$library->checkedOutBooks; // 読み込む->保存される
$library->checkedOutBooks; // 読み込まない
```
こうなるように実装されており、なるべく余計なSQLを発行しないような作りになっていると。

ただReadouble.comで紹介されている通り、このまま`foreach ($library->books as $book)`してしまうと、一冊ごとにクエリを発行して値を取ってきます。N+1問題発生。

筆者は`$library`の取得にはすべてルートモデル結合を使っているので、`RouteServiceProvider`内で`with`メソッドを付けることでこれを防止します。

```RouteServiceProvider.php
    public function boot()
    {
        + Route::bind('library', function ($id) {
        +     return Library::with('books')->findOrFail($id);
        + });
```