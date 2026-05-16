---
title: Context Compressor コンテキスト圧縮アーキテクチャ
created: 2026-04-08
updated: 2026-04-17
type: concept
tags: [architecture, module, component, agent, context-compression]
sources: [agent/context_engine.py, agent/context_compressor.py, run_agent.py, hermes_state.py, plugins/context_engine/__init__.py]
translation: ja
original: ../../concepts/context-compressor-architecture.md
---

# Context Compressor — コンテキスト圧縮アーキテクチャ

## 概要

Context Compressor は `agent/context_compressor.py` に実装、**自動コンテキストウィンドウ圧縮**クラス。対話がモデルのコンテキスト制限に近づいた時、補助 LLM（安価 / 高速モデル）で中間ターンを構造化要約、同時に頭部と尾部のコンテキストを保護。

### Context Engine プラグイン化（2026-04-10）

以前はコンテキスト管理が 1 方式のみ — `ContextCompressor`（要約圧縮）、戦略を変えるにはソースコード変更が必要でした。現在は `ContextEngine` ABC を抽出、`ContextCompressor` はその 1 実装、第三者がプラグインを書いて置き換え可能、Hermes ソースコード変更不要。

**本質：「コンテキストがほぼ満杯になったらどうするか」という決定をハードコードからプラガブルに変えた。**

```yaml
# config.yaml — 1 行で切替
context:
  engine: "compressor"   # デフォルト要約圧縮；プラグイン名指定で切替（例 "lcm"）
```

**代替エンジン例（仮想）：**

| エンジン | 戦略 | 適用シーン |
|------|------|---------|
| compressor（内蔵） | LLM 要約圧縮 | 汎用、デフォルト |
| lcm（仮想） | 旧対話をベクトル DB に保存、必要時セマンティック検索 | 超長セッション、精密リコール必要 |
| sliding-window（仮想） | シンプルスライディングウィンドウ切り詰め、要約しない | 低コスト、auxiliary モデル不要 |

**ContextEngine ABC は 3 つのコアメソッド実装が必要**：
- `name` — エンジン識別子（property）
- `should_compress(prompt_tokens)` — 圧縮必要か
- `compress(messages, current_tokens)` — 圧縮実行、新メッセージリスト返却

**オプションメソッド**：`on_session_start/end`、`get_tool_schemas`（エンジンが agent にツールを公開可能、例 `lcm_grep`）、`handle_tool_call`、`update_model`。

**プラグインディレクトリ**：`plugins/context_engine/<name>/`、`plugin.yaml` + `__init__.py`（`register(ctx)` 実装または `ContextEngine` サブクラス公開）を含む。

**1 つのエンジンのみアクティブ許可**、MemoryProvider の「外部最大 1 つ」制約と同じ。

コア理念：**長対話でコンテキストを破棄する必要なし — 構造化要約で旧ターンを置換、重要情報を保持。**

## アーキテクチャ原理

### 圧縮アルゴリズム

```text
アルゴリズムフロー（v3）:
  Phase 1: 安価前処理（純ローカル、LLM 呼ばない、token コストゼロ）
    ├── Pass 1: MD5 重複排除 — 同じファイル 5 回読みは最新 1 つだけ残す
    ├── Pass 2: Smart Collapse — 旧ツール出力を情報化された 1 行要約に置換
    └── Pass 3: tool_call パラメータ切り詰め — >500 文字を 200 に切る
  Phase 2: 境界決定
    頭部保護（システムプロンプト + 初ターン）+ token 予算による尾部保護
  Phase 3: LLM 構造化要約（Phase 1 でスリム化した中間部分のみ処理）
  Phase 4: 組み立て + 孤立 tool_call / tool_result ペアのクリーンアップ
```

#### 旧版（v2）vs 新版（v3）実行方式比較

