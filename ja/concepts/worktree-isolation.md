---
title: Git Worktree 隔離
created: 2026-04-10
updated: 2026-04-10
type: concept
tags: [git, worktree, isolation, parallel]
sources: [cli.py, hermes_cli/main.py, cli-config.yaml.example]
translation: ja
original: ../../concepts/worktree-isolation.md
---

# Git Worktree 隔離

## 概要

Hermes は git worktree を使って**複数の agent が同じリポジトリで並列に操作しても衝突しない**ようにサポートしています。各 agent セッションは独立した worktree ブランチで作業し、ファイル変更は互いに影響しません。

## 使い方

```bash
hermes -w              # 起動時に隔離 worktree を作成
hermes --worktree      # 同上
```

または config.yaml でグローバル有効化：
```yaml
worktree: true         # git リポジトリで起動するたびに自動で worktree を作成
```

## 動作原理

```text
hermes -w
    ↓
_setup_worktree()
    ↓
1. 現在のディレクトリが git リポジトリ内か検出（外なら エラー）
2. .worktrees/ 下に新 worktree を作成（git worktree add）
3. ブランチ hermes/hermes-{8桁ランダムID} を作成、HEAD ベース
4. .worktrees/ を自動的に .gitignore に追加
5. .worktreeinclude にリストされたファイルをコピー（gitignored だが agent が必要）
6. CWD を worktree ディレクトリに切り替え
    ↓
agent が隔離環境で作業
    ↓
セッション終了 → _cleanup_worktree()
    ↓
worktree ディレクトリ + ブランチを削除（git worktree remove + git branch -D）
```

## .worktreeinclude ファイル

一部のファイルは .gitignore で無視されるが agent が必要なものがある（例：`.env`、`node_modules`）。プロジェクトルートに `.worktreeinclude` を作成：

```text
# 1 行 1 パス、ファイルとディレクトリ両方対応
.env
node_modules
```

- ファイル：`shutil.copy2` でコピー
- ディレクトリ：symlink を作成（ディスク容量節約）
- パストラバーサル攻撃防止：ソースパスとターゲットパスは両方とも各ルートディレクトリ内にある必要

## 適用シーン

- 複数の agent が同じリポジトリの異なる部分を同時修正
- メインブランチを実験的修正で汚染しないように保護
- マルチ Profile と組み合わせ（異なる Profile + 異なる worktree = 完全隔離の並列開発）

## 関連ページ

- [[configuration-and-profiles]] — マルチ Profile アーキテクチャ
- [[multi-agent-architecture]] — マルチ Agent 連携

## 主要ソース

- `cli.py` — `_setup_worktree()` / `_cleanup_worktree()`
- `hermes_cli/main.py` — `-w` / `--worktree` パラメータ解析
