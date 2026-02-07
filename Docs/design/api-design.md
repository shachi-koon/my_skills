# お問い合わせ管理基盤SaaS API設計書

## 0. 設計選択理由（決定ログ）

| 選択 | 理由 | 再検討条件 |
|------|------|-----------|
| RESTful API | シンプルで広く理解されており、Next.js/NestJSとの親和性が高い。OpenAPI/Swaggerでドキュメント自動生成可能 | GraphQLへの移行検討はフェーズ2以降、複雑なクエリ要件が発生した場合 |
| URLパスベースのバージョニング | APIゲートウェイでのルーティングが容易、クライアント側での切り替えが明示的 | ヘッダーベースへの変更はマイクロサービス分割時に検討 |
| JSON形式 | 業界標準、フロントエンド・バックエンド双方での取り扱いが容易 | バイナリ効率が必要な場合はProtobufを検討 |
| RFC 7807エラーフォーマット | 標準化されたエラー形式で、クライアント側のエラーハンドリングが統一可能 | 独自形式への変更は互換性維持が困難なため非推奨 |
| ULID（ID採番） | ソート可能（時系列順）かつ分散環境で衝突しにくい。UUIDv4より短い | PostgreSQL 17以降でUUIDv7ネイティブ対応時に移行検討 |
| JWT Bearer認証 | ステートレス、Cognitoとの親和性、マイクロサービス間での検証が容易 | セッション管理が必要な場合はRedis併用を検討 |

## 1. システムアーキテクチャ

### 1.1 マイクロサービス構成

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Frontend (Next.js)                           │
│                         MVVM Architecture                           │
└─────────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          BFF Service                                │
│                     (API Gateway + 集約)                            │
│                      NestJS + TypeScript                            │
└─────────────────────────────────────────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Inquiry Service │  │  User Service   │  │ Tenant Service  │
│   (問い合わせ)    │  │   (ユーザー)     │  │   (テナント)     │
│  Clean Arch.    │  │  Clean Arch.    │  │  Clean Arch.    │
└─────────────────┘  └─────────────────┘  └─────────────────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                                ▼
                    ┌─────────────────────┐
                    │ Notification Service │
                    │     (通知・メール)     │
                    │   SQS + SES + Slack  │
                    └─────────────────────┘
```

### 1.2 サービス責務

| サービス | 責務 | 技術スタック |
|---------|------|-------------|
| **BFF Service** | フロントエンド向けAPI集約、認証トークン検証、レスポンス変換 | NestJS, TypeScript |
| **Inquiry Service** | 問い合わせCRUD、ステータス管理、期限計算、添付ファイル管理 | NestJS, TypeScript, PostgreSQL |
| **User Service** | エンドユーザー・管理者管理、認証連携（Cognito） | NestJS, TypeScript, PostgreSQL |
| **Tenant Service** | テナント管理、サービス・カテゴリ管理、設定管理 | NestJS, TypeScript, PostgreSQL |
| **Notification Service** | メール送信、Slack通知、通知キュー管理 | NestJS, TypeScript, SQS, SES |

### 1.3 サービス間通信

| 通信種別 | プロトコル | 用途 |
|---------|----------|------|
| 同期 | HTTP/REST | BFF → 各サービス（リアルタイム必要なケース） |
| 非同期 | SQS | 通知、バッチ処理（結果整合性で十分なケース） |

---

## 2. API設計方針

### 2.1 基本方針
- **RESTful API**: リソース指向設計
- **認証**: JWT Bearer Token（Cognito発行）
- **フォーマット**: JSON
- **バージョニング**: URLパスベース（`/api/v1/...`）
- **エラーレスポンス**: RFC 7807準拠のProblem Details形式

### 2.2 共通仕様

#### リクエストヘッダー
| ヘッダー | 必須 | 説明 |
|---------|------|------|
| Authorization | ○ | `Bearer {jwt_token}` |
| Content-Type | ○ | `application/json` |
| X-Request-ID | - | トレーシング用リクエストID（省略時は自動生成） |

#### ページネーション
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "totalPages": 5,
    "totalCount": 100
  }
}
```

#### エラーレスポンス（RFC 7807）
```json
{
  "type": "https://api.inquiry-platform.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "件名は100文字以内で入力してください",
  "instance": "/api/v1/inquiries",
  "errors": [
    { "field": "subject", "message": "100文字以内で入力してください" }
  ]
}
```

---

