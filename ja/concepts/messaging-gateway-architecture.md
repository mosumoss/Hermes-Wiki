---
title: Messaging Gateway Architecture
created: 2026-04-07
updated: 2026-04-29
type: concept
tags: [gateway, architecture, module, telegram, discord, messaging, qq, proxy]
sources: [gateway/run.py, gateway/platforms/, hermes_cli/config.py]
translation: ja
original: ../../concepts/messaging-gateway-architecture.md
---

# メッセージングゲートウェイアーキテクチャ

## 概要

Gateway は Hermes Agent の**統一メッセージングゲートウェイ**、14+ メッセージングプラットフォームをサポート、単一プロセスから全プラットフォームの接続とメッセージディスパッチを管理。

## アーキテクチャ

```
gateway/
├── run.py              # メインループ、スラッシュコマンド、メッセージディスパッチ
├── session.py          # SessionStore — 対話永続化
├── delivery.py         # メッセージ配信
├── config.py           # ゲートウェイ設定
├── hooks.py            # フックシステム
├── pairing.py          # DM ペアリング
├── status.py           # 状態管理
├── mirror.py           # クロスプラットフォームミラー
├── sticker_cache.py    # スタンプキャッシュ
├── stream_consumer.py  # ストリーミング消費
├── channel_directory.py # チャンネルディレクトリ
└── platforms/          # プラットフォームアダプタ
    ├── telegram.py
    ├── telegram_network.py
    ├── discord.py
    ├── slack.py
    ├── whatsapp.py
    ├── signal.py
    ├── email.py
    ├── sms.py
    ├── matrix.py
    ├── mattermost.py
    ├── dingtalk.py
    ├── feishu.py
    ├── wecom.py
    ├── weixin.py
    ├── bluebubbles.py
    ├── homeassistant.py
    ├── webhook.py
    ├── api_server.py
    └── base.py
```

## プラットフォームサポート

| プラットフォーム | タイプ | 特性 |
|------|------|------|
| Telegram | Bot API | グループ / DM、音声転写、スタンプ、プロキシ対応、リンクプレビュー制御 |
| Discord | Bot API | サーバー / DM、音声チャンネル、Slash Commands、ロール権限制御、channel_prompts |
| Slack | Bot API | Workspace 統合、Thread 対応 |
| WhatsApp | Bridge (Node.js) | グループ / DM、許可リスト |
| Signal | Bot API | 暗号化メッセージ、ネイティブ書式、reply 引用、reactions（v2026.4.23+） |
| Email | IMAP/SMTP | メール対話 |
| SMS | Twilio | SMS、文字数制限 |
| Home Assistant | WebSocket | スマートホームイベント |
| Matrix | E2E 暗号化 | 分散型メッセージ |
| Mattermost | Bot API | セルフホストチームメッセージ |
| DingTalk | Stream | 企業メッセージ、QR 認証、require_mention + allowed_users 権限制御 |
| Feishu / Lark | Stream | 企業メッセージ |
| WeCom | Stream | 企業 WeChat メッセージ |
| BlueBubbles | REST + Webhook | iMessage（macOS）、tapback、既読 |
| WeChat | iLink Bot API | ロングポーリング受信、AES-128-ECB メディア暗号化、QR ログイン |
| QQ Bot | Official API v2 | WebSocket 受信（C2C / 群 / 频道 / DM）+ REST 送信、音声転写（Tencent ASR）、allowlist + DM ペアリング |
| Webhook | HTTP | 外部イベント受信 |
| **Tencent Yuanbao** | API | ネイティブテキスト + メディア配信、スタンプサポート（v2026.4.23+） |
| **IRC**（プラグイン） | TLS asyncio | 外部依存ゼロ、TLS、PING/PONG、nick 衝突、NickServ、チャンネルアドレッシング（v2026.4.23+、リファレンス実装） |

## プラットフォームアダプタのプラグイン化（v2026.4.23+）

