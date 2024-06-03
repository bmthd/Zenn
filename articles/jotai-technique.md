---
title: "jotaiをフル活用するテクニック"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに

### 大きいオブジェクトの状態を利用する

### 大量の派生atomを使いやすくする

jotaiの魅力の一つに、派生値をjotai側で管理できるというメリットがあります。
コンポーネントに派生値を書く場合は、重複してしまうことがあります。
jotaiから取得する値を加工する場合は派生atomとして定義するように決めておけば、同一のロジックを再利用できるように派生値を一元管理できます。
しかし、派生値が多くなってくると毎回その値に対してカスタムフックを定義したりしないといけないなど、手間がかかります。
そこで、派生値を一括で管理するためのテクニックを紹介します。

```ts
import { atom, useAtom } from 'jotai'
type CartItem = {
  id: number;
  name: string;
  price: number;
  quantity: number;
};

const cartItemsAtom = atom<CartItem[]>([]);

const totalAtom = atom((get) => {
  const cartItems = get(cartItemsAtom);
  return cartItems.reduce((acc, item) => acc + item.price * item.quantity, 0);
});

const itemCountAtom = atom((get) => {
  const cartItems = get(cartItemsAtom);
  return cartItems.reduce((acc, item) => acc + item.quantity, 0);
});

const isShippingFreeAtom = atomFamily((minPrice: number) =>
  atom((get) => {
    const total = get(totalAtom);
    return total >= minPrice;
  }),
);

const cartItemDeriveAtomsAtom = atom({
  total: totalAtom,
  itemCount: itemCountAtom,
  isShippingFree: isShippingFreeAtom,
} as const);

type CartItemDeriveKey = keyof ExtractAtomValue<typeof cartItemDeriveAtomsAtom>;

type DeriveArg<K extends CartItemDeriveKey> =
  ExtractAtomValue<typeof cartItemDeriveAtomsAtom>[K] extends AtomFamily<infer P, unknown>
    ? P
    : never;

/**
 * カートの派生値を取得する
 * @param key 派生値のキー
 * @param arg 派生値Atomが受け取る引数
 */
export const useCartDeriveValue = <K extends CartItemDeriveKey,A extends DeriveArg<K>>(key: K, arg: A) => {
  const cartItemDeriveAtoms = useAtomValue(cartItemDeriveAtomsAtom);
  const cartItemDeriveAtom = useMemo(() => cartItemDeriveAtoms[key], [cartItemDeriveAtoms, key]);
  
  const atom = typeof cartItemDeriveAtom === "function" ? cartItemDeriveAtom(isAtomFamily(atom,arg)) : cartItemDeriveAtom;
  return useAtomValue(atom);
};