## 3. APIエンドポイント一覧

### 3.1 エンドユーザー向けAPI（/api/v1/user/）

| メソッド | パス | 説明 | 対応機能 |
|---------|------|------|----------|
| POST | /inquiries | 問い合わせ投稿 | FR-U001 |
| GET | /inquiries | 問い合わせ履歴一覧 | FR-U002 |
| GET | /inquiries/{id} | 問い合わせ詳細 | FR-U003 |
| POST | /inquiries/{id}/messages | 追加情報送信 | FR-U004 |
| POST | /inquiries/{id}/attachments | 添付ファイルアップロード | FR-U001 |
| GET | /attachments/{id}/download | 添付ファイルダウンロード | FR-U003 |
| GET | /services/{code}/categories | カテゴリ一覧取得 | FR-U001 |

### 3.2 オペレーター/管理者向けAPI（/api/v1/admin/）

| メソッド | パス | 説明 | 対応機能 |
|---------|------|------|----------|
| GET | /inquiries | 問い合わせ一覧（フィルタ対応） | FR-A001 |
| GET | /inquiries/{id} | 問い合わせ詳細 | FR-A002 |
| POST | /inquiries/{id}/messages | 回答送信 | FR-A002 |
| POST | /inquiries/{id}/internal-memos | 内部メモ追加 | FR-A007 |
| PATCH | /inquiries/{id}/assignee | 担当者アサイン | FR-A003 |
| PATCH | /inquiries/{id}/status | ステータス変更 | FR-A004 |
| PATCH | /inquiries/{id}/deadline | 期限変更 | FR-A005 |
| POST | /inquiries/{id}/attachments | 添付ファイルアップロード | FR-A002 |
| GET | /attachments/{id}/download | 添付ファイルダウンロード | FR-A002 |
| GET | /dashboard | ダッシュボードデータ | FR-A006 |
| GET | /dashboard/summary | 対応状況サマリ | FR-A006 |
| GET | /dashboard/alerts | アラート一覧 | FR-A006 |

### 3.3 テナント管理者向けAPI（/api/v1/tenant/）

| メソッド | パス | 説明 | 対応機能 |
|---------|------|------|----------|
| GET | /services | サービス一覧 | FR-T001 |
| POST | /services | サービス登録 | FR-T001 |
| GET | /services/{id} | サービス詳細 | FR-T001 |
| PUT | /services/{id} | サービス更新 | FR-T001 |
| DELETE | /services/{id} | サービス削除 | FR-T001 |
| GET | /services/{id}/categories | カテゴリ一覧 | FR-T002 |
| POST | /services/{id}/categories | カテゴリ登録 | FR-T002 |
| PUT | /services/{id}/categories/{cid} | カテゴリ更新 | FR-T002 |
| DELETE | /services/{id}/categories/{cid} | カテゴリ削除 | FR-T002 |
| GET | /users | ユーザー一覧 | FR-T003 |
| POST | /users/invite | ユーザー招待 | FR-T003 |
| PUT | /users/{id} | ユーザー更新 | FR-T003 |
| DELETE | /users/{id} | ユーザー無効化 | FR-T003 |
| GET | /dashboard | テナント全体ダッシュボード | FR-T004 |
| GET | /settings/business-hours | 営業時間設定取得 | FR-T005 |
| PUT | /settings/business-hours | 営業時間設定更新 | FR-T005 |

### 3.4 システム管理者向けAPI（/api/v1/system/）

| メソッド | パス | 説明 | 対応機能 |
|---------|------|------|----------|
| GET | /tenants | テナント一覧 | FR-S001 |
| POST | /tenants | テナント登録 | FR-S001 |
| GET | /tenants/{id} | テナント詳細 | FR-S001 |
| PUT | /tenants/{id} | テナント更新 | FR-S001 |
| PATCH | /tenants/{id}/status | テナントステータス変更 | FR-S001 |
| GET | /monitoring | システム監視ダッシュボード | FR-S002 |
| POST | /tenants/{id}/export | データエクスポートジョブ開始 | FR-S003 |
| GET | /tenants/{id}/export/{jobId} | エクスポートジョブ状況 | FR-S003 |
| DELETE | /tenants/{id} | テナント解約（論理削除） | FR-S003 |

---

## 4. API詳細仕様

### 4.1 問い合わせ投稿（POST /api/v1/user/inquiries）

