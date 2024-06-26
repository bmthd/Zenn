---
title: "Reactでwindow.confirmのように使用できるリーダブルな確認ダイアログを作る"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript]
published: false
---

## はじめに

ブラウザ標準の`window.confirm`を使ったことはありますか？
![confirm](/images/confirm.png)
最もシンプルな確認ダイアログを表示する方法です。
戻り値がboolean型であり、OKボタンが押された場合はtrue、キャンセルボタンが押された場合はfalseを返します。

```ts
const handleClick = () => {
  const result = window.confirm("本当に削除しますか？");
  if (result) {
    // 削除処理
  }
};
```

このAPIが優れているのは、ダイアログの見た目や開閉状態の管理を内部でやってくれているため、はいが選ばれたときに何をしたいのか、どんなメッセージを表示させるのかのみに集中できる点です。

このように、実装者がダイアログの見た目や挙動を気にすることなく、簡単に使えるAPIを作ることができれば、開発効率が向上します。
UIフレームワークのDialog系コンポーネントでもこのAPIを実現できないでしょうか？

```tsx:delete-button.tsx
import { useConfirm } from "@/ui/confirm";
import { Button } from "@yamada-ui/react";
import { type FC, useCallback } from "react";

export const DeleteButton: FC = () => {
  const { Dialog, confirm } = useConfirm();

  const handleClick = useCallback(async () => {
    const result = await confirm();
    if (result) {
      // 削除処理
    }
  }, [confirm]);

  return (
    <>
      <Button onClick={handleClick}>削除</Button>
      <Dialog>本当に削除しますか？</Dialog>
    </>
  );
};
```

`Promise`とRender Hooksで、このようなAPIを実現できます！

:::details　一般的に使用されているダイアログの使い方

```tsx
import { Button, Dialog, useDisclosure } from "@yamada-ui/react";
import { type FC, useCallback } from "react";

export const DeleteButton: FC = () => {
  const { isOpen, onOpen, onClose } = useDisclosure();

  const handleSuccess = useCallback(() => {
    // 削除処理
    onClose();
  }, [onClose]);

  return (
    <>
      <Button onClick={onOpen}>Open</Button>
      <Dialog
        isOpen={isOpen}
        onClose={onClose}
        cancel={<Button onClick={onClose}>キャンセル</Button>}
        success={
          <Button bg={"blue.500"} onClick={handleSuccess}>
            OK
          </Button>
        }
      >
        本当に削除しますか？
      </Dialog>
    </>
  );
};

```

コンポーネント名はボタンなのに、`useDisclosure`で謎の開閉状態が露呈していますね。
ダイアログの開閉状態を同じようなコンポーネントを作成するたびに毎回組み付ける必要があります。
この例では処理が記載されておらず、ダイアログの中身も1行ですが、実際の開発では複雑な処理や複数のコンポーネントが含まれることもあります。
ダイアログの中にダイアログが入ることもあるかもしれません。
ダイアログとそれに使用される状態は必ずセットで使われるべきもののため、それらが単独でインポート可能なAPIを作ることが望ましいです。
共通であり、単調な処理を隠蔽してあげることで何をしたいのかが明確になります。

:::
いかがでしょうか。
ダイアログの状態管理をする必要性のため、カスタムフックとし、内部に状態を隠蔽。
更に文字列だけでなく、内部に好きなコンポーネントを配置できるよう、コンポーネントを返すRender Hooksパターンを採用。
`window.confirm`のように、削除ボタンが押されたときに期待する処理と、ダイアログ内に表示したい内容に集中することができるようになりました。

## 実装

内部の実装を紹介していきます。

### 状態管理

