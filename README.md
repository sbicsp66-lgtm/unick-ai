# Unick AI — One-Click Samadhan

AI marketing platform for local businesses in India. Phase 1 (foundation) is complete: docs, schema, project setup, auth, app shell and a protected dashboard.

Read first: `docs/01-positioning-and-prd.md`, `docs/02-architecture-and-folder-structure.md`.

## Run it
```bash
npm install
cp .env.example .env.local          # fill NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY, NEXT_PUBLIC_APP_URL
```
1. Create a Supabase project. In the SQL editor run `supabase/migrations/0001_init.sql` (or `supabase db push`).
2. Supabase → Authentication → Providers → enable **Email** and **Google** (add your Google OAuth client ID/secret). Add `http://localhost:3000/auth/callback` and your production URL to Redirect URLs.
3. `npm run dev` → http://localhost:3000
4. To make yourself admin: in the SQL editor run
   `update public.profiles set role = 'admin' where id = (select id from auth.users where email = 'you@example.com');`
5. Optional: `npm run db:types` to generate typed DB helpers.

## Notes
- Google sign-in (Supabase Auth) is separate from connecting a Google Business Profile. The latter uses its own OAuth client with the `business.manage` scope (module 2.3).
- PWA: the service worker registers in production builds only. Test with `npm run build && npm start`.
- shadcn/ui: `components.json` is configured; run `npx shadcn@latest add dialog sheet tabs ...` for more components. Button, Card, Input, Label are already included.

## Phase 2.1: Authentication + Onboarding
1. Run `supabase/migrations/0002_onboarding.sql` after `0001` (adds `locality`, `category_other`, the `replace_business_services` function, and logo limits).
2. Supabase → Authentication → URL Configuration → Redirect URLs: add `<APP_URL>/auth/callback` (needed for Google sign-in, email confirmation and password reset).
3. Flow: sign in → if `profiles.onboarding_completed` is false the app shell sends you to `/onboarding` → 4 steps → `/dashboard`. Progress is saved after steps 1 and 2, so closing the tab resumes where you left off.

## Phase 2.2: AI Growth Dashboard
- Shows **sample data** by default (a "Sample data" tag appears in the header). Set `DASHBOARD_DATA_SOURCE=live` in `.env.local` to read real data from Supabase.
- Preview the empty state: open `/dashboard?demo=empty` (sample mode only).
- Code: data in `src/modules/dashboard/` (types, mock, live queries, one switch in `get-dashboard-data.ts`), UI in `src/components/dashboard/`.
- Channel grouping: Google = `google` + `google_ads`; Meta Ads = `meta_ads` + `facebook` + `instagram`; WhatsApp = `whatsapp`; Other = `website`, `manual`.
- "Today" and "this week" use India time (IST); the week starts on Monday.

## Phase 2.3: Google Business Profile AI Agent (mock)
- Open `/google-business`. First visit shows the connect screen; "Connect" only simulates it (a cookie), no Google call is made.
- Tabs: Overview (health score + SEO description), Reviews, Posts & Photos, Performance. State lives in the browser so actions feel real; it resets on refresh.
- Mock data uses your real onboarding details (name, city, services, description), so the health score and AI text match your business.
- **Going live later:** implement `GbpProvider` and `GbpAi` (see `src/modules/gbp/service.ts`) and return them from `getGbpProvider()` / `getGbpAi()`. UI, actions and health scoring stay as they are. Real Google access needs API approval and a separate OAuth client (`business.manage` scope).
- Health score is calculated in `src/modules/gbp/health.ts` (pure function, 9 weighted checks totalling 100).

## Phase 2.4: AI Creative Studio
- Open `/creative-studio`. Four kinds: Social post, Meta ad, Google ad, Festival / offer (post + Meta ad + WhatsApp message).
- **Real:** the Library. Creatives are saved in `creatives`, publish records in `creative_publications` (needs migration `0001`).
- **Mock:** text (`mock-ai.ts`), images (`mock-image.ts`, drawn from a seed and marked "SAMPLE IMAGE") and publishing (`mock-publisher.ts`). Ads are only ever sent as *drafts*.
- **Going live:** implement `CreativeAi`, `ImageGenerator` and `CreativePublisher` (see `src/modules/creative/service.ts`). For real images return a `url` (upload to the `creatives` storage bucket); the UI shows it automatically.
- **Festival dates** live in `src/modules/creative/festivals.ts`. Fixed-date festivals repeat yearly; lunar-calendar ones carry an explicit date and need a yearly update (current dates checked Oct 2026, some regions differ by a day).
- The AI never invents discounts: with no offer typed, festival copy stays generic.
