---
title: "【React】childrenを関数にできるの知ってた？"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript,nextjs,conform]
published: true
---

## はじめに

実はReactの`children`は関数にできます。

```tsx:普通のコンポーネント
import { FC, ReactNode } from 'react'

export const App: FC = () => {
  return (
    <Component>
      <h1>こんにちは</h1> {/* ここがchildren */}
    </Component>
  )
}

const Component: FC<{ children: ReactNode }> = ({ children }) => {
  return <div>{children}</div>
}
```

```tsx:子要素が関数のコンポーネント
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
現在は、`useContext`フックが登場したため、`<Context.Consumer>`を使うメリットが薄くなっていますが、`value`変数のスコープを`children`内の式の中に閉じ込めることができるというメリットがあります。
`children`が関数でも問題ないことがわかります。

:::message
`Context.Consumer`は、テンプレート部分のネストが深くなってしまうという問題があるため、非推奨となっています。
:::

## 何が嬉しいのか？

関数であることで、引数にpropsを渡すことができるようになり、親コンポーネントが子コンポーネントの中身をカスタマイズするための手段が増えます。
`children(prop)`を使わずに子コンポーネントの内部にアクセスしようとすると、Context APIや、`useImperativeHandle`を使う必要があり、不必要にグローバルな変数を増やすことになります。
そのために必要なコードも増えてしまいます。
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
Conformでは、fieldオブジェクトに含まれるネストしたフィールドをメソッドチェーンで取得することができるのですが、複雑な入力フォームを作るときにテンプレート部分でしか使わない変数を大量に宣言することになります。
コンポーネントが複雑になると、本来であればコンポーネント分割を検討すべきですが、フォームで分割を行ってしまうと凝集度が下がってしまいます。
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

この構造をFieldsetとFieldコンポーネントで視覚的に表現することで、可読性の高いフォームコンポーネントを作ることができます。

## まとめ

`children`を関数にするという実装方法を知っておくことで、コンポーネントに多態性を持たせ、多様なUIを実現することができます。
特にConformの例のような、複雑なコンポーネントでは、テンプレート部分の変数のスコープを狭めることができ、可読性の高いコードを書くことができます。
凝集度を高めつつ、コンポーネントの再利用性を高めるために、`children`を関数にすることを検討してみてください。
