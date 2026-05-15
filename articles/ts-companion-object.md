---
title: "【TypeScript】値・型・名前空間の「三重定義」でReactコンポーネントをより柔軟に設計する"
emoji: "📐"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "react", "designpattern", "yamadaui"]
published: true
---
## はじめに

TypeScriptにおける、コンパニオンオブジェクトを知っていますか？
<!-- textlint-disable -->
```ts
export type Rectangle = {
  height: number;
  width: number;
};
 
export const Rectangle = {
  from(height: number, width: number): Rectangle {
    return {
      height,
      width,
    };
  },
};

const rectangle: Rectangle = Rectangle.from(1, 3); // Rectangleという宣言が型と値の両方で使える
```

[サバイバルTypeScript](https://typescriptbook.jp/tips/companion-object)を読んだ方は、型（Type）と値（Value）を同じ名前で定義するテクニックとしてご存知かもしれません。
実はこれ、名前空間（naamespace）も加えた **「三重定義」** が可能なんです。

今回は、この「三重定義」を活用することで実現できる、Reactコンポーネントの柔軟性とシンプルさを両立する設計パターンを紹介します。
加えて、それを実現するための技術的な前提条件や、実装時の注意点についても解説します。

## コンパニオンオブジェクトのおさらい

TypeScriptには「Declaration Merging（宣言のマージ）」という機能があります。
これにより、異なる種類の宣言（型、値、名前空間）が同じ名前を持っていても、互いに衝突せず、1つのエンティティとしてマージされます。

宣言空間は以下の3つに分類されており、これらすべてを重ね合わせることが可能です。

- Type（型：interface, type alias）
- Value（値：const, functionなど）
- Namespace（名前空間：namespace）

なお、`class` や `enum` は型と値の両方を生成するため、単独でコンパニオンオブジェクトのような振る舞いをします。

これらをフル活用して、Reactコンポーネントを設計してみます。

## Reactコンポーネントにおける「三重定義」の実践例

論より証拠、具体的なコードを見てみましょう。
あるリストコンポーネントを実装する際、コンポーネント自体（値）、そのProps定義（型）、そして関連するサブコンポーネントの管理（名前空間）をすべて `List`、あるいは `List.Item` という名前に集約します。
以下は `.tsx` ファイル内での定義イメージです。

```tsx: list.tsx
import * as React from "react";

// 名前空間としての List
namespace List {
  // --- Root (リストの親コンテナ) ---
  
  export namespace Root {
    export interface Props extends React.PropsWithChildren {
      items?: Item[];
    }
  }
  // 値（コンポーネント）としての Root
  export declare const Root: React.FC<Root.Props>;

  // --- Item (リストの要素) ---

  // 1. 型としての Item（データ構造やHTML属性など）
  export interface Item extends React.HTMLAttributes<HTMLLIElement> {
    name?: string;
  }

  // 2. 名前空間としての Item（Item専用のPropsなどを格納）
  export namespace Item {
    export interface Props extends Item, React.PropsWithChildren {}
  }

  // 3. 値（コンポーネント）としての Item
  export declare const Item: React.FC<Item.Props>;
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?#code/JYWwDg9gTgLgBAKjgQwM5wEoFNkGN4BmUEIcARFDvmQNwBQdAdsiFqmHlnADLCrwBvOnDhYAHpFhxmrdp0wQIg4SNETo8YIxhYoBeQAViYdOJ2MAJumx4YAOiMQTAdWAwAFgGF3wADYXKRjghVVU3LBBUAH4ALjgASR0QAG0AXXpQgF8VbJFxSXgLLFxfZEo4XAhGfgUlOJt8OwAxTwAeDEV7RxMAPnoVfI04LR09eUSItXMrTCp7AAkAFQBZbgBBGBgoYAAjAFcdVFal1e54gFFfCKxtHuCVERksWLh+bcYAcwy4bIH1KSeclwXAmpBCqkGUhGun0wLg3VMYmm6FBABpZrYHMZUK4PN4-AEbsFcj8GHl-oViqVypVqvBQfU5s02qCsU5UH06L9oWM4csAJ68fieEg7LTIHZXBFTG4zIX2DpKNkmYkMAD0arggAsGQBmDIAghkA7QyAZ4ZAOsMgGuGQCA-1a6JD4LSagL5SKQGLmJKsNKALxwAAUYGxcUdfBgztdEql2IAlHBPXdwfbNEl0N6GvY9qgsMsIhBWvK7KC0j0fQ9fdHY3BkiWRODQo8WFg4mRkGAwFcyKiqz8O7X7j266xGztkMxmO3O5lu7Wa7Wno3oMOPlgxz2JyXUpORGlJ5HvpQYHsoEFc8G7IrBHYL-72ZlhknPQJwpEb2rOZl+hq4IBqhkAnQyAfoZAHsMgAlDIAXQyAOoMgAhDIAmgyACIMVoWjaFIVFUDqCsGnjpjAJAxqWMZxioCa3hEXoYo06aZtmx78PmSTKhyPo+mWdw+gImSRuiaQ7ioe4HkExahJRCqdD0nYCdRkwCBediPgiN5PJ6TYtm2FSlKgqAAHL1vJOwfAAtJQFhkHAL4iXmoLBJJ0nYrJmlkEOI7IIZJRoOpNnaTp-JYL4vgQAA7oZxk9qJZkSRelnXtINnzp8S7Kc5GmsFpulRYu-nCfxap5meaVwJxb50EAA)

:::message
※ Playground等で型定義を確認するための `declare` を用いた例です。実実装では `export const Root = ...` のように関数を定義します。
:::

## ユースケース：なぜ使い分けるのか

この設計の最大のメリットは、「シンプルに使いたい場合」と「カスタマイズしたい場合」の両方で、セマンティクスを変えずに実装できることです。

### 1. データ駆動でシンプルに使いたい場合

データ（Props）として渡す場合、`List.Item` は **「型」** として振る舞います。

```tsx
interface MyListCombinableProps extends List.Root.Props {}

// アプリケーション色に少しだけカスタマイズしたコンポーネントとして共有
export const MyListCombinable = (props: MyListCombinableProps) => {
  // ここでの List.Item は「型」として参照されている
  const items = React.useMemo<List.Item[]>(
    () => [
      { name: "apple" },
      { name: "banana" },
      { name: "orange" },
    ],
    [],
  );
  
  // データを流し込むだけで描画
  return <List.Root {...props} items={items} />;
};
```

このパターンは、プリミティブなコンポーネントにアプリケーション独自のスタイルやロジックを薄く適用し、「設定済みの部品」として手軽に再利用したい場合に非常に便利です。

### 2. JSXで細かくカスタマイズしたい場合

コンポーネントとして配置する場合、`List.Item` は **「値（コンポーネント）」** として振る舞います。

```tsx
// 各構成パーツを細かくカスタマイズしてガッツリ組み込む
export const MyListCustom = () => {
  // ここでの List.Item.Props は名前空間の中の型を参照
  const itemProps = React.useMemo<List.Item.Props>(() => ({}), []);
  
  return (
    <List.Root>
      {/* ここでの List.Item は「値」として使われている */}
      <List.Item {...itemProps} name="apple" className="bg-red" />
      <List.Item {...itemProps} name="banana" className="bg-yellow" />
      <List.Item {...itemProps} name="orange" className="bg-orange" />
    </List.Root>
  );
};
```

こちらは、レイアウトやスタイルを個別に調整したり、特定のアイテムだけ別の振る舞いをさせたりと、柔軟性が求められる高度な組み込みに適しています。

どちらのケースでも `List.Item` という一貫した名前を使えるため、利用者はコンテキストスイッチなしに実装を選択できます。

## 技術的な前提条件：namespaceは「型のみ」であるべき

この「型・値・名前空間」の三重コンパニオンを実現するには、重要な前提条件があります。
それは、**「namespace 内には型のみを含めること」** です。

なぜか？

TypeScript独自構文である `namespace` が「値」を含んでしまうと、コンパイル後のJavaScriptにも即時関数やオブジェクトとして出力され、ランタイムの挙動を変えてしまうからです。
もし `namespace List` の中に値の実体が含まれていると、別途定義した `const List = ...`（値）と名前が衝突してしまいます。

TypeScriptコンパイラは、`namespace` が型情報しか持っていない（＝ランタイムにJSが出力されない）ことを知っている場合のみ、同名の `const` や `function` とのマージを許可します。

### import * as では代替できない理由

ファイルを分割して `import * as List from "./list"` のようにインポートする場合、一見すると似たようなことができそうですが、これでは完全なコンパニオンオブジェクトにはなりません。

`import * as` はモジュール名前空間オブジェクトを生成する構文であり、インポート先ファイルの中に「型のみしかないこと」を保証するのが難しいためです。
この方法で導入された名前はあくまで「値（オブジェクト）」として扱われるため、同名の型定義と綺麗にマージしてコンパニオンとして宣言することは、（`d.ts` を駆使するなどハック的なことをしない限り）基本的にはできません。

したがって、素直に単一ファイル内で `namespace` と `const` を並べる、あるいは `d.ts` で宣言をマージさせる手法が最も確実です。

## 【重要】宣言順序の罠

この三重定義の際に、非常にハマりやすいポイントがあります。
それは **「宣言の順序」** です。

今回の例で、`Root` は `Item`（型）に依存しています。
プログラマの心理として、「依存されるもの（`interface Item`）を先に書いて、その後に依存するもの（`Root`）を書きたい」と思うかもしれません。
しかし、以下のように書いてしまうとエラーになる場合があります。

```tsx
// ❌ バッドパターン：宣言を分散させてしまう

// 先に型だけ定義
export interface Item extends React.HTMLAttributes<HTMLLIElement> {
  name?: string;
}

// ...間に別のコード（Rootなど）が入る...

export namespace List {
  export namespace Root {
     // ここで Item を使うとき、マージが完了していないとみなされる場合がある
     export interface Props { items?: Item[] } 
  }
}

// 後ろで名前空間や値を定義
export namespace Item { ... } // ここでエラーになる可能性大
```

特に `Root` の定義内で `Item` を参照している状態で、`interface Item` と `namespace Item` の定義が離れていると、TypeScriptコンパイラがマージを正しく処理できず、以下のようなエラーが出ます。

```
Duplicate identifier 'Root'.(2300)
```

これは、分散した定義によって型のインスタンス化のタイミングがズレたり、意図しないスコープの解決が行われるためです。

### ✅ 解決策：素直にまとめる

回りくどいことはせず、**同名の宣言は隣り合わせて書く（グルーピングする）** のが鉄則です。

```tsx
// ⭕️ グッドパターン：同名の宣言はセットにする

// Item 関連
export interface Item {...}
export namespace Item {...}
export const Item = ...

// Root 関連
export namespace Root {...}
export const Root = ...
```

こうすることで、TypeScriptはこれらを「ひとつのマージされたシンボル」としてスムーズに認識してくれます。

## おまけ

このような設計を採用しているライブラリはあるのでしょうか？
あります！
https://yamada-ui.com/ja
私がPRしました！
https://github.com/yamada-ui/yamada-ui/pull/5328

## まとめ

TypeScriptのコンパニオンオブジェクト、特に「型・値・名前空間」の三重定義を活用することで、Reactコンポーネントの設計において「手軽さ」と「柔軟性」という相反する要素を、意味論を統一したまま両立させることができます。

- データ駆動: `MyListCombinable` のように、型定義として利用しシンプルに実装。
- JSX駆動: `MyListCustom` のように、コンポーネントとして利用し細かくカスタマイズ。

ただし、これを実現するには `namespace` を型専用コンテナとして扱いランタイムへの影響を避けることや、宣言順序に気をつけるといった作法が必要です。
これらを理解した上で活用すれば、OSSライブラリのような洗練されたAPIをアプリケーションコードでも実現できるでしょう。