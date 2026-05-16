---
title: ターミナルバックエンドと環境抽象層
created: 2026-04-07
updated: 2026-04-29
type: concept
tags: [architecture, environments, terminal, isolation]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/terminal-backends.md
---

# ターミナルバックエンドと環境抽象層

## 設計原理

Hermes は 7 種類のターミナルバックエンドをサポートし、異なるレベルの隔離と永続化を提供します。統一された `terminal` ツール抽象により、Agent は異なるバックエンド間をシームレスに切り替えられます。

## バックエンドの種類

| バックエンド | 隔離レベル | 永続化 | 適用シーン |
|------|----------|--------|----------|
| **Local** | なし | ✅ ローカルディスク | 開発、個人利用 |
| **Docker** | コンテナ | ✅ ボリュームマウント | テスト、CI/CD |
| **SSH** | リモートホスト | ✅ リモートディスク | リモートサーバー |
| **Modal** | サーバーレス | ✅ スナップショット | クラウド実行、オンデマンド起動 |
| **Daytona** | サンドボックス | ✅ 永続化サンドボックス | セキュア実行 |
| **Singularity** | コンテナ | ✅ ボリュームマウント | HPC、研究 |
| **Vercel Sandbox** | microVM | ✅ スナップショット（task_id 単位） | クラウド microVM、FileSyncManager で認証情報 / スキル同期（v2026.4.23+） |

### Docker コンテナをホストユーザーで実行（v2026.4.23+）

`feat(docker): run container as host user` でコンテナ内プロセスがホストの UID/GID で起動するようになり、bind mount から出るファイルが root 所有になって sudo でしかクリーンアップできない問題を回避。

## ターミナルツール

```python
# tools/terminal_tool.py

def terminal(
    command: str,
    background: bool = False,
    timeout: int = 180,
    workdir: str = None,
    pty: bool = False,
) -> dict:
    """ターミナルコマンドを実行"""
    
    # バックエンド種別を解析
    backend = os.getenv("TERMINAL_ENV", "local")
    
    # 対応バックエンドにディスパッチ
    if backend == "local":
        return _run_local(command, timeout, workdir)
    elif backend == "docker":
        return _run_docker(command, timeout, workdir)
    elif backend == "ssh":
        return _run_ssh(command, timeout, workdir)
    elif backend == "modal":
        return _run_modal(command, timeout, workdir)
    elif backend == "daytona":
        return _run_daytona(command, timeout, workdir)
    elif backend == "singularity":
        return _run_singularity(command, timeout, workdir)
```

## 統一実行モデル：Spawn-per-call

すべての 6 バックエンドは同一の実行モデルを共有 — **各コマンドが独立して `bash -c` プロセスを spawn**、session snapshot で環境一貫性を保つ：

```text
初期化時:
  login shell → session snapshot を取得（env vars、functions、aliases）

各コマンド実行時:
  spawn bash -c → snapshot を source → コマンド実行 → CWD を取得 → 終了
```

**BaseEnvironment**（`tools/environments/base.py`）が統一インターフェースを定義：

- `init_session()` — login shell を 1 回起動、環境スナップショット取得
- `_wrap_command(cmd)` — snapshot source + CWD 追跡マーカーを注入
- `execute(cmd)` — 統一エントリ：wrap → spawn → 待機 → `{output, returncode}` を返す
- `_run_bash(wrapped_cmd)` → 抽象メソッド、各バックエンドが具体的なプロセス作成を実装

**CWD の呼び出し間永続化**は出力マーカーで実現：
- ローカルバックエンド：一時ファイル
- リモートバックエンド（Docker / SSH / Modal）：stdout 内のインラインマーカー

> 注：旧版の `PersistentShellMixin`（`persistent_shell.py`）は 2026-04-09 に削除され、spawn-per-call + session snapshot で完全に置き換えられました。

## 環境コンテキスト

```python
# environments/tool_context.py

class ToolContext:
    """ツール実行コンテキスト"""
    
    def __init__(self, environment: BaseEnvironment):
        self.environment = environment
        self.working_directory = "/root"
        self.env_vars = {}
    
    async def run_command(self, command: str, **kwargs) -> dict:
        return await self.environment.run_command(
            command,
            workdir=self.working_directory,
            env=self.env_vars,
            **kwargs
        )
```

## 統一ファイル同期（file_sync.py、2026-04-10）

SSH / Modal / Daytona バックエンドはローカルとリモート環境間で `tools/environments/file_sync.py` を使ってファイル（認証情報、スキル、キャッシュなど）を同期。Docker / Singularity は bind mount を使うため不要。

- **変更検出**：mtime + ファイルサイズベース、変更のあったファイルのみアップロード
- **削除検出**：ローカルファイル削除後、対応するリモートファイルもクリーンアップ
- **トランザクションロールバック**：アップロード / 削除のいずれかが失敗したら前回状態にロールバック、次回再試行
- **レート制限**：デフォルト 5 秒に 1 回同期（`HERMES_FORCE_FILE_SYNC=1` で毎回強制同期）

## バックグラウンドプロセス監視（watch_patterns、2026-04-10）

`terminal` ツールに `watch_patterns` パラメータ追加。バックグラウンドプロセスの出力が指定文字列にマッチした時、リアルタイムで agent に通知：

```python
terminal(command="pytest -v", background=True, watch_patterns=["ERROR", "FAIL", "listening on port"])
```

| パラメータ | 値 |
|------|-----|
| マッチ方式 | 部分文字列マッチ（正規表現ではない） |
| レート制限 | 10 秒ウィンドウで最大 8 回通知 |
| 過負荷保護 | 45 秒継続過負荷で自動無効化 |
| 出力切り捨て | 最大 20 行、2,000 文字 |

通知は `ProcessRegistry.completion_queue` で CLI / Gateway のメインループに伝達され、agent の自動応答をトリガー。

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | Claude Code |
|------|--------|--------|-------------|
| バックエンド数 | ✅ 6 種 | ❌ 1 | ❌ 1 |
| サーバーレスサポート | ✅ Modal | ❌ | ❌ |
| サンドボックス隔離 | ✅ Daytona | ❌ | ❌ |
| HPC サポート | ✅ Singularity | ❌ | ❌ |
| Session Snapshot | ✅ | ❌ | ❌ |
| 環境スナップショット | ✅ Modal | ❌ | ❌ |

## 設定ファイル

```yaml
# ~/.hermes/config.yaml
terminal:
  backend: "local"  # local/docker/ssh/modal/daytona/singularity
  
  docker:
    image: "ubuntu:22.04"
    volumes: ["~/work:/root/work"]
  
  ssh:
    host: "remote-server"
    user: "ubuntu"
    key_path: "~/.ssh/id_rsa"
  
  modal:
    app_name: "hermes-agent"
    image: "python:3.11"
  
  daytona:
    api_key: "${DAYTONA_API_KEY}"
    image: "ubuntu:22.04"
```

## 関連ページ

- [[credential-pool-and-isolation]] — 認証情報プールと環境隔離（ターミナルバックエンド環境）
- [[multi-agent-architecture]] — サブエージェントが独立ターミナルバックエンドで実行
- [[tool-registry-architecture]] — ターミナルツールが registry 経由で登録

## 関連ファイル

- `tools/terminal_tool.py` — ターミナルツール
- `tools/environments/` — 6 バックエンドの実装
- `environments/tool_context.py` — ツール実行コンテキスト
