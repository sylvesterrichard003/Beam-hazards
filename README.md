# beamhazard

This is a [Next.js](https://nextjs.org) project bootstrapped with [v0](https://v0.app).

## Built with v0

This repository is linked to a [v0](https://v0.app) project. You can continue developing by visiting the link below -- start new chats to make changes, and v0 will push commits directly to this repo. Every merge to `main` will automatically deploy.

[Continue working on v0 →](https://v0.app/chat/projects/prj_KMk1V9n66cT6rk6YA0sto5GwRlIa)

## Getting Started

First, run the development server:

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

You can start editing the page by modifying `app/page.tsx`. The page auto-updates as you edit the file.

## Accounts & sessions

The municipal dashboard (`components/beam/admin-dashboard.tsx`) is **not** part of
the public UI. Regular visitors only ever get the *Map* and *List* views plus a
**Sign up** button in the header — the *Admin* tab and the dashboard markup are
rendered only for a server-verified account.

- `lib/auth-shared.ts` — types and validation rules shared by the browser and the server.
- `lib/auth.ts` *(server only)* — scrypt password hashing, the JSON account store,
  HMAC-signed session tokens and `getSessionUser()`.
- `app/api/auth/signup` · `app/api/auth/login` · `app/api/auth/logout` — route handlers
  that validate credentials server-side and set/clear the session cookie.
- `app/page.tsx` is a server component: it verifies the session cookie before
  rendering, then hands the user to `components/beam/app-shell.tsx`. That is what
  makes the gate real (and why a signed-up user stays signed in across reloads).

Notes and limits:

- Accounts live in `.data/accounts.json` (gitignored, created on first sign-up).
  Passwords are hashed with scrypt — plaintext is never written or returned.
- Sessions are stateless: a signed payload in the `beam_session` cookie
  (HttpOnly, SameSite=Lax, 30 days). Set **`AUTH_SECRET`** to a 32+ character
  random value in production; without it a development fallback is used.
  Set **`AUTH_URL`** (e.g. `https://beam.example.com`) so the cookie gets the
  `Secure` flag only when the app is really served over HTTPS.
- The filesystem store fits a long-running Node server. On read-only or ephemeral
  filesystems (e.g. Vercel functions) swap `readAccounts`/`writeAccounts` in
  `lib/auth.ts` for a database — nothing else needs to change.

## Reports

Community reports are stored server-side too, so they survive reloads and every
visitor sees the same feed (seeded from `MOCK_HAZARDS` the first time
`.data/hazards.json` is read).

| Route | Access | Purpose |
| --- | --- | --- |
| `GET /api/hazards` | public | Feed for the map and list views |
| `POST /api/hazards` | public | File a new report |
| `POST /api/hazards/[id]/upvote` | public | Vote `{ delta: 1 \| -1 }` |
| `PATCH /api/hazards/[id]` | signed-in only | Change repair status |
| `DELETE /api/hazards/[id]` | signed-in only | Delete a report |

- `lib/hazards-store.ts` owns the store: seeding, id allocation (`BM-####`),
  payload validation (`parseHazardReport`), status/upvote/delete mutations.
- `lib/json-store.ts` is the shared `.data/*.json` reader/writer used by both the
  account and hazard stores.
- `app/page.tsx` loads the feed on the server; `components/beam/app-shell.tsx`
  updates optimistically, then replaces its copy with the API response. A failed
  write rolls back and re-syncs from the store, showing a banner instead of
  silently drifting.
- Writes that only the dashboard should perform are gated on the same
  server-verified session as the dashboard itself, so the API can't be used to
  change or delete reports without signing in.
- Photos picked in the browser are `blob:` previews that die with the page, so a
  saved report keeps the default hazard image (see `persistableImage`). Add an
  upload route if you want real images persisted.

## Deploying on Render

`render.yaml` in the repo root is a ready-to-use [Blueprint](https://render.com/docs/blueprints):
**New → Blueprint → pick the repo**, and Render creates the web service with the
right commands and environment variables.

Four things about this project decide how it can be hosted, and each one is a
reason a stock "connect the repo and hit deploy" attempt fails or quietly
misbehaves:

1. **Render deploys from Git, not from a zip.** A folder downloaded from GitHub
   (or exported from v0) has no `.git` directory, so there is nothing for Render
   to connect to. Push it to GitHub/GitLab/Bitbucket first:

   ```bash
   git init -b main && git add . && git commit -m "beamhazard"
   git remote add origin git@github.com:<you>/beamhazard.git
   git push -u origin main
   ```

2. **It has to be a *Web Service*, not a *Static Site*.** `app/api/**` holds real
   route handlers and `app/page.tsx` verifies the session cookie server-side, so
   there is no `out/` folder to publish. `next.config.mjs` deliberately does *not*
   set `output: "export"`, and it must stay that way. Command reference:

   | Setting | Value |
   | --- | --- |
   | Build command | `pnpm install --frozen-lockfile && pnpm run build` |
   | Start command | `pnpm run start` (`next start`) |
   | Node version | `.node-version` → `24.21.0`, `engines.node` → `>=20.9.0 <25` |

   `next start` reads `process.env.PORT` itself, so Render's injected port is used
   with no extra flag — never replace `pnpm run start` with `next start -p 3000`.

   The build command names pnpm because the repo ships **two** lockfiles
   (`package-lock.json` from npm, `pnpm-lock.yaml` from v0). Render picks the
   package manager from lockfiles, and two of them is a coin toss. Either keep the
   explicit command, or delete the lockfile you don't use. Both work here —
   `pnpm install --frozen-lockfile` and `npm ci` were both verified against them.
   Prefer npm? Use `npm ci && npm run build` / `npm run start` and remove
   `pnpm-lock.yaml` and `pnpm-workspace.yaml`.

3. **The `.data/` JSON stores do not survive a Render deploy.** `lib/json-store.ts`
   writes to the filesystem, but `.data/` is gitignored (so nothing is there after
   a clone) and a Render instance boots with an empty filesystem on every deploy,
   restart and spin-down. Free instances cannot attach a disk at all, so on the
   free plan every redeploy resets `hazards.json` to `MOCK_HAZARDS` and deletes
   every signed-up account. Two ways out:

   - **Paid instance + persistent disk** — uncomment the `disk`/`BEAM_DATA_DIR`
     block in `render.yaml`. `lib/json-store.ts` already honours it (`DATA_DIR`
     resolves `BEAM_DATA_DIR` first, then falls back to `./.data`). No other code
     changes.
   - **Move to a database** — swap `readAccounts`/`writeAccounts` in `lib/auth.ts`
     and the read/write calls in `lib/hazards-store.ts` for Render Postgres. This
     is also the only option that works when the service scales past one instance
     (each instance would otherwise get its own copy of the file).
     Note that free Postgres expires 30 days after creation.

4. **`AUTH_SECRET` and `AUTH_URL` are not optional in production.** With no
   `AUTH_SECRET`, `lib/auth.ts` signs sessions with the literal
   `"beam-dev-secret-change-me"`, so anybody can mint an admin cookie and the
   dashboard gate in `app/page.tsx` stops meaning anything. `render.yaml` sets it
   with `generateValue: true`; add `AUTH_URL=https://<service>.onrender.com` so
   `useSecureCookies()` follows the real scheme instead of guessing from
   `NODE_ENV`.

Free instances also spin down after ~15 minutes of inactivity and cold-start on
the next request, which is fine for a demo. Nothing else in the app needs Render
specific handling: images are already `unoptimized`, `lib/geocode.ts` calls
Nominatim over plain outbound HTTPS (no inbound port), and there are no other
environment variables besides the optional `GEOCODER_URL` / `GEOCODER_EMAIL`.

## Learn More

To learn more, take a look at the following resources:

- [Next.js Documentation](https://nextjs.org/docs) - learn about Next.js features and API.
- [Learn Next.js](https://nextjs.org/learn) - an interactive Next.js tutorial.
- [v0 Documentation](https://v0.app/docs) - learn about v0 and how to use it.
