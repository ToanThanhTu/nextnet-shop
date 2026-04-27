# Wishlist feature: implementation plan

The Wishlist is the live demo content for Tuesday. This plan lists every file to create and the order to create them in, mapped to the established DDD/MVC pattern. Follow it linearly and the feature is done in ~25-30 minutes.

> Personal prep doc; gitignored. Companion to `WISHLIST_DEMO.md` (the live-demo script).

## Spec recap

Users can heart a product to save it. Persists per-authenticated-user. New `/wishlist` page lists saved products. The heart icon on the product detail page (and product card if time allows) toggles favorite state.

**Backend:**
- `WishlistItem` entity: `Id`, `UserId`, `ProductId`, `CreatedAt`. Unique index on `(UserId, ProductId)`.
- EF migration: `AddWishlist`.
- Three endpoints, all `[Authorize]`, user from JWT:
  - `GET /wishlist`: current user's wishlist with embedded product summary
  - `POST /wishlist/items`: add product to wishlist (idempotent: if already there, returns existing instead of duplicate-key error)
  - `DELETE /wishlist/items/{productId:int}`: remove

**Frontend:**
- New domain module `src/modules/wishlist/` with entity, API client, optional slice.
- Heart icon Client Component on the product detail page.
- New `/wishlist` page that lists wishlist items.

## Backend file plan

Create in this order so each file's imports already exist when the next file references them.

### 1. Entity (`Data/Types/`)

`net-backend/Data/Types/WishlistItem.cs`:

```csharp
using System.ComponentModel.DataAnnotations.Schema;

namespace net_backend.Data.Types;

public class WishlistItem
{
    [Column("id")]
    public int Id { get; set; }

    [Column("user_id")]
    public int UserId { get; set; }

    [Column("product_id")]
    public int ProductId { get; set; }

    [Column("created_at")]
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public User? User { get; set; }
    public Product? Product { get; set; }
}
```

### 2. DbContext registration

In `Data/AppDbContext.cs`:
- Add `public DbSet<WishlistItem> WishlistItems => Set<WishlistItem>();`
- In `OnModelCreating`:

```csharp
modelBuilder.Entity<WishlistItem>()
    .ToTable("wishlist_items")
    .HasKey(w => w.Id);

modelBuilder.Entity<WishlistItem>()
    .HasOne(w => w.User)
    .WithMany()
    .HasForeignKey(w => w.UserId)
    .OnDelete(DeleteBehavior.Cascade);

modelBuilder.Entity<WishlistItem>()
    .HasOne(w => w.Product)
    .WithMany()
    .HasForeignKey(w => w.ProductId);

modelBuilder.Entity<WishlistItem>()
    .HasIndex(w => new { w.UserId, w.ProductId })
    .IsUnique();
```

### 3. Domain interface (`Modules/Wishlist/Domain/`)

`IWishlistRepository.cs`:

```csharp
public interface IWishlistRepository
{
    Task<List<WishlistItem>> ListByUserAsync(int userId, CancellationToken ct = default);
    Task<WishlistItem?> FindAsync(int userId, int productId, CancellationToken ct = default);
    Task<WishlistItem> AddAsync(WishlistItem item, CancellationToken ct = default);
    Task DeleteAsync(WishlistItem item, CancellationToken ct = default);
}
```

### 4. EF implementation (`Modules/Wishlist/Infrastructure/`)

`EfWishlistRepository.cs`:
- `ListByUserAsync`: `.AsNoTracking().Include(w => w.Product).OrderByDescending(w => w.CreatedAt)`.
- `FindAsync`: by `(userId, productId)`, no Include.
- `AddAsync`: `Add` + `SaveChangesAsync`, return; reload with `.Include(Product)` like Cart does.
- `DeleteAsync`: `Remove` + `SaveChangesAsync`.

### 5. Contracts (`Modules/Wishlist/Contracts/`)

`WishlistItemDto.cs`:
```csharp
public record WishlistItemDto(
    int Id,
    int ProductId,
    DateTime AddedAt,
    WishlistProductSummaryDto Product)
{
    public static WishlistItemDto FromEntity(WishlistItem w) => new(
        w.Id, w.ProductId, w.CreatedAt,
        WishlistProductSummaryDto.FromEntity(w.Product
            ?? throw new InvalidOperationException(...)));
}

public record WishlistProductSummaryDto(int Id, string Title, string? Slug, decimal Price, decimal SalePrice, int Sale)
{
    public static WishlistProductSummaryDto FromEntity(Product p) =>
        new(p.Id, p.Title, p.Slug, p.Price, p.SalePrice, p.Sale);
}
```

`AddWishlistItemRequest.cs`:
```csharp
public record AddWishlistItemRequest(
    [Required, Range(1, int.MaxValue)] int ProductId);
```

### 6. Application handlers (`Modules/Wishlist/Application/`)

