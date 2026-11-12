# Adeel Traders — Development Setup

> For developers on the Adeel Traders project. Confidential (Nesway + Adeel Traders).
> Based on the official frappe_docker development docs + VS Code "Improve disk performance" guide.

## 1. What runs where

```
Your computer                                   Docker (frappe_docker devcontainer)
─────────────                                   ───────────────────────────────────
adeel-traders/
├── apps/adeel_traders    ──bind mount──►  benches/frappe-bench/apps/adeel_traders
├── apps/pk_integrations  ──bind mount──►  benches/frappe-bench/apps/pk_integrations
├── web/  (Next.js, runs natively: npm)     benches/  = named volume "benches" (fast disk)
├── docs/                                   ├── frappe-bench (frappe, erpnext, hrms, lending,
└── frappe_docker/ (official, env only)     │   payments, webshop, env, node_modules)
                                            MariaDB 11.8 · Redis cache · Redis queue
```

| Rule | Why |
|---|---|
| Our code = `apps/*` and `web/` (Git repos) | Stays on your computer, safe, editable |
| Frappe/ERPNext live in the **named volume** | Bind mounts on macOS are slow (we measured `bench migrate` 212 s → 41 s after moving) |
| 🚫 Never run `bench get-app` for our apps | It **deletes** an existing app folder (`shutil.rmtree` in bench source) |
| 🚫 Never edit apps inside the volume (frappe, erpnext…) | Core is never changed |

## 2. Versions (must match everywhere)

| Tool | Version |
|---|---|
| Frappe / ERPNext / HRMS / Lending / Payments / Webshop | branch `version-16` (stable) |
| Python | 3.14 (in the bench image) |
| Node | 24 |
| MariaDB | 11.8 |
| Our apps | branch `develop` |

## 3. First-time setup

```bash
# 1. Get the code
mkdir adeel-traders && cd adeel-traders
git clone <adeel_traders repo>   apps/adeel_traders
git clone <pk_integrations repo> apps/pk_integrations
git clone <web repo>             web
git clone https://github.com/frappe/frappe_docker.git

# 2. Devcontainer config (official step)
cd frappe_docker
cp -R devcontainer-example .devcontainer
```

