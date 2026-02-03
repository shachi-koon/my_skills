# Claude Code Foundation - クイックリファレンス

## 開発フロー（7つの承認ポイント）

```
① 要件定義 → [承認] → ② 非機能要件 → [承認] → ③ スコープ → [承認]
    ↓
④ 設計 → [承認] → ⑤ タスク分解 → [承認] → ⑥ 実装・レビュー → [承認] → ⑦ リリース → [承認]
```

---

## スキル呼び出しコマンド

### 壁打ちスキル（明示的呼び出し）

| コマンド | 用途 | 承認ポイント |
|---------|------|-------------|
| `/skill:requirements-definition` | 要件定義 | ① |
| `/skill:non-functional-requirements` | 非機能要件 | ② |
| `/skill:scope-definition` | スコープ定義 | ③ |
| `/skill:design-review` | 設計レビュー | ④ |
| `/skill:task-breakdown` | タスク分解 | ⑤ |
| `/skill:release-planning` | リリース計画 | ⑦ |

### 推奨スキル（自動発動）

| トリガー例 | スキル | 用途 |
|-----------|--------|------|
| 「開発を始めたい」 | `sdd-workflow` | 仕様駆動開発 |
| 「コードレビューして」 | `code-review` | コードレビュー |
| 「コードベースを調査して」 | `explore-codebase` | コード調査 |
| 「エラー処理を設計して」 | `error-handling-design` | エラーハンドリング |
| 「アーキテクチャを検討」 | `architecture-selection` | 技術選定 |
| 「ドキュメント生成して」 | `document-generator` | ドキュメント作成 |
| 「ブランチ作成して」 | `git-workflow` | Git操作 |
| 「テスト検証して」 | `test-verification` | テスト確認 |

---

## 18の普遍原則（抜粋）

### 必ず守る5つのルール

| # | 原則 | 具体例 |
|---|------|--------|
| 1 | **曖昧語禁止** | ❌「適切に」 → ✅「タイムアウト3秒、3回リトライ」 |
| 3 | **AC記述ルール** | ❌「ログインできる」 → ✅「正しい認証情報で200が返る（API確認）」 |
| 6 | **失敗時挙動先行** | 成功パスより先に、タイムアウト値/リトライ回数/エラーログを決める |
| 9 | **1タスク=1PR** | 各タスクに「Done条件」「確認方法」を記載 |
| 17 | **テスト改変禁止** | テストを通すための実装改修は絶対禁止 |

### スコープ定義（原則4）

```markdown
## スコープ内
- メールログイン（FR-001）
- パスワードリセット（FR-002）

## スコープ外
- OAuth連携 → v2.0で実装
- 2要素認証 → v3.0で検討
```

### 受入条件（AC）の書き方（原則3）

```markdown
- [ ] 正しい認証情報でログインできる - 確認方法: API /auth/login に POST して200が返る
- [ ] 間違った認証情報で401エラーが返る - 確認方法: API /auth/login に POST して401が返る
- [ ] 5回連続失敗でアカウントロック - 確認方法: ログに "account_locked" が記録される
```

**ポイント**: Yes/No判定可能 + 確認方法（画面/API/ログ）をセット

---

## 典型的な開発フロー

### パターン1: 新機能開発（フルフロー）

```bash
# Phase 1: 計画
/skill:requirements-definition       # ① 要件定義 → 承認
/skill:non-functional-requirements   # ② 非機能要件（性能、セキュリティ） → 承認
/skill:scope-definition              # ③ スコープ内/外を明確化 → 承認

# Phase 2: 設計
/skill:design-review                 # ④ アーキテクチャ、エラーハンドリング → 承認
/skill:task-breakdown                # ⑤ 1タスク=1PRに分解 → 承認

# Phase 3: 実装
「TASK-001から実装して」              # sdd-workflowが自動発動
「コードレビューして」                 # code-reviewが自動発動
「テストを検証して」                   # test-verificationが自動発動
                                     # ⑥ 実装・レビュー完了 → 承認

# Phase 4: リリース
/skill:release-planning              # ⑦ リリース手順 + 切り戻し手順 → 承認
```

### パターン2: バグ修正（軽量フロー）

```bash
# 小規模な修正は承認ポイント省略可
「src/auth/login.ts の typo を修正して」
「コードレビューして」
「テスト通るか確認して」
```

### パターン3: 既存コード調査