**旧版は 1 ステップのみ**：token が閾値到達 → 中間対話をそのまま LLM に渡して要約 → 置換。問題はツール出力がしばしば数 KB（`npm test` 200 行、`read_file` ファイル全体）、全部 LLM に要約させると**要約自体が token 高コスト**；同じファイルを 5 回読んで 5 つの完全コンテンツが全て；圧縮効果が悪いと繰り返しトリガー、毎回 LLM 呼んで空回り。

**新版 3 段階**：Phase 1 はコストゼロのローカル操作（文字列ハッシュ、正規表現置換、切り詰め）、しばしば 30-50% の token を削減可能。Phase 3 の LLM 呼び出しの処理データ量はそのため大幅に少ない。アンチスラッシング機構（連続 2 回非効率なら停止）と合わせて、全体的に LLM 呼び出し回数と各呼び出しの入力量が大幅減少。

### 進化履歴

| 改善 | v1 | v2 | v3（2026-04-14+） |
|---|---|---|---|
| 要約テンプレート | 構造なし | Goal/Progress/Decisions/Files/Next Steps | **番号付き Completed Actions + Active State**（action-log スタイル） |
| 要約更新 | 毎回ゼロから生成 | 反復更新 | 反復更新（番号継続） |
| 尾部保護 | 固定メッセージ数 | Token 予算（比例スケール） | v2 と同 |
| ツール出力修剪 | なし | 汎用プレースホルダ `_PRUNED_TOOL_PLACEHOLDER` | **Smart Collapse**：ツールタイプ別に情報化 1 行要約生成 |
| 重複排除 | なし | なし | **MD5 重複排除**：同じ tool result は最新のみ保持 |
| tool_call パラメータ | そのまま保持 | そのまま保持 | **>500 文字で自動 200 文字に切り詰め** |
| 要約予算 | 固定 | 圧縮内容に比例スケール | v2 と同、ただし `max_tokens` が 2× から **1.3×** に（膨張防止） |
| アンチスラッシング | なし | なし | **連続 2 回圧縮 <10% でスキップ**、揺れループ回避 |
| マルチモーダルメッセージ | クラッシュ可能性 | クラッシュ可能性 | dedup/prune パスで list content スキップ |
| 圧縮 note 冪等性 | 初回圧縮のみ追加 | v1 と同 | **既存検出**で重複追加しない |
| Failure cooldown | 10 分固定 | 10 分固定 | **provider なし 10 分、瞬間エラー 60 秒** |
| ツール呼び出し完全性 | 失われる可能性 | _sanitize_tool_pairs が孤児ペア修復 | v2 と同 |

## コアコンポーネント

### 1. Token 予算管理

```python
class ContextCompressor:
    def __init__(self, model, threshold_percent=0.50):
        self.context_length = get_model_context_length(model)
        self.threshold_tokens = int(self.context_length * 0.50)  # 50% でトリガー
        self.tail_token_budget = int(self.threshold_tokens * 0.20)  # 尾部予算
        self.max_summary_tokens = min(int(self.context_length * 0.05), 12_000)  # 要約上限
```

**スケール設計**：尾部予算と要約上限はどちらもモデルコンテキストウィンドウに比例、大ウィンドウモデルはより豊富な要約を得る。

### 2. ツール出力修剪（3 段階前処理）

`_prune_old_tool_results()` は現在 3 つのことを行う、すべて LLM 呼ばない：

**Pass 1 — MD5 重複排除**：同じ tool result（>200 chars、非マルチモーダル）を MD5 hash で重複排除、最新 1 つのみ保持、旧コピーを以下に置換：
```
[Duplicate tool output — same content as a more recent call]
```
典型シーン：同じファイルを繰り返し read、または同じパターンを繰り返し search。

**Pass 2 — Smart Collapse**（2026-04-14）：`tool_call_id` でツール名 + パラメータを調べ、**情報化された 1 行要約**を生成して元の汎用プレースホルダを置換。異なるツールは異なるテンプレート：

```text
[terminal] ran `npm test` -> exit 0, 47 lines output
[read_file] read config.py from line 1 (1,200 chars)
[search_files] content search for 'compress' in agent/ -> 12 matches
[patch] replace in config.py (1,500 chars result)
[web_search] query='cache control' (5,200 chars result)
[delegate_task] 'refactor auth module' (8,400 chars result)
[memory] save on long-term
```

