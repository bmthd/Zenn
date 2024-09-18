---
title: "UIライブラリとルーティングFWを組み合わせて型付きLinkを作る"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript,yamadaui,tanstackrouter,nextjs]
published: true
---

## はじめに

UIライブラリから提供されている、デザイン付きのLinkコンポーネントを使いたい。
でも、フレームワークが提供している型付きのhrefやtoも使いたい！
そんな欲張りな要望を満たすために、どちらも使えるLinkコンポーネントを作ります。

![Linkコンポーネントの例](/images/typed-designed-link/link.png)

UIライブラリのコンポーネントは、デザインやスタイルのみを提供しており、renderされるタグは標準HTMLの`<a>`タグとなります。
このままでは遷移がソフトウェアナビゲーションとならず、読み込みによるちらつきが発生してしまいます。
また、ルーティングフレームワークが提供するLinkコンポーネントは、ソフトウェアナビゲーションが実装されており、使用する際にはアプリケーション内のURL以外を指定した際に、型エラーで補足できるようにしてくれるものもありますが、デザインは自分で適用する必要があります。
そこで、コンポーネントライブラリにフレームワークのLinkを実装し、型を付けることで、デザインと遷移、型安全を両立させます。

:::message
テキストリンクのコンポーネントの例ですが、ボタンやアイコン、ページネーションやパンくずリストなど、様々なコンポーネントに適用できます。
現在はYamada UIの例のみ掲載していますが、今後増やすかもしれません。
:::

## YamadaUI × Next.js

```ts:theme/components/link.ts
import { ComponentStyle, LinkProps } from "@yamada-ui/react";
import NextLink from "next/link";

export const Link: ComponentStyle<"Link", LinkProps> = {
  defaultProps: {
    as: NextLink,
  },
};
```

Yamada UIでは、as Propにコンポーネントを指定することで、内部でrenderするコンポーネントを変更することができます。
この機能を利用して、Next.jsのLinkコンポーネントを指定します。
指定の抜け漏れが無いように、Theme機能を利用して、すべてのLinkコンポーネントにNext.jsのLinkコンポーネントを指定します。

https://yamada-ui.com/ja/styled-system/theming/customize-theme

Next.jsでは、typedRoutesという機能を有効にすることで全てのLinkコンポーネントに型が付きます。

```js:next.config.mjs
/** @type {import('next').NextConfig} */
export default {
    experimental: {
      typedRoutes: true,
    },
  };
```

この`Route`型を使って、Yamada UIの`Link`コンポーネントの型を拡張します。

```ts:yamada-ui.d.ts
import type { Component, LinkProps } from "@yamada-ui/react";
import type NextLink from "next/link";
  
declare module "@yamada-ui/react" {
  declare const Link: <T>(
    props: ComponentProps<Component<typeof NextLink<T>, LinkProps>>,
  ) => React.JSX.Element;
}
```

このようにすればいけそうですが、なぜかLinkの型が上書きできず…。
ライブラリ側の型定義が優先されてしまいました。

```ts:yamada-ui.d.ts
import type { ThemeProps } from "@yamada-ui/react";
import type { Route } from "next";
import type { LinkProps as NextLinkProps } from "next/link";
import type { ComponentProps } from "react";

type LinkOptions = {
  /**
   * If `true`, the link will open in new tab.
   *
   * @default false
   */
  isExternal?: boolean;
};

declare module "@yamada-ui/react" {
  interface LinkProps<T = Route>
    extends Omit<NextLinkProps<T>, "as">,
      HTMLUIProps<"a">,
      ThemeProps<"Link">,
      LinkOptions {}
}
```

型エイリアスであるLinkPropsを上書きすることはできました。
ファイルの名前は`**.d.ts`形式で、tsconfig.jsonで読み込みさえされれば何でも構いません。

Yamada UI側のLinkOptionsはexportされていないため、バージョンアップなどで新たな型が増えた場合その都度、こちらにも追加していく必要があります。
より良い方法があればご提示ください！

![Next.js Link](/images/typed-designed-link/next-link.png)

hrefの型をNext.jsのRouteで上書きすることに成功しました。

:::message

Next.jsのTypedRoute、内部の実装を見ると`DynamicRoutes<T>`という型が自動生成されてはいるものの、Dynamic Segmentsの型が取得できないケースがあるようです。
以前は取れていたはずなのですが、私は取れなくなってしまいました…。
![Next.js Dynamic Route Param](/images/typed-designed-link/next-dynamic.png)

現在Next.jsのTypedRoutesはExperimentalな機能となっており、こちらのIssueで議論されています。
https://github.com/vercel/next.js/discussions/55499
:::

## YamadaUI × TanStack Router

```ts
import { Link as TSRLink } from "@tanstack/react-router";
import { type ComponentStyle } from "@yamada-ui/react";

export const Link: ComponentStyle<"Link"> = {
  defaultProps: {
    as: TSRLink,
  },
};
```

TanStack Routerでは、`LinkComponent`という型が提供されており、この型にLinkコンポーネントを渡すことで、型が付いたLinkコンポーネントを作成できます。

```ts:yamada-ui.d.ts
import { type LinkComponent } from "@tanstack/react-router";
import { type Component, type LinkProps } from "@yamada-ui/react";

declare module "@yamada-ui/react" {
  const Link: LinkComponent<Component<"a", LinkProps>>;
}
```

![TanStack Router Link](/images/typed-designed-link/tsr-link.png)

これでtoやparamsを使った遷移を型付きで使えるようになりました！

:::message
このモジュール上書きの方法は、私のプロジェクトではうまくいきましたが、別のリポジトリで試したところ、型が上書きされませんでした。
うまくいかない原因が分かれば、Next.jsの方法も同様の方法で解決できるかもしれません。
どなたかわかりましたらコメントでぜひ教えてください！
:::

また、TanStack Routerでは、`createLink`という関数でコンポーネントをラップするだけで、型が付くLinkコンポーネントを作成できます。

```ts
import { createLink } from "@tanstack/react-router";
import { Button } from "@yamada-ui/react";

export const ButtonLink = createLink(Button);
```

`as`のような仕組みを持たないコンポーネントライブラリでも、これを使うことで簡単に型付きLinkコンポーネントを作成できます。

設定しても型が効かない場合は、d.tsファイルを読み込めていないため、以下の設定を追加してください。

```ts:tsconfig.json
{
  "compilerOptions": {
    ...
  },
    "include": ["yamada-ui.d.ts"]
}
```



## まとめ

Yamada UIとルーティングフレームワークを組み合わせて、型付きLinkコンポーネントを作る方法を紹介しました。
アプリケーション内のリンクは、どんなに気をつけていても間違いが起こりやすい部分です。
以前はこのような仕組みを構築するのに、外部ライブラリを導入する必要がありましたが、最近ではフレームワークが型安全なリンク機能を提供してくれるようになり、手軽に導入することができるようになりました。
型を使って、リンクの遷移先を安全に管理しましょう！
