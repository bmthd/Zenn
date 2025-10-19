---
title: "React でモーダル・ダイアログを実装するときの5つのパターン"
emoji: "🚪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript, frontend]
published: false
---

## はじめに

皆さんはDialogの状態管理、どうしていますか。

React でモーダルやダイアログを実装しようとすると、状態管理の方法がいくつも存在します。
要件や規模、使いたいアーキテクチャによって適した方法が変わります。
この記事では代表的なパターンを整理し、それぞれの特徴を比較できるようにします。

## 各パターンの紹介

### 1. 親コンポーネントに useState を持たせる

公式ドキュメントでも紹介されている、もっともシンプルな方法です。
親が `open` state を管理し、Dialog に渡します。

```tsx
import { Dialog, Button } from "@workspaces/ui";
import { useState } from "react";

const ConfirmDialog = () => {
  const [open, setOpeen] = useState(false);

  return (
    <>
      <Button onClick={() => setOpen(true)}>Open Dialog</Button>
      <Dialog open={open} onClose={() => setOpen(false)} />
    </>
  );
};
```

### 2. Render Hooks パターン

専用の Hook が Dialog の状態を管理し、Dialog コンポーネントを返す仕組みです。状態が Hook 内に閉じるため宣言的に扱えます。

```tsx
import { useDisclosure, Dialog as Component, Button } from "@workspaces/ui";
import { useCallback } from "react";

const ConfirmDialog = () => {
  const { Dialog, open } = useDialog();

  return (
    <>
      <Button onClick={open}>Open Dialog</Button>
      <Dialog />
    </>
  );
};

const useDialog = () => {
  const [open, { onOpen, onClose }] = useDisclosure();
  
  const Dialog = useCallback(() => (
    <Component open={open} onClose={onClose} />
  ), [open, onClose]);

  return { Dialog, open };
};
```

### 3. useImperativeHandle + ref

Dialog がハンドルを公開し、親が`Ref`経由で命令的に制御する方式です。

```tsx
import { Dialog as Component, Button } from "@workspaces/ui";
import { useImperativeHandle, type Ref } from "react";

export const ConfirmDialog = () => {
  const dialogRef = useRef<ComponentRef<typeof Dialog>>(null);

  return (
    <>
      <Button onClick={() => dialogRef.current?.open()}>Open Dialog</Button>
      <Dialog ref={dialogRef} />
    </>
  );
}

type ConfirmDialogProps = {
  ref: Ref<{
    onOpen: () => void;
  }>;
};

export const Dialog = ({ ref }: ConfirmDialogProps) => {
  const [open, { onOpen, onClose }] = useDisclosure();

  useImperativeHandle(ref, () => ({
    onOpen,
  }), [onOpen]);

  return <Component open={open} onClose={onClose} />;
}
```

### 4. callback（children as function）

Dialog に children を関数として渡し、`open` や `close` などのコールバックを子に提供する方式です。宣言的に書けます。

```tsx
import { Dialog, Button } from "@workspaces/ui";

export const ConfirmDialog = () => {
  return (
    <Dialog.Root>
      {({ onOpen, onClose }) => (
        <>
          <Button onClick={onOpen}>Open Dialog</Button>
          <Dialog.Content>
            <Dialog.Body>Are you sure you want to proceed?</Dialog.Body>
            <Dialog.Footer>
              <Button onClick={onClose}>Cancel</Button>
              <Button onClick={onClose}>OK</Button>
            </Dialog.Footer>
          </Dialog.Content>
        </>
      )}
    </Dialog.Root>
  );
};

```

### 5. Dialog.OpenTrigger コンポーネント方式

Radix UI などで見られる設計です。`<Dialog.OpenTrigger>` や `<Dialog.Content>` などの専用サブコンポーネントを用意し、宣言的な API として提供します。

```tsx
import { Dialog } from "@workspaces/ui";

export const ConfirmDialog = () => {
  return (
    <Dialog.Root>
      <Dialog.OpenTrigger>
        <Button>Open Dialog</Button>
      </Dialog.OpenTrigger>
      <Dialog.Content>
        <Dialog.Body>Are you sure you want to proceed?</Dialog.Body>
        <Dialog.Footer>
          <Dialog.CloseTrigger>
            <Button>Cancel</Button>
          </Dialog.CloseTrigger>
          <Dialog.CloseTrigger>
            <Button>OK</Button>
          </Dialog.CloseTrigger>
        </Dialog.Footer>
      </Dialog.Content>
    </Dialog.Root>
  );
};
```

これらの実装を以下の観点で比較します。

* アンマウントは発生するか
  アンマウントが発生すると、即座にコンポーネントが破棄されるため、アニメーションやフォーカス管理に影響。
* 再レンダリングの最適化が効くか
その書き方で作成したコンポーネントを使ったときに、親コンポーネントが再レンダリングされるかどうかを確認。
* サーバーコンポーネントとして使用できるか
  RSCにおいて、その書き方で作成したコンポーネントを使ったときに、親コンポーネントに`use client`を付与することなく使用ができるかどうかを確認。
* 宣言的
記述がそれを使用することを宣言する書き方になっているか、つまり副作用的なコードを書かずに済むかを確認します。
* 追加の API が必要か

## パターンごとの比較表

| パターン                   | アンマウントされないか | 再レンダリング最適  | サーバーコンポーネント対応 | 宣言的  | 実装難易度 |
| ------------------------- | --- | --- | --- | --- | --- | 
| 親に useState            　| ⭕ | ❌ | ❌ | ⭕ | ⭕ |
| Render Hooks              | ❌ | ❌ | ❌ | ⭕ | ⭕ |
| useImperativeHandle + ref | ⭕ | ⭕ | ❌ | ❌ | ⭕ |
| callback children          | ⭕ | ⭕ | ❌ | ⭕ | ❌ |
| Dialog.OpenTrigger        | ⭕ | ⭕ | ⭕ | ⭕ | ❌ |

## どう選ぶか

* **小規模・シンプルな実装** → 親に useState / callback
* **状態をコンポーネントに閉じたい** → Render Hooks
* **命令的制御が必要** → useImperativeHandle
* **UI ライブラリ的に理想的な API** → Dialog.OpenTrigger

## まとめ

Dialog の状態管理には複数のアプローチがあり、どれが正解というものはありません。アンマウントや再レンダリング、サーバーコンポーネントとの相性などを比較しながら、自分のユースケースに合った方法を選んでいきましょう。