旧 `_PRUNED_TOOL_PLACEHOLDER` と比べて、要約は**具体的なコマンド / ファイルパス / 結果規模**を保持、モデルが履歴を見ても「以前何をしたか」が分かる。内蔵テンプレートは terminal / read_file / write_file / search_files / patch / browser_* / web_search / web_extract / delegate_task / execute_code / skill_* / vision_analyze / memory / todo / clarify / text_to_speech / cronjob / process をカバー、他のツールは汎用フォールバックを通る。

**Pass 3 — tool_call パラメータ切り詰め**：assistant メッセージで `tool_calls.function.arguments` 長 > 500 なら、先頭 200 文字 + `...[truncated]` に切り詰め。修正シーン：`write_file(content=50KB)` のような呼び出しでは、ツール結果が修剪されてもパラメータ自体がコンテキストを占める。

**マルチモーダル保護**：全 3 Pass が `isinstance(content, list)` を検出してマルチモーダルメッセージをスキップ、画像 / 音声内容の破壊を回避。

### 3. 要約予算計算

```python
def _compute_summary_budget(turns_to_summarize):
    content_tokens = estimate_messages_tokens_rough(turns_to_summarize)
    budget = int(content_tokens * 0.20)  # 20% に圧縮
    return max(2000, min(budget, self.max_summary_tokens))
```

**設計**：要約予算は圧縮対象内容に比例、ただし上下限制御。

### 4. 要約用テキスト直列化

```python
def _serialize_for_summary(turns):
    """
    対話ターンをタグ付きテキストに直列化:
    [TOOL RESULT xxx]: 内容（3000 chars に切り詰め：先 2000 + ... + 後 800）
    [ASSISTANT]: 内容 + [Tool calls: tool_name(args), ...]
    [USER]: 内容（3000 chars に切り詰め）
    """
```

**重要**：ツール呼び出し名とパラメータを含む、要約器が具体的なファイルパス、コマンド、出力を保持できるように。

### 5. 構造化要約生成

#### 初回圧縮（v3 action-log テンプレート、2026-04-14）

```text
## Goal
[ユーザーが達成したいこと]

## Constraints & Preferences
[ユーザーの嗜好、コーディングスタイル、制約、重要決定]

## Completed Actions
[番号付きアクションリスト、各エントリ形式: N. ACTION target — outcome [tool: name]
例:
1. READ config.py:45 — found `==` should be `!=` [tool: read_file]
2. PATCH config.py:45 — changed `==` to `!=` [tool: patch]
3. TEST `pytest tests/` — 3/50 failed: test_parse, test_validate, test_edge [tool: terminal]
具体的に必須：ファイルパス、コマンド、行番号、結果すべて保持]

## Active State
[現在の作業状態:
- ワーキングディレクトリとブランチ
- 修正 / 作成したファイルと簡単な説明
- テスト状態（X/Y passing）
- 実行中プロセスまたはサーバー
- 主要環境情報]

## In Progress
[圧縮トリガー時に進行中の作業]

## Blocked
[未解決のブロッカー / エラー、完全エラーメッセージ含む]

## Key Decisions
[重要な技術決定とその WHY]

## Resolved Questions
[ユーザーが尋ねて既に回答した質問 — 回答含む、次の agent が同じ回答を繰り返さないように]

## Pending User Asks
[ユーザーが尋ねたが未回答の質問]

## Remaining Work
[未完了タスク]

## Relevant Files
[読み込み / 修正 / 作成したファイル]

## Critical Context
[具体的な値、エラーメッセージ、設定詳細など失えない情報]
```

#### v2 要約テンプレート（旧版、参考比較）

