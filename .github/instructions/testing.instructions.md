---
applyTo: "**/*Tests/**,**/*TestUtilities/**"
---

# Testing Rules

<!-- Shared baseline, owned by the build.net repository (https://github.com/corvish-hq/build.net). -->
<!-- Consumer repos receive a copy on every build — never edit the copy, change build.net instead. -->

> [!NOTE]
> Rules for .NET test projects and test infrastructure (xUnit, bUnit, Coverlet). Tests in other languages get their
> own topic file.
>
> Placeholders (`<Product>`, `<UiPrefix>`) and general rules are explained in
> [base.instructions.md](base.instructions.md); the concrete placeholder values come from the repo instructions,
> which also win where they contradict this file.

## Testing

| Section | Covers |
|---|---|
| [Running Tests](#running-tests) | When a test result may be reported as green |
| [Authoring Rules](#authoring-rules) | Naming, one member per test, AAA, assertions, test data, test models |
| [Test Structure](#test-structure) | File organization, ordering, narratives, `#region`, time, async, refactoring, integration tests |
| [Common Test Topics](#common-test-topics) | Equality, `GetHashCode()`, comparison/ordering |
| [Component Tests (bUnit)](#component-tests-bunit) | Rendering, markup assertions, JS interop, variant parameters |
| [Coverage](#coverage) | Coverage target, exceptions, abstract base classes |
| [SQL Statements in Test Helpers](#sql-statements-in-test-helpers) | Raw string literals for SQL (data-layer repos only) |

### Running Tests

> [!IMPORTANT]
> Report tests as green **only** after a fresh build and a real `dotnet test` run in that session.

- Build first, then run `dotnet test` (solution or the affected projects). Never conclude from existing test
  executables or older results — they may be stale and hide compile errors in test projects.
- Tests run on **Microsoft.Testing.Platform** (`"test": { "runner": "Microsoft.Testing.Platform" }` in
  `global.json`), so `dotnet test` takes `--project <path>` / `--solution <path>` instead of a positional path, e.g.
  `dotnet test --project src/Foo.Tests -c Release --no-build`.
- Coverage is collected by `coverlet.MTP`: add `--coverlet --coverlet-output-format cobertura` to the `dotnet test`
  call (the CI workflow is the reference for the full argument set).
- State what ran: command, configuration, and the summary counts (total / succeeded / failed / skipped).
- Skipped or not-run tests are not green. Name them and why they did not run.
- Never opt into tests that call paid or external services (e.g. live model evaluations) unless the user asks for it.

### Authoring Rules

> [!WARNING]
> These rules are treated as **hard requirements**.

#### Naming

Test method names must follow: **`Method_State_Expectation`**

| Pattern | Example |
|---------|---------|
| Regular method | `SomeMethod_WhenInputIsNull_ThrowsArgumentNullException` |
| Constructor | `Constructor_WhenInputIsNull_ThrowsArgumentNullException` |
| `Validate()` (Options) | `Validate_AllowedOrigins_WhenEmpty_Fails` |

**`Validate()` tests** for `IValidatableObject` options classes use a **4-part** name:
`Validate_<Property>_<Condition>_<Expectation>`. The `<Property>` segment identifies which property the test targets.
Cross-cutting tests that exercise the overall validation gate (e.g., disabled feature, fully configured options) may
omit the property segment and use the standard 3-part name.

**Overloaded members** get an overload segment after the member name that names what distinguishes the overload —
usually the differing parameter type (`Write_ByteArray_WhenCountIsNegative_Throws…` /
`Write_Span_WhenEmpty_WritesNothing`; `Constructor_Capacity_…`; `Ignore_Generic_…` for `Ignore<T>()`). Members without
overloads use the plain 3-part name; add the segment only once an overload exists (and then to all tests of that
member, so the overloads stay distinguishable).

#### One Member per Test Method

- Each test method must **target exactly one** member (method, property, or constructor overload): Act invokes that
  member, and Assert verifies its effect.
- **Arrange may use other members** (constructor, setup calls, preparatory operations) to bring the object into the
  required state. The coverage they get incidentally does **not** count as testing them — they still need their own
  tests. If Arrange grows large, move it into a helper (`FooTests.Helpers.cs`); if it stays unwieldy, consider it a
  signal of excessive coupling in the production code.
- When the behavior only **emerges from several members together** (e.g. write → read roundtrip), write an
  [integration test](#integration-tests) instead.
- Do **not** consolidate tests for different members into a single `[Theory]` test.
- Different constructor overloads count as **separate members** and require their own dedicated test methods.
- A `[Theory]` with `MemberData` is appropriate when testing the **same member** with different input values, not when
  testing different members.

#### No Trivial or Tautological Tests

Every test must verify observable behavior or an invariant that could realistically break. Do **not** write tests
that:

- only check that a settable auto-property returns the value just assigned to it (constructor tests that verify the
  resulting state, including that arguments were stored, are **not** trivial — a constructor that drops an argument
  is broken);
- restate the implementation (e.g. compute the expected value with the same formula as the production code);
- assert on mocks/fakes set up in the same test, so the production code is never actually exercised;
- exercise framework or library behavior instead of own code.

#### AAA Pattern

Each test must follow the AAA pattern with inline comments:

```csharp
// Arrange
var sut = new MyClass();

// Act
var result = sut.DoSomething();

// Assert
Assert.Equal(expected, result);
```

**Special cases:**
- If Act and Assert coincide (e.g., `Assert.Throws(...)`), use: `// Act + Assert`
- For exception tests: `var ex = Assert.Throws<...>(() => ...)` then assert on `ex`.
- When the message contains dynamic parts (e.g., file paths), use `Assert.Matches` with a regex that validates the full
  message structure while tolerating the dynamic segments.
- **Always verify observable exception state** — asserting only the exception type is insufficient. Capture the
  exception and assert on its domain-relevant properties:

| Exception type | Property to assert |
|---|---|
| `ArgumentException` (and derived types) | `ParamName` |
| `ArgumentOutOfRangeException` | `ParamName` and `ActualValue` (set by the BCL `ThrowIf…` helpers as well) |
| `ObjectDisposedException` | `ObjectName` |
| `InvalidOperationException` (custom message) | `Message` (exact match) |
| Any exception with a **custom** domain-specific message | `Message` (exact match) |

> [!WARNING]
> **Always assert on `Message` precisely** when the message is a **custom string set by production code**. Verifying
> only the exception type is insufficient — the message carries domain intent.
>
> **Never assert on default BCL exception messages** (e.g., `"Value cannot be null."` from `ArgumentNullException`,
> `"The value cannot be an empty string..."` from `ArgumentException.ThrowIfNullOrWhiteSpace`). These messages are
> **localized** by .NET satellite assemblies and will fail on non-English systems. Do not rebuild them via the
> exception constructor either — that tests the BCL, not our code.

- **Assert the exact exception type.** Use `Assert.Throws<T>()` / `Assert.ThrowsAsync<T>()`, not `ThrowsAny`: derived
  types carry meaning (e.g. `ArgumentNullException` vs. `ArgumentException` from `ThrowIfNullOrWhiteSpace()`), and
  `ThrowsAny` hides which one was thrown. The only exception is `OperationCanceledException`, where whether
  `TaskCanceledException` surfaces is an implementation detail — use `ThrowsAny` there with a comment saying so.
- **Distinct failure cases need distinct observable state.** If two failure cases of one member are only
  distinguishable by message text (same type, `ParamName`, and `ActualValue`), the production code throws with its
  own (English, non-localized) message, and the test asserts it exactly.
- **Test value ranges at every boundary.** For each limit, test the last valid value (does **not** throw) and the
  first invalid one (throws, with `ParamName` and `ActualValue`). Only the valid-side test catches an off-by-one
  such as `<` vs. `<=`.

#### Semantic Inline Comments for Complex Tests

For complex tests (multi-step scenarios, fault injection, OS-specific behavior, cross-component interactions), add
inline comments that explain **why** something is done, not **what** the code does. Focus on:

| Area | Example |
|---|---|
| **Detection mechanisms** | Explain that `FileShare.None` is how `CleanupOrphanedFolders()` distinguishes active from orphaned folders |
| **Execution order in fault-injection** | Which step succeeds before the fault fires (e.g., "Directory.Delete already ran, then the monitor intercepts File.Delete") |
| **Logger setup** | Why `logger.Entries.Clear()` is needed (clearing constructor-time entries to isolate explicit call output) |
| **Timing dependencies** | Why setup must happen before/after a specific call (e.g., "orphans must exist before manager construction because the constructor triggers cleanup") |
| **Cross-references** | Link to counterpart tests that verify the same scenario from a different perspective |
| **Dead code in setups** | If a variable is created only as scaffolding (not part of the assertion), explain its purpose — or remove it |

#### Staged Approach for Test Implementation

When implementing tests (especially for complex classes), use a staged approach:

| Stage | Focus | Checkpoints |
|-------|-------|-------------|
| **1. Implementation** | Functionality, test logic, AAA pattern | Compiles, tests green |
| **2. Coverage Review** | Run Coverlet, identify gaps, close with targeted tests | 100% coverage, or each remaining gap justified (see [Accepted Gaps](#accepted-gaps)) |
| **3. Structure** | Test ordering (see [Test Ordering](#test-ordering-within-a-file)), `#region` | Order correct |
| **4. XMLDocs** | `<see cref=""/>`, `<paramref/>`, `<see langword=""/>` | All tags correct |

Coverage review comes **before** structure/XMLDocs to avoid rework — once all tests exist, structure and documentation
can be finalized in one pass.

#### XML Documentation

- Every test method must have complete XMLDocs.
- For `[Theory]` tests, include `<param>` for each method parameter.

#### Assertions

- Validate the **complete expected state** of the object — prefer exact string matches over substring checks.
- For data structures with multiple observable properties, use state verification helpers to ensure all properties are
  checked consistently.
- **Verify all elements in small collections** — when a list/array has a known, small number of elements (e.g., 3–10),
  assert on **every** element. Checking only the first element or the count alone is insufficient — it leaves
  uncertainty about unchecked elements.
  - For large collections (e.g., chunk-sized datasets), verifying count + first + last element is acceptable.
- Test both:
  - the *positive expectation* (what must happen)
  - and a *negative expectation* (what must **not** happen), to rule out side effects.

#### Strongest Assertion Principle

> [!WARNING]
> **Always use the strongest assertion the scenario allows.** Weak assertions that merely confirm existence or
> non-emptiness when the exact expected state is known are considered defects in the test — they give false confidence
> without catching regressions.

| ❌ Weak (avoid) | ✅ Strong (prefer) | When |
|---|---|---|
| `Assert.NotEmpty(collection)` | `Assert.Single` + `Assert.Equal` | Exactly 1 element expected |
| `Assert.Contains(item, collection)` | `Assert.Single` / `Assert.Equal` | Collection size is known |
| `Assert.NotNull(x)` alone | `Assert.NotNull` + property assertions | Object has observable state |
| `Assert.True(x > 0)` | `Assert.Equal(expected, x)` | Exact value is known |
| `Assert.Contains("substring", text)` | `Assert.Equal(exactExpected, text)` | Full string is deterministic |
| `Assert.True(x)` / `Assert.False(x)` | Typed assertion (`Assert.Equal`, `Assert.Single`, etc.) | A more specific overload exists |

The guiding question: *"Do I know the exact expected value?"* If yes, assert it exactly. If the test only checks *part*
of a known value, future regressions in the unchecked part go undetected.

#### State Verification Helpers

When several tests verify the same multi-property state (e.g. `Count`, `IsEmpty`, `Capacity`, and the elements of a
collection type), put those assertions into one `AssertXyzState(...)` helper in `FooTests.Helpers.cs`. Every test
then checks the complete state the same way, and a newly added property only has to be added in one place.

#### Theory-First Planning

> [!IMPORTANT]
> Plan `[Theory]` vs. `[Fact]` **before** writing test code — don't create individual `[Fact]` methods first and
> consolidate later.

- **Group scenarios by Act+Assert pattern up front.** If multiple scenarios share the same Act call and Assert structure
  and differ only in inputs/expected outputs, write them as `TheoryData` rows from the start. The goal is reduced
  boilerplate while preserving line and branch coverage.
- **Per data row:** add a `caseName` / `scenario` string as the first element (readable test output) plus a short
  comment describing the branch or edge case the row covers.
- **Don't over-generalize.** Keep dedicated `[Fact]` tests when a scenario needs special setup/teardown, distinct
  assertions, or markedly better readability standalone.
- Prefer `TheoryData<…>` over `IEnumerable<object[]>` for type safety.
- **Test data placement:** when `TheoryData` / `MemberData` lives in the **same file** as its test, place it **before**
  the test method. If it grows too large, move it to `FooTests.Bar.TestData.cs` (see [Organization
  Model](#organization-model)).

#### Test Data

- Prefer neutral English test data strings/examples in tests.

#### Test Models

Minimize test-only model types — reuse existing ones where possible. New test models go in `FooTests.TestModels.cs`
(partial file) and must have XMLDocs explaining their purpose and which scenario they support.

### Test Structure

#### Organization Model

| File Type | Content |
|-----------|---------|
| `FooTests.cs` | Anchor file: `partial class`, `[Trait(...)]`, optionally simple property tests (fewer than 5) |
| `FooTests.Construction.cs` | Constructor and factory method tests |
| `FooTests.Bar.cs` | Tests for method `Bar()` |
| `FooTests.Bar.TestData.cs` | Test data (`MemberData`, `TheoryData`) for `Bar()` tests |
| `FooTests.Properties.cs` | All property tests, each in `#region` |
| `FooTests.Comparison.cs` | `Equals()`, `GetHashCode()`, equality/relational operators, `CompareTo` |
| `FooTests.Helpers.cs` | Helper methods (factory, setup, assertion helpers) |
| `FooTests.TestModels.cs` | Test-only data models |

**Splitting guideline:** Create a separate file when a feature scope has **5 or more** tests. Scopes with fewer than 5
tests should be placed in the **anchor file** (`FooTests.cs`) using `#region` blocks. Avoid one-file-per-method
granularity — it fragments navigation without adding clarity.

#### Test Ordering (within a file)

When a test file contains tests for **exactly one** method/property, order test methods top-to-bottom along the path
through the member:

1. **Regular operation** — typical usage, then boundary and edge cases that complete normally. This includes every
   outcome the API presents as normal: negative results (`false`, `null`, empty collection), a `TryX()` returning
   `false`, a result object carrying a failure.
2. **Argument checks** — inputs rejected on entry, before any work is done (`ArgumentNullException`,
   `ArgumentOutOfRangeException`, …).
3. **Failure cases** — valid input, but execution fails because of object state, environment, or a collaborator
   (`ObjectDisposedException`, `InvalidOperationException`, I/O or network errors, cancellation).

If a file contains tests for **multiple** methods/properties, group them using `#region` blocks (one region per
method/property), and apply the same ordering *within* each region.

When adding new tests, immediately insert them in the correct position. Do not append out of order even temporarily.

#### Options / Validation Tests

All tests of an options class's `Validate()` call the same method, so a single ordered list would mix all
properties. Instead, give the file one numbered section per property (see [Narrative
Structure](#narrative-structure-test-files)):

```csharp
// --- 1. General ---         cross-cutting: feature disabled skips validation, fully configured options pass
// --- 2. AllowedOrigins ---  accepted values first, then rejected ones
// --- 3. MaxAge ---          accepted values first, then rejected ones
```

Order the property sections as `Validate()` checks them. This shows at a glance that every property has both an
accepted and a rejected case.

#### Narrative Structure (Test Files)

Test files whose tests follow a **multi-step progression or escalation path** should tell a coherent story. Readers
opening a 500-line test file need a map — not a random collection of methods. (The 5-test splitting guideline is a
useful signal, but the real trigger is *narrative content* — a file with 8 independent property tests needs no
escalation story.)

**File-level narrative comment** — Place a block comment directly above the `partial class` declaration that:
- Names the theme in one line.
- Lists the numbered sections with a one-line summary each, referencing test method name suffixes in parentheses.
- Ends with a cross-reference to related test files if the subject is split across partials.

```csharp
// Connection lifecycle: from first connect through repeated failures.
//
// These tests follow the connection from its initial handshake to the edge cases
// that arise when things go wrong — and how it recovers (or doesn't):
//
//   1. Happy path: connect, authenticate, ready (ConnectsSuccessfully).
//      Connecting twice on the same instance is idempotent (CalledTwice).
//
//   2. Configuration gates: missing credentials → ConfigurationRequired.
//      No retry — the operator must fix the configuration.
//
//   3. Error classification: transient errors → auto-retry;
//      permanent errors → manual intervention.
//      ...
//
// For reconnection-specific scenarios, see Reconnection.
public sealed partial class ConnectionManagerTests
```

**Section dividers** — Between groups of related tests, insert a numbered divider comment:

```csharp
// --- 1. Happy path: connect and idempotency ---
```

The number matches the file-level overview. The title is short and descriptive. When inserting a new section, renumber
subsequent dividers and update the file-level overview to match.

**Ordering** — Tests within a file should follow an **escalation path** or **logical story** (e.g., simplest success →
configuration gates → transient errors → permanent failures → double failures). The [test
ordering](#test-ordering-within-a-file) (regular operation → argument checks → failure cases) still applies *within*
each numbered section.

**Inline narrative comments** — When one test is the logical counterpart or continuation of another, say so in the
Arrange comment (e.g., "same schema as the previous test, but this time we inject a timeout instead of an I/O error").

**Cross-file narrative (partials)** — When a test class is split across multiple partial files:
- The **anchor file** (`FooTests.cs`) carries the overall plot: a numbered `<list>` in XMLDoc that maps each partial
  file to its chapter in the story, plus a "Reading order" paragraph.
- Each **partial file** has its own file-level narrative for its internal sections, plus cross-references to neighbor
  files and the anchor.
- Keep cross-references to **file suffixes only** (e.g., "see StartAsync()", "see Reconnection"), not to specific
  sections or line numbers — suffixes are stable across refactors.

#### `#region` Usage

The general [Regions](csharp.instructions.md#regions) rules apply; test files add these specifics:

- **Avoid** `#region` for "Happy Path" vs "Error Cases" grouping — test ordering handles this.
- **Avoid** redundant `#region` in files that contain tests for **exactly one** member and are already clearly named.
- **Always use** `#region` when a single test file contains:
  - **Multiple method overloads** (e.g., `Ignore()` and `Ignore<T>()`)
  - **Multiple properties** in `FooTests.Properties.cs`
  - **Multiple distinct methods** being tested in the same file

- **Region names identify the member exactly**, in the same form used in text references: properties by name
  (`#region Count`), methods with parentheses (`#region Clear()`). Add the parameter types **only** when the method
  is overloaded, so each region names exactly one overload (`#region Write(byte[], int, int)` /
  `#region Write(ReadOnlySpan<byte>)`, `#region Ignore(Task)` / `#region Ignore<T>(Task<T>)`). Constructors use
  `#region Constructor` or, if overloaded, `#region Constructor(int)`.

Example:

```csharp
#region Ignore(Task)
// Tests for non-generic Ignore(Task)
#endregion

#region Ignore<T>(Task<T>)
// Tests for generic Ignore<T>(Task<T>)
#endregion
```

#### Deterministic Time

- Never use `TimeProvider.System` (or `DateTime.UtcNow` / `DateTimeOffset.UtcNow`) in tests when it can be avoided.
  Inject a deterministic time source (e.g. `FakeTimeProvider` from `Microsoft.Extensions.TimeProvider.Testing`) so
  time-dependent assertions can check **exact** values (see [Strongest Assertion
  Principle](#strongest-assertion-principle)).
- Production code that depends on the current time must therefore accept a `TimeProvider` instead of reading the
  system clock directly. If the code under test still reads the clock itself, **adapt the production code** (inject
  `TimeProvider`, `TimeProvider.System` as the production registration) instead of working around it in the test —
  name this change in the plan, since it touches production code.

#### Async Tests & Deadlock Prevention

- **Long-running detection, not a hard timeout.** `xunit.runner.json` sets `longRunningTestSeconds`, which
  *reports* tests that exceed the threshold — it does **not** abort them. A truly hung test is only stopped by the CI
  job timeout, so every intentional wait must be bounded in the test itself.
- **Bound intentional waits with `Task.WaitAsync(TimeSpan)`** (BCL). On expiry it throws `TimeoutException`, failing
  the test at exactly that line, and it releases its timer when the task completes first. Add a short comment naming
  what the test waits for. Do **not** hand-roll `Task.WhenAny(task, Task.Delay(...))` timeouts or a custom helper.

| Operation | Bound with `WaitAsync`? | Why |
|-----------|-------------|-----|
| `await signal.Task` / `await semaphore.WaitAsync()` | ✅ Yes | Explicit wait on an async primitive |
| `await Assert.ThrowsAsync(() => waitTask)` | ✅ Yes | Waiting for a task to throw — pass `waitTask.WaitAsync(...)` |
| Synchronous `.Wait()` on a primitive | ✅ Yes | Prefer the primitive's timeout overload (`Wait(TimeSpan)` + assert the result); `Task.Run(...)` + `WaitAsync` only if none exists — a hang then blocks a pool thread, so comment it |
| Enqueue/set calls, property access, assertions | ❌ No | Not intentional waits |

Do **not** use `await using` for the **SUT** in unit tests — if the implementation is buggy, `DisposeAsync()` may hang,
causing the test to hang instead of failing with a clear error message.

#### Refactoring Tests

- Ensure **all existing test cases remain present**.
- **Preserve test count:** Run tests before and after refactoring. The discovered test count must remain unchanged. If
  it changes, explain which cases were added/removed/merged.
- **Do not** delete and regenerate — only move/copy existing blocks.
- Prefer consolidating regular-operation cases into existing `Theory`/`MemberData` tests instead of adding new `Fact`
  tests.

#### Integration Tests

Integration tests verify the **interaction between multiple components** rather than a single member in isolation.
Because they are inherently cross-cutting, the unit-test rules above are relaxed:

| Rule | Unit tests | Integration tests |
|------|-----------|-------------------|
| One member per test | **Required** | **Relaxed** — a test may exercise a workflow spanning multiple types |
| `Method_State_Expectation` naming | **Required** | **Relaxed** — use `Scenario_Expectation` (e.g., `Roundtrip_WriteAndRead_PreservesAllData`) |
| `FooTests` class naming | Mirrors the SUT type | Use descriptive names: `*RoundtripTests`, `*IntegrationTests`, `*EndToEndTests` |
| Organization model (`FooTests.Bar.cs`) | **Required** | Optional — keep all tests in one file unless the file grows large |

**When to write an integration test instead of (or in addition to) a unit test:**

- The behavior under test only emerges from the **cooperation** of two or more components (e.g., writer → reader
  roundtrip).
- Mocking the collaborator would remove the very thing you want to verify (e.g., database file format compatibility).

**Trait annotation:** Mark integration tests with `[Trait("Category", "<scope>")]` (e.g., `"Import"`,
`"Initialization"`) so they can be filtered in CI.

**External or costly resources (opt-in):** The criterion is not "integration test or not", but whether the test runs
on every machine without setup and free of charge.

- **Run by default** (locally and in CI): tests whose dependencies are part of the regular development setup (e.g. a
  local database or a container the setup already provides).
- **Opt-in only**: tests that use paid, external, or hardware-dependent resources (cloud AI models, metered APIs,
  special hardware, real accounts). They must
  - carry a dedicated `[Trait("Category", "<scope>")]` so they can be filtered,
  - **skip themselves** when their configuration is missing (`Assert.SkipUnless(configured, "<what to set up, where
    it is documented>")`) — a trait alone is not enough, because an unfiltered `dotnet test` would still run them and
    incur costs,
  - read credentials only from environment variables or a local, untracked configuration file — never from the
    repository,
  - document the setup in a README next to the test project.

### Common Test Topics

#### Equality / GetHashCode

- When testing `GetHashCode()`, always compare **multiple equal instances**.
  - Rationale: this detects accidental reference-based hash implementations.
  - Example: create `a` and `b` with the same logical contents and assert `a.Equals(b)` and
    `a.GetHashCode() == b.GetHashCode()`.
- Additionally, test that `GetHashCode()` is stable across multiple calls on the same instance.
- **Test every equality path with the same cases:** `Equals(T)`, `Equals(object)`, and — if defined — `==` / `!=`.
  Each is a separate member, so each gets its own `[Theory]` (one member per test), all fed by **one shared**
  `MemberData` source of equal/unequal pairs. This catches paths that drift apart (e.g. `==` comparing references).
- `Equals(null)` and `Equals(object)` with an instance of a different type return `false` and never throw.

#### Comparison / Ordering

- Do **not** write tests that merely exercise .NET framework sorting infrastructure (`List<T>.Sort()`, `Array.Sort()`,
  LINQ `OrderBy`) as a wrapper around an already-tested `CompareTo` / `IComparable` implementation. Such tests do not
  cover additional production code paths.
- Focus comparison tests on the mathematical properties of the ordering:
  - **Reflexivity**: `a.CompareTo(a) == 0` (same instance).
  - **Antisymmetry**: if `a < b` then `b > a` (verify both directions inline).
  - **Transitivity**: if `a < b` and `b < c` then `a < c` (requires 3+ elements — use dedicated `[Fact]` tests).
- When a type implements multiple comparison-related members (`Equals`, `==`, `!=`, `CompareTo`, `<`, `>`, `<=`, `>=`),
  prefer a **single shared `TheoryData`** dataset containing `(scenario, valueA, valueB, expectedSign)` rows that all
  members consume. This validates that all members agree on the same ordering.
  - Use `expectedSign = -1` for ordered pairs and `expectedSign = 0` for equal pairs.
  - Each Theory verifies **both directions** (A vs B and B vs A) to cover antisymmetry / commutativity inline — no need
    for separate `+1` rows.
- Assert on `Math.Sign(result)` rather than exact `CompareTo` return values. The `IComparable<T>` contract only
  guarantees the **sign**, not a specific value like `-1`/`0`/`1`.
- Prefer `TheoryData<…>` over `IEnumerable<object[]>` for type safety.

### Component Tests (bUnit)

UI components are tested with **bUnit**, a component-rendering library used inside regular xUnit tests (the test
class derives from `BunitContext`). A bUnit test is an xUnit test, so all authoring rules above apply; these add the
UI specifics:

- **Assert on rendered output, not internals.** Query the markup (`cut.Find(...)`, `cut.FindAll(...)`) and assert on
  elements, attributes, text, and CSS classes the user or a stylesheet depends on. Do not reach into private fields
  or rely on render-tree details.
- **Variant parameters are tested through their CSS modifier class.** For each boolean/enum style parameter, assert
  the modifier class is present when set **and** absent by default (positive + negative expectation).
- **JS interop mode is a conscious choice.** Keep the default strict mode and set up the expected calls
  (`JSInterop.Setup(...)`) when the JS call is part of the behavior under test. Use
  `JSInterop.Mode = JSRuntimeMode.Loose` only when the component calls JS incidentally and the test does not care
  about it — and say so in a comment.
- **Interactions go through the DOM.** Trigger events via the rendered element (`Click()`, `Change(...)`, `Input(...)`)
  instead of invoking component methods directly.
- **Register only what the component needs** in `Services` (fakes over real implementations); a missing service
  should fail loudly rather than be hidden by a catch-all registration.

### Coverage

> [!NOTE]
> Target **100% line and branch coverage** for all reachable production code.

Use **Coverlet** to measure coverage. Focus on uncovered lines/branches, edge cases, error paths.

- **Coverage must come from direct tests.** Every non-private member (public, internal, protected) is covered by
  tests that target it — not incidentally through tests of other members. Coverlet cannot tell the two apart, so a
  green report alone does not prove a member is tested.
- **Private members are the exception:** they are implementation details and are covered through the members that
  use them. If a private member's logic is complex enough to need its own tests, extract it into a testable
  (`internal`) type or method instead of testing it indirectly with contrived scenarios.

#### Coverage Exceptions

| Pattern | Reason |
|---------|--------|
| Static initializers, logging infrastructure | Wiring without own logic |
| `throw new UnreachableException(...)` | By-design unreachable (see [csharp](csharp.instructions.md#assertions--unreachable-code)) |

Do **not** hide reachable logic behind `ExcludeFromCodeCoverage` just to satisfy coverage.

#### Accepted Gaps

100% is the target, not an end in itself. A reachable path may stay uncovered when covering it would cost
disproportionate effort — but only **visibly justified**:

- **Typical cases:** error paths that require simulating platform failures (e.g. an I/O error mid-system-call),
  thread races that cannot be reproduced deterministically, code that only runs on another platform/host, unreachable
  guards (`throw new UnreachableException()`, see [csharp.instructions.md](csharp.instructions.md)). A state that
  external input (database, files, deserialization, callers) can produce is **not** unreachable and must be tested.
- **Improve testability first.** Before accepting a gap, check whether a small seam (`TimeProvider`, an injectable
  dependency, an extracted pure method) makes the path testable — often cheaper than expected.
- **Document the gap** with a short comment at the code or in the test narrative explaining why the path is not
  tested. Unexplained gaps remain defects.
- The gap stays visible in the coverage report — `ExcludeFromCodeCoverage` is still not an option for it.

#### Abstract Base Classes

- Prefer testing via concrete derived classes when that provides meaningful coverage.
- If the abstract base class is part of the public API and its behavior cannot be adequately exercised through existing
  derived types, add **direct tests** using a minimal test-only derived type.
- Prefer tests that are explicitly written for the type/member under test — avoid relying on incidental/indirect
  coverage only.

### SQL Statements in Test Helpers

Applies only to repositories with a data layer.

- SQL statements in test helpers should use raw string literals (`""" ... """`) or raw interpolated string literals
  (`$""" ... """`).
- **Multi-clause** SQL (SELECT/FROM/WHERE, CREATE TABLE with multiple columns, multi-row INSERT) should use multi-line
  raw strings with readable line breaks.
- **Single-clause** SQL that fits within the 120-character line limit should use a **single-line** raw string (e.g.,
  `"""DELETE FROM "Table" WHERE "key" = 'value'"""`).
