# GSTMind Platform — Complete Architectural Analysis

**Date:** 2026-05-12
**Repos:** `agrmadhur123/gstmind-backend` (FastAPI on Railway.app) · `agrmadhur123/gstmind-frontend` (Vanilla JS on GoDaddy)
**Task:** Read-only analysis — no code changes.

---

## 1. High-Level Architecture Summary

```
┌──────────────────────────────────────────────────────────────────────┐
│  EXTERNAL USERS (Indian CAs / Tax Professionals)                     │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ HTTPS
┌─────────────────────────────▼────────────────────────────────────────┐
│  FRONTEND — GoDaddy Hosting (gstmind.in)                             │
│                                                                      │
│  index.html (154KB)              admin.html (83KB)                   │
│  ├── Landing + pricing           ├── Dashboard / analytics           │
│  ├── Auth (email+pw, OTP)        ├── User management (CRUD)          │
│  ├── Chat UI (SSE streaming)     ├── KB document library             │
│  ├── Query usage meter           ├── Drive sync controls             │
│  ├── File attachment picker      ├── System prompt editor            │
│  └── Dark/light theme            └── Subscription management         │
│                                                                      │
│  Auth in localStorage: gm_token, gm_user,                           │
│                         gm_admin_token, gm_admin_email               │
└─────────────────────────────┬────────────────────────────────────────┘
                              │ REST + SSE
                              │ API = 'https://web-production-399b0.up.railway.app'
┌─────────────────────────────▼────────────────────────────────────────┐
│  BACKEND — Railway.app (single Uvicorn process)                      │
│  FastAPI 0.110.0 / Python                                            │
│                                                                      │
│  main.py                                                             │
│  ├── /auth/*       — Login, OTP, admin signup                        │
│  ├── /admin/*      — User CRUD, analytics (revenue calc)             │
│  ├── /settings/*   — Key/value system config                         │
│  └── /payment/*    — Razorpay order create + verify                  │
│                                                                      │
│  routers/query.py  — POST /query  (SSE streaming AI responses)       │
│  routers/kb.py     — /kb/*        (KB document CRUD + reindex)       │
│  routers/sync.py   — /sync/*      (Drive↔Gemini sync + reconcile)   │
│                                                                      │
│  core/drive_sync.py          — 3-way Drive/Gemini/DB reconciliation  │
│  core/retrieval.py           — RetrievalProvider interface           │
│  core/providers/gemini_provider.py — Gemini File Search impl         │
└────┬──────────────┬────────────────┬──────────────┬──────────────────┘
     │              │                │              │
     ▼              ▼                ▼              ▼
┌─────────┐  ┌────────────┐  ┌──────────────┐  ┌─────────────────────┐
│Supabase │  │ Google     │  │ Google       │  │ Resend (OTP email)  │
│Postgres │  │ Gemini     │  │ Drive v3     │  │ Razorpay (payments) │
│via REST │  │ File Search│  │ service acct │  │                     │
│RLS on   │  │ store      │  │ GST_Raw/     │  │                     │
│(bypassed│  │gemini-2.5  │  │ folder tree  │  │                     │
│by svc   │  │            │  │              │  │                     │
│key)     │  │            │  │              │  │                     │
└─────────┘  └────────────┘  └──────────────┘  └─────────────────────┘
```

### Core Query Flow
1. `index.html` → `POST /query` with `Authorization: Bearer <token>`
2. `verify_token()` validates HMAC-SHA256 custom token
3. DB: read `users.query_count` vs `query_limit`; reject if at limit
4. DB: read `settings.system_prompt` (or use hardcoded `DEFAULT_SYSTEM_PROMPT`)
5. Gemini File Search called — metadata filter: `superseded="false"` + year regex
6. Gemini response chunked back to frontend via SSE (`type: meta/text/done/error`)
7. `asyncio.create_task()` fires non-blocking increment + `query_log` upsert

