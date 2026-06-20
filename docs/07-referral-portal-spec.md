# 送客機能 ポータル組込 技術仕様書（Node.js/TypeScript ＋ PostgreSQL）

> 既存の自社ポータル（**Node.js / TypeScript ＋ PostgreSQL**）に「送客機能」を組み込むための仕様書。
> 設計の背景は [`06-referral-system.md`](./06-referral-system.md)、送客先・課金は [`01-routing-priority-master.md`](./01-routing-priority-master.md) §4、入力データは [`05-clinic-profile-questions.md`](./05-clinic-profile-questions.md)。
>
> **前提（ヒアリング確定）**
> - 認証：既存（自社ユーザー）に**パートナーロールを新規追加**
> - リアルタイム表示：**ポーリング**（数秒ごと自動更新）
> - 受信元：Lステップの**Webhook転送**（ボタン操作トリガー）
>
> ⚠️ コード例は参考。**実装は既存ポータルの規約（フレームワーク・ORM・認証・ディレクトリ構成）に合わせる**こと。

---

## 1. スコープ

1. Lステップ Webhook転送を受ける**受信API**（署名/トークン検証・冪等）
2. 送客を1件ずつ保存する**送客台帳**
3. **パートナー専用ダッシュボード**（送客数をポーリングでリアルタイム表示）＋ ステータス更新
4. 送客発生時の**即時通知**（メール/Chatwork/Slack/LINE WORKS）
5. **成果報酬の自動集計**と管理画面（自社スタッフ）

非スコープ：Lステップ自体の構築（[`04`](./04-implementation-steps.md)）、請求書発行システム連携（集計までを範囲とし、出力は連携可能な形に）。

---

## 2. アーキテクチャ

```
 先生のLINE（Lステップ）
   └ 送客ボタンをタップ（資料請求／相談予約／見積依頼）
   └ Webhook転送（リアルタイム）
        │  POST /api/webhooks/lstep/referral   ← 署名/トークン検証・冪等
        ▼
 自社ポータル（Node.js/TS）
   ├─ referrals テーブルに1行 INSERT（PostgreSQL）
   ├─ 通知サービス（非同期）→ パートナーへ即時通知
   └─ パートナーダッシュボード
        └ GET /api/partner/referrals/summary を 5〜10秒ごとにポーリング（件数リアルタイム）
```

---

## 3. データモデル（PostgreSQL）

ORM非依存の正規DDL。enum・命名・FKは既存規約に合わせて調整可。

```sql
-- 送客先（パートナー企業）
CREATE TABLE partners (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name             text NOT NULL,
  slug             text NOT NULL UNIQUE,           -- Lステップのボタンに埋める識別子・URL用
  billing_type     text NOT NULL DEFAULT 'performance', -- placement|performance|hybrid
  placement_fee_monthly integer,                   -- 掲載料（円/月）
  performance_fee_type  text,                       -- fixed|rate
  performance_fee_value numeric,                    -- fixed=円, rate=百分率(例 10.0)
  categories       text[] NOT NULL DEFAULT '{}',    -- 受け取るお悩みカテゴリ
  notify_channels  jsonb NOT NULL DEFAULT '{}',     -- {email, chatwork, slack, lineworks}
  status           text NOT NULL DEFAULT 'active',  -- active|paused
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now()
);

-- 送客台帳（1送客=1行）
CREATE TABLE referrals (
  id               uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  referral_code    text NOT NULL UNIQUE,            -- R-YYYYMMDD-#### 人間可読
  partner_id       uuid NOT NULL REFERENCES partners(id),
  category         text NOT NULL,                   -- お悩みカテゴリ
  referral_type    text NOT NULL,                   -- document_request|consultation|quote
  clinic_name      text NOT NULL,
  contact_name     text,
  prefecture       text,
  city             text,
  phone            text,                            -- 同意者のみ保存
  email            text,                            -- 同意者のみ保存
  lstep_friend_id  text,                            -- 内部ID（パートナー非共有）
  consent_third_party boolean NOT NULL DEFAULT false,
  consent_at       timestamptz,
  status           text NOT NULL DEFAULT 'referred',-- referred|in_progress|negotiating|won|lost
  deal_amount      integer,                         -- 成約金額（円）
  performance_fee  integer,                         -- 成果報酬（円・won時に計算）
  memo             text,
  created_at       timestamptz NOT NULL DEFAULT now(),
  updated_at       timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_referrals_partner_created ON referrals(partner_id, created_at DESC);
CREATE INDEX idx_referrals_status ON referrals(status);

-- ステータス変更履歴（監査）
CREATE TABLE referral_status_history (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  referral_id   uuid NOT NULL REFERENCES referrals(id),
  from_status   text,
  to_status     text NOT NULL,
  changed_by    uuid,                               -- users.id
  note          text,
  created_at    timestamptz NOT NULL DEFAULT now()
);

-- Webhook生ログ＋冪等
CREATE TABLE webhook_events (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  source        text NOT NULL DEFAULT 'lstep',
  event_id      text NOT NULL UNIQUE,               -- 重複排除キー
  payload       jsonb NOT NULL,
  referral_id   uuid REFERENCES referrals(id),
  processed     boolean NOT NULL DEFAULT false,
  received_at   timestamptz NOT NULL DEFAULT now()
);
```

