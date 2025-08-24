---
title: "Amazon Cognito User Poolを本番導入するときは、メールアドレスの文字長制限に注意"
emoji: "⚠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cognito", "email", "authentication"]
published: true
---

## 背景

Amazon Cognito User Pool という認証認可を肩代わりしてくれるサービスをユーザー向けに本番導入した。

いざリリースとなり、テストケースを作っていると「このサービスって email の文字制限はどうなってるんだっけ?」となり検証のメモを残す。

## 前提

### CognitoのAdmin Create Userを使ってユーザー登録を行うこと。

開発時は、Hosted UI経由の登録がメインでなく、API経由で登録されるサービスだった。

このようなユースケースは、誰でも登録できない管理画面系の開発では多いと思う。 (特にエンタープライズ系)

### サインイン識別子のオプションが、メールアドレスで設定されていること。

![サインイン識別子のオプション設定画面](/images/20250824-cognito-long-email/cognito-signin-options.png)

:::message
※ Hosted UIでEmailによるログインをさせたい場合は、上記の設定でないとできなかった。
※ サインイン識別子のオプション -> ユーザー名、 サインアップのための必須属性 -> メールアドレスだと、そもそもHosted UIでEmailログインができない。(試した限りは)
:::

:::message
User Poolを一度作成するとこの属性は変えられないので、注意すること。
:::

terraformでの設定方法は下記。

```hcl
resource "aws_cognito_user_pool" "example" {
  name = "example-user-pool"
  username_attributes = ["email"]
}
```

## 発生事象

### Admin Create Userでは 128 文字を超えるEmailを登録できない。

サインイン識別子のオプション -> メールアドレス(`username_attributes = ["email"]`)に設定すると、User Pool内部では usernameが自動的にUUIDに変換され、emailアドレスがサインイン識別子として機能する。

しかし**Admin Create User APIのusernameパラメータは128文字上限**のため、254文字(RFC 5321上限)のemailアドレスを直接指定して登録することはできない。

```bash
# 254文字のemail(RFC上限)のEmailを作成
aws cognito-idp admin-create-user \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username "$(printf 'a%.0s' {1..242})@example.com" \
  --user-attributes Name=email,Value="$(printf 'a%.0s' {1..242})@example.com"

# エラー: Member must have length less than or equal to 128
```

## 回避方法

### UpdateUserAttributes を使い、登録後に手動更新する

上記事象のため、(API経由でユーザー登録する場合) まずはダミーemailで登録してから、更新をかける。

```bash
# 1. 短いemailで作成
aws cognito-idp admin-create-user \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username "temp@example.com" \
  --user-attributes Name=email,Value="temp@example.com" \
  --temporary-password "TempPass123!" \
  --message-action SUPPRESS

# 2. 254文字emailに更新（email_verifiedと同時設定が必須）
aws cognito-idp admin-update-user-attributes \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username [UUID] \
  --user-attributes \
    Name=email,Value="$(printf 'k%.0s' {1..242})@example.com" \
    Name=email_verified,Value=true # email_verified をtrueにしないと更新されないので注意
```

注意事項としては

1. email_verified をtrueにしないとEmailが更新されないので注意
1. 長いemailアドレスの検索はAdmin Get User APIの検索文字列長上限(128文字)に引っかかるため、subを指定するか、List Users APIで前方一致検索をかける必要がある

## まとめ

そもそも254文字近くのemailは(通常の開発では)ほぼ存在しないはず、、、

※ローカルパートが64文字上限なのでそもそも100文字程度にしかならない

なのでCognito導入時は、Email文字長上限をusernameと同等に128文字で組むのが良いだろう。

## 感想

Cognitoの導入理由としては、AWSに閉じられること、価格が安いこと、などあるが細かいAPIセットのインターフェースにハマりどころが多く、やはり価格なりなのかなと感じてしまった。

しかし最近はHosted UIの日本語対応など、大きなアップデートもあるため、今後に期待したい。

以上。

## 補足

間違いあればご指摘お願いします。