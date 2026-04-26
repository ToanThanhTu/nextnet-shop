# AuditCover Claude Code Demo — Prep

**Audience:** CEO + COO
**Format:** Zoom, screen-share, live coding
**Meeting:** Tuesday (3 days out)
**Assumed meeting length:** 30–45 min; demo portion probably 20–25 min

---

## The plan in one paragraph

Use `nextnet-shop` — your existing personal project (Next.js frontend + .NET 9 + EF Core + Postgres, already deployed to Vercel + Fly.io). Spend Friday stabilizing it (the 2-year-old dependencies will need updating). Spend Saturday migrating the backend from Minimal API to MVC Controllers, which doubles as your .NET ramp-up exercise *and* brings the stack closer to AuditCover's. Sunday, rehearse adding a Wishlist feature on a throwaway branch. Tuesday in the meeting, add the Wishlist feature live, end-to-end.

## Why this is the right plan

The CEO asked to see **how you use Claude Code in day-to-day development work**. He didn't ask to see a bespoke audition piece. `nextnet-shop` is what you've actually worked on. It's on your public GitHub. It's in the language he cares about. Demoing a project you genuinely own and will keep maintaining is more authentic than scaffolding something fresh.

Facts he already knows:
- You don't have commercial .NET experience (read from your resume)
- Your current work is TypeScript / Node (told him on call one)
- You use Claude Code heavily (why he scheduled this call)

What he's actually verifying:
- Can you drive Claude Code fluently on code you know?
- When Claude gets something wrong, do you catch it?
- Is your AI workflow real or marketing?

None of those require a new project.

---

## 3-day schedule

### Friday (today) — stabilize

**Target: 3–4 hours**

The project is 2 years stale. Dependencies are definitely behind. Getting it running is Friday's whole job.

1. **Commit the current state first** so you can revert if you break something:
   ```bash
   cd ~/projects/trevor/nextnet-shop
   git status
   # if dirty, commit or stash
   ```

2. **Sync submodules**:
   ```bash
   git submodule update --init --recursive
   cd net-backend && git pull origin main
   cd ../next-frontend && git pull origin main
   ```

3. **Backend: restore + run**:
   ```bash
   cd net-backend
   dotnet restore
   dotnet run
   ```
   Fix anything that breaks. Likely issues: EF Core migration format changes, Npgsql upgrades, JWT package name changes.

4. **Frontend: install + run**:
   ```bash
   cd ../next-frontend
   bun install
   bun dev
   ```
   Likely issues: Next.js major version bumps, React 18→19 compatibility, MUI/Tailwind version skew.

5. **End-to-end verify**: open `http://localhost:3000`, browse products, log in, add to cart. Confirm the whole flow still works. If something's broken, fix it now.

6. **Optional Friday reading (60 min max)** — refresh these specifically:
   - [ASP.NET Core MVC Controllers overview](https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/actions) — because Saturday's refactor targets this
   - EF Core 9 migrations
   - C# 13 primary constructors and collection expressions

**End-of-Friday state:** both servers start cleanly, you can browse the site end-to-end, nothing on fire.

**If the project is unrecoverable** (unlikely but possible): stop, pivot to a different Plan B. Don't burn Saturday on Friday work.

---

### Saturday — migrate Minimal API → MVC Controllers

**Target: 4–5 hours**

This is the bulk of your .NET ramp-up. AuditCover's codebase uses MVC Controllers + Razor + EF + SQL Server. Your `net-backend` currently uses Minimal API. Migrating closes the biggest stack gap. The migration is also **pure Claude Code workflow practice** — exactly what you'll be doing on Tuesday.

#### Migration approach

One module at a time. Commit after each. Verify after each.

**Suggested order** (low-risk to high-risk):

1. **Categories** (smallest, simplest CRUD) — 30 min
2. **Products** (biggest module, sets the pattern) — 90 min
3. **Users / Auth** (involves JWT, trickier) — 60 min
4. **Cart** — 45 min
5. **Orders** — 45 min

#### How to migrate one module (use this pattern)

