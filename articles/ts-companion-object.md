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
実はこれ、名前空間（namespace）も加えた **「三重定義」** が可能なんです。

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
※ Playground等で型定義を確認するための `declare` を用いた例です。実際の実装では `export const Root = ...` のように関数を定義します。
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

## 命名の一貫性：認知負荷を下げる

わざわざ同じ `Item` として重ねないで、`ItemType`、`IItem`、`ItemComponent` などのように一意の名前を与えれば良いのでは？と思うかもしれません。しかし、これはOSSでは問題を引き起こします。

例として、`ButtonGroup`、`Tabs`、`List` という3つのコンポーネントのケースを比較してみます。

### `ButtonGroup.Item`【値のみ】

`ButtonGroup` は複数のボタンを横並びにするなど、子要素をまとめてレイアウトするためのコンポーネントを想定してください。内部で `Item` という子要素コンポーネントを持ちますが、ボタン自体は `children` として渡すだけなので、Item を型として外部に公開する必要はありません。

```tsx
<ButtonGroup.Root>
  <ButtonGroup.Item>Button 1</ButtonGroup.Item>
  <ButtonGroup.Item>Button 2</ButtonGroup.Item>
</ButtonGroup.Root>
```

### `Tabs.Item`【型のみ】

`Tabs` は少し特殊です。`items` をデータとして渡すと、内部で「Tab（操作UI）」と「Panel（表示領域）」のペアを離れた位置に描画するような実装を想定します。この場合、利用者が `Tabs.Item` を JSX の値（コンポーネント）として直接使う場面はなく、実質的に **型としての `Tabs.Item` だけが公開される** ことになります。

```tsx
<Tabs.Root>
  <Tabs.List>
    <Tabs.Tab>Tab 1</Tabs.Tab>
    <Tabs.Tab>Tab 2</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panels>
    <Tabs.Panel>Panel 1</Tabs.Panel>
    <Tabs.Panel>Panel 2</Tabs.Panel>
  </Tabs.Panels>
</Tabs.Root>

// または
<Tabs.Root items={[{ tab: "Tab 1", panel: "Panel 1" }, { tab: "Tab 2", panel: "Panel 2" }]} />
```

### `List.Item`【型 + 値】

`List` は先述の通り、Item をデータとして流し込む使い方もコンポーネントとして配置する使い方も両方サポートします。

```tsx
<List.Root>
  <List.Item name="apple">Apple</List.Item>
  <List.Item name="banana">Banana</List.Item>
</List.Root>

// または
<List.Root items={[{ name: "apple" }, { name: "banana" }]} />
```

`ItemType` / `ItemComponent` スタイルで命名すると、命名がブレます。
`ButtonGroup` には `ButtonGroupItemComponent`、`List` には `ListItemType` と `ListItemComponent`、`Tabs` には値としての Item が存在しないのに `TabsItemType` のような名前を採用することになります。
これは「Itemが型としてしか存在しないのに、わざわざ `Type` を後置するのか？」という不自然さを生みます。
それが型であるとか、コンポーネントであるかはユーザーが宣言する際の文脈で自ずと決まるものであり、わざわざ名前で区別する必要はないのです。
大体、エディタでホバーすればそれがコンポーネントなのか型なのかは分かる話です。

一方、三重定義を用いると `ButtonGroup.Item`、`List.Item`、`Tabs.Item` というパターンで統一できます。`ButtonGroup.Item` が値のみ、`List.Item` が型と値の両方、`Tabs.Item` が型のみであっても、利用者は「Item は `コンポーネント名.Item` で参照できる」というルール一つだけ覚えれば済みます。これはOSSのように学習コストや認知負荷が重要視される洗練されたAPIを設計する場面で特に効いてきます。

## 技術的な前提条件：マージ対象のnamespaceは「型のみ」であるべき

この「型・値・名前空間」の三重コンパニオンを実現するには、重要な前提条件があります。
それは、**「同名の `const` や `function` とマージしたい `namespace` には、型のみを含めること」** です。

なぜか？

TypeScript独自構文である `namespace` が「値」を含んでしまうと、コンパイル後のJavaScriptにも即時関数やオブジェクトとして出力され、ランタイムの挙動を変えてしまうからです。
もし `namespace Item` の中に値の実体が含まれていると、別途定義した `const Item = ...`（値）と名前が衝突してしまいます。

TypeScriptコンパイラは、`namespace` が型情報しか持っていない（＝ランタイムにJSが出力されない）ことを知っている場合のみ、同名の `const` や `function` とのマージを許可します。