### Core Sync Flow
1. Admin → `POST /sync/run` → FastAPI `BackgroundTasks` queues sync
2. Service account scans Google Drive `ROOT_SYNC_FOLDER_ID` recursively
3. Diff: Drive file IDs vs `gst_documents` table
4. New files: download to `/tmp` → Gemini upload → poll until active → upsert DB
5. Deleted files: remove from Gemini + DB
6. Reconciliation: 3-way check (Drive vs Gemini vs DB)

---

## 2. Dependency Map

### External Services

| Service | Purpose | Env Vars | Failure Impact |
|---|---|---|---|
| Supabase | All persistent data | `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` | Total outage |
| Google Gemini File Search | Doc retrieval + AI response | `GEMINI_API_KEY`, `GEMINI_STORE_NAME`, `GEMINI_QUERY_MODEL` | No queries answerable |
| Google Drive | Document source of truth | `GOOGLE_SERVICE_ACCOUNT_JSON`, `ROOT_SYNC_FOLDER_ID` | Sync fails; existing indexed docs still work |
| Resend | Email OTP delivery | `RESEND_API_KEY`, `FROM_EMAIL` | Falls back to `dev_otp` in response (see SEC-3) |
| Razorpay | Payment orders | `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET` | Upgrade flow broken |
| Railway.app | Backend hosting | — | Full backend outage |
| GoDaddy | Frontend hosting | — | Frontend unavailable |

### Python Packages (`requirements.txt`)

| Package | Version | Usage |
|---|---|---|
| fastapi | 0.110.0 | Web framework |
| uvicorn | 0.27.0 | ASGI server |
| httpx | >=0.28.1 | All outbound HTTP (Supabase, Resend, Razorpay) |
| pydantic | 2.10.6 | Request/response models |
| google-api-python-client | 2.118.0 | Google Drive v3 |
| google-auth | >=2.48.1 | Service account credentials |
| google-genai | 2.0.1 | Gemini File Search |
| pypdf2 | 3.0.1 | Listed as "for merger" — **not used anywhere in current code** |
| python-dotenv | 1.0.0 | .env loading |
| python-multipart | 0.0.9 | File upload parsing |

### Internal Module Dependency Graph

```
main.py
├── imports routers.sync, routers.kb, routers.query
├── defines: supabase_query(), simple_token(), verify_token(),
│            get_current_user(), verify_admin(), send_email_otp()
└── all auth + payment + admin + settings endpoints

routers/query.py
├── DUPLICATES verify_token(), get_current_user() from main.py
├── calls Gemini REST API directly via httpx (bypasses provider abstraction)
└── no imports from core/

routers/kb.py
├── imports core.retrieval.get_provider()
├── DUPLICATES supabase_query() helper
└── blocking time.sleep() inside async route

routers/sync.py
├── imports core.drive_sync.*
├── Depends(lambda: True) — no auth on ANY sync endpoint
└── DUPLICATES supabase_query() helper

core/drive_sync.py
├── imports core.retrieval.get_provider(), IndexResult
├── calls Google Drive API
└── module-level _stop_requested global

core/retrieval.py
└── factory: get_provider() → GeminiFileSearchProvider

core/providers/gemini_provider.py
└── implements RetrievalProvider via google.genai SDK
```

### Schema vs. Runtime Discrepancy

`01_schema.sql` defines: `users`, `query_history`, `query_log`, `subscriptions`, `otp_codes`, `kb_documents`, `settings`

Runtime code actually uses:
- `gst_documents` — real document index (all of drive_sync.py, kb.py, sync.py)
- `sync_runs` — sync history
- `reconciliation_runs` — reconciliation history

**`kb_documents` (in schema) is not referenced anywhere in runtime code.**
Running `01_schema.sql` on a fresh Supabase project produces a broken deployment.

---

## 3. Risks & Problems Found

### CRITICAL — Security

**SEC-1: Hardcoded secrets with insecure defaults** · `main.py`
```python
JWT_SECRET = (os.getenv("JWT_SECRET") or "gstmind-secret-change-this").strip()
ADMIN_SECRET = (os.getenv("ADMIN_SECRET") or "gstmind@admin123").strip()
ADMIN_INVITE_CODE = (os.getenv("ADMIN_INVITE_CODE") or "GSTMIND-ADMIN-2024").strip()
```
If env vars are unset, the app silently uses the published defaults. Anyone reading this public repo can forge tokens or create admin accounts.

