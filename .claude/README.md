# Claude Code Foundation

スペック駆動開発（SDD）のためのClaude Code基盤設定集。

## 概要

個人開発・チーム開発で使える、Claude Codeの設定ファイル一式です。

- **普遍原則**: どのプロジェクトでも守るべき18のルール
- **推奨Skills**: お任せ時の標準的なワークフロー（8つ）
- **壁打ちSkills**: プロジェクトごとに対話で決める（6つ）
- **Agents**: 品質保証・進捗管理・技術調査・並列リサーチのサブエージェント（5つ）
- **Hooks**: 自動フォーマット、機密ファイル保護

## 思想

### 3レイヤー運用

| レイヤー | 役割 | 変更頻度 |
|---------|------|---------|
| 普遍原則 | どのプロジェクトでも守るルール | ほぼ変えない |
| 推奨 | お任せ時の標準的なやり方 | たまに更新 |
| 壁打ち | プロジェクトごとに決める | 毎回決める |

### 承認ポイント（Human in the Loop）

```
要件定義 → [承認①] → 非機能要件 → [承認②] → スコープ → [承認③]
    → 設計 → [承認④] → タスク分解 → [承認⑤] → 実装・レビュー → [承認⑥] → リリース → [承認⑦]
```

## ファイル構成

```
.
├── CLAUDE.md                    # 普遍原則（18項目）
├── settings.json                # Hooks設定
├── agents/                      # サブエージェント
│   ├── qa-general.md            # 品質保証エージェント
│   ├── status-updater.md        # 進捗管理エージェント
│   ├── web-researcher.md        # 技術調査エージェント
│   ├── chatgpt-parallel-research.md  # ChatGPT並列リサーチ（オプション）
│   └── x-automation-agent.md    # X/Grok自動化（オプション）
│
└── skills/
    ├── 推奨Skills（自動呼び出し）
    │   ├── sdd-workflow/        # 仕様駆動開発
    │   ├── architecture-selection/  # アーキテクチャ選定
    │   ├── code-review/         # コードレビュー
    │   ├── explore-codebase/    # コードベース調査
    │   ├── document-generator/  # ドキュメント生成
    │   ├── git-workflow/        # Gitワークフロー
    │   ├── error-handling-design/   # エラーハンドリング設計
    │   └── test-verification/   # テスト検証
    │
    └── 壁打ちSkills（明示的呼び出し）
        ├── requirements-definition/     # 要件定義
        ├── non-functional-requirements/ # 非機能要件
        ├── scope-definition/            # スコープ定義
        ├── design-review/               # 設計レビュー
        ├── task-breakdown/              # タスク分解
        └── release-planning/            # リリース計画
```

## インストール

### 方法1: ユーザー設定として配置（全プロジェクト共通）

```bash
# リポジトリをクローン
git clone https://github.com/YOUR_USERNAME/claude-code-foundation.git
cd claude-code-foundation

# ファイルを配置
cp CLAUDE.md ~/.claude/CLAUDE.md
cp settings.json ~/.claude/settings.json
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r skills/* ~/.claude/skills/
cp -r agents/* ~/.claude/agents/
```

### 方法2: プロジェクト単位で配置

```bash
# プロジェクトルートで実行
git clone https://github.com/YOUR_USERNAME/claude-code-foundation.git .claude-foundation
cp .claude-foundation/CLAUDE.md ./CLAUDE.md
mkdir -p .claude
cp .claude-foundation/settings.json .claude/settings.json
cp -r .claude-foundation/skills .claude/skills
cp -r .claude-foundation/agents .claude/agents
```

## 使い方

### 推奨Skills（自動で呼び出される）

通常の会話で自動的に適切なSkillが使用されます。

```
「コードレビューして」→ code-review が自動発動
「アーキテクチャを検討したい」→ architecture-selection が自動発動
```

### 壁打ちSkills（明示的に呼び出す）

開発フェーズに応じて明示的に呼び出します。

```
/skill:requirements-definition      # 要件定義フェーズ
/skill:non-functional-requirements  # 非機能要件定義
/skill:scope-definition             # スコープ定義
/skill:design-review                # 設計レビュー
/skill:task-breakdown               # タスク分解
/skill:release-planning             # リリース計画
```

### Agents（サブエージェント）

品質保証や進捗管理をサブエージェントに委譲できます。

| Agent | 役割 | 使用タイミング |
|-------|------|---------------|
| `qa-general` | 成果物の品質チェック | レビュー依頼時、QC実施時 |
| `status-updater` | 進捗状況の追跡・更新 | フェーズ完了時、進捗確認時 |
| `web-researcher` | 技術情報の検証・調査 | API仕様確認、ライブラリ調査、実現可能性確認時 |
| `chatgpt-parallel-research` | ChatGPT並列検索・深掘り調査 | 横断検索、ブレスト、技術調査時（オプション） |
| `x-automation-agent` | X/Grok操作・トレンド収集 | SNS情報収集、トレンド調査時（オプション） |

