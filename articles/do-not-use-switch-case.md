---
title: "【TypeScript】まだswitch case文使ってるの？"
emoji: "🔀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,javascript]
published: true
---

## はじめに

突然ですが皆さん、`switch case`文を使っていますか？

```ts:switch-case.ts
switch (key) {
    case "ArrowLeft": {
        move(rowIndex, columnIndex - 1);
        break;
    }
    case "ArrowRight": {
        move(rowIndex, columnIndex + 1);
        break;
    }
    case "ArrowUp": {
        move(rowIndex - 1, columnIndex);
        break;
    }
    case "ArrowDown": {
        move(rowIndex + 1, columnIndex);
        break;
    }
    default:{
        console.log("指定外のキーが押されました");
        break;
    }
}
```

条件の振り分けを記述する際に真っ先に検討される方法だと思います。
このような単純な例であれば、シンプルで読みやすいですが、今後の改修で、選択肢を追加するとしたらどうでしょうか？
例えば、A,S,D,Fキーを追加する場合を考えてみてください。
改修担当者は上から下までスクロールして、すべての選択肢を確認し、条件とその処理内容を把握する必要があります。
なぜなら、全ての選択肢で同じ処理が行われている保証がないためです。
`break`にも注意が必要です。
更に、押されたキーを記録する関数や、ログを出力する関数を追加するとしたらどうでしょうか？
各選択肢に全く同じ処理を何度も追加するハメになります。
途端に見通しが悪くなります。

```ts:switch-case2.ts
switch (key) {
    case "ArrowLeft": {
        move(rowIndex, columnIndex - 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowRight": {
        move(rowIndex, columnIndex + 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowUp": {
        move(rowIndex - 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowDown": {
        move(rowIndex + 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyA": {
        move(rowIndex, columnIndex - 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyS": {
        move(rowIndex, columnIndex + 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyD": {
        move(rowIndex - 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyF": {
        move(rowIndex + 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    default:{
        console.log("指定外のキーが押されました");
        break;
    }
}
```

読む気が起きませんね。
この`switch case`文は以下のように書き換えることができます。

## オブジェクトリテラルと型ガード関数、ブラケット記法を使った実装

```ts:object-map.ts
const keyMap = {
    ArrowLeft: [0, -1],
    ArrowRight: [0, 1],
    ArrowUp: [-1, 0],
    ArrowDown: [1, 0],
} as const;

const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);

if (isKey(key)) {
    move(rowIndex + keyMap[key][0], columnIndex + keyMap[key][1]);
} else {
    console.log("指定外のキーが押されました");
}
```

```ts:object-map2.ts
const keyMap = {
    ArrowLeft: [0, -1],
    ArrowRight: [0, 1],
    ArrowUp: [-1, 0],
    ArrowDown: [1, 0],
    KeyA: [0, -1],
    KeyS: [0, 1],
    KeyD: [-1, 0],
    KeyF: [1, 0],
} as const;

const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);

if (isKey(key)) {
    move(rowIndex + keyMap[key][0], columnIndex + keyMap[key][1]);
    recordKey(key);
    logger.debug(key);
} else {
    console.log("指定外のキーが押されました");
}
```

いかがでしょうか？
オブジェクトリテラルと、型ガード関数、ブラケット記法でのアクセスを活用することでこのようにシンプルに書くことができます。
`switch case`では選択肢を１つ増やす毎に４行以上のコードが追加されていました。
この書き方であれば、選択肢を増やしても１行追加するだけで済みます。
選択肢を追加する担当者も、処理の詳細が共通であることが保証されているため、追加しやすくなります。
更に、処理内容を変更する場合にわざわざ全ての選択肢に変更を加える必要がなくなります。

その代わり、TypeScriptの機能を活用する必要から、少し複雑な書き方になっているため解説を挟んでおきます。

## 解説

- オブジェクトを定義

```ts
const keyMap = {
    ArrowLeft: [0, -1],
    ArrowRight: [0, 1],
    ArrowUp: [-1, 0],
    ArrowDown: [1, 0],
} as const;
```

はじめに、keyMapという定数オブジェクトを定義しています。
オブジェクトのキーが、switch文におけるcaseの値に対応しています。
このように書くことで、選択肢を宣言的に記述することができ、１箇所で全ての選択肢を確認することができます。
今回の例の場合は、同一の関数を呼び出しているため、値には引数にアレンジを加えるためのタプルを格納しています。
ここは場合によっては、関数そのものを格納することもできます。
より複雑な要件では、更にネストしたオブジェクトを定義し、2レベル以上の分岐を行うこともできます。
switch caseでそのようなことを行うと、読解が困難になりますが、オブジェクトの場合は比較的読みやすいです。

- 型ガード関数を定義

```ts
const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);
```

次に、isKeyという型ガード関数を定義しています。
文字列がオブジェクトのキーとして存在するかどうかを判定しないと、ブラケット記法でのアクセスができません。
また、存在しなかった場合の処理の分岐にも必要です。
key変数の値の型が既に絞り込まれている場合、これは不要です。
`Object.hasOwn`は、第一引数のオブジェクトが第二引数のキーを持っているかどうかを判定し、booleanを返す関数です。
この関数は実は新しい関数で、プロジェクトによっては使えない可能性があります。
もし、プロジェクトの設定でtsconfig.jsonのlibがES2022未満の場合は、`Object.hasOwn`は使えないので、代わりに`Object.prototype.hasOwnProperty`を使用してください。
こちらも同様の機能を持っています。

- オブジェクトのキーを取り出して処理実装

```ts
if (isKey(key)) {
    move(rowIndex + keyMap[key][0], columnIndex + keyMap[key][1]);
} else {
    console.log("指定外のキーが押されました");
}
```

最後に、isKey関数を使って、keyMapオブジェクトにアクセスして関数に渡す値を取り出し、処理を実行しています。
型ガード関数で絞り込んだあとであれば、ブラケット記法でオブジェクトに格納された値にアクセスすることができます。
if文のtrueのブロックがswitch case文のcase内の処理、falseのブロックがdefaultの処理に対応しています。

## まとめ

switch case文は、選択肢が増えるにつれて読みづらくなり、保守性が低下します。
オブジェクトと型ガード関数を使うことで、選択肢を宣言的に記述し、保守性を向上させることができます。
複雑なswitch case文のリファクタリングの際には、この方法を検討してみてください。
