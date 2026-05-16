---
title: Hermes マルチ Agent アーキテクチャ
created: 2026-04-08
updated: 2026-04-18
type: concept
tags: [architecture, module, agent, delegation, concurrency]
sources: [tools/delegate_tool.py, tools/mixture_of_agents_tool.py, run_agent.py]
translation: ja
original: ../../concepts/multi-agent-architecture.md
---

# Hermes マルチ Agent アーキテクチャ

## 概要

Hermes のマルチ Agent 能力は**3 つのランタイム機構**に分かれ、すべて Agent 対話プロセスでトリガーされ、外部スクリプトやオフラインツールを含みません：

| 機構                    | トリガー方式                  | 用途                   |
| --------------------- | --------------------- | -------------------- |
| **Delegate Task**     | LLM tool call（モデル自律判断） | 並列サブタスク、最大 3 路       |
| **Mixture of Agents** | LLM tool call（モデル自律判断） | マルチモデル協調推論           |
| **Background Review** | システムカウンタ自動トリガー         | バックグラウンドで経験抽出 → skill 作成 / 改善 |

## トリガー機構

4 つの機構は 2 つのトリガー方式に分類：

### LLM 自律呼び出し（Delegate Task / MoA）

`web_search`、`read_file` と完全に同じ — モデルがシステムプロンプトでツール記述を見て、ユーザー質問に基づき**自分で判断**して呼び出す、強制トリガーするコードロジックなし。

```text
ユーザー質問 → LLM 推論 → delegate_task / mixture_of_agents 呼び出し決定
                              │
                              ▼
                  run_agent._invoke_tool()
                              │
            ┌─────────────────┼──────────────────┐
            ▼                                    ▼
    delegate_task                          registry.dispatch()
    (特殊分岐、                            → mixture_of_agents
     parent_agent 参照を注入要)
```

LLM が見るツール記述：

| ツール                  | LLM が見る記述（判断根拠）                                                                                             |
| ------------------- | ----------------------------------------------------------------------------------------------------------- |
| `delegate_task`     | *"Spawn subagents to work on tasks in isolated contexts. Only the final summary is returned."*              |
| `mixture_of_agents` | *"Route a hard problem through multiple frontier LLMs collaboratively. Makes 5 API calls — use sparingly."* |

`delegate_task` は `_invoke_tool()` で明示的分岐あり（line 6108）、`parent_agent` 注入が必要のため。`mixture_of_agents` は汎用 registry dispatch を通る。

### システム自動トリガー（Background Review）

**LLM は決定に関与しない**。2 つの独立カウンタがメインループでサイレントに増加、閾値到達後に**ユーザーが応答を受け取った後**で自動トリガー：

```python
# run_agent.py — 2 つの独立カウンタ

# 記憶 review：各 LLM turn +1（line 7008）
self._turns_since_memory += 1
if self._turns_since_memory >= self._memory_nudge_interval:  # デフォルト 10
    _should_review_memory = True
    self._turns_since_memory = 0

# スキル review：各 tool call +1（line 7242）
self._iters_since_skill += 1
if self._iters_since_skill >= self._skill_nudge_interval:    # デフォルト 10
    _should_review_skills = True
    self._iters_since_skill = 0
```

```text
ユーザー質問 → Agent 推論 + ツール呼び出し → 応答ユーザーに配信
                                        │
                                  カウンタチェック（line 9158）
                                        │
                                   閾値超過？──否──→ 何もしない
                                        │
                                       是
                                        │
                                        ▼
                              _spawn_background_review()
                              （デーモンスレッド、非ブロッキング）
                                        │
                                        ▼
                                  サイレント AIAgent fork
                                  max_iterations=8
                                  stdout → /dev/null
                                        │
                                  対話履歴をレビュー
                                  "試行錯誤、戦略変更の経験は？"
                                        │
                              ┌─────────┴─────────┐
                              ▼                   ▼
                      skill_manage()         memory.add()
                      skill 作成 / 改善        永続事実抽出
                              │                   │
                              └─────────┬─────────┘
                                        ▼
                              callback: "💾 Skill updated"
```