```bash
# 調査・理解フェーズ
「認証関連のコードを調査して」         # explore-codebaseが自動発動
「アーキテクチャの選定理由を教えて」   # architecture-selectionが自動発動
「READMEを生成して」                  # document-generatorが自動発動
```

---

## よく使うコマンド

### コードレビュー
```
「src/auth/login.ts をコードレビューして」
「セキュリティの観点でレビューして」
```

### エラーハンドリング設計
```
「エラーハンドリングを設計して」
「タイムアウトとリトライの戦略を決めて」
```

### タスク分解
```
/skill:task-breakdown
「認証機能を1タスク=1PR単位で分解して」
```

### アーキテクチャ選定
```
「アーキテクチャを検討したい。選択肢を提示して」
「REST vs GraphQL どちらがいいか比較して」
```

---

## 禁止事項

### ❌ 絶対にやってはいけないこと

1. **曖昧語を使う**
   - 「適切に」「なるべく」「よしなに」

2. **承認なしで次工程に進む**（新機能開発時）
   - 要件定義 → いきなり実装

3. **テストを通すために実装を変更**
   - テストが失敗 → 実装を曲げる

4. **スコープ外を明記しない**
   - 「スコープ内」だけ書いて、「スコープ外」を書かない

5. **失敗時の挙動を後回し**
   - 正常系だけ実装 → 後からエラーハンドリング追加

---

## セーフティネット（Hooks）

### 自動でブロックされるもの

#### 機密ファイル
- `.env`, `.env.local`, `.env.production`
- `/secrets/**`, `/credentials/**`, `.secret`

#### 危険なコマンド
- `rm -rf /`, `rm -rf ~`, `rm -rf .`
- `> /dev/sda`
- Fork Bomb

### 自動で実行されるもの

#### ファイル保存後
- **自動フォーマット**: Prettier（.ts, .tsx, .js, .jsx, .json, .md）
- **型チェック**: TypeScript（.ts, .tsx）

**注意**: HooksはPython3で実装されています。Python3がインストールされていない環境では動作しません。

---

## エージェント（Agents）

### 3つのサブエージェント

| Agent | 役割 | 呼び出し例 |
|-------|------|----------|
| `qa-general` | 成果物の品質チェック | 「品質チェックして」 |
| `status-updater` | 進捗状況の追跡・更新 | 「進捗を確認して」 |
| `web-researcher` | 技術情報の検証・調査 | 「〇〇のAPI仕様を調査して」 |

### 評価基準（Evaluation Criteria）

各スキルには `evaluation/evaluation_criteria.md` が用意されています。

**評価の仕組み:**

| 区分 | 判定 |
|------|------|
| 構造チェック | Pass/Fail（必須項目の有無） |
| 内容チェック | 100点満点のスコアリング |

**最終判定:**
- **Pass**: 全Critical項目Pass + 80点以上
- **Conditional Pass**: 全Critical項目Pass + 60-79点
- **Fail**: Critical項目Failあり または 60点未満

---

## 数値目標

| 項目 | 目標値 |
|------|--------|
| CLAUDE.md | 60行以下 |
| SKILL.md | 5,000 words以下 |
| 同時有効スキル | 20-50個以下 |
| スキルトリガー率 | 90%以上 |
| テストカバレッジ | 80%以上 |

---

## トラブルシューティング

### スキルが自動で発動しない
```bash
# 明示的に呼び出す
/skill:code-review でsrc/auth/login.tsをレビューして
```

### Hooksが動作しない
```bash
# Prettierをインストール
npm install --save-dev prettier

# settings.jsonを確認
cat .claude/settings.json
```

### 機密ファイルがブロックされない
```bash
# permissionsを確認
jq '.permissions' .claude/settings.json
```

---

## チートシート: 受入条件（AC）の書き方

### テンプレート
```markdown
- [ ] [Yes/No判定可能な条件] - 確認方法: [画面/API/ログ/テスト]
```

### 良い例
```markdown
✅ 正しい認証情報でログインできる - 確認方法: POST /auth/login で200が返る
✅ レスポンスタイムが500ms以内 - 確認方法: 負荷テストツールで計測
✅ エラー時にログが記録される - 確認方法: ログファイルに "ERROR" が出力される
```

### 悪い例
```markdown
❌ ログインできること（確認方法が不明）
❌ 適切にエラーハンドリングされること（曖昧）
❌ 高速に動作すること（数値がない）
```

---

