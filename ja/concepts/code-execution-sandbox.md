---
title: コード実行サンドボックス（execute_code）
created: 2026-04-10
updated: 2026-04-10
type: concept
tags: [sandbox, code-execution, tools, architecture]
sources: [tools/code_execution_tool.py]
translation: ja
original: ../../concepts/code-execution-sandbox.md
---

# コード実行サンドボックス

## 概要

`execute_code` ツールは LLM が Python スクリプトを書き、それを隔離されたサブプロセスで実行できるようにします。スクリプトは RPC コールバックで限定的な Hermes ツールセットを呼び出せます。複数ステップのツールチェーンを 1 回の推論に圧縮し、token 消費とレイテンシを削減します。

## コア価値

```text
従来方式：10 ターンのツール呼び出し = 10 回の LLM 推論 + 10 回の context 膨張
execute_code：1 回の LLM がスクリプトを書く + 1 回実行、中間結果は context に入らない
```

## サンドボックス制限

### 許可されるツール（7 個のみ）

```python
SANDBOX_ALLOWED_TOOLS = [
    "web_search",      # 検索
    "web_extract",     # Web ページ抽出
    "read_file",       # ファイル読み込み
    "write_file",      # ファイル書き込み
    "search_files",    # ファイル検索
    "patch",           # ファイル修正
    "terminal",        # ターミナルコマンド
]
```

### リソース制限

```python
DEFAULT_TIMEOUT = 300         # 5 分タイムアウト
DEFAULT_MAX_TOOL_CALLS = 50   # 最大 50 回のツール呼び出し
MAX_STDOUT_BYTES = 50_000     # 出力上限 50KB
MAX_STDERR_BYTES = 10_000     # エラー出力上限 10KB
```

config.yaml の `code_execution.*` で上書き可能。

## 2 つの通信モード

| モード | 適用バックエンド | 通信方式 |
|------|---------|---------|
| **UDS（Unix Domain Socket）** | local | 親プロセスが RPC listener を起動、子プロセスは socket 経由でツール呼び出し |
| **File-based RPC** | Docker / SSH / Modal / Daytona | 子プロセスがリクエストファイルを書き込み → 親プロセスがポーリング → 応答ファイル書き込み |

### フロー

```text
1. 親プロセスが hermes_tools.py スタブを生成（RPC 関数を含む）
2. 親プロセスが RPC リスナーを起動（UDS socket またはファイルポーリングスレッド）
3. 子プロセスが LLM の書いたスクリプトを実行
4. スクリプト内で hermes_tools.web_search(...) などを呼び出す
   → RPC で親プロセスに送信 → 親プロセスが実ツールを呼び出し → 結果を返す
5. 最終 stdout のみ LLM に返す、中間結果は context に入らない
```

## Terminal Backend との関係

execute_code のスクリプトは**現在の terminal backend で実行**されます。backend が Docker なら、スクリプトは Docker で動き、file-based RPC でローカルのツールにコールバックします。

## 関連ページ

- [[terminal-backends]] — スクリプトがどのバックエンドで実行されるか
- [[large-tool-result-handling]] — ツール結果の溢れ防御

## 主要ソース

- `tools/code_execution_tool.py`（1347 行）— サンドボックスの完全実装
