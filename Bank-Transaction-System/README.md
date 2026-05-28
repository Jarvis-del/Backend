# 📒 backend-ledger

### A production-grade double-entry ledger API — real money movement with idempotency, MongoDB transactions, and atomic balance tracking.

<br/>



</div>

---

## 🚀 What is this?

**backend-ledger** is a financial ledger microservice built with Node.js, Express v5, and MongoDB. Unlike a simple CRUD accounting app,
it implements a real **double-entry bookkeeping system** — every money transfer creates both a DEBIT ledger entry and a CREDIT ledger entry
inside a single atomic MongoDB session, exactly how production fintech systems work.

Key highlights:
- Account balances are **derived from the ledger** (sum of credits minus debits), not stored directly — making the system auditable and
  tamper-evident
- Every transfer requires an **idempotency key** — safe to retry without double-charging
- All ledger mutations happen inside **MongoDB ACID transactions** — either both entries commit or neither does
- Post-transfer **email notifications** are sent to the sender via Nodemailer

---

## ✨ Features

- 🔐 **JWT Auth** — Register/login with bcrypt (10 rounds) + JWT in HTTP cookie + token blacklist on logout
- 📧 **Email Notifications** — Registration welcome email + transaction confirmation via Nodemailer
- 🏦 **Account Management** — Create accounts per user, derive live balance from ledger
- 💸 **Double-Entry Transfers** — Atomic DEBIT + CREDIT ledger entries in a MongoDB session
- 🔁 **Idempotency** — Duplicate transfer requests are safely detected and handled by status (PENDING / COMPLETED / FAILED / REVERSED)
- 🛡️ **Token Blacklist** — Logged-out JWTs are stored and rejected on subsequent requests
- ⚡ **Express v5** — Native async error handling, no `try/catch` boilerplate in routes

---

## 📁 Project Structure

backend-ledger/
├── server.js                        # HTTP server entry point
├── package.json
├── .gitignore
└── src/
    ├── app.js                       # Express app, middleware, route mounting
    ├── routes/
    │   ├── auth.routes.js           # /api/auth
    │   ├── account.routes.js        # /api/accounts
    │   └── transaction.routes.js    # /api/transactions
    ├── controllers/
    │   ├── auth.controller.js       # register, login, logout
    │   ├── account.controller.js    # create, list, balance
    │   └── transaction.controller.js# createTransaction, createInitialFunds
    ├── models/
    │   ├── user.model.js            # email, name, password (bcrypt), systemUser flag
    │   ├── account.model.js         # user ref, status (ACTIVE), getBalance()
    │   ├── transaction.model.js     # fromAccount, toAccount, amount, idempotencyKey, status
    │   ├── ledger.model.js          # account ref, transaction ref, amount, type (DEBIT/CREDIT)
    │   └── blackList.model.js       # invalidated JWT tokens
    ├── middleware/
    │   └── auth.middleware.js       # verify JWT, check blacklist, attach req.user
    └── services/
        └── email.service.js         # sendRegistrationEmail, sendTransactionEmail

## 📡 API Reference

All responses are JSON. Protected routes require the JWT cookie set on login — pass it automatically via your HTTP client's cookie jar.

### Base URL
```
http://localhost:3000
```

---

### 🔑 Auth — `/api/auth`

#### `POST /api/auth/register`

**Request body**
```json
{
  "name": "Ankur Prajapati",
  "email": "ankur@example.com",
  "password": "secret123"
}
```

**Response `201`** — also sets `Set-Cookie: token=<jwt>` and sends a welcome email
```json
{
  "user": {
    "_id": "6833a1bc7f4e2d001c8a4f21",
    "email": "ankur@example.com",
    "name": "Ankur Prajapati",
    "createdAt": "2025-05-25T10:24:44.312Z",
    "updatedAt": "2025-05-25T10:24:44.312Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Error `422`** — email already registered
```json
{
  "message": "User already exists with email.",
  "status": "failed"
}


#### `POST /api/auth/login`

**Request body**
```json
{
  "email": "ankur@example.com",
  "password": "secret123"
}


**Response `200`** — sets `Set-Cookie: token=<jwt>`
json
{
  "user": {
    "_id": "6833a1bc7f4e2d001c8a4f21",
    "email": "ankur@example.com",
    "name": "Ankur Prajapati",
    "createdAt": "2025-05-25T10:24:44.312Z",
    "updatedAt": "2025-05-25T10:24:44.312Z"
  },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}

**Error `401`** — wrong credentials json
{
  "message": "Email or password is INVALID"
}


#### `POST /api/auth/logout`

Blacklists the current token and clears the cookie. No request body needed.

**Response `200`**
```json
{
  "message": "User logged out successfully"
}


