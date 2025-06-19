---
title: "Valibotのカスタムスキーマを包括的に扱うベストプラクティス"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [valibot,typescript]
published: true
---

## はじめに

Valibot、使っていますか？

https://valibot.dev/

全てのスキーマが関数であり、pipeやネストでComposableに組み合わせ可能で、スキーマ同士を組み合わせた独自の検証スキーマを作ることができる柔軟性が魅力です。
このカスタムスキーマをどのような命名で扱うかを検討してみました。

## 一般的な使い方

カスタムスキーマを作る場合、愚直に命名するとこうなると思います。

```ts
import * as v from "valibot";

const nameSchema = v.pipe(
  v.string(),
  v.minLength(2),
  v.maxLength(100),
  v.trim(),
);

const carrierMailSchema = v.pipe(
  v.string(),
  v.email(),
  v.regex(/^[a-zA-Z0-9._%+-]+@(docomo|softbank|au|ezweb)\.co\.jp$/),
);

const schema = v.object({
  name: nameSchema,
  email: carrierMailSchema,
  phone: v.pipe(v.string(), ...),
});
```

単にnameなどの端的な命名にすると、他で同じ名前で変数を定義することができなくなったり、スキーマであることがわかりづらくなります。
かといって、上記のような`nameSchema`や`carrierMailSchema`のように命名すると、スキーマであることはわかりますが、長くなってしまいます。
これが、今回のテーマです。

## prefixを使う

スキーマの命名に独自の記号を付けることでそれがスキーマであることを表明します。

```ts
import * as v from "valibot";

const $name = v.pipe(
  v.string(),
  v.minLength(2),
  v.maxLength(100),
  v.trim(),
);
```

例えば、変数名に使用できる記号である`$`を付けるなどです。
命名が短くて済む反面、初見ではそれが何かがわからないデメリットがあります。
チームでこのようにする取り決めをしておき、統一するという方法でカバーは可能です。
ただし、RxJSなど、既に`$`を変数名のプレフィックスにする文化を持っているライブラリが存在しているのでそれらとの混同や、同時に使えないデメリットがあります。

## 独自モジュールを作る

カスタムスキーマをそれぞれexportしておき、使用側でnamespaceインポートする方法です。

```ts
// validation.ts
import * as v from "valibot";

/** アプリケーション内で使用できる名前のスキーマ */
export const name = () => v.pipe(
  v.string(),
  v.minLength(2),
  v.maxLength(100),
  v.trim(),
);

/** キャリアメールのスキーマ */
export const carrierMail = () => v.pipe(
  v.string(),
  v.email(),
  v.regex(/^[a-zA-Z0-9._%+-]+@(docomo|softbank|au|ezweb)\.co\.jp$/),
);

// imple.ts
import * as validation from "../validation";
import * as v from "valibot";

const schema = v.schema({
  name: validation.name(),
  email: validation.carrierMail(),
  phone: v.pipe(v.string(), ...),
});
```

import時に`validation.name`のように名前空間を指定することで、スキーマであることがわかります。
悪くなさそうですが、スキーマ名が長くなってしまいました。
スキーマがネストしたり、pipeするValibotでは少し使いづらそうに感じます。
かと言って、`v`以外のアルファベットにするのは少し気持ち悪さがあります。
他と被らないかつ短いものにしたいですよね。
`m` や `sc` などの省略名も考えましたが、しっくり来るものが見つからずこの案は却下しました。

## ラッパーとして定義する

そこでおすすめしたいのが、私が使っているValibotのラッパーモジュールを定義するという方法です。

```ts
// /lib/validation.ts
import * as v from "valibot";
import { name } from "./name";
import { carrierMail } from "./carrierMail";

export {
  name,
  carrierMail,
};

export * from "valibot";

// imple.ts
import * as v from "@/lib/validation";

const schema = v.object({
  name: v.name(), // ✅valibotと同じ感覚で使える！
  email: v.carrierMail(),
  phone: v.pipe(v.string(), ...)
});
```

これならValibotと全く同じ使い方を保ったまま、カスタムのバリデーション関数を同じ名前空間に定義できます！
Valibotの使い方と合わせるために、スキーマを変数ではなく関数として定義するようにしているのもポイントです。

この手法が優れているのは、ライブラリを使用するための共通のエントリポイントが生まれるため、そこに`v.setGlobalConfig`や`v.setSpecificMessage`などの共通設定が書けるようになることです。
バリデーションライブラリを呼び出すだけで特に関数を呼び出すことなく共通の設定を適用することが可能になります。

関数単位のexportのため、ツリーシェイクへの悪影響もありません。
lintルールとしてValibotの関数を直接使用するのを禁止するルールを作るのがおすすめです。

## 結論：ラッパーモジュール方式が最適

Valibotの使い勝手を最大限に活かしつつ、共通設定や名前空間の統一が可能な「ラッパーモジュール方式」が、私のおすすめするベストプラクティスです。
この方法により、Valibotの柔軟性を保ちながら、チーム全体で一貫したスキーマ定義が可能になります。
ぜひ、あなたのプロジェクトでも試してみてください！