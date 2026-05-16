---
title: ファジーマッチングエンジン — 8 戦略チェーン
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [architecture, tool, reliability, fuzzy-matching]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/fuzzy-matching-engine.md
---

# ファジーマッチングエンジン — 8 戦略チェーン

## 設計原理

Agent がファイルを修正する時、置換対象のテキストを見つける必要があります。LLM が生成するテキストは元と微妙な差異（空白、インデント、エスケープシーケンス等）があり得ます。Hermes は **8 戦略チェーン**を実装し、完全一致から徐々にファジーマッチングへダウングレード、マッチング成功率を最大化します。

OpenCode のファジーマッチング実装にインスパイア。

## 8 戦略チェーン

```python
strategies = [
    ("exact", _strategy_exact),                    # 1. 完全一致
    ("line_trimmed", _strategy_line_trimmed),      # 2. 行ごと修剪
    ("whitespace_normalized", _strategy_whitespace_normalized),  # 3. 空白正規化
    ("indentation_flexible", _strategy_indentation_flexible),    # 4. インデント柔軟
    ("escape_normalized", _strategy_escape_normalized),          # 5. エスケープ正規化
    ("trimmed_boundary", _strategy_trimmed_boundary),            # 6. 境界修剪
    ("block_anchor", _strategy_block_anchor),      # 7. ブロックアンカー
    ("context_aware", _strategy_context_aware),    # 8. コンテキスト感知
]

for strategy_name, strategy_fn in strategies:
    matches = strategy_fn(content, old_string)
    if matches:
        # マッチ発見 → 置換実行
        if len(matches) > 1 and not replace_all:
            return content, 0, f"{len(matches)} 個マッチ、より多くのコンテキストを提供してください"
        new_content = _apply_replacements(content, matches, new_string)
        return new_content, len(matches), None

# 全戦略失敗
return content, 0, "マッチが見つかりません"
```

## 戦略詳細

### 戦略 1：完全一致

```python
def _strategy_exact(content, pattern):
    """直接文字列マッチ"""
    matches = []
    start = 0
    while True:
        pos = content.find(pattern, start)
        if pos == -1:
            break
        matches.append((pos, pos + len(pattern)))
        start = pos + 1
    return matches
```

**適用シーン**：LLM 生成テキストが元と完全一致

### 戦略 2：行ごと修剪

```python
def _strategy_line_trimmed(content, pattern):
    """各行の先頭末尾空白を除去してマッチ"""
    pattern_lines = [line.strip() for line in pattern.split('\n')]
    pattern_normalized = '\n'.join(pattern_lines)
    
    content_lines = content.split('\n')
    content_normalized_lines = [line.strip() for line in content_lines]
    
    # 正規化コンテンツで検索、元位置にマップバック
    return _find_normalized_matches(...)
```

**適用シーン**：LLM 生成テキストの各行先頭末尾に余分な空白

### 戦略 3：空白正規化

```python
def _strategy_whitespace_normalized(content, pattern):
    """複数の空白 / タブを単一空白に折り畳む"""
    def normalize(s):
        return re.sub(r'[ \t]+', ' ', s)
    
    pattern_normalized = normalize(pattern)
    content_normalized = normalize(content)
    
    # 正規化コンテンツで検索、元位置にマップバック
    return _map_normalized_positions(content, content_normalized, matches)
```

**適用シーン**：LLM 生成テキストの空白数が一貫しない

### 戦略 4：インデント柔軟

```python
def _strategy_indentation_flexible(content, pattern):
    """インデント差を完全に無視"""
    content_stripped_lines = [line.lstrip() for line in content.split('\n')]
    pattern_lines = [line.lstrip() for line in pattern.split('\n')]
    
    # 全先頭空白を除去してマッチ
    return _find_normalized_matches(...)
```

**適用シーン**：LLM 生成テキストのインデントレベルが異なる

### 戦略 5：エスケープ正規化

```python
def _strategy_escape_normalized(content, pattern):
    """エスケープシーケンスを実文字に変換"""
    def unescape(s):
        return s.replace('\\n', '\n').replace('\\t', '\t').replace('\\r', '\r')
    
    pattern_unescaped = unescape(pattern)
    if pattern_unescaped == pattern:
        return []  # エスケープシーケンスなし、スキップ
    
    return _strategy_exact(content, pattern_unescaped)
```

**適用シーン**：LLM 生成テキストにリテラルエスケープシーケンスが含まれる

### 戦略 6：境界修剪

