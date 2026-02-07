# お問い合わせ管理基盤SaaS コーディングルール

## 0. 決定ログ

| 選択 | 理由 | 再検討条件 |
|------|------|-----------|
| TypeScript strict mode | 型安全性の担保、実行時エラーの削減 | なし（必須） |
| Conventional Commits | 変更履歴の自動生成、セマンティックバージョニング対応 | なし |
| ESLint + Prettier | コードスタイル統一、自動フォーマット | なし |
| Jest + Testing Library | TypeScriptとの親和性、モック機能の充実 | E2EはPlaywright検討 |
| GitHub Flow | シンプル、少人数チーム向け | チーム拡大時はGit Flowへ |

---

## 1. 基本原則

### 1.1 品質基準

- **型安全性**: TypeScript strict modeを必須とし、`any`型の使用を禁止
- **可読性**: 経験3年目のエンジニアがレビュー可能なコードを書く
- **一貫性**: チームメンバー間でコード品質を均一に保つ
- **テスト優先度**: 実装完了後にテストを書く（テストを通すための実装改修は禁止）

### 1.2 リーダブルコードの原則

| 原則 | 具体的なルール |
|------|---------------|
| 明確な命名 | 変数・関数名は意図を明確に表す。略語は禁止（`usr`→`user`） |
| 単一責任 | 1関数は1つの責務のみ。20行を超えたら分割を検討 |
| 早期リターン | ネストを減らすためにガード節を使用 |
| マジックナンバー禁止 | 定数として定義し、意味を明示 |
| コメントは「なぜ」 | 「何をしているか」ではなく「なぜそうするか」を書く |

---

## 2. TypeScript規約

### 2.1 型定義

```typescript
// Good: 明示的な型定義
interface Inquiry {
  id: string;
  subject: string;
  status: InquiryStatus;
  createdAt: Date;
}

// Bad: any型の使用
const data: any = fetchData(); // 禁止

// Good: unknown + 型ガード
const data: unknown = fetchData();
if (isInquiry(data)) {
  // data is Inquiry
}
```

### 2.2 Enum vs Union Types

```typescript
// 推奨: Union Types（型の絞り込みが効く）
type InquiryStatus = 'pending' | 'in_progress' | 'answered' | 'completed' | 'on_hold';

// 非推奨: Enum（バンドルサイズ増加、Tree shakingが効かない）
enum InquiryStatus {
  Pending = 'pending',
  // ...
}
```

### 2.3 Null/Undefined

```typescript
// strictNullChecks: true 必須

// Good: Optional chaining + Nullish coalescing
const name = user?.profile?.name ?? 'Unknown';

// Bad: 非null assertion（型システムのバイパス）
const name = user!.profile!.name; // 禁止
```

### 2.4 tsconfig.json 必須設定

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

---

## 3. クリーンアーキテクチャ規約

### 3.1 レイヤー構成

```
src/
├── domain/           # ビジネスロジック（外部依存なし）
│   ├── entities/     # エンティティ
│   ├── value-objects/# 値オブジェクト
│   ├── repositories/ # リポジトリインターフェース
│   └── services/     # ドメインサービス
├── application/      # ユースケース
│   ├── use-cases/    # アプリケーションサービス
│   ├── dto/          # Data Transfer Objects
│   └── ports/        # 入出力ポート
├── infrastructure/   # 外部システム連携
│   ├── persistence/  # DB実装
│   ├── external/     # 外部API連携
│   └── messaging/    # メッセージング
└── presentation/     # API/UI層
    ├── controllers/  # HTTPコントローラー
    ├── middleware/   # ミドルウェア
    └── validators/   # 入力バリデーション
```

### 3.2 依存関係ルール

```
domain ← application ← infrastructure ← presentation
           ↑                    ↑
           └────────────────────┘
           （依存性注入で逆転）
```

**禁止事項**:
- domain層からapplication/infrastructure/presentation層への依存
- application層からinfrastructure/presentation層への依存
- 循環依存

### 3.3 レイヤー間のデータ変換

```typescript
// presentation → application: RequestDTO
// application → domain: Entity/ValueObject
// domain → application: Entity/ValueObject
// application → presentation: ResponseDTO

// 変換は各境界で明示的に行う
class InquiryController {
  async create(req: CreateInquiryRequest): Promise<InquiryResponse> {
    const dto = CreateInquiryDto.fromRequest(req);
    const inquiry = await this.useCase.execute(dto);
    return InquiryResponse.fromEntity(inquiry);
  }
}
```

---

