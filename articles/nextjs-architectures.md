---
title: "個人開発を見据えたNext.js周りの技術"
emoji: "🖼"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,]
published: false
---

## はじめに

## 概要

### Next.js

### Pages RouterとApp Routerの違い

'''
src
├── components
│   └── ...
├── pages
│   ├── api
│   │   ├── inquiry.ts
│   │   └── search.ts
│   ├── _app.tsx
│   ├── document.tsx
│   ├── index.tsx
│   ├── [slug]
│   │     └── post.tsx
│   └── grobals.css
└── types
'''

### CSSフレームワーク

Tailwind CSS
Tailwindは初学者に教えるとCSSを覚えないからダメと言う人もいますが、個人的には手っ取り早く成果物を完成させたほうが座学よりよほど学べるところが多いと感じました。
私はFlexboxとGrid LayoutはTailwindで覚えました。
チートシートを見て元のCSSをイメージしながら使っていました。
https://nerdcave.com/tailwind-cheat-sheet
Tailwindだけでも全然いけると思います。

### おすすめライブラリ

- Jest
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