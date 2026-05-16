---
title: 設定管理とマルチ Profile アーキテクチャ
created: 2026-04-07
updated: 2026-04-09
type: concept
tags: [architecture, configuration, profile, isolation]
sources: [hermes_cli/profiles.py, hermes_cli/config.py, hermes_cli/main.py, hermes_cli/gateway.py, hermes_constants.py, plugins/memory/honcho/cli.py, agent/prompt_builder.py]
translation: ja
original: ../../concepts/configuration-and-profiles.md
---

# 設定管理とマルチ Profile アーキテクチャ

## 概要

Hermes は**階層設定 + Profile 隔離**で複雑な多次元設定を管理します。Profile はコア設計 — 各 Profile は完全に独立した `HERMES_HOME` ディレクトリで、自前の設定、記憶、セッション、スキル、ゲートウェイ、定期タスクを持ちます。

## 設定階層

```text
優先度（低 → 高）:
  1. ハードコードデフォルト         (hermes_cli/config.py DEFAULT_CONFIG)
  2. ユーザー設定ファイル           (~/.hermes/config.yaml)
  3. 環境変数                     (.env ファイル + shell 環境変数)
  4. CLI パラメータ                (--model, --provider 等のコマンドライン引数)
  5. Profile オーバーライド         (HERMES_HOME 環境変数で異なるディレクトリ指定)
```

## 設定ファイル構造

Hermes には責任の異なる 2 つの設定ファイル：

| ファイル | 何を保管 | 有効化方法 |
|------|--------|---------|
| `.env` | API Keys、機密認証情報 | 環境変数として注入 |
| `config.yaml` | ランタイム動作設定 | `load_config()` で読み込み |

```yaml
# ~/.hermes/config.yaml コア設定項目
model:
  default: "anthropic/claude-opus-4.6"
  provider: "auto"
  base_url: "https://openrouter.ai/api/v1"

terminal:
  backend: "local"
  cwd: "."
  timeout: 180

compression:
  enabled: true
  threshold: 0.50
  summary_model: "google/gemini-3-flash-preview"

memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200
  user_char_limit: 1375
  nudge_interval: 10
  flush_min_turns: 6
```

## マルチ Profile アーキテクチャ

### コア原理

全 Hermes モジュールは `get_hermes_home()` でパスを解決：

```python
# hermes_constants.py — グローバル単一のパスソース
def get_hermes_home() -> Path:
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))
```

Profile 切替 = `HERMES_HOME` 環境変数を変更。コードベース 119+ ファイルが `get_hermes_home()` を呼び、Profile 切替時に全パスが自動的にリダイレクトされ、どのモジュールも Profile の存在を意識する必要なし。

### ディレクトリ構造

```text
~/.hermes/                              ← "default" Profile（後方互換）
  active_profile                        ← スティッキーデフォルトポインタ（Profile 名保管）
  config.yaml, .env, SOUL.md            ← default Profile の設定
  memories/, sessions/, skills/          ← default Profile のデータ
  state.db                              ← default Profile のデータベース
  profiles/                             ← 名前付き Profile ルートディレクトリ
    coder/                              ← 名前付き Profile（アクティベート時に HERMES_HOME になる）
      config.yaml                       ← 独立したモデル / ターミナル / 圧縮設定
      .env                              ← 独立した API Keys
      SOUL.md                           ← 独立した Agent アイデンティティ定義
      memories/MEMORY.md, USER.md       ← 独立した永続記憶
      sessions/                         ← 独立したセッションログ
      skills/                           ← 独立したスキルセット
      state.db                          ← 独立した SQLite データベース
      honcho.json                       ← 独立した Honcho 設定
      logs/, cron/, skins/, plans/, workspace/
    ops/                                ← 別の Profile
      ...

~/.local/bin/
  coder   → #!/bin/sh\nexec hermes -p coder "$@"
  ops     → #!/bin/sh\nexec hermes -p ops "$@"
```

**各 Profile 内部構造は完全に同一**、以下のディレクトリを含む：`memories`、`sessions`、`skills`、`skins`、`logs`、`plans`、`workspace`、`cron`。

