---
title: 中断伝播と耐障害機構
created: 2026-04-07
updated: 2026-04-15
type: concept
tags: [architecture, reliability, fault-tolerance, interrupt]
sources: [run_agent.py, gateway/run.py, agent/error_classifier.py, tools/credential_pool.py]
translation: ja
original: ../../concepts/interrupt-and-fault-tolerance.md
---

# 中断伝播と耐障害機構

## 設計原理

Agent は長時間実行タスク（複数ツール呼び出し、サブエージェント委譲）を実行することがあります。ユーザーは以下が可能である必要があります：
1. **現在の操作を中断** — 新メッセージ送信または Ctrl+C
2. **失敗の優雅な処理** — API エラー、ネットワーク切断、認証期限切れ
3. **自動復旧** — リトライ、フォールバック、認証情報ローテーション

Hermes は**多層中断と耐障害機構**を実装。

## 中断機構

### 中断フラグ

```python
class AIAgent:
    def __init__(self):
        self._interrupt_requested = False
        self._interrupt_message = None
    
    @property
    def is_interrupted(self) -> bool:
        """中断要求されたか確認"""
        return self._interrupt_requested
    
    def clear_interrupt(self):
        """中断状態をクリア"""
        self._interrupt_requested = False
        self._interrupt_message = None
```

### サブエージェントへの中断伝播

```python
# 親エージェントは全サブエージェントを中断可能
def _propagate_interrupt(self):
    with self._active_children_lock:
        for child in self._active_children:
            child._interrupt_requested = True
```

### API 呼び出し中断

```python
def _interruptible_api_call(self, api_kwargs: dict):
    """API 呼び出しをバックグラウンドスレッドで実行、メインループが中断検出可能に"""
    
    result = {"response": None, "error": None}
    request_client_holder = {"client": None}
    
    def _call():
        try:
            if self.api_mode == "codex_responses":
                request_client_holder["client"] = self._create_request_openai_client(...)
                result["response"] = self._run_codex_stream(...)
            elif self.api_mode == "anthropic_messages":
                result["response"] = self._anthropic_messages_create(api_kwargs)
            else:
                request_client_holder["client"] = self._create_request_openai_client(...)
                result["response"] = request_client_holder["client"].chat.completions.create(**api_kwargs)
        except Exception as e:
            result["error"] = e
        finally:
            # リクエストクライアントをクリーンアップ
            request_client = request_client_holder.get("client")
            if request_client is not None:
                self._close_request_openai_client(request_client, reason="request_complete")
    
    t = threading.Thread(target=_call, daemon=True)
    t.start()
    
    while t.is_alive():
        t.join(timeout=0.3)  # 300ms ごとに中断チェック
        if self._interrupt_requested:
            # 進行中の HTTP 接続を強制クローズ
            try:
                if self.api_mode == "anthropic_messages":
                    self._anthropic_client.close()
                    self._anthropic_client = build_anthropic_client(...)
                else:
                    request_client = request_client_holder.get("client")
                    if request_client is not None:
                        self._close_request_openai_client(request_client, reason="interrupt_abort")
            except Exception:
                pass
            raise InterruptedError("Agent interrupted during API call")
    
    if result["error"] is not None:
        raise result["error"]
    return result["response"]
```

### メインループでの中断チェック

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    # 中断リクエストをチェック
    if self._interrupt_requested:
        interrupted = True
        if not self.quiet_mode:
            self._safe_print("⚠️ Interrupted by user")
        break
    
    # ... 通常処理
```

### ストリーミング API 呼び出し中断

```python
def _interruptible_streaming_api_call(self, api_kwargs: dict, ...):
    """ストリーミング版、リアルタイム token 配信対応"""
    
    for chunk in stream:
        if self._interrupt_requested:
            break  # ストリーム受信停止
        
        # ... chunk 処理
    
    # クリーンアップ
    if self._interrupt_requested:
        raise InterruptedError("Agent interrupted during streaming")