3. Edit `.devcontainer/docker-compose.yml`, `frappe` service (copy from an existing team member's file):
   - `environment`: add `FRAPPE_BIND_ADDR=0.0.0.0`; `ports`: `127.0.0.1:8000-8005` and `127.0.0.1:9000-9005` (official CLI-only setup)
   - `volumes`: add `benches:/workspace/development/benches`; `working_dir: /workspace/development/benches`
   - top-level `volumes`: add `benches:`
   - ⚠️ do **not** add the app bind mounts yet
4. `.devcontainer/devcontainer.json`: `"workspaceFolder": "/workspace/development/benches"`
5. Copy `development/apps-adeel.json` (list of the 6 Frappe apps, `version-16`).

```bash
# 6. Start and prepare the volume
docker compose -f .devcontainer/docker-compose.yml up -d
docker exec -u root devcontainer-frappe-1 chown frappe:frappe /workspace/development/benches
docker exec devcontainer-frappe-1 bash -lc 'cd /workspace/development/benches && cp -R ../vscode-example .vscode'

# 7. Official installer, run INSIDE the volume (so bench init really runs)
docker exec -w /workspace/development/benches devcontainer-frappe-1 \
  bash -lc 'python ../installer.py -j ../apps-adeel.json -s adeel.localhost -t version-16'
```

8. Now add the 2 bind mounts to the `frappe` service and recreate the container:
```yaml
      - ../../apps/adeel_traders:/workspace/development/benches/frappe-bench/apps/adeel_traders:cached
      - ../../apps/pk_integrations:/workspace/development/benches/frappe-bench/apps/pk_integrations:cached
```
```bash
docker compose -f .devcontainer/docker-compose.yml up -d
```

9. Register our apps with bench's own `install_app()` (same steps `bench get-app` does after cloning, without deleting anything):
```bash
docker exec -w /workspace/development/benches/frappe-bench devcontainer-frappe-1 bash -lc '
BPY=$(head -1 $(which bench) | sed "s/^#!//")
for a in pk_integrations adeel_traders; do
  $BPY -c "from bench.utils import setup_logging; setup_logging()
from bench.app import install_app
install_app(\"$a\", bench_path=\".\", restart_bench=False)"
done
bench --site adeel.localhost install-app pk_integrations adeel_traders
bench validate-dependencies'
```
If an asset build fails at this step, run `bench build --app <app>` once more.

## 4. Daily use

```bash
# Start (no file watcher — it used ~60% CPU while idle)
docker exec -d -w /workspace/development/benches/frappe-bench devcontainer-frappe-1 \
  bash -lc 'honcho start web socketio schedule worker > logs/honcho.log 2>&1'
```
- ⚠️ After your Mac **sleeps**, the worker loses Redis (`Redis connection timeout, quitting`) and honcho then stops **all** processes, so the site goes down. Just run the start command above again. (Production does not have this problem: supervisor/Docker restart processes automatically.)
- Site: **http://adeel.localhost:8000** · `Administrator` / `admin` (dev only)
- Staff app: **http://localhost:3000**. Staff (everyone except admins and the accountant) who open Desk are sent there. The address comes from the site config key `staff_app_url` (dev: `bench --site adeel.localhost set-config staff_app_url "http://localhost:3000"`; production: the real staff app URL, or leave it out when the app is served at `/` on the same domain).
- **Go-live: the site's public address.** Share links (WhatsApp, email) are built from the site config key `host_name`. In dev they start with `http://127.0.0.1:8000`, which opens only on this computer. Before go-live set it to the real address: `bench --site <site> set-config host_name "https://erp.example.com"`. Phones can only share the PDF file itself over HTTPS.
- Editing Desk JS/CSS? Also run `bench watch` (or `bench build --app <app>`).
- Shell: `docker exec -it -w /workspace/development/benches/frappe-bench devcontainer-frappe-1 bash`
- VS Code alternative (official): open `frappe_docker` → **Dev Containers: Reopen in Container**.
- Stop: `docker compose -f .devcontainer/docker-compose.yml stop`
- After the container is **recreated**, reinstall the lint tool (the devcontainer's `postCreateCommand` does this in VS Code):
  `docker exec devcontainer-frappe-1 bash -lc 'uv tool install pre-commit'`
- Tests (dev site has `allow_tests` on):
  `bench --site adeel.localhost run-tests --module adeel_traders.setup.test_core_setup`
- Apply our setup/patches to the site: `bench --site adeel.localhost migrate`
  - Setup rule: every setting lives in `setup/core_setup.run_setup` (a fresh site runs it on install). Add a file to `patches.txt` only when an **existing** site must change, and put the same logic in `run_setup` so a fresh site and a migrated site end up identical. The 14 old wrapper patches (v0_1 to v0_14) were removed on 12 Nov 2026 after a before/after database comparison showed no difference.
- Demo products (dev sites only; refuses without developer_mode). Run from `frappe-bench/sites`, because `bench execute` hides errors raised inside the function:
  ```bash
  ../env/bin/python -c "import frappe; frappe.init(site='adeel.localhost', sites_path='.'); frappe.connect(); from adeel_traders.setup.demo import create_demo_data; create_demo_data(); frappe.destroy()"
  ```

Next.js:
```bash
cd web && npm install
npm run dev --workspace staff        # http://localhost:3000  (staff app)
npm run dev --workspace storefront   # http://localhost:3001
```

How the staff app talks to Frappe (same origin, no CORS, no API keys in the browser):
- `apps/staff/next.config.ts` rewrites `/api/*`, `/files/*`, `/private/files/*` to `FRAPPE_URL`
  (default `http://127.0.0.1:8000`).
- Frappe's dev server picks the site from the hostname, so the bench's `Procfile` `web:` line is
  pinned to our site (official CLI option `bench --site … serve`):
  `web: bench --site adeel.localhost serve  --port 8000`
  (in production, Nginx passes the site name instead).
- Login uses Frappe's `/api/method/login`; every write sends the `X-Frappe-CSRF-Token` header from
  `adeel_traders.api.v1.session.me`. All permissions are checked by Frappe.

## 5. Measured performance (Intel i5, 8 GB to Docker)

| Action | Bind mount (old) | Named volume (now) |
|---|---|---|
| List all bench files (127k) | 30.8 s | 2.4 s |
| `bench migrate` (no changes) | 212 s | 41 s |
| `bench build --app adeel_traders` | 35 s | 13 s |
| Idle CPU of bench container | ~60% | ~13% |
