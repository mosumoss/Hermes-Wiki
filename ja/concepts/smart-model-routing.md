---
title: Smart Model Routing 知的モデルルーティング
created: 2026-04-08
updated: 2026-04-29
type: concept
tags: [architecture, module, model-routing, performance, caching, anthropic]
sources: [agent/model_metadata.py, agent/models_dev.py, hermes_cli/model_switch.py, hermes_cli/model_normalize.py]
translation: ja
original: ../../concepts/smart-model-routing.md
---

# Smart Model Routing — 知的モデルルーティング

## 概要

> **注**：本ページは**複数モジュール**の協調をカバーします、`agent/smart_model_routing.py` のみではありません。`smart_model_routing.py` 自体は約 195 行の軽量ヒューリスティックモジュールで、cheap/strong メッセージルーティング（現在のメッセージを安いモデル or 強いモデルで処理するか決定）を担当。本ページが議論するより広範なモデルインフラ — メタデータ解析、コンテキスト長探査、モデル切替パイプライン — は以下 4 コアモジュールに分散：

Smart Model Routing は Hermes Agent の**モデルメタデータ解析と自動コンテキスト長検出**システム、4 つのコアモジュールで構成：

| モジュール | ソース | 責任 |
|---|---|---|
| **model_metadata.py** | 36KB / 941 行 | コンテキスト長検出、エンドポイント探査、token 推定 |
| **models_dev.py** | 25KB / 781 行 | models.dev 4,000+ モデルデータベース統合 |
| **model_switch.py** | 32KB / 927 行 | モデル切替パイプライン（エイリアス解決 → 認証情報 → メタデータ） |
| **model_normalize.py** | 外部モジュール | 各プロバイダのモデル名正規化 |

コア理念：**10 段階コンテキスト長解決チェーン + models.dev 4,000+ モデルデータベース + ローカルサーバー自動探査。**

## アーキテクチャ原理

### コンテキスト長解決チェーン（10 段階）

```python
def get_model_context_length(model, base_url, api_key, config_context_length, provider):
    """
    0. config 明示オーバーライド → ユーザーがベストを知る
    1. 永続化キャッシュ（以前探査した model@base_url）
    2. アクティブエンドポイントメタデータ（/models エンドポイント、カスタムエンドポイント限定）
    3. ローカルサーバークエリ（Ollama/LM Studio/vLLM/llama.cpp）
    4. Anthropic /v1/models API（API Key のみ、OAuth 除外）
    5. models.dev レジストリ（プロバイダ認識、Nous サフィックスマッチ含む）
    6. OpenRouter リアルタイム API メタデータ
    7. ハードコードデフォルト（ファジーマッチ、最長 key 優先）
    8. ローカルサーバー最終試行
    9. デフォルトフォールバック: 128K
    """
```

**設計哲学**：最も精密から最も緩い方向へ、各段階失敗時のみ次へ。

### ローカルサーバー自動探査

```python
def detect_local_server_type(base_url):
    """
    探査順序:
    1. LM Studio → /api/v1/models（最も特定）
    2. Ollama → /api/tags（response が "models" を含むことを検証）
    3. llama.cpp → /v1/props または /props（default_generation_settings チェック）
    4. vLLM → /version（"version" フィールドチェック）
    """
```

各サーバータイプは異なるメタデータ取得方式を持つ：

| サーバー | エンドポイント | コンテキスト長ソース |
|---|---|---|
| Ollama | /api/show | model_info.context_length または num_ctx パラメータ |
| LM Studio | /api/v1/models | loaded_instances.config.context_length |
| vLLM | /v1/models/{model} | max_model_len |
| llama.cpp | /v1/props | n_ctx（実際に割り当てたコンテキスト） |

### エンドポイントメタデータ取得

```python
def fetch_endpoint_model_metadata(base_url, api_key):
    """
    1. {base_url}/models と {base_url}/v1/models を試す
    2. 各モデルの context_length、max_completion_tokens、pricing を解析
    3. llama.cpp なら → /v1/props を追加クエリして実 n_ctx 取得
    4. 5 分間キャッシュ
    """
```

### 永続化キャッシュ

```python
# キャッシュ key: model@base_url
# 同名モデルが異なるプロバイダで提供される時、異なる制限を持つ可能性
def save_context_length(model, base_url, length):
    # ~/.hermes/context_length_cache.yaml に書き込み
    # 形式: {context_lengths: {"qwen3@http://localhost:11434/v1": 131072}}
```

### エラーメッセージからのコンテキスト長抽出

```python
def parse_context_limit_from_error(error_msg):
    """
    API エラーメッセージから実コンテキスト制限を抽出:
    - "maximum context length is 32768 tokens"
    - "context_length_exceeded: 131072"
    - "250000 tokens > 200000 maximum"
    """
```