1. **Read the existing Minimal API endpoints** for the module (e.g. `Products/ProductsEndpoints.cs` or wherever the `MapGet`/`MapPost` calls live):
   ```
   /read net-backend/Products/ProductsEndpoints.cs
   ```

2. **Ask Claude Code to convert** with specific context:
   > "Convert this Minimal API file to an MVC Controller. Target: `Controllers/ProductsController.cs`. Use `[ApiController]`, `[Route("api/[controller]")]`, constructor injection for AppDbContext, and preserve all existing routes and logic. The old file will be deleted once you confirm the controller compiles and the routes still work."

3. **Review the generated controller diff before accepting.**

4. **Update `Program.cs`** — remove the `MapGroup(...)` or whatever wires the old endpoints, add `builder.Services.AddControllers()` + `app.MapControllers()` if not already present.

5. **Delete the old endpoints file.**

6. **Verify**:
   ```bash
   dotnet build
   dotnet run
   # hit the endpoints via curl or Swagger
   ```

7. **Commit**:
   ```bash
   git add .
   git commit -m "refactor(products): migrate from Minimal API to MVC Controller"
   ```

8. **Next module.**

#### What to watch for during migration

- **Route conventions**: `[Route("api/[controller]")]` yields `/api/products`. If the old routes were `/api/product` (singular), override with `[Route("api/product")]` to preserve URLs.
- **Result types**: `Results.Ok(x)` in Minimal API becomes `Ok(x)` in a controller (the method is on `ControllerBase`). `Results.NotFound()` becomes `NotFound()`.
- **DI**: parameters used to be function arguments; now they come through the constructor and land on a private field.
- **Async signatures**: keep them async. `Task<IActionResult>` is the standard return type for async actions.
- **Authorization**: `.RequireAuthorization()` becomes `[Authorize]` at the class or method level.

This migration will teach you more about MVC Controllers in one afternoon than any tutorial. You're doing the real thing.

**End-of-Saturday state:** backend runs entirely on MVC Controllers. Frontend still works (no URL changes). Five commits on main, clean history.

---

### Sunday — rehearse Wishlist feature

**Target: 2–3 hours**

Sunday is the live-feature rehearsal. The goal: by end of day, you've built the wishlist feature twice (once on a rehearsal branch, then deleted it). Tuesday you build it a third time, from clean main, in front of the CEO.

#### Wishlist spec

Users can favorite products. Heart icon on the product card toggles favorite state. New `/wishlist` page lists saved products. Persists to the backend per authenticated user.

**Backend** (`net-backend/`):
- New `WishlistItem` entity: `Id`, `UserId`, `ProductId`, `CreatedAt`
- EF migration with a unique constraint on `(UserId, ProductId)`
- New `WishlistController` with three endpoints:
  - `GET /api/wishlist` — current user's wishlist (populated with product info)
  - `POST /api/wishlist/{productId}` — add to wishlist (idempotent; returns existing if present)
  - `DELETE /api/wishlist/{productId}` — remove from wishlist
- All three require `[Authorize]`

**Frontend** (`next-frontend/`):
- Heart icon button component (filled when favorited, outline when not)
- Add the button to the product card
- New `/wishlist` page that fetches from `/api/wishlist` and renders product cards
- Redux slice (or whatever pattern nextnet-shop currently uses) for wishlist state
- API client methods for the three endpoints

#### Rehearsal protocol

1. **Branch**: `git checkout -b rehearsal/wishlist`

2. **Dry run 1**: build the feature end-to-end with Claude Code. Time yourself. Aim for 20 min.

3. **Note every hiccup**:
   - Did `dotnet ef migrations add WishlistFeature` fail? Why?
   - Did the frontend type errors require fixes Claude missed?
   - Did the JWT middleware not attach `User.Identity` the way you expected?
   - Each hiccup is a rehearsal gift — fix it in main so it won't happen Tuesday.

4. **Reset**:
   ```bash
   git checkout main
   git branch -D rehearsal/wishlist
   ```

