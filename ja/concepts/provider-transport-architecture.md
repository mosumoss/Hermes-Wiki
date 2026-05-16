---
title: Provider Transport アーキテクチャ
created: 2026-04-18
updated: 2026-04-18
type: concept
tags: [architecture, module, provider, transport, api-dispatch]
sources: [agent/transports/base.py, agent/transports/anthropic.py, agent/transports/chat_completions.py, agent/transports/bedrock.py, agent/transports/codex.py, agent/transports/types.py, agent/transports/__init__.py, run_agent.py]
translation: ja
original: ../../concepts/provider-transport-architecture.md
---

# Provider Transport — API パスの統一抽象

## 概要

Provider Transport は **v2026.4.17+** で導入されたアーキテクチャレベルのリファクタリングで、すべての provider の API データパス（Anthropic Messages、OpenAI Chat Completions、OpenAI Responses API、AWS Bedrock）を統一 ABC で抽象化します。`agent/transports/`（1,217 行）に位置し、以前 `run_agent.py` のあちこちに散在していた `if api_mode == "anthropic_messages": ... elif ...` 分岐判断を置き換えます。

**コア理念**：**1 つの provider のメッセージ変換、ツール変換、パラメータ構築、応答正規化は、呼び出し箇所に散在するのではなく、1 つのクラスに集約されるべき。**

## アーキテクチャ原理

### 4 つの抽象メソッド + 3 つのオプションフック

```python
# agent/transports/base.py
class ProviderTransport(ABC):
    @property
    @abstractmethod
    def api_mode(self) -> str:
        """処理する api_mode 文字列（例：'anthropic_messages'）"""

    @abstractmethod
    def convert_messages(self, messages, **kwargs) -> Any:
        """OpenAI 形式メッセージ → provider ネイティブ形式"""

    @abstractmethod
    def convert_tools(self, tools) -> Any:
        """OpenAI ツール定義 → provider ネイティブ形式"""

    @abstractmethod
    def build_kwargs(self, model, messages, tools=None, **params) -> Dict:
        """完全な API 呼び出し kwargs を組み立てる（通常は内部で前 2 メソッドを呼ぶ）"""

    @abstractmethod
    def normalize_response(self, response, **kwargs) -> NormalizedResponse:
        """生の応答 → 共有の NormalizedResponse 型（transport 層型を返す唯一のメソッド）"""

    # ── オプションフック ───────────────────────────────────────────
    def validate_response(self, response) -> bool: ...       # 構造検証
    def extract_cache_stats(self, response) -> Optional[Dict]: ...  # cache hit/create 抽出
    def map_finish_reason(self, raw_reason) -> str: ...      # stop reason マッピング
```

**設計ポイント**：
- Transport は**データパスのみ担当**、client ライフサイクル、streaming、auth、credential refresh、retry、interrupt handling は管理しない — これらは `AIAgent` 側
- `normalize_response` は transport 層型（`NormalizedResponse`）を返す唯一のメソッド、他のメソッドは provider ネイティブ構造を返す

### 実装済み Transport

| Transport | ファイル | 行数 | api_mode | カバー範囲 |
|-----------|------|------|----------|------|
| `AnthropicTransport` | `transports/anthropic.py` | 177 | `anthropic_messages` | Claude（直接接続、Nous Portal） |
| `ChatCompletionsTransport` | `transports/chat_completions.py` | 387 | `chat_completions`、`openai` 等 | OpenAI、OpenRouter、Gemini、xAI、カスタム OpenAI 互換 |
| `ResponsesApiTransport` | `transports/codex.py` | 217 | `openai_responses` | OpenAI Codex、Responses API |
| `BedrockTransport` | `transports/bedrock.py` | 154 | `bedrock_converse` | AWS Bedrock（Converse API） |
| `NormalizedResponse` | `transports/types.py` | 142 | — | 共有応答型 |
| 基底クラス + レジストリ | `transports/base.py` + `__init__.py` | 89 + 51 | — | ABC + `get_transport()` 遅延発見 |

### レジストリ：遅延発見

```python
# agent/transports/__init__.py
def get_transport(api_mode: str) -> ProviderTransport:
    """対応する transport モジュールを必要時にインポート、モジュールレベルの register_transport() 呼び出しをトリガー"""
    ...

def register_transport(api_mode: str, transport_cls: type) -> None:
    """transport モジュールがインポート時に呼ぶ、自身を registry に登録"""
    ...
```

