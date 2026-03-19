# CLAUDE.md — AI アシスタント向けガイド

このファイルは、Claude などの AI アシスタントがこのリポジトリで作業する際に参照するためのガイドです。

---

## プロジェクト概要

**Desktop Commander MCP** は、Claude AI にターミナル操作・ファイル編集・システムアクセスを提供する [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) サーバーです。

- **npm パッケージ名:** `@wonderwhy-er/desktop-commander`
- **現在のバージョン:** 0.2.37
- **ライセンス:** MIT
- **Node.js 要件:** >=18.0.0
- **モジュール形式:** ESM (ES Modules)
- **主要エントリーポイント:** `dist/index.js`

---

## ディレクトリ構造

```
DesktopCommanderMCP/
├── src/                        # TypeScript ソースコード
│   ├── index.ts                # メインエントリーポイント（CLI 引数処理・サーバー起動）
│   ├── server.ts               # MCP サーバー定義（30+ ツール登録）
│   ├── config-manager.ts       # 設定の読み書き管理
│   ├── terminal-manager.ts     # ターミナルセッション・プロセス管理
│   ├── command-manager.ts      # コマンド検証・セキュリティチェック
│   ├── search-manager.ts       # ripgrep ベースの検索セッション管理
│   ├── custom-stdio.ts         # JSON-RPC 準拠のカスタム stdio トランスポート
│   ├── error-handlers.ts       # 共通エラーレスポンス生成
│   ├── handlers/               # MCP ツールハンドラー（server.ts から呼び出し）
│   │   ├── index.ts
│   │   ├── filesystem-handlers.ts
│   │   ├── terminal-handlers.ts
│   │   ├── process-handlers.ts
│   │   ├── edit-search-handlers.ts
│   │   ├── search-handlers.ts
│   │   └── history-handlers.ts
│   ├── tools/                  # ツール実装本体
│   │   ├── schemas.ts          # 全ツールの Zod 入力スキーマ
│   │   ├── filesystem.ts       # ファイル操作（読み書き・ディレクトリ・移動）
│   │   ├── edit.ts             # テキスト編集（fuzzy マッチ・Excel・DOCX）
│   │   ├── process.ts          # プロセス一覧・管理
│   │   ├── improved-process-tools.ts  # 拡張プロセス操作
│   │   ├── config.ts           # 設定 get/set
│   │   └── pdf/                # PDF 生成・解析
│   ├── utils/                  # ユーティリティ
│   │   ├── capture.ts          # テレメトリイベント送信
│   │   ├── logger.ts           # ログ出力
│   │   ├── system-info.ts      # OS・環境情報取得
│   │   ├── feature-flags.ts    # A/B テスト・機能フラグ（Supabase）
│   │   ├── toolHistory.ts      # ツール呼び出し履歴（インメモリ）
│   │   ├── usageTracker.ts     # 使用統計
│   │   ├── fuzzySearchLogger.ts # fuzzy 検索マッチのログ
│   │   ├── lineEndingHandler.ts # CRLF/LF 正規化
│   │   ├── process-detection.ts # REPL プロンプト検出
│   │   ├── ripgrep-resolver.ts  # ripgrep バイナリのパス解決
│   │   └── files/              # ファイル形式別ハンドラー（ファクトリーパターン）
│   ├── ui/                     # UI コンポーネント（設定エディタ・ファイルプレビュー）
│   │   ├── config-editor/
│   │   ├── file-preview/
│   │   ├── resources.ts
│   │   └── contracts.ts
│   └── remote-device/          # リモートデバイス機能（ChatGPT/Claude Web 連携）
│       ├── device.ts
│       ├── device-authenticator.ts
│       ├── remote-channel.ts
│       └── desktop-commander-integration.ts
├── dist/                       # コンパイル済み JavaScript（ビルド成果物）
├── test/                       # テストスイート（40+ テストファイル）
│   └── run-all-tests.js        # テストランナー
├── scripts/                    # ビルド・ユーティリティスクリプト
├── .github/                    # GitHub Actions CI/CD
├── config.json                 # デフォルト設定（ブロックコマンド一覧）
├── tsconfig.json               # TypeScript 設定
├── package.json                # npm パッケージ設定
├── README.md                   # ユーザー向け公開ドキュメント
├── SECURITY.md                 # セキュリティポリシー
└── Dockerfile                  # Docker サポート
```