### Profile アクティベーションフロー

```text
hermes -p coder chat
       │
       ▼
_apply_profile_override()          ← main.py モジュールレベル、すべての import より前に実行
       │
       ├─ sys.argv を解析して -p/--profile 引数を探す
       ├─ 見つからない → ~/.hermes/active_profile（スティッキーデフォルト）を読む
       │
       ▼
os.environ["HERMES_HOME"] = "~/.hermes/profiles/coder"
       │
       ▼
get_hermes_home() → Profile ディレクトリを返す
       │
       ▼
全モジュールが自動的に coder Profile で動作
（config、memory、skills、gateway、session すべて隔離）
```

重要：`_apply_profile_override()` は**モジュールレベル**で実行され、全 `import` より先 — 多くのモジュールが import 時に `HERMES_HOME` をキャッシュするため。

### CLI コマンド

```bash
# 作成
hermes profile create coder              # 空白 Profile + 内蔵スキルを seed
hermes profile create coder --clone      # config.yaml + .env + SOUL.md + 記憶をクローン
hermes profile create coder --clone-all  # 全状態を完全コピー（ランタイムファイル除く）
hermes profile create coder --no-alias   # ラッパーショートカットコマンドを生成しない

# 使用
hermes -p coder chat                     # Profile を指定して起動
coder chat                               # ラッパー経由でショートカット起動
hermes profile use coder                 # スティッキーデフォルトに設定

# 管理
hermes profile list                      # 全 Profile 状態を表示
hermes profile show coder                # 詳細情報（モデル / ゲートウェイ / スキル数）
hermes profile rename coder developer    # リネーム
hermes profile alias coder --name dev    # カスタムエイリアス
hermes profile export coder              # tar.gz としてエクスポート
hermes profile import archive.tar.gz     # インポート
hermes profile delete coder              # 削除（確認必要）
```

### Profile 命名ルール

```text
正規表現：^[a-z0-9][a-z0-9_-]{0,63}$
  ✅ coder, ops-team, dev2, my_profile
  ❌ Coder（大文字）, -ops（プレフィックスハイフン）, hermes（予約名）, chat（サブコマンド衝突）
```

予約名：`hermes`、`default`、`test`、`tmp`、`root`、`sudo` + 全 hermes サブコマンド名。

### クローン動作

| モード | コピー内容 |
|------|---------|
| `--clone` | config.yaml、.env、SOUL.md、MEMORY.md、USER.md |
| `--clone-all` | 完全 copytree（gateway.pid 等のランタイムファイル除く） |
| 引数なし | ディレクトリ構造のみ作成 + 内蔵スキルを seed |

記憶ファイル（MEMORY.md / USER.md）は `--clone` 時に一緒にコピー、ソースコードコメント："Memory files are part of the agent's curated identity — just as important as SOUL.md for continuity."

### エクスポート / インポートセキュリティ

**エクスポート時に機密ファイルを除外**：
- `.env`（API Keys）
- `auth.json`（OAuth tokens）
- `state.db`（機密会話を含む可能性）
- 各種キャッシュ（image_cache、audio_cache、checkpoints）

