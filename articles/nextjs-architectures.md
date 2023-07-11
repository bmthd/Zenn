---
title: "【初心者向け】フルスタック個人開発を見据えたReact,Next.js周りの技術選定"
emoji: "🏠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,]
published: false
---

## はじめに

1年前に自分が知りたかったReact中心に考えたときのWebアプリケーション周りの最適解とその手順を順を追ってまとめます。
これからWebアプリ、サイトを制作したいと思っている方の参考になって欲しいのでできる限り初めての人でもわかりやすい用語を使って解説していきます。
取っ掛かりに必要な知識のみ解説するので、詳細は公式ドキュメントを確認するようにしてください。
特にNext.jsの公式ドキュメントは非常にわかりやすいのでおすすめです。
私が個人的に使っている技術がメインになりますので宗教上の理由などで合わない場合もあるかと思いますが予めご了承ください。
間違っていたらコメントで遠慮なく指摘お願いします！

## Node.js

Node.jsはJavaScriptで動くサーバーアプリケーションです。
実行したときに足し算をする処理のように1度限りの処理で良ければサーバーは不要ですが、localhost:xxxxで待機しておき、アクセスがあったときに何かをするというような仕組みにはサーバーアプリケーションは必須です。
基本的にJavaScriptで何かしようと思ったらNode.jsは必須と思って差し支えないです。
OS毎のインストールファイルをサイトからダウンロードすることでインストールできます。
https://nodejs.org/ja

## Reactは何が良い？

個人開発目線のメリットを簡単に言うと、画面を作る際の行ったり来たりが減ることです。
このHTMLを見てください。(長いのでさらりと見るだけで大丈夫です)

```html:block.html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
 xmlns:th="http://www.thymeleaf.org/"
 xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"
 layout:decorate="~{layout/base}">
<head>
<meta charset="UTF-8">
<title
 th:text="${event.Id} + ' ' + ${day.dayCount} + '日目ブロック&quot;' + ${block.Name} + '&quot;お品書きまとめ'">ブロック詳細ページ</title>
<script defer src="/javascript/createLink.js"></script>
</head>
<body>
 <div layout:fragment="contents" class="container">
  <script async="false" src="https://platform.twitter.com/widgets.js"
   charset="utf-8"></script>

  <ul class="row c-header">
   <li class=" col-6 col-md-3 " th:each="circleView : ${circleViews}">
    <a class="btn btn-info btn-block btn-sm m-1 font-weight-bold"
    style="background-color: #87cefa; color: rgb(255, 255, 255); text-overflow: ellipsis; overflow: hidden; white-space: nowrap;"
    th:text="${circleView.blockName} + ${circleView.spaceNumber} + ${circleView.ab} + ' ' + (${circleView.circleName} ? ${circleView.circleName} : '')"
    th:href="'#'+${circleView.spaceNumber} + ${circleView.ab}"></a>
   </li>
  </ul>

  <ul class="row"
   style="border-radius: 5px; border: 3px dashed #6495ed; background: #e0f0ff;">
   <li class="col-xs-12 col-md-6 col-lg-3"
    th:each="circleView : ${circleViews}">
    <h3 class="text-center"
     th:id="${circleView.spaceNumber} + ${circleView.ab}">
     <span
      th:text="${circleView.dayCount} + '日目' + ${circleView.hallName} + ${circleView.blockName} + '-' + ${#numbers.formatInteger(circleView.spaceNumber, '2')} + ${circleView.ab}"
      style="white-space: pre-wrap;">スペース番号</span>
    </h3>
    <h4>サークル名</h4> <a th:href="'/circle/' + ${circleView.circleId}"
    target="_blank" rel="noopener" th:text="${circleView.circleName}"
    class="font-weight-bold" style="white-space: pre-wrap;"></a> <th:block
     th:if="${circleView.circleName == null||circleView.circleName == ''}">データなし
    </th:block>

    <h4>執筆者名</h4> <th:block th:if="${circleView.author != null}">
     <span th:text="${circleView.author}">執筆者名</span>
    </th:block> <th:block
     th:if="${circleView.author == null||circleView.author == ''}">データなし
    </th:block>

    <h4>Webサイト</h4> <th:block
     th:if="${circleView.webcatalogUrl != null}">
     <a th:href="${circleView.webcatalogUrl}" target="_blank"
      rel="noopener" style="white-space: pre-wrap;">コミケWebカタログ</a>
    </th:block>
    <div th:if="${circleView.pixivUrl != null}">
     <a th:href="${circleView.pixivUrl}" target="_blank" rel="noopener"
      style="white-space: pre-wrap;">pixiv</a>
    </div>
    <div th:if="${circleView.hpUrl != null}">
     <a th:href="${circleView.hpUrl}" target="_blank" rel="noopener"
      style="white-space: pre-wrap;">ホームページ</a>
    </div>
    <div
     th:if="${circleView.webcatalogUrl == null && circleView.pixivUrl == null && circleView.hpUrl == null}">データなし

    </div>

    <h4>Twitter</h4> <script th:inline="javascript">
     var tweetAuthor = /*[[${circleView.twitterId}]]*/null;
    </script>
    <blockquote th:if="${circleView.tweetText!=null}"
     class="twitter-tweet" data-lang="ja" dir="ltr">
     <p lang="ja" dir="ltr" th:text="${circleView.tweetText}"></p>
     <script>
      document.write("&mdash; (" + tweetAuthor + ")");
     </script>
     <a th:href="${circleView.tweetUrl} +'?ref_src=twsrc%5Etfw'"
      th:text="${#dates.format(new java.util.Date(circleView.tweetCreatedAt), 'yyyy年MMMdd日')}">MMM
      dd,yyyy</a>
    </blockquote> <th:block th:if="${circleView.tweetText==null}">
     <a th:href="'https://twitter.com/'+${circleView.twitterId}"
      th:text="'@'+${circleView.twitterId}"></a>
     <p>このサークルはまだお品書きを公開していないようです。</p>
    </th:block>
   </li>
  </ul>
  <script th:inline="javascript">
    var eventId = /*[[${event.Id}]]*/'null';
  </script>

 </div>
</body>
</html>
```