```text
## Goal
[What the user is trying to accomplish]

## Constraints & Preferences
[User preferences, coding style, constraints, important decisions]

## Progress
### Done
[Completed work — include specific file paths, commands run, results obtained]
### In Progress
[Work currently underway]
### Blocked
[Any blockers or issues encountered]

## Key Decisions
[Important technical decisions and why they were made]

## Resolved Questions
[Questions the user asked that were ALREADY answered — include the answer]

## Pending User Asks
[Questions or requests from the user that have NOT yet been answered]

## Relevant Files
[Files read, modified, or created — with brief note on each]

## Remaining Work
[What remains to be done — framed as context, not instructions]

## Critical Context
[Any specific values, error messages, configuration details]

## Tools & Patterns
[Which tools were used, how they were used effectively, and any tool-specific discoveries]
```

#### v2 → v3 プロンプト項目別差分

| セクション | v2 | v3 | 変更理由 |
|------|----|----|----------|
| 完了記録 | `## Progress > ### Done` 自由テキスト | `## Completed Actions` 強制番号付き + 固定形式 `N. ACTION target — outcome [tool: name]` | 自由テキストは曖昧な記述になりやすい（"modified some files"）、番号形式は LLM に具体的パス、コマンド、行番号を強制 |
| 形式例 | なし | 3 つの例を提示（READ/PATCH/TEST） | Few-shot で LLM に形式遵守を誘導 |
| 現状 | 独立セクションなし、情報が Progress に散在 | 新規 `## Active State`（ワーキングディレクトリ、ブランチ、修正ファイル、テスト状態、実行プロセス） | 続行 agent が最も必要なのは「今どこにいるか、状態はどうか」、旧版には情報を載せる明確な場所なし |
| ツールパターン | `## Tools & Patterns` 独立セクション | **削除**、ツール情報を Completed Actions の `[tool: name]` に融合 | ツールと操作自体が結びついている、別セクション化は冗長で token 浪費 |
| 具体性要求 | "Be specific — include file paths, command outputs, error messages, and concrete values" | "Be CONCRETE — include file paths, command outputs, error messages, **line numbers**, and specific values. **Avoid vague descriptions like 'made some changes' — say exactly what changed.**" | 曖昧記述を明示禁止、行番号要求を新規追加 |
| 反復更新 | "ADD new progress. Move from 'In Progress' to 'Done'" | "ADD new completed actions to numbered list **(continue numbering)**. Update 'Active State' to reflect current state. **Remove information only if it is clearly obsolete.**" | "continue numbering" で圧縮ごとに番号リセットして情報喪失を防止；"only if clearly obsolete" で過度削除を防止 |
| 要約予算 | `max_tokens = budget × 2` | `max_tokens = budget × 1.3` | 2× は緩すぎて要約膨張、1.3× がよりコンパクト |

**コア設計思想**：v2 のテンプレートは LLM に自由度を与えすぎ、出力品質が不安定；v3 は強制番号、具体例、明示禁止で「要約の書き方」を開放型から穴埋め型に変え、圧縮出力がより予測可能、情報密度が高い。

#### Preamble（ロール設定）— 両版一致

```text
You are a summarization agent creating a context checkpoint.
Your output will be injected as reference material for a DIFFERENT
assistant that continues the conversation.
Do NOT respond to any questions or requests in the conversation —
only output the structured summary.
Do NOT include any preamble, greeting, or prefix.
```

インスピレーション：OpenCode の "do not respond to any questions" + Codex の "another language model" フレームワーク。両版で変更なし。

#### 反復更新

旧要約がある時、Prompt は以下に変更：

```text
PREVIOUS SUMMARY: [旧要約]
NEW TURNS TO INCORPORATE: [新ターン]

要約を更新、まだ有用な旧情報をすべて保持。
新しい completed actions を番号リストに追加（番号継続）。
"In Progress" を "Completed Actions" に移動（完了時）。
回答済み質問を "Resolved Questions" に移動。
"Active State" を更新して現状反映。
明らかに古くなった場合のみ情報を削除。
```

### 6. 適応的失敗 cooldown 機構

