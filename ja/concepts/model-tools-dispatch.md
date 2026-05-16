---
title: Model Tools ツール編成とディスパッチアーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [architecture, module, component, tool, toolset]
sources: [model_tools.py, tools/registry.py, toolsets.py]
translation: ja
original: ../../concepts/model-tools-dispatch.md
---

# Model Tools — ツール編成とディスパッチアーキテクチャ

## 概要

Model Tools は `model_tools.py`（22KB / 577 行）に実装され、Tool Registry の上に位置する**軽量編成層**です。ツール発見のトリガー、ツールセットフィルタリング、非同期ブリッジ、関数呼び出しディスパッチを担います。

コア理念：**model_tools.py はもう自前のデータ構造を持たない — すべてのデータは Tool Registry から取得。**

## アーキテクチャ原理

### ファイル依存チェーン

```
tools/registry.py  (外部依存ゼロ — 全ツールファイルからインポートされる)
       ↑
tools/*.py  (各ファイルがモジュールレベルで registry.register() を呼ぶ)
       ↑
model_tools.py  (registry をインポート + _discover_tools() をトリガー)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### 公開 API（後方互換）

```python
# これらの API シグネチャは元の 2400 行版から維持、下流コードが直接使用
get_tool_definitions(enabled, disabled, quiet) → list
handle_function_call(name, args, task_id, user_task) → str
TOOL_TO_TOOLSET_MAP: dict          # batch_runner.py で使用
TOOLSET_REQUIREMENTS: dict         # cli.py, doctor.py で使用
get_all_tool_names() → list
get_available_toolsets() → dict
check_tool_availability(quiet) → tuple
```

## コアコンポーネント

### 1. 非同期ブリッジ（_run_async）

これは model_tools.py で最も重要な基盤 — **同期から非同期への変換の単一真実源**：

```python
def _run_async(coro):
    """
    3 つの実行パス:
    
    1. 既に実行中のイベントループがある場合 (gateway / RL env)
       → 独立スレッドを起動 + asyncio.run()、衝突回避
    
    2. ワーカースレッド (delegate_task の ThreadPoolExecutor)
       → スレッドレベルの永続ループを使用 (_get_worker_loop)
       → メインスレッドとループを共有しない、かつ GC でループが閉じられない
    
    3. メインスレッド (CLI 通常パス)
       → グローバル永続ループを使用 (_get_tool_loop)
       → キャッシュされた httpx/AsyncOpenAI クライアントは活発なループに紐付く
    """
```

**なぜ asyncio.run() を使わないか**：`asyncio.run()` はループを作成、コルーチン実行、その後**ループを閉じます**。しかしキャッシュされた httpx / AsyncOpenAI クライアントは閉じられたループに紐付いたまま、GC 時に `RuntimeError: Event loop is closed` をトリガー。

### 2. ツール発見（_discover_tools）

```python
def _discover_tools():
    """全ツールモジュールをインポート、それらの registry.register() 呼び出しをトリガー"""
    _modules = [
        "tools.web_tools",
        "tools.terminal_tool",
        "tools.file_tools",
        "tools.browser_tool",
        "tools.code_execution_tool",
        "tools.delegate_tool",
        # ... 20 ツールモジュール
    ]
    for mod_name in _modules:
        try:
            importlib.import_module(mod_name)
        except Exception:
            pass  # オプションツールのインポート失敗は他に影響しない

# 注：MCP ツール発見は _discover_tools() のモジュールリストにはない、
# _discover_tools() の外側で別途処理（約 lines 173-177）:
#   from tools.mcp_tool import discover_mcp_tools
#   discover_mcp_tools()
```

**3 層発見機構**：
1. **静的インポート**：`_discover_tools()` が事前定義モジュールリストをインポート
2. **MCP 発見**：外部 MCP サーバーから動的にツール発見
3. **プラグイン発見**：ユーザー / プロジェクト / pip プラグインからツール発見

### 3. ツール定義取得（get_tool_definitions）

```python
def get_tool_definitions(enabled_toolsets, disabled_toolsets, quiet_mode):
    """
    1. toolset フィルタに従い、含めるツール名を決定
    2. registry に schema を要求（check_fn を通過したツールのみ返す）
    3. execute_code schema を動的調整（利用可能なサンドボックスツールのみリスト）
    4. browser_navigate 説明を動的調整（Web ツール利用不可時、参照を削除）
    5. _last_resolved_tool_names を記録、downstream で使用
    """