**呼び出し例:**
```
「品質チェックして」→ qa-general が品質検査を実施
「進捗を確認して」→ status-updater が状況を整理
「〇〇のAPI仕様を調査して」→ web-researcher が技術情報を調査
「横断検索して」→ chatgpt-parallel-research が並列検索（Browser Controller必須）
「Xのトレンドを調べて」→ x-automation-agent がSNS情報収集（Browser Controller必須）
```

**オプションエージェントの前提条件:**
- `chatgpt-parallel-research`: Browser Controller Chrome拡張機能、ChatGPTログイン済み
- `x-automation-agent`: Browser Controller Chrome拡張機能、X（Twitter）ログイン済み

**qa-generalの出力例:**
```
### QC結果: Pass
スコア: 85/100

#### 指摘事項
1. [ドキュメント]: 曖昧表現「適切に」を具体化 (重要度: Medium)

#### 修正推奨
- 「適切にバリデーション」→「入力値を正規表現でチェック」
```

### 評価基準（Evaluation Criteria）

各スキルには `evaluation/evaluation_criteria.md` が用意されており、qa-generalエージェントはこれを参照して品質判定を行います。

**評価の仕組み:**

| 区分 | チェック内容 | 判定 |
|------|-------------|------|
| 構造チェック | 必須セクション・Critical項目の有無 | Pass/Fail |
| 内容チェック | 中身の質（100点満点） | スコアリング |

**最終判定:**
- **Pass**: 全Criticalチェック項目がPass かつ スコア80点以上
- **Conditional Pass**: 全Criticalチェック項目がPass かつ スコア60-79点
- **Fail**: CriticalチェックにFailあり または スコア60点未満

**ファイル配置:**
```
skills/
├── code-review/
│   ├── SKILL.md
│   └── evaluation/
│       └── evaluation_criteria.md  # ← 評価基準
├── sdd-workflow/
│   ├── SKILL.md
│   └── evaluation/
│       └── evaluation_criteria.md
└── ...
```

## 普遍原則（18項目）

| # | カテゴリ | 原則 |
|---|---------|------|
| 1 | 言語 | 曖昧語禁止（適切に/なるべく/よしなに） |
| 2 | 言語 | 用語統一（日本語ベース） |
| 3 | ドキュメント | AC記述ルール（Yes/No判定可能） |
| 4 | ドキュメント | スコープ明記 |
| 5 | ドキュメント | 未決事項管理 |
| 6 | 設計 | 失敗時挙動先行 |
| 7 | 設計 | 非機能先行 |
| 8 | 設計 | 運用設計必須 |
| 9 | タスク | 1タスク=1PR |
| 10 | タスク | タスク紐付け必須 |
| 11 | タスク | 変更経緯の記録 |
| 12 | タスク | 影響範囲の明記 |
| 13 | タスク | コミットメッセージ規約 |
| 14 | 判断 | 決定ログ |
| 15 | リリース | 手順と切り戻しはセット |
| 16 | 粒度 | 3年目基準 |
| 17 | テスト | テスト改変禁止 |
| 18 | コンテキスト | 軽量維持 |

## 数値目標

| 項目 | 目標値 |
|------|--------|
| CLAUDE.md | 60行以下 |
| SKILL.md | 5,000 words以下 |
| 同時有効スキル | 20-50個以下 |
| 有効MCP | 10個以下 |
| スキルトリガー率 | 90%以上 |
| テストカバレッジ | 80%以上 |

## カスタマイズ

### 普遍原則の追加・変更

`CLAUDE.md` を直接編集してください。

### Skillsの追加

```bash
mkdir -p skills/your-new-skill
```

`skills/your-new-skill/SKILL.md` を作成：

```markdown
---
name: your-new-skill
description: スキルの説明。Use when「〇〇したい」と言われた時。
---

# スキル名

## 概要
...
```

### Hooksの追加

`settings.json` の `hooks` セクションを編集してください。

**注意**: 本基盤のHooksは `python3` を使用しています。Hooksを有効にする場合はPython3がインストールされている必要があります。

```bash
# Python3がインストールされているか確認
python3 --version

# Windowsの場合は以下も確認
python --version
```

## 詳細ドキュメント

より詳しい解説は `Docs/` ディレクトリを参照してください。

| ドキュメント | 内容 | 所要時間 |
|-------------|------|---------|
| [クイックリファレンス](./Docs/quick-reference.md) | コマンド一覧、チートシート | 3分 |
| [実用ガイド](./Docs/usage-guide.md) | シナリオ別の使い方、ベストプラクティス | 15-20分 |
| [思想と実現内容](./Docs/claude-code-foundation-overview.md) | 設計思想、18の普遍原則の詳細 | 30-40分 |

## 参考資料

- [Claude Code公式ドキュメント](https://code.claude.com/docs)
- [Extend Claude with skills](https://code.claude.com/docs/en/skills)
- [Manage Claude's memory](https://code.claude.com/docs/en/memory)
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [everything-claude-code](https://github.com/anthropics/everything-claude-code)

## ライセンス

MIT License

## 貢献

Issue、Pull Request歓迎です。
