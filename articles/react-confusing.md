---
title: "Reactにおける疑問と回答"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,typescript]
published: false
---

## はじめに

React、TypeScriptを学習中に遭遇した疑問とその回答をまとめました。

## コンポーネントの型

Reactの関数コンポーネントには種類がいくつもあります。

- `React.FC`
- `React.FunctionComponent`
- `React.VFC`
- `React.VoidFunctionComponent`
- `JSX.Element`
- `React.ReactElement`
- `React.ReactNode`

## 関数の書き方

関数の書き方には2種類あり、微妙に違いがあるようです。
どちらを使うべきかは、議論が分かれるところのようです。

- function()
- () => {}
 
例えば、ReactやNext.js の公式サイトでは、`function()`を使っています。
こだわりのある人は、`() => {}`を使うべきだと主張しています。
classで使用する際は`this`の指すものが変わるので注意すべきですが、関数型コンポーネントでは、どちらを使っても大きな違いはないので好みで選びましょう。
無名関数には`() => {}`を使い、名前付き関数には`function()`を使うというのもおすすめです。

## hookとコンポーネント

```tsx:コンポーネントを返すhook
export const useComponent = () => {
  return <div>Component</div>
}
```

コンポーネントしか返さないhookは文法上何も問題ありませんが、なぜ通常の関数コンポーネントとして定義しないのかが疑問です。
Propsを渡せないし、一見すると使い方がわかりづらいです。
特別な事情がなければ混乱を招くので避けましょう。

```tsx:render hooks
export const useComponentWithState = () => {
  const [state, setState] = useState(0)
  const component = <button onClick={ () => setState( i => i++ ) }></button>
  return [state, component] as const
}
```

```tsx:処理しかしないコンポーネント
'use client';

const ClientLogger = () => {
    useEffect(() => {
        console.log("mounted")
        return () => {
        console.log("unmounted")
    }
  }, [])
  return null
}
```

クライアント側のAPIを使うためにコンポーネントに直接処理を書いているのをたまに見かけます。
そのコンポーネントが何かJSXを返すのであれば良いのですが、何も返さないならばhookとして定義するべきです。

```tsx:
const useClientLogger = () => {
  useEffect(() => {
    console.log("mounted")
    return () => {
      console.log("unmounted")
    }
  }, [])
}
```