**SEC-2: Unauthenticated sync endpoints** · `routers/sync.py`
```python
admin=Depends(lambda: True)  # bypasses verify_admin entirely
```
`POST /sync/run`, `POST /sync/stop`, `POST /sync/reconcile`, `GET /sync/folder-structure` are completely open — no authentication. Any anonymous caller can trigger a full Drive sync or dump Drive folder structure.

**SEC-3: OTP returned in production API response** · `main.py` (all OTP send endpoints)
```python
if not sent:
    resp["dev_otp"] = code
```
`sent = False` whenever `RESEND_API_KEY` is unset or Resend is down. In that scenario, the live OTP is sent in the HTTP response body to any caller.

**SEC-4: Unsalted SHA-256 password hashing** · `main.py`, `login_email()`, `set_password()`
```python
pwd_hash = hashlib.sha256(data.password.encode()).hexdigest()
```
Fast, no salt, GPU-crackable, rainbow-table vulnerable. Should be bcrypt/argon2.

**SEC-5: Settings mutation open to all authenticated users** · `main.py`
`PUT /settings/{key}` uses `Depends(get_current_user)` — any free-plan user can overwrite the system prompt or toggle maintenance_mode. Should require `verify_admin`.

**SEC-6: Non-constant-time token comparison** · `main.py`, `routers/query.py`, `verify_token()`
```python
if sig != expected:  # timing attack
```
Should be `hmac.compare_digest(sig, expected)`.

**SEC-7: Hardcoded test user backdoor** · `main.py`, `routers/query.py`
```python
if user_id == "test-user-000":
    return {..., "plan": "unlimited", "query_limit": 99999}
```
If `JWT_SECRET` is the default (SEC-1), forging a token for `test-user-000` is trivial. Even with proper secrets, this is a permanently open bypass.

**SEC-8: Sync/KB endpoints expose internal document IDs** · `routers/sync.py`, `routers/kb.py`
`GET /sync/documents`, `GET /kb/documents` have no auth. They expose Drive file IDs, Gemini doc IDs, SHA-256 hashes, folder paths for every document in the KB.

**SEC-9: No rate limiting on OTP send** · `main.py`
`/auth/email/otp/send` and `/auth/forgot/send` have no rate limiting. Unlimited OTP spam causes Resend cost abuse and user harassment.

**SEC-10: CORS fully open** · `main.py`
`allow_origins=["*"]` — any domain can make cross-origin requests with user tokens.

---

### HIGH — Reliability

**REL-1: Blocking `time.sleep()` inside async route** · `routers/kb.py`, `mark_superseded()`
`time.sleep(5)` (not `await asyncio.sleep(5)`) blocks the entire event loop for up to 2 minutes during a supersede operation. All concurrent user queries hang.

**REL-2: Query count race condition** · `routers/query.py`
Count is read before streaming starts; increment fires via `create_task` after the stream. Two simultaneous queries both read the same count and both proceed — free users can exceed limits by issuing concurrent requests.

**REL-3: Sync runs in the web server process** · `routers/sync.py`
`BackgroundTasks` runs in the same Uvicorn process. A large sync with hundreds of PDFs (each polling Gemini for up to 5 minutes) blocks async workers and degrades query response times.

**REL-4: Global mutable stop signal** · `core/drive_sync.py`
```python
_stop_requested = False  # module-level global
```
If a sync crashes or Railway restarts mid-sync, the flag may be left `True`, silently preventing all future syncs until the process restarts.

**REL-5: No DB connection pooling** · `main.py`, `core/drive_sync.py`, `routers/kb.py`
Every DB operation opens a new `httpx.AsyncClient`. Under load, hundreds of short-lived HTTPS connections are made to Supabase with no reuse.

**REL-6: Temp file leak on exception** · `routers/kb.py`, `reindex_document()`, `mark_superseded()`
`os.remove(tmp.name)` only runs on the success path. Exceptions leave temp files in `/tmp`, which can fill disk on a long-running instance.

---

### HIGH — Technical Debt