**重要な違い**：ユーザーは既に応答を受け取り済み、review はバックグラウンドサイレント動作。GC（ガベージコレクション）に似る — 定期自動実行、ユーザー無感知。

## 1. Delegate Task — サブエージェント委譲

Agent ランタイムのコアマルチ Agent 能力。親 Agent が隔離されたサブ Agent を生成して独立タスクを実行。

### コア定数

```python
# tools/delegate_tool.py
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 再帰的委譲を禁止
    "clarify",         # サブエージェントはユーザーに質問できない
    "memory",          # 共有 MEMORY.md に書き込めない
    "send_message",    # クロスプラットフォーム副作用を生成できない（send_message はメッセージ配信ツール、マルチ Agent 機構ではない）
    "execute_code",    # サブエージェントは段階的推論すべき
])

MAX_DEPTH = 2                # 親(0) → 子(1) → 孫子は拒否(2)
MAX_CONCURRENT_CHILDREN = 3  # 最大 3 並列サブエージェント
DEFAULT_MAX_ITERATIONS = 50  # 各サブエージェントのデフォルトイテレーション上限
DEFAULT_TOOLSETS = ["terminal", "file", "web"]
```

### Orchestrator ロール + 設定可能深度（v2026.4.18+）

`delegate_task` に `role` パラメータ追加、`leaf`（デフォルト）と `orchestrator` をサポート：

```yaml
# config.yaml
delegation:
  max_concurrent_children: 3   # 並行数下限、上限なし
  max_spawn_depth: 1           # 1=フラット（デフォルト）、2-3 でネスト委譲解禁
  orchestrator_enabled: true   # グローバルスイッチ
```

- **leaf**：以前と同じ、サブ agent はもう delegate できない
- **orchestrator**：サブ agent が `delegation` toolset を保持、自分の worker を派生継続可能

**デフォルトフラット姿勢**：`max_spawn_depth=1` の時、orchestrator ロールはサイレントに leaf にダウングレード。ユーザーが能動的に `max_spawn_depth` を 2 または 3 に上げてネスト委譲を解禁。

新規 `DelegateEvent` enum 追加（legacy 文字列の後方互換あり）、gateway/ACP/CLI 進捗消費者向け。

### Agent 跨ぎファイル状態調整（v2026.4.18+）

複数並行サブ agent が同時にファイルを修正する時、親 agent は一貫したファイル状態を確認可能：
- サブ agent が書き込んだファイルは他のサブ agent から可視
- 親 agent は集約時に全サブ agent のファイル操作を確認可能
- 並行 patch の喪失を防止

### 関数シグネチャ

```python
def delegate_task(
    goal: Optional[str] = None,          # 単タスクモード
    context: Optional[str] = None,       # 背景情報
    toolsets: Optional[List[str]] = None,# 利用可能ツールセット
    tasks: Optional[List[Dict]] = None,  # バッチモード（最大 3 個）
    max_iterations: Optional[int] = None,
    acp_command: Optional[str] = None,   # ACP サブプロセスコマンド
    acp_args: Optional[List[str]] = None,
    parent_agent=None,                   # フレームワークが自動注入
) -> str:  # JSON 返却
```

2 モード：
- **単タスク**：`goal` 渡す、直接実行（スレッドプールオーバーヘッドなし）
- **バッチ**：`tasks` 配列渡す、`ThreadPoolExecutor(max_workers=3)` で並列

### 隔離モデル

```text
親 Agent から継承                  サブエージェント固有（完全隔離）
─────────────────                 ─────────────────────
✓ Model / Provider / API Key      ✗ 対話履歴（空白開始）
✓ ワーキングディレクトリ (cwd)       ✗ ターミナル session（独立）
✓ Credential Pool（同 provider）   ✗ 中間ツール呼び出し（親不可視）
✓ プラットフォーム / session_db 参照 ✗ 推論プロセス（親不可視）
✓ Max tokens / reasoning config   ✗ コンテキストファイル（skip_context_files=True）
                                   ✗ 記憶（skip_memory=True）
```

### サブエージェント構築フロー（`_build_child_agent`）

