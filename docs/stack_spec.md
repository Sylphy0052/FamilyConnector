## 技術スタック仕様書に必要な情報（要点整理・収集項目一覧）

### 0. 本ドキュメントの目的

- v1.0（ローカルPC実行）→ vNext（無料最小構成）へ、**コード・API契約・DB設計の手戻りを最小化**して移行するために、
  技術スタックに関する決定事項／未決定事項／受け入れ条件を整理する。

---

## 1. 前提条件（ビジネス/利用条件）

- 対象プラットフォーム：iOS + Web（PC）
- 利用人数：2人
- レシート画像：最大 10枚/週（≒40枚/月）
- 画像/PDF添付：ほぼ無し
- Push通知：必須（iOS APNs + Web Push）
- 認証：必須（Supabase Auth利用可、口頭共有運用も許容）
- OCR方針：
  - v1.0：Tesseract（自前）オンデマンドのみ
  - vNext：Tesseract精度が悪い場合に GPT/VLM を採用（トリミング機能を前処理として導入）
- v1.0動作環境：ローカルPCでサーバを起動して検証
- vNext動作環境：無料最小構成に移行

---

## 2. 非機能要件（技術スタックに影響する必須項目）

- 同期：Delta Sync（`updatedAt > lastSyncedAt`）で差分取得
- 競合：楽観ロック（`version`）＋後勝ち受理＋警告イベント生成（Warning）
- 冪等性：`clientRequestId` 等による冪等処理（更新/派生処理含む）
- 通知：同一イベント/同一受信者へ **1回**（DBで一意制約を担保）
- 監査ログ：保持 3ヶ月、削除バッチ
- バッチ/ジョブ：
  - OCR以外：期限削除（在庫/買い物/ログ）、通知再送（必要時）等
  - OCR：オンデマンド（API呼び出し時のみ）
- メディア：軽量版生成等は当面簡略化可（添付ほぼ無し前提）

---

## 3. アーキテクチャ方針（v1.0 / vNext 共通の“形”）

### 3.1 サービス分割（移行容易性のため固定）

- `api`：REST API（OpenAPI契約の提供、同期/集約/更新、Push送信）
- `db`：PostgreSQL（整合性の正、一意制約の担保）
- `worker`：ジョブ実行（OCR以外の定期処理、必要時の再送）
- `ocr`：OCRエンジン（v1.0は同居でもよいが、将来差し替え可能な境界を持つ）

### 3.2 契約固定（重要）

- OpenAPI：A〜Gの必須エンドポイントを先に固定
- OCR API：`POST /receipts/{id}/ocr` を **エンジン差し替え可能**な形で固定（Tesseract → GPT）
- 通知送信：DB Outbox/一意制約を前提に設計（再試行でも二重配信しない）

---

## 4. v1.0 技術スタック（ローカルPC実行）

### 4.1 実行・配布

- Docker Composeで起動（`api`/`db`/`worker`/`ocr`）
- ローカルでWeb/ API/ DB/ Job が一体で動くこと

### 4.2 Web（PC）

- Next.js（React + TypeScript）
- ローカル開発サーバで検証

### 4.3 iOS

- Swift + SwiftUI
- APNs：ローカルAPI到達性を満たすための開発用HTTPS到達手段（※方式は後決め）

### 4.4 Backend（API）

- TypeScript / Node.js
- REST（OpenAPI）＋CORS設計
- Push送信（APNs/Web Push）実装の置き場はAPI側

### 4.5 DB

- PostgreSQL（Docker）
- 主要テーブル群（例）：
  - households / members / auth linkage
  - messages / reads / reactions
  - inventory / shopping / receipts / schedules
  - assets（最小）
  - notifications(outbox)（一意制約必須）
  - idempotency（冪等性テーブル）
  - audit_logs / deletion_logs（保持期限あり）
  - ocr_runs / ocr_results（将来差し替え前提）

### 4.6 OCR（v1.0）

