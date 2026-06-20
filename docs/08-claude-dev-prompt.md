# Claude（Claude Code）開発用プロンプト集 ― 送客機能のポータル組込

> [`07-referral-portal-spec.md`](./07-referral-portal-spec.md) を、既存の自社ポータル（Node.js/TS＋PostgreSQL）に実装するための、
> **そのまま貼って使えるプロンプト**集。Claude Code を**ポータルのリポジトリ内**で起動して使う想定。
>
> **使い方**
> 1. `07-referral-portal-spec.md` をポータルのリポジトリに置く（例 `docs/referral-spec.md`）。
> 2. 下の **［共通前提ブロック］** を毎回プロンプトの先頭に貼る。
> 3. 続けて **［ステップNブロック］** を1つ貼って実行 → レビュー → 次のステップへ。
> 4. まず **プロンプト0（探索）** から。いきなり実装させない。

---

## ⚙️ 事前に埋める変数（«» を置き換え）

- «ORM»：Prisma / TypeORM / Knex / Sequelize など、**リポジトリで実際に使っているもの**
- «AUTHの仕組み»：例「JWT＋`requireAuth` ミドルウェア」「Passport」「セッション」など
- «通知で使えるもの»：メール（SendGrid/SMTP）／Chatwork／Slack／LINE WORKS のうち使えるもの
- «シークレット管理»：例「環境変数 `.env`」「AWS Secrets Manager」
- «テスト»：例「Jest＋Supertest」「Vitest」

---

## ［共通前提ブロック］（毎回いちばん先頭に貼る）

```
あなたは、既存の自社ポータル（Node.js / TypeScript ＋ PostgreSQL）に「歯科医院向け送客機能」を
組み込むシニアエンジニアです。仕様は docs/referral-spec.md（添付）にあります。

# 絶対に守るルール
1. まず既存コードを読み、リポジトリの規約に「合わせる」。新しいフレームワーク/ORM/認証方式を
   勝手に導入しない。使用ORMは «ORM»、認証は «AUTHの仕組み» に従う。
2. 依存パッケージの追加・DBスキーマ変更・破壊的変更をする前に、理由と影響を説明し、許可を得る。
3. セキュリティ最優先：Webhookの署名/トークン検証、パートナーは自社partner_idのみ参照できる
   行レベル認可、個人情報(phone/email)は同意(consent=true)時のみ保存・返却。
4. 小さく進める。1ステップ＝1つの論理的な変更にまとめ、最後に変更点の要約と次の提案を出す。
5. テストを書く（«テスト»）。受け入れ基準は仕様書§11に従う。
6. 不明点・仕様の曖昧さ・既存実装と矛盾する点があれば、推測で進めず必ず質問する。
7. 秘密情報はコードに直書きせず «シークレット管理» を使う。

# 機能の要点（詳細は docs/referral-spec.md）
- LステップのWebhook転送 → POST /api/webhooks/lstep/referral で送客を受信・台帳化(referrals)
- パートナー専用ダッシュボードが GET /api/partner/referrals/summary を数秒ごとにポーリングし
  「今日/今月/累計の送客数」をリアルタイム表示
- 送客発生時にパートナーへ即時通知（«通知で使えるもの»）
- 成約(won)で成果報酬を自動計算、/api/admin/billing で月次集計
```

---

## プロンプト0：探索（まず現状把握・実装はしない）

```
［共通前提ブロックを貼る］

# タスク（今回は実装しない・調査と計画のみ）
1. リポジトリ構成を調べ、次を報告して：使用フレームワーク、«ORM»とマイグレーション方法、
   認証/認可の実装場所と仕組み、APIルーティングの規約、フロントの構成、テストの仕組み、
   環境変数の扱い、既存のusersテーブル定義。
2. 仕様書(docs/referral-spec.md)を、この既存規約にどう載せるかの「実装計画」を、
   仕様書§12のPR分割に沿って提案して（各PRの変更ファイル想定つき）。
3. 既存と矛盾する点・要確認点・リスクがあれば列挙して。

コードはまだ書かないで。計画と質問だけ出して。
```

---

## プロンプト1：DBマイグレーション

```
［共通前提ブロックを貼る］

# タスク：仕様書§3のスキーマを «ORM» のマイグレーションで追加
- partners / referrals / referral_status_history / webhook_events を作成。
- 既存 users に role (staff|admin|partner) と partner_id を追加（既存データは role=staff）。
- enum・命名・型はリポジトリ既存の規約に合わせる（既存がCHECK制約ならそれに合わせる）。
- ロールバック(down)も用意。seed に partners のサンプル1件を追加。
完了後：マイグレーション内容の要約と、ローカルでの適用手順を提示して。
```

---

## プロンプト2：Webhook受信API