## 4. 命名規約

### 4.1 ファイル・ディレクトリ

| 対象 | 規約 | 例 |
|------|------|-----|
| ファイル名 | kebab-case | `inquiry-repository.ts` |
| クラス名 | PascalCase | `InquiryRepository` |
| インターフェース | PascalCase（I接頭辞なし） | `InquiryRepository`（not `IInquiryRepository`） |
| 定数 | UPPER_SNAKE_CASE | `MAX_ATTACHMENT_SIZE` |
| 変数・関数 | camelCase | `createInquiry` |
| 型定義 | PascalCase | `InquiryStatus` |

### 4.2 サービス間通信

| 対象 | 規約 | 例 |
|------|------|-----|
| APIパス | kebab-case | `/api/v1/admin/inquiries` |
| JSONフィールド | camelCase | `{ "inquiryId": "..." }` |
| DBカラム | snake_case | `inquiry_id` |
| ヘッダー | X-Prefix + PascalCase | `X-Correlation-Id` |

### 4.3 JSON/DB命名変換

JSONフィールド（camelCase）とDBカラム（snake_case）間の変換は、以下の方法で統一する。

**NestJS（バックエンド）**

```typescript
// class-transformerの@Exposeデコレータで変換
import { Expose } from 'class-transformer';

export class InquiryResponseDto {
  @Expose({ name: 'inquiry_id' })
  inquiryId: string;

  @Expose({ name: 'created_at' })
  createdAt: Date;
}

// またはTypeORMの命名戦略を使用
// ormconfig.ts
{
  namingStrategy: new SnakeNamingStrategy()
}
```

**変換ライブラリ**

| 用途 | ライブラリ | 設定 |
|------|----------|------|
| DTO変換 | class-transformer | `@Expose({ name: 'snake_case' })` |
| ORM命名 | typeorm-naming-strategies | `SnakeNamingStrategy` |
| 汎用変換 | lodash | `_.camelCase()` / `_.snakeCase()` |

**変換対象外**

- JWTクレーム名（Cognito準拠: `cognito:groups`, `custom:role`）
- 外部API連携時の形式（相手方APIの仕様に準拠）

---

## 5. エラーハンドリング

### 5.1 例外の分類

```typescript
// ドメイン例外（ビジネスルール違反）
abstract class DomainException extends Error {
  abstract readonly code: string;
  abstract readonly httpStatus: number;
}

class InquiryNotFoundException extends DomainException {
  readonly code = 'INQ-001';
  readonly httpStatus = 404;
}

class InvalidStatusTransitionException extends DomainException {
  readonly code = 'INQ-002';
  readonly httpStatus = 400;
}

// インフラ例外（技術的エラー）
abstract class InfrastructureException extends Error {
  abstract readonly code: string;
  readonly httpStatus = 500;
}

class DatabaseConnectionException extends InfrastructureException {
  readonly code = 'INFRA-001';
}
```

### 5.2 エラーコード体系

| プレフィックス | サービス | 例 |
|--------------|---------|-----|
| INQ | Inquiry Service | INQ-001: 問い合わせが見つからない |
| USR | User Service | USR-001: ユーザーが見つからない |
| TNT | Tenant Service | TNT-001: テナントが見つからない |
| NTF | Notification Service | NTF-001: 通知送信失敗 |
| AUTH | 認証・認可 | AUTH-001: トークン無効 |
| INFRA | インフラ共通 | INFRA-001: DB接続失敗 |

### 5.3 リトライポリシー

| シナリオ | リトライ回数 | 間隔 | バックオフ |
|---------|------------|------|-----------|
| DB接続失敗 | 3回 | 1秒 | 指数（1s, 2s, 4s） |
| 外部API呼び出し | 3回 | 2秒 | 指数（2s, 4s, 8s） |
| メッセージ送信 | 5回 | 5秒 | 固定 |
| 4xx エラー | 0回 | - | リトライしない |

---

## 6. ログ標準

### 6.1 構造化ログフォーマット

```json
{
  "timestamp": "2026-02-04T10:30:00.000Z",
  "level": "INFO",
  "service": "inquiry-service",
  "traceId": "abc123",
  "spanId": "def456",
  "userId": "usr_xxx",
  "tenantId": "tnt_xxx",
  "message": "Inquiry created",
  "data": {
    "inquiryId": "inq_xxx",
    "status": "pending"
  }
}
```

### 6.2 ログレベル基準

