# RECAST — One Source. Endless Content.

AI content-repurposing SaaS: paste an article/transcript, upload a PDF/TXT, or add a YouTube link, and RECAST writes
platform-native Reels, Shorts, LinkedIn posts, carousels, X threads, blog articles and caption packs. Editable,
saved, credit-metered, workspace-isolated.

**Stack:** React + TypeScript + Vite · Tailwind · shadcn-style components (Radix) · Lucide · Supabase (Postgres, Auth,
Storage, Edge Functions) · Gemini (provider abstraction ready for OpenAI).

---

## 1. Run it

```bash
npm install
cp .env.example .env        # fill in VITE_SUPABASE_URL and VITE_SUPABASE_ANON_KEY
npm run dev                 # http://localhost:8080
npm run build               # type-check + production build
```

Without the two env vars the app shows a "Connect Supabase" screen instead of breaking.

## 2. Set up Supabase

1. Create a project at supabase.com.
2. **Database:** apply `supabase/migrations/20261121000001_init.sql`
   (`supabase link --project-ref <ref> && supabase db push`, or paste into the SQL editor).
   It creates all tables, RLS policies, the signup trigger (profile + personal workspace + starter credits),
   the atomic credit functions, and the private `sources` storage bucket with policies.
3. **Auth → Providers → Email:** keep "Confirm email" on (the app routes unverified users to `/verify-email`).
   **Auth → URL Configuration:** add your site URL and `https://<your-site>/verify-email`, `/reset-password`
   to the redirect allow-list.
4. **Edge Functions:**
   ```bash
   supabase functions deploy generate
   supabase functions deploy youtube-transcript
   ```
5. **Secrets (server-side only):**
   ```bash
   supabase secrets set GEMINI_API_KEY=...            # real AI generation
   supabase secrets set ALLOWED_ORIGIN=https://your-site.example   # lock CORS in production
   # optional
   supabase secrets set GEMINI_MODEL=gemini-2.5-flash
   supabase secrets set DEMO_MODE=true                # ONLY when no AI key: placeholder output, clearly labelled, 0 credits
   supabase secrets set YOUTUBE_TRANSCRIPTS_ENABLED=false   # disable caption lookup
   ```
   `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY` are injected automatically into Edge Functions and are never sent to the browser.

### AI modes
| State | Behaviour |
|---|---|
| `GEMINI_API_KEY` set | Real generation. |
| No key, `DEMO_MODE=true` | Placeholder text built from your source by simple rules, tagged **"DEMO — not AI-generated"** in the UI and data. No credits charged. |
| No key, no demo | Generation refuses with a clear "not configured" message. |

To add OpenAI: implement `AiProvider` in `supabase/functions/_shared/ai.ts` and return it from `getProvider()`.

## 3. How generation works (`supabase/functions/generate`)

1. Authenticates the caller (JWT) and verifies **workspace membership and role** in code (service role is used only after that).
2. Validates the request with Zod (UUIDs, enum platforms/tones, length limits).
3. Creates a `generation_jobs` row; a unique `(workspace_id, idempotency_key)` blocks duplicate submissions, and
   `claim_generation_job()` lets a job be processed exactly once. Stale `processing` jobs are reaped and refunded.
4. Loads the source (reads PDF/TXT from private storage server-side, with a path-prefix check against the workspace).
5. **Charges credits atomically** (`charge_credits`: row lock + ledger insert in one transaction).
6. Calls the AI with a system prompt that treats the source as untrusted data, asks for analysis → key ideas → per-platform
   output as JSON, validates it, fills in any missing formats, and re-writes any asset that copies too much source text.
7. Saves `content_assets`; on **any** failure calls `refund_charge` (idempotent — a charge can be refunded once).
8. Regeneration returns a **proposal only**; the UI shows current vs. new and replaces nothing without confirmation.

## 4. Configuration

