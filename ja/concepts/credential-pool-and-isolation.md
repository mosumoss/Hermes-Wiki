---
title: 認証情報プールと環境隔離システム
created: 2026-04-07
updated: 2026-04-11
type: concept
tags: [architecture, credentials, security, isolation]
sources: [agent/credential_pool.py, hermes_cli/auth.py]
translation: ja
original: ../../concepts/credential-pool-and-isolation.md
---

# 認証情報プールと環境隔離システム

## 設計原理

企業利用では複数の API キーで以下を実現する必要があります：
1. **負荷分散** — 複数キーでリクエストを分散
2. **フェイルオーバー** — キーがレート制限された時に自動切替
3. **コスト管理** — キーごとに異なる予算

Hermes は**認証情報プールシステム**を実装し、複数キーの自動ローテーションをサポートします。

## 認証情報プールアーキテクチャ

コアデータ構造は `agent/credential_pool.py`（`tools/` ではない）に：

- **`PooledCredential`** — 単一の認証情報エントリ（dataclass）。`runtime_api_key`、`runtime_base_url`、枯渇状態、カウントを含む
- **`CredentialPool`** — 認証情報プール。複数認証情報の選択、ローテーション、復旧を管理

### 4 種類のプール選択戦略

```yaml
# config.yaml
credential_pool:
  strategy: round_robin  # デフォルト
```

| 戦略 | 動作 |
|------|------|
| `fill_first` | 最初のキーが枯渇するまで使い続け、その後次に切替 |
| `round_robin` | 順番にローテーション、均等に分担 |
| `random` | ランダムに利用可能なものを選ぶ |
| `least_used` | 使用回数が最も少ないものを選ぶ |

### 主要メソッド

- `select()` — 戦略に従って次の利用可能な認証情報を選択
- `mark_exhausted(entry)` — 枯渇マーク + 自動ローテーション（枯渇 TTL は 1 時間、期限切れで自動復旧）
- `try_refresh(entry)` — OAuth token のリフレッシュ
- `has_available()` — 利用可能な認証情報がまだあるか

## 認証情報ローテーションロジック

```python
# 402（請求枯渇）— 即座にローテーション
if status_code == 402:
    next_entry = pool.mark_exhausted_and_rotate(status_code=402, ...)
    if next_entry:
        self._swap_credential(next_entry)
        return True, False

# 429（レート制限）— 1 回目は再試行、2 回目でローテーション
if status_code == 429:
    if not has_retried_429:
        return False, True  # 同じ認証情報で再試行
    next_entry = pool.mark_exhausted_and_rotate(status_code=429, ...)
    if next_entry:
        self._swap_credential(next_entry)
        return True, False

# 401（未認可）— 先にリフレッシュ、失敗ならローテーション
if status_code == 401:
    refreshed = pool.try_refresh_current()
    if refreshed:
        self._swap_credential(refreshed)
        return True, has_retried_429
    # リフレッシュ失敗 — ローテーション
    next_entry = pool.mark_exhausted_and_rotate(status_code=401, ...)
    if next_entry:
        self._swap_credential(next_entry)
        return True, False
```

## 認証情報交換

```python
def _swap_credential(self, entry) -> None:
    """認証情報を交換"""
    runtime_key = getattr(entry, "runtime_api_key", None)
    runtime_base = getattr(entry, "runtime_base_url", None) or self.base_url
    
    if self.api_mode == "anthropic_messages":
        self._anthropic_client.close()
        self._anthropic_api_key = runtime_key
        self._anthropic_base_url = runtime_base
        self._anthropic_client = build_anthropic_client(runtime_key, runtime_base)
        self._is_anthropic_oauth = _is_oauth_token(runtime_key)
        self.api_key = runtime_key
        self.base_url = runtime_base
        return
    
    # OpenAI 互換モード
    self.api_key = runtime_key
    self.base_url = runtime_base.rstrip("/")
    self._client_kwargs["api_key"] = self.api_key
    self._client_kwargs["base_url"] = self.base_url
    self._replace_primary_openai_client(reason="credential_rotation")
```

## 環境隔離

```python
# HERMES_HOME 隔離
def get_hermes_home() -> Path:
    """Hermes ホームディレクトリを取得（Profile オーバーライド対応）"""
    env_override = os.getenv("HERMES_HOME")
    if env_override:
        return Path(env_override)
    return Path.home() / ".hermes"

# Profile サポート
# ~/.hermes/ がデフォルト Profile
# HERMES_HOME=/path/to/custom でカスタム Profile
```

### Profile 隔離の内容

| 内容 | 隔離 | 共有 |
|------|------|------|
| 設定 (config.yaml) | ✅ | ❌ |
| キー (.env) | ✅ | ❌ |
| スキル (~/.hermes/skills/) | ✅ | ❌ |
| 記憶 (~/.hermes/memories/) | ✅ | ❌ |
| セッション DB | ✅ | ❌ |
| コードリポジトリ | ❌ | ✅ |

## ターミナルバックエンド環境隔離

```python
# tools/environments/
# 各ターミナルバックエンドが隔離された実行環境を提供

local.py      # ローカル実行（ファイルシステム共有）
docker.py     # Docker コンテナ隔離
ssh.py        # SSH リモート実行
modal.py      # Modal サーバーレス隔離
daytona.py    # Daytona サンドボックス隔離
singularity.py # Singularity コンテナ隔離
```

## 優位性分析

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | OpenCode |
|------|--------|--------|----------|
| 認証情報プール | ✅ 複数キーローテーション | ❌ | ❌ |
| 自動フェイルオーバー | ✅ 402/429/401 | ❌ | ❌ |
| OAuth リフレッシュ | ✅ 自動 | ❌ | ❌ |
| Profile 隔離 | ✅ HERMES_HOME | ❌ | ❌ |
| ターミナルバックエンド隔離 | ✅ 6 種 | ❌ | ✅ Docker |

## 関連ページ

- [[interrupt-and-fault-tolerance]] — 中断伝播と耐障害機構（認証情報ローテーションロジック）
- [[auxiliary-client-architecture]] — 補助クライアントは認証情報プールを使って認証取得
- [[configuration-and-profiles]] — Profile 隔離と認証情報管理

## 関連ファイル

- `agent/credential_pool.py` — 認証情報プール（4 戦略 + 枯渇復旧）
- `hermes_cli/auth.py` — 認証情報解析
- `tools/environments/` — ターミナルバックエンド環境