:::message
**補足**
本記事の例では、外側の `namespace List` は `Root` や `Item` といった値の宣言（`declare const`）を含んでおり、型のみではありません。これは `namespace List` 自体を同名の `const List` とマージする必要がないためです。
一方、内側の `namespace Item` は `interface Item` や `const Item` とマージする必要があるため、型（`interface Props`）のみを含めています。
つまり、「型のみにすべき」という制約は **すべての `namespace` に対してではなく、同名の値とマージしたい `namespace` に限った話** です。
:::

### import * as では代替できない理由

ファイルを分割して `import * as List from "./list"` のようにインポートする方法では、導入された名前は「値（オブジェクト）」として扱われるため、同名の型定義とマージしてコンパニオンとして宣言することはできません。素直に単一ファイル内で `namespace` と `const` を並べる手法が解法となります。

## 補足:「namespaceは非推奨」は誤解

この設計パターンについてAIと議論すると、「`namespace` は現代のTypeScriptでは非推奨では？」「ESLintで禁止されているのでは？」「Tree Shakingが効かなくなるのでは？」といった指摘をしてくることがあります。最近は設計の相談相手としてまずAIに壁打ちする開発者も多いと思いますが、これらはいずれも誤解です。

### namespaceベースのコンポーネント設計は主要ライブラリで一般的

`Accordion.Item`、`Tabs.List` のように、ドット記法でサブコンポーネントにアクセスするパターンは、Chakra UI、Base UI、Radix UIなど主要なUIライブラリで広く採用されています。これらのライブラリでは、Tree Shakingを効かせるためにnamespaceベースでコンポーネントを構成しており、「`Component.SubComponent`」というAPI設計は業界標準です。

### ESLintの `@typescript-eslint/no-namespace` について

このルールはデフォルトで有効になっていますが、これは `namespace` 自体が非推奨であることを意味しません。アプリケーションコードでは `namespace` を使うケースが稀であり、使ったり使わなかったりでコードがブレるのを防ぐために、デフォルトで有効化されているに過ぎません。

逆に言えば、`namespace` を使う判断をする場合は積極的に活用すべきTypeScriptの機能です。実際、Node.jsのStrip Typesで使われるtsconfigの `erasableSyntaxOnly` オプションでも、値を含まない `namespace` は禁止されていません。

```ts
import * as React from "react";

// 消せないのでNG
enum Fruit {}

// 型情報のみで消去可能！
namespace List {
  export interface Props {}
}

// 値のあるnamespaceは消せないのでNG
namespace Accordion {
  export declare const Item: React.FC;
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?erasableSyntaxOnly=true#code/JYWwDg9gTgLgBAKjgQwM5wEoFNkGN4BmUEIcARFDvmQFA0D09cgEbaDaDIFYMgIgyB2DIOYMAcgHEaWAHYBXUgDEo44PADeAXzqM4gaPVAowaBGDW6B-Bl7NA3cqB75UC-AYEB-mqOQgsqMHixwAMsFSKacOFgAekWDhgURgsKAInOAAFYjB0ZRoVBiZAEgVuQCEGQGiGGzsHJ0B7BjYuPiFrW3tHXGcAQVxcaAATYAhROAUvOAasXAAbZEo4etEPOABJUJAALkwqGAA6ACkAZQANOYBRHqw7EMSgA)

前述の通り、型のみを含む `namespace` はコンパイル後に消去されるため、`erasableSyntaxOnly` の制約も満たします。

参考:
https://qiita.com/yuki153/items/a51878ad6a1ce913af48

### namespaceはむしろTree Shakingのために選択する

「`namespace` を使うとTree Shakingが効かない」という指摘は因果関係が逆です。

オブジェクトにコンポーネントをまとめる方法（`const List = { Root, Item }`）では、バンドラはオブジェクトのプロパティ間に相互参照がある可能性を考慮しなければならず、個別のプロパティを安全に除去できません。つまり、`List.Root` だけを使っていても `List.Item` がバンドルに含まれる可能性があります。

一方、型のみの `namespace` はコンパイル後に消えるため、バンドラから見ると個別の `const Root` と `const Item` が独立して存在しているのと同じです。**コンポーネント同士が相互に参照しないことをコンパイラに明示できるからこそ、Tree Shakingの対象になります。** むしろオブジェクトにまとめる方がTree Shakingを阻害します。

AIとの議論でハルシネーションを安易に信じないようにお気をつけください。

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

「型・値・名前空間」の三重定義を活用することで、Reactコンポーネントの「手軽さ」と「柔軟性」を意味論を統一したまま両立できます。

- **データ駆動**: 型定義として利用しシンプルに実装
- **JSX駆動**: コンポーネントとして利用し細かくカスタマイズ

実現にあたっては、マージ対象の `namespace` を型専用にすることと、宣言順序のグルーピングに注意してください。