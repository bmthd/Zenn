---
title: "【Next.js】App Routerでリダイレクトする方法"
emoji: "🥏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,approuter]
published: true
---
Next.jsのApp Routerでリダイレクト先を動的に決定したい場合、どうやるのかを調べた。

## 前提

ヘッダーやフッターのナビゲーションで最新の記事などにアクセスさせるURLを埋め込んでおき、そのURLにアクセスがあったときに動的に遷移先を決めたいという要件があった。

![フッターのイメージ](/images/app-router-redirect/screenshot.png)
*フッターのイメージ*

Server ComponentではuseRouter()などのブラウザを直接操作するhooksは使えず、クライアントコンポーネントでは非同期処理が使えないため最新の情報を取得できないので、どうやるのか一瞬迷った。
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

## 追記

App Router調べ始めの頃に書いた記事ですが、結構アクセスされていたのでもう少し詳しく書きます。

リダイレクトの方法は3種類あります。
- Middlewareを使用したリダイレクト
- サーバーコンポーネントでのリダイレクト
- クライアントコンポーネントでのリダイレクト

上記のような実装はサーバーコンポーネントでのリダイレクトとなりますが、リダイレクト処理以外にも不要なページ読み込みが発生したあとに転送されるという挙動になってしまいます。
このような処理は、Middlewareで実装すべきです。

### Middlewareを使用したリダイレクト

```ts:middleware.ts
import { fetchLatestEvent } from "@/repositories/eventRepository";
import { NextRequest, NextResponse } from "next/server";

export const middleware = async (request:NextRequest) => {
  const { nextUrl } = request;
  const { pathname } = nextUrl;
  if (pathname.startsWith("/event/latest")) {
    const {id:eventId} = await fetchLatestEvent();
    if(eventId){
      return NextResponse.redirect(new URL(`/event/${eventId}`,nextUrl))
    } else {
      return NextResponse.redirect(new URL("/event",nextUrl))
    }
  }
  return NextResponse.next();
}

```


### サーバーサイドでのリダイレクト

`redirect()`関数を使用します。
サーバーコンポーネントか、API Routeで使用できます。
この関数は、後述する`'use server'`ディレクティブを付与したServer Action関数をインポートすれば、クライアントコンポーネントでも使用することができます。

```tsx:server.tsx
import { fetchData } from "@/repositories/eventRepository";
import { redirect } from 'next/navigation';
import { ChildComponent } from '@/components/ChildComponent';

export const Component = () => {
    const data = await fetchData();
    if(!data) {
        redirect('/event');
    }

    return (
        <ChildComponent data={data} />
    );
};
```

```tsx:route.ts
import { redirect } from 'next/navigation';
import { NextRequest,NextResponse } from 'next/server';
export const GET = (request:NextRequest) => {
    const searchParams = request.nextUrl.searchParams
    const requestID = searchParams.get('id');
    const data = fetchData(requestID);
    if(!data) {
        redirect('/event');
    }
    return NextResponse.json({...data})
};
```

サーバーコンポーネントで認証ガードとしてユーザーがログインしていない場合の早期リターン代わりに使用したり、API Routeでリクエストパラメータに応じてリダイレクトする場合に使用します。

### クライアントサイドでのリダイレクト

ボタンクリックなどの副作用を伴う処理内でリダイレクトしたい場合は、`useRouter()`を使用します。
`'use client'`ディレクティブを付与したクライアントコンポーネントで使用できます。

```tsx:client.tsx
'use client';
import { useRouter } from 'next/navigation';
import { FC, useCallback } from 'react';

export const Component:FC<{condition:boolean}> = ({condition}) => {
    const router = useRouter();

    const handleClick = useCallback(()=>{
        if(condition) {
            router.push('/event');
        } 
    },[])

    return (
        <button onClick={handleClick}>イベントページへ</button>
    );
};
```

前述したサーバーアクション関数をインポートしてクライアントコンポーネントで使用することもできます。
サーバー側処理が絡む場合はこちらのほうが使いやすいでしょう。

```tsx:actions.ts
import { redirect } from 'next/navigation';

export const handleClick = () => {
    redirect('/event');
};
```

```tsx:client.tsx
'use client';
import { handleClick } from './actions';

export const Component = () => {
    return (
        <button onClick={handleClick}>イベントページへ</button>
    );
};
```