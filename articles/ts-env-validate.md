---
title: "TypeScriptにおける環境変数のベストプラクティス"
emoji: "🗃️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [typescript,nextjs,nodejs,vite]
published: false
---

## はじめに

Node.jsを使ったTypeScriptプロジェクトでは、環境変数を管理するのに.envファイルを使うというのはよくあるパターンかと思います。
この環境変数を安全に扱うベストプラクティスをまとめていきます。

## 基本

- viteの場合
- Node.jsの場合
- Playwrightの場合

## サーバーサイドとクライアントサイドで使用する環境変数を分ける

## フロントエンドにおいて、使用していない環境変数をバンドルに含めない工夫

手前味噌ですが、ライブラリを作成しました。