---
title: Memory System Architecture
created: 2026-04-07
updated: 2026-04-29
type: concept
tags: [memory, architecture, module]
sources: [tools/memory_tool.py, agent/memory_manager.py, agent/memory_provider.py, agent/builtin_memory_provider.py, run_agent.py, agent/prompt_builder.py, plugins/memory/__init__.py]
translation: ja
original: ../../concepts/memory-system-architecture.md
---

# 記憶（メモリ）システムアーキテクチャ

## 概要

Hermes の記憶システムは**三層アーキテクチャ**で構成されます：ストレージ層（MemoryStore）、編成層（MemoryManager）、プラグイン層（MemoryProvider）。

```text
┌─────────────────────────────────────────────┐
│              run_agent.py                   │
│  (prefetch → 注入 → tool 拦截 → sync → flush) │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         MemoryManager (編成層)               │
│  「内蔵 + 外部 Provider は最大 1 つ」          │
│  ツール schema 統合 / ライフサイクルフック配信  │
└────────┬────────────────────┬───────────────┘
         │                    │
┌────────▼────────┐  ┌───────▼────────────────┐
│ BuiltinProvider │  │ External Provider (任意) │
│ MEMORY.md       │  │ honcho / mem0 / 8 種類    │
│ USER.md         │  └────────────────────────┘
│ MemoryStore     │
└─────────────────┘
```

## 1. ストレージ層：MemoryStore

ファイル：`tools/memory_tool.py`（561 行）

### デュアルファイルストレージ

- **`MEMORY.md`**（デフォルト上限 2,200 文字）— Agent の個人ノート（環境事実、プロジェクト規約、ツール特性）
- **`USER.md`**（デフォルト上限 1,375 文字）— ユーザープロファイル（嗜好、コミュニケーションスタイル、期待値）
- 保管パス：`{HERMES_HOME}/memories/`
- エントリ区切り：`§`（section sign）、複数行エントリをサポート

### フリーズスナップショット方式

最も重要な設計決定：

```text
セッション開始 → load_from_disk() → ファイル読み込み → _system_prompt_snapshot に スナップショット 取得
                                                          │
                                              スナップショットをシステムプロンプトに注入
                                              (セッション全体で不変)
                                                          │
セッション中の書き込み → ディスクファイル + memory_entries 更新 ─── システムプロンプトは変えない
                                                          │
次回セッション → 再度 load_from_disk() → 新スナップショット有効化
```

**なぜ？** システムプロンプトを安定させることで → Anthropic の prefix cache を無効化させない。書き込みは即時ディスクに永続化されるが、現在のセッションのシステムプロンプトには自身の書き込みが反映されません。

### アトミック書き込み + ファイルロック

```python
# アトミック書き込み：temp file + fsync + os.replace()
def _write_file(path, entries):
    fd, tmp_path = tempfile.mkstemp(dir=path.parent)
    os.fsync(f.fileno())
    os.replace(tmp_path, str(path))  # アトミック操作

# ファイルロック：独立した .lock ファイル + fcntl 排他ロック
def _file_lock(path):
    lock_path = path.with_suffix(path.suffix + ".lock")  # データファイル自体はロックしない
    fcntl.flock(fd, fcntl.LOCK_EX)
```

読み手は常に「完全な古いファイル」または「完全な新ファイル」を見ることになり、中間状態は存在しません。

### セキュリティスキャン

すべての書き込み内容は 12 種類の脅威パターン検出 + 不可視 Unicode 文字検出を通ります：

```python
_MEMORY_THREAT_PATTERNS = [
    # プロンプトインジェクション
    ("ignore previous instructions", "prompt_injection"),
    ("you are now", "role_hijack"),
    ("do not tell the user", "deception_hide"),
    ("act as if you have no restrictions", "bypass_restrictions"),
    # 機密漏洩
    ("curl ... $KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API", "exfil_curl"),
    ("wget ... $KEY|TOKEN|SECRET", "exfil_wget"),
    ("cat .env|credentials|.netrc|.pgpass|.npmrc|.pypirc", "read_secrets"),
    # バックドア
    ("authorized_keys|~/.ssh", "ssh_backdoor"),
    # ... 計 12 種類
]
```

### システムプロンプト書式

```text
════════════════════════════════════════════════════
MEMORY (your personal notes) [65% — 1,430/2,200 chars]
════════════════════════════════════════════════════
エントリ 1
§
エントリ 2
```

### MemoryStore のコア API

