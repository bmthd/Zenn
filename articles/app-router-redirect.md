---
title: "【Next.js】App Routerでリダイレクトする方法"
emoji: "🥏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,approuter]
published: true
---
Next.jsのApp Routerでリダイレクト先を動的に決定したい場合、どうやるのかを調べた。

## 前提

ヘッダーやフッターのナビゲーションで最新の記事などにアクセスさせるURLを埋め込んでおき、動的に遷移先を決めたいという要件があった。

![フッターのイメージ](/images/app-router-redirect/screenshot.png)

Server ComponentではuseRouter()は使えず、クライアントコンポーネントでは非同期処理が使えないため、どうやるのか一瞬迷った。
ドキュメントを確認するとnext/navigationにredirect()というそのまんまの関数を発見。
https://nextjs.org/docs/app/api-reference/functions/redirect

## コード

```typescript:/app/event/latest/page.ts
import { fetchLatestEvent } from "@/services/eventService";
import { redirect } from "next/navigation";

const page = async () => {
    const latest = await fetchLatestEvent();
    if (latest) {
        redirect(`/event/${latest.id}`);
    } else {
        redirect(`/`);
    }
};

export default page;
```

JSXを返す必要がないので処理のみを行う非同期関数とし、page.tsとしてexport。
実際に動作させてみたところ爆速で動作した。
これならコーディングも楽だし、ユーザー体験を損ねることなくリダイレクトすることができそう。
