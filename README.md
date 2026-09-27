# Timeo

Restaurant staff scheduling. This repo has two apps:

- `frontend` — Next.js (App Router)
- `backend` — Express API

Run them side by side in two terminals. The web app is [http://localhost:3000](http://localhost:3000). The API is [http://localhost:4000](http://localhost:4000).

## Prerequisites

- Node.js 22 or newer
- PostgreSQL

## Backend

```bash
cd backend
cp .env.example .env
npm install
npx prisma generate
npm run dev
```

This skeleton starts Express with CORS and cookie parsing. Routes and database models come later.

The API uses TypeScript 5.9, the newest release `ts-node-dev` can load.

`DATABASE_URL` is the connection the API uses at runtime. `DIRECT_URL` is the direct connection Prisma CLI uses for migrations and introspection. On a local Postgres instance, set both to the same string. Leave `COOKIE_DOMAIN` empty for local development.

Google OAuth, JWT, bcrypt, rate limiting, and link tokens (`nanoid`) are installed and not wired up yet. `nanoid` v6 is ESM-only, so import it with `await import("nanoid")` from this CommonJS API.

## Frontend

```bash
cd frontend
cp .env.example .env.local
npm install
npm run dev
```

`NEXT_PUBLIC_API_URL` should point at the API (`http://localhost:4000` locally). The fetch client in `frontend/lib/api.ts` always sends `credentials: "include"` so cookie auth can work with TanStack Query.

## Production cookies

The frontend and API will be served from different domains (Vercel and Render). Auth cookies must be set with `SameSite=None; Secure`, and CORS must keep `credentials: true`. That is called out in `backend/src/app.ts`.