```
［共通前提ブロックを貼る］

# タスク：POST /api/webhooks/lstep/referral を実装（仕様書§4.1）
- 検証：X-Signature の HMAC-SHA256（secretは環境変数）。Lステップがヘッダを付けられない
  場合に備え、URL秘密トークン＋event_id冪等のフォールバックも実装し、コメントで明記。
- 冪等：event_id が既存なら二重INSERTせず既存 referral_code を返す。
- 処理：partner_slug から partner 解決 → referral_code（R-YYYYMMDD-####）採番 →
  referrals へINSERT（consent=false の場合 phone/email は保存しない）→ webhook_events 記録。
- 通知発火は次ステップ。ここでは TODO コメントとインターフェースだけ用意。
- レスポンス/エラーは仕様書§4.1の通り。
- テスト：署名OK/NG、必須欠落、重複event_id、consent=falseでPII非保存 を網羅。
```

---

## プロンプト3：認可（パートナーロール＋行レベル）

```
［共通前提ブロックを貼る］

# タスク：認可を実装（仕様書§5）
- 既存 «AUTHの仕組み» を拡張し、requireRole('partner' | 'staff' | 'admin') を追加。
- partner ロールのリクエストは、全ての送客クエリを req.user.partner_id で必ずスコープする
  共通ヘルパ/ミドルウェアを用意（取りこぼし防止のため一箇所に集約）。
- テスト：partner A のトークンで partner B の送客（一覧/詳細/summary/CSV）が
  取得できないこと（403 もしくは空）を必ず検証。
```

---

## プロンプト4：パートナーAPI（summary中心）

```
［共通前提ブロックを貼る］

# タスク：パートナー向けAPIを実装（仕様書§4.2）
- GET /api/partner/referrals/summary?range=today|month|all
    → { today, month, total, by_type, by_status, updated_at }（ダッシュボードのポーリング先）
- GET /api/partner/referrals（フィルタ・ページング）/ :code（詳細）
- PATCH /api/partner/referrals/:code/status（status, deal_amount?, note?）
    → won のとき performance_fee を計算（fixed=固定 / rate=deal_amount×%）、status_history に記録
- GET /api/partner/referrals/export.csv（自社分）
- 全て partner_id スコープ。PIIは consent=true のみ返す。friend_id は返さない。
- summary はインデックス前提の集計クエリにする。
- テスト：件数の正しさ、scope、won時の成果報酬計算、consent制御。
```

---

## プロンプト5：ダッシュボードUI（ポーリング）

```
［共通前提ブロックを貼る］

# タスク：パートナー用ダッシュボード画面を、既存フロント規約で実装（仕様書§6）
- /api/partner/referrals/summary を 5〜10秒間隔でポーリング（既存のデータ取得ライブラリを使う。
  React Query/SWR があれば refetchInterval、なければ setInterval）。
- 表示：今日/今月/累計の送客数を大きく、内訳（資料請求/相談予約/見積依頼）、直近の送客リスト。
- 各送客のステータスを更新できるUI（PATCH連携）。
- タブ非アクティブ時(document.hidden)はポーリング停止。既存のデザイン/コンポーネントに合わせる。
- 認可エラー時の表示も用意。
```

---

## プロンプト6：通知サービス

```
［共通前提ブロックを貼る］

# タスク：送客発生時の即時通知を実装（仕様書§7）
- referral作成後に「非同期」で発火（受信APIの応答をブロックしない。既存のジョブ基盤があれば利用、
  なければ fire-and-forget＋リトライ）。
- partner.notify_channels に従い «通知で使えるもの» へ送信（チャンネルごとにアダプタを分離）。
- 本文：送客ID・医院名・地域・お悩み・タイプ・(同意時)連絡先・ダッシュボードリンク。
- プロンプト2で用意した TODO/インターフェースに接続。失敗時はログとリトライ。
- テスト：チャンネル選択、リトライ、本文整形。
```

---

## プロンプト7：管理API（自社スタッフ）

```
［共通前提ブロックを貼る］

# タスク：管理機能を実装（仕様書§4.3 / §8）
- partners の CRUD（課金条件 billing_type/placement_fee/performance_fee、通知先、カテゴリ）。
- GET /api/admin/referrals（全件・フィルタ）、PATCH 手動補正。
- GET /api/admin/billing?month=YYYY-MM
    → パートナー別に「掲載料(active×月額) ＋ 成果報酬(該当月のwon合計)」を集計。
- すべて requireRole('staff'|'admin')。テストで month集計の正しさを検証。
```

---

## レビュー用プロンプト（各PRの最後に）

```
直前の変更を、次の観点でセルフレビューして指摘・修正して：
1. パートナーが他社の送客を見られる経路が残っていないか（行レベル認可の漏れ）
2. Webhookの署名/トークン/冪等の穴
3. consent=false 時に phone/email が保存・返却されていないか
4. 仕様書§11の受け入れ基準を満たすテストがあるか
5. 既存規約からの逸脱・不要な依存追加
```

---

## 進め方のコツ

- **プロンプト0 → 1 → 2 …** と順に。各ステップでレビュー用プロンプトを回す。
- **1〜5までで「ワンクリック送客＋送客数のリアルタイム共有」が動く**ので、まずそこを最優先。
- 6（通知）・7（課金集計）は後追いでよい。
- 途中で曖昧さが出たら、Claudeは質問するルール（共通前提6）なので、その都度判断を返す。