`gateway/platform_registry.py` で `PlatformRegistry` シングルトン + `PlatformEntry` dataclass を導入、誰でも新プラットフォーム（IRC、Viber、Line 等）を**純粋プラグイン**として接続可能、gateway コアコード変更不要。

```python
# プラグイン登録エントリ
def register(ctx):
    ctx.register_platform(
        name="irc",
        label="IRC",
        adapter_factory=create_irc_adapter,
        check_fn=check_irc_available,
        validate_config=validate_irc_config,
        required_env=["IRC_NICK", "IRC_PASS"],
        install_hint="pip install ...",
    )
```

### 重要な改造点

| モジュール | 改造 |
|------|------|
| `Platform` enum | `_missing_()` が未知文字列を受け入れ、キャッシュされた pseudo-member 作成（`Platform('irc') is Platform('irc')` 常に真） |
| `GatewayConfig.from_dict` | config.yaml のプラグインプラットフォーム名を解析、未知プラットフォームを拒否しない |
| `gateway/run.py` の `_create_adapter()` | 先に registry をクエリ、未ヒット時に内蔵 if/elif チェーンへフォールスルー |
| `get_connected_platforms()` | 未知プラットフォームを registry に委任 |
| `PluginContext.register_platform()` | `register_tool()` / `register_hook()` パターンをミラー |

### IRC リファレンス実装

`plugins/platforms/irc/` は最初のプラグインプラットフォーム：
- 全 async（`asyncio` stdlib、外部依存ゼロ）
- TLS 接続、PING/PONG ハートビート、nick 衝突リネーム、NickServ 自動認証
- チャンネルメッセージは `nick: msg` アドレッシング要、DM 全てディスパッチ
- 出力 Markdown 自動剥がし（IRC 非対応）、メッセージ分割（IRC 長さ制限）
- 対話的 `setup` ウィザード（v2026.4.23+）

### プラットフォームプラグイン 12 統合ポイント完全カバー

`feat: complete plugin platform parity`（2e20f6ae2）+ `feat: final platform plugin parity`（e464cde58）でプラグインプラットフォームと内蔵プラットフォームの動作を統一：
- webhook 配信、PLATFORM_HINTS、`get_connected_platforms`、cron 配信、動的 toolset 生成、setup wizard 等
- 同梱プラグインプラットフォーム（IRC 等）は起動時自動ロード（`feat(plugins): bundled platform plugins auto-load by default`）

## プラットフォームアダプタ基底クラス

```python
# gateway/platforms/base.py
class BasePlatform:
    """プラットフォームアダプタ基底クラス"""
    
    def __init__(self, config: dict, gateway):
        self.config = config
        self.gateway = gateway
        self.platform_name = self.__class__.__name__.lower()
    
    async def start(self):
        """プラットフォーム接続を開始"""
        raise NotImplementedError
    
    async def stop(self):
        """プラットフォーム接続を停止"""
        raise NotImplementedError
    
    async def send_message(self, chat_id: str, text: str, **kwargs):
        """メッセージ送信"""
        raise NotImplementedError
    
    async def handle_message(self, event: MessageEvent):
        """受信メッセージ処理"""
        await self.gateway.process_event(event)
```

## メッセージ処理フロー

```
ユーザーがメッセージ送信
  ↓
プラットフォームアダプタが受信
  ↓
MessageEvent 作成
  ↓
GatewayRunner.process_event(event)
  ↓
スラッシュコマンド解析（あれば）
  ↓
Session を検索 / 作成
  ↓
AIAgent を呼ぶ
  ↓
応答取得
  ↓
プラットフォームアダプタ経由で返信送信
```

## セッション管理

```python
# gateway/session.py
class SessionStore:
    """対話永続化ストア"""
    
    def get_or_create_session(self, chat_id, platform):
        """セッション取得 / 作成"""
    
    def save_session(self, session_id, messages):
        """セッション保存"""
    
    def get_session(self, session_id):
        """セッション取得"""
```

