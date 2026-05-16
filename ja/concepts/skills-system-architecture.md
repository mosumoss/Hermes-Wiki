---
title: Skills System Architecture
created: 2026-04-07
updated: 2026-04-29
type: concept
tags: [skill, architecture, module, prompt-builder]
sources: [tools/skills_tool.py, tools/skill_manager_tool.py, tools/skills_hub.py, tools/skills_guard.py, run_agent.py, agent/prompt_builder.py, hermes_cli/plugins.py, agent/skill_utils.py]
translation: ja
original: ../../concepts/skills-system-architecture.md
---

# スキルシステムアーキテクチャ

## 概要

Hermes Agent のスキルシステムは**段階的開示（Progressive Disclosure）**アーキテクチャで、Anthropic の Claude Skills システムにインスパイアされています。コア理念は「完全な指示は必要時にのみロード、平時は軽量メタデータのみ保持」。これにより token 予算を節約します。

## コアコンポーネント

### 1. ツール層 (`tools/skills_tool.py`)

2 つのツールを提供：
- **`skills_list`** — スキルメタデータ一覧を返す（名前、説明、カテゴリ）。token 効率が高い
- **`skill_view`** — 完全なスキル内容をロード（SKILL.md + オプションの参照ファイル）

### 2. Prompt 構築層 (`agent/prompt_builder.py`)

各システムプロンプト構築時に：
- `~/.hermes/skills/` ディレクトリをスキャン
- 各 SKILL.md の YAML frontmatter を解析
- スキルインデックス一覧を構築してシステムプロンプトに注入
- [[prompt-builder-architecture]] のキャッシュ結果を使用

### 3. スキルディレクトリ構造

```
~/.hermes/skills/
├── my-skill/
│   ├── SKILL.md              # メイン指示ファイル（必須）
│   ├── references/           # 補助ドキュメント
│   │   ├── api.md
│   │   └── examples.md
│   ├── templates/            # 出力テンプレート
│   └── assets/               # 補足ファイル（agentskills.io 標準）
└── category/                 # カテゴリディレクトリ
    └── another-skill/
        └── SKILL.md
```

### 4. SKILL.md 形式

```yaml
---
name: skill-name                    # 必須、最大 64 文字
description: Brief description      # 必須、最大 1024 文字
version: 1.0.0                      # 任意
license: MIT                        # 任意
platforms: [macos]                  # 任意 — OS プラットフォーム制限
prerequisites:                      # 任意 — 実行時要件
  env_vars: [API_KEY]               #   環境変数
  commands: [curl, jq]              #   コマンドチェック
setup:                              # 任意 — 対話的セットアップ
  help: "Get key at https://..."    #   ヘルプテキスト
  collect_secrets:                  #   秘密情報収集
    - env_var: API_KEY
      prompt: "Enter your API key"
      secret: true
metadata:                           # 任意
  hermes:
    tags: [fine-tuning, llm]
    related_skills: [peft, lora]
---

# Skill Title

完全な指示と内容をここに...
```

## スキル発見フロー

```python
# 1. 全スキルディレクトリを取得
get_all_skills_dirs() → [Path, Path, ...]

# 2. 各 SKILL.md の frontmatter を解析
parse_frontmatter(raw_content) → (dict, body)

# 3. プラットフォーム互換性チェック
skill_matches_platform(frontmatter) → bool

# 4. 条件付き活性化ルールを抽出
extract_skill_conditions(frontmatter) → {
    "requires_tools": [...],
    "requires_toolsets": [...],
    "fallback_for_tools": [...],
    "fallback_for_toolsets": [...]
}

# 5. スキルインデックスを構築してシステムプロンプトに注入
_build_skills_index(available_tools, available_toolsets) → str
```

## 条件付き活性化メカニズム

スキルは現在利用可能なツール / ツールセットに応じて条件的に表示されます：

