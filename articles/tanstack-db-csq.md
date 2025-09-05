---
title: "TanStack DB ~状態管理の新しい考え方~"
emoji: "🗃️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [tanstackdb,react,typescript,javascript]
published: false
---

## はじめに

[TanStack](https://tanstack.com/) が新たに公開した **TanStack DB** について調べたので、その概要を紹介します。

https://tanstack.com/db/latest

## TanStack DBとは

TanStack DBは、フロントエンドに **永続化層（DB）** を設け、コンポーネントから直接クエリできる仕組みを提供します。

従来は、バックエンドのDBを正とし、フロントエンドはそれを定期的に問い合わせる形が一般的でした。
TanStack DBはこれを逆転させ、**フロントエンドがデータの主導権を持つ** 形にします。

つまり「DBをフロントエンドに持ってきた」感覚で使えるのが魅力です。

## TanStack DBは何が嬉しいのか

## 楽観的更新の自動化

TanStack Queryの更新はデフォルトで悲観的更新です。
つまり、失敗することを前提に、成功した場合だけUIが更新されるということです。
楽観的更新に対応したい場合は手動で仮の更新、ロールバック、再フェッチを実装する必要がありました。

```ts: TanStack Queryによる楽観的更新の例
const mutation = useMutation({
  mutationFn: updateTodo,
  // 楽観的更新の実装
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] });
    const previousTodos = queryClient.getQueryData(['todos']);
    queryClient.setQueryData(['todos'], (old) => [...old, newTodo]);
    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos);
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] });
  },
});
```

このような処理を毎回書くのは面倒ですし、ミスも起こりやすいです。
TanStack DBでは、**ローカルのCollectionを直接更新するだけで、楽観的更新が自動的に行われる** ため、上記のような複雑なコードを書く必要がありません。

```ts: TanStack DBによる楽観的更新の例
const addTodo = (newTodo: TODO) => {
  todosCollection.insert(newTodo); // これだけで楽観的更新が完了
};
```

快適なUIを実現するうえで、楽観的更新は重要な要素です。
処理が失敗することは稀である場合が多いため、**成功を前提にUIを即座に更新する**ことで、ユーザー体験が向上します。

## 永続化層を自由に差し替え可能

フロントエンドでストレージを扱う場合、localStorageやIndexedDBなど様々な選択肢があります。
例えば、IndexedDBの場合はDexie.js、GraphQLの場合はApollo Client、FirebaseはFirebase SDKなどのラッパーライブラリを使用するのが一般的です。
これらはそれぞれに独自のAPIのため、使い方が異なり、学習コストがかかります。
TanStack DBは特定のストレージ技術に依存しません。アダプタを差し替えることで、様々なストレージに対応できます。

* localStorage
* インメモリ
* REST API
* Electric SQL
* TrailBase

また、標準でサポートされているアダプタ以外にも、独自にアダプタを実装できます。
https://tanstack.com/db/latest/docs/guides/collection-options-creator

将来的にはSupabase, IndexedDB, DuckDB Wasmなども使えるようになることでしょう。

### バックエンド実装が不要になるかも

TanStack DBを使えば、バックエンドはデータベーススキーマの設計と、Row Level Security（RLS）を設定するだけで済む可能性があります。
従来も、SupabaseやGraphQLのHasura, PostGraphileなどを使えば同様のことが可能でした。

しかし、これらの技術はそれぞれに独自のAPIがあり、学習コストが発生してしまいます。
フロントエンド側のロジックが特定のSaaSに依存し、将来UIライブラリやバックエンドのどちらかを変更したいとなった場合、移行が困難になるリスクがあります。
TanStack DBであれば、アダプタを差し替えるだけで済むため、特定のストレージ技術に依存しないで済みます。
どのバックエンドを採用していたとしても、フロントエンド側の実装は変わりません。
しかも、UIライブラリもマルチフレームワークに対応しているため、ReactからSvelteに乗り換えたくなった場合でも全く同じ実装を使い回せます。

---

## 基本的な使い方

早速使い方を説明していきます。
今回はReactでの利用をベースに解説していきます。

:::message
TanStack DBはまだv0のため、APIは変わる可能性があります。
:::

TanStack DBのパッケージは以下の通りです。

```bash
npm install @tanstack/db @tanstack/react-db
```

まずはデータを永続化するためのCollectionを定義します。

```ts
import { createCollection } from "@tanstack/db";

export const drawCalcCollection = createCollection(
  localStorageCollectionOptions({
    storageKey: "tcg-tool-draw-calculations",
    id: "draw-calculations",
    getKey: (item) => item.id,
    schema: drawCalcSchema
  }),
);

// もしくは
export const drawCalcCollection = createCollection(
  localStorageCollectionOptions<DrawCalc>({
    storageKey: "tcg-tool-draw-calculations",
    id: "draw-calculations",
    getKey: (item) => item.id,
  }),
);
```

このCollectionはSQLでいうテーブルに相当します。
型定義は型パラメータとして渡しても良いし、ZodやValibotなどのランタイムバリデーションのスキーマを渡して推論させることもできます。

続いて、クエリでデータを取得します。

```ts
import { useLiveQuery } from "@tanstack/react-db";
import { eq } from "@tanstack/db";

const { data } = useLiveQuery((q) =>
  q
    .from({ calculations: calculationsCollection })
    .where(({ calculations }) => eq(calculations.id, calculationId))
    .select({
      id: calculations.id,
      name: calculations.name,
      result: calculations.result,
    })
    .orderBy(({ calculations }) => calculations.updatedAt, "desc"),
);
```

`useLiveQuery`を利用することで、ストレージをリアルタイムで購読し、その変更に応じてコンポーネントが再レンダリングされるようになります。
イメージとしては、Reduxの`useSelector`と同じですが、そのセレクター部分がSQLライクなクエリで書けるようになっています。
ReduxのセレクターはJavaScriptで記述するため柔軟性がありますが、その分アクセス方法が定まりません。
ここを**SQLライクなAPIで統一することにより、アクセス速度の向上と認知負荷の軽減**が期待できます。
これは目から鱗の発想でした。

:::message
現状では各UIライブラリ向けのインテグレーションAPIはそれぞれ1つずつのみとなっており、とてもシンプルです。
Reactの場合はSuspenseを使って読み込みをしたいところですが、現時点では`isLoading`, `isError`を確認して自前で制御しなければなりません。
`useSuspenseLiveQuery`や, `{ suspense: true }`オプションの追加を期待したいですね。
:::

## 派生Collection

クエリをあらかじめ定義した「派生Collection」も作れます。

```ts
// 履歴用コレクション
export const drawCalcHistoryCollection = createLiveQueryCollection((q) =>
  q.from({ calculations: drawCalcCollection })
   .orderBy(({ calculations }) => calculations.updatedAt, "desc"),
);

// 引数付きコレクション
export const createDrawCalcByGameCollection = (gameTemplate: string) =>
  createLiveQueryCollection((q) =>
    q.from({ calculations: drawCalcCollection })
     .where(({ calculations }) => eq(calculations.gameTemplate, gameTemplate))
     .orderBy(({ calculations }) => calculations.updatedAt, "desc"),
  );
```

これにより「利用頻度の高いクエリ」を再利用可能な形でまとめられます。

## コンポーネントでの利用

```tsx:親コンポーネント
const CalculationList = () => {
  const { data } = useLiveQuery((q) =>
    q
      .from({ calculations: calculationsCollection })
      .select({
        id: calculations.id, // idだけをselectしているので、この部分のみに反応する
      })
  );

  return (
    <>
      {data.map((item) => (
        <CalculationItem key={item.id} calculationId={item.id} />
      ))}
    </>
  );
};
```

```tsx:子コンポーネント
const CalculationItem = ({ calculationId }: { calculationId: string }) => {
  const { data } = useLiveQuery((q) =>
    q.from({ calculations: calculationsCollection })
      .where(({ calculations }) => eq(calculations.id, calculationId)), // 特定のidのCollectionの変更のみに反応する
  );

  const [calculation] = data
  if (!calculation) return <FallbackItem />;

  return (
    <Card.Root>
      <Card.Header>
        {calculation.name}
      </Card.Header>
      <Card.Body>
        {calculation.result}
      </Card.Body>
    </Card.Root>
  );
}
```

このように、コンポーネントの中で直接クエリを書き、部分的に購読可能です。

## 実際に触ってみた

Claude Codeと一緒に既存アプリをTanStack DBに載せ替えてみました。

* 公開デモ: [https://tcg-tool.pages.dev/draw-calc-db](https://tcg-tool.pages.dev/draw-calc-db)
* ソースコード: [https://github.com/bmthd/tcg-tool/pull/3/files](https://github.com/bmthd/tcg-tool/pull/3/files)

ぜひ動作感を体験してみてください。

---

## まとめ

* TanStack DBは「**フロントエンド中心のデータ設計**」を加速する
* すべてのServer Stateの抽象化レイヤーになりうる
* 状態管理とデータ保管を統合し、バックエンド設計にも影響を与える可能性がある

---

今後のアップデートでどのように進化していくか非常に楽しみです。
