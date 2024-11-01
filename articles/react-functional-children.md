---
title: "引数付きchildrenで広がるコンポーネントの表現力"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript,nextjs,conform]
published: true
---

## はじめに

Reactのchildrenを関数として受け取ることができるのをご存知でしょうか？
これを活用することでコンポーネントの表現力が向上します。

```tsx:app.tsx
import { FC, ReactNode } from 'react'

export const App: FC = () => {
  return (
    <Component>
      {() => <h1>こんにちは</h1>} {/* ここがchildren */}
    </Component>
  )
}

const Component: FC<{ children:() => ReactNode }> = ({ children }) => {
  return <div>{children()}</div>
}
```

親コンポーネントで渡した式が、子コンポーネント側のテンプレート部分で実行され、関数の戻り値がレンダリングされます。
何の黒魔術もありません！
TypeScriptでは、`children`の型を`() => ReactNode`にすることで、`{}`内に関数を渡さないと型エラーが出るようになります。

## `children`が関数である例

- Context APIの`<Context.Consumer>`

ReactのContext APIにおける、`<Context.Consumer>`の子要素は関数です。

```tsx:Context.Consumer
import { FC, createContext } from 'react'

const Context = createContext('default')

export const App: FC = () => {
  return (
    <Context.Provider value="value">
      <Context.Consumer>
        {(value) => <h1>{value}</h1>}
      </Context.Consumer>
    </Context.Provider>
  )
}
```

