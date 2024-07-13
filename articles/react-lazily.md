---
title: "【React】named exportのlazy importを綺麗に行う"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react]
published: true
---

Reactのコンポーネントを遅延読み込みする場合に使用する`React.lazy`。

```tsx
import { lazy, Suspense, useState } from 'react';

const MarkdownPreview = lazy(() => import("./MarkdownPreview"));

export default function App() {
  const [show, setShow] = useState(false);
  return (
    <div>
      <button onClick={() => setShow(true)}>show</button>
      <Suspense>
        {show && <MarkdownPreview />}
      </Suspense>
    </div>
  );
}
```

https://react.dev/reference/react/lazy

非同期関数であるimportを内部でよしなにしてくれ、コンポーネントを遅延読み込みすることを可能にしてくれます。
`React.use`に似ていますね。
if文で分岐しているなどで、コンポーネントが必要になるまで読み込まない場合にパフォーマンスを向上させることができます。
`<Suspense />`で囲わないとエラーになるので注意が必要です。
このlazyはdefault exportを想定しており、そのままではnamed exportされたコンポーネントに対しては使用できず、ひと工夫が必要です。

## 個別に対応する方法

以下の.thenによる即時の定型文を書くことで個別にimportを解決することは可能です。

```ts
import { lazy } from "react";

const SomeComponent = lazy(() => import("./foo/bar").then((module) => ({ default: module.SomeComponent })));
```

やっていることは、importしたモジュールの中から希望のexportを選択してそれをdefault exportに変換しているだけです。
ただしこれを毎回書くのは面倒です。

## Utility関数を作成して再利用する方法

https://qiita.com/KokiSakano/items/b6d4e6875443064032b4

こちらの記事で紹介されていた方法です。
util関数を作成しておくことでnamed exportをこの関数に任せることができます。

```ts
import { lazy, type ComponentType } from "react";

export const lazyImport = <T extends { [P in U]: ComponentType<any> }, U extends string>(
  factory: () => Promise<T>,
  name: U,
) => ({
  [name]: lazy(() => factory().then((module) => ({ default: module[name] }))),
});
```

```ts:使用例
const { SomeComponent } = lazyImport(() => import("./foo/bar"), "SomeComponent");
```

左辺で名前を指定しているにも関わらず、右辺にも名前を指定する必要があるのが少し気になります。
また、複数のコンポーネントを同一のモジュールからまとめてインポートすることはできないようです。
ユニオン型で無理やり指定しても型エラーになり、実行時にもエラーが発生しました。

## react-lazilyを使用する方法

良い方法は無いものかと探していたところ、以下のライブラリを見つけました。
ライブラリをインストールすることが許されている場合、react-lazilyを使用することでnamed exportを綺麗にimportすることができます。
https://www.npmjs.com/package/react-lazily

ライブラリとはいうものの関数単体で268byteと軽量なので、依存を増やすことの罪悪感も少ないです。

```ts
import { lazily } from "react-lazily";

const { } = lazily(() => import("@/ui/form/elements"))
```

通常の`React.lazy`と同様の使用感で、named exportされたコンポーネントをそのまま使用することができます。

![複数のコンポーネントを単一のインポートで型補完を効かせながらインポートする図](/images/react-lazily/multi-components.png)

このように複数のコンポーネントを単一のインポートで型補完を効かせながら遅延インポートすることができます。
型定義がLazyExoticComponentにならないところも気に入っています。
記載したJSDocコメントが失われないため、Propsの指定の際も記載したコメントが残ります。

考えてみれば当然のことではありますが、同一のモジュールからインポートした複数のコンポーネントを別のSuspense境界で使用する場合、両方がSuspendされる点は注意が必要です。
同一のSuspense内であれば問題ありません。
以下のリポジトリで念の為雑に検証しました。
ネットワークタブを見ると、両方のコンポーネントが含まれるソースが同時に読み込まれていることが確認できます。
また、片方のコンポーネントが読み込まれると両方がSuspendされています。
<https://bmthd.github.io/lazy-import/>
[ソースコード](<https://github.com/bmthd/lazy-import>)


実装を見に行ったところProxyと、asによる型キャストで実装されていました。

https://github.com/JLarky/react-lazily/blob/main/src/core/lazily.ts

これは思いつかないですね…！

私は使用していませんが、サーバーフレームワークのSSR向けに@loadable/componentというライブラリがあるようで、これとの併用可能なAPIもあるようです。

## まとめ

React.lazyを使用する際にnamed exportを綺麗にlazy importする方法を紹介しました。
react-lazilyは軽量で使いやすいので、lazy importを使用する際にはぜひ使用してみてください。
