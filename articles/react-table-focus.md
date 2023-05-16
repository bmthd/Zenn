---
title: "【React】表形式フォームで上下左右にフォーカスを移動させる方法"
emoji: "📅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React,TypeScript]
published: true
---
行と列の数が変動する表形式のフォームでExcelのようにEnterキー、Shift+Enterキーで上下移動できるようにするカスタムフックを作成しました。
制作していたWebアプリのなかで、表形式のフォームを扱うところがありExcelのような動きを期待されるUIだったので実装方法を考えました。
素の状態ではTabキーとShift+Tabキーでの移動はできますが、Enterキーで次の行の金額を打ちに行けないのは不便です。
そのついでに矢印キーでの移動、最初の行か最後の行の場合にもそれぞれ繋げられるように対応しました。

https://qiita.com/chelproc/items/de83a6f2959490109b49

こちらの記事のReact-hook-formのregisterを模した方法がとてもしっくり来たので参考にさせていただきました。

:::message
状態管理はRecoilで記載しています。
:::

```typescript:useFocusControl.ts
import { useRef } from 'react';
import { useRecoilValue } from 'recoil';
import { campaignsSelector } from '@/recoil/atom';
import { itemRowsSelector } from '@/recoil/selector';

type FocusControlResult = {
	onKeyDown: (event: React.KeyboardEvent<HTMLInputElement>) => void;
	ref: (element: HTMLInputElement) => void;
};

export const useFocusControl = () => {
	const ref = useRef<Record<string, HTMLInputElement>>({});
	const campaigns = useRecoilValue(campaignsSelector);
	const columns = campaigns.length; //現在の列の数を割り当てる
	const itemRows = useRecoilValue(itemRowsSelector);
	const rows = itemRows.length; //現在の行の数を割り当てる

	return (rowIndex: number, columnIndex: number): FocusControlResult => {
		const key = `${rowIndex}-${columnIndex}`;
		ref.current[key] = ref.current[key] || null;

		/**
		 * Key押下時の処理を決める関数
		 * @param event
		 */
		const handleKeyDown = (event: React.KeyboardEvent<HTMLInputElement>) => {
			const isEnterKey = event.key === 'Enter';
			const isShiftKeyPressed = event.shiftKey;
			const isArrowKeyPressed = ['ArrowLeft', 'ArrowRight', 'ArrowUp', 'ArrowDown'].includes(
				event.key,
			);
			//対象のキーでない場合は早期リターン
			if (!isEnterKey && !isArrowKeyPressed) {
				return;
			}
			//その要素本来の動作を止める
			event.preventDefault();
			//キーに応じた処理を実行する
			if (isEnterKey) {
				if (isShiftKeyPressed) {
					handleShiftEnter(rowIndex, columnIndex);
				} else {
					handleEnter(rowIndex, columnIndex);
				}
			} else if (isArrowKeyPressed) {
				handleArrow(event.key, rowIndex, columnIndex);
			}
		};

		/**
		 * refとHTMLInputElementを紐付ける関数
		 * @param element
		 */
		const setRef = (element: HTMLInputElement) => {
			ref.current[key] = element;
		};

		// 以下、フォーカスを移動させる関数
		const handleShiftEnter = (rowIndex: number, columnIndex: number) => {
			const prevRow = rowIndex - 1;
			const prevCol = prevRow < 0 ? columnIndex - 1 : columnIndex;

			if (prevRow < 0 && prevCol < 0) {
				focusLastInput();
			} else if (prevRow < 0) {
				focusInput(rows - 1, prevCol);
			} else {
				focusInput(prevRow, prevCol);
			}
		};

		const handleEnter = (rowIndex: number, columnIndex: number) => {
			const nextRow = rowIndex + 1;
			const nextCol = nextRow === rows ? columnIndex + 1 : columnIndex;
			if (nextRow === rows && nextCol === columns) {
				focusFirstInput();
			} else if (nextRow === rows) {
				focusInput(0, nextCol);
			} else {
				focusInput(nextRow, nextCol);
			}
		};

		const handleArrow = (key: string, rowIndex: number, columnIndex: number) => {
			const move = (r: number, c: number) => {
				focusInput(r < 0 ? rows - 1 : r % rows, c < 0 ? columns - 1 : c % columns);
			};

			switch (key) {
				case 'ArrowLeft':
					move(rowIndex, columnIndex - 1);
					break;
				case 'ArrowRight':
					move(rowIndex, columnIndex + 1);
					break;
				case 'ArrowUp':
					move(rowIndex - 1, columnIndex);
					break;
				case 'ArrowDown':
					move(rowIndex + 1, columnIndex);
					break;
				default:
					break;
			}
		};

		const focusFirstInput = () => {
			focusInput(0, 0);
		};

		const focusLastInput = () => {
			const lastRowIndex = rows - 1;
			const lastColumnIndex = columns - 1;
			focusInput(lastRowIndex, lastColumnIndex);
		};

		const focusInput = (rowIndex: number, columnIndex: number) => {
			const key = `${rowIndex}-${columnIndex}`;
			const input = ref.current[key];
			if (input) input.focus();
		};

		return {
			onKeyDown: handleKeyDown,
			ref: setRef,
		};
	};
};

```

使い方

```typescript:TableBody.tsx
import { useRecoilValue} from 'recoil';
import { TableRow } from './TableRow';
import { itemRowsSelector } from '@/recoil/selector';
import { useFocusControl } from '@/hooks';

export const TableBody = () => {
	const itemRows = useRecoilValue(itemRowsSelector);
	const register = useFocusControl();
	
	return (
		<>
			{itemRows.map((itemRow) => (
				<TableRow key={itemRow.id} itemRow={itemRow} register={register}/>
			))}
		</>
	);
};
```
親コンポーネント側でhookの使用を宣言し、行の配列をmapすると同時にregisterオブジェクトを渡してあげ…


```typescript:TableRow.tsx
import { ItemRow } from '@/types/Types';
import { useItemRow } from '@/hooks/useItemRow';

//registerの型定義を行う
interface Props {
	itemRow: ItemRow & { id: number };
	register: (
		rowIndex: number,
		columnIndex: number,
	) => {
		onKeyDown: (event: React.KeyboardEvent<HTMLInputElement>) => void;
		ref: (element: HTMLInputElement) => void;
	};
}

export function TableRow(props: Props) {
	const { itemRow, register } = props;
    const { updateItemRow } = useItemRow();

	return (
		<tr key={itemRow.id}>
			<td>
				<input
					type="text"
					placeholder="商品名メモ"
					value={itemRow.itemName}
					onChange={e => updateItemRow(itemRow.id, { itemName: e.target.value })}
					{...register(itemRow.id, 0)} //ここで受け渡し
				/>
			</td>
			<td>
				<input
					type="number"
					placeholder="金額を入力"
					value={itemRow.price === undefined ? '' : String(itemRow.price)}
					onChange={e => {
						const value = e.target.value.trim();
						updateItemRow(itemRow.id, { price: value ? parseInt(value) : undefined });
					}}
					{...register(itemRow.id, 1)}
				/>
			</td>
		</tr>
	);
}

```
行コンポーネントのinputタグ内で、関数の受け渡しを行います。
スプレッド構文を使いonKeyDown関数とref関数を渡しつつ、引数に行と列を受け取ります。
これで今どこの要素にいて何行目何列なのかをuseFocusControlに伝えつつinputタグのonKeyDown,refに指定する関数を受け取ることができます。