# Claude Development Guidelines (root)

Project-wide conventions and the local development workflow. Stack-specific rules live in each submodule's `CLAUDE.md`.

- [next-frontend/CLAUDE.md](next-frontend/CLAUDE.md)
- [net-backend/CLAUDE.md](net-backend/CLAUDE.md)

## Architecture

Both stacks follow Domain-Driven Design (per `~/.claude/rules/backend-ddd.md` and `~/.claude/rules/react-architecture.md`), with a shared `Modules/` (or `modules/`) parent folder that separates bounded contexts from cross-cutting infrastructure.

- **Frontend** (`next-frontend/src/modules/<domain>/`): products, categories, users, cart, orders. Each module owns its entities, API client, and (where applicable) Redux slice. UI components live in `next-frontend/src/app/components/`.
- **Backend** (`net-backend/Modules/<Feature>/`): each module is a bounded context with full DDD layers: `Domain/` (interfaces + domain services), `Infrastructure/` (EF Core repositories), `Application/Queries/` and `Application/Commands/` (one Handler per use case), `Contracts/` (DTOs + validated requests), plus a thin MVC `<Feature>Controller` and `<Feature>Module` for DI registration. Cross-cutting code stays at the project root: `Configuration/` (service registration), `Common/` (Auth helpers + typed exception hierarchy), `Data/` (DbContext), `Migrations/`.

Authorization is attribute-driven at the controller (`[Authorize]`, `[Authorize(Policy = "Admin")]`); the `"Admin"` policy and JWT issuance/validation share a single `JwtOptions` config section. User identity for "operate on the caller's own data" endpoints comes from the JWT `NameIdentifier` claim via `User.GetRequiredUserId()`, never from a path parameter.

Domain exceptions extend `AppException` and map to RFC 7807 ProblemDetails responses via a global `IExceptionHandler`. Multi-aggregate writes (e.g. `OrderPlacement.PlaceFromCartAsync`) wrap in `BeginTransactionAsync`.

## Stack at a glance

| Layer | Tech | Runs in |
|---|---|---|
| Frontend | Next.js 15 (App Router, Turbopack), React 19, TypeScript, Tailwind, MUI, Redux Toolkit | host (Bun) |
| Backend | .NET 9 Minimal API, EF Core 9, NSwag, JWT | Docker (dotnet watch) |
| Database | PostgreSQL 17 | Docker (`postgres:17-alpine`) |

Frontend deploys to Vercel; backend deploys to Fly.io (`nextnetshop-backend`); prod database is an unmanaged Fly Postgres cluster (`nextnetshop-db`).

## Local development

The recommended local setup mirrors prod for the bits prod actually containerises (backend + Postgres) and runs the frontend natively.

```bash
# Postgres + backend (with hot reload via dotnet watch)
docker compose up -d

# Frontend (separate terminal)
cd next-frontend
bun install
bun dev
```

URLs:
- Frontend: http://localhost:3000
- Backend: http://localhost:8080
- Swagger: http://localhost:8080/swagger
- Postgres: `localhost:15432` (host) / `db:5432` (compose network), DB `nextnetshop_backend`, user `nextnetshop_backend`

### Hot reload behaviour

- **Frontend**: Turbopack handles all edits. No restart needed.
- **Backend**: `dotnet watch` picks up most edits via hot reload. Edits to DI/startup wiring (`Program.cs`, `ConfigureServices.cs`, `ConfigureApp.cs`) need a full restart:
  ```bash
  docker compose restart backend
  ```
  The polling file watcher (`DOTNET_USE_POLLING_FILE_WATCHER=1`) is set in compose so the bind mount triggers reloads reliably.

### Resetting / seeding the local DB

The repo includes a `prod-dump.dump` (gitignored): a snapshot of the prod data. Use it to seed or reset the local database.

Take a fresh snapshot from prod (needs `fly proxy`):