```

## 耐障害機構

### 認証情報プールローテーション

```python
def _recover_with_credential_pool(self, *, status_code, has_retried_429, ...):
    """認証情報プールローテーションで復旧試行"""
    
    pool = self._credential_pool
    if pool is None or status_code is None:
        return False, has_retried_429
    
    if status_code == 402:
        # 請求枯渇 — 即座にローテーション
        next_entry = pool.mark_exhausted_and_rotate(status_code=402, ...)
        if next_entry is not None:
            self._swap_credential(next_entry)
            return True, False
    
    if status_code == 429:
        if not has_retried_429:
            return False, True  # 1 回目の 429、同じ認証情報でリトライ
        # 2 回目の 429、次の認証情報にローテーション
        next_entry = pool.mark_exhausted_and_rotate(status_code=429, ...)
        if next_entry is not None:
            self._swap_credential(next_entry)
            return True, False
    
    if status_code == 401:
        # 現在の認証情報リフレッシュ試行
        refreshed = pool.try_refresh_current()
        if refreshed is not None:
            self._swap_credential(refreshed)
            return True, has_retried_429
        # リフレッシュ失敗 — 次の認証情報にローテーション
        next_entry = pool.mark_exhausted_and_rotate(status_code=401, ...)
        if next_entry is not None:
            self._swap_credential(next_entry)
            return True, False
    
    return False, has_retried_429
```

### Fallback モデルチェーン

```python
# 設定例
fallback_chain:
  - model: "anthropic/claude-opus-4.6"
    provider: "anthropic"
  - model: "openai/gpt-4o"
    provider: "openrouter"
  - model: "google/gemini-2.5-pro"
    provider: "openrouter"

def _try_activate_fallback(self):
    """次の fallback モデルをアクティベート"""
    if self._fallback_index >= len(self._fallback_chain):
        return False  # これ以上 fallback なし
    
    fallback = self._fallback_chain[self._fallback_index]
    self._fallback_index += 1
    
    # モデル / 認証情報切替
    self.model = fallback["model"]
    self.provider = fallback["provider"]
    # ... クライアント再構築
    
    return True
```

### 構造化エラー分類（error_classifier.py）

2026-04-09 導入の集中エラー分類器、`run_agent.py` の分散文字列マッチを置き換え。すべての API エラーは 13 種類の `FailoverReason` に分類され、各々が異なる復旧戦略に対応：

| エラータイプ | 復旧戦略 |
|---------|---------|
| `auth` | 認証情報リフレッシュ / ローテーション |
| `billing` | 即座に Provider 切替 |
| `rate_limit` | バックオフ待機後ローテーション |
| `context_overflow` | コンテキスト圧縮 |
| `payload_too_large` | payload 圧縮 |
| `timeout` | クライアント再構築 + リトライ |
| `model_not_found` | 他のモデルに fallback |
| `server_error` / `overloaded` | リトライ / バックオフ |
| `thinking_signature` | Anthropic thinking block 署名無効 |
| `long_context_tier` | 200K 標準階層にダウングレード |

分類結果は構造化 `ClassifiedError`、復旧ヒントを含む：

```python
@dataclass
class ClassifiedError:
    reason: FailoverReason
    retryable: bool = True
    should_compress: bool = False
    should_rotate_credential: bool = False
    should_fallback: bool = False
```

リトライループはこれらのフィールドを直接読んで判断、エラーメッセージを再解析しない。

### 接続ヘルスチェック

```python
def _cleanup_dead_connections(self) -> bool:
    """プロバイダ障害による死 TCP 接続を検出してクリーンアップ"""
    
    # 共有接続プール内の死接続チェック
    cleaned = 0
    for conn in self._connection_pool:
        if not conn.is_healthy():
            conn.close()
            cleaned += 1
    
    return cleaned > 0