---

## 開発セットアップ

```bash
# 依存パッケージのインストール
npm install

# TypeScript コンパイル + アセットコピー + UI ビルド
npm run build

# サーバーの起動（ビルド済み）
npm start

# 開発中（watch モード）
npm run watch
```

---

## 主要スクリプト一覧

| スクリプト | 説明 |
|---|---|
| `npm run build` | `tsc` + アセットコピー + UI ランタイムビルド |
| `npm test` | ビルド後に全テストを実行 |
| `npm run validate:tools` | ツール定義の整合性チェック |
| `npm run setup` | インストール + Claude Desktop への登録 |
| `npm run remove` | Claude Desktop から削除 |
| `npm run device:start` | リモートデバイスモードの起動 |
| `npm run logs:view` | fuzzy 検索ログの表示 |
| `npm run logs:analyze` | fuzzy 検索ログの分析 |
| `npm run count-tokens` | MCP ツール定義のトークン数カウント |
| `npm run inspector` | MCP インスペクターの起動（デバッグ用） |
| `npm run bump` | パッチバージョンのインクリメント |
| `npm run release` | npm + MCP レジストリへの公開 |

---

## アーキテクチャ概要

```
Claude Desktop / AI クライアント
        ↓ stdio (JSON-RPC)
FilteredStdioServerTransport  (src/custom-stdio.ts)
        ↓
MCP Server  (src/server.ts)
  ├── ListToolsRequestSchema   → 利用可能ツール一覧を返す
  ├── CallToolRequestSchema    → ツール実行 → handlers/ へ委譲
  ├── ReadResourceRequestSchema → UI リソース配信
  └── InitializeRequestSchema  → クライアントハンドシェイク
        ↓
handlers/ (引数バリデーション → tools/ へ委譲)
        ↓
tools/ (実装本体)
  ├── filesystem.ts   → ファイル操作
  ├── edit.ts         → テキスト編集
  ├── process.ts      → プロセス管理
  └── pdf/            → PDF 操作
        ↓
managers (状態管理)
  ├── TerminalManager → ターミナルセッション
  ├── SearchManager   → 検索セッション
  └── ConfigManager   → 設定
```

### カスタムトランスポート (`src/custom-stdio.ts`)

MCP プロトコルの準拠のため、`console.log` などの標準出力を JSON-RPC 通知にラップする独自トランスポートを実装しています。クライアント接続前のメッセージはバッファリングされます。

---

## ソースコードマップ (`src/`)

### `src/index.ts`
- CLI 引数解析（`setup` / `remove` / `remote` サブコマンド）
- 設定・機能フラグの初期化
- `FilteredStdioServerTransport` の生成と接続
- 未処理例外ハンドラーの登録

### `src/server.ts`（約 1,550 行）
MCP サーバーの中核。以下のカテゴリで 30+ ツールを定義・登録：

| カテゴリ | ツール名 |
|---|---|
| 設定 | `get_config`, `set_config_value` |
| ターミナル | `start_process`, `read_process_output`, `interact_with_process`, `force_terminate`, `list_sessions` |
| プロセス管理 | `list_processes`, `kill_process` |
| ファイル操作 | `read_file`, `write_file`, `read_multiple_files`, `create_directory`, `list_directory`, `move_file`, `get_file_info` |
| 検索 | `start_search`, `get_more_search_results`, `stop_search`, `list_searches` |
| テキスト編集 | `edit_block` |
| PDF | `write_pdf` |
| 分析 | `get_usage_stats`, `get_recent_tool_calls`, `give_feedback_to_desktop_commander` |

### `src/config-manager.ts`
- シングルトンパターン
- 設定ファイル: `~/.claude-server-commander/config.json`
- デフォルト値を持ち、ディスクへ永続化

