# お問い合わせ管理基盤SaaS DB設計書

## 0. 設計選択理由（決定ログ）

| 選択 | 理由 | 再検討条件 |
|------|------|-----------|
| Aurora PostgreSQL Serverless v2 | オートスケーリング、マルチAZ標準、RLSサポート、AWS環境との親和性 | コスト最適化のためRDSスタンダードへの変更はトラフィック安定後に検討 |
| Row Level Security（RLS） | アプリケーション層でのテナント分離漏れを防止、SQLインジェクション対策の二重防御 | 性能問題発生時はスキーマ分離を検討（マイグレーション工数大） |
| ULID形式ID | 時系列ソート可能、分散環境で衝突しにくい、URLに含めても安全 | PostgreSQL 17以降でUUIDv7移行を検討 |
| JSONB型（設定カラム） | スキーマ柔軟性、部分更新可能、GINインデックスでの検索可能 | 構造固定が必要な場合は正規化テーブルに分離 |
| スネークケース命名 | PostgreSQLの標準規約、大文字小文字の混乱を回避 | 変更不可（既存コードへの影響大） |
| 月次パーティション（audit_logs） | データ量増加に対応、古いデータの効率的な削除/アーカイブが可能 | inquiries/messagesは将来的にテナントIDパーティションを検討 |

## 1. 設計方針

### 1.1 基本方針
- **RDBMS**: Amazon Aurora PostgreSQL Serverless v2
- **マルチテナント**: テナントIDによる論理分離（Row Level Security使用）
- **文字コード**: UTF-8
- **タイムゾーン**: UTC（アプリケーション層でタイムゾーン変換）
- **ID採番**: ULID形式（ソート可能なユニークID）

### 1.2 命名規則
- テーブル名: スネークケース、複数形（例: `inquiries`）
- カラム名: スネークケース（例: `created_at`）
- インデックス名: `idx_{テーブル名}_{カラム名}`
- 外部キー名: `fk_{テーブル名}_{参照テーブル名}`

---

## 2. ER図

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   tenants   │────<│  services   │────<│ categories  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │
       │                   │
       ▼                   ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ admin_users │────<│  inquiries  │────<│  messages   │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       │                   │                   ▼
       ▼                   │            ┌─────────────┐
┌─────────────┐            └───────────>│ attachments │
│service_roles│                         └─────────────┘
└─────────────┘

