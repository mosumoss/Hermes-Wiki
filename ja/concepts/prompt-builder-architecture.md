---
title: Prompt Builder システムプロンプト構築アーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [architecture, module, component, agent, prompt-builder]
sources: [agent/prompt_builder.py]
translation: ja
original: ../../concepts/prompt-builder-architecture.md
---

# Prompt Builder — システムプロンプト構築アーキテクチャ

## 概要

Prompt Builder は `agent/prompt_builder.py`（44KB / 959 行）に実装され、**システムプロンプトの組み立て**を担います — アイデンティティ定義、プラットフォームヒント、スキルインデックス、コンテキストファイル。すべての関数はステートレスで、`AIAgent._build_system_prompt()` から呼ばれて各モジュールを結合します。

コア理念：**システムプロンプトはモジュラーに結合されており、各コンポーネントは独立してテスト・置換可能。**

## アーキテクチャ原理

### プロンプトコンポーネントの階層（実測構造）

`_build_system_prompt()` は固定順序で `prompt_parts` 配列を結合し、最後に `"\n\n".join(prompt_parts)` で完全な system prompt を返します。実際の API リクエストキャプチャ（約 36K chars / 10K tokens）で検証された実構造：

```
システムプロンプト =
  ① Agent アイデンティティ — SOUL.md（~/.hermes/SOUL.md、存在すれば使用、無ければ DEFAULT_AGENT_IDENTITY）
  ② ツール使用の強制（TOOL_USE_ENFORCEMENT_GUIDANCE、モデルファミリーで絞り込み）
  ③ モデル固有実行ガイダンス（OpenAI / Google 等の専用、モデルファミリーで絞り込み）
  ④ ユーザー / Gateway システムメッセージ（run_conversation に system_message を渡した場合）
  ⑤ Memory ガイダンス（ハードコードプロンプト、モデルに memory ツールの使い方を伝える）
  ⑥ MEMORY スナップショット — ~/.hermes/memories/MEMORY.md（フリーズ、セッション全体で不変）
  ⑦ USER PROFILE スナップショット — ~/.hermes/memories/USER.md（フリーズ、セッション全体で不変）
  ⑧ 外部 Memory Provider ブロック（mem0 / honcho / holographic 等、有効化された場合）
  ⑨ Skills インデックス（build_skills_system_prompt、~/.hermes/skills/ をスキャン）
  ⑩ プロジェクトコンテキストファイル（.hermes.md → AGENTS.md → CLAUDE.md → .cursorrules、first match wins）
  ⑪ セッションメタデータ（タイムスタンプ、Model、Provider、Session ID）
  ⑫ プラットフォームヒント（PLATFORM_HINTS、Telegram / Discord / CLI 等）
  ⑬ セッションコンテキスト（Gateway 注入：ソース、Home Channel、配信オプション）
```

**重要ポイント**：
- **SOUL.md は独立ロード**、「プロジェクトコンテキストファイル」の first-match-wins 競争には参加しない
- **メモリスナップショットはフリーズ方式**を使用 — system prompt はセッション内で 1 回だけ構築されキャッシュ（`self._cached_system_prompt`）、コンテキスト圧縮後にのみ再構築。prefix cache を保護
- **system prompt 全体が 1 つの message**（`role: "system"`）、複数 message の結合ではない

### コンテキストファイル注入の防御

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'disregard\s+(your|all|any)\s+(instructions|rules|guidelines)', "disregard_rules"),
    (r'bypass_restrictions', ...),
    (r'curl\s+.*\${?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\\n]*(\\.env|credentials)', "read_secrets"),
]

_CONTEXT_INVISIBLE_CHARS = {
    '​', '‌', '‍', '⁠', '﻿',  # ゼロ幅文字
    '‪', '‫', '‬', '‭', '‮',  # 双方向テキスト制御
}
```

**二重防御**：
1. **脅威パターン検出**：10 種類の一般的注入パターン（指示無視、行動隠蔽、実行注入、秘密情報流出など）
2. **不可視 Unicode 検出**：10 種類のゼロ幅文字と双方向テキスト制御文字（5 種類のゼロ幅 + 5 種類の双方向制御、視覚的欺瞞に利用される可能性）

脅威検出時：`[BLOCKED: filename contained potential prompt injection]` に置換。

## コアコンポーネント

### 1. コンテキストファイル発見

**2 つの独立したロードパス**：

```python
# パス A: SOUL.md — Agent アイデンティティ、固定パス、常にロード
load_soul_md()  → ~/.hermes/SOUL.md  (HERMES_HOME)