**既存 users テーブルの拡張（パートナーロール追加）**

```sql
ALTER TABLE users ADD COLUMN role text NOT NULL DEFAULT 'staff';      -- staff|admin|partner
ALTER TABLE users ADD COLUMN partner_id uuid REFERENCES partners(id); -- role=partner のとき必須
```

> 既存authを活かす方針。別テーブルが望ましい場合は `partner_users` を新設してもよい（要件次第）。

---

## 4. API仕様

### 4.1 Webhook受信（Lステップ → ポータル）

`POST /api/webhooks/lstep/referral`

- **認証**：共有シークレットによる検証。①Lステップが任意ヘッダを付与できる場合は `X-Signature: HMAC-SHA256(body, secret)` を検証。②付与できない場合は**URLに秘密トークン**（例 `/api/webhooks/lstep/referral?t=XXXX`）＋ IP許可リスト＋下記event_id冪等で代替。
- **冪等**：`event_id` が既存なら処理せず既存 `referral_code` を返す。
- **リクエスト例**

```jsonc
{
  "event_id": "evt_8f3a...",            // ユニーク（必須）
  "friend_id": "U123...",               // Lステップ友だちID
  "clinic_name": "さくら歯科",
  "contact_name": "院長 佐藤",
  "prefecture": "東京都",
  "city": "渋谷区",
  "phone": "03-xxxx-xxxx",              // 同意時のみ
  "email": "info@example.com",          // 同意時のみ
  "category": "クラウドアポ帳",
  "partner_slug": "iqalte",             // ボタンごとに固定
  "referral_type": "document_request",  // document_request|consultation|quote
  "consent": true,
  "consent_at": "2026-06-01T09:00:00+09:00",
  "occurred_at": "2026-06-20T14:32:00+09:00"
}
```

- **処理**：署名/トークン検証 → `partner_slug` から partner解決 → `referral_code` 採番 → `referrals` INSERT（consent=false なら phone/email を保存しない）→ `webhook_events` 記録 → 通知を非同期発火。
- **レスポンス**：`201 { "referral_code": "R-20260620-0001" }` ／ 重複は `200 { "referral_code": "..." , "duplicated": true }`
- **エラー**：署名NG=`401`、partner不明/必須欠落=`422`。

### 4.2 パートナー向け（要 role=partner・**自社partner_idに必ずスコープ**）

| メソッド | パス | 説明 |
|---|---|---|
| GET | `/api/partner/referrals/summary?range=today\|month\|all` | **ダッシュボードのポーリング先**。`{ today, month, total, by_type:{...}, by_status:{...}, updated_at }` |
| GET | `/api/partner/referrals?from=&to=&status=&type=&page=` | 送客一覧（自社分のみ） |
| GET | `/api/partner/referrals/:code` | 送客詳細 |
| PATCH | `/api/partner/referrals/:code/status` | ステータス更新。`{status, deal_amount?, note?}`。`won` で成果報酬を計算 |
| GET | `/api/partner/referrals/export.csv` | CSV出力（自社分） |

### 4.3 管理（自社スタッフ・要 role=staff/admin）

| メソッド | パス | 説明 |
|---|---|---|
| GET/POST/PATCH | `/api/admin/partners` | パートナーCRUD（課金条件・通知先・カテゴリ） |
| GET | `/api/admin/referrals?...` | 全送客（フィルタ） |
| PATCH | `/api/admin/referrals/:code` | 手動補正 |
| GET | `/api/admin/billing?month=YYYY-MM` | パートナー別 掲載料＋成果報酬の集計 |

