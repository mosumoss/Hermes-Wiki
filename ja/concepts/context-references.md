---
title: Context References（@ 引用システム）
created: 2026-04-10
updated: 2026-04-10
type: concept
tags: [context, references, input, architecture]
sources: [agent/context_references.py, cli.py]
translation: ja
original: ../../concepts/context-references.md
---

# Context References（@ 引用システム）

## 概要

Hermes はユーザー入力の中で `@` プレフィックスを使って外部コンテンツを参照できます。LLM に送信する前に、システムが自動的に実際の内容に展開してメッセージに注入します。

## サポートされる参照タイプ

| 構文 | 動作 | 例 |
|------|------|------|
| `@file:パス` | ファイル内容を注入 | `@file:src/main.py` |
| `@file:パス:行番号` | ファイルの指定行を注入 | `@file:main.py:10-50` |
| `@folder:パス` | ディレクトリ構造を注入 | `@folder:src/` |
| `@diff` | 現在の git diff を注入 | `@diff で問題を見て` |
| `@staged` | git staged 変更を注入 | `@staged のコードを確認` |
| `@url:アドレス` | Web ページ内容を取得して注入 | `@url:https://example.com` |
| `@git:参照` | git オブジェクト内容を注入 | `@git:HEAD~1` |

## 処理フロー

```text
ユーザー入力: "@file:main.py と @diff の問題を見て"
    ↓
parse_context_references() — 正規表現で全 @ 参照をマッチ
    ↓
_expand_reference() — それぞれ実際の内容に展開
    ↓
セキュリティチェック:
  - パスは cwd または allowed_root 内である必要（パスエスケープ防止）
  - 機密ファイル拒否（.ssh/*, .env, .netrc など）
  - 注入総量がコンテキストウィンドウの 50% を超えない（ハード制限）、25% 超で警告
    ↓
メッセージ末尾の "--- Attached Context ---" ブロックに注入
    ↓
LLM に送信（@ 参照マーカーは原文から除去）
```

## セキュリティ機構

**機密ファイルインターセプト**：以下のパスは注入を拒否：
- `~/.ssh/*`（鍵、config）
- `~/.bashrc`, `~/.zshrc`, `~/.profile`（shell 設定）
- `~/.netrc`, `~/.pgpass`, `~/.npmrc`, `~/.pypirc`（認証情報ファイル）
- `skills/.hub/`（スキルリポジトリ内部ファイル）

**注入量制限**：
- ハード制限：注入内容がモデルコンテキストウィンドウの **50%** を超えない
- ソフト制限：**25%** 超過時に警告を表示
- ハード制限を超えた場合は参照操作全体を拒否（`blocked=True`）

**パスセキュリティ**：参照パスは絶対パスに解決された後、`cwd` または `allowed_root` の範囲内でなければなりません。`@file:../../etc/passwd` のようなパストラバーサル攻撃を防止。

## Context Files との違い

| | Context References（@ 引用） | Context Files（AGENTS.md など） |
|---|---|---|
| 発火方法 | ユーザーが入力で能動的に `@` を書く | システムが自動ロード |
| 注入位置 | ユーザーメッセージ末尾 | system prompt |
| 内容ソース | ファイル / diff / URL / git | 固定ファイル名 |
| ライフサイクル | 単一ターン | セッション全体 |

## 関連ページ

- [[prompt-builder-architecture]] — Context Files（AGENTS.md など）のロード機構
- [[security-defense-system]] — セキュリティチェック体系

## 主要ソース

| ファイル | 責任 |
|------|------|
| `agent/context_references.py` | 参照解析、展開、セキュリティチェック |
| `cli.py` | `preprocess_context_references()` の呼び出し入口 |