┌─────────────┐     ┌─────────────┐
│  end_users  │     │ audit_logs  │
└─────────────┘     └─────────────┘
```

---

## 3. テーブル定義

### 3.1 tenants（テナント）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| name | VARCHAR(100) | NO | - | テナント名 |
| plan | VARCHAR(50) | NO | 'standard' | 契約プラン |
| status | VARCHAR(20) | NO | 'active' | active/inactive/pending_deletion |
| business_hours | JSONB | YES | NULL | 営業時間設定 |
| additional_holidays | JSONB | YES | NULL | 追加休日（日付配列） |
| timezone | VARCHAR(50) | NO | 'Asia/Tokyo' | タイムゾーン |
| contract_start_date | DATE | NO | - | 契約開始日 |
| deletion_scheduled_at | TIMESTAMP | YES | NULL | 削除予定日時 |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_tenants_status (status)

**business_hours JSONBスキーマ例**
```json
{
  "weekdays": [1, 2, 3, 4, 5],
  "start_time": "09:00",
  "end_time": "18:00",
  "use_national_holidays": true
}
```

---

### 3.2 services（サービス）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID |
| name | VARCHAR(100) | NO | - | サービス名 |
| service_code | VARCHAR(50) | NO | - | サービスコード（テナント内ユニーク） |
| sso_config | JSONB | YES | NULL | SSO設定（IdPタイプ、メタデータURL等） |
| return_url | VARCHAR(500) | YES | NULL | 戻りURL |
| cognito_group_name | VARCHAR(100) | YES | NULL | Cognitoグループ名 |
| status | VARCHAR(20) | NO | 'active' | active/inactive |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- UNIQUE idx_services_tenant_code (tenant_id, service_code)
- idx_services_tenant_id (tenant_id)
- fk_services_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)

**sso_config JSONBスキーマ例**
```json
{
  "idp_type": "saml",
  "idp_identifier": "company-idp",
  "metadata_url": "https://idp.example.com/metadata.xml",
  "attribute_mapping": {
    "email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
    "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name"
  }
}
```

---

### 3.3 categories（カテゴリ）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| service_id | VARCHAR(26) | NO | - | サービスID |
| name | VARCHAR(100) | NO | - | カテゴリ名 |
| default_deadline_hours | INTEGER | NO | 72 | デフォルト対応期限（時間） |
| sort_order | INTEGER | NO | 0 | 表示順 |
| status | VARCHAR(20) | NO | 'active' | active/inactive |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_categories_service_id (service_id)
- idx_categories_service_order (service_id, sort_order)
- fk_categories_service FOREIGN KEY (service_id) REFERENCES services(id)

---

### 3.4 admin_users（管理者ユーザー）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID |
| cognito_sub | VARCHAR(100) | NO | - | Cognito Subject ID |
| email | VARCHAR(255) | NO | - | メールアドレス |
| name | VARCHAR(100) | NO | - | 氏名 |
| role | VARCHAR(30) | NO | - | operator/service_admin/tenant_admin/system_admin |
| status | VARCHAR(20) | NO | 'active' | active/inactive/pending |
| notification_setting | JSONB | YES | NULL | 通知設定 |
| last_login_at | TIMESTAMP | YES | NULL | 最終ログイン日時 |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- UNIQUE idx_admin_users_cognito_sub (cognito_sub)
- UNIQUE idx_admin_users_email (email)
- idx_admin_users_tenant_id (tenant_id)
- fk_admin_users_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)

**notification_setting JSONBスキーマ例**
```json
{
  "new_inquiry": true,
  "assigned": true,
  "user_reply": true,
  "deadline_warning": true,
  "digest_mode": "instant"
}
```

---

### 3.5 service_roles（サービス権限）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| admin_user_id | VARCHAR(26) | NO | - | 管理者ユーザーID |
| service_id | VARCHAR(26) | NO | - | サービスID |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |

**インデックス**
- PRIMARY KEY (id)
- UNIQUE idx_service_roles_user_service (admin_user_id, service_id)
- idx_service_roles_admin_user_id (admin_user_id)
- idx_service_roles_service_id (service_id)
- fk_service_roles_admin_user FOREIGN KEY (admin_user_id) REFERENCES admin_users(id)
- fk_service_roles_service FOREIGN KEY (service_id) REFERENCES services(id)

---

### 3.6 end_users（エンドユーザー）

**設計方針**: エンドユーザーとサービスの多対多関係は、DBではなくCognito Groupsで管理する。理由は以下の通り：
1. 認証・認可の一元管理（Cognitoを信頼できる唯一の情報源とする）
2. サービス権限変更時のDB同期処理が不要
3. JWTトークンにグループ情報が含まれるため、APIアクセス時のDB参照が不要

サービス単位でのエンドユーザー一覧が必要な場合は、inquiriesテーブルのservice_idとend_user_idを結合して取得する。

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID |
| cognito_sub | VARCHAR(100) | NO | - | Cognito Subject ID |
| idp_user_id | VARCHAR(255) | YES | NULL | IdPでのユーザーID |
| email | VARCHAR(255) | NO | - | メールアドレス |
| name | VARCHAR(100) | YES | NULL | 氏名 |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- UNIQUE idx_end_users_cognito_sub (cognito_sub)
- idx_end_users_tenant_id (tenant_id)
- idx_end_users_email (tenant_id, email)
- fk_end_users_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)

---

### 3.7 inquiries（問い合わせ）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID（RLS用） |
| inquiry_number | VARCHAR(30) | NO | - | 問い合わせ番号（INQ-YYYYMMDD-NNNN） |
| service_id | VARCHAR(26) | NO | - | サービスID |
| category_id | VARCHAR(26) | NO | - | カテゴリID |
| end_user_id | VARCHAR(26) | NO | - | エンドユーザーID |
| subject | VARCHAR(100) | NO | - | 件名 |
| status | VARCHAR(20) | NO | 'pending' | pending/in_progress/answered/completed/on_hold |
| deadline | TIMESTAMP | NO | - | 対応期限 |
| assigned_to | VARCHAR(26) | YES | NULL | 担当者ID |
| answered_at | TIMESTAMP | YES | NULL | 回答日時（最初の回答） |
| completed_at | TIMESTAMP | YES | NULL | 完了日時 |
| auto_complete_reminder_sent_at | TIMESTAMP | YES | NULL | 自動完了確認メール送信日時 |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |
| updated_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 更新日時 |

**インデックス**
- PRIMARY KEY (id)
- UNIQUE idx_inquiries_number (inquiry_number)
- idx_inquiries_tenant_id (tenant_id)
- idx_inquiries_service_id (service_id)
- idx_inquiries_status (tenant_id, status)
- idx_inquiries_deadline (tenant_id, deadline) WHERE status NOT IN ('completed')
- idx_inquiries_assigned_to (assigned_to)
- idx_inquiries_end_user_id (end_user_id)
- idx_inquiries_created_at (tenant_id, created_at DESC)
- fk_inquiries_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)
- fk_inquiries_service FOREIGN KEY (service_id) REFERENCES services(id)
- fk_inquiries_category FOREIGN KEY (category_id) REFERENCES categories(id)
- fk_inquiries_end_user FOREIGN KEY (end_user_id) REFERENCES end_users(id)
- fk_inquiries_assigned_to FOREIGN KEY (assigned_to) REFERENCES admin_users(id)

---

### 3.8 messages（メッセージ）

**設計方針**: 問い合わせ投稿時の本文は、messagesテーブルの最初のレコードとして格納する。inquiriesテーブルには本文カラムを持たない。これにより、チャット形式の履歴表示が統一的に扱える。

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| inquiry_id | VARCHAR(26) | NO | - | 問い合わせID |
| sender_type | VARCHAR(20) | NO | - | end_user/operator |
| sender_id | VARCHAR(26) | NO | - | 送信者ID（end_user_id or admin_user_id） |
| body | TEXT | NO | - | 本文（初回投稿の本文もここに格納） |
| is_internal_memo | BOOLEAN | NO | FALSE | 内部メモフラグ |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_messages_inquiry_id (inquiry_id)
- idx_messages_inquiry_created (inquiry_id, created_at)
- fk_messages_inquiry FOREIGN KEY (inquiry_id) REFERENCES inquiries(id)

---

### 3.9 attachments（添付ファイル）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| message_id | VARCHAR(26) | NO | - | メッセージID |
| filename | VARCHAR(255) | NO | - | ファイル名 |
| s3_key | VARCHAR(500) | NO | - | S3オブジェクトキー |
| size | INTEGER | NO | - | ファイルサイズ（バイト） |
| content_type | VARCHAR(100) | NO | - | MIMEタイプ |
| virus_scan_status | VARCHAR(20) | NO | 'pending' | pending/clean/infected |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_attachments_message_id (message_id)
- fk_attachments_message FOREIGN KEY (message_id) REFERENCES messages(id)

---

### 3.10 audit_logs（監査ログ）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID |
| actor_type | VARCHAR(20) | NO | - | end_user/admin_user/system |
| actor_id | VARCHAR(26) | YES | NULL | 操作者ID |
| action | VARCHAR(50) | NO | - | 操作種別 |
| resource_type | VARCHAR(50) | NO | - | 対象リソース種別 |
| resource_id | VARCHAR(26) | NO | - | 対象リソースID |
| changes | JSONB | YES | NULL | 変更前後の値 |
| ip_address | INET | YES | NULL | IPアドレス |
| user_agent | TEXT | YES | NULL | User-Agent |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_audit_logs_tenant_id (tenant_id)
- idx_audit_logs_resource (resource_type, resource_id)
- idx_audit_logs_actor (actor_type, actor_id)
- idx_audit_logs_created_at (tenant_id, created_at DESC)

**パーティショニング**: 月次パーティション（created_at）

---

### 3.11 notification_queue（通知キュー）

| カラム名 | 型 | NULL | デフォルト | 説明 |
|---------|------|------|-----------|------|
| id | VARCHAR(26) | NO | - | ULID |
| tenant_id | VARCHAR(26) | NO | - | テナントID（RLS用、データ削除用） |
| notification_type | VARCHAR(50) | NO | - | 通知種別 |
| recipient_type | VARCHAR(20) | NO | - | end_user/admin_user |
| recipient_id | VARCHAR(26) | NO | - | 受信者ID |
| recipient_email | VARCHAR(255) | NO | - | 送信先メールアドレス |
| subject | VARCHAR(200) | NO | - | メール件名 |
| body | TEXT | NO | - | メール本文 |
| status | VARCHAR(20) | NO | 'pending' | pending/sent/failed |
| retry_count | INTEGER | NO | 0 | リトライ回数 |
| scheduled_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 送信予定日時 |
| sent_at | TIMESTAMP | YES | NULL | 送信日時 |
| error_message | TEXT | YES | NULL | エラーメッセージ |
| created_at | TIMESTAMP | NO | CURRENT_TIMESTAMP | 作成日時 |

**インデックス**
- PRIMARY KEY (id)
- idx_notification_queue_tenant_id (tenant_id)
- idx_notification_queue_status (status, scheduled_at) WHERE status = 'pending'
- idx_notification_queue_recipient (recipient_type, recipient_id)
- fk_notification_queue_tenant FOREIGN KEY (tenant_id) REFERENCES tenants(id)

---

## 4. Row Level Security（RLS）

### 4.1 ポリシー定義例

```sql
-- テナント分離ポリシー
ALTER TABLE inquiries ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON inquiries
  USING (tenant_id = current_setting('app.current_tenant_id')::VARCHAR);

-- サービス分離ポリシー（オペレーター用）
CREATE POLICY service_isolation ON inquiries
  USING (
    service_id IN (
      SELECT service_id FROM service_roles
      WHERE admin_user_id = current_setting('app.current_user_id')::VARCHAR
    )
    OR current_setting('app.current_role') IN ('tenant_admin', 'system_admin')
  );
```

---

## 5. 改訂履歴

| 日付 | バージョン | 変更内容 | 作成者 |
|------|-----------|---------|--------|
| 2026-02-04 | 0.1 | 初版作成（全テーブル定義、RLS設計） | Claude |
| 2026-02-04 | 0.2 | 設計選択理由（決定ログ）追加 | Claude |
| 2026-02-05 | 0.3 | notification_queueにtenant_id追加、messages設計方針（初回投稿本文格納）追記 | Claude |
| 2026-02-05 | 0.4 | end_usersとservicesの関係についてCognito Groups依存の設計方針追記 | Claude |
