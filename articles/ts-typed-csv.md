---
title: "CSVの型カオスに立ち向かう、TypeScriptでの完全型安全な扱い方"
emoji: "📊"
type: "tech"
topics: ["typescript", "csv", "nodejs", "valibot"]
published: true
---

## はじめに

CSVファイルをTypeScriptで型安全に扱う方法について、具体的な実装を交えながらご紹介します。
この記事で紹介するコードのデモと実際のソースコードはこちらから確認できます。

https://exp.bmth.dev/csv

![デモページが表示されている](/images/ts-typed-csv/image.png)

## CSVの悩み：なぜJSONのようにいかないのか

JSONファイルは、TypeScriptプロジェクトにおいて非常に扱いやすい存在です。
`import data from './data.json'`のように書くだけで、TypeScriptが型を推論し、中身を直接オブジェクトとして利用できます。
とても便利ですよね。

しかし、CSVファイルではそうはいきません。
CSVは構造化されたテキストデータですが、JSONのように直接インポートしてオブジェクトとして扱う標準的な仕組みが存在しないのです。

そのため、サーバーサイドでCSVを扱う場合、多くのケースで`fs`モジュールを使ってファイルを文字列として読み込み、自前でパース処理を実装することになります。

```typescript:よくあるCSVの読み込み
import fs from "node:fs";
import path from "node:path";
import { parse } from "papaparse"; // CSVパーサーの例

const csvPath = path.join(process.cwd(), "data", "sales_data.csv");
const csvFile = fs.readFileSync(csvPath, "utf-8");

const { data } = parse(csvFile, { header: true });

// dataは any[] もしくは Record<string, any>[] となり、型安全ではない
data.forEach((row) => {
  console.log(row.productName); // ❌️プロパティが存在するか不明。型補完も効かない...
});
```

この方法では、`data`の中身は`any`の配列になってしまい、プロパティ名や値の型が全く保証されません。
これでは、せっかくTypeScriptを使っているメリットが半減してしまいます。

## Valibotで実現する型安全なCSVパース

この課題を解決するため、CSVパーサーとスキーマバリデーションライブラリを組み合わせます。
今回は私が愛用している[**Valibot**](https://valibot.dev/)を使ってみましょう。

[**Conformの記事**](https://zenn.dev/bmth/articles/conform-to-complex)でも紹介しましたが、Valibotは非常に軽量でパフォーマンスに優れたバリデーションライブラリです。

以下のようなデータをAIにお願いして用意してもらいました。

```csv:sales_data.csv
itemId,productName,category,unitPrice,salesCount,saleDate
P001,高性能ノートPC,PC・周辺機器,150000,5,2025-05-10
P002,ワイヤレスイヤホン,オーディオ,12000,30,2025-05-11
P003,"多機能バックパック, 30L",バッグ・アクセサリ,8500,15,2025-05-12
P004,有機栽培コーヒー豆 500g,食品・飲料,2200,50,2025-05-12
P005,ゲーミングマウス,PC・周辺機器,7800,25,2025-05-13
P006,小説「夏空の約束」,書籍,1650,120,2025-05-14
P007,スマートウォッチ,ウェアラブル,25000,18,2025-05-15
P008,"アロマディフューザー, ウッド調",生活家電,4500,40,2025-05-16
P009,天然水 2L (6本入),食品・飲料,800,200,2025-05-17
P010,グラフィックTシャツ,ファッション,3200,60,2025-05-18
```

まず、CSVの行データがどのような構造を持つべきか、スキーマを定義します。
このスキーマがCSVの1行分の型を定義します。

```ts:schema.ts
import * as v from "valibot";

const salesDataSchema = v.object({
  itemId: v.string(),
  productName: v.string(),
  category: v.string(),
  unitPrice: v.number(),
  salesCount: v.number(),
  saleDate: v.string(),
});

export type SalesData = v.InferOutput<typeof salesDataSchema>;
```

次に、このスキーマを使ってパース処理を実装します。

```tsx:CSVを型安全にパースする処理
import fs from "node:fs";
import path from "node:path";
import papa from "papaparse";
import * as v from "valibot";
import { salesDataSchema, type SalesData } from "./schema";

export const getSalesData = (): SalesData[] => {
  const csvPath = path.join(process.cwd(), "data", "sales_data.csv");
  const csvFile = fs.readFileSync(csvPath, "utf-8");

  const { data } = papa.parse(csvFile, {
    header: true,
    dynamicTyping: true, // 数値や日付を自動で型変換してくれる
  });

  // Valibotでデータの検証と型変換を行う
  const validationResult = v.safeParse(v.array(salesDataSchema), data);

  if (!validationResult.success) {
    // 実際のアプリケーションでは、より丁寧なエラーハンドリングを
    console.error(validationResult.issues);
    throw new Error("CSV data validation failed");
  }

  // ここで得られるデータは完全に型安全！
  return validationResult.output;
};
```

`papaparse`の`dynamicTyping`オプションで基本的な型変換を行い、その結果をValibotのスキーマで検証・整形します。
これにより、`getSalesData`の返り値は`SalesData[]`型であることが完全に保証され、エディタの型補完も最大限に活用できるようになります。

## 実装のポイントと補足

この手法を扱う上で、いくつか知っておくと良いポイントがあります。

### サーバー実行の前提とビルド時の挙動

このコードはNode.jsの`fs`モジュールに依存しているため、**Node.jsが動作するサーバーサイドでの実行が前提**となります。

ここで一つ面白い点があります。
Next.jsなどのReact Server Components（RSC）環境でこのコードを利用する場合、`fs.readFileSync`はリクエストごとではなく**ビルド時に一度だけ実行**されます。
そのため、一見動的にファイルを読み込んでいるように見えますが、`revalidatePath`や`revalidateTag`などでキャッシュの無効化を明示しない限り、その内容はビルド成果物にバンドルされているのと実質的に同じ挙動となります。

### Vite環境でのスマートな読み込み

もしフロントエンドの開発でViteを使っている場合は、もっと簡単な方法があります。
`?raw`サフィックスを使うことで、バンドルしたCSVファイルを文字列として直接インポートできます。

```ts:viteでのCSV読み込み
import csvString from "./sales_data.csv?raw";
```

この機能を使うには、アンビエント宣言でTypeScriptに型を教えてあげる必要があります。

```ts:vite-env.d.ts
declare module "*.csv?raw" {
  const content: string;
  export default content;
}
```

これにより、`fs`モジュールが不要になり、クライアントサイドでも同様のパース処理をシンプルに記述できます。

### この手法の「嬉しみ」

この手法の最大のメリットは、**あらゆるCSVデータを型安全に扱える**点です。

- **ローカルファイルも動的データもOK**
 今回はローカルのCSVファイルを扱いましたが、APIから取得したCSV文字列をパースする場合も、全く同じバリデーションロジックを適用できます。これにより、外部データソースに起因する予期せぬエラーを未然に防げます。
- **最高の開発体験**
  `data.productName`のようなコードを書く際に、エディタがプロパティを補完してくれます。タイプミスによるバグはもう過去のものです。

## まとめ

CSVファイルはそのままではTypeScriptにとって扱いにくい存在ですが、**CSVパーサーとValibotのようなスキーマバリデーションライブラリを組み合わせる**ことで、JSONのように型安全に、そして快適に扱うことが可能になります。

サーバーサイドのデータ処理はもちろん、工夫次第で様々な環境に応用できるテクニックです。
ぜひ、あなたのプロジェクトでも型安全なCSVライフをお楽しみください！
