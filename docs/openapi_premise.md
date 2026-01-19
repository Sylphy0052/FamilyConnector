## OpenAPI作成の前提（確定事項まとめ）

### 0. 目的

本ドキュメントは、家族向けアプリ（iOS + Web）の **OpenAPI（v1）を作成するための前提条件**を固定し、以降の設計・実装（Nest + Prisma）で手戻りが発生しないようにする。

---

## 1. 対象範囲・前提条件

- クライアント：**iOS + Web（PC）**
- 利用人数：**2人**
- レシート画像：最大 **10枚/週（≒40枚/月）**
- 画像/PDF添付：**ほぼ無し**
- 認証：**必須**
- Push通知：**必須**
  - iOS：APNs
  - Web：Web Push（VAPID + Service Worker）
- v1.0：**ローカルPCでサーバを起動**してテスト
- vNext：**無料最小構成**へ移行（Workers/Pages/Supabase/GHA想定）
- ローカル到達性：**トンネル方式採用**（APNs/Web Push検証の前提）

---

## 2. 技術スタック前提（OpenAPIに影響するもの）

- API：**NestJS**
- Worker：**NestJS（別アプリ/プロセス）**
- DBアクセス：**Prisma**
- DB：PostgreSQL
- OCR：**Tesseract（自前）**
  - v1.0：Tesseractのみ
  - vNext：精度が悪ければ **GPT/VLM**採用（前処理としてトリミング導入）
  - OCRは **Pythonサービス（HTTP）**として分離

---

## 3. 共通設計ルール（横断の契約）

### 3.1 ID / 時刻 / タイムゾーン

- ID：**UUID v4** を全エンティティで統一
- DB保存時刻：`createdAt` / `updatedAt` は **UTCで保存**
- API返却：日時は **ISO8601（UTC）**で返す
- 集計・判定（Home/通知/期限判定）：
  - **JST固定**
  - 週区切り：**月曜開始**
- ソート規約（既定）：
  - `updatedAt desc, id desc`

### 3.2 共通カラム（適用方針）

- 原則として全主要エンティティに以下を持つ：
  - `householdId`
  - `updatedAt`（サーバ時刻）
  - `version`（楽観ロック）
  - `deletedAt`（論理削除、v1.0で統一）

### 3.3 論理削除（deletedAt）

- v1.0の削除は **deletedAt を立てる**（物理削除しない）
- 物理削除は保持期限バッチ（worker）でのみ実施

---

## 4. 認証・権限（OpenAPIの前提）

- 認証：**Supabase Auth**
  - クライアントでログインし、APIへ **Bearer JWT** を付与
  - APIはJWT検証し `memberId/householdId` を確定
- 権限：`householdId` 一致を最小ガードとする
- 家族表示制御：
  - **カレンダー詳細は本人のみ**
  - 家族には **hasEventフラグのみ**を返す（詳細非公開）

---

## 5. 同期（Delta Sync）前提（OpenAPI骨格）

- 方式：**差分取得（Delta Sync）**
  - `since = lastSyncedAt`（クライアント保持）
  - `updatedAt > since` の更新データを返す
- エンドポイント：`GET /api/v1/sync?since=<timestamp>`
- レスポンス形式：**まとめ返し**
  - `data: { domain1: [], domain2: [], ... }` の入れ子形式で固定
- 個人領域（themes/assets）：**Syncに含めるが本人のときだけ返す**
- ページング：v1.0は必須ではないが、将来拡張のため `nextCursor` を予約可能

---

## 6. Home集約API前提（v1.0はサマリ中心）

- Homeは **サーバ集約**で1回の呼び出しで必要サマリを返す
- v1.0 Homeサマリは以下を中心（詳細は各ドメインAPI）：
  - 未読メッセージ件数
  - 今日予定あり（本人は詳細別、家族はhasEventのみ）
  - 在庫の期限接近件数（閾値は設定）
  - 今週支出合計（週区切り：月曜開始）
- 期限接近閾値：
  - **設定可能**
  - Default：**3日**
  - 設定スコープ：**Household共有**

---

## 7. 書き込みAPIの共通契約（冪等性・競合・警告）

### 7.1 冪等性（Idempotency）

- 更新系（POST/PUT/PATCH/DELETE）は原則すべて
- ヘッダ：`Idempotency-Key: <uuid>` を使用
- サーバ側で冪等性テーブル等により二重実行を抑止

### 7.2 競合（version）

- 更新系リクエストは `version` を必須
- 競合時の既定動作：
  - **後勝ちで受理**
  - **warning（警告）イベント**を生成し、Syncで配布

### 7.3 通知の一意性

- 仕様：同一イベント/同一受信者へ **1回**
- DBで一意制約（Outbox）を持ち、再試行でも二重配信しない

---

## 8. Worker（ジョブ）前提

- v1.0：**常駐（cron相当）**で実行
- vNext：**GitHub Actions Cron**へ移行
- 対象ジョブ（OCR以外）：
  - 期限削除（在庫/買い物）
  - 監査ログ/削除ログの保持期限削除（3ヶ月）
  - 通知送信の再試行（必要時）

---

## 9. OCR前提（v1.0：Tesseractのみ）

- OCRは **オンデマンド**（リクエスト時のみ）
- OCRサービス：**Python（HTTP） + Tesseract**
- OCR API（エンジン差し替え可能なI/Fを固定）：
  - `POST /api/v1/receipts/{id}/ocr`
  - v1.0レスポンス最小：
    - `engine = "tesseract"`
    - `rawText`（必須）
    - `totalCandidates`（任意）
- vNextでGPT/VLM採用時：
  - 前処理として「レシート該当部分のトリミング」を導入
  - OpenAPI上は `crop` パラメータを予約可能（未実装でもI/F維持）

---

## 10. OpenAPIの管理方式（作成・固定の前提）

- OpenAPI生成元：Nest DTO + `@nestjs/swagger`
- APIバージョン：`/api/v1`
- OpenAPI成果物（JSON/YAML）をリポジトリ管理し、差分レビュー可能にする
- 共通エラー形式（全API共通）：
  - `{ code, message, details?, requestId }`

---
