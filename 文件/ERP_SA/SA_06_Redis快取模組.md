# 零售業簡易ERP系統 - Redis 快取模組

**文件版本**: v1.0
**建立日期**: 2025-12-31
**所屬專案**: Ejo ERP Demo System
**前置文件**: [SA_00_主文件.md](SA_00_主文件.md)

---

## 目錄

1. [模組概述](#1-模組概述)
2. [架構設計](#2-架構設計)
3. [功能規格](#3-功能規格)
   - 3.1 [JWT Token 黑名單](#31-jwt-token-黑名單-f06-001)
   - 3.2 [使用者在線狀態](#32-使用者在線狀態-f06-002)
   - 3.3 [API 速率限制](#33-api-速率限制-f06-003)
   - 3.4 [熱門資料快取](#34-熱門資料快取-f06-004)
   - 3.5 [分散式鎖](#35-分散式鎖-f06-005)
   - 3.6 [即時通知推播](#36-即時通知推播-f06-006)
   - 3.7 [操作紀錄佇列](#37-操作紀錄佇列-f06-007)
4. [Redis Key 命名規範](#4-redis-key-命名規範)
5. [API 端點](#5-api-端點)
6. [效能考量](#6-效能考量)

---

## 1. 模組概述

### 1.1 模組說明

Redis 快取模組負責系統中所有與記憶體快取、即時狀態管理、非同步處理相關的功能。透過 Redis 的高效能特性，提升系統整體效能並實現分散式環境下的狀態同步。

### 1.2 設計目標

| 目標 | 說明 |
|------|------|
| **效能提升** | 減少資料庫查詢負載，加速 API 回應時間 |
| **安全強化** | 實現 Token 黑名單機制，強化登出安全性 |
| **流量控管** | 防止 API 濫用，保護系統穩定性 |
| **即時性** | 提供即時通知與狀態更新能力 |
| **削峰填谷** | 透過佇列機制平滑處理高峰流量 |

### 1.3 使用的 Redis 資料結構

| 資料結構 | 用途 | 應用功能 |
|---------|------|---------|
| **String** | 單一值儲存 | Token 黑名單、API 計數器 |
| **Set** | 無序集合 | 在線使用者列表 |
| **Hash** | 雜湊表 | 使用者 Session 資訊 |
| **List** | 佇列 | 操作紀錄暫存 |
| **Sorted Set** | 有序集合 | 排行榜、過期資料管理 |
| **Pub/Sub** | 發布訂閱 | 即時通知推播 |

---

## 2. 架構設計

### 2.1 系統架構圖

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Application Layer                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Auth Filter  │  │ Rate Limiter │  │ Cache Aspect │               │
│  │ (黑名單檢查)  │  │  (速率限制)   │  │  (快取切面)   │               │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘               │
│         │                 │                 │                        │
│         └─────────────────┼─────────────────┘                        │
│                           │                                          │
│                           ▼                                          │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                     RedisService                             │    │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │    │
│  │  │TokenBlacklist│ │OnlineTracker│ │DistributeLock│            │    │
│  │  └─────────────┘ └─────────────┘ └─────────────┘            │    │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐            │    │
│  │  │ CacheService│ │NotifyService│ │ AuditQueue  │            │    │
│  │  └─────────────┘ └─────────────┘ └─────────────┘            │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                           │                                          │
└───────────────────────────┼──────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         Redis Server                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │
│  │ DB 0    │ │ DB 1    │ │ DB 2    │ │ DB 3    │ │ DB 4    │       │
│  │ Token   │ │ Session │ │ Cache   │ │ Queue   │ │ Pub/Sub │       │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 資料流程

```
使用者登出流程:
┌────────┐    ┌────────────┐    ┌─────────────┐    ┌───────┐
│ Client │───>│ Auth API   │───>│ BlacklistSvc│───>│ Redis │
└────────┘    └────────────┘    └─────────────┘    └───────┘
                                       │
                                       ▼
                               Token 加入黑名單
                               設定 TTL = Token 剩餘有效時間

API 請求驗證流程:
┌────────┐    ┌────────────┐    ┌─────────────┐    ┌───────┐
│ Client │───>│ JWT Filter │───>│ BlacklistSvc│───>│ Redis │
└────────┘    └────────────┘    └─────────────┘    └───────┘
                   │                   │
                   │              檢查黑名單
                   │                   │
                   ▼                   ▼
              Token 有效？         存在於黑名單？
                   │                   │
                 Yes/No              Yes/No
                   │                   │
                   └───────────────────┘
                            │
                     ┌──────┴──────┐
                     │ 有效且不在   │
                     │ 黑名單中     │
                     └─────────────┘
                            │
                            ▼
                      允許存取 API
```

---

## 3. 功能規格

### 3.1 JWT Token 黑名單 (F06-001)

#### 功能說明

當使用者登出時，將其 JWT Token 加入 Redis 黑名單，確保已登出的 Token 無法再次使用。這是 JWT 無狀態認證的標準安全實作。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-001-01 | Token 加入黑名單時，TTL 設定為該 Token 的剩餘有效時間 |
| BR06-001-02 | 每次 API 請求需檢查 Token 是否在黑名單中 |
| BR06-001-03 | Token 過期後自動從黑名單移除（Redis TTL 機制）|
| BR06-001-04 | 管理員可強制將特定使用者的所有 Token 加入黑名單 |

#### Redis 資料結構

```
Key Pattern: token:blacklist:{tokenHash}
Type: String
Value: userId
TTL: Token 剩餘有效秒數

範例:
SET token:blacklist:a1b2c3d4e5f6 "1" EX 3600
```

---

### 3.2 使用者在線狀態 (F06-002)

#### 功能說明

追蹤系統中目前在線的使用者，供 Dashboard 顯示「目前在線人數」，並可查詢特定使用者的在線狀態。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-002-01 | 使用者登入成功後加入在線列表 |
| BR06-002-02 | 使用者登出或 Token 過期後從在線列表移除 |
| BR06-002-03 | 使用者每次 API 請求更新最後活動時間 |
| BR06-002-04 | 超過 30 分鐘無活動視為離線，自動移除 |

#### Redis 資料結構

```
# 在線使用者集合
Key: online:users
Type: Set
Members: userId

# 使用者 Session 資訊
Key Pattern: session:user:{userId}
Type: Hash
Fields:
  - loginTime: 登入時間
  - lastActiveTime: 最後活動時間
  - ip: 登入 IP
  - userAgent: 瀏覽器資訊
TTL: 1800 (30分鐘)

範例:
SADD online:users 1 2 3
HSET session:user:1 loginTime "2026-01-10T10:00:00" lastActiveTime "2026-01-10T10:30:00" ip "192.168.1.100"
EXPIRE session:user:1 1800
```

#### API 回應範例

```json
GET /api/v1/system/online-status

{
  "success": true,
  "data": {
    "onlineCount": 3,
    "users": [
      {
        "userId": 1,
        "username": "admin",
        "name": "系統管理員",
        "loginTime": "2026-01-10T10:00:00",
        "lastActiveTime": "2026-01-10T10:30:00"
      }
    ]
  }
}
```

---

### 3.3 API 速率限制 (F06-003)

#### 功能說明

限制單一使用者或 IP 在特定時間窗口內的 API 請求次數，防止惡意攻擊或程式錯誤導致的過量請求，保護系統穩定性。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-003-01 | 預設限制：每分鐘 60 次請求 |
| BR06-003-02 | 登入 API 限制：每分鐘 5 次（防暴力破解）|
| BR06-003-03 | 匯出 API 限制：每小時 10 次（防資源濫用）|
| BR06-003-04 | 超過限制回傳 HTTP 429 Too Many Requests |
| BR06-003-05 | 回應 Header 包含剩餘次數與重置時間 |

#### 限制配置

| API 類型 | 時間窗口 | 最大請求數 | 說明 |
|---------|---------|-----------|------|
| 一般 API | 1 分鐘 | 60 | 預設限制 |
| 登入 API | 1 分鐘 | 5 | 防暴力破解 |
| 匯出 API | 1 小時 | 10 | 防資源濫用 |
| 報表 API | 1 分鐘 | 10 | 運算密集 |

#### Redis 資料結構

```
# 滑動窗口計數器
Key Pattern: ratelimit:{type}:{identifier}:{window}
Type: String (計數器)
TTL: 時間窗口秒數

範例:
# 使用者 1 的一般 API 限制（每分鐘）
INCR ratelimit:api:user:1:202601101030
EXPIRE ratelimit:api:user:1:202601101030 60

# IP 的登入限制
INCR ratelimit:login:ip:192.168.1.100:202601101030
EXPIRE ratelimit:login:ip:192.168.1.100:202601101030 60
```

#### HTTP 回應 Header

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1704873600

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1704873600
Retry-After: 30
```

---

### 3.4 熱門資料快取 (F06-004)

#### 功能說明

將頻繁存取的資料快取至 Redis，減少資料庫查詢負載。包含商品資訊、分類樹狀結構、下拉選單選項等。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-004-01 | 資料更新時自動清除對應快取 |
| BR06-004-02 | 快取未命中時從資料庫載入並寫入快取 |
| BR06-004-03 | 不同資料類型設定不同 TTL |
| BR06-004-04 | 記錄 Cache Hit/Miss 比率供監控 |

#### 快取配置

| 快取類型 | TTL | 說明 |
|---------|-----|------|
| products | 30 分鐘 | 商品資訊 |
| categories | 1 小時 | 分類樹狀結構 |
| suppliers | 1 小時 | 供應商列表 |
| units | 1 天 | 計量單位 |
| tax_types | 1 天 | 稅別 |
| customer_levels | 1 天 | 會員等級 |
| system_params | 1 天 | 系統參數 |
| inventory | 5 分鐘 | 庫存（變動頻繁）|

#### Redis 資料結構

```
# 單一商品快取
Key Pattern: cache:product:{productId}
Type: String (JSON)
TTL: 1800

# 商品列表快取（分頁）
Key Pattern: cache:products:page:{page}:size:{size}
Type: String (JSON)
TTL: 300

# 分類樹快取
Key: cache:categories:tree
Type: String (JSON)
TTL: 3600

# 下拉選單快取
Key Pattern: cache:dropdown:{type}
Type: String (JSON Array)
TTL: 86400

範例:
SET cache:product:1 '{"id":1,"sku":"PROD001","name":"商品A"...}' EX 1800
SET cache:categories:tree '[{"id":1,"name":"飲料","children":[...]}]' EX 3600
```

#### 快取統計 API

```json
GET /api/v1/system/cache-stats

{
  "success": true,
  "data": {
    "products": {
      "hits": 1250,
      "misses": 45,
      "hitRate": "96.5%",
      "size": 156,
      "memoryUsage": "2.3 MB"
    },
    "categories": {
      "hits": 890,
      "misses": 12,
      "hitRate": "98.7%",
      "size": 1,
      "memoryUsage": "15 KB"
    }
  }
}
```

---

### 3.5 分散式鎖 (F06-005)

#### 功能說明

在分散式環境下，確保關鍵操作的原子性與互斥性。例如：訂單編號產生、庫存扣減等需要避免併發衝突的操作。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-005-01 | 使用 Redis SETNX 實現互斥鎖 |
| BR06-005-02 | 鎖必須設定 TTL，防止死鎖 |
| BR06-005-03 | 只有持有鎖的執行緒可以釋放鎖 |
| BR06-005-04 | 支援可重入鎖（同一執行緒可多次獲取）|
| BR06-005-05 | 獲取鎖失敗時支援重試機制 |

#### 應用場景

| 場景 | 鎖名稱 | TTL | 說明 |
|------|--------|-----|------|
| 訂單編號產生 | lock:order:number | 5秒 | 防止重複編號 |
| 庫存扣減 | lock:inventory:{productId}:{warehouseId} | 10秒 | 防止超賣 |
| 會員編號產生 | lock:member:number | 5秒 | 防止重複編號 |
| 報表產生 | lock:report:{type}:{userId} | 60秒 | 防止重複產生 |

#### Redis 資料結構

```
# 分散式鎖
Key Pattern: lock:{resource}
Type: String
Value: lockId (UUID + ThreadId)
TTL: 依場景設定

範例:
SET lock:order:number "uuid-123-thread-1" NX EX 5
```

---

### 3.6 即時通知推播 (F06-006)

#### 功能說明

透過 Redis Pub/Sub 機制實現即時通知功能，搭配 WebSocket 將訊息推送至前端。適用於新訂單通知、庫存警示等場景。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-006-01 | 通知訊息透過 Redis Channel 發布 |
| BR06-006-02 | 各 Application Instance 訂閱對應 Channel |
| BR06-006-03 | 收到訊息後透過 WebSocket 推送至前端 |
| BR06-006-04 | 支援廣播（所有使用者）與定向（特定使用者）通知 |

#### 通知類型

| 通知類型 | Channel | 說明 |
|---------|---------|------|
| 新訂單 | notify:order:new | 有新訂單進來 |
| 庫存警示 | notify:inventory:low | 庫存低於安全水位 |
| 系統公告 | notify:system:broadcast | 系統廣播訊息 |
| 個人訊息 | notify:user:{userId} | 發送給特定使用者 |

#### Redis 資料結構

```
# Pub/Sub Channel
Channel: notify:{type}

# 訊息格式 (JSON)
{
  "id": "msg-uuid-001",
  "type": "NEW_ORDER",
  "title": "新訂單通知",
  "content": "訂單 ORD202601100001 已建立",
  "data": {
    "orderId": 1,
    "orderNo": "ORD202601100001",
    "amount": 1500.00
  },
  "targetUsers": [],  // 空陣列表示廣播
  "createdAt": "2026-01-10T10:30:00"
}

範例:
PUBLISH notify:order:new '{"id":"msg-001","type":"NEW_ORDER"...}'
PUBLISH notify:user:1 '{"id":"msg-002","type":"PERSONAL"...}'
```

#### WebSocket 端點

```
WebSocket URL: ws://localhost:8005/ws/notifications

# 連線時需帶 Token
ws://localhost:8005/ws/notifications?token={accessToken}

# 推送訊息格式
{
  "event": "notification",
  "data": {
    "id": "msg-001",
    "type": "NEW_ORDER",
    "title": "新訂單通知",
    "content": "訂單 ORD202601100001 已建立",
    "createdAt": "2026-01-10T10:30:00"
  }
}
```

---

### 3.7 操作紀錄佇列 (F06-007)

#### 功能說明

將操作日誌（Audit Log）先暫存至 Redis List，再由背景程序批次寫入資料庫。實現「削峰填谷」，避免高峰期間資料庫寫入壓力過大。

#### 業務規則

| 規則代碼 | 規則說明 |
|---------|---------|
| BR06-007-01 | 操作日誌先 LPUSH 至 Redis List |
| BR06-007-02 | 背景排程每 5 秒批次處理佇列 |
| BR06-007-03 | 每次最多處理 100 筆記錄 |
| BR06-007-04 | 佇列長度超過 10000 筆觸發告警 |
| BR06-007-05 | 處理失敗的記錄移至 Dead Letter Queue |

#### 架構流程

```
                    ┌─────────────────────────────────────┐
                    │           Application               │
                    │  ┌───────────────────────────────┐  │
                    │  │      AuditLogAspect           │  │
                    │  │   (攔截需要記錄的操作)          │  │
                    │  └───────────────┬───────────────┘  │
                    │                  │                  │
                    │                  ▼                  │
                    │  ┌───────────────────────────────┐  │
                    │  │    AuditQueueService          │  │
                    │  │    (推送至 Redis Queue)        │  │
                    │  └───────────────┬───────────────┘  │
                    └──────────────────┼──────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                          Redis                                   │
│  ┌─────────────────┐              ┌─────────────────┐           │
│  │ audit:log:queue │              │ audit:log:dead  │           │
│  │   (待處理佇列)    │   ─────>    │  (Dead Letter)  │           │
│  │   LPUSH / BRPOP │   處理失敗    │                 │           │
│  └────────┬────────┘              └─────────────────┘           │
│           │                                                      │
└───────────┼──────────────────────────────────────────────────────┘
            │
            │ 背景排程 (每 5 秒)
            │ BRPOP 批次取出
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AuditLogProcessor                             │
│              (批次寫入資料庫，每次最多 100 筆)                      │
└─────────────────────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Database                                   │
│                   audit_logs Table                               │
└─────────────────────────────────────────────────────────────────┘
```

#### Redis 資料結構

```
# 待處理佇列
Key: audit:log:queue
Type: List
操作: LPUSH (寫入) / BRPOP (讀取)

# 死信佇列
Key: audit:log:dead
Type: List

# 佇列統計
Key: audit:log:stats
Type: Hash
Fields:
  - totalPushed: 累計推送數
  - totalProcessed: 累計處理數
  - totalFailed: 累計失敗數
  - lastProcessTime: 最後處理時間

# 訊息格式
{
  "id": "audit-uuid-001",
  "userId": 1,
  "username": "admin",
  "action": "CREATE",
  "module": "ORDER",
  "targetId": "1",
  "targetType": "Order",
  "oldValue": null,
  "newValue": "{\"orderNo\":\"ORD001\"...}",
  "ip": "192.168.1.100",
  "userAgent": "Mozilla/5.0...",
  "createdAt": "2026-01-10T10:30:00"
}

範例:
LPUSH audit:log:queue '{"id":"audit-001"...}'
BRPOP audit:log:queue 5  # 阻塞等待 5 秒
```

#### 佇列監控 API

```json
GET /api/v1/system/audit-queue-stats

{
  "success": true,
  "data": {
    "queueSize": 45,
    "deadLetterSize": 2,
    "stats": {
      "totalPushed": 15680,
      "totalProcessed": 15633,
      "totalFailed": 2,
      "lastProcessTime": "2026-01-10T10:30:00",
      "avgProcessTime": "0.5ms"
    },
    "health": "HEALTHY"  // HEALTHY / WARNING / CRITICAL
  }
}
```

---

## 4. Redis Key 命名規範

### 4.1 命名規則

| 規則 | 說明 | 範例 |
|------|------|------|
| 使用冒號分隔 | 階層式命名 | `cache:product:1` |
| 使用小寫 | 統一小寫字母 | `token:blacklist` |
| 前綴表示用途 | 便於管理 | `cache:`, `lock:`, `queue:` |
| 避免特殊字元 | 只用英數字和冒號 | ✓ `user:1` ✗ `user@1` |

### 4.2 前綴對照表

| 前綴 | 用途 | 範例 |
|------|------|------|
| `token:` | Token 相關 | `token:blacklist:{hash}` |
| `session:` | Session 相關 | `session:user:{userId}` |
| `online:` | 在線狀態 | `online:users` |
| `cache:` | 資料快取 | `cache:product:{id}` |
| `lock:` | 分散式鎖 | `lock:order:number` |
| `ratelimit:` | 速率限制 | `ratelimit:api:user:{id}` |
| `notify:` | 通知頻道 | `notify:order:new` |
| `audit:` | 審計日誌 | `audit:log:queue` |
| `stats:` | 統計資料 | `stats:cache:hits` |

### 4.3 完整 Key 清單

```
# Token 黑名單
token:blacklist:{tokenHash}

# Session
session:user:{userId}
online:users

# 速率限制
ratelimit:api:user:{userId}:{window}
ratelimit:api:ip:{ip}:{window}
ratelimit:login:ip:{ip}:{window}

# 快取
cache:product:{productId}
cache:products:page:{page}:size:{size}
cache:categories:tree
cache:dropdown:{type}
cache:supplier:{supplierId}
cache:inventory:{productId}:{warehouseId}

# 分散式鎖
lock:order:number
lock:member:number
lock:inventory:{productId}:{warehouseId}
lock:report:{type}:{userId}

# 通知頻道
notify:order:new
notify:inventory:low
notify:system:broadcast
notify:user:{userId}

# 審計佇列
audit:log:queue
audit:log:dead
audit:log:stats

# 統計
stats:cache:{cacheName}
stats:ratelimit:blocked
```

---

## 5. API 端點

### 5.1 系統管理 API

| 方法 | 端點 | 說明 | 權限 |
|------|------|------|------|
| GET | /api/v1/system/online-status | 取得在線狀態 | ADMIN, MANAGER |
| GET | /api/v1/system/cache-stats | 取得快取統計 | ADMIN |
| POST | /api/v1/system/cache/clear | 清除指定快取 | ADMIN |
| POST | /api/v1/system/cache/clear-all | 清除所有快取 | ADMIN |
| GET | /api/v1/system/audit-queue-stats | 取得審計佇列統計 | ADMIN |
| POST | /api/v1/system/audit-queue/reprocess | 重新處理死信 | ADMIN |
| GET | /api/v1/system/rate-limit/status | 取得速率限制狀態 | ADMIN |

### 5.2 認證相關 API（更新）

| 方法 | 端點 | 說明 | 變更 |
|------|------|------|------|
| POST | /api/v1/auth/logout | 使用者登出 | 新增：Token 加入黑名單 |
| POST | /api/v1/auth/force-logout/{userId} | 強制登出使用者 | 新增 |
| GET | /api/v1/auth/sessions | 取得使用者 Session 列表 | 新增 |

---

## 6. 效能考量

### 6.1 連線池配置

```yaml
spring:
  data:
    redis:
      lettuce:
        pool:
          max-active: 20      # 最大連線數
          max-idle: 10        # 最大閒置連線
          min-idle: 5         # 最小閒置連線
          max-wait: 3000ms    # 最大等待時間
```

### 6.2 記憶體估算

| 功能 | 預估 Key 數量 | 平均大小 | 總記憶體 |
|------|-------------|---------|---------|
| Token 黑名單 | ~1,000 | 100 bytes | ~100 KB |
| Session | ~100 | 500 bytes | ~50 KB |
| 快取（商品）| ~10,000 | 2 KB | ~20 MB |
| 快取（其他）| ~1,000 | 1 KB | ~1 MB |
| 審計佇列 | ~1,000 | 500 bytes | ~500 KB |
| **總計** | | | **~25 MB** |

### 6.3 監控指標

| 指標 | 說明 | 警示閾值 |
|------|------|---------|
| redis_connected_clients | 連線數 | > 50 |
| redis_used_memory | 記憶體使用 | > 100 MB |
| redis_keyspace_hits | 快取命中數 | - |
| redis_keyspace_misses | 快取未命中數 | - |
| cache_hit_rate | 快取命中率 | < 80% |
| audit_queue_size | 審計佇列長度 | > 10,000 |

---

## 版本歷史

| 版本 | 日期 | 修改內容 | 作者 |
|------|------|---------|------|
| 1.0 | 2025-12-31 | 初版建立 | Kamesan |

---

**文件結束**