**TD-1: Duplicate code in 3+ files**
- `verify_token()` + `get_current_user()`: defined in `main.py` and `routers/query.py`
- `supabase_query()` helper: in `main.py`, `core/drive_sync.py`, `routers/kb.py` (3 slightly different versions)
- `SUPABASE_HEADERS` dict: duplicated in `main.py`, `core/drive_sync.py`

**TD-2: Dead code — `02_api_server.py` (158KB)**
Never imported or referenced anywhere. Appears to be the original monolithic server before the refactor. Should be deleted.

**TD-3: Provider abstraction bypassed in the most critical path**
`routers/query.py` calls Gemini REST directly via `httpx` instead of using `get_provider()` from `core/retrieval.py`. The entire point of the `RetrievalProvider` interface is bypassed for the primary user-facing query.

**TD-4: `GEMINI_STORE_NAME` hardcoded in 5+ locations**
`"fileSearchStores/gstmindcbic-8iqlexnxmrph"` appears as a default in `routers/query.py`, `core/retrieval.py`, `core/providers/gemini_provider.py`, `core/drive_sync.py`, `routers/kb.py`. A store migration touches all of them.

**TD-5: Revenue calculation ignores payment records**
`admin_analytics()` in `main.py` computes revenue as `sum(999 if plan=="pro" else 1999 ...)` — hardcoded magic numbers, not from the `subscriptions` table. Refunds, failures, and promos are invisible.

**TD-6: `pypdf2` in requirements with no active usage**
Adds install time and attack surface for no benefit in the current codebase.

---

### MEDIUM — Feature Gaps

**GAP-1: Payments not wired end-to-end**
Backend `/payment/create-order` + `/payment/verify` are complete and correct. Frontend has `showToast('Razorpay payment — coming soon')`. The two halves are never connected.

**GAP-2: No subscription expiry enforcement**
`subscriptions.expires_at` column exists but is never checked. One payment grants permanent plan status.

**GAP-3: No email verification on password signup path**
Users can claim any email via `/auth/set-password` without proving ownership.

**GAP-4: File attachment UI not wired to backend**
Frontend has a file picker; `POST /query` accepts `has_attachment: bool`; but the file content is never sent to or processed by the backend/Gemini.

**GAP-5: No scheduled sync**
Comments reference a "daily scraper" but no cron job, Railway scheduled service, or scheduler is configured in the repo.

**GAP-6: No pagination on `/admin/users`**
Returns all users with no `limit`/`offset`. Unbounded at scale.

