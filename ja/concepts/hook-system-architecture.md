---
title: Hook システムアーキテクチャ
created: 2026-04-08
updated: 2026-04-18
type: concept
tags: [architecture, module, extensibility, mcp, plugins]
sources: [gateway/hooks.py, hermes_cli/plugins.py, model_tools.py, run_agent.py]
translation: ja
original: ../../concepts/hook-system-architecture.md
---

# Hook システムアーキテクチャ

## 概要

Hermes Agent には相補的な 2 つの拡張システムがあります：

| システム                | 場所                    | 責任                                      | コード量 |
| ----------------- | --------------------- | --------------------------------------- | ----- |
| **Gateway Hooks** | gateway/hooks.py      | ゲートウェイイベント駆動フック（startup/session/agent/command） | 170 行 |
| **Plugin System** | hermes_cli/plugins.py | プラグインライフサイクルフック + ツール登録 + CLI コマンド拡張   | 609 行 |

コア理念：**Hooks はイベント通知を処理、Plugins は機能拡張を処理 — 両者は相補的。**

## アーキテクチャ原理

### Gateway Hooks — イベント駆動

Gateway Hooks は**軽量イベントシステム**で、ゲートウェイライフサイクルの重要ポイントでハンドラを発火：

| イベント | 発火タイミング |
|---|---|
| `gateway:startup` | ゲートウェイプロセス起動 |
| `session:start` | 新セッション作成（最初のメッセージ） |
| `session:end` | セッション終了（ユーザーが /new または /reset 実行） |
| `session:reset` | セッションリセット完了 |
| `agent:start` | Agent がメッセージ処理開始 |
| `agent:step` | ツール呼び出しループの各ラウンド |
| `agent:end` | Agent がメッセージ処理完了 |
| `command:*` | 任意のスラッシュコマンド実行（ワイルドカード） |

### Plugin System — 機能拡張

Plugin System はプラグインによる**ツール、フックコールバック、CLI サブコマンド**の登録、および対話へのメッセージ注入をサポート。

**3 階層プラグインソース**：
1. **ユーザープラグイン** — `~/.hermes/plugins/<name>/`
2. **プロジェクトプラグイン** — `./.hermes/plugins/<name>/`（`HERMES_ENABLE_PROJECT_PLUGINS` 必要）
3. **Pip プラグイン** — `hermes_agent.plugins` entry-point グループ経由でインストール

## コアコンポーネント

### Gateway Hooks

#### HookRegistry

```python
class HookRegistry:
    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}  # event_type → handlers
        self._loaded_hooks: List[dict] = []             # メタデータ

    def discover_and_load(self):
        """
        1. 内蔵フックを登録（boot-md）
        2. ~/.hermes/hooks/ ディレクトリをスキャン
        3. 各フックディレクトリには以下が必要:
           - HOOK.yaml (name, description, events)
           - handler.py (async def handle(event_type, context))
        4. handler.py モジュールを動的ロード
        5. 各宣言イベントを登録
        """
```

#### イベント発射

```python
async def emit(self, event_type, context=None):
    """
    全登録ハンドラを発火:
    1. 完全一致: handlers["agent:start"]
    2. ワイルドカード一致: handlers["command:*"] が "command:reset" にマッチ
    3. 同期 / 非同期ハンドラ両対応
    4. エラーキャッチ、メインフローをブロックしない
    """
```

#### 内蔵フック: boot-md

```python
# gateway/builtin_hooks/boot_md.py
# ゲートウェイ起動時に ~/.hermes/BOOT.md を実行
# ユーザーがゲートウェイ起動時にカスタム初期化指示を注入可能
```

#### フックディレクトリ構造

```
~/.hermes/hooks/
  notify-on-start/
    HOOK.yaml          # name: notify-on-start
                       # events: [agent:start]
    handler.py         # async def handle(event_type, context):
                       #     ...
```

### Plugin System

#### PluginContext — プラグインの API ファサード

```python
class PluginContext:
    """プラグインに提供されるファサード、ツール / フック / CLI コマンド登録を許可"""
    
    def register_tool(name, toolset, schema, handler, ...):
        """グローバル registry にツール登録"""
    
    def inject_message(content, role="user"):
        """
        アクティブな対話にメッセージ注入:
        - Agent アイドル時 → 次の入力としてキュー
        - Agent 実行中 → 中断して注入
        """
    
    def register_cli_command(name, help, setup_fn, handler_fn):
        """CLI サブコマンド登録（例: hermes honcho ...）"""
    
    def register_hook(hook_name, callback):
        """ライフサイクルフックコールバック登録"""
```

#### PluginManager

```python
class PluginManager:
    def discover_and_load(self):
        """
        1. ユーザープラグインスキャン（~/.hermes/plugins/）
        2. プロジェクトプラグインスキャン（./.hermes/plugins/、任意）
        3. pip entry-points スキャン
        4. 各プラグインの register(ctx) ロード
        5. config で無効化されたプラグインをスキップ
        """
```

#### プラグイン構造

