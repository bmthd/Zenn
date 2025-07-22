---
title: "【Next.js】Server Functionsにおける引数検証のベストプラクティス"
emoji: "🛠️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs, react, typescript, valibot, decorator]
published: true
---

## はじめに

Next.jsをはじめとするReactのサーバーフレームワークで使われるServer Functions。

https://ja.react.dev/reference/rsc/server-functions

イベントハンドラなど、クライアントサイドからRPC的にサーバー側でしか行うことができないセキュアな処理を行うことができ非常に便利です。
従来のPages RouterのAPI Routesと異なり、サーバー側で定義した関数はクライアント側から型安全に呼び出すことができるため、まるでサーバーとクライアントの境界が消えたかのように扱えます。

```ts:server.ts
"use server";
import { db } from "@/lib/db";
import { User } from "@/domain/types";
import { ResultAsync, success, failure } from "@/lib/result";

/** ユーザを更新するServer Function */
export const updateUser = async (id: string, user: User): ResultAsync<User, string> => {
  try {
    const updatedUser = await db.user.update({
      where: { id },
      data: user,
    });
    return success(updatedUser);
  } catch (error) {
    return failure("Failed to update user");
  }
}
```

しかし、データベースへの書き込みなど副作用を伴う処理の場合、セキュリティを考慮する必要があります。
悪意のあるユーザによる不正なデータの書き込みを防ぐために、引数の検証は**必須**です。
この記事では、Server Functionsの引数検証を気持ちよく行うことができる手法を紹介します。