**GAP-7: Netlify config files present but hosting is GoDaddy**
`netlify.toml` defines security headers (`X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `X-Robots-Tag: noindex` on `/admin.html`) and `_redirects` defines SPA routing (`/*` → `/index.html`). On GoDaddy static hosting, both files are ignored — the security headers are never sent and deep-link navigation to the admin panel will likely 404. The `admin.html` is also no longer protected from search engine indexing.

---

## 4. Missing Documentation

| ID | What's Missing |
|---|---|
| MD-1 | No `.env.example` — 10+ required env vars scattered across 6 files |
| MD-2 | `01_schema.sql` missing `gst_documents`, `sync_runs`, `reconciliation_runs` tables |
| MD-3 | No deployment runbook (Supabase provisioning, Google service account setup, Gemini store creation, first sync) |
| MD-4 | `02_api_server.py` has no explanation — is it active? retired? |
| MD-5 | No documentation on Gemini metadata filter syntax or limitations |
| MD-6 | Query limit source of truth is ambiguous (settings table, hardcoded values in `verify_payment()`, and `admin_signup()` all define limits differently) |
| MD-7 | Admin panel has no help text explaining "supersede vs delete", reconciliation consequences, or bulk delete warnings |
| MD-8 | README endpoint list is out of date (references `/auth/signup/email`, `/auth/otp/send` which don't exist in current code) |

---

## 5. Suggested Improvements

### Priority 1 — Security (Fix Before Production Traffic)

| # | Action | File(s) |
|---|---|---|
| IMP-1 | Fail fast on startup if `JWT_SECRET`/`ADMIN_SECRET`/`ADMIN_INVITE_CODE` match their insecure defaults. Remove fallback string literals. | `main.py` |
| IMP-2 | Apply `verify_admin` to ALL sync and KB mutation endpoints. Replace `Depends(lambda: True)` with `Depends(verify_admin)` at the router level. | `routers/sync.py`, `routers/kb.py` |
| IMP-3 | Remove `dev_otp` from response body entirely (or gate on `ENVIRONMENT != "production"`). | `main.py` |
| IMP-4 | Replace SHA-256 password hashing with `bcrypt`. Add `bcrypt` to `requirements.txt`. Plan migration for existing hashes (detect by hash length, force reset on mismatch). | `main.py` |
| IMP-5 | Replace `if sig != expected:` with `if not hmac.compare_digest(sig, expected):`. | `main.py`, `routers/query.py` |
| IMP-6 | Change `PUT /settings/{key}` to `Depends(verify_admin)`. | `main.py` |
| IMP-7 | Add OTP rate limiting (3 sends per email per 15 min, in-memory dict). | `main.py` |

### Priority 2 — Architecture / Reliability

| # | Action | File(s) |
|---|---|---|
| IMP-8 | Extract `core/auth.py` (single canonical `verify_token`, `get_current_user`, `verify_admin`) and `core/db.py` (single `supabase_query`). Remove all duplicates. | All routers + `main.py` + `drive_sync.py` |
| IMP-9 | Replace `time.sleep(5)` with `await asyncio.sleep(5)` in `mark_superseded()`. | `routers/kb.py` |
| IMP-10 | Fix query count race condition: use a Supabase RPC that atomically increments and checks in one `UPDATE ... WHERE query_count < query_limit RETURNING *`. | `routers/query.py` |
| IMP-11 | Move Drive sync to a separate Railway service or scheduled job, not `BackgroundTasks` in the web process. | `routers/sync.py`, new service |
| IMP-12 | Wrap temp file operations in `try/finally` to guarantee cleanup on exceptions. | `routers/kb.py` |
| IMP-13 | Replace Python-side document text filter with Supabase `ilike` operator: `&doc_name=ilike.*{search}*`. | `routers/kb.py` |
| IMP-14 | Add `limit`/`offset` pagination to `GET /admin/users`. | `main.py` |

### Priority 3 — Feature Completion & Docs

| # | Action | File(s) |
|---|---|---|
| IMP-15 | Wire frontend payment buttons to `/payment/create-order` + `/payment/verify`. Remove `showToast('coming soon')` stub. | `index.html` |
| IMP-16 | Route `routers/query.py` through `get_provider().search_with_response()` instead of direct httpx calls. | `routers/query.py`, `core/providers/gemini_provider.py` |
| IMP-17 | Delete `02_api_server.py` (158KB dead code). | `02_api_server.py` |
| IMP-18 | Remove `pypdf2` from `requirements.txt`. | `requirements.txt` |
| IMP-19 | Update `01_schema.sql` to include `gst_documents`, `sync_runs`, `reconciliation_runs`. Add schema version comment. | `01_schema.sql` |
| IMP-20 | Create `.env.example` with all required vars and descriptions. | new file |
| IMP-21 | Centralise plan limits in `settings` table. Read at runtime in `verify_payment()` and `admin_signup()` instead of hardcoding 100 / 999999. | `main.py` |
| IMP-22 | Add email verification step to the email+password signup path. | `main.py` |

---

## Critical Files

| File | Issues |
|---|---|
| `main.py` | SEC-1,3,4,5,6,7,9, REL-2, TD-5, GAP-1,2,3 |
| `routers/sync.py` | SEC-2,8, REL-3,4, TD-1 |
| `routers/query.py` | SEC-6, REL-2, TD-1,3 |
| `routers/kb.py` | SEC-8, REL-1,6, TD-1, GAP-6 |
| `01_schema.sql` | Schema drift — 3 missing tables (MD-2) |
| `02_api_server.py` | Dead code — delete (TD-2) |
| `index.html` | GAP-1 (payment stub), GAP-4 (attachment), GAP-7 (GoDaddy routing) |
