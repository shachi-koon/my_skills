---
name: git-workflow
description: ブランチ戦略、コミット規約、PR作成の標準ワークフロー。Use when「ブランチを切りたい」「コミットしたい」「PRを作りたい」「Git操作を教えて」と言われた時。
---

# Gitワークフロー

## 概要
一貫したGit運用で履歴の追跡可能性を担保する。

## ブランチ戦略

### ブランチ命名規約
```
<type>/<ticket-number>-<short-description>

例:
feature/123-login-function
bugfix/456-fix-null-pointer
hotfix/789-security-patch
refactor/101-cleanup-auth-module
```

### ブランチ種別
| タイプ | 用途 | 派生元 | マージ先 |
|--------|------|--------|---------|
| `main` | 本番リリース | - | - |
| `develop` | 開発統合 | main | main |
| `feature/*` | 機能開発 | develop | develop |
| `bugfix/*` | バグ修正 | develop | develop |
| `hotfix/*` | 緊急修正 | main | main, develop |
| `refactor/*` | リファクタ | develop | develop |

## コミット規約

### フォーマット
```
<type>: <subject> #<ticket-number>

<body>（任意）

<footer>（任意）
```

### Type一覧
| Type | 用途 |
|------|------|
| `feat` | 新機能 |
| `fix` | バグ修正 |
| `docs` | ドキュメント |
| `style` | フォーマット（コード動作に影響なし） |
| `refactor` | リファクタリング |
| `test` | テスト追加・修正 |
| `chore` | ビルド・ツール関連 |

### 例
```
feat: ログイン機能を追加 #123

- メールアドレスとパスワードによる認証
- JWTトークンの発行
- セッション管理

Refs: #120, #121
```

### コミットのルール
- 1コミット = 1つの論理的変更
- コミットメッセージは日本語OK（チーム規約に従う）
- チケット番号は必須

## PR作成

### PRタイトル
```
[<type>] <subject> #<ticket-number>

例: [feat] ログイン機能を追加 #123
```

### PRテンプレート
```markdown
## 概要
<!-- 変更内容の概要 -->

## 関連チケット
- #XXX

## 変更内容
- [ ] 変更点1
- [ ] 変更点2

## 影響範囲
<!-- 影響を受けるファイル・機能 -->

## テスト
- [ ] 単体テスト追加/更新
- [ ] 動作確認済み

## レビュー観点
<!-- レビュアーに見てほしいポイント -->

## スクリーンショット（該当する場合）
```

## ワークフロー手順

### 1. ブランチ作成
```bash
git checkout develop
git pull origin develop
git checkout -b feature/123-login-function
```

### 2. 作業・コミット
```bash
git add .
git commit -m "feat: ログイン画面のUIを作成 #123"
```

### 3. プッシュ・PR作成
```bash
git push origin feature/123-login-function
# GitHubでPR作成
```

### 4. レビュー・マージ
```bash
# レビュー指摘対応後
git push origin feature/123-login-function
# Squash and Merge推奨
```

## 注意事項
- `main`/`develop`への直接コミット禁止
- Force pushは原則禁止（共有ブランチでは絶対禁止）
- マージ前にdevelopを取り込んでコンフリクト解消
