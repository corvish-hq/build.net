---
applyTo: "**/*.cs"
---

# C# Code Rules

<!-- Shared baseline, owned by the build.net repository (https://github.com/corvish-hq/build.net). -->
<!-- Consumer repos receive a copy on every build — never edit the copy, change build.net instead. -->

> [!NOTE]
> C#-specific rules: XML documentation, code patterns, and class layout.
>
> Placeholders (`<Product>`, `<UiPrefix>`) and general rules are explained in
> [base.instructions.md](base.instructions.md); the concrete placeholder values come from the repo instructions,
> which also win where they contradict this file.

## XML Documentation

- Add complete XML documentation (`<summary>`, `<param>`, `<returns>`, `<exception>`) for **every member** you add
  or change — not just public APIs; internal and private members matter to maintainers too.
- Use `<remarks>` for additional details, `<example>` for usage samples when helpful.
- **Use the full 140-character line width** in XML documentation. Break at logical points (end of sentence) rather than
  at arbitrary positions.
- For `cancellationToken` parameters, use the standard phrasing: `A token to cancel the operation.` Context-specific
  phrasings are allowed as exceptions.
- **Partial types carry their type-level XMLDoc in exactly one file.** The compiler merges `<summary>` / `<remarks>`
  from all parts, so documenting several parts produces concatenated, contradictory text. The documented part is the
  main file (`Foo.cs`), for Blazor components the code-behind (`Foo.razor.cs`), for test classes the anchor file
  (`FooTests.cs`). Other parts describe their slice with a plain `//` comment above the declaration (e.g.
  `// Restore half of DataPortService: …`); member XMLDoc stays in every file as usual.

### Common Inline Tags

| Tag | Usage |
|-----|-------|
| `<see cref="..."/>` | Link to types, methods, properties |
| `<see langword="..."/>` | Language keywords (`null`, `true`, `false`, etc.) |
| `<c>...</c>` | Inline code |
| `<code>...</code>` | Multi-line code block |
| `<paramref name="..."/>` | Reference a parameter |
| `<typeparamref name="..."/>` | Reference a type parameter |

### Style Guidelines

- Prefer the **shortest possible** type names (use `using` directives to enable short names).
- Ensure proper list and paragraph structure in XMLDocs.
- When updating XML documentation, prioritize improved readability.
- **Keep public XMLDoc consumer-focused.** Public `<summary>`, `<remarks>`, and `<example>` must describe *what* the
  type/member does for its consumers — not *why* it is implemented a certain way. Implementation rationale (e.g.,
  "attributes must be on properties because the validation filter reads them") belongs in **code comments**, not in
  XMLDoc visible to API consumers.
  - **Observable behavior IS contract** and must stay in public docs. If the consumer can see, measure, or react to a
    consequence, document it — even if it sounds implementation-flavored. Examples that **must** appear in public
    XMLDoc:
    - **Side effects** ("a file written to disk during the unit of work remains on disk after the rollback completes if
      a compensation fails").
    - **Process termination** ("aborts the process immediately rather than risk silently corrupting data") — describe
      the consequence, not the mechanism (do **not** name the internal helper that terminates the process).
    - **Ordering guarantees** ("compensations run in LIFO order"), **best-effort vs. all-or-nothing semantics**,
      **cancellation policy** (honored / pre-flight only / not honored), and **state transitions** the caller can
      observe.
  - **Mechanism stays out of public docs.** Internal types, private field names, counter bookkeeping, why a `List<T>`
    was chosen over a `Dictionary<TKey, TValue>`, etc., belong in code comments or in XMLDoc on private/internal
    members. The litmus test: *"Could the consumer act differently based on this information?"* If yes, it is contract;
    if no, it is mechanism.
  - **`<inheritdoc/>` warning:** `<remarks>` and `<exception>` on a method marked `<inheritdoc/>` are aggregated into
    the inherited public documentation by most doc-generation tools. Do **not** put implementation rationale in
    `<remarks>` on `<inheritdoc/>` methods — move it to body comments. Define the public contract (including all
    observable consequences) on the **interface or base class**, then let the implementation inherit it cleanly.

### `<see cref="..."/>` Brevity

Omit the parameter list in `cref` attributes when the method has **no overloads** — the compiler resolves it
unambiguously, and the tooltip stays short.

✅ `<see cref="WriteTableAsync"/>` — ❌ `<see cref="WriteTableAsync(TableSnapshot, ILogger, CancellationToken)"/>`

This is safe: when an overload is added later, the now ambiguous `cref` raises CS0419, which fails the build under
`TreatWarningsAsErrors`.

When **multiple overloads** exist and disambiguation is required, the full parameter list is unavoidable — the compiler
demands it. Accept the longer tooltip in those cases; do **not** use display-text overrides
(`<see cref="...">short name</see>`) because they introduce staleness risk on renames (the `cref` warns, the display
text silently stays wrong).

### Tag Ordering

Tags must appear in this order: `<summary>` → `<typeparam>` → `<param>` → `<value>` → `<returns>` → `<exception>` →
`<remarks>` → `<example>` → `<seealso>`.

`<remarks>` therefore always comes **after** `<param>`, `<returns>`, and `<exception>` — never directly below
`<summary>`. Only `<example>` and `<seealso>` may follow it.

### Exception Tag Style

Use **bare condition** style — do **not** prefix with "Thrown when" or "Thrown if"; the `<exception>` tag already
implies it (same principle as `<returns>` not starting with "Returns").

✅ `<paramref name="value"/> is <see langword="null"/>.` — ❌ `Thrown when <paramref name="value"/> is …`

### `OperationCanceledException` Convention

Do **not** document `OperationCanceledException` for methods that accept a `CancellationToken` — this is an implicit
contract of the parameter itself (same BCL convention as not documenting `OutOfMemoryException`).

**Do** document it when the cancellation behavior is **non-obvious**:

| Document? | Scenario | Example |
|---|---|---|
| ❌ No | Standard `CancellationToken` pass-through | Import/export methods, data service methods |
| ✅ Yes | Method throws it **without** a `CancellationToken` parameter | A custom exception derived from `OperationCanceledException` |
| ✅ Yes | Cancellation races with another signal and the **priority matters** | "Token canceled before the event is signaled" |
| ✅ Yes | Cancellation has **observable side effects** (partial state) | Document in `<remarks>`, not `<exception>` |

---

## Code Patterns

> [!IMPORTANT]
> These patterns are **mandatory**.

### Async/Await

- **Use `ConfigureAwait(false)` on every `await` in library code** to prevent deadlocks and keep code portable.
- **Exception:** UI code that must resume on the UI thread / renderer context (e.g. Blazor components, especially
  those using `IJSRuntime`; MAUI/WPF/WinForms view code) must use `ConfigureAwait(true)` (or omit it).
- **Test projects** follow a two-tier rule ([xUnit1030](https://xunit.net/xunit.analyzers/rules/xUnit1030)):
  - `[Fact]`/`[Theory]` **test methods** — do **not** use `ConfigureAwait(false)` (xUnit controls scheduling via
    `SynchronizationContext`).
  - **Test infrastructure** (harnesses, helper classes, test-operation abstractions) — **do** use
    `ConfigureAwait(false)`, same as production library code. `ConfigureAwait(false)` inside a helper does not propagate
    up the call stack; the test method's own `await` still resumes on the xUnit context.
- Returning a `Task` without `await` is OK; do not introduce `async` only to add `ConfigureAwait(false)`.
- **Async all the way** — always use async APIs in async methods (e.g., `File.ReadAllTextAsync` instead of
  `File.ReadAllText`).

### Task vs ValueTask

- **Default to `Task`** for all async operations.
- **Use `ValueTask`** only in profiled hot paths where synchronous completion is common (e.g., cache hits).

### Span<T> Support

- Implement parsing/encoding/buffer logic on `ReadOnlySpan<T>` / `Span<T>` and let `string` / array overloads
  delegate to it; expose the span overload publicly (e.g. `Parse(string value)` → `Parse(value.AsSpan())`).
- Do not add span overloads to APIs whose logic isn't span-based (e.g. ones that only pass the string on).

### Thread Safety

> [!WARNING]
> Using `ConfigureAwait(false)` means continuations may run on **different threads**. Library code with mutable shared
> state **must be thread-safe**.

### Assertions & Unreachable Code

- `Debug.Assert()` for internal invariant checks is allowed and intended — it documents assumptions and catches
  violations during development. The expression must be **side-effect free** (it is compiled out in Release), and it
  never replaces argument validation or error handling that callers rely on.
- **Unreachable guards** (code that should never execute because all cases are covered): use
  `throw new UnreachableException()` (`System.Diagnostics`) instead of `#if DEBUG` / `#if RELEASE` blocks, and always
  add a comment explaining **why** the path is unreachable (e.g. "All enum values handled above"). A state that
  external input (database, files, deserialization, callers) can produce is **not** unreachable — handle it properly.

### Equality

- Prefer a `record` (or `record struct`) when value equality is wanted — the compiler generates consistent members.
  Generated equality compares every field via its own `Equals()`; collections (`List<T>`, arrays, dictionaries,
  `ImmutableArray<T>`) compare **by reference**. Records with collection members therefore need a custom
  `Equals(T)` / `GetHashCode()` (or a value-equal collection type).
- When implementing equality manually: implement `IEquatable<T>`, let `Equals(object)` delegate to `Equals(T)`,
  override `GetHashCode()` over the same fields, and implement `==` / `!=` (if provided) via `Equals(T)`. All paths
  must agree; never let one of them compare references.

### Disposal

Disposal follows a **three-tier model** to balance `ConfigureAwait(false)` consistency with test ergonomics:

| Context | Pattern | Rationale |
|---|---|---|
| **Library / production code** | `try/finally` + `DisposeAsync().ConfigureAwait(false)` | Ensures `ConfigureAwait(false)` on the disposal `await`. |
| **Test infrastructure** (harnesses, helpers) | `try/finally` + `DisposeAsync().ConfigureAwait(false)` | Infrastructure is library-like code — same rule. |
| **Test methods** (`[Fact]` / `[Theory]`) | `await using` allowed | `ConfigureAwait(false)` is forbidden in test methods (xUnit1030), so `try/finally` adds no value — it becomes pure boilerplate. `await using` is concise and safe here — except for the SUT (see testing rules). |

### Magic Strings

Avoid magic strings in code. Use framework-provided constants (e.g., `HeaderNames.Authorization` instead of
`"Authorization"`).

### Type Choices

- **Classes are `sealed` by default.** Unseal only for a deliberate inheritance design (abstract base, documented
  extension point).
- Fields assigned only during construction are `readonly`. Value types are `readonly struct` / `readonly record
  struct` unless mutation is a measured performance need in local, non-public code.
- **Named types over tuples in public APIs.** Return and accept records or structs, never value tuples, on public
  surface areas.
- **Static classes** only for stateless helpers, extension methods, and constants — anything with state or
  dependencies is an injectable service.
- `=>` bodies only for single expressions (forwarding, computed properties); anything with statements, control flow,
  or side effects gets a block body.

### Null Handling

- Nullable reference types are on; annotations are contract. Treat nullable warnings as design feedback.
- **Collections are never `null`** — use empty collections (`[]`). External input (DTOs, database, JSON) that may
  carry `null` is normalized to empty **once, at the boundary**.
- The null-forgiving operator `!` only expresses knowledge the compiler lacks (e.g. `null!` passed in a null-guard
  test, a framework-initialized property) — never to silence a warning about a value that really can be `null`.
- Guard arguments of public and important internal members with the BCL helpers (`ArgumentNullException.ThrowIfNull`,
  `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative`, …) at method entry.

### Exceptions

- Throw the most specific BCL type (`ArgumentException` family, `InvalidOperationException`,
  `ObjectDisposedException`, …); never `Exception` / `ApplicationException`.
- **Custom messages** state the failed condition, include relevant values (never secrets), and, where possible, how
  to fix it (e.g. the missing configuration key). They are English and not localized.
- **Custom exception types** only for domain failures callers handle specifically. They are `sealed`, small, and
  carry the relevant data as properties so callers never parse message text.
- Rethrow with `throw;` (never `throw ex;`) to keep the stack trace.
- `async void` only for event handlers — exceptions from it cannot be observed by the caller.

### JSON Contracts

Types that define a persisted or exchanged JSON format (seed files, exports, transfer codes, external API payloads)
are contracts — renaming a C# member must never silently change the format.

- JSON property names are **camelCase**.
- Every property of a contract type pins its name with `[JsonPropertyName]`; the naming policy of the serializer
  options is not relied on.
- Boolean properties carry `Is` / `Has` / `Are` / `Can` in C#, but drop the prefix in JSON
  (`IsEnabled` → `enabled`, `AreToolsEnabled` → `toolsEnabled`).
- Purely internal JSON (e.g. caches never read by another version) may rely on the serializer naming policy.

---

## Class Layout

### Naming

- Methods returning `Task` / `ValueTask` (or `IAsyncEnumerable<T>`) end in `Async`.
- Booleans read as a positive question: `IsEnabled`, `HasAccess`, `CanRetry` — no `Enabled`, no negations
  (`IsNotDisabled`, `HasNoAccess`) unless that is the domain term.
- Collections get a plural name without the type: `Users`, not `UserList` / `UserCollection`.
- File-scoped namespaces matching the folder structure; one primary type per file (private nested helpers may stay).

### Field Naming

Follow the .NET runtime convention:

| Field kind | Style | Example |
|---|---|---|
| Private / internal instance field | `_camelCase` | `_contributors` |
| Private / internal static field (mutable) | `s_camelCase` | `s_instanceCount` |
| `[ThreadStatic]` field | `t_camelCase` | `t_buffer` |
| Constant (any visibility) | `PascalCase` | `DefaultTimeout` |
| `static readonly` field (any visibility) — used like a constant | `PascalCase` | `DefaultStalenessThreshold` |
| Public / protected field | `PascalCase` | `Empty` |

### Member Ordering

Members of a type appear in this order:

1. Constants
2. Static fields
3. Instance fields
4. Constructors (static constructor first)
5. `Dispose()` / `DisposeAsync()`
6. Properties and indexers
7. Events (field-like and with accessors alike)
8. Public methods, then operators
9. Protected / internal methods
10. Private methods
11. Nested types

Within each group, static members come before instance members.

### Type Structure

Regions extend the member ordering; the full skeleton of a type is:

1. Fields, constructors, `Dispose()`, properties, events — never in a region.
2. Public methods and operators that do not belong to a feature region.
3. Feature regions (see [Regions](#regions)), each with its own private helpers.
4. Remaining protected/internal and private methods, including helpers shared by several regions.
5. Explicit interface implementations, in a region.
6. Nested types.

### Regions

`#region` groups **cohesive blocks** of a type. Use it only for these cases:

| Use | Example |
|---|---|
| Explicit interface implementations | `#region Implementation of IList<T>` |
| Feature sections of a type with several distinct responsibilities | `#region Read` / `#region Write` in a stream |
| Fixed structural sections defined by these instructions | Data service categories, test files covering several members |

- **A feature region owns its private helpers.** A private helper used only by one region lives inside that region,
  so readers of the rest of the type never meet it. Inside a region, the member ordering applies.
- **Never** group by visibility or member kind (`#region Private Methods`, `#region Fields`) — the member ordering
  already does that.
- **Never** put a region inside a method body.
- **Never** use regions to hide the size of a type. If a type needs many regions to stay readable, split it (partial
  files per feature or separate types).
- Name a region after its content, without decoration (`#region Read`, not `#region ===== READ =====`).

### Collection/Data Structure Layout

For classes implementing `IList<T>`, `ICollection<T>`, etc., the type structure above looks like this:

```csharp
// Public core API — no region, prominently visible at the top
public void Insert(int index, T item) { }
public bool Remove(T item) { }

// Private helpers
private void CopyToArray(...) { }

#region Implementation of IList<T>
bool ICollection<T>.IsReadOnly => false;
void ICollection<T>.Add(T item) { }
#endregion

// Nested types — public API, outside the region
public struct Enumerator { }
```

Implicit implementations (`public void Bar()`) and base class overrides are core API — they belong at the top without
a region.