```python
def _build_child_agent(
    task_index: int,              # バッチでのインデックス
    goal: str,                    # 委譲目標
    context: Optional[str],       # 背景
    toolsets: Optional[List[str]],# ツールセット（親と交集、ブラックリスト除外）
    model: Optional[str],         # 親モデルをオーバーライド可能
    max_iterations: int,          # 独立イテレーション上限
    parent_agent,                 # 親 Agent 参照
    override_provider=None, override_base_url=None,
    override_api_key=None, override_api_mode=None,
    override_acp_command=None, override_acp_args=None,
):
```

**ツールセット計算ルール**：サブエージェントは親より多いツールを取得できない。

```text
サブエージェントツール = (ユーザー指定 ∩ 親利用可能) - DELEGATE_BLOCKED_TOOLS
```

### 認証情報プール共有

```python
def _resolve_child_credential_pool(effective_provider, parent_agent):
    # 同 provider → 親 pool 共有（ローテーション同期）
    # 異なる provider → その provider 自前 pool ロード
    # pool なし → 親の固定認証情報継承
```

### 中断伝播

```python
# run_agent.py — 親 Agent の interrupt() メソッド
with self._active_children_lock:
    children_copy = list(self._active_children)
for child in children_copy:
    child.interrupt(message)  # スレッドセーフ伝播
```

サブエージェントは `_build_child_agent` 時に `_active_children` に登録、実行完了後 `finally` で解除。

### 結果構造

親 Agent はこの構造化サマリーのみを見る、**サブエージェントの中間ツール呼び出しと推論は見ない**：

```json
{
  "results": [
    {
      "task_index": 0,
      "status": "completed",
      "summary": "Fixed the login bug by...",
      "api_calls": 12,
      "duration_seconds": 45.3,
      "model": "qwen3.6-plus",
      "exit_reason": "completed",
      "tokens": {"input": 8432, "output": 2341},
      "tool_trace": [
        {"tool": "read_file", "args_bytes": 45, "result_bytes": 1234, "status": "ok"},
        {"tool": "patch", "args_bytes": 234, "result_bytes": 56, "status": "ok"}
      ]
    }
  ],
  "total_duration_seconds": 52.1
}
```

### ACP 異種編成

ACP プロトコルで外部 Agent（Claude Code 等）に委譲：

```python
delegate_task(
    goal="Refactor this module",
    acp_command="claude",
    acp_args=["--acp", "--stdio", "--model", "claude-opus-4-6"]
)
```

Hermes が orchestrator、外部 Agent が executor。

---

## 2. Mixture of Agents — マルチモデル協調推論

サブエージェントではなく、**複数の外部 LLM が同じ質問に協調して回答**。

### アーキテクチャ

```text
                    ユーザー質問
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼            ▼
    Claude Opus    Gemini Pro    GPT-5.4     DeepSeek V3
    (temp=0.6)     (temp=0.6)   (temp=0.6)   (temp=0.6)
          │            │            │            │
          └────────────┼────────────┘
                       ▼
                Claude Opus アグリゲータ
                  (temp=0.4)
                       │
                  ベスト統合回答
```

### 定数

```python
# tools/mixture_of_agents_tool.py
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"

REFERENCE_TEMPERATURE = 0.6     # 多様性
AGGREGATOR_TEMPERATURE = 0.4    # 一貫性
MIN_SUCCESSFUL_REFERENCES = 1   # 最低 1 成功で集約可能
```

### Delegate Task との違い

|        | Delegate Task | Mixture of Agents       |
| ------ | ------------- | ----------------------- |
| **目的** | 異なるタスクの並列実行   | 同じ問題の多角度推論              |
| **隔離** | 完全対話隔離         | 参照応答のみ共有                |
| **モデル** | 同モデルまたはオーバーライド可能 | 4 参照 + 1 集約（5 API 呼び出し） |
| **出力** | 各タスク独立サマリー   | 単一統合回答                   |
| **シーン** | 研究、デバッグ、マルチワークフロー | 複雑な数学、アルゴリズム、高難度推論       |

---

## 3. Background Review — バックグラウンド経験抽出

