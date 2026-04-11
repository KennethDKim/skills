---
name: schwab-trading-api
description: "Skill for working on Kenny's personal finance PWA with Schwab Trader API (Accounts and Trading Production) integration. Use this skill whenever the task involves: Schwab accounts, orders, transactions, positions, balances, OAuth2 flow and token refresh, PocketBase hooks for Schwab token refresh or transaction sync, the offline-first sync engine, IndexedDB caching, or budget/transaction contexts. Also trigger when the user mentions schwab trading, brokerage, portfolio, positions, order placement, transaction history, or the finance dashboard — even if they don't name the app explicitly. Do NOT use for: quotes, option chains, price history, movers, market hours, instruments, or the WebSocket streamer (LEVELONE_*, CHART_*, SCREENER_*, BOOK services) — use schwab-market-data-api for those."
---

# Schwab Finance App

Kenny's personal finance PWA: React 19 + TypeScript + Vite + Tailwind CSS 4 + PocketBase.

## Tech Stack

- **Frontend**: React 19, TypeScript, Vite, Tailwind CSS 4, React Router v7, shadcn/ui
- **Backend**: PocketBase at `https://pb.kennys.cloud`
- **Path alias**: `@/*` → `./src/*`
- **Schwab SDK**: `@KennethDKim/schwab-ts`

## Architecture Overview

### Offline-First Sync (Critical Pattern)

Outbox pattern in IndexedDB via `src/lib/syncEngine.ts` + `src/contexts/SyncContext.tsx`:

- **add**: client-generated ID used as PB record ID; 409 = already exists (idempotent)
- **update**: 404 = drop (deleted server-side); coalesces sequential updates
- **delete**: cancels pending add if never synced; 404 = idempotent
- **4xx** = unrecoverable → drop entry + toast + `sync-entry-dropped` event

SWR everywhere: serve IDB cache first, revalidate from PB in background.
Delta pull on reconnect. SSE subscriptions for realtime updates.

### IDB Cache Keys (`src/lib/db.ts`)

```
kennys_tx_{userId}_{YYYY-MM}        — monthly transactions
kennys_budgets_{userId}_{YYYY-MM}   — monthly budgets
kennys_budget_templates_{userId}    — recurring budget templates
kennys_credit_cards_{userId}        — credit cards
kennys_activity_{userId}_p{N}       — activity feed pages
kennys_categories_{userId}
kennys_uncategorized_{userId}
```

### Custom Window Events

Cross-component communication — not obvious from reading any single file:

- `transaction-added` — new tx, triggers cache invalidation
- `remote-transactions-changed` — SSE update (action: create/update/delete)
- `remote-budgets-changed` — SSE budget update
- `remote-credit-cards-changed` — SSE credit card update
- `remote-schwab-changed` — SSE schwab update
- `sync-entry-dropped` — 4xx outbox drop, triggers server re-fetch
- `budgets-month-change` — BudgetsPage notifies App of selected month
- `calendar-day-view-change` — CalendarPage notifies App of day view state

## Schwab Integration

**Gated to user `vepgcpa0x4x85hj` only.**

OAuth2 flow + token auto-refresh handled in PB hooks. Route: `/schwab/callback`.

### OAuth2 Three-Legged Flow

Schwab uses OAuth 2 `authorization_code` grant. **Access token = 30 min; refresh token = 7 days.** After refresh expiry (or user password reset), the full CAG flow must be restarted — refresh alone won't recover.

**Step 1 — Authorize (redirect user to LMS):**
```
GET https://api.schwabapi.com/v1/oauth/authorize
  ?client_id={CLIENT_ID}
  &redirect_uri={CALLBACK_URL}
```
Callback URL must be HTTPS (localhost allowed as `https://127.0.0.1`). After CAG consent, user is redirected to:
```
https://{CALLBACK_URL}/?code={AUTH_CODE}&session={SESSION_ID}
```
Landing page will 404 — the `code` query param is what matters. **URL-decode `code` before using** (e.g., `%40` → `@`).

**Step 2 — Exchange code for tokens:**
```
POST https://api.schwabapi.com/v1/oauth/token
Authorization: Basic {BASE64(client_id:client_secret)}
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code={AUTH_CODE}
&redirect_uri={CALLBACK_URL}
```
Response:
```json
{
  "expires_in": 1800,
  "token_type": "Bearer",
  "scope": "api",
  "refresh_token": "...",   // 7 days
  "access_token": "...",    // 30 min
  "id_token": "..."
}
```

**Step 3 — Call APIs:** `Authorization: Bearer {access_token}`

**Step 4 — Refresh access token (before expiry):**
```
POST https://api.schwabapi.com/v1/oauth/token
Authorization: Basic {BASE64(client_id:client_secret)}
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token={REFRESH_TOKEN}
```
Returns a new access_token (and refresh_token). If refresh fails (401/invalid_grant), fall back to Step 1 — user must re-consent via LMS.

**Implementation notes:**
- Refresh proactively before 30-min expiry; PB cron in `schwab.pb.js` handles this
- Store `refresh_token` encrypted in PB (never expose to client)
- Client Secret stays server-side only (PB hooks) — never bundled into the React app
- `/schwab/callback` route captures `code` → POSTs to PB hook → hook exchanges for tokens