| レベル | 用途 | 例 |
|-------|------|-----|
| ERROR | 即時対応が必要なエラー | DB接続失敗、外部API障害 |
| WARN | 監視対象だが即時対応不要 | リトライ発生、期限間近 |
| INFO | 正常系の重要イベント | 問い合わせ作成、ステータス変更 |
| DEBUG | 開発時のデバッグ情報 | リクエスト詳細、SQL |

### 6.3 機密データ取り扱い

**ログ出力禁止**:
- パスワード、トークン
- メールアドレス（マスキング必須: `xxx@example.com` → `x**@***.com`）
- 氏名（マスキング必須: `山田太郎` → `山*太*`）
- 電話番号
- クレジットカード情報

```typescript
// ログユーティリティで自動マスキング
const logger = createLogger({
  maskFields: ['email', 'name', 'phone', 'password', 'token']
});
```

---

## 7. サービス間通信

### 7.1 タイムアウト設定

| 通信種別 | タイムアウト | 理由 |
|---------|------------|------|
| 同期HTTPリクエスト | 5秒 | ユーザー体験を損なわない範囲 |
| 非同期ジョブ | 30秒 | バッチ処理の余裕 |
| DB接続 | 3秒 | 早期失敗でリトライ |
| Redis | 1秒 | キャッシュは高速であるべき |

### 7.2 サーキットブレーカー

```typescript
const circuitBreakerConfig = {
  failureThreshold: 5,      // 連続失敗5回でオープン
  successThreshold: 3,      // 連続成功3回でクローズ
  timeout: 30000,           // オープン状態の維持時間（30秒）
  halfOpenRequests: 3       // ハーフオープン時の試行回数
};
```

### 7.3 相関ID（Correlation ID）

```typescript
// 全リクエストにX-Correlation-Idを伝播
@Middleware()
class CorrelationIdMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const correlationId = req.headers['x-correlation-id'] || generateUlid();
    req.correlationId = correlationId;
    res.setHeader('X-Correlation-Id', correlationId);
    AsyncLocalStorage.run({ correlationId }, next);
  }
}
```

---

## 8. テスト規約

### 8.1 テスト戦略

| レイヤー | テスト種別 | カバレッジ目標 |
|---------|----------|---------------|
| Domain | 単体テスト | 80%以上 |
| Application | 単体テスト + 統合テスト | 70%以上 |
| Infrastructure | 統合テスト | 50%以上 |
| Presentation | E2Eテスト | クリティカルパスのみ |

### 8.2 テストファイル配置

```
src/
├── domain/
│   ├── entities/
│   │   ├── inquiry.ts
│   │   └── inquiry.spec.ts  # 同階層に配置
```

### 8.3 テストの原則

```typescript
// Good: 実装を検証するテスト
describe('InquiryService', () => {
  it('should change status from pending to in_progress', async () => {
    const inquiry = await service.changeStatus(inquiryId, 'in_progress');
    expect(inquiry.status).toBe('in_progress');
  });
});

// Bad: テストを通すための実装変更（禁止）
// 既存のテストが失敗する場合、実装側の改修内容を掌握した上で修正する
```

### 8.4 テストデータ管理

```typescript
// ファクトリパターンを使用
const inquiryFactory = createFactory<Inquiry>({
  id: () => generateUlid(),
  subject: 'テスト問い合わせ',
  status: 'pending',
  createdAt: () => new Date()
});

// 使用例
const inquiry = inquiryFactory.build({ status: 'in_progress' });
```

### 8.5 E2Eテスト対象

クリティカルパスのみE2Eテストを実施:
1. ログイン → 問い合わせ投稿 → 確認画面表示
2. 管理者ログイン → 問い合わせ一覧 → 回答送信
3. ステータス変更フロー（pending → in_progress → answered → completed）

---

## 9. Git運用

### 9.1 ブランチ戦略（GitHub Flow）

```
main                    # 本番リリース可能な状態を常に維持
  └── feature/INQ-123-add-deadline-api  # 機能開発
  └── fix/INQ-456-fix-status-transition # バグ修正
  └── hotfix/INQ-789-critical-fix       # 緊急修正
```

### 9.2 コミットメッセージ（Conventional Commits）

```
<type>(<scope>): <subject> #<issue-number>

<body>

<footer>
```

| Type | 用途 |
|------|------|
| feat | 新機能 |
| fix | バグ修正 |
| docs | ドキュメントのみの変更 |
| style | コードの意味に影響しない変更（フォーマット等） |
| refactor | バグ修正でも機能追加でもないコード変更 |
| test | テストの追加・修正 |
| chore | ビルドプロセスや補助ツールの変更 |

