---
title: "App Routerで考えるディレクトリ構成"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,approuter,react]
published: false
---

## はじめに

Next.js14でServer Actionsがstableになったことで、導入を検討し始めているところもあるのではないでしょうか。
App RouterではPages Routerと違い、ファイル名でのルーティングでは無くなったので、そのファイルで使われる機能はその近くに配置するという、コロケーションという概念を実践可能になりました。
それを踏まえた私なりのディレクトリ構成をまとめてみました。

考え方の根底には、コロケーションやpackage by featureと呼ばれる概念と、ディレクトリ自体の責務の単一化があります。
特に何かを参考にせずとも、その原則を考えていけば自ずとシンプルな構成が実現できるはずです！

```
 ├─src
    ├─app
 │  │  ├─(static)
 │  │  │  ├─compliance
 │  │  │  ├─cookie-policy
 │  │  │  ├─data-security
 │  │  │  ├─digital
 │  │  │  ├─guide
 │  │  │  │  ├─action
    │  │  │  └─product
    │  │  ├─legal-check
    │  │  ├─peps
    │  │  ├─privacy-policy
    │  │  ├─shikin-kessai
    │  │  ├─tokutei
    │  │  └─tos
    │  ├─add-listing
    │  │  ├─complete
    │  │  └─_listingForm
    │  ├─age-check
    │  ├─api
    │  │  └─auth
    │  │      └─[...nextauth]
    │  ├─inquiry
    │  ├─listing
    │  │  └─[id]
    │  │      └─_commentSection
    │  ├─mypage
    │  │  ├─draft
    │  │  ├─listings
    │  │  ├─personal-info
    │  │  └─purchases
    │  ├─no-available-service
    │  ├─search
    │  ├─transactions
    │  │  └─[transactionId]
    │  └─_layout
    │      ├─footer
    │      └─header
    ├─ui
    │  ├─dialog
    │  ├─form
    │  │  ├─imageInput
    │  │  └─securityVerifier
    │  ├─itemsList
    │  └─structure
    ├─constants
    ├─images
    ├─lib
    ├─middlewares
    ├─services
    ├─test
    ├─types
    └─utils
│  .env
│  .env.example
│  .eslintrc.cjs
│  .gitignore
│  .npmrc
│  .prettierignore
│  .prettierrc.cjs
│  docker-compose.yml
│  jest.config.js
│  jest.setup.ts
│  next-env.d.ts
│  next.config.js
│  package.json
│  pnpm-lock.yaml
│  postcss.config.js
│  prettier.config.js
│  README.md
│  tailwind.config.ts
│  tsconfig.build.json
│  tsconfig.json
```

## srcディレクトリを作り、エイリアスを設定する

```dir
 ├─ src
 │   ├─ app
 │   ├─ components
 │   ├─ hooks
 │   ├─ types
 │   └─ utils
 ├─ .env
 ├─ eslint.config.js
 ├─ jest.config.js
 ├─ next-env.d.ts
 ├─ next.config.js
 ├─ package.json
 ├─ pnpm-lock.yaml
 ├─ postcss.config.js
 ├─ prettier.config.js
 ├─ README.md
 └─ tsconfig.json
```

srcディレクトリはpagesとappを併用するためだけのものではありません。
その命名からソースコードしか格納されていないことがわかるので、コード中でインポートしないconfig系ファイルと、ソースコードを分離し、明確化できるメリットがあります。
プロジェクトが大きくなると直下のファイルがconfigだらけになることは必然。
なので、それらとインポート対象ディレクトリは分けて見られた方がスッキリします！
また、srcに@/でエイリアスを設定してインポートできるようにすることで、パスの見通しを確保するとともに、ルートのファイルはインポートしないということを明示します。