```python
_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600   # 10 分、provider なし用
_TRANSIENT_COOLDOWN_SECONDS      = 60    # 1 分、瞬間エラー用

def _generate_summary(self, turns):
    if time.monotonic() < self._summary_failure_cooldown_until:
        return None  # cooldown 期間内スキップ

    try:
        response = call_llm(task="compression", ...)
        self._summary_failure_cooldown_until = 0.0  # 成功でリセット
    except RuntimeError:
        # provider 設定なし — 10 分以内自然復旧しない
        self._summary_failure_cooldown_until = time.monotonic() + 600
    except Exception:
        # 瞬間エラー（タイムアウト / レート制限 / ネットワーク） — 短 cooldown で高速リトライ
        self._summary_failure_cooldown_until = time.monotonic() + 60
```

**設計考慮**（2026-04-14 改善）：2 種類の失敗を区別、`RuntimeError` は設定問題で 10 分の長 cooldown、他の例外はデフォルトで瞬間問題として 60 秒の短 cooldown、圧縮が短期障害から速く復旧できるように。

### 6b. アンチスラッシング保護（Anti-Thrashing、2026-04-14）

```python
def should_compress(self, prompt_tokens=None) -> bool:
    if tokens < self.threshold_tokens:
        return False
    # 連続 2 回圧縮が 10% 未満節約 → 今回スキップ
    if self._ineffective_compression_count >= 2:
        logger.warning(
            "Compression skipped — last %d compressions saved <10%% each. "
            "Consider /new to start a fresh session, or /compress <topic> ..."
        )
        return False
    return True
```

各圧縮後、`saved_estimate / display_tokens` で実節約パーセントを計算：
- `>= 10%` → `_ineffective_compression_count = 0` リセット
- `< 10%`  → `_ineffective_compression_count += 1`

**解決する問題**：特定シーン（尾部 + 頭部 + 要約自体が大きい）で圧縮が 1-2 メッセージしか絞り出せず、毎ターントリガーされるがほぼ無用、圧縮スラッシングループ形成。連続 2 回無効なら諦め、ユーザーに `/new` または `/compress <topic>` で手動処理を提示。

### 7. ツール呼び出しペア完全性保証

```python
def _sanitize_tool_pairs(messages):
    """
    圧縮後の孤児 tool_call / tool_result ペアを修復:
    
    故障モード 1: ツール結果が参照する call_id 対応の assistant tool_call が削除済み
    → API エラー "No tool call found for function call output..."
    → 解決: 孤児結果を削除
    
    故障モード 2: assistant に tool_calls あるが対応する結果が破棄済み
    → API エラー "every tool_call must be followed by a tool result..."
    → 解決: スタブ結果挿入 "[Result from earlier conversation]"
    """
```

**重要性**：修復しないと API が全メッセージリストを拒否、圧縮失敗。

### 8. 境界アラインメント

```python
def _align_boundary_forward(messages, idx):
    """境界が tool result に落ちる場合、非ツールメッセージまで前に進める"""

def _align_boundary_backward(messages, idx):
    """境界が tool call/result グループ中間に落ちる場合、後ろに引いてグループ完全包含"""
```

**データロス防止**：assistant + tool_results グループの分割回避、そうしないと `_sanitize_tool_pairs` が末尾孤児結果を削除してサイレントデータロス。

**v0.10.0 修正**：新規 `_ensure_last_user_message_in_tail()` メソッド、`_find_tail_cut_by_tokens` 末尾で呼び出し、**最後のユーザーメッセージが永遠に尾部に残る**ことを保証。以前は特定シーンで圧縮がユーザーのアクティブタスク指示を要約領域に圧縮、agent が現タスクコンテキストを失い、停滞または完了済み作業を繰り返す事態が発生（#10896）。

### 9. 尾部 token 予算保護

```python
def _find_tail_cut_by_tokens(messages, head_end, token_budget):
    # ハード底線：最低 3 つの尾部メッセージ保護
    min_tail = min(3, n - head_end - 1)
    
    # ソフト上限：予算 1.5 倍超過許可、超大メッセージ中間切断回避
    soft_ceiling = int(token_budget * 1.5)
    
    # 末尾から前へ累積、soft_ceiling 超過かつ min_tail 満たすまで
    # 予算が min_tail カバーに不足 → n - min_tail にフォールバック（3 つ強制保護）
    # 予算が全カバー → head 後で強制切断、圧縮実行保証
```

