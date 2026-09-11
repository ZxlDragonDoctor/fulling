# Vercel Deployment

Fulling's public application can be deployed to Vercel without configuring
GitHub OAuth, Better Auth, or PostgreSQL. Those integrations are currently
optional so they do not block preview and production builds while workspace
access is being rebuilt.

## Project settings

Connect `FullAgent/fulling` to a Vercel project and keep these settings:

- Framework preset: Next.js
- Root directory: repository root
- Install command: Vercel default (`npm install` from `package-lock.json`)
- Build command: `npm run build`
- Output directory: Vercel default
- Node.js: 24.x (Project Settings), matching `package.json#engines.node`
- Production branch: `main`

No `vercel.json` file or application environment variables are required for this
deployment mode. Do not set `SKIP_ENV_VALIDATION`; absent optional integrations
are handled by the application itself.

`output: 'standalone'` remains enabled because the Docker image consumes it.
Vercel uses its native Next.js output.

## Node.js runtime source of truth

`package.json#engines.node` is the repository-owned source of truth for the
Node.js major used by Vercel builds and Functions. Vercel documents that
`engines.node` overrides the project dashboard selection, so a dashboard value
that disagrees with the repository produces a version override warning even when
the deployment succeeds.

Keep these locations on the same major (currently **24**):

| Location | Role |
| -------- | ---- |
| `package.json#engines.node` | Authoritative for Vercel and local installs |
| `package-lock.json` root `engines` | Must match `package.json` after `npm install` |
| `.github/workflows/ci.yml` `NODE_VERSION` | CI and release verification pin |
| `Dockerfile` `FROM node:<version>-alpine` | Container build/runtime image |
| Vercel Project Settings → Node.js Version | Dashboard mirror of the same major |
| This document | Operator guidance |

When changing the Node.js major, update every row in one pull request and verify
a clean Preview deploy (and Docker build) before merge. Do not add a separate
`.nvmrc` or `.node-version` unless the team deliberately changes this policy.

To silence an override warning without a code change: set Vercel Project
Settings → Node.js Version to the same major as `package.json` (currently `24.x`).
A dashboard default that still points at a different major (for example Node.js
22 or 24 while the repository declares another) will keep warning until both
sides match.

## Zero-configuration behavior

Without the legacy Auth and database variables:

- `/` renders the public landing page.
- `/login` renders an explicit sign-in unavailable state.
- `/api/auth/*` returns `503 AUTH_UNAVAILABLE` instead of initializing Better
  Auth with an unsafe default secret.
- Protected workspace routes redirect to `/login`.
- No Prisma query or Kubernetes credential operation is attempted for anonymous
  requests.

The legacy Auth path is enabled only when all five variables are present:

- `DATABASE_URL`
- `BETTER_AUTH_URL`
- `BETTER_AUTH_SECRET`
- `GITHUB_CLIENT_ID`
- `GITHUB_CLIENT_SECRET`

This compatibility path is not required for the current Vercel deployment and
will be replaced with the next authentication design.

## Deploy and verify

With Git integration, push a feature branch to create a Preview deployment and
merge the verified commit to `main` for Production.

Verify the zero-configuration deployment:

1. Confirm `/` returns a successful response and renders the landing page.
2. Confirm `/login` returns a successful response and shows the unavailable
   state without a Better Auth error.
3. Confirm a request to `/api/auth/session` returns a structured 503 response.
4. Inspect the build and Function logs and confirm there are no missing-secret,
   Prisma connection, or Kubernetes connection errors.

Inspect a deployment in the Vercel dashboard or with the CLI:

```bash
vercel inspect <deployment-url>
vercel logs <deployment-url>
```

Roll back a bad production deployment with:

```bash
vercel rollback
```