| メソッド | 動作 |
|------|------|
| `load_from_disk()` | ファイル読み込み → 重複除去 → フリーズスナップショット取得 |
| `add(target, content)` | セキュリティスキャン → 重複チェック → 文字数制限チェック → 追記 → 永続化 |
| `replace(target, old_text, new_content)` | 部分文字列で旧エントリにマッチ → 置換 → セキュリティスキャン → 永続化 |
| `remove(target, old_text)` | 部分文字列マッチ → 削除 → 永続化 |
| `format_for_system_prompt(target)` | **フリーズスナップショット**を返す（リアルタイム状態ではない） |

---

## 2. 編成層：MemoryManager

ファイル：`agent/memory_manager.py`（367 行）

### コア制約

```python
class MemoryManager:
    def __init__(self):
        self._providers: List[MemoryProvider] = []
        self._tool_to_provider: Dict[str, MemoryProvider] = {}  # ツール名 → provider ルーティング
        self._has_external: bool = False  # 外部 provider は最大 1 つ
```

**「内蔵 + 外部は最大 1 つ」ルール**：`add_provider()` は 2 つ目の builtin 以外の provider を拒否し、warning を出します。

### 編成メソッド

| メソッド | 動作 |
|------|------|
| `build_system_prompt()` | 全 provider の `system_prompt_block()` を収集して連結 |
| `prefetch_all(query)` | 全 provider の `prefetch()` 結果をマージ |
| `queue_prefetch_all(query)` | 全 provider に対して次ターンのコンテキストをバックグラウンド予取するよう通知 |
| `sync_all(user, assistant)` | 完了した turn を全 provider に同期 |
| `get_all_tool_schemas()` | 全 provider のツール schema をマージ（名前で重複排除） |
| `handle_tool_call(name, args)` | `_tool_to_provider` で正しい provider にルーティング |
| `has_tool(name)` | 指定ツールを処理する provider があるか確認 |

### ライフサイクルフック

すべてのフックは**全 provider にブロードキャスト**され、各 provider の失敗は隔離されます（try/except、伝播しない）：

| フック | 発火タイミング | 用途 |
|------|---------|------|
| `on_turn_start(turn_number, message)` | 毎ターン開始前 | ターンカウント、scope 管理 |
| `on_session_end(messages)` | セッション終了時 | 永続事実の抽出、flush キュー |
| `on_pre_compress(messages)` | コンテキスト圧縮前 | 圧縮で消えそうな情報をレスキュー |
| `on_memory_write(action, target, content)` | 内蔵 memory ツール書き込み後 | **外部 provider のみに通知**（builtin はスキップ）、ミラー書き込み |
| `on_delegation(task, result)` | サブエージェント完了後 | 親 Agent が委譲結果を観察 |

### Memory Context Fence

```python
def build_memory_context_block(raw_context: str) -> str:
    # <memory-context> タグで包むことで、モデルが召喚内容をユーザー入力と混同しないようにする
    return f"<memory-context>\n{sanitized}\n</memory-context>"
```

---

## 3. プラグイン層：MemoryProvider ABC

ファイル：`agent/memory_provider.py`（232 行）

### 抽象インターフェース

```python
class MemoryProvider(ABC):
    # 実装必須
    @abstractmethod
    def name(self) -> str: ...              # "builtin", "honcho", "mem0"
    @abstractmethod
    def is_available(self) -> bool: ...     # ネットワーク呼び出しなしの高速チェック
    @abstractmethod
    def initialize(self, session_id, **kwargs): ...  # セッション初期化
    @abstractmethod
    def get_tool_schemas(self) -> List[Dict]: ...    # LLM に公開するツール

    # 任意オーバーライド（デフォルト no-op）
    def system_prompt_block(self) -> str: ...     # システムプロンプトに注入する静的テキスト
    def prefetch(self, query) -> str: ...         # 毎ターン前の高速召喚
    def queue_prefetch(self, query): ...          # バックグラウンド予取
    def sync_turn(self, user, assistant): ...     # 完了 turn の永続化
    def handle_tool_call(self, name, args) -> str: ...  # ツール呼び出し処理
    def shutdown(self): ...                       # リソースクリーンアップ
    def on_turn_start(self, turn_number, message): ...
    def on_session_end(self, messages): ...
    def on_pre_compress(self, messages) -> str: ...
    def on_memory_write(self, action, target, content): ...
    def on_delegation(self, task, result): ...
    def on_session_switch(self, new_session_id, parent_session_id, reset, **kw): ...  # v2026.4.23+
```