Agent が対話プロセスで**自動的にサイレント Agent を fork**、対話をレビューして skill 作成 / 改善。

### トリガー条件

```python
# run_agent.py
self._iters_since_skill  # 各 tool call +1
self._skill_nudge_interval = 10  # 10 回ごとに review トリガー
```

### Fork 機構

```python
def _spawn_background_review(self, messages_snapshot, review_memory, review_skills):
    # デーモンスレッド（非ブロッキング）
    fork = AIAgent(
        model=self.model,
        provider=self.provider,
        max_iterations=8,         # 軽量、最大 8 ステップ
        quiet_mode=True,          # stdout → /dev/null
        conversation_history=messages_snapshot,  # 親対話スナップショット
        skip_context_files=True,
    )
    # fork は親の _memory_store と skill ディレクトリを共有
    # skill_manage(action='create/patch') を呼べる
```

### 3 種類の Review Prompt

| Prompt | 関心事 |
|--------|-------|
| `_MEMORY_REVIEW_PROMPT` | 記憶すべき事実を抽出 → MEMORY.md に書き込み |
| `_SKILL_REVIEW_PROMPT` | 再利用可能なフローを抽出 → skill 作成 / 改善 |
| `_COMBINED_REVIEW_PROMPT` | memory + skill review を同時実行 |

### Delegate Task との違い

| | Delegate Task | Background Review |
|---|---|---|
| **トリガー** | Agent が能動呼び出し | 10 イテレーション毎に自動トリガー |
| **ブロック** | 親 Agent が結果待ちブロック | デーモンスレッド、完全非ブロッキング |
| **隔離** | 完全隔離 | **memory store と skill ディレクトリを共有** |
| **結果** | JSON 構造化サマリー | callback 通知："💾 Skill 'xxx' updated" |
| **用途** | ユーザータスクの並列実行 | 経験の自動抽出、skill 改善 |

---

## 4. Agent 間通信機構

Hermes の agent 間通信は**メッセージキューなし、共有メモリなし、IPC なし** — すべて単一プロセス内で Python ネイティブ機構を通じて完了。

### Delegate Task の通信

親子 agent は **ThreadPoolExecutor + Future** で通信、本質的にスレッド間関数呼び出し：

```python
# delegate_tool.py line 619-633
# 親スレッド：タスクをスレッドプールに提出
with ThreadPoolExecutor(max_workers=MAX_CONCURRENT_CHILDREN) as executor:
    for i, t, child in children:
        future = executor.submit(
            _run_single_child,              # サブ agent 実行関数
            task_index=i, goal=t["goal"],
            child=child, parent_agent=parent_agent,
        )
        futures[future] = i

    # 親スレッド：ブロッキング待機、先に完了した方から受け取り
    for future in as_completed(futures):
        entry = future.result()             # ← これが「通信」
        results.append(entry)
```

```python
# _run_single_child 内部（line 373）
result = child.run_conversation(user_message=goal)  # サブ agent 実行完了
summary = result.get("final_response") or ""         # 返り値を直接取得
```

**単タスク時はもっとシンプル**、スレッドプールすら不要（line 612）：
```python
result = _run_single_child(0, _t["goal"], child, parent_agent)
```

### Mixture of Agents の通信

MoA はスレッドすらない、**同一スレッド内の非同期 HTTP リクエスト + メモリ集約**：

```python
# mixture_of_agents_tool.py line 311
# 4 つの HTTP リクエストを並行発行（asyncio コルーチン、マルチスレッドではない）
model_results = await asyncio.gather(*[
    _run_reference_model_safe(model, user_prompt, REFERENCE_TEMPERATURE)
    for model in ref_models
])

# 結果を list に収集（純メモリ変数）
successful_responses = []
for model_name, content, success in model_results:
    if success:
        successful_responses.append(content)

# prompt を作って 5 つ目のリクエスト発行
aggregator_system_prompt = _construct_aggregator_prompt(
    AGGREGATOR_SYSTEM_PROMPT, successful_responses
)
```

4 モデルの中間応答は関数スタックの list に保存、集約完了後にガベージコレクション、ディスクに落とさない。

