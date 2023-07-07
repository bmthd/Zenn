---
title: "【初心者向け】フルスタック個人開発を見据えたReact,Next.js周りの技術選定"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,]
published: false
---

## はじめに

1年前に自分が知りたかったReact中心に考えたときのWebアプリケーション周りの最適解をまとめます。
これからWebアプリ、サイトを制作したいと思っている方の参考になって欲しいのでできる限り初めての人でもわかりやすい用語を使って解説していきます。
私が個人的に使っている技術がメインになりますので宗教上の理由などで合わない場合もあるかと思いますが予めご了承ください。
公式ドキュメントをご確認ください。
特にNext.jsの公式ドキュメントは非常にわかりやすいのでおすすめです。

## Node.js

Node.jsはJavaScriptで動くサーバーアプリケーションです。
実行したときに足し算をする処理のような1度限りの処理で良ければサーバーは不要ですが、localhost:xxxxで待機しておき、アクセスがあったときに何かをするというような仕組みにはサーバーアプリケーションは必須です。
JavaScriptの中ではデファクトスタンダードなので、基本的にJavaScriptで何かしようと思ったらNode.jsは必須と思って差し支えないです。
OS毎のインストールファイルをサイトからダウンロードすることでインストールできます。
https://nodejs.org/ja

## Reactは何が良い？

個人開発目線のメリットを簡単に言うと、画面を作る際の行ったり来たりが減ることです。
この画像を見てください。

これは私がThymeleafというSpring bootでよく採用されるテンプレートエンジンで作成したページの一部です。
これだけ見ても何がどうなっているのかいまいち伝わってこないと思います。

こちらは同じサイトを後にReactで書き直したものです。
一目瞭然で非常にシンプルです。
まとまりに名前がついており、中の処理を詳しく知らなかったとしてもこれをパズルのピースのように組み合わせることでサイトを組み立てることができます。

Reactはコンポーネント志向で作られており、JSXという技術でJavaScript上にHTMLを直接記述する事ができます。

```typescript:Component.tsx
const Component = () => {
    const list = {
        title: "",
    }
    return(
        <>
         <h1>ページタイトル</h1>
         {list.map((child) => (
            <ChildComponent child={child} />
                )
            )  
         }
    )
}
```

()で囲うことでその中にJSXを記述することができる。
JSXの中で変数や関数を呼び出したいときは{}で囲うことでその内部に記述できる。
もちろん入れ子にできる。

従来のHTMLやテンプレートエンジンでは、一度作成したものと似たものを作る際に再利用ができずに同じものだとしてもコピー・アンド・ペーストする必要がありました。
ReactではまるでHTMLをプログラミングできるかのように要素を再利用し、似ているものは再度作成しなくても引数として渡してあげるだけで作れてしまうのです。
加えて、ReactはJavaScript製です。
どんなフレームワークも最終的に画面に出力する際にHTML,CSS,JavaScriptを返します。
JavaScriptは必須ではありませんが、企業レベルのサイトともなるとJavaScriptを使わないWebサイトはほとんど無いといえるレベルです。
最後に出力されるのがJavaScriptになる都合、テンプレートエンジンとしての役目もJavaScriptに任せてしまったほうが工数が減ります。
後述しますが、Server Componentの登場でバックエンド自体の役目もReactに任せられるようになりつつあるのでいずれはフロントエンドとバックエンドの境界が無くなり1つのフレームワークでどちらもこなす時代が来る可能性すらあります！

※テンプレートエンジンにも再利用できる仕組みはありますが、引数を渡したりはできません。

### Next.js

React単体でもアプリケーションとして動作するのですが、そのままだとURLのルーティング周りや、キャッシュの仕組み、エラーハンドリングなどを自作する必要があります。
そこで、Next.jsを使います。
Next.jsは、クラウドを運営しているVercelが開発したReactフレームワークです。
React公式も何らかのフレームワークを推奨するようになっています。

Reactのフレームワークの中には他にもGatsby.jsという静的サイトジェネレーターに特化したものなどが存在します。知りたい方は調べてみてね。

```shell:console
npm install next react react-dom // Next.jsに最低限必要なパッケージをインストール
npx create-next-app // 新規プロジェクトを作成する際のコマンド
npm run dev // 開発モードでローカルにサーバーを立ち上げるコマンド
```

### Pages RouterとApp Routerの違い

Next.jsには全く別物と言っていいほどに違う2つのモードがあります。

- Pages Router

```dir:directory
src
├── components
│   └── ...
├── pages
│   ├── api
│   │   ├── inquiry.ts // 中に記述した問い合わせ先に
│   │   └── search.ts　
│   ├── _app.tsx // サイト全体に適用されるCSSやメタデータなどを設定するためのファイル
│   ├── document.tsx // サイト全体に適用されるHTML構造を設定するためのファイル(_app.tsxと基本的には同じ)
│   ├── index.tsx // index.tsxがサイトのトップページになる。
│   ├── [slug] // []で囲うとURLのクエリパラメータになり、変数として受け取れる
│   │     └── post.tsx　// "/[slug]/post" フォルダ名とファイル名がそのままディレクトリになる
│   └── grobals.css // 普通のCSSファイル。
└── types
```