```json:tsconfig.json
{
  "compilerOptions": {
    //...省略
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

## appディレクトリにはルーティング対象のディレクトリ、ファイルのみを配置する

appはルーティングのためのディレクトリです。
その中にコンポーネントだけをまとめたcomponentsのような、ルーティングしないディレクトリがあると、中を見にいかなければ/comopnentsというURLがあるのか、コンポーネントが格納されているのかがわからなくなってしまいます。
特に、appに関してはプロジェクトが大きくなると大量のページで溢れるのでcomponentsがどこにあるのかを探すのも大変になるというデメリットもあります。
そのようなディレクトリは別で切り出すべきです。

```dir:not good
├─ app
│   ├─ about
│   ├─ components // コンポーネントが格納されているのか、ページなのかがわからない
│   ├─ contact
│   ├─ products
│   ├─ layout.tsx
│   └─ page.tsx
```

```dir:good
├─ app
│   ├─ about
│   ├─ contact
│   ├─ products
│   ├─ layout.tsx
│   └─ page.tsx
├─ components
```

## 共通系のコンポーネントはuiディレクトリに置く

定番はcomponentsという命名ですが、コンポーネントが局所的に利用するhooksやutil関数、型定義などはその近くに置いておきたくなります。
その場合コンポーネント以外もcomponentsに含まれてしまい違和感があるためuiという命名にしています。

## そのページ固有のファイルはコロケーションする

そこでしか使わないファイルは近くに置いておいた方が、依存関係が明確化します。
アンダースコア始まりのファイルはルーティング対象外にできるので、それを利用します。
そのページでインポートするものが1つだけの場合は同一ディレクトリに置き、2つ以上になってからフォルダを切るようにしています。
また、ページで使うコンポーネント内で使うものが2つ以上になった場合は更にディレクトリを切って、_

## 2つ以上の

## named exportを使い、再エクスポートを行う

default exportは、1つのファイルから複数のファイルをエクスポートできないのでnamed exportに統一するようにしています。
例えば、inputにcssを当てただけのtype毎に似たようなコンポーネントは別々のファイルにするより、1つにまとめておいた方が見通しが良くなります。
また、再エクスポートを行えば複数ファイルを同一のパスでインポートできるようになり、インポート元の記述を簡素化することができます。
注意点としては、一部のコードでserver側とclient側を同時に同じindex.tsを介してimportするとエラーが出てしまいます。
単体なら再エクスポートしないようにし、複数になる場合はserverとclientでフォルダを切って行うようにしています。

## ESLint Import Accessを使う

Javaなどのオブジェクト指向言語では一般的なパッケージごとの

```shell
│  .env
│  .env.example
│  .eslintrc.cjs
│  .gitignore
│  .npmrc
│  .prettierignore
│  .prettierrc.cjs
│  docker-compose.yml
│  jest.config.js
│  jest.setup.ts
│  next-env.d.ts
│  next.config.js
│  package.json
│  pnpm-lock.yaml
│  postcss.config.js
│  prettier.config.js
│  README.md
│  tailwind.config.ts
│  tsconfig.build.json
│  tsconfig.json
│  
├─.github
│  │  CODEOWNERS
│  │  pull_request_template.md
│  │  
│  └─workflows
│          ai-review.yaml
│          branch_restrictions.yml
│          test.yml
│          

