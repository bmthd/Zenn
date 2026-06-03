---
title: "Vite の env をスキーマで管理したら、`import.meta.env` の直読みまで型安全になった"
emoji: "🔐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, vite, valibot]
published: true
---

## はじめに

Vite の環境変数を `import.meta.env` からそのまま使っていませんか？

```ts
const timeoutMs = Number(import.meta.env.VITE_API_TIMEOUT_MS ?? '5000');
const isDarkMode = import.meta.env.VITE_FEATURE_DARK_MODE === 'true';
```

この形を見て「うちのコードにもあるな…」と感じた方に向けて、この記事を書きました。

`import.meta.env` の値は基本的に文字列なので、アプリケーションで使いたい型とズレています。
解決策として「検証・変換済みの `env` オブジェクトを作る」アプローチは広く知られていますが、Vite には追加の問題があります。

**DCE（Dead Code Elimination）のために `import.meta.env` を直接参照しなければならない箇所がある**、という問題です。

この記事では t3-env と Valibot を組み合わせ、次の 2 つを両立する方法を紹介します。

- アプリケーションロジックでは、検証・変換済みの `env` を使う
- DCE が必要な箇所では、`import.meta.env` を型安全に直接参照する

## 何が困るのか

### 変換ロジックが散らばる

`import.meta.env` から取り出した値は `string | undefined` です。
アプリケーションで使いたい型（`number`・`boolean`・`Date`）とズレているため、あちこちで変換処理を書くことになります。

```ts
// タイムアウト: number に変換
const timeoutMs = Number(import.meta.env.VITE_API_TIMEOUT_MS ?? '5000');

// feature flag: 文字列比較
const isDarkMode = import.meta.env.VITE_FEATURE_DARK_MODE === 'true';

// リリース日: Date に変換
const releaseAt = new Date(import.meta.env.VITE_RELEASE_AT ?? '');
```

変換だけが問題ではありません。必須チェック・デフォルト値・範囲チェックも各ファイルに散らばります。
コードベースが広がると、同じロジックが重複し、チェックが漏れ、デッドコードが増えます。
必要のない`??`のせいで、カバレッジが落ちるのは嫌ですよねぇ…。

### DCE のために `import.meta.env` を直接参照しなければならない

DCE とは、ビルド時に到達しないと判断できるコードを成果物から削除する最適化のことです。

Vite はビルド時に `import.meta.env.VITE_FOO` を実際の文字列リテラルへ置換します。
これにより、以下のコードは本番ビルドで `if ('false' === 'true')` に変換され、分岐ごと bundle から落ちます。

```ts
// ビルド時に 'false' === 'true' に置換 → DCE で消える
if (import.meta.env.VITE_FEATURE_DARK_MODE === 'true') {
  registerDarkModePlugin();
}
```

一方、検証・変換済みの `env` を経由すると、値はランタイムまで確定しません。
そのため **DCE が効きません**。

```ts
// ランタイムでしか評価されない → bundle から落ちない
if (env.VITE_FEATURE_DARK_MODE) {
  registerDarkModePlugin();
}
```

「検証済みの `env` を使いたい」と「DCE のために `import.meta.env` を直接読みたい」は、一見矛盾します。
この記事では両方を型安全に実現する方法を紹介します。

## t3-env とは

https://env.t3.gg/

t3-env（パッケージ名: `@t3-oss/env-core`）は、env の検証・変換を一箇所にまとめるためのライブラリです。
バンドルサイズは 1.8 kB 程度と小さいです。

最大の特徴は **Standard Schema** に対応していることです。

https://standardschema.dev/

Standard Schema とは、TypeScript のバリデーションライブラリ間で共通のインターフェースを定義する仕様です。
t3-env はこの仕様に対応しているため、Zod・Valibot・Arktype など好みのライブラリを自由に選べます。

この記事では Valibot を使いますが、Zod でも同じパターンをそのまま再現できます。
`z.coerce.number()` のような Zod の coercion を使えば、後述する橋渡しスキーマを自作する手間も省ける場合があります。

## Valibot と組み合わせる

https://valibot.dev/

