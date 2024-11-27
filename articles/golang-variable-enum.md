# はじめに

TypeScriptでEnum的な定数を表現するテクニックとして、オブジェクトを使う方法があります。

const END_POINT = {
  USER_LIST: "/USER"
  USER_DETAIL: (userId:string) => `/user/${userId}` as const
} as const satisfies Record<string, string | (...args string[]) => string>

オブジェクトのパラメータには文字列だけでなく関数をセットすることができます。
使用側に任意の文字列を指定させたいようなユースケースでこのパターンを利用することで、定数に任意値を埋め込むことが可能になります。
このような定数表現を、Golangでも実現できないでしょうか？

# 実装例

# おわりに