主要変更（2026-04-09）：固定メッセージ数保護から **token 予算 + ハード底線 min_tail=3** に、長短メッセージ両方により合理的。

### 10. 要約ロール選択

```python
# 要約メッセージ挿入時、適切なロール選択で連続同ロール回避
if last_head_role in ("assistant", "tool"):
    summary_role = "user"
else:
    summary_role = "assistant"

# 選択したロールが尾部と衝突なら反転試行
# 両ロール衝突 → 最初の尾部メッセージにマージ
```

## コンテキスト管理全景

### 無限ターン対話

Hermes は**対話ターン数を制限しない**。`max_history` なし、固定ターン数切り詰めなし。全対話履歴はメモリに保持、圧縮ループで維持：

```text
対話開始 → メッセージ累積 → コンテキストウィンドウ 50% 到達 → 自動圧縮
                                              │
                                        修剪 + 要約 + 再構成
                                              │
                                        累積継続 → 再度 50% 到達 → 再圧縮 → ...
```

理論的に無限対話可能。各圧縮で反復更新された要約生成、ゼロから再要約ではない。

### Session 分裂

圧縮時に **session 分裂**、目的は完全な元メッセージを `session_search` で後日検索可能にするため。

```text
圧縮前:
  session "abc"（DB に msg 0-49 完全な元メッセージ保管済み）
  メモリ内 msg 2-40 が要約に圧縮される予定

圧縮後:
  session "abc"（終了、reason="compression"）
    → DB の msg 0-49 完全保持 ← session_search で元内容検索可能

  session "abc-2"（新規、parent_session_id="abc"）
    → 要約 + 尾部メッセージ + 後続新メッセージ
    → _last_flushed_db_idx を 0 にリセット

複数圧縮でチェーン形成:
  abc → abc-2 → abc-3 → ...
  各セグメント完全、parent_session_id チェーンで血縁保持
```

**なぜ in-place 置換しない？** 圧縮後のメッセージを同じ session に上書きすると、DB 前半は元メッセージ、後半は要約、session_search が一貫しないデータを検索することに。分裂で各 session セグメント内容の完全一貫性を保証。

### メッセージ永続化機構

メッセージは**リアルタイム逐一 DB 書き込みではない**、退出点でバッチ flush：

```python
def _flush_messages_to_session_db(self, messages, conversation_history):
    # 増分書き込み：前回の水位線から、新規メッセージのみ書く
    flush_from = max(start_idx, self._last_flushed_db_idx)
    for msg in messages[flush_from:]:
        db.append_message(session_id, role, content, ...)
    self._last_flushed_db_idx = len(messages)  # 水位線更新
```

**トリガータイミング**（コード内 20 呼び出し点、全退出パスカバー）：

| シーン | 保証 |
|------|------|
| 対話正常完了 | ✅ 書き込み |
| API エラーで max retry 使い切り | ✅ 放棄前書き込み |
| ユーザー中断（Ctrl+C） | ✅ 中断前書き込み |
| Rate limit 待機中の中断 | ✅ 書き込み |
| 413/context overflow 圧縮失敗 | ✅ 書き込み |
| ツール実行例外 | ✅ 書き込み |
| Fallback provider 全失敗 | ✅ 書き込み |

**水位線で重複防止**：`_last_flushed_db_idx` が書き込み済位置を記録。複数の退出パスが `_persist_session()` を重複呼び出ししても、同じメッセージが 2 回書かれない（issue #860 修正）。

```text
1 回目 flush:  messages[0:15] → DB、 水位線 = 15
2 回目 flush:  messages[15:23] → DB、水位線 = 23
3 回目 flush:  messages[23:23] → スキップ（新メッセージなし）
```

## 設計の優位性