5. **Fix anything main needs** so the Tuesday run will be smooth. Commit those fixes.

6. **Dry run 2** (if time allows): do the wishlist again on a new rehearsal branch. Should be noticeably smoother.

7. **Delete the second rehearsal branch.**

8. **Prepare talking points** (30 min):
   - Memorize the opening and closing lines
   - Pick which two Claude Code skills/rules you'll show in the setup tour
   - Decide what you'll "push back on" during plan mode

9. **Stop by 9 PM. Rest.**

---

### Monday — no new code

**Target: 45 min, morning**

1. `dotnet run` + `bun dev` — confirm both start cleanly
2. Open `http://localhost:3000`, browse, log in, add-to-cart. Everything works.
3. Read this document once.
4. Re-read the opening and closing scripted lines until they flow.
5. Close the laptop.

**Absolutely no new code today.** Arriving fresh beats arriving crammed.

---

### Tuesday — meeting day

- 30 min before: both servers up, VS Code ready, Claude Code CLI logged in on Opus 4.7
- Zoom tested, mic + screen share working
- Notifications off
- Water nearby
- This file on a second monitor or printed (not on shared screen)

---

## Meeting structure (25 min demo)

### 00:00–02:00 — Opening

**Say:**

> "I'll demo with nextnet-shop, a personal e-commerce project I have on GitHub. Next.js frontend, .NET 9 with MVC Controllers, EF Core, Postgres, JWT auth. I hadn't touched it in about two years, so I spent the weekend getting it back up and refactoring the backend from Minimal API to MVC Controllers so it's closer to your stack. Using Claude Code throughout. What you'll see today is me adding a Wishlist feature end to end — heart icon, wishlist page, backend endpoints. Should take about 20 minutes. Please interrupt anytime."

**Show:**

- VS Code open on `nextnet-shop` root
- Browser at `http://localhost:3000` showing the homepage
- `dotnet run` log in one terminal pane
- `bun dev` log in another
- Claude Code CLI in a third

### 02:00–04:00 — Tour the Claude Code setup

**Show, briefly:**

- Repo-level `CLAUDE.md`. "This file tells Claude the project conventions. I set it up early on any project — it's the first thing I do."
- One or two rule files from `~/.claude/rules/`. "These are cross-project rules — things like no `any` in TypeScript, always parameterize queries."
- One or two skills from `~/.claude/skills/`. "Reusable workflows. `/commit-message` for Conventional Commits, `/update-docs` for keeping JSDoc / CLAUDE.md in sync after changes."

**Keep this to 2 minutes.** It's a setup tour, not a training session.

### 04:00–07:00 — Describe feature + plan mode

**Say the spec out loud:**

> "Wishlist feature. Users hit a heart icon on a product to favorite it. A dedicated `/wishlist` page shows their saved products. Persists per authenticated user. Three backend endpoints, a heart component, a page."

**Enter plan mode** (`Shift+Tab` twice) and send:

> "Add a Wishlist feature to this project. Backend: new WishlistItem entity with UserId, ProductId, CreatedAt, unique constraint on (UserId, ProductId). EF migration. New WishlistController with GET /api/wishlist, POST /api/wishlist/{productId}, DELETE /api/wishlist/{productId}. All require [Authorize]. Frontend: heart icon toggle on the product card, /wishlist page, Redux slice for state. Plan the full implementation end to end. Call out the migration, controller methods, DTOs if needed, frontend components, and state wiring."

**Wait for the plan. Read it out loud.** Then **push back on at least one thing** — any of these work:

- "I want POST to be idempotent — if the item's already in the wishlist, return the existing record, don't throw a duplicate-key error."
- "Let me name the controller route `/api/wishlist` not `/api/wishlist-items` so it's cleaner."
- "I'll skip the Redux slice for now. This project uses Redux but the wishlist is small enough that a React context or TanStack Query might be simpler. Let me think about it after I see the backend."

**Exit plan mode** and go.

### 07:00–20:00 — Implementation

