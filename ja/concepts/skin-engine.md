---
title: スキンエンジン（Skin / Theme）
created: 2026-04-10
updated: 2026-04-10
type: concept
tags: [cli, theme, customization]
sources: [hermes_cli/skin_engine.py]
translation: ja
original: ../../concepts/skin-engine.md
---

# スキンエンジン

## 概要

Hermes CLI の見た目は完全に YAML で駆動されます。ユーザーはコードを変更せずに色、スピナーアニメーション、ブランド文言をカスタマイズできます。

## スキンファイル構造

スキンファイルは `~/.hermes/skins/*.yaml` に置きます。すべてのフィールドは任意で、欠落値は `default` スキンから継承されます。

```yaml
name: mytheme
description: カスタムテーマ

colors:
  banner_border: "#CD7F32"     # Banner 枠線
  banner_title: "#FFD700"      # Banner タイトル
  banner_accent: "#FFBF00"     # セクションタイトル
  ui_accent: "#FFBF00"         # UI 強調色
  ui_ok: "#4caf50"             # 成功
  ui_error: "#ef5350"          # エラー
  ui_warn: "#ffa726"           # 警告
  prompt: "#FFF8DC"            # 入力プロンプト
  response_border: "#FFD700"   # 回答ボックス枠線

spinner:
  waiting_faces: ["(⚔)", "(⛨)"]
  thinking_faces: ["(⌁)", "(<>)"]
  thinking_verbs: ["forging", "plotting"]
  wings: [["⟪⚔", "⚔⟫"], ["⟪▲", "▲⟫"]]

branding:
  agent_name: "My Agent"
  welcome: "Welcome!"
  goodbye: "Bye! ⚕"
  response_label: " ⚕ Response "
  prompt_symbol: "❯ "
```

## スキン切り替え

```bash
/skin mytheme          # セッション内で切替
hermes config set display.skin mytheme  # 永続化設定
```

## Profile ごとに異なるスキンを使用可能

スキンファイルは Profile の `skins/` ディレクトリに配置されます。Profile ごとに異なるビジュアルテーマを使用できます。

## 関連ページ

- [[configuration-and-profiles]] — Profile システム（各 Profile ごとに独立した skins ディレクトリ）
- [[cli-architecture]] — CLI アーキテクチャ

## 主要ソース

- `hermes_cli/skin_engine.py` — スキンロード、継承、レンダリング
