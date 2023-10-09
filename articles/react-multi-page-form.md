---
title: "【React】複数ページに分割し進捗を表示するフォームの設計"
emoji: "⌨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript,nextjs,useReducer]
published: true
---

## 概要

入力する項目が多い入力フォームは離脱率を考えると、1ページにまとめるよりも複数ページに分割したほうが良いとされています。

ではReactで、以下のように複数ページに分割し、進捗を表示するフォームを作成するにはどうすれば良いでしょうか。

![入力フォーム画面](/images/react-multi-page-form/photo1.png)

![入力値確認画面](/images/react-multi-page-form/photo2.png)

## 設計

自分なりに考えた結果、以下のような設計になりました。

- render hooksパターンを使用
今回の要件では、親コンポーネントでフォームのページ数を管理し、小コンポーネント側でフォームの状態を保持させる必要があります。
フォームのページ数が必然的に複数ページに分かれてしまうため、通常であればuseContextやRedux,Recoilなどのグローバルで状態を管理できるライブラリを使ってフォームの状態を保持させることになります。
そこで思いついたのが、render hooksパターン。
複数のページコンポーネントと状態を1つのファイルに同居させることで、グローバルな状態を使用しなくても複数のページから状態を参照できるようになります。

- 状態管理はuseReducerを採用
useStateを使うと、フォームの数1つにつき1つの状態と更新用関数の定義が必要になってしまいます。
useReducerを使えば、フォームの名前を型定義するだけで、フォームの数に関わらず状態管理と更新用関数を定義できます。
初期値はまとめて渡すことができるようになります。
加えて、reducer関数はsetStateと異なり純粋関数なので、テストがしやすくなります。
また、あとから別のフォームが増えたときにreducer関数自体の再利用もできます。

- ページコンポーネントを動的に生成
愚直に複数のページコンポーネントを別々に作成してそれをインポートしても良いでしょう。
凝り性な私は、ページ自体も自動で生成する方法を考えてみました。
入力ページだけではなく、確認画面でも値を表示させる必要があるため、変更の都度複数の箇所を書き換える必要があるためです。
この方法を取れば、フォームの項目を入れ替えたり、追加したりしても、自動的にページが生成されるため、コードの変更箇所が少なく済みます。

実装したコードを紹介します。

:::message
インポート文は省略しています。
CSSを充てただけのラッパーコンポーネントも登場しますが、雰囲気で読み取っていただければと思います。
:::

## ディレクトリ構成

```dir
app
└─ register
    │  page.tsx
    │
    └─ _registerForm
            ConfirmFormCard.tsx
            formData.ts
            index.ts
            RegisterForm.tsx
            types.ts
            useFormPages.tsx
```

Next.jsのApp Routerを使用しているため、本体はpage.tsxです。
コロケーションを意識してフォームの関連ファイルは全て_registerFormディレクトリにまとめています。
pagesルーターや素のReactなどでもフォルダの位置や、呼び出し元ファイルが変わるだけで同様に実装できると思います。

## 型定義

```typescript:types.ts
/**フォームの入力値保管用Key Value */
export type FormValues = {
  email: string;
  password: string;
  password_confirmation: string;
  last_name: string;
  first_name: string;
  last_name_kana: string;
  first_name_kana: string;
  zip_code: string;
  prefecture: string;
  city: string;
  address: string;
  building: string;
  phone_number: string;
  birthday: string;
  blood_type: string;
  gender: string;
　twitter: string;
  emergency_name: string;
  emergency_phone_number: string;
  emergency_relation: string;
};

/**フォーム自動生成用型定義 */
export type InputField<T extends keyof JSX.IntrinsicElements> = {
  key: keyof FormValues;
  title:string;
  wrapperClassName?: string;
  attributes?: T extends keyof JSX.IntrinsicElements
  ? Omit<
  Partial<JSX.IntrinsicElements[T]>,
  'ref'
>
  : never;
  options?: T extends 'select' ? string[]: never;
} 

/**複数のInputFieldをカテゴリ毎に一纏めにするための型定義 */
export type Forms = {
  category:string;
  inputFields: InputField<keyof JSX.IntrinsicElements>[];
}[]
```

