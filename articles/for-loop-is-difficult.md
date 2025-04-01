---
title: "【チートシート】JavaScriptのforループ難しすぎ！？まとめてみた"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [javascript, typescript]
published: true
---

## はじめに

JavaScript, TypeScriptにはループを実現する方法がいくつもあり、ユースケースによって最適なものが異なります。
今では理解できるようになりましたが、初心者の頃はループの書き方が多すぎてどれを使っていいのかわからず、適切でない使用方法をしていたことがありました。
forループの種類を誰かに説明するにはあまりにパターンが多く、まとまっている記事も見当たらなかったため、この記事でまとめます。
ぜひチートシートとしてご活用ください！

:::message
簡単のため配列という言葉を使っていますが、SetなどのIterableなオブジェクトでも同様の処理ができます。

サンプルコードはTypeScriptで記述しています。
コードはTypeScript Playgroundで実行できるように記述しました。
リンクを貼り付けていますのでぜひ実行してみてください。
型注釈を省略するとJavaScriptとしても動作します。
:::

### 配列の要素を繰り返し処理する

```ts
const items = [1, 2, 3, 4, 5];
for (const item of items) {
  console.log(item);
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQAZiAE4wAUoksCKMIJvCSoAlDADeDGDG4QQAG0QA6BSADmHPslGMAvkA)

まずは基本型です。
`for ... in`という似たような構文もあり混乱してしまいますが、`for ... in`は継承元のプロパティも含めて列挙する仕様となっており、デバッグ用途以外で使うことはほとんどありません。
多くのスタイルガイドで非推奨とされていることが多いので、`for ... of`を使いましょう。

### indexを取りたい場合

```ts
const items = [1, 2, 3, 4, 5];
for (const [index, item] of items.entries()) {
  console.log(index, item);
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQAZiAE4wAUoksWcYAE0QAPAghQ0YIJvCSoAdIjBRWcRBA4BKTTADeDGDG4QQAG0TzTIAOYd+Q0bJSbGAXyA)

`items.entries()`で配列のインデックスと値のペアを取得できます。

### 別の書き方

```ts
const items = [1, 2, 3, 4, 5];
items.forEach((item, index) => {
  console.log(index, item);
});
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQCKEAdAGYgBOAogIbAAFgAphzZAThgAJogAeASgwA+GAG8GMGKEggANolZ6QAczEz5kpMgWMAvraA)

`forEach`を使うことでより簡潔に書くことができます。

### 式として値を返したい場合

```ts
const items = [1, 2, 3, 4, 5];
const result = items.map((item, index) => {
  return `${index}: ${item}`;
});
console.log(result);
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99&ssl=5&ssc=21&pln=1&pc=2#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQoksATohAK4A2smCKCADpkAQwAOACkkDkBOGAAmiAB4BKDAD4YAbwYwY7KJ1ZgYAAwAkOhcpUBfAFwxrs++cb21jZhBDdEIW4QAHNJdi5ebyA)

`forEach`は戻り値が`void`となっており、値を返せない仕様になっています。
値を返したい場合にはそのための関数として`map`が用意されています。
逆に、値を返さないのに`map`を使うのはコードの意図が伝わりづらくなってしまうため、やめましょう。
配列には他にも多くの便利なメソッドがあり面白いので、興味を持った方はぜひ調べてみてくださいね。

## 指定回数繰り返す

他の言語では`for n in range(10)`のような配列を生成する組み込み関数が用意されていることもありますが、JavaScriptにそれはないため、自力で用意する必要があります。
そのため書き方が複数あり、好みになってしまいます。

```ts
const count = 10;
for (let i = 0; i < count; i++) {
  console.log(i);
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBKCuZYF4YEYAMBuAUAMxACcYAKAGwFNYBLGVbGWgHjhESi0YGouBKGAN44YrSCEoA6MiADmJar1wBfIA)

一番良く見るパターンです。
`i`が`let`で宣言されているため、ループ内で値を変更できてしまうデメリットがあります。
実害はあまりないもしれませんが、コーディング規約によっては禁止されていることもあるので、使わずに済む方法も覚えておくとよいでしょう。

```ts
const count = 10;
for (const i of Array.from({ length: count }, (_, i) => i)) {
  console.log(i);
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBKCuZYF4YEYAMBuAUAMxACcYAKUSWASxhDxgEFDCBDATwDo9CQBbEgbxgAbAKZgA5lAAWALjghEsAL4AaUgH01lAJQxkAPhg7d-HDHmQQo9kJDiSO3EqA)

第一引数で生成する配列の数を指定し、第2引数では`Array.prototype.map`と同様の配列をどのように生成するかを決める関数を渡します。
指定回数繰り返すだけなのにちょっと複雑ですね…。

```ts
const count = 10;
for (const i of Array(count).fill(0).map((_, i) => i)) {
  console.log(i);
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99&ssl=4&ssc=2&pln=1&pc=1#code/MYewdgzgLgBKCuZYF4YEYAMBuAUAMxACcYAKUSWASxhDxgEFDCBDATzJESgEoA6PSgBtBJDHwC2zAA4kSAfQA0MStxjIAfMu6qA3jhhxwEEIICmvQSADmJFbgC+QA)

こちらは別解です。
配列を0で埋め、`map`でインデックスを生成しています。
インデックスが要らなければ`map`を省略すれば`fill`だけで生成することもできます。

```ts
import * as R from "remeda";