```
~/.hermes/plugins/my-plugin/
  plugin.yaml          # name, version, description
                       # requires_env: [MY_API_KEY]
                       # provides_tools: [my_tool]
                       # provides_hooks: [pre_tool_call]
  __init__.py          # def register(ctx):
                       #     ctx.register_tool(...)
                       #     ctx.register_hook(...)
```

#### ライフサイクルフック

```python
VALID_HOOKS = {
    "pre_tool_call",      # ツール呼び出し前
    "post_tool_call",     # ツール呼び出し後
    "pre_llm_call",       # LLM 呼び出し前
    "post_llm_call",      # LLM 呼び出し後
    "pre_api_request",    # API リクエスト前
    "post_api_request",   # API リクエスト後
    "on_session_start",   # セッション開始
    "on_session_end",     # セッション終了
}
```

#### フック呼び出し

```python
def invoke_hook(self, hook_name, **kwargs):
    """
    全登録コールバックを呼ぶ:
    1. 各コールバックは独立して try/except（エラーは伝播しない）
    2. 非 None の返り値を収集
    3. pre_llm_call では、ユーザーメッセージに注入する context を返せる
    
    重要: context はシステムプロンプトではなくユーザーメッセージに注入
    → システムプロンプトを変えない → キャッシュヒット
    → 注入内容は一時的、session DB には永続化しない
    """
```

#### Hook 呼び出しポイント

`model_tools.py` は `handle_function_call()` でプラグインフックを呼ぶ：

```python
def handle_function_call(function_name, function_args, ...):
    # pre_tool_call フック
    invoke_hook("pre_tool_call", tool_name=..., args=...)
    
    result = registry.dispatch(function_name, function_args, ...)
    
    # post_tool_call フック
    invoke_hook("post_tool_call", tool_name=..., args=..., result=...)
    
    return result
```

#### pre_tool_call でツール実行ブロック（2026-04-13）

`pre_tool_call` フックは現在**ツール実行をブロックできる**。プラグインが以下を返す：

```python
def my_pre_tool_call(tool_name, args):
    if tool_name == "terminal" and "rm -rf" in args.get("command", ""):
        return {"action": "block", "message": "Destructive commands disabled by policy"}
```

フレームワークは `get_pre_tool_call_block_message()`（`hermes_cli/plugins.py:658`）で全プラグインの返り値を収集、**最初の** `{"action": "block", "message": ...}` が有効化：
- ツールは実行スキップ（`registry.dispatch` に入らない）
- `message` はツール結果としてモデルに返却、モデルが次のステップを調整
- 全副作用をスキップ：カウンタリセット、checkpoints、コールバック、read-loop tracker いずれもトリガーされない

**2 つの実行パスをカバー**：
- `handle_function_call()`（`model_tools.py:429`）
- `run_agent.py _invoke_tool`（順次 / 並行両パス）

二重発火回避のため、`handle_function_call()` は `skip_pre_tool_call_hook=True` をサポート：`run_agent.py` が外側で既にチェックした場合、`handle_function_call` 呼び出し時にこのフラグを渡して二重チェックをスキップ。

**典型用途**：
- セキュリティポリシー（危険コマンドをブロック）
- クォータ / レート制限
- ホワイトリストモード（特定ツールのみ許可）
- 承認フロー（人間の確認後のみ許可）

## 設計の優位性

### Gateway Hooks vs Plugin System

| 観点 | Gateway Hooks | Plugin System |
|---|---|---|
| 適用範囲 | Gateway モードのみ | CLI + Gateway |
| 登録方法 | ディレクトリスキャン（HOOK.yaml） | ディレクトリ / entry-point スキャン（plugin.yaml） |
| 機能 | イベント通知 | ツール登録 + フック + CLI コマンド + メッセージ注入 |
| 複雑度 | 軽量（170 行） | 完全（609 行） |
| 利用シーン | 起動通知、監査、モニタリング | ツール拡張、カスタム動作、サードパーティ統合 |

### エラー隔離

```python
# 両システムとも「エラー非伝播」設計
try:
    result = fn(event_type, context)
except Exception as e:
    print(f"[hooks] Error in handler: {e}")  # ログのみ、ブロックしない
```

**設計哲学**：拡張システムのエラーはコア Agent フローに影響すべきでない。

### コンテキスト注入のキャッシュフレンドリー設計

Plugin hooks が返す context はシステムプロンプトではなく**ユーザーメッセージ**に注入：

```
システムプロンプト（キャッシュヒット ✓）
  ├── アイデンティティ定義（不変）
  ├── プラットフォームヒント（不変）
  └── スキルインデックス（不変）

ユーザーメッセージ（毎ターン異なる）
  ├── ユーザー元入力
  └── [注入された context]  ← フック返却内容
```

これによりシステムプロンプトの prompt cache が動的注入内容で無効化されないことを保証。

## 設定と操作

### プラグイン無効化

```yaml
# config.yaml
plugins:
  disabled: ["some-plugin", "another-plugin"]
```

### プロジェクトプラグイン

```bash
export HERMES_ENABLE_PROJECT_PLUGINS=true
```

### pip プラグインのインストール

```bash
# プラグインパッケージは pyproject.toml で宣言:
# [project.entry-points."hermes_agent.plugins"]
# my-plugin = "my_plugin:register"

pip install hermes-agent-my-plugin
```

