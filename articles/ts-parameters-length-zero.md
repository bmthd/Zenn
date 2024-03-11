---
title: "【TS】渡された関数の引数をもとに、引数が0個以上の任意の関数を型推論させる方法"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,react]
published: true
---

## はじめに

業務で遭遇した型パズルの備忘録です。

このような関数を作成していました。
架空のコードなので、あくまでイメージとしてみてください。

```ts
const fetch<T> = ():Promise<Result<T>> => {}

export const useFetch =<T extends typeof fetch> (fetcher:T) => {
const [data,setData] = useState<T>(null);
const [isLoading,setIsLoading] = useState(false);
const [error,setError] = useState<ErrorResponse | null>(null);

const fetch = (arg:any) => {
  setIsLoading(true)
  const result = fetcher(arg);
  if(result.isFailure) serError(result.error)
  setData(result.value)
  setIsLoading(false)
}

return {fetch ,data ,loading ,error}
}
```

```ts:使用側
const fetchUsers = (id: string) => fetch<User>(id);

const route = useRoute()
const { id } = route.params
const { fetch ,data ,isLoading ,error } = useFetch(fetchUsers);

useEffect(async() => {
  await fetch(id)
},[id,fetch])
```

`useFetch` は `fetch` を継承した任意の引数を持つデータ取得関数を受け取り、その関数のデータ取得結果を `useState` で状態管理するフックです。
無数に存在するAPI取得関数全てに状態管理のロジックを書かなくてもフロント側で状態管理ができるようにするためのものです。
このとき、渡される関数の引数は、無いものもあれば複数あるものもあります。
JavaScriptでは、引数が渡されないことを`undefined`として扱うことになりますが、
TypeScriptでは、関数の引数が無いことを表現する型と、引数があることを表現する型が別になっており、
このままだと、使用側で引数が無い関数を渡したときに、`undefined`を渡すことになりとても違和感があります。
そのような型をどう表現するか。

## 解決策

`(...args: never[]) => unknown` と、それをextendsさせたGenericを使うことで解決できます。

```ts
/** 任意の個数の引数を中継するfetch<T>のラッパー関数の型定義 */
type Fetcher<T> = (...args: never[]) => ReturnType<typeof fetch<T>>;

/**
 * 任意の引数を持つデータ取得関数を受け取り、その関数のデータ取得結果を状態管理するフック
 * @param fetcher データ取得関数
 * @template T データ取得関数の戻り値の型 (指定不要)
 * @template U データ取得関数の型 (指定不要)
 */
export const useFetch = <T, U extends Fetcher<T>>(fetcher:U) => {
  const [data,setData] = useState<T> | null>(null);
  const [isLoading,setIsLoading] = useState(false);
  const [error,setError] = useState<ErrorResponse | null>(null);

  const fetch = async (...args:Parameters<U>) => {
    setIsLoading(true)
    const result = await fetcher(...args);
    if(result.isFailure) setError(result.error)
    setData(result.value)
    setIsLoading(false)
  }

  return { fetch ,data ,isLoading ,error }
}
```

`...args: never[]` は、これ自体は値を何も代入できない型ですが、型としては任意のタプル型を代入できる型です。
また、要素が存在しないタプル型は、`[]` と表現され、スプレッド演算子を使って展開すると、何も展開されないことになります。
しかし、関数の引数にnever[]を指定してしまうと使用する際に型の相違が発生します。
そこで、`Parameters` という型を使います。
`Parameters` は、関数の引数の型をタプル型で取得する型で、`Parameters<U>` は、関数 `U` の引数の型を取得することになります。
`Parameters`を経由することで、使用する際には渡された`U`の引数となり、整合性が取れるようになります。

## おまけ

- ReturnType

関数の戻り値を推論してくれるユーティリティ型です。
今回のケースでは、`Fetcher<T>`の戻り値を特定の関数を継承したものと一致するように縛るために使用しています。

- Parameters

関数の引数の型を名前付きタプル型で取得するユーティリティ型です。
名前付きタプル型は、使用される機会が少ないため、あまり知られていないかもしれません。

```ts
type Func = (a: number, b: string) => void;

type Params = Parameters<Func>; // ['a': number, 'b': string]
```

通常のタプルでは、`[number, string]` となりますが、名前付きタプル型では、`['a': number, 'b': string]` となり、付与された名前をキーとして、オブジェクトliteralのようにアクセスできるようになります。

## 参考

こちらの記事のおかげで解決することができました。

https://zenn.dev/junkor/articles/743d57dea9f6cf