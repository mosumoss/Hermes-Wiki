---
title: Prompt Caching 最適化アーキテクチャ
created: 2026-04-07
updated: 2026-04-08
type: concept
tags: [architecture, module, performance, cost-optimization, anthropic]
sources: [agent/prompt_caching.py, run_agent.py]
translation: ja
original: ../../concepts/prompt-caching-optimization.md
---

# Prompt Caching — Anthropic キャッシュ最適化アーキテクチャ

## 概要

Prompt Caching は `agent/prompt_caching.py`（2KB / 72 行）に実装され、**Anthropic `system_and_3` キャッシュ戦略**を実現します。複数ターン対話で入力 token コストを約 75% 削減します。

コア理念：**最大 4 つの cache_control ブレークポイント — システムプロンプト + 直近 3 つの非システムメッセージ。**

## アーキテクチャ原理

### system_and_3 戦略

Anthropic の prompt cache はメッセージ内で `cache_control` ブレークポイントをマークできます。ブレークポイント前の内容はキャッシュされ、後続リクエストがキャッシュにヒットすると非常に低い cache_read 費用（通常費用の約 10%）のみ課金されます。

Anthropic は**最大 4 ブレークポイント**に制限しています。Hermes の配分戦略：

| ブレークポイント | 位置 | キャッシュ内容 | 安定性 |
|---|---|---|---|
| 1 | システムプロンプト | アイデンティティ + プラットフォームヒント + スキルインデックス | 最高（全ターンで不変） |
| 2 | 末尾から 3 番目のメッセージ | 初期対話内容 | 高（直近 2 ターンで不変） |
| 3 | 末尾から 2 番目のメッセージ | 中期対話内容 | 中（直近 1 ターンで不変） |
| 4 | 最後のメッセージ | 最新の対話内容 | 低（毎ターン更新） |

### スライドウィンドウ機構

```
ターン 1: [システムプロンプト★] [ユーザー1★] [アシスタント1] [アシスタント2]
                                ↑BP2      ↑BP3        ↑BP4

ターン 2: [システムプロンプト★] [ユーザー1] [アシスタント1★] [ユーザー2★] [アシスタント2★]
                                          ↑新 BP2          ↑新 BP3      ↑新 BP4

ターン 3: [システムプロンプト★] [ユーザー1] [アシスタント1] [ユーザー2] [アシスタント2★] [ユーザー3★] [アシスタント3★]
                                                                  ↑新 BP2      ↑新 BP3      ↑新 BP4
```

★ = cache_control マーカー。新規リクエスト時、ブレークポイントウィンドウが後ろにスライド。

## コアコンポーネント

### 1. cache_control マーカーの注入

```python
def _apply_cache_marker(msg, cache_marker, native_anthropic=False):
    """
    すべてのメッセージ形式バリエーションを処理:
    
    1. tool ロール → native_anthropic モードでのみマーク
    2. 空コンテンツ → メッセージレベルで直接マーク
    3. 文字列コンテンツ → [{"type": "text", "text": ..., "cache_control": ...}] に変換
    4. リストコンテンツ → 最後の要素に cache_control を追加
    """
```

**設計考慮**：Anthropic API は複数のメッセージ形式（文字列、オブジェクトリスト、ツール結果）を受け入れます。`_apply_cache_marker` はすべての形式を統一処理します。

### 2. メイン関数

```python
def apply_anthropic_cache_control(
    api_messages,
    cache_ttl="5m",        # キャッシュ TTL: 5分または 1時間
    native_anthropic=False # ネイティブ Anthropic 形式を使うか
):
    """
    1. メッセージをディープコピー（元データは変更しない）
    2. marker 作成: {"type": "ephemeral"} または {"type": "ephemeral", "ttl": "1h"}
    3. システムプロンプトにブレークポイント追加（最初のメッセージなら）
    4. 後ろから前へ最大 3 つの非システムメッセージにブレークポイント追加
    5. マーク済みメッセージリストを返す
    """
```

### 3. ロール別の処理

| ロール | キャッシュ戦略 |
|---|---|
| system | 常にマーク（最も安定したキャッシュポイント） |
| tool | native_anthropic モードでのみメッセージレベルでマーク |
| assistant/user | content の最後の要素にマーク |

## TTL 設定

```python
marker = {"type": "ephemeral"}         # デフォルト: 5 分 TTL
marker = {"type": "ephemeral", "ttl": "1h"}  # 1 時間 TTL
```

**利用シーン**：
- **5m（デフォルト）**：高速連続対話に適、キャッシュヒット率高い
- **1h**：長時間の対話間隔に適、より高いキャッシュミスを許容

## コスト効果

仮定：システムプロンプト 2,000 tokens、各対話で平均 5,000 tokens：

| シナリオ | キャッシュなしコスト | キャッシュヒットコスト | 節約 |
|---|---|---|---|
| 単ターン（システムプロンプト + 1 メッセージ） | ~7,000 tokens × 価格 | ~2,000 tokens × cache_read + 5,000 × 通常 | ~70% |
| 10 ターン対話 | 10 × 7,000 = 70K tokens | ~2,000 × cache_read + (70K-2,000) × 通常 | ~75% |
| 50 ターン対話 | 50 × 7,000 = 350K tokens | ~2,000 × cache_read + (350K-2,000) × 通常 | ~85% |

## 設計の優位性

### キャッシュなしとの比較

| 観点 | キャッシュなし | Prompt Caching |
|---|---|---|
| システムプロンプトコスト | 毎回課金 | 初回のみ課金 |
| 初期対話コスト | 毎回課金 | ヒット時は cache_read のみ |
| レイテンシ | 影響なし | キャッシュヒット時に低減 |
| コード複雑度 | 低 | 72 行の純関数 |
| 適用範囲 | 全モデル | Anthropic モデルのみ |

### 純関数設計

```python
# クラス状態なし、AIAgent 依存なし
# メッセージリスト入力 → マーク済みメッセージリスト出力
# ディープコピーで元データの変更を防ぐ
```

これによりキャッシュロジックは独立してテスト可能で、メイン対話フローに影響しません。

## 統合ポイント

prompt caching は `run_agent.py` の `_build_api_kwargs()` で呼ばれます：
1. 完全なメッセージリストを構築
2. 現在の provider が Anthropic なら → `apply_anthropic_cache_control()` を呼ぶ
3. `developer_role` 切替が必要なら → system メッセージを developer ロールに変換
4. API に送信

## 他システムとの関係

- [[smart-model-routing]] — キャッシュコスト情報は models.dev から
- [[auxiliary-client-architecture]] — 補助モデルは prompt caching を使わない
- [[context-compressor-architecture]] — コンテキスト圧縮はメッセージ数を減らし、間接的にキャッシュブレークポイント位置に影響
