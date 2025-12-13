# GitHub Activity Leaderboard

A production‑ready **Next.js 14** app that builds a leaderboard from GitHub org activity (PRs opened/merged, issues, reviews), caches results in **Upstash Redis**, and serves a beautiful UI with optional client auto‑refresh.

> Use this README to evaluate and run the project locally, compare repos, and deploy. Everything you need—env vars, setup steps, testing tricks—is below.

---

## ✨ Features
- **Server‑first** data fetching with an **API route** that aggregates GitHub activity
- **Fast caching strategy**: *fresh* window + *serve‑stale* with background rebuilds
- **Upstash Redis** integration (automatic if env vars present; in‑memory fallback locally)
- **Configurable periods**: `week`, `month`, `year`
- **Debug endpoints** to verify health, timestamps, and force rebuilds
- **Optional client auto‑refresh** (polls status and refreshes UI when cache updates)
- **Zero vendor lock‑in** for storage: works without Redis in local dev

---

## 🧱 Tech Stack
- **Next.js 14+ / React 18** (App Router, RSC)
- **TypeScript**
- **Upstash Redis** for cache (or in‑memory fallback)
- **GitHub REST API v3**
- **Tailwind CSS**

---

## 📁 Project Structure (relevant parts)
```
app/
  leaderboard/
    [period]/
      page.tsx              # Server Component – renders leaderboard
      AutoRefresh.tsx       # Client Component (optional)
      AutoRefreshWrapper.tsx# Client wrapper for dynamic import
  api/
    leaderboard/
      [period]/route.ts     # API: builds & caches leaderboard
lib/
  config.ts                 # Org/config helpers (optional)
```

---

## 🧪 Scoring Model (default)
| Event          | Points |
|----------------|--------|
| PR opened      | 2      |
| PR merged      | 5      |
| Issue opened   | 1      |
| Review added   | 1      |

> Tweak in `route.ts`: `const scores = { prOpened: 2, prMerged: 5, issueOpened: 1, review: 1 }`.

---

## 🕒 Caching Model
- **Fresh window (TTL):** default `CACHE_TTL_SECONDS=3600` (1h)
- **Serve stale window:** default `STALE_TTL_SECONDS=86400` (24h)
- While stale **but not expired**, API serves cached data **and kicks a background rebuild**.
- If expired (older than stale window), the API builds **before** responding.

**Testing override (optional):** You can temporarily set `week` to 1‑minute TTL / 5‑minute stale for quick iteration. See *Testing Tips* below.

---

## ⚙️ Requirements
- **Node.js 18+**
- A **GitHub token** (classic or fine‑grained) with `public_repo` scope (read‑only access is sufficient for public orgs)
- (Recommended) **Upstash Redis** database

---

## 🔐 Environment Variables
Create `.env.local` in project root:

```ini
# Which GitHub org to scan (default: CircuitVerse)
GITHUB_ORG=YourOrgName

# Personal Access Token (PAT). For public orgs, low-scope token is OK.
# Never commit this; store securely in Vercel/Secrets.
GITHUB_TOKEN=ghp_your_token_here

# Cache freshness and staleness (seconds)
CACHE_TTL_SECONDS=3600      # 1 hour fresh
STALE_TTL_SECONDS=86400     # 24 hours serve-stale

# Upstash Redis (optional locally, recommended in prod)
UPSTASH_REDIS_REST_URL=https://us1-awlong-url.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxxxx:yyyyy
```

> **Security**: Do not commit real tokens. Use Vercel Environment Variables for production.

---

## 🚀 Quick Start (Local)
```bash
# 1) Install deps
pnpm i     # or npm i / yarn

# 2) Add .env.local (see above)

# 3) Run dev server
pnpm dev   # or npm run dev / yarn dev

# 4) Visit the UI
http://localhost:3000/leaderboard/week
```

If **Upstash** env vars are missing, the API automatically falls back to **in‑memory** cache.

---

## 🧰 API Reference
Base: `/api/leaderboard/[period]` where `[period]` ∈ `week | month | year`

### 1) Get leaderboard data
```
GET /api/leaderboard/week
```
**Response**
```json
{
  "period": "week",
  "updatedAt": 1734012345678,
  "entries": [
    {
      "username": "octocat",
      "name": "Mona",
      "avatar_url": "https://...",
      "total_points": 8,
      "breakdown": {
        "PR opened": { "count": 1, "points": 2 },
        "PR merged": { "count": 1, "points": 5 },
        "Review": { "count": 1, "points": 1 }
      }
    }
  ]
}
```

### 2) Lightweight status (no rebuild triggers)
```
GET /api/leaderboard/week?head=1
```
**Response**
```json
{
  "period": "week",
  "updatedAt": 1734012345678,
  "cache": "fresh | stale | miss",
  "nextRefreshAt": 1734015945678,
  "staleUntil": 1734102345678,
  "store": "upstash | memory"
}
```

