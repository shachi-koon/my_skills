# Claude Code Foundation

スペック駆動開発（SDD）のためのClaude Code基盤設定集。

## 概要

個人開発・チーム開発で使える、Claude Codeの設定ファイル一式です。

- **普遍原則**: どのプロジェクトでも守るべき18のルール
- **推奨Skills**: お任せ時の標準的なワークフロー（8つ）
- **壁打ちSkills**: プロジェクトごとに対話で決める（6つ）
- **Hooks**: 自動フォーマット、機密ファイル保護

## クイックスタート

### 1. 基盤のインストール

```bash
# この基盤をユーザーレベルに配置
cp -r .claude/* ~/.claude/
```

### 2. 使い方

詳しい使い方は [Docs/README.md](Docs/README.md) を参照してください。

- [クイックリファレンス](Docs/quick-reference.md) - 1分で確認
- [実用ガイド](Docs/usage-guide.md) - 具体的な使い方
- [思想と実現内容](Docs/claude-code-foundation-overview.md) - 詳細解説

## ディレクトリ構成

```
.
├── README.md                    # このファイル
├── .claude/                     # 基盤ファイル（~/.claude/にコピー）
│   ├── CLAUDE.md                # 18の普遍原則
│   ├── settings.json            # Hooks・Permissions
│   ├── README.md                # 基盤の公式README
│   ├── LICENSE                  # ライセンス
│   └── skills/                  # 14個のスキル
│       ├── sdd-workflow/        # 推奨スキル（8個）
│       ├── code-review/
│       ├── explore-codebase/
│       ├── architecture-selection/
│       ├── document-generator/
│       ├── git-workflow/
│       ├── error-handling-design/
│       ├── test-verification/
│       ├── requirements-definition/  # 壁打ちスキル（6個）
│       ├── non-functional-requirements/
│       ├── scope-definition/
│       ├── design-review/
│       ├── task-breakdown/
│       └── release-planning/
│
└── Docs/                        # 解説ドキュメント
    ├── README.md                # ドキュメントガイド
    ├── quick-reference.md       # クイックリファレンス
    ├── usage-guide.md           # 実用ガイド
    └── claude-code-foundation-overview.md  # 詳細解説
```

## 特徴

### スペック駆動開発（SDD）
- 仕様→実装→テストの順で開発
- 失敗時挙動先行、非機能先行、運用設計必須

### 7つの承認ポイント（Human in the Loop）
- AIに丸投げせず、人間が意思決定に関与
- 要件定義→非機能要件→スコープ→設計→タスク分解→実装・レビュー→リリース

### セーフティネット
- 機密ファイルへのアクセスをブロック
- 危険なコマンドの実行を防止
- 自動フォーマット・型チェック

## ライセンス

MIT License

## 貢献

Issue、Pull Request歓迎です。