**Queries/GetMyWishlistHandler.cs**: list wrapped in DTOs.

**Commands/AddWishlistItemHandler.cs**: idempotent. Call `FindAsync` first; if exists, return existing as DTO; else `AddAsync` then return new.

**Commands/RemoveWishlistItemHandler.cs**: `FindAsync`; throw `NotFoundException` if missing; else `DeleteAsync`.

### 7. Controller (`Modules/Wishlist/`)

`WishlistController.cs`:

```csharp
[ApiController]
[Route("wishlist")]
[Authorize]
public class WishlistController(
    GetMyWishlistHandler getHandler,
    AddWishlistItemHandler addHandler,
    RemoveWishlistItemHandler removeHandler) : ControllerBase
{
    [HttpGet("")]
    public async Task<ActionResult<List<WishlistItemDto>>> GetMine(CancellationToken ct)
    {
        var userId = User.GetRequiredUserId();
        return Ok(await getHandler.ExecuteAsync(userId, ct));
    }

    [HttpPost("items")]
    public async Task<ActionResult<WishlistItemDto>> Add(
        [FromBody] AddWishlistItemRequest req, CancellationToken ct)
    {
        var userId = User.GetRequiredUserId();
        var item = await addHandler.ExecuteAsync(userId, req, ct);
        return Ok(item);
    }

    [HttpDelete("items/{productId:int}")]
    public async Task<IActionResult> Remove(int productId, CancellationToken ct)
    {
        var userId = User.GetRequiredUserId();
        await removeHandler.ExecuteAsync(userId, productId, ct);
        return NoContent();
    }
}
```

### 8. Module DI registration (`Modules/Wishlist/`)

`WishlistModule.cs`:

```csharp
public static class WishlistModule
{
    public static WebApplicationBuilder AddWishlistFeature(this WebApplicationBuilder builder)
    {
        builder.Services.AddScoped<IWishlistRepository, EfWishlistRepository>();
        builder.Services.AddScoped<GetMyWishlistHandler>();
        builder.Services.AddScoped<AddWishlistItemHandler>();
        builder.Services.AddScoped<RemoveWishlistItemHandler>();
        return builder;
    }
}
```

### 9. Wire into `ConfigureServices.cs`

Add `using net_backend.Modules.Wishlist;` and append `builder.AddWishlistFeature();` to the per-feature registration block.

### 10. EF migration

```bash
cd net-backend
dotnet ef migrations add AddWishlist
# Review the generated Migrations/*.cs file:
#   - CreateTable wishlist_items with id, user_id, product_id, created_at
#   - Foreign keys to users (cascade delete) and products
#   - Unique index on (user_id, product_id)
dotnet ef database update
```

### 11. Verify backend (curl)

```bash
TOKEN=$(curl -s -X POST http://localhost:8080/users/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"audit@test.local","password":"testpass123"}' \
  | python3 -c 'import sys,json; print(json.load(sys.stdin)["token"])')

# empty initial state
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/wishlist

# add
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -X POST http://localhost:8080/wishlist/items -d '{"productId":1}'

# add same product again: idempotent, returns existing item, no duplicate
curl -s -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -X POST http://localhost:8080/wishlist/items -d '{"productId":1}'

# get
curl -s -H "Authorization: Bearer $TOKEN" http://localhost:8080/wishlist

# remove
curl -s -o /dev/null -w '%{http_code}\n' -H "Authorization: Bearer $TOKEN" \
  -X DELETE http://localhost:8080/wishlist/items/1

# remove non-existent → 404 with errorCode WISHLIST_ITEM_NOT_FOUND
curl -s -H "Authorization: Bearer $TOKEN" -X DELETE http://localhost:8080/wishlist/items/9999

# auth gating: no token → 401
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8080/wishlist
```

## Frontend file plan

### 12. Domain entity

`src/modules/wishlist/entities/wishlist-item.ts`:

```typescript
import type { Product } from "@/modules/products/entities"

export type WishlistItem = {
  id: number
  productId: number
  addedAt: string
  product: Pick<Product, "id" | "title" | "slug" | "price" | "salePrice" | "sale">
}
```

`src/modules/wishlist/entities/index.ts`:
```typescript
export * from "./wishlist-item"
```

### 13. API client

`src/modules/wishlist/wishlist.api.ts`:

```typescript
import { apiSlice } from "@/lib/api-slice"
import type { WishlistItem } from "./entities"

export const wishlistApi = apiSlice.injectEndpoints({
  endpoints: (builder) => ({
    getWishlist: builder.query<WishlistItem[], void>({
      query: () => "/wishlist",
      providesTags: ["Wishlist"],
    }),
    addToWishlist: builder.mutation<WishlistItem, number>({
      query: (productId) => ({
        url: "/wishlist/items",
        method: "POST",
        body: { productId },
      }),
      invalidatesTags: ["Wishlist"],
    }),
    removeFromWishlist: builder.mutation<void, number>({
      query: (productId) => ({
        url: `/wishlist/items/${productId}`,
        method: "DELETE",
      }),
      invalidatesTags: ["Wishlist"],
    }),
  }),
})

export const {
  useGetWishlistQuery,
  useAddToWishlistMutation,
  useRemoveFromWishlistMutation,
} = wishlistApi
```

