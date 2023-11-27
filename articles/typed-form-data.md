---
title: "【React】Formの型付けとどう向き合うか"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,react,typescript]
published: false
---

Next.js14でFormActionがStableになりました。
Next.jsのApp Routerでは、クライアントコンポーネント内でサーバーコンポーネントを呼ぶことができないため、要素をChildrenとして渡すCompositionパターンを多用します。

```tsx:Compositionパターン
const Form = () => {
  return (
    <form>
        <Input name="mail" /> // クライアントコンポーネント
        <Input name="password" />
        <ServerComponent />
        <button type="submit" />
    </form>
  )
}
```

クライアントコンポーネントであれば、親コンポーネントにStateを持たせたり、状態管理ライブラリを使うことができますが、
formの子要素にサーバーコンポーネントをChildrenとして渡したい場面もあるため、formタグをサーバーコンポーネントとして定義したくなります。
FormActionの機能について軽くおさらいしておきます。

```tsx:FormAction

```

![FormActionのイメージ](/images/typed-form-data/photo1.png)

Reactが`form`タグを拡張してくれており、onSubmitと同様のFormData型を受け取れるようになっています。
このFormDataが曲者で、値を取得する際にハードコードされたキーを使うことになります。

```tsx:FormDataの例
const mail = formData.get('mail');
const password = formData.get('password');
```

このような宣言は、型チェックを受けることができず、後々フォームの項目が増えた際にその都度コードの変更が必要なため、避けたいところです。

