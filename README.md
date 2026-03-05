# git worktree 使い方ガイド

## 1. 現在のディレクトリで未コミットがない状態にする
```sh
git status
```
`nothing to commit, working tree clean`ならOK

## 2. mainを最新にする
```sh
git switch main
git pull --ff-only
```
`master`ブランチがデフォルトの場合は`main`を`master`に変える

## 3. worktreeを作成する
```sh
git worktree add -b <ブランチ名> <作成するディレクトリ>
```
例:
```bash
git worktree add -b feature/worktree-a ../__wt_worktree_a
```

## 4. 作ったworktreeに移動して確認
```sh
cd __wt_worktree_a
git status
git branch --show-current
```
※`git branch`を実行すると2つのブランチが有効になっているのがわかる

## 5. 変更を加えてcommitする
例えばREADME.mdを追加した後
```sh
git add README.md
git commit -m "add readme"
```
## 6. 元のファイルに影響していないことを確認
元の作業フォルダに戻って
```sh
git status
```
ここで変更が出ていないのがポイント

## 7. worktree一覧を見る
```sh
git worktree lsit
```

## 8. マージ/リモートリポジトリにpush
この操作は通常通りでOK
```sh
git push -u origin HEAD
```

## 9. wroktreeを消す
削除は元のフォルダで行うのが安全
```sh
git worktree remove ../__wt_worktree-a
git worktree prune
```
※git worktree prune は、**もう存在しない（または参照できない）worktreeの情報を、リポジトリ内部から掃除するコマンド**

## 10. ClaudeCodeを並行稼働させる
メインのフォルダに`./CLAUDE.md`を配置する

worktreeを作成し作業開始