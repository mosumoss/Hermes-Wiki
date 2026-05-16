---
title: Agent Loop and Prompt Assembly
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [agent-loop, prompt-builder, architecture, component]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/agent-loop-and-prompt-assembly.md
---

# Agent ループとプロンプト組み立て

## AIAgent のコアループ

```python
# run_agent.py
class AIAgent:
    def __init__(self,
        model: str = "anthropic/claude-opus-4.6",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,           # "cli", "telegram" 等
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        # ... 他のパラメータ
    ): ...

    def chat(self, message: str) -> str:
        """シンプルなインターフェース — 最終応答文字列を返す"""

    def run_conversation(self, user_message, system_message=None,
                         conversation_history=None, task_id=None) -> dict:
        """完全なインターフェース — dict {final_response, messages} を返す"""
```

## 対話ループ

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    response = client.chat.completions.create(
        model=model,
        messages=messages,
        tools=tool_schemas
    )
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content  # 最終応答
```

- 完全同期実行
- メッセージ形式は OpenAI 標準に準拠：`{"role": "system/user/assistant/tool", ...}`
- 推論内容は `assistant_msg["reasoning"]` に格納

## システムプロンプトの構築

`AIAgent._build_system_prompt()` は固定順序で `prompt_parts` を結合し、最後に `"\n\n".join(prompt_parts)` で完全なシステムプロンプトを返します。実装の詳細は [[prompt-builder-architecture]] を参照。組み立て順序は以下の通り：

1. **SOUL.md** — Agent のアイデンティティ（`~/.hermes/SOUL.md`、無い場合は `DEFAULT_AGENT_IDENTITY`）
2. **ツール使用の強制指示** — モデルファミリーごとにフィルタ
3. **モデル固有の実行ガイダンス** — OpenAI / Google 等の専用
4. **ユーザー / Gateway システムメッセージ** — `run_conversation` に `system_message` を渡した場合
5. **Memory 使用ガイダンス** — モデルに `memory` ツールの使い方を伝える
6. **MEMORY スナップショット** — `~/.hermes/memories/MEMORY.md`（フリーズ）
7. **USER PROFILE スナップショット** — `~/.hermes/memories/USER.md`（フリーズ）
8. **外部 Memory Provider ブロック** — mem0 / honcho / holographic 等、有効化されていれば
9. **Skills インデックス** — `~/.hermes/skills/` をスキャンして生成
10. **プロジェクトコンテキストファイル** — `.hermes.md → AGENTS.md → CLAUDE.md → .cursorrules`（first match wins）
11. **セッションメタデータ** — タイムスタンプ、Model、Provider、Session ID
12. **プラットフォームヒント** — `PLATFORM_HINTS[platform]`
13. **セッションコンテキスト** — Gateway が注入するソース、Home Channel、配信オプション

**キャッシュ機構**：システムプロンプトはセッション内で 1 回だけ構築（`self._cached_system_prompt`）、コンテキスト圧縮後にのみ再構築されます。これにより毎ターンの対話で同じプロンプトが再利用され、**LLM の prefix cache ヒット率が最大化**されます。

**メモリのフリーズ方式**：MEMORY.md / USER.md の 6-7 番目の内容は**ロード時のスナップショット**です。会話中にモデルが新しいメモリを書き込んでも、現在のセッションのシステムプロンプトには反映されず、次回のセッションで初めて有効になります。これは prefix cache を保護するための意図的な設計です。

## プラットフォームヒント (PLATFORM_HINTS)

異なるメッセージングプラットフォームに対して特定のガイダンスを注入します：

| プラットフォーム | 主なガイダンス |
|------|----------|
| `telegram` | Markdown を使わない、`MEDIA:` パスでファイル送信 |
| `discord` | Markdown サポート、ファイルは添付として |
| `whatsapp` | Markdown を使わない、ネイティブメディア |
| `slack` | ファイルは添付として |
| `signal` | Markdown を使わない、プレーンテキスト |
| `email` | 構造化されたクリアな出力、メール向け |
| `cron` | ユーザー不在、完全自律実行 |
| `cli` | ターミナルでレンダリング可能なプレーンテキスト |
| `sms` | プレーンテキスト、約 1600 文字制限 |

## 実行ガイダンス

### 共通：ツール使用の強制指示

```text
# Tool-use enforcement
You MUST use your tools to take action — do not describe what you would do
or plan to do without actually doing it.
```

対象モデル：`gpt`, `codex`, `gemini`, `gemma`, `grok`

### OpenAI モデル向けの追加ガイダンス

```xml
<tool_persistence>
- Use tools whenever they improve correctness
- Do not stop early when another tool call would improve the result
- Keep calling tools until: task complete AND verified
</tool_persistence>

