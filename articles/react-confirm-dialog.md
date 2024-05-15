---
title: "Reactでwindow.confirmと同じようなリーダブルな確認ダイアログを作る"
emoji: "😊"
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

このAPIが優れているのは、ダイアログの見た目や開閉状態の管理を内部でやってくれているため、はいが選ばれたときに何をしたいのか、どんなメッセージを表示させるのかのみに集中できる点です。[^1]
[^1]: `window.confirm`はなぜ`await`を付けずに使用できるのでしょうか。
`window.confirm`は、ダイアログを表示するためにブラウザのUIスレッドをブロックします。
そのため、`window.confirm`の処理が終わるまで、JavaScriptの実行が停止します。
関数を使う側が、その関数を呼び出すことで処理が止まってしまうのを知る方法が無いのは不親切のため、
[参考](https://medium.com/@yn2011/ui%E3%82%B9%E3%83%AC%E3%83%83%E3%83%89%E3%81%A8window-alert-c3aafa8c8f62)

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
ダイアログの見た目を自由にカスタマイズできるようになりました。
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
   isOpenはダイアログが閉じられている状態としてfalse, resolveはまだ無いため、空の関数を設定。
1. ボタンが押されて`confirm`関数が呼ばれます。
   ダイアログを表示するべく、isOpenをtrueに変更し、resolve関数を設定。
   この`resolve`が呼ばれるまでは`await confirm()`が待ち続けるため、ダイアログが閉じられるまで処理が進みません。
2. ダイアログのどちらかのボタンが押されます。
   このとき、stateに保持していたresolve関数が呼ばれます。
   ここで渡した引数が、`await confirm()`の返り値として返り、Promiseが解決されることになります。
   最後に、ダイアログを閉じるため、stateを初期状態に戻します。

ユーザーがボタンを押すまでの状態と、どちらが押されたのかをPromiseの解決で表現することで、`window.confirm`のようなAPIを実現できました！

### UIライブラリの組付け

```ts:index.tsx
import { Button, Dialog as Component } from "@yamada-ui/react";
import { useCallback, useMemo, type FC, type ReactNode } from "react";
import { useConfirmState } from "./hooks";

/**
 * 確認ダイアログを抽象化したカスタムフック
 * ダイアログ内に表示される内容と、確認関数の結果を利用側で実装する
 */
export const useConfirm = () => {
  const { isOpen, confirm, handleOk, handleCancel } = useConfirmState();

  const successButton = useMemo(
    () => (
      <Button bg={"blue.500"} onClick={handleOk}>
        <span>OK</span>
      </Button>
    ),
    [handleOk]
  );
  const cancelButton = useMemo(
    () => (
      <Button onClick={handleCancel}>
        <span>キャンセル</span>
      </Button>
    ),
    [handleCancel]
  );

  const Dialog: FC<{ children: ReactNode }> = useCallback(
    ({ children }) => (
      <Component
        isOpen={isOpen}
        onClose={handleCancel}
        cancel={cancelButton}
        success={successButton}
      >
        {children}
      </Component>
    ),
    [isOpen, handleCancel, cancelButton, successButton]
  );
  return {
    /** ダイアログコンポーネント */
    Dialog,
    /** 確認ダイアログの操作を待機する関数 */
    confirm,
  };
};

```

### 使用側ボタンコンポーネント

## 発展

### HTMLのdialog要素を使った実装

### ダイアログ非表示時に不要な再レンダリングを発生させない方法

ダイアログの中身に親から渡されたPropsやContextを使うと、ダイアログが非表示のときにも再レンダリングが発生してしまいます。
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

これを考えた人マジで天才。

## 参考

- 発案者(おそらく)の記事

https://medium.com/@kch062522/useconfirm-a-custom-react-hook-to-prompt-confirmation-before-action-f4cb746ebd4e

- 丁寧な解説記事

https://qiita.com/Yametaro/items/b6e035fe06530a9f47bc

- Promiseを使うことの意義

https://zenn.dev/yumemi_inc/articles/f6ec17edf13670

## 蛇足

Render Hooksパターンではなく、`useImperativeHandle`を使う方法もあります。