### `src/terminal-manager.ts`
- ターミナルセッションの生成・管理
- bash / zsh / PowerShell / fish / cmd 対応
- REPL プロンプト検出（`>>>`, `>`, `$`, `#` など）
- ページネーション付き出力読み取り
- セッション完了後も最大 100 件保持

### `src/command-manager.ts`
- シェルコマンド文字列の解析・検証
- パイプ（`|`）、セミコロン、コマンド置換（`$(...)`, バッククォート）の検出
- ブロックリストとのマッチング
- エラー時はフェイルクローズ（実行拒否）

### `src/search-manager.ts`（約 1,000 行）
- ripgrep プロセスを非同期で起動
- ファイル名検索 / コンテンツ検索の 2 モード
- ページネーション付き結果読み取り
- 検索セッションの並列管理

---

## ツールの追加方法

新しい MCP ツールを追加する手順：

1. **スキーマ定義** (`src/tools/schemas.ts`)
   ```typescript
   export const MyToolArgsSchema = z.object({
     param: z.string(),
     optional: z.number().optional(),
   });
   ```

2. **ハンドラー作成** (`src/handlers/my-handler.ts`)
   ```typescript
   import { MyToolArgsSchema } from '../tools/schemas.js';
   import { ServerResult } from '../types.js';

   export async function handleMyTool(args: unknown): Promise<ServerResult> {
     const parsed = MyToolArgsSchema.parse(args);
     // 実装
     return { content: [{ type: 'text', text: 'result' }] };
   }
   ```

3. **ツール登録** (`src/server.ts`)
   - `ListToolsRequestSchema` ハンドラー内のツール配列に追加
   - `CallToolRequestSchema` ハンドラーの switch 文に case を追加
   - `handleMyTool` をインポート

---

## 設定システム

### 設定ファイルの場所
```
~/.claude-server-commander/config.json
```

### 既知の設定キー

| キー | デフォルト | 説明 |
|---|---|---|
| `blockedCommands` | `["sudo","su","passwd","adduser",...]` | 実行を拒否するコマンド一覧 |
| `defaultShell` | OS 依存 (zsh/bash/PowerShell) | コマンド実行に使うシェル |
| `allowedDirectories` | `[]`（= 制限なし） | ファイル操作を許可するパス一覧 |
| `fileReadLineLimit` | `1000` | 1 回の読み取り最大行数 |
| `fileWriteLineLimit` | `50` | 1 回の書き込み最大行数 |
| `telemetryEnabled` | `true` | テレメトリの有効/無効 |

### 設定の変更
- MCP ツール `set_config_value` 経由（既知キーのみ受け付け）
- 直接ファイル編集も可能

---

## セキュリティモデル

### 保護対象
- **コマンドブロックリスト:** デフォルトで `sudo`, `su`, `passwd`, `dd`, `mkfs`, `fdisk`, `format`, `mount` などをブロック
- **allowedDirectories:** 設定されている場合、ファイル操作をそのパス以下に制限
- **コマンド置換検出:** `$()` やバッククォートによる迂回を検出

### 既知の制限（`SECURITY.md` 参照）
- シンボリックリンクで `allowedDirectories` を迂回できる
- 絶対パス指定や変数展開でブロックリストを迂回できる
- ターミナルコマンドは `allowedDirectories` を無視する
- このツールは「AIの意図しない操作を防ぐガードレール」であり、堅牢なセキュリティ境界ではない
- 本番環境での堅牢なセキュリティが必要な場合は Docker 使用を推奨

---

## ファイル形式サポート

`src/utils/files/` のファクトリーパターンで実装：

| 形式 | ハンドラー | 対応拡張子 |
|---|---|---|
| テキスト | `TextFileHandler` | `.ts`, `.js`, `.md`, `.txt` など |
| Excel | `ExcelFileHandler` | `.xlsx`, `.xls`, `.xlsm` |
| PDF | `PdfFileHandler` | `.pdf` |
| 画像 | `ImageFileHandler` | `.png`, `.jpg`, `.gif`, `.webp` |
| DOCX | `DocxFileHandler` | `.docx` |
| バイナリ | `BinaryFileHandler` | その他 |

