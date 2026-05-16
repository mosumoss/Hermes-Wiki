---
title: AIAgent Class
created: 2026-04-07
updated: 2026-04-07
type: entity
tags: [component, agent, module]
sources: [hermes-agent ソースコード解析 2026-04-07]
translation: ja
original: ../../entities/aiagent-class.md
---

# AIAgent Class

## 場所

`run_agent.py`

## 概要

AIAgent は Hermes Agent のコア対話ループクラスで、LLM とのやりとり、ツール呼び出し、セッション状態の管理を担います。

## コンストラクタ

```python
class AIAgent:
    def __init__(self,
        model: str = "",  # デフォルト空文字列、ランタイムで "anthropic/claude-opus-4.6" に解決
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        quiet_mode: bool = False,
        save_trajectories: bool = False,
        platform: str = None,           # "cli", "telegram" など
        session_id: str = None,
        skip_context_files: bool = False,
        skip_memory: bool = False,
        # ... 他のパラメータ：provider, api_mode, callbacks, routing パラメータ
    ):
```

## コアメソッド

### `chat(self, message: str, stream_callback: Optional[callable] = None) -> str`

シンプルなインターフェース、最終応答文字列を返す。

### `run_conversation(self, user_message: str, system_message: str = None, conversation_history: List[Dict] = None, task_id: str = None, stream_callback: Optional[callable] = None, persist_user_message: Optional[str] = None) -> Dict[str, Any]`

完全なインターフェース、`{final_response, messages}` 辞書を返す。

## 対話ループ

```python
while api_call_count < self.max_iterations and self.iteration_budget.consume():  # consume() が残り予算をアトミックに確認・減算
    response = client.chat.completions.create(
        model=model,
        messages=messages,
        tools=tool_schemas
    )
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id, tool_call.id, session_id, user_task, enabled_tools)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        return response.content
```

## 主要特性

- **完全同期** — asyncio を使わない
- **ツールループ** — 複数ターンのツール呼び出しに対応
- **イテレーション予算** — 最大 API 呼び出し回数を制御
- **プラットフォーム認識** — プラットフォームに応じて異なるプロンプトを注入
- **記憶統合** — 自動で記憶をロード・注入
- **スキル統合** — スキルインデックスを構築
- **コンテキスト圧縮** — 自動でコンテキスト長を管理

## 関連ページ

- [[agent-loop-and-prompt-assembly]] — Agent コアループとシステムプロンプト組み立て
- [[multi-agent-architecture]] — サブエージェント委譲とイテレーション予算システム
- [[prompt-builder-architecture]] — システムプロンプト構築アーキテクチャ

## 関連ファイル

- `run_agent.py` — 実装
- `model_tools.py` — ツール編成
- `agent/prompt_builder.py` — システムプロンプト構築
