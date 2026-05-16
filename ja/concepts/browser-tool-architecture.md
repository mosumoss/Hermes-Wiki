---
title: Browser Tool ブラウザ自動化アーキテクチャ
created: 2026-04-08
updated: 2026-04-08
type: concept
tags: [tool, toolset, architecture, component, browser]
sources: [tools/browser_tool.py, tools/browser_providers/]
translation: ja
original: ../../concepts/browser-tool-architecture.md
---

# Browser Tool — ブラウザ自動化アーキテクチャ

## 概要

Browser Tool は `tools/browser_tool.py`（84KB / 2,202 行）に実装され、**複数バックエンドのブラウザ自動化**機能を提供します。4 種類の実行モードをサポート、すべて Agent には完全に同一のツールインターフェース（navigate / click / type / scroll / vision 等）を公開。

コア理念：**アクセシビリティツリー（ariaSnapshot）ベースのテキスト化ページ表現**、LLM Agent が視覚能力なしで Web ページ操作可能。

## アーキテクチャ原理

### 複数バックエンド

| バックエンド | モード | 依存 | コスト |
|---|---|---|---|
| **ローカル Chromium** | デフォルト | `agent-browser` CLI + Chromium | コストゼロ |
| **Browser Use** | クラウド | BROWSER_USE_API_KEY または Nous ホスト | 従量課金 |
| **Browserbase** | クラウド | BROWSERBASE_API_KEY + PROJECT_ID | 従量課金 |
| **Firecrawl** | クラウド | FIRECRAWL_API_KEY | 従量課金 |
| **Camofox** | 検出回避 | CAMOFOX_URL 環境変数 | 自前 / 有料 |
| **CDP Override** | 直接接続 | BROWSER_CDP_URL | 既存ブラウザインスタンス |

### バックエンド解析チェーン

```python
def _get_cloud_provider():
    """解析優先度:
    1. config.yaml browser.cloud_provider（明示指定）
    2. Browser Use（managed Nous gateway または直接 API key）
    3. Browserbase（直接認証情報）
    4. None → ローカルモード
    """
```

**重要な設計**：`cloud_provider` を `local` に設定すると、クラウドフォールバックを完全無効化、ローカル Chromium を強制使用。

## コアコンポーネント

### 1. 統一 Provider インターフェース

```python
class CloudBrowserProvider:
    """全クラウドブラウザプロバイダの抽象基底クラス"""
    def is_configured() -> bool
    def create_session(task_id) -> Dict  # {session_name, cdp_url, features} を返す
    def close_session(session_id) -> None
    def provider_name() -> str

# 具体実装
class BrowserbaseProvider(CloudBrowserProvider)
class BrowserUseProvider(CloudBrowserProvider)
class FirecrawlProvider(CloudBrowserProvider)
```

**優位性**：新規バックエンドは 4 メソッド実装のみ、ツールロジックは一切変更不要。

### 2. セッション管理（スレッドセーフ）

```python
_active_sessions: Dict[str, Dict[str, str]] = {}  # task_id → session_info
_session_last_activity: Dict[str, float] = {}     # task_id → timestamp
_cleanup_lock = threading.Lock()
```

**設計詳細**：
- 各 task_id で独立セッション、サブエージェントの並列ブラウザ操作をサポート
- 二重チェックロックパターン：ネットワーク呼び出しはロック外、他スレッドをブロックしない
- 競合保護：ネットワーク呼び出し完了後に `_active_sessions` を再チェック、重複作成防止

### 3. コマンド実行アーキテクチャ

```python
def _run_browser_command(task_id, command, args, timeout):
    # 1. agent-browser CLI を見つける
    # 2. セッション情報を取得（作成 / 再利用）
    # 3. コマンドを構築：--cdp <websocket>（クラウド）または --session <name>（ローカル）
    # 4. 一時ファイル（パイプではない）で stdout/stderr キャプチャ
    # 5. JSON 出力を解析
```

**重要な決定 — 一時ファイルでパイプを代替**：

`agent-browser` 起動後にバックグラウンド daemon プロセスを開始、daemon はファイルディスクリプタを継承します。`capture_output=True`（パイプ）使用時、daemon はパイプ fd を開いたままにし、`communicate()` が EOF を永遠に待ってタイムアウト。