- Tesseract（自前、オンデマンドのみ）
- 前処理：最小（回転/二値化/ノイズ除去等）
- 成功条件（v1.0）：rawText（全文）＋合計候補（任意）

---

## 5. vNext 技術スタック（無料最小構成へ移行）

### 5.1 Web

- Cloudflare Pages（無料）

### 5.2 API

- Cloudflare Workers（無料枠）
- 役割：REST API、Push送信、軽量処理

### 5.3 DB / Auth

- Supabase Free（Postgres + Auth）
- 役割：認証、整合性の正、一意制約、監査ログ

### 5.4 Worker（定期ジョブ）

- GitHub Actions Cron（無料枠）
- 対象：期限削除、監査ログ削除、通知再送（必要時）等
- 注意：正確な時刻保証が弱いので「1日1回で成立する設計」に寄せる

### 5.5 OCR（vNext）

- 優先：Tesseract継続（精度が許容なら）
- 代替：GPT/VLM API採用（精度が悪い場合）
  - 前処理として「レシート該当部分のトリミング」を導入（手動/半自動）
  - OCR API契約は維持（エンジン差し替えのみ）

---

## 6. セキュリティ/鍵管理（技術スタック仕様書に必須）

- Auth：Supabase Auth の方式（メール/パスワード、招待コード等の運用方針）
- Push鍵：
  - APNs：キーID/チームID/トピック、鍵の保管方法（ローカル/Workers Secret）
  - Web Push：VAPID鍵の保管方法（同上）
- 外部APIキー（vNextでGPT採用する場合）：Secrets管理方式
- ログのPII方針：レシート画像やrawTextの保管/マスキング方針（最小でも方針を記載）

---

## 7. 運用・可観測性（最小でも仕様書に必要）

- 監視：ヘルスチェック（/healthz）、障害時の確認手順
- ログ：APIログ/ジョブログの保管先（ローカルファイル、Supabase Logs等）
- バックアップ：
  - v1.0：ローカルVolumeバックアップ方針
  - vNext：Supabase側のバックアップ/エクスポート方針（最低限の手順）
- 移行：DBマイグレーション（Prisma等）採用有無と運用方針

---

## 8. 受け入れ条件（Definition of Done）

### v1.0（ローカル）

- Docker Composeで `api`/`db`/`worker` が起動
- iOS実機でPush受信（APNs疎通）
- Web Push疎通（Service Worker）
- OCRオンデマンドが動く（TesseractでrawTextが得られる）
- Delta Sync が動く（lastSyncedAt差分）

### vNext（無料最小）

- Cloudflare PagesにWebがデプロイ
- Cloudflare WorkersにAPIがデプロイ
- Supabase FreeでAuth/DB稼働
- GitHub Actions Cronで期限削除が動作
- Push（iOS/Web）が本番相当で受信できる

---

## 9. 未決定事項（仕様書に“空欄”として明示すべき）

- ローカルでのHTTPS到達手段（APNs/Web Push検証用：トンネル/開発ドメイン等）
- API実装フレームワーク（Hono/Fastify/Nest等）
- DBマイグレーションツール（Prisma / Drizzle / Flyway 等）
- OCR前処理の具体（回転補正/トリミングUIの方式）
- GPT採用時のモデル選定/コスト上限/ハルシネーション対策（検証ルール）
- 画像保管方針（保存期間、原本/処理後、容量制限）

---

## 10. 技術スタック仕様書の“最低限の章立て”（提案）

1. 前提・スコープ
2. 非機能要件（同期・冪等・通知一意・ログ保持）
3. v1.0（ローカル）構成（Compose、サービス分割）
4. vNext（無料最小）構成（Pages/Workers/Supabase/GHA）
5. 認証・鍵管理（APNs/Web Push/Secrets）
6. OCR方針（Tesseract→必要ならGPT、トリミング）
7. 運用（監視、ログ、バックアップ、移行）
8. DoD（受け入れ条件）
9. 未決定事項・今後の検討項目