- **`requires_tools`** — 特定ツールがある場合のみ表示
- **`requires_toolsets`** — 特定ツールセットがある場合のみ表示
- **`fallback_for_tools`** — 主ツールが利用可能な場合は非表示（代替として）
- **`fallback_for_toolsets`** — 主ツールセットが利用可能な場合は非表示

## プラットフォームフィルタ

`platforms` frontmatter フィールドで特定 OS のみにスキルをロード制限：
- `macos` → `sys.platform == "darwin"`
- `linux` → `sys.platform == "linux"`
- `windows` → `sys.platform == "win32"`

## プラグイン名前空間スキル（2026-04-14）

`~/.hermes/skills/` のフラットディレクトリスキャンに加え、プラグインは**名前空間付きスキル**を登録できます。内蔵スキルとの名前衝突を回避できます。

### 登録方法

```python
# プラグインの __init__.py
def register(ctx):
    ctx.register_skill(
        name="deploy",
        path=Path(__file__).parent / "skills" / "deploy" / "SKILL.md",
        description="Deploy a service to production",
    )
```

`PluginContext.register_skill()` は内部的に `{plugin_name}:{name}` 形式の qualified name で保管します。例えばプラグイン `myops` が登録した `deploy` スキルの実名は `myops:deploy` になります。

**検証ルール**（`hermes_cli/plugins.py:267`）：
- `name` に `:` を含めることはできない（名前空間はプラグイン名から自動派生）
- `name` は `[a-zA-Z0-9_-]+` にマッチする必要がある
- `path` が指す SKILL.md は存在する必要がある

### ディスパッチロジック

`skill_view(name)` は `tools/skills_tool.py:822` で `:` 区切りを検出：
- **`:` を含む名前** → `parse_qualified_name(name)` → `_serve_plugin_skill(namespace, bare)` にルーティング
- **裸の名前** → 既存の `~/.hermes/skills/` フラットツリースキャンを継続

プラグインスキルロード時には完全な防御が走ります：
1. プラグインが disable → エラー返却（`hermes plugins enable` のヒント付き）
2. プラットフォーム不一致（`skill_matches_platform`）→ UNSUPPORTED 返却
3. インジェクションパターンスキャン（`_INJECTION_PATTERNS`）→ ログ記録するがロードは継続（ローカルスキルと同じ動作）
4. 返却時に **bundle context banner** を付帯、同プラグインの他スキルを agent に提示

### システムプロンプトインデックスには載らない

**重要な違い**：プラグインスキルは system prompt の `<available_skills>` リストに**表示されません**。これらは**明示的 opt-in** — agent は名前を知っている必要があり（ドキュメントやプラグイン README 経由）、それを使って `skill_view("myops:deploy")` を呼び出します。

このように設計した理由：
- プラグインがメインプロンプトを汚染するのを避ける（システムプロンプトは既に大きい）
- 第三者プラグインの数の変動で prefix cache が無効化されるのを避ける
- ユーザーが何のプラグインを入れたかを agent が自動的に全部知る必要はない

### 関連 API

| シンボル | 場所 | 用途 |
|---|---|---|
| `PluginContext.register_skill()` | `hermes_cli/plugins.py:267` | プラグイン登録入口 |
| `PluginManager._plugin_skills` | `hermes_cli/plugins.py` | 登録テーブル保管 |
| `parse_qualified_name()` | `agent/skill_utils.py:451` | `ns:bare` を分解 |
| `is_valid_namespace()` | `agent/skill_utils.py` | 名前空間の妥当性検証 |
| `_serve_plugin_skill()` | `tools/skills_tool.py:718` | ロード + 防御 + banner |
| `_INJECTION_PATTERNS` | `tools/skills_tool.py`（モジュールレベル） | ローカルスキルと共有のインジェクション検出 |

## 秘密情報管理

