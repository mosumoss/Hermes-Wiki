---
title: Tool Registry ツール登録システムアーキテクチャ
created: 2026-04-08
updated: 2026-04-15
type: concept
tags: [tool, toolset, tool-registry, architecture, component]
sources: [tools/registry.py, model_tools.py]
translation: ja
original: ../../concepts/tool-registry-architecture.md
---

# Tool Registry — ツール登録システムアーキテクチャ

## 概要

Tool Registry は Hermes Agent ツールシステムの**中央骨格**で、`tools/registry.py`（275 行 / 10KB）に実装されています。**宣言的なツール登録 + 集中ディスパッチ**という設計パターンを採用し、初期の `model_tools.py` に分散していた並列データ構造を置き換えています。

すべてのツールファイル（`tools/*.py`）はモジュール読み込み時に `registry.register()` で自動登録され、`model_tools.py` は登録テーブルを照会してディスカバリーをトリガーするだけです。

## アーキテクチャ原理

### インポートチェーン（循環インポート安全）

```
tools/registry.py  (外部依存ゼロ — すべてのツールファイルからインポートされる)
       ↑
tools/*.py  (各ファイルはモジュール宣言時に registry.register() を呼ぶ)
       ↑
model_tools.py  (registry をインポート + _discover_tools() をトリガー)
       ↑
run_agent.py, cli.py, batch_runner.py
```

この設計は**循環インポート問題を完全に回避**しています。registry はツールファイルをインポートせず、ツールファイルは registry のみをインポート、model_tools が registry とすべてのツールを同時にインポートする唯一のモジュールです。

### コアデータ構造

```python
class ToolEntry:
    """単一ツールのメタデータ"""
    __slots__ = (
        "name", "toolset", "schema", "handler", "check_fn",
        "requires_env", "is_async", "description", "emoji",
    )

class ToolRegistry:
    """シングルトン登録テーブル、全ツールの schema + handler を集約"""
    def __init__(self):
        self._tools: Dict[str, ToolEntry] = {}         # ツール名 → メタデータ
        self._toolset_checks: Dict[str, Callable] = {}  # toolset → チェック関数
```

**設計のポイント**：`__slots__` を使うことでメモリオーバーヘッドを削減（各 ToolEntry で約 40% メモリ節約）。100+ ツール登録時に効果が顕著です。

### 組み込みツールの自動発見（2026-04-14）

初期の `model_tools.py` はハードコードされたツールインポートリストを管理しており、新ツール追加には 2 ファイル変更が必要でした。現在は `tools/registry.py` が `discover_builtin_tools()` を提供し、`model_tools.py` が起動時に呼び出します：

```python
def discover_builtin_tools(tools_dir=None) -> List[str]:
    """tools/*.py をスキャン、自己登録するツールモジュールをすべてインポート"""
    tools_path = Path(tools_dir) or Path(__file__).resolve().parent
    module_names = [
        f"tools.{path.stem}"
        for path in sorted(tools_path.glob("*.py"))
        if path.name not in {"__init__.py", "registry.py", "mcp_tool.py"}
        and _module_registers_tools(path)  # AST チェック
    ]
    # 各モジュールを importlib.import_module() で読み込み、モジュールレベルの registry.register() をトリガー
```

**AST レベルのフィルタ**：`_module_registers_tools()` は `ast.parse` でモジュールを解析し、**モジュールトップレベル**で `registry.register(...)` の呼び出しを検出した場合にのみインポートします。これにより：
- 通常のツールファイル（`tools/terminal_tool.py` 等）は識別されてロードされる
- ヘルパーモジュール（トップレベルでツール登録しない）はスキップ
- ヘルパー関数内部の `registry.register()` 呼び出しは誤判定されない

**除外リスト**：`__init__.py`、`registry.py` 自身、`mcp_tool.py`（MCP ツールはオンデマンドで動的ロードされるため、このパスを通らない）。

**新ツール追加フローの簡略化**：以前は 3 箇所変更（ツールファイル + `model_tools.py` import + toolsets 定義）が必要でしたが、現在は 2 箇所（ツールファイル + toolsets 定義）だけで済みます。auto-discovery が新ファイルを自動的に拾います。

## コア操作

### 1. 登録（register）

各ツールファイルはインポート時に自動登録されます：

```python
# tools/terminal_tool.py 内
registry.register(
    name="terminal",
    toolset="terminal",
    schema={"name": "terminal", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: terminal_tool(...),
    check_fn=lambda: True,           # 可用性チェック
    requires_env=[],                 # 環境変数依存
    is_async=False,
)
```

- **名前衝突検出**：同名ツールが異なる toolset に属する場合、warning を出して上書き
- **check_fn キャッシュ**：各 toolset は最初の check_fn だけを記録、重複チェックを回避

### 2. 可用性チェック（get_definitions）

OpenAI 形式のツール schema リストを返します。check_fn を通過したツールのみが含まれます：

```python
def get_definitions(self, tool_names: Set[str], quiet: bool = False) -> List[dict]:
    # check_fn 結果をキャッシュ — 同じ toolset は 1 回だけチェック
    check_results: Dict[Callable, bool] = {}
    for name in sorted(tool_names):
        entry = self._tools.get(name)
        if entry.check_fn:
            if entry.check_fn not in check_results:
                check_results[entry.check_fn] = bool(entry.check_fn())
            if not check_results[entry.check_fn]:
                continue  # 利用不可ツールをスキップ
        result.append({"type": "function", "function": {**entry.schema, "name": entry.name}})
    return result
```

