---
title: Cron スケジューリングと自動化ワークフロー
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [architecture, cron, automation, scheduling]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/cron-scheduling.md
---

# Cron スケジューリングと自動化ワークフロー

## 設計原理

Hermes は内蔵 Cron スケジューラを持ち、**自然言語による定期タスク**をサポートします。繰り返し作業を自動実行して、結果を任意のプラットフォームに配信できます。

## Cron ツール

```python
# tools/cronjob_tools.py

def cronjob(
    action: str,           # create/list/update/pause/resume/remove
    prompt: str = None,    # タスクプロンプト
    schedule: str = None,  # スケジュール式
    name: str = None,      # タスク名
    deliver: str = None,   # 配信先
    job_id: str = None,    # タスク ID
) -> dict:
    """定期タスクを管理"""
    
    if action == "create":
        return _create_job(prompt, schedule, name, deliver)
    elif action == "list":
        return _list_jobs()
    elif action == "update":
        return _update_job(job_id, prompt, schedule, name, deliver)
    elif action == "pause":
        return _pause_job(job_id)
    elif action == "resume":
        return _resume_job(job_id)
    elif action == "remove":
        return _remove_job(job_id)
```

## スケジューラ

スケジューラは**モジュールレベル関数**アーキテクチャ（クラスではない）。Gateway が 60 秒ごとに `tick()` を呼び出して駆動：

```python
# cron/scheduler.py — モジュールレベル関数アーキテクチャ

def tick():
    """Gateway から 60 秒ごとに呼ばれ、期限到達タスクをチェック・実行"""
    now = datetime.now()
    jobs = _load_jobs()  # jobs.json からロード
    for job in jobs.values():
        if _should_run(job, now):
            run_job(job)

def run_job(job: dict):
    """単一タスクを実行"""
    # 新しい Agent インスタンスを作成
    agent = AIAgent(
        model=job.get("model"),
        platform="cron",
        enabled_toolsets=job.get("toolsets", ["terminal", "web", "file"]),
    )
    
    # タスク実行
    result = agent.run_conversation(job["prompt"])
    
    # 結果配信
    if job.get("deliver"):
        _deliver_result(job["deliver"], result)

async def _deliver_result(target: str, result: dict):
    """結果を対象プラットフォームに配信"""
    ...
```

## タスクデータ構造

タスクは**純粋な dict** として `jobs.json` に保存（クラスではない）：

```python
# cron/jobs.py — タスクは純 dict、jobs.json に保存

# タスク dict 構造例
job = {
    "id": "daily-report",
    "prompt": "今日の業務サマリーレポートを生成",
    "schedule": "0 18 * * *",       # cron 式
    "name": "daily-report",
    "deliver": "telegram",
    "model": "gpt-4",
    "toolsets": ["terminal", "web", "file"],
    "is_paused": False,
    "created_at": "2026-04-07T10:00:00",
    "last_run": None,
    "next_run": "2026-04-07T18:00:00",
}

# スケジュール式のサポート形式:
# - cron: "0 9 * * *" （毎日 9 時）
# - 相対: "30m", "every 2h", "daily"
# - ISO: "2026-04-08T09:00:00"
```

## 配信先

```python
# 既知の配信プラットフォーム
_KNOWN_DELIVERY_PLATFORMS = {
    "telegram", "discord", "slack", "whatsapp", "signal",
    "matrix", "mattermost", "homeassistant",
    "dingtalk", "feishu", "wecom",
    "sms", "email", "webhook",
}

async def _deliver_result(target: str, result: dict):
    """結果を対象に配信"""
    if target == "origin":
        # 元のチャットに返信（Gateway 経由）
        await self.gateway.send_message(result["final_response"])
    elif target == "local":
        # ローカルファイルに保存
        output_dir = get_hermes_home() / "cron" / "output"
        output_dir.mkdir(parents=True, exist_ok=True)
        output_file = output_dir / f"{self.job_id}.txt"
        output_file.write_text(result["final_response"])
    elif target in DELIVER_TARGETS:
        # プラットフォーム経由で送信
        await self.platform_send(target, result["final_response"])
```

## 使用例

```python
# 日次レポートタスクを作成
cronjob(
    action="create",
    name="daily-report",
    prompt="今日の業務サマリーレポートを生成。完了タスク、未対応事項、明日の計画を含める",
    schedule="0 18 * * *",  # 毎日 18:00
    deliver="telegram",
)

# 毎時チェックタスクを作成
cronjob(
    action="create",
    name="hourly-check",
    prompt="サーバー状態を確認、異常があればアラート送信",
    schedule="every 1h",
    deliver="origin",
)

# 単発タスクを作成
cronjob(
    action="create",
    name="backup-database",
    prompt="データベースをバックアップしてクラウドストレージにアップロード",
    schedule="2026-04-08T02:00:00",  # ISO 時刻
    deliver="local",
)
```

## Gateway 統合

```bash
# Gateway 起動（スケジューラを含む）
hermes gateway start

# Gateway は 60 秒ごとに scheduler.tick() を呼ぶ
# スケジューラは独立イベントループを持たず、Gateway 駆動
```

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Claude Code | Cursor |
|------|--------|-------------|--------|
| 内蔵スケジューラ | ✅ | ❌ | ❌ |
| 自然言語スケジューリング | ✅ | ❌ | ❌ |
| 複数プラットフォーム配信 | ✅ 14 プラットフォーム | ❌ | ❌ |
| Cron 式 | ✅ | ❌ | ❌ |
| 相対時刻 | ✅ "30m", "every 2h" | ❌ | ❌ |
| タスク管理 | ✅ CLI/Gateway | ❌ | ❌ |

## 設定

```yaml
# ~/.hermes/config.yaml
cron:
  enabled: true
  timezone: "Asia/Tokyo"
  output_dir: "~/.hermes/cron/output"
```

## 関連ページ

- [[messaging-gateway-architecture]] — Gateway がスケジューラ tick() ループを駆動
- [[hook-system-architecture]] — Gateway イベントフックと Cron タスクの連携
- [[gateway-session-management]] — セッション origin が Cron 配信ルーティングに使用される

## 関連ファイル

- `tools/cronjob_tools.py` — Cron ツール
- `cron/scheduler.py` — スケジューラ
- `cron/jobs.py` — タスク定義
- `gateway/run.py` — Gateway 統合