### 旧メッセージ破棄との比較

| 観点 | 旧メッセージ破棄 | Context Compressor |
|---|---|---|
| 情報保持 | 完全喪失 | 構造化要約で重要情報保持 |
| 連続性 | Agent が完了作業を忘れる | 進捗と決定を知る |
| ファイル追跡 | 喪失 | 関連ファイルリスト |
| 反復更新 | 該当なし | 要約反復更新可能 |
| ユーザー体験 | Agent が作業繰り返す | Agent が要約から継続 |

### コスト効果

圧縮は**補助 LLM**（安価モデル、例 Gemini 3 Flash）を使用、主対話モデルではない。典型シーン：
- 補助モデルコスト：$0.01-0.05 / 圧縮回
- 回避された重複作業コスト：圧縮コストを遥かに超える
- コンテキスト節約：30-70%

## 設定と操作

### 設定パラメータ

```yaml
# config.yaml
compression:
  summary_provider: auto      # または openrouter, nous, custom
  summary_model: ""           # 空 = 自動選択
  threshold_percent: 0.50     # 50% コンテキスト使用でトリガー
```

### 環境変数

```bash
# 圧縮タスクに特定モデル設定
export AUXILIARY_COMPRESSION_MODEL=claude-haiku-4-5
export CONTEXT_COMPRESSION_PROVIDER=openrouter
```

### ランタイム状態

```python
compressor.get_status()
# 返却: {
#   "last_prompt_tokens": 45000,
#   "threshold_tokens": 65536,
#   "context_length": 131072,
#   "usage_percent": 34,
#   "compression_count": 2
# }
```

## OpenClaw（Claude Code）圧縮機構との比較

OpenClaw の圧縮実装は `src/agents/compaction.ts` に位置、**チャンク要約**戦略を採用、Hermes の**3 段階前処理 + 単発要約**と鮮明な対比。

### 全体アーキテクチャ差異

| 観点 | Hermes v3 | OpenClaw |
|------|-----------|----------|
| 全体戦略 | ローカル前処理 → 境界分割 → 単発 LLM 要約 | チャンク + 複数 LLM 要約（2 パス、下記） |
| LLM 呼び出し回数 | **1 回**（スリム化後の中間部分のみ） | **複数回**（ローリング N 回、または並列 N+1 回） |
| 前処理 | MD5 重複排除 + Smart Collapse + パラメータ切り詰め（token ゼロ） | `stripToolResultDetails()` でツール詳細削除（軽量） |
| チャンク | チャンクなし、頭 - 中 - 尾 3 セグメント | 2 種類のチャンク戦略（下記） |

OpenClaw は実際 2 つの圧縮パス（`src/agents/compaction.ts`）：

- **`summarizeChunks`（ローリング式）**：token 上限でチャンク化、直列処理 — chunk1 要約を chunk2 の `previousSummary` として渡す、徐々にローリング。LLM 呼び出し N 回。
- **`summarizeInStages`（並列+マージ）**：`splitMessagesByTokenShare()` で N チャンクに分割（デフォルト `DEFAULT_PARTS=2`）、各チャンク独立要約、最後に `MERGE_SUMMARIES_INSTRUCTIONS` でマージ。LLM 呼び出し N+1 回。

### 要約テンプレート比較

**Hermes v3（11 セクション）：**

```
Goal / Constraints & Preferences / Completed Actions（番号+形式） /
Active State / In Progress / Blocked / Key Decisions /
Resolved Questions / Pending User Asks / Relevant Files /
Remaining Work / Critical Context
```

**OpenClaw（5 セクション）：**

```
Decisions / Open TODOs / Constraints/Rules /
Pending user asks / Exact identifiers
```

Hermes テンプレートはより詳細（Active State、Relevant Files、番号付き Completed Actions）、OpenClaw テンプレートはより精錬、ただし `Exact identifiers` セクションが明示的に IDs/URLs/ハッシュ/ポート等の字面値保持を要求。

### 項目別比較

