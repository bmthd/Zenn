---
title: "Reactのrefの使い方 型定義からReact19まで"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript]
published: false
---

## はじめに

### 使用側コンポーネント

#### DOMコンポーネント

```tsx:dom-components.tsx
import { useRef, type FC } from 'react';

const DomComponent :FC = () => {
    const ref = useRef<HTMLDialogElement>(null);

    const handleOpen = useCallback(() => {
    ref.current?.showModal();
    }, []);

    return (
        <button onClick={handleOpen}>Open</button>
        <dialog ref={ref}>
            <p>Dialog content</p>
        </dialog>
    );
}
```

DOMコンポーネントのrefの型には、Web標準で定義されているHTMLElement interfaceを継承したそのDOM要素の型を指定します。
> 例 `div` -> `HTMLDivElement`, `input` -> `HTMLInputElement`, `form` -> `HTMLFormElement` など

上記のようにCamelCaseになっているだけのものはわかりやすいですが、`section`など何も実装が無いため`HTMLElement`をそのまま実装していたり、`h1`-> `HTMLHeadingElement`のように複数のタグが共通の型になっていることもあります。
その他にも、`a` -> `HTMLAnchorElement`など、略称が復元された名前になっていたり、何も実装が無いのに`div` -> `HTMLDivElement`だったりするので、調べてみると面白いかもしれません。
そのコンポーネントがどのinterfaceを実装しているのかは、エディタの補完機能を使って確認できます。

DOMコンポーネントにはもう一つの指定のしかたもあります。

```tsx
const ref = useRef<ElementRef<'div'>>(null);
```

propsにおけるComponentPropsのrefバージョンと考えるとわかりやすいかもしれません。
どちらを使用するかはお好みで。

#### カスタムコンポーネント

```tsx:custom-components.tsx
import { useRef, type ElementRef, type FC } from 'react';
import { Component } from './Component';

const Component :FC = () => {
    const ref = useRef<ElementRef<typeof Component>>(null);

    const handleClick = useCallback(() => {
    ref.current?.confirm();
    }, []);

    return (
        <button onClick={handleClick}>Confirm</button>
        <Component ref={ref} />
    );
}
```

カスタムコンポーネントのrefの型には、ElementRefを使用します。
コンポーネントの型を型引数に渡してあげることで、そのコンポーネントのrefの型を取得できます。

#### 手動定義

型なので、使用するメソッドのみを手動定義してしまっても動作に支障はありません。

```tsx
const ref = useRef<{ showModal:() => void }>(null);
```

ただし、タイポが検知できなかったり、他のメソッドを使用したくなったときのメンテナンス性や可読性が下がるため、可能な限り型を正確に定義することをおすすめします。

### 定義・公開側コンポーネント



