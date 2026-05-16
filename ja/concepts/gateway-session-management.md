---
title: Gateway Session セッション管理アーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [architecture, module, component, gateway, session-store, multi-platform]
sources: [gateway/session.py, gateway/config.py]
translation: ja
original: ../../concepts/gateway-session-management.md
---

# Gateway Session — ゲートウェイセッション管理アーキテクチャ

## 概要

Gateway Session は `gateway/session.py`（44KB / 1,081 行）に実装され、ゲートウェイの**セッションライフサイクル**を管理します：セッションコンテキスト追跡、メッセージ永続化、リセット戦略評価、動的システムプロンプト注入。

コア理念：**各プラットフォーム / ユーザー / スレッドの組み合わせが独立したセッションを持ち、セッションはどこから来てどこへ向かうかを知っている。**

## アーキテクチャ原理

### コアデータモデル

```text
SessionSource（メッセージソース）
    ↓
SessionContext（完全なセッションコンテキスト）
    ↓
SessionEntry（セッション保管エントリ）
    ↓
SessionStore（セッションストアマネージャ）
```

### SessionSource — メッセージソースの記述

```python
@dataclass
class SessionSource:
    platform: Platform           # telegram, discord, slack, whatsapp...
    chat_id: str                 # チャット ID
    chat_name: Optional[str]     # チャット名
    chat_type: str               # "dm", "group", "channel", "thread"
    user_id: Optional[str]       # ユーザー ID
    user_name: Optional[str]     # ユーザー名
    thread_id: Optional[str]     # スレッド / トピック ID
    chat_topic: Optional[str]    # チャンネルトピック
    user_id_alt: Optional[str]   # Signal UUID 等の予備 ID
    chat_id_alt: Optional[str]   # Signal グループ内部 ID
```

**マルチプラットフォーム対応**：異なるプラットフォームは異なる ID 形式（Telegram は数値 ID、Signal は UUID + グループ内部 ID）を使用。SessionSource が統一抽象化。

### SessionKey 構築ルール

```python
def build_session_key(source, group_sessions_per_user=True, thread_sessions_per_user=False):
    """
    DM セッション:
    → agent:main:{platform}:dm:{chat_id}
    → agent:main:{platform}:dm:{chat_id}:{thread_id}（スレッド付き）
    
    グループセッション:
    → agent:main:{platform}:group:{chat_id}:{user_id}（ユーザー単位で隔離）
    → agent:main:{platform}:group:{chat_id}（共有セッション）
    
    スレッドセッション:
    → agent:main:{platform}:thread:{chat_id}:{thread_id}（デフォルト共有）
    → agent:main:{platform}:thread:{chat_id}:{thread_id}:{user_id}（per-user）
    """
```

**設計考慮**：
- DM セッション：チャットごとに隔離、プライベート対話の独立を保証
- グループセッション：デフォルトでユーザー単位隔離（各ユーザーが自分の対話を持つ）
- スレッドセッション：デフォルト共有（全参加者が同じ対話を見る）、`thread_sessions_per_user` で隔離有効化可能

### PII マスキング

```python
_PHONE_RE = re.compile(r"^\+?\d[\d\-\s]{6,}$")

def _hash_id(value: str) -> str:
    """確定的な 12 文字 16 進数ハッシュ"""
    return hashlib.sha256(value.encode()).hexdigest()[:12]

def _hash_sender_id(value: str) -> str:
    return f"user_{_hash_id(value)}"

def _hash_chat_id(value: str) -> str:
    """プラットフォームプレフィックスを保持: telegram:12345 → telegram:<hash>"""
    colon = value.find(":")
    if colon > 0:
        return f"{value[:colon]}:{_hash_id(value[colon+1:])}"
    return _hash_id(value)
```

**Discord 例外**：Discord は `<@user_id>` メンションシステムを使用、LLM は実 ID が必要でユーザーを @ する。よって Discord は `_PII_SAFE_PLATFORMS` に含まれない。

### SessionContext — 動的システムプロンプト注入

```python
def build_session_context_prompt(context, redact_pii=False):
    """
    システムプロンプトに注入するコンテキスト情報を生成:
    
    ## Current Session Context
    **Source:** Telegram (DM with lnisang La)
    **User:** lnisang La
    **Connected Platforms:** local, telegram: Connected ✓
    
    **Delivery options for scheduled tasks:**
    - "origin" → Back to this chat (lnisang La)
    - "local" → Save to local files only
    - "telegram" → Home channel (...)
    """
```

**プラットフォーム固有動作ヒント**：

```python
if platform == SLACK:
    "You do NOT have access to Slack-specific APIs..."
elif platform == DISCORD:
    "You do NOT have access to Discord-specific APIs..."
```

Agent が実行できない操作を約束してしまうのを防止。

### SessionStore — セッションストアマネージャ

```python
class SessionStore:
    def __init__(self, sessions_dir, config):
        # SQLite (SessionDB) を優先使用
        # JSONL ファイルにフォールバック
        self._db = SessionDB()  # 利用可能なら
```

**デュアルストレージ戦略**：
1. **SQLite**（優先）：`hermes_state.SessionDB` 経由、FTS5 全文検索対応
2. **JSONL**（フォールバック）：シンプルな JSON ファイルストア

### セッションリセット戦略

```python
def _is_session_expired(self, entry):
    """
    セッションが期限切れかチェック:
    1. アクティブなバックグラウンドプロセスがあるか確認（あれば期限切れにしない）
    2. プラットフォーム / チャットタイプのリセット戦略を取得
    3. idle タイムアウトまたは daily リセットをチェック
    """
```

**バックグラウンド期限切れ監視**：

