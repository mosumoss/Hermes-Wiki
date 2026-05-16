---
title: Auxiliary Client 補助クライアントアーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [architecture, module, component, agent, tool]
sources: [agent/auxiliary_client.py]
translation: ja
original: ../../concepts/auxiliary-client-architecture.md
---

# Auxiliary Client — 補助クライアントアーキテクチャ

## 概要

Auxiliary Client は `agent/auxiliary_client.py`（85KB / 2,127 行）に実装され、Hermes Agent の**補助 LLM クライアントルーター**です。すべての非主対話 LLM タスク（コンテキスト圧縮、セッション検索要約、視覚分析、Web 抽出、スキルスナップショット生成等）に統一されたプロバイダ解析と呼び出しインターフェースを提供します。

コア理念：**すべての補助タスクは同じプロバイダ解析チェーンを共有、各消費者がフォールバックロジックを重複実装することを回避。**

## アーキテクチャ原理

### 設計目標

補助タスクは主対話と異なります：
- **コスト敏感**：最高価のモデルは不要、高速で安価で十分
- **信頼性要求高**：1 プロバイダの料金切れで全機能停止は不可
- **マルチモーダル需要**：一部タスクは視覚能力が必要
- **非同期対応**：Web 抽出等は async 必要

Auxiliary Client は**多層 provider 解析 + 自動ダウングレード + クライアントキャッシュ**でこれらを解決。

### プロバイダ解析チェーン（Text タスク）

```
優先度（auto モード）:
  1. メインプロバイダ（アグリゲータでなければ）→ メインモデル認証情報を直接使用
  2. OpenRouter（OPENROUTER_API_KEY）
  3. Nous Portal（~/.hermes/auth.json のアクティブプロバイダ）
  4. カスタムエンドポイント（config.yaml model.base_url + OPENAI_API_KEY）
  5. Codex OAuth（Responses API、gpt-5.2-codex）
  6. ネイティブ Anthropic
  7. 直接 API Key プロバイダ（z.ai/GLM、Kimi/Moonshot、MiniMax 等）
  8. None → 機能利用不可
```

**重要設計**：ユーザーのメインプロバイダが Alibaba、DeepSeek、ZAI 等の非アグリゲータの場合、Auxiliary Client は**メインプロバイダの認証情報を直接使用**、追加の OpenRouter key 設定不要。これにより利用障壁が大幅に低減。

### プロバイダ解析チェーン（Vision タスク）

```
  1. メインプロバイダ（サポートされる視覚バックエンドなら）
  2. OpenRouter
  3. Nous Portal
  4. Codex OAuth（gpt-5.2-codex は vision サポート）
  5. ネイティブ Anthropic
  6. カスタムエンドポイント（ローカル視覚モデル: Qwen-VL, LLaVA, Pixtral）
  7. None
```

## コアコンポーネント

### 1. アダプタ層（Adapter Pattern）

Auxiliary Client 最大のアーキテクチャの特徴は**アダプタパターン** — 全ての異なる API 形式を統一して `client.chat.completions.create()` インターフェースとして表現する。

#### Codex Responses API アダプタ

```python
class _CodexCompletionsAdapter:
    """Drop-in shim: chat.completions.create() kwargs を受け取り、
    Codex Responses streaming API にルーティング"""

class CodexAuxiliaryClient:
    """OpenAI クライアント互換ラッパー、Codex Responses API 経由でルーティング"""
```

**変換詳細**：
- chat.completions の `content` 形式 → Responses API の `input` 形式
- `{"type": "text", "text": "..."}` → `{"type": "input_text", "text": "..."}`
- `{"type": "image_url", ...}` → `{"type": "input_image", ...}`
- ストリーミング応答 → output items + text deltas を収集 → chat.completions 形式を再構築
- ツール呼び出し対応（function_call）
- `get_final_response()` が空を返す時、ストリームイベントから補完

#### Anthropic Messages API アダプタ

```python
class _AnthropicCompletionsAdapter:
    """OpenAI クライアント互換ラッパー、ネイティブ Anthropic クライアントベース"""
```

