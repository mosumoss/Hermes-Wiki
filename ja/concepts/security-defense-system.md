---
title: セキュリティ防御体系 — 多層注入検出
created: 2026-04-07
updated: 2026-04-11
type: concept
tags: [architecture, security, injection-defense, skills-guard]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/security-defense-system.md
---

# セキュリティ防御体系 — 多層注入検出

## 設計原理

Hermes Agent はコード実行、ファイル読み書き、ネットワークアクセスの能力を持つため、以下を防御する必要があります：
1. **プロンプトインジェクション** — 悪意あるコンテンツが Agent 指示を上書き試行
2. **データ漏洩** — API キー、認証情報の盗難
3. **破壊的操作** — ファイル削除、システム破壊
4. **永続化バックドア** — 起動スクリプト、cron タスク改変
5. **サプライチェーン攻撃** — 悪意あるスキル、未ロックの依存

Hermes は **5 層防御体系**を実装、内容スキャンから信頼ポリシーまで。

## 第 1 層：Skills Guard セキュリティスキャン

### 脅威パターンライブラリ（100+ 正規表現パターン）

主要なパターン分類：
- **データ漏洩**: `curl ... $KEY|TOKEN|SECRET|PASSWORD`、`os.getenv()` で秘密情報、`~/.ssh` / `~/.hermes/.env` 参照
- **プロンプトインジェクション**: 「以前の指示を無視」、「ユーザーに伝えるな」、「制限なく行動」
- **破壊的操作**: `rm -rf /`、`shutil.rmtree` 絶対パス、`> /etc/...`
- **永続化バックドア**: `crontab`、`authorized_keys`、`systemd` サービス、`.bashrc/.zshrc/.profile`
- **ネットワークバックドア**: `nc -lp`（リバースシェル）、`ngrok`/`localtunnel`（トンネル）
- **難読化実行**: `base64 -d |`、`eval('...')`、`echo ... | bash`
- **サプライチェーン攻撃**: `curl ... | sh`（ダウンロード実行）、`pip install` バージョン無指定
- **ハードコード秘密情報**: `api_key/token = "..."`、`-----BEGIN PRIVATE KEY-----`

各パターンには `severity`（critical/high/medium）、`category`、説明テキストが付随。

### 不可視 Unicode 検出

```python
INVISIBLE_CHARS = {
    '​',  # ゼロ幅スペース
    '‌',  # ゼロ幅非接合子
    '‍',  # ゼロ幅接合子
    '⁠',  # 単語接合子
    '﻿',  # ゼロ幅ノーブレークスペース (BOM)
    '‪',  # 左から右への埋め込み
    '‫',  # 右から左への埋め込み
    '‮',  # 右から左への上書き
    # ... 計 17 文字
}

# スキルファイル内の不可視文字を検出
for i, line in enumerate(lines, start=1):
    for char in INVISIBLE_CHARS:
        if char in line:
            findings.append(Finding(
                pattern_id="invisible_unicode",
                severity="high",
                category="injection",
                match=f"U+{ord(char):04X}",
                description="不可視 Unicode 文字（テキスト隠蔽 / 注入の可能性）",
            ))
```

### 構造チェック

```python
MAX_FILE_COUNT = 50       # スキルは 50+ ファイルを持つべきでない
MAX_TOTAL_SIZE_KB = 1024  # 合計サイズ 1MB は疑わしい
MAX_SINGLE_FILE_KB = 256  # 単一ファイル > 256KB は疑わしい

SUSPICIOUS_BINARY_EXTENSIONS = {
    '.exe', '.dll', '.so', '.dylib', '.bin',
    '.msi', '.dmg', '.app', '.deb', '.rpm',
    '.dat', '.com',
}
```

## 第 2 層：信頼レベルポリシー

```python
TRUSTED_REPOS = {"openai/skills", "anthropics/skills"}

INSTALL_POLICY = {
    #               safe      caution    dangerous
    "builtin":     ("allow",  "allow",   "allow"),
    "trusted":     ("allow",  "allow",   "block"),
    "community":   ("allow",  "block",   "block"),
    "agent-created":("allow", "allow",   "ask"),
}

VERDICT_INDEX = {"safe": 0, "caution": 1, "dangerous": 2}
```

### 判定ロジック