解決策：`os.open()` で一時ファイル作成、実行後すぐ fd を閉じる、daemon は読み取りをブロックしなくなる。

### 4. 並行安全 — 独立 Socket ディレクトリ

```python
task_socket_dir = os.path.join(
    tempfile.gettempdir(),
    f"agent-browser-{session_name}"
)
os.makedirs(task_socket_dir, mode=0o700, exist_ok=True)
browser_env["AGENT_BROWSER_SOCKET_DIR"] = task_socket_dir
```

**問題**：並列サブエージェントがデフォルト socket パスを共有、「Failed to create socket directory: Permission denied」エラー。

**解決**：各 task_id ごとに独立した socket ディレクトリ、権限 0o700 で隔離保証。

### 5. macOS Unix Socket パス修正

```python
def _socket_safe_tmpdir():
    """macOS TMPDIR=/var/folders/xx/.../T/（~51 chars）
    agent-browser-hermes_... を追加すると 104 バイト AF_UNIX 制限を超過
    → macOS では強制的に /tmp 使用"""
    if sys.platform == "darwin":
        return "/tmp"
    return tempfile.gettempdir()
```

## セキュリティ設計

### 3 層セキュリティ防御

| 層 | 保護 | 実装 |
|---|---|---|
| **URL 注入防御** | URL に API Key 埋め込み阻止 | `_PREFIX_RE` が sk-ant- 等のプレフィックス検出 |
| **SSRF 防御** | プライベート / 内部アドレスへのアクセス阻止 | `_is_safe_url()` が 10.x/192.168.x/localhost 検出 |
| **サイトポリシー** | ブラックリストドメインのインターセプト | `check_website_access(url)` |
| **リダイレクト後チェック** | 内部アドレスへのリダイレクト阻止 | ナビゲーション後に final_url チェック |
| **秘密情報マスキング** | 補助 LLM 送信前にスナップショットをマスキング | `redact_sensitive_text()` |

**重要**：SSRF 防御はクラウドバックエンドのみ有効。ローカルバックエンド（Camofox / ローカル Chromium）はこのチェックをスキップ、Agent は terminal ツール経由で既に完全なローカルネットワークアクセス権限を持つため。

### Bot 検出警告

```python
blocked_patterns = ["access denied", "bot detected", "cloudflare", 
                    "captcha", "just a moment", "checking your browser"]
if any(pattern in title_lower for pattern in blocked_patterns):
    response["bot_detection_warning"] = "..."
```

ナビゲーションで返されたページタイトルに bot 検出キーワードが含まれる時、能動的に警告し解決策（操作遅延 / シークレットモード有効化 / サイト変更）を提示。

## ツールセット（10 ツール）

| ツール | 機能 |
|---|---|
| `browser_navigate` | URL ナビゲーション、自動でコンパクトスナップショット返却 |
| `browser_snapshot` | ページのアクセシビリティツリースナップショット取得 |
| `browser_click` | ref で識別された要素をクリック（@e1, @e5） |
| `browser_type` | 入力フィールドにテキスト入力 |
| `browser_scroll` | 上 / 下にスクロール（5 回繰り返して有効な移動を保証） |
| `browser_back` | ブラウザ戻る |
| `browser_press` | キー押下（Enter / Tab / Escape 等） |
| `browser_console` | コンソール出力と JS エラー取得 |
| `browser_get_images` | ページ画像 URL と alt テキスト抽出 |
| `browser_vision` | スクリーンショット + 視覚 AI 分析 |

### 自動スナップショット最適化

`browser_navigate` 成功後**自動でコンパクトスナップショット取得**、モデルは追加で `browser_snapshot` を呼ぶ必要なし。これにより API 往復を 1 回削減。

### Vision ツール

```python
def browser_vision(question, annotate=False):
    # 1. スクリーンショット（--annotate で要素ラベルオーバーレイサポート）
    # 2. Base64 エンコード
    # 3. call_llm(task="vision") で視覚モデル呼び出し
    # 4. 分析結果 + スクリーンショットパス返却
    # 5. 失敗時はスクリーンショットファイルを保持、ユーザーが確認可能
```

