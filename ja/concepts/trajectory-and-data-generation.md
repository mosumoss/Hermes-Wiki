---
title: 軌跡保存と訓練データ生成
created: 2026-04-07
updated: 2026-04-14
type: concept
tags: [architecture, data-generation, training, trajectory, batch-runner]
sources: [agent/trajectory.py, batch_runner.py, toolset_distributions.py, environments/, run_agent.py]
translation: ja
original: ../../concepts/trajectory-and-data-generation.md
---

# 軌跡（trajectory）保存と訓練データ生成

## このシステムが解決する問題

Hermes リポジトリには**訓練データ生産インフラ**が組み込まれています（デフォルト無効、明示的有効化が必要）。コアアイデア：Agent に実タスクを実行させ、対話プロセス（ツール呼び出しと推論を含む）を標準フォーマットで保存し、Nous Research の次世代ツール呼び出しモデル訓練に提供。日常利用ではこの機能には関与しません。

```
バッチタスクデータセット（JSONL）
    ↓ batch_runner.py（マルチプロセス並列実行）
AIAgent が各タスクを実際に実行
    ↓ save_trajectories=True
対話軌跡フォーマット変換（ShareGPT 形式）
    ↓ trajectory.py
JSONL 訓練データ
    ↓ environments/ + Atropos
RL 強化学習訓練
```

## どのシーンで使うか

| シーン | 説明 |
|------|------|
| **Nous Research 内部** | バッチでツール呼び出し訓練データを生成、Hermes 系列モデルをイテレート |
| **モデルファインチューニング** | 自前のツール呼び出しモデルを訓練したい時、batch_runner で高品質 SFT データを生成 |
| **単発デバッグ** | `--save_trajectories` で 1 回の対話軌跡を保存、Agent 行動分析に便利 |
| **RL 訓練** | environments/ 経由で Atropos フレームワークに接続し強化学習 |
| **日常利用** | デフォルト無効（`save_trajectories=False`）、通常の対話には影響なし |

## 軌跡保存（trajectory.py）

`save_trajectories=True` の時、各対話終了後に自動保存。

**発火位置**：`run_agent.py` の `_save_trajectory()` メソッド（line 2358）、`run_conversation()` 終了時に呼ばれます。

**出力形式**：ShareGPT 形式の JSONL、各レコードに以下が含まれます：

```json
{
  "conversations": [
    {"from": "system", "value": "You are a function calling AI model..."},
    {"from": "human", "value": "ユーザー質問"},
    {"from": "gpt", "value": "<think>\n推論プロセス\n</think>\n<tool_call>\n{...}\n</tool_call>"},
    {"from": "tool", "value": "<tool_response>\n{...}\n</tool_response>"},
    {"from": "gpt", "value": "<think>\n...\n</think>\n最終回答"}
  ],
  "timestamp": "2026-04-14T...",
  "model": "qwen3.6-plus",
  "completed": true
}
```

**出力ファイル**：
- 成功した対話 → `trajectory_samples.jsonl`
- 失敗した対話 → `failed_trajectories.jsonl`

### フォーマット変換の詳細

`_convert_to_trajectory_format()`（`run_agent.py:2193`）が内部 OpenAI 形式を訓練形式に変換：

| 変換ルール | 説明 |
|----------|------|
| `role: assistant` → `from: gpt` | ロールマッピング |
| `role: user` → `from: human` | ロールマッピング |
| `role: tool` → `from: tool` | ツール結果は `<tool_response>` XML で包む |
| reasoning フィールド → `<think>` タグ | ネイティブ思考連鎖を保持 |
| `<REASONING_SCRATCHPAD>` → `<think>` | 非ネイティブ推論も統一形式に |
| 推論内容なし → 空の `<think></think>` | 各 gpt turn の形式一貫性を保証、訓練に便利 |
| tool_calls → `<tool_call>` XML | ツール呼び出しは XML で包む |