```python
# セッション期限切れ時:
entry.was_auto_reset = True
entry.auto_reset_reason = "idle"  # または "daily"
entry.reset_had_activity = bool(entry.total_tokens > 0)
```

次のメッセージ到着時、ゲートウェイが通知を注入：

```
"⚠️ Previous session expired (idle for 24h). Starting fresh conversation."
```

### Token 追跡

```python
@dataclass
class SessionEntry:
    input_tokens: int = 0
    output_tokens: int = 0
    cache_read_tokens: int = 0
    cache_write_tokens: int = 0
    total_tokens: int = 0
    estimated_cost_usd: float = 0.0
    cost_status: str = "unknown"
    last_prompt_tokens: int = 0  # 圧縮プリチェック用
    memory_flushed: bool = False  # メモリフラッシュ標記（永続化）
```

### アトミック保存

```python
def _save(self):
    """tempfile + os.replace でアトミックに sessions.json を書き込む"""
    fd, tmp_path = tempfile.mkstemp(dir=sessions_dir, suffix=".tmp")
    with os.fdopen(fd, "w") as f:
        json.dump(data, f, indent=2)
        f.flush()
        os.fsync(f.fileno())
    os.replace(tmp_path, sessions_file)  # アトミック置換
```

**なぜアトミック書き込み**：ゲートウェイクラッシュ時に不完全な sessions.json が書かれるのを防ぐ。

## 設計の優位性

### セッション隔離の柔軟性

| シーン | デフォルト動作 | 設定可能 |
|---|---|---|
| DM | チャット単位で隔離 | 変更不可 |
| グループ | ユーザー単位で隔離 | group_sessions_per_user=False → 共有 |
| スレッド | 共有 | thread_sessions_per_user=True → ユーザー単位隔離 |

### シンプルなセッション管理との比較

| 観点 | シンプル方式 | Gateway Session |
|---|---|---|
| マルチプラットフォーム | 手動処理が必要 | SessionSource が統一抽象 |
| セッション隔離 | 固定戦略 | 設定可能（per-user / shared） |
| PII 保護 | なし | 自動ハッシュマスキング |
| コンテキスト注入 | なし | 動的システムプロンプト |
| リセット戦略 | なし | idle/daily 自動リセット |
| コスト追跡 | なし | token 使用量 + コスト推定 |
| 永続化 | メモリ | SQLite + JSON デュアルストレージ |

## 設定と操作

### セッションリセット戦略

```yaml
# config.yaml
gateway:
  reset_policy:
    dm: idle:24h        # DM 24 時間無活動でリセット
    group: daily        # グループは毎日リセット
    thread: idle:12h    # スレッドは 12 時間無活動でリセット
```

### セッション隔離

```yaml
gateway:
  group_sessions_per_user: true    # グループ内で各ユーザー独立セッション
  thread_sessions_per_user: false  # スレッド内では共有セッション（デフォルト）
```

### アクティブセッションの確認

```python
# ゲートウェイ内部 API 経由
store._entries  # Dict[session_key, SessionEntry]
```

## Agent 実行中の新メッセージ処理（gateway/run.py line 1920+）

同じ session の agent が実行中にユーザーが新メッセージを送った時の処理ロジック：

```text
同じ session で新メッセージ受信
    │
    ├── /stop         → ハード中断：interrupt + _running_agents ロック強制クリア、即座に session 解放
    ├── /reset /new   → 中断 + pending queue クリア（旧テキスト再生防止 #2170）→ reset 実行
    ├── /queue <text> → キュー追加：中断せず、現在のターン終了後に次ターンの入力として
    ├── /status       → 中断せず、現在の状態を返す
    ├── /model        → 拒否："Agent is running — wait or /stop first"
    ├── /approve /deny→ 中断をバイパス、承認ハンドラに直接ルーティング（agent は approval event でブロック中）
    ├── 写真          → キューに追加して中断せず、複数写真は同じ pending event に自動マージ
    └── 通常テキスト  → interrupt(event.text) + テキストを _pending_messages に追加
```

### 通常テキスト中断の完全フロー

```python
# gateway/run.py line 2033-2038
running_agent.interrupt(event.text)      # 中断シグナルを設定
if _quick_key in self._pending_messages:
    self._pending_messages[_quick_key] += "\n" + event.text  # 追加
else:
    self._pending_messages[_quick_key] = event.text          # 新規作成
```

agent は次のチェックポイントで中断シグナルを発見 → 現ターン停止 → pending テキストを新ターンの入力として処理継続。

### 跨 Session 完全隔離

`_running_agents` の key は `_quick_key`（platform + user_id + chat_id で構成）、異なる session は独立した key を持つ：

| シーン | 中断するか | 理由 |
|------|:---:|------|
| 同じチャットウィンドウで通常テキスト | ✅ | interrupt() で現在の agent を中断 |
| 同じチャットウィンドウで /queue | ❌ | キューで現在の完了待ち |
| 同じチャットウィンドウで写真 | ❌ | 自動でキュー追加・マージ |
| 異なるチャットウィンドウ / 異なるユーザー | ❌ | 異なる _quick_key、独立スレッド並列 |

異なる session の agent は `run_in_executor` でスレッドプール内実行、真に並列。

## 他システムとの関係

- [[messaging-gateway-architecture]] — Session はゲートウェイのコアコンポーネント
- [[multi-agent-architecture]] — 中断はサブ agent に伝播（`_active_children`）
- [[session-search-and-sessiondb]] — SQLite SessionDB が FTS5 検索を提供
- [[cron-scheduling]] — セッション origin が cron 配信ルーティングに使用される
- [[memory-system-architecture]] — 期限切れセッションが memory flush をトリガー
