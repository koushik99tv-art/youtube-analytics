# 99TV Analytics Dashboard

Daily analytics command center for 99TV's 30 YouTube channels — tracks real-time video uploads, views, subscriber growth, and manually editable revenue & staff data.

## Run & Operate

- `pnpm --filter @workspace/api-server run dev` — run the API server (port 8080)
- `pnpm --filter @workspace/99tv-dashboard run dev` — run the frontend (port 21670)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from the OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- Required env: `DATABASE_URL` — Postgres connection string
- Required env: `YOUTUBE_API_KEY` — YouTube Data API v3 key
- Required env: `SESSION_SECRET` — secret for signing session cookies (API server throws on startup if missing)
- Optional env: `GOOGLE_CLIENT_ID` + `GOOGLE_CLIENT_SECRET` — Google OAuth credentials for YouTube Analytics API (channel-wide views-during-period)
- Optional env: `YOUTUBE_OAUTH_REDIRECT_URI` — override the OAuth redirect URI (auto-derives from `REPLIT_DOMAINS` if unset)

## Stack

- pnpm workspaces, Node.js 24, TypeScript 5.9
- Frontend: React + Vite (artifacts/99tv-dashboard)
- API: Express 5 (artifacts/api-server)
- DB: PostgreSQL + Drizzle ORM
- Validation: Zod (`zod/v4`), `drizzle-zod`
- API codegen: Orval (from OpenAPI spec)
- Build: esbuild (CJS bundle)
- YouTube Data API v3 for real-time channel stats

## Where things live

- `lib/api-spec/openapi.yaml` — API contract (source of truth)
- `lib/db/src/schema/channels.ts` — channels table (30 99TV channels)
- `lib/db/src/schema/manualEntries.ts` — revenue + staff entries table
- `lib/db/src/schema/subscriberSnapshots.ts` — daily subscriber + lifetime view-count snapshots (subscriber column feeds the now-hidden Subs Added delta; view_count is no longer read by any column)
- `artifacts/api-server/src/routes/analytics.ts` — YouTube API integration + analytics routes
- `artifacts/api-server/src/routes/channels.ts` — channels listing
- `artifacts/99tv-dashboard/src/pages/dashboard.tsx` — main dashboard page
- `artifacts/99tv-dashboard/src/components/inline-edit-cell.tsx` — inline editable cells
- `artifacts/api-server/src/lib/auth.ts` — hardcoded login credentials + verifyCredentials
- `artifacts/api-server/src/middlewares/requireAuth.ts` — gate for protected routes
- `artifacts/api-server/src/routes/auth.ts` — login / logout / me endpoints
- `artifacts/api-server/src/lib/ensureSessionTable.ts` — creates `user_sessions` table at startup
- `artifacts/99tv-dashboard/src/pages/login.tsx` — login page
- `artifacts/99tv-dashboard/src/App.tsx` — AuthGate (gates whole app via /auth/me)

## Authentication

- Session-cookie auth (express-session + connect-pg-simple, cookie name `sid`, 7-day expiry, sessions stored in `user_sessions` Postgres table)
- Two hardcoded users (small internal tool, plaintext by request): `superadmin` / `@stats/99tv/ai.2` and `admin` / `@99tv/stats/ai.1` — defined in `artifacts/api-server/src/lib/auth.ts`
- All `/channels` and `/analytics` routes require auth; `/healthz` and `/auth/*` are public
- Login regenerates the session (fixation protection); logout destroys it and clears the cookie
- Frontend `AuthGate` shows the login page when `/auth/me` returns 401, dashboard otherwise; dashboard header has a Logout button
- `connect-pg-simple` uses `createTableIfMissing: false` because esbuild does not bundle its `table.sql`; `ensureSessionTable()` runs idempotent DDL at startup instead

## Architecture decisions

- YouTube Data API v3 used for real-time data: `playlistItems.list` (uploads playlist) to find videos in range, `videos.list` for per-video views/duration, `channels.statistics` for subscribers
- Date filters are interpreted in **IST (UTC+5:30)**, not UTC — 99TV is an India-based network, so a calendar date means the IST day. `parseDateRange` converts IST day boundaries to UTC instants before comparing to YouTube's UTC publish timestamps
- Shorts vs long videos are classified by probing `youtube.com/shorts/<id>` (200 = Short, 303 = regular), NOT by duration — a 2-min news clip and a 2-min Short both exist. Only videos ≤180s are probed (Shorts max length); the probe uses no API quota
- Revenue (online/offline) and staff names (Anchor Head, Digital Head, AI Head) stored manually in `manual_entries` DB table per channel per date
- Date filtering (Today / Yesterday / This Week / Custom) controlled on the frontend, passed as `startDate`/`endDate` query params
- One row per channel in the analytics table aggregated over the selected date range
- Grand Total row computed client-side from all visible rows

## Product

- Dashboard with all 30 channels showing: S.No., Date, Channel Name, Videos Uploaded, Views, Subscribers Added, Revenue Online, Revenue Offline, Anchor Head, Digital Head, AI Head
- Revenue and staff name cells are inline-editable — click to edit, blur/Enter to save
- Date filters: Today, Yesterday, This Week, Custom (range picker)
- Summary strip at top showing totals across all channels
- Grand Total row at bottom of table

## User preferences

- Real YouTube data only — no mock data
- Revenue fields (online/offline) must be manually entered with ₹ formatting
- Staff names (Anchor Head, Digital Head, AI Head) manually editable per channel per date
- 30 channels total seeded in database

## Gotchas

- Do NOT run `drizzle push` — it tries to DROP the runtime-created `user_sessions` table (data loss) and needs a TTY. Apply additive schema changes with idempotent startup DDL in `artifacts/api-server/src/lib/ensure*.ts` (awaited before `app.listen`), and update the Drizzle schema file for types. For one-off dev changes, run `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` directly.

- YouTube Data API `channels.statistics` returns **cumulative** totals (not daily deltas) — subscriber count and view count are totals at time of query
- "Subs Added" delta (`subscribersAdded`) is still computed server-side (`currentSubscribers - snapshot[startDate - 1 day]`) but the column is hidden in the dashboard UI. Snapshots are written daily for today's date on every analytics fetch.
- "Channel Views" column = total views of all videos (long + shorts) uploaded in the selected range, computed **client-side** as `viewsTotal + shortsViews` from data already in the analytics response (i.e. the "Views" + "Views (Shorts)" columns combined). Pulled directly from YouTube per-video `videos.list` viewCounts — NO snapshot/baseline math. There is no `channelViewsAdded` field anymore (removed from backend response and OpenAPI spec).
- `subscriber_snapshots.view_count` is a nullable **bigint** column (added via idempotent startup DDL `ensureSnapshotViewColumn.ts`) that is still **written** by the analytics fetch + daily snapshot job but is **no longer read** by any column. Harmless leftover; safe to ignore.
- "Views" column = sum of **lifetime** views of videos *uploaded* in the range (matches the public channel page), NOT YouTube Studio's "views during the period" (that needs owner-level Analytics API / OAuth)
- Day boundaries are IST: a date sent as `YYYY-MM-DD` is expanded with `+05:30` offset, so videos uploaded just after midnight IST land on the correct day (UTC boundaries previously shifted them ~5.5h)
- API server route for `/analytics/summary` must come BEFORE `/analytics/:channelId/:date` in Express router to avoid path conflict (already handled — summary is registered first)
- Do not use `zod/v4` as a direct import in `api-server` — it's not a direct dependency; use plain validation helpers instead

## Pointers

- See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details