```python
def _determine_verdict(findings):
    if not findings:
        return "safe"
    
    has_critical = any(f.severity == "critical" for f in findings)
    has_high = any(f.severity == "high" for f in findings)
    
    if has_critical:
        return "dangerous"
    if has_high:
        return "caution"
    return "safe"  # medium/low のみ
```

### インストール判断

| ソース | safe | caution | dangerous |
|------|------|---------|-----------|
| builtin（内蔵） | allow | allow | allow |
| trusted（OpenAI/Anthropic） | allow | allow | block |
| community（コミュニティ） | allow | **block** | block |
| agent-created（Agent 作成） | allow | allow | **ask** |

## 第 3 層：Memory 内容スキャン

```python
_MEMORY_THREAT_PATTERNS = [
    # プロンプトインジェクション
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'act\s+as\s+(if|though)\s+.*no\s+(restrictions|limits)', "bypass_restrictions"),
    # 秘密情報漏洩
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc)', "read_secrets"),
    (r'base64\s+(-d|--decode)\s*\|', "base64_decode_pipe"),
    # 永続化バックドア
    (r'authorized_keys', "ssh_backdoor"),
    (r'\$HOME/\.ssh|\~/\.ssh', "ssh_access"),
    (r'crontab', "persistence_cron"),
    (r'\.(bashrc|zshrc|profile)', "shell_rc_mod"),
]

def _scan_memory_content(content: str) -> Optional[str]:
    """記憶内容をスキャン、脅威発見でエラー文字列を返す"""
    # 不可視 Unicode 検出
    for char in _INVISIBLE_CHARS:
        if char in content:
            return f"Blocked: 不可視 Unicode 文字 U+{ord(char):04X}"
    
    # 脅威パターン検出
    for pattern, pid in _MEMORY_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            return f"Blocked: 脅威パターン '{pid}' に一致"
    
    return None  # 安全
```

## 第 4 層：コンテキストファイル注入スキャン

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'you\s+are\s+now\s+', "role_hijack"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'act\s+as\s+(if|though)\s+.*no\s+(restrictions|limits)', "bypass_restrictions"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials)', "read_secrets"),
    (r'<!--[^>]*(?:ignore|override|system|secret|hidden)[^>]*-->', "html_comment_injection"),
    (r'<\s*div\s+style\s*=\s*["\'].*display\s*:\s*none', "hidden_div"),
    (r'base64\s+(-d|--decode)\s*\|', "base64_decode_pipe"),
]

def _scan_context_content(content: str, filename: str) -> str:
    """コンテキストファイル（SOUL.md, AGENTS.md 等）をスキャン"""
    findings = []
    
    # 不可視 Unicode 検出
    for char in _CONTEXT_INVISIBLE_CHARS:
        if char in content:
            findings.append(f"invisible unicode U+{ord(char):04X}")
    
    # 脅威パターン検出
    for pattern, pid in _CONTEXT_THREAT_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            findings.append(pid)
    
    if findings:
        logger.warning("Context file %s blocked: %s", filename, ", ".join(findings))
        return f"[BLOCKED: {filename} contained potential prompt injection ({', '.join(findings)}). Content not loaded.]"
    
    return content  # 安全、元コンテンツを返す
```

## 第 5 層：ターミナルコマンドヒューリスティック検出

```python
_DESTRUCTIVE_PATTERNS = re.compile(
    r"""(?:^|\s|&&|\|\||;|`)(?:
        rm\s|rmdir\s|
        mv\s|
        sed\s+-i|
        truncate\s|
        dd\s|
        shred\s|
        git\s+(?:reset|clean|checkout)\s
    )""",
    re.VERBOSE,
)

_REDIRECT_OVERWRITE = re.compile(r'[^>]>[^>]|^>[^>]')

def _is_destructive_command(cmd: str) -> bool:
    """ヒューリスティック：このターミナルコマンドはファイル修正 / 削除のようか？"""
    if not cmd:
        return False
    if _DESTRUCTIVE_PATTERNS.search(cmd):
        return True
    if _REDIRECT_OVERWRITE.search(cmd):
        return True
    return False
