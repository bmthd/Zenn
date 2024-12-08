---
title: "【Go】動的なパラメータを受け取る定数を表現する"
emoji: "🐀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go,typescript]
published: true
---

## はじめに

TypeScriptでEnum的な定数を表現するテクニックとして、オブジェクトを使う方法があります。

```ts
export const END_POINT = {
  USER_LIST: "/user",
  USER_DETAIL: (userId:string) => `/user/${userId}` as const,
} as const satisfies Record<string, string | ((...args:string[]) => string)>

type ExtractReturnType<T> = T extends (...args: any[]) => infer R ? R : T;

export type END_POINT = ExtractReturnType<typeof END_POINT[keyof typeof END_POINT]>
```

オブジェクトのvalueには文字列だけでなく関数をセットすることができます。
使用側に任意の文字列を指定させたいようなユースケースでこのパターンを利用することで、定数に動的にパラメータを埋め込むことが可能になります。

```ts
const fetcher = (url:END_POINT) => fetch(url)

const getUserList = fetcher(END_POINT.USER_LIST)
const getUserDetail = fetcher(END_POINT.USER_DETAIL("1"))
```

定数と同名の型として、関数だった場合にはその戻り値の型を抽出する`ExtractReturnType`を用意することで、使用側で関数だった場合にはその戻り値の型を取得できるようにしています。
このような定数表現を、Golangでも実現できないでしょうか？

```go
package endpoint

import "fmt"

type APIEndpoint interface {
	isEndPoint()
}

type endPointString string

func (endPointString) isEndPoint() {}

type endPointFunc func(arg string) endPointString

func (endPointFunc) isEndPoint() {}

// 再代入禁止のURI エンドポイント定数
var (
  USER_LIST   endPointString = "/user"
  USER_DETAIL endPointFunc   = func(userID string) endPointString {
    return endPointString(fmt.Sprintf("/user/%s", userID))
  }
  ORDER_LIST   endPointString = "/order"
  ORDER_DETAIL endPointFunc   = func(orderID string) endPointString {
    return endPointString(fmt.Sprintf("/order/%s", orderID))
  }
)

```

interfaceとそれを実装するstruct、そのコンストラクタで実現できます！

## 解説

Golangでは、定数を宣言するときは本来`const`を使います。
しかし、`const`には固定のprimitive値しか指定できないため、TypeScirptのように、関数を混在させて宣言することはできません。
そこで、`var`を使います。
`var`で定数を宣言するというと、なにか悪いことをしているような気がしてきますが、その感性は正しいです。
`const`と違い、`var`で宣言した定数は、再代入ができてしまいます。
そこで、代入できる型をinterfaceで制限し、そのinterfaceを実装するstructを定義することで、定数の再代入を防ぎます。

```go
type APIEndpoint interface {
  isEndPoint()
}

type endPointString string

func (endPointString) isEndPoint() {}

type endPointFunc func(arg string) endPointString

func (endPointFunc) isEndPoint() {}
```

ここでは、`APIEndpoint`というinterfaceを定義し、`isEndPoint`というメソッドを持つことを強制します。
`isEndPoint`メソッドは、それ自体は何もしない空のメソッドです。
これを満たしている型は、`APIEndpoint`として扱うことができます。

加えてそれが定数としての役割を持っているということを大文字スネークケースの命名で示すことで、再代入を防ぎましょう。

## 使い方

```go
package main

import (
  "fmt"
  "net/http"
  "github.com/yourname/yourproject/endpoint"
)

func fetcher(endpoint endpoint.APIEndpoint) (*http.Response, error) {
  return http.Get(string(endpoint))
}

func main() {
  fetcher(endpoint.USER_LIST)
  fetcher(endpoint.USER_DETAIL("1"))
}
```


## おわりに

Golangでは、TypeScriptのようにオブジェクトを使って定数を表現することはできませんが、interfaceとstructを使うことで、同様の表現を実現できます。
このテクニックを使って、Golangでも動的なパラメータを受け取る定数を表現してみてください！