## PluginContext 新規 API（v0.10.0、2026-04-16）

### `register_command()` — プラグインスラッシュコマンド

以前 `cli.py` と `gateway/run.py` のディスパッチコードは既に `get_plugin_command_handler()` を呼んでいたが、登録側が未実装。v0.10.0 でこのチェーンを補完：

```python
def register(ctx):
    ctx.register_command(
        name="deploy",
        description="Deploy the current project",
        handler=my_deploy_handler,
    )
```

- 名前正規化 + 内蔵コマンドとの衝突検出
- 登録されたコマンドは自動的に Telegram bot メニューと CLI 自動補完に表示
- `/plugins` で各プラグインが登録したコマンド数を表示

### `dispatch_tool()` — プラグインのツールディスパッチ

プラグインのスラッシュコマンドハンドラは登録テーブル経由でツール呼び出しをディスパッチ可能、親 agent コンテキストを自動注入：

```python
async def my_handler(ctx, args):
    result = ctx.dispatch_tool("delegate_task", {
        "task": "refactor auth module",
        "instructions": "..."
    })
```

- CLI モード：`_cli_ref` から親 agent を遅延解決
- Gateway モード：`_cli_ref` なし、ツールは優雅に劣化
- 利用シーン：`/deliver` と `/fanout` 等のプラグインコマンドが `delegate_task` でサブ agent を派生

### Shell Hooks（v2026.4.18+）

実装は `agent/shell_hooks.py`（831 行）+ `hermes_cli/hooks.py`（385 行）。Hook コールバックは Python に限定されない — ユーザーは `config.yaml` で shell スクリプトをフックとして宣言可能：

```yaml
hooks:
  pre_tool_call:
    - command: /path/to/my-hook.sh
  subagent_stop:
    - command: /path/to/audit.sh
```

スクリプトは stdin から JSON イベント（tool_name/args 等）を受け取り、stdout から JSON 決定を返す（ツール呼び出しブロック、context 注入が可能）。

**重要設計**：
- `PluginManager._hooks` に closure 登録、`invoke_hook()` 呼び出し点に変更なし
- `subprocess.run(shell=False)` + `shlex.split` — shell injection なし
- 初回利用時に `(event, command)` ペアごとにユーザー同意を求め、allowlist JSON に保管
- `--accept-hooks` / `HERMES_ACCEPT_HOOKS=1` / `hooks_auto_accept` でバイパス可能
- `hermes hooks list/test/revoke/doctor` CLI サブコマンド
- Claude Code 互換応答形式（Claude Code エコシステムの hook スクリプトを再利用可能）
- 新規 `subagent_stop` イベント（`delegate_task` サブ agent 終了時にトリガー）

### プラグインスラッシュコマンドのクロスプラットフォームネイティブ化（v2026.4.18+）

`register_command()` で登録されたプラグインスラッシュコマンドは現在、各 gateway プラットフォームでネイティブ表示：

- Discord ネイティブ slash コマンドセレクタ
- Telegram BotCommand メニュー
- Slack `/hermes` サブコマンドマッピング

各プラットフォームに個別のプラグイン API を書く必要なし。`register_command()` に新規 `args_hint` オプションパラメータ追加、プラグインがパラメータ構造を宣言でき、Discord が自動的にパラメータセレクタを生成。

#### 決定型 command フック

`command:<name>` gateway hook は**決定型**にアップグレード、`HookRegistry.emit_collect()` で返り値を収集：

```python
def my_command_hook(event_type, context):
    if context["command"] == "deploy" and not user_has_permission(context["user"]):
        return {"decision": "deny", "message": "Permission denied"}
```

決定タイプ：`deny` / `handled` / `rewrite` / `allow`、コア処理前にインターセプト。後方互換 — fire-and-forget テレメトリフックは引き続き `emit()` を通る。

### Dashboard プラグインシステム

プラグインは Web Dashboard にカスタムタブを追加可能：

```
~/.hermes/plugins/<name>/dashboard/
  manifest.json     # name, label, icon, tab config, entry point
  dist/index.js     # 事前ビルド JS bundle（IIFE、SDK グローバル変数使用）
  plugin_api.py     # オプション FastAPI ルート、/api/plugins/<name>/ にマウント
```

- `GET /api/dashboard/plugins` — 発見されたプラグイン manifest リストを返す
- `GET /api/dashboard/plugins/rescan` — 強制再スキャン
- `GET /dashboard-plugins/<name>/<path>` — 静的アセット提供（パストラバーサル防御付き）
- オプションのバックエンド API ルート自動マウント対応

加えて **Dashboard テーマシステム**新規追加、リアルタイム切替対応。

## 他システムとの関係

- [[tool-registry-architecture]] — プラグインは registry.register() でツール登録
- [[mcp-and-plugins]] — MCP は別のツール発見機構、プラグインシステムと相補
- [[messaging-gateway-architecture]] — Gateway Hooks はゲートウェイライフサイクルで発火
- [[model-tools-dispatch]] — pre/post_tool_call フックは handle_function_call で呼ばれる
