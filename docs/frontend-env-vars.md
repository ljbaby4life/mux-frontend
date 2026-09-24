# Frontend environment variables

Reference for every environment variable this Next.js app reads, where it's
validated, and how it affects behavior across local dev, testnet, and
mainnet. The authoritative schema lives in `src/lib/env.ts`; this doc is a
narrative companion to the table in the root `README.md`.

## Quick start

```bash
cp .env.example .env.local
# edit .env.local with real values, then:
pnpm run dev
```

Every variable is optional locally. Leaving `.env.local` empty (or not
creating it at all) still works — `next dev` and `pnpm test` both run
against in-repo mocks (`/api/auth/login`, `/api/wallets`, etc.).

## Variables

### Client-visible (`NEXT_PUBLIC_*`)

These are inlined into the browser bundle at build time. Never put secrets
in a `NEXT_PUBLIC_*` variable.

- **`NEXT_PUBLIC_API_URL`** — canonical backend base URL. Read directly in
  `src/app/api/auth/login/route.ts` to decide whether to proxy to a real
  backend or fall back to the mock login response, and in
  `src/lib/api/config.ts::getApiBaseUrl()` as the **first** candidate for
  all other API calls (e.g. `useWallets`, `GET /api/requests/today`, and
  `POST /api/transactions` for the wallet "Send" flow). **Set this in new
  deploys.**
- **`NEXT_PUBLIC_MUX_API_URL`** — **legacy alias**, checked second in the
  `getApiBaseUrl()` fallback chain (see below). Defaults to
  `https://api.muxprotocol.com` in production when all three aliases are
  unset. Predates `NEXT_PUBLIC_API_URL`; kept for older deploy configs.