Valibot を選ぶ理由は明快です。バンドルサイズがとにかく小さい！

https://valibot.dev/blog/

アプリケーションの他の部分で Valibot を使っていれば追加コストはゼロです。
**使っていなくても、env のためだけに入れる価値がある**くらいサイズへのこだわりが強いです。
詳細は公式 Blog を読んでみてください。

t3-env と Valibot を組み合わせた基本的な書き味はこうなります。

```ts:src/env.ts
import { createEnv } from '@t3-oss/env-core';
import * as v from 'valibot';

export const env = createEnv({
  clientPrefix: 'VITE_',
  client: {
    VITE_APP_ENV: v.pipe(v.string(), v.nonEmpty()),
    VITE_API_BASE_URL: v.pipe(v.string(), v.url()),
  },
  runtimeEnv: import.meta.env,
  emptyStringAsUndefined: true,
  skipValidation: import.meta.env.MODE === 'test',
});
```

使い慣れた Valibot のスキーマをそのまま `client` に渡すだけです。

主なオプションの意味は次の通りです。

| オプション | 意味 |
| --- | --- |
| `clientPrefix` | クライアント側 env のプレフィックス（Vite では `VITE_`） |
| `emptyStringAsUndefined` | 空文字を `undefined` として扱う（`.env` の `VITE_FOO=` を未設定扱いにする） |
| `skipValidation` | テスト実行時など、検証をスキップしたい場合に使う |

起動時に必須 env が欠落していればエラーになるため、「本番でいきなり `undefined`」という事態を防げます。
また、プロジェクトにジョインしたばかりのメンバーが `.env` の設定を忘れていても、`npm run dev` 直後のコンソールエラーメッセージで何が不足しているのか簡単に気づけます。

![t3-envのバリデーションエラーメッセージ。どのキーが不正かパスと理由が列挙されている](/images/vite-env-schema/console-error.png)

## t3-env に足りない橋渡しスキーマを自作する

ここで1つ問題があります。

env の値は文字列ですが、アプリケーションでは `number`・`boolean`・`Date` として扱いたいです。
Zod には `z.coerce.number()` のような文字列からの型変換が組み込まれていますが、Valibot にはそれに相当する env 用のスキーマが用意されていません。

t3-env も Valibot 向けの変換スキーマを提供していないため、自作が必要です。

以下はよく使うパターンをまとめたものです。用途に合わせて組み合わせてください。

なお、URL の検証は `v.url()` として Valibot に組み込まれているため、自作は不要です。

### 型変換

```ts:src/env.ts
/** 'true' | 'false' → boolean */
function envStringToBoolean() {
  return v.pipe(
    v.picklist(['true', 'false']),
    v.transform((s) => s === 'true'),
  );
}

/** '1000' → number（整数・0以上を保証） */
function envStringToNumber() {
  return v.pipe(
    v.string(),
    v.transform(Number),
    v.number(),
    v.integer(),
    v.minValue(0),
  );
}

/** '2025-01-01' → Date */
function envStringToDate() {
  return v.pipe(
    v.string(),
    v.transform((s) => new Date(s)),
    v.date(),
    v.check((date) => !Number.isNaN(date.getTime())),
  );
}
```

これらは変数として定義できますが、関数にすることを推奨します。
理由は 2 つです。Valibot 自身のスキーマ（`v.string()` や `v.number()` など）がすべて関数であるパターンに揃えることで、使い勝手が統一されます。
また、関数にすることでスキーマの構築が呼び出し時まで遅延されるため、モジュール読み込み時の副作用を避けられます。

`v.picklist(['true', 'false'])` を使うことで、`'true'` と `'false'` 以外の文字列は検証エラーになります。
`Number` への変換後に `v.integer()` と `v.minValue(0)` を重ねることで、範囲外の値も弾けます。

### デフォルト値

`emptyStringAsUndefined: true` を設定していれば、`v.optional` でデフォルト値を持たせられます。

```ts
// 未設定なら 5000 を使う
VITE_API_TIMEOUT_MS: v.optional(envStringToNumber(), 5000),

// 未設定なら false を使う
VITE_FEATURE_DARK_MODE: v.optional(envStringToBoolean(), false),
```