### 3) Force a rebuild (testing only)
```
GET /api/leaderboard/week?force=1&debug=1
```
- Rebuilds immediately and returns new payload; also updates cache timestamps.

### 4) Probe GitHub rate limits
```
GET /api/leaderboard/week?probe=1
```

### 5) Ping/Health
```
GET /api/leaderboard/week?ping=1
```

### 6) (Optional) Test overrides for org and window
```
GET /api/leaderboard/week?org=YourOrg&sinceHours=6&force=1&debug=1
```
- `org` — temporarily scan a different org without redeploying
- `sinceHours` — limit window to recent hours (e.g. last 6h) for easy testing

> These overrides are safe for local/dev. Gate or remove for production if desired.

---

## 🧭 UI Usage
Navigate to:
- `/leaderboard/week`
- `/leaderboard/month`
- `/leaderboard/year`

Each entry card shows rank, avatar/name, total points, and a per‑activity breakdown.

**Footer** displays: _“Fresh ~1h, served‑stale up to 24h with background refresh.”_ (Based on your env settings.)

---

## 🔄 Optional: Client Auto‑Refresh
If you want the page to update as soon as the server cache refreshes, enable the client poller:

1. Keep `AutoRefresh.tsx` (client) that polls `/api/leaderboard/[period]?head=1` every ~20s.
2. In `[period]/page.tsx` (server), conditionally render the wrapper only when `cache === "stale"`:

```tsx
{/* Auto-refresh (only while stale) */}
{cache === "stale" && (
  <AutoRefresh period={valid} initialUpdatedAt={updatedAt} intervalMs={20000} />
)}
```

This keeps idle pages quiet while still providing fresh data during stale windows.

---

## 🧪 Testing Tips
- **Short TTL for `week`**: Temporarily set inside API route:
  ```ts
  if (period === "week") {
    effectiveTTL = 60;      // 1 minute fresh
    effectiveStale = 300;   // 5 minutes serve-stale
  }
  ```
- **Force rebuild**: `/api/leaderboard/week?force=1&debug=1`
- **Check timestamps**: `/api/leaderboard/week?head=1`
- **Generate activity quickly**: use `?org=YourOrg&sinceHours=1` and make a tiny PR/issue/review
- **Widen scan bounds** (more activity):
  ```ts
  const MAX_SEARCH_PAGES = 3; // was 2
  const REPOS_CAP = 10;       // was 5
  ```

> Note: Larger bounds burn more GitHub API quota. Keep small for demos.

---

## 🧩 Implementation Details
- **Pagination**: Uses GitHub `Link` headers to follow pages up to small caps
- **Events scanned**:
  - Search API for PRs created
  - Search API for PRs merged
  - Search API for Issues created
  - Repository loop to fetch **PR reviews** (bounded by latest repos and recent PRs)
- **Enrichment**: Up to 25 top users are enriched with display name & avatar via `/users/:login`
- **Deterministic sorting**: by `total_points` desc

---

## 📦 Deployment (Vercel)
1. **Import repo** into Vercel
2. Set **Environment Variables** (see `.env.local` above)
3. Ensure **Node.js 18+** runtime
4. Recommended: `export const runtime = "nodejs";` in the API route for Windows stability
5. Deploy

**CI/Previews**: The API uses cache keys per period, so preview deploys can safely reuse the same Upstash instance. For isolation, prefix keys with env name.

---

## 🩺 Troubleshooting
- **Build error: `ssr: false` not allowed in Server Components**
  - Wrap your dynamic client import in a `"use client"` component (see `AutoRefreshWrapper.tsx`).
- **No new data appears**
  - Check `head=1` for `updatedAt` changes
  - Use `force=1` to rebuild
  - Ensure your token has enough scope and API quotas are available (`probe=1`)
  - Increase `MAX_SEARCH_PAGES` / `REPOS_CAP` during tests
- **429 / rate limits**
  - Reduce polling; narrow `sinceHours`; run in off‑peak; increase token trust level
- **Redis not set**
  - App falls back to in‑memory cache for local dev; add Upstash for persistence and cross‑instance cache

---

## 🤝 Contributing
1. Fork the repo
2. Create a feature branch: `git checkout -b feat/xyz`
3. Commit with conventional messages
4. Open a PR

Please keep changes small and add context in PR description. For questions, open a Discussion.

---

## 📜 License
MIT © Contributors

---

## 👥 Maintainers / Contacts
- Primary: _Your Name_ (@your‑handle)
- Co‑maintainer: _Teammate Name_ (@their‑handle)

> If this repo is selected by vote, we’ll maintain a stable main branch, document releases, and respond to issues within 48h.

