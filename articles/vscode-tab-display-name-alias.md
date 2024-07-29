---
# title: "【VSCode】名前が被りがちなファイルをディレクトリ名表示するタブ設定"
title: "【index.ts】そのVSCodeタブ名、わかりづらくない？【page.tsx】"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vscode,typescript,javascript,react,nextjs]
published: true
---

## はじめに

VSCodeで、`index.ts`や`page.tsx`など、同名のファイルを複数同時に開いてしまい、わからなくなってしまったことはありませんか？
![alt text](/images/vscode-tab-display-name-alias/image1.png)

実はよく見ると同一名称のファイルを開いているときには右側にディレクトリ名も表示されているのですが、薄い文字かつ、目線移動が必要で分かりづらいです。
この記事では、VSCodeの設定を変更することで、タブ表示名にディレクトリ名を含めて表示する方法を紹介します。

## 設定方法

VSCodeの設定ファイル`settings.json`に以下の設定を追加します。

```json
  "workbench.editor.customLabels.patterns": {
      "**/index.*": "${dirname} .../${dirname(1)}",
      "**/{page,layout,template,route,actions,hooks,components,utils,types}.{js,ts,jsx,tsx}": "${dirname}/${filename}.${extname} .../${dirname(1)}",
    }
```

globパターンで、対象のファイル名と拡張子を指定します。
この設定例では、Next.jsのプロジェクトでよくあるファイル名を対象にしています。

工夫しているポイントとしては、`.../${dirname(1)}`の部分です。
`${dirname(N)}`表記で、N階層上のディレクトリ名を取得できるのでそれを利用し、VSCodeのもとの設定に寄せた表示ができるようにしています。
また、index.*については特別なファイルのため、そのディレクトリ名を表示するようにしています。

![alt text](/images/vscode-tab-display-name-alias/image2.png)
これで同名になりがちなファイルは、ディレクトリ名を含めて表示されるようになりました！

絶対パスを利用して、絵文字を表示するやり方を提示している方もいました。
これは面白いアイデアですね。
https://dev.classmethod.jp/articles/vscode-custom-tab-labels-for-better-coding/
より詳細な記法についてはsettings.jsonでプロパティ名にホバーすると表示されるドキュメントを参照してください。

## おわりに

最近のフレームワークがそれ前提の設計になっていることもあり、co-location / package by featureの概念が一般にも理解され、責務に応じたディレクトリ構成が受け入れられやすくなってきています。
ですが、このファイル名被りが原因で、旧来型のlayerベースのディレクトリ構成に慣れたメンバーからの理解を得るのが難しくなることがあります。
実際私もメンバーからそのような指摘を受けたことがあり、自分でも不便な点だと感じていたため、解決方法を探していました。
VSCode拡張機能でそのようなものが無いのかをずっと探していたのですが、まさか標準機能で解決できるとは思いませんでした。
どうやら2024年4月5日リリースのver.1.88からの新機能のようでした。
ググラビリティが低い問題のため、この記事が一助になれば幸いです。

検索用キーワード:
vscode, tab, display, name, alias, directory, path, filename, extension, settings.json
ディレクトリ名 ファイル名 フォルダ名 パス タブ 表示 名前 エイリアス 別名 設定 VSCode拡張機能