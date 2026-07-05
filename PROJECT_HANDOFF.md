# J-Techno Engineering — Project Handoff

Single source of truth for two linked projects owned by the same person. Written to be handed to any AI tool or a fresh Claude Code session with zero prior context — it states only the current, correct state, not the history of how it got there. If something below conflicts with what you find in the actual code, trust the code and treat this file as stale on that one point.

**Last verified accurate:** 2026-07-05, both repos pushed and matching their remotes.

## 1. Who this is for

- **Company:** J-Techno Engineering, a sole proprietorship registered in India.
- **Proprietor:** Vipul Rameshbhai Joshina. **Founder / lead engineer:** Prince Vipul Joshina, M.Sc. Mechanical Engineering (Leibniz University Hannover).
- **Based in:** Surat, Gujarat, India. **Serves:** German/EU industrial and automation clients.
- **GST:** 24ADEPJ6770B1ZV.
- **GitHub account:** `Prince2893`.

Two separate repos, one brand. They must look and feel like one product.

## 2. The two properties

| | Marketing website | BA-Generator app |
|---|---|---|
| Purpose | Company marketing site | SaaS tool: generates CE-compliant Betriebsanleitungen (operating instructions manuals) |
| Live URL | https://jtechnoengineering.com | https://jtechno-ba-generator.onrender.com (also intended at https://app.jtechnoengineering.com — subdomain not yet wired, see §6) |
| GitHub repo | `Prince2893/j-techno-website` | `Prince2893/ba-generator` |
| Local path | `D:\Business Plan\J-Techno Engineering\Documents\Company registeration & website\j-techno-website-main` | `D:\Local Setup\Generator` |
| Hosting | GitHub Pages (static) | Render (web service + managed Postgres) |
| Stack | Plain HTML, Tailwind v2 via CDN, Inter font, Font Awesome 6.4.0 via CDN — **no build step, no framework, keep it that way** | React 18 + TypeScript + Vite + Tailwind, Node/Express backend, Prisma ORM |

## 3. Marketing website — structure

No templating: every page has its own copy-pasted `<header>`/`<nav>`/`<footer>` block. Editing nav or footer means editing all 7 pages individually.

- **7 marketing pages** (full header/nav/footer each): `index.html`, `about.html`, `engineering-services.html`, `automation-integration.html`, `industries.html`, `working-model.html`, `contact.html`.
- **2 legal pages** (no header/nav at all): `impressum_full.html`, `privacy-policy.html`.
- `cookie-banner-code.html` — a code snippet file, not a real page.
- `CNAME` file in repo root pins GitHub Pages to `jtechnoengineering.com`.
- Nav order on every page: Home · Engineering Services · Automation & Integration · Industries · Working Model · About · **BA Generator** · Contact (button).
- `index.html` has a full `id="ba-generator"` section (between "What We Do" and "Our Engineering Approach") introducing the BA-Generator with a CTA button linking to `https://app.jtechnoengineering.com`.
- Footer's "Engineering Services" column includes a "BA Generator" link on all 7 pages.

## 4. BA-Generator app — architecture

