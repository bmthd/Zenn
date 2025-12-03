---
title: "JotaiのAtom散らかる問題、`import * as Namespace` で解決する"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["jotai", "react", "state-management"]
published: false
---
:::message
この記事は [React Tokyo Advent Calendar 2025](https://qiita.com/advent-calendar/2025/react-tokyo) の 5日目の記事です。
:::

## JotaiのAtom散らかる問題、`import * as Namespace` で解決する

Reactの状態管理において、Jotaiは非常に強力です。しかし、アプリケーションが大きくなるにつれて、誰もが一度は直面する問題があります。

> **「Atom、増えすぎじゃない？」**

今回は、**Namespace Import（`import * as ...`）を活用すべき理由**とメリットをまとめます。

---

## 背景: Jotaiの郷に従うとファイルがカオスになる

Jotaiの基本思想は「Atomic（原子的）」な状態管理です。巨大なストアを持つのではなく、**最小単位の状態（Atom）**を定義し、それらを組み合わせてアプリを作ります。

しかし、真面目にJotaiを使えば使うほど次のようなAtomが量産されます。

* 基本のAtom: primitiveAtom。
* 派生Atom: derivedAtom（Read-only）。
* 書き込み用Atom: writeOnlyAtom（Action）。

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

// コンポーネント内
const [input] = useAtom(userNameAtom);
const [result] = useAtom(cartTotalPriceAtom);
```

---

## なぜ Zustand のように「オブジェクト」にまとめないのか？

Zustandのような単一Storeライブラリを使ったことがある人にはこの疑問が生まれます。

> 「Atomも一つのオブジェクトにまとめて、`atoms.count` みたいにアクセスできればいいのに」

```ts
// Zustandならこう書ける（イメージ）
const state = useStore();
console.log(state.calc.result);
```

しかし、Jotaiでは **Atomをオブジェクトにまとめるのは悪手（あるいは不可能）**になります。

理由は次の通りです。

### ● JotaiのAtomは「参照の同一性」が命

コンポーネント内や関数内で動的にオブジェクト化すると参照が変わり、**無限再レンダリングの原因**になります。

### ● トップレベルで手動オブジェクト化するとDXが悪化する

* TypeScriptの型定義が煩雑
* Code Splitting による不要Atomの除外が効きにくい

Jotaiの良さを削ぐ結果になります。

---

## 解決策: `import * as`（Namespace Import）を使う

そこで推奨したいのが、**「ファイル（モジュール）自体をオブジェクトとして扱う」**というアプローチになります。

関連するAtomをひとつのファイルにまとめ、利用側では次のように **Namespace としてimport** します。

### 1. Atom定義側

```ts
// atoms/userAtoms.ts
import { atom } from 'jotai';

export const name = atom('');
export const isLoggedIn = atom(false);
export const profile = atom((get) => {
  const userName = get(name);
  const loggedIn = get(isLoggedIn);
  return loggedIn ? { name: userName } : null;
});
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
import { useAtom, useAtomValue, useSetAtom } from 'jotai';
// ✨ Namespace Import
import * as userAtoms from '@/atoms/userAtoms';
import * as cartAtoms from '@/atoms/cartAtoms';

export const HeaderCartSummary = () => {
  const [userName] = useAtom(userAtoms.name);
  const [isLoggedIn] = useAtom(userAtoms.isLoggedIn);
  const [items] = useAtom(cartAtoms.items);
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

### 1. 変数名に `Atom` をつけなくて済む（命名の苦しみから解放）

これまでは以下のようでした。

* `userAtom`
* `userProfileAtom`
* `isLoggedInAtom`

これからは以下のようになります。

* `user.data`
* `user.profile`
* `user.isLoggedIn`

`useAtom(user.data)` と書く時点で「これはAtom」であることが明確です。

---

### 2. 出自（ドメイン）がコード上で明示される

`result` という名前だけでは「カート結果」なのか「API結果」なのか判断がつきません。

しかし、次のように書くと明確になります。

```ts
useAtom(cart.result);
```

と書いていれば、**cartというドメインのAtom**であると一目でわかります。Zustandの `state.slice.value` に近い感覚です。

---

### 3. 関連するAtomをファイルに分ける習慣がつく

Namespace Importを使うには、関連するAtomをひとつのファイルにまとめる必要があります。

つまり、次のような動機が自然に生まれます。

> 「Namespaceで読みやすくするために、Atomを機能ごとに切り分けよう」

その結果、**ディレクトリ構成が綺麗に保たれます**。

---

## まとめ

Atomが増えるのはJotaiの仕様ですが、**コードが散らかるのは仕様ではありません**。

1. オブジェクトに無理にまとめない
2. ファイル（モジュール）単位でまとめる
3. `import * as Name` で呼び出す

これだけで、Jotaiの開発体験は大きく改善されます。

Zustandの「まとまり感」が恋しくなったJotaiユーザーの方は、ぜひこのパターンを試してみてください。
