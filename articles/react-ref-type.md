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
const ref = useRef<ComponentRef<'div'>>(null);
```

propsにおけるComponentPropsのrefバージョンと考えるとわかりやすいかもしれません。
どちらを使用するかはお好みで。

#### カスタムコンポーネント

```tsx:custom-components.tsx
import { useRef, type ComponentRef, type FC } from 'react';
import { Component } from './Component';

const Component :FC = () => {
    const ref = useRef<ComponentRef<typeof Component>>(null);

    const handleClick = useCallback(() => {
    ref.current?.confirm();
    }, []);

    return (
        <button onClick={handleClick}>Confirm</button>
        <Component ref={ref} />
    );
}
```

カスタムコンポーネントのrefの型には、ComponentRefを使用します。
コンポーネントの型を型引数に渡してあげることで、そのコンポーネントのrefの型を取得できます。

#### 手動定義

型なので、使用するメソッドのみを手動定義してしまっても動作に支障はありません。

```tsx
const ref = useRef<{ showModal:() => void }>(null);
```

ただし、タイポが検知できなかったり、他のメソッドを使用したくなったときのメンテナンス性や可読性が下がるため、可能な限り型を正確に定義することをおすすめします。

### 定義・公開側コンポーネント

公開側コンポーネントはReactのバージョンによってrefの型が異なります。

#### React18以前

```tsx
const Child = forwardRef<Handle, Props>(({ children },ref) => {
  return ()
 }
);

```

18以前はforwardRefを使用しないとrefという名前のPropを使用できません。
forwardRefは、親コンポーネントに渡されたrefを子コンポーネント側でrefに受け渡し可能な状態に変換してくれる関数です。
コンポーネントは第二引数でrefを受け取ることができるので、それをDOMやuseImperativeHandleに渡せるようになります。

#### React19以降

React19以降で、refはもはやPropsです。
forwardRefの第一型引数に指定していたHandleをrefの型としてそのまま指定してしまえば良いです。