[公式ドキュメント](https://react.dev/reference/react/createContext#consumer)にも、`<Context.Consumer>`の子要素は関数であると書かれています。
`Context.Consumer`を使うと、Contextの値（例では`value`）をテンプレート内で直接参照できるようになり、スコープが`children`内で完結します。
現在は、`useContext`フックが登場したため、`<Context.Consumer>`を使うメリットが薄くなっていますが、`value`変数のスコープを`children`内の式の中に閉じ込められるというのは、依然として有用です。
`children`が関数でも問題ないことがわかります。

:::message
`Context.Consumer`は、テンプレート部分のネストが深くなってしまうという問題があるため、非推奨となっています。
:::

## 何が嬉しいのか？

childrenを関数として渡すと、親コンポーネントから子コンポーネント内部の挙動や状態を自在に制御できます。
例えば、状態に応じて異なるUIを表示したり、動的に生成されるデータに基づいてUIを更新することが可能です。
もし、`children(prop)`を使わずに子コンポーネントの内部にアクセスしようとすると、Context APIや、`useImperativeHandle`などの追加のAPIを使う必要があります。
不必要にグローバルな変数を増やすことになり、そのために必要なコードも増えてしまいます。
また、これらはフックのためClient Componentを強制されますが、'children(prop)'だけであればServer Componentを保つことができます。(後述の例はどちらもSCです。)
Server - Client間の境界をよりパフォーマンスが良いServer側に寄せた実装で行うことができるわけですね。
つまり、以下のような多態性のある表現が変数のスコープをchildren内に狭めながら可能になります。

- 状態を親コンポーネントに公開することなく、親コンポーネント側から中身をカスタマイズする
- ジェネリクスを使い、渡されたpropの型に応じて処理済みの値を返す

実際に、これらのテクニックはライブラリではよく使われています。

## 実装例

```tsx:checkbox.tsx
import { ReactNode, useCallback, useState } from "react";

export const Checkbox: FC<{
  children: FC<{ isChecked: boolean; handleToggle: () => void }> | ReactNode;
}> = ({ children }) => {
  const id = useId();
  const [isChecked, setChecked] = useState(false);
  const handleToggle = useCallback(() => setChecked((prev) => !prev), [isChecked]);
  return (
    <label htmlFor={id}>
      <input id={id} type="checkbox" checked={isChecked} className="sr-only" />
      {typeof children === "function" ? children({ isChecked, handleToggle }) : children}
    </label>
  );
};
```

```tsx:confirmation.tsx
import { Checkbox } from "./checkbox";
import { Check, X } from "./icons";

export const Confirmation: FC = () => {
  return (
    <Checkbox>
      {({ isChecked, handleToggle }) => (
        <div>
          {isChecked ? <Check /> : <X />}
          <button onClick={handleToggle}>確認する</button>
        </div>
      )}
    </Checkbox>
  );
};
```

子要素に渡されたコンポーネントにチェックボックス機能を付与するコンポーネントです。
状態管理を`<Checkbox>`コンポーネントに閉じ込め、`children`を使って状態を参照できるようにしています。
`children`は関数でない場合も考慮して、`typeof children === "function"`で分岐しています。
これにより、状態にアクセスする必要がない場合は従来のように子要素を渡すこともできます。

```tsx:fieldset.tsx
import { FC, ReactNode } from "react";
import { FieldMetadata, getFieldsetProps } from "@conform-to/react";

type FieldsetProps<T extends FieldMetadata<Record<string, any> | undefined, any, any>> = {
  field: T;
  children: ({ field: ReturnType<T["getFieldset"]> }) => ReactNode;
} & Omit<ComponentProps<"fieldset">, "children">;

export const Fieldset = <T extends FieldMetadata<Record<string, any> | undefined, any, any>>({
  field,
  children,
  ...props
}: FieldsetProps<T>) => {
  return (
    <fieldset {...props} {...getFieldsetProps(field)}>
      {children({ field: field.getFieldset() })}
    </fieldset>
  );
};
```

Conformの`FieldMetadata`を使ってフォームのフィールドをラップするコンポーネントです。
Conformでは、fieldオブジェクトに含まれるネストしたフィールドをメソッドチェーンで取得することができ便利なのですが、複雑な入力フォームを作るときは同じメソッドチェーンを何度も繰り返すか、さもなくばテンプレート部分でしか使わない変数を大量に宣言することになります。
コンポーネントが複雑になると、本来であればコンポーネント分割を検討すべきですが、フォームのような共通の目的のあるコンポーネントを分割すると、凝集度が下がり、可読性が落ちてしまいます。
以下の例のように`Fieldset`コンポーネントを使うことで、テンプレート部分で使う変数を`children`内に狭めることができ、同一名称で変数を宣言できるようにしています。

```tsx:fields.tsx
import { FC } from "react";
import { Fieldset } from "./fieldset";
import { TextField, SelectField } from "./form";
import type { FormSchema } from "./schema";
import { useForm } from "@conform-to/react";

export const Fields: FC<{ field: ReturnType<(typeof useForm<FormSchema>)["field"]> }> = ({
  field,
}) => {
  return (
    <>
      <TextField name={field.userName.name} label="ユーザ名" />
      <TextField name={field.email.name} label="メールアドレス" />
      <Fieldset field={field.address}>
        {({ field }) => (
          <>
            <TextField name={field.postalCode.name} label="郵便番号" />
            <TextField name={field.city.name} label="市区町村" />
            <TextField name={field.street.name} label="番地" />
            <TextField name={field.building.name} label="建物名" />
          </>
        )}
      </Fieldset>
      <Fieldset field={field.phone}>
        {({ field }) => (
          <>
            <TextField name={field.number.name} label="電話番号" />
            <SelectField name={field.type.name} label="種類" options={["携帯", "固定電話"]} />
          </>
        )}
      </Fieldset>
    </>
  );
};
```

`FormSchema`は以下のようなJSON構造を持っています。

```ts:schema.ts
export type FormSchema = {
    userName: string,
    email: string,
    address: {
        postalCode: string,
        city: string,
        street: string
        building: string
    },
    phone: {
        number: string,
        type: "携帯" | "固定電話"
    }
}
```

この構造をFieldsetとFieldコンポーネントで視覚的に表現するができ、可読性の高いフォームコンポーネントを作ることができました。

## まとめ

childrenを関数にすることで、コンポーネントの再利用性や拡張性が向上し、UI設計がより柔軟になります。
特に凝集度が高い（1つの目的に集中している）コンポーネントでは、スコープを絞ることができ、コードの可読性が向上します。
ぜひ、複雑なコンポーネントやUI設計で活用してみてください。