# 各ターン対話開始前にチェック
if self.api_mode != "anthropic_messages":
    try:
        if self._cleanup_dead_connections():
            self._emit_status(
                "🔌 Detected stale connections from a previous provider "
                "issue — cleaned up automatically."
            )
    except Exception:
        pass
```

### 認証情報自動リフレッシュ

```python
def _try_refresh_nous_client_credentials(self, *, force: bool = True) -> bool:
    """Nous Portal 認証情報リフレッシュ"""
    try:
        creds = resolve_nous_runtime_credentials(
            min_key_ttl_seconds=max(60, int(os.getenv("HERMES_NOUS_MIN_KEY_TTL_SECONDS", "1800"))),
            timeout_seconds=float(os.getenv("HERMES_NOUS_TIMEOUT_SECONDS", "15")),
            force_mint=force,
        )
    except Exception:
        return False
    
    api_key = creds.get("api_key")
    base_url = creds.get("base_url")
    if not isinstance(api_key, str) or not api_key.strip():
        return False
    
    self.api_key = api_key.strip()
    self.base_url = base_url.strip().rstrip("/")
    self._client_kwargs["api_key"] = self.api_key
    self._client_kwargs["base_url"] = self.base_url
    
    return self._replace_primary_openai_client(reason="nous_credential_refresh")

def _try_refresh_anthropic_client_credentials(self) -> bool:
    """Anthropic 認証情報リフレッシュ（OAuth token ローテーション）"""
    if self.api_mode != "anthropic_messages" or self.provider != "anthropic":
        return False
    
    try:
        new_token = resolve_anthropic_token()
    except Exception:
        return False
    
    if not isinstance(new_token, str) or not new_token.strip():
        return False
    if new_token == self._anthropic_api_key:
        return False  # 変化なし
    
    self._anthropic_client.close()
    self._anthropic_client = build_anthropic_client(new_token, self._anthropic_base_url)
    self._anthropic_api_key = new_token
    
    # OAuth フラグ更新 — token タイプが変わった可能性
    self._is_anthropic_oauth = _is_oauth_token(new_token)
    return True
```

## 活動追跡

```python
# ゲートウェイタイムアウトハンドラと「まだ動作中」通知用
self._last_activity_ts: float = time.time()
self._last_activity_desc: str = "initializing"
self._current_tool: str | None = None
self._api_call_count: int = 0

def _touch_activity(self, description: str):
    """活動タイムスタンプ更新"""
    self._last_activity_ts = time.time()
    self._last_activity_desc = description

def get_status(self) -> dict:
    """現在の状態取得（タイムアウト検出用）"""
    elapsed = time.time() - self._last_activity_ts
    return {
        "last_activity_ts": self._last_activity_ts,
        "last_activity_desc": self._last_activity_desc,
        "seconds_since_activity": round(elapsed, 1),
        "current_tool": self._current_tool,
        "api_call_count": self._api_call_count,
        "budget_used": self.iteration_budget.used,
        "budget_max": self.iteration_budget.max_total,
    }
