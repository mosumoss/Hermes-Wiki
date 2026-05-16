---
title: CLI アーキテクチャとターミナル対話設計
created: 2026-04-07
updated: 2026-04-11
type: concept
tags: [architecture, cli, terminal, ux]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/cli-architecture.md
---

# CLI アーキテクチャとターミナル対話設計

## 設計原理

Hermes CLI は完全なターミナルユーザー体験を提供します：オートコンプリート、複数行編集、ストリーミング出力、ツール呼び出し可視化。`prompt_toolkit` と `rich` をベースに構築されています。

## コアコンポーネント

```python
# cli.py
class HermesCLI:
    """Hermes CLI メインクラス"""
    
    def __init__(self):
        self.agent = None
        self.config = load_cli_config()
        self.session_db = SessionDB(...)
        self.todo_store = TodoStore()
    
    def run(self):
        """メインループ"""
        while True:
            user_input = self._get_input()  # prompt_toolkit 入力
            if user_input.startswith("/"):
                self._handle_command(user_input)
            else:
                self._handle_message(user_input)
```

## 入力システム

```python
from prompt_toolkit import PromptSession
from prompt_toolkit.history import FileHistory
from prompt_toolkit.auto_suggest import AutoSuggestFromHistory

session = PromptSession(
    history=FileHistory("~/.hermes/input_history"),
    auto_suggest=AutoSuggestFromHistory(),
    completer=SlashCommandCompleter(),  # hermes_cli/commands.py で定義
)

user_input = session.prompt(get_active_prompt_symbol())  # プロンプト記号は skin engine で設定
```

### スラッシュコマンド補完

```python
class SlashCommandCompleter(Completer):
    def get_completions(self, document, complete_event):
        text = document.text_before_cursor
        if text.startswith("/"):
            for cmd_name, cmd_def in COMMANDS.items():
                if cmd_name.startswith(text[1:]):
                    yield Completion(cmd_name, start_position=-len(text[1:]))
```

## 表示システム

### KawaiiSpinner

```python
# agent/display.py
class KawaiiSpinner:
    """アニメーションローディングインジケータ"""
    
    SPINNERS: dict          # 9 種類の名前付きアニメーション集 ('dots', 'bounce', 'grow', ...)
    KAWAII_WAITING: list     # 10 個の複数文字顔文字
    KAWAII_THINKING: list    # 15 個の複数文字顔文字
    THINKING_VERBS: list    # 15 個の動詞 ("pondering", "contemplating", "musing", "cogitating", "ruminating", ...)
    
    def show(self, message: str):
        """ローディングアニメーション表示"""
        # Rich パネルとアニメーションを使用
```

### ツール呼び出しプレビュー

```python
def build_tool_preview(tool_name: str, args: dict) -> str:
    """ツール呼び出しプレビューを構築"""
    preview = f"🔧 {tool_name}("
    for key, value in list(args.items())[:3]:
        preview += f"\n  {key}={preview_value(value)},"
    preview += "\n)"
    return preview

def get_cute_tool_message(tool_name: str) -> str:
    """かわいいツール実行メッセージを取得"""
    emoji = _get_tool_emoji(tool_name)
    return f"{emoji} Calling {tool_name}..."
```

## Skin エンジン

```python
# hermes_cli/skin_engine.py
@dataclass
class SkinConfig:
    """スキン設定データクラス"""
    ...

# モジュールレベル関数（クラスではない）
def init_skin_from_config(): ...
def get_active_skin() -> SkinConfig: ...
def list_skins() -> list: ...
def set_active_skin(name: str): ...

# 設定例
# ~/.hermes/config.yaml
display:
  skin: "default"  # またはカスタムスキン名
```

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Claude Code | Codex CLI |
|------|--------|-------------|-----------|
| スラッシュコマンド補完 | ✅ 自動 | ✅ | ❌ |
| 複数行編集 | ✅ | ✅ | ✅ |
| 入力履歴 | ✅ ファイル永続化 | ✅ | ✅ |
| アニメーションローディング | ✅ KawaiiSpinner | ✅ シンプル | ✅ シンプル |
| テーマシステム | ✅ Skin Engine | ❌ | ❌ |
| ツール呼び出しプレビュー | ✅ 整形済み | ✅ | ❌ |

## 関連ページ

- [[configuration-and-profiles]] — 設定管理と Profile システム
- [[hook-system-architecture]] — Hook とプラグイン拡張システム
- [[session-search-and-sessiondb]] — セッション検索と SessionDB
- [[voice-mode-architecture]] — 音声モード（Push-to-talk → STT → TTS）
- [[skin-engine]] — スキン / テーマカスタマイズ
- [[context-references]] — @file / @diff / @url 引用システム
- [[worktree-isolation]] — Git Worktree 並列隔離
- [[code-execution-sandbox]] — コード実行サンドボックス

## 関連ファイル

- `cli.py` — CLI メインクラス
- `hermes_cli/main.py` — エントリポイントとサブコマンド
- `hermes_cli/commands.py` — スラッシュコマンド定義
- `hermes_cli/dump.py` — `hermes dump` 環境サマリー（プレーンテキスト、デバッグ / issue 提出用）
- `agent/display.py` — 表示システム
- `hermes_cli/skin_engine.py` — スキンエンジン
