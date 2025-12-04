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

---

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
  userNameAtom,
  cartTotalPriceAtom,
  clearCartAtom,
  userIsLoggedInAtom,
  cartItemsAtom,
} from "./atoms";
import { useAtom, useSetAtom, useAtomValue } from "jotai";

...
// コンポーネント内
const userName = useAtomValue(userNameAtom);
const isLoggedIn = useAtomValue(userIsLoggedInAtom);
...
```

---

## なぜ Zustand のように「オブジェクト」にまとめないのか？

Zustandのような単一Storeライブラリを使ったことがある人にはこの疑問が生まれます。

> 「Atomも一つのオブジェクトにまとめて、`atoms.count` みたいにアクセスできればいいのに」

```ts
// Zustandならこう書ける
const userName = useUserStore((state) => state.user.name);
const isLoggedIn = useUserStore((state) => state.user.isLoggedIn);
```

しかし、Jotaiでは **Atomをオブジェクトにまとめるのは悪手（あるいは不可能）** です。

理由は次の通りです。

### JotaiのAtomは「参照の同一性」が命

コンポーネント内や関数内で動的にオブジェクト化すると参照が変わり、無限再レンダリングの原因になります。

### トップレベルで手動オブジェクト化するとDXが悪化する

* Code Splitting による不要Atomの除外が効きにくい

Jotaiの良さを削ぐ結果になります。

---

## 解決策: `import * as`（namespace import）を使う

そこで推奨したいのが、ファイル（モジュール）自体をオブジェクトとして扱うというアプローチです。

関連するAtomをひとつのファイルにまとめ、利用側では次のように **namespace としてimport** します。

### 1. Atom定義側

```ts
// atoms/userAtoms.ts
import { atom } from 'jotai';

export const name = atom('');
export const isLoggedIn = atom(false);
```

```ts
// atoms/cartAtoms.ts
import { atom } from 'jotai';

export const items = atom([]);
export const totalPrice = atom((get) => {
  const cartItems = get(items);
  return cartItems.reduce((sum, item) => sum + item.price, 0);
});
export const clear = atom(null, (_get, set) => set(items, []));
```

### 2. 利用側（コンポーネント）

```tsx
import { useAtomValue, useSetAtom } from 'jotai';
// ✨ namespace import
import * as userAtoms from '@/atoms/userAtoms';
import * as cartAtoms from '@/atoms/cartAtoms';

export const HeaderCartSummary = () => {
  const userName = useAtomValue(userAtoms.name);
  const isLoggedIn = useAtomValue(userAtoms.isLoggedIn);
  const items = useAtomValue(cartAtoms.items);
  const totalPrice = useAtomValue(cartAtoms.totalPrice);
  const clearCart = useSetAtom(cartAtoms.clear);

  if (!isLoggedIn) {
    return <p>ログインしてカートを表示</p>;
  }

  return (
    <div>
      <p>{userName}さんのカート</p>
      <p>商品点数: {items.length}点</p>
      <p>合計: {totalPrice.toLocaleString()}円</p>
      <button onClick={clearCart}>カートを空にする</button>
    </div>
  );
};
```

---

## このパターンのメリット

### 1. 出自（ドメイン）がコード上で明示される

import文を見に行かずとも、どのドメインのAtomなのかが判断できます。

例えば、次のように書いていれば。

```ts
useAtom(cartAtoms.result);
```

**cartAtomsというドメインのAtom**であると一目でわかります。Zustandの `useCartStore(state => state.items)` のような感覚で使えます。

---

### 2. 変数名に `Atom` をつけなくて済む（より簡潔な命名）

これまでは以下のようでした。

* `userAtom`
* `userProfileAtom`
* `isLoggedInAtom`

これからは以下のようになります。

* `userAtoms.data`
* `userAtoms.profile`
* `userAtoms.isLoggedIn`

`useAtom(userAtoms.data)` と書く時点で「これはAtom」であることが明確です。

---

### 3. 関連するAtomをファイルに分ける習慣がつく

namespace importを使うには、関連するAtomをひとつのファイルにまとめる必要があります。

つまり、次のような動機が自然に生まれます。

> 「namespaceで読みやすくするために、Atomを機能ごとに切り分けよう」

その結果、**ディレクトリ構成が綺麗に保たれます**。

---

## まとめ

Atomが増えるのはJotaiの仕様ですが、**コードが散らかるのは仕様ではありません**。

1. オブジェクトに無理にまとめない
2. ファイル（モジュール）単位でまとめる
3. `import * as ns` で呼び出す

これだけで、Jotaiの開発体験は大きく改善されます。

Zustandのまとまり感が恋しくなったJotaiユーザーの方は、ぜひこのパターンを試してみてください。
