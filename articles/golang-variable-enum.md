# はじめに

TypeScriptでEnum的な定数を表現するテクニックとして、オブジェクトを使う方法があります。

```ts
export const END_POINT = {
  USER_LIST: "/USER"
  USER_DETAIL: (userId:string) => `/user/${userId}` as const
} as const satisfies Record<string, string | (...args string[]) => string>

type EndpointEnum = keyof END_POINT
type END_POINT = typeof END_POINT[EndpointEnum]
```

オブジェクトのパラメータには文字列だけでなく関数をセットすることができます。
使用側に任意の文字列を指定させたいようなユースケースでこのパターンを利用することで、定数に任意値を埋め込むことが可能になります。
このような定数表現を、Golangでも実現できないでしょうか？

```go
type EndPoint interface {
  isEndPoint()
}

type EndPointString string
func (EndPoint) isEndPoint(){}

type EndPointFunc func(...args string) string
func (EndPoint) isEndPoint(){}

var (
  USER_LIST EndPointString("/user")
  USER_DETAIL EndPointFunc(
    func(userID string) string {
      return fmt.Sprintf("/user/%s",userID)
    }
  )
)



```

interfaceとそれを実装するstruct、そのコンストラクタで実現できます！

# 実装例

# おわりに