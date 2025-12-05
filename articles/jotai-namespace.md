---
title: "JotaiのAtom散らかる問題、`import * as ns` で解決する"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jotai", "react", "frontend"]
published: true
---
:::message
この記事は [React Tokyo Advent Calendar 2025](https://qiita.com/advent-calendar/2025/react-tokyo) の 5日目の記事です。
:::

## はじめに

Reactの状態管理において、Jotaiは非常に強力です。
しかし、アプリケーションが大きくなるにつれて、誰もが一度は直面する問題があります。

> **「Atom、増えすぎじゃない？」**

今回は、namespace importを活用すべき理由とメリットをまとめます。

## 背景: Jotaiの郷に従うとファイルがカオスになる

Jotaiの基本思想は「Atomic」な状態管理です。
巨大なストアを持つのではなく、 **最小単位の状態（Atom）** を定義し、それらを組み合わせてアプリを作ります。

しかし、真面目にJotaiを使えば使うほど次のようなAtomが量産されます。

* 基本のAtom: primitiveAtom
* 派生Atom: derivedAtom（Read-only）
* 書き込み用Atom: writeOnlyAtom（Action）

これらを素直に実装し、コンポーネントで利用すると、`import` 部分が爆発します。

```ts
// 😫 従来のimport、何がどこのAtomなのかパッと見でわからない
import {
  customerNameAtom,
  isLoggedInAtom,
  cartTotalAtom,
  cartCountAtom,
  clearCartAtom,
} from "./atoms";
import { useAtom, useSetAtom, useAtomValue } from "jotai";

...
// コンポーネント内
const customerName = useAtomValue(customerNameAtom);
const isLoggedIn = useAtomValue(isLoggedInAtom);
const cartTotal = useAtomValue(cartTotalAtom);
const cartCount = useAtomValue(cartCountAtom);
const clearCart = useSetAtom(clearCartAtom);
...
```

## なぜ Zustand のように「オブジェクト」にまとめないのか？

Zustandのような単一Storeライブラリを使ったことがある人にはこの疑問が生まれます。

> 「Atomも一つのオブジェクトにまとめて、`customerAtoms.name` みたいにアクセスできればいいのに」

```ts
import { useCustomerStore } from './stores/customerStore';
import { useCartStore } from './stores/cartStore';
// Zustandならこう書ける
const customerName = useCustomerStore((state) => state.name);
const isLoggedIn = useCustomerStore((state) => state.isLoggedIn);
const cartTotal = useCartStore((state) => state.total);
const cartCount = useCartStore((state) => state.count);
const clearCart = useCartStore((state) => state.clear);
```

しかし、Jotaiでは **Atomをオブジェクトにまとめるのは悪手（あるいは不可能）** です。

理由は次の通りです。

### JotaiのAtomは「参照の同一性」が命

コンポーネント内や関数内で動的にオブジェクト化すると参照が変わり、無限再レンダリングの原因になります。

### トップレベルで手動オブジェクト化するとDXが悪化する

* Code Splitting による不要Atomの除外が効きにくい

Jotaiの良さを削ぐ結果になります。

## 解決策: `import * as`（namespace import）を使う

そこで推奨したいのが、ファイル（モジュール）自体をオブジェクトとして扱うというアプローチです。

関連するAtomをひとつのファイルにまとめ、利用側では次のように **namespace としてimport** します。

### 1. Atom定義側

```ts:atoms/customerAtoms.ts
import { atom } from 'jotai';

interface Customer {
  id: string;
  name: string;
  email: string;
}

export const data = atom<Customer | null>(null);
export const name = atom((get) => get(data)?.name ?? '');
export const email = atom((get) => get(data)?.email ?? '');
export const isLoggedIn = atom((get) => get(data) !== null);
```

```ts:atoms/cartAtoms.ts
import { atom } from 'jotai';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

export const items = atom<CartItem[]>([]);
export const total = atom((get) => {
  const cartItems = get(items);
  return cartItems.reduce((sum, item) => sum + (item.price * item.quantity), 0);
});
export const count = atom((get) => {
  const cartItems = get(items);
  return cartItems.reduce((sum, item) => sum + item.quantity, 0);
});
export const clear = atom(null, (_get, set) => set(items, []));
```

### 2. 利用側（コンポーネント）

```tsx:components/HeaderCartSummary.tsx
import { useAtomValue, useSetAtom } from 'jotai';
// ✨ namespace import
import * as customerAtoms from '@/atoms/customerAtoms';
import * as cartAtoms from '@/atoms/cartAtoms';

export const HeaderCartSummary = () => {
  const customerName = useAtomValue(customerAtoms.name);
  const isLoggedIn = useAtomValue(customerAtoms.isLoggedIn);
  const cartTotal = useAtomValue(cartAtoms.total);
  const cartCount = useAtomValue(cartAtoms.count);
  const clearCart = useSetAtom(cartAtoms.clear);

  if (!isLoggedIn) {
    return <p>ログインしてカートを表示</p>;
  }

  return (
    <div>
      <p>{customerName}さんのカート</p>
      <p>商品点数: {cartCount}点</p>
      <p>合計: {cartTotal.toLocaleString()}円</p>
      <button onClick={clearCart}>カートを空にする</button>
    </div>
  );
};
```

## このパターンのメリット

### 1. 出自（ドメイン）がコード上で明示される

import文を見に行かずとも、どのドメインのAtomなのかが判断できます。

例えば、次のように書いていれば。

```ts
useAtom(cartAtoms.items);
```

**cartAtomsというドメインのAtom**であると一目でわかります。
Zustandの `useCartStore(state => state.items)` のような感覚で使えます。

### 2. 変数名に `Atom` をつけなくて済む（より簡潔な命名）

before:

* `customerNameAtom`
* `isLoggedInAtom`
* `cartTotalAtom`
* `cartCountAtom`
* `clearCartAtom`

after:

* `customerAtoms.name`
* `customerAtoms.isLoggedIn`
* `cartAtoms.total`
* `cartAtoms.count`
* `cartAtoms.clear`

`useAtom(customerAtoms.name)` と書く時点で「これはAtom」であることが明確です。

---

### 3. 関連するAtomをファイルに分ける習慣がつく

namespace importを使うには、関連するAtomをひとつのファイルにまとめる必要があります。

つまり、次のような動機が自然に生まれます。

> 「namespaceで読みやすくするために、Atomを機能ごとに切り分けよう」

その結果、**ディレクトリ構成が綺麗に保たれます**。

### 4. namespaceはネストできる

機能が複雑化した場合、namespaceをネストして更に細かく整理できます。

```bash: ディレクトリ構造のイメージ
atoms/
└── carts
    ├── index.ts ## ここをエントリポイントとして、全てのatomをまとめる
    ├── items.ts
    └── prices.ts
```

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details コード例
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

```ts:atoms/carts/items.ts
import { atom } from 'jotai';

interface CartItem {
  productId: string;
  name: string;
  price: number;
  quantity: number;
}

export const list = atom<CartItem[]>([]);
export const count = atom((get) => {
  const items = get(list);
  return items.reduce((sum, item) => sum + item.quantity, 0);
});
```

```ts:atoms/carts/prices.ts
import { atom } from 'jotai';
import * as itemAtoms from './items';

export const total = atom((get) => {
  const items = get(itemAtoms.list);
  return items.reduce((sum, item) => sum + (item.price * item.quantity), 0);
});
export const tax = atom((get) => Math.floor(get(total) * 0.1));
export const totalWithTax = atom((get) => get(total) + get(tax));
```

```ts:atoms/carts/index.ts
export * as items from './items';
export * as prices from './prices';
```

```tsx:components/cart-summary.tsx
import { useAtomValue } from 'jotai';
import * as cartAtoms from '@/atoms/carts';
// ✅️cartAtoms.items.list, cartAtoms.prices.total のように使える！
const itemCount = useAtomValue(cartAtoms.items.count);
const totalWithTax = useAtomValue(cartAtoms.prices.totalWithTax);
```
:::

## まとめ

Atomが増えるのはJotaiの仕様ですが、**コードが散らかるのは仕様ではありません**。

1. オブジェクトに無理にまとめない
2. ファイル（モジュール）単位でまとめる
3. `import * as ns` で呼び出す

これだけで、Jotaiの開発体験は大きく改善されます。

Zustandのまとまり感が恋しくなったJotaiユーザーの方は、ぜひこのパターンを試してみてください。
