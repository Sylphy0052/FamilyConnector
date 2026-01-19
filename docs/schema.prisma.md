generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

enum AssetKind {
  image
}

enum ReactionType {
  LIKE
  OK
  THANKS
}

enum ReceiptOcrStatus {
  NONE
  REQUESTED
  SUCCEEDED
  FAILED
}

enum WarningType {
  CONFLICT_LAST_WRITE_WINS
}

enum NotificationType {
  MESSAGE_NEW
  INVENTORY_EXPIRING
  SHOPPING_REMIND
  SCHEDULE_REMIND
  RECEIPT_ADDED
}

model Household {
  id        String   @id @default(uuid()) @db.Uuid
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  // Relations
  members   Member[]
  settings  Settings?
}

model Member {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  version     Int      @default(1)
  deletedAt   DateTime?

  // Supabase subject (user id) for mapping JWT -> member
  authSub     String   @unique

  household   Household @relation(fields: [householdId], references: [id])

  // Relations
  messagesSent Message[] @relation("MessageSender")
  reads        MessageRead[]
  reactions    Reaction[]
  themes       Theme[]
  userMedia    UserMedia[]
  devices      Device[]
}

model Settings {
  id        String   @id @default(uuid()) @db.Uuid
  householdId String @unique @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  inventoryExpiryThresholdDays Int @default(3)

  household Household @relation(fields: [householdId], references: [id])
}

model Message {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  senderMemberId String @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  body      String

  household Household @relation(fields: [householdId], references: [id])
  sender    Member    @relation("MessageSender", fields: [senderMemberId], references: [id])
  reads     MessageRead[]
  reactions Reaction[]

  @@index([householdId, updatedAt])
  @@index([householdId, deletedAt])
}

model MessageRead {
  // composite key: messageId + memberId
  messageId String @db.Uuid
  memberId  String @db.Uuid

  readAt    DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)

  message Message @relation(fields: [messageId], references: [id])
  member  Member  @relation(fields: [memberId], references: [id])

  @@id([messageId, memberId])
  @@index([memberId])
}

model Reaction {
  messageId String @db.Uuid
  memberId  String @db.Uuid
  type      ReactionType

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)

  message Message @relation(fields: [messageId], references: [id])
  member  Member  @relation(fields: [memberId], references: [id])

  @@id([messageId, memberId, type])
  @@index([memberId])
}

model InventoryItem {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  name     String
  category String?

  lots InventoryLot[]

  @@index([householdId, updatedAt])
  @@index([householdId, deletedAt])
}

model InventoryLot {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  itemId      String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  quantity  Float
  unit      String?
  expiryDate DateTime? @db.Date
  note      String?

  item InventoryItem @relation(fields: [itemId], references: [id])

  @@index([householdId, updatedAt])
  @@index([householdId, expiryDate])
  @@index([householdId, deletedAt])
}

model ShoppingItem {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  name     String
  quantity Float?
  note     String?
  doneAt   DateTime?

  @@index([householdId, updatedAt])
  @@index([householdId, doneAt])
  @@index([householdId, deletedAt])
}

model Receipt {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  spentAt    DateTime @db.Date
  totalAmount Int
  storeName  String?
  assetId    String? @db.Uuid

  ocrStatus  ReceiptOcrStatus @default(NONE)
  ocrRawText String?
  ocrImageHash String?

  asset Asset? @relation(fields: [assetId], references: [id])

  @@index([householdId, updatedAt])
  @@index([householdId, spentAt])
  @@index([householdId, deletedAt])
}

model Theme {
  id        String   @id @default(uuid()) @db.Uuid
  householdId String @db.Uuid
  memberId  String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  name      String
  isActive  Boolean  @default(false)
  themeJson Json

  member Member @relation(fields: [memberId], references: [id])

  @@index([memberId, updatedAt])
  @@index([memberId, isActive])
}

model UserMedia {
  id        String   @id @default(uuid()) @db.Uuid
  householdId String @db.Uuid
  memberId  String   @db.Uuid
  assetId   String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  type      String   @default("image")

  member Member @relation(fields: [memberId], references: [id])
  asset  Asset  @relation(fields: [assetId], references: [id])

  @@index([memberId, updatedAt])
  @@index([memberId, deletedAt])
}

model Asset {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  kind      AssetKind
  localPath String
  mimeType  String
  sizeBytes Int
  sha256    String?

  @@index([householdId, updatedAt])
  @@index([householdId, deletedAt])
}

model Device {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  memberId    String   @db.Uuid

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  platform  String // "ios" | "web"
  deviceId  String?
  token     String? // APNs
  endpoint  String? // WebPush
  p256dh    String?
  auth      String?

  member Member @relation(fields: [memberId], references: [id])

  @@index([memberId, updatedAt])
  @@index([memberId, deletedAt])
  // Upsert constraints
  @@unique([memberId, deviceId])
  @@unique([endpoint])
}

model Schedule {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  memberId    String   @db.Uuid // owner

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  version   Int      @default(1)
  deletedAt DateTime?

  startAt   DateTime
  endAt     DateTime
  title     String?
  location  String?
  description String?

  member Member @relation(fields: [memberId], references: [id])

  @@index([memberId, updatedAt])
  @@index([memberId, startAt])
}

model Warning {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  createdAt DateTime @default(now())
  type      WarningType

  sourceEntityType String
  sourceEntityId   String @db.Uuid
  message          String?

  @@index([householdId, createdAt])
}

model IdempotencyKey {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid
  memberId    String   @db.Uuid

  key         String   // Idempotency-Key header
  createdAt   DateTime @default(now())
  expiresAt   DateTime

  // Store the result to replay (minimal; can be extended)
  responseStatus Int
  responseBody   Json

  @@unique([memberId, key])
  @@index([householdId, createdAt])
}

model NotificationOutbox {
  id          String   @id @default(uuid()) @db.Uuid
  householdId String   @db.Uuid

  type        NotificationType
  sourceEntityId String @db.Uuid
  recipientMemberId String @db.Uuid

  createdAt   DateTime @default(now())
  lastTriedAt DateTime?
  sentAt      DateTime?
  tryCount    Int      @default(0)
  error       String?

  @@unique([type, sourceEntityId, recipientMemberId])
  @@index([householdId, createdAt])
}