**Narration rhythm: speak every 15–30 seconds.** Never silent for more than 30 seconds.

**Things to deliberately show:**

1. **Reading diffs before accepting.** Scroll through each generated file visibly.
2. **Running `dotnet build`** after the controller lands to confirm compile.
3. **Running the migration**:
   ```bash
   dotnet ef migrations add WishlistFeature
   dotnet ef database update
   ```
4. **Verifying the endpoint** via curl or Swagger before touching the frontend:
   ```bash
   curl -H "Authorization: Bearer <token>" http://localhost:8080/api/wishlist
   ```
5. **Correcting Claude at least once.** If no natural mistake emerges, prompt ambiguously so one does, then course-correct:
   > "Actually that validation belongs in the controller, not the entity. Let me move it."

**If something breaks — excellent.** That's the highest-value moment of the demo. Read the error, paste it back to Claude with the relevant file as context, verify the fix. This is how you actually work; let them see it.

**Avoid auto-accept for long chains.** Each step visible. CEOs want to see you in the loop.

### 20:00–23:00 — Manual test

**Show in the browser:**

1. Log in (if not already)
2. Click a product's heart icon — it should fill
3. Navigate to `/wishlist` — product appears
4. Click the heart again — removed
5. Refresh the wishlist page — product is still gone (persisted)

**Say:**

> "Let me verify this actually works end to end, not just compiles."

### 23:00–25:00 — Commit + close

**Use your commit-message skill** or type manually:

```
feat(wishlist): add favorites feature with backend endpoints and frontend UI

Backend: WishlistItem entity with unique (user, product) constraint,
WishlistController with authenticated GET/POST/DELETE at /api/wishlist.
Frontend: heart toggle on ProductCard, /wishlist page, state slice.
```

**`git commit`** (no need to push unless you want to — local commit is fine for the demo).

**Close with:**

> "That's how I work. Every prompt has context. I verify output before accepting. I push back when Claude gets something wrong. Claude handles the mechanical work fast — boilerplate, wiring, migrations. My job is the decisions: what to build, how to structure it, what to reject. AI is already how I ship, not something I'm exploring."

**Stop talking.** Let them ask questions.

---

## .NET / EF / MVC cheat sheet

Things you'll see Claude generate. Skim Friday evening if any feel unfamiliar.

