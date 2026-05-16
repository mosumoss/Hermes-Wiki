---
title: Skills and Memory Interaction
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [skill, memory, architecture, best-practice]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/skills-and-memory-interaction.md
---

# スキルと記憶の相互作用

## 設計哲学

Skills と Memory は Hermes Agent の**異なる種類の永続化機構**で、競合ではなく相補関係にあります：

| 観点 | Memory | Skills |
|------|--------|--------|
| **保存内容** | 事実、嗜好、経験から得た教訓 | 手続き的知識、ワークフロー |
| **容量** | MEMORY.md: 2,200 文字<br>USER.md: 1,375 文字 | ハード制限なし |
| **形式** | エントリリスト（`§` 区切り） | Markdown ドキュメント + ファイル構造 |
| **用途** | 安定した事実の素早い参照 | 複雑タスクの完全ガイド |
| **ロード方式** | システムプロンプトに注入 | 段階的開示（メタデータ → 完全内容） |
| **いつ使うか** | ユーザー嗜好、環境事実、ツール特性 | 5+ ツール呼び出しの複雑ワークフロー |

## 意思決定ツリー

```text
タスク完了後に問う：

この知識は...
├─ シンプルで安定した事実？ → Memory に保存
│   （例：「ユーザーは日本語を好む」「サーバーは /root にある」）
│
└─ 複雑な手続き的フロー？ → Skill として作成
    （例：「ML モデルのデプロイ手順」「X 問題のデバッグフロー」）
```

## 行動ガイダンス

### Memory ガイダンス（システムプロンプトに注入）

```text
You have persistent memory across sessions. Save durable facts using the memory tool:
user preferences, environment details, tool quirks, and stable conventions.
Memory is injected into every turn, so keep it compact and focused on facts
that will still matter later.

Prioritize what reduces future user steering — the most valuable memory is one
that prevents the user from having to correct or remind you again.

Do NOT save task progress, session outcomes, completed-work logs, or temporary
TODO state to memory; use session_search to recall those from past transcripts.
```

### Skills ガイダンス（システムプロンプトに注入）

```text
After completing a complex task (5+ tool calls), fixing a tricky error,
or discovering a non-trivial workflow, save the approach as a skill
with skill_manage so you can reuse it next time.

When using a skill and finding it outdated, incomplete, or wrong,
patch it immediately with skill_manage(action='patch') — don't wait to be asked.
Skills that aren't maintained become liabilities.
```

## スキル自己改善ループ

```text
1. Agent が複雑タスクを実行（5+ ツール呼び出し）
   ↓
2. 新パターンやワークフローを検出
   ↓
3. skill_manage(action='create') でスキル作成
   ↓
4. 次回類似タスク → skills_list でスキル発見
   ↓
5. skill_view で完全指示をロード
   ↓
6. 実行中に問題発見 → skill_manage(action='patch') で修正
   ↓
7. スキルが継続的に改善される
```

## Session Search の役割

`session_search` は第 3 の永続化機構で、**過去の対話**を呼び戻すために使います：

```text
When the user references something from a past conversation or you suspect
relevant cross-session context exists, use session_search to recall it before
asking them to repeat themselves.
```

3 つの機構の比較：

| 機構 | 内容 | 検索方式 |
|------|------|----------|
| **Memory** | 安定事実 | 毎ターン自動的にシステムプロンプトに注入 |
| **Skills** | 手続き的知識 | オンデマンドロード（段階的開示） |
| **Session Search** | 過去対話記録 | FTS5 全文検索 + LLM 要約 |

## 実例

### Memory に保存

```python
# ユーザー訂正
memory(action='add', target='user', content='ユーザーは日本語でのコミュニケーションを好む')

# 環境事実
memory(action='add', target='memory', content='サーバーは Ubuntu 22.04、Python 3.11')

# ツール特性
memory(action='add', target='memory', content='patch ツールはファジーマッチングを使用、minor whitespace の差異では壊れない')
```

### Skill として作成

```python
# 複雑ワークフロー
skill_manage(
    action='create',
    name='deploy-ml-model',
    content='---\nname: deploy-ml-model\n...'
)
```

## メンテナンス優先度

```text
Memory > Skills > Session Search
```

- **Memory** が最重要 — 毎ターン注入、行動に直接影響
- **Skills** が次 — オンデマンドロード、ただし複雑タスクの品質に影響
- **Session Search** が最後 — コンテキスト呼び戻し用、コア行動ではない

## 関連ページ

- [[skills-system-architecture]] — スキルシステムの段階的開示アーキテクチャ
- [[memory-system-architecture]] — 記憶システムのフリーズスナップショットとアトミック書き込み
- [[session-search-and-sessiondb]] — 第 3 の永続化機構としてのセッション検索

## 関連ファイル

- `agent/prompt_builder.py` — ガイダンステキスト定義
- `tools/memory_tool.py` — Memory 実装
- `tools/skills_tool.py` — Skills 実装
- `tools/session_search_tool.py` — Session Search 実装
- `hermes_state.py` — SessionDB（FTS5 検索）