```python
def _strategy_trimmed_boundary(content, pattern):
    """先頭行と末尾行の空白のみ修剪"""
    pattern_lines = pattern.split('\n')
    pattern_lines[0] = pattern_lines[0].strip()
    if len(pattern_lines) > 1:
        pattern_lines[-1] = pattern_lines[-1].strip()
    
    # コンテンツ内でスライディングウィンドウマッチ
    for i in range(len(content_lines) - pattern_line_count + 1):
        block_lines = content_lines[i:i + pattern_line_count]
        check_lines = block_lines.copy()
        check_lines[0] = check_lines[0].strip()
        if len(check_lines) > 1:
            check_lines[-1] = check_lines[-1].strip()
        
        if '\n'.join(check_lines) == modified_pattern:
            matches.append(...)
```

**適用シーン**：先頭末尾行のみ空白差異

### 戦略 7：ブロックアンカー

```python
def _strategy_block_anchor(content, pattern):
    """先頭末尾行ベースでアンカリング、中間部は類似度マッチ"""
    # Unicode 正規化
    norm_pattern = _unicode_normalize(pattern)
    norm_content = _unicode_normalize(content)
    
    pattern_lines = norm_pattern.split('\n')
    first_line = pattern_lines[0].strip()
    last_line = pattern_lines[-1].strip()
    
    # 先頭末尾行マッチ位置を検索
    for i in range(len(norm_content_lines) - pattern_line_count + 1):
        if (norm_content_lines[i].strip() == first_line and 
            norm_content_lines[i + pattern_line_count - 1].strip() == last_line):
            
            # 中間部の類似度計算
            content_middle = '\n'.join(norm_content_lines[i+1:i+pattern_line_count-1])
            pattern_middle = '\n'.join(pattern_lines[1:-1])
            similarity = SequenceMatcher(None, content_middle, pattern_middle).ratio()
            
            # 閾値：唯一マッチ 0.10、複数候補 0.30
            threshold = 0.10 if candidate_count == 1 else 0.30
            if similarity >= threshold:
                matches.append(...)
```

**適用シーン**：先頭末尾行マッチ、中間内容に微妙な差異

### 戦略 8：コンテキスト感知

```python
def _strategy_context_aware(content, pattern):
    """行ごと類似度マッチ、50% 閾値"""
    pattern_lines = pattern.split('\n')
    content_lines = content.split('\n')
    
    for i in range(len(content_lines) - pattern_line_count + 1):
        block_lines = content_lines[i:i + pattern_line_count]
        
        # 行ごと類似度計算
        high_similarity_count = 0
        for p_line, c_line in zip(pattern_lines, block_lines):
            sim = SequenceMatcher(None, p_line.strip(), c_line.strip()).ratio()
            if sim >= 0.80:  # 単一行 80% 類似度
                high_similarity_count += 1
        
        # 少なくとも 50% の行が高類似度であること
        if high_similarity_count >= len(pattern_lines) * 0.5:
            matches.append(...)
```

**適用シーン**：全体内容に 50% 以上の行類似性

## Unicode 正規化

```python
UNICODE_MAP = {
    "“": '"', "”": '"',  # スマートダブルクォート
    "‘": "'", "’": "'",  # スマートシングルクォート
    "—": "--", "–": "-", # ダッシュ
    "…": "...", " ": " ", # 三点リーダーと改行禁止スペース
}

def _unicode_normalize(text: str) -> str:
    """Unicode 文字を標準 ASCII 等価物に正規化"""
    for char, repl in UNICODE_MAP.items():
        text = text.replace(char, repl)
    return text
```

## 優位性分析

### マッチング成功率

| シーン | 完全一致 | 8 戦略チェーン |
|------|----------|----------|
| 完全一致 | ✅ | ✅ |
| 先頭末尾空白差異 | ❌ | ✅ 戦略 2 / 6 |
| インデント差異 | ❌ | ✅ 戦略 4 |
| エスケープシーケンス差異 | ❌ | ✅ 戦略 5 |
| スマートクォート | ❌ | ✅ 戦略 7 |
| 中間内容微調整 | ❌ | ✅ 戦略 7 / 8 |

### 他のツールとの比較

| 特徴 | Hermes | sed/awk | Cursor |
|------|--------|---------|--------|
| 完全一致 | ✅ | ✅ | ✅ |
| 空白耐性 | ✅ 8 戦略 | ❌ | ✅ 部分 |
| Unicode 正規化 | ✅ | ❌ | ✅ |
| 類似度マッチング | ✅ | ❌ | ❌ |
| 位置マッピング | ✅ 精密 | N/A | ✅ |

## 関連ページ

- [[model-tools-dispatch]] — ツール編成とディスパッチ（ファジーマッチング呼び出しの上層）
- [[tool-registry-architecture]] — ツール登録システム（ファイルツールは registry 経由で登録）
- [[skills-system-architecture]] — スキル管理ツール内でファジーマッチング使用

## 関連ファイル

- `tools/fuzzy_match.py` — ファジーマッチングエンジン実装
- `tools/skill_manager_tool.py` — スキル管理でファジーマッチング呼び出し
- `tools/file_tools.py` — ファイルツールでファジーマッチング呼び出し