1つ目の型は、フォームの入力値を保管するためのもの、2つ目と3つ目は、フォーム自動生成用に書くオブジェクトの型定義です。
InputFieldのattributesには、VSCodeの補完が働くようにしています。
ジェネリクスでHTMLのタグ名を指定してあげれば、そのタグに対応したattributeを補完してくれます。

## フォームの定義

```tsx:formData.ts
export const forms: Forms = [
  {
    category: "ID登録",
    inputFields: [
      {
        key: "email",
        title: "メールアドレス",
        attributes: {
          placeholder: "mail@example.com",
          type: "email",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+.[a-zA-Z]{2,}$",
          autoComplete: "email",
        },
      },
      {
        key: "password",
        title: "パスワード",
        attributes: {
          placeholder: "********",
          type: "password",
          pattern: "^[a-zA-Z0-9]{8,}$",
          autoComplete: "new-password",
        },
      },
      {
        key: "password_confirmation",
        title: "パスワード(確認)",
        attributes: {
          placeholder: "********",
          type: "password",
          pattern: "^[a-zA-Z0-9]{8,}$",
          autoComplete: "new-password",
        },
      },
    ],
  },...] //省略
```

こんな感じで、HTMLタグに設定する属性をオブジェクトで定義することができます。

![フォームの補完](/images/react-multi-page-form/photo3.png =500x)

## フォームのページ数を管理する

ページを管理するための親コンポーネントを作成していきます。

```tsx:RegisterForm.tsx
'use client';

export const initialState: FormValues = {
  email: "",
  password: "",
  password_confirmation: "",
  twitter: "",
  last_name: "",
  first_name: "",
  last_name_kana: "",
  first_name_kana: "",
  zip_code: "",
  prefecture: "",
  city: "",
  address: "",
  building: "",
  phone_number: "",
  birthday: "",
  blood_type: "",
  gender: "",
  emergency_name: "",
  emergency_phone_number: "",
  emergency_relation: "",
};

export const RegisterForm = () => {
  const [formValues, formPages] = useFormPages(forms,initialState);
  const router = useRouter();
  const searchParams = useSearchParams();
  const progress = Number(searchParams.get("progress")) || 0;

  const isFirstPage = progress === 0;
  const isLastPage = progress === formPages.length - 2;
  const isConfirmPage = progress === formPages.length - 1;
  const CurrentPage = formPages[progress];

  const handleBack = () => {
    router.push(`/register?progress=${progress - 1}`);
  };

  const handleConfirm = () => {
    if (isConfirmPage) {
      console.log(formValues);
    } else {
      router.push(`/register?progress=${progress + 1}`);
    }
  };

  return (
    <div className="grid gap-3">
      <div className="flex flex-col gap-3 min-h-[60vh]">{CurrentPage}</div>
      <div className="grid grid-cols-2 gap-3">
        <OutlineButton
          className={`${isFirstPage ? "opacity-0 pointer-events-none disabled" : ""}`}
          onClick={handleBack}>
          戻る
        </OutlineButton>
        <PrimaryButton onClick={handleConfirm}>
          {isConfirmPage ? "登録する" : isLastPage ? "確認画面へ" : "次へ"}
        </PrimaryButton>
      </div>
      <ProgressBar className="m-6" value={progress + 1} max={formPages.length} />
    </div>
  );
};
```

このコンポーネントの関心は、フォームのページ数を管理することと、ページの切り替えです。
この記事の肝であるuseFormPagesというhookに実装を押し込むことによって、このコンポーネントはかなりシンプルになりました。
このフックに定義したフォームのオブジェクトを渡し、フォームの入力値と入力用ページの配列を受け取ることで、`CurrentPage`変数で現在のページを表示しています。
`<CurrentPage />`としたいところですが、そのためにhookの戻り値を関数`() => JSX.Element[]`としてしまうと、useCallbackでも回避不能なアンマウントが発生し、文字を1文字入力するごとにフォーカスが外れてしまうため、変数として埋め込んでいます。

この例ではフォームのページ数の管理はクエリパラメータを使用していますが、Next.js以外を利用する場合はuseStateなどで現在のページ数を保管できます。
進捗の表示はシンプルに`<progress>`タグを使用しています。
定義済みオブジェクトからカテゴリーを取得して表示させても良いかもしれません。
余談ですが、この実装中に初めてHTMLの`<progress>`タグを知りました。すごい！