```

## セキュリティスキャン実行タイミング

| タイミング | スキャン内容 | スキャナ |
|------|----------|--------|
| スキル作成 | スキルディレクトリ全体 | Skills Guard |
| スキル編集 / パッチ | スキルディレクトリ全体 | Skills Guard |
| 記憶書き込み | エントリ内容 | Memory Scanner |
| コンテキストファイルロード | SOUL.md, AGENTS.md 等 | Context Scanner |
| スキルインストール（Hub） | スキルディレクトリ全体 | Skills Guard |

## ロールバック機構

```python
# スキル作成 / 編集後のスキャン
scan_error = _security_scan_skill(skill_dir)
if scan_error:
    # 修正前状態に自動ロールバック
    _atomic_write_text(target, original_content)
    return {"success": False, "error": scan_error}
```

## 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor | Claude Desktop |
|------|--------|--------|----------------|
| スキルセキュリティスキャン | ✅ 100+ パターン | N/A | N/A |
| 信頼レベルポリシー | ✅ 4 レベル | N/A | N/A |
| 記憶内容スキャン | ✅ | N/A | N/A |
| コンテキストファイルスキャン | ✅ | N/A | N/A |
| Unicode 注入検出 | ✅ 17 文字 | ❌ | ❌ |
| 自動ロールバック | ✅ | N/A | N/A |
| 破壊的コマンド検出 | ✅ ヒューリスティック | ❌ | ❌ |

## 危険コマンド承認システム（tools/approval.py — 877 行）

agent が実行するターミナルコマンドが危険パターンに一致した時、システムがインターセプトしてユーザー確認を要求。

### 3 つの承認モード

```yaml
# config.yaml
approvals:
  mode: smart   # manual | smart | off
```

| モード | 動作 |
|------|------|
| `manual` | 危険パターンに一致するコマンドすべて人間確認 |
| `smart` | 先に auxiliary LLM でリスク評価、低リスク自動許可、高リスクのみユーザーに確認 |
| `off`（yolo） | 全承認をスキップ（危険、信頼できる環境のみ） |

### 承認オプション（CLI 対話）

ユーザーは危険コマンドを見た後選択可能：
- **once** — 今回のみ許可
- **session** — 今回のセッション内の同種コマンドすべて許可
- **always** — 永久許可（config.yaml に書き込み）
- **deny** — 実行拒否

タイムアウト未応答（45 秒）→ デフォルト拒否（fail-closed）。

### 危険パターン検出

マッチルールがカバー：
- 破壊的操作：`rm -rf`、`mkfs`、`dd`、`truncate` 等
- 権限昇格：`sudo`、`su`、`chmod 777`
- 機密ファイル書き込み：`/etc/`、`~/.ssh/`、`~/.hermes/.env`
- ネットワーク操作：`curl | bash`、ポートリスニング
- 環境変数操作：`PATH`、`LD_PRELOAD` の上書き

### Per-session 状態

承認状態はセッション別に隔離（`contextvars.ContextVar`）、gateway 多ユーザー並行でも互いに影響しない。「session」レベルの許可は現在のセッションのみ有効、セッションを跨がない。

## 追加セキュリティ層

- `tools/tirith_security.py` — Tirith セキュリティポリシーエンジン（homograph URL、pipe-to-shell、ターミナル注入）
- `tools/url_safety.py` — URL セキュリティチェック（SSRF 防御：プライベートネットワーク・クラウドメタデータアドレスのインターセプト、リダイレクト検証）
- `tools/osv_check.py` — 依存マルウェアスキャン（OSV データベース）

## 関連ページ

- [[memory-system-architecture]] — 記憶内容セキュリティスキャン機構
- [[skills-system-architecture]] — スキルインストール時のセキュリティスキャンと信頼ポリシー
- [[prompt-builder-architecture]] — コンテキストファイル注入スキャン防御

## 関連ファイル

- `tools/skills_guard.py` — Skills Guard セキュリティスキャン
- `tools/memory_tool.py` — Memory 内容スキャン
- `agent/prompt_builder.py` — コンテキストファイルスキャン
- `run_agent.py` — ターミナルコマンドヒューリスティック検出
- `tools/approval.py` — コマンド承認（31 パターン）
- `tools/tirith_security.py` — Tirith セキュリティポリシー
- `tools/url_safety.py` — SSRF 防御
- `tools/osv_check.py` — マルウェアスキャン
