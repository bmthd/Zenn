---
title: "【結論】TypeScriptの型定義はtypeよりinterfaceを使うべき理由"
emoji: "📐"
type: "tech"
topics: ["typescript", "react", "frontend", "performance", "zennfes2025free"]
published: true
---

## はじめに

TypeScriptでコンポーネントのPropsやオブジェクトの型を定義するとき、typeとinterfaceのどちらを使うべきか、一度は悩んだことがあるのではないでしょうか。

巷では「どちらでも良い」「チームで統一されていればOK」といった意見もよく見かけます。
しかし、私は **明確な理由をもって「基本的にはinterfaceを使うべき」** だと主張します。

この記事では私の実体験で遭遇したReactのPropsの深刻なパフォーマンス問題を例に交えながら、なぜinterfaceが優れているのか、そしてtypeはどのような場面で使うべきなのかを解説します。

## type aliasを使いたくなる魅力と、その裏に潜む罠

まず、なぜ多くの開発者がtypeを選びがちなのでしょうか。
それは、開発体験の良さにあります。

`type`で定義した型は、VSCodeなどのエディタでホバーすると、最終的に解決された具体的な型情報がインラインで表示されます。

例として、ReactのPropsを定義してみましょう。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details interfaceの型定義
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

```tsx
export interface IconButtonProps extends HTMLAttributes<HTMLButtonElement> {
  icon: React.ReactNode;
}

export const IconButton = (props: IconButtonProps) => {
  // ...
};
```

:::

![interface ButtonWithIconProps](/images/interface-extends-props/interface.png)

interfaceの場合は名前しか表示されませんでした。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details type aliasの型定義
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->

```tsx
export type IconButtonProps = HTMLAttributes<HTMLButtonElement> & {
  icon: React.ReactNode;
};

export const IconButton = (props: IconButtonProps) => {
  // ...
};
```

:::

![type ButtonWithIconProps = ButtonBaseProps & IconProps & {
  icon: React.ReactNode;
}](/images/interface-extends-props/type.png)

宣言した内容が表示されました。

このように、`type`の場合、合成された型であっても、最終的な構造が一目でわかります。
私もこの「分かりやすさ」を理由に、業務では常に`type`を使うようにしていました。
しかし、これが後に大きな問題を引き起こすことになるのです。

## type aliasが引き起こしたパフォーマンス地獄

私のチームが開発していたプロダクトが成長し、コードベースが大きくなってきたある日のことです。
なんの前触れもなく、エディタの型チェックが異常に遅くなりました。
文字を打つたびにCPUが唸り、一向にTypeScriptの型情報が表示されません。

さらに深刻だったのは、CI/CDでのビルド時間です。それまで1分程度で完了していたtscによる型チェックが、突然30分以上かかるようになったのです。

幸いなことに、この現象はまだマージ前の特定のブランチでのみ発生していたため、対象のブランチでの作業を一時停止し、他のブランチでの開発を続けることができました。

しかし、このブランチでの変更は重要な変更であり、早急にマージしたいものでした。

チームメンバー全員で1週間以上、設定を見直したり、ライブラリをダウングレードしたりと、あらゆることを試しました。しかし、原因は一向に分かりませんでした。

問題が発生したコミットは、共通フォームコンポーネントを使ったフォームフィールドの追加でした。

この共通フォームコンポーネントは、バリデーションライブラリのスキーマから型を推論する仕組みを持つ、比較的複雑な構造をしています。

しかし、さらに大規模かつ複雑な型システムを持つTanStackなどのライブラリでは同様の問題が起きていないことから、「型が複雑だから遅い」という単純な話ではないのではないかと考え始めました。

万策尽きた私は藁にもすがる思いで、ある仮説を試しました。「もしかして、`type`が原因なのでは？」と。

試しにプロジェクト全体の型定義を、機械的に`type`から`interface`に一括置換する正規表現を書き、実行してみました。

すると、どうでしょう。あれだけ私たちを苦しめていたエディタの遅延はなくなり、**ビルド時間は1分に戻った**のです。
犯人は、私たちが便利だと信じて使っていた`type`でした。

:::details もし同様の問題でお困りの場合
ここに、VSCodeの一括置換に使える正規表現を用意して置きました！

`Ctrl + Shift + F`または`Cmd + Shift + F`で検索パネルを開き、正規表現を有効にしたうえで以下の設定で検索・置換できます。

```
Find: ^(\s*)(?:export\s+|declare\s+)*type\s+([A-Za-z_$][\w$]*(?:<[^>]*>)?)\s*=\s*(.+?)\s*&\s*\{\s*\r?\n([\s\S]*?)^\1\}\s*;?

Replace: $1interface $2 extends $3 {
$4$1}
```

複数交差している場合はこちらを繰り返し適用してください。

```
Find: ^(\s*interface\b[^{]*\bextends\b[^{}]*?)\s*&\s*

Replace: $1, 
```

:::

## なぜ`interface`はパフォーマンスに優れるのか？

理由は単純で、**`type`と`interface`では型の計算タイミングが異なる**からです。