`.env` に記載がない場合や空文字の場合に fallback するため、必須ではない env の管理が楽になります。

### 日付ライブラリへの最適化

アプリケーションで Day.js を使っている場合、`Date` に変換するより直接 `Dayjs` に変換してしまうほうが便利です。

```ts
import dayjs, { type Dayjs } from 'dayjs';

/** '2025-01-01' → Dayjs */
function envStringToDayjs() {
  return v.pipe(
    v.string(),
    v.transform((s) => dayjs(s)),
    v.custom<Dayjs>((v) => dayjs.isDayjs(v) && v.isValid()),
  );
}
```
`env.VITE_RELEASE_AT` が `Dayjs` 型で取り出せるため、アプリ側で毎回 `dayjs(env.VITE_RELEASE_AT)` と書く必要がなくなります。

このように、Valibot の `v.pipe()` と `v.transform()` を組み合わせれば、どんな型へも変換できます。
自作関数はアプリケーションの要件に合わせてカスタマイズしてください。

これらを `client` スキーマに組み込みます。

```ts:src/env.ts
export const env = createEnv({
  clientPrefix: 'VITE_',
  client: {
    VITE_APP_ENV: v.pipe(v.string(), v.nonEmpty()),
    VITE_API_BASE_URL: v.pipe(v.string(), v.url()),
    VITE_API_TIMEOUT_MS: envStringToNumber(),
    VITE_RELEASE_AT: envStringToDate(),
    VITE_FEATURE_DARK_MODE: envStringToBoolean(),
  },
  runtimeEnv: import.meta.env,
  emptyStringAsUndefined: true,
  skipValidation: import.meta.env.MODE === 'test',
});
```

`env` を経由すると、変換済みの値が型付きで取り出せます。

```ts
// ✅ number として使える
const res = await fetch(env.VITE_API_BASE_URL + '/users', {
  signal: AbortSignal.timeout(env.VITE_API_TIMEOUT_MS),
});

// ✅ boolean として使える
if (env.VITE_FEATURE_DARK_MODE) {
  applyDarkTheme();
}
```

## Vite との型統合

`env` の型は整いました。しかし、`import.meta.env.VITE_API_TIMEOUT_MS` の型はまだ `string | undefined` のままです。
Vite の型定義（`vite-env.d.ts`）は env のキーを知らないからです。

これを解決するために、Valibot スキーマから `ImportMetaEnv` を生成し、declaration merging で `import.meta.env` の型に反映します。

まずスキーマを関数として切り出します。

```ts:src/env.ts
const ClientSchema = () => ({
  VITE_APP_ENV: v.pipe(v.string(), v.nonEmpty()),
  VITE_API_BASE_URL: v.pipe(v.string(), v.url()),
  VITE_API_TIMEOUT_MS: envStringToNumber(),
  VITE_RELEASE_AT: envStringToDate(),
  VITE_FEATURE_DARK_MODE: envStringToBoolean(),
});

type ClientSchema = ReturnType<typeof ClientSchema>;
```

次に、`import.meta.env` の型を `InferInput` で生成し、`ImportMetaEnv` にマージします。

```ts:src/env.ts
type ClientEnv = {
  readonly [K in keyof ClientSchema]: v.InferInput<ClientSchema[K]>;
};

declare global {
  interface ImportMetaEnv extends ClientEnv {}
}

export const env = createEnv({
  clientPrefix: 'VITE_',
  client: ClientSchema(),
  runtimeEnv: import.meta.env,
  emptyStringAsUndefined: true,
  skipValidation: import.meta.env.MODE === 'test',
});
```

`vite-env.d.ts` 自体は最低限の土台だけを置きます。
env のキーをここに手書きで並べると、Valibot スキーマと型定義の二重管理になるからです。

```ts:src/vite-env.d.ts
/// <reference types="vite/client" />

interface ImportMeta {
  readonly env: ImportMetaEnv;
}
```

SSoT（Single Source of Truth）は Valibot スキーマのまま保たれます。
env を追加・削除・変更するときに触るファイルは `env.ts` の一箇所だけです。

