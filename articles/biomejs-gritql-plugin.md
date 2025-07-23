---
title: "Biome2.0のGritQLプラグインでDOM要素の直接利用を禁止してみた"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["biome", "gritql",  "javascript", "typescript", "claudecode"]
published: true
---

# はじめに

Claude CodeなどのAI Agentを使ってコーディングを行う際に、プロジェクト固有のUIライブラリやデザインシステムを使ってくれなかった経験はありませんか？
これらを指示文に明記しても、コンテキストが大きいと忘れてしまい、従ってくれない場合があります。
このような場合、ESLintやBiomeなどのLinterを使って、プロジェクト固有のコーディング規約を自動化することができます。
ただし、ESLintは実行速度の問題があり、AI Agentと組み合わせて使う際のiteration速度を遅くする要因となりえます。

そこで、この記事では先日v2.0がリリースされ、Beta版として提供が開始されたGritQLプラグインを使って、Biomeでプロジェクト固有のコーディング規約を自動化する方法を紹介します。

## GritQLとは

https://docs.grit.io/

GritQLはBiomeがPluginとして採用した、コードパターンを記述するためのクエリ言語です。
Biomeのプラグインシステムを使用することで、プロジェクト固有のルールを定義し、コードベース全体に適用できます。


### プラグインファイルの作成

GritQLプラグインは、`.grit`拡張子を持つファイルで定義されます。

格納先のディレクトリに制約は無いため、好きな名称で作成できます。
今回は、`biome-rules`というディレクトリを作成します。

ファイル名についても制約はありませんが、Biomeのルール名と統一し、kebab-caseが良さそうです。
今回は`no-lowercase-jsx-components.grit`としました。

```bash
biome-rules
└── no-lowercase-jsx-components.grit
```


## 実装例

コードは以下のように記述します。

```grit
language js

or {
  `<$tag />`,
  `<$tag $props />`,
  `<$tag>$body</$_>`,
  `<$tag $props>$body</$_>`
} where {
  $tag <: r"^[a-z][a-z0-9-]*$",
  register_diagnostic(
    span = $tag,
    message = "Do not use direct DOM elements. Use the project's design system instead.",
    severity = "error"
  )
}
```

↑まだシンタックスハイライトが無いのが残念です。

### コードの解説

このGritQLクエリは以下の動作をします。

1. **or句で複数のJSXパターンを定義**: 自己閉じタグとコンテンツありタグの両方に対応
2. **where句で正規表現パターン**: `r"^[a-z][a-z0-9-]*$"`で小文字で始まるHTMLタグを検出
3. **エラーメッセージ**: デザインシステムの使用を促すメッセージを表示

単体のタグ、閉じタグあり、propsの有無により認識してくれない事があったため、`or`を使って複数のパターンを定義しました。

GritQLでは、`function`を使って関数を定義可能です。
`register_diagnostic`という関数はBiomeが提供している関数で、これを使うことで、Biomeにエラー情報の登録ができます。

### Biome設定での有効化

`biome.json`の`plugins`セクションにプラグインを追加します。

```json
{
  "linter": {
    "rules": {
      ...,
    }
  },
	"plugins": ["rules/no-lowercase-jsx-components.grit"]
}
```

実際に実行してみます。

```bash
biome check ./src
```

![BiomeのCLI上でerrorが発生した行と、その内容が表示されている](/images/biomejs-gritql-plugin/image.png)

これでプラグインを有効化できました！

VSCodeを再起動した後、エディタ上でもエラーが表示されるようになります！

![html, bodyタグと同じラインにerrorメッセージが表示されている](/images/biomejs-gritql-plugin/error.png)

## ルールの一時的な無効化

特定の箇所でルールを無効化したい場合は、以下のコメントを使用します。

```jsx
// biome-ignore lint: plugin
<div>この箇所ではプラグインを無効化</div>
```

これはドキュメントに記載されていなかったので、実際に試してみて発見しました。
Biomeの標準ルールの場合は名前で無効化できますが、GritQLプラグインの場合は`plugin`を指定する必要があります。
特定のルールのみの無効化は現時点ではサポートされていないようです。

## まとめ

BiomeのGritQLプラグインの書き方を紹介しました。
他のツールと比べ、宣言的な記述が可能なところが良さそうに感じました。
まだGritQLプラグインを使っている記事がネット上に見当たらず、情報が少なかったため、Claude Codeがこの記事を見て実装できるようになってくれると良いですね。
今後もGritQLプラグインを使ったルールを追加していきたいと思います。
