---
title: "【個人開発】楽天市場のキャンペーン攻略サイト作った"
emoji: "🐼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [TypeScript,Nextjs,React,Recoil]
published: false
---
# はじめに
もともとJavaしか開発経験がなかったのでもう一つくらいポートフォリオに書ける言語の経験が欲しいなと思い、モダンなスタックでアプリケーションを作ろうと考えました。
筆者のスペックとしては
- プログラミング系職業訓練校卒業
- Java Silver,Oracle Blonze SQL
- 実務未経験
こんな感じでして、TypeScript自体ガチの未経験状態でGoogle検索とChatGPTを駆使して独学で１ヶ月かけて作り上げました。
テストが無かったり、プロの人から見たら設計が雑な可能性があります。
Qiitaを見ているとたまに現れるめちゃくちゃ理解力のあるつよつよ駆け出しエンジニアと違って、アウトプットも全然してこなかったし、まだ全てを理解できているわけではありません。
目に余るかもしれませんが大目に見てください…！
これからWebアプリを作りたい未経験の方の参考になれば嬉しいです。

# 概要
楽天市場のお買い物マラソンというキャンペーンを攻略するための獲得ポイント計算ツールを作成しました。
https://point-sprint.bmth.dev/
楽天市場では、大きく分けて3種類のキャンペーンがあります。
- SPU 楽天のサービスを利用すればするほど還元率が増える恒常キャンペーン
- お買い物マラソン 毎月2回、定期的に開催される大型キャンペーン
- 突発キャンペーン 数%ポイント還元率が増える基本的に1日だけのキャンペーン

この3つを組み合わせることで20%前後のポイント還元を受けてお買い物ができるのが楽天のベネフィットです。
このうち、お買い物マラソンは購入店舗数に応じてもらえるポイントが変動する仕組みとなっていて、最大9%の還元水増しが可能です。
これを攻略しようというのがコンセプトです。


# 何を解決したかったか

- 誰もが管理不要で使えるツールに
楽天市場ではキャンペーン上限額まであといくら購入できるのかがわからないので、購入計画が立てづらいというデメリットがあります。
そのため、わたしも今まではスプレッドシートであといくら購入できるのかを管理していました。
Excelなどで管理している人は散見されましたが誰もができるわけではありません。
また、お買い物マラソンだけではなく項目が多岐にわたるSPU、更には毎回内容が異なる突発キャンペーンに対応するとなると手間です。
その上キャンペーンが変更されるたびに対応する必要があります。
これを解決します！
- 還元率の明確化
最近ポイントの倍付け制度の仕組みが変わり、購入金額の税引き後の金額に対しての付与になりました。
(例　画面上の還元率10倍→実際には9.09%)
これにより、実際の還元率が直感的に分かりづらくなります。
お得になる割合が不明瞭なまま注文してしまうということが体感上ありました。
それを解消します。

# こだわりポイント
- レスポンシブ対応
モバイルではカード形式カルーセル、PCでは表形式でUIを作り分けました。
初めはモバイルも表形式で作成してみたのですが、スマートフォンで表をいじくるのは難しく専用UIでの対応の必要性を実感しました。
2週間くらいでリリースしたかったのですがこのせいで工数が倍程度になりましたw
- 使っていて楽しいUI
「とりあえず動く」だけでは使ってもらえないと思ったので、見栄えするデザインを研究しました。
やるからにはとことんこだわり、操作していて楽しいUIを目指しました。
SPUの設定にはモーダル、計算結果の表示にカウントアップなど見た目に楽しい機能を入れたのもポイント。
実際の動きもぜひ見てください！
- PC版ではEnterキーでのフォーカスに対応
Excelと同じ感覚で使ってもらえるよう、Enterキーで次のフォーム、Shift+Enterで前のフォームに移動できるようにしました。

# 技術スタック
Next.js,TypeScript,Tailwind CSS,Recoil,Netlify