**インポート時の安全チェック**：
- パストラバーサル攻撃を拒否（`../`）
- 絶対パスを拒否（`/etc/passwd`）
- Windows ドライブ文字を拒否（`C:\`）
- シンボリックリンクを拒否
- 通常ファイルとディレクトリのみ許可

## Profile とサブシステムの連携

### Gateway 隔離

各 Profile は独立した Gateway（Telegram / Slack 等）を実行可能：

```text
default Profile  → hermes-gateway          (サービス名)
coder Profile    → hermes-gateway-coder    (サービス名にサフィックス付き)
```

- PID ファイルは各自の HERMES_HOME スコープ、互いに衝突しない
- systemd / launchd サービス名は自動的に Profile サフィックス付き
- 2 つの Profile が同じ Bot Token を使う場合、2 番目の Gateway は阻止されエラー報告

### Honcho 記憶隔離

各 Profile は Honcho 内で独立した host block を持つ：

```text
default → hermes          (host key)
coder   → hermes.coder    (host key サフィックス付き)
```

AI Peer は Profile 別に隔離（独立したユーザーモデリング）、workspace は共有（全 Profile が同じユーザーの観察データを見る）。

新 Profile 作成時に Honcho 設定を自動クローン、`hermes update` 時に全 Profile の Honcho host blocks を自動同期。

### SOUL.md アイデンティティ

各 Profile は自分の `SOUL.md` を持ち、Agent のアイデンティティと行動規範を定義。`prompt_builder.py` は `get_hermes_home() / "SOUL.md"` でロード、Profile 切替後は自動的に対応ファイルを指す。

### スキル同期

`hermes update` は内蔵スキルを**全**Profile に自動同期：

```text
hermes update
  → 現在の Profile スキル更新
  → 他の全 Profile をスキャン
  → 各 Profile で seed_profile_skills() を実行
  → ユーザーカスタムスキルは上書きされない
```

スキル seed は**サブプロセス**経由で実行（in-process ではない）、`sync_skills()` がモジュールレベルで HERMES_HOME をキャッシュするため。

### Banner と Prompt

- 起動 Banner で現在の Profile 名を表示（default 以外時）
- CLI 入力プロンプトに Profile プレフィックス：`coder >`（`>` ではなく）
- Gateway は `/profile` コマンドで現在の Profile 表示をサポート

## 典型的な利用シーン

```bash
# シーン：職能別隔離
hermes profile create coder --clone       # 日常開発
hermes profile create ops --clone         # 運用操作
hermes profile create research --clone    # 研究調査

# それぞれ異なるセキュリティ境界を設定
hermes -p coder config set terminal.backend local
hermes -p ops config set terminal.backend docker
hermes -p research config set terminal.backend ssh

# それぞれ異なるモデルを設定
hermes -p coder config set model.default "anthropic/claude-opus-4.6"
hermes -p research config set model.default "google/gemini-2.5-pro"

# それぞれの Gateway を実行
hermes -p coder telegram &
hermes -p ops telegram &
```

## Multi-Agent との関係

マルチ Profile は Hermes の**第 2 のマルチ Agent 方式**と見なせます。セッション内 multi-agent（delegate_task）は「1 タスク内の並列分業」に適し、マルチ Profile は「異なる職能ロールの長期隔離」に適します。両者は相補的：

- delegate_task サブ agent は**親 agent の terminal backend を継承**、タスク別に隔離レベルを切替えられない
- マルチ Profile は各ロールに**独立 backend 設定**可能（coder は local、ops は docker）
- 代償としてマルチ Profile 間は自動連携なし、ユーザーが手動切替必要

詳細 → [[multi-agent-architecture]]

## 関連ページ

- [[multi-agent-architecture]] — セッション内マルチ Agent（delegate_task / MoA / Background Review）
- [[terminal-backends]] — ターミナルバックエンド選択（Profile は各ロールに異なるバックエンドを設定可能）
- [[memory-system-architecture]] — 記憶システム（各 Profile 独立の MEMORY.md / USER.md）
- [[skills-system-architecture]] — スキルシステム（各 Profile 独立のスキルセット）
- [[credential-pool-and-isolation]] — 認証情報隔離
- [[hook-system-architecture]] — Hook システム（Gateway Hooks は Profile 別隔離）

## 主要ソース

| ファイル | 責任 |
|------|------|
| `hermes_constants.py` | `get_hermes_home()` — グローバルパスソース |
| `hermes_cli/profiles.py` | Profile CRUD、エクスポート / インポート、エイリアス管理 |
| `hermes_cli/main.py` | `_apply_profile_override()` — 起動時 Profile アクティベーション |
| `hermes_cli/config.py` | `load_config()` — Profile スコープの config.yaml 読み込み |
| `hermes_cli/gateway.py` | Gateway サービス名サフィックス、PID 隔離 |
| `plugins/memory/honcho/cli.py` | Honcho host block の Profile 別隔離 |
| `agent/prompt_builder.py` | SOUL.md の Profile 別ロード |
