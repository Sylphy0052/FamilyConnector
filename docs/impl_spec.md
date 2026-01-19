# 実装仕様まとめ

本ドキュメントは、**DBスキーマ確定以降に決定した内容**を整理し、以降の実装（Nest / Prisma / Worker / OCR）で迷いが出ないようにするための **実装指針まとめ**である。

---

## 1. 全体像（この段階で何が確定したか）

- DBスキーマ（Prisma）は **v1.0確定**
- Idempotency（冪等性）の **設計・TTL・スコープ** を確定
- 通知Outboxの **DB一意制約による二重防止** を確定
- OCR（Tesseract）の **結果キャッシュ方式** を確定
- Schedule（Google同期結果）の **DBキャッシュ方針** を確定
- 次工程は **Home集約API設計** に進める状態

---

## 2. DBスキーマ設計の要点（schema.prismaの意図）

### 2.1 横断ルール

- ID：UUID v4
- 共通カラム（主要エンティティ）：
  - `householdId`
  - `createdAt`
  - `updatedAt`
  - `version`（楽観ロック）
  - `deletedAt`（論理削除）
- Sync最適化：
  - `@@index([householdId, updatedAt])` をほぼ全テーブルに付与
- 削除：
  - v1.0は **論理削除のみ**
  - 物理削除は Worker で保持期限超過分のみ

---

## 3. Idempotency（冪等性）設計【確定】

### 3.1 Idempotencyとは（API文脈）

- **同じ更新リクエストが複数回来ても、結果が1回分に収まる性質**
- モバイル/Webの再送・連打・自動リトライ対策として必須

### 3.2 適用範囲

- 対象：**全更新系API**
  - POST / PUT / PATCH / DELETE
- GETは対象外

### 3.3 スコープ

- **member単位**
  - `(memberId, idempotencyKey)` を一意制約

### 3.4 TTL

- **7日**（確定）
- 期限切れレコードは Worker で削除

### 3.5 保存するレスポンス（最小）

- 方針：**最小**
- 保存内容（WriteResult）：
  - `id`（作成/更新対象のID）
  - `serverNow`
  - `warningIds?`
  - `statusCode`（HTTP整合用、推奨）

### 3.6 サーバ側処理フロー（仕様）

1. 更新系APIで `Idempotency-Key` 必須チェック
2. `(memberId, key)` で Idempotencyテーブル検索
   - 存在 → **保存済み結果をそのまま返す**
3. 存在しない → DBトランザクション開始
4. 実処理（CRUD + Outbox生成）
5. WriteResult を Idempotencyテーブルに保存
6. commit → 結果返却

※ 実処理とIdempotency保存は **同一トランザクション**

---

## 4. Notification Outbox（通知一意性）

### 4.1 要件

- 同一イベント / 同一受信者へ **1回だけ通知**

### 4.2 DB担保方式（確定）

- `NotificationOutbox` にユニーク制約：
  - `(type, sourceEntityId, recipientMemberId)`
- INSERT時は `ON CONFLICT DO NOTHING`
- Idempotencyが抜けても **通知だけは二重にならない**

### 4.3 Worker役割

- 未送信Outboxを取得
- Push送信
- 成功 → `sentAt` 更新
- 失敗 → `tryCount` / `error` 更新（再試行）

---

## 5. OCR（Tesseract）設計【確定】

### 5.1 方針

- v1.0：**Tesseract自前実装のみ**
- OCRは **オンデマンド（リクエスト時）**
- OCRサービスは **Python HTTPサービス**

### 5.2 キャッシュ戦略（確定）

- **同一画像は再OCRしない**
- キー：
  - `imageHash`（sha256推奨）
- 保存先：
  - `Receipt.ocrImageHash`
  - `Receipt.ocrRawText`
  - `Receipt.ocrStatus`

### 5.3 OCR API挙動

`POST /receipts/{id}/ocr`：

1. Receipt → Asset 取得
2. 画像から `imageHash` 計算
3. 同一 `imageHash` かつ `ocrStatus=SUCCEEDED` が存在
   - → **Tesseractを呼ばず、保存済み結果を返す**
4. なければ
   - Tesseract実行
   - 結果をReceiptに保存
   - 結果返却

### 5.4 将来拡張（vNext前提）

- 精度不足なら GPT/VLM 採用
- 前処理として **レシート部分トリミング**
- API I/F は維持（engine差し替え）

---

## 6. Schedule（Googleカレンダー連携）

### 6.1 方針（確定）

- Google同期結果を **DBにキャッシュ**
- APIは外部APIを叩かず **DBから返す**

### 6.2 APIビュー

- `/schedules/self`
  - 本人の詳細（title, time, description 等）
- `/schedules/family`
  - 家族向け：**date + hasEvent のみ**

### 6.3 判定ルール

- タイムゾーン：**JST固定**
- 日付単位で hasEvent を計算

---

## 7. Assets（v1.0運用）

### 7.1 方針

- **メタ作成 + ローカル保存**
- 返却値は **クライアントが参照可能なパス（URL）**

### 7.2 保存内容

- `localPath`
- `mimeType`
- `sizeBytes`
- `sha256?`

※ vNextでS3/GCS/署名URLへ移行可能な形

---

## 8. Receipts / Expenses（統合）

- 支出は **receiptsに統合**
- `spentAt`：`date (YYYY-MM-DD)`
- `totalAmount`：**int（円）**
- `deletedAt != null` は集計から除外

---

## 9. 集計・判定の共通ルール（再確認）

- DB保存時刻：UTC
- API返却：UTC ISO8601
- 集計・判定：
  - **JST固定**
  - 週区切り：**月曜開始**
- 在庫期限：
  - `expiryDate = null` は対象外
  - 閾値は `Settings.inventoryExpiryThresholdDays`（default 3日）

---

## 10. 現在地と次の工程

### 現在地

- OpenAPI前提：確定
- DBスキーマ：確定
- Idempotency / Outbox / OCR / Schedule：確定

### 次に進む工程（チェックリスト5）

**Home集約APIの詳細設計**

- 集計を「都度クエリ」で行う（v1.0想定）
- Prismaクエリ設計
- JST/date前提の集計ロジック
- Syncとの整合（serverNow / lastSyncedAt）

---