## スラッシュコマンド

CLI と共有のスラッシュコマンドシステム：

| コマンド | 説明 |
|------|------|
| `/new` | 新対話 |
| `/reset` | 対話リセット |
| `/model [provider:model]` | モデル切替 |
| `/personality [name]` | 個性設定 |
| `/retry` | 前回再試行 |
| `/undo` | 前回取消 |
| `/compress` | コンテキスト圧縮 |
| `/usage` | token 使用量チェック |
| `/insights [days]` | 使用量分析 |
| `/skills` | スキル閲覧 |
| `/stop` | 現作業中断 |
| `/status` | プラットフォーム状態 |
| `/sethome` | ホームプラットフォーム設定 |

## DM ペアリング

`GATEWAY_ALLOWED_USERS` 環境変数で誰が bot と話せるかを制御：

```bash
# 許可された Telegram ユーザー ID
GATEWAY_ALLOWED_USERS=telegram:123456789,discord:987654321
```

未承認ユーザーがメッセージ送信時、bot は応答しない（サイレント無視）。

## メディア処理

```
ユーザーが画像 / ファイル送信
  ↓
プラットフォームアダプタがダウンロード
  ↓
一時ディレクトリに保存
  ↓
Agent に渡す（vision_analyze またはファイル処理）
  ↓
Agent 応答に MEDIA: パス含む
  ↓
ローカルファイル抽出
  ↓
プラットフォームネイティブで送信
```

## ゲートウェイサービス管理

### Linux (systemd)

```ini
# ~/.config/systemd/user/hermes-gateway.service
[Unit]
Description=Hermes Agent Gateway
After=network-online.target

[Service]
ExecStart=/path/to/hermes gateway run
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

```bash
hermes gateway start    # サービス起動
hermes gateway stop     # サービス停止
hermes gateway status   # 状態確認
```

サービスユニット：`hermes-gateway.service` または `hermes-gateway-<profile>.service`

### macOS (launchd)

```xml
<!-- ~/Library/LaunchAgents/com.nousresearch.hermes-gateway.plist -->
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.nousresearch.hermes-gateway</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/hermes</string>
        <string>gateway</string>
        <string>run</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

```bash
hermes gateway start    # launchd サービス起動
hermes gateway stop     # 停止
hermes gateway status   # 状態
```

ラベル：`com.nousresearch.hermes-gateway`

## 更新時の自動再起動

`hermes update` コマンドは自動的に：
1. 実行中の gateway サービスをすべて検出
2. systemd / launchd サービス再起動
3. 非サービスモードの手動プロセス停止

## プラットフォーム固有機能

### Telegram
- グループと DM 対応
- グループメッセージは @mention でトリガー
- 音声メッセージ転写
- スタンプサポート
- トピック / スレッドサポート
- **プロキシサポート**（v0.10.0）：`TELEGRAM_PROXY` 環境変数または `config.yaml` の `proxy_url`
- **リンクプレビュー制御**（v0.10.0）：`config.yaml` の `telegram.disable_link_preview` でメッセージリンクプレビュー無効化

### Discord
- サーバーと DM 対応
- @mention または DM 必要
- 音声チャンネル対応
- Opus 音声エンコード
- Slash commands 統合
- **ロール権限制御**（v0.10.0）：`DISCORD_ALLOWED_ROLES` 環境変数、カンマ区切り Role ID。`DISCORD_ALLOWED_USERS` と OR 関係 — ユーザー ID またはロールが 1 つでもマッチで通過、両方未設定なら全員利用可能
- **channel_prompts**（v0.10.0）：チャンネル / トピック別に異なるシステムプロンプト注入、Telegram（グループ / フォーラムトピック）、Slack、Mattermost にも拡張
- **@everyone とロール ping ブロック**：`allowed_mentions` はデフォルトで bot による全員通知をブロック

