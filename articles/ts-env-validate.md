---
title: "TypeScriptにおける環境変数検証のベストプラクティス"
emoji: "🗃️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,valibot,dotenv,nodejs,vite,]
published: false
---

## はじめに

機密事項や、環境によって変わる設定値を環境変数として外部から指定できるように扱うことは、現代のアプリケーション開発において推奨される手法です。
ただし、環境変数を毎回環境に対して設定したり、コマンドライン上に打ち込むことは、手間がかかるだけでなく、ミスが発生しやすいです。
TypeScriptのプロジェクトではこれを解決するライブラリとして、dotenvを使うことが一般的です。
Node.jsやVite、Next.jsなどのフレームワークにはあらかじめdotenvが組み込まれており、特別な設定をせずともファイルを作成するだけで環境変数を読み込むことができます。

```sh:.env
API_URL=https://example.com
SKIP_LOGIN=true
TIMEOUT_MS=1000
```

```ts
const API_URL = process.env.API_URL; // string | undefined
const SKIP_LOGIN = process.env.SKIP_LOGIN; // string | undefined
const TIMEOUT_MS = process.env.TIMEOUT_MS; // string | undefined
```

しかし、このままでは環境変数の型が不明瞭であり、エラーが発生しやすいです。
そこで、TypeScriptの型システムを使って環境変数を検証する方法を紹介します。

## 基本

```ts
import * as v from "valibot"

const schema = v.object({
  API_URL: v.pipe(v.string(), v.url()),
  SKIP_LOGIN: v.optional(v.literal("true"), "false"),
  DB_PASSWORD: v.string(),
  DB_USER: v.string(),
  DB_NAME: v.string(),
  DB_HOST: v.union([v.literal("localhost"), v.pipe(v.string(), v.ipv4())]),
  DB_PORT: v.pipe(
    v.string(),
    v.transform((v) => Number(v)),
    v.pipe(v.number(), v.minValue(0), v.maxValue(65535)),
  ),
});

export const env = v.parse(schema, process.env)
```

各ファイルが環境変数を直接参照するのではなく、このenvを参照するようにします。
直接参照する場合と違い、以下のメリットがあります。

- nullチェックが不要になる
- 既に変換済みのため、変換処理が不要になる
- 型が付いているため、`env.`と入力するだけで候補が出る
- デフォルト値を設定することができる
- docコメントを付与可能
- 環境変数が指定されていない場合、ビルドが落ちるようになる
- 使用時の宣言が短くなる
- あり得ない値を指定した場合にエラーにできる

URLを求めるスキーマの場合

こんなにたくさんメリットがあります！

```ts:before
const API_URL = process.env.BASE_URL;

if (!API_URL) {
  throw new Error("BASE_URL is required");
}

const SKIP_LOGIN = process.env.SKIP_LOGIN ?? "false";

await fetch(API_URL, {
  headers: {
    Authorization: SKIP_LOGIN === "true" ? "Bearer skip" : "Bearer token",
  },
});
```

```ts:after
await fetch(env.API_URL, {
  headers: {
    Authorization: env.SKIP_LOGIN ? "Bearer skip" : "Bearer token",
  },
});
```

特に、複数のファイルから参照される環境変数の場合、変換ロジックが１箇所に集約され、メンテナンス性が向上します。

- viteの場合
- Node.jsの場合
- Playwrightの場合

## サーバーサイドとクライアントサイドで使用する環境変数を分ける

## ESLintルールで環境変数の参照を禁止する

せっかく環墧変数を検証するためのスキーマを作成したのに、直接参照してしまうと意味がありません。
チームで開発している場合は、ESLintルールを設定することで環境変数への直接参照を禁止することができます。

```js:eslint.config.js
import js from "@eslint/js";

/**
 * @type {import('eslint').Linter.Config}
 */
const customRule = {
  rules: {
    "no-restricted-syntax": [
      "error",
      // process.env.の参照を禁止
      {
        selector: "MemberExpression[object.name='process'][property.name='env']",
        message: "process.envへの直接アクセスは禁止です。@/utils/envを使用してください。",
      },
    ],
  },
};

/**
 * @type {import('eslint').Linter.Config[]}
 */
export default [js.configs.recommended, customRule];

```