これは私がSpring bootで採用されるThymeleafというテンプレートエンジンで作成したページの一部です。
これだけを見ても前提知識が無い人には何がどうなっているのかがすぐには伝わって来ないと思います(私の書き方が下手というのもあるとは思いますが)。

そして、こちらは同じサイトを後にReactで書き直したものです。

```typescript:block/[bockName]/page.tsx
export const generateStaticParams = async ({
  params: { eventId, dayCount},
}: {
  params: { eventId: string; dayCount: string; };
}) => {
  const blockNames = await fetchBlockNames(eventId);
  return blockNames.map((blockName) => ({
    eventId: eventId,
    dayCount: dayCount,
    blockName: blockName,
  }));
}

const Page = async ({
  params,
  searchParams,
}: {
  params: { eventId: string; dayCount: string; blockName: string };
  searchParams?: { page?: string; size?: string };
}) => {
  const page = convertToNumber(searchParams!.page!)||1;
  const size = convertToNumber(searchParams!.size!)||38;
  const dayCount = parseInt(params.dayCount);
  const event = await fetchEvent(params.eventId);
  const days = await fetchDays(params.eventId);
  const day = await fetchDay(params.eventId, dayCount);
  const blockName = decodeURIComponent(params.blockName);
  const block = await fetchBlock(params.eventId, blockName);
  const spaces = await fetchSpaces(params.eventId, dayCount, blockName,page,size);
  const count = await fetchSpaceCountByBlock(params.eventId, dayCount, blockName);
  const totalPage = Math.ceil(count / size);
  const lastUpdate =spaces.flatMap((space) => space.tweets)
    .map((tweet) => tweet.createdAt)
      .reduce((latest, current) => (current! > latest! ? current : latest), null)!.toLocaleString("ja-JP");
  const pageTitle = `${event.id} ${day.dayCount}日目ブロック\"${block?.name}\"お品書きまとめ`;

  return (
    <>
    <Head>
      <title>{pageTitle}</title>
    </Head>
        <H2>{pageTitle}</H2>
        <span>最終更新日時:{lastUpdate}</span>
        <Pagenation className='my-4' current={page} total={totalPage} />
        <SpaceListHeader spaces={spaces} />
        <SpaceList spaces={spaces} />
        <Pagenation className='my-4' current={page} total={totalPage} />
        <div className="max-w-md mx-auto">
        <WallList event={event} />
        <BlockListForm event={event} days={days} />
        </div>
    </>
  );
};
export default Page;
```

非常にシンプルで一目瞭然です。
まとまりに名前がついており、これをパズルのピースのように組み合わせることでサイトを組み立てることができます。
Reactはコンポーネント志向で作られており、JSXという技術でJavaScript上にHTMLを直接記述する事ができます。

```typescript:Component.tsx
const Component = () => {
    const list = [{
        title: "1つ目の項目",
        body:"こんにちは"
    },{
        title: "2つ目の項目",
        body:"ハロー"
    }]
    return(
        <>
         <h1>ページタイトル</h1>
         {list.map((child) => (
            <ChildComponent child={child} />
            ));}
        </>
    )
}
```

()で囲うことでその中にJSXを記述することができる。
JSXの中で変数や関数を呼び出したいときは{}で囲うことでその内部に記述できる。
もちろん入れ子にできる。

従来のHTMLやテンプレートエンジンでは、一度作成したものと似たものを作る際にコピー・アンド・ペーストが必要で再利用ができませんでした。※
ReactではまるでHTMLをプログラミングできるかのように要素を再利用し、似ているものは再度作成しなくても引数として渡してあげるだけで作れてしまうのです。
加えて、ReactはJavaScript製です。
どんなフレームワークも最終的に画面に出力する際にHTML,CSS,JavaScriptを返します。
無くても動きますが凝ったUIや演出にはJavaScriptが必須なため、JavaScriptを使わないWebサイトはほとんど無いといえるレベルです。
最後に出力されるのがJavaScriptになる都合、テンプレートエンジンとしての役目もJavaScriptに任せてしまったほうが工数が減ります。
また、Reactはサーバー側で予めJavaScriptを実行した結果のみをクライアント側に送信する仕組みがあるのでサーバークライアント間の通信容量を軽量化しやすいです。
後述しますが、Server Componentの登場でバックエンド自体の役目もReactに任せられるようになりつつあるのでいずれはフロントエンドとバックエンドの境界が無くなり1つのフレームワークでどちらもこなす時代が来る可能性すらあります。

※テンプレートエンジンにも再利用できる仕組みはありますが、引数を渡したりはできません。

### Next.js

React単体でもアプリケーションとして動作するのですが、そのままだとURLのルーティング周りや、キャッシュの仕組み、エラーハンドリングなどを自作する必要があります。
そこで、Next.jsを使います。
Next.jsは、ホスティングサービスを運営しているVercelが開発したReactフレームワークです。

Reactのフレームワークは他にもGatsby.jsという静的サイトジェネレーターに特化したものなどが存在します。知りたい方は調べてみてください。

```shell:console
npm install next react react-dom # Next.jsに最低限必要なパッケージをインストール
npx create-next-app # 新規プロジェクトを作成する際のコマンド
npm run dev # 開発モードでローカルにサーバーを立ち上げるコマンド
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
│   │   └── search.ts　HTMLを返さずにHTTPパラメータだけを返す
│   ├── _app.tsx // サイト全体に適用されるCSSやメタデータなどを設定するためのファイル
│   ├── document.tsx // サイト全体に適用されるHTML構造を設定するためのファイル(_app.tsxと基本的には同じ)
│   ├── index.tsx // index.tsxがサイトのトップページになる。
│   ├── [slug] // []で囲うとURLのクエリパラメータになり、変数として受け取れる
│   │     └── post.tsx　// "/[slug]/post" フォルダ名とファイル名がそのままディレクトリになる
│   └── grobals.css // 普通のCSSファイル。
└── types
```

Pages Routerではpagesフォルダ配下に置かれたファイルのファイル名がそのままパスに変換されます。
見た目にわかりやすく、シンプルなので小さなプロジェクトなどにおすすめです。

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

export const getServerSideProps = async (){
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
コンポーネント内ではデータの取得が行えないので、propsとして受け取る必要があります。

- App Router

```dir:directory
src
├── components
│   └── ...
├── app
│   ├── api
│   │   └── search
│   │       └── route.ts // HTMLを返さずにHTTPパラメータだけを返す
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
ファイル名はパスとして扱われず、フォルダ配下の特定のファイル名が意味を持つ仕組みになっています。
API用ファイルはroute.ts,ページ用ファイルはpage.tsx,レイアウト用ファイルはlayout.tsxといった具合です。
いままでは特定のディレクトリ配下に共通のCSSを適用したい場合は全てのページでインポートが必要でしたがApp Routerではそのディレクトリにlayout.tsxを置いておけばそれ以下は自動で共通レイアウトが適用できるようになりました。
Pages Routerと比べると少し複雑で学習コストが高いこと、まだ出たばかりなのもあり情報量は少ないこと、開発途上の機能もあるので一部自前で実装が必要なことはデメリットですが、これらが苦にならないのであればApp Routerがおすすめです。

```typescript:SomeComponent.tsx
'use client' //クライアントコンポーネントであることを明示

const SomeComponent = () => {
    useEffect(()=>{
    processingSomeData()
    ,[]
    })
    return (
            {data.map((...)=>(</>))}
    )
}
export default SomeComponent;
```

```typescript:page.tsx
export const revaridate = 3600;

const page = async () => {
const data = await fetchSomeData();
    return (
        <>
            <h1>何らかのページ</h1>
            <SomeComponent />
        </>
    )
}
export default page;
```

非同期関数を使ったデータの取得はコンポーネント内でそのまま行えます。
トップレベルの制約も無いため、この例のようにpageで取得せずに末端コンポーネント側で取得が可能です。
必要なときに必要なデータを取得することができ、わかりやすいです。
その代わり、クライアント側の操作に関連するhooksは'use client'を付けたクライアントコンポーネント内でしか行うことができません。

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
GoogleのNoSQL

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