├─.vscode
│      extensions.json
│      settings.json
│      
├─db
│      dump.gz
│      ERD.svg
│      mongodb_dump.sh
│      mongodb_restore.sh
│      replicaset_init.sh
│      schema.prisma
│      seed.js
│      
├─docs
│      CODINGME.md
│      page.md
├─public
│      manifest.json
│      
└─src
    │  middleware.ts
    │  
    ├─app
    │  │  error.tsx
    │  │  favicon.ico
    │  │  globals.css
    │  │  layout.tsx
    │  │  loading.tsx
    │  │  not-found.tsx
    │  │  opengraph-image.png
    │  │  page.tsx
    │  │  
    │  ├─(static)
    │  │  ├─compliance
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─cookie-policy
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─data-security
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─digital
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─guide
    │  │  │  ├─action
    │  │  │  │      page.tsx
    │  │  │  │      
    │  │  │  └─product
    │  │  │          page.tsx
    │  │  │          
    │  │  ├─legal-check
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─peps
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─privacy-policy
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─shikin-kessai
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─tokutei
    │  │  │      page.tsx
    │  │  │      
    │  │  └─tos
    │  │          page.tsx
    │  │          
    │  ├─add-listing
    │  │  │  ListingForm.tsx
    │  │  │  page.tsx
    │  │  │  PriceInput.tsx
    │  │  │  
    │  │  ├─complete
    │  │  │      CompletedListing.tsx
    │  │  │      page.tsx
    │  │  │      
    │  │  └─_listingForm
    │  │          actions.ts
    │  │          index.ts
    │  │          ListingTagInput.tsx
    │  │          SubmitContainer.tsx
    │  │          TestDataButton.tsx
    │  │          types.ts
    │  │          
    │  ├─age-check
    │  │      AgeCheckContainer.tsx
    │  │      page.tsx
    │  │      server.ts
    │  │      
    │  ├─api
    │  │  └─auth
    │  │      └─[...nextauth]
    │  │              route.ts
    │  │              
    │  ├─inquiry
    │  │      actions.ts
    │  │      mailConfig.ts
    │  │      MailForm.tsx
    │  │      page.tsx
    │  │      types.ts
    │  │      
    │  ├─listing
    │  │  │  page.ts
    │  │  │  
    │  │  └─[id]
    │  │      │  actions.ts
    │  │      │  Carousel.tsx
    │  │      │  CommentSection.tsx
    │  │      │  page.tsx
    │  │      │  protAction.ts
    │  │      │  ProtButton.tsx
    │  │      │  PurchaseButton.tsx
    │  │      │  
    │  │      └─_commentSection
    │  │              DeleteModal.tsx
    │  │              index.ts
    │  │              ReportModal.tsx
    │  │              state.ts
    │  │              TestTransactionButton.tsx
    │  │              
    │  ├─mypage
    │  │  │  page.tsx
    │  │  │  
    │  │  ├─draft
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─listings
    │  │  │      page.tsx
    │  │  │      
    │  │  ├─personal-info
    │  │  │      actions.ts
    │  │  │      AddressForm.tsx
    │  │  │      page.tsx
    │  │  │      type.ts
    │  │  │      
    │  │  └─purchases
    │  │          page.tsx
    │  │          
    │  ├─no-available-service
    │  │      close.webp
    │  │      page.tsx
    │  │      
    │  ├─search
    │  │      page.tsx
    │  │      
    │  ├─transactions
    │  │  │  page.ts
    │  │  │  
    │  │  └─[transactionId]
    │  │          actions.ts
    │  │          MessageSection.tsx
    │  │          page.tsx
    │  │          TransactionStatus.tsx
    │  │          
    │  └─_layout
    │      │  AnchorMenu.tsx
    │      │  anchorMenuHooks.ts
    │      │  ClientProvider.tsx
    │      │  Container.tsx
    │      │  GoogleTagManager.tsx
    │      │  index.ts
    │      │  viewPort.ts
    │      │  
    │      ├─footer
    │      │      Footer.tsx
    │      │      FooterContents.tsx
    │      │      FooterIcons.tsx
    │      │      index.ts
    │      │      LinkContents.ts
    │      │      
    │      └─header
    │              Header.tsx
    │              index.ts
    │              SearchWindow.tsx
    │              TitleLogo.tsx
    │              UserMenuButton.tsx
    │              
    ├─components
    │  │  Badge.tsx
    │  │  Button.tsx
    │  │  index.ts
    │  │  ListingButton.tsx
    │  │  PaginationBar.tsx
    │  │  Skeleton.tsx
    │  │  TextLink.tsx
    │  │  
    │  ├─dialog
    │  │      Dialog.tsx
    │  │      index.ts
    │  │      useDialog.tsx
    │  │      useFormActionModal.tsx
    │  │      useImageModal.tsx
    │  │      
    │  ├─form
    │  │  │  Elements.tsx
    │  │  │  hooks.ts
    │  │  │  index.ts
    │  │  │  LimitElements.tsx
    │  │  │  SubmitButton.tsx
    │  │  │  type.ts
    │  │  │  utils.ts
    │  │  │  
    │  │  ├─imageInput
    │  │  │      addGrayBackground.ts
    │  │  │      fetcher.ts
    │  │  │      ImageInput.tsx
    │  │  │      ImagePreview.tsx
    │  │  │      index.ts
    │  │  │      
    │  │  └─securityVerifier
    │  │          fetcher.ts
    │  │          hooks.ts
    │  │          type.ts
    │  │          verifyForm.ts
    │  │          VerifyProvider.tsx
    │  │          
    │  ├─itemsList
    │  │      index.ts
    │  │      ItemsList.tsx
    │  │      ItemsListContainer.tsx
    │  │      ListingCard.tsx
    │  │      SoldOutBadge.tsx
    │  │      
    │  └─structure
    │          context.ts
    │          H.tsx
    │          index.ts
    │          Main.tsx
    │          PageTitle.tsx
    │          Section.tsx
    │          TitleUnderbar.tsx
    │          
    ├─constants
    │      constantsSeed.ts
    │      listing.ts
    │      prefectures.ts
    │      site.ts
    │      
    ├─images
    │      logo.png
    │      profile-pic-placeholder.png
    │      
    ├─lib
    │      ImageUploadCloudinary.ts
    │      mail.ts
    │      prisma.ts
    │      
    ├─middlewares
    │      ageCheck.ts
    │      auth.ts
    │      exclude.ts
    │      
    ├─services
    │      address.ts
    │      listing.ts
    │      listingComment.ts
    │      tag.ts
    │      transaction.ts
    │      
    ├─test
    │      config.test.ts
    │      sample.ts
    │      
    ├─types
    │      next-auth.d.ts
    │      window.d.ts
    │      
    └─utils
            converter.test.ts
            converter.ts
            format.ts
            index.ts
            parseRelativeTime.ts
            serverEnv.ts
            session.ts
            typeGuard.ts

```