```ts:hooks.ts
import { useCallback, useState } from "react";

type State = {
  isOpen: boolean;
  resolve: (isSuccess: boolean) => void;
};

const initialState: State = {
  isOpen: false,
  resolve: () => {},
};

export const useConfirmState = () => {
  const [{ isOpen, resolve }, setState] = useState<State>(initialState);

  const confirm = useCallback(
    () =>
      new Promise<boolean>((resolve) => {
        setState({ isOpen: true, resolve });
      }),
    [],
  );

  const handleSuccess = useCallback(() => {
    resolve(true);
    setState(initialState);
  }, [resolve]);

  const handleCancel = useCallback(() => {
    resolve(false);
    setState(initialState);
  }, [resolve]);

  return {
    isOpen,
    confirm,
    handleSuccess,
    handleCancel,
  };
};

```

このHooksの処理の流れをイメージしてみましょう。

0. コンポーネントのマウントと共に、初期状態を設定。
   isOpenはダイアログが閉じられている状態としてfalseを設定, resolveはまだ無いため、空の関数を設定。
1. ボタンが押されて`confirm`関数が呼ばれます。
   ダイアログを表示するべく、isOpenをtrueに変更し、resolve関数を設定。
   この`resolve`が呼ばれるまでは`await confirm()`が待ち続けるため、ダイアログが閉じられるまで処理が進みません。
2. ダイアログのどちらかのボタンが押されます。
   このとき、stateに保持していたresolve関数が呼ばれます。
   ここで渡した引数が、`await confirm()`の返り値として返り、Promiseが解決されることになります。
   最後に、ダイアログを閉じるため、stateを初期状態に戻します。

ユーザーがボタンを押すまでの状態と、どちらが押されたのかをPromiseの解決で表現することで、`window.confirm`のようなAPIを実現できました！

### コンポーネントライブラリへの組付け

