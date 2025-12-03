---
title: "コンポーネントを\"生成\"する関数 (Component Factory パターン) でロジックと型をカプセル化する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript, design-pattern]
published: true
---

:::message
この記事は [React Tokyo Advent Calendar 2025](https://qiita.com/advent-calendar/2025/react-tokyo) の 4日目の記事です。
:::

## はじめに

モダンなコンポーネントライブラリのソースコードを読んでいて、最近見かけることが増えてきた「Component Factoryパターン」について解説します。

### やりたいことのイメージ

まず、どのようなコードを目指しているのか、具体的な利用例から見てみましょう。
あるページの一部を、共通レイアウト側のヘッダーに「転送（Hoist）」したいという要件があるとします。
各ページのレイアウトを共通にしたいのですが、ヘッダーボタンに固有の要件があります。レイアウト側で直接定義するのを避けるべきです。
しかし、各ページで共通するレイアウトは維持したい、という状況です。
通常Contextを作成し、それを使う側で`use(Context)`するという方法もありますが、今回は「コンポーネントを生成する関数（Component Factory）」パターンを使って実装します。

```tsx: page.tsx
import { HeaderAction } from './header-factory';

export default function Page() {
  return (
    <>
      {/* ここに書いたものがヘッダーに飛んでいく */}
      <HeaderAction.Hoist>
        <Button>このページのみでヘッダーに表示したいコンポーネント</Button>
      </HeaderAction.Hoist>
      
      <div>メインコンテンツ</div>
    </>
  )
};
```

```tsx:layout.tsx
// レイアウトコンポーネント
import { HeaderAction } from './header-factory';

export default function Layout({ children }) {
  return (
    // Factoryから生成されたProviderで囲む
    <HeaderAction.Provider>
      <header>
        <heading>example site</heading>
        {/* ここにページからHoistされたコンテンツが表示される */}
        <HeaderAction.Slot />
      </header>
      <body>{children}</body>
      <footer>©️example</footer>
    </HeaderAction.Provider>
  );
}
```

```tsx:header-factory.tsx
"use client";
import { createHoistableComponent } from "@/lib/hoistable-component";

export const HeaderAction = createHoistableComponent();
//             ^? { Provider: React.FC<{ children: React.ReactNode }>; Slot: React.FC; Hoist: React.FC<{ children: React.ReactNode }> }
```

このように、`createHoistableComponent` という関数が、**互いに関連し合うコンポーネント群を一括で生成**しています。

これはJavaScriptのクロージャと高階関数の特性を活かし、特定のスコープ（ContextやStore）を共有するコンポーネントを動的に定義するパターンです。


## このパターンのメリット

- "use client" ディレクティブを1箇所にまとめられる

先程の例を通常のContextを宣言した場合をイメージしてみてください。

```tsx: header-context.tsx
"use client";
import { createContext, useContext, useState } from "react";

const HeaderActionContext = createContext<{
  hoistedContent: React.ReactNode;
  setHoistedContent: (content: React.ReactNode) => void;
} | null>(null);
```

useContextを呼び出すコンポーネントはすべてクライアントコンポーネントである必要があります。
そのため、Contextだけを共通化すると、使う側でContextを読み取るコンポーネントを宣言する必要があります。
その点、Component Factoryパターンを使うと、Contextの宣言とそれを使うコンポーネントの宣言を1箇所にまとめることができます。
その結果、"use client" ディレクティブを1箇所に付与するだけで済みます。
これが、コードの可読性と保守性を向上させる大きなメリットとなります。

- 型情報が一箇所にまとまる

複数の関連するコンポーネント群が似通った構成を持つ場合に共通化をしようとしたとき、それぞれのコンポーネントに型引数を渡す必要が出てきます。
場合によっては、型推論をさせるためだけに本来不要な引数を渡す必要が出てくることもあります。
Component Factoryパターンを使うと、特定の引数に基づいて生成されたコンポーネント群が同じ型情報を共有するため、型引数の冗長な指定を避けることができます。

```tsx: form-factory.tsx
import { createFormComponents } from "@/lib/form-factory";
import { z } from "zod";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

export const { Form: UserForm, Field: UserField } = createFormComponents(UserSchema);

const ItemSchema = z.object({
  title: z.string(),
  price: z.number(),
});

export const { Form: ItemForm, Field: ItemField } = createFormComponents(ItemSchema);
```
<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details 使わない場合
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
```tsx: form-without-factory.tsx
import { z } from "zod";
import { Form } from "@/components/form";
import { Field } from "@/components/field";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

export default function Page() {
  return (
    <Form schema={UserSchema}>
      <Field schema={UserSchema} name="name" />
      <Field schema={UserSchema} name="age" />
    </Form>
  );
}
```

Fieldコンポーネントは本来スキーマを受け取る必要はありませんが、この例では`name`に型推論を効かせるためだけにわざわざスキーマを渡しています。
そのためだけに、コードが冗長になってしまいます。
`createFormComponents`は内部でスキーマから推論した型情報だけを`Field`にわたすことができるため、実行時のオーバーヘッドを増やすことなく、型情報を共有できています。
:::


- APIのカプセル化
Component Factoryパターンを使うと、内部で使用するContextやStoreの存在を隠蔽し、利用者にはシンプルなコンポーネントだけを露出できます。
これにより、APIの設計がシンプルになり、利用者が誤って内部の実装に依存するリスクを減らせます。

## 実装例

以下に、先程の`createHoistableComponent`関数の実装例を示します。

```tsx: hoistable-component.tsx
import {
  Fragment,
  type JSX,
  type PropsWithChildren,
  useEffect,
  useMemo,
  useRef,
  useSyncExternalStore,
} from "react";
import { createLiftStore, LiftStoreContext, useLiftStore } from "./store.ts";

export interface ProviderProps extends PropsWithChildren {}

export interface HoistProps extends PropsWithChildren {
  priority?: number;
}

export const createHoistableComponent = () => {
  const Provider = ({ children }: ProviderProps): JSX.Element => {
    const store = useMemo(createLiftStore, []);
    return <LiftStoreContext.Provider value={store}>{children}</LiftStoreContext.Provider>;
  };

  const Slot = (): JSX.Element | null => {
    const store = useLiftStore();
    const snap = useSyncExternalStore(store.subscribe, store.getSnapshot, store.getSnapshot);
    if (snap.length === 0) return null;

    return (
      <>
        {snap.map(({ key, node }) => (
          <Fragment key={String(key)}>{node}</Fragment>
        ))}
      </>
    );
  };

  const Hoist = ({ children, priority = 0 }: HoistProps): JSX.Element | null => {
    const store = useLiftStore();
    const keyRef = useRef<symbol>();
    if (!keyRef.current) {
      keyRef.current = Symbol("hoist-entry");
    }

    useEffect(() => {
      if (!keyRef.current) return;
      store.upsert(keyRef.current, children, priority);

      return () => {
        if (!keyRef.current) return;
        store.remove(keyRef.current);
      };
    }, [children, priority, store]);

    return null;
  };

  return { Provider, Slot, Hoist };
};
```

## なぜこれが React のルール違反にならないのか？

React の公式ドキュメントやベストプラクティスでは、「コンポーネントの内部で別のコンポーネントを定義してはいけない」 と強く警告されています。

このルールが存在する理由は、親コンポーネントが再レンダリングされるたびに Child 関数が再生成され、React がそれを「全く別の新しいコンポーネント」と認識してしまうためです。その結果、アンマウントと再マウントが繰り返され、状態（State）が失われたり、パフォーマンスが著しく低下したりします。

### Factory パターンが安全な理由
今回紹介した Component Factory パターンは、一見すると関数内でコンポーネントを作っているため、このルールに抵触しそうに見えます。しかし、この関数（Factory）を呼び出す場所が異なります。

```tsx
// モジュールレベル（トップレベル）で一度だけ実行される
export const { Provider, Slot, Hoist } = createHoistableComponent();

export default function Page() {
  // Provider, Slot, Hoist の参照は常に一定
  return <Provider>...</Provider>;
}
```

このように、createHoistableComponent は**モジュールのトップレベル（コンポーネントの外側）で呼び出されます。そのため、アプリケーションのライフサイクルを通じて Provider や Slot といったコンポーネント関数の参照は不変（Stable）**です。

React から見れば、これらは通常の `function Component() {}` で定義されたコンポーネントと何ら変わりありません。
クロージャを通じて特定のスコープを共有しているだけで、レンダリングの仕組みは標準的な React の挙動に従っています。

ただし、Factory 関数をコンポーネント内部で呼び出すのはNGです。

```tsx
export default function Page() {
  // ❌ これだとレンダリングのたびに別の Provider/Slot クラスが生成される
  const { Provider, Slot } = createHoistableComponent(); 
  return <Provider>...</Provider>;
}
```

Component Factory はあくまでコンポーネント定義を生成するユーティリティであり、hooks のようにレンダリング中に使うものではありません。
この点を守れば、このパターンは安全かつ強力な武器になります。

## まとめ

Component Factory パターンは、複雑な依存関係を持つコンポーネント群を、クリーンかつ型安全に提供するための設計パターンです。
このパターンは、Headless UI ライブラリの内部実装や、大規模なアプリケーションにおける共通機能（トースト通知、モーダル管理、フォーム生成など）で非常に有効です。
「`Context`の`Provider`を毎回書くのが面倒だな」「型引数を毎回渡すのが冗長だな」と感じたときは、ぜひこの Component Factory パターンを思い出してみてください。