### `on_session_switch` — Session ID 途中切替通知（v2026.4.23+）

以前は provider が初期化時に 1 回だけ `session_id` を受け取っていました。しかし `session_id` は `/resume`、`/branch`、`/reset`、`/new`、コンテキスト圧縮などのシーンで**再割り当て**されます — provider が気づかないと、後続の書き込みが誤ったセッション記録に保存されてしまいます。

`agent/memory_manager.py:on_session_switch()` は session_id が変わったときに全 provider の `on_session_switch(new_session_id, parent_session_id, reset, **kwargs)` を呼び出し、provider が per-session 状態キャッシュを更新できるようにします。Provider は tear down + 再構築する必要はなく、内部ハンドルを更新するだけで OK。エラーは swallow されます（log debug のみ、メインフローはブロックしない）。

### initialize() の kwargs

**常に提供**：`hermes_home`（HERMES_HOME パス）、`platform`（"cli" / "telegram" / "discord" ...）

**提供される可能性**：`agent_context`（"primary" / "subagent" / "cron" / "flush"）、`agent_identity`（profile 名）、`agent_workspace`（共有ワークスペース名）、`parent_session_id`（サブエージェントの親 session）、`user_id`（プラットフォームユーザー ID）

### 8 個の利用可能なプラグイン

| プラグイン          | パス                                           |
| ----------- | -------------------------------------------- |
| honcho      | `plugins/memory/honcho/` — Honcho AI 弁証法的ユーザーモデリング |
| mem0        | `plugins/memory/mem0/`                       |
| hindsight   | `plugins/memory/hindsight/`                  |
| holographic | `plugins/memory/holographic/`                |
| openviking  | `plugins/memory/openviking/`                 |
| retaindb    | `plugins/memory/retaindb/`                   |
| supermemory | `plugins/memory/supermemory/`                |
| byterover   | `plugins/memory/byterover/`                  |

プラグイン発見機構：`plugins/memory/` をスキャンし、`__init__.py` を含むサブディレクトリを見つけ、`is_available()` の高速チェックを実行。

---

## 4. Agent 統合フロー

### Memory ツールの特殊インターセプト

Memory ツールは**ツールレジストリ（tool registry）にありません**。`run_agent.py` で明示的にインターセプトされます：

```python
# run_agent.py:6078-6100 — 特殊分岐、registry.dispatch() を通らない
elif function_name == "memory":
    result = memory_tool(
        action=args.get("action"),
        target=args.get("target", "memory"),
        content=args.get("content"),
        old_text=args.get("old_text"),
        store=self._memory_store,
    )
    # 外部 provider にミラー書き込みを通知
    if self._memory_manager and args.get("action") in ("add", "replace"):
        self._memory_manager.on_memory_write(action, target, content)
```

**なぜ registry を通さない？** memory ツールは `self._memory_store` インスタンスへの直接アクセスが必要ですが、registry の handler シグネチャは agent 内部状態を渡さないためです。

### 完全なライフサイクル

```text
セッション開始
    │
    ├── MemoryStore.load_from_disk() → フリーズスナップショット
    ├── MemoryManager.add_provider(builtin)
    ├── MemoryManager.add_provider(honcho)  ← 設定されていれば
    ├── provider.initialize(session_id, hermes_home=..., platform=...)
    └── システムプロンプト = builtin.system_prompt_block() + external.system_prompt_block()

毎ターンの対話
    │
    ├── [API 呼び出し前]
    │   ├── prefetch_all(user_message) → 全 provider の召喚結果をマージ
    │   └── <memory-context> fence で包む → 現在のターンのユーザーメッセージに注入
    │       （一時的注入、元メッセージは変更しない、session には永続化しない）
    │
    ├── [ツール呼び出し]
    │   ├── "memory" → 特殊インターセプト → MemoryStore.add/replace/remove
    │   │                        → on_memory_write() で外部 provider に通知
    │   └── "honcho_*" 等 → MemoryManager.handle_tool_call() → 外部 provider にルーティング
    │
    └── [API 呼び出し後]
        ├── sync_all(user_message, assistant_response) → 全 provider に永続化
        └── queue_prefetch_all(user_message) → 次ターンのコンテキストをバックグラウンド予取

コンテキスト圧縮前
    │
    ├── flush_memories(messages) → モデルに重要情報を memory に書き込ませる
    └── on_pre_compress(messages) → 外部 provider に情報レスキューを通知

セッション終了
    │
    ├── on_session_end(messages) → 全履歴を provider に渡す
    └── shutdown_all() → リソースクリーンアップ
```

