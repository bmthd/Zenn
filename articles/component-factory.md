---
title: "コンポーネントを\"生成\"する関数 (Component Factory パターン) でロジックと型をカプセル化する"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

:::message
この記事は [React Tokyo Advent Calendar 2025](https://qiita.com/advent-calendar/2025/react-tokyo) の 4日目の記事です。
:::

## はじめに

モダンなコンポーネントライブラリのソースコードを読んでいて、最近見かけることが増えてきた「Component Factoryパターン」について解説します。

### やりたいことのイメージ

まず、どのようなコードを目指しているのか、具体的な利用例から見てみましょう。
あるページの一部を、共通レイアウト側のヘッダーに「転送（Hoist）」したいという要件があるとします。
各ページのレイアウトを共通にしたいのですが、ヘッダーに表示するボタンは各ページ固有の要件を持っているため、レイアウト側では直接定義できません。
しかし、各ページで共通するレイアウトは維持したい、という状況です。
通常はContextを作成し、それを使う側で`use(Context)`するという方法もありますが、今回は「コンポーネントを生成する関数（Component Factory）」パターンを使って実装します。

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
        {/* ここにHoistされたコンテンツが表示される */}
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

- "use client" ディレクティブを1箇所にまとめられる。
  先程の例を通常のContextを宣言した場合をイメージしてみてください。

  ```tsx: header-context.tsx
  "use client";
  import { createContext, useContext, useState } from "react";

  const HeaderActionContext = createContext<{
    hoistedContent: React.ReactNode;
    setHoistedContent: (content: React.ReactNode) => void;
  } | null>(null);
  ```
  このようなContextを使う側のコンポーネントはすべてクライアントコンポーネントになる必要がある。
  そのため、Contextだけを共通化すると、使う側でContextを読み取るためのコンポーネントを宣言する必要がある。
  一方、Contextを共通化する場合は、読み出しに必要な情報が不足する。
  その点、Component Factoryパターンを使うと、Contextの宣言とそれを使うコンポーネントの宣言を1箇所にまとめることができる。
  その結果、"use client" ディレクティブを1箇所に付与するだけで済む。





## なぜこれが React のルール違反にならないのか？

このパターンは一見、Reactのルールに違反しているように見える場合がある。
しかし、Component FactoryパターンはReactのベストプラクティスに沿った設計です。
詳細な説明については、今後の記事で詳しく解説する予定です。

## まとめ

Component Factoryパターンは、関連するコンポーネント群を動的に生成する強力な手法です。
このパターンを活用することで、より保守性の高いコードを実現できます。