## コアコンポーネント

### 1. models.dev 統合

```python
# 4,000+ モデル、109+ プロバイダ
# オフライン優先: パックスナップショット → ディスクキャッシュ → ネットワーク取得 → バックグラウンドリフレッシュ（60 分）

@dataclass
class ModelInfo:
    id: str
    name: str
    family: str
    provider_id: str
    reasoning: bool
    tool_call: bool
    attachment: bool       # 視覚サポート
    context_window: int
    max_output: int
    cost_input: float      # 100 万 token あたり
    cost_output: float
    cost_cache_read: float
    # ... 他のフィールド
```

**3 段階キャッシュ**：
1. **メモリキャッシュ**：1 時間 TTL
2. **ディスクキャッシュ**：`~/.hermes/models_dev_cache.json`
3. **ネットワーク取得**：`https://models.dev/api.json`

### 2. モデル能力クエリ

```python
def get_model_capabilities(provider, model) -> ModelCapabilities:
    """
    返却:
    - supports_tools: ツール呼び出しサポートか
    - supports_vision: 視覚サポートか
    - supports_reasoning: 推論サポートか
    - context_window: コンテキストウィンドウ
    - max_output_tokens: 最大出力
    - model_family: モデルファミリー
    """
```

### 3. モデル切替システム

```python
def switch_model(raw_input, current_provider, current_model, ...) -> ModelSwitchResult:
    """
    2 つのパス:
    
    A. --provider 指定あり:
       1. プロバイダ解決 → 認証情報解決 → エイリアス解決または原形使用
       2. モデルなし → エンドポイントから自動検出
    
    B. --provider 指定なし:
       1. 現在のプロバイダでエイリアス試行
       2. エイリアス存在するが現プロバイダになし → 他の認証済みプロバイダにフォールバック
       3. アグリゲータ → vendor/model slug 変換
       4. アグリゲータディレクトリ検索
       5. detect_provider_for_model() で兜底
       6. 認証情報解決 → モデル名正規化
    """
```

### 4. エイリアスシステム

```python
MODEL_ALIASES = {
    "sonnet":  ModelIdentity("anthropic", "claude-sonnet"),
    "opus":    ModelIdentity("anthropic", "claude-opus"),
    "gpt5":    ModelIdentity("openai", "gpt-5"),
    "gemini":  ModelIdentity("google", "gemini"),
    "qwen":    ModelIdentity("qwen", "qwen"),
    # ... 20+ 短いエイリアス
}
```

エイリアス解決は**動的** — models.dev カタログクエリでマッチする最新モデルバージョンを見つける、ハードコードではない。

### 5. Provider プレフィックス処理

```python
_PROVIDER_PREFIXES = frozenset({
    "openrouter", "nous", "openai-codex", "anthropic", "alibaba",
    "google", "glm", "kimi", "deepseek", "qwen", ...
})

def _strip_provider_prefix(model):
    """
    "local:my-model" → "my-model"
    "qwen3.5:27b" → "qwen3.5:27b"（Ollama tag 保持）
    "deepseek:latest" → "deepseek:latest"（Ollama tag 保持）
    """
```

**重要**：provider プレフィックスと Ollama の model:tag 形式を区別。

### 6. 知的ファジーマッチ

コンテキスト長デフォルトは**最長 key 優先**のファジーマッチ：

```python
DEFAULT_CONTEXT_LENGTHS = {
    "claude-sonnet-4.6": 1000000,   # 特定バージョン
    "claude": 200000,               # 兜底（必ず後ろに並べる）
    "gpt-5": 128000,
    "gemini": 1048576,
    "qwen": 131072,
    # ...
}

# default_model in model のみチェック（逆方向ではない）
# "claude-sonnet-4" が "claude-sonnet-4-6" に誤マッチするのを回避
```

### 7. コンテキスト探査ダウングレード

```python
CONTEXT_PROBE_TIERS = [128_000, 64_000, 32_000, 16_000, 8_000]

def get_next_probe_tier(current_length):
    """128K から開始、エラー遭遇時に段階的ダウングレード"""
```

### 8. Token 推定

```python
def estimate_tokens_rough(text):
    """~4 chars/token の粗推定"""
    return len(text) // 4

def estimate_request_tokens_rough(messages, system_prompt, tools):
    """
    完全リクエスト推定、以下を含む:
    - システムプロンプト
    - 対話メッセージ
    - ツール schemas（50+ ツールで 20-30K tokens に達する可能性）
    """
```

## 設計の優位性

### ハードコード方式との比較

