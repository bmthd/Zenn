---
title: "Reactのレスポンシブ表示にコードは使わない方が良い"
emoji: "📱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['React','TailwindCSS']
published: true
---
# 概要

作成しているWebアプリにおいて、スマホ表示とPC表示でコンポーネントを出し分けたかった。
始めはreact-responsiveのuseMediaQueryを使用し下記のように書いていた。

```typescript:index.tsx
export default function Home() {
	const isMobile = useMediaQuery('(max-width: 1024px)');
	const setIsMobile = useSetRecoilState(isMobileState);
	useEffect(() => {
		setIsMobile(isMobile);
	}, [isMobile]);
	return (
		<>
            {isMobile ? (
                <MobileComponent />
            ) : (
                <PCComponent />
            )}
        </>
	);
}  
```
状態管理にisMobileという状態を保管し、画面幅に変更があったときにuseEffectで変更を状態に書き込む。
JSX部ではisMobileの値に応じてコンポーネントを出し分けるようにしていた。
ところが、この方法だと読み込み後に一瞬別のレイアウトが表示されてしまう問題が発生した。
モバイル表示の状態で再読み込みをすると、一瞬PCで表示されたあとにモバイル表示になる。
ちらつきがストレスであまり気持ちの良いものではない。
読み込みが終わるまで画面を表示しないようにしたり、読み込み中表示をするというのも考えたが、パフォーマンス面やユーザーの離脱率にも関わってくるのでできれば避けたい。

# 解決策
このように書き換えた。

```typescript:index.tsx
export default function Home() {
    const isMobile = useMediaQuery('(max-width: 1024px)');
	const setIsMobile = useSetRecoilState(isMobileState);
	useEffect(() => {
		setIsMobile(isMobile);
	}, [isMobile]);
    return (
        <>
            <MobileComponent className="lg:hidden"/>
            <PCComponent className="max-lg:hidden"/>
        </>
	);
}  
```
原因はuseEffectが画面の描画後に動作するため、それまでの間isMobileがfalseになっていたからだった。
Tailwind CSSのmediaQueryを使って画面幅をブレークポイントにdisplay:noneを付与するようにした。
シンプルで良かったんだ…。

# useMediaQueryの使い道は？
画面幅の状態に応じてボタンを押したときの動きを変えたい内容があったので、このアプリではuseMediaQueryを残している。
表示だけの問題であればCSSを使った出し分けの部分だけで問題ない。
useMediaQueryはあくまでも画面幅の状態をアプリのロジック部に伝えるためだけに使用するべきであって、レイアウトの変更には使わないほうが良さそう。

# 参考
https://www.ultra-noob.com/blog/2022/63/