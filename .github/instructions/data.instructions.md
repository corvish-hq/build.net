---
applyTo: "**/*Data*/**,**/Migrations/**"
---

# Data Layer Rules

<!-- Shared baseline, owned by the build.net repository (https://github.com/corvish-hq/build.net). -->
<!-- Consumer repos receive a copy on every build — never edit the copy, change build.net instead. -->

> [!NOTE]
> Applies to repositories with a data layer (EF Core and/or other storage backends). The repo instructions name the
> data project, its namespaces, the storage backends, and the `DbContext` (`<Product>DbContext` below).
>
> Placeholders (`<Product>`, `<UiPrefix>`) and general rules are explained in
> [base.instructions.md](base.instructions.md); the concrete placeholder values come from the repo instructions,
> which also win where they contradict this file.

## Data Layer / EF Core

### Entity Documentation

- Entity XMLDoc describes the **database semantics**: constraints, indexes, delete behavior, and EF Core usage hints
  (e.g., which `Include(...)` a navigation needs). It does not describe UI usage.
- The repository's `<Product>DbContext` is the single source of truth for constraints and indexes; the XMLDoc
  mirrors it.
- Document indexed properties using `<b>Index:</b> ...`, wrap structural tokens in `<c>...</c>`.

### Entity Layout

Every entity is either a **standard entity** or a **join entity**; each kind has its own section numbering. The
marker titles below are fixed — explanations go into the XMLDoc or a comment line below the marker, never into the
marker itself.

#### Standard Entities

Every entity that is not a join entity, whatever its key:

| Order | Marker title (entity class) | Content |
|---|---|---|
| 1 | `Primary key` | `Id` — also when it is a shared primary key that is at the same time the FK of a one-to-one relationship |
| 1 | `Composite key (none)` | Composite key of one FK plus a value column (e.g. `ProductId` + `CultureName` for a translation); declared via `HasKey(e => new { ... })` in `Configure*()`, so the section has no member |
| 2 | `Public identifier` | `PublicId` |
| 3 | `Foreign keys + Navigation properties` | FK + navigation pairs, single-entity navigations, also the FK part of a composite key |
| 4 | `Timestamps` | Audit timestamps (see [Classification Rules](#classification-rules)) |
| 5 | `Scalar domain fields` | Required before optional; also the value part of a composite key |
| 6 | `Collection navigation properties` | `ICollection<T>` navigations |

#### Join Entities

Only entities whose key consists of **exactly two foreign keys** that connect two other entities:

| Order | Marker title (entity class) | Content |
|---|---|---|
| 1 | `Composite key (none)` | Declared via `HasKey(e => new { ... })` in `Configure*()`; no member in the entity class |
| 2 | `First FK + Navigation` | First FK + navigation pair |
| 3 | `Second FK + Navigation` | Second FK + navigation pair |
| 4 | `Timestamps` | Audit timestamps |
| 5 | `Scalar domain fields` | Enums, flags, etc. — required before optional |

An entity class contains only persisted state and navigations — no sections beyond these, e.g. for helper methods.

#### Section Markers

To make the section boundaries visible in the source — especially when timestamps are pushed downward
or when sections are skipped entirely — every entity class **and** the corresponding `Configure*()` method
in `<Product>DbContext` carries inline section-divider comments.

**Style:**

| Case | Marker style | Example |
|---|---|---|
| Section has members | Number and fixed title from the entity kind's layout table above | `// --- 4. Timestamps ---` |
| Section is empty | Numbered placeholder with `(none)` suffix — preserves the visual sequence and signals intent | `// --- 2. Public identifier (none) ---` |

**Always emit the full canonical sequence** — including Section 1. Skipping markers (even for single-section
entities or empty sections) breaks the side-by-side correspondence between the entity class and its
`Configure*()` counterpart and creates ambiguous gaps that read as oversights rather than deliberate omissions.

A `(none)` placeholder marker has no body — it sits between the surrounding sections as a single comment
line with a blank line above and below, just like a populated marker. This makes "Section 2 deliberately
empty" instantly readable instead of requiring the reader to count.

**Placement:**

- **Entity class:** marker goes directly above the XMLDoc of the section's first member. Empty-section
  placeholders sit between the neighboring sections on their own.
- **`Configure*()` method:** marker goes directly above the section's first `Property(...)` /
  `HasOne(...)` / `HasIndex(...)` call. **Section 1 is marked too** — its marker goes at the very top of
  the lambda body, above `ToTable(...)`. `ToTable` / `HasKey` / `Property(Id)` all belong to Section 1
  because the table mapping is part of *how the primary key is persisted* (table + key form one
  storage-identity unit). To make this explicit in the marker text, Section 1 is titled
  **`Table mapping & primary key`** in `Configure*()` methods for entities with an `Id`, and
  **`Table mapping & composite key`** for entities with a composite key (standard or join). It is never `(none)` in
  `Configure*()`, because `ToTable(...)` always lives there. All other sections use the same titles as the entity
  class.

#### Classification Rules

To remove ambiguity when deciding which section a property belongs to:

- **Single-entity reverse navigations** (`T?` properties whose foreign key column lives on the *other*
  side of the relationship — e.g. `UserEntity.Preferences`, `OrderEntity.Invoice`) are still single-entity
  navigations and belong to **Section 3**.
  The decisive question is the property type, not where the FK column lives (same rule as the
  predictable-includes guidance under [Read APIs](#read-apis)).

- **Polymorphic foreign keys without a CLR navigation** (e.g. `AttachmentEntity.OwnerKind` +
  `OwnerId`) belong to **Section 3**. They behave like FK + Nav pairs at the persistence level even
  though no `T?` navigation property exists, and grouping them with Section 5 scalars would obscure
  their role as the entity's primary join column. Because the database cannot enforce or cascade such a key:
  - The discriminator precedes the id (`OwnerKind` before `OwnerId`) — the kind gives the id its meaning.
  - A composite index on `(OwnerKind, OwnerId)` is mandatory; owner lookups would otherwise scan the table.
  - Orphan cleanup must be handled explicitly in the service layer (e.g. a sweep that removes rows whose owner
    no longer exists) and documented on the entity.

- **Audit timestamps** (e.g. `CreatedAtUtc`, `UpdatedAtUtc`, `AssignedAtUtc`, `RevokedAtUtc`, `LastLoginAtUtc`,
  `LastRunAtUtc`) belong to **Section 4**.
  They describe *when something happened to the row itself*.

- **Domain timestamps** (e.g. `CancelledAtUtc`, `ExpiresAtUtc`) belong to **Section 5** alongside the scalar
  fields they semantically pair with (e.g. `CancelledAtUtc` next to `CancellationReason`, `ExpiresAtUtc`
  next to the token payload columns). They describe *the data*, not the row's lifecycle.

### DbContext Configuration

The `<Product>DbContext` follows two distinct ordering conventions — **alphabetical** for the top-level
`Configure*()` calls in `OnModelCreating`, and the **Entity Layout order** (see above) for property/relationship
calls *inside* each `Configure*()` method.

#### `Configure*()` Method Calls in `OnModelCreating` — Alphabetical

Keep the call list alphabetical and matching the physical method ordering in the file:

- No domain-grouping debates — entities are independent, so any grouping would be arbitrary.
- New entities slot in trivially.
- A single `Configure*()` method is easy to locate by name.

#### Inside Each `Configure*()` — Mirror the Entity Layout

The calls inside a `Configure*()` method use the **same sections as the entity class** (see
[Entity Layout](#entity-layout)), with the same numbers and markers. Within the sections (standard entities; join
entities analogously):

| Section | Configuration calls |
|---|---|
| 1 | `ToTable(...)`, `HasKey(...)`, then `Property(Id).ValueGeneratedOnAdd()` (`ValueGeneratedNever()` for singletons) |
| 2 | `Property(PublicId)` |
| 3 | Per FK, in entity order: `Property(FK)`, then its `HasOne(...)`, then its `HasIndex(...)` |
| 4 | Timestamps, in entity order |
| 5 | Scalar fields, in entity order |
| 6 | `HasMany(...)` only for collections whose relationship is not configured on the dependent side |

- **Relationships are configured on the dependent side** — the entity that holds the FK column — in Section 3.
  Section 6 therefore stays `(none)` in most `Configure*()` methods.
- **Every `HasIndex(...)` goes directly after the `Property(...)` call of its first column.** This also applies to
  composite and unique indexes; there is no separate index block.
- **Never sort alphabetically inside `Configure*()`** — unlike the method list above, the calls mirror the entity
  so both can be read side by side.

### Migrations

Migration discipline is **phased** and tied to the MVP cut-over. The repo instructions state which phase the
repository is currently in; if they don't, assume **Post-MVP** (the safe default).

| Phase | Rule |
|---|---|
| **Pre-MVP** | Create a **new migration** for each schema change using `dotnet ef migrations add <Name>`. **Folding** schema changes into existing migrations (editing the migration `.cs`, its `.Designer.cs`, and the model snapshot in place) is **allowed** because the DB is not yet distributed. |
| **MVP cut-over** | Before the first deploy: delete **all** files in the data project's `Migrations/` folder (migrations, designers, and `<Product>DbContextModelSnapshot.cs`), then run `dotnet ef migrations add InitialCreate` to produce a single consolidated baseline that matches the current model exactly. Delete local dev/test `.db` files so they regenerate cleanly. |
| **Post-MVP** | **Folding is forbidden.** Every schema change must be a **new** migration. Existing migrations and designer files are append-only history — editing them in place would desynchronize deployed databases from the migration ledger. |

> [!IMPORTANT]
> The Pre-MVP folding allowance is a deliberate, time-limited exception — not a long-term pattern. A repository in
> the Pre-MVP phase must carry a `TODO(MVP): Consolidate migrations` note in its repo instructions and switch to
> Post-MVP after the cut-over.

### Service API Conventions

Data service APIs (e.g. `IUserDataService`, `IOrderDataService`, `IRoleDataService`) follow these conventions to keep
the surface predictable and the EF Core change tracker contained.

> [!IMPORTANT]
> **Scope per storage backend.** Method categories, ordering, naming, return types, argument validation, and the
> observable contract (e.g. detached results, include shape) apply to **every** implementation of a data service
> interface, whatever the storage backend. Mechanism rules — `AsNoTracking()`, change tracking, `Include(...)`, and
> Compiled Queries — apply to EF Core implementations only; other backends (e.g. browser storage) must produce the same
> observable result with their own means.

#### Return Types

Collections returned from a data service interface use **read-only interfaces**: `IReadOnlyList<T>`,
`IReadOnlyDictionary<TKey, TValue>`, `IReadOnlySet<T>`. They signal "do not mutate" at the API boundary and leave the
implementation free to choose the concrete type. Internal helpers and infrastructure (import/export, providers, test
fixtures) may use concrete types.

#### Argument Validation

Every data service method validates its arguments on entry, in every backend, with the standard BCL guards
(`ArgumentNullException.ThrowIfNull()`, `ArgumentException.ThrowIfNullOrWhiteSpace()`,
`ArgumentOutOfRangeException.ThrowIfNegativeOrZero()`, …). Invalid arguments throw the matching BCL exception and are
never answered with `null`, `false`, or an empty result.

#### Method Categories

Every method in a data service interface (and its implementing partial class) belongs to exactly one of four categories.
The category determines section placement, ordering, and naming.

| Category | Returns | Examples | Section header |
|---|---|---|---|
| **Read API** | Entity (or list of entities) with predictable includes | `GetOrderByPublicIdAsync`, `ListRolesAsync`, `ListOrdersByCustomerAsync` | `#region Read APIs` |
| **Projection API** | Scalar, ID, dictionary, sub-entity, or non-entity DTO | `ListRoleNamesByUserIdAsync`, `GetOrderCountsAsync`, `GetCurrentShippingAddressAsync`, `ListCustomerIdsWithAvatarAsync`, `GetAvatarAsync`, `GetPreferencesJsonAsync`, `ListOrderIdsWithoutItemsAsync` | `#region Projection APIs` |
| **Existence Check** | `bool` (yes/no, no data) | `EmailExistsAsync`, `UsernameExistsAsync`, `IsItemInOrderAsync`, `HasOpenOrdersAsync`, `UserHasRoleAsync` | `#region Existence Checks` |
| **Mutation API** | `bool`/`int`/freshly-loaded detached entity | `CreateOrderAsync`, `UpdateOrderAsync`, `CancelOrderAsync`, `AssignRoleToUserAsync`, `CleanupOrdersWithoutItemsAsync` | `#region Mutation APIs` |

#### Method Naming

The verb tells the caller the shape of the result:

| Prefix | Returns | Example |
|---|---|---|
| `Get…Async` | A single object, scalar, or lookup dictionary — `null` when not found | `GetOrderAsync`, `GetOrderCountsAsync` |
| `List…Async` | A sequence (`IReadOnlyList<T>`, `IReadOnlySet<T>`, or a page wrapping one) | `ListOrdersAsync`, `ListOrderIdsWithoutItemsAsync` |

Never use `Get` for a sequence (`GetOrdersAsync`, `GetAllOrdersAsync`). Existence Checks and Mutation APIs follow their
own naming rules below.

#### Method Ordering

- **Section order** is fixed: Read APIs → Projection APIs → Existence Checks → Mutation APIs. Reader flow goes from
  "common + harmless" to "rare + state-changing".
- **Within each section**, methods are ordered **alphabetically**. Deterministic placement avoids semantic disputes
  (e.g. "is Update before or after Delete?").
- **Interface and implementation must match exactly.** The order in `IFooDataService.cs` mirrors the order in **every**
  implementing partial file (e.g. `SqlServerDataService.Foo.cs` and `InMemoryDataService.Foo.cs` — a repository may have
  several storage backends; the repo instructions list them).
- **Test files mirror the same order.** The tests of every implementation (e.g. `SqlServerDataServiceTests.Foo.cs`) use
  `#region` blocks per method in the same sequence.
- **Empty sections are omitted.** If a service has no Existence Checks, simply skip that `#region`. (Unlike the *comment
  markers* in [Section Markers](#section-markers), an empty `#region` block is purely decorative — there is nothing
  inside it to navigate to — so it adds noise without value.)

#### Read APIs

- **No tracking, ever.** All Read APIs use `AsNoTracking()`. Returned entities are detached — callers cannot
  mutate-and-save them.
- **Predictable includes.** Each Read API loads a **fixed, documented** set of navigations: the method's `<remarks>`
  list every populated navigation; everything else is `null` / empty. Collections are loaded only on explicit
  request. Callers read the contract instead of guessing. See the explicit definition below.
- **No tracking flag.** A `bool tracked = false` parameter is forbidden — it hides intent at the call site and tempts
  callers into ad-hoc mutation patterns.
- **No combined include selector.** A `[Flags] enum XxxIncludes` parameter is forbidden — the combinations obscure the
  cost of a call. Each opt-in collection gets its own `bool` parameter instead (see below).
- **If a hot-path needs a leaner shape**, add a dedicated method with a clear suffix (e.g.
  `GetOrderSummaryByPublicIdAsync` returning only the order header). Do *not* add flags that *remove* default includes.

##### "Predictable includes" — explicit definition

A Read API loads:

| Navigation property type | Included? | Rationale |
|---|---|---|
| `T?` or `T` (single-entity navigation) | 📌 Fixed per method and documented — never behind a flag | 0–1 additional row, indexed FK join, cheap |
| `ICollection<T>` (one-to-many) | ⚙️ Only via an opt-in `bool includeXxx = false` parameter | Potentially unbounded, Cartesian explosion risk |

**Opt-in collection flags:** one `bool` per collection, named `include` + the navigation name (e.g.
`includeItems` for `OrderEntity.Items`), defaulting to `false`, and documented in the `<param>` XMLDoc with what is
loaded. A collection that callers need almost always is still opt-in — the cost must stay visible at the call site.

The **FK direction does not matter** — a one-to-one reverse navigation (e.g. `UserEntity.Preferences` where the FK lives
on `UserPreferencesEntity.UserId`) is **also a single-entity navigation** and follows the same rule. The decisive
question is the property type, not where the FK column lives.

**Choosing the fixed set:** the method's author includes the single-entity navigations its typical consumers need.
This is a design decision made once per method — not a per-call choice and not a promise that "everything needed" is
there.

**Depth limit**: maximum `Include(...).ThenInclude(...)` (2 levels). Deeper navigation requires a dedicated method.

**Cycle avoidance**: when `A → B` is included, **do not** also load `B → A`. EF Core would detect this but it
adds zero value and confuses the contract.

**Intentional omissions** must be documented in the method's XMLDoc with a `<remarks>` paragraph explaining why (e.g.
*"`OrderItemEntity.Order` is intentionally excluded because callers reach items through their order context."*).

**Code-review heuristic**: look at the navigation property type. `ICollection<T>` → opt-in flag. `T?` / `T` → fixed
set, listed in `<remarks>` (or documented as an omission).

#### Projection APIs

- **Same `AsNoTracking()` rule** as Read APIs.
- **Return type is intentionally lean** — scalar, ID collection, dictionary, or single sub-entity. Never the full entity
  graph.
- **Naming**: same `Get` / `List` rule as Read APIs (see [Method Naming](#method-naming)); the return type signals the
  projection (e.g. `Task<IReadOnlyList<string>>` for `ListRoleNamesByUserIdAsync`, `Task<IReadOnlySet<CustomerId>>`
  for `ListCustomerIdsWithAvatarAsync`, `Task<AddressEntity?>` for `GetCurrentShippingAddressAsync` because it returns
  a single sub-entity, not the parent).
- **Sub-entity returns** (e.g. `GetCurrentShippingAddressAsync` returning an `AddressEntity` instead of the parent
  `CustomerEntity`) belong here, not in Read APIs — they expose only a slice of the parent's graph.
- **Same Compiled Query rules** as Read APIs apply (see [Compiled Queries](#compiled-queries)).

#### Existence Checks

- **Return type is `Task<bool>`.** No data, just yes/no.
- **No `Include(...)`** — these queries should compile to `SELECT EXISTS(...)` or equivalent.
- **Naming**: `XxxExistsAsync`, `IsXxxAsync`, `HasXxxAsync`, `UserHasXxxAsync`. Always reads as a yes/no question.

#### Mutation APIs

- **One service method per use case.** Mutations are exposed as dedicated methods that take only the changeable fields
  as parameters (e.g. `UpdateOrderAsync(publicId, shippingAddress, note, …)`), not as a generic `UpdateAsync(entity)`
  overload.
- **Tracked entities never leave the service.** A mutation method loads the entity internally with tracking, validates,
  mutates, calls `SaveChangesAsync`, and returns either a `bool` success indicator or a freshly loaded *detached*
  entity. Callers never see a tracked entity.
- **Each mutation method controls its own includes.** Load only the navigations needed for that specific mutation —
  typically *less* than what the corresponding Read API loads.
- **No "load → mutate → update" pattern across the API boundary.** Callers must not retrieve an entity via a Read API,
  modify it, and pass it back. This pattern is unsafe (no field-level control over what is changeable, double
  validation, concurrency leaks) and breaks down across process boundaries (client-side UIs such as Blazor WASM have no
  `DbContext`).

#### Compiled Queries

These rules apply only if the repository uses EF Core compiled queries.

- Each compiled query must be wired to at least one service-API method. Unused compiled queries are dead code — either
  wire them up or delete them.
- The compiled query's `Include(…)` shape must match the service method's documented contract exactly so it can serve as
  a drop-in hot-path replacement.
- If the compiled hot-path is switchable (the repo instructions name the toggle), both branches must yield identical
  results — verify with `[Theory] [InlineData(false)] [InlineData(true)]` toggle tests.
- Compiled queries cannot accept a `CancellationToken`. Document on the hot-path branch that cancellation is best-effort
  only.
