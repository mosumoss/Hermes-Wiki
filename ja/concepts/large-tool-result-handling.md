---
title: 大型ツール結果処理とコンテキスト保護
created: 2026-04-07
updated: 2026-04-11
type: concept
tags: [architecture, context-management, performance]
sources: [tools/tool_result_storage.py, tools/budget_config.py, run_agent.py]
translation: ja
original: ../../concepts/large-tool-result-handling.md
---

# 大型ツール結果処理とコンテキスト保護

## 設計原理

ツールは大型の結果（`search_files` がコードベース全体を検索、`terminal` が長い出力コマンドを実行する等）を返すことがあります。対話履歴に直接入れるとコンテキストウィンドウを急速に消費します。Hermes は**知的ファイル化機構**を実装し、大型結果をディスクに保存してプレビューのみ保持します。

## 3 層オーバーフロー防御

大型ツール結果は 3 層機構で段階的に防御（`tools/tool_result_storage.py` + `tools/budget_config.py`）：

```text
Layer 1: ツール内切り詰め        — 各ツールが出力を事前切り詰め（search_files 等）
Layer 2: 単一結果永続化         — 100K 文字超 → sandbox ディスクへ書き出し、context には 1.5K プレビューのみ保持
Layer 3: ターン集約予算         — 1 ターン全結果合計 200K 超 → 最大のものをディスクへオーバーフロー
```

### 閾値設定（`tools/budget_config.py`）

```python
DEFAULT_RESULT_SIZE_CHARS  = 100_000   # Layer 2: 単一結果永続化閾値
DEFAULT_TURN_BUDGET_CHARS  = 200_000   # Layer 3: ターン集約上限
DEFAULT_PREVIEW_SIZE_CHARS = 1_500     # 永続化後のインラインプレビューサイズ

# read_file は ∞ に固定、「永続化 → 読み込み → 再永続化」の無限ループ防止
PINNED_THRESHOLDS = {"read_file": float("inf")}
```

閾値解決優先度：`PINNED_THRESHOLDS > tool_overrides > registry per-tool > default`

### Layer 2: 単一結果永続化（`maybe_persist_tool_result()`）

ツール返却後、出力が閾値超過時：
1. `env.execute()` で完全結果を sandbox の `/tmp/hermes-results/{tool_use_id}.txt` に書き出し
2. context 内容を `<persisted-output>` タグに置換、1,500 文字プレビュー + ファイルパス含む
3. agent は `read_file` で完全出力にアクセス可能
4. sandbox 書き込み失敗時はインライン切り詰めにフォールバック

### Layer 3: ターン集約予算（`enforce_turn_budget()`）

1 ターン内で複数の中サイズ結果合計が 200K 文字超過時：
- 未永続化の結果をサイズ降順に並べる
- 1 つずつディスクへオーバーフロー、総量が予算以下になるまで

この層は「単一は超過しないが合計で超過する」シーンをキャッチ。

## コンテキストウィンドウ保護

### プリフライト圧縮

```python
# メインループ進入前、ロードした対話履歴が既にコンテキスト閾値超過か確認
if (
    self.compression_enabled
    and len(messages) > self.context_compressor.protect_first_n
                    + self.context_compressor.protect_last_n + 1
):
    # ツール schema tokens を含む — 多ツール時 20-30K+ tokens 増加可能性
    _preflight_tokens = estimate_request_tokens_rough(
        messages,
        system_prompt=active_system_prompt or "",
        tools=self.tools or None,
    )
    
    if _preflight_tokens >= self.context_compressor.threshold_tokens:
        # API エラー待たず能動的に圧縮
        for _pass in range(3):  # 最大 3 ラウンド
            _orig_len = len(messages)
            messages, active_system_prompt = self._compress_context(...)
            if len(messages) >= _orig_len:
                break  # これ以上圧縮不能
            if _preflight_tokens < self.context_compressor.threshold_tokens:
                break  # 既に閾値以下
```

