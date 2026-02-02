# Claude Code Foundation

スペック駆動開発（SDD）のためのClaude Code基盤設定集。

## 概要

個人開発・チーム開発で使える、Claude Codeの設定ファイル一式です。

- **普遍原則**: どのプロジェクトでも守るべき18のルール
- **推奨Skills**: お任せ時の標準的なワークフロー（8つ）
- **壁打ちSkills**: プロジェクトごとに対話で決める（6つ）
- **Agents**: 品質保証・進捗管理のサブエージェント（2つ）
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
│   └── status-updater.md        # 進捗管理エージェント
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
/requirements-definition      # 要件定義フェーズ
/non-functional-requirements  # 非機能要件定義
/scope-definition             # スコープ定義
/design-review                # 設計レビュー
/task-breakdown               # タスク分解
/release-planning             # リリース計画
```

### Agents（サブエージェント）

品質保証や進捗管理をサブエージェントに委譲できます。

| Agent | 役割 | 使用タイミング |
|-------|------|---------------|
| `qa-general` | 成果物の品質チェック | レビュー依頼時、QC実施時 |
| `status-updater` | 進捗状況の追跡・更新 | フェーズ完了時、進捗確認時 |

**呼び出し例:**
```
「品質チェックして」→ qa-general が品質検査を実施
「進捗を確認して」→ status-updater が状況を整理
```

**qa-generalの出力例:**
```
### QC結果: Pass
スコア: 85/100

#### 指摘事項
1. [ドキュメント]: 曖昧表現「適切に」を具体化 (重要度: Medium)

#### 修正推奨
- 「適切にバリデーション」→「入力値を正規表現でチェック」
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