### 14. Add `"Wishlist"` to `apiSlice.tagTypes`

In `src/lib/api-slice.ts`:
```typescript
tagTypes: ["User", "Category", "Cart", "Orders", "Wishlist"],
```

### 15. Module barrel

`src/modules/wishlist/index.ts`:
```typescript
export * from "./entities"
export * from "./wishlist.api"
```

### 16. Heart-icon Client Component

`src/app/components/products/wishlist-button.tsx`:

```tsx
"use client"

import { useGetWishlistQuery, useAddToWishlistMutation, useRemoveFromWishlistMutation } from "@/modules/wishlist"
import { useAppSelector } from "@/lib/hooks"
import { Heart } from "lucide-react"
import { cn } from "@/lib/utils"

export default function WishlistButton({ productId }: { productId: number }) {
  const user = useAppSelector((s) => s.auth.user)
  const { data: items = [] } = useGetWishlistQuery(undefined, { skip: !user })
  const [add, { isLoading: adding }] = useAddToWishlistMutation()
  const [remove, { isLoading: removing }] = useRemoveFromWishlistMutation()

  if (!user) return null
  const isFavorited = items.some((i) => i.productId === productId)
  const busy = adding || removing

  const toggle = () => (isFavorited ? remove(productId) : add(productId))

  return (
    <button onClick={toggle} disabled={busy} aria-label={isFavorited ? "Remove from wishlist" : "Add to wishlist"}>
      <Heart className={cn("h-6 w-6", isFavorited ? "fill-red-500 text-red-500" : "text-muted-foreground")} />
    </button>
  )
}
```

### 17. Place the button on product detail

In `src/app/components/products/product/product-page.tsx`, drop `<WishlistButton productId={product.id} />` next to the title or price block.

### 18. Wishlist page

`src/app/(shop)/(browse)/wishlist/page.tsx`:
- Client Component (uses `useGetWishlistQuery`).
- `useAuth({ needSignIn: true })` to gate.
- Show empty state if list is empty; else grid of items with title, price, slug link to product, and a remove button.

### 19. Optional: add to product card too

If time, drop `<WishlistButton>` on `src/app/components/products/product-card.tsx`. Skip on Tuesday if running short.

### 20. Verify frontend

```bash
cd next-frontend
npx tsc --noEmit
# then in the browser: log in → product page → click heart → /wishlist shows it → remove → empty state
```

## Order-of-operations checklist for the demo

Tape this on the side monitor:

- [ ] Backend: WishlistItem entity + DbContext + migration → run + verify schema
- [ ] Backend: repo interface + EF impl
- [ ] Backend: contracts (DTO + request)
- [ ] Backend: 3 handlers (Get, Add, Remove)
- [ ] Backend: controller (3 actions, [Authorize])
- [ ] Backend: WishlistModule + ConfigureServices wire-up
- [ ] Backend: smoke curl (get, add x2, remove)
- [ ] Frontend: entity + api + barrel + tagTypes
- [ ] Frontend: WishlistButton component
- [ ] Frontend: place on product-page
- [ ] Frontend: /wishlist page
- [ ] Manual browser test
- [ ] Commit per submodule with the messages from the patterns we've established

## Estimated time (rehearsal numbers)

| Step | Range |
|---|---|
| Backend skeleton (1-9) | 8-12 min |
| EF migration + verify (10-11) | 3-5 min |
| Frontend module + API (12-15) | 3-5 min |
| Heart button + page (16-18) | 5-8 min |
| Manual test + commit (19-20) | 2-3 min |
| Buffer for course-corrects | 5 min |
| **Total** | **~25-35 min** |

## Things to watch out for

- **Cross-aggregate validation**: Add handler should `IProductRepository.GetByIdAsync` to ensure the product exists before insert. Catches "wishlist contains stale productId" cleanly.
- **Idempotency**: Add operation must check `FindAsync` first. The unique constraint protects DB integrity, but throwing on duplicate-key is a poor UX; client-side it's just "the heart should be filled now."
- **Frontend `skip`**: `useGetWishlistQuery(undefined, { skip: !user })`: without skip, the unauthenticated `/wishlist` call returns 401 and pollutes the cache.
- **`DateTime.UtcNow` not `DateTime.Now`** on `CreatedAt`. Npgsql 8+ rejects `Local` kind for `timestamptz`.
- **Migration baseline**: We're at `20260426151538_InitialCreate`. The new migration will get a new timestamp; that's fine. EF tracks both in `__EFMigrationsHistory`.