<prerequisite_checks>
- Check whether prerequisite discovery steps are needed
- Do not skip prerequisite steps
</prerequisite_checks>

<verification>
- Correctness: does output satisfy every requirement?
- Grounding: are factual claims backed by tool outputs?
- Formatting: does output match requested format?
- Safety: confirm scope before executing side effects
</verification>
```

### Google モデル向けの操作ガイダンス

- 常に絶対パスを使用
- 修正前に検証する（read_file / search_files）
- 依存関係を確認（ライブラリが利用可能と仮定しない）
- ツールの並列呼び出し
- 非対話型コマンド（`-y`, `--yes` フラグ）

## コンテキストファイルの注入

**2 つの独立したロードパス**：

| ファイル | 場所 | 検索範囲 |
|------|------|---------|
| **SOUL.md** | `~/.hermes/SOUL.md` | グローバル単一パス、独立ロード（Agent アイデンティティスロット） |
| **.hermes.md** | cwd から git root まで遡る | プロジェクトレベル設定、優先度 1 |
| **AGENTS.md** | cwd のみ | コードベース開発ガイド、優先度 2 |
| **CLAUDE.md** | cwd のみ | Anthropic 形式互換、優先度 3 |
| **.cursorrules** | cwd のみ | Cursor 形式互換、優先度 4 |

**プロジェクトコンテキストファイルは排他的**（first match wins）— 最初に見つかったものでロードを停止し、後続はロードされません。SOUL.md はこの競合に参加せず、常にプロジェクトファイルと共存します。

**ロード制御の方法**：
- `TERMINAL_CWD` — Gateway モードで、プロジェクトファイルを探すディレクトリを決定。デフォルトは `Path.home()`
- `skip_context_files=True` — プロジェクトファイル読み込みを完全にスキップ（サブ Agent でよく使用）
- `build_context_files_prompt(skip_soul=True)` — SOUL.md をスキップ（アイデンティティスロットとして既にロード済みの場合に使用）

注入内容に対するセキュリティスキャン：

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'curl\s+[^\n]*\$?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials)', "read_secrets"),
    # ... 他のパターン
]
```

## スキルインデックスの注入

スキルインデックスは**システムプロンプトの一部**で、`_build_system_prompt()` が `build_skills_system_prompt()` を呼んで `prompt_parts` に結合します。アイデンティティ、メモリ、コンテキストファイル等と一緒に完全なシステムプロンプトを構成します。

システムプロンプトはセッション内で 1 回だけ構築され（`self._cached_system_prompt` にキャッシュ）、コンテキスト圧縮後にのみ再構築されます。これにより毎ターンの対話で同じプロンプトが再利用され、**LLM の prefix cache ヒット率を最大化**します。

## Skills Prompt の 2 層キャッシュ

スキルインデックスのテキストを構築するには `~/.hermes/skills/` 配下のすべてのファイルをスキャンして frontmatter を解析する必要があります。重複した file I/O を避けるため、2 層キャッシュで高速化：

| レイヤー | 保管 | ヒット条件 | 無効化条件 |
|------|------|----------|----------|
| Layer 1 | メモリ LRU（`OrderedDict`、最大 8 件） | cache_key 一致（skills_dir + tools + toolsets + platform） | プロセス再起動 |
| Layer 2 | ディスクスナップショット（`.skills_prompt_snapshot.json`） | mtime + size マニフェスト検証通過 | スキルファイル変更 |

両方ともキャッシュミスした場合、ファイルシステム全体をスキャン → ディスクスナップショット書き込み + メモリキャッシュ書き込み。

**区別すべき点**：このキャッシュ最適化は「インデックステキストの生成」の I/O 速度を上げるもので、LLM API レイヤーの prefix cache（既に計算済みのトークンを再利用）とは別レイヤーです。

## ロール切り替え

一部のモデルは `system` ロールの代わりに `developer` ロールを使用します：

```python
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
# API 境界の _build_api_kwargs() で切り替え
```

## 関連ページ

- [[agent-loop-and-prompt-assembly]] — AIAgent コア対話ループクラス（本ページ）
- [[prompt-builder-architecture]] — システムプロンプトのモジュラー構築アーキテクチャ
- [[context-compressor-architecture]] — コンテキスト圧縮と要約機構

## 関連ファイル

- `run_agent.py` — AIAgent クラス実装
- `agent/prompt_builder.py` — システムプロンプト組み立て（959 行）
- `model_tools.py` — ツール編成
- `agent/context_compressor.py` — コンテキスト圧縮
- `agent/prompt_caching.py` — Anthropic prompt キャッシュ
