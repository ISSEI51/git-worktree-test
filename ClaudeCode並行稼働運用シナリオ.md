# worktree_parallel_ops.md
# worktree + ClaudeCode 並行稼働 運用シナリオ（GitHub / 直接編集前提）

> 目的：複数のClaudeCodeを同時に走らせても、変更が混ざらず、PR単位で安全に統合できる運用を確立する。  
> 前提：**パッチ出力→git apply は使わない**。ClaudeCodeは **自分のworktree内で直接ファイル編集**し、commit/push/PRまで行う。

---

## 1. 役割設計（3並列の鉄板）

- **GPT（司令塔）**
  - 要件確定 / タスク分割 / 衝突回避設計 / 統合順序の判断
  - 各Claudeに貼る「唯一の指示文」を作る
  - 状況スナップショット（git status / diff / テスト結果）を読み、次の一手を決める

- **Claude A（実装1）**
  - Feature A を担当
  - 自分のworktree内で直接編集 → commit → push → PR作成

- **Claude B（実装2）**
  - Feature B を担当
  - 自分のworktree内で直接編集 → commit → push → PR作成

- **Claude C（QC/デバッグ）**
  - テスト追加 / 再現手順作成 / エッジケース確認 / 不具合発見
  - 必要なら最小修正を自分のworktreeで実施しPR化

> 推奨：バック/フロント分割より、まずは **Feature縦割り + QC** が最も安定。

---

## 2. 大原則（並列が崩壊しない条件）

1. **1 worktree = 1 branch = 1 ClaudeCode**
2. 各Claudeは **自分のworktree配下しか触らない**（`..`禁止、兄弟worktree禁止）
3. 統合（mainへのマージ）は **統合用worktree（repo/）で1本ずつ**
4. “横断変更”は最初から **2フェーズ**で分割（例：API契約→実装→接続）

---

## 3. worktree構成（例）

統合用が `repo/`（main）。並列worktreeは兄弟に作成：

- `repo/`（統合用：main）
- `../repo__wt_featA`（Claude A）
- `../repo__wt_featB`（Claude B）
- `../repo__wt_qc`（Claude C）

### 作成コマンド例
```bash
cd repo
git fetch --all --prune
git switch main && git pull --ff-only

git worktree add -b feature/featA ../repo__wt_featA main
git worktree add -b feature/featB ../repo__wt_featB main
git worktree add -b test/qc       ../repo__wt_qc    main
```

---

## 4. 進行フロー（毎回この型で回す）

### Phase 0：司令塔が「契約と境界」を固定（最初にやる）
GPT（司令塔）が以下を決める：
- 目的 / 受け入れ条件（AC）
- 変更してよい範囲、触ってはいけない範囲（ディレクトリ/ファイル）
- タスク分割（featA / featB / QC）
- 統合順（衝突しそうなものから順に）

### Phase 1：並列実装（Claude A/B）＋ QC（Claude C）
各Claudeは自分のworktreeで：
1. **直接ファイル編集**（パッチ出力は禁止）
2. 小さくcommit
3. `git push -u origin HEAD`
4. PR作成（PR本文に「何を変えた / どう確認した」を記載）

### Phase 2：統合（司令塔の順序で1本ずつ）
- 統合用 `repo/` で、PRを **1本ずつ**マージ
- マージ後に `repo/` で `git pull --ff-only` して状態を揃える

### Phase 3：片付け
- マージ済みworktreeは削除：
```bash
cd repo
git worktree remove ../repo__wt_featA
git worktree remove ../repo__wt_featB
git worktree remove ../repo__wt_qc
git worktree prune
```

---

## 5. 司令塔へ「状況を正確に伝える」プロトコル

### 5.1 統合用 repo/ のスナップショット（毎ターン）
```bash
git status -sb
git worktree list
git log --oneline -5
```

### 5.2 各worktree（A/B/QC）のスナップショット（毎ターン）
```bash
git status -sb
git log --oneline -5
git diff --stat
```

可能なら、担当領域のテスト/コマンドも添付（例）：
- Node: `npm test`, `npm run lint`, `npm run typecheck`
- Python: `pytest`, `ruff`, `mypy`

> ポイント：パッチ運用をしない代わりに、**差分は `git diff --stat` とテスト結果**で司令塔が把握する。

---

## 6. 衝突回避のコツ（重要）

### 6.1 推奨：Feature縦割り（最も事故が少ない）
- 1つの機能単位で、UI/API/DBが絡んでも「その機能だけ」に閉じる

### 6.2 バック/フロント分割は「API契約固定」後に
- API/DTO/型が揺れている状態で分割すると意図衝突が起きやすい
- 先に `docs/api.md` 等で契約を固定し、その後並列化

### 6.3 「コンフリクトが出ない」事故に注意
- ファイル競合が出なくても、仕様のズレで壊れることがある（API変更 vs 呼び出し側など）
- 最後はテスト・型・lint・差分レビューで担保する

---

## 7. 並列を回す “小さめ練習” シナリオ（おすすめ）

まずは衝突が起きにくい小タスクで1周回す：
- **Claude A**：小機能追加（ログ/設定追加/軽微なUI改善）
- **Claude B**：別の小改善（バリデーション/エラーメッセージ改善）
- **Claude QC**：テスト1本追加＋エッジケース確認

この3本で「PR→統合→worktree削除」まで回せれば、運用は実務レベルで安定する。

---

## 8. 完了条件（運用としてのDone）

- [ ] worktreeを3本用意し、並列でPRが作れる
- [ ] 司令塔にスナップショットを渡し、統合順に従って1本ずつマージできる
- [ ] マージ後にworktreeを安全に削除し、`git worktree prune` で掃除できる
- [ ] 次の並列ラウンドへ移行できる（ループ可能）

---