Pages Routerではpagesフォルダ配下に置かれたファイルのファイル名がそのままパスに変換されます。
見た目にわかりやすく、覚えるべき内容は少ないので初心者が初めて触れるのにおすすめです。

```typescript:index.tsx

const Home = (data:any) => {
    const buttonState = useState(false);
    return (
        <>
            <h1>何らかのページ</h1>
            <SomeComponent data={data}>
        </>
    )

    
}
export const getServerSideProps = async (fallbackData){
    const data = await fetchSomeData();
    return {
        props:{
            data: data;
            fallback: true;
            revaridate: 3600;
        }
    }
}

export default Home;

```

非同期関数を使ったデータの取得はgetServerSideProps関数内でのみ行なえます。
この名前の関数をトップレベルのページに配置しておくと読み込み前にサーバー側で処理が実行され、ページにデータが渡される格好です。

- App Router

```dir:directory
src
├── components
│   └── ...
├── app
│   ├── api
│   │   ├── inquiry.ts // 中に記述した問い合わせ先に
│   │   └── search.ts　
│   ├── post // フォルダ名のみがディレクトリになる
│   │   ├── [slug] // []で囲うとURLのクエリパラメータになり、変数として受け取れる
│   │   │   └── page.tsx
│   │   └── layout.tsx フォルダ配下のページのみに適用されるHTML構造やCSSを設定するためのファイル
│   ├── page.tsx // ページを表すコンポーネントはpage.tsxのみ
│   ├── layout.tsx // サイト全体に適用されるHTML構造やCSSを設定するためのファイル
│   └── grobals.css // 普通のCSSファイル。layoutでインポートする。
└── types
```

App Routerではpagesの代わりにappフォルダ配下のフォルダ名がそのままパスとして扱われます。

```typescript:SomeComponent.tsx
const SomeComponent = async () => {
    const data = await fetchSomeData();
    return (
            {data.map((...)=>(</>))}
    )
}
export default SomeComponent;
```

```typescript:page.tsx
'use client' //クライアントコンポーネントであることを明示
const Page =  () => {
    const data = await fetchSomeData();
    return (
        <>
            <h1>何らかのページ</h1>
            <SomeComponent data={data}>
        </>
    )
}
export default Page;
```

非同期関数を使ったデータの取得はなんと、コンポーネント内でそのまま行えます。
トップレベルの制約も無いため、この例のようにpageで取得せずにコンポーネント側で取得が可能です。
必要なときに必要なデータを取得することができ、わかりやすいです。
その代わり、

### CSSフレームワーク

Tailwind CSS
Tailwindは初学者に教えるとCSSを覚えないからダメと言う人もいますが、個人的には手っ取り早く成果物を完成させたほうが座学よりよほど学べるところが多いと感じました。
私はFlexboxとGrid LayoutはTailwindで覚えました。
チートシートを見て元のCSSをイメージしながら使っていました。
https://nerdcave.com/tailwind-cheat-sheet
Tailwindだけでも全然いけると思います。

### おすすめライブラリ

- Jest

JavaScriptテスト用フレームワークのデファクトスタンダード。


- Prisma
- StoryBook

### データベースの選択肢

- Firebase
Googleの

- Supabase
PostgresSQL
UIが使いやすく動きも早い。

- PlanetScale
他のDBaaS系サービスと比べると無料枠が多く、スケールした場合の料金も比較的安いというのも魅力です。
GitのようなDBのblanch機能、自動バックアップなど独自性もあり活用できれば便利そうです。
MySQLで構築したデータが自宅サーバーにあり、互換性的に他のSQLに移行が難しかったため採用しました。

### おすすめデプロイ先

商用利用でない場合はVercelが一番良いです。
複雑な設定をせずとも自動でデプロイされるだけではなく、Github上のbranchやcommit毎に差分を保存しておいてくれます。
何らかの不具合が発生した場合に過去のバージョンをいつでも気軽に見に行け、とても楽です。

次点はNetlify。
NetlifyはVercelと違い、商用利用でも問題ありません。
Githubからの自動デプロイにも対応しており、画面も使いやすいです。
ただし、日本リージョンのサーバーが無いためか画像Fetchなどが非常に低速でストレスがあります。
Cloudflareを噛ませたり、画像のみ国内サーバを使うなどの対策は必須です。

## Github Copilot

必須ではありませんが初学者にこそおすすめしたいのがGithub Copilot。
生成AIをプログラミングに最適化したもので、開いているタブに記載されているコードの文脈を読み続きのコードを提案してもらえます。
単調な繰り返しの部分をAIが推測して自動で生成してくれるので、そこに時間を割く必要がなくなります。
空いた時間でサクサク進めることができるのでその時間を学習時間に充てることができます。
単調なコード以外にも、自分が知らなかった書き方を提案してくれることもあるので理解の妨げになるどころかより深い知識をつけることができます。
その名の通りまるでできの良いペアプログラミング相手が常にいるかのような感覚です。
60日間無料で試すことができるのでぜひ試してみてください。
マジで良いです。

## 終わりに
