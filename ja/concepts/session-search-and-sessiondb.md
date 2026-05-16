---
title: Session Search and SessionDB
created: 2026-04-07
updated: 2026-04-18
type: concept
tags: [session-search, session-store, memory, architecture]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/session-search-and-sessiondb.md
---

# セッション検索と SessionDB

## 概要

`session_search` は**会話を跨いで対話を呼び戻す能力**を提供します。SQLite FTS5 全文検索 + LLM 要約生成を使用。

## SessionDB

```python
# hermes_state.py
class SessionDB:
    """SQLite セッションストア、FTS5 検索対応"""
    
    def __init__(self, db_path: str):
        # セッションテーブルと FTS5 仮想テーブルを作成
        ...
    
    def save_session(self, session_id, messages, ...):
        """セッションを DB に保存"""
    
    def search_sessions(self, query, ...):
        """FTS5 全文検索"""
```

## FTS5 検索

SQLite の FTS5 拡張を使って効率的な全文検索を実現：

```sql
-- FTS5 仮想テーブル（messages テーブルのインデックス）
CREATE VIRTUAL TABLE messages_fts USING fts5(
    content,
    content=messages,
    content_rowid=id
);

-- 検索クエリ
SELECT * FROM messages_fts WHERE messages_fts MATCH 'elevenlabs OR baseten OR funding';
```

検索構文サポート：
- **キーワード OR** — `elevenlabs OR baseten`
- **フレーズマッチ** — `"docker networking"`
- **論理演算** — `python NOT java`
- **前方一致** — `deploy*`

## Session Search ツール

```python
def session_search(query: str, role_filter: str = None, limit: int = 3):
    """
    過去の対話セッションを検索
    
    2 つのモード:
    1. クエリなし — 直近セッションを閲覧（タイトル、プレビュー、タイムスタンプ）
    2. クエリあり — キーワード検索 + LLM 要約生成
    """
```

### モード 1: 直近セッション閲覧

```text
パラメータなし呼び出し → 直近セッションリストを返す:
- セッションタイトル
- 内容プレビュー
- タイムスタンプ
LLM コストゼロ、即時返却
```

### モード 2: キーワード検索

```text
クエリ付き呼び出し → FTS5 検索 → LLM 要約生成:
- マッチしたメッセージを検索
- LLM がセッション内容を要約
- 構造化された要約を返す
```

## 検索のヒント

```text
最良の結果を得るには検索時に OR でキーワードを繋ぐ:
  elevenlabs OR baseten OR funding

FTS5 はデフォルトで AND を使うため、一部のキーワードしか言及していないセッションを取りこぼします。
広範な OR クエリで結果が出ない場合、単一キーワードを並列検索してみてください。
```

## Memory との違い

| 観点 | Memory | Session Search |
|------|--------|----------------|
| **内容** | 安定事実、嗜好 | 完全な対話履歴 |
| **容量** | 限定（~3,500 文字） | 無制限（SQLite） |
| **検索** | 毎ターン自動注入 | 必要時に検索 |
| **形式** | エントリリスト | 構造化対話 |
| **用途** | コア行動ガイダンス | コンテキスト呼び戻し |

## 使用シーン

```text
ユーザーが以下を発言した時:
- 「以前これやったよね」 → session_search
- 「いつだったか覚えてる...」 → session_search
- 「前回俺たちは...」 → session_search
- 「X について何をしたっけ？」 → session_search

以下を疑う時:
- 過去セッションに関連コンテキストが存在 → session_search
- ユーザーに繰り返し説明させない → session_search
```

## データフロー

```text
セッション終了
  ↓
SessionDB.save_session()
  ↓
SQLite + FTS5 インデックスに書き込み
  ↓
ユーザーが検索を発起
  ↓
FTS5 全文検索
  ↓
LLM が要約生成
  ↓
構造化結果を返す
```

## セッション削除と剪定

`delete_session()` と `prune_sessions()` は**カスケード削除ではなく orphan 戦略**を採用：

- 親 session 削除時、子 session の `parent_session_id` は `NULL` にセット（orphan 化）、一緒に削除しない
- 圧縮分裂で生まれた子 session は親 session がクリーンアップされても検索可能
- `prune_sessions(older_than_days=90)` は終了済 session のみクリーンアップ、アクティブ session は影響なし

設計意図：履歴データの完全性を保護、価値ある対話記録の誤削除を回避。

### 起動時の自動剪定 + VACUUM（v2026.4.18+）

`state.db` は以前無制限に成長していました — 重度ユーザー（gateway + cron）が 384MB / 982 sessions / 68K messages でパフォーマンス低下を報告。手動で `hermes sessions prune --older-than 7` + `VACUUM` 後 43MB に低減。v2026.4.18+ では起動時に自動実行：

```python
# hermes_state.py
class SessionDB:
    def vacuum(self): ...

    def maybe_auto_prune_and_vacuum(
        self,
        retention_days: int = 90,        # 90 日以上前の終了済 session をクリーンアップ
        min_interval_hours: int = 24,    # デフォルト 1 日 1 回
        vacuum: bool = True,
    ) -> Dict[str, Any]:
        """冪等：state_meta テーブルに last_auto_prune を記録、同一 HERMES_HOME 上のプロセス間共有ロック
        返り値 {'skipped', 'pruned', 'vacuumed', 'error'?}"""
```

- 新規 `state_meta` key/value テーブルで前回実行タイムスタンプを保管（key: `last_auto_prune`）
- 同一 `HERMES_HOME` 配下の全 Hermes プロセスで共有、`min_interval_hours` 内は no-op
- **スマート VACUUM**：`pruned > 0` の時のみ実際に VACUUM 実行（`hermes_state.py:1567`）、空クリーンアップで I/O 浪費しない
- 絶対に例外を投げない — 失敗は warning ログのみ、起動には影響しない

## `/usage` 表示にアカウント制限追加（v2026.4.18+）

`/usage` コマンドは既存の token テーブルの下に**アカウントレベル割当情報**（provider 側から返される残額、サイクル、レート制限）を追記：

- CLI（`cli.py`）：`concurrent.futures.ThreadPoolExecutor(max_workers=1)` + 10s タイムアウトで fetch、遅い provider でも prompt 入力を妨げない
- Gateway（`gateway/run.py`）：`asyncio.to_thread` 経由で fetch；agent 不在時は `billing_provider` / `billing_base_url` の永続化フィールドから provider を解決
- 新モジュール `agent/account_usage.py`（326 行）が `fetch_account_usage(provider, base_url, api_key)` と `render_account_usage_lines(snapshot, markdown)` の 2 エントリポイントを提供

## 関連ページ

- [[gateway-session-management]] — Gateway セッション管理（SessionStore が SessionDB を使用）
- [[cli-architecture]] — CLI でのセッション管理と検索コマンド
- [[skills-and-memory-interaction]] — 第 3 の永続化機構としての Session Search

## 関連ファイル

- `hermes_state.py` — SessionDB 実装
- `tools/session_search_tool.py` — Session Search ツール
- `agent/trajectory.py` — 軌跡保存補助
