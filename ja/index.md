# Wiki インデックス（日本語版）

> 目次。各 wiki ページを種別ごとに整理し、一行要約を添えています。
> 調査前にまずこのファイルを読んで関連ページを見つけてください。
> 最終更新: 2026-04-08 | 総ページ数: 33

> **原文（中国語）**: [`../index.md`](../index.md)

## Entities（エンティティ）

- [[aiagent-class]] — コア対話ループクラス。LLM とのやりとりとツール呼び出しを管理
- [[memorystore-class]] — 記憶システムのコアクラス。`MEMORY.md` と `USER.md` を管理

## Concepts（概念）

### コアアーキテクチャ

- [[tool-registry-architecture]] — 中央ツール登録システム。宣言的登録 + 集中ディスパッチ、循環インポート安全
- [[auxiliary-client-architecture]] — 補助 LLM クライアントのルーター。複数プロバイダ解決チェーン + アダプタパターン + 自動フォールバック
- [[browser-tool-architecture]] — 複数バックエンドのブラウザ自動化。アクセシビリティツリーのテキスト表現 + 三層セキュリティ防御 + 並行隔離
- [[web-tools-architecture]] — 複数バックエンドの検索/抽出/クロール。LLM による知的コンテンツ圧縮（チャンク分割 + 合成）、四層セキュリティ防御
- [[skills-system-architecture]] — 段階的開示（progressive disclosure）アーキテクチャ。スキル発見、条件付き活性化、シークレット管理
- [[memory-system-architecture]] — フリーズスナップショット方式、アトミック書き込み、セキュリティスキャン
- [[agent-loop-and-prompt-assembly]] — Agent ループ、システムプロンプト構築、プラットフォームプロンプト、実行ガイダンス
- [[skills-and-memory-interaction]] — Skills と Memory の補完関係と意思決定ツリー
- [[toolsets-system]] — ツールグループ化システム、再帰解決、14+ プラットフォーム ツールセット
- [[session-search-and-sessiondb]] — FTS5 検索 + LLM 要約による会話横断リコール

### パフォーマンスと最適化

- [[parallel-tool-execution]] — 知的並列実行の安全検出。三層分類 + パス競合検出
- [[prompt-caching-optimization]] — Anthropic の `system_and_3` キャッシュ戦略、75% コスト削減
- [[fuzzy-matching-engine]] — 8 段階チェーンのファジーマッチング。完全一致から類似度マッチングまで
- [[smart-model-routing]] — 知的モデルルーティング。10 段階のコンテキスト長解決チェーン + ローカルサーバー自動検出
- [[large-tool-result-handling]] — 大型結果のファイル化、プリフライト圧縮、Surrogate クリーンアップ

### セキュリティと信頼性

- [[security-defense-system]] — 5 層防御体系、100+ 脅威パターン検出
- [[interrupt-and-fault-tolerance]] — 中断伝播、認証情報プールのローテーション、Fallback モデルチェーン
- [[credential-pool-and-isolation]] — 複数鍵自動ローテーション、Profile 隔離
- [[multi-agent-architecture]] — マルチ Agent 体系。サブエージェント委譲 + バッチ処理 + プラットフォーム横断通信

### プラットフォームと拡張

- [[cli-architecture]] — CLI アーキテクチャ、スラッシュコマンド補完、Skin エンジン
- [[configuration-and-profiles]] — 階層型設定、Profile 隔離、自動マイグレーション
- [[hook-system-architecture]] — Hook システム（Gateway Hooks + Plugin System）。イベント駆動 + ツール登録 + コンテキスト注入
- [[mcp-and-plugins]] — MCP 統合、プラグインフックシステム、OAuth サポート
- [[terminal-backends]] — 6 種類のターミナルバックエンド、環境抽象化、永続シェル
- [[cron-scheduling]] — 内蔵スケジューラ、自然言語スケジューリング、複数プラットフォーム配信
- [[trajectory-and-data-generation]] — 軌跡（trajectory）保存、バッチランナー、RL 訓練環境
- [[prompt-builder-architecture]] — システムプロンプトのモジュラー組み立て。インジェクション防御 + スキルキャッシュ + モデル固有ガイダンス
- [[context-compressor-architecture]] — 自動コンテキスト圧縮。構造化要約 + 反復更新 + ツールペア完全性保証
- [[model-tools-dispatch]] — ツール編成とディスパッチ。非同期ブリッジ + 動的スキーマ調整 + パラメータ型強制
- [[gateway-session-management]] — ゲートウェイセッション管理。マルチプラットフォームセッション隔離 + PII マスキング + リセット戦略
- [[messaging-gateway-architecture]] — メッセージングゲートウェイアーキテクチャ。プラットフォームアダプタ、DM ペアリング