`agent.anthropic_adapter` の `build_anthropic_kwargs` と `normalize_anthropic_response` で双方向変換を実現。

#### 非同期アダプタ

```python
class _AsyncCodexCompletionsAdapter:
    """asyncio.to_thread() で同期アダプタをラップ"""

class AsyncCodexAuxiliaryClient:
    """AsyncOpenAI.chat.completions.create() に合致する非同期ラッパー"""
```

### 2. 中央ルーター（resolve_provider_client）

```python
def resolve_provider_client(
    provider: str,          # "openrouter", "nous", "openai-codex", "auto"...
    model: str = None,      # モデルオーバーライド
    async_mode: bool = False,
    raw_codex: bool = False,
    explicit_base_url: str = None,
    explicit_api_key: str = None,
) -> Tuple[client, resolved_model]:
```

**単一エントリポイント**：すべての補助消費者はこの関数または公開補助関数経由でクライアントを取得すべき、認証環境変数の即席ルックアップは禁止。

### 3. 自動検出（_resolve_auto）

```python
def _resolve_auto():
    # Step 1: 非アグリゲータのメインプロバイダ → メインモデル直接使用
    main_provider = _read_main_provider()
    if main_provider not in {"openrouter", "nous"}:
        client, resolved = resolve_provider_client(main_provider, main_model)
        if client: return client, resolved
    
    # Step 2: アグリゲータ / ダウングレードチェーン
    for label, try_fn in _get_provider_chain():
        client, model = try_fn()
        if client: return client, model
```

**優位性**：先にメインプロバイダ使用（追加設定削減）、その後ダウングレードチェーン（信頼性保証）。

### 4. タスクレベル設定システム

```python
def _resolve_task_provider_model(task, provider, model, base_url, api_key):
    """
    優先度:
      1. 明示パラメータ（provider/model/base_url/api_key）
      2. 環境変数オーバーライド（AUXILIARY_{TASK}_*, CONTEXT_{TASK}_*）
      3. 設定ファイル（auxiliary.{task}.* または compression.*）
      4. "auto"（完全自動検出チェーン）
    """
```

**柔軟性**：各タスクは独立して provider、model、base_url、api_key を設定可能。

### 5. クライアントキャッシュとイベントループ管理

```python
_client_cache: Dict[tuple, tuple] = {}
_client_cache_lock = threading.Lock()
```

**キャッシュ戦略**：
- Key: `(provider, async_mode, base_url, api_key, loop_id)`
- 非同期クライアントは**イベントループ ID** を含む、クロスループ再利用によるデッドロックを防止
- ループクローズ検出時に古いキャッシュを自動クリーンアップ

**イベントループ安全防御**：
```python
def neuter_async_httpx_del():
    """AsyncHttpxClientWrapper.__del__ の aclose() スケジュール無効化
    
    AsyncOpenAI クライアントが GC される時、__del__ は prompt_toolkit の
    イベントループで aclose() をスケジュール、しかし基底 TCP transport は
    別のループに紐付いていて RuntimeError("Event loop is closed") トリガー
    """
    AsyncHttpxClientWrapper.__del__ = lambda self: None

def cleanup_stale_async_clients():
    """各 agent ループの後で古い非同期クライアントをクリーンアップ"""
    
def shutdown_cached_clients():
    """CLI 終了前に全キャッシュクライアントをクリーンアップ"""
```

これは Hermes Agent が **prompt_toolkit + async OpenAI SDK** 互換性問題を解決する重要コード。

### 6. 支払い / 割当枯渇の自動ダウングレード

```python
def _is_payment_error(exc: Exception) -> bool:
    """HTTP 402 と残高不足エラーを検出"""
    if status_code == 402: return True
    if "credits" in err or "insufficient funds" in err: return True
    if "can only afford" in err or "billing" in err: return True

def _try_payment_fallback(failed_provider, task):
    """失敗したプロバイダをスキップ、チェーンの次の利用可能なプロバイダを試す"""
```