render hooksパターンとアンマウントに関しては以下の記事で触れられています。

https://zenn.dev/fizumi/articles/083db23e25106e


## フォームの入力値と入力用ページを返すhookを作成する

```tsx:useFormPages.tsx
export const reducer = <T extends { [key: string]: any }>(
  state: T,
  action: { key: keyof T; value: any }
) => {
  return { ...state, [action.key]: action.value };
};

export const useFormPages = (forms: Forms, initialValues:{ [key: string]: any }) => {
  const [formValues, dispatch] = useReducer(reducer, initialValues);

  /**定義済みの値から自動生成した、入力フォームのJSX.Element型配列 */
  const formPages = forms.map((form) => {
    return (
      <div className="grid grid-cols-2 items-end gap-8 px-8 py-2">
        {form.inputFields.map((param) => {
          const handleChange = useCallback(
            (e: React.ChangeEvent<any>) => {
              dispatch({ key: param.key, value: e.currentTarget.value });
            },
            [dispatch]
          );

          if (param.options) {
            return (
              <Select
                id={param.key}
                wrapperClassName={param.wrapperClassName || "col-span-2"}
                labelText={param.title}
                {...(param.attributes as ComponentProps<typeof Select>)}
                value={formValues[param.key]}
                onChange={handleChange}>
                {param.options.map((option) => (
                  <option key={option} value={option}>
                    {option}
                  </option>
                ))}
              </Select>
            );
          } else {
            return (
                <Input
                  id={param.key}
                  wrapperClassName={param.wrapperClassName || "col-span-2"}
                  labelText={param.title}
                  {...(param.attributes as ComponentProps<typeof Input>)}
                  value={formValues[param.key]}
                  onChange={handleChange}
                />
            );
          }
        })}
      </div>
    );
  });

  /** 定義済みの値から自動生成した確認画面のJSX.Element */
  const confirmPage = (
    <>
      {forms.map((form, index) => (
        <div className="grid gap-8 px-8 py-2">
          <ConfirmFormCard inputFields={form.inputFields} page={index} formValues={formValues} />
        </div>
      ))}
    </>
  );

  const pages = [...formPages, confirmPage] as JSX.Element[];

  return [formValues, pages] as const;
};
```

useReducerを使って、フォームの初期値と更新用関数を定義しています。
この記事を書くに当たり、再利用を意識して型パズルをしたので、少し読みづらいかもしれません。
改変前のコードを張っておきます。

```tsx:reducer.tsx
export const reducer = (state: FormValues, action: { key: keyof FormValues; value: string }) => {
  return { ...state, [action.key]: action.value };
};

```

型さえ定義してあれば、reducer関数は数行で済んでしまうから驚きですね。

formPages変数の中ではインポートした定義値を元にフォームの中身を1ページ毎に配列として自動生成しています。
やっていることはラッパーコンポーネントにオブジェクトの値を渡しているだけ。
selectタグとinputタグで分岐して違うコンポーネントを生成しています。
ここの実装を変えるだけで、違うタグを使ったフォームを作ることもできます。

confirm変数では、確認画面に表示されるカード形式の確認用カードを自動生成しています。
ページ番号も渡しているため、そのページに飛ぶボタンも埋め込むことができます。

hookの戻り値はas constを使用することでタプルが使えるようになり、useStateやuseReducerのように[状態, 更新用関数]という形で受け取ることができ、おしゃれです。(関数じゃないケド…)

## あとがき

本来のrender hooksパターンは、使用側が内部の実装を知らなくても使用できるところが魅力ですが、今回は1つのファイルの中で複数のコンポーネントが同居でき、配列として返せる特性を使って実装してみました。
実装していて楽しく、面白い構成だなと思ったので紹介させていただきました。
趣味のコードなので、実装を真似したり、改造して使っていただいても構いません！
ガチガチに各ページのレイアウトをいじる必要があったり、特殊な入力フォームを作成する場合には対応できない可能性があるので、その場合は自動生成させずにコンポーネントを直接変数に埋め込むことになるかもしれません。
また、はじめは再利用することをあまり意識せずに実装していたため、ところどころ型定義が雑な部分があります。
より良い実装方法や、間違いを見つけましたら教えていただけると嬉しいです。
お読みいただきありがとうございました。