| 観点 | Hermes v3 | OpenClaw |
|------|-----------|----------|
| 操作記録 | `Completed Actions` 番号リスト `N. ACTION target — outcome [tool: name]` | 専用セクションなし、Decisions に融合 |
| ランタイム状態 | `Active State`（ブランチ、テスト状態、実行プロセス） | なし |
| 精密値保持 | `Critical Context` セクション | `Exact identifiers` セクション（IDs/URLs/ハッシュ/ポート） |
| 未回答質問追跡 | `Pending User Asks` + `Resolved Questions`（回答済 / 未回答区別） | `Pending user asks`（未回答のみ追跡） |
| ファイル追跡 | `Relevant Files` 独立セクション | なし、Exact identifiers でパス保持 |
| 品質チェック | なし（LLM 出力を信頼） | `auditSummaryQuality()` が 5 セクションチェック、不合格でリトライ、フォールバック骨格生成 |
| 反復更新 | "continue numbering" で旧要約番号継続 | `previousSummary` を次チャンクに渡す |
| 要約上限 | 圧縮内容 × 0.2、12K tokens 上限 | ハード 16,000 文字 |
| アンチスラッシング | 連続 2 回 <10% → スキップ | 6 種類 skip reason 分類（`already_compacted_recently` 等） |
| 失敗処理 | RuntimeError 600s / 瞬間 60s cooldown | 15 分セーフタイムアウト + 3 回リトライ + 構造化フォールバック |
| ツールペア修復 | `_sanitize_tool_pairs()` で孤児ペア補完 | `repairToolUseResultPairing()` で孤児削除 |
| 尾部保護 | token 予算動的 + ハード底線 3 つ | `DEFAULT_RECENT_TURNS_PRESERVE=3`（上限 12） |
| 多言語 | 特殊処理なし | "Write summary in the primary language" |

### 各々長所

**Hermes の優位点：**
- ローカル前処理（MD5 重複排除 + Smart Collapse）が LLM 前に 30-50% token 削減、OpenClaw にはこの層なし
- 単発 LLM 呼び出し、対話の長さに関わらず 1 回のみ
- テンプレートがより詳細（11 vs 5 セクション）、続行 agent が得るコンテキストがより豊富

**OpenClaw の優位点：**
- 品質チェックループ（審査 → リトライ → フォールバック骨格）、Hermes にはなし
- チャンク戦略が超長対話に自然適応（単発 LLM 入力ウィンドウ制限、チャンクでオーバーフロー回避）
- `Exact identifiers` セクションで重要字面値を明示保持
- スキップ理由分類がより詳細（6 種類 reason）、デバッグに役立つ

## Prompt Caching との相互作用

Anthropic の prompt caching はシステムプロンプトプレフィックスに最も効果的。圧縮戦略はキャッシュと協調：

1. **システムプロンプト不変保持** — キャッシュヒット最大化
2. **対話履歴のみ圧縮** — メッセージ部分は可変
3. **同じシステムプロンプト構造使用** — キャッシュキー安定

## Agent ループでのトリガー

```python
while api_call_count < max_iterations and iteration_budget.remaining > 0:
    # token 予算チェック
    if token_usage > threshold:
        compressed = compressor.compress(messages, current_tokens=token_usage)
        messages = [system_prompt] + [compressed] + recent_messages
```

## 他システムとの関係

- [[auxiliary-client-architecture]] — 圧縮は `call_llm(task="compression")` 経由で呼ばれる
- [[smart-model-routing]] — `get_model_context_length()` でコンテキストウィンドウ取得
- [[prompt-builder-architecture]] — 圧縮後メッセージを prompt builder に渡してプロンプト再構築
- [[prompt-caching-optimization]] — 圧縮戦略は prompt caching と協調
- [[large-tool-result-handling]] — ツール出力修剪と大型結果処理の理念が共通
- [[session-search-and-sessiondb]] — Session 分裂後の元メッセージは DB に保持されて検索可能
- [[memory-system-architecture]] — 圧縮前 flush_memories と on_pre_compress 通知
