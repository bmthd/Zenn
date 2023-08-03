---
title: "【React】props分岐のベストプラクティス【AまたはB】"
emoji: "🔠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript]
published: true
---

## はじめに

Reactでコンポーネントを作成する際に、複数の条件でpropsを受け取り、受け取ったpropsの型や指定されたprop名に応じて処理を分岐させたい場合のベストプラクティスを考える。
典型的な例で言うと、URL文字列とコールバック関数どちらを受け取るかによって、`<a>`要素か`<button>`要素かを分岐させたい場合がある。
複雑な例では、Container Presentationalパターンで、Containerコンポーネントで受け取る引数によって呼び出すfetch関数を変え、Presentationalコンポーネントで受け取ったデータを表示するというような場合がある。

## 結論

```tsx:Button.tsx
type Props = {
  children?: ReactNode;
  className?: string;
};

type LinkProps = Props & {
  href: string;
  onClick?: never;
};

type ButtonProps = Props & {
  onClick: () => void;
  href?: never;
};

export const Button = ({ children, className, href, onClick }: LinkProps | ButtonProps) => {
  const mergedClassName = `bg-blue-500 hover:bg-blue-600 inline-flex items-center text-white font-bold font-md rounded-md p-4 ${className}`;

  if (onClick) {
    return (
      <button className={mergedClassName} onClick={onClick}>
        {children}
      </button>
    );
  } else {
    const target = href?.startsWith("http") ? "_blank": undefined;
    return (
      <a href={href} className={mergedClassName} target={target}>
        {children}
      </a>
    );
  }
};
```

```tsx:page.tsx
const Page = () => {
  return (
    <div>
      <Button href="https://google.com">Google</Button> // OK
      <Button onClick={() => alert("clicked")}>Alert</Button> // OK

      <Button href="https://google.com" onClick={() => alert("clicked")}>Google</Button> // NG
    </div>
  );
};
```

AのPropsとBのPropsを別の型として定義し、それぞれに含まれないプロパティに関してはnever型で定義することで、呼び出す際に不要なプロパティを指定することを防ぐことができる。

never型の参考

https://typescriptbook.jp/reference/statements/never

## 蛇足

ここまでたどり着くのが長かった…。

検索して出てくる例ではPropsにtypeというプロパティを生やして、その値で判定させる方法が多かった。
この方法では不要なプロパティの指定をしないといけずなんとなく気持ちが悪いので、もっと良い方法はないかを考え始める。

```tsx:Button.tsx
type Props = {
  children?: ReactNode;
  className?: string;
};

type LinkProps = Props & {
  type: "link";
  href: string;
};

type ButtonProps = Props & {
  type: "button";
  onClick: () => void;
};
```

まず初めに、propsの型を判別しようとしたが、コンポーネントで受け取る際にオブジェクトではなく展開された状態で受け取ってしまうため、型の情報が消えてしまっていた。

次に、Union型を使ってみたが、Union型の場合は両方の型に共通するプロパティしか使えないため、`href`と`onClick`の両方を使うことができなかった。

検索して出てきた下記記事のやり方は一見良さそうに見えたが、可読性が悪く初見では難しい。

https://zenn.dev/tnyo43/articles/2014089257521a

最終的には、Union型を使って型を分岐させ、使わないプロパティはnever型を定義することで、不要なプロパティの指定を防ぐことができた。

Reactの記事は多いが、それ故かゆいところに手が届く内容はあったとしても埋もれている気がするので今後も書いていこうと思う。