### `type`：即時評価（Eager Evaluation）
`type`は**型エイリアス**です。
つまり、既存の型に別の名前を付ける機能です。
TypeScriptコンパイラは`type`を見つけると、その場で型を再帰的に解決・展開し、具体的な1つの型にしてから先に進みます。

<!-- textlint-disable ai-writing/ai-tech-writing-guideline -->
先の例でホバー時に具体的な型が見えていたのは、まさにこの「即時評価」が行われている証拠です。
<!-- textlint-enable ai-writing/ai-tech-writing-guideline -->
交差型 (`&`) が幾重にも重なった複雑な型定義では、この計算コストが指数関数的に増大し、型チェック全体のパフォーマンスを著しく低下させるのです。

### `interface`：遅延評価（Lazy Evaluation）
一方、`interface`は新しい名前付きのオブジェクト型を**宣言**するものです。コンパイラは`interface`を1つの名前（シンボル）として扱います。

`interface`で定義された型は、実際にそのプロパティが参照されるなど、**型情報が必要になるまで、内部構造の完全な計算を遅延させます。** これが遅延評価です。

`interface`にホバーしても名前しか表示されないのは、このためです。


この遅延評価の仕組みにより、`interface`はどんなに複雑に継承されても、プロジェクトがどれだけ巨大になっても、パフォーマンスへの影響を最小限に抑えることができます。大規模なOSSライブラリのコードを見ると、そのほとんどが`interface`で型定義されているのは、このスケーラビリティが理由です。

interfaceのほうがパフォーマンスに優れることが多いという事実は、TypeScriptの公式ドキュメントにも明記されています。

> - Using interfaces with extends can often be more performant for the compiler than type aliases with intersections

訳: interfaceを extends で拡張する方が、型エイリアスを intersection で組み合わせるよりも、コンパイラにとってはパフォーマンスの良いことが多いです。

https://www.typescriptlang.org/docs/handbook/2/everyday-types.html?utm_source=chatgpt.com#type-aliases

## `type`はいつ使うべきか？

では、`type`は全く使うべきでないのでしょうか？ いいえ、そんなことはありません。
**`interface`では表現できない型を定義する場合にのみ、`type`を使うべき**です。

具体的には、以下のようなケースです。

1.  **Union型（合併型）を定義するとき**
    ```ts
    type Status = 'success' | 'error' | 'loading';
    ```

2.  **タプル型を定義するとき**
    ```ts
    type UserTuple = [name: string, age: number];
    ```

3.  **Mapped Typesなど、複雑な型操作をするとき**
    ```ts
    type ReadonlyUser = {
      readonly [K in keyof User]: User[K];
    };
    ```

4.  **プリミティブ型に別名を付けるとき**
    ```ts
    type UserID = string;
    ```

オブジェクトの「形状」を定義する以外の、上記のようなケースで`type`は真価を発揮します。

## おまけ OSSに学ぶ、interfaceにおける可読性を高める工夫

大規模なOSSライブラリでは、`interface`を使いながらも可読性を高めるために、いくつかの工夫がされていました。

### JSDocコメントの活用

![interface PageProps<AppRoute extends AppRoutes>にフォーカスした際、具体的な説明が記載されている](/images/interface-extends-props/nextjs.png)

例えば、Next.jsの型定義では、`interface`に対してJSDocコメントを充実させることで、ホバー時に具体的な説明が表示されるようにしています。

また、interface名を具体的かつ説明的に命名することで、コードを読むだけでその役割が理解できるように工夫されています。

### ユーティリティ型の活用

![alt text](/images/interface-extends-props/hono.png)

Honoの型定義では、ユーザーに公開する直前で`Simplify<T>`というユーティリティ型を使い、ホバー時に見やすくしています。

これは以下のように実装されている、継承に含まれる型を展開するための型ユーティリティです。

```ts
type Simplify<T> = { [K in keyof T]: T[K] } & {};
```

末端で一度だけの利用かつ、含まれるプロパティの数が少ない場合には有効です。



このように工夫することで、`interface`の利点を活かしつつ、開発体験の向上も図ることができます。

## 結論 基本は`interface`、できないことだけ`type`

「どちらかに統一した方が良い」という主張を時々見かけますが、私はこれが間違いだと考えています。
`type`と`interface`はそもそも役割が違うものであり、統一するべきではありません。

私たちが従うべきルールはシンプルです。

> **オブジェクトの形状を定義する際は、まず`interface`を使う。`interface`で表現できない型（Union型など）を定義する必要がある場合に限り、`type`を使う。**

この指針に従うことで、TypeScriptの強力な型システムの恩恵を受けつつ、アプリケーションのパフォーマンスとスケーラビリティを将来にわたって維持できるでしょう。

`type`のホバー時の分かりやすさは魅力的ですが、それはプロジェクトを蝕むパフォーマンス問題と引き換えになる可能性があることを、ぜひ覚えておいてください。

この記事が、`type` vs `interface`論争に1つの結論をもたらし、あなたの開発体験をより良いものにする一助となれば幸いです。

