---
title: "【App Router】Server Actionsが実行できる汎用モーダル作ってみた"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react,nextjs,approuter,serveractions]
published: false
---



```typescript:
import { useCallback, type ReactNode } from "react";

import { useDialog } from "@/ui/dialog";
import { SubmitButton } from "@/ui/form/SubmitButton";

type useFormActionModalResult = {
  /** モーダルを開く関数 */
  handleOpen: () => void;
  /** モーダルを表示するコンポーネント */
  FormActionModal: ({ children }: { children: ReactNode }) => JSX.Element;
};

/**
 * 何らかの動作をするかどうかの確認用モーダルを表示するためのフック
 * @param action 実行ボタンを押した時に実行する関数
 * @param actionText 実行ボタンのテキスト
 * @returns handleOpen,  FormActionModal
 */
export const useFormActionModal = (
  action: (formData: FormData) => void | Promise<void>,
  actionText: string = "実行する",
): useFormActionModalResult => {
  const { handleOpen, handleClose, Dialog } = useDialog();

  const onAction = useCallback(
    async (f: FormData) => {
      await action(f);
      handleClose();
    },
    [action, handleClose],
  );

  const FormActionModal = useCallback(
    ({ children }: { children: ReactNode }) => (
      <Dialog>
        <div className="modal-box mx-auto">
          <button
            className="btn btn-circle btn-ghost btn-md absolute right-2 top-2"
            onClick={handleClose}
          >
            ✕
          </button>
          <form className="flex flex-col gap-4" action={onAction}>
            {children}
            <SubmitButton className="btn-error">{actionText}</SubmitButton>
          </form>
        </div>
      </Dialog>
    ),
    [Dialog, actionText, handleClose, onAction],
  );

  return { handleOpen, FormActionModal };
};
```