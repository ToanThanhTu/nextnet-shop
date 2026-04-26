# Next Net Shop

A learning-project e-commerce site. Two submodules behind a thin parent: a Next.js frontend and a .NET 9 backend, with PostgreSQL.

## Stack

- **Frontend** ([next-frontend/](next-frontend)): Next.js 15 (App Router, Turbopack), React 19, TypeScript, Tailwind CSS, Material UI, Redux Toolkit. Package manager: Bun.
- **Backend** ([net-backend/](net-backend)): .NET 9 Minimal API, EF Core 9, JWT bearer auth (config-driven via `JwtOptions`), NSwag/Swagger. Domain-driven feature folders with `RequireAuthorization`/`"Admin"` policies on writes.
- **Database**: PostgreSQL 17. Local dev runs in Docker; prod is an unmanaged Fly Postgres cluster (`nextnetshop-db`).
- **Deployment**: Frontend → Vercel. Backend → Fly.io (`nextnetshop-backend`).

## Repository layout

```
nextnet-shop/
├── docker-compose.yml          # Local dev: Postgres + backend (with hot reload)
├── prod-dump.dump              # gitignored, local copy of prod data, optional
├── net-backend/                # submodule → ToanThanhTu/nextnet-shop-backend
└── next-frontend/              # submodule → ToanThanhTu/nextnet-shop-frontend
```

`net-backend` and `next-frontend` are git submodules. Each has its own `README.md` and `CLAUDE.md` covering its stack-specific conventions.

## First-time setup on a fresh machine

A new machine needs prerequisites installed once, then the project itself bootstrapped through six steps.

### Prerequisites (one-time install)

| Tool | Why | Install |
|---|---|---|
| Git | Clone + submodules | `apt install git` (Linux) or system equivalent |
| Docker 20.10+ with the Compose plugin | Local Postgres + backend | https://docs.docker.com/engine/install/ |
| Bun 1.x | Frontend dev server + package manager | `curl -fsSL https://bun.sh/install \| bash` |
| .NET 9 SDK | EF migrations and (optional) host-level dotnet commands | https://dotnet.microsoft.com/download/dotnet/9.0 |
| `dotnet-ef` global tool | Run EF migrations | `dotnet tool install --global dotnet-ef` (then add `~/.dotnet/tools` to PATH) |
| `flyctl` (optional) | Required only for Path B in step 3 (prod restore) | https://fly.io/docs/flyctl/install/ |

Verify with `git --version`, `docker --version`, `docker compose version`, `bun --version`, `dotnet --version`, `dotnet ef --version`.

### 1. Clone with submodules

```bash
git clone --recurse-submodules <repo-url> nextnet-shop
cd nextnet-shop
```

If you cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

### 2. Start Postgres

```bash
docker compose up -d db
docker compose ps          # wait until db shows "healthy"
```

The container exposes Postgres on `localhost:15432`; data persists in the `pgdata` Docker volume.

### 3. Seed the database

Pick **one** path. The repo ships with a sample-catalog SQL file (`net-backend/seed-data.sql`) so a working homepage doesn't require Fly access.

#### Path A: Schema + sample catalog (no Fly access required)

```bash
# Create the schema from the InitialCreate migration
cd net-backend
dotnet ef database update         # connects to localhost:15432 via appsettings.json
cd ..

# Load the sample catalog (3 categories, 9 subcategories, 19 products, no users)
docker compose exec -T db psql -U nextnetshop_backend -d nextnetshop_backend \
  < net-backend/seed-data.sql
```

After this you have an empty users/cart/orders state and a populated catalog. Register a user via `/users/register` (Swagger or the frontend) when you need one.

#### Path B: Full restore from prod (requires Fly access)

Use this when you want the exact prod schema and all current data.