| 観点 | ハードコード | Smart Model Routing |
|---|---|---|
| 新モデルサポート | コード更新が必要 | models.dev で自動更新 |
| ローカルサーバー | 手動設定 | 4 サーバータイプを自動探査 |
| コンテキスト長 | 静的辞書 | 10 段階解決チェーン（0-9） |
| 認証情報管理 | ハードコード | runtime_provider で解決 |
| エラー復旧 | なし | エラーメッセージから制限抽出 |
| オフラインサポート | なし | パックスナップショット + ディスクキャッシュ |

## 設定と操作

### 明示的オーバーライド

```yaml
# config.yaml
model:
  context_length: 128000  # 全検出を直接オーバーライド
```

### エイリアス拡張

```yaml
# config.yaml
model_aliases:
  qwen:
    model: "qwen3.5:397b"
    provider: custom
    base_url: "https://ollama.com/v1"
```

## 価格推定

```python
# agent/usage_pricing.py

def estimate_usage_cost(model: str, prompt_tokens: int, completion_tokens: int) -> float:
    """API 呼び出しコスト推定"""
    pricing = {
        "claude-opus-4.6": {"input": 15.0, "output": 75.0},  # $/MTok
        "claude-sonnet-4": {"input": 3.0, "output": 15.0},
        "gpt-4o": {"input": 2.5, "output": 10.0},
        # ...
    }
    
    prices = pricing.get(model, {"input": 5.0, "output": 15.0})
    input_cost = (prompt_tokens / 1_000_000) * prices["input"]
    output_cost = (completion_tokens / 1_000_000) * prices["output"]
    return input_cost + output_cost
```

## OpenRouter プロバイダルーティング

```python
# プロバイダ嗜好
provider_preferences = {}
if self.providers_allowed:
    provider_preferences["order"] = self.providers_allowed
if self.providers_ignored:
    provider_preferences["ignore"] = self.providers_ignored
if self.providers_order:
    provider_preferences["order"] = self.providers_order
if self.provider_sort:
    provider_preferences["sort"] = self.provider_sort

# OpenRouter に送信
extra_body["provider"] = provider_preferences
```

### プロバイダソートオプション

```python
# sort オプション
"sort": "price"       # 価格順
"sort": "throughput"  # スループット順
"sort": "latency"     # レイテンシ順
```

## メタデータキャッシュ

```python
# OpenRouter モデルメタデータキャッシュ（1 時間 TTL）
_model_metadata_cache: dict = {}
_metadata_cache_time: float = 0
_METADATA_CACHE_TTL = 3600  # 1 時間

def fetch_model_metadata(model: str = None) -> dict:
    """モデルメタデータ取得（キャッシュ付き）"""
    now = time.time()
    if now - _metadata_cache_time < _METADATA_CACHE_TTL:
        return _model_metadata_cache
    
    # バックグラウンドスレッドでキャッシュをプリウォーム
    threading.Thread(
        target=lambda: fetch_model_metadata(),
        daemon=True,
    ).start()
```

## 推論モデルサポート

```python
def _supports_reasoning_extra_body(self) -> bool:
    """reasoning extra_body を安全に送信できるか判定"""
    
    # 直接 Nous Portal
    if "nousresearch" in self._base_url_lower:
        return True
    
    # OpenRouter ルーティング
    if "openrouter" not in self._base_url_lower:
        return False
    
    # 推論サポートが知られているモデルプレフィックス
    reasoning_model_prefixes = (
        "deepseek/",
        "anthropic/",
        "openai/",
        "x-ai/",
        "google/gemini-2",
        "qwen/qwen3",
    )
    return any(self.model.lower().startswith(prefix) for prefix in reasoning_model_prefixes)
```

## セッション状態追跡

```python
# 累積 token 使用量
self.session_prompt_tokens = 0
self.session_completion_tokens = 0
self.session_total_tokens = 0
self.session_api_calls = 0
self.session_input_tokens = 0
self.session_output_tokens = 0
self.session_cache_read_tokens = 0
self.session_cache_write_tokens = 0
self.session_reasoning_tokens = 0
self.session_estimated_cost_usd = 0.0
self.session_cost_status = "unknown"
self.session_cost_source = "none"

def reset_session_state(self):
    """全セッションレベル token カウンタをリセット"""
    self.session_total_tokens = 0
    self.session_input_tokens = 0
    self.session_output_tokens = 0
    # ... 全カウンタリセット
    self._user_turn_count = 0
```

## 新規 Provider（v0.10.0、2026-04-16）

### AWS Bedrock（ネイティブ Converse API）

デュアルパスアーキテクチャ（`agent/bedrock_adapter.py`、1,098 行）：
- **Claude モデル** → AnthropicBedrock SDK（prompt caching、thinking budgets 保持）
- **非 Claude モデル** → Converse API via boto3（Nova、DeepSeek、Llama、Mistral）