const count = 10;
R.times(count, i => {
  console.log(i);
});
```

Remedaのような外部ライブラリに用意されている関数を用意すればもっと簡単に書く方法もあります。
この関数は値も返せます。

## 条件を満たす間繰り返す

### `while`を使った方法

```ts
class SmartPhone {
  private battery: number = 0;

  charge(): void {
    this.battery += 10;
    console.log('charge:', this.battery);
  }

  isLowBattery(): boolean {
    console.log('isLowBattery:', this.battery < 100);
    return this.battery < 100;
  }
}

const smartPhone = new SmartPhone();

while(smartPhone.isLowBattery()) { // trueの間繰り返す
  smartPhone.charge();
}
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYGwhgzhAEDKC2YBOAXACgCwPYDsCm0A3gFDTQAOSAlgG5goEBG9DSAngFzQ4Cu8jeJNAC80AAwBuYqWjAMyAOZ4AFAEouNLFQAmRGWRQYqEAHTMUrNtADUogIyT9s3BCwg8JkFgXKA5HMU8Dl8AGmhDYzMWQTZVKTIAX2kyYwAZLAB3ACFo9jUuRiw3PDAcPTIyYBdiz28-NMycixjgsIjTc0toAB5oBzE4pyQ8FB4kMvao5vYevrFHROIk4iqcCBRoCERUTFwCUXwMuG30bHw1KWIMo3dlLeRTvZMG7Ny2NVUiaAB6b-CkHh4QB2DIBk1MADn6AKIZACvxgE0GGT3HZnDwBJBKC5LIA)

`while`を使うことで条件を満たす間繰り返す処理を実現できます。

### `do while`を使った方法

```ts
const smartPhone = new SmartPhone();

do {
  smartPhone.charge();
} while(smartPhone.isLowBattery());
```

`do while`を使うことで最初の一回だけ条件を無視して処理を実行することもできます。

### 前回の結果を次のループに使いたい

```ts
const hanoi = (height: number, first: number, temp: number, last: number): void => {
  if (height === 0) return; // 早期リターンで止め時を指定しておく
  hanoi(height - 1, first, last, temp);
  console.log(`${first} -> ${last}`);
  hanoi(height - 1, temp, first, last);
};

hanoi(3, 1, 2, 3);
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAFgQzCAljAvDAFHApigczigC4YwBXAWwCNcAnAGhgDMV7ozLaHmpcqABy7U6TGABsEncqIYBKMgDdUAEwwA+GAG8AUDBgoW2PIWIZ0mAAzyY9XFAr0wAbhgB6dzECXpoHxzQFcMgP0MgD8MgM8MgOYMgEbWgIEMgEJmgEkMgODGgFnagOoMgGYMgFIMgPIM+vBIqDj4RLAAtDAAjMxsHFDMUtB8AoLyLnmgkCASuAB0EiAEWAAGACTaNdAAvjClWmMNUJNDrXmIyChFZmWVTULV7I2S0lArk226a4UAzMxVMABMzFetQA)

自分自身を呼び出すことで次のループに前回の結果を引き継ぐことができます。
この方法は末尾再帰の実装があるSafariやBun以外のランタイムではStack Overflowを起こす可能性があるため、注意が必要です。

## 非同期処理

非同期処理を含む場合、ループはさらに難しくなります。

:::message
例文はトップレベルに記載しているため、どこでも実行できるように`void run();`で戻り値のPromiseを無視して関数を呼び出しています。
フレームワークやライブラリを使う場合は、内部で呼んでくれることが多いため、このような書き方は必ずしも必要ではありません。
:::

### 直列処理

```ts
const items = [1, 2, 3, 4, 5];

const process = async (item: number) => new Promise((resolve) => {
    console.log(item);
    setTimeout(resolve, 1000);
});

const run = async () => {
  for (const item of items) {
    await process(item);
  }
};

void run();
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQDoksADgE4jCIRqYBDCAE8wwGAAoEKAFwwwAV2QAjRBwCUGAHzzEAdxgAFLsjgREEiR14gANgDdEm9DoDeDGJ5gsIdxADpbEABzKSRkdUYvGHMoABU4ZEQQBSgrGwdEAhwABjzIhgBfAuZwaBgOBTAMGCFRcQlnNw8YADMQDkkfWGlkGBBW+HCITXdogT0BBBhObl4IMJQCz0KixgZ7EDgAEwqqxrogA)

直列処理は、配列の要素を順番に一つずつ待ち、終わり次第次の要素を処理するものです。
このような処理がしたい場合はforEachでは実現できません。
`for ... of`のブロック内では`await`を使うことができるため、非同期関数を呼び出すことができます。

### 直列処理かつ結果を取得したい場合