### 🏦 Accounts — `/api/accounts`

> All routes are protected — requires JWT cookie.

#### `POST /api/accounts`

Creates a new ledger account for the authenticated user.

No request body needed.

**Response `201`** json
{
  "account": {
    "_id": "6833a2cc9f3e1d001c7b5a88",
    "user": "6833a1bc7f4e2d001c8a4f21",
    "status": "ACTIVE",
    "createdAt": "2025-05-25T10:31:00.000Z",
    "updatedAt": "2025-05-25T10:31:00.000Z"
  }
}

 All routes are protected — requires JWT cookie.

#### `POST /api/transactions` — Transfer funds

Executes a double-entry transfer. Atomically creates a DEBIT ledger entry on `fromAccount` and a CREDIT ledger entry on `toAccount`
inside a single MongoDB session.


### ❌ Common Error Responses

| Status | Meaning |
|--------|---------|
| 400 | Validation error / bad request body |
| 401 | Missing, expired, or blacklisted JWT |
| 404 | Resource not found |
| 422 | Duplicate resource (e.g. email already registered) |
| 500 | Internal error / transaction failure |


## ⚙️ How It Works

### The 10-Step Transfer Flow
Every call to `POST /api/transactions` goes through this exact sequence:

1. Validate request fields (fromAccount, toAccount, amount, idempotencyKey)
2. Check idempotency key — return early if transaction already exists
3. Verify both accounts are ACTIVE
4. Derive sender's balance from ledger (sum of credits − debits)
5. Check balance ≥ amount
6. Open MongoDB session → startTransaction()
7. Create transaction document with status: PENDING
8. Create DEBIT ledger entry for fromAccount
9. Create CREDIT ledger entry for toAccount
10. Update transaction status → COMPLETED → commitTransaction()
    → Send email notification to sender
If anything fails between steps 6–10, the session is rolled back — no partial ledger entries are written.


### Double-Entry Bookkeeping
Balances are never stored on the account document. Instead, `account.getBalance()` aggregates the ledger:
balance = SUM(ledger.amount WHERE type=CREDIT) − SUM(ledger.amount WHERE type=DEBIT)

This means every balance change is permanently auditable — you can reconstruct an account's full history from the ledger collection
at any point in time.


### Token Blacklist
On logout, the JWT is written to a `blackList` collection. The auth middleware checks this collection on every protected request — even
 a valid, non-expired token is rejected if it appears in the blacklist.


### How Transfer ACtually Works

POST /api/transactions
        │
        ├─ 1. Validate fields (fromAccount, toAccount, amount, idempotencyKey)
        ├─ 2. Check idempotency key → return early if already processed
        ├─ 3. Verify both accounts are ACTIVE
        ├─ 4. getBalance() → aggregate ledger → check sufficient funds
        │
        ├─ 5. mongoose.startSession() → session.startTransaction()
        │       ├─ Create transaction { status: PENDING }
        │       ├─ Create DEBIT ledger entry  (fromAccount)
        │       ├─ [15s simulated processing delay]
        │       ├─ Create CREDIT ledger entry (toAccount)
        │       └─ Update transaction { status: COMPLETED }
        ├─ 6. session.commitTransaction()  ← atomic, all-or-nothing
        │
        └─ 7. Send email notification to sender



## 🧠 Design Decisions

**Idempotency keys on every transfer**
Financial APIs must be safe to retry — a network timeout shouldn't cause a double charge. By requiring a unique idempotencyKey
per transaction, the API can safely return the existing result instead of creating a duplicate entry.

**Balance derived from ledger, not stored**
Storing a `balance` field and updating it on every transaction introduces race conditions. Deriving balance from an append-only
 ledger is atomic, consistent, and fully auditable — the correct approach for any financial system.

**MongoDB ACID transactions for ledger entries**
The DEBIT and CREDIT entries must both succeed or both fail. Using mongoose.startSession() with session.commitTransaction() gives
 us this guarantee, the same pattern used in production fintech services.

**HTTP-only cookies for JWT**
Storing the token in `localStorage` exposes it to XSS. An HTTP-only cookie is inaccessible to JavaScript — the browser sends it
 automatically and a script can never read it.

**Token blacklist on logout**
JWTs are stateless by design — you can't truly "invalidate" one without keeping server-side state. The blacklist collection solves
this: logged-out tokens are rejected even before expiry.

**bcrypt with 10 salt rounds**
Cost factor 10 is the current industry standard — fast enough for real users (~100ms), slow enough to make bulk brute-force attacks
 impractical.
