---
title: MemoryStore Class
created: 2026-04-07
updated: 2026-04-07
type: entity
tags: [component, memory, module]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../entities/memorystore-class.md
---

# MemoryStore Class

## 場所

`tools/memory_tool.py`

## 概要

MemoryStore は記憶システムのコアクラスで、MEMORY.md と USER.md の読み書き操作を管理します。

## コンストラクタ

```python
class MemoryStore:
    def __init__(self, memory_char_limit=2200, user_char_limit=1375):
        self.memory_entries: List[str] = []
        self.user_entries: List[str] = []
        self.memory_char_limit = memory_char_limit
        self.user_char_limit = user_char_limit
        self._system_prompt_snapshot: Dict[str, str] = {"memory": "", "user": ""}
```

## コアメソッド

### `load_from_disk()`

ディスクからエントリをロードし、フリーズスナップショットを取得。

### `add(target, content) -> Dict`

新規エントリを追加、重複と文字数制限をチェック。

### `replace(target, old_text, new_content) -> Dict`

短く一意な部分文字列マッチを使ってエントリを置換。

### `remove(target, old_text) -> Dict`

指定テキストを含むエントリを削除。

### `format_for_system_prompt(target) -> Optional[str]`

システムプロンプト注入用のフリーズスナップショットを返す。

## 重要な設計

- **フリーズスナップショット方式** — システムプロンプトはセッション中不変
- **アトミック書き込み** — 一時ファイル + `os.replace()` で整合性を保証
- **ファイルロック** — 並列安全のため `fcntl.flock()` を使用
- **セキュリティスキャン** — 注入・漏洩パターンを検出

## 関連ページ

- [[memory-system-architecture]] — 記憶システム全体アーキテクチャ
- [[skills-and-memory-interaction]] — スキルと記憶の相互作用設計
- [[security-defense-system]] — 記憶内容セキュリティスキャン

## 関連ファイル

- `tools/memory_tool.py` — 実装
- `agent/memory_manager.py` — マネージャ
- `agent/prompt_builder.py` — システムプロンプト統合