### サブ agent 間

**完全に通信なし。** 複数サブ agent は各自のスレッドで並列実行、互いの存在を知らない、調整機構なし。

### 中断時の結果処理

ユーザーがサブ agent 実行中に新メッセージを送る時：

```python
# run_agent.py line 2527-2538
def interrupt(self, message):
    self._interrupt_requested = True
    # 実行中の全サブ agent に伝播
    with self._active_children_lock:
        children_copy = list(self._active_children)
    for child in children_copy:
        child.interrupt(message)    # 1 つずつ中断
```

中断後：
- サブ agent は `status: "interrupted"` を返す、既に出した部分結果は返り値に保持
- 親 agent の `_persist_session` がトリガー、中断結果を含むメッセージを SQLite に書き込む
- **しかし中断結果は有効回答としてユーザーに表示されない** — 親 agent は `completed: False` を標記、新メッセージ処理の次ラウンドへ
- サブ agent 自身は `persist_session=False`、独立して DB 書き込みしない — 部分結果は「親 agent 対話のツール返却メッセージ」として DB に存在

### 通信モデルまとめ

| 機構 | 通信方式 | 並行モデル | 中間結果保管 |
|------|---------|---------|------------|
| Delegate Task | `Future.result()`（スレッド間返り値） | ThreadPoolExecutor マルチスレッド | ディスクに落とさず、関数返り値 |
| Mixture of Agents | `asyncio.gather`（非同期コルーチン収集） | 単一スレッド非同期 | ディスクに落とさず、メモリ list |
| Background Review | デーモンスレッド fire-and-forget | 独立デーモンスレッド | skill/memory ファイルに直接書き込み |

**一言：すべて単一プロセス内で完了、プロセス間通信は一切なし。**

---

## 5. イテレーション予算システム

すべてのマルチ Agent 機構が共有するリソース管理層。

### IterationBudget クラス

```python
class IterationBudget:
    """スレッドセーフなイテレーションカウンタ（run_agent.py:167-209）"""
    
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()
    
    def consume(self) -> bool:
        """アトミックチェック + 減算。各 LLM turn ごとに 1 回呼ぶ。"""
    
    def refund(self) -> None:
        """1 回返却（execute_code 呼び出し後に返却、複数検証を奨励）。"""
    
    @property
    def remaining(self) -> int: ...
```

### 予算隔離

```text
親 Agent:        IterationBudget(90)   ← デフォルト 90
サブエージェント A: IterationBudget(50)   ← 独立、親予算を消費しない
サブエージェント B: IterationBudget(50)   ← 独立
サブエージェント C: IterationBudget(50)   ← 独立
──────────────────────────────
理論総イテレーション: 90 + 150 = 240
```

### 予算圧迫警告

```python
self._budget_caution_threshold = 0.7   # 70% — "まとめ始め"
self._budget_warning_threshold = 0.9   # 90% — "即応答"
```

警告はツール結果 JSON に注入（メッセージ構造を破壊せず、prompt cache を無効化しない）。

---

## 6. 設定

```yaml
# config.yaml
delegation:
  provider: openrouter            # 任意：サブエージェント専用 provider
  model: google/gemini-3-flash    # 任意：サブエージェント専用安価モデル
  max_iterations: 50              # 各サブエージェント最大イテレーション数
  reasoning_effort: low           # 任意：サブエージェント推論深度制御（low/medium/high/xhigh）
  # またはエンドポイント直接指定
  base_url: https://api.openai.com/v1
  api_key: sk-xxx
```

### 使用例

```python
# 単タスク
delegate_task(
    goal="Debug the login failure issue",
    context="User reports 500 error on /api/login",
    toolsets=["terminal", "file"]
)

# 並列 3 タスク
delegate_task(tasks=[
    {"goal": "Fix login bug", "toolsets": ["terminal", "file"]},
    {"goal": "Update API docs", "toolsets": ["terminal", "file"]},
    {"goal": "Run test suite", "toolsets": ["terminal"]},
])

# マルチモデル協調推論
mixture_of_agents(user_prompt="P ≠ NP の証明の既知の最強結果は何ですか？")
```

