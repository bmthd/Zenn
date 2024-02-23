---
title: "【TypeScript】配列でネストしたオブジェクトの型を再帰的にフラットにする型パズル"
emoji: "🧩"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['typescript']
published: true
---

## はじめに

業務で遭遇した、TypeScriptでのオブジェクトの型定義から別の型を得るやり方の備忘録です。
求められていたのは、表示用のデータをAPIから取得し、それをテーブル形式で画面に表示するというものでした。
テーブルを実装する際に画面との紐づけに、オブジェクトから取得したキーをそのまま使うことができれば、型安全に共通化できると思ったため、再帰的にフラットにするやり方を調べていました。
結局、時間内に解決できなかったため実装については別のやり方で進めることになりましたが、その後後学のために自分で実装してみたものを残しておきます。

:::message
以下の前提のもと実装しています。

- すべてのオブジェクトのプロパティは一意のものであること
- 1つのオブジェクト内には1つの配列しかないこと

:::

## 前提となるデータ構造

以下のようなデータ構造を想定します。
APIからの返却値などでよくある、オブジェクトの中に配列がネストしている形です。

```ts:データ構造
type Item = {
  itemId: string;
  itemName: string;
  price: number;
  releasedAt: Date;
  isAvailable: boolean;
};

type Shop = {
  shopId: string;
  shopName: string;
  items: Array<Item>;
};

type Company = {
  companyId: string;
  companyName: string;
  location: string;
  shops: Array<Shop>;
};

type Response = {
  data: Array<Company>;
};

// データの例
const response: Response = {
  data: [
    {
      companyId: "1",
      companyName: "company1",
      location: "tokyo",
      shops: [
        {
          shopId: "1",
          shopName: "shop1",
          items: [
            {
              itemId: "1",
              itemName: "item1",
              price: 100,
              releasedAt: new Date(),
              isAvailable: true,
            },
            {
              itemId: "2",
              itemName: "item2",
              price: 200,
              releasedAt: new Date(),
              isAvailable: false,
            },
          ],
        },
      ],
    },
  ],
};
```

```ts:求める型
type FlattenedResponse = {
  shopId: string;
  shopName: string;
  itemId: string;
  itemName: string;
  price: number;
  releasedAt: Date;
  isAvailable: boolean;
  companyId: string;
  companyName: string;
  location: string;
};

type KeyofFlattenedResponse = keyof FlattenedResponse;
// "shopId" | "shopName" | "itemId" | "itemName" | "price" | "releasedAt" | "isAvailable" | "companyId" | "companyName" | "location"

```

## 実装

まず、APIが返却しうるデータの型を定義します。

```ts:型定義
type APIValue = string | number | boolean | Date;

type APIObject = Record<string, APIValue | Array<APIObject>>;
```

ここでのポイントは、APIが返却しうるデータの型を`APIValue`として定義し、それを使って`APIObject`を再帰的に定義しているところです。
この構造を定義しておかないと、再帰的に処理することができません。

次に、処理するに当たって必要になるユーティリティ型を定義します。

```ts:ユーティリティ型
type Merge<T extends Record<string, unknown>, U extends Record<string, unknown>> = {
  [K in keyof T | keyof U]: K extends keyof U ? U[K] : K extends keyof T ? T[K] : never;
};

type KeyofNotNested<T extends Record<string, unknown>> = {
  [K in keyof T]: T[K] extends Array<infer U> ? (U extends APIValue ? never : never) : K;
}[keyof T];

type KeyofNested<T extends Record<string, unknown>> = {
  [K in keyof T]: T[K] extends Array<infer U> ? (U extends APIValue ? never : K) : never;
}[keyof T];

type PickNotNested<T extends Record<string, unknown>> = Pick<T, KeyofNotNested<T>>;

type PickNested<T extends Record<string, unknown>> = Pick<T, KeyofNested<T>>;
```

`Merge`は、型を通した後の交差型の可読性を良くするための型です。
`{ foo: string }`と`{ bar: number }`をそのまま`&`で交差させても、`{ foo: string } & { bar: number }`となってしまい、`{ foo: string; bar: number }`とはならないため、それを解決します。

`KeyofNotNested`と、`KeyofNested`は、それぞれ配列でネストしていないキーとネストしているキーを取得するための型です。
TypeScript の型の中では、`Object.entries()`のようなオブジェクトに対する柔軟な処理が行えず、Mapped types (`[K in keyof T]: T[K]`)を使用しても一括で処理することしかできないため、別々に取得して処理を行います。

使うとこんな感じ。

```ts:使用例
type KeyofCompany = KeyofNotNested<Company>; // "companyId" | "companyName" | "location"

type KeyofNestedCompany = KeyofNested<Company>; // "shops"

type PickCompany = PickNotNested<Company>; // { companyId: string; companyName: string; location: string; }

type PickNestedCompany = PickNested<Company>; // { shops: Array<Shop>; }
```

