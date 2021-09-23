---
title: "【Laravel 8】Bulk insertはEloquent::upsertメソッドが便利"
emoji: "👨‍👩‍👧‍👦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [PHP,Laravel]
published: true
---

# 背景

Laravel 7までは複数のデータを同時にINSERTするには`DB`ファサードの`insert`メソッドで対応する必要がある。

しかしLaravel 8で[`Eloquent::upsert`](https://laravel.com/docs/8.x/eloquent#upserts)というイケてるメソッドが実装されたのでメモ。

# 使い方

## 基本

[usersテーブル](https://github.com/yasuaki640/laravel-excel-sample/blob/for-qiita-09-18/backend/database/migrations/2014_10_12_000000_create_users_table.php)を考える。  
複数レコードのinsertでは以下のように書ける。

```php
    $data = [
        ['name' => 'Ruffy', 'email' => 'ruffy@ruffy.com', 'sex' => User::SEX_MALE, 'password' => \Hash::make('password')],
        ['name' => 'Tonny', 'email' => 'tonny@tonny.com', 'sex' => User::SEX_MALE, 'password' => \Hash::make('password')],
        ['name' => 'Robin', 'email' => 'robin@robin.com', 'sex' => User::SEX_FEMALE, 'password' => \Hash::make('password')],
    ];

    User::upsert($data, ['email']);
```

第2引数にはDBで設定されたプライマリーキーかユニークキーを指定する(SQL Server以外)。

> All databases systems except SQL Server require the columns in the second argument provided to the upsert method to have a "primary" or "unique" index.

、、、とドキュメントには書いてあるMySQLの場合は違うようで、パラメータを渡さずともDBで設定されたユニークキーをもとにinsertが走る。[^1]

## insertメソッドとの比較

### timestamp系のカラムを更新してくれる

`DB::insert`では`created_at`,`updated_at`に何も指定しなければnullが入るが`Eloquent::upsert`は指定せずとも現在時刻が入る。

```php
    // DB::insert
    DB::table('users')->insert(['name' => 'Robin', 'email' => 'robin@robin.com', 'sex' => User::SEX_FEMALE, 'password' => \Hash::make('password')],);
    $this->assertDatabaseHas('users', [
        'name' => 'Robin',
        'created_at' => null, // timestampにnullが入る
        'updated_at' => null
    ])

    // Eloquent::upsert
    User::upsert(['name' => 'Ruffy', 'email' => 'ruffy@ruffy.com', 'sex' => User::SEX_MALE, 'password' => \Hash::make('password')], ['id', 'email']);
    $this->assertDatabaseHas('users', [
        'name' => 'Ruffy',
        'created_at' => now(), //timestampに現在時刻が入る
        'updated_at' => now()
    ]);
```

### 同一ユニークキーのレコードはupdateしてくれる

第1引数に渡したパラメータに同一IDもしくは同一ユニークキーのレコードが存在すれば、該当レコードが更新される。(`updated_at`もよしなに更新される)

```php
    User::factory()->create(['name' => 'Robin', 'email' => 'robin@robin.com', 'sex' => User::SEX_MALE]);

    $updatedParams = ['name' => 'Nico Robin', 'email' => 'robin@robin.com', 'sex' => User::SEX_FEMALE,'password' => \Hash::make('password')];

    User::upsert($updatedParams, ['id', 'email']);
```

また題3引数にはupdate時に更新されるカラムを絞ることができる。

```php
    User::upsert($updatedParams, ['id', 'email'], ['name']);
```

## ベンチマーク

1万件のデータをinsertすると処理時間は以下の様になった。  
(ここでは10回実行した平均をとっている。)

```
DB::insert処理時間:       0.2898751826秒  
Eloquent::upsert処理時間: 0.3419602284秒
```

`Eloquent::upsert`のほうが約1.18倍時間がかかる結果に。  
微妙に遅いが許容範囲内？要求されるレスポンスタイム次第では使い分ける必要があるのだろうか、、

# 結言

というわけで複数レコードinsert時は`Eloquent::upsert`を使っていきやしょう。  
([サンプルコード](https://github.com/yasuaki640/laravel-excel-sample/blob/for-qiita-09-18/backend/tests/Feature/UserRepostitoryTest.php)おいてあります。)  
(記事へのご指摘歓迎です。)


[^1]: [insert文を発行するメソッド](https://github.com/laravel/framework/blob/8.x/src/Illuminate/Database/Query/Grammars/MySqlGrammar.php#L155-L175)では[`ON DUPLICATE KEY UPDATE`](https://dev.mysql.com/doc/refman/5.6/ja/insert-on-duplicate.html)構文が使われているが、これはそもそもユニークキーやプライマリーキーを指定する必要がない。