### 413 エラー処理

```python
is_payload_too_large = (
    status_code == 413
    or 'request entity too large' in error_msg
    or 'payload too large' in error_msg
)

if is_payload_too_large:
    compression_attempts += 1
    if compression_attempts > max_compression_attempts:
        return {"error": "Request payload too large: max compression attempts reached."}
    
    # 圧縮後再試行
    messages, active_system_prompt = self._compress_context(...)
    if len(messages) < original_len:
        time.sleep(2)  # 圧縮後の短い一時停止
        restart_with_compressed_messages = True
        break
```

### コンテキスト長エラー検出

```python
is_context_length_error = any(phrase in error_msg for phrase in [
    'context length', 'context size', 'maximum context',
    'token limit', 'too many tokens', 'reduce the length',
    'exceeds the limit', 'context window',
    'request entity too large',  # OpenRouter/Nous 413 セーフティネット
    'prompt is too long',  # Anthropic
    'prompt exceeds max length',  # Z.AI / GLM
])

# ヒューリスティック：Anthropic は時々汎用 400 エラーを返す
if not is_context_length_error and status_code == 400:
    ctx_len = getattr(self.context_compressor, 'context_length', 200000)
    is_large_session = approx_tokens > ctx_len * 0.4 or len(api_messages) > 80
    is_generic_error = len(error_msg.strip()) < 30
    if is_large_session and is_generic_error:
        is_context_length_error = True  # コンテキストオーバーフロー扱い

# サーバー切断もコンテキスト過大の可能性
if not is_context_length_error and not status_code:
    _is_server_disconnect = (
        'server disconnected' in error_msg
        or 'peer closed connection' in error_msg
    )
    if _is_server_disconnect and approx_tokens > ctx_len * 0.6:
        is_context_length_error = True  # コンテキストオーバーフロー扱い
```

### 429 長コンテキスト階層エラー

```python
# Anthropic は 429 "Extra usage is required for long context requests" を返す
# Claude Max サブスクが 1M コンテキスト階層を含まない時
_is_long_context_tier_error = (
    status_code == 429
    and "extra usage" in error_msg
    and "long context" in error_msg
    and "sonnet" in self.model.lower()
)

if _is_long_context_tier_error:
    _reduced_ctx = 200000  # 標準階層 200K にダウングレード
    compressor.context_length = _reduced_ctx
    compressor.threshold_tokens = int(_reduced_ctx * compressor.threshold_percent)
    # 永続化しない — これはサブスク階層制限、モデル能力ではない
    compressor._context_probe_persistable = False
```

## プロキシ安全書き込み

```python
class _SafeWriter:
    """透過的 stdio ラッパー、broken pipe の OSError/ValueError をキャッチ"""
    
    def write(self, data):
        try:
            return self._inner.write(data)
        except (OSError, ValueError):
            return len(data) if isinstance(data, str) else 0
    
    def flush(self):
        try:
            self._inner.flush()
        except (OSError, ValueError):
            pass

def _install_safe_stdio() -> None:
    """stdout/stderr をラップ、ベストエフォートのコンソール出力で Agent がクラッシュしないように"""
    for stream_name in ("stdout", "stderr"):
        stream = getattr(sys, stream_name, None)
        if stream is not None and not isinstance(stream, _SafeWriter):
            setattr(sys, stream_name, _SafeWriter(stream))
```

**なぜ必要？**
- systemd サービス / Docker コンテナで stdout/stderr パイプが利用不可な場合
- サブエージェントスレッド終了後、共有 stdout ハンドルが閉じている可能性
- `OSError: [Errno 5] Input/output error` での Agent クラッシュを防ぐ

## Surrogate 文字クリーンアップ

