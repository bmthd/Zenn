---
title: "どうせエクスポートするだけなら変数に代入しなくて良くない？"
emoji: "🧊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [nextjs,react,typescript,javascript]
published: true
---

## Next.jsのPageコンポーネント

Next.jsのよくあるPageコンポーネント。

```tsx
const Page: FC = () => {
    return <div>Page</div>
}

export default Page;

// または
export default function Page() {
    return <div>Page</div>
}
```

これで良くないですか？

```tsx:page.tsx
export default (() => {
    return <div>Page</div>
}) satisfies FC
```

```tsx:layout.tsx
export default (({children}) => {
    return <div>{children}</div>
}) satisfies FC<{ children: ReactNode }>
```

- 変数名を考えなくて良い！
- exportを先に書くので、export忘れを防げる！
- 関数宣言と違い、関数コンポーネントであることを型で保証できる！
- なんかかっこいい！

## 設定ファイルでありがちなObject

```ts
const config:Config = {
    key: 'value'
}

export default config;
```

これで良くないですか？

```ts
export default {
    key: 'value'
} satisfies Config
```

## まとめ

不要な変数宣言は撲滅しよう。