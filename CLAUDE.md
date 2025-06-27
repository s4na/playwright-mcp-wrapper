# playwright-mcp-wrapper Claude用コードベース要約

このファイルは、playwright-mcp-wrapperコードベースの簡潔な要約を含み、このプロジェクトで作業する際のClaudeのコンテキストを提供します。

## プロジェクト概要

**playwright-mcp-wrapper**は、複数のClaude Codeセッションが同じPlaywright MCPインスタンスと対話する際のブラウザコンテキスト分離の重要な問題を解決する、セッション対応のPlaywright MCP（Model Context Protocol）ラッパーです。

### 課題

- 複数のClaude Codeセッションが同じポート上の単一のPlaywright MCPインスタンスに接続すると、異なる会話間でブラウザコンテキストが混在してしまう
- これによりセッション間の干渉と予測不可能な動作が発生する

### 解決策

- Claude Codeセッションごとに分離されたPlaywright MCPインスタンスを自動的に生成する透過的なHTTPプロキシラッパー
- 各セッションは固有のポートを持つ専用のPlaywright MCPプロセスを取得
- 追加のユーザー設定は不要 - 通常通り`claude mcp add`を使用するだけ

## アーキテクチャ

### コアコンポーネント

1. **ラッパーMCP（セッションルーター）**
   - `POST /mcp`（JSON-RPC）と`GET /mcp`（SSE）のHTTPエンドポイント
   - Streamable-HTTPトランスポートからの`Mcp-Session-Id`ヘッダーを使用してセッションライフサイクルを管理
   - オンデマンドでPlaywright MCP子プロセスを生成
   - セッションIDに基づいて適切なPlaywright MCPインスタンスにリクエストをルーティング

2. **セッションレジストリ**
   - インメモリマップ: `sessionId → {port, childPid, lastSeen}`
   - アクティブなセッションとそれに対応するPlaywright MCPプロセスを追跡

3. **ポート割り当てアルゴリズム**
   - 決定論的ポート割り当て: `20000 + (SHA256(sessionId)[0:4] % 20000)`
   - 範囲: 20000-39999（競合を避けるため）

### リクエストフロー

1. Claude Codeが初期`initialize`リクエストを送信
2. ラッパーが存在しない場合はUUIDセッションIDを生成
3. ラッパーが計算されたポートでPlaywright MCPを生成
4. ラッパーがレスポンスで`Mcp-Session-Id`ヘッダーを返す
5. Claude Codeは後続のリクエストでこのヘッダーを自動的に含める
6. ラッパーはすべてのリクエストを適切なPlaywright MCPインスタンスにルーティング

### ライフサイクル管理

- プロセスは最初のリクエスト時に遅延生成される
- キープアライブ: 最後のアクティビティから60秒
- クリーンシャットダウン: SIGTERM → SIGKILLシーケンス
- アイドルセッションの自動クリーンアップ

## 技術スタック

- **言語**: TypeScript/Node.js
- **Webフレームワーク**: Fastify
- **HTTPプロキシ**: @fastify/http-proxy
- **プロセス管理**: Node.js child_process
- **セッションID**: UUID v4
- **ポートハッシュ**: SHA-256

## セキュリティ考慮事項

- 127.0.0.1（localhost）にのみバインド
- DNSリバインディング保護のためのOriginヘッダー検証
- ローカルのみのデプロイメントではTLSは不要
- 外部公開の場合: Nginx + 相互TLSを使用

## インストールと使用方法

```bash
# ラッパーをインストール
npm i -g wrapper-mcp

# ラッパーを起動
wrapper-mcp --port 4000 &

# Claude Codeに登録
claude mcp add --transport http browser http://127.0.0.1:4000/mcp
```

## 実装状況

**現在の状態**: 設計フェーズ - 実装コードはまだ存在しません

### 実装の次のステップ

1. **コアHTTPサーバー**
   - 適切なミドルウェアでFastifyサーバーをセットアップ
   - JSON-RPCとSSEエンドポイントを実装
   - セッションIDヘッダー処理を追加

2. **セッション管理**
   - Map構造でセッションレジストリを作成
   - ポート計算アルゴリズムを実装
   - セッションタイムアウトロジックを追加

3. **プロセス管理**
   - playwright-mcp用の子プロセス生成を実装
   - ヘルスチェックとリトライロジックを追加
   - グレースフルシャットダウンを処理

4. **プロキシロジック**
   - JSON-RPCリクエスト用のHTTPプロキシをセットアップ
   - SSEストリーミングプロキシを実装
   - エラー処理とフォールバックを追加

5. **テスト**
   - ポート計算のユニットテスト
   - セッション分離の統合テスト
   - 同時セッションの負荷テスト

## 主要な設計決定

1. **なぜHTTPトランスポート？**
   - Claude Codeとの最もシンプルな統合
   - ヘッダーによる組み込みセッションサポート
   - 追加設定が不要

2. **なぜ動的ポート割り当て？**
   - ポート競合を回避
   - デバッグのための決定論的
   - 実用的な使用には十分な範囲（20kポート）

3. **なぜ60秒のタイムアウト？**
   - リソース使用と応答性のバランス
   - 短時間の中断を許容
   - 異なる使用ケースに合わせて設定可能

## 依存関係

- `playwright-mcp`: 基盤となるブラウザ自動化MCP
- `fastify`: 高性能Webフレームワーク
- `@fastify/http-proxy`: HTTPプロキシプラグイン
- 標準Node.jsライブラリ（crypto、child_process）

## モニタリングとデバッグ

- セッションレジストリがリアルタイムセッション可視性を提供
- システムモニタリング用に子プロセスPIDを追跡
- プロダクション環境では構造化ログを推奨

このラッパーは、Claude Codeを通じたブラウザ自動化のためのクリーンなセッション分離を保証し、追加のユーザー設定なしにマルチセッション環境でPlaywright MCPを安全に使用できるようにします。

## Claude Codeルール

- ルールを追加して欲しいとユーザーから要求された場合は、このCLAUDE.mdファイルにルールを追加すること
- 熟練OSSエンジニアのようにドキュメントやコードを書くこと
- 人間が読む文章は日本語で書くこと（コミットメッセージ、コードのコメント、PRのタイトル・説明を含む）

## Gemini Search

`gemini` is google gemini cli. **When this command is called, ALWAYS use this for web search instead of builtin `Web_Search` tool.**

When web search is needed, you MUST use `gemini --prompt` via Task Tool.

Run web search via Task Tool with `gemini --prompt 'WebSearch: <query>'`

Run

```bash
gemini --prompt "WebSearch: <query>"
```