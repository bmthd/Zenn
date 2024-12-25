---
title: "Config系ファイルに型補完が無い！？JSON Schemaのご紹介"
emoji: "⛳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## はじめに

TypeScript関連のConfigファイルでは、必ずと言っていいほどTypeScript用の型が提供されているため型補完が受けられます。
tsconfig.jsonをVSCodeで開くと何も設定を行うことなく型補完が効くことは有名です。
PlaywrightやViteなどのTypeScriptに対応しているライブラリでは、型定義ファイルが提供されているため、以下のように型補完が受けられます。

```ts
```

まだTypeScriptに対応していなくても、JSDocコメントの型指定を行うことで型補完を受ける方法もあります。

```ts
```

では、JavaScriptに依存しないツールチェーンの設定ファイルにおいてはどうでしょうか？
このようなツールはJSONやYAMLを設定ファイルとして使用することが多いため、TypeScriptの型を指定することはできません。
このように、標準でサポートされていないファイルに対して型補完を行う方法があることをご存知でしょうか？

JSON Schemaという仕組みを使うことで、JSONファイルに対しても型補完を行うことができます！

CSpell
biome
prettier