初めて触る技術ばかりだったので、手探りでいろいろ試しながら技術選定をしていきました。
- Next.js,React
PagesやAPI Routesが直感的で使いやすそうだったので。
Vercelのライブラリがどれも素晴らしく感動してます。
- TypeScript
Pythonや素のJavaScript触ったあとだとわかる。
型があるって良い…！
- Tailwind CSS
みんなが使っていたので。
Bootstrapと使い方が似ていてとても使いやすかったです。
- Recoil
Propsバケツリレーでは状態管理に無理が生じ、必要に迫られて導入しました。
後ほど解説します。
- Netlify
Vercelが便利すぎたので使いたかったのですが商用NGの制約があるのでNetlifyに。
Netlifyは現状日本リージョンのサーバが無いのでLighthouseのスコアが少し落ちました。

# 内部構成紹介

```d
src
├── components
│   ├── carousel //
│   │   └── …
│   ├── common
│   │   └── …
│   ├── layouts
│   │   └── …
│   └── table
│       └── …
├── hooks
│   ├── useFocusControl.ts
│   └── useItemRow.ts
├── pages
│   ├── api
│   │   ├── inquiry.ts
│   │   └── search.ts
│   ├── _app.tsx
│   ├── _document.tsx
│   ├── index.tsx
│   └── inquiry.tsx
├── recoil
│   ├── atom.ts
│   └── selector.ts
├── styles
│   └── grobals.css
└── types
    └── types.ts 

```
お問い合わせページ(inquiry.tsx)を除くと１ページに収まるシンプルな構成です。
- クライアントサイド
```typescript:index.tsx
```

- サーバーサイド
サーバー側での処理はNext.jsのAPI Routeでリクエストを受け取り、レスポンスを返しています。
このアプリでは広告用に楽天市場APIのリクエストを行うsearch.ts,お問い合わせ内容をメールで送信するinquiry.tsの2つです。

# 一部コード紹介

難しかったところや、工夫したところを紹介させていただきます。

### Recoil
ページ部品のコンポーネント化を進めていくうちに、子コンポーネントから孫コンポーネントへ状態を受け渡すことになったり、親子の関係がないコンポーネントでも状態を見に行く必要が出てきてuseStateだけでは無理が生じてきました。
ここで、状態管理ライブラリについて調べ始めました。
Recoilに決めた理由は、比較的新しいこと、React開発元のMeta製ライブラリだからなのもありますが、何より使ってみて非常にシンプルだったことです。

簡単に説明すると
- atom 単一の状態
- atomFamily key-valueで同じ種類のatomの状態を複数保管できる。配列の代わりに使えて全体を再レンダリングせずに済む。
- selector atomや他のSelectorの値をもとに加工した値を返したり、セットしたりできる。
- selectorFamily 受け取ったkeyをもとにatomFamilyを扱えるselector。

これらをuseRecoilValue,useSetRecoilValue,use
途中までは値の状態をもとに計算結果を出力する関数をコンポーネント内に配置していましたが、Recoilを導入したことによってそれらの関数をselectorとして別ファイルに纏められるようになり、表示と処理を分けられるようになり責務が明確化されました。

### useFocusControl.ts
PC版ページでのフォーカス移動は長くなったので別記事にしました。

https://zenn.dev/bmth/articles/react-table-focus


RakutenItem



# 終わりに

この記事を書くなかでも、気付きが多く、動いてはいたものの変なところが見つかったので何箇所か修正しています。
これからはアウトプットもしていこうと思いました。
4月12日に制作を開始して、5月11日にリリースしたので開発期間はちょうど1ヶ月程度でした。
初めてのNext.jsだったのでだいぶ時間がかかってしまいましたが、これで次からはもっと短期間で制作できそうです。
2022年4月からITエンジニアに転職し、開発志望で1年間やってきたのですがめぐり合わせが悪く開発案件にはまだ携われていません…。
ポートフォリオ代わりになればいいなと思って今回自分がほしいサービスを作成してみました。
せっかく作ったからには使ってもらえないと寂しいのでぜひ使ってみてください！
特にPCパーツとか、ふるさと納税、Apple製品を購入するときに使えるApple Giftカードを買うときに火を吹きます。
楽天市場は極めればガチでお得なのでぜひ！