**優雅な劣化**：スクリーンショット成功するが視覚分析失敗時、スクリーンショットファイルを保持し `MEDIA:<path>` でユーザーに通知。

### JavaScript 評価

`browser_console(expression="...")` でページコンテキスト内 JavaScript 実行、DevTools Console 相当：

```javascript
// 例：ページタイトル取得
document.title

// 例：リンク数カウント
document.querySelectorAll("a").length
```

## ライフサイクル管理

### バックグラウンドクリーンアップスレッド

```python
BROWSER_SESSION_INACTIVITY_TIMEOUT = 300  # 5 分無活動

def _browser_cleanup_thread_worker():
    """30 秒ごとにチェック、5 分以上無活動セッションをクリーンアップ"""
    while _cleanup_running:
        _cleanup_inactive_browser_sessions()
        time.sleep(30)
```

**設計考慮**：タイムアウトを 5 分に設定、LLM 推論に十分な時間を確保（特にサブエージェントが複数ステップのブラウザタスク実行時）。

### 緊急クリーンアップ

```python
atexit.register(_emergency_cleanup_all_sessions)  # プロセス終了時
```

**atexit のみ使用、SIGINT/SIGTERM はハイジャックしない**：初期バージョンでシグナルハンドラを installed して `sys.exit()` を呼んでいたが、prompt_toolkit の非同期イベントループと競合してプロセスを kill できなくなった。

### 自動録画

```yaml
# config.yaml
browser:
  record_sessions: true
```

初回ナビゲーション時に録画自動開始、セッションクローズ時に `.webm` ファイル保存。72 時間超過した録画は自動クリーンアップ。

## 設計の優位性

### 従来 Selenium/Playwright 方式との比較

| 観点 | 従来方式 | Hermes Browser Tool |
|---|---|---|
| ページ表現 | HTML/DOM（LLM が理解しづらい） | accessibility tree（構造化テキスト） |
| 要素特定 | XPath/CSS セレクタ | ref ID（@e1, @e5） |
| 複数バックエンド | コード書き直し必要 | 統一インターフェース、バックエンド自動選択 |
| セキュリティ | 内蔵保護なし | SSRF + 注入 + ポリシー 3 層防御 |
| 並行 | 手動管理必要 | task_id で自動隔離 |
| クリーンアップ | リーク起こしやすい | バックグラウンドスレッド + atexit 二重保証 |
| 視覚 | 追加統合必要 | 内蔵 vision ツール |

### Accessibility Tree の優位性

従来の HTML スナップショットには大量のスタイル / 構造ノイズが含まれます。Accessibility tree は以下のみ保持：
- インタラクティブ要素（ボタン、リンク、入力フィールド）
- セマンティックロール（heading, button, link, textbox）
- 可視テキストコンテンツ
- 要素関係

これにより LLM はより少ない token でページ構造を理解し操作判断できます。

## 設定と操作

### ローカルモード（コストゼロ）

```bash
# agent-browser インストール
npm install -g agent-browser
agent-browser install --with-deps  # Chromium + システムライブラリダウンロード
```

### クラウドモード

```yaml
# config.yaml
browser:
  cloud_provider: browser-use  # または browserbase, firecrawl, local
  allow_private_urls: false    # SSRF 保護（デフォルト有効）
  command_timeout: 30          # コマンドタイムアウト（秒）
  record_sessions: false       # 自動録画
```

### CDP 直接接続モード

```bash
export BROWSER_CDP_URL="ws://localhost:9222/devtools/browser/xxx"
# または HTTP 発見エンドポイント
export BROWSER_CDP_URL="http://localhost:9222"
```

### Camofox 検出回避モード

```bash
export CAMOFOX_URL="http://camofox-server:8080"
```

設定後、全ブラウザ操作は Camofox REST API 経由でルーティング。

## 他システムとの関係

- [[auxiliary-client-architecture]] — browser_vision は call_llm(task="vision") 経由で呼ばれる
- [[tool-registry-architecture]] — 10 個のブラウザツールは registry.register() で登録
- [[web-tools-architecture]] — ドキュメントは単純な情報取得には web_search/web_extract 優先と提案
- [[security-defense-system]] — ブラウザツールの SSRF と注入防御は全体セキュリティの一部
