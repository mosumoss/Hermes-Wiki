---
title: Toolsets System
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [toolset, tool, tool-registry, architecture]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/toolsets-system.md
---

# ツールセットシステム

## 概要

Toolsets は Hermes Agent の**ツールグループ化システム**で、ツールを意味のあるセットに組み合わせ、異なるシーン / プラットフォームで異なるツールセットを有効化できるようにします。

## コア設計

```python
# toolsets.py
_HERMES_CORE_TOOLS = [
    # Web
    "web_search", "web_extract",
    # Terminal + プロセス管理
    "terminal", "process",
    # ファイル操作
    "read_file", "write_file", "patch", "search_files",
    # Vision + 画像生成
    "vision_analyze", "image_generate",
    # Skills
    "skills_list", "skill_view", "skill_manage",
    # ブラウザ自動化
    "browser_navigate", "browser_snapshot", "browser_click",
    "browser_type", "browser_scroll", "browser_back",
    "browser_press", "browser_get_images",
    "browser_vision", "browser_console",
    # 音声合成
    "text_to_speech",
    # 計画 + 記憶
    "todo", "memory",
    # セッション履歴検索
    "session_search",
    # 明確化質問
    "clarify",
    # コード実行 + 委譲
    "execute_code", "delegate_task",
    # Cronjob 管理
    "cronjob",
    # クロスプラットフォームメッセージング
    "send_message",
    # Home Assistant
    "ha_list_entities", "ha_get_state", "ha_list_services", "ha_call_service",
]
```

## ツールセット定義

```python
TOOLSETS = {
    # 基本ツールセット
    "web": {
        "description": "Web 研究とコンテンツ抽出ツール",
        "tools": ["web_search", "web_extract"],
        "includes": []  # 他のツールセットを含まない
    },
    
    # 組み合わせツールセット
    "debugging": {
        "description": "デバッグとトラブルシューティング ツールキット",
        "tools": ["terminal", "process"],
        "includes": ["web", "file"]  # 他のツールセットを組み合わせ
    },
    
    # プラットフォーム固有ツールセット
    "hermes-telegram": {
        "description": "Telegram ボットツールセット",
        "tools": _HERMES_CORE_TOOLS,  # コアツールリストを使用
        "includes": []
    },
    
    "hermes-acp": {
        "description": "エディタ統合（VS Code、Zed、JetBrains）",
        "tools": [...],  # コーディング専用、メッセージ / 音声 / clarify なし
        "includes": []
    },
    
    "hermes-api-server": {
        "description": "OpenAI 互換 API サーバー",
        "tools": [...],  # 完全ツールセット、対話 UI ツールなし
        "includes": []
    },
    
    # 注：「all」は TOOLSETS のエントリではない
    # resolve_toolset() で特殊ケースとして処理される
    # if name in {"all", "*"}: ...
}
```

## 再帰解決

```python
def resolve_toolset(name: str, visited: Set[str] = None) -> List[str]:
    """ツールセットを再帰的に解決、組み合わせ依存を処理"""
    
    # 特殊エイリアス: all または *
    if name in {"all", "*"}:
        all_tools = set()
        for toolset_name in get_toolset_names():
            resolved = resolve_toolset(toolset_name, visited.copy())
            all_tools.update(resolved)
        return list(all_tools)
    
    # 循環検出
    if name in visited:
        return []  # サイレントに返す
    
    visited.add(name)
    toolset = TOOLSETS.get(name)
    
    # 直接ツールを収集
    tools = set(toolset.get("tools", []))
    
    # includes のツールセットを再帰的に解決
    for included_name in toolset.get("includes", []):
        included_tools = resolve_toolset(included_name, visited)
        tools.update(included_tools)
    
    return list(tools)
```

## プラットフォームツールセット

| ツールセット | プラットフォーム | 特徴 |
|--------|------|------|
| `hermes-cli` | ターミナル CLI | 完全ツールセット |
| `hermes-telegram` | Telegram | 完全ツールセット |
| `hermes-discord` | Discord | 完全ツールセット |
| `hermes-whatsapp` | WhatsApp | 完全ツールセット |
| `hermes-slack` | Slack | 完全ツールセット |
| `hermes-signal` | Signal | 完全ツールセット |
| `hermes-homeassistant` | Home Assistant | スマートホーム制御 |
| `hermes-email` | Email (IMAP/SMTP) | メール対話 |
| `hermes-sms` | SMS (Twilio) | SMS、文字数制限 |
| `hermes-mattermost` | Mattermost | セルフホストチームメッセージ |
| `hermes-matrix` | Matrix | 分散暗号化メッセージング |
| `hermes-dingtalk` | DingTalk | 企業メッセージング |
| `hermes-feishu` | Feishu / Lark | 企業メッセージング |
| `hermes-wecom` | WeCom | 企業 WeChat メッセージング |
| `hermes-webhook` | Webhook | 外部イベント受信 |
| `hermes-acp` | エディタ統合 | コーディング専用 |
| `hermes-api-server` | HTTP API | HTTP 経由アクセス |

## プラグイン拡張

ツールセットはプラグインの動的登録をサポート：

```python
def _get_plugin_toolset_names() -> Set[str]:
    """プラグインが登録したツールセット名を返す"""
    from tools.registry import registry
    return {
        entry.toolset
        for entry in registry._tools.values()
        if entry.toolset not in TOOLSETS
    }
```

## ツール登録テーブル

```python
# tools/registry.py
class ToolRegistry:
    def register(self, name, toolset, schema, handler, ...):
        """中央レジストリにツールを登録（toolset パラメータ必須）"""
    
    def get_schema(self, name):
        """ツールの schema 定義を取得"""
    
    def get_all_tool_names(self):
        """全登録済みツール名を取得"""
```

各ツールファイルはインポート時に自動登録：

```python
# tools/terminal_tool.py
from tools.registry import registry

registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=terminal_handler,
    ...
)
```

## ツール有効化 / 無効化

`hermes tools` コマンドまたは設定で管理：

```yaml
# ~/.hermes/config.yaml
tools:
  disabled:
    telegram: ["image_generate"]
    discord: ["text_to_speech"]
```

## ファイル依存チェーン

```
tools/registry.py  (依存なし — 全ツールファイルからインポートされる)
       ↑
tools/*.py  (各ファイルがインポート時に registry.register() を呼ぶ)
       ↑
model_tools.py  (tools/registry をインポート + ツール発見をトリガー)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

## 関連ページ

- [[tool-registry-architecture]] — 中央ツールレジストリ（Registry は toolset 単位でツールを整理）
- [[model-tools-dispatch]] — ツール編成層は toolset でツール定義をフィルタ
- [[mcp-and-plugins]] — プラグインが動的に登録してツールセットを拡張

## 関連ファイル

- `toolsets.py` — Toolset 定義と解決
- `tools/registry.py` — 中央ツールレジストリ
- `model_tools.py` — ツール編成、`_discover_tools()`, `handle_function_call()`
- `hermes_cli/tools_config.py` — ツール有効化 / 無効化設定
