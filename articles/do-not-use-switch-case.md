---
title: "【TS】switch文を使わずに、オブジェクトリテラルを使って条件分岐を宣言的に書く"
emoji: "🔀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,javascript]
published: true
---

## はじめに

皆さん、`switch`文は使っていますか？
条件の振り分けを記述する際に真っ先に検討される方法だと思います。
ですが、TypeScriptに限らず、`switch`文は複雑な条件分岐を記述する際にはアンチパターンとなります。

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
    case "Enter": {
        handleEnter();
        break;
    }
    default:{
        console.log("指定外のキーが押されました");
        break;
    }
}
```

このような単純な例であれば、シンプルで読みやすいです。
ですが、今後の改修で、選択肢を追加するとしたらどうでしょうか？
例えば、W,A,S,D,キーを追加する場合を考えてみてください。
改修担当者は上から下までスクロールして、すべての選択肢を確認し、条件とその処理内容を把握する必要があります。
なぜなら、全ての選択肢で同じ処理が行われている保証がないためです。
`break`の記載漏れにも注意が必要です。
更に、押されたキーを記録する関数や、ログを出力する関数などの追加の処理を実装するとしたらどうでしょうか？
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
    case "KeyD": {
        move(rowIndex, columnIndex + 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyW": {
        move(rowIndex - 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "KeyS": {
        move(rowIndex + 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "Enter": {
        handleEnter();
        recordKey(key);
        logger.debug(key);
        break;
    }
    default: {
        console.log("指定外のキーが押されました");
        break;
    }
}
```

読む気が起きませんね。
この`switch`文は以下のように書き換えることができます。

## オブジェクトリテラルと型ガード関数、ブラケット記法を使った実装

```ts:object-map.ts
/* define mapping */
const keyMap = {
    ArrowLeft: () => move(rowIndex, columnIndex - 1),
    ArrowRight: () => move(rowIndex, columnIndex + 1),
    ArrowUp: () => move(rowIndex - 1, columnIndex),
    ArrowDown: () => move(rowIndex + 1, columnIndex),
    Enter: () => handleEnter(),
} as const;

const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);

/* execute */
if (isKey(key)) {
    keyMap[key]();
} else {
    console.log("指定外のキーが押されました");
}
```

```ts:object-map2.ts
/* define mapping */
const keyMap = {
    ArrowLeft: () => move(rowIndex, columnIndex - 1),
    ArrowRight: () => move(rowIndex, columnIndex + 1),
    ArrowUp: () => move(rowIndex - 1, columnIndex),
    ArrowDown: () => move(rowIndex + 1, columnIndex),
    KeyA: () => move(rowIndex, columnIndex - 1),
    KeyD: () => move(rowIndex, columnIndex + 1),
    KeyW: () => move(rowIndex - 1, columnIndex),
    KeyS: () => move(rowIndex + 1, columnIndex),
    Enter: () => handleEnter(),
} as const;

const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);

/* execute */
if (isKey(key)) {
    keyMap[key]();
    recordKey(key);
    logger.debug(key);
} else {
    console.log("指定外のキーが押されました");
}
```

いかがでしょうか？
オブジェクトリテラルと、型ガード関数、ブラケット記法でのアクセスを活用することでこのようにシンプルに書くことができます。
選択肢の分岐部分と、実行部分が分離されているため、それぞれを独立して記述することができています。
選択肢を追加する担当者は、処理の詳細が共通であることが保証されているため、追加しやすくなります。
処理内容を変更する場合は、わざわざ全ての選択肢に変更を加える必要がなくなるうえに、選択肢だけ追加して実装は漏れていた、ということも防げます。
keyが取りうる値の型が予め決まっている場合は、特別なLintルールを設定しなくても実装漏れに対して型エラーが発生するのも嬉しいですね。

その代わり、TypeScriptの機能を活用する必要から、少し複雑な書き方になっているため解説を挟んでおきます。

## 解説

- オブジェクトを定義

```ts
const keyMap = {
    ArrowLeft: () => move(rowIndex, columnIndex - 1),
    ArrowRight: () => move(rowIndex, columnIndex + 1),
    ArrowUp: () => move(rowIndex - 1, columnIndex),
    ArrowDown: () => move(rowIndex + 1, columnIndex),
    Enter: () => handleEnter(),
} as const;
```