特性：
- IAM credential chain + Bedrock API Key の 2 認証モード
- `ListFoundationModels` + `ListInferenceProfiles` で動的モデル発見
- Streaming + delta callbacks + guardrails
- `/usage` 価格表示で 7 つの Bedrock モデルサポート
- `hermes doctor` + `hermes auth` 統合

### Google Gemini CLI OAuth

Cloud Code Assist バックエンド（`cloudcode-pa.googleapis.com`）経由で Gemini にアクセス、Google 公式 `gemini-cli` と同じバックエンドを使用。

`agent/` 配下に 2 つの新モジュール：
- `google_oauth.py`（1,048 行）：PKCE Authorization Code flow、プロセス間ファイルロック（fcntl POSIX / msvcrt Windows）、refresh token 自動更新、並行リフレッシュ重複排除
- `gemini_cloudcode_adapter.py`：provider 登録、モデル発見、streaming

無料層（個人アカウント日次クォータ）と有料層（Standard/Enterprise via GCP project）両対応。

### Ollama Cloud

内蔵 provider として登録（gemini、xai 等と同等）：
- `OLLAMA_API_KEY` 環境変数で認証
- Provider エイリアス：`ollama` → custom（ローカル）、`ollama_cloud` → ollama-cloud
- models.dev 統合で正確なコンテキスト長取得
- 動的モデル発見 + ディスクキャッシュ（1 時間 TTL）
- Ollama `model:tag` 形式を保持（正規化しない）

### MiniMax OAuth（v2026.4.23+）

新規 `minimax-oauth` 一等市民 provider、PKCE device-code flow を使用（`openclaw/extensions/minimax/oauth.ts` から移植）。`hermes_cli/auth.py` に追加：

- 8 つの `MINIMAX_OAUTH_*` 定数（client ID、scope、grant type、global/CN base URLs、inference URLs、refresh skew）
- `auth_type="oauth_minimax"` provider タイプ、device-code/external OAuth と並列
- エイリアス：`minimax-portal` / `minimax-global` / `minimax_oauth`
- 標準 OAuth2 refresh_token grant で自動更新、`invalid_grant` / `refresh_token_reused` で relogin トリガー
- MiniMax-M2.7 モデルと統合（`agent/minimax_oauth_provider.py`）

### Step Plan（v2026.4.18+）

StepFun 初の API-key provider（Step Plan）、国際版と中国版両対応。`/step_plan/v1/models` から動的モデル発見、オフライン時はハードコードフォールバックカタログあり。

### Vercel AI Gateway（v2026.4.18+）

新規 `ai-gateway` provider（エイリアス `vercel-ai-gateway`）追加、Vercel AI Gateway 経由で複数モデルに統一アクセス：
- カスタムモデルリスト（`hermes_cli/models.py` の `VERCEL_AI_GATEWAY_MODELS`、OSS 優先、Kimi K2.5 推奨デフォルト）
- ライブ価格翻訳（Vercel input/output → prompt/completion 形式）
- 無料 Moonshot モデルを picker 先頭に自動配置
- プロバイダ picker ソート優先度向上
- Vercel の deep-link で API key 作成

### OpenRouter ツールサポートフィルタ（v2026.4.18+）

hermes-agent はツール呼び出し優先 agent、`tools` をサポートするモデルのみが agent ループを駆動可能。`fetch_openrouter_models()` は `supported_parameters` に `tools` を明示的に含まないモデル（純画像、completion-only 等）をフィルタアウト。

寛容モード：`supported_parameters` 欠落時はデフォルト許可（Nous Portal、プライベートミラー、旧 snapshot は未記入の可能性）。明示宣言があり `tools` を含まないモデルのみ非表示。

### Tool Gateway（Nous サブスクリプション制ツールゲートウェイ）

web 検索、TTS、ブラウザ、画像生成等のツール API 呼び出しを Nous ホストの統一ゲートウェイにルーティング、ユーザーは各社 API key を持参不要：

```yaml
# config.yaml — ツールカテゴリ別 opt-in
web:
  use_gateway: true
tts:
  use_gateway: true
image_gen:
  use_gateway: true
browser:
  use_gateway: true
```

- `managed_nous_tools_enabled()` で Nous ログイン状態 + サブスクリプション階層をチェック
- `prefers_gateway(section)` 共有補助関数、4 ツールランタイムで統一使用
- `hermes model` 対話フロー：Nous ログイン後、利用可能ツールリスト表示、ユーザーが全有効化 / 未設定のみ / スキップを選択
- 無料層ユーザーにアップグレード提示を表示

## 他システムとの関係

- [[context-compressor-architecture]] — `get_model_context_length()` でコンテキスト制限を決定
- [[prompt-caching-optimization]] — キャッシュコスト情報は models.dev から
- [[auxiliary-client-architecture]] — 補助モデルは models.dev でコンテキスト長解決
