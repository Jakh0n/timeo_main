# FINAL BUILD PLAN — Restaurant Scheduling SaaS

Bu hujjat oldingi barcha suhbatlarimizni (stack pivotlari, Calendly
modeli, Node.js scheduler) **bitta yakuniy, ziddiyatsiz** rejaga
birlashtiradi. Oldingi fayllardagi (`PROJECT.md`, `mvp-prompts.md`,
`auth-onboarding-prompt.md`, `ui-design-prompt-v2.md`,
`SAAS-MASTER-PLAN.md`) g'oyalar shu yerda yakuniy holatga keltirilgan —
endi shu FAYLNI asos qiling, boshqalari tarixiy qoralama hisoblanadi.

---

## QISM A: Men topgan bo'shliqlar va takliflarim

Loyihani boshidan oxirigacha ko'rib chiqqanda, quyidagi joylar hali
hal qilinmagan yoki o'zaro ziddiyatli edi. Har biriga tajribam
asosida qaror qabul qildim va pastda tushuntirdim:

### 1. "Workers" sahifasi — endi kerak emas

Worker'da account yo'qligi sababli (Calendly-uslub qaroriga ko'ra),
alohida "Workers" boshqaruv sahifasi mantiqsiz — boshqariladigan
"worker" entity'si yo'q, faqat submission'lar bor. **Qaror**: sidebar
navigatsiyasidan "Workers"ni olib tashladim.

### 2. Telegram bot — MVP uchun ortiqcha murakkablik

Oldin Telegram orqali avtomatik xabar yuborish rejalashtirilgan edi,
lekin bu worker-account talab qiladigan arxitektura uchun mo'ljallangan
edi. Endi manager linkni **qo'lda** ulashadi (sizning talabingiz).
**Qaror**: workerlarga natijani ko'rsatish uchun **aynan o'sha bir link**
qayta ishlatiladi — jadval tasdiqlangach, worker o'sha linkka qaytib,
ism+employeeID kiritsa, o'z smenalarini ko'radi. Bu Telegram bot,
ikkinchi link, yoki email yuborish infratuzilmasi shart emasligini
anglatadi — MVP uchun ancha soddalashadi.

### 3. Xavfsizlik bo'shlig'i: public (login'siz) endpoint

Availability submission formasi hech qanday auth talab qilmagani
uchun, uni **spam/abuse**dan himoya qilish kerak. **Qaror**: IP-based
rate limiting (masalan bir IP'dan daqiqada 5 ta so'rovdan ko'p emas)
va linkToken yetarlicha uzun/tasodifiy (kamida 21 belgi, nanoid orqali)
bo'lishi promptlarga qo'shildi.

### 4. Bir xil worker ikki marta submit qilsa nima bo'ladi?

**Qaror**: `employeeId + shiftRequirementId` bo'yicha **upsert** —
ikkinchi marta to'ldirsa, birinchisini almashtiradi (Calendly'da ham
booking'ni qayta tuzatish shunday ishlaydi).

### 5. Cross-origin auth cookie muammosi (Vercel + Render)

Frontend (Vercel) va backend (Render) turli domenlarda bo'lgani uchun,
JWT cookie standart sozlamalarda ishlamaydi. **Qaror**: cookie
`SameSite=None; Secure` bilan sozlanadi, CORS'da `credentials: true`
yoqiladi, frontend'dagi har bir so'rovda `credentials: "include"`
ishlatiladi — bu promptlarga aniq yozib qo'yildi (bu ko'p developerlar
duch keladigan, lekin kam hujjatlashtiriladigan muammo).

### 6. Row-Level Security — MVP'da emas, lekin tayyorlangan

Hozircha bitta Organization bilan ishlaysiz, shuning uchun Postgres RLS
ni hozir yoqish ortiqcha murakkablik. **Qaror**: schema'da
`organizationId` hamma joyda bor, lekin RLS yoqish **Bosqich 2**ga
(boshqa restoranlar qo'shilganda) qoldirildi — MVP'da oddiy
`WHERE organizationId = ...` filter yetarli, chunki bitta tenant bor.

### 7. Email tasdiqlash va parolni tiklash — MVP'dan tashqarida

Bular muhim, lekin MVP'ni bloklamasligi kerak. **Qaror**: promptlarda
"stub" sifatida belgilandi — sahifa bor, lekin to'liq email yuborish
logikasi keyingi bosqichga qoldiriladi.

### 8. HiGHS'ni har so'rovda qayta yuklash — performance muammosi

WASM modulini har HTTP so'rovda qayta init qilish sekin bo'ladi.
**Qaror**: promptda aniq ko'rsatilgan — HiGHS instance Express
serverida **bitta marta**, server ishga tushganda yuklanadi (singleton),
har so'rovda qayta ishlatiladi.

---

## QISM B: Yakuniy arxitektura xulosasi

```
Frontend (Vercel):  Next.js + TypeScript + Tailwind + shadcn/ui
                     + Zustand (UI state) + TanStack Query (server state)

Backend (Render):   Node.js + Express.js + Prisma + PostgreSQL (Supabase)
                     + HiGHS (npm: "highs") — scheduler shu backend ICHIDA,
                       alohida servis EMAS
                     + Passport.js (Google OAuth) + JWT (httpOnly cookie)

Database:           PostgreSQL (Supabase, bepul tier)

Deploy:             2 ta joy (frontend + backend) — Python servisi YO'Q
```

**Asosiy oqim (yakuniy):**

1. Manager signup/login (email+parol yoki Google) → onboarding
   (restoran nomi) → dashboard
2. Manager filial qo'shadi, keyin "Shift Requirement" yaratadi
   (filial + weekday/weekend uchun day/night necha kishi kerak)
3. Sistema link generatsiya qiladi, manager copy qilib o'zi yuboradi
4. Worker linkni ochadi (login yo'q) → ism, employeeID, qaysi kunlar
   ishlay oladi, day/night, soat oralig'ini kiritadi → Submit
5. Manager dashboard'da submission'larni real-vaqtda ko'radi
6. Manager "Generate Schedule" bosadi → HiGHS orqali avtomatik taqsimot
7. Manager ko'rib chiqadi, tuzatadi, **Confirm** bosadi
8. Worker **o'sha linkka qaytib**, ism+ID kiritib, o'z smenasini ko'radi

---

## QISM C: Cursorga beriladigan promptlar (ketma-ket)

### PROMPT 0 — Loyiha skeleti

```
Set up a new project with two parts in one repo: a Next.js frontend
and an Express.js backend, for a restaurant staff scheduling SaaS.

/frontend
  Next.js 14+ (App Router), TypeScript strict mode, Tailwind CSS,
  shadcn/ui, Zustand (UI state only), TanStack Query (server state).
/backend
  Express.js + TypeScript, Prisma ORM targeting PostgreSQL, Passport.js
  (for Google OAuth, to be configured later), jsonwebtoken, bcrypt,
  cors, express-rate-limit, nanoid (for generating link tokens).

Tasks:
1. Scaffold /frontend: Next.js + TS + Tailwind + shadcn/ui init
   (install button, card, dialog, table, select, badge, tabs, toast,
   input, form components). Set up TanStack Query's QueryClientProvider
   at the root layout. Set up a Zustand store file at /frontend/lib/store.ts
   (empty for now).
2. Scaffold /backend: Express + TypeScript project (tsconfig, ts-node-dev
   for local dev), a basic app.ts with CORS configured for the frontend's
   origin with `credentials: true`, and cookie-parser middleware.
3. Set up Prisma in /backend, configured for PostgreSQL via DATABASE_URL.
4. Create .env.example in both /frontend and /backend:
   - /backend: DATABASE_URL, DIRECT_URL, JWT_SECRET, GOOGLE_CLIENT_ID,
     GOOGLE_CLIENT_SECRET, GOOGLE_CALLBACK_URL, FRONTEND_URL,
     COOKIE_DOMAIN (empty for local dev)
   - /frontend: NEXT_PUBLIC_API_URL
5. In /frontend, set up a small typed API client (e.g. /lib/api.ts)
   wrapping fetch with `credentials: "include"` always set, pointing
   at NEXT_PUBLIC_API_URL, for use with TanStack Query hooks.
6. Write a root README explaining how to run both services locally
   side by side.

Do not implement any routes or database models yet — skeleton only.
Note for later: because frontend and backend will be on different
domains in production (Vercel + Render), auth cookies must be set with
SameSite=None; Secure and CORS must allow credentials — flag this with
a comment in the CORS setup so it isn't forgotten later.
```

---

### PROMPT 1 — Ma'lumot modeli (Prisma schema)

```
Design the Prisma schema (/backend/prisma/schema.prisma) for a
multi-tenant restaurant scheduling SaaS, built around a Calendly-style
workflow: managers have accounts, workers do NOT — they interact only
through a public link tied to a specific shift requirement.

Models:

1. Organization
   - id, name, slug (unique), createdAt

2. ManagerUser
   - id, organizationId, name, email (unique), passwordHash (nullable —
     null if the user only ever signed up via Google), googleId
     (nullable, unique), role: OWNER | MANAGER

3. Branch
   - id, organizationId, name, address

4. ShiftRequirement  (the "Calendly event" — one per branch per cycle)
   - id, organizationId, branchId, cycleLabel (e.g. "Fall2026")
   - weekdayDayRequired, weekdayNightRequired: Int
   - weekendDayRequired, weekendNightRequired: Int
   - requiredSeniorPerShift, maxSeniorPerShift: Int (default 1)
   - linkToken: String, unique, generated with nanoid (21+ chars)
   - status: DRAFT | COLLECTING | GENERATED | CONFIRMED
   - createdAt

5. AvailabilitySubmission  (NO relation to any user account)
   - id, shiftRequirementId
   - workerName: String
   - employeeId: String
   - submittedAt, updatedAt
   - @@unique([shiftRequirementId, employeeId]) — enforces the
     upsert-on-resubmit behavior at the database level

6. AvailabilityEntry
   - id, availabilitySubmissionId
   - day: MON..SUN (enum)
   - shiftType: DAY | NIGHT (enum)
   - startHour, endHour: Int (0-23) — if endHour <= startHour, the
     window wraps past midnight; add a schema comment explaining this

7. ScheduleAssignment
   - id, shiftRequirementId, branchId
   - workerName, employeeId (denormalized from the submission — copied
     at generation time so historical schedules stay correct even if
     a submission is later edited)
   - day, shiftType
   - status: PROPOSED | CONFIRMED

Requirements:
- @@index on (shiftRequirementId) for AvailabilitySubmission and
  ScheduleAssignment.
- @@unique on ShiftRequirement.linkToken.
- Run `npx prisma migrate dev --name init`.
- Write /backend/prisma/README.md with a short relationship diagram,
  and a one-paragraph note explaining why AvailabilitySubmission has
  no foreign key to any user table (workers don't have accounts —
  identity is just name + employeeId, validated for uniqueness only
  within one ShiftRequirement).
```

---

### PROMPT 2 — Auth: signup, login, Google OAuth, onboarding

```
Implement the full manager authentication flow on the Express backend,
consumed by the Next.js frontend, following this architecture: Express
is the single source of truth for auth (issues the JWT), so Google
OAuth must also be handled on the Express side via Passport.js
(passport-google-oauth20), not inside Next.js — this way both
email/password and Google sign-in issue the exact same JWT cookie.

Backend routes (/backend/src/routes/auth.ts):
- POST /api/auth/signup — { name, email, password } -> creates
  ManagerUser (role OWNER, no organizationId yet), issues JWT httpOnly
  cookie (SameSite=None; Secure in production, Lax in local dev —
  branch on NODE_ENV), returns { hasOrganization: false }
- POST /api/auth/login — { email, password } -> verifies bcrypt hash,
  issues the same JWT cookie, returns { hasOrganization: boolean }
- GET /api/auth/google — Passport redirect to Google
- GET /api/auth/google/callback — Passport handles callback,
  finds-or-creates ManagerUser by googleId (or by matching email if an
  account already exists), issues the JWT cookie, then redirects the
  browser (302) to `${FRONTEND_URL}/onboarding` or `${FRONTEND_URL}/dashboard`
  depending on whether an organizationId is already set
- GET /api/auth/me — returns the current user (from JWT) plus whether
  they have an organization, or 401 if not authenticated
- POST /api/auth/logout — clears the cookie
- Rate limit /api/auth/signup and /api/auth/login with express-rate-limit
  (e.g. 10 requests per 15 minutes per IP) to reduce brute-force risk.

Organization routes:
- POST /api/organizations — { name } -> creates an Organization
  (auto-generate a URL-safe slug), sets it on the current ManagerUser,
  rejects (400) if the current user already has an organizationId
  (prevents duplicate-org creation from a double-submitted onboarding form)

Frontend pages (Calendly/Apple visual style: centered single column,
max-width ~420px, one accent color, generous whitespace, no clutter):

- /signup: name, email, password fields, primary "Create account"
  button, divider, "Continue with Google" button (styled to match this
  product, not Google's raw default), link to /login. On success,
  redirect to /onboarding.
- /login: email, password, "Log in" button, "Continue with Google",
  "Forgot password?" (stub page for now — a form that doesn't yet call
  a real endpoint is fine), link to /signup. On success, redirect to
  /onboarding or /dashboard based on the hasOrganization flag returned.
- /onboarding: single field "What's your restaurant called?", one
  primary "Continue to dashboard" button. Calls POST /api/organizations,
  then redirects to /dashboard. If the user already has an organization
  (checked via GET /api/auth/me on page load), redirect straight to
  /dashboard instead of showing the form.

Route protection (Next.js middleware or a layout-level check calling
GET /api/auth/me):
- Unauthenticated -> /dashboard/* redirects to /login
- Authenticated, no organization -> /dashboard/* redirects to /onboarding
- Authenticated, has organization -> /login, /signup, /onboarding
  redirect straight to /dashboard
- / redirects based on the above three states

Inline validation on forms (on blur, not just on submit), phrased
plainly, not as raw error codes.
```

---

### PROMPT 3 — Manager dashboard shell (sidebar) + design tokens

```
Build the manager dashboard shell using shadcn/ui's Sidebar component
(SidebarProvider, SidebarHeader, SidebarContent, SidebarFooter).

Before building, define and apply a small design token set across the
whole app (Calendly + Apple inspired): ONE neutral base (near-white/
near-black, not pure #FFF/#000) + ONE accent color used consistently
for every primary action and active state, reserved status colors
(red = problem, green = success) used only for real state — never
decoratively. One clean sans-serif type family, a small consistent
type scale, generous whitespace. No gradients, no ALL-CAPS labels, no
emoji-as-icons, no ID card-heavy shadow stacking.

Layout: /frontend/app/(dashboard)/manager/layout.tsx
- Sidebar nav items: Overview, Branches, Shift Requirements, Settings
  (no "Workers" item — there's no worker account entity in this product)
- Sidebar header: organization name (from session) + collapse toggle
- Sidebar footer: manager's name/email + working logout (calls
  POST /api/auth/logout, clears TanStack Query cache, redirects to /login)
- Top bar in the main content area showing the current page title

--- Page: Overview (/manager/dashboard)
- Compact summary stats: number of branches, number of active Shift
  Requirements, and for the most recent active one: submission count
  and status (Draft / Collecting / Generated / Confirmed).
- A first-time user with zero branches sees a clear, friendly empty
  state pointing to "Add your first branch" — not blank stat cards.
- Any Shift Requirement with an under-filled generated schedule is the
  most visually prominent element on this page, linking directly to it.
```

---

### PROMPT 4 — Branches page

```
Build /frontend/app/(dashboard)/manager/branches/page.tsx.

- Simple table: branch name, address, "Edit" / "Delete" actions.
- "Add branch" button opens a small dialog (shadcn Dialog) with name +
  address fields.
- Backend: GET/POST/PATCH/DELETE /api/branches, scoped to the current
  user's organizationId (always filter by it — this is the manual
  tenant-isolation check to apply consistently everywhere, since RLS
  isn't enabled yet per the Part A decision above).
- This is a low-frequency admin page — keep it simple and table-based.
```

---

### PROMPT 5 — Shift Requirements: create + get shareable link

```
Build /frontend/app/(dashboard)/manager/shift-requirements/page.tsx
(list view) and /frontend/app/(dashboard)/manager/shift-requirements/[id]/page.tsx
(detail view — this becomes the core workflow panel built in Prompt 7).

--- List view
- Table of all ShiftRequirements for the organization: branch name,
  cycle label, status (as a clear badge: Draft/Collecting/Generated/
  Confirmed), submission count, created date. Clicking a row goes to
  the detail page.
- "New Shift Requirement" as the single clearly primary action.

--- Create flow (dialog or dedicated page — your choice, keep it to
one screen, no multi-step wizard)
- Branch selector, cycle label input (e.g. "Fall2026")
- Four number inputs: weekday day-shift required, weekday night-shift
  required, weekend day-shift required, weekend night-shift required
- Required senior (min) and max senior per shift (default both to 1)
- On submit: POST /api/shift-requirements — creates the row with
  status=DRAFT and a generated linkToken (use nanoid, 21+ chars) on
  the backend.
- After creation, redirect to the detail page, which should immediately
  show the shareable link prominently with a "Copy link" button
  (format: `${FRONTEND_URL}/s/{linkToken}`), and set status to
  COLLECTING once the manager has seen/copied it (or add an explicit
  "Start collecting" action — pick whichever feels more natural, but
  make the transition from DRAFT to COLLECTING an explicit, visible
  manager action, not an invisible side effect).
```

---

### PROMPT 6 — Public worker submission page (NO auth)

```
Build the public, unauthenticated page at
/frontend/app/s/[linkToken]/page.tsx — this is a Calendly-style public
booking page. No sidebar, no app chrome, centered single column,
max-width ~480px, mobile-first (most workers will open this on their
phone from a shared link).

Behavior depends on the ShiftRequirement's status (fetched via a public
backend endpoint GET /api/public/shift-requirements/:linkToken — this
endpoint returns only what's needed to render the form, NOT other
workers' submissions or any organization-internal data):

- If status is DRAFT or COLLECTING: show the availability submission
  form.
- If status is GENERATED or CONFIRMED: show a "check your schedule"
  view instead (see below) — the same link now serves double duty.

--- Availability submission form (status = DRAFT/COLLECTING)
- Fields: Full name, Employee ID
- One row per day of the week (Mon-Sun). Each day: a toggle "Available
  this day?" — when off, the row collapses to just the toggle
  (progressive disclosure, Calendly-style: don't show time pickers for
  days the worker can't work). When on, reveal: a Day/Night selector,
  then start-hour/end-hour selectors for that shift type. Allow
  multiple entries per day if a worker is free for both a day and a
  night shift on the same day (or don't restrict to one — check with a
  clear "+ add another window" affordance if this comes up naturally).
- A short, plain-language note, shown visually (not just in a tooltip),
  explaining that an end hour earlier than the start hour means the
  window continues into the next day.
- Submit button: POST /api/public/shift-requirements/:linkToken/submit
  with { workerName, employeeId, entries: [...] }. On the backend,
  upsert on (shiftRequirementId, employeeId) — a second submission with
  the same employeeId replaces the first, and updates AvailabilityEntry
  rows accordingly (delete-and-reinsert inside a transaction).
- Rate limit this endpoint (e.g. 5 requests per minute per IP) since
  it's public and unauthenticated.
- After successful submit, show a clear, calm confirmation message —
  not a redirect elsewhere, just an unmistakable "you're all set" state
  on the same page.
- If a worker revisits with the same employeeId while still in
  COLLECTING status, pre-fill their previously submitted entries and
  make clear they're editing an existing submission.

--- "Check your schedule" view (status = GENERATED/CONFIRMED)
- A simple form: Employee ID (that's enough to look up — pair it with
  the name they already gave to confirm a light match, but don't
  require a password for this; this is intentionally low-friction, not
  a secure portal).
- On lookup: GET /api/public/shift-requirements/:linkToken/my-schedule?employeeId=...
  returns that person's assigned shifts (day, shift type, time range,
  branch name) if status is CONFIRMED. If status is GENERATED but not
  yet CONFIRMED, show a plain message: "Your manager hasn't confirmed
  the schedule yet — check back soon" rather than partial/tentative data.
- If the employeeId isn't found in the assignments, show a clear,
  non-alarming message rather than a generic error.
```

---

### PROMPT 7 — HiGHS scheduler module (Node.js, no Python)

```
Implement the scheduling algorithm as a module inside the Express
backend at /backend/src/scheduler/. This is a Mixed Integer Program
solved with HiGHS (npm package "highs", WASM-based) — NOT a separate
service, NOT Python. I have a working prototype (attached: scheduler.js)
that builds an LP-format model string and solves it — adapt this into
a clean, typed module.

Requirements:

1. Load the HiGHS WASM module ONCE at server startup (module-level
   singleton, not per-request) — re-initializing it on every request
   is slow. Export a function that awaits the already-loading/loaded
   instance.

2. A typed function `generateSchedule(shiftRequirement, submissions)`
   that:
   - Expands the ShiftRequirement's weekday/weekend day/night required
     counts into concrete shift slots for the week (e.g. Mon-Fri get
     the weekday counts, Sat-Sun get the weekend counts, each day
     having a DAY slot starting 09:00 and a NIGHT slot starting 21:00
     — 12-hour shifts, matching the existing domain rules)
   - Converts each AvailabilitySubmission + its AvailabilityEntry rows
     into the worker/availability shape used by the prototype's
     `isAvailable` logic (handle the midnight-wraparound math exactly
     as in the prototype)
   - Builds the LP model string (coverage, senior min/max, no-overlap/
     rest, weekly cap is not really needed here since there's no
     account-based "worker" identity across cycles — but DO still
     apply the no-overlap constraint so the same employeeId isn't
     double-booked into overlapping shifts) and solves it with HiGHS
   - Returns { status: "ok" | "infeasible", assignments, unfilledSlots }
     — for infeasible cases, relax the coverage constraint (drop it to
     a soft, best-effort objective term) and retry once, returning a
     partial result with clearly labeled unfilled slots rather than
     nothing

3. Wire this into POST /api/shift-requirements/:id/generate — this
   endpoint gathers the ShiftRequirement and its submissions via
   Prisma, calls generateSchedule, saves the result as
   ScheduleAssignment rows with status=PROPOSED (replacing any
   previous PROPOSED rows for this ShiftRequirement in a transaction),
   sets the ShiftRequirement's status to GENERATED, and returns the
   result to the frontend.

4. Write a small test file exercising generateSchedule with a sample
   dataset, asserting a valid schedule comes back (adapt the demo data
   from the attached prototype).

Keep the core LP-building logic close to the attached prototype —
only adapt data shapes to match the Prisma schema and add the
infeasibility-retry logic in point 2.
```

---

### PROMPT 8 — Schedule workflow panel (the core screen)

```
Build the ShiftRequirement detail page
(/frontend/app/(dashboard)/manager/shift-requirements/[id]/page.tsx)
as a single guided panel with a stepper/tabs pattern reflecting the
ShiftRequirement's status field — this is the most important screen in
the product, prioritize clarity above all else.

Stage 1 — "Setup" (status DRAFT)
- Read-only summary of the requirement counts, with an edit action.
- The shareable link, prominent, with a "Copy link" button.
- A "Start collecting availability" action -> PATCH status to COLLECTING.

Stage 2 — "Collecting" (status COLLECTING)
- The link remains visible/copyable at the top.
- A live submission list (poll or refetch via TanStack Query): worker
  name, employee ID, submitted time. Count summary at the top
  ("14 submissions so far").
- A "Generate Schedule" primary action (calls Prompt 7's endpoint),
  available whenever the manager decides there's enough — don't
  hard-gate on any particular count.

Stage 3 — "Review" (status GENERATED)
- The weekly schedule grid: shifts as rows (grouped by day, DAY/NIGHT
  shown with the token plan's day/night visual language), assigned
  workers as chips (name + employeeId).
- Any under-filled shift is the single most visually obvious thing on
  the page, with a specific inline reason. A fully-covered schedule
  looks calm, not celebratory.
- Manual adjustment: remove a chip, or add one via a dropdown listing
  ONLY submitted workers who are actually available for that slot
  (cross-reference their AvailabilityEntry rows) and not already
  assigned to an overlapping slot.
- A "Confirm and finalize" primary action.

Stage 4 — "Confirmed" (status CONFIRMED)
- Read-only final schedule view.
- A note reminding the manager that workers can now revisit the same
  link to check their assigned shifts (employeeId lookup).
- No further editing here — to make a change, the manager would need
  to explicitly reopen/regenerate (out of scope for this prompt; a
  simple "Reopen for editing" button that reverts status to GENERATED
  is a reasonable addition if it fits naturally, otherwise skip it).
```

---

### PROMPT 9 — Deployment prep

```
Prepare both /frontend and /backend for deployment: frontend to
Vercel, backend to Render, database on Supabase (PostgreSQL).

1. /backend: ensure the Express server reads PORT from process.env
   (Render assigns this dynamically), add a health-check endpoint
   GET /health returning 200, and confirm the HiGHS WASM file is
   included in the deployed build (check the "highs" package's
   post-install/build output ends up in the deployed node_modules —
   flag if any manual step is needed for Render's build process).
2. Document required environment variables for each service in a
   DEPLOYMENT.md at the repo root: which go on Vercel (frontend),
   which go on Render (backend), and a reminder that
   GOOGLE_CALLBACK_URL and FRONTEND_URL must point to the real
   production URLs, not localhost, once deployed.
3. Confirm the CORS and cookie settings (SameSite=None; Secure) are
   correctly conditioned on NODE_ENV=production, and that the frontend
   API client's NEXT_PUBLIC_API_URL is set to the Render backend URL
   in Vercel's environment settings.
4. Add a Prisma migration deploy step to the backend's Render build
   command (`npx prisma migrate deploy`) so schema changes apply
   automatically on each deploy.
```

---

## QISM D: Nima hali ham keyingi bosqichga qoldirilgan (ataylab)

Bular MVP'ni bloklamaydi, lekin unutilmasligi uchun ro'yxatlab qo'yaman:

- Multi-tenant Row-Level Security yoqish (boshqa restoranlar qo'shilganda)
- Billing/subscription oqimi
- Email tasdiqlash va to'liq "parolni unutdim" oqimi
- Shift-swap, analytics, semester autopilot (avvalgi hujjatlarda bor)

Shu 10 ta promptni (0-9) ketma-ket, har birini tekshirib, Cursorga
bering. `scheduler.js` (men yozib, sinab ko'rgan Node.js/HiGHS versiyasi)
faylini 7-promptga biriktiring.