---

## 5. 認可（行レベル・最重要）

- `requireRole('partner')` ミドルウェアで、**全クエリを `req.user.partner_id` で必ず絞る**。パートナーは他社送客を一切取得不可（テスト必須）。
- `requireRole('staff'|'admin')` は全件可。
- PII（phone/email）は **consent=true の送客のみ**返す。パートナーには `lstep_friend_id` を返さない。

---

## 6. リアルタイム表示（ポーリング）

- フロント：パートナーダッシュボードが `GET /summary` を **5〜10秒間隔**でポーリング（React Query/SWR の `refetchInterval`、または `setInterval`）。
- 負荷軽減：`ETag` か `updated_since` で差分応答。タブ非アクティブ時はポーリング停止（`document.hidden`）。
- 表示要素：**今日／今月／累計の送客数（大きく）**、内訳（資料請求／相談予約／見積依頼）、直近の送客リスト（N件）、(フェーズB)成約数・成果報酬額。

---

## 7. 通知（即時）

- referral作成後に**非同期**で発火（ジョブキュー or fire-and-forget＋リトライ）。受信APIの応答はブロックしない。
- 宛先：`partner.notify_channels` に従い メール（SMTP/SendGrid）／Chatwork API／Slack Incoming Webhook／LINE WORKS。
- 本文：送客ID・医院名・地域・お悩み・タイプ・(同意時)連絡先・**ダッシュボードへのリンク**。

---

## 8. 成果報酬の計算

- ステータスが `won` になった時、`partner.performance_fee_type/value` から算出：
  - `fixed` → `performance_fee = performance_fee_value`
  - `rate`  → `performance_fee = round(deal_amount × performance_fee_value / 100)`
- 月次（`/api/admin/billing`）：`Σ placement_fee_monthly(active) + Σ performance_fee(該当月 won)`。

---

## 9. Lステップ側の設定（受信のための約束ごと）

- トリガー：送客ボタンの**ボタン操作** → **Webhook転送** → POST先＝本API。
- 送信項目：§4.1のJSON。`partner_slug`・`category`・`referral_type` は**ボタンごとに固定値**で埋める。`event_id` はユニーク値、連絡先は**同意者のみ**。
- 署名：Lステップが任意ヘッダを付けられない場合は **URL秘密トークン＋IP許可＋event_id冪等**で担保（§4.1）。

---

## 10. 非機能要件

- **セキュリティ**：署名/トークン検証、行レベル認可、PII最小化（consentで連絡先保存制御）、TLS、レート制限、監査ログ。
- **個人情報**：第三者提供同意のない送客は連絡先を保存・共有しない。保持期間ポリシーを定義。
- **冪等・信頼性**：webhook dedup、通知リトライ、失敗アラート。
- **観測性**：受信件数・失敗のログ／メトリクス。
- **パフォーマンス**：`summary` はインデックス前提の集計クエリ（必要なら短TTLキャッシュ）。

---

## 11. 受け入れ基準（テスト観点）

1. 署名/トークン不正→`401`、正当→`201`、重複`event_id`→冪等（2回目も同じcode・二重INSERTなし）。
2. **パートナーAはパートナーBの送客を取得できない**（一覧・詳細・summary・CSV全て）。
3. `summary` の件数が実データと一致し、新規送客後のポーリングで増える。
4. `won` 時に `performance_fee` が `fixed`/`rate` で正しく計算される。
5. `consent=false` の送客は phone/email が保存されず、APIでも返らない。
6. `/admin/billing` の月次合計が掲載料＋成果報酬と一致。

---

## 12. 実装ステップ（PR分割の推奨）

1. **DBマイグレーション**（partners / referrals / status_history / webhook_events / users.role・partner_id）
2. **Webhook受信API**（検証・冪等・採番・INSERT）
3. **認可**（partnerロール＋行レベルスコープ）
4. **パートナーAPI**（summary / list / detail / status / export）
5. **ダッシュボードUI**（ポーリング）
6. **通知サービス**（チャンネル別アダプタ＋リトライ）
7. **管理API**（partners CRUD / billing）
8. **テスト・seed・ドキュメント**

> 各ステップを1PRに。**まず1〜5でワンクリック送客＋件数リアルタイム共有が動く**ところまでを最優先。