```

**重要設計 — 動的 schema 調整**：

```python
# 問題: execute_code の schema に全ての可能なサンドボックスツールがリストされる
# しかし web_search の API key が設定されていないと、モデルは「web_search 利用可能」を見て
# 存在しないツールを呼ぼうとする → hallucination

# 解決: 実際に利用可能なツールに基づいて schema を再構築
if "execute_code" in available_tool_names:
    sandbox_enabled = SANDBOX_ALLOWED_TOOLS & available_tool_names
    dynamic_schema = build_execute_code_schema(sandbox_enabled)
```

同じパターンを browser_navigate にも適用：

```python
# web_search / web_extract が利用不可な時、browser_navigate の説明から
# "prefer web_search or web_extract" 参照を削除
if not {"web_search", "web_extract"} & available_tool_names:
    desc = desc.replace("For simple information retrieval, prefer web_search...", "")
```

### 4. パラメータ型強制（coerce_tool_args）

```python
def coerce_tool_args(tool_name, args):
    """
    LLM は以下をよく返す:
    - 数字を文字列として: "42" の代わりに 42
    - bool を文字列として: "true" の代わりに true
    
    JSON Schema と照合して型を安全に変換
    """
    # サポート: integer, number, boolean, ユニオン型 [integer, string]
    # 安全: 変換失敗時は元の値を保持
```

### 5. 関数呼び出しディスパッチ（handle_function_call）

```python
def handle_function_call(function_name, function_args, task_id, ...):
    """
    1. パラメータ型強制 (coerce_tool_args)
    2. read-loop tracker に通知 (read_file 連続カウンタリセット)
    3. Agent レベルツールをインターセプト (todo, memory, session_search, delegate_task)
    4. pre_tool_call プラグインフックをトリガー
    5. registry.dispatch() にディスパッチ
    6. post_tool_call プラグインフックをトリガー
    """
```

**Agent レベルツールのインターセプト**：

```python
_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}

if function_name in _AGENT_LOOP_TOOLS:
    return json.dumps({"error": f"{function_name} must be handled by the agent loop"})
```

これらのツールは Agent レベルの状態（TodoStore、MemoryStore 等）を必要とするため、`run_agent.py` で直接処理します。

### 6. 後方互換マッピング

```python
_LEGACY_TOOLSET_MAP = {
    "web_tools": ["web_search", "web_extract"],
    "terminal_tools": ["terminal"],
    "browser_tools": ["browser_navigate", "browser_snapshot", ...],
    "rl_tools": ["rl_list_environments", "rl_select_environment", ...],
    # ...
}
```

旧ツールセット名（例：`"web_tools"`）を自動的に新ツール名リストにマッピング。

## 設計の優位性

### 2400 行から 577 行へ

| 指標 | リファクタ前 | リファクタ後 |
|---|---|---|
| コード行数 | 2,400+ | 577 |
| データ構造 | 複数 dict の並行管理 | 全て Registry に委譲 |
| 可用性チェック | model_tools.py に分散 | Registry が集中処理 |
| 非同期ブリッジ | 複数箇所でコピー | 単一の _run_async() |
| テスト難度 | mock 困難 | Registry 置換可能 |

### 動的 Schema 調整の優位性

従来の静的 schema はモデルに利用不可ツールの参照を見せてしまいます。Model Tools は**ランタイム動的調整**で以下を保証：
- execute_code は現在利用可能なサンドボックスツールのみリスト
- browser_navigate は Web ツール利用可能時のみそれらの優先使用を提案
- 全ての cross-tool 参照は実際の利用可能状態に基づく

## 他システムとの関係

- [[tool-registry-architecture]] — 全データは Registry から
- [[toolsets-system]] — toolsets.py 経由でツールセット解決と検証
- [[large-tool-result-handling]] — execute_code schema の動的調整
- [[mcp-and-plugins]] — MCP とプラグインツールは discover 機構で統合