### DingTalk
- Stream プロトコル接続
- **QR 認証**（v0.10.0）：`hermes_cli/dingtalk_auth.py`（292 行）で Device Flow 実装 — ターミナルに QR コード表示、ユーザーが DingTalk でスキャン、自動的に AppKey / AppSecret 取得、手動アプリ作成不要
- **require_mention + allowed_users 権限制御**（v0.10.0）：Telegram / Discord と整合
- dingtalk-stream 0.24+ SDK と oapi webhooks 対応

### WeChat
- SILK エンコード音声応答（v0.10.0）
- メディア添付抽出と送信
- ネイティブ Markdown レンダリング
- CDN ホワイトリスト SSRF 防御（セキュリティ修正）
- macOS SSL 証明書修正

### WhatsApp
- WhatsApp Bridge (Node.js) 必要
- グループメッセージはプレフィックストリガー必要
- 許可リスト制御

### Home Assistant
- スマートホームイベント監視
- デバイス制御
- 自動化トリガー

### Gateway 運用強化（v0.10.0）
- **Agent キャッシュ LRU + アイドル TTL 退避**：`_agent_cache` に上限とアイドルタイムアウト追加、長期運用 gateway のメモリリーク防止
- **一時 agent クローズ**：単発タスク完了後に一時 agent 自動クローズ
- **WebSocket 再接続待機**：送信前に再接続完了待機、メッセージロス回避

### v2026.4.18+ 強化

- **WeCom QR 認証**：setup ウィザード（`hermes_cli/gateway.py:_setup_wecom`）が `gateway.platforms.wecom.qr_scan_for_bot_info` で QR スキャンして bot 認証情報取得、手動設定不要
- **プラグインスラッシュコマンドのクロスプラットフォームネイティブ化**：`register_command()` のプラグインコマンドが Discord native slash、Telegram BotCommand、Slack `/hermes` サブコマンドに自動公開、各プラットフォーム別実装不要
- **決定型 command hook**：`command:<name>` フックが `{"decision": "deny"|"handled"|"rewrite"|"allow"}` を返してコア処理前にインターセプト
- **Slack リアクションライフサイクル**：`SLACK_REACTIONS` 環境変数で bot 送受信時のリアクション（絵文字）制御
- **Feishu @mention コンテキスト保持**：受信メッセージで @mention コンテキストを保持
- **Feishu ストリーミング編集改行修正**：ストリーミング出力で余分な前置空行なし
- **Session 状態メンテナンス**：`hermes_state.py` に `maybe_auto_prune_and_vacuum()` 追加、起動時冪等実行（`state_meta` テーブルで前回実行時刻をプロセス間記録）。session と FTS5 インデックスの無制限増大を防止（重度ユーザーが 384MB / 982 sessions でパフォーマンス影響、prune + VACUUM 後 43MB）
- **MEDIA: タグ拡張**：PDF、document、archive 拡張子の自動抽出対応
- **グローバルトンネル / プロキシシーン URL スイッチ**：`security.allow_private_urls` / `HERMES_ALLOW_PRIVATE_URLS` でプライベート IP 範囲（198.18.0.0/15、100.64.0.0/10）の解決許可、OpenWrt / TUN プロキシ（Clash/Mihomo/Sing-box）/ 企業 VPN / Tailscale シーンに対応。クラウドメタデータエンドポイント（169.254.169.254 等）は常にブロック
- **プラットフォーム hints**：`PLATFORM_HINTS` で Matrix、Mattermost、Feishu のシステムプロンプトをオーバーライド

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | OpenClaw | Claude |
|------|--------|----------|--------|
| プラットフォーム数 | 14+ | 14+ | 1 |
| 統一ゲートウェイ | 単一プロセス | 対応 | N/A |
| セッション共有 | クロスプラットフォーム | 対応 | N/A |
| 音声転写 | Telegram/Discord | 対応 | N/A |
| グループサポート | マルチプラットフォーム | 対応 | N/A |
| サービス管理 | systemd/launchd | 対応 | N/A |