# パス B: プロジェクトコンテキストファイル — 排他的、first match wins
project_context = (
    _load_hermes_md(cwd_path)    # .hermes.md / HERMES.md（git root まで遡る）
    or _load_agents_md(cwd_path) # AGENTS.md（cwd のみ）
    or _load_claude_md(cwd_path) # CLAUDE.md（cwd のみ）
    or _load_cursorrules(cwd_path) # .cursorrules / .cursor/rules/*.mdc（cwd のみ）
)
```

| ファイル | 場所 | 検索範囲 | 役割 |
|------|------|---------|------|
| **SOUL.md** | `~/.hermes/SOUL.md` | HERMES_HOME（グローバル単一） | Agent アイデンティティ / 人格、独立ロード |
| **.hermes.md** | cwd から git root まで遡る | 遡及検索 | プロジェクトレベル設定、優先度 1 |
| **AGENTS.md** | cwd のみ | 再帰しない | コードベース開発ガイド、優先度 2 |
| **CLAUDE.md** | cwd のみ | 再帰しない | Anthropic 形式互換、優先度 3 |
| **.cursorrules** | cwd のみ（`.cursor/rules/*.mdc` 含む） | 再帰しない | Cursor 形式互換、優先度 4 |

**よくある誤解**：
- SOUL.md は**プロジェクトコンテキストの優先度競争に参加しない** — 独立したアイデンティティスロット
- プロジェクトコンテキストファイルは**排他的ロード**（first match wins）、「全部ロード」ではない
- 現在の cwd に `.hermes.md` と `CLAUDE.md` が同時にあると、`.hermes.md` のみがロードされる

**スキップ機構**：
- `AIAgent(skip_context_files=True)` — サブ Agent でよく使用、親 Agent のプロジェクトコンテキスト継承を回避
- 異なるディレクトリで Hermes を起動（例：`TERMINAL_CWD=~` 設定）— 自然にプロジェクトファイルをスキップ
- SOUL.md もスキップ可：`build_context_files_prompt(skip_soul=True)`、SOUL.md が既にアイデンティティスロットとしてロード済みの場合に重複注入回避

**コンテンツ保護**：
- 各ファイル内容の上限 20,000 文字、超過は自動的に先頭・末尾を切り捨て（`[...truncated...]`）
- YAML frontmatter は自動的に剥がす（構造化設定は別途処理）
- 脅威パターンをスキャン（次節参照）

### 2. スキルインデックスとキャッシュ

```python
_SKILLS_PROMPT_CACHE_MAX = 8
_SKILLS_PROMPT_CACHE: OrderedDict[tuple, str] = OrderedDict()

def build_skills_system_prompt(
    available_tools: set,
    available_toolsets: set,
    disabled_skills: set,
) -> str:
    """
    1. skills ディレクトリをスキャン
    2. 各 SKILL.md の frontmatter を解析
    3. プラットフォーム互換性 + 条件付き活性化ルールを確認
    4. スキル一覧プロンプトを構築
    5. 結果をキャッシュ（mtime / size マニフェストベース）
    """
```

### 3. スキルスナップショットの永続化

```python
def _load_skills_snapshot(skills_dir: Path) -> Optional[dict]:
    """ディスクからスナップショットをロード、マニフェスト一致なら再利用"""

def _write_skills_snapshot(skills_dir, manifest, skill_entries, category_descriptions):
    """スナップショットをアトミック書き込み（atomic_json_write）"""
```

**コールドスタート最適化**：スキルファイルが変更されていない場合、ディスクスナップショットから直接ロード、全 SKILL.md の再解析不要。

### 4. スキル条件付き活性化

```python
def _skill_should_show(conditions, available_tools, available_toolsets):
    """
    fallback_for_toolsets: 主ツールセット利用可能時に非表示（バックアップスキル）
    fallback_for_tools: 主ツール利用可能時に非表示
    requires_toolsets: 依存ツールセットが存在しない時に非表示
    requires_tools: 依存ツールが存在しない時に非表示
    """
```

### 5. プラットフォームヒント

```python
PLATFORM_HINTS = {
    "telegram": "You are on Telegram. No markdown. MEDIA:/path for files...",
    "discord": "You are in Discord. MEDIA:/path for attachments...",
    "cli": "You are a CLI AI. Use simple text renderable in terminal.",
    "cron": "You are running as a cron job. No user present. Execute fully...",
    "whatsapp": "You are on WhatsApp. No markdown...",
    "slack": "You are in Slack...",
    "signal": "You are on Signal...",
    "email": "You are communicating via email. Plain text...",
    "sms": "You are communicating via SMS. ~1600 chars limit...",
}
```

### 6. モデル固有実行ガイダンス

#### OpenAI / GPT Codex シリーズ

```python
OPENAI_MODEL_EXECUTION_GUIDANCE = """
<tool_persistence>
- Use tools whenever they improve correctness
- Do not stop early
- If a tool returns empty, retry with different strategy
- Keep calling tools until task is complete AND verified
</tool_persistence>

<prerequisite_checks>
- Check prerequisite discovery before action
- Don't skip steps just because final action seems obvious
</prerequisite_checks>

<verification>
- Correctness: does output satisfy every requirement?
- Grounding: are claims backed by tool outputs?
- Formatting: does output match requested schema?
- Safety: confirm scope before side-effect operations
</verification>

<missing_context>
- Do NOT guess or hallucinate
- Use lookup tools for missing information
- Label assumptions explicitly
</missing_context>
"""
```

#### Gemini / Gemma シリーズ

```python
GOOGLE_MODEL_OPERATIONAL_GUIDANCE = """
- Absolute paths: always use absolute file paths
- Verify first: check file contents before changes
- Dependency checks: check package.json before importing
- Conciseness: keep text brief, focus on actions
- Parallel tool calls: batch independent operations
- Non-interactive: use -y, --yes flags
- Keep going: execute fully, don't stop with a plan
"""
```

### 7. Developer Role 切り替え

```python
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
# OpenAI の新モデルは 'developer' role に対する指示遵守の重みが高い
# API 境界で自動切り替え、内部表現は "system" で統一
```

## 設計の優位性

### モジュラー化の利点

| 観点 | モノリシックプロンプト | モジュラー Prompt Builder |
|---|---|---|
| テスト | ユニットテスト困難 | 各コンポーネントを独立テスト |
| カスタマイズ | 全量置換が必要 | プラットフォーム / モデル / スキルで動的組み立て |
| セキュリティ | 注入検出が困難 | コンテキストファイル独立スキャン |
| メンテナンス | 1 箇所修正が全体に影響 | 各コンポーネントが独立進化 |
| キャッシュ | キャッシュ不可 | スキルインデックスがキャッシュ可能 |

### セキュリティ防御の優位性

従来のコンテキストファイル注入には防御がありません。Prompt Builder は**多層検出**で注入内容が Agent 動作を変えないことを保証します：
1. 脅威パターンの正規表現マッチング
2. 不可視 Unicode 文字の検出
3. 脅威検出時、直接破棄するのではなく BLOCKED マーカーに置換（Agent に問題を認識させる）

## 設定と操作

### Agent アイデンティティのカスタマイズ

`~/hermes-agent/SOUL.md` を作成して個人化されたアイデンティティを定義。

### プロジェクトレベル設定

プロジェクトルートに `.hermes.md` を作成、内容がシステムプロンプトに注入される。

### 特定スキルの無効化

```yaml
# config.yaml
skills:
  disabled: ["some-skill", "another-skill"]
```

## 他システムとの関係

- [[tool-registry-architecture]] — スキル条件活性化は利用可能なツールセットに依存
- [[context-compressor-architecture]] — 圧縮後のメッセージリストが prompt builder に渡されてプロンプト再構築
- [[memory-system-architecture]] — memory ガイダンスはプロンプトの一部
- [[agent-loop-and-prompt-assembly]] — prompt builder は agent ループから呼ばれる
