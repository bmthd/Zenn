---
title: "【TypeScript】型安全なのにコンパイラに拒絶されるコードと、oneof制約への期待"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript, ジェネリクス, 型安全]
published: false
---
人間が読めば100%型安全だと分かるコードを、なぜTypeScriptコンパイラは受け付けないのでしょうか。

この問いに直面したのは、コマンド名に応じて適切な処理を実行するハンドラーマップを実装しているときのことでした。
シンプルなディスパッチ関数を書いたところで、コンパイラに止められてしまいました。

## このコード、なんで通らないの？

書いたのは次のようなコードです。
コマンド名と処理を定義したオブジェクトを用意し、指定されたコマンドを実行する汎用関数です。

```typescript
const handlers = {
  getUser: (id: string) => Promise.resolve({ id, name: "Alice" }),
  updateUser: (id: string, name: string) => Promise.resolve(true),
  deleteUser: (id: string) => Promise.resolve(),
};

type Handlers = typeof handlers;

export function executeCommand<K extends keyof Handlers>(
  key: K,
  ...args: Parameters<Handlers[K]>
) {
  // ❌ エラーが発生する！
  return handlers[key](...args);
}
```

`key` に応じた引数 `args` を渡し、その実行結果を返すだけのシンプルな実装です。
型定義の整合性は取れているように見えました。

しかし、TypeScriptコンパイラは `handlers[key](...args)` の行で次のエラーを出します。

```text
A spread argument must either have a tuple type or be passed to a rest parameter.
```

## 原因：`T extends A | B | C` は「どれか1つ」を意味しない

この問題の根底にあるのは、ジェネリクス制約に対するTypeScriptの解釈です。

直感的には、`K extends keyof Handlers` （つまり `K extends "getUser" | "updateUser" | "deleteUser"`）と書いたとき、`K` は **「"getUser" か "updateUser" か "deleteUser" のうち、どれか単一の文字列」** になると思ってしまいます。

しかし、TypeScriptの型システムにおいて、この制約は**「`K` が `"getUser" | "updateUser" | "deleteUser"` の部分集合である」**という意味にしかなりません。

つまり、呼び出し側で以下のように `K` に**Union型そのもの**が渡される可能性を、コンパイラは考慮しなければなりません。

```typescript
// K が "getUser" | "updateUser" の Union型として推論されるケース
function dispatch(key: "getUser" | "updateUser") {
  executeCommand(key, ...);
}
```

こうなると、実装内部における型関係は一気に崩壊します。

### 関数のUnionと相関の喪失

`K` が単一のキーではなく Union型になり得るため、コンパイラから見た `handlers[key]` の型は「関数のUnion」になります。

```typescript
((id: string) => Promise<{ id: string; name: string }>)
| ((id: string, name: string) => Promise<boolean>)
| ((id: string) => Promise<void>)
```

関数のUnionを呼び出す場合、TypeScriptはすべての関数に安全に渡せる引数（パラメータの交差型）を要求します。
しかし、`...args` の型である `Parameters<Handlers[K]>` もまたタプル型のUnionになってしまうため、どの関数とどの引数が対応しているかという「相関関係（Correlation）」をTypeScriptは維持できなくなります。

戻り値についても同様です。
明示的な型指定を行わない場合でも、評価される戻り値型は個別のキーに対応した型ではなく Union型となってしまい、特定のキーに対する正確な戻り値との対応関係（相関）を保証できません。

TypeScriptからすれば**「`K` に複数のキーが交ざったUnionが入ってきたら安全性を保障できない」**から拒絶しているわけです。

---

## 暫定対応：高階関数に切り出す

このエラーに即座に対処したいなら、ディスパッチを二段階の呼び出しに分ける方法があります。

`executeCommand(key, ...args)` の代わりに、`key` からハンドラーを取り出す関数を用意します。

```typescript
export const createHandler =
  <K extends keyof Handlers>(key: K): Handlers[K] =>
  (...args: any[]) => {
    const handler = handlers[key] as (...args: any[]) => any;

    return handler(...args);
  };
```

`createHandler` の戻り値の型は `Handlers[K]` と明示されています。
呼び出し側では `createHandler(key)` の時点で戻り値の型計算が確定するため、続けて渡す引数と返ってくる値の型を、コンパイラが個別のキーに対応した型として算出してくれます。

