---
title: Web Tools 検索 / 抽出アーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [tool, toolset, architecture, component]
sources: [tools/web_tools.py]
translation: ja
original: ../../concepts/web-tools-architecture.md
---

# Web Tools — 検索 / 抽出アーキテクチャ

## 概要

Web Tools は `tools/web_tools.py`（88KB / 2,099 行）に実装され、**複数バックエンドの Web 検索 / 抽出 / クロール**機能を提供します。4 種類のバックエンドプロバイダをサポート、すべて Agent には同一の `web_search`、`web_extract`、`web_crawl` ツールインターフェースを公開。

コア理念：**コンテンツ取得をブラウザ自動化より優先** — 単純な情報検索は web_search / web_extract（高速・低コスト）、対話が必要な時のみ browser ツール。

## アーキテクチャ原理

### 4 大バックエンド

| バックエンド | Search | Extract | Crawl | 認証 |
|---|---|---|---|---|
| **Firecrawl** | ✅ | ✅ | ✅ | API Key または Nous Gateway |
| **Exa** | ✅ | ✅ | ❌ | EXA_API_KEY |
| **Parallel** | ✅ | ✅ | ❌ | PARALLEL_API_KEY |
| **Tavily** | ✅ | ✅ | ✅ | TAVILY_API_KEY |

### バックエンド選択チェーン

```python
def _get_backend():
    """解決優先度:
    1. config.yaml web.backend（明示指定: parallel/firecrawl/tavily/exa）
    2. FIRECRAWL_API_KEY / FIRECRAWL_API_URL / tool-gateway
    3. PARALLEL_API_KEY
    4. TAVILY_API_KEY
    5. EXA_API_KEY
    6. デフォルト: firecrawl（後方互換）
    """
```

### Firecrawl デュアルパスアーキテクチャ

Firecrawl はデフォルトバックエンド、2 種類の接続モードをサポート：

| モード | パス | 適用対象 |
|---|---|---|
| **直接モード** | `FIRECRAWL_API_KEY` / `FIRECRAWL_API_URL` | 全ユーザー |
| **ホストされたゲートウェイ** | Nous がホストする tool-gateway | Nous サブスクライバー |

```python
def _get_firecrawl_client():
    """優先度:
    1. 直接 Firecrawl 設定（api_key + api_url）
    2. Nous ホスト Gateway（nous_user_token + gateway_origin）
    """
    # クライアントキャッシュ — 設定変更なしなら同じインスタンス再利用
    if _firecrawl_client is not None and _firecrawl_client_config == client_config:
        return _firecrawl_client
```

**優位性**：Nous サブスクライバーは Firecrawl を別途購入不要、tool-gateway 経由で共有アクセス。

## コアコンポーネント

### 1. web_search_tool — ウェブ検索

```python
def web_search_tool(query: str, limit: int = 5) -> str:
    """
    バックエンドルーティング:
    - parallel → _parallel_search()（agentic/fast/one-shot モード対応）
    - exa → _exa_search()（highlights 抽出対応）
    - tavily → _tavily_request("search")
    - firecrawl → client.search()
    """
```

統一形式で返却：`{"success": true, "data": {"web": [{"title", "url", "description", "position"}]}}`

### 2. web_extract_tool — URL コンテンツ抽出

```python
async def web_extract_tool(
    urls: List[str],
    format: str = "markdown",      # markdown または html
    use_llm_processing: bool = True,
    model: Optional[str] = None,
    min_length: int = 5000         # LLM 処理をトリガーする最小長
) -> str:
```

**コアフロー**：
1. セキュリティチェック（秘密情報注入 + SSRF + サイトポリシー）
2. バックエンド抽出（Firecrawl scrape / Exa get_contents / Parallel extract / Tavily extract）
3. LLM 知的圧縮（`process_content_with_llm`）
4. 出力切り詰め（url / title / content / error のみ保持）

### 3. web_crawl_tool — サイトクロール

```python
async def web_crawl_tool(
    url: str,
    instructions: str = None,    # 抽出指示（Tavily のみサポート）
    depth: str = "basic",        # basic または advanced
    use_llm_processing: bool = True
) -> str:
```

現状 Firecrawl と Tavily のみ crawl をサポート。Parallel は crawl API なし。

## LLM コンテンツ処理エンジン

Web Tools 最大の革新部分 — LLM で Web ページコンテンツを自動圧縮。

### 処理戦略

```python
def process_content_with_llm(content, url, title, model, min_length):
    """
    コンテンツ階層処理:
    < 5,000 chars → 処理スキップ、元コンテンツをそのまま返す
    5,000 ~ 500K chars → 単発 LLM 要約
    500K ~ 2M chars → チャンク処理 + 合成
    > 2M chars → 処理拒否
    """
```

### チャンク処理（Chunked Processing）

```python
async def _process_large_content_chunked(content, chunk_size=100K):
    # 1. コンテンツを 100K chars のチャンクに分割
    # 2. 各チャンクを並列要約（asyncio.gather）
    # 3. 全チャンク要約を統一要約に合成
    # 4. ハード制限：最終出力 ≤ 5,000 chars
```

