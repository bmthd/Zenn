---
# title: "【VSCode】名前が被りがちなファイルをディレクトリ名表示するタブ設定"
title: "【index.ts】そのVSCodeタブ名、わかりづらくない？【page.tsx】"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [vscode,typescript,javascript,react,nextjs]
published: true
---

## はじめに

VSCodeで、index.tsやpage.tsxなど、同名のファイルを複数同時に開いてしまい、わからなくなってしまったことはありませんか？
![alt text](/images/vscode-tab-display-name-alias/image1.png)

実はよく見ると右側にディレクトリ名が表示されているのですが、薄い文字かつ、目線移動のコストが発生するため分かりづらいです。
この記事では、VSCodeの設定を変更することで、ファイル名にディレクトリ名を含めて表示する方法を紹介します。

## 設定方法

VSCodeの設定ファイル`settings.json`に以下の設定を追加します。

```json
{
      "workbench.editor.customLabels.patterns": {
      "**/{index,page,layout,template,route,actions,hooks,components,utils,types}.{js,ts,jsx,tsx,md,mdx}": "${dirname}/${filename}.${extname}",
    }
}
```

globパターンで、対象のファイル名と拡張子を指定します。
この設定例では、Next.jsのプロジェクトでよくあるファイル名を対象にしています。
2階層以上の表示としたい場合など、より複雑な設定も可能です。
`${dirname(N)}`で、N番目のディレクトリ名を取得できるので、`path/to/index.js`と表示したい場合は`${dirname(1)}/${dirname}/${filename}.${extname}`と指定します。
書き方次第で、「index.jsは、ディレクトリ名のみ表示する」など、さまざまな設定が可能です。
より詳細な記法についてはsettings.jsonでプロパティ名にホバーすると表示されるドキュメントを参照してください。

![alt text](/images/vscode-tab-display-name-alias/image2.png)
これで同名になりがちなファイルは、ディレクトリ名を含めて表示されるようになりました。

## おわりに

フロントエンドをはじめ、最近はコロケーション / package by featureの概念が一般にも理解されつつあり、責務に応じたディレクトリ構成が受け入れられやすくなってきています。
ですが、このファイル名被りが原因で、旧来型のlayerベースのディレクトリ構成に慣れたメンバーからの理解を得るのが難しくなることがあります。
実際私もメンバーからそのような指摘を受けたことがあり、自分でも不便な点だと感じていたため、解決方法を探していました。
VSCode拡張機能でそのようなものが無いのかをずっと探していたのですが、まさか標準機能で解決できるとは思いませんでした。
どうやら2024年4月5日リリースのver.1.88からの新機能のようでした。
ググラビリティが低い問題のため、この記事が一助になれば幸いです。

検索用キーワード:
vscode, tab, display, name, alias, directory, path, filename, extension, settings.json
ディレクトリ名 ファイル名 フォルダ名 パス タブ 表示 名前 エイリアス 別名 設定 VSCode拡張機能