**動作フロー**：
1. LLM API 呼び出し
2. max_tokens パラメータエラー遭遇 → max_completion_tokens で再試行
3. 支払いエラー（402 / 残高不足）遭遇 → 次の利用可能 provider に自動切替
4. ダウングレードをログに記録、ユーザーに通知

### 7. 公開 API

| 関数 | 用途 |
|---|---|
| `get_text_auxiliary_client(task)` | テキストタスク用同期クライアント取得 |
| `get_async_text_auxiliary_client(task)` | テキストタスク用非同期クライアント取得 |
| `get_vision_auxiliary_client()` | 視覚タスク用同期クライアント取得 |
| `get_async_vision_auxiliary_client()` | 視覚タスク用非同期クライアント取得 |
| `call_llm(task, messages, ...)` | 中央同期 LLM 呼び出しエントリ |
| `async_call_llm(task, messages, ...)` | 中央非同期 LLM 呼び出しエントリ |
| `extract_content_or_reasoning(response)` | 応答内容抽出、reasoning モデル対応 |
| `get_available_vision_backends()` | 現在利用可能な視覚バックエンドリスト取得 |
| `get_auxiliary_extra_body()` | provider 固有の extra_body 取得 |
| `auxiliary_max_tokens_param(value)` | 正しい max tokens パラメータ名を返す |

## 設計の優位性

### 分散方式との比較

| 観点 | 分散方式（各消費者が独立実装） | Auxiliary Client（集中式） |
|---|---|---|
| 認証ロジック | 各ファイルで env/config を独自に読む | 一箇所で解析、各所で使用 |
| Fallback | 各消費者が独自実装 | 統一されたダウングレードチェーン |
| 支払いダウングレード | 通常欠落 | 自動検出 + 切替 |
| クライアントキャッシュ | 接続を重複作成 | 共有キャッシュ、オーバーヘッド削減 |
| イベントループ安全 | 漏れやすい | 統一管理 |
| 新 provider 統合 | N ファイル変更必要 | try_* 関数を 1 つ追加するだけ |

### アダプタパターンの優位性

- **呼び出し側はゼロ意識**：context_compressor、web_tools、session_search はすべて `client.chat.completions.create()` を呼ぶだけ、基底が Chat Completions、Responses API、Messages API のどれかを意識する必要なし
- **テスト可能性**：各アダプタは独立してテスト可能
- **拡張性**：新 API 形式はアダプタクラスを 1 つ追加するだけ

## 設定と操作

### config.yaml 設定

```yaml
auxiliary:
  compression:
    provider: auto        # または openrouter, nous, custom
    model: gemini-3-flash
    timeout: 30
  vision:
    provider: auto
    model: claude-sonnet-4-5-20250514
  web_extract:
    provider: openrouter
    model: google/gemini-3-flash-preview
    api_key: sk-xxx
    base_url: https://custom-endpoint.com/v1
```

### 環境変数オーバーライド

```bash
# 特定タスクの provider 設定
export AUXILIARY_VISION_PROVIDER=anthropic
export AUXILIARY_COMPRESSION_MODEL=claude-haiku-4-5
export AUXILIARY_WEB_EXTRACT_BASE_URL=https://my-endpoint/v1
export AUXILIARY_WEB_EXTRACT_API_KEY=sk-xxx
```

### 利用可能な視覚バックエンドの確認

```python
from agent.auxiliary_client import get_available_vision_backends
print(get_available_vision_backends())
# 出力: ['openrouter', 'nous', 'anthropic']（設定に依存）
```

## 他システムとの関係

- [[context-compressor-architecture]] — `get_text_auxiliary_client("compression")` 使用
- [[tool-registry-architecture]] — web_tools と browser_tool は registry 経由で登録
- [[credential-pool-and-isolation]] — `load_pool()` 使用で認証情報取得
- [[prompt-builder-architecture]] — 補助クライアントは主対話プロンプト構築に参加しない
- [[model-tools-dispatch]] — model_tools.py が auxiliary_client 経由でサイドタスク処理