初回の `get_transport("anthropic_messages")` 呼び出し時に `transports/anthropic.py` をインポート — **実際の利用まで遅延**、起動時に大量の SDK インポートで遅くならない。

## run_agent.py での統合ポイント

`AnthropicTransport`、`ChatCompletionsTransport`、`BedrockTransport`、`ResponsesApiTransport` は `run_agent.py` 内で **20+ 箇所の provider アダプタ関数の直接呼び出し**を置き換え：

| シーン | 新メソッド |
|------|--------|
| メイン kwargs 構築（api_mode 別にディスパッチ） | `transport.build_kwargs(...)` |
| 記憶 flush（build_kwargs + normalize） | `_tflush.build_kwargs` / `_tfn.normalize_response` |
| イテレーション上限要約 + リトライ | `_tsum.build_kwargs` / `_tsum.normalize_response` |
| 応答構造検証 | `transport.validate_response` |
| finish reason マッピング（Anthropic stop_reason → OpenAI） | `transport.map_finish_reason` |
| 切り詰め応答の正規化 | `transport.normalize_response` |
| cache hit / create 統計抽出 | `transport.extract_cache_stats` |
| メイン normalize loop | `transport.normalize_response` |

すべての transport メソッド呼び出しパス下の adapter import は transport クラス内部に完全に収束、`run_agent.py` 自体は `anthropic_adapter` 等の関数を直接 import しなくなった。

**直接 adapter import の残存ゼロ**（transport メソッドの呼び出しパスにおいて）。

補助クライアント（`agent/auxiliary_client.py`）も transport に移行（compression、memory flush、session summarization パス）。

## 設計の優位性

### 旧アーキテクチャとの比較

| 観点 | 旧方式 | Transport ABC |
|------|--------|---------------|
| 分岐コード | `run_agent.py` に `if api_mode == ...` 判断が散在 | 単一ポイント `get_transport(api_mode)` |
| 新 provider 追加 | 複数箇所変更（変換、normalize、cache stats...） | transport サブクラスを 1 つ追加 |
| テスト | メッセージ / ツール変換を単独テストしにくい | 各メソッドを独立して単体テスト可能 |
| 循環依存 | 起こりやすい | ゼロ — transport は `base` / `types` のみ import |
| 起動オーバーヘッド | 全 SDK を eager import する可能性 | 遅延 import、必要時のみロード |

### 単一責任

- **Transport**：メッセージ / ツール形式変換 + 応答正規化
- **AIAgent**：client ライフサイクル、streaming、auth、retry、interrupt
- **Adapter**（旧コード）：保持、transport が内部で委譲、段階的に廃止

### マイグレーション状況

| Provider | Transport カバー | 状態 |
|----------|---------------|------|
| Anthropic | AnthropicTransport（`anthropic_adapter.py` に委譲） | 全パス完了 |
| Chat Completions（OpenAI 互換） | ChatCompletionsTransport | 全パス完了 |
| OpenAI Responses API（Codex） | ResponsesApiTransport | 全パス完了 |
| AWS Bedrock | BedrockTransport | 全パス完了 |
| Auxiliary Client（圧縮 / 記憶） | Transport に移行済み | 完了 |

## 他システムとの関係

- [[auxiliary-client-architecture]] — auxiliary_client は Transport に移行済み
- [[smart-model-routing]] — transport は api_mode ベースでディスパッチ、モデルルーティングと連携
- [[interrupt-and-fault-tolerance]] — 中断、retry は AIAgent 層、transport の責任ではない
- [[prompt-caching-optimization]] — cache 統計は `extract_cache_stats` フックで公開

## 関連ファイル

- `agent/transports/base.py`（89 行） — `ProviderTransport` ABC
- `agent/transports/types.py`（142 行） — `NormalizedResponse` 共有型
- `agent/transports/__init__.py`（51 行） — レジストリ + 遅延発見
- `agent/transports/anthropic.py`（177 行） — Anthropic Messages
- `agent/transports/chat_completions.py`（387 行） — Chat Completions
- `agent/transports/codex.py`（217 行） — OpenAI Responses API
- `agent/transports/bedrock.py`（154 行） — AWS Bedrock Converse
- `run_agent.py` — 10+ 統合ポイント
- `agent/auxiliary_client.py` — 補助パス移行済み