#### リクエスト
```json
{
  "categoryId": "cat_001",
  "subject": "商品の返品について",
  "body": "先日購入した商品を返品したいのですが..."
}
```

#### レスポンス（201 Created）
```json
{
  "id": "inq_abc123",
  "inquiryNumber": "INQ-20260204-0001",
  "serviceId": "svc_001",
  "categoryId": "cat_001",
  "subject": "商品の返品について",
  "status": "pending",
  "createdAt": "2026-02-04T10:00:00Z"
}
```

#### 添付ファイル
添付ファイルは別エンドポイント `POST /api/v1/user/inquiries/{id}/attachments` で multipart/form-data でアップロード

---

### 4.2 問い合わせ一覧（GET /api/v1/admin/inquiries）

#### クエリパラメータ
| パラメータ | 型 | 説明 |
|-----------|------|------|
| serviceId | string | サービスIDでフィルタ |
| status | string | ステータスでフィルタ（pending/in_progress/answered/completed/on_hold） |
| categoryId | string | カテゴリIDでフィルタ |
| assignedTo | string | 担当者IDでフィルタ（`unassigned`で未アサイン） |
| deadline | string | 期限でフィルタ（overdue/today/soon/within） |
| q | string | 検索クエリ（問い合わせ番号、件名、本文） |
| page | number | ページ番号（デフォルト: 1） |
| perPage | number | 1ページあたり件数（デフォルト: 20、最大: 100） |
| sort | string | ソート項目（createdAt/updatedAt/deadline） |
| order | string | ソート順（asc/desc） |

#### レスポンス（200 OK）
```json
{
  "data": [
    {
      "id": "inq_abc123",
      "inquiryNumber": "INQ-20260204-0001",
      "service": { "id": "svc_001", "name": "ECサイト" },
      "category": { "id": "cat_001", "name": "返品・交換" },
      "subject": "商品の返品について",
      "status": "pending",
      "deadline": "2026-02-06T17:00:00Z",
      "deadlineStatus": "within",
      "assignedTo": null,
      "endUser": { "id": "usr_001", "name": "山田太郎", "email": "yamada@example.com" },
      "createdAt": "2026-02-04T10:00:00Z",
      "updatedAt": "2026-02-04T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "perPage": 20,
    "totalPages": 5,
    "totalCount": 100
  }
}
```

---

### 4.3 ダッシュボード（GET /api/v1/admin/dashboard）

#### レスポンス（200 OK）
```json
{
  "summary": {
    "pending": 15,
    "inProgress": 23,
    "answered": 8,
    "onHold": 3
  },
  "alerts": {
    "overdue": 5,
    "dueSoon": 12,
    "longPending": 7,
    "unassigned": 10
  },
  "metrics": {
    "avgFirstResponseTimeHours": 4.5,
    "avgResolutionTimeHours": 24.2
  },
  "byService": [
    { "serviceId": "svc_001", "name": "ECサイト", "count": 45 },
    { "serviceId": "svc_002", "name": "会員アプリ", "count": 30 }
  ],
  "byCategory": [
    { "categoryId": "cat_001", "name": "返品・交換", "count": 20 },
    { "categoryId": "cat_002", "name": "配送", "count": 15 }
  ],
  "period": {
    "start": "2026-02-01T00:00:00Z",
    "end": "2026-02-04T23:59:59Z"
  }
}
```

---

### 4.4 ステータス変更（PATCH /api/v1/admin/inquiries/{id}/status）

#### リクエスト
```json
{
  "status": "in_progress"
}
```

#### 許可されるステータス遷移
| 現在 | 変更先 | 条件 |
|------|--------|------|
| pending | in_progress | 担当者アサイン時（自動）or 手動 |
| in_progress | answered | 回答送信時（自動）or 手動 |
| in_progress | on_hold | 手動のみ |
| answered | in_progress | ユーザー追加情報送信時（自動）or 手動 |
| answered | completed | 手動 or 自動完了フロー |
| completed | in_progress | 手動（通知あり） |
| completed | on_hold | 手動（通知なし） |
| completed | pending | 手動（通知なし） |
| on_hold | in_progress | 手動 |

#### レスポンス（200 OK）
```json
{
  "id": "inq_abc123",
  "status": "in_progress",
  "updatedAt": "2026-02-04T11:00:00Z"
}
```

---

### 4.5 期限変更（PATCH /api/v1/admin/inquiries/{id}/deadline）

#### リクエスト
```json
{
  "deadline": "2026-02-10T17:00:00Z"
}
```