### バックグラウンド Memory Review

システムは 10 ターン毎（`_memory_nudge_interval`）に自動でバックグラウンド review をトリガー：

```python
# run_agent.py — ターンカウンタ
self._turns_since_memory += 1
if self._turns_since_memory >= 10:
    _should_review_memory = True  # メインループ終了後に _spawn_background_review() を発火
```

Review Agent は `_MEMORY_REVIEW_PROMPT` で対話履歴を見直し、自動的に memory ツールを呼んで永続事実を抽出します。

---

## 5. LLM から見える Memory ツール

```python
# tools/memory_tool.py:489-538 — ツール schema
{
    "name": "memory",
    "description": "Save durable facts about the user or environment...",
    "parameters": {
        "action": "add | replace | remove",
        "target": "memory | user",
        "content": "追加 / 置換する内容",
        "old_text": "マッチさせる旧テキスト（replace / remove 時必須）"
    }
}
```

システムプロンプト内のガイダンス（`prompt_builder.py:144-156`）：

```text
MEMORY_GUIDANCE:
- ユーザー嗜好、環境詳細、ツール特性、安定した規約を保存
- 「ユーザーが将来訂正する回数を減らす」情報を優先
- 保存しないもの：タスク進捗、セッション結果、完了作業ログ、一時 TODO
- 新しい方法を発見？skill ツールで保存、memory は使わない
```

---

## 6. 設定

```yaml
# config.yaml
memory:
  memory_enabled: true           # MEMORY.md 有効化（デフォルト false）
  user_profile_enabled: true     # USER.md 有効化（デフォルト false）
  memory_char_limit: 2200        # MEMORY.md 文字数上限
  user_char_limit: 1375          # USER.md 文字数上限
  nudge_interval: 10             # 何ターンごとにバックグラウンド memory review を発火
  flush_min_turns: 6             # 圧縮前、最低何ターン経過すれば flush 許可
  provider: honcho               # 外部 provider 名（任意）
```

---

## 7. 設計のポイント

### 失敗の隔離

MemoryManager の各 provider メソッド呼び出しは try/except に包まれます。1 つの provider がクラッシュしても他に影響せず、Agent 実行をブロックしません。

### 内蔵 memory 書き込みのミラー

LLM が `memory(action="add", target="user", content="ユーザーはダークモード好み")` を呼んだとき：
1. MemoryStore が `USER.md` に書き込み（ローカルファイル）
2. `on_memory_write("add", "user", "ユーザーはダークモード好み")` で外部 provider に通知
3. 外部 provider（例：Honcho）はこの事実を自身のバックエンドに同期可能

**add と replace のみがミラーをトリガー、remove はしない。**

### 予取キャッシュ

`prefetch_all()` は毎回の API 呼び出し前に 1 回呼ばれ、結果は `_ext_prefetch_cache` にキャッシュされます。同一ターン内の複数 tool call では再予取しません（10 回のツール呼び出し = 10 倍の遅延を回避）。

---

## 8. FAQ

### Q1：フリーズスナップショット下で、現在のセッションが書き込んだばかりの記憶を見るには？

**ツール返り値で補う。** 各 `memory(action="add/replace/remove")` 呼び出し後、返り値には**リアルタイム全エントリ**が含まれます：

```json
{
  "success": true,
  "entries": ["エントリ1", "エントリ2", "今追加したエントリ3"],
  "usage": "65% — 1,430/2,200 chars",
  "entry_count": 3
}
```

モデルは対話コンテキストの中で最新内容を既に見られるため、システムプロンプトの更新は不要です。

```text
Turn 1:  システムプロンプトにフリーズスナップショット [エントリ1, エントリ2] を含む
Turn 3:  LLM が memory(add, "エントリ3") を呼ぶ
         → 返り値に [エントリ1, エントリ2, エントリ3] を含む  ← モデルから見える
Turn 5:  LLM が memory(replace, old="エントリ1", new="更新したエントリ1") を呼ぶ
         → 返り値に [更新したエントリ1, エントリ2, エントリ3] を含む

システムプロンプトは常に [エントリ1, エントリ2] を表示  ← 凍結不変、prefix cache 保護
対話コンテキストには完全なリアルタイム状態が存在         ← 機能は影響なし
```

### Q2：文字数制限を超えたらどうなる？

