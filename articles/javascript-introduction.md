---
title: "忙しい人の為のJavaScript入門【他言語差異】"
emoji: "👧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [javascript,typescript]
published: false
---
## この記事の想定読者

既に他言語の経験がある方や、初心者向けの解説記事が簡単すぎて面白くない方向けにJavaScriptの解説をしていきます。
変数、関数、if文、繰り返し処理などの非常に簡単な要素を除いています。
初めての方でも理解が早い方であれば問題なく読めるかと思います。

## 前提

執筆当時最新のECMAScript2021に則った文法を解説します。
本当に基礎的な内容は敢えて記載しないようにしていますので、分からなければGoogleで検索していただくか、ChatGPTに質問してください。
TypeScriptにも通ずる内容となっています。

## 変数

変数宣言は3種類あります。

```javascript
var number1 = 1;

let number2 = 2;

const number3 = 3;
```

var > let > const の順に登場したため、後者のほうが言語仕様が新しい設計です。

順に解説していきます。

- var
基本的にこれは使わないので覚える必要はありません。
スコープが緩く、どこからでも書き換えできます。
長いコードの中でこの宣言を使ってしまうと自分が気づかないところで再代入されバグを生み出す危険があります。

- let
スコープがメソッド内に限定されるため、varと比べると安全です。
よくfor文のカウントアップで使用されます。

```javascript
for(let i = 0; i > 10; i++){
    someProcess(i);
}
```

- const
イミュータブルな変数です。
1度宣言した変数に再代入ができないため、安全です。
関数型プログラミングではconstのみを使用し、変更可能な変数は避けます。
Reactなどのコンポーネント志向のライブラリは関数型プログラミング前提の設計になっているため、それらを使う場合はこれだけ覚えておけば十分かもしれません。

## 関数

関数宣言のやり方には2つのパターンがあります。

```javascript:function
function calc(a,b){
return a + b;
};
```

```javascript:アロー関数
const calc = (a,b) => {
    return a + b;
};
```

JavaScriptでは関数型プログラミングのエッセンスを取り入れ、変数に関数を代入可能となっています。
変数に代入する際には関数に名前が必要ないため、無名関数のまま定義が可能です。
そうなると、functionという文字も冗長なため、矢印のような=>で関数であることを表現するようになりました。
他の言語でラムダ式、ラムダ関数を知っていれば理解しやすいと思います。

thisキーワードを使う場合にスコープの違いがあるようですが、基本的に両者は同じと思って問題ありません。
簡潔に書けるため後者のアロー関数を好む人が多いようですが、2015年に導入されたものなので古い環境では動かない可能性があります。

```javascript
const count = n => (
    return n + 1;
);

```

引数が1個であれば()は省略できるようです。

## インポート/エクスポート

インストールしたライブラリや別ファイルでエクスポートした関数などをインポートする方法を説明します。

- ES6以降のやり方

```javascript:export
export const add = (a,b) => {
    return a + b;
}
```

```javascript:import
import {math} = './math';

math.add(1+2);
```

関数にexportと付けるだけでpublicにすることができます。
インポートする際はエディタがインポート文を補完してくれるので手書きすることが少なく楽です。

```javascript:exportDefault
const add = (a,b) => {
    return a + b;
}
const minus = (a,b) => {
    return a - b;
}
export {add,minus};
```

このように複数の関数をまとめてexportすることもできます。

- ES5以前のやり方

時々使うことがあるので古いやり方も載せておきます。

```javascript:export
function add(a,b){
    return a + b;
}

module.export = {
    add:add;
}
```


```javascript:import
const math = require('/math')

math.add(1+2);
```

直感的でないため新しいやり方でやることが多いです。

## 配列

```javascript
const list = [a,b,c]

const immutableList = [1,2,3] as const;
```

配列の宣言はこの通りです。
as constを付けて宣言すると配列内部の値も不変にすることができます。
意図しないところで並び替えされてしまったことがあるので付けおくのが無難です。

配列にはそのコピーを生み出すためのArray.prototypeメソッドが用意されています。

```javascript:map
const list = [1,2,3,4,5]

const copiedList = list.map((item)=>{
    return item + 1;    
})

```

配列の要素を受け取り加工した結果から新しい配列を作成します。

```javascript:forEach
const list = [1,2,3,4,5]

const a =list.forEach(item,index,list) =>{

}

```

配列の要素、添字、全体を受け取り処理が可能です。

```javascript:Array.prototype.filter
const list = [0,1,2,3,4,5]

const newList = list.filter((item)=>{
    return item > 0
})

console.log(newList); // => [1,2,3,4,5]
```

に
boolean値を返すコールバック関数を指定することで、その条件で絞り込んだ結果の配列を取得できるメソッドです。

他のも様々なメソッドが用意されており、面白いので興味があれば調べてみてください。

## TypeScript

TypeScriptは厳密な型定義を設けたJavaScriptと互換性のある言語です。
JSのコードをTSで呼び出すことができ、書き味も型の部分以外変わらないため、JSが分かれば学習コストほぼ無しで導入可能です。
厳密に型チェックが入るおかげで想定外の代入をコンパイル前にチェックできるため、プロジェクト開発ではJavaScriptよりもTypeScriptのほうが利用されます。
型があることで、エディタ上での自動補完も効くようになるため開発体験も良くなります。
非常に楽なので、TypeScriptで始めることをおすすめします。

```typescript:type.ts
export type User {
    id:number;
    name:string;
    
}

```

## おわりに

私が初めてJavaScriptを触れたとき既に多言語を経験済みだったのですが、その際に他言語との言語仕様の違いがわからず迷った経験がありました。
検索しても他言語経験者向けの解説記事はあまり出てこなかったため、そういった方向けに手っ取り早くJSを書けるようになってもらうための記事は需要がありそうだと思い、筆を執らせていただきます。
お役に立てたら幸いです。