```python
_SURROGATE_RE = re.compile(r'[\ud800-\udfff]')

def _sanitize_surrogates(text: str) -> str:
    """孤立した surrogate コードポイントを U+FFFD（置換文字）に置換"""
    if _SURROGATE_RE.search(text):
        return _SURROGATE_RE.sub('�', text)
    return text

# surrogate は UTF-8 で無効、OpenAI SDK の json.dumps() をクラッシュさせる
def _sanitize_messages_surrogates(messages: list) -> bool:
    """メッセージリスト全文字列内の surrogate 文字をクリーンアップ"""
    found = False
    for msg in messages:
        content = msg.get("content")
        if isinstance(content, str) and _SURROGATE_RE.search(content):
            msg["content"] = _SURROGATE_RE.sub('�', content)
            found = True
    return found
```

**なぜ必要？**
- クリップボード貼り付けのリッチテキスト（Google Docs、Word）が孤立 surrogate を注入する可能性
- JSON シリアライズクラッシュにつながる

## 予算警告クリーンアップ

```python
_BUDGET_WARNING_RE = re.compile(
    r"\[BUDGET(?:\s+WARNING)?:\s+Iteration\s+\d+/\d+\..*?\]",
    re.DOTALL,
)

def _strip_budget_warnings_from_history(messages: list) -> None:
    """ツール結果メッセージから予算圧迫警告を削除"""
    for msg in messages:
        if not isinstance(msg, dict) or msg.get("role") != "tool":
            continue
        content = msg.get("content")
        if not isinstance(content, str) or "_budget_warning" not in content and "[BUDGET" not in content:
            continue
        
        # JSON 解析を試す（一般的ケース）
        try:
            parsed = json.loads(content)
            if isinstance(parsed, dict) and "_budget_warning" in parsed:
                del parsed["_budget_warning"]
                msg["content"] = json.dumps(parsed, ensure_ascii=False)
                continue
        except (json.JSONDecodeError, TypeError):
            pass
        
        # フォールバック：プレーンテキストツール結果からパターン削除
        cleaned = _BUDGET_WARNING_RE.sub("", content).strip()
        if cleaned != content:
            msg["content"] = cleaned
```

**なぜ必要？**
- 予算警告は**ターンスコープ**シグナル、リプレイ履歴に漏れるべきではない
- GPT 系列モデルはこれをまだアクティブな指示として解釈し、後続全ターンでツール呼び出しを避けてしまう

## 優位性分析

### コンテキスト節約

| シーン | 保護なし | 保護あり | 節約 |
|------|--------|--------|------|
| 大型検索出力 | 100K chars | 1.5K + ファイル参照 | ~98.5% |
| 長ターミナル出力 | 50K chars | 1.5K + ファイル参照 | ~97% |
| プリフライト圧縮 | API エラー待ち | 能動的圧縮 | 失敗回避 |

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | OpenCode |
|------|--------|--------|----------|
| 大型結果ファイル化 | ✅ 自動 | ✅ 自動 | ❌ 切り詰め |
| 設定可能閾値 | ✅ BudgetConfig | ❌ 固定 | N/A |
| プリフライト圧縮 | ✅ | ✅ | ❌ |
| Surrogate クリーンアップ | ✅ | ❌ | ❌ |
| 予算警告クリーンアップ | ✅ | N/A | N/A |
| 安全 stdio | ✅ | N/A | N/A |

## 関連ページ

- [[context-compressor-architecture]] — コンテキスト圧縮とプリフライト圧縮機構
- [[parallel-tool-execution]] — 並列ツール実行が大型結果を生成するシーン
- [[model-tools-dispatch]] — ツール結果は統一形式で処理される

## 関連ファイル

- `tools/tool_result_storage.py` — 3 層オーバーフロー防御（Layer 2 + Layer 3）
- `tools/budget_config.py` — 閾値設定と優先度解決
- `run_agent.py` — Surrogate クリーンアップ、予算警告クリーンアップ
- `agent/context_compressor.py` — コンテキスト圧縮