```bash
# 例
feat(inquiry): 期限変更APIを追加 #INQ-123

- PATCH /api/v1/admin/inquiries/{id}/deadline エンドポイントを追加
- 期限変更時に期限変更通知を送信

Closes #INQ-123
```

### 9.3 PRルール

| ルール | 内容 |
|-------|------|
| レビュー | 最低1人の承認必須 |
| CI | 全テスト通過必須 |
| マージ方式 | Squash Merge |
| タイトル | Conventional Commits形式 |
| 本文 | 変更内容、テスト方法、影響範囲を記載 |

### 9.4 コードレビューチェックリスト

- [ ] 型安全性: any型を使用していないか
- [ ] エラーハンドリング: 例外が適切に分類されているか
- [ ] ログ: 機密情報がログに出力されていないか
- [ ] テスト: カバレッジ目標を満たしているか
- [ ] 命名: 規約に従っているか
- [ ] 依存関係: クリーンアーキテクチャの依存方向に違反していないか
- [ ] ドキュメント: 変更に伴うドキュメント更新があるか

---

## 10. フロントエンド規約（MVVM）

### 10.1 ディレクトリ構成

```
src/
├── app/                    # Next.js App Router
│   ├── (user)/            # エンドユーザー向けルート
│   └── (admin)/           # 管理者向けルート
├── components/            # 再利用可能なUIコンポーネント
│   ├── ui/               # 基本UI部品（Button, Input等）
│   └── features/         # 機能別コンポーネント
├── view-models/          # ViewModel（状態管理・ロジック）
├── models/               # 型定義・ドメインモデル
├── services/             # API通信
└── hooks/                # カスタムフック
```

### 10.2 MVVM責務分離

| レイヤー | 責務 | 例 |
|---------|------|-----|
| Model | データ構造、型定義 | `Inquiry`, `InquiryStatus` |
| ViewModel | 状態管理、ビジネスロジック | `useInquiryListViewModel` |
| View | UI描画のみ | `InquiryListPage` |

```typescript
// ViewModel
export function useInquiryListViewModel() {
  const [inquiries, setInquiries] = useState<Inquiry[]>([]);
  const [filter, setFilter] = useState<InquiryFilter>({});
  const [isLoading, setIsLoading] = useState(false);

  const fetchInquiries = async () => {
    setIsLoading(true);
    const result = await inquiryService.list(filter);
    setInquiries(result.data);
    setIsLoading(false);
  };

  return { inquiries, filter, isLoading, setFilter, fetchInquiries };
}

// View
export function InquiryListPage() {
  const { inquiries, isLoading, fetchInquiries } = useInquiryListViewModel();

  useEffect(() => { fetchInquiries(); }, []);

  if (isLoading) return <Loading />;
  return <InquiryList items={inquiries} />;
}
```

### 10.3 コンポーネント規約

- **純粋コンポーネント**: UIコンポーネントはpropsのみに依存
- **状態はViewModelに**: コンポーネント内でのuseState最小化
- **CSS**: Tailwind CSS使用、インラインスタイル禁止

---

## 11. ドキュメント規約

### 11.1 ドキュメント更新タイミング

| 変更種別 | 更新対象 |
|---------|---------|
| API追加・変更 | api-design.md, openapi.yaml |
| DB変更 | db-design.md |
| 画面追加・変更 | screen-design.md |
| インフラ変更 | infra-design.md |
| コーディングルール変更 | coding-rules.md |

### 11.2 コードコメント

```typescript
// Good: 「なぜ」を説明
// 期限計算は営業日ベースで行う（祝日は除外）
const deadline = calculateBusinessDays(createdAt, slaHours);

// Bad: 「何」を説明（コードを読めばわかる）
// deadlineを計算する
const deadline = calculateBusinessDays(createdAt, slaHours);
```

---

## 12. 静的解析・フォーマット

### 12.1 ESLint設定

```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked"
  ],
  "rules": {
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/explicit-function-return-type": "error",
    "@typescript-eslint/no-unused-vars": "error",
    "no-console": "warn"
  }
}
```

### 12.2 Prettier設定

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

### 12.3 Pre-commit Hooks

```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS"
    }
  },
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

---

## 13. 改訂履歴

| 日付 | バージョン | 変更内容 | 作成者 |
|------|-----------|---------|--------|
| 2026-02-04 | 0.1 | 初版作成 | Claude |
| 2026-02-05 | 0.2 | JSON/DB命名変換ルール（4.3節）追加 | Claude |