## なぜ `import.meta.env` の直接参照が型安全なのか

ここが今回の設計の肝です。

`InferInput` は変換パイプラインへの入力型を表します。
`import.meta.env` が保持しているのは変換前の文字列です。だから `InferInput` で生成した型が実態と一致します。

各キーの型がどう変わるか確認してみましょう。

| キー | `import.meta.env`（`InferInput`） | `env`（`InferOutput`） |
| --- | --- | --- |
| `VITE_API_TIMEOUT_MS` | `string` | `number` |
| `VITE_RELEASE_AT` | `string` | `Date` |
| `VITE_FEATURE_DARK_MODE` | `'true' \| 'false'` | `boolean` |

`InferOutput`（変換後の型）を使ってしまうと、`import.meta.env.VITE_FEATURE_DARK_MODE` が `boolean` という実態と違う型になります。
型と実装が乖離した状態では、型安全の恩恵が逆に仇となります。

`InferInput` を使うことで、`import.meta.env.VITE_FEATURE_DARK_MODE` の型は `'true' | 'false'` になります。
誤った文字列との比較はコンパイルエラーになるため、タイポや仕様外の値を静的に弾けます。

```ts
// ❌ コンパイルエラー: '"on"' は '"true" | "false"' に代入できない
if (import.meta.env.VITE_FEATURE_DARK_MODE === 'on') { ... }

// ✅ 正しい比較
if (import.meta.env.VITE_FEATURE_DARK_MODE === 'true') { ... }
```

使い分けをまとめます。

```ts
// DCE が目的の箇所 → import.meta.env を直接読む（Vite がリテラルに置換してくれる）
if (import.meta.env.VITE_FEATURE_DARK_MODE === 'true') {
  registerDarkModePlugin();
}

// アプリケーションロジック → 検証・変換済みの env を使う
if (env.VITE_FEATURE_DARK_MODE) {
  applyDarkTheme();
}
```

Valibot スキーマを SSoT とし、`InferInput` / `InferOutput` でそれぞれの型を生成することで、両者の責務を型レベルで分離できます。

## おまけ: よくある懸念

「ライブラリを増やしたくない」という声はよく聞きます。以下に典型的な懸念と答えをまとめました。

| 懸念 | 答え |
| --- | --- |
| **バンドルサイズが増える** | t3-env は 1.8 kB。Valibot はアプリケーションで兼用すれば純増ゼロ。兼用しない場合でも、各所に散らばっていた変換ロジックが消えるため、トータルのサイズはほぼ変わらない |
| **アップデートのメンテコストが増える** | t3-env も Valibot も純粋関数ベースの設計で、バージョンアップによる破壊的変更が少ない。万が一 env の検証が壊れてもビルド時にエラーになるため、気づかず本番に出ることがない |
| **t3-env のアップデートを見逃したら怖い** | env の検証が壊れた場合はビルドが落ちる設計なので、ノールックでマージしても安全。壊れたまま本番に出ることがない |
| **Valibot に不安がある** | t3-env は Standard Schema に対応しているため、Zod など他のライブラリに差し替えられる。Valibot を試して合わなければ移行コストは小さい |

## まとめ

この記事で紹介した構成のポイントをまとめます。

- **t3-env** — env の検証・変換を一箇所に集約するライブラリ。Standard Schema 対応なので Zod・Valibot・Arktype など好みのライブラリで使える
- **Valibot の橋渡しスキーマ** — t3-env は Valibot 向けの変換スキーマを提供しない。`envStringToBoolean` / `envStringToNumber` / `envStringToDate` を自作して補う
- **SSoT** — Valibot スキーマから `InferInput` / `InferOutput` で型を生成し `ImportMetaEnv` にマージすることで、型定義の二重管理をなくせる
- **DCE との両立** — `import.meta.env` の直接参照は `InferInput` 由来の型で守られているため、型安全に DCE を利用できる

各ファイルで `Number(...)` や `=== 'true'` を書き続けるより、env の境界で一度だけ検証・変換するほうが、コード量もバグの入口も減らせます。
ぜひ試してみてください！