## マルチ Agent の 2 つのレベル

Hermes は実際 2 種類のマルチ Agent 方式を持ち、異なるシーンに対応：

|                  | セッション内 multi-agent（本ページ） | マルチ Profile             |
| ---------------- | --------------------- | ------------------------- |
| 粒度               | 1 セッション内のサブタスク         | 完全に独立した agent インスタンス     |
| コンテキスト           | サブ agent が親 agent の対話を継承 | 完全隔離、互いに不可視             |
| terminal backend | 親 agent 継承、**切替不可**    | 各 Profile **独立設定**        |
| 記憶               | 共有（同じ MemoryManager） | 各自独立の MEMORY.md / USER.md |
| モデル              | 異なる可能                | 異なる可能                    |
| 連携方式             | 自動ディスパッチ + 結果返却        | 手動切替、自動連携なし              |

**セッション内 multi-agent は「1 つの脳が複数の手を指揮」** — 1 タスク内の並列分業に適合。

**マルチ Profile は「複数の独立した人が各自管理」** — 職能別に異なるセキュリティ境界、モデル、スキルセットを隔離するのに適合。例：`coder` Profile は `local` backend で日常開発、`ops` Profile は `docker` backend で危険操作。

### マルチ Profile 間で通信可能か？

**ネイティブな通信チャネルなし。** 各 Profile は独立プロセス、独立 DB、独立記憶、互いの存在を知らない。

しかし**メッセージプラットフォーム経由で間接的に対話可能** — 2 つの Profile がそれぞれ bot を 1 つずつ持ち、同じチャンネルで：

```text
Profile A (Bot A) → send_message ツールでチャンネルにメッセージ送信
                              ↓
                      Discord / Slack チャンネル（メッセージ中継）
                              ↓
Profile B (Bot B) → ALLOW_BOTS=all → 受信、通常ユーザーメッセージとして処理
```

Discord と Slack は両方 `allow_bots` 設定対応（3 モード：none/mentions/all）：

```bash
# Discord — Bot B の .env
DISCORD_ALLOW_BOTS=none       # デフォルト：全 bot メッセージ無視
DISCORD_ALLOW_BOTS=mentions   # 自分に @ された bot メッセージのみ受信
DISCORD_ALLOW_BOTS=all        # 全 bot メッセージ受信

# Slack — Bot B の .env
SLACK_ALLOW_BOTS=none         # 同上
SLACK_ALLOW_BOTS=mentions
SLACK_ALLOW_BOTS=all
```

Discord はさらに**マルチ bot フィルタ**あり：メッセージが他の bot に @ しているが自分には @ していない時に自動スキップ、マルチ bot チャンネルでの相互干渉を回避。

**注意**：これは Hermes が設計した agent 間通信機能ではなく、2 つの独立 bot がプラットフォームメッセージ経由でたまたま対話するだけ。遅延あり、トランザクション保証なし、デッドループ（A 送信 → B 返信 → A また返信 → 無限ループ）が起きやすい、使用時には制御注意必要。

詳細 → [[configuration-and-profiles]]

## 関連ページ

- [[configuration-and-profiles]] — マルチ Profile アーキテクチャ（別のマルチ Agent 方式）
- [[tool-registry-architecture]] — サブエージェントは registry で制限ツールセット取得
- [[auxiliary-client-architecture]] — サブエージェントは独立補助モデル設定可能
- [[credential-pool-and-isolation]] — 認証情報プール共有とローテーション
- [[skills-system-architecture]] — Background Review が自動作成 / 改善した skill はここに保管
- [[trajectory-and-data-generation]] — Batch Runner（Nous 内部訓練ツール、Agent ランタイムには属さない）

## 関連ファイル

- `tools/delegate_tool.py` — サブエージェント委譲実装
- `tools/mixture_of_agents_tool.py` — マルチモデル協調推論
- `tools/send_message_tool.py` — クロスプラットフォームメッセージ配信（マルチ Agent には属さず、messaging-gateway に分類）
- `run_agent.py` — IterationBudget クラス、Background Review、中断伝播
