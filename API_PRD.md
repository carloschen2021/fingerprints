# API Procedure Requirement Document

**Project:** Browser Fingerprint Tracking System
**Version:** 1.0
**Date:** 2026-02-28
**Backend:** Node.js + Express.js | **Database:** SQLite3

---

## 1. Overview

This system provides a REST API for browser fingerprint collection, user authentication, and identity matching. All endpoints return `application/json`. Session state is maintained via HTTP cookies (`fingerprint.sid`).

---

## 2. Global Conventions

| Item | Detail |
|---|---|
| Base URL | `http://localhost:3000` |
| Content-Type | `application/json` |
| Auth mechanism | Session cookie (`fingerprint.sid`) |
| Session duration | 24 hours (default), 30 days (remember me) |
| Cookie flags | `httpOnly: true`, `sameSite: lax` |
| Error format | `{ "error": "<message>" }` |

**Standard HTTP status codes used:**

| Code | Meaning |
|---|---|
| `200` | Success |
| `400` | Bad request / validation failure |
| `401` | Unauthenticated |
| `500` | Internal server error |

---

## 3. Authentication Endpoints

### 3.1 `GET /api/captcha`

Generates a math CAPTCHA question and stores the answer in the server session.

**Authentication required:** No

**Request:** _(no body)_

**Processing steps:**
1. Generate two random integers (1–10) and a random operator (`+`, `-`, `*`)
2. Compute the answer (subtraction is absolute-valued to ensure positive result)
3. Store answer in `req.session.captchaAnswer`
4. Force session save before responding
5. Return only the question string — never the answer

**Response `200`:**
```json
{
  "question": "7 × 3 = ?",
  "timestamp": 1709123456789
}
```

**Response `200` (session save failed):**
```json
{
  "question": "7 × 3 = ?",
  "timestamp": 1709123456789,
  "warning": "Session 保存失敗，驗證可能不穩定"
}
```

**Response `500`:**
```json
{ "error": "無法生成驗證碼" }
```

**Notes:**
- CAPTCHA answer is single-use; it is deleted from session after first verification
- Client must call this endpoint before every register or login attempt
- CAPTCHA expires when the session expires

---

### 3.2 `POST /api/auth/register`

Creates a new user account.

**Authentication required:** No
**Prerequisite:** Must have called `GET /api/captcha` in the same session first

**Request body:**
```json
{
  "username": "alice",
  "password": "secret123",
  "email": "alice@example.com",
  "captcha": "21"
}
```

| Field | Type | Required | Constraints |
|---|---|---|---|
| `username` | string | Yes | Min 3 characters, must be unique |
| `password` | string | Yes | Min 6 characters |
| `email` | string | No | Valid email format, must be unique if provided |
| `captcha` | string | Yes | Must match session CAPTCHA answer |

**Processing steps:**
1. Validate `username` and `password` are present
2. Validate email format if provided (regex)
3. Verify `captcha` against `req.session.captchaAnswer`
4. Delete `captchaAnswer` from session (single-use)
5. Check username length ≥ 3 chars
6. Check password length ≥ 6 chars
7. Query DB: reject if username already exists
8. Query DB: reject if email already exists (if provided)
9. `bcrypt.hash(password, saltRounds=10)`
10. `INSERT INTO accounts (username, email, password_hash)`
11. Return success with new user ID

**Response `200`:**
```json
{
  "success": true,
  "message": "註冊成功！",
  "userId": 5
}
```

**Error responses:**

| Status | Condition | Message |
|---|---|---|
| `400` | Missing username or password | `請填寫所有欄位` |
| `400` | Invalid email format | `Email 格式不正確` |
| `400` | No CAPTCHA provided | `請完成驗證碼` |
| `400` | Session has no CAPTCHA answer | `驗證碼已過期，請重新載入` |
| `400` | Wrong CAPTCHA answer | `驗證碼錯誤，請重試` |
| `400` | Username too short | `使用者名稱至少需要 3 個字元` |
| `400` | Password too short | `密碼至少需要 6 個字元` |
| `400` | Username taken | `使用者名稱已存在` |
| `400` | Email taken | `Email 已被使用` |
| `500` | DB error | `系統錯誤` / `註冊失敗` |

---

### 3.3 `POST /api/auth/login`

Authenticates a user and creates an authenticated session.

**Authentication required:** No
**Prerequisite:** Must have called `GET /api/captcha` in the same session first

