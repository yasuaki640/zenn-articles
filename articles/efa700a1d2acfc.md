---
title: "TypeScriptで簡易DBを実装する"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,nodejs,sqlite,db]
published: false
---

# はじめに

バックエンドエンジニアとして3年目に入ると、インフラ構築やDBREとの折衝など言語の知識だけでは対処できない業務を任されることが増えてきた、、、

そこでDBや低レイヤについて知見を深めるべく、2か月前ほどに読了した [Let's Build a Simple Database](https://cstack.github.io/db_tutorial/)をTypeScriptで書き直すことにしました。

# 作ったもの

実際の実装がこちら。

https://github.com/yasuaki640/tsqlite

SQLiteのようなファイルベースのDBで、userテーブルの`insert`と`select`しかできません。

元ネタのPart 5(ディスクへの永続化)までの実装となっており、B-Treeの導入はスコープ外としました。

```shell
$ npm start mydb.db

> tsqlite@0.0.0 start
> node build/src/main.js mydb.db

db > insert 1 mask eron@twitter.com
Executed.
db > insert 2 trump donald@fakenews.co.jp
Executed.
db > select
(1, mask, eron@twitter.com)
(2, trump, donald@fakenews.co.jp)
Executed.
```

# 学んだこと

## 他言語への移植の学習効果

実装のきっかけとなった記事にも書かれていますが、何かしらのコードを理解する際、コピペやただ写経するだけだと理解が薄くなりがちでした。

これが他言語への移植となると、元ネタのコードを詳細に理解しさらにC言語とTypeScript(Node.js)両方のAPIについて詳しくなければなりません。

例えばファイルシステム回りのAPIについて、C言語の`open()`とNode.jsの`fs.open`、Nodeの[CLI用便利メソッド](https://nodejs.org/api/readline.html#rlquestionquery-options)、など両方のライブラリについて知見が深くなりました。

デメリットは時間がかかることです。

## キャッシュの仕組み

元ネタの実装にはデータのキャッシュ機構が存在し、ページが要求されて初めてファイルストレージからメモリにマッピングする、デマンドページング的な処理を実装しています。

ここではファイルの全体ではなく**必要な部分だけ**読み込むなど少ないリソースで大量のデータを高速に扱う工夫が凝らされており、自身で実装しなおすことでキャッシュの仕組みを感じることができました。

デメリットは普段の開発で使わなかったAPIの仕様を読み込まないといけないので時間がかかることです。

## JavaScript(Node.js)でも意外とバイナリは扱える

JSはメモリに直接アクセスできませんが、[`Buffer.alloc`](https://nodejs.org/api/buffer.html#static-method-bufferallocsize-fill-encoding)を使えば任意のサイズのバイナリを生成し疑似的なメモリとして扱うことができます。

またC言語だと`lseek`等の関数でファイルの読み書き位置を指定できますが、Nodeだと[`fs.read`](https://nodejs.org/api/fs.html#fsreadfd-buffer-offset-length-position-callback)で読み取り開始位置を指定することができます。

上記のようにC言語に相当する`malloc`や`open`等のAPIは知らなかっただけで一通りは用意されてると感じました。

## 車輪の再発明は楽しい

普段何気なくライブラリをダウンロードして使っていますが、その裏には地道なエラーハンドリングやバリデーション、メモリ管理など膨大な処理が隠れていることに気づきました。

例えば対話的な処理を実装するにも、使い慣れないNode.jsのAPIに詳しくなる必要があったり、文字列を地道に解析しバリデーションする必要があったり、、、言語機能やAPIをフル活用するため**必然的に言語に詳しくなります。**

意外だったのは、JSでDBを実装するという「さすがにそれは向いてないんじゃないの(笑)」みたいな技術選定だと、言語機能やあまり使わないようなAPIをフル活用するため**必然的に言語に詳しくなります。**

# 終わりに

今回はTypeScriptで超簡素なDBを実装してみました。
ご意見、間違いありましたらコメントお願いします。


- この記事で伝えたいこと
  - 低レイヤは面白い
  - **低レイヤの学習はエンジニアのレベルを一段階挙げる**
    - アプリ側の知識だけで太刀打ちすることしんどくなってきた
  - 仕組み系の学習は他言語で書き直すのがおすすめ
  - 大変だった部分
  - 次の学習へどう生かすか
    - メモリに直接アクセスできる言語へのりかえ
  - どんな学びがあったか
    - nodejsにもsys call呼び出しのapiがあり、アプリとosの相互作用を実感した
    - fsについて部分的にファイルを読み込むことができることを知った
    - jsでもバイト列を扱うことができるが