はじめに、keyMapという定数オブジェクトを定義し、与えられたキーに対して実行する関数をマッピングしています。
オブジェクトのキーが、`switch`文におけるcaseの値に対応しています。
このように書くことで、選択肢を宣言的に記述することができ、１箇所で全ての選択肢を確認することができます。
今回の例の場合は、呼び出された際に実行される関数をオブジェクトのメソッドとして格納しています。
引数を取れなくなりますが、渡す値だけを格納して、呼び出し元で関数を実行する方法も良いでしょう。
より複雑な要件では、更にネストしたオブジェクトを定義し、2レベル以上の分岐を行うこともできます。
`switch`文でそのようなことを行うと、読解が困難になりますが、オブジェクトの場合は比較的読みやすいです。
もちろん、関数内でif文を使用することもできます。

- 型ガード関数を定義

```ts
const isKey = (key: string): key is keyof typeof keyMap => Object.hasOwn(keyMap, key);
```

次に、isKeyという型ガード関数を定義しています。
文字列がオブジェクトのキーとして存在するかどうかを判定しないと、ブラケット記法でのアクセスができません。
また、存在しなかった場合の処理の分岐にも必要です。
key変数の値の型が既に絞り込まれている場合、これは不要です。
`Object.hasOwn()`は、第一引数のオブジェクトが第二引数のキーを持っているかどうかを判定し、booleanを返す関数です。
似たような処理を行える`in`演算子のように継承元の`Object.prototype`に存在するプロパティを含めてしまうことがないため、こちらを使います。
もし、プロジェクトの設定でtsconfig.jsonのlibがES2022未満の場合は、`Object.hasOwn()`は使えないので、代わりに`Object.prototype.hasOwnProperty.call()`を使用してください。
こちらも同様の機能を持っています。

:::message
`Object.prototype.hasOwnProperty.call()`はprototypeを介して呼び出すため、`Object.prototype`を上書きして消してしまうと呼び出せなくなってしまいます。
そのため、消すことができないstaticメソッドとして`Object.hasOwn()`が生まれたようです。
:::

- オブジェクトのキーを取り出して処理実装

```ts
if (isKey(key)) {
    keyMap[key]();
} else {
    console.log("指定外のキーが押されました");
}
```

最後に、`isKey`関数を使って、keyMapオブジェクトにアクセスし、処理を実行しています。
型ガード関数で絞り込んだあとであれば、ブラケット記法でオブジェクトに格納された値にアクセスすることができます。
if文のtrueのブロックが`switch`文の`case`内の処理、falseのブロックが`default`の処理に対応しています。

## 蛇足

一応、`switch`文の分岐をシンプルにする方法として、同じ処理を共通化する方法もあります。
以下のように、`case`をまとめることで、共通の処理を一度だけ書くことができます。

```ts:switch-case3.ts
switch (key) {
    case "ArrowLeft":
    case "KeyA": {
        move(rowIndex, columnIndex - 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowRight":
    case "KeyD": {
        move(rowIndex, columnIndex + 1);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowUp":
    case "KeyW": {
        move(rowIndex - 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "ArrowDown":
    case "KeyS": {
        move(rowIndex + 1, columnIndex);
        recordKey(key);
        logger.debug(key);
        break;
    }
    case "Enter": {
        handleEnter();
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

この方法が活きるケースは、複数の選択肢で共通の処理がある場合に限られます。

## まとめ

||switch|object|
|----|----|----|
|記述|手続き的|宣言的|
|行数|多い|少ない|
|実装漏れ|Lintルール要|TypeSafe|
|学習コスト|低い|高い|

`switch`文は、選択肢が増えるにつれて読みづらくなり、保守性が低下します。
オブジェクトと型ガード関数を使うことで、選択肢を宣言的に記述し、保守性を向上させることができます。
TypeScriptにおけるオブジェクトリテラルを使った実装例を示しましたが、JavaScriptはもちろん、他の言語でもMapやDictionaryを使って同様の実装が可能です。
複雑な`switch`文のリファクタリングの際には、この方法を検討してみてください。

