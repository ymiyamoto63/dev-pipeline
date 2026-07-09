# dev-pipeline

カスタムサブエージェントを使い、要件定義 → 設計 → 実装 → テスト → レビュー → PR作成 の各工程を専任のサブエージェントに委譲する開発パイプライン。Claude Code 版（`agents/` + `commands/`）と GitHub Copilot 版（`copilot/`）の2形式を提供する。

## 構成

```
dev-pipeline/
├── agents/                       … Claude Code 版サブエージェント
│   ├── requirements-analyst.md   … 要件定義（docs/requirements.md を出力）
│   ├── software-architect.md     … 設計（docs/design.md を出力）
│   ├── implementer.md            … 実装（docs/implementation-notes.md に追記）
│   ├── test-engineer.md          … テスト・検証（docs/test-report.md を出力）
│   ├── code-reviewer.md          … コードレビュー（docs/review.md を出力）
│   └── pr-publisher.md           … コミット・PR作成（docs/pr-description.md を出力）
├── commands/
│   └── dev-pipeline.md           … 上記6エージェントを順に呼び出すオーケストレーター（/dev-pipeline コマンド）
└── copilot/                      … GitHub Copilot 版（VS Code / Copilot CLI 用）
    ├── agents/
    │   ├── dev-pipeline.agent.md          … オーケストレーター（カスタムエージェント）
    │   ├── requirements-analyst.agent.md
    │   ├── software-architect.agent.md
    │   ├── implementer.agent.md
    │   ├── test-engineer.agent.md
    │   ├── code-reviewer.agent.md
    │   └── pr-publisher.agent.md
    └── prompts/
        └── dev-pipeline.prompt.md         … /dev-pipeline スラッシュコマンド（プロンプトファイル）
```

各サブエージェントは、対象プロジェクトの `docs/` ディレクトリにドキュメントを出力する。フェーズ間の受け渡しはこのファイルを介して行われるため、パイプライン実行の記録がそのままプロジェクトに残る。

## インストール（Claude Code 版）

Claude Code のユーザー設定ディレクトリ（`~/.claude/`）に配置する。

### Windows (PowerShell)

```powershell
Copy-Item agents\*.md "$HOME\.claude\agents\" -Force
Copy-Item commands\*.md "$HOME\.claude\commands\" -Force
```

### macOS / Linux

```bash
cp agents/*.md ~/.claude/agents/
cp commands/*.md ~/.claude/commands/
```

配置後、**Claude Code の再起動が必要**（サブエージェント/コマンドディレクトリの監視は、セッション開始時に存在していたディレクトリのみが対象のため）。

## 使い方（Claude Code 版）

```
/dev-pipeline <タスクの説明>
```

例:

```
/dev-pipeline ユーザープロフィール編集画面にアバター画像アップロード機能を追加する
```

パイプラインの流れ:

1. **要件定義** (`requirements-analyst`) — タスクを要件定義書に変換。曖昧な点は Open Questions としてユーザーに確認
2. **設計** (`software-architect`) — 要件定義書から実装設計書を作成。非自明な変更はユーザー確認後に次へ
3. **実装** (`implementer`) — 設計のステップごとにスコープを絞って実装
4. **テスト** (`test-engineer`) — 受け入れ基準を実コマンド実行で検証。失敗時は実装に差し戻して再検証（最大3回）
5. **レビュー** (`code-reviewer`) — 実装差分をレビューし、正しさの欠陥を報告。指摘があれば実装に差し戻して再検証（最大2回）
6. **PR作成** (`pr-publisher`) — ユーザー確認後、feature branch にコミットしPRを作成

## GitHub Copilot 版

`copilot/` 配下は同じパイプラインの GitHub Copilot 対応版。VS Code のカスタムエージェント + サブエージェント委譲の仕組みで動作する（`.agent.md` は Copilot CLI でも同形式で読み込まれる）。

### インストール

**対象プロジェクトのリポジトリに**配置する（Claude Code 版と違いユーザーグローバルではなくプロジェクト単位が基本）。

```powershell
# Windows (PowerShell) — <target-project> は開発対象リポジトリのルート
New-Item -ItemType Directory -Force <target-project>\.github\agents, <target-project>\.github\prompts
Copy-Item copilot\agents\*.agent.md <target-project>\.github\agents\ -Force
Copy-Item copilot\prompts\*.prompt.md <target-project>\.github\prompts\ -Force
```

```bash
# macOS / Linux
mkdir -p <target-project>/.github/{agents,prompts}
cp copilot/agents/*.agent.md <target-project>/.github/agents/
cp copilot/prompts/*.prompt.md <target-project>/.github/prompts/
```

全プロジェクト共通で使いたい場合は、VS Code のコマンドパレットから「Chat: New Custom Agent File」→ User を選んでユーザープロファイルに置く（Copilot CLI なら `~/.copilot/agents/`）。

### 使い方

VS Code の Copilot Chat で:

- `/dev-pipeline <タスクの説明>` （プロンプトファイル経由）、または
- エージェントドロップダウンから **dev-pipeline** を選択してタスクを入力

オーケストレーターが `agents` frontmatter に列挙された6つのフェーズエージェントをサブエージェントとして順に呼び出す。各フェーズエージェントは単体でもドロップダウンから直接呼び出せる。

### Claude Code 版との相違点

- **モデル指定なし**: Claude Code 版はフェーズごとに sonnet/opus を固定しているが、Copilot 版はモデルピッカーの選択に従う。フェーズごとに固定したい場合は各 `.agent.md` の frontmatter に `model: <モデル名>`（モデルピッカー表示名）を追記する。設計・レビューには高性能モデルの指定を推奨
- **ユーザー確認**: Claude Code の AskUserQuestion の代わりに、チャット上で直接質問して回答を待つ
- **ツール名**: `tools` は VS Code の統一ツール名（`read` / `edit` / `search` / `execute` / `web` / `agent` / `todos`）を使用。未知のツール名は無視されるだけなので、旧名しか認識しない環境でも定義自体は壊れない
- パイプラインの流れ・ドキュメント受け渡し（`docs/<issue-number>/`）・lessons-learned・リトライ予算は Claude Code 版と同一

## 前提・制約

- 開発対象は SPA（フロントエンド: Vue 3 + TypeScript + Pinia + Vuetify + Vite + pnpm、バックエンド: Java Spring Boot + Flyway + PostgreSQL、開発環境: devcontainer）を既定とする。各エージェントはこのスタック固有の規約・欠陥パターンを持つが、実際のリポジトリの規約が常に優先される
- 設計フェーズでは、フロント/バック横断の変更に対して API 契約（エンドポイント・リクエスト/レスポンス型・エラー応答）の明文化が必須。両サイドはこの契約に対して実装する
- 受け入れ基準には ID（AC-1, AC-2, …）を付与し、テストレポートで基準ごとの検証結果をトレースする
- `pr-publisher` は feature/topic ブランチにのみコミットし、main/master への直接コミットや force-push は行わない
- push・PR作成の前には必ずユーザーへの確認を挟む
- 各サブエージェントは前の対話の記憶を持たない（ステートレス）。フェーズ間の情報は `docs/` 配下のファイルを介して受け渡される
