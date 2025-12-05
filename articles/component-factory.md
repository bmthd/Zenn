---
title: "コンポーネントを\"生成\"する関数でロジックと型をカプセル化する"
emoji: "🎄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, typescript, designpattern, yamadaui]
published: true
---

:::message
この記事は [React Tokyo Advent Calendar 2025](https://qiita.com/advent-calendar/2025/react-tokyo) の 4日目の記事です。
:::

## はじめに

モダンなコンポーネントライブラリのソースコードを読んでいて、最近見かけることが増えてきた「Component Factoryパターン」について解説します。
ちなみに、このComponent Factoryパターンは通常のファクトリ関数をもとに私が勝手に名付けたもので、一般的な名称ではありません。
流行らせたい！

### やりたいことのイメージ

まず、どのようなコードを目指しているのか、具体的な利用例から見てみましょう。
あるページの一部を、共通レイアウト側のヘッダーに「転送（Hoist）」したいという要件があるとします。
例えば、SNSサイトで **「ヘッダー部分は共通のデザインを採用したい。でもユーザー詳細ページではヘッダー部分にフォローボタンを配置したい。」** のようなケースです。
レイアウト側の条件分岐で出し分ける事もできますが、レイアウトなのに出し分けの責務をもたせるのは不自然です。
また、そもそもそのページでしか使わないAPIを必要とする場合はレイアウトでの出し分けは不可能です。
Contextを作成し、それを使う側で`use(Context)`するという方法もありますが、今回はパターンを使って実装します。

```tsx: page.tsx
// ページコンポーネント
import { HeaderAction } from './header-action';

export default function Page() {
  return (
    <>
      {/* ここに書いたものがヘッダーに飛んでいく */}
      <HeaderAction.Hoist>
        <Button>このページのみでヘッダーに表示したいコンポーネント</Button>
      </HeaderAction.Hoist>
      
      <div>メインコンテンツ</div>
    </>
  )
};
```

```tsx:layout.tsx
// レイアウトコンポーネント
import { HeaderAction } from './header-action';

export default function Layout({ children }) {
  return (
    // Factoryから生成されたProviderで囲む
    <HeaderAction.Provider>
      <header>
        <heading>example site</heading>
        {/* ここにページからHoistされたコンテンツが表示される */}
        <HeaderAction.Slot />
      </header>
      <body>{children}</body>
      <footer>©️example</footer>
    </HeaderAction.Provider>
  );
}
```

```tsx:header-action/index.tsx
export * as HeaderAction from './header-action';
```

```tsx:header-action/header-action.tsx
"use client";
import { createHoistableComponent } from "@/lib/hoistable-component";

export const { Provider, Slot, Hoist } = createHoistableComponent();
```

このように、`createHoistableComponent` という関数が、**互いに関連し合うコンポーネント群を一括で生成**しています。

これはJavaScriptのクロージャと高階関数の特性を活かし、特定のスコープ（ContextやStore）を共有するコンポーネントを動的に定義するパターンです。

コンポーネントとFactoryの間にある再exportの部分については後述します。

## このパターンのメリット

### "use client" ディレクティブを1箇所にまとめられる

先程の例を通常のContextで宣言した場合をイメージしてみてください。

```tsx: header-context.tsx
"use client";
import { createContext, useState } from "react";

const HeaderActionContext = createContext<{
  hoistedContent: React.ReactNode;
  setHoistedContent: (content: React.ReactNode) => void;
} | null>(null);
```

`use(Context)`を呼び出すコンポーネントはすべてクライアントコンポーネントである必要があります。
そのため、`Context`だけを共通化の対象とする場合、使う側は`Context`を読み取るだけにクライアントコンポーネントを宣言する必要があります。
その点、Component Factoryパターンを使うと、Contextの宣言とそれを使うコンポーネントの宣言を1箇所にまとめることができます。
その結果、"use client" ディレクティブを1箇所に付与するだけで済みます。
これが、コードの可読性と保守性を向上させる大きなメリットとなります。

### 型情報が一箇所にまとまる

複数の関連するコンポーネント群が似通った構成を持つ場合に共通化をしようとしたとき、それぞれのコンポーネントに型引数を渡す必要が出てきます。
場合によっては、型推論をさせるためだけに本来不要な引数を渡す必要が出てくることもあります。
Component Factoryパターンを使うと、特定の引数に基づいて生成されたコンポーネント群が同じ型情報を共有するため、型引数の冗長な指定を避けることができます。

以下は架空のフォームライブラリにおけるComponent Factoryパターンの利用例です。

```tsx: form-factory.tsx
import { createFormComponents } from "@/lib/form-factory";
import { z } from "zod";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

export const { Form: UserForm, Field: UserField } = createFormComponents(UserSchema);

const ItemSchema = z.object({
  title: z.string(),
  price: z.number(),
});

// スキーマを差し替えるだけで、簡単にフォームのパーツを生成できる！
export const { Form: ItemForm, Field: ItemField } = createFormComponents(ItemSchema);
```
```tsx: page.tsx
import { UserForm, UserField } from "./form-factory";

export default function Page() {
  return (
    // 😊 UserSchemaに基づいた型情報が自動的に適用される
    <UserForm>
      <UserField name="name" /> {/* nameは"name" | "age"と推論される型情報を予め持っている！ */}
      <UserField name="age" />
    </UserForm>
  );
}
```

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details 使わない場合
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
```tsx: page.tsx
import { z } from "zod";
import { Form } from "@/components/form";
import { Field } from "@/components/field";

const UserSchema = z.object({
  name: z.string(),
  age: z.number(),
});

export default function Page() {
  return (
    //  😢 毎回スキーマを渡す必要がある…
    <Form schema={UserSchema}>
      <Field schema={UserSchema} name="name" />
      <Field schema={UserSchema} name="age" />
    </Form>
  );
}

// or
type User = z.infer<typeof UserSchema>;

export default function Page() {
  return (
    //  😢 毎回型引数を渡す必要がある…
    <Form schema={UserSchema}>
      <Field<User> name="name" />
      <Field<User> name="age" />
    </Form>
  );
}
```

Fieldコンポーネントは本来スキーマを受け取る必要はありませんが、この例では`name`に型推論を効かせるためだけにわざわざスキーマを渡しています。
そのためだけに、コードが冗長になってしまいます。
`createFormComponents`は内部でスキーマから推論した型情報だけを`Field`にわたすことができるため、実行時のオーバーヘッドを増やすことなく、型情報を共有できています。
:::


### 実装をカプセル化することができる
Component Factoryパターンを使うと、内部で使用するContextやStoreの存在を隠蔽し、利用者にはシンプルなコンポーネントだけを露出できます。
これにより、APIの設計がシンプルになり、コードリーディング時の理解が容易になります。

## 実装例

以下に、先程の`createHoistableComponent`関数の実装例を示します。

https://github.com/bmthd/lift/blob/c8571ad9885b0985d910733eab7862931cc46e61/lib/src/create-slottable-component.tsx

このように、関数の中に定義してreturnする以外は至って普通のReactコンポーネントの実装です。

手前味噌ですが、こちらのライブラリはnpmで公開しており、@bmthd/lift から誰でもインストールして使うことができます。
[Next.jsでの実装例のdemo](https://github.com/bmthd/lift/tree/main/demo/nextjs)も用意していますので、興味があれば触ってみてください。

## なぜこれが React のルール違反にならないのか？

React の公式ドキュメントやベストプラクティスでは、「コンポーネントの内部で別のコンポーネントを定義してはいけない」 と強く警告されています。

このルールが存在する理由は、親コンポーネントが再レンダリングされるたびに Child 関数が再生成され、React がそれを「全く別の新しいコンポーネント」と認識してしまうためです。その結果、アンマウントと再マウントが繰り返され、状態（State）が失われたり、パフォーマンスが著しく低下したりします。

### Factory パターンが安全な理由
今回紹介した Component Factory パターンは、一見すると関数内でコンポーネントを作っているため、このルールに抵触しそうに見えます。しかし、この関数（Factory）を呼び出す場所が異なります。

```tsx
// モジュールレベル（トップレベル）で一度だけ実行される
export const { Provider, Slot, Hoist } = createHoistableComponent();

export default function Page() {
  // Provider, Slot, Hoist の参照は常に一定
  return <Provider>...</Provider>;
}
```

このように、createHoistableComponent はモジュールのトップレベル（コンポーネントの外側）で呼び出されます。
そのため、アプリケーションのライフサイクルを通じて Provider や Slot といったコンポーネント関数の参照は不変（Stable）です。

React から見れば、これらは通常の `function Component() {}` で定義されたコンポーネントと何ら変わりありません。
クロージャを通じて特定のスコープを共有しているだけで、レンダリングの仕組みは標準的な React の挙動に従っています。

ただし、戻り値のオブジェクトを展開せずにそのままexportはできません。

```tsx
// ❌ これはNG
export const HeaderAction = createHoistableComponent();
```

:::details 回避方法
変数に名前空間をつけたい場合は、以下のように一旦変数に分割代入してからモジュールとして再exportしてください。

```tsx
// /header-action/header-action.tsx
export const { Provider, Slot, Hoist } = createHoistableComponent();

// /header-action/index.tsx
export * as HeaderAction from './header-action';

// /page.tsx
import { HeaderAction } from './header-action';

export default function Page() {
  return <HeaderAction.Provider>...</HeaderAction.Provider>;
}
```

参照を安定させながら、名前空間経由でアクセスできます。
:::

また、Factory 関数をコンポーネント内部で呼び出すのはNGです。

```tsx
export default function Page() {
  // ❌️ これだとレンダリングのたびに別の Provider/Slot コンポーネントが生成される
  const { Provider, Slot } = createHoistableComponent(); 
  return <Provider>...</Provider>;
}
```

Component Factory はあくまでコンポーネント定義を生成するユーティリティであり、hooks のようにレンダリング中に使うものではありません。
この点を守れば、このパターンは安全かつ強力な武器になります。

## どのようなライブラリが実装している？

一般的に、Component Factoryパターンは`createXXXXComponent`のような名前で提供されることが多いです。
コンポーネントだけでなく、hooksや、Factoryを生成でき、同時に様々なAPIを生成できます。
Factoryを返す場合、その返り値は`withXXXX`のように、それがどの文脈で使われるのかを示す命名が多いです。
この場合Factoryを生成するFactoryなので、Meta Factory関数とも言えますね。

### Yamada UI

Yamada UIは日本発のコンポーネントライブラリです。
ライブラリとしてのメンテナンス性を高めるために、ほぼ全てのコンポーネントを`createComponent`, `createSlotComponent`で生成しています。
この記事を書こうと思った理由も、私がYamada UI v2の開発に携わる中でこのAPIに感動したことがきっかけです。

https://yamada-ui.com/ja/docs/components/create-component

そしてなんと、Yamada UIはこのAPIを内部で使うだけでなく、ユーザーにも公開しています。
あろうことか、日本語のドキュメント付きで！
ユーザーランドでこのAPIを使用できるケースは少ないので、触ってみてください。
きっと感動するはずです。

### React Call

React Callは、window.confirmのような感覚でモーダルダイアログのような任意のUIを手続き的に呼び出せるようにするライブラリです。

https://react-call.desko.dev/


ダイアログ、トースト通知などのAPIに特化したヘッドレスライブラリで、Component Factoryパターンを活用しています。
`createCallable`関数が型情報と、コンポーネントを受け取れるのが特徴で、その受け取った型情報を徹底的に使い倒している点が非常に面白いです。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details サンプルコード
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
```tsx
import { createCallable } from 'react-call'

interface Props { message: string }
type Response = boolean

const Confirm = createCallable<Props, Response>(({ call, message }) => (
  <div role="dialog">
    <p>{message}</p>
    <button onClick={() => call.end(true)}>Yes</button>
    <button onClick={() => call.end(false)}>No</button>
  </div>
))

export default function Page() {
  const handleClick = async () => {
    await Confirm.call()
  }
  return (
    <Confirm.Root>
      <button onClick={handleClick}>Show Confirm</button>
    </Confirm.Root>
  )
}
```
:::

### TanStack Form

TanStack Formは、2025年にv1がリリースされた、新世代のフォームライブラリです。

https://tanstack.com/form/latest

アプリケーションが使用するべきフォームのパーツを受け取り、それらがフォームのコンテキストにアクセス可能なようにラップしたコンポーネント群を生成する`createFormHook`関数を提供しています。

<!-- textlint-disable ja-technical-writing/ja-no-mixed-period -->
:::details サンプルコード
<!-- textlint-enable ja-technical-writing/ja-no-mixed-period -->
```tsx: from-context.tsx
import { createFormHookContexts } from "@tanstack/react-form";

export const { fieldContext, useFieldContext, formContext, useFormContext } =
  createFormHookContexts();
```

```tsx: form-hook.tsx
import { Button, InputNumber, RadioGroup, Select, TextField } from "@/ui/form";
import { createFormHook } from "@tanstack/react-form";
import { fieldContext, formContext } from "./form-context";

export const { useAppForm, withForm } = createFormHook({
  fieldComponents: {
    TextField,
    InputNumber,
    Select,
    RadioGroup,
  },
  formComponents: {
    SubmitButton,
  },
  fieldContext,
  formContext,
});
```

```tsx: page.tsx
import { useAppForm } from "./form-hook"; // 😊 アプリケーション側の依存がこれだけで済む
import { z } from "zod";

const schema = z.object({
  firstName: z.string().min(2, "First name must be at least 2 characters"),
  lastName: z.string().min(2, "Last name must be at least 2 characters"),
  phoneNumber: z.string().regex(/^\d{10}$/, "Phone number must be 10 digits"),
});

export default function Page() {
  const form = useAppForm({
    validators: { onChange: schema }
  });

  return (
    <form>
      <form.AppField name="firstName" >
        {(field) => <field.TextField label="First Name" />}
      </form.AppField>
      <form.AppField name="lastName" >
        {(field) => <field.TextField label="Last Name" />}
      </form.AppField>
      <form.AppField name="phoneNumber" >
        {(field) => <field.InputNumber label="Phone Number" />}
      </form.AppField>
      <form.SubmitButton>Submit</form.SubmitButton>
    </form>
  );
}
```
:::

もっとあるかも？
見つけたらぜひコメントで教えてください！

## まとめ

Component Factory パターンは、複雑な依存関係を持つコンポーネント群を、クリーンかつ型安全に提供するための設計パターンです。
このパターンは、Headless UI ライブラリの内部実装や、大規模なアプリケーションにおける共通機能（トースト通知、モーダル管理、フォーム生成など）で非常に有効です。
「`Context`の`Provider`を毎回書くのが面倒だな」「型引数を毎回渡すのが冗長だな」と感じたときは、ぜひこの Component Factory パターンを思い出してみてください。