## チートシート: タスク分解

### テンプレート
```markdown
- [ ] TASK-XXX: [タスク名] #[チケット番号]
  - **Done条件**: [完了の定義]
  - **確認方法**: [どうやって確認するか]
```

### 良い例
```markdown
✅ TASK-001: POST /auth/login API実装 #123
  - **Done条件**: ログインしてJWTトークンが取得できる
  - **確認方法**: Postmanで200とトークンが返る

✅ TASK-002: アカウントロック機能実装 #124
  - **Done条件**: 5回連続失敗でロックされる
  - **確認方法**: ユニットテストが通る
```

### 悪い例
```markdown
❌ TASK-001: 認証機能実装（大きすぎる、Done条件不明）
❌ TASK-002: バグ修正（何のバグか不明）
```

---

## スキルの分類と配置

### スキルの3分類

| 分類 | 呼び出し方 | 例 |
|------|-----------|-----|
| **普遍原則** | Always読み込み | CLAUDE.md（18項目） |
| **推奨スキル** | 自動発動 | code-review, sdd-workflow |
| **壁打ちスキル** | 明示的呼び出し | requirements-definition |

### ファイル配置の2層構造（公式）

| レベル | パス | 用途 | Git管理 |
|--------|------|------|---------|
| **ユーザー** | `~/.claude/` | 全プロジェクト共通 | しない |
| **プロジェクト** | `./.claude/` | プロジェクト固有 | する |
| **ローカル** | `./.claude/*.local.*` | 個人用上書き | しない |

**優先順位**: プロジェクト > ユーザー

---

## 推奨ファイル配置

### ユーザーレベル（個人基盤）
```
~/.claude/                      # 全プロジェクト共通
├── CLAUDE.md                   # 普遍原則（18項目）
├── settings.json               # 基本Hooks・Permissions
└── skills/
    ├── sdd-workflow/           # 推奨スキル（8個）
    ├── code-review/
    ├── explore-codebase/
    ├── architecture-selection/
    ├── document-generator/
    ├── git-workflow/
    ├── error-handling-design/
    ├── test-verification/
    ├── requirements-definition/ # 壁打ちスキル（6個）
    ├── non-functional-requirements/
    ├── scope-definition/
    ├── design-review/
    ├── task-breakdown/
    └── release-planning/
```

### プロジェクトレベル（プロジェクト固有）
```
project/
├── .claude/                    # プロジェクト固有設定
│   ├── CLAUDE.md               # プロジェクト固有ルール（Git管理）
│   ├── CLAUDE.local.md         # 個人用上書き（gitignore）
│   ├── settings.json           # プロジェクト固有Hooks（Git管理）
│   ├── settings.local.json     # 個人用上書き（gitignore）
│   └── skills/                 # プロジェクト固有スキル
│       └── project-workflow/
│
└── Docs/                       # このリポジトリの解説（任意）
    ├── claude-code-foundation-overview.md
    ├── usage-guide.md
    └── quick-reference.md
```

---

## セットアップ手順

### 基盤のインストール

```bash
# リポジトリをクローン
git clone https://github.com/YOUR_USERNAME/claude-code-foundation.git
cd claude-code-foundation

# ファイルをユーザーレベルに配置
cp CLAUDE.md ~/.claude/CLAUDE.md
cp settings.json ~/.claude/settings.json
mkdir -p ~/.claude/skills ~/.claude/agents
cp -r skills/* ~/.claude/skills/
cp -r agents/* ~/.claude/agents/
```

**注意**: Hooksの実行にはPython3が必要です。

---

## 開発の開始手順

```bash
# 1. 要件定義から始める
/skill:requirements-definition

# 2. 非機能要件を決める
/skill:non-functional-requirements

# 3. スコープを明確化
/skill:scope-definition

# 4. 設計をレビュー
/skill:design-review

# 5. タスクに分解
/skill:task-breakdown

# 6. 実装開始
「TASK-001から実装して」

# 7. リリース計画
/skill:release-planning
```

---

## 参考資料

- [詳細解説](./claude-code-foundation-overview.md) - 思想と実現内容
- [実用ガイド](./usage-guide.md) - シナリオ別の使い方
- [.claude/README.md](../.claude/README.md) - 公式README
- [.claude/CLAUDE.md](../.claude/CLAUDE.md) - 18の普遍原則

---

**Claude Code Foundation** - スペック駆動開発を実現するAI協調開発基盤
