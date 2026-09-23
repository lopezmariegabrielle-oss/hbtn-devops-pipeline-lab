# Deploying to staging

Operational runbook for the `deploy` job in `.github/workflows/ci.yml`. Read this
before triggering, verifying, rolling back, or tearing down the staging
environment.

## Trigger

The `deploy` job runs only when both are true:

- the workflow event is a `push` (not a `pull_request`), and
- the push landed on `refs/heads/main`.

It also declares `needs: build`, so it only starts after the `test` and
`build` jobs have both succeeded. A pull request run tests the code but never
triggers a deploy; only a merge (or a direct push) to `main` does.

Concretely: merging a PR into `main`, or pushing to `main` directly, runs
`test → build → deploy` in sequence. Any failure in `test` or `build` stops
the chain before `deploy` starts.

## Target

Staging runs on [Render](https://render.com), as two resources under one
Render account:

- **Web service** `hbtn-devops-pipeline-lab` — a "Deploy an existing image"
  service pointed at `ghcr.io/lopezmariegabrielle-oss/hbtn-devops-pipeline-lab:latest`.
  Free instance tier. Health check path is set to `/health` in the Render
  service settings, independent of the check the workflow performs.
- **PostgreSQL database** `hbtn-pipeline-lab-db` — a disposable free-tier
  Render Postgres instance, same region as the web service.

The GHCR package is **public**. That was a deliberate choice, made after
checking the image contents: the runtime stage only copies `package.json`,
`package-lock.json` and `src/` (see the Dockerfile and `.dockerignore`, which
excludes `.env`, `.env.*`, `.git`, and `*.md`). No credentials, secrets, or
local config ever enter the image, so a public pull-only package carries no
risk here. If that ever changes (a future stage bakes in a config file, a
license key, anything non-public), switch the package back to private and
give the Render service a scoped GHCR pull credential instead.

The CI workflow triggers the deploy via Render's API:

```
POST https://api.render.com/v1/services/$RENDER_SERVICE_ID/deploys
Authorization: Bearer $RENDER_API_KEY
```

This tells Render to re-pull `:latest` and restart the service. It does not
build anything on Render's side — the image was already built and pushed by
the `build` job.

## Database configuration

`DATABASE_URL` is set as an environment variable directly on the Render web
service (Render dashboard → service → Environment), pointing at the
**Internal Database URL** of the Render Postgres instance — the private,
same-network connection string, not the external one. It is never stored in
the GitHub repository, in GitHub Secrets, or in the workflow file; it only
exists inside Render's own environment variable store for that service.

No manual migration step is needed. `src/server.js` runs
`runMigrationsWithRetry()` on every boot before it starts listening — the
migrations in `src/db/migrations/` are idempotent (`CREATE TABLE IF NOT
EXISTS`, a guarded seed insert), so they're safe to replay on every deploy.
The seed migration inserts two rows ("Alpha Item", "Beta Item") the first
time the `items` table is empty, which is what makes `/items` non-empty on a
fresh database.

## Verification

After triggering the deploy, the `deploy` job polls the live service through
`STAGING_URL` (a GitHub **repository variable**, not a secret — it holds no
sensitive information, just the public staging hostname, and is set without a
trailing slash):

- `GET $STAGING_URL/health` — must return HTTP 200. Confirms the process is
  up and responding at all; no database involved.
- `GET $STAGING_URL/items` — must return HTTP 200. Confirms the API can
  reach and query Postgres, since this route runs a `SELECT` against the
  `items` table.

Each check retries up to **10 times, 10 seconds apart** (a 100-second budget
per endpoint, bounded — not unlimited retries). If either endpoint never
returns 200 within its budget, the step exits non-zero and the `deploy` job
fails, even though the Render deploy itself may have "succeeded" from
Render's point of view. A green `deploy` job is the only signal that both
process liveness and database-backed behavior are confirmed.

To verify independently of CI, from any machine:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" "$STAGING_URL/health"
curl -s "$STAGING_URL/health"

curl -s -o /dev/null -w "HTTP %{http_code}\n" "$STAGING_URL/items"
curl -s "$STAGING_URL/items"
```

Expect `HTTP 200` both times, `{"status":"ok"}` from `/health`, and a JSON
array (at least "Alpha Item" and "Beta Item") from `/items`.

## Rolling back

Every build tags the image twice: `ghcr.io/<owner>/<repo>:<commit-sha>` (an
immutable, commit-addressable tag) and `:latest` (the mutable tag Render
actually deploys). To roll back:

1. Find the commit SHA of the last known-good build — either from
   `git log` on `main`, or from the GHCR package's version history (each
   version is labeled with its SHA tag).
2. On Render, open the web service → **Settings** → **Image**. Change the
   image reference from `ghcr.io/<owner>/<repo>:latest` to
   `ghcr.io/<owner>/<repo>:<good-sha>`.
3. Trigger a manual deploy (**Manual Deploy** button, or the same API call
   used by CI) to pull that exact tag.
4. Re-run the verification requests above against `STAGING_URL` to confirm
   the rollback is healthy.
5. Once satisfied, either leave the service pinned to that SHA tag
   (deliberately skipping `:latest` until the issue is fixed), or point it
   back at `:latest` after a corrected commit has been built and pushed.

Because the SHA tag is immutable, this always redeploys the exact image that
was built from that exact commit — never a moving target.

## Cleaning up the lab

When the staging environment is no longer needed:

1. **Delete the Render web service** (`hbtn-devops-pipeline-lab`) — service
   Settings → scroll to the bottom → Delete Web Service.
2. **Delete the Render Postgres database** (`hbtn-pipeline-lab-db`) the same
   way, from its own Settings page. It was always meant to be disposable.
3. **Revoke the Render API key** used by CI — Render → Account Settings →
   API Keys → delete the key that was created for this lab. This
   immediately invalidates `RENDER_API_KEY` even if it's still stored in
   GitHub.
4. **Remove the GitHub repository secrets and variable**: `Settings →
   Secrets and variables → Actions`, delete `RENDER_API_KEY` and
   `RENDER_SERVICE_ID` from the Secrets tab, and `STAGING_URL` from the
   Variables tab.
5. **Decide on the GHCR package visibility.** It was made public
   deliberately (see "Target" above) because the image carries no sensitive
   content. Leave it public if the package should remain available as a
   reference/example, or delete the package entirely (repository → Packages
   → `hbtn-devops-pipeline-lab` → Package settings → Delete this package) if
   nothing should remain reachable after the lab is torn down.

None of the values removed in steps 3–4 ever appeared in the workflow file,
the repository contents, or any commit — only their names (`RENDER_API_KEY`,
`RENDER_SERVICE_ID`, `STAGING_URL`) are referenced from `ci.yml`, so deleting
the secrets/variable and revoking the key is enough; there is nothing to
scrub from git history.