```bash
fly proxy 25432:5432 -a nextnetshop-db &
PASSWORD=$(fly ssh console -a nextnetshop-db -C 'printenv OPERATOR_PASSWORD' 2>/dev/null | tr -d '\r\n')
docker run --rm --network host -e PGPASSWORD=$PASSWORD \
  -v "$PWD":/dump postgres:17-alpine \
  pg_dump -h localhost -p 25432 -U postgres -d nextnetshop_backend \
  -Fc --no-owner --no-acl --no-comments -f /dump/prod-dump.dump
kill %1
```

Restore into the local container:

```bash
docker compose exec -T db pg_restore -U nextnetshop_backend -d nextnetshop_backend \
  --no-owner --no-acl --clean --if-exists --exit-on-error \
  < prod-dump.dump
```

`--no-owner` and `--no-acl` are required because the local container only has the `nextnetshop_backend` role; the prod dump references roles that don't exist locally.

### Connecting GUI tools (DBeaver) to prod

Fly Postgres is on a private 6PN network (`.flycast` / `.internal`); those hostnames don't resolve from your laptop. Use `fly proxy` to tunnel:

```bash
fly proxy 25432:5432 -a nextnetshop-db
# Then connect DBeaver to localhost:25432 with user "postgres" and the
# password from: fly ssh console -a nextnetshop-db -C 'printenv OPERATOR_PASSWORD'
```

Pick any free local port. Don't reuse `15432`; that's the local container.

## Tooling

### Frontend

```bash
cd next-frontend
bun dev                  # Turbopack dev server
bun build                # production build
bun lint                 # ESLint
npx tsc --noEmit         # type check
```

### Backend

Inside the container:
```bash
docker compose exec backend dotnet build
docker compose logs -f backend
```

On the host (requires `dotnet tool install --global dotnet-ef` once):
```bash
cd net-backend
dotnet ef migrations add <Name>          # generates migration code
dotnet ef migrations list                # connects to localhost:15432
dotnet ef database update                # applies pending migrations
dotnet ef migrations remove              # undo last (before applying)
dotnet csharpier .                       # format
```

The `appsettings.json` connection string points at `localhost:15432` so host-level `dotnet ef` works against the Docker DB. Inside the backend container, the env var `ConnectionStrings__DATABASE_URL` overrides it to `db:5432`.

## Submodule workflow

```bash
git submodule update --init --recursive    # after first clone
git submodule update --remote --merge      # pull latest main on each submodule
```

When committing changes that span the parent and a submodule, commit inside the submodule first, then commit the updated submodule pointer in the parent. Don't push the parent without pushing the submodule, or others will get a "missing commit" error.

## Production secrets (Fly.io)

The backend reads `Jwt__SigningKey` from env. Set it once per Fly app:

```bash
fly secrets set Jwt__SigningKey=$(openssl rand -hex 32) -a nextnetshop-backend
fly secrets list -a nextnetshop-backend
```

Rotate by re-running `fly secrets set` with a new value; all existing JWTs become invalid (everyone is logged out). The dev value in `appsettings.json` is a non-secret placeholder and is fine to commit.

`Cors__AllowedOrigins__0` should be set to the prod frontend origin (e.g. the Vercel URL).

## Style and conventions

- Conventional Commits: `type(scope): subject`. Common types: `feat`, `fix`, `refactor`, `docs`, `chore`, `perf`. Examples: `feat(wishlist): add controller`, `refactor(products): migrate to MVC`.
- One concern per commit. Don't mix backend and frontend in a single commit; they live in different submodules anyway.
- No em-dashes in committed prose (docs, READMEs, commit messages, code comments). Use colons, semicolons, periods, or rewrite.
- Don't commit secrets. `.env` files are gitignored; verify with `git check-ignore <file>` before committing anything that looks credential-shaped.
- Don't commit `prod-dump.dump` or any `*.sql.gz` files; they're gitignored at root.
