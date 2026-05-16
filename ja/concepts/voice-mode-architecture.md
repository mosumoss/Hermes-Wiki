---
title: 音声モードアーキテクチャ
created: 2026-04-10
updated: 2026-04-18
type: concept
tags: [voice, stt, tts, architecture]
sources: [tools/voice_mode.py, tools/tts_tool.py, tools/transcription_tools.py, cli.py]
translation: ja
original: ../../concepts/voice-mode-architecture.md
---

# 音声モードアーキテクチャ

## 概要

Hermes は Push-to-talk 音声対話をサポートします：ユーザーがキーを押して録音 → STT で文字化 → LLM が処理 → TTS で音声応答。フロー全体が CLI 内で完結し、オプションのオーディオライブラリに依存します。

## 依存

```bash
pip install sounddevice numpy   # または
pip install hermes-agent[voice]
```

オーディオライブラリは**必要時に遅延ロード**、インストールしなくてもテキストモードに影響しません。オーディオデバイスのない環境（SSH、Docker、WSL）では自動検出して無効化。

## フロー

```text
ユーザーが Ctrl+B を押して録音開始
    ↓
sounddevice が音声をキャプチャ → WAV 一時ファイル
    ↓
再度 Ctrl+B を押して録音停止
    ↓
STT で文字化（3 つの Provider から選択）:
  - local: faster-whisper（ローカル、API Key 不要）
  - groq: Whisper via Groq（無料枠あり）
  - openai: Whisper via OpenAI
    ↓
転写されたテキストをユーザーメッセージとして LLM に送信
    ↓
LLM が応答（簡潔指示を自動注入："respond concisely, 2-3 sentences max"）
    ↓
TTS で音声再生（5 つの Provider から選択）:
  - ElevenLabs（ストリーミング、生成しながら再生）
  - OpenAI TTS
  - Google TTS
  - macOS の say コマンド
  - NeuTTS（セルフホスト）
```

## STT 設定

```yaml
# config.yaml
stt:
  provider: local   # local | groq | openai（優先度：local > groq > openai）
  model: base       # faster-whisper モデルサイズ（base ~150MB、初回自動 DL）
```

```bash
# .env
GROQ_API_KEY=...              # Groq Whisper（無料）
VOICE_TOOLS_OPENAI_KEY=...    # OpenAI Whisper
```

## TTS 設定

TTS Provider の選択と音声設定は `tools/tts_tool.py` で管理。ElevenLabs のストリーミング再生をサポート — LLM が 1 文生成するごとに 1 文再生、完全な応答を待たない。

### 新規 TTS Provider（v0.10.0）

| Provider | 由来 |
|----------|------|
| ElevenLabs | 既存 |
| OpenAI | 既存 |
| **Google Gemini TTS** | v0.10.0 新規、Gemini API 経由 |
| **xAI TTS** | v0.10.0 で xAI Responses API アップグレードと共に導入 |
| **KittenTTS（ローカル）** | v2026.4.18+ 導入、ローカル CPU 動作、GPU と API key 不要、デフォルトモデル `KittenML/kitten-tts-nano-0.8-int8`（25MB）、デフォルト声 `Jasper`、他の声は KittenTTS パッケージで提供（25-80MB のモデルレンジ） |

これらの provider は Nous Tool Gateway 経由で統一アクセス可能（自前 API key 不要）。

### STT Provider 拡張（v2026.4.18+）

| Provider | 説明 |
|----------|------|
| Groq Whisper（無料） | 既存 |
| OpenAI Whisper | 既存 |
| Deepgram | 既存 |
| **xAI Grok STT** | 新規、POST `/v1/stt`、ITN（Inverse Text Normalization）+ オプションで diarization をサポート |

## 音声モード特殊動作

- LLM が音声入力を受け取った時、システムは前置指示を自動注入して簡潔応答を要求
- この前置指示は API 呼び出しにのみ使用、**会話履歴には永続化しない**（`persist_user_message` パラメータで元の転写テキストを保存）
- 連続音声モードで永続エラー（例：429）に遭遇すると自動停止、エラー → 録音 → エラーの無限ループを防止

## 関連ページ

- [[cli-architecture]] — CLI への音声モード統合
- [[auxiliary-client-architecture]] — STT / TTS は auxiliary モデル設定を使用

## 主要ソース

| ファイル | 責任 |
|------|------|
| `tools/voice_mode.py`（812 行）| 録音、STT ディスパッチ、音声再生 |
| `tools/tts_tool.py`（983 行）| TTS Provider ルーティング、ストリーミング再生 |
| `tools/transcription_tools.py` | STT Provider 統一インターフェース |
| `cli.py` | Push-to-talk キーバインド（Ctrl+B） |