### MVC Controller skeleton

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class WishlistController(AppDbContext db) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> GetWishlist()
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var items = await db.WishlistItems
            .Where(w => w.UserId == userId)
            .Include(w => w.Product)
            .ToListAsync();
        return Ok(items);
    }

    [HttpPost("{productId:int}")]
    public async Task<IActionResult> AddToWishlist(int productId)
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var existing = await db.WishlistItems
            .FirstOrDefaultAsync(w => w.UserId == userId && w.ProductId == productId);
        if (existing is not null) return Ok(existing);

        var item = new WishlistItem { UserId = userId, ProductId = productId };
        db.WishlistItems.Add(item);
        await db.SaveChangesAsync();
        return CreatedAtAction(nameof(GetWishlist), new { }, item);
    }

    [HttpDelete("{productId:int}")]
    public async Task<IActionResult> RemoveFromWishlist(int productId)
    {
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        var item = await db.WishlistItems
            .FirstOrDefaultAsync(w => w.UserId == userId && w.ProductId == productId);
        if (item is null) return NotFound();

        db.WishlistItems.Remove(item);
        await db.SaveChangesAsync();
        return NoContent();
    }
}
```

**Things to note**:
- **Primary constructor** (`(AppDbContext db)`) — C# 12+ syntax, dependency injected directly into the class
- **`ControllerBase` inheritance** — base class for API controllers (use `Controller` if you return views)
- **`[ApiController]`** — enables automatic model validation and problem-details responses
- **`[Route("api/[controller]")]`** — `[controller]` token resolves to the class name minus the `Controller` suffix
- **`IActionResult` return type** — wraps any result type
- **`Ok()`, `NotFound()`, `NoContent()`, `CreatedAtAction()`** — helper methods on `ControllerBase`
- **`[HttpPost("{productId:int}")]`** — method-level route with type constraint on the parameter

### EF Core entity

```csharp
public class WishlistItem
{
    public int Id { get; set; }
    public string UserId { get; set; } = string.Empty;
    public int ProductId { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation property
    public Product? Product { get; set; }
}
```

### DbContext with unique constraint

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<WishlistItem> WishlistItems => Set<WishlistItem>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        modelBuilder.Entity<WishlistItem>()
            .HasIndex(w => new { w.UserId, w.ProductId })
            .IsUnique();
    }
}
```

### Migration commands

```bash
dotnet ef migrations add WishlistFeature
dotnet ef database update
# if you need to undo (before DB update):
dotnet ef migrations remove
```

### LINQ patterns you'll see

```csharp
// Get with include (eager-load relationship)
await db.WishlistItems.Include(w => w.Product).ToListAsync();

// Filter + first
await db.WishlistItems.FirstOrDefaultAsync(w => w.UserId == userId && w.ProductId == productId);

// No tracking (for read-only)
await db.Products.AsNoTracking().Where(p => p.CategoryId == id).ToListAsync();
```

### JWT auth wiring (already in nextnet-shop)

You don't need to set this up — it's already there. But recognize it:

```csharp
// Program.cs
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options => { /* token validation */ });
builder.Services.AddAuthorization();

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

### Things to ask Claude, not guess

- Exact `Include` / `ThenInclude` chains for multi-level eager loads
- `HasOne`, `WithMany`, `HasForeignKey` fluent API for relationships
- Custom `JsonSerializerOptions` to control DTO serialization
- Migration quirks (cascading deletes, seed data placement)

Don't waste time memorizing these. Look them up on demand.

---

## Scripted phrases

### Opening

> "I'll demo with nextnet-shop — a personal e-commerce project on my GitHub. Next.js frontend, .NET 9, MVC Controllers, EF Core, Postgres. I hadn't touched it in two years so I spent the weekend getting it back up, refactoring Minimal API to MVC Controllers so it's closer to your stack. What you'll see today is me adding a Wishlist feature end to end, with Claude Code, the way I work day to day."

### On your .NET background

> "Honest frame: my commercial backend is TypeScript and Node. I've worked in .NET on personal projects and academically. My shorthand for the stack is adjacent — I can read it, I can drive Claude to generate it, I verify what it produces. I'm not pretending to be a senior .NET engineer. I'm showing you I can ramp into your codebase quickly."

### When course-correcting Claude

> "That's not quite what I want. Let me be more specific — [specifically what]."

### When debugging a live error

> "OK, that's an error. Let me read it before asking Claude to fix it. I don't want Claude guessing at what went wrong."

### If they ask a .NET question you don't know

> "I'd normally check the docs for that. Claude can suggest, but for anything I haven't shipped before I verify."

### Closer

> "That's how I work. Every prompt has context. I verify output before accepting. I push back when Claude's wrong. Claude handles the mechanical parts fast — boilerplate, wiring, migrations. My job is the decisions. AI is already how I ship, not something I'm exploring."

---

## What to show, what to hide, what to say, what not to

### Show

- Repo-level `CLAUDE.md`
- One or two rule files
- Two skills you actually use
- Plan mode
- Reading diffs before accepting
- Running `dotnet build`, `dotnet ef migrations add`, `dotnet run`
- At least one course-correct
- Browser manual test

### Don't show

- Your entire `~/.claude/` tree
- Other projects, personal tabs, irrelevant code
- API keys, `.env`, credentials
- Auto-accept mode racing through 10 tool calls without narration

### Don't say

- "Claude told me to do this" (you're the author)
- "I'm not really a .NET developer but..." (weak frame; use the scripted version above)
- "Sorry, Claude is being slow" (don't apologize on its behalf)
- "Um, let me think" (silence is fine; apologizing for silence isn't)

### Do say

- "The reason I chose X over Y was..."
- "Let me verify this before I move on."
- "That's not quite what I want. Let me rephrase..."
- "Good question. Honest answer is..."

---

## Risk mitigation

### If the project won't start Monday morning

- `dotnet ef database update` — migrations might need reapplying
- Check connection string in `appsettings.Development.json`
- `bun install` fresh if frontend is broken
- **Last resort**: demo from the Sunday rehearsal branch if main is broken

### If Claude Code misbehaves

- Rate limit hit: switch to Sonnet 4.6 as backup
- Session hangs: `/resume` or restart terminal
- Bad output persistently: you have the manual fallback. Type the code yourself. Showing you can write code without AI is a strength, not weakness.

### If you hit an unfamiliar .NET error

- Read the full stack trace, not just the top line
- Ask Claude with the full context (paste the error, paste the relevant file)
- Don't guess. Admit uncertainty if needed: "I haven't seen this one before, let me check what Claude says."

### If you run out of time

- At 22:00 with the feature half-done: stop cleanly. "I'd normally keep going but I want to leave time for questions. What I have at this point is the backend fully working and verified with curl, frontend wired up, styling needs a pass."
- **Panel cares about the workflow, not completion.** A 70% feature with clean narration beats a 100% feature that was rushed.

---

## Pre-meeting checklist (Tuesday morning, 30 min before)

- [ ] `dotnet run` in `net-backend/` — up on port 8080
- [ ] `bun dev` in `next-frontend/` — up on port 3000
- [ ] Browse `http://localhost:3000` — homepage loads, can log in
- [ ] VS Code open on `nextnet-shop` root
- [ ] Font size 16+ in VS Code
- [ ] Terminal font bumped too
- [ ] Claude Code CLI ready in a terminal, logged in, Opus 4.7 selected
- [ ] Git on `main`, clean working tree
- [ ] Zoom open, screen share tested
- [ ] Mic + audio tested
- [ ] Notifications off (Slack, email, Discord)
- [ ] Water nearby
- [ ] This file open on a second monitor or printed (not on the shared screen)

---

## Psychological frame

The CEO scheduled this meeting because your first call went well. He's confirming an instinct, not reevaluating.

You're demoing how you already work. You already know how to do this. The last 3 days are motion practice, not skill acquisition.

**You built Weathered on your feet in front of the RFS panel.** This is that, with:
- More prep time
- A smaller feature
- A more sympathetic audience (he's already a fan)
- Your own project that you legitimately own

You're going to be fine.

---

## Continuing this work on another device

This section is for when you sit down at a different machine (laptop swap, desk swap, whatever) and want to pick up where you left off. It's also the "brief a fresh Claude Code session" pack — paste the prompt at the bottom of this section and the new session will be on the same page.

### File inventory: read these in order on the new device

1. **`~/projects/trevor/auditcover-demo-prep.md`** ← this file. The canonical plan. Read top to bottom first.
2. **`~/projects/trevor/interview-prep.md`** — broader interview prep doc (RFS context). Has voice rules, behavioural answers, AI-usage scripted answer. Cross-references this file for AuditCover specifics.
3. **`~/projects/trevor/walkthrough.md`** — the RFS walkthrough script. Useful if you need to recalibrate the Tuesday demo timing or narration rhythm.
4. **`~/projects/trevor/nextnet-shop/CLAUDE.md`** (when you create it) — repo-level conventions for Claude Code on this project.
5. **`~/projects/trevor/nextnet-shop/net-backend/`** — the .NET backend submodule.
6. **`~/projects/trevor/nextnet-shop/next-frontend/`** — the Next.js frontend submodule.
7. **`~/projects/trevor/Resume_TrevorTu.docx`** — only if a fresh session needs to know your background. Usually not needed.

### Decisions log (don't re-decide these)

The following decisions are settled. Don't relitigate them with a fresh Claude session:

| Decision | Settled value |
|---|---|
| **Project for the demo** | `nextnet-shop` (not a fresh scaffold, not Weathered) |
| **Backend pattern** | MVC Controllers (migrating from Minimal API over Saturday) |
| **Frontend** | Stay with whatever Next.js / Bun / Redux is already there |
| **Database** | Postgres (already configured in nextnet-shop, don't switch to SQLite) |
| **Live demo feature** | Wishlist (heart icon, /wishlist page, three backend endpoints) |
| **Tone of all docs** | Semi-casual, semi-formal. No em-dashes. No "I'm thrilled to..." filler. Documentation voice in prose; first person OK in personal sections like AI-usage statements |
| **Style for code** | Match nextnet-shop's existing patterns. No new conventions introduced just for the demo. |
| **Meeting time budget** | ~25 min demo + 5 min Q&A |

### Current state checklist

Update this as you complete each step. The state of these checkboxes is the source of truth for "where are we in the plan."

**Friday — stabilize:**
- [ ] Working tree clean, everything committed
- [ ] `git submodule update --init --recursive` ran cleanly
- [ ] `dotnet restore` in `net-backend/` succeeded
- [ ] `dotnet run` in `net-backend/` starts the API
- [ ] `bun install` in `next-frontend/` succeeded
- [ ] `bun dev` in `next-frontend/` starts the frontend
- [ ] `http://localhost:3000` loads, products render
- [ ] Auth works end-to-end (can log in)
- [ ] Friday reading done (60 min on ASP.NET Core MVC)

**Saturday — Minimal API → Controllers migration:**
- [ ] Categories module migrated, committed, verified
- [ ] Products module migrated, committed, verified
- [ ] Users / Auth module migrated, committed, verified
- [ ] Cart module migrated, committed, verified
- [ ] Orders module migrated, committed, verified
- [ ] `Program.cs` cleaned up (no leftover `MapGroup` / `MapPost` calls)
- [ ] All routes still reachable (frontend hasn't broken)

**Sunday — rehearsal:**
- [ ] Wishlist dry-run #1 on `rehearsal/wishlist` branch, timed
- [ ] Hiccups noted and fixed in main
- [ ] Wishlist dry-run #2 on a fresh rehearsal branch, smoother
- [ ] Both rehearsal branches deleted
- [ ] Opening + closing scripted phrases memorized
- [ ] Pre-meeting checklist reviewed

**Monday — light touch:**
- [ ] Sanity check: both servers start, app works
- [ ] Re-read this doc once
- [ ] Stop by 3 PM. Rest.

**Tuesday — meeting:**
- [ ] Pre-meeting checklist completed 30 min before
- [ ] Demo executed
- [ ] Follow-up email sent within 4 hours

### Setting up a fresh device

If the new device doesn't have the project at all, run these once:

```bash
mkdir -p ~/projects/trevor
cd ~/projects/trevor

# clone the prep docs (if you keep them in a private repo) or scp from the other device
# example via scp:
# scp old-device:~/projects/trevor/auditcover-demo-prep.md .
# scp old-device:~/projects/trevor/interview-prep.md .
# scp old-device:~/projects/trevor/walkthrough.md .

# clone nextnet-shop with submodules
git clone --recurse-submodules git@github.com:ToanThanhTu/nextnet-shop.git
cd nextnet-shop

# verify submodules
git submodule status

# .NET prerequisites (if not installed)
# https://dotnet.microsoft.com/en-us/download/dotnet/9.0
dotnet --version  # should be 9.x

# Bun (if not installed)
# curl -fsSL https://bun.sh/install | bash
bun --version

# backend setup
cd net-backend
dotnet restore
# create the dev appsettings if it's gitignored:
cp appsettings.json appsettings.Development.json   # then edit DB connection string + JWT key
dotnet ef database update                           # apply migrations
dotnet run                                          # verify it boots

# frontend setup (in a new terminal)
cd ../next-frontend
bun install
bun dev                                             # verify it loads
```

Once both come up and the homepage renders, the device is ready to continue from whichever Friday/Saturday/Sunday step is next on the checklist above.

### Briefing a fresh Claude Code session

When you start Claude Code on the new device, paste the following prompt as your first message. It loads enough context that the session picks up the plan without you re-explaining anything.

```
I'm continuing prep for a Tuesday Zoom meeting with the AuditCover CEO + COO.
The meeting is a live screenshare demo where I'll show how I use you (Claude
Code) in my day-to-day workflow by adding a feature to a personal project.

Read these files first, in order:

1. ~/projects/trevor/auditcover-demo-prep.md — canonical plan, decisions, schedule
2. ~/projects/trevor/interview-prep.md — broader interview prep, voice rules, AI-usage statement
3. ~/projects/trevor/nextnet-shop/CLAUDE.md (if it exists) — project conventions

Project I'm demoing: ~/projects/trevor/nextnet-shop/. Two submodules:
net-backend (.NET 9 + EF Core + Postgres) and next-frontend (Next.js + Bun + Redux).

Decisions already made (do not relitigate):
- Use nextnet-shop, not a fresh project
- Migrate backend from Minimal API to MVC Controllers over Saturday
- Wishlist feature for Tuesday's live demo
- Postgres stays, no switch to SQLite

Voice rules for any docs you produce:
- No em-dashes (replace with colons, semicolons, periods, or rewrite)
- Semi-casual, semi-formal
- Documentation voice in prose; first person OK in personal sections
- No AI-stylistic tells ("I'm thrilled to...", overuse of "**", etc.)

Check the "Current state checklist" section in auditcover-demo-prep.md. Tell me
which step we're on, and let's continue from there.
```

That prompt loads everything the session needs and explicitly tells it not to redecide settled questions. Update the prompt as the project state evolves (e.g., once Friday is done, mention "Friday is complete, we're on Saturday's migration").

### Voice and style reminders for any docs Claude generates

These rules came from earlier sessions and apply to anything written into this prep doc, the README, the CLAUDE.md tree, or any code comments:

- **No em-dashes.** Use colons, semicolons, periods, or rewrite the sentence.
- **No "I'm excited / thrilled / happy to..."** filler.
- **No AI tells**: "It's worth noting that...", "Importantly, ...", "Furthermore, ..." — strip these.
- **Documentation voice for technical prose**: declarative, third-person, no first-person commentary in CLAUDE.md or README files.
- **First person is fine** in personal sections: AI-usage disclosure, walkthrough scripts you read aloud, behavioural answers.
- **Match the existing project's tone**: nextnet-shop's commit messages are short and Conventional Commits style. Don't introduce a flowery new style.

### Sync rhythm if you're switching devices mid-task

If you're working on Saturday's migration on one device and want to continue on another:

1. **Commit and push** before switching:
   ```bash
   cd ~/projects/trevor/nextnet-shop/net-backend
   git status
   git add .
   git commit -m "wip: migration of <module> in progress"
   git push
   ```

2. **On the new device**:
   ```bash
   cd ~/projects/trevor/nextnet-shop/net-backend
   git pull
   ```

3. **If you used a temporary `wip:` commit**, soft-reset it before continuing real work:
   ```bash
   git reset --soft HEAD~1
   ```
   Now your previous changes are staged but not committed, and you can keep building on them with cleaner commits.

4. **Update the Current state checklist** in this file (`auditcover-demo-prep.md`) on the new device, commit, push back. The checklist is the cross-device source of truth.

### Quick troubleshooting on a fresh device

| Problem | Fix |
|---|---|
| `dotnet ef` not found | `dotnet tool install --global dotnet-ef` |
| Postgres connection error | Check `appsettings.Development.json` connection string; ensure Postgres is running locally or update the string to point at the right host |
| `bun` not found | Install via `curl -fsSL https://bun.sh/install \| bash` |
| Frontend build error: peer dep mismatch | `rm -rf node_modules bun.lock && bun install` |
| `git submodule status` shows uninitialized | `git submodule update --init --recursive` |
| Auth doesn't work in browser | Check that the JWT signing key in `appsettings.Development.json` is non-empty and the frontend's API base URL points at the right backend port |

If something else breaks, paste the error verbatim into Claude Code with the relevant file as context. Don't guess.