**ハード拒否 + 現在のエントリを返してモデル自身に空間管理させる。** 自動淘汰なし、LRU なし、溢れなし。

```json
{
  "success": false,
  "error": "Memory at 2,100/2,200 chars. Adding this entry (200 chars) would exceed the limit. Replace or remove existing entries first.",
  "current_entries": ["エントリ1", "エントリ2", "エントリ3"],
  "usage": "2,100/2,200"
}
```

**設計意図**：Memory はデータベースではなく、**注意深くキュレートされた小さなカードボックス**。空間を制限することで、モデルにキュレーション行為を強制 — 古いものを replace、重要でないものを remove、新発見を add。

### Q3：より大量の履歴情報はどうする？

Memory には**永続事実**のみ保存（2,200 + 1,375 文字）。より大量の履歴情報は **session_search ツール**で検索します。

両者の分業は明確で、システムプロンプトはモデルに明示的にガイダンス：

```text
MEMORY_GUIDANCE:
  "Do NOT save task progress, session outcomes, completed-work logs...
   use session_search to recall those from past transcripts."
```

---

## 9. Session Search との関係

Session Search は Memory の一部ではありませんが、Memory システムの**補完機構**です。

|          | Memory ツール          | Session Search ツール                        |
| -------- | ------------------ | ---------------------------------------- |
| **保管対象** | 永続事実（嗜好、環境、規約）     | 全履歴対話の原文                                 |
| **容量**   | 2,200 + 1,375 文字（有限） | 無限（SQLite、全セッション）                          |
| **検索方式** | 検索なし（システムプロンプトに直接注入）      | FTS5 キーワード検索 + LLM 要約                      |
| **書き込み側**  | LLM が能動的に memory ツールを呼ぶ | 自動（毎ターン対話を SQLite に自動永続化）                    |
| **読み取りコスト** | ゼロ（システムプロンプトにフリーズスナップショット）      | FTS5 クエリ + Gemini Flash 要約（各セッション 1 回 LLM 呼び出し） |

### Session Search のワークフロー

```text
session_search(query="nginx 設定")
        │
  ┌─────▼──────┐
  │ FTS5 検索   │  BM25 ソート、top 50 件のマッチメッセージを取得
  └─────┬──────┘
        │
  session ごとにグループ化 → 重複除去 → 現セッションを除外 → top 3 を取得
        │
  ┌─────▼──────────────────┐
  │ LLM 要約（並列）         │  各セッションのマッチ位置 ±50K 文字を抜粋
  │ Gemini Flash, temp=0.1  │  検索語にフォーカスした構造化要約を生成
  └─────┬──────────────────┘
        │
  per-session 要約を返す（原対話テキストではない）
```

**注意**：Session Search は FTS5 キーワードマッチングであり、セマンティックベクトル検索ではありません。「nginx 設定」を検索しても、「リバースプロキシ」とだけ書かれたセッションにはマッチしません。検索構文は `OR`、`NOT`、`"完全フレーズ"`、`前方一致*` をサポート。

2 つのモード：
- **空 query** → 直近セッションを一覧（LLM コストゼロ、タイトル / プレビュー / タイムスタンプを返すだけ）
- **クエリあり** → FTS5 検索 + 並列 LLM 要約（最大 3-5 セッション）

## 関連ページ

- [[memory-system-architecture]] — MemoryStore コアクラス詳細 API（本ページ）
- [[security-defense-system]] — 記憶内容のセキュリティスキャン
- [[skills-and-memory-interaction]] — スキルと記憶の相互作用決定木
- [[context-compressor-architecture]] — 圧縮前の flush_memories と on_pre_compress
- [[prompt-caching-optimization]] — フリーズスナップショットがどう prefix cache を保護するか
- [[session-search-and-sessiondb]] — Session Search ツール（FTS5 + LLM 要約）

## 関連ファイル

- `tools/memory_tool.py` — MemoryStore クラス + memory ツール schema（561 行）
- `agent/memory_manager.py` — MemoryManager 編成層（367 行）
- `agent/memory_provider.py` — MemoryProvider ABC インターフェース（232 行）
- `agent/builtin_memory_provider.py` — 内蔵 Provider（114 行）
- `plugins/memory/` — 8 個の外部 Provider プラグイン
- `run_agent.py` — Agent 統合（ツールインターセプト、prefetch、sync、flush）
- `agent/prompt_builder.py` — MEMORY_GUIDANCE システムプロンプト
- `tools/session_search_tool.py` — Session Search ツール（FTS5 + LLM 要約、505 行）