### Schwab Trader API Reference

Base URL: `https://api.schwabapi.com/trader/v1`

#### Accounts

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/accounts/accountNumbers` | Get encrypted account number mappings (must call first — plain text IDs can't be used in URLs) |
| GET | `/accounts` | All linked accounts with balances; pass `?fields=positions` for positions |
| GET | `/accounts/{accountNumber}` | Single account; `accountNumber` is the **encrypted hash**, not plain text |

Account response contains `securitiesAccount` with: `positions[]`, `initialBalances`, `currentBalances`, `projectedBalances`. Account types: MarginAccount or CashAccount.

#### Orders

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/accounts/{accountNumber}/orders` | Orders for account. **Required**: `fromEnteredTime`, `toEnteredTime` (ISO-8601, max 1yr range). Optional: `status`, `maxResults` (default 3000) |
| POST | `/accounts/{accountNumber}/orders` | Place order. Returns 201 + `Location` header with new order URL |
| GET | `/accounts/{accountNumber}/orders/{orderId}` | Get specific order |
| DELETE | `/accounts/{accountNumber}/orders/{orderId}` | Cancel order |
| PUT | `/accounts/{accountNumber}/orders/{orderId}` | Replace order |
| GET | `/orders` | Orders across ALL accounts. Same params as account-specific but max 60-day lookback |
| POST | `/accounts/{accountNumber}/previewOrder` | Preview order — returns projected commission, fees, validation alerts/rejects |

Order object key fields: `session` (NORMAL), `duration` (DAY), `orderType` (MARKET/LIMIT/etc), `orderStrategyType` (SINGLE), `orderLegCollection[]` with `instruction` (BUY/SELL), `instrument.symbol`, `quantity`.

Order statuses: AWAITING_PARENT_ORDER, AWAITING_CONDITION, ACCEPTED, WORKING, REJECTED, CANCELED, FILLED, EXPIRED, etc.

#### Transactions

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/accounts/{accountNumber}/transactions` | All transactions. **Required**: `startDate`, `endDate` (ISO-8601, max 1yr), `types`. Max 3000 results |
| GET | `/accounts/{accountNumber}/transactions/{transactionId}` | Single transaction |

Transaction types: TRADE, RECEIVE_AND_DELIVER, DIVIDEND_OR_INTEREST, ACH_RECEIPT, ACH_DISBURSEMENT, CASH_RECEIPT, CASH_DISBURSEMENT, ELECTRONIC_FUND, WIRE_OUT, WIRE_IN, JOURNAL, MEMORANDUM, MARGIN_CALL, MONEY_MARKET, SMA_ADJUSTMENT.

#### User Preferences

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/userPreference` | Returns accounts list, streamer WebSocket info, market data permissions |

StreamerInfo contains `streamerSocketUrl` for real-time market data via WebSocket.

### Key API Patterns

1. **Always call `/accounts/accountNumbers` first** to get encrypted hashes — use `hashValue` for all subsequent calls
2. All date params are ISO-8601: `yyyy-MM-dd'T'HH:mm:ss.SSSZ`
3. All error responses share the same shape: `{ message: string, errors: string[] }`
4. Correlation ID returned in `Schwab-Client-CorrelId` header on every response

### PocketBase Hooks (Schwab)

Located at `/mnt/nas/configs/pocketbase/pb_hooks/`:

- `schwab.pb.js` — OAuth2, token refresh, transaction proxy, cron refresh
- Route handlers use `e.app` (NOT `$app`), `e.auth`, `e.requestInfo().body`
- `$app` is ONLY valid inside `cronAdd` callbacks

## Key Gotchas

- **setState in effects**: Never call synchronously. Use `startTransition(() => setState(...))` for async data-fetching effects.
- **React Compiler lint rule active** (`react-hooks/react-compiler`)
- **DashboardPage** does its own data fetching — does NOT use TransactionContext. Uses same IDB cache keys directly.
- **Recurring Budgets**: Template record (`recurring=true`, `month=null`) + instance records per month linked by `template_id`. Cascade delete removes template + all instances.
- **Login Form**: Uses native `<input>` (not Base UI) with `autocomplete` for iOS Bitwarden autofill.
- **MonthlySpendChart**: Pre-allocates 36 months of bar slots to prevent iOS momentum scroll jank.

## Mobile & PWA

- Safe-area insets: `env(safe-area-inset-*)` throughout
- `touch-action: manipulation` on `<html>`
- Sidebar: swipe-to-open from 15px left-edge gesture zone
- Custom service worker: `src/sw.ts` via VitePWA `injectManifest`
- Right-click context menu disabled globally

## Routes (`src/App.tsx`)

`/` DashboardPage, `/calendar`, `/settings`, `/quickadd` (overlay), `/bank-accounts`, `/budgets`, `/credit-cards`, `/expenses`, `/activity`, `/schwab/callback` (OAuth2), `/verify-confirm/{token}`

Auth routes wrapped in `ProtectedLayout` → `LogoutProvider > SyncProvider > TransactionProvider > BudgetProvider > CreditCardProvider`
