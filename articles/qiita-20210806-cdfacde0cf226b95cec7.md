---
title: "Laravel-Excelで大容量のエクセルファイルをエクスポートする際の対策"
emoji: "📂"
type: "tech"
topics: [PHP,S3,Laravel,queue,Laravel-Excel]
published: true
---
## 背景
Laravelには特定のモデルのレコードをエクセル形式で読み書きしてくれる[Laravel-Excel](https://laravel-excel.com/)というライブラリが存在する。

しかし多くのレコードをエクスポートすると処理時間が長引き、クレームやユーザーの離脱等が発生すると考えられる。

その際の対策をサンプルコードも含めてメモ。

## 大容量ファイルのエクスポート

### 方針

1. 処理をLaravelの[Queue](https://laravel.com/docs/8.x/queues)に入れる
2. エクスポート処理
3.  エクスポートが終わったらエクセルファイルをS3にアップロード
4.  ユーザーにダウンロードリンクをメール通知

### 実装

※本記事ではLaravel-Excelのインストール方法は対象外とします。

以下のartisanコマンドを実行

```shell
php artisan make:export UsersExport --model=User
```

下記のようなファイルが生成される。

```php:backend/app/Exports/UsersExport.php
<?php

namespace App\Exports;

use App\Models\User;
use Illuminate\Support\Collection;
use Maatwebsite\Excel\Concerns\FromCollection;

class UsersExport implements FromCollection
{
    /**
    * @return Collection
    */
    public function collection(): Collection
    {
        return User::all();
    }
}
```
S3にアップロードするためのライブラリを追加

```shell
composer require --with-all-dependencies league/flysystem-aws-s3-v3 "^1.0"
```

以下のartisanコマンドを実行し、DBにキューを管理するテーブルを追加

```shell
php artisan queue:table
php artisan migrate
```

キューに入れるジョブを定義

```shell
php artisan make:job NotifyUserOfCompletedExport
```

エクスポート完了時に送信する通知(Notification)を定義

```shell
php artisan make:notification ExportCompleted
```

通知が送信されるようにジョブを編集する。

```php:app/Jobs/NotifyUserOfCompletedExport.php
<?php
declare(strict_types=1);

namespace App\Jobs;

use App\Models\User;
use App\Notifications\ExportCompleted;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

/**
 * Class NotifyUserOfCompletedExport
 * @package App\Jobs
 */
class NotifyUserOfCompletedExport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * @var User
     */
    private User $user;

    /**
     * @var string
     */
    private string $fileName;

    /**
     * Create a new job instance.
     *
     * @param User $user
     * @param string $fileName
     */
    public function __construct(User $user, string $fileName)
    {
        $this->user = $user;
        $this->fileName = $fileName;
    }

    /**
     * Execute the job.
     *
     * @return void
     */
    public function handle()
    {
        $this->user->notify(new ExportCompleted($this->fileName));
    }
}

```

通知にS3のリンクを含めるように修正する。

```php:app/Notifications/ExportCompleted.php
<?php
declare(strict_types=1);

namespace App\Notifications;

use App\Http\Controllers\UserController;
use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;
use Illuminate\Support\Facades\Storage;

/**
 * Class ExportCompleted
 * @package App\Notifications
 */
class ExportCompleted extends Notification
{
    use Queueable;

    /**
     * @var string
     */
    private string $fileName;

    /**
     * Create a new notification instance.
     *
     * @param string $fileName
     * @return void
     */
    public function __construct(string $fileName)
    {
        $this->fileName = $fileName;
    }

    /**
     * Get the notification's delivery channels.
     *
     * @param mixed $notifiable
     * @return array
     */
    public function via($notifiable): array
    {
        return ['mail'];
    }

    /**
     * Get the mail representation of the notification.
     *
     * @param mixed $notifiable
     * @return MailMessage
     */
    public function toMail($notifiable): MailMessage
    {
        $s3 = Storage::disk('s3');

        return (new MailMessage)
            ->line('Export has been completed.')
            ->line('Please click link to download a exported file.')
            ->action('Download ' . $this->fileName, $s3->url($this->fileName));
    }

    /**
     * Get the array representation of the notification.
     *
     * @param mixed $notifiable
     * @return array
     */
    public function toArray($notifiable): array
    {
        return [
            //
        ];
    }
}

```

Controllerクラスで定義したジョブを呼び出す。
※通常はサービス層などをで処理を分離すべきかと思われます。

```php:UserController.php
    public function queue(): View|RedirectResponse
    {
        try {
            DB::beginTransaction();
            Excel::queue(new UsersExport, 'users.xlsx', 's3')->chain([
                new NotifyUserOfCompletedExport(
                    request()->user() ?? User::factory()->create(),
                    'users.xlsx'
                )
            ]);

            DB::commit();
        } catch (Throwable $e) {
            DB::rollBack();
            logger()->error($e);
            return redirect(route('users.excel.export.download-form'))
                ->withErrors($e->getMessage());
        }

        $message = 'Successfully queued an export job';
        return \view('index', compact('message'));
    }
```


#### 動作確認

[mailtrap](https://mailtrap.io/)に登録し、`.env`に必要な情報を入力する。

```.env:.env
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=********
MAIL_PASSWORD=********
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=test@test.com
MAIL_FROM_NAME="${APP_NAME}"
```

S3のバケットにアップロードするためAWSのクレデンシャル等を`.env`に追加

```.env:.env
AWS_ACCESS_KEY_ID=********
AWS_SECRET_ACCESS_KEY=********
AWS_DEFAULT_REGION=********
AWS_BUCKET=********
AWS_USE_PATH_STYLE_ENDPOINT=false
```


適当なBladeファイルを定義しキューエクスポート用のリンクを追加

```html:index.blade.php
@extends('layouts.app')

@section('content')
    <section>
        <h1>Laravel/Excel Sample app</h1>
        <div>
            <a href="{{route('users.excel.queue')}}">Add a job of export user models in queue</a>
        </div>
    </section>
@endsection
```

`localhost/users/excel`にアクセスし「Add a job of export user models in queue」リンクをクリック
画面上部に成功メッセージが表示されることを確認。

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/290859/59a7f34b-9837-2682-d3c5-7db55f50d81b.png" width=50%>

mailtrapでファイルダウンロードリンク付きのメールが送信されることを確認する。
※ファイルダウンロードのためにはバケットの公開範囲を設定してください

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/290859/1dc89cfe-a14e-26a6-e115-d29df95c9b6f.png">

## 結言

下記のようにQueueを使う際にファイルストレージが指定できることは公式ドキュメントから読み取れないので注意

```php
Excel::queue(new UsersExport, 'users.xlsx', 's3');
```

## 謝辞

実装方針共有いただいた先輩ARIGATOH

## 補足

※記事へのご指摘歓迎します。