`getFileHandler(filePath)` を呼び出すと適切なハンドラーが返される。

---

## テスト

```bash
# 全テストを実行（ビルドも含む）
npm test

# デバッグ付きで実行
npm run test:debug
```

### テスト構造 (`test/`)
- `run-all-tests.js` がテストランナー
- 40+ のテストファイルをカバー:
  - ファイル操作（読み書き・権限）
  - 検索（リテラル・正規表現・コンテンツ）
  - 編集ブロック（fuzzy マッチ・複数箇所置換）
  - プロセス実行・ターミナルインタラクション
  - 設定・allowedDirectories
  - Excel / PDF ファイル処理
  - セキュリティ（ブロックリスト迂回・シンボリックリンク）
  - 行末処理・負のオフセット読み取り
  - REPL インタラクション（Python・Node.js）

---

## コーディング規約

### 言語・モジュール
- **TypeScript** strict モード（`tsconfig.json` 参照）
- **ESM** (`"type": "module"`) — `import` / `export` を使用
- インポートパスには `.js` 拡張子を付ける（ESM 要件）
  ```typescript
  import { foo } from './foo.js';
  ```

### 型・バリデーション
- ツールの引数は必ず **Zod スキーマ** でバリデーション（`src/tools/schemas.ts`）
- ハンドラー内では `Schema.parse(args)` を使用（`safeParse` ではなく）

### 戻り値の型
```typescript
interface ServerResult {
  content: ServerResponseContent[];
  structuredContent?: FilePreviewStructuredContent;
  isError?: boolean;
  _meta?: Record<string, unknown>;
}
```

### エラーハンドリング
- エラー時は `isError: true` を含む `ServerResult` を返す
- `src/error-handlers.ts` の `createErrorResponse()` を使用
- テレメトリへ送信する前にファイルパスなどの機密情報を除去

### ページネーションパターン
ファイル読み取り・プロセス出力・検索結果で共通のページネーション構造:
```typescript
{
  lines: string[];
  totalLines: number;
  readFrom: number;
  readCount: number;
  remaining: number;
  isComplete: boolean;
}
```

### REPL 検出
`src/utils/process-detection.ts` の `analyzeProcessState()` でプロンプト待ち状態を検出。ターミナルコマンドはこれを使用して早期リターンを判断する。

### Fuzzy Edit マッチング
`edit_block` ツールは類似度 0.7 以上で fuzzy マッチを行う。マッチしない場合は文字レベルの diff を返す。`expected_replacements` パラメーターで複数箇所の置換に対応。

---

## リモートデバイス機能

ChatGPT や Claude Web からローカルの Desktop Commander を操作できる機能。

### 主要ファイル
- `src/remote-device/device.ts` — WebSocket 接続管理・メインループ
- `src/remote-device/device-authenticator.ts` — OAuth 2.0 デバイスフロー認証
- `src/remote-device/remote-channel.ts` — クラウドサービスとの通信
- `src/remote-device/desktop-commander-integration.ts` — ローカル MCP サーバーへのブリッジ

### 起動方法
```bash
cd src/remote-device && npm install
npm run device:start
```

---

## テレメトリ

### 収集内容
- プラットフォーム・バージョン・ランタイム情報
- ツール名・クライアント名（ファイルパスや個人情報は**含まない**）
- 擬似匿名のクライアント UUID

### 無効化
```json
// ~/.claude-server-commander/config.json
{ "telemetryEnabled": false }
```

または MCP ツール経由:
```
set_config_value("telemetryEnabled", false)
```

### サニタイズ
`src/utils/capture.ts` でエラーメッセージからファイルパス・機密キーを自動除去してから送信する。

---

## よくある作業パターン

### 新しいファイル形式のサポートを追加する
1. `src/utils/files/` に新しいハンドラークラスを作成（`FileHandler` 基底クラスを継承）
2. ファクトリー関数 `getFileHandler()` に拡張子の条件分岐を追加

### ブロックコマンドを追加する
設定ファイルまたは `set_config_value` ツールで `blockedCommands` 配列を更新する。

### ビルド確認
```bash
npm run build && npm run validate:tools
```