```

## Gateway 再起動後の自動継続（2026-04-14）

Gateway プロセスが agent のツール呼び出し**後、最終応答生成前**に再起動されると（SIGTERM、クラッシュ、`drain_timeout`）、session transcript は `role: "tool"` で停止 — 以前はユーザーが手動で `/retry`（最初から再生、全進捗喪失）または "continue" と言う必要がありました。現在は次のユーザーメッセージ到達時、Gateway が履歴末尾が tool result であることを検出し、自動的に system note を注入：

```
[System note: Your previous turn was interrupted before you could process the
last tool result(s). The conversation history contains tool outputs you haven't
responded to yet. Please finish processing those results and summarize what was
accomplished, then address the user's new message below.]

<ユーザーの新メッセージ原文>
```

### 実装（`gateway/run.py:8679-8692`）

```python
# Auto-continue: ロードした履歴が tool result で終わっている場合、
# 前回の agent ターンは作業中に中断された
if agent_history and agent_history[-1].get("role") == "tool":
    message = SYSTEM_NOTE + "\n\n" + message
```

注入ポイントは `_run_agent()` の run_sync closure 内、`agent.run_conversation()` の**直前**。Agent は完全な履歴（未処理 tool results を含む）+ この system note を見て、続行 — まず以前の作業を要約し、その後ユーザーの新メッセージを処理。

### 設計の重要ポイント

| 設計決定 | 説明 |
|---|---|
| **schema 変更なし** | session flags や永続化フィールドを追加せず、純粋に末尾メッセージのロール検出 |
| **全再起動シーンに適用** | Clean shutdown / crash / SIGTERM / drain timeout すべてカバー |
| **ユーザーメッセージを保持** | ユーザー元メッセージは system note の後に残る、失われない |
| **suspended session はトリガーしない** | session が suspended 状態（異常クローズではない）なら、履歴は破棄、ユーザーは空白から、古い内容での誤 auto-continue を回避 |
| **シャットダウン通知文言変更** | シャットダウン時の通知が "Use /retry after restart to continue" から "Send any message after restart to resume where it left off" に — これが今の正確な動作 |

### 旧動作との比較

```text
旧フロー（手動 /retry）:
  ユーザー: "deploy v2.3"
  agent: [calls terminal "kubectl apply"] → [tool result: "deployment started"]
  [Gateway クラッシュ / 再起動]
  ユーザー: "did it work?"
  → ユーザーが手動 /retry 入力で対話再生 → agent が kubectl apply を最初から実行（重複デプロイ可能性！）

新フロー（auto-continue）:
  ユーザー: "deploy v2.3"
  agent: [calls terminal "kubectl apply"] → [tool result: "deployment started"]
  [Gateway クラッシュ / 再起動]
  ユーザー: "did it work?"
  → Gateway が末尾 tool 検出 → system note 注入 → agent が tool result + ユーザー新メッセージを見る
  → agent: "デプロイは開始されました（kubectl apply 成功）。質問について..."
```

**重要なセキュリティ**：旧フローの `/retry` は副作用を再生（kubectl apply を再実行）；新フローは agent に**既に発生した** tool result を解釈させるだけ、重複実行しない。

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | OpenCode |
|------|--------|--------|----------|
| ユーザー中断 | ✅ Ctrl+C / 新メッセージ | ✅ | ✅ |
| サブエージェント中断伝播 | ✅ | N/A | N/A |
| 認証情報プールローテーション | ✅ 複数キー自動ローテーション | ❌ | ❌ |
| Fallback モデルチェーン | ✅ 自動切替 | ❌ | ❌ |
| 接続ヘルスチェック | ✅ 自動クリーンアップ | ❌ | ❌ |
| 認証情報自動リフレッシュ | ✅ OAuth/token | ❌ | ❌ |
| 活動追跡 | ✅ タイムアウト検出 | ❌ | ❌ |

## 設定ガイド

### 環境変数

```bash
# Nous 認証情報リフレッシュ
HERMES_NOUS_MIN_KEY_TTL_SECONDS=1800  # 最小キー TTL
HERMES_NOUS_TIMEOUT_SECONDS=15        # リフレッシュタイムアウト

# ストリーミング読み取りタイムアウト
HERMES_STREAM_READ_TIMEOUT=60.0       # ストリーミング読み取りタイムアウト（秒）
HERMES_API_TIMEOUT=1800.0             # API 合計タイムアウト（秒）
```

## 関連ページ

- [[credential-pool-and-isolation]] — 認証情報プールとローテーション機構
- [[multi-agent-architecture]] — サブエージェント中断伝播と予算隔離
- [[agent-loop-and-prompt-assembly]] — AIAgent 中断フラグとメインループ

### 関連ファイル

- `agent/error_classifier.py` — 構造化 API エラー分類（13 種 FailoverReason）
- `run_agent.py` — 中断機構、リトライループ
- `tools/credential_pool.py` — 認証情報プール
- `tools/interrupt.py` — 中断ツール
