---
title: 並列ツール実行システム
created: 2026-04-07
updated: 2026-04-07
type: concept
tags: [architecture, tool, performance, concurrency]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../concepts/parallel-tool-execution.md
---

# 並列ツール実行システム — 知的並行安全検出

## 設計原理

現代の LLM は 1 回の応答で複数のツール呼び出しを返すことが多いです（parallel tool calling）。Hermes の設計目標：**安全を確保した上で並列度を最大化、総待機時間を削減**。

従来の手法はすべて直列（遅い）か全て並列（危険）の二択でした。Hermes は**三層安全検出 + パスのスコープ解析**による知的並列戦略を採用。

## コアアーキテクチャ

### 1. ツール分類体系

```python
# 絶対に並列にできないツール（対話 / ユーザー向け）
_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})

# 読み取り専用ツール、共有可変状態なし
_PARALLEL_SAFE_TOOLS = frozenset({
    "ha_get_state", "ha_list_entities", "ha_list_services",
    "read_file", "search_files", "session_search",
    "skill_view", "skills_list",
    "vision_analyze", "web_extract", "web_search",
})

# ファイルツール、並列可能だがパスが衝突しないこと
_PATH_SCOPED_TOOLS = frozenset({"read_file", "write_file", "patch"})

# 最大並行ワーカースレッド数
_MAX_TOOL_WORKERS = 8
```

### 2. 並行安全検出アルゴリズム

```python
def _should_parallelize_tool_batch(tool_calls) -> bool:
    """ツール呼び出しバッチが安全に並列実行できるか判定"""
    
    # 1. 単一ツールは並列不要
    if len(tool_calls) <= 1:
        return False
    
    # 2. 絶対並列不可ツールを含む → 直列にダウングレード
    tool_names = [tc.function.name for tc in tool_calls]
    if any(name in _NEVER_PARALLEL_TOOLS for name in tool_names):
        return False
    
    # 3. パススコープチェック（ファイルツール）
    reserved_paths: list[Path] = []
    for tool_call in tool_calls:
        tool_name = tool_call.function.name
        function_args = json.loads(tool_call.function.arguments)
        
        if tool_name in _PATH_SCOPED_TOOLS:
            scoped_path = _extract_parallel_scope_path(tool_name, function_args)
            if scoped_path is None:
                return False  # パス解析不可 → 直列にダウングレード
            if any(_paths_overlap(scoped_path, existing) for existing in reserved_paths):
                return False  # パス衝突 → 直列にダウングレード
            reserved_paths.append(scoped_path)
            continue
        
        if tool_name not in _PARALLEL_SAFE_TOOLS:
            return False  # 未知ツール → 保守的に直列ダウングレード
    
    return True  # 全チェック通過 → 安全に並列
```

### 3. パス衝突検出

```python
def _paths_overlap(left: Path, right: Path) -> bool:
    """2 つのパスが同じサブツリーを指す可能性があるか判定"""
    left_parts = left.parts
    right_parts = right.parts
    
    # 短い方のパス長を共通プレフィックス長とする
    common_len = min(len(left_parts), len(right_parts))
    return left_parts[:common_len] == right_parts[:common_len]
```

**例：**
- `/root/wiki/index.md` と `/root/wiki/log.md` → 衝突なし（並列可）
- `/root/wiki/index.md` と `/root/wiki/index.md` → 衝突（直列）
- `/root/wiki/` と `/root/wiki/concepts/` → 衝突（直列、親ディレクトリが重複）

## 並列実行の実装

```python
if _should_parallelize_tool_batch(tool_calls):
    # 並列実行：ThreadPoolExecutor を使用
    with concurrent.futures.ThreadPoolExecutor(
        max_workers=_MAX_TOOL_WORKERS
    ) as executor:
        futures = {
            executor.submit(_execute_single_tool, tc): tc
            for tc in tool_calls
        }
        for future in concurrent.futures.as_completed(futures):
            tool_call = futures[future]
            result = future.result()
            # 結果処理...
else:
    # 直列実行：順番に処理
    for tool_call in tool_calls:
        result = _execute_single_tool(tool_call)
        # 結果処理...
```

## 安全ダウングレード戦略

Hermes は**保守的デフォルト**戦略を採用：あらゆる不確実性は直列にダウングレード。

| 検出項目 | 失敗条件 | ダウングレード理由 |
|--------|----------|----------|
| 絶対並列不可ツール | `clarify` を含む | ユーザー対話は直列必須 |
| パス解析失敗 | JSON パラメータ解析不可 | 安全性検証不能 |
| パス重複 | ファイルパスに共通プレフィックス | 競合状態を回避 |
| 未知ツール | 安全リストにない | 保守的デフォルト |
| 非辞書パラメータ | パラメータが dict ではない | スコープ解析不能 |

## 優位性分析

### パフォーマンス向上

| シーン | 直列時間 | 並列時間 | 加速比 |
|------|----------|----------|--------|
| 3 つの読み取り専用ツール | 3 × 待機時間 | max(待機時間) | ~3x |
| 5 つの独立ファイル操作 | 5 × 待機時間 | max(待機時間) | ~5x |
| 混合シーン（2 並列 + 1 直列）| 3 × 待機 | 2 × 待機 | ~1.5x |

### 安全性保証

1. **競合状態ゼロ** — パス重複検出で同時書き込みを防止
2. **保守的デフォルト** — 不確定時は直列ダウングレード、エラーを起こさない
3. **ユーザー対話保護** — `clarify` 等のツールは絶対に直列、混乱を回避
4. **最大スレッド制限** — 8 ワーカースレッドでリソース枯渇を防止

### 他の Agent フレームワークとの比較

| 特徴 | Hermes | Cursor/Claude | OpenCode |
|------|--------|---------------|----------|
| 並列ツール実行 | ✅ 知的検出 | ✅ 全並列 | ✅ 全並列 |
| パス衝突検出 | ✅ プレフィックス重複チェック | ❌ なし | ❌ なし |
| 保守的ダウングレード | ✅ 不確定なら直列 | ❌ 競合可能性 | ❌ 競合可能性 |
| 設定可能スレッド数 | ✅ _MAX_TOOL_WORKERS | ❌ 固定 | ❌ 固定 |

## 設定ガイド

### 環境変数

```bash
# 環境変数による制御なし、run_agent.py 内にハードコード
# 将来追加可能性:
# HERMES_MAX_TOOL_WORKERS=8
# HERMES_PARALLEL_TOOLS=true/false
```

### カスタム並列戦略

新規の並列安全ツールを追加する場合：

```python
# run_agent.py で修正:
_PARALLEL_SAFE_TOOLS = frozenset({
    # ... 既存ツール ...
    "your_new_read_only_tool",  # 読み取り専用ツール追加
})

# または新規パススコープツール追加:
_PATH_SCOPED_TOOLS = frozenset({
    # ... 既存ツール ...
    "your_file_tool",  # path パラメータを必要とするツール
})
```

## 関連ページ

- [[model-tools-dispatch]] — ツール編成とディスパッチ（並列実行の上層制御）
- [[tool-registry-architecture]] — ツール登録システムとメタデータ管理
- [[large-tool-result-handling]] — 並列ツールが大型結果を生成した時の処理

## 関連ファイル

- `run_agent.py` — 並列検出アルゴリズムと実行ロジック
- `tools/registry.py` — ツール登録とメタデータ
