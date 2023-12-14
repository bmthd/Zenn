---
title: "オブジェクトのKeyであることを保証する型ガード作ってみた"
emoji: "🛡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript]
published: true
---

## はじめに

TypeScriptでオブジェクトのプロパティにアクセスするために、そのKeyのいずれかであることを保証したい場合があります。
この、ユーザー定義型ガードの作成に詰まってしまい時間を食ってしまったので、備忘録として残しておきます。

## ユーザー定義型ガードとは

型情報は、ユーザーがコードを書く際に定義した型情報を元に、コンパイラが推論しているものです。
実行後に型情報は消えてしまうため、別の方法で型情報を保証する必要があります。

```typescript:example.tsx
type User = {
  name: string;
  age: number;
};

const user: User = {
  name: "John",
  age: 20,
};

const isUser = (arg: unknown): arg is User => {
  return typeof arg === "object" && arg !== null && "name" in arg && "age" in arg;
};
```

戻り値の型がbooleanである関数の型注釈で`arg is User`のような書き方をすることで、コンパイラにこの関数がユーザー定義型ガードであることを伝えることができます。

このisUser関数をif文の条件式に使うことで、そのブロック内で変数がUser型であることをコンパイラに知らせることができ、型推論を行うことができるようになります。

```typescript:example.tsx
if (isUser(user)) {
  console.log(user.name); // userはUser型であることが保証される
}
```

あくまでコンパイラに知らせるだけであり、実装はユーザーに委ねられています。
仮に以下のような実装だったとしても、コンパイラはエラーを出さずに通してしまいます。

```typescript:example.tsx
const isUser = (arg: unknown): arg is User => {
  return true;
};
```

User型のオブジェクトは、JavaScriptの世界ではobject型であり、nameとageというプロパティを持つオブジェクトです。
この例ではそれを1個ずつ確認することで、User型であることを保証しています。

## オブジェクトのKeyであることを保証する型ガード

オブジェクトのKeyであることを保証する型ガードを生成する汎用関数を作成してみました。

```typescript:typeguard.tsx
export const generateKeyGuard = <T extends string | number>(
  obj: Record<T, string | number>,
) => {
  const keySet = new Set(Object.keys(obj) as T[]);
  return (arg: unknown): arg is T => keySet.has(arg as T);
};
```

`Object.keys()`の戻り値は受け取ったオブジェクトのKey配列なので`T[]`型で間違いないのですが、コンパイラは`string[]`と推論してしまうため、`T[]`型に変換しています。

## 使い方

```typescript:usage.tsx
const obj = {
  a: "a",
  b: "b",
  c: "c",
} as const;

const isKey = generateKeyGuard(obj);

const data = fetchData(); // 外部から取得した値なので、string型になっている

if (isKey(data)) {
  console.log(obj[data]); // dataはa, b, cのいずれかであることが保証されるので、objのKeyとして使ってもエラーが出ない
}
```

オブジェクトのプロパティにアクセスする際は、`obj["a"]`のように、Keyを文字列で指定する必要があります。
そこに`string`型の変数を渡すと、コンパイラはその変数が`string`型であることしか知らないため、Keyとして使うことができず、エラーが出てしまいます。
このように使うことで、外部からのデータだとしてもKeyとして使うことができます。

## 終わりに

ユーザー定義型ガードについては知っていましたが、コンパイル後に消える型情報を確認して何が嬉しいのかがよくわかっておらず、理解の妨げになっていました。
今回これを作ることで、型エラーを消すことができ、嬉しみを感じることができました。
よかったら使ってみてください！

## 参考

以下の記事が理解の参考になるので、ぜひ読んでみてください。

制御フロー分析と型ガードによる型の絞り込み
https://typescriptbook.jp/reference/statements/control-flow-analysis-and-type-guard

型ガード
https://typescriptbook.jp/reference/functions/type-guard-functions