```bash
# Dump prod through a fly proxy
fly proxy 25432:5432 -a nextnetshop-db &
PROXY_PID=$!
PASSWORD=$(fly ssh console -a nextnetshop-db -C 'printenv OPERATOR_PASSWORD' 2>/dev/null | tr -d '\r\n')
docker run --rm --network host -e PGPASSWORD=$PASSWORD \
  -v "$PWD":/dump postgres:17-alpine \
  pg_dump -h localhost -p 25432 -U postgres -d nextnetshop_backend \
  -Fc --no-owner --no-acl --no-comments -f /dump/prod-dump.dump
kill $PROXY_PID

# Restore into the local container
docker compose exec -T db pg_restore -U nextnetshop_backend -d nextnetshop_backend \
  --no-owner --no-acl --clean --if-exists --exit-on-error \
  < prod-dump.dump

# Mark the InitialCreate migration as already applied (the dump's schema is the baseline)
docker compose exec -T db psql -U nextnetshop_backend -d nextnetshop_backend <<'SQL'
CREATE TABLE IF NOT EXISTS "__EFMigrationsHistory" (
  "MigrationId" character varying(150) NOT NULL,
  "ProductVersion" character varying(32) NOT NULL,
  CONSTRAINT "PK___EFMigrationsHistory" PRIMARY KEY ("MigrationId")
);
INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion")
VALUES ('20260426151538_InitialCreate', '9.0.2')
ON CONFLICT DO NOTHING;
SQL
```

### 4. Start the backend

```bash
docker compose up -d backend
docker compose logs -f backend       # watch until you see "Now listening on: http://0.0.0.0:8080"
curl -I http://localhost:8080/swagger # expect HTTP/1.1 200 OK
```

### 5. Start the frontend

```bash
cd next-frontend
bun install
bun dev
```

`.env.local` already points at `http://localhost:8080` for the backend.

### 6. Verify end-to-end

Open http://localhost:3000. The homepage should render with categories and products. Try registering a user and adding to cart.

---

## Production deploy checklist (Fly.io backend)

The backend reads its JWT signing key from the `Jwt__SigningKey` env var. In dev, `appsettings.json` has a placeholder key; in prod, set a real secret on the Fly app once:

```bash
fly secrets set Jwt__SigningKey=$(openssl rand -hex 32) -a nextnetshop-backend
fly secrets list -a nextnetshop-backend     # confirms the digest
```

Other prod env you may need to set:

| Secret / config | Purpose |
|---|---|
| `Jwt__SigningKey` | HMAC-SHA256 signing key for issued tokens |
| `Jwt__Issuer`, `Jwt__Audience` | Override `appsettings.json` defaults if you change them |
| `Cors__AllowedOrigins__0` | Frontend origin (e.g. `https://your-app.vercel.app`); add `__1`, `__2` for more |
| `DATABASE_URL` | Already set to the Fly Postgres `.flycast` URL |

Rotating `Jwt__SigningKey` invalidates every existing token (everyone gets logged out). That's the only failure mode worth knowing.

## Daily workflow (after setup)

```bash
docker compose up -d                    # Postgres + backend
cd next-frontend && bun dev             # frontend
docker compose logs -f backend          # backend logs
docker compose restart backend          # after DI/startup edits
docker compose down                     # stop (keeps DB data)
docker compose down -v                  # nuke (also wipes pgdata volume)
```

To re-seed after a wipe, repeat step 3.

## Sub-system docs

- [next-frontend/README.md](next-frontend/README.md): frontend setup, routes, component structure
- [next-frontend/CLAUDE.md](next-frontend/CLAUDE.md): App Router conventions, Redux patterns
- [net-backend/README.md](net-backend/README.md): backend setup, endpoint reference, EF Core
- [net-backend/CLAUDE.md](net-backend/CLAUDE.md): feature module pattern, gotchas, security notes

## Project-wide guidelines

See [CLAUDE.md](CLAUDE.md) at the root for cross-cutting conventions, dev commands, and the local DB workflow (including how to refresh the local schema from a prod dump).

## Submodule sync

```bash
# After pulling the parent repo
git submodule update --init --recursive

# Pull latest main on each submodule
git submodule update --remote --merge
```

The parent repo pins each submodule to a specific commit. After updating, commit the new pointers in the parent if you want them tracked.