**設計のポイント**：
- 各チャンクで**専用プロンプト**を使用（「これは大ドキュメントの 1 セクション、序文と結論を書くな」）
- 全チャンクを並列処理、直列待機しない
- 合成ステップで**冗長性を除去**して一貫した要約に統合
- 合成が失敗したら、**全チャンク要約を連結する**フォールバック

### 圧縮率

典型的圧縮比：10-50x（元コンテンツ → LLM 要約）

```
元: 50,000 chars → 処理後: 2,000 chars (4%)
元: 200,000 chars → 処理後: 4,500 chars (2.25%)
```

## セキュリティ設計

### 4 層防御

| 層 | 保護 | 実装 |
|---|---|---|
| **URL 秘密情報注入** | URL に API Key 埋め込みを阻止 | `_PREFIX_RE` 検出 |
| **SSRF 防御** | プライベートアドレスへのアクセス阻止 | `is_safe_url()` |
| **サイトポリシー** | ブラックリストドメインのインターセプト | `check_website_access()` |
| **リダイレクトチェック** | 内部アドレスへのリダイレクト阻止 | 抽出後に `sourceURL` チェック |

### Base64 画像クリーンアップ

```python
def clean_base64_images(text: str) -> str:
    """base64 エンコード画像を削除、[BASE64_IMAGE_REMOVED] に置換"""
    # 大量の base64 データがコンテキストウィンドウを圧迫するのを防止
```

## 標準化レイヤー

異なるバックエンドが異なるデータ形式を返します。Web Tools は**標準化関数**で出力を統一：

```python
_extract_web_search_results(response)    # Firecrawl 複数形式抽出
_normalize_tavily_search_results(raw)    # Tavily → 標準形式
_normalize_tavily_documents(raw)         # Tavily extract/crawl → 標準形式
_to_plain_object(value)                  # SDK オブジェクト → Python dict
_normalize_result_list(values)           # 混合 SDK/list → dict list
```

**優位性**：Agent は常に統一形式のデータを受け取る、バックエンドタイプ別の異なる解析が不要。

## Debug モード

```bash
export WEB_TOOLS_DEBUG=true
```

有効化すると自動的に記録：
- 全ツール呼び出しとパラメータ
- 生の API 応答
- LLM 圧縮指標（元サイズ / 処理後サイズ / 圧縮比）
- 最終処理結果

ログ保存先：`~/.hermes/logs/web_tools_debug_UUID.json`

## 設計の優位性

### 直接 API 呼び出しとの比較

| 観点 | 直接 API 呼び出し | Web Tools |
|---|---|---|
| バックエンド切替 | コード変更が必要 | config.yaml で 1 行切替 |
| コンテンツ圧縮 | 手動処理 | 自動 LLM 要約 |
| 大量コンテンツ処理 | コンテキスト超過しやすい | チャンク + 合成 |
| セキュリティ防御 | 自前実装が必要 | SSRF + 注入 + ポリシー 3 層防御 |
| 形式統一 | API ごとに形式が異なる | 統一出力形式 |
| デバッグ | 手動 print 必要 | 内蔵 Debug モード |

### LLM 処理の優位性

LLM 処理なしでは、Agent は生の HTML / markdown 全文（数十万字の可能性）を受け取ります。LLM 処理ありなら：
- **コンテキスト節約**：10-50 倍圧縮
- **情報密度向上**：重要な事実とデータのみ保持
- **形式統一**：すべてのページが構造化 Markdown 要約
- **優雅な劣化**：LLM 失敗時は切り詰め元コンテンツにフォールバック

## 設定と操作

### バックエンド選択

```yaml
# config.yaml
web:
  backend: firecrawl  # または exa, parallel, tavily
```

### 環境変数

```bash
# Firecrawl 直接モード
export FIRECRAWL_API_KEY=fc-xxx
export FIRECRAWL_API_URL=https://your-self-hosted.com  # 任意

# Exa
export EXA_API_KEY=exa-xxx

# Parallel
export PARALLEL_API_KEY=par-xxx

# Tavily
export TAVILY_API_KEY=tav-xxx

# LLM 処理設定
export AUXILIARY_WEB_EXTRACT_MODEL=google/gemini-3-flash-preview
```

### LLM 処理を無効化

```python
# 高速抽出、圧縮不要
content = await web_extract_tool(["https://example.com"], use_llm_processing=False)
```

## 他システムとの関係

- [[auxiliary-client-architecture]] — LLM コンテンツ処理は `async_call_llm(task="web_extract")` 経由で呼ばれる
- [[tool-registry-architecture]] — web_search / web_extract は registry 経由で登録
- [[browser-tool-architecture]] — ドキュメントは単純な情報取得には web_tools 優先と提案
- [[context-compressor-architecture]] — 類似の LLM 圧縮理念を異なるシーンに適用