スキルは必要な環境変数を宣言でき、システムは：
1. `~/.hermes/.env` に既に設定されているかチェック
2. 欠落かつ CLI モードなら、コールバックで対話的に収集
3. Gateway モードなら、ユーザーに手動設定を促す
4. 保存後は `.env` ファイルに永続化

## 自動 Skill Review（Background Review）

Hermes は Skill を受動的に使うだけでなく、**自律的にスキルを作成・更新**できます。これが Hermes の「自己進化」メカニズムです。

### 発火条件

3 つの条件が同時に満たされたときに発火：

```python
if (self._skill_nudge_interval > 0                          # 機能が無効化されていない
        and self._iters_since_skill >= self._skill_nudge_interval  # ツール呼び出しの累計が閾値到達
        and "skill_manage" in self.valid_tool_names):        # skill_manage ツールが利用可能
```

```yaml
# config.yaml
skills:
  creation_nudge_interval: 15   # ツール呼び出し 15 回ごとに review を発火（0 = 無効）
```

注意：カウンタが加算するのは**ツールループ回数**（対話ターン数ではない）、ターンを跨いで累積します。agent が能動的に `skill_manage` を呼ぶとカウンタはゼロにリセット。

### 実行フロー

```text
ツール呼び出し累計が 15 回到達
    ↓
ターン終了後、バックグラウンド agent を派生（独立スレッド、max_iterations=8）
    ↓
バックグラウンド agent が完全な対話スナップショットを取得し審査：
  「試行錯誤を経た、方向転換した、またはユーザーが異なる方法を期待した
   非自明な経験があったか？」
    ↓
3 つの結果：
  ├── 既存スキルあり → skill_manage を呼んで更新
  ├── ないが新規作成の価値あり → skill_manage を呼んで作成
  └── 保存に値するものなし → "Nothing to save." で終了
    ↓
ターミナルに表示：💾 Skill "docker-network-debug" created
```

### 設計特性

- **ユーザーをブロックしない**：ユーザーへの返信後に起動、対話遅延を生まない
- **メイン対話を変更しない**：バックグラウンド agent は独立実行、メイン agent のメッセージ履歴には影響しない
- **メモリストア共有**：バックグラウンド agent はメイン agent と `_memory_store` を共有、スキル書き込みは即座に利用可能
- **Memory Nudge と統合可能**：skill review と memory review が同時発火した場合、統合プロンプトで一括処理

### 手動作成との違い

| | 手動作成（ユーザー指示） | 自動作成（Background Review） |
|---|---|---|
| 発火方法 | ユーザーが「skill を作って」と言う | システムカウンタの自動発火 |
| 内容ソース | ユーザー指定 | バックグラウンド agent が対話から抽出 |
| 品質 | ユーザーがコントロール | agent が自律判断、作成することもスキップすることもある |
| LLM 消費 | メイン対話の一部 | 追加消費（バックグラウンド agent は最大 8 回イテレーション） |

## Curator — バックグラウンドスキルメンテナンス（v2026.4.23+）

新規追加された**補助モデル駆動のバックグラウンドメンテナンス機構**（`agent/curator.py`、869 行 + `hermes_cli/curator.py`、235 行 + `tools/skill_usage.py`）。**agent が作成した**スキルを定期的に審査し、使用状況を追跡し、アイドル状態の skill を状態機械を介してアーカイブします。

### 不変条件（load-bearing invariants）

- **bundled または hub-installed スキルには絶対に触れない**（`.bundled_manifest` + `.hub/lock.json` の二重フィルタ）
- **絶対に自動削除しない** — アーカイブのみ、`hermes curator restore <skill>` で復元可能
- **Pinned skills は全自動変換をスキップ**：`tools/skill_manager_tool.py:_pinned_guard()` が `skill_manage` 書き込みパスで pinned skill の修正をインターセプト
- aux client を使用、**メイン session の prompt cache を絶対に汚染しない**

### 発火ロジック

