# Deploying greenplan on a cloud VM

Both services (`backend`, `frontend`) run as sibling Docker Compose services
behind one Caddy instance, on two subdomains of the same VM. This directory
(`ci/`) holds the three files that glue them together — `docker-compose.yml`,
`Caddyfile`, this doc — and is itself plain, hand-maintained files on the VM,
not tracked by either repo's git history (see the comment at the top of
`docker-compose.yml`). Each repo's own `DEPLOY.md` (`../backend/DEPLOY.md`,
if still present) covers concerns specific to that service only; this is the
one place that covers the whole stack.

## 1. VM prerequisites

Any small Ubuntu-like cloud VM works (matches the ТЗ's МосТех.ОС target); 1-2
vCPU / 2GB RAM is enough (`GREENPLAN_API_MAX_CONCURRENT_JOBS` in
`docker-compose.yml` controls how many DXF-processing jobs the backend runs
at once — keep it low on a small VM).

```bash
# on the VM
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # log out/in once for this to take effect
```

## 2. DNS

Point A (and AAAA, if the VM has IPv6) records at the VM's public IP for
both subdomains:

- `backend.greenleaders.online`
- `app.greenleaders.online`

Caddy requests a TLS certificate from Let's Encrypt automatically the first
time it sees a request for each hostname, which only works once DNS has
propagated and ports 80/443 are open on the VM's firewall/security group.

## 3. First-time setup

**If `/opt/greenplan/backend` already has its own `docker-compose.yml` running**
(the pre-frontend, backend-only setup) **, stop it first:**

```bash
# on the VM
cd /opt/greenplan/backend
docker compose down   # frees ports 80/443 -- its own caddy is bound to them too
```

Its `caddy` service is a *separate* Compose project from the one in `ci/`
(different project directory, different project name), so it holds ports
80/443 in its own right -- the new stack's `caddy` can't bind them until the
old one is stopped. Skipping this step is exactly what produces
`Bind for 0.0.0.0:80 failed: port is already allocated` on the first
`docker compose up` below. This is also why the backend goes briefly
offline during this one-time migration (the reconnect window mentioned in
the backend repo's own migration notes) -- there's no way to hand off a
bound port between two Compose projects without a gap.

```bash
# on the VM
sudo mkdir -p /opt/greenplan && sudo chown "$USER" /opt/greenplan
cd /opt/greenplan
git clone <backend repo URL> backend
git clone <frontend repo URL> frontend
git clone <ci repo URL, if ci/ is its own repo -- otherwise copy this directory's 3 files by hand> ci

cd ci
docker compose up -d --build
docker compose logs -f   # watch everything come up; Ctrl-C to stop watching
```

That's it — `docker compose up -d --build` builds and starts `backend` and
`frontend` (see "What `frontend` actually is" below) and starts `caddy`,
which terminates TLS and reverse-proxies both subdomains over the internal
Docker network; neither `backend` nor `frontend` publishes ports directly.

### What `frontend` actually is

Unlike `backend`, `frontend` is not a long-running service — its image just
builds the SPA and its container copies `dist/` into the `frontend_data`
volume, then exits (exit code 0 is expected and correct, check
`docker compose ps` if unsure). Caddy mounts that same volume and serves the
files directly — there's no separate web server for the frontend, and no
nginx anywhere in this stack. See `../frontend/Dockerfile` and the `caddy`
service's `app.greenleaders.online` block in `Caddyfile`.

## 4. Verify

```bash
curl https://backend.greenleaders.online/openapi.json    # the OpenAPI schema
curl https://app.greenleaders.online/                      # the SPA shell
curl https://app.greenleaders.online/config.json            # runtime config
curl https://app.greenleaders.online/api/openapi.json        # proxied to backend, same origin
```

Open `https://backend.greenleaders.online/docs` for FastAPI's interactive
Swagger UI.

## 5. The basemap archive (optional, degrades gracefully without it)

The PMTiles basemap archive (`moscow.pmtiles`) is local data, not in git --
same as `.data/basemap/` in frontend local development (see
`../frontend/vite.config.ts`). If you have the file, copy it into the
`basemap_data` volume once:

```bash
docker compose cp moscow.pmtiles caddy:/srv/basemap/moscow.pmtiles
```

Without it, `/basemap/*` 404s and the app just shows the map without a
basemap layer (see `../frontend/src/shared/map/map-view.tsx`'s "missing"
state) -- not an error, no redeploy needed once you do add it.

## 6. Data persistence & backups

Backend project data (`GREENPLAN_API_DATA_DIR=/data` inside the container)
lives in the named volume `backend_data`, not in the container filesystem,
so `docker compose up -d --build` to deploy a new backend image never
touches it. Back it up with:

```bash
docker run --rm -v ci_backend_data:/data -v "$PWD":/backup alpine \
    tar czf /backup/greenplan-data-$(date +%F).tar.gz -C /data .
```

(The `ci_` volume-name prefix is Compose's project-name default, taken from
this directory's name -- check the actual name with `docker volume ls` if
you've overridden the project name.)

## 7. Continuous deployment (auto-rebuild on push)

Each repo's own `.github/workflows/deploy.yml` does the equivalent of
`git reset --hard origin/main` in its own checkout, then
`docker compose up -d --build <service>` from this directory, automatically,
over SSH, on every push to that repo's `main` (and on-demand via the Actions
tab's "Run workflow" button). The frontend workflow also runs
lint/typecheck/test/build first and only deploys if those pass. Builds still
happen *on the VM* -- same `ci/docker-compose.yml`, no container registry
involved.

**One-time VM setup** -- a dedicated SSH keypair for the Actions (don't reuse
your personal key; both repos' workflows can share one keypair, since
they SSH to the same VM):

```bash
# on your own machine, not the VM
ssh-keygen -t ed25519 -C "github-actions-deploy" -f deploy_key -N ""
ssh-copy-id -i deploy_key.pub <user>@backend.greenleaders.online
```

**One-time GitHub setup** -- in *each* repo's Settings -> Secrets and
variables -> Actions (secrets don't carry over between repos, even when
both deploy to the same VM), add:

- `DEPLOY_SSH_KEY` -- contents of `deploy_key` (the private half; delete the
  local copy once it's pasted into both repos)
- `DEPLOY_HOST` -- `backend.greenleaders.online` (or the VM's IP) -- the SSH
  target, same value in both repos regardless of which subdomain each one
  deploys
- `DEPLOY_USER` -- the VM username from `ssh-copy-id` above

**Consequences worth knowing:**

- Both workflows do `git reset --hard origin/main` on the VM, not `git pull`
  -- deterministic (no merge commits, no drift), but it means **don't
  hand-edit anything in `/opt/greenplan/backend` or `/opt/greenplan/frontend`
  on the VM** -- any local edit gets silently discarded on the next push.
  Make changes via commits. `ci/`'s three files are the exception -- they
  aren't reset by either workflow, so hand edits there persist (but also
  aren't backed up by git unless you commit them somewhere yourself).
- If a pushed commit fails to build (or, for frontend, fails a test), the
  deploy step never runs, the workflow run shows red in the Actions tab, and
  -- importantly -- the *previously running* container for that service is
  untouched and keeps serving. A broken push fails loudly without taking
  anything down.
- To roll back, `git revert` the bad commit and push (triggers a normal
  redeploy), or SSH in directly and `git reset --hard <good-sha> && cd
  /opt/greenplan/ci && docker compose up -d --build <service>`.
- Redeploying one service never restarts `caddy` or the other service -- no
  TLS cert reissue or cross-service downtime from either repo's deploy.

## 8. One important constraint (backend)

Don't add `--workers N` (N>1) to the uvicorn command in `../backend/Dockerfile`,
and don't run more than one `backend` container/replica. The concurrency-limit
gate (`GREENPLAN_API_MAX_CONCURRENT_JOBS`) and the metadata cache are both
in-memory state private to a single process; more than one process/replica
means each gets its own independent limit and its own possibly-stale cache.
If you need more throughput, give the one container more CPU/RAM, or (bigger
change, not done here) move that shared state into something external
(Redis, a file lock, etc.) first.
