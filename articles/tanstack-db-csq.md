---
title: "TanStack DB ~状態管理の新しい考え方~"
emoji: "🗃️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [tanstackdb,tanstack,react,typescript,zennfes2025free]
published: true
---

## はじめに

[TanStack](https://tanstack.com/) が新たに公開した **TanStack DB** について調べたので、その概要を紹介します。

https://tanstack.com/db/latest

:::message
こちらの記事はReact Tokyo ミートアップ #8にて発表させていただいた内容をより深堀りした記事となっています。
登壇資料はこちらです。
[TanStack DB ~状態管理の新しい考え方~](https://speakerdeck.com/bmthd/tanstack-db-zhuang-tai-guan-li-noxin-siikao-efang)
:::

### TanStack DBとは

TanStack DBは、フロントエンドに **永続化層（DB）** を設け、コンポーネントから直接クエリできる仕組みを提供します。

従来は、バックエンドのDBを正とし、フロントエンドはそれを定期的に問い合わせる形が一般的でした。
TanStack DBはこれを逆転させ、**フロントエンドがデータの主導権を持つ** 形にします。

つまり「DBをフロントエンドに持ってきた」感覚で使えるのが魅力です。

## TanStack DBは何が嬉しいのか

### 楽観的更新の自動化

同じTanStackシリーズのTanStack QueryはデフォルトではUIの楽観的更新をしません。
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
const updateTodo = async (id: string, newTodo: TODO) => {
  await todosCollection.update(id, (draft) => {
    draft = newTodo; // これだけで楽観的更新が完了
  });
};
```

快適なUIを実現するうえで、楽観的更新は重要な要素です。
処理が失敗することは稀である場合が多いため、**成功を前提にUIを即座に更新する**ことで、ユーザー体験が向上します。

### 永続化層を自由に差し替え可能

フロントエンドでストレージを扱う場合、localStorageやIndexedDBなど様々な選択肢があります。
また、バックエンドと同期する場合はSupabaseやFirebase、GraphQLなどのSaaSを利用することもあります。
そして、それらをフロントエンドから利用する場合はラッパーライブラリを使用するのが一般的です。
例えば、IndexedDBの場合はDexie.js、GraphQLの場合はApollo Client、FirebaseはFirebase SDKなどです。
これらはそれぞれに独自のAPIのため、使い方が異なり、学習コストがかかります。
TanStack DBは特定のストレージ技術に依存しません。アダプタを差し替えることで、様々なストレージに対応できます。

* localStorage
* インメモリ
* REST API
* Electric SQL
* TrailBase

また、標準でサポートされているアダプタ以外にも、独自にアダプタを実装できます。
https://tanstack.com/db/latest/docs/guides/collection-options-creator

Supabase, IndexedDB, DuckDB Wasmなどのアダプタも技術的に実装が可能なため、登場が期待されます。

### バックエンド実装を簡素化できるかも

フロントエンドが欲しいデータを直接クエリできるため、バックエンドの実装を簡素化できる可能性があります。
TanStack DBを使えば、バックエンドがやるべきことはデータベーススキーマの設計と、Row Level Security（RLS）の設定、認証の実装程度に留められます。
上記で上げた、Electric SQLとTrailBaseは、**フロントエンドのDBとバックエンドのDBを自動的に同期**する仕組みを提供しています。
従来も、SupabaseやGraphQLのHasura, PostGraphileなどを使えば同様のことが可能でした。

しかし、これらの技術はそれぞれに独自のAPIがあり、学習コストが発生してしまいます。
フロントエンド側のロジックが特定のSaaSに依存し、将来UIライブラリやバックエンドのどちらかを変更したいとなった場合、移行が困難になるリスクがあります。
TanStack DBであれば、アダプタを差し替えるだけで済むため、特定のストレージ技術に依存しないで済みます。
どのバックエンドを採用していたとしても、フロントエンド側の実装は変わりません。
しかも、UIライブラリもマルチフレームワークに対応しているため、ReactからSvelteに乗り換えたくなった場合でも差し替えが容易です。

---

## 使い方

早速使い方を説明していきます。
今回はReactでの利用をベースに解説していきます。

:::message
TanStack DBはまだv0のため、APIは変わる可能性があります。
:::

TanStack DBのパッケージは以下の通りです。

```bash
npm install @tanstack/db @tanstack/react-db
```

### Collectionの定義とクエリ

まずはデータを永続化するためのCollectionを定義します。

```ts
import { createCollection, localStorageCollectionOptions } from "@tanstack/db";
import * as v from "valibot"; 

const drawCalcSchema = v.object({
  id: v.string(),
  name: v.string(),
  gameTemplate: v.string(),
  result: v.string(),
  createdAt: v.date(),
  updatedAt: v.date(),
});

export const drawCalcCollection = createCollection(
  localStorageCollectionOptions({
    storageKey: "tcg-tool-draw-calculations",
    id: "draw-calculations",
    getKey: (item) => item.id,
    schema: drawCalcSchema
  }),
);

// もしくは

type DrawCalc = {
  id: string;
  name: string;
  gameTemplate: string;
  result: string;
  createdAt: Date;
  updatedAt: Date;
};

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
ここを**SQLライクなAPIで統一することにより、効率の良いデータ取得と認知負荷の軽減**が期待できます。
これは目から鱗の発想でした。

:::message
現状では各UIライブラリ向けのインテグレーションAPIはそれぞれ1つずつのみとなっており、とてもシンプルです。
Reactの場合はSuspenseを使って読み込みをしたいところですが、現時点では`isLoading`, `isError`を確認して自前で制御しなければなりません。
`useSuspenseLiveQuery`や`{ suspense: true }`オプションの追加を期待したいですね。
:::

### 派生Collection

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

### コンポーネントでの利用

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
      // 特定のidのCollectionの変更のみに反応する
      .where(({ calculations }) => eq(calculations.id, calculationId)), 
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

---

## おまけ

### 状態管理のベストプラクティスを変える

これまでデータフェッチやバックエンドとの同期を主軸に書いてきましたが、ローカルな状態管理もTanStack DBが置き換えてしまうのではと考えています。
今までJotaiやZustandなどで状態管理していた部分です。
これらは複雑なデータ構造を表現しようとすると、状態の正規化やセレクターの実装が必要になり、認知負荷が高くなりがちです。
データのあるべき姿をスキーマで定義し、操作をSQLライクなクエリで行うことができるため、**状態管理のベストプラクティスを大きく変える可能性があります**。
そして、それらをそのままバックエンドに同期できる点も見逃せません。

### 想定されるユースケース

では一体どんな場面で使うと効果的なのか、少し考えてみました。

1. **リアルタイム性が求められるアプリケーション**

   チャットアプリやコラボレーションツールなど、データの変更が即座に反映されることが重要な場合。
   例えば、MiroやFigma、Google Spreadsheetのような共同編集ツール。

2. **オフラインでの利用が求められるアプリケーション**

   ネットワークが不安定な環境でも快適に動作することを求められる場合。
   例えば、建設業、フィールドワーク、医療現場など。
   
3. **大容量のデータの分析や可視化が求められるアプリケーション**

   大量のデータを効率的に処理し、ユーザーに分かりやすく提示することが求められる場合。
   例えば、ダッシュボードやBIツール、株取引アプリでの利用など。

### 実際に触ってみた

Claude Codeと一緒に既存アプリをTanStack DBに載せ替えてみました。

* 公開デモ: [https://tcg-tool.pages.dev/draw-calc-db](https://tcg-tool.pages.dev/draw-calc-db)
* ソースコード: [https://github.com/bmthd/tcg-tool/pull/3/files](https://github.com/bmthd/tcg-tool/pull/3/files)

このアプリケーションはlocalStorageのみにデータを保存するシンプルな計算アプリですが、状態管理アプリを採用せずにTanStack DBだけで実装できました。
書き心地が良く、状態管理とデータ保管が統合されているため、コードがシンプルになりました。

---

## まとめ

* TanStack DBは「**フロントエンド中心のデータ設計**」を加速する
* すべてのServer Stateの抽象化レイヤーになりうる
* 状態管理とデータ保管を統合し、バックエンド設計にも影響を与える可能性がある

今後のアップデートでどのように進化していくか非常に楽しみです。
