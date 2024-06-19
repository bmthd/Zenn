---
title: "【React】表形式フォームで上下左右にフォーカスを移動させる方法"
emoji: "📅"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [React,TypeScript]
published: true
---
行と列の数が変動する表形式のフォームでExcelのようにEnterキー、Shift+Enterキーで上下移動できるようにするカスタムフックを作成しました。
制作していたWebアプリのなかで、表形式のフォームを扱うところがありExcelのような動きを期待されるUIだったので実装方法を考えました。
素の状態ではTabキーとShift+Tabキーでの横移動はできますが、Enterキーで次の行の金額を打ちに行けないのは不便です。
そのついでに矢印キーでの移動と、最初の行か最後の行の場合にもそれぞれ繋げられるように対応しました。

https://qiita.com/chelproc/items/de83a6f2959490109b49

こちらの記事内のReact-hook-formのregisterを模した方法がとてもしっくり来たので参考にさせていただきました。

:::message
動的な行,列数の増減は状態管理ライブラリRecoilで現在の行,列数を取得することで実現しています。
:::

```typescript:useFocusControl.ts
import { campaignsSelector } from '@/states/atom';
import { itemRowsSelector } from '@/states/selector';
import { useCallback, useRef, type KeyboardEventHandler, type RefCallback } from "react";
import { useRecoilValue } from 'recoil';

type FocusElementController = {
  ref: RefCallback<HTMLElement>;
  onKeyDown: KeyboardEventHandler;
};

export const useFocusControl = (): ((
  rowIndex: number,
  columnIndex: number,
) => FocusElementController) => {
  const ref = useRef<Record<string, HTMLElement>>({});
  const campaigns = useRecoilValue(campaignsSelector);
  const itemRows = useRecoilValue(itemRowsSelector);
  const columns = campaigns.length + 3; //現在の列の数を割り当てる
  const rows = itemRows.length; //現在の行の数を割り当てる

  return (rowIndex, columnIndex) => {
    const key = `${rowIndex}-${columnIndex}`;
    ref.current[key] = ref.current[key] || null;

    /**
     * Key押下時の処理を決める関数
     * @param event
     */
    const handleKeyDown: KeyboardEventHandler = (event) => {
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
    const setRef: RefCallback<HTMLElement> = (node) => {
      if (!node) return;
      ref.current[cellIndex] = node;
    };

    const focusInput = (rowIndex: number, columnIndex: number) => {
        ref.current[`${rowIndex}-${columnIndex}`]?.focus();
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
import { ItemRow } from '@/types';
import { useItemRow } from '@/hooks/useItemRow';
import { type useFocusControl } from "@/app/_home/table/body/hooks";

type Props = {
 itemRow: ItemRow & { id: number };
 register: ReturnType<typeof useFocusControl>;
}

export const TableRow: FC<Props> = ({ itemRow, register }) => {
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

useFocusControlを使用したコンポーネントに、useRefでその子コンポーネント達の参照が保持されます。
子コンポーネント側は、親から渡してもらったregister関数を使ってref callback関数経由で親のrefに自身の参照を登録します。
register関数にはイベントハンドラも含まれており、自分自身がどの行、列にいるかをもとに次のフォーカス先を決定し、refから探してフォーカスを移動させます。

下記リンク先に幅1024px以上の端末でアクセスすると表形式フォームを体験できます。

https://point-sprint.bmth.dev/