:::message
[Yamada UI](https://yamada-ui.com/ja)の`Dialog`を使用して説明していますが、他のダイアログ系のコンポーネントでも同様の実装が可能です。
:::

```ts:index.tsx
import { Button, Dialog as Component } from "@yamada-ui/react";
import { useCallback, useMemo, type FC, type ComponentProps } from "react";
import { useConfirmState } from "./hooks";

export const useConfirm = () => {
  const { isOpen, confirm, handleOk, handleCancel } = useConfirmState();

  const Dialog: FC<Omit<ComponentProps<typeof Component>, "isOpen" | "onClose">> = useCallback(
    (props) => (
      <Component
        isOpen={isOpen}
        onClose={handleCancel}
        cancel={<Button onClick={handleCancel}>キャンセル</Button>}
        success={
          <Button bg={"blue.500"} onClick={handleSuccess}>
            OK
          </Button>
        }
        {...props}
      />
    ),
    [isOpen, handleCancel, handleSuccess],
  );
  return {
    /** ダイアログコンポーネント */
    Dialog,
    /** 確認ダイアログの操作を待機する関数 */
    confirm,
  };
};

```

複雑な状態管理を隠蔽したため、コンポーネントへの組付けが簡単になりました。

## 発展

### Render Hooksパターンの問題点

Render Hooksパターンでは、ダイアログ非表示時に不要なコンポーネントアンマウントが発生してしまう問題があります。
状態が変更された際に即座にコンポーネントがアンマウントされるため、アニメーションが付いている場合、再生を待たずにダイアログが閉じてしまいます。

https://www.asobou.co.jp/blog/web/reactfc-renderhooks

この問題の解説は上記記事で詳しく書かれており、この方はコンポーネントを静的に定義し、別で返したPropsを使用側に注入させることで解決しているようです。
この方法では結局使用側が状態を組み付ける必要があるのと、使用側が不必要に状態にアクセスすることを許すことになってしまうのが難点です。

そこで、`useImperativeHandle`を使う方法を考えました。
Render Hooksパターンでは無くなってしまいますが、よりReactらしい方法で解決できます。
閉じるアニメーションが付いている場合はこちらを使うと良いでしょう。

```tsx:index.tsx
import { Button, Dialog } from "@yamada-ui/react";
import { useCallback, useImperativeHandle, forwardRef, type FC, type ComponentProps } from "react";
import { useConfirmState } from "./hooks";

export const ConfirmDialog = forwardRef<
  { confirm: () => Promise<boolean> },
  Omit<ComponentProps<typeof Dialog>, "isOpen">
>((props, ref) => {
  const { isOpen, confirm, handleSuccess, handleCancel } = useConfirmState();
  useImperativeHandle(
    ref,
    () => ({
      confirm,
    }),
    [confirm],
  );

  return (
    <Dialog
      ref={ref}
      isOpen={isOpen}
      onClose={handleCancel}
      cancel={<Button onClick={handleCancel}>キャンセル</Button>}
      success={
        <Button bg={"blue.500"} onClick={handleSuccess}>
          OK
        </Button>
      }
      {...props}
    />
  );
});

```

```tsx:delete-button.tsx
import { ConfirmDialog } from "@/ui/confirm";
import { Button } from "@yamada-ui/react";
import { type FC, useCallback, useRef } from "react";

export const DeleteButton: FC = () => {
  const ref = useRef<{ confirm: () => Promise<boolean> }>(null);

  const handleClick = useCallback(async () => {
    if (await ref.current?.confirm()) {
      console.log("削除処理を実行");
    }
  }, []);

  return (
    <>
      <Button onClick={handleClick}>削除</Button>
      <ConfirmDialog ref={ref}>削除しますか？</ConfirmDialog>
    </>
  );
};

```

`useImperativeHandle`を使うことで、useRef経由で親コンポーネントが子コンポーネントのメソッドを呼び出すことができるようになります。
よく、DOMコンポーネントのメソッドを呼ぶ方法として`useRef`を使うことがありますが、カスタムコンポーネントでもメソッドを公開できるんですね。
`confirm`関数は、`ref.current?.confirm()`で呼び出すことができます。
React19では`forwardRef`しなくても、`ref`を直接渡すことができるようになるので、より使いやすくなりこの方法が流行るかもしれません。

<!-- ### HTMLのdialog要素を使った実装 -->
### ダイアログ非表示時に不要な再レンダリングを発生させない方法

ダイアログの中身にContextを使うと、ダイアログが非表示のときにも再レンダリングが発生してしまいます。
これを防ぐには、中に表示される内容をコンポーネントに分離し、動的インポートすると良いです。

```tsx:delete-button.tsx

import { useConfirm } from "@/ui/confirm";
import { Button } from "@yamada-ui/react";
import { type FC, lazy, useCallback } from "react";

export const DeleteButton: FC<{ id:string }> = ({ id }) => {
  const { Dialog, confirm } = useConfirm();

  const handleClick = useCallback(async () => {
    const result = await confirm();
    if (result) {
      // 削除処理
    }
  }, [confirm]);

  const ItemInformation = lazy(() => import("@/components/ItemInformation"));

  return (
    <>
      <Button onClick={handleClick}>削除</Button>
      <Dialog>
        <ItemInformation id={id} />
      </Dialog>
    </>
  );
};
```

## まとめ

ダイアログの状態管理を隠蔽することで、ダイアログの見た目や挙動に集中できるAPIを作ることができました。
Render Hooksパターンを推す記事として書こうと思っていたのですが、実装する中でアニメーションが消えてしまう問題があることを知り、`useImperativeHandle`を使う方法を提案しました。
このフック自体があまり知られていないような気がするので、ぜひ使ってみてください。

## 参考

- 発案者(おそらく)の記事

https://medium.com/@kch062522/useconfirm-a-custom-react-hook-to-prompt-confirmation-before-action-f4cb746ebd4e

- Promiseを使ったこのパターンの丁寧な解説記事

https://qiita.com/Yametaro/items/b6e035fe06530a9f47bc

- Promiseを使うことの意義を、凝集度と結合度の観点から解説されている記事

https://zenn.dev/yumemi_inc/articles/f6ec17edf13670