**利点**：
- **必要に応じてフィルタ**：環境依存を満たすツールのみが LLM に送られ、モデルが存在しないツールを呼ぶことを防ぐ
- **チェックキャッシュ**：同じ toolset の check_fn は 1 回だけ実行、各ツールごとに実行しない
- **サイレントモード**：`quiet=True` はデバッグログを抑制、バッチ照会に適する

### 3. ディスパッチ実行（dispatch）

```python
def dispatch(self, name: str, args: dict, **kwargs) -> str:
    entry = self._tools.get(name)
    if not entry:
        return json.dumps({"error": f"Unknown tool: {name}"})
    try:
        if entry.is_async:
            from model_tools import _run_async
            return _run_async(entry.handler(args, **kwargs))
        return entry.handler(args, **kwargs)
    except Exception as e:
        return json.dumps({"error": f"Tool execution failed: {type(e).__name__}: {e}"})
```

**利点**：
- **エラー形式の統一**：すべての例外がキャッチされ `{"error": "..."}` JSON で返る、LLM がパース可能
- **非同期ブリッジ**：`is_async` フラグを自動検出し `_run_async` でブリッジ、呼び出し側は意識不要
- **未知ツールのセーフフェイル**：例外を投げるのではなく JSON エラーを返す

### 4. 動的登録解除（deregister）

```python
def deregister(self, name: str) -> None:
    entry = self._tools.pop(name, None)
    # その toolset に他のツールがなくなった場合、check_fn をクリーンアップ
    if entry.toolset in self._toolset_checks and not any(
        e.toolset == entry.toolset for e in self._tools.values()
    ):
        self._toolset_checks.pop(entry.toolset, None)
```

**利用シーン**：MCP の動的ツール発見 — MCP サーバが `notifications/tools/list_changed` を送ってきたら、古いツールを nuke-and-repave して再登録する必要があります。

### 5. クエリ補助メソッド

| メソッド | 用途 |
|---|---|
| `get_all_tool_names()` | 登録済みすべてのツール名を返す（ソート済） |
| `get_schema(name)` | check_fn をバイパスして生の schema 取得、token 推定に使用 |
| `get_toolset_for_tool(name)` | ツールの所属 toolset を照会 |
| `get_emoji(name)` | ツールに対応する絵文字を取得 |
| `get_tool_to_toolset_map()` | `{tool_name: toolset_name}` マッピングを返す |
| `is_toolset_available(toolset)` | toolset が要件を満たすか確認 |
| `check_toolset_requirements()` | 全 toolset の可用性ステータスを返す |
| `get_available_toolsets()` | toolset メタデータ（ツールリスト、環境依存等）を返す |
| `check_tool_availability()` | 利用可能 / 利用不可な toolset の分類を返す |

## 設計の優位性

### 旧アーキテクチャとの比較

| 観点 | 旧方式（model_tools.py に分散） | 新方式（Tool Registry） |
|---|---|---|
| データ構造 | 複数の dict を並行管理 | 単一登録テーブル |
| 循環インポート | 起こりやすい | 依存ゼロ、インポート安全 |
| 拡張性 | ツール追加に model_tools.py の変更が必要 | ツールファイルで register() を呼ぶだけ |
| 動的発見 | 非対応 | deregister + 再登録に対応 |
| テスト | mock しにくい | シングルトンを置換可能 |
| 可用性チェック | ロジックが分散 | 集中キャッシュ |

### 単一責任の原則

- **Registry**：登録、照会、ディスパッチのみ担当
- **Tool files**：自身の実装と登録のみ担当
- **Model tools**：発見とルーティングのみ担当
- **Run agent**：実行ループのみ担当

各モジュールの責任が明確で、依存方向は一方向です。

## 設定と操作

### 新ツールの追加

1. `tools/your_tool.py` でツール関数を実装
2. ファイル末尾で `registry.register(...)` を呼び出す
3. `hermes_cli/toolsets.py` で toolset を追加

> 注意：`model_tools.py` の import リストを**手動編集する必要はありません**。`discover_builtin_tools()` が起動時に `tools/*.py` をスキャンし、トップレベルに `registry.register(...)` がある限り、モジュールは自動的にインポートされます。

### 登録済みツールの確認

```python
from tools.registry import registry
print(registry.get_all_tool_names())
print(registry.get_tool_to_toolset_map())
```

### ツールセット可用性の確認

```python
print(registry.check_toolset_requirements())
# 出力: {'terminal': True, 'web': False, 'browser': True, ...}
```

## 他システムとの関係

- [[toolsets-system]] — Registry は toolset 単位でツールを整理
- [[model-tools-dispatch]] — model_tools.py は Registry を通じてツールを発見
- [[mcp-and-plugins]] — MCP は deregister/register で動的ツール発見を実装
- [[large-tool-result-handling]] — ディスパッチ結果は統一エラー形式で処理される
- [[fuzzy-matching-engine]] — patch ツールが使用する 8 段階ファジーマッチングエンジン
- [[code-execution-sandbox]] — execute_code サンドボックスツール