- **`NEXT_PUBLIC_API_BASE`** — **legacy alias**, third and final candidate
  in the fallback chain, for deploys that used this older name.

  **API URL resolution chain (#693):** `getApiBaseUrl()` in
  `src/lib/api/config.ts` walks the three aliases above in priority order
  and returns the first *non-empty* value. An alias set to an empty string
  (e.g. `NEXT_PUBLIC_API_URL=`) is treated as unset and the chain
  continues to the next alias. This means a mis-set deploy that blanks the
  primary var still picks up the legacy alias instead of silently falling
  back to mock data. The `API_URL_CANDIDATES` constant exported from
  `config.ts` documents the exact order so tests can verify it without
  reimplementing it. Use `getActiveApiUrlVar()` (also exported from
  `config.ts`) to log which alias is actually in effect at startup.
- **`NEXT_PUBLIC_APP_URL`** — this app's own public URL; defaults to
  `http://localhost:3000`.
- **`NEXT_PUBLIC_WALLET_CONNECT_PROJECT_ID`** — only relevant if
  WalletConnect-based wallet flows are enabled.

There is intentionally no client-visible Mux API key. A project API key
is a real credential, and anything under `NEXT_PUBLIC_*` is inlined into
the browser bundle for every visitor to read — see #636. `ApiContext.tsx`
(a client component) never reads `MUX_API_KEY`/`MUX_API_SECRET`; it only
constructs an unauthenticated client that talks to this app's own
same-origin `/api/*` routes.

### Server-only

These never reach the browser and are safe for secrets.

- **`MUX_API_KEY`** / **`MUX_API_SECRET`** — read by
  `getUpstreamAuthHeaders()` in `src/lib/api/config.ts` and attached
  (`x-api-key` / `x-api-secret`) to every upstream request a Next.js API
  route makes to the Mux backend. Only ever read inside `src/app/api/**`
  route handlers or other server-only modules — **never import
  `getApiKey()`/`getApiSecret()` from a client component** (#694).

  As an extra defence-in-depth measure, `assertServerSide()` and
  `getServerOnlyEnv()` in `src/lib/env.ts` throw at runtime whenever they
  are called from a browser context (`window` is defined), so accidentally
  importing these helpers in a `"use client"` file causes an immediate,
  visible error in development rather than silently returning `undefined`.
  Use `getServerOnlyEnv("MUX_API_SECRET")` in server-only code instead of
  reading `process.env.MUX_API_SECRET` directly.
- **`MUX_BACKEND_URL`** — server-only base URL of `mux-backend`, read by
  `getBackendApiBaseUrl()` in `src/lib/api/config.ts`. `/api/spending-limits`
  proxies `GET`/`PUT` here (forwarding the server API key and any caller
  `Authorization` header) so spending limits and the real `todayUsage`
  counter live in the backend, not the frontend process. No default: when
  unset the route responds `503 { error: "Spending limits backend is not
  configured" }` instead of returning a fabricated figure. The
  `/api/demo/spending-limits` route needs no backend — it derives its
  `todayUsage` from the mock transaction store
  (`computeTodayUsage()` in `src/lib/spending-limits/todayUsage.ts`).

### Implicit

- **`NODE_ENV`** — standard Next.js variable. Gates verbose
  console logging in the analytics/tracking hooks
  (`useAnalytics.ts`, `useAnalyticsMetrics.ts`, `useAnalyticsTracking.ts`,
  `recoveryAnalyticsTracking.ts`, `spendingLimitsTracking.ts`) outside of
  `production`, makes `validateEnv()` in `src/lib/env.ts` throw
  (instead of warn) on missing *required* vars when set to `production`,
  and controls whether `getEnv()` merges in documented defaults (see
  "Production defaults" below — it only does so when `NODE_ENV=production`).

### Production defaults

`getEnv()` merges each var's documented `defaultValue` (from the schema
in `src/lib/env.ts`) into whatever is set, but only when
`NODE_ENV=production`. Concretely: if a production deploy forgets to set
`NEXT_PUBLIC_API_URL`/`NEXT_PUBLIC_MUX_API_URL`, it now resolves to the
documented default `https://api.muxprotocol.com` instead of silently
falling through every API route's mock branch (#637). Local dev and test
runs are untouched — `NODE_ENV` isn't `production`, so leaving vars unset
still exercises the in-repo mocks described throughout this doc.

## Testnet vs. mainnet

Two independent things decide "which network" a request is scoped to:

1. **Which backend** — `NEXT_PUBLIC_API_URL` (or its aliases) points this
   app at a specific Mux backend:

   | Environment | `NEXT_PUBLIC_API_URL` example |
   | --- | --- |
   | Local dev (mocked) | _(unset)_ |
   | Testnet / staging | `https://testnet-api.muxprotocol.com` |
   | Mainnet / production | `https://api.muxprotocol.com` |

2. **Which network within that backend** — the in-app Testnet/Mainnet
   switcher in the top nav (`NetworkContext`, `src/context/NetworkContext.tsx`,
   persisted to `localStorage` under `mux_network`). `useWallets({ network })`
   sends this as a `?network=` query param on `/api/wallets`, so the backend
   itself scopes the response to one network — wallets are not additionally
   re-filtered client-side. (An earlier version of the wallets page *did*
   also run a second, independent client-side "all/testnet/mainnet" filter
   on top of that already-scoped fetch, which could show a false "no
   wallets on this network" empty state whenever it disagreed with the
   in-app switcher. That double-filtering has been removed — see
   `src/app/dashboard/wallets/page.tsx`.)

The wallet rows themselves also carry a per-wallet `network` field
(`"testnet"` \| `"mainnet"`, see `src/types/wallet.ts`) that both the
backend proxy and the mock fallback in `/api/wallets` use to honor that
query param.

## Production never silently serves mock data

`/api/auth/login`, `/api/auth/refresh`, `/api/wallets`,
`/api/wallets/[id]`, `/api/overview`, and `/api/api-keys` (`GET`/`POST`/
`PATCH`) fall back to in-repo mock responses (fake wallets, dashboard
stats, API keys, and a hardcoded mock bearer/refresh token) whenever no
backend URL is configured — that's what lets `pnpm run dev`, CI, and the
`/demo` routes run with no live backend. `isMockFallbackAllowed()`
(`src/lib/api/config.ts`) disables that fallback whenever
`NODE_ENV=production`: those routes return `503 backend_unavailable`
instead. This matters because the mock fallback accepts a hardcoded
bearer token (`mock-access-token`) and refresh token
(`mock-refresh-token`) as valid, and `/api/api-keys` would otherwise
create/list/revoke against a `localStorage`-backed

### Fail-closed env validation (#754)

`isMockFallbackAllowed()` is the runtime gate, but it only checks
`NODE_ENV`. A production deploy that *also* sets a mock-enabling flag
(e.g. `NEXT_PUBLIC_USE_MOCKS=true`, `MUX_USE_MOCKS=1`) or points
`NEXT_PUBLIC_API_URL` at a testnet host would previously start up and
serve mock data anyway. `validateEnv()` in `src/lib/env.ts` now
**fails closed**: when `NODE_ENV=production` it throws a typed
`EnvValidationError` (stable code `ENV_MOCKS_IN_PRODUCTION`) if any
mock-enabling variable is truthy or if the resolved API base URL is a
known testnet host. The error message names the offending variable and
never echoes secret values, so it is safe to surface in logs and CI.

Invariants enforced by `validateEnv()`:

- In `production`, mock flags must be unset/false and the API base URL
  must not be a testnet endpoint. Violations abort startup instead of
  silently serving mocks.
- Outside `production`, mock flags are allowed and the in-repo mocks
  described above remain available for `pnpm run dev`, CI, and `/demo`.
- `getEnv()`/`getServerOnlyEnv()` continue to redact secret values in
  any thrown error; only variable *names* and stable error codes appear.

Negative cases are covered by unit tests in `src/lib/env.test.ts`
(production + mock flag ⇒ throws; non-production + mock flag ⇒ allowed).