* **Prices / plans:** `src/lib/config.ts` (`PLANS`). Display only — there is no checkout.
* **Credit costs and limits:** `app_config` table (`credit_costs`, `limits`, `plan_credits`). The UI reads them; the server enforces them.
  Edit with SQL, e.g. `update app_config set value='{"basic":1,"multi":3,"long_transcript":5,"regeneration":1}' where key='credit_costs';`

## 5. Security model

* RLS enabled on every table; access goes through `is_workspace_member()` / `has_workspace_role()` (SECURITY DEFINER, no recursion).
* Roles: owner / admin / editor / viewer. Viewers are read-only everywhere.
* **Clients cannot change credits, plan, ledger, generations, jobs, subscriptions or payments** — those tables are read-only
  to `authenticated` (column-level grants), and the credit functions are executable by `service_role` only.
* Asset inserts are server-only; clients may edit only `title/content/status/metadata`.
* Storage bucket `sources` is private; paths must be `{workspace_id}/{project_id}/{file}` and policies check membership.
* No secrets in frontend code. YouTube lookup only follows `youtube.com` caption URLs (SSRF guard).

## 6. What was verified (and what wasn't)

Verified in the build environment:
* `tsc` strict type-check and production build pass; both Edge Functions pass `deno check`.
* **Database tests against a real PostgreSQL 16** with the migration applied (Supabase `auth`/`storage` stubbed): 50+ checks —
  cross-workspace reads/writes denied, credit/plan tampering denied, forged ledger/generation rows denied, viewer read-only,
  storage path isolation, calendar cross-workspace asset link denied, 5-workspace limit, double-refund is a no-op, and
  **12 parallel 1-credit charges on a 10-credit balance → exactly 10 succeed, balance 0, never negative**.
* Request/output validation, JSON parsing, copy-detection and cost logic unit-tested under Deno.
* Browser run (Playwright) of every public and app page at 1280px and 390px: no page errors, no horizontal scroll;
  logged-out `/app/*` redirects to `/login`; the full Create flow (validation → formats → options → generate → project page,
  edit, regenerate confirmation) was driven end-to-end against a **mocked** Supabase API.

**Not verified — you need to do these against your own project:**
* Real signup/email verification/login against Supabase Auth, and the Edge Functions running on Supabase.
* Real Gemini output quality and latency (needs your `GEMINI_API_KEY`). Free-tier Edge Function wall-clock limits may cap
  very long generations; reduce formats or source length if you see timeouts.
* PDF text extraction on your real documents (scanned/image PDFs are unsupported and report a clear error).
* YouTube captions: best-effort scraping of publicly listed caption tracks. It can break when YouTube changes, and may conflict
  with YouTube's terms in production — consider a licensed transcript provider, or disable it (`YOUTUBE_TRANSCRIPTS_ENABLED=false`).
  When it fails, users are asked to paste the transcript.

## 7. Known limits of this MVP

* No payments: plans are display-only, accounts stay on Free (10 starter credits); `subscriptions`/`payments` tables exist for later.
* No member-invite UI (roles and RLS exist; add members via SQL or build an invite flow). Extra workspaces start with 0 credits.
* Calendar is planning only — no social publishing.
* Generation runs in a single request (progress is a time-based indicator, not streamed).
* No email-sending of notifications; the bell shows recent generation activity.

## 8. Project layout

```
src/
  components/ui/        design-system primitives (button, card, dialog, select, tabs…)
  components/layout/    marketing + app shells, top bar, route guards
  components/studio/    source step (paste / upload with progress / YouTube)
  components/project/   editable AssetCard, status badge
  contexts/             auth, workspace, theme
  pages/{marketing,auth,app}/
  lib/                  config, platforms, api client, export helpers
supabase/
  migrations/           schema + RLS + credit functions + storage policies
  functions/generate/   secure AI generation + regeneration
  functions/youtube-transcript/
  functions/_shared/    cors, auth, config, AI provider, prompts, validation, demo
```

## 9. Deploying the frontend

Any static host (Vercel, Netlify, Cloudflare Pages, or Lovable via GitHub import). Build command `npm run build`, output `dist`,
set the two `VITE_` variables, and add an SPA fallback so all paths serve `index.html`.