```ts
const items = [1, 2, 3, 4, 5];

const process = async (item: number): Promise<number> => 
  new Promise((resolve) => {
      console.log(item);
      setTimeout(() => resolve(item), 1000);
  });

const run = async () => {
  const result = await Array.fromAsync(items, process);
  console.log(result);
}

void run();
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQDoksADgE4jCIRqYBDCAE8wwGAAoEKAFwwwAV2QAjRBwCUcgApdkcCIgA8ilWoB8GCwxjzEAdxg6QegxIkdeIADYA3ROssYAG9rGzCWCG9EADovEABzKSRkdUYwsIMoABU4ZEQQBSg3APQLD0jfRCSUdQIcAAZG1NCAX2bmcGgYDgUwDBghUXEJEosQmwjYcoUvWEE7AQQYAEEODgFhaIAzXWWRMWrUAk5uXghmic6o2IT3XhmoZpamHxA4ABNu3pG6IA)

ES2024で追加された新しい書き方です。
`Promise.all`と`Array.from`の間の子のような関数で、こちらも配列の要素を順番に待機して処理することができます。
以前はこの書き方ができず、結果を可変の配列に追加する必要がありました。
新しい書き方のため、環境によってはpolyfillを用意しないといけない点に注意してください。

https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Array/fromAsync

### 並列処理

```ts
const items = [1, 2, 3, 4, 5];

const process = async (item: number): Promise<number> => 
  new Promise((resolve) => {
      console.log(item);
      setTimeout(() => resolve(item), 1000);
  });

const run = async () => {
  const result = await Promise.all(items.map(item => process(item)));
  console.log(result);
}

void run();
```

[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQDoksADgE4jCIRqYBDCAE8wwGAAoEKAFwwwAV2QAjRBwCUcgApdkcCIgA8ilWoB8GCwxjzEAdxg6QegxIkdeIADYA3ROssYAG9rGzCWCG9EADovEABzKSRkdUYwsIMoABU4ZEQQBSg3APQLD0jfRCSUdQIcAAZG1NCAX2bmcGgYDgUwDBghUXEJEosQmwjYcoUvWEE7AQRHXX0YgS8vatRo5AE2LcDObl4ILfVztJgIqNiE914ZqGaWph8QOAATbt6RuiA)

並列処理は、実行順序を気にせずに全ての要素を一度に処理する場合に使います。
`Promise.all`は`Promise`の配列を受取り、全てのPromiseが完了するまで待機する関数となっています。
引数は`ArrayLike<Promise<T>>`を取るため、あらかじめ実行するPromiseの配列を作成して渡してあげる必要があります。
`await`しないで呼び出した非同期関数は`Promise`を返すため、`map`のコールバックに渡してあげることでPromiseの配列を作成できます。

### 並列処理かつ、失敗しても処理を続行したい場合

```ts
const items = [1, 2, 3, 4, 5];

const process = async (item: number) => {
  await new Promise(resolve => setTimeout(resolve, 1000));
  if (item === 3) {
    throw new Error("error");
  }
  return item;
};

const run = async () => {
  const result = await Promise.allSettled(items.map(item => process(item)));
  result.forEach((promise, index) => {
    if (promise.status === "fulfilled") {
      console.log(index, promise.value);
    } else {
      console.error(index, promise.reason);
    }
  });
}

void run();
```
[TypeScript Playground](https://www.typescriptlang.org/play/?target=99#code/MYewdgzgLgBAllApgWwjAvDA2gRgDQwBMBAzAQCwECsAugNwBQDoksADgE4jCIRqYBDCAE8wwGAAoEKAFwwwAV2QAjRBwCUGAHwwA3gxgwBAdwEJ5iYzAAKXZHAiIJHXiAA2AN0TaYjqABU4ZEQQBShnV09EAhwABnj1dUZDOAAzSWlkDHRMEk19Q0MoAAsuKzBLGABRDi4OCQAiNTqGpIMYAF92lygFDjB4JGRGDsZmcGgYDgUBwRExSU10HQKYFkmXCAU3WEFTc1sQe0cAOgE3NwBlRCgoN0QAEykhiBPkATZnlB9Obl4IL7IRJtQybbZQE6pEAcKoCYDFCQSX7HaLwMAPRAADyWK3aKXSSLsDkQJ2gAl6-ByMAaqW2qTgF0erT0eMK63cJLcIAA5lJ0ViCMjiScPOcFIgQYUOjBEG5HCzCmyJhyTs1oXyMZjBUTTi4hOBJYYuka2l0GB4QHAHlMZhIkkA)
`Promise.allSettled`は`Promise.all`と似ていますが、失敗してもErrorをthrowすることなく処理を続行します。
戻り値は`{ status: "fulfilled", value: T } | { status: "rejected", reason: any }`の配列で、Result型になっています。
戻り値の`status`プロパティで成功か失敗かを判定でき、それぞれ`value`で値、`reason`でエラーを取得できます。

## まとめ

JavaScript, TypeScriptにはループを実現する方法がいくつもあり、ユースケースによって最適なものが異なります。
意外にも最近追加されたばかりの関数もあり、知らなかったという方もいるのではないでしょうか。
ぜひこの記事を参考にして、最適なループの書き方を見つけてください！