## Gateway Proxy Mode（薄リレーモード、2026-04-14）

通常 Gateway と Agent は同一プロセスで動作：Gateway がメッセージ受信 → 直接 `AIAgent.run_conversation()` 呼び出し。**Proxy mode** は両者を分離 — Gateway はプラットフォーム I/O（暗号化、分割、メディア）のみ担当、全 Agent 作業をリモート Hermes API server に転送。

### 典型用途

```
[Matrix/Discord/...]  ←→  [Gateway (Linux Docker, E2EE keys)]
                                    │ POST /v1/chat/completions (SSE)
                                    ↓
                              [Hermes API server (macOS host)]
                                    │
                                    ↓
                     ローカルファイル、memory、skills、統一 session store
```

**解決する問題**：Matrix E2EE を使いたいが、E2EE は暗号化キー永続化が必要で Docker で動かす方が安定；しかし agent 自体は macOS ホストでローカルファイル / スキル / 記憶にアクセスしたい。以前は二者択一、proxy mode が両者をつなぐ。

### 有効化方法

```yaml
# ~/.hermes/config.yaml — 設定優先
gateway:
  proxy_url: "http://host.docker.internal:8080"
```

または環境変数（Docker フレンドリー、config マウント不要）：

```bash
GATEWAY_PROXY_URL=http://host.docker.internal:8080
GATEWAY_PROXY_KEY=<matches upstream API_SERVER_KEY>
```

### 実装位置

`gateway/run.py:7709` 以降：
- `_get_proxy_url()` — 先に env var、その後 config.yaml チェック
- `_run_agent_via_proxy()` — HTTP + SSE streaming 転送、ストリーミング応答解析
- `_run_agent()` — proxy_url 検出で proxy パス、それ以外はローカル agent
- `GatewayStreamConsumer` は通常通り動作、ストリーミング分割は引き続き Gateway 側

### 重要特性

| 機構 | 説明 |
|---|---|
| `X-Hermes-Session-Id` header | session id を渡してリクエスト間の session 連続性保証 |
| `GATEWAY_PROXY_KEY` | リモートの `API_SERVER_KEY` と一致、Bearer 認証 |
| SSE streaming | 応答が chunk で到達、Gateway がストリーミングでプラットフォームに送信 |
| エラー互換 | 返却 result dict 構造はローカル agent と一致、session store も通常通り記録 |
| プラットフォーム非依存 | Matrix 限定ではなく、任意のプラットフォーム adapter で proxy モード可能 |

### 呼び出しチェーン

```
ユーザーが Matrix でメッセージ送信
    ↓ E2EE 復号（Gateway 側）
gateway.process_event()
    ↓
_run_agent() → proxy_url 検出
    ↓
_run_agent_via_proxy():
    POST {proxy_url}/v1/chat/completions
      + X-Hermes-Session-Id: <sid>
      + Authorization: Bearer <GATEWAY_PROXY_KEY>
      + body: { messages: [...], stream: true }
    ↓ SSE stream 到達
    chunk ごとに GatewayStreamConsumer 経由でプラットフォームに転送
    ↓ E2EE 暗号化（Gateway 側）
ユーザーに送信
```

## 関連ページ

- [[gateway-session-management]] — ゲートウェイセッション管理アーキテクチャ
- [[cron-scheduling]] — Cron スケジューラはゲートウェイ駆動
- [[hook-system-architecture]] — ゲートウェイイベントフックシステム

## 関連ファイル

- `gateway/run.py` — メインループとメッセージディスパッチ
- `gateway/session.py` — SessionStore
- `gateway/platforms/base.py` — プラットフォーム基底クラス
- `gateway/delivery.py` — メッセージ配信
- `gateway/config.py` — ゲートウェイ設定
- `gateway/platforms/` — プラットフォームアダプタディレクトリ
- `hermes_cli/gateway.py` — Gateway CLI コマンド