:::message
バリデーションライブラリの例として、[Valibot](https://valibot.dev/)を使用しています。
関数をComposableに組み合わせてバリデーションスキーマを定義できる、おすすめのバリデーションライブラリです！
コード例はValibotのみとなっていますが、ZodやYupなど他のバリデーションライブラリでも同様の実装が可能です。
:::

## 結論

以下のようなdecorator関数を作ることで、バリデーションと型推論を一体化させることができるようになります！

```ts:server.ts
"use server";
import * as v from "valibot";
import { db } from "./db";
import { withValidate } from "./decorators";
import { type User, userSchema } from "./domain";
import { failure, type ResultAsync, success } from "./result";

export const updateUser = withValidate(
  v.string(),
  userSchema(),
)(async (id, user): ResultAsync<User, string> => {
  //     ^? (id: string, user: User)
  // バリデーション済みかつ、引数の型が推論されている！
  try {
    const updatedUser = await db.user.update({
      where: { id },
      data: user,
    });
    return success(updatedUser);
  } catch (error) {
    return failure("Failed to update user");
  }
});
```

![型推論が効いているVSCodeの例](/images/server-function-decorator/image.png)

## なぜバリデーションが必要なのか？

厳密に言えば、Server Functionsの中でもデータの書き込みを伴わないGET系の処理においてはバリデーションを行う必要はありません。
しかし、はじめに述べたようにPOSTやPUTなどのデータの書き込みを伴う処理においては、悪意のあるユーザによる不正なデータの書き込みを防ぐために、引数の検証は**必須**です。
公開APIでない、アプリケーション内部からしか叩かれることを想定していないAPIは不要なのでは？と思うかもしれません。
しかし、アプリケーション内部からのAPIであっても、挙動を再現された場合にはリスクになり得ます。
Next.jsではServer FunctionsのURLが難読化されており、意図せずAPIが叩かれる可能性は低く見えますが、ソースコードはオープンソースであり、再現は不可能ではありません。
過去にはServer Functions周りで脆弱性が発見・修正された例もあり、「内部だけだから大丈夫」という油断は禁物です。

| ケース | バリデーションの必要性 | 理由 |
|----|----|----|
| 外部に公開するAPIや、Webhook | 🔏必須 | 悪意のあるユーザからの不正なデータの書き込みを防ぐため。 |
| アプリケーション内部のみのAPI | 🔏推奨 | 挙動を再現された場合、リスクになりうる |
| 副作用のないGET系のAPI | 🔓️不要 | データの書き込みを伴わないため。 |

## よくあるバリデーションの実装例

以下のように、Server Functionsの引数を手動で検証する実装はよく見られます。

```ts:server.ts
"use server";
import { db } from "@/lib/db";
import { User, userSchema } from "@/domain/types";
import { ResultAsync, success, failure } from "@/lib/result";
import * as v from "valibot";

const updateUserArgsSchema = v.tuple(
  [v.string(), userSchema()]
);

export const updateUser = async (...args: [id: string, user: User]): ResultAsync<User, string> => {
  // 引数の検証
  const validationResult = v.safeParse(updateUserArgsSchema, args);
  if (!validationResult.success) {
    return failure("Invalid arguments");
  }
  const [id, user] = validationResult.data;

  try {
    const updatedUser = await db.user.update({
      where: { id },
      data: user,
    });
    return success(updatedUser);
  } catch (error) {
    return failure("Failed to update user");
  }
};
```

この例では引数の検証を手動で行っていますが、以下のような問題があります。

1. 型定義を2回書く必要がある
   → 冗長でメンテナンスコスト高

2. 処理と無関係なロジックが混在する
   → 関数の責務が不明瞭

3. バリデーション処理が煩雑
   → エラー処理の記載漏れの原因に

## ベストプラクティス：Decoratorパターンを使った引数検証

そこで、Server Functionsをラップするdecorator関数を作成することで、引数の検証と型推論を一体化させることができます。

```ts:decorators.ts
import * as v from "valibot";
import { failure, type ResultAsync } from "./result";

export const withValidate = <T extends v.TupleItems>(...schemas: T) => {
  return <R>(
    handler: (...args: v.InferOutput<v.TupleSchema<T, any>>) => ResultAsync<R, string>,
  ) => {
    return async (...args: v.InferOutput<v.TupleSchema<T, any>>): ResultAsync<R, string> => {
      const validationResult = v.safeParse(v.tuple(schemas), args);
      if (!validationResult.success) {
        return failure("Invalid arguments");
      }
      return handler(...args);
    };
  };
};
```

decoratorパターンに馴染みのない方もいるかもしれません。
簡単に説明すると、関数やクラスの振る舞いを変更するための関数です。
ベースとなる関数を引数に取り、その関数の前後に処理を追加することができます。

今回の場合は、引数としてバリデーション用のスキーマを受け取り、第2引数としてServer Functionの本体を受け取ります。
受け取ったそれぞれの引数を使って、Server Functionsの実行を行います。
Server Functions実行の前段でバリデーションを行い、バリデーションに失敗した場合はエラーメッセージを返す処理を、このdecorator内で肩代わりしてあげることで、各Server Functionの実装から引数の型推論とバリデーションの処理を切り離すことができます。

:::message
Javaをご存知の方は、アノテーションを想像するとわかりやすいかもしれません。
PythonやTypeScriptではその名もDecoratorと呼ばれる機能があり、クラスであれば`@decorator`のように呼び出してそのクラスの振る舞いを変更することができます。
ただし、現時点ではTypeScriptに関数をデコレーションするdecoratorは存在しません。
特にそのような言語機能を使わなくても、高階関数を使うことで同様の処理を実現可能なため、上記実装では高階関数を使用しています。
現時点ではStage0となっていますが、ECMAScriptに関数Decoratorが提案されています。
https://github.com/tc39/proposal-function-and-object-literal-element-decorators
:::

```ts
"use server";
import { db } from "@/lib/db";
import { User, userSchema } from "@/domain/types";
import { ResultAsync, success, failure } from "@/lib/result";
import { withValidate } from "@/lib/decorators";
import * as v from "valibot";

export const updateUser = withValidate(
  v.string(),
  userSchema(),
)(async (id, user): ResultAsync<User, string> => {
  //     ^? (id: string, user: User)
  try {
    const updatedUser = await db.user.update({
      where: { id },
      data: user,
    });
    return success(updatedUser);
  } catch (error) {
    return failure("Failed to update user");
  }
});
```

これでバリデーションに書いた引数の型から、Server Functionsの引数の型を推論することができるようになり、引数の型定義を2回書く必要がなくなりました！

## まとめ

- Server Functionsでデータを書き込む処理にはバリデーションが必須

- 手動バリデーションは冗長で可読性が低く、記述漏れの原因に

- decorator関数で型推論とバリデーションを統一すると安全かつ読みやすくなる

今後の開発にぜひ取り入れてみてください！
