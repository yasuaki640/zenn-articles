---
title: "Hono x JSXでオレオレgptクライアントを作る"
emoji: "🚧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

もうChat GPTなしの生活は耐えられないが、たまにしか使わないのに定額料金はしんどい、、円安だし、、、
↓
じゃあOpenAIのAPIを直接呼ぶアプリを自作すれば安く済ませられるのでは?
↓
でもそれだけのために、SPA x JSON API のような構成を作るのは面倒、、、
↓
**Hono x JSX**ならフロントもバックエンドも簡単に実装して月々のGPT料金を抑えられそう！！

ということでオレオレgptクライアントを実装したのでメモ。
※デプロイはまだできていません。

## 実装

公式の手順に従ってスキャフォールドを作成
※環境は`cloudflare-workers`を選択

https://hono.dev/getting-started/basic#starter

JSXを使えるように`tsconfig.json`をいじる

https://hono.dev/guides/jsx#settings

### ルーティング & JSXで画面表示

`index.ts`を`index.tsx`にリネーム

共通レイアウトの作成

```tsx:Layout.tsx
export const Layout: FC = ({ children }) => (
  <html lang={"ja"}>
    <body>{children}</body>
  </html>
);
```

共通レイアウト描画用のミドルウェアを定義

```typescript:layout.ts
export const LayoutMiddleware = jsxRenderer(Layout);
```

トップ画面用のレイアウトを定義

```tsx:Top.tsx
export const Top: FC = () => (
  <>
    <h1>gpt-web-client</h1>
    <a href={"/chats"}>Chats</a>
  </>
);
```

ルーティングして画面にHTMLを返す。

```tsx:index.tsx
const app = new Hono<AppEnv>();

app.use(LayoutMiddleware);

app.get("/", (c) => c.render(<Top />));
```

Yay!!

![](/images/gpt-web-client/top.png =400x)

### DBとテーブルの用意

今回はCloudflare D1 x Drizzle ORMを使う。

下記に従ってD1を用意。(今回はlocal環境のみ用意)

https://developers.cloudflare.com/d1/get-started/

下記に従って、Drizzle ORMをインストール

https://orm.drizzle.team/docs/get-started-sqlite#cloudflare-d1

`wrangler.toml`に`migrations_dir`を追加

```toml
[[d1_databases]]
binding = "DB" 
database_name = "gpt-web-client"
database_id = "xxxxxx"
migrations_dir = "drizzle/" # !!ここを追加!!
```

設定ファイルを追加

```ts:drizzle.config.ts
import { defineConfig } from "drizzle-kit";

export default defineConfig({
    schema: "./src/schema.ts",
    out: "./drizzle"
})
```

適当なスキーマ定義を作成

```ts:schema.ts
export const Rooms = sqliteTable("Rooms", {
  roomId: text("roomId").primaryKey(),
  roomTitle: text("roomTitle"),
  ...
```

マイグレーションファイルを出力

```sh
npx drizzle-kit generate:sqlite --schema=src/schema.ts
```

ローカルにマイグレーションを実行

```sh
npx  wrangler d1 migrations apply gpt-web-client --local
```

ロジックで適当なデータを取得

```ts:index.tsx
app.get("/chats", async (c) => {
  const db = drizzle(c.env.DB);
  const rooms = await db
    .select()
    .from(Rooms)
    .orderBy(desc(Rooms.roomUpdated))
    .all();

  return c.render(<RoomList props={{ rooms }} />);
});
```

Yay!!

![](/images/gpt-web-client/rooms.png =400x)

### OpenAI APIをつなぎこむ

## 目標規定文

hono jsxならUIのあるアプリケーションを簡単に書くことができる
個人開発におすすめなのでみんな触ってみよう

## アウトライン

- 背景
- 実装の仕方
- まとめ

