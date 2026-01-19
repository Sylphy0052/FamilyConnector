# 実装前 最終確定仕様（v1.0 決定版）

本ドキュメントは、これまでの仕様書（A〜G、全体アーキテクチャ、実装仕様、OpenAPI前提、Prisma）および  
追加Q&A（Q1〜Q3）を踏まえ、**実装前に決めるべき事項をすべて確定値で一括整理**したものである。  
本書を前提に、以降の実装では“判断を戻さない”。

---

## 0. 前提・不変ルール（再確認）

- クライアント：**iOS + Web（PC）**
- 利用人数：**2人**
- Household：**単一（v1.0）**
- 権限：**ロールなし**
- 時刻：
  - DB保存・API返却：UTC
  - 判定・集計：JST固定
  - 週区切り：月曜開始
- 削除：**論理削除（deletedAt）**
- 同期：**Delta Sync（updatedAt > lastSyncedAt）**
- 競合：**後勝ち適用 + Warning生成**
- 通知：**同一イベント × 同一受信者 = 1回**
- 通知疲労対策：**抑制ではなく Home(UI)で回収**

---

## 1. 認証・オンボーディング（確定）

### 1.1 認証方式

- 認証基盤：**Supabase Auth**
- ログイン方式：**メール + パスワード**
- API認可：
  - クライアントが Bearer JWT を付与
  - APIでJWT検証 → `memberId / householdId` を確定

### 1.2 Household 作成・参加

- 初回ログイン：
  - **Household を自動作成**
  - owner / admin 概念は持たない
- 2人目参加：
  - **招待コード（英数字8桁）手入力**
  - 有効期限：**24時間**
  - 同時に有効な招待は **最大1件**
- 退出：
  - **退出不可**
  - 必要時は Member を `disabled` にする

### 1.3 追加DBモデル（確定）

- `HouseholdInvite`
  - `householdId`
  - `codeHash`（平文保存しない）
  - `expiresAt`
  - `usedAt`
  - `createdAt`

---

## 2. API 契約・共通ルール（確定）

### 2.1 API 基本

- Base Path：`/api/v1`
- OpenAPI：Nest DTO + Swagger で生成、YAMLをリポジトリ管理
- 共通エラー形式：

  ```json
  {
    "code": "ERROR_CODE",
    "message": "human readable message",
    "details": {},
    "requestId": "uuid"
  }
  ```

### 2.2 冪等性（Idempotency）

- 対象：**全更新系API（POST / PUT / PATCH / DELETE）**
- 表現：**`Idempotency-Key` ヘッダで統一**
- スコープ：`(memberId, Idempotency-Key)`
- TTL：**7日**
- 再送時：
  - **同一 HTTP status**
  - **同一 response body**
  - 副作用（通知・ログ・派生処理）は再発火しない

---

## 3. 同期（Delta Sync）仕様（確定）

### 3.1 Sync API

- Endpoint：`GET /api/v1/sync?since=<timestamp>`
- レスポンス：
  - ドメインごとに配列でまとめ返し
  - **必ず `serverNow`（UTC）を含める**

### 3.2 lastSyncedAt 更新ルール

- クライアントは：

  ```text
  lastSyncedAt = serverNow
  ```

- 最大 `updatedAt` 方式は **採用しない**

---

## 4. 競合・Warning（確定）

- 更新系APIは `version` 必須
- `baseVersion != currentVersion` の場合：
  - 更新は **後勝ちで適用**
  - **Warning レコードを生成**
- Warning の扱い：
  - OS Push 通知はしない
  - **Home の「要対応」ブロックで回収**

---

## 5. 既読管理（ReadState）（確定）

### 5.1 方針

- **タイムライン単位の既読を正**とする
- メッセージ単位既読は v1.0 では採用しない

### 5.2 DBモデル（確定）

- `TimelineReadState`
  - `memberId`（unique）
  - `lastReadAt`
  - `lastReadMessageId`
  - `updatedAt`
  - `version`

### 5.3 未読判定

- 以下のいずれかを満たすものを未読とする：
  - `message.createdAt > lastReadAt`
  - `reply / reaction.createdAt > lastReadAt`

---

## 6. 通知（Outbox / Push）（確定）

### 6.1 通知生成と配信

- 通知生成：各ドメイン（B / C / A）
- 配信：**Worker が Outbox を処理**
- 一意性担保：
  - `(notificationType, sourceEntityId, recipientMemberId)`