デフォルトで有効、**inactivity-triggered**（cron デーモンなし）：CLI 起動 + gateway 起動時にチェック、以下 2 条件を満たすと実行：
1. 前回実行が `interval_hours`（デフォルト `24 * 7 = 168`、7 日、`agent/curator.py:39`）以上前
2. agent が `min_idle_hours`（デフォルト `2`、`agent/curator.py:40`）以上アイドル

Gateway モードでは cron-ticker スレッドに hook して定期チェック。

### 状態機械

```
active ──N 日未使用──> stale ──さらに未使用──> archived
   ↑                                              │
   └──────── 再使用 ──────────────────────────────┘
```

純関数的（`agent/curator.py` 内の state-machine 遷移）、LLM 呼び出しなし。Forked AIAgent は**重複統合 + ドリフト修正**が必要な時のみ介入。

### sidecar telemetry

`tools/skill_usage.py` は各 skill に `.usage.json` sidecar ファイルを維持：
- アトミック書き込み + provenance フィルタ
- 使用回数と最近使用時刻を記録、状態機械の入力信号となる

### CLI

```bash
hermes curator status        # 現在の状態、保留中の skill
hermes curator run           # 即座に 1 ラウンド実行
hermes curator pause/resume  # 一時停止 / 再開
hermes curator pin <skill>   # 特定 skill を pin（自動変換をスキップ）
hermes curator unpin <skill>
hermes curator restore <skill>  # アーカイブから復元
```

`/curator` スラッシュコマンドが同じサブコマンドを公開。

## /reload-skills と /reload-mcp（v2026.4.23+）

**`/reload-skills`**：`~/.hermes/skills/` を再スキャンして新規インストール / アンインストールされた skill を発見、プロセス再起動不要。**ユーザー起動の rescan** — prompt cache をリセットしない（skill は必要時に `/skill-name`、`skills_list`、`skill_view` 経由で呼び出されるので、システムプロンプトに常駐する必要なし）。再スキャン後、next-turn note で agent に通知、各追加 / 削除された skill に 60 文字の説明が付きます。

> 補足：元の PR には `skills_reload` agent ツールが含まれていましたが、後続の refactor（`dd2d1ba5e`）で明示的に削除されました — agent は既に `skill_view` / `skills_list` を通じてディスク上の新規インストール skill を見られるため、追加の schema surface は不要。

**`/reload-mcp` に確認プロンプト追加**：MCP 再読込は prompt cache を無効化するため、gateway は確認ダイアログを出すように（「今後確認しない」オプトアウトオプション付き）、誤操作で高価なキャッシュを消すのを防ぎます。

## Pinned skills の書き込み拒否（v2026.4.23+）

`tools/skill_manager_tool.py:134` に `_pinned_guard(name)` を新規追加、`skill_manage` の create / update / archive / delete パスで pinned skill の修正をインターセプト：

```python
if rec.get("pinned"):
    return f"Skill '{name}' is pinned and cannot be modified by skill_manage..."
```

これは Curator の不変条件の延長 — pinned 状態は agent にとっても禁区、`hermes curator unpin` で明示的にロック解除する必要があります。

## 関連ページ

- [[prompt-builder-architecture]] — スキルインデックスの構築と条件付き活性化
- [[skills-and-memory-interaction]] — スキルと記憶の相互作用設計
- [[security-defense-system]] — スキルのセキュリティスキャンと信頼レベル戦略

## 関連ファイル

- `tools/skills_tool.py` — スキルツール実装（1378 行）
- `agent/prompt_builder.py` — Prompt 構築とスキルインデックス
- `agent/skill_utils.py` — スキル解析ユーティリティ関数
- `agent/skill_commands.py` — スキルスラッシュコマンド
- `tools/skills_sync.py` — スキル同期機構
- `tools/skills_hub.py` — スキルハブ（検索 / インストール）
- `tools/skill_manager_tool.py` — スキル管理ツール
