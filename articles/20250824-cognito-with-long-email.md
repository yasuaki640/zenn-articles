---
title: "Amazon Cognito User Poolを使うときは、長いメールアドレスに注意"
emoji: "📧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cognito", "email", "authentication"]
published: false
---

## 背景

AWS Cognitoでユーザー管理を行う際、長いメールアドレスを持つユーザーの登録で予期しない制限に遭遇することがあります。本記事では、実際にAWS CLIを使って検証した結果をまとめ、Cognitoの文字数制限とその回避方法について解説します。

## 発見した制限

### UsernameAttributes設定による動作の違い

Cognito User Poolの`UsernameAttributes`設定により、メールアドレスの扱いが大きく変わります。

#### 従来型（UsernameAttributesなし）
```hcl
resource "aws_cognito_user_pool" "example" {
  name = "example-user-pool"
  
  # username_attributes は指定しない（デフォルト）
}
```
- usernameとemailが別々に管理される
- 長いemailアドレスでも直接作成可能
- サインイン時はusername/email両方使用可能

#### モダン型（UsernameAttributes: ["email"]）
```hcl
resource "aws_cognito_user_pool" "example" {
  name = "example-user-pool"
  
  username_attributes = ["email"]
}
```
- emailがサインイン識別子として機能
- usernameは自動生成UUID
- **128文字制限が適用される**

### API別の制限一覧

| API | 制限内容 | 最大文字数 |
|-----|---------|-----------|
| AdminCreateUser | username parameter | 128文字 |
| AdminUpdateUserAttributes | email属性 | 254文字 |
| AdminGetUser | username parameter | 128文字 |
| ListUsers (filter) | filter文字列全体 | 256文字 |

## 実際の検証結果

### 128文字超のemail作成試行

```bash
# 254文字のemailで作成試行 → 失敗
aws cognito-idp admin-create-user \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username "$(printf 'a%.0s' {1..242})@example.com" \
  --user-attributes Name=email,Value="$(printf 'a%.0s' {1..242})@example.com"

# エラー: Member must have length less than or equal to 128
```

### 回避方法：UpdateUserAttributesを使用

```bash
# 1. 短いemailで作成
aws cognito-idp admin-create-user \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username "temp@example.com" \
  --user-attributes Name=email,Value="temp@example.com"

# 2. 254文字emailに更新（email_verifiedと同時設定が必須）
aws cognito-idp admin-update-user-attributes \
  --user-pool-id ap-northeast-1_E8sXPyQA5 \
  --username [UUID] \
  --user-attributes \
    Name=email,Value="$(printf 'k%.0s' {1..242})@example.com" \
    Name=email_verified,Value=true
```

### 重要な発見

1. **email単独更新は失敗**：`UsernameAttributes: ["email"]`環境では、email属性のみの更新はサイレントに失敗
2. **email_verifiedとの同時更新が必要**：セキュリティ機能として、認証済み状態の設定が必須
3. **UUID経由でのみ操作可能**：254文字emailユーザーは、管理操作でUUIDが必要

## 実用的な対処法

### 長いemailユーザーへの対応策

```bash
# 254文字emailユーザーの確認方法
aws cognito-idp list-users --user-pool-id [USER_POOL_ID] | \
  jq '.Users[] | select(.Attributes[] | select(.Name=="email" and (.Value | length) > 128))'

# UUIDでの操作
aws cognito-idp admin-get-user \
  --user-pool-id [USER_POOL_ID] \
  --username [UUID]
```

### 運用上の注意点

- **AdminGetUser**：254文字emailでは直接アクセス不可、UUIDが必要
- **Filter検索**：長いemailでのfilter検索は文字列制限で失敗
- **サインイン**：実際のサインイン動作への影響は要検証

## RFC標準との乖離

| 標準 | 最大長 | Cognitoでの実際の制限 |
|------|--------|---------------------|
| RFC 5321 | 254文字 | CreateUser: 128文字 / UpdateUser: 254文字 |
| RFC 5322 | 320文字 | 対応不可 |

## まとめ

- **`UsernameAttributes: ["email"]`設定では128文字制限**がCreateUser時に適用される
- **254文字までのemailは技術的には保存可能**だが、UpdateUserAttributesの2段階操作が必要
- **実運用では管理APIの制限**により、長いemailアドレスユーザーの取り扱いが困難
- **設計時の考慮事項**として、emailアドレスの長さ制限を事前に検討することが重要

RFC標準に完全準拠したemailアドレス対応が必要な場合は、Cognitoの制限を十分に理解した上で実装方針を検討しましょう。

