---
title: "Reactのメモ化って結局いつするの?"
emoji: "✍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React]
published: false
---

## はじめに

Reactの概念の中でも、メモ化については理解が難しく後回しにされがちです。
説明用に、一通りまとめておきます。

## メモ化とは

メモ化とは、関数や変数の戻り値をキャッシュしておくことです。

- 関数のメモ化
useCallbackというhookを使用します。
関数への参照を記憶し、同一の引数が渡されたときにはキャッシュした関数を返します。

例

```tsx:useCallback
const [count, setCount] = useState(0);
const increment = useCallback(() => {
    setCount((prev) => prev + 1);
}, []);
```

関数

- コンポーネントのメモ化
`memo()`という関数を使用します。
本来、コンポーネント内のコンポーネントは、親コンポーネントが再レンダリングされたときに、子コンポーネントもまとめて再レンダリングされます。
このmemo
関数や変数を受け取るコンポーネントは全てmemoでメモ化したほうが良いです。

- useMemo


## メモ化のタイミング

### 関数のメモ化

- propsに依存しない関数はコンポーネントの外に書く

コンポーネントはpropsに**反応**して再レンダリングされます。
そのため、propsに依存しない関数はコンポーネントの外に書くことで、メモ化せずとも再レンダリングの度に関数が再定義されることを防ぐことができます。
その関数が他でも使う共通処理の場合は、別ファイルに定義してimportしても同一の参照が使われるため、メモ化する必要はありません。

```tsx:メモ化が不要な例
const handleClick = () => {
    console.log('clicked')
}

const Component = () => {
    return (
        <button onClick={handleClick} />
    )
}
```

- useEffect内でしか使うことがない関数は、useEffect内に書く

useEffectは、その外の変数を参照するときに、その変数を依存配列に含める必要があります。
そのため、関数が再実行されたときにメモ化されていないと、依存配列が変更されたと判断されてしまい、再実行されてしまいます。
useEffect内でしか使わない関数は、useEffect内に書くことで、依存配列に含める必要がなくなります。

```tsx:useEffect内でしか使わない関数
const Component = ({props}:Props) => {

    useEffect(() => {
        const handleClick = () => {
            console.log('clicked')
        }
        return () => {
            console.log('unmounted')
        }
    }, [])

    return (
        <button onClick={handleClick} />
    )
}
```

- propsに依存する関数は、useCallbackでメモ化する

引数で渡ってきた値を使用する関数は、コンポーネントの外に書くことができません。
最終手段として、useCallbackでメモ化することで、再レンダリングの度に関数が再定義されることを防ぐことができます。

```tsx:propsに依存する関数


### コンポーネントのメモ化

- 