## Batch Runner（batch_runner.py）

スケール化データ生成のコアコンポーネント、1287 行。

**使い方**：

```bash
# 基本利用：データセットからバッチ実行
python batch_runner.py --dataset_file=data.jsonl --batch_size=10 --run_name=my_run

# 中断した実行を再開
python batch_runner.py --dataset_file=data.jsonl --batch_size=10 --run_name=my_run --resume

# ツールセット分布を指定
python batch_runner.py --dataset_file=data.jsonl --batch_size=10 --run_name=my_run --distribution=image_gen
```

**主要特性**：

| 特性 | 実装 |
|------|------|
| 並列実行 | `multiprocessing.Pool`（スレッドプールではない）、本物のマルチプロセス |
| ブレークポイント再開 | チェックポイント機構、中断後に `--resume` で復旧可能 |
| ツールセットサンプリング | `toolset_distributions.py` で確率分布に従いランダムにツールセットを選択 |
| 軌跡自動保存 | 各サブタスクで `save_trajectories=True`、`skip_context_files=True` |
| ツール統計 | 全バッチのツール使用統計を集約 |
| HuggingFace 互換 | 出力 JSONL schema を正規化、HF datasets に直接アップロード可能 |

### ツールセット分布（toolset_distributions.py）

データ生成時にどのツール組み合わせを有効化するか、出現確率を制御：

```python
DISTRIBUTIONS = {
    "default": {...},        # 全ツール 100%
    "image_gen": {...},      # 画像生成ツール重点
    "web_research": {...},   # Web 検索ツール重点
    ...
}
```

これにより特定のツール組み合わせの訓練データを狙って生成可能。

### データ生成設定例

`datagen-config-examples/` ディレクトリに既製設定を提供：

```
trajectory_compression.yaml    # 軌跡圧縮設定
web_research.yaml              # Web 研究タスク設定
run_browser_tasks.sh           # ブラウザタスクバッチスクリプト
example_browser_tasks.jsonl    # ブラウザタスクデータセット例
```

## RL 訓練環境（environments/）

Tinker-Atropos フレームワークと統合された強化学習環境：

| 環境 | 用途 |
|------|------|
| `hermes_base_env.py` | 基底 Agent 環境 |
| `agentic_opd_env.py` | Agentic 対話環境 |
| `web_research_env.py` | Web 研究タスク環境 |
| `terminal_test_env/` | ターミナルコマンドテスト環境 |
| `hermes_swe_env/` | ソフトウェアエンジニアリングタスク環境 |
| `tool_call_parsers/` | ツール呼び出しパーサー |
| `agent_loop.py` | Agent ループと環境のブリッジ |

## 一般ユーザーは気にする必要があるか

**通常不要**。このシステムはデフォルト全て無効で、日常のチャットには一切影響ありません。

使いたい場合：

```bash
# 単回対話の軌跡保存（デバッグ用）
python run_agent.py --save_trajectories --query="あなたの質問"

# 訓練データバッチ生成（モデル訓練用）
python batch_runner.py --dataset_file=your_tasks.jsonl --batch_size=10 --run_name=run1
```

## 関連ページ

- [[agent-loop-and-prompt-assembly]] — `save_trajectories` パラメータと `_convert_to_trajectory_format()` メソッド
- [[multi-agent-architecture]] — Batch Runner は大規模バッチ処理エンジンとして
- [[context-compressor-architecture]] — 圧縮後の軌跡データはよりコンパクト

## 関連ファイル

- `agent/trajectory.py` — 軌跡ファイル書き込みと形式変換のユーティリティ関数
- `run_agent.py:2193-2371` — `_convert_to_trajectory_format()` + `_save_trajectory()`
- `batch_runner.py` — バッチランナー（1287 行）
- `toolset_distributions.py` — ツールセット確率分布定義
- `environments/` — RL 訓練環境
- `datagen-config-examples/` — データ生成設定例