```typescript
createHandler("getUser")("123");            // Promise<{ id: string; name: string }>
createHandler("updateUser")("123", "Alice"); // Promise<boolean>
```

一方、内部で実際にハンドラーを呼び出す場所は、`as (...args: any[]) => any` で書き換えます。
`handlers[key]` がどの引数を受け取る関数なのかのチェックは、この `any` によって実装の内側から外へ逃がしています。
つまり、境界の外側（呼び出し側）では型安全を保ちつつ、境界の内側（実装）は `any` でごまかしているのが、この対応の実態です。

利点は、呼び出し側に型安全なインターフェースを残したまま、即座に動くものを用意できることです。
その代わり、`any` に逃がした分、実装の中で `key` と引数の対応を間違えてもコンパイラは検出できません。「`key` と引数が正しく対応する」ことは、ここでは人間の責任になります。

この「人間の責任になる」部分を型システムの側で解決しようとするのが、次の提案です。

---

## 課題を根本解決しようとする提案：Issue #27808

この「ジェネリクス制約がUnion全体を許容してしまう問題」に対して、TypeScriptリポジトリで議論されているのが **[Issue #27808: One-of generic constraints](https://github.com/microsoft/TypeScript/issues/27808)** です。

このIssueでは、型引数に対して「Unionの構成要素のどれか**1つ**にしか適用できない」という新しい制約構文（仮に `oneof`）が提案されています。

```typescript
// 提案されているイメージ（※実際の構文は確定していません）
function executeCommand<K extends oneof keyof Handlers>(...)
```

### `extends A | B | C` と `extends oneof(A, B, C)` の違い

集合論的に見ると、両者の違いは明白です。

* **通常のUnion制約 (`T extends A | B | C`)**
  $$T \subseteq A \cup B \cup C$$
  $T$ は $A, B, C$ を組み合わせたUnion型（例: $A \cup B$）になることが許されます。
* **One-of制約 (`T extends oneof(A, B, C)`)**
  $$T \subseteq A \quad \text{or} \quad T \subseteq B \quad \text{or} \quad T \subseteq C$$
  $T$ は $A, B, C$ のいずれか**単一の要素の構成範囲内**に制限されます。$A \cup B$ のような跨がりは不可能です。

`oneof` 制約が導入されれば、「`K` には単一のキーしか渡ってこない」という前提をコンパイラに持たせることができます。

---

## Generic Narrowing への影響

Issue #27808 の関心事は、単に関数呼び出しのチェックだけでなく **「ジェネリクス型引数の絞り込み（Generic Narrowing）」** にも及んでいます。

例えば、以下のようなコードです。

```typescript
function handle<T extends oneof(Foo, Bar)>(value: T) {
  if (isFoo(value)) {
    // T が "Foo または Bar" ではなく、明確に "T extends Foo" へ絞り込める
  }
}
```

従来の `T extends Foo | Bar` では、`value` を条件分岐で絞り込んでも型引数 `T` そのものは絞り込まれず、戻り値や他のジェネリクス変数との対応関係が壊れてしまう問題がありました。`oneof` があれば、「`T` は最初からどちらか一方に属している」ため、制御フロー解析によるジェネリクスの絞り込みが自然に行えるようになります。

---

## まとめ

* `handlers[key](...args)` がエラーになるのは、`K extends keyof T` が「`K` が単一のキーであること」を保証せず、**Union型全体を受け入れられるから**です。
* その結果、関数型と引数型の対応関係（相関）が失われ、安全に呼び出せないと判定されます。
* 暫定対応としては、`key` を引数に取ってハンドラーを返す高階関数に切り出す方法があります。戻り値の型計算を境界で確定させ、内部を `any` でごまかすことで、呼び出し側の型安全は保てます。
* Issue #27808 は `extends oneof(...)` によって **「Unionのどれか1つだけに制限するジェネリクス制約」** を導入しようとする提案です。
* これにより、関数の相関問題やジェネリクス型変数の絞り込み（Generic Narrowing）を型システムレベルで自然に扱える世界を目指しています。

実務で「なぜTypeScriptはこれを分かってくれないんだ」と頭を抱えたときは、型システムが「Union型が丸ごと渡ってくる可能性」を警戒しているのだと疑ってみると、エラーの理由が見えてくるかもしれません。

---