最後に、再帰的にフラットにする関数を定義します。

```ts:再帰的にフラットにする関数
type FlattenObject<T extends APIObject, A extends APIObject = {}> =
  T extends Record<string, APIValue>
    ? Merge<A, T>
    : PickNested<T> extends Record<string, Array<infer U>>
      ? U extends APIObject
        ? FlattenObject<U, Merge<PickNotNested<T>, A>>
        : never
      : never;
```

型の中で変数を宣言することはできないため、型引数の第二引数に空のオブジェクトを用意しておき、ネストしていくたびにそこに結果をマージしていくやり方を取りました。
この実装は、配列操作に使用する`reduce()`関数のaccumulatorに似ていたため、型引数の名前は`A`としています。
ネストするたびに6行目の処理が再帰的に行われ、最終的に`T`が`Record<string, APIValue>`になる、つまりネストしなくなるまで処理が続きます。
ネストが終わると、第二引数に今までの結果が渡ってきているため、それと最後のオブジェクトを3行目でマージして最終結果を得ます。
仕様上、`infer`や`APIObject`での絞り込みが必要なため、Conditional Types(三項演算子)を使用していますが、`never`になることはありません。

使うとこんな感じ。

```ts:使用例
type FlattenedResponse = FlattenObject<Response>; 
// {
//     shopId: string;
//     shopName: string;
//     itemId: string;
//     itemName: string;
//     price: number;
//     releasedAt: Date;
//     isAvailable: boolean;
//     companyId: string;
//     companyName: string;
//     location: string;
// }

type KeyofFlattenedResponse = keyof FlattenedResponse;
// KeyofNotNested<Shop> | KeyofNotNested<Company> | keyof Item
```

ここで`FlattenedResponse`のキー一覧のユニオン型でなく、`KeyofNotNested<Shop> | KeyofNotNested<Company> | keyof Item`が推論されるのは驚きました。
ただ、使用側が`KeyofNotNested`の実装を知っている必要があり、見に行く必要が出てきてしまうため、ユニオン型で取得する方法があればより良いと思いました。
どなたかわかれば教えてください…。

```ts:全コード
type APIValue = string | number | boolean | Date;

type APIObject = Record<string, APIValue | Array<APIObject>>;

type Merge<T extends Record<string, unknown>, U extends Record<string, unknown>> = {
  [K in keyof T | keyof U]: K extends keyof U ? U[K] : K extends keyof T ? T[K] : never;
};

type KeyofNotNested<T extends Record<string, unknown>> = {
  [K in keyof T]: T[K] extends Array<infer U> ? (U extends APIValue ? never : never) : K;
}[keyof T];

type KeyofNested<T extends Record<string, unknown>> = {
  [K in keyof T]: T[K] extends Array<infer U> ? (U extends APIValue ? never : K) : never;
}[keyof T];

type PickNotNested<T extends Record<string, unknown>> = Pick<T, KeyofNotNested<T>>;

type PickNested<T extends Record<string, unknown>> = Pick<T, KeyofNested<T>>;

type FlattenObject<T extends APIObject, A extends APIObject = {}> =
  T extends Record<string, APIValue>
    ? Merge<A, T>
    : PickNested<T> extends Record<string, Array<infer U>>
      ? U extends APIObject
        ? FlattenObject<U, Merge<PickNotNested<T>, A>>
        : never
      : never;

type Item = {
    itemId: string;
    itemName: string;
    price: number;
    releasedAt: Date;
    isAvailable: boolean;
    };

type Shop = {
    shopId: string;
    shopName: string;
    items: Array<Item>;
    };

type Company = {
    companyId: string;
    companyName: string;
    location: string;
    shops: Array<Shop>;
    };

export type Response = {
    data: Array<Company>;
    };

type FlattenedResponse = FlattenObject<Response>; 

type KeyofFlattenedResponse = keyof FlattenedResponse;

```

## まとめ

業務では限られた時間内に実装する必要があったため、解けずに苦しい思いをさせられましたが、自分の時間を使ってゆっくりと解いてみれば、楽しい型パズルになりました。
一意でないプロパティにも対応したい場合は、ネストするたびに上位オブジェクトの名称を付け、`company.shop.id`のような形でキーを作成することで対応できそうです。
配列が2つ以上存在する場合のケースについても工夫すれば対応できそうですが、自力では実装できませんでした。
また、再帰を使っている関係なのか、この型定義を開いたり参照していると、頻繁にコンパイラが落ちます。
より良い実装や、上記に対応できる実装方法など、どなたか解けた方がいれば教えていただきたいです。
業務時間中、ネットの海を死にものぐるいで検索しても出てこず諦めざるを得なくなったので、どなたかの参考になれば幸いです。