#### レスポンス（200 OK）
```json
{
  "id": "inq_abc123",
  "deadline": "2026-02-10T17:00:00Z",
  "deadlineStatus": "within",
  "updatedAt": "2026-02-04T12:00:00Z"
}
```

---

### 4.6 添付ファイルアップロード（POST /api/v1/admin/inquiries/{id}/attachments）

#### リクエスト
- Content-Type: `multipart/form-data`
- フィールド: `file` （1件ずつアップロード）

#### バリデーション
- 最大ファイルサイズ: 10MB
- 許可フォーマット: jpg, png, gif, pdf, doc, docx, xls, xlsx
- 1問い合わせあたり最大5件

#### レスポンス（201 Created）
```json
{
  "id": "att_xyz789",
  "filename": "receipt.pdf",
  "size": 524288,
  "contentType": "application/pdf",
  "virusScanStatus": "pending",
  "createdAt": "2026-02-04T12:00:00Z"
}
```

#### ウイルススキャン
- アップロード後、非同期でスキャン実行
- スキャン結果: `pending` → `clean` or `infected`
- `infected`の場合: ファイル削除、管理者にSlack通知、監査ログ記録

---

### 4.7 添付ファイルダウンロード（GET /api/v1/admin/attachments/{id}/download）

#### レスポンス（302 Found）
- S3署名付きURLへリダイレクト（有効期限: 1時間）

#### エラーケース
- ウイルス検出済み: 403 Forbidden
- ファイル未存在: 404 Not Found

---

## 5. 認証・認可

### 5.1 認証フロー（エンドユーザー）
```
1. サービス → Cognito（Federation）→ IdP（SAML/OIDC）
2. IdP認証成功 → SAMLアサーション/OIDCトークン → Cognito
3. Cognito → JWTトークン発行（cognito:groups にサービスID含む）
4. JWTトークンでAPIアクセス
```

### 5.2 認証フロー（管理者/オペレーター）
```
1. 管理者ログイン画面 → Cognito（User Pools）
2. メール/パスワード + MFA（TOTP）
3. Cognito → JWTトークン発行（custom:roleにロール、custom:tenant_idにテナントID）
4. JWTトークンでAPIアクセス
```

### 5.3 認可チェック

各APIエンドポイントでJWTクレームから以下を検証：
- `custom:role`: ロール（operator/service_admin/tenant_admin/system_admin）
- `custom:tenant_id`: テナントID
- `custom:service_ids`: 担当サービスIDリスト（カンマ区切り）
- `cognito:groups`: サービスID（エンドユーザー用）

---

## 6. 非機能仕様

### 6.1 レート制限
| 対象 | 制限 |
|------|------|
| エンドユーザーAPI | 100リクエスト/分/ユーザー |
| 管理者API | 300リクエスト/分/ユーザー |
| システム管理者API | 1000リクエスト/分/ユーザー |

超過時は `429 Too Many Requests` を返却

### 6.2 タイムアウト
| 対象 | タイムアウト |
|------|-------------|
| API全般 | 30秒 |
| ファイルアップロード | 60秒 |
| エクスポートジョブ開始 | 30秒（非同期処理） |

### 6.3 キャッシュ
| 対象 | TTL | 条件 |
|------|-----|------|
| ダッシュボードデータ | 60秒 | Redisキャッシュ |
| カテゴリ一覧 | 300秒 | Redisキャッシュ |
| サービス一覧 | 300秒 | Redisキャッシュ |

---

## 7. 改訂履歴

| 日付 | バージョン | 変更内容 | 作成者 |
|------|-----------|---------|--------|
| 2026-02-04 | 0.1 | 初版作成（エンドポイント一覧、詳細仕様） | Claude |
| 2026-02-04 | 0.2 | 設計選択理由追加、期限変更API追加、添付ファイルAPI追加 | Claude |
| 2026-02-05 | 0.3 | マイクロサービス構成追加、コーディングルール作成 | Claude |
| 2026-02-05 | 0.4 | エンドユーザー向け添付ファイルAPI追加 | Claude |
| 2026-02-05 | 0.5 | エンドユーザー向け添付ファイルダウンロードAPI追加 | Claude |
| 2026-02-07 | 0.6 | JSONフィールドをcamelCaseに統一（coding-rules.md準拠） | Claude |
| 2026-02-07 | 0.7 | クエリパラメータをcamelCaseに統一 | Claude |