### 6.2 NotificationType（最終確定）

- C（メッセージ）
  - `MESSAGE_NEW`
  - `MESSAGE_REPLY`
  - `MESSAGE_REACTION`
- B（スケジュール）
  - `EVENT_FIRE`
  - `BUSY_FIRE`
  - `REMINDER_FIRE`
- A（生活系）
  - `INVENTORY_EXPIRING`
  - `RECEIPT_ADDED`

※ **C通知（新規/返信/リアクション）は全員へ通知**（確定）

### 6.3 Push / 静音

- Push 種別：
  - iOS：APNs
  - Web：Web Push（VAPID）
- 静音（Quiet Hours）：
  - **個人ローカル設定**
  - デフォルト：静音なし
  - 抑制されても Home で回収可能

---

## 7. Push 開発環境到達性（v1.0 確定）

- HTTPS 到達手段：**Cloudflare Tunnel**
- APNs：
  - Auth Key（.p8）方式
  - ローカルでは `.env` 管理（gitignore）
- Web Push：
  - VAPID 鍵を `.env` 管理
  - Service Worker を Web 側で配布

---

## 8. Asset（メディア）管理（確定）

### 8.1 保存・参照

- v1.0 保存先：**ローカルファイル（docker volume）**
- DB 参照：**assetId のみ**
- URL 直持ちは禁止

### 8.2 配信

- **署名付きURL**
  - 有効期限：**10分**
- 軽量版：
  - 長辺：1280px
  - JPEG quality：75
  - 失敗時は原本にフォールバック

---

## 9. OCR（v1.0 確定）

- 実装：**Python HTTP サービス + Tesseract**
- 実行方式：オンデマンド
- キャッシュ：
  - `ocrImageHash` が一致する場合は再実行しない
- 前処理（最小）：
  - 回転補正（EXIF）
  - グレースケール
  - 二値化
- レスポンス：
  - `engine = "tesseract"`
  - `rawText`（必須）
  - `totalCandidates`（任意）

---

## 10. Home（集約API）（確定）

### 10.1 構成（4ブロック）

1. **C：インボックス（未読/未確認）**
2. **A：要対応**
   - 在庫期限接近
   - 出費差分警告
   - Warning
3. **B：時間軸**
   - 今日の event / busy / reminder
4. **クイックアクション**
   - レシート
   - 在庫
   - 買い物
   - リマインド
   - 投稿

### 10.2 優先順位

- **C未読があれば最上段**
- なければ A → B

### 10.3 API返却上限（確定）

- C：最大 10件
- A：最大 10件
- B：最大 20件（当日 + 直近24h）
- Quick：固定

---

## 11. 集計・期限判定（確定）

- 在庫期限接近：
  - `inventoryExpiryThresholdDays`
  - デフォルト：**3日**
- 週集計：
  - JST 月曜 00:00 〜 日曜 23:59:59

---

## 12. バッチ / ジョブ（v1.0 確定）

- 実行基盤：**worker プロセス + node-cron**
- 実行スケジュール（JST）：
  - 通知送信：毎分
  - 在庫 qty=0 削除：毎日 00:10
  - 買い物 done 削除：毎日 00:10
  - 監査 / 削除ログ削除（3ヶ月）：毎日 02:00

※ 「翌日0:00削除」は **翌日中に削除されればOK** とする。

---

## 13. テーマ（パーソナライズ）（確定）

- 可読性判定：
  - WCAG 2.1
  - 通常テキスト：4.5:1
  - Large：3:1
- NG時：
  - テーマ単位でデフォルトへフォールバック
  - 設定画面に理由を1行表示

---

## 14. Prisma スキーマ差分（実装前必須）

### 追加

- `TimelineReadState`
- `HouseholdInvite`

### 変更

- `NotificationType` enum 拡張（上記確定値）

### 非採用

- `MessageRead`（v1.0では利用しない）

---

## 15. 実装開始の前提条件（Definition of Ready）

- 本ドキュメントの内容を **唯一の正**とする
- Prisma 差分を反映済み
- OpenAPI に以下が反映済み：
  - `Idempotency-Key`
  - `serverNow`
  - Home 4ブロック構造
- 実装順：
  1. Phase1：E / G（認証・同期・冪等・Warning）
  2. Phase2：F（Asset）
  3. Phase3：D（Home）
  4. Phase4：C（メッセージ）

---