**Request body:**
```json
{
  "username": "alice",
  "password": "secret123",
  "captcha": "21",
  "rememberMe": false
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `username` | string | Yes | Accepts username or email |
| `password` | string | Yes | |
| `captcha` | string | Yes | |
| `rememberMe` | boolean | No | Extends cookie to 30 days |

**Processing steps:**
1. Validate `username` and `password` are present
2. Verify `captcha` against `req.session.captchaAnswer`
3. Delete `captchaAnswer` from session
4. `SELECT * FROM accounts WHERE username = ? OR email = ?`
5. If no user found → `401`
6. `bcrypt.compare(password, user.password_hash)`
7. If mismatch → `401`
8. Set `req.session.userId` and `req.session.username`
9. Set cookie `maxAge`: 30 days if `rememberMe`, else 24 hours
10. `UPDATE accounts SET last_login = NOW WHERE id = ?`
11. Return user info

**Response `200`:**
```json
{
  "success": true,
  "message": "登入成功！",
  "user": {
    "id": 5,
    "username": "alice",
    "email": "alice@example.com"
  }
}
```

**Error responses:**

| Status | Condition | Message |
|---|---|---|
| `400` | Missing fields | `請輸入使用者名稱/Email 和密碼` |
| `400` | No CAPTCHA | `請完成驗證碼` |
| `400` | Session missing CAPTCHA | `驗證碼已過期，請重新載入` |
| `400` | Wrong CAPTCHA | `驗證碼錯誤，請重試` |
| `401` | User not found | `使用者名稱/Email 或密碼錯誤` |
| `401` | Wrong password | `使用者名稱/Email 或密碼錯誤` |
| `500` | DB error | `登入失敗` |

---

### 3.4 `POST /api/auth/logout`

Destroys the current session.

**Authentication required:** No (safe to call when not logged in)

**Request:** _(no body)_

**Processing steps:**
1. Call `req.session.destroy()`
2. Return success

**Response `200`:**
```json
{ "success": true, "message": "登出成功！" }
```

**Response `500`:**
```json
{ "error": "登出失敗" }
```

---

### 3.5 `GET /api/auth/me`

Returns current session user information.

**Authentication required:** No (returns `loggedIn: false` if not authenticated)

**Request:** _(no body)_

**Processing steps:**
1. Check `req.session.userId` — if absent, return `loggedIn: false`
2. `SELECT id, username, created_at, last_login FROM accounts WHERE id = ?`
3. Return user record

**Response `200` (authenticated):**
```json
{
  "loggedIn": true,
  "user": {
    "id": 5,
    "username": "alice",
    "createdAt": "2026-01-15T08:30:00.000Z",
    "lastLogin": "2026-02-28T10:00:00.000Z"
  }
}
```

**Response `200` (not authenticated):**
```json
{ "loggedIn": false }
```

---

## 4. Fingerprint Endpoints

### 4.1 `POST /api/fingerprint`

Core endpoint. Receives multi-layered fingerprint data, then branches based on login state.

**Authentication required:** No (behavior differs by auth state)

**Request body:**
```json
{
  "visitorId": "abc123xyz",
  "confidence": { "score": 0.92, "comment": "high" },
  "version": "4.6.2",
  "components": { "canvas": {}, "webgl": {}, "...": "..." },
  "clientId": "1709123456_k3x9m_Mozilla",
  "canvas": "data:image/png;base64,...",
  "webgl": { "vendor": "Google Inc.", "renderer": "ANGLE", "extensions": [] },
  "audio": { "sampleRate": 44100, "fingerprint": "0.12345" },
  "fonts": { "available": ["Arial", "Helvetica", "..."], "count": 42 },
  "plugins": { "list": [], "count": 3 },
  "hardware": { "cores": 8, "memory": 16, "touchPoints": 0 },
  "collectionTime": 1240
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `visitorId` | string | **Yes** | From FingerprintJS V4 |
| `confidence` | object | No | `{ score, comment }` |
| `version` | string | No | FingerprintJS version |
| `components` | object | No | Raw FingerprintJS component map |
| `clientId` | string | No | localStorage UUID |
| `canvas` | string | No | Base64 PNG dataURL |
| `webgl` | object | No | GPU info + extensions |
| `audio` | object | No | AudioContext properties |
| `fonts` | object | No | Available font list |
| `plugins` | object | No | Browser plugin list |
| `hardware` | object | No | CPU / memory / touch |
| `collectionTime` | number | No | Collection duration (ms) |

**Processing — Branch A: Logged-in user (`req.session.userId` present)**

1. Query: `SELECT FROM fingerprints WHERE linked_user_id = ?`
2. **Record exists:**
   - Load stored fingerprint data
   - Run `calculateMultiFingerprintSimilarity(oldData, newData)`
   - `UPDATE fingerprints SET visitor_id, components, last_seen WHERE linked_user_id = ?`
   - Return similarity score; flag `fingerprintChanged: true` if similarity < 90%
3. **No record:**
   - `INSERT INTO fingerprints (..., linked_user_id = userId)`
   - Return `isNewUser: true`

**Response `200` (existing record updated):**
```json
{
  "isNewUser": false,
  "userId": 12,
  "similarity": 94.2,
  "message": "已登入用戶 alice 的指紋已更新",
  "fingerprintChanged": false,
  "isLoggedIn": true
}
```

**Response `200` (first fingerprint for user):**
```json
{
  "isNewUser": true,
  "userId": 12,
  "message": "已登入用戶 alice 的指紋已存儲",
  "isLoggedIn": true
}
```

**Processing — Branch B: Guest user (no session)**

1. `SELECT all FROM fingerprints LEFT JOIN accounts`
2. For each stored record: run `calculateMultiFingerprintSimilarity()`
3. Filter results with similarity > 0, sort descending, take top 5
4. Return matches where top similarity ≥ 20%; otherwise return empty
5. **Guest fingerprint is NOT stored in the database**

**Response `200` (matches found):**
```json
{
  "isNewUser": true,
  "similarity": 87.3,
  "topMatches": [
    { "id": 5, "username": "alice", "fingerprintId": 12, "similarity": 87.3 },
    { "id": 7, "username": "bob",   "fingerprintId": 15, "similarity": 42.1 }
  ],
  "message": "找到 2 個相似用戶：\n1. 用戶alice: 87.3%\n2. 用戶bob: 42.1%",
  "isGuest": true
}
```

**Response `200` (no matches):**
```json
{
  "isNewUser": true,
  "similarity": 0,
  "topMatches": [],
  "message": "完全新的訪客，沒有找到相似的指紋",
  "isGuest": true
}
```

**Error responses:**

| Status | Condition |
|---|---|
| `400` | `visitorId` missing |
| `500` | DB query or update failure |

---

### 4.2 Similarity Algorithm — `calculateMultiFingerprintSimilarity()`

Called internally by `POST /api/fingerprint`. Weighted calculation:

| Signal | Weight | Comparison Method |
|---|---|---|
| FingerprintJS V4 components | **40%** | Per-component JSON value match; important components weighted 70% within |
| Canvas fingerprint | **20%** | Exact hash match (binary: 100% or 0%) |
| WebGL fingerprint | **15%** | vendor + renderer + version + extension overlap |
| Audio fingerprint | **10%** | Exact fingerprint hash; partial match on sampleRate |
| Font fingerprint | **10%** | Jaccard-style array overlap |
| Hardware fingerprint | **5%** | cores + memory + touchPoints exact match |
| Custom fingerprint | **5%** | Screen dimensions + timezone + locale |

**FingerprintJS component tiers:**

| Tier | Components | Behavior |
|---|---|---|
| Important (70% weight) | `canvas`, `webgl`, `audio`, `fonts`, `screenResolution`, `hardwareConcurrency`, `deviceMemory`, `platform` | Full score on match |
| Volatile (partial) | `viewport`, `timezone` | 0.5 score if mismatched |
| Ignored | `domBlockers`, `sessionStorage`, `localStorage`, `indexedDB` | Skipped entirely |

**Thresholds:**

| Threshold | Value | Usage |
|---|---|---|
| Guest match display | ≥ 20% | Show in top-5 results |
| Fingerprint changed warning | < 90% | `fingerprintChanged: true` in response |
| `/api/identify` match | ≥ 70% | Report `likelyUser` |

---

### 4.3 `GET /api/fingerprints`

Returns all stored fingerprint records with linked account info.

**Authentication required:** No

**Request:** _(no body)_

**Processing steps:**
1. `SELECT f.*, a.username FROM fingerprints f LEFT JOIN accounts a ON f.linked_user_id = a.id ORDER BY last_seen DESC`
2. Return array

**Response `200`:**
```json
[
  {
    "id": 12,
    "visitor_id": "abc123xyz",
    "confidence_score": 0.92,
    "version": "4.6.2",
    "created_at": "2026-01-15T08:30:00.000Z",
    "last_seen": "2026-02-28T10:00:00.000Z",
    "linked_user_id": 5,
    "username": "alice"
  }
]
```

---

### 4.4 `GET /api/debug/fingerprint/:id`

Returns full fingerprint record including raw component data. Intended for development/debugging.

**Authentication required:** No

**Path parameter:** `id` — fingerprint record ID (integer)

**Processing steps:**
1. `SELECT f.*, a.username FROM fingerprints f LEFT JOIN accounts a WHERE f.id = ?`
2. Parse `components` JSON
3. Return full record with component name list and count

**Response `200`:**
```json
{
  "id": 12,
  "visitor_id": "abc123xyz",
  "username": "alice",
  "components": { "canvas": {}, "webgl": {} },
  "componentNames": ["audio", "canvas", "fonts", "..."],
  "componentCount": 28,
  "canvas_fingerprint": "data:image/png;base64,...",
  "webgl_fingerprint": "{...}",
  "audio_fingerprint": "{...}"
}
```

**Response `404`:**
```json
{ "error": "找不到指紋記錄" }
```

---

## 5. Statistics & Identity Endpoints

### 5.1 `GET /api/stats`

Returns aggregate database statistics.

**Authentication required:** No

**Processing steps:**
1. `SELECT COUNT(*), AVG(confidence_score), COUNT(DISTINCT linked_user_id) FROM fingerprints`
2. Format and return

**Response `200`:**
```json
{
  "totalFingerprints": 142,
  "totalLinkedUsers": 38,
  "averageConfidence": "87.4"
}
```

---

### 5.2 `GET /api/identify?visitorId=<id>`

Attempts to identify which registered user a visitor most likely is, based on FingerprintJS component similarity alone.

**Authentication required:** No

**Query parameter:** `visitorId` (string)

**Processing steps:**

**If logged in:**
1. Return current session user immediately (no DB lookup needed)

**If guest:**
1. `SELECT FROM fingerprints WHERE visitor_id = ?` — find current visitor's stored record
2. `SELECT FROM fingerprints INNER JOIN accounts` — get all linked records
3. For each linked record: run `calculateFingerprintJSSimilarity()` (simplified, components only)
4. Return best match if similarity ≥ 70%

**Response `200` (logged in):**
```json
{
  "loggedIn": true,
  "user": { "id": 5, "username": "alice" }
}
```

**Response `200` (likely match found):**
```json
{
  "loggedIn": false,
  "likelyUser": {
    "userId": 5,
    "username": "alice",
    "similarity": 83
  },
  "message": "新使用者，最可能是 alice (83%)"
}
```

**Response `200` (no match):**
```json
{
  "loggedIn": false,
  "message": "新使用者，無法關聯到已知用戶"
}
```

**Response `200` (no visitorId provided):**
```json
{ "loggedIn": false, "message": "沒有指紋資料" }
```

---

## 6. Database Schema Reference

### `fingerprints`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `visitor_id` | TEXT UNIQUE | FingerprintJS V4 visitor hash |
| `confidence_score` | REAL | 0.0 – 1.0 |
| `confidence_comment` | TEXT | |
| `version` | TEXT | FingerprintJS version |
| `components` | TEXT | JSON blob of all components |
| `client_id` | TEXT | localStorage UUID |
| `canvas_fingerprint` | TEXT | Base64 PNG |
| `webgl_fingerprint` | TEXT | JSON blob |
| `audio_fingerprint` | TEXT | JSON blob |
| `fonts_fingerprint` | TEXT | JSON blob |
| `plugins_fingerprint` | TEXT | JSON blob |
| `hardware_fingerprint` | TEXT | JSON blob |
| `collection_time` | INTEGER | ms |
| `created_at` | DATETIME | Auto |
| `last_seen` | DATETIME | Updated on each visit |
| `linked_user_id` | INTEGER FK | → `accounts.id`, nullable |

### `accounts`

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | Auto-increment |
| `username` | TEXT UNIQUE | Min 3 chars |
| `email` | TEXT UNIQUE | Optional |
| `password_hash` | TEXT | bcrypt, 10 salt rounds |
| `created_at` | DATETIME | Auto |
| `last_login` | DATETIME | Updated on login |

---

## 7. API Endpoint Summary

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/api/captcha` | No | Generate math CAPTCHA |
| `POST` | `/api/auth/register` | No | Register new account |
| `POST` | `/api/auth/login` | No | Login + create session |
| `POST` | `/api/auth/logout` | No | Destroy session |
| `GET` | `/api/auth/me` | No | Get current session user |
| `POST` | `/api/fingerprint` | No* | Submit fingerprint data |
| `GET` | `/api/fingerprints` | No | List all fingerprint records |
| `GET` | `/api/debug/fingerprint/:id` | No | Full record for debugging |
| `GET` | `/api/stats` | No | Aggregate statistics |
| `GET` | `/api/identify` | No* | Identify likely user by fingerprint |

\* Behavior changes depending on whether session is authenticated