- **Frontend:** React 18 + TypeScript + Vite. React Router v7 for client-side routing. Full DE/EN bilingual UI via a custom `LanguageContext` + `src/i18n/translations.ts` (every UI string keyed in both languages — check `TKey = keyof typeof de` when adding strings; `en` must have identical keys or it won't compile).
- **Routing/auth gating:** `/login`, `/signup`, `/forgot-password`, `/reset-password` are public. Everything else requires a session (`RequireAuth`). The wizard route additionally requires an active subscription (`SubscriptionGate`) — but since every account gets the Free tier automatically at signup (see §5), this gate is effectively invisible in normal use.
- **Backend:** Node + Express (`server/index.ts`, `server/routes/*.ts`), run via `tsx` (no separate compile step — `npm start` runs `tsx server/index.ts` directly in both dev and prod).
- **Database:** Prisma ORM with **two separate schema files that must be kept in sync by hand**:
  - `prisma/schema.prisma` — SQLite, for local dev only. Migrations in `prisma/migrations/`.
  - `prisma-production/schema.prisma` — Postgres, for Render. Migrations in `prisma-production/migrations/`.
  - Any model change needs a migration generated for *both* (dev: `npx prisma migrate dev --name X`; prod: `npx prisma migrate diff --from-schema-datamodel <old-prod-schema> --to-schema-datamodel prisma-production/schema.prisma --script`, saved by hand into a new `prisma-production/migrations/<timestamp>_X/migration.sql`).
- **Document generation:** `docx.js`, entirely client-side in the browser (`src/services/docxGenerator.ts`).
- **AI drafting (Step 5 of the wizard):** Mistral AI via the official `@mistralai/mistralai` SDK (`Mistral` is a **named** export, not default), model `mistral-small-latest`, called server-side only via `POST /api/generate`. Key env var: `MISTRAL_API_KEY` — **currently not set anywhere** (empty in local `.env`; unconfirmed whether set on Render). Until set, Step 5 returns a clear "not configured" message in the user's language instead of failing silently.
- **Billing:** Razorpay (not Stripe — Stripe is invite-only for new India-registered businesses and settles India accounts in INR only). Checkout, webhook signature verification (HMAC-SHA256, raw body), and cancellation all implemented in `server/routes/billing.ts`. Env vars `RAZORPAY_KEY_ID`/`RAZORPAY_KEY_SECRET`/`RAZORPAY_WEBHOOK_SECRET`/`RAZORPAY_PLAN_ID_*` — **currently not set anywhere**. Until set, paid-tier checkout returns "billing not configured"; the Free tier works regardless since it never touches Razorpay.
- **Auth:** bcryptjs password hashing, opaque server-side session tokens in httpOnly cookies (no JWT).
- **PDF export:** optional, requires LibreOffice (`soffice`) installed on whatever machine runs the server — falls back to a clear "not available" message if it isn't found. The `.docx` download always works regardless.

## 5. Subscription plans (`src/services/plans.ts` — single source of truth, used by both client UI and server enforcement)

| Tier | Price | Docs/month | Notes |
|---|---|---|---|
| `free` | €0 | 1 | **Assigned automatically at signup, no Razorpay interaction.** Generated documents get a "Erstellt mit J-Techno BA-Generator" watermark line in the footer. Users can switch back to this tier anytime via `POST /api/billing/select-free` (best-effort cancels any existing Razorpay subscription first). |
| `starter` | €99 | 3 | |
| `professional` | €299 | unlimited | |
| `complianceSuite` | €599 | unlimited | |
| `enterprise` | custom (~€1,500+) | unlimited | |

Only the **doc-count cap** and the **Free-tier watermark** are actually enforced. Every tier above Free also carries feature flags (`riskAssessmentStandalone`, `declarationOfConformity`, `technicalFileIndex`, `maintenancePlanStandalone`, `whiteLabel`, `apiAccess`, `customTemplates`, `prioritySupport`) that are defined in the data model but **not implemented anywhere** — they're placeholders for standalone generators that don't exist yet (see §7).

**Billing period mechanics:** a subscription's `currentPeriodStart`/`currentPeriodEnd` is auto-advanced by 30 days server-side (`server/subscriptionGuard.ts`) whenever it's found expired. This is what makes "N docs per month" actually reset monthly for the Free tier and any subscription not driven by a Razorpay webhook — there is no calendar-month alignment, it's a rolling 30-day window from whenever it was last checked.

## 6. Deployment status — what's live vs. what still needs a human

**Done and pushed:**
- Both repos are on `main`, pushed, matching their remotes (`ba-generator` @ `a5fc276`, `j-techno-website` @ `f820e68`).
- Render Blueprint (`render.yaml`) deploys the web service + managed Postgres together. Build command must include `npm install --include=dev` — Render sets `NODE_ENV=production` which makes plain `npm install` skip devDependencies (including `typescript`/`vite`/`@types/react`/`tsx`), breaking the build. This is already fixed in `render.yaml`; don't remove `--include=dev`.
- The app is live and working at `https://jtechno-ba-generator.onrender.com`.

**Not yet done — needs the account owner, not code:**
1. **Namecheap DNS:** add a CNAME record, Host = `app`, Value = `jtechno-ba-generator.onrender.com`, TTL = Automatic.
2. **Render custom domain:** on the `jtechno-ba-generator` web service → Settings → Custom Domains → add `app.jtechnoengineering.com`. SSL auto-provisions once the DNS record is detected.
3. **Real `MISTRAL_API_KEY`** — get one from Mistral's console, set it in Render's environment variables. The SDK call shape was verified against the installed package's own type declarations (not against a live API response, since no key was available while building this) — the very first live call is still worth a manual smoke-test.
4. **Real Razorpay account** — needed before any paid tier can actually be purchased. Requires: business KYC with Razorpay, one monthly Plan created per paid tier (Starter/Professional/Compliance Suite/Enterprise) with their Plan IDs set as env vars, a webhook pointed at `POST https://app.jtechnoengineering.com/api/billing/webhook` with its secret set as `RAZORPAY_WEBHOOK_SECRET`.
5. **GST/LUT filing** — exporting SaaS services internationally as an Indian sole proprietorship needs either GST charged on the invoice or a valid Letter of Undertaking (Form GST RFD-11) to zero-rate the export. Needed before real international payments, independent of the Razorpay setup itself.
6. **Real transactional email** — password-reset links currently just log to the server console (`server/email.ts`); no email provider is wired in.
7. **Upgrade Render plans** before relying on this for real customers — the free web service spins down after 15 minutes idle (next visitor waits ~30-60s), and free Postgres databases expire after a fixed period.

## 7. Explicitly NOT built (so nobody re-requests these thinking they're missing)

- Standalone **Risk Assessment**, **Declaration of Conformity**, **Technical File Index**, and **Maintenance Plan** generators implied by the paid-tier feature flags — only the BA-Generator (Betriebsanleitung) itself exists. This was a deliberate scope boundary, not an oversight.
- Automated test suite — all verification so far has been manual: type-check, production build, and live browser walkthroughs (signup → wizard → generate document → check output) repeated each time something changed.
- Admin dashboard, team/multi-user accounts, per-tenant API keys.
- Any CI/CD beyond Render's own auto-deploy-on-push.

## 8. Shared branding — exact values (read from the live website, not guessed)

| Token | Value | Where it's used |
|---|---|---|
| Primary gradient | `#F97316` → `#EA580C`, 135deg | Buttons, links, accents |
| Heading/dark text | `#111827` (Tailwind `gray-900`) | Headings, logo wordmark, footer background |
| Alt section background | `#F9FAFB` (Tailwind `gray-50`) | Alternating page sections, app shell background |
| Standard grays | `gray-700 #374151`, `gray-600 #4B5563`, `gray-400 #9CA3AF`, `gray-300 #D1D5DB` | Body text, borders |
| Font | Inter, weights 400/500/600/700, via Google Fonts | Everywhere, both properties |
| Button shape | `border-radius: 0.375rem` (Tailwind `rounded-md`) | Both properties |
| Button hover | `translateY(-2px)` + soft shadow, `transition: all 0.3s ease` | Both properties |
| Logo | `images/Logo.svg` on the website (raster PNG wrapped in SVG); same art as `public/logo.png` in the app | Header/footer on both properties |

In the generator's `tailwind.config.js`, this lives under the `jtechno` namespace: `dark: '#111827'`, `orange: '#f97316'`, `orangeDark: '#ea580c'`, `accent: '#fb923c'`, `gray: '#f9fafb'`.

**Cross-linking pattern (already implemented, keep consistent if either property changes):**
- App header has a dark "← Back to J-Techno Engineering" strip above the main white header row; the logo and a "Home" link both point to `https://jtechnoengineering.com`.
- App footer mirrors the website's 4-column footer exactly (Engineering Services / Company / Contact columns, copyright bar with Impressum/Privacy Policy/Datenschutz) — every link points at the real marketing site's actual pages, not internal app routes.
- Website nav/footer link to the app via `index.html#ba-generator` (the section) and the section's own CTA links directly to `https://app.jtechnoengineering.com`.

## 9. Known pitfalls already hit once — don't repeat these

- **Prisma v7** breaks the classic `datasource { url }` config style used here — stay on Prisma **v6.x** (`prisma`/`@prisma/client` both pinned to `^6.19.3`) until there's a deliberate reason to migrate and rewrite for v7's driver-adapter model.
- **`cookie` npm package v2** renamed `parse`/`serialize` to `parseCookie`/`stringifySetCookie` with a different argument shape — stay on `cookie@0.7.2` / `@types/cookie@0.6.0`.
- **Render + `NODE_ENV=production`**: makes plain `npm install` skip devDependencies during the build, breaking `tsc`/`vite`. Already fixed with `--include=dev` in `render.yaml` — don't remove it.
- **`@mistralai/mistralai` SDK**: `Mistral` is a **named** export (`import { Mistral } from '@mistralai/mistralai'`), not a default export.
- Two Prisma schemas (`prisma/` for dev SQLite, `prisma-production/` for prod Postgres) mean model changes are **not** automatically kept in sync — see §4.

## 10. Running each project locally

**Website:** no build step. Open the HTML files directly, or serve the folder with `python -m http.server 8899` for a realistic preview (relative links, anchors, etc.).

**Generator:**
```bash
cd "D:\Local Setup\Generator"
npm install
npx prisma migrate dev          # creates prisma/dev.db (SQLite)
# then two terminals:
npm run server                  # API on http://localhost:8787
npm run dev                     # Vite dev server on http://localhost:5173, proxies /api to the server above
```
Full environment variable reference (with comments explaining each) is in `.env.example`. To test the subscription-gated wizard without a real Razorpay account, use `npx tsx scripts/simulate-subscription.ts <email> <tier> <status>` and `npx tsx scripts/cleanup-test-user.ts <email>` to clean up afterward.

Full technical documentation (AI data-handling rationale, document-generation details, norm/compliance coverage, accessibility notes, known limitations) lives in the generator repo's own `README.md` — this handoff file is the cross-project summary; the README is the deep-dive for that one repo.
