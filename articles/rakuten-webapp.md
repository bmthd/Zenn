---
title: "【個人開発】ガチ未経験で楽天市場のキャンペーンを攻略するサイト作ったら案件決まった"
emoji: "🐼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [個人開発,TypeScript,Nextjs,React,Recoil]
published: true
---
# はじめに
もともとJavaしか開発経験がなかったのでもう一つくらいポートフォリオに書ける言語の経験が欲しいなと思い、モダンなスタックでアプリケーションを作ろうと考えました。
筆者のスペックとしては
- プログラミング系職業訓練校卒業
- Java Silver,Oracle Blonze SQL
- 実務未経験

こんな感じでして、TypeScript自体完全に未経験の状態でGoogle検索とChatGPTを駆使して独力で１ヶ月かけて作り上げました。
テストが無かったり、プロの人から見たら設計が雑な可能性があります。
QiitaやZennを見ているとたまに現れるめちゃくちゃ理解力のあるつよつよ駆け出しエンジニアと違って、アウトプットも全然してこなかったし、まだ全てを理解できているわけではありません。
ベテランのエンジニアの方には目に余るかもしれませんが生暖かい目で見ていただければと思います…！
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

動作の様子はTwitterに投稿した動画を見るか、実際にアクセスしてご確認ください。
https://twitter.com/j_ktwr/status/1656241836544135169


# 何を解決したかったか

- 誰もが管理不要で使えるツールに

楽天市場ではキャンペーン上限額まであといくら購入できるのかがわからないので、購入計画が立てづらいというデメリットがあります。
そのため、わたしも今まではスプレッドシートであといくら購入できるのかを管理していました。
Excelなどで管理している人は散見されましたが誰もができるわけではありません。
また、お買い物マラソンだけではなく項目が多岐にわたるSPU、更には毎回内容が異なる突発キャンペーンに対応するとなると手間です。
その上キャンペーンが変更されるたびに対応する必要があります。
これを解決します。

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
│   ├── carousel //スマホ向けコンポーネント
│   │   └── …
│   ├── common //共通部品
│   │   └── …
│   ├── layouts //ヘッダー、フッターなど
│   │   └── …
│   └── table //PC向けコンポーネント
│       └── …
├── hooks
│   ├── useFocusControl.ts //PC表示でのフォーカス移動に使うフック
│   └── useItemRow.ts　//行操作用フック
├── pages
│   ├── api
│   │   ├── inquiry.ts //メール送信用API
│   │   └── search.ts //広告取得用API
│   ├── _app.tsx
│   ├── _document.tsx
│   ├── index.tsx //トップページ
│   └── inquiry.tsx //お問合せページ
├── recoil
│   ├── atom.ts
│   └── selector.ts
├── styles
│   └── grobals.css
└── types
    └── types.ts 

```
お問い合わせページ(inquiry.tsx)を除くと１ページに収まるシンプルな構成です。

```typescript:index.tsx
import Head from 'next/head';
import { useEffect } from 'react';
import { useSetRecoilState } from 'recoil';
import { useMediaQuery } from 'react-responsive';
import { isMobileState } from '@/recoil/atom';
import { CarouselFooter, ItemCarousel } from '@/components/carousel';
import { Adsense, CalculateResult, Information, RakutenItems, SpuModal } from '@/components/common';
import { TableHeader, TableBody, TableFooter } from '@/components/table';

export default function Home() {
	const setIsMobile = useSetRecoilState(isMobileState);
	const isMobile = useMediaQuery({ maxWidth: 1024 });

	useEffect(() => {
		setIsMobile(isMobile);
	}, [isMobile]);

	return (
		<>
			<Head>
				<title>ポイントスプリント 楽天市場お買い物マラソン攻略計算ツール</title>
			</Head>

			<div className="lg:hidden">
				<Information />
				<div className="bg-blue-100 m-6">
					<Adsense />
				</div>
				<SpuModal />
				<ItemCarousel />
				<CarouselFooter />
			</div>

			<div className="max-lg:hidden">
				<div className="flex max-w-5xl justify-center items-center mx-auto">
					<Information />
					<SpuModal />
				</div>
				<form className="grid justify-center">
					<table className='border-spacing-0'>
						<TableHeader />
						<tbody>
							<TableBody />
							<TableFooter />
						</tbody>
					</table>
				</form>
			</div>

			<CalculateResult />
			<div className="bg-blue-100 m-6">
				<Adsense />
			</div>
			<div className="flex flex-col justify-center items-center">
				<RakutenItems />
			</div>
		</>
	);
}
```
- クライアントサイド

スマホ表示とPC表示でそれぞれ専用のコンポーネントをCSSで出し分けています。

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
- atom
  単一の状態
- atomFamily
  key-valueで同じ種類のatomの状態を複数保管できる。配列の代わりに使えて全体を再レンダリングせずに済む。
- selector
  atomや他のSelectorの値をもとに加工した値を返したり、セットしたりできる。
- selectorFamily
  受け取ったkeyをもとにatomFamilyを扱えるselector。

これらをuseRecoilValueフックを使うことでuseStateと同様の呼び出し方で呼び出すことができます。
途中までは値の状態をもとに計算結果を出力する関数をコンポーネント内に配置していましたが、Recoilを導入したことによってそれらの関数をselectorとして別ファイルに纏められるようになり、表示と処理を分けられるようになり責務が明確化されました。

### useFocusControl.ts
PC版ページでのフォーカス移動について、別記事に詳しく纏めました。

https://zenn.dev/bmth/articles/react-table-focus



# 終わりに
この記事を書いている最中におかしなところに気づき、修正した箇所が結構あります。
これからはアウトプットもしていこうと思いました。

制作は4月12日に開始し、5月11日にリリースしたため、開発期間はちょうど1ヶ月程度でした。
初めて触るフレームワークだったのでかなり時間がかかってしまいましたが、次回からはもっと短期間で制作できそうです。

2022年4月にITエンジニアに転職し、開発の道を志して1年間頑張ってきましたが、開発案件に関わることができませんでした。
しかし、このサービスを営業に見せたり面談でアピールした結果、**来月からJavaの案件に参画できることが決まりました…！**
ポートフォリオを見せることの重要性を改めて感じました。

今回はポートフォリオ代わりになればいいなと思って、自分が欲しいと思うサービスを作成してみました。
もしこれを参考になにか作るのであれば、データベース接続やoAuth認証を組み込んだほうがよりアピールに繋がると思います。

目的の一部である開発案件への参画が決まったとはいえ、せっかく作ったので使ってもらえないと寂しいです。
もう一度リンクを貼るのでぜひ、このサービスを試してみてください！
特にPCパーツや、ふるさと納税、Apple製品購入時に使えるApple Giftカードを購入するときに火を吹きます。
**楽天市場を徹底的に極めると本当にお得なので、ぜひご利用ください！**

https://point-sprint.bmth.dev/