---
applyTo: "**"
---

# Copilot Instructions

<!-- Shared baseline, owned by the build.net repository (https://github.com/corvish-hq/build.net). -->
<!-- Consumer repos receive a copy on every build — never edit the copy, change build.net instead. -->

> [!NOTE]
> **Scope of this file.** These are the repository-independent rules shared by every repo that includes the
> `build.net` submodule. Each consumer repo extends them in `.github/instructions/repo.instructions.md`
> (product names, paths, domain rules, concrete design tokens). Where a rule here names a placeholder, the
> repo instructions define its concrete value:
>
> | Placeholder | Meaning | Example |
> |---|---|---|
> | `<Product>` | Namespace root / product name | `Contoso` → `Contoso.Data`, `ContosoDbContext` |
> | `<UiPrefix>` | Prefix of project-owned UI components (components, controls, views) | `Ui` → `UiDialog` |
>
> The repo instructions also define the **target framework** and the **project layout** (which project holds UI
> primitives, the data layer, …). This file never assumes concrete paths.
>
> If repo instructions contradict this file, the **repo instructions win** for that repository.

## Table of Contents

1. [General](#general)
2. [Working with Code](#working-with-code)
3. [Workflow](#workflow)
4. [Privacy & Data Minimization](#privacy--data-minimization)
5. [Logging & Configuration](#logging--configuration)
6. [Code Layout & Naming](#code-layout--naming)
7. [Commit Messages](#commit-messages)

### Topic Files

The rules are split by topic. Each file is loaded automatically for the files matching its `applyTo` front matter
(the only place the patterns are defined); when planning work that will touch a topic before any matching file is
open, **read the topic file first**.

| File | Covers |
|---|---|
| `base.instructions.md` | General, architecture, workflow, review, logging, naming, commits (this file) |
| [csharp.instructions.md](csharp.instructions.md) | XMLDoc, type choices, null handling, exceptions, equality, async |
| [ui.instructions.md](ui.instructions.md) | UI discipline, rendered elements, accessibility, localization, theming |
| [testing.instructions.md](testing.instructions.md) | Test authoring, structure, determinism, coverage, test SQL |
| [data.instructions.md](data.instructions.md) | Entities, `DbContext`, migrations, service APIs |

---

## General

### Standard Stack

All repositories share this tool stack. Repo instructions may override individual entries; they always define
the **target framework**, which is repository-specific.

| Area | Standard |
|---|---|
| Runtime / language | .NET (version per repo instructions), C# |
| Test framework | xUnit (global settings in `xunit.runner.json`) |
| UI component tests | bUnit (on top of xUnit; only in repos with a Blazor UI) |
| Coverage | Coverlet |
| Data access | EF Core (see [data.instructions.md](data.instructions.md); only repos with a data layer) |
| Configuration | `appsettings.json` + options classes |
| Third-party attribution | `THIRD-PARTY-NOTICES.md` in the repository root |

### Repository Constraints

- Do **not** perform any **write** operations using Git (e.g., commit, push, reset, rebase, merge, tag, branch
  creation/deletion).
- **English everywhere in the repository** — code, comments, XMLDoc, documentation, commit messages, test data.
- Localization is only active in the user-facing UI. API responses (including API validation errors), log
  messages and exception messages remain in **English**.

### Third-Party Notices

When **adding, removing, or updating** a third-party dependency (NuGet package, npm package, bundled JS library, or any
other external asset), **always update `THIRD-PARTY-NOTICES.md`** in the repository root accordingly:

- **Adding:** Insert a new entry with Author, License, and URL. For **copied assets or adapted source code** (no
  package manifest records them), also record the **Version** (release tag or commit) — otherwise nobody can later
  tell which upstream state was copied. **Package dependencies** (NuGet, npm) carry **no** version here: the package
  manifest is the single source of truth, and a second hand-maintained copy goes stale on every update.
- **Removing:** Delete the corresponding entry.
- **Updating:** Adjust the entry if the license or author changed. Package version bumps with no license change need
  no update; for copied assets and adapted source, always update the recorded version.

### Terminal Usage

- **Never run interactive commands.** A command waiting for input (pager, confirmation prompt) hangs your
  terminal. Disable pagers (`git --no-pager ...`) and pre-answer prompts with flags (`-y`, `--yes`, `-Force`).
- **Prefer your edit tools for file changes.** They change exactly one visible spot. When a script modifies
  several files or many spots at once, verify the result afterwards (diff, match counts) — bulk replacements can
  silently miss or truncate content.

### Project & Solution Files

- **Package versions live only in `Directory.Packages.props`** (Central Package Management). A `PackageReference` in
  a project file never carries a `Version` attribute.
- **Shared build standards come from `build.net.props`** (nullable, `TreatWarningsAsErrors`, analyzers, XML doc
  generation), imported by the repository's `Directory.Build.props`. Do not weaken them per project; a new warning
  exception needs a justifying comment.
- **Shared properties belong in `Directory.Build.props`** (shared targets in `Directory.Build.targets`) — never
  copy the same property into several project files. A project file contains only what is specific to that project.
- **One location for these files.** `Directory.Build.props`, `Directory.Build.targets` and `Directory.Packages.props`
  live together in the directory that contains all projects (the repo instructions name it). No nested copies —
  MSBuild stops at the first one it finds, so a nested file silently cuts off the shared settings.
- **Every new project is added to the solution file** (`*.slnx` / `*.sln`).
- **No behavior differences** between Debug and Release builds. Language-specific exceptions (debug-only
  assertions) are defined in the language topic files (e.g. [csharp.instructions.md](csharp.instructions.md)).

---

## Working with Code

> [!IMPORTANT]
> These principles apply to **all** code interactions — writing, modifying, reviewing, or debugging.

### Architectural Integrity (Binding Agreement)

> [!CAUTION]
> **This is a binding agreement, not a guideline.** It overrides the natural reflex to propose the smallest possible
> change.

For every non-trivial change, **the architecturally cleanest and most maintainable solution must be named explicitly
first** — even if it is larger than a quick fix. The user then decides, with full information, whether to take the main
path or — for concrete reasons (time pressure, scope) — an interim step. The choice belongs to the user, never to a
convenience reflex.

**What this means in practice:**

1. **No hacks — fix the root cause.** A hack is any change that treats a symptom instead of its cause: a workaround,
   an extra guard around fragile behavior, a fight against library internals, a reliance on undocumented assumptions.
   Hacks are never proposed as a solution. The moment you catch yourself reaching for one, stop and name the
   structural cause and the clean fix instead.
2. **Always show where the steps lead.** When a change is done in several steps, name the end state and all planned
   steps up front — not just the next one — so the user can judge the whole direction before approving the first step.
3. **Respect third-party library boundaries.** Use only the documented public API. If the library cannot do what is
   needed, do not escalate against its internals (undocumented CSS classes, private DOM structure, reflection over
   non-public members) — build a small component or helper the project owns instead.
4. **The user owns the trade-off decision.** Your job is to surface the choice with full context (effort, risk,
   maintenance cost of each option), not to pre-decide based on perceived effort.

#### The Only Exception: User-Approved Interim Steps

A hack enters the code **only** when the user explicitly decides so — after the clean solution has been named, and
for concrete reasons (time pressure, scope, risk). Never take this path on your own initiative. Even then,
**all** of the following must hold:

1. **It has an expiration date.** It is marked with `TODO(<replacement>): <reason>` and a **planned end state** (what
   will replace it and when). A hack without a replacement plan is technical debt by design — forbidden.
2. **It is contained.** It does not leak into other components or contracts. If it does, it is not an interim step —
   it is an architectural change disguised as one.
3. **It is visible.** The `TODO(...)` marker makes it discoverable and tracked. Silent hacks are a defect.

> [!WARNING]
> Violating this agreement — silently taking the path of least resistance, hiding the architectural option, proposing
> a hack instead of naming the structural cause — is a defect. If you notice yourself doing this mid-task, stop and
> surface the architectural option before continuing.

### Code Style & Changes

- **Small scope, full quality.** Change only what the task requires — but implement that part properly, without
  shortcuts (see [Architectural Integrity](#architectural-integrity-binding-agreement)). Build for requirements that
  are known today, not for imagined ones:
  - **Allowed:** abstractions at real boundaries (storage, platform, external services, UI library) and preparation
    for requirements that are already decided (roadmap, second backend, upcoming platform). Doing this early is
    cheaper than rebuilding later.
  - **Not allowed:** layers, interfaces, or generic machinery for a "maybe someday" with no concrete requirement
    behind it. It costs readability and is as much a defect as a hack.
  - When unsure which side a design falls on, name both options with their trade-offs and let the user decide.
- **Follow the codebase-wide conventions**, not just the surrounding code. The reference is these instructions and
  the repository's formatter settings (`.editorconfig`, ReSharper settings). If the surrounding code deviates (e.g.
  imported or legacy code), do not copy the deviation: point it out and propose aligning it as a separate step before
  building on it.
- **Line length limit: 140 characters.** Use the available width — don't break lines unnecessarily short.
- **Comment non-obvious decisions.** When a choice isn't immediately self-evident (empty templates, suppressed defaults,
  workarounds, defensive guards, intentional no-ops), add a comment explaining *why* — not *what*. If someone might
  later ask "why was this done this way?", a comment is mandatory.

### Core Principles

| Principle | Meaning |
|-----------|---------|
| **Verify, don't assume** | Inspect actual implementation before stating how something works |
| **Code > Comments** | When code and comment disagree, the code shows what happens — but ask which one is intended |
| **Ask > Assume** | When unclear, ask instead of guessing |
| **Quality > Speed** | Take time to be thorough; rushing causes false positives |
| **No implicit assumptions** | A class must be self-contained and correct regardless of external operating conditions |

#### No Implicit Assumptions About Operating Conditions

Classes must never silently rely on external context (e.g., "this only runs during shutdown", "the caller always
disposes first"). Either:

1. **Implement defensively** — handle all code paths (including edge cases) so the class behaves correctly regardless of
   how or where it is called.
2. **Document and assert** — if an assumption is truly required, document it explicitly in XMLDoc and enforce it with a
   guard/assert.

> [!WARNING]
> Implicit assumptions are technical debt that causes hard-to-diagnose bugs when someone changes the calling code months
> later. Every exception handler, state transition, and resource cleanup must leave the object in a consistent state —
> **unconditionally**.

### Verification & Review

#### Before Reporting an Issue

Before claiming *any* finding (bug, inconsistency, missing API, wrong behavior), apply these three checks — in order, no
shortcuts:

| Check | Question |
|-------|----------|
| **1. Real?** | Read the code again, check related files, look for comments. Is this actually a problem? |
| **2. Intentional?** | Known pattern? Matches BCL behavior? Defensive reasoning? A reason that's not obvious? |
| **3. Evidence?** | Can you point to specific line numbers? Would "fixing" this break something? |

> [!CAUTION]
> **If ANY doubt remains → ASK, don't claim.** False positives waste time and destroy trust. A clarifying question
> costs far less than a false finding the user has to investigate and refute.

**Stop signals** — if you catch yourself doing any of these, slow down and restart at Check 1:

- Using words like "might", "could", "possibly" in the finding
- Can't provide specific line numbers
- Haven't read all related files
- Trusting a comment without verifying the code

#### Code vs. Comments

When a comment contradicts the code: the **code is the source of truth**, the comment is probably outdated. But — *ask
first*. Sometimes the comment reveals the bug. Report as: *"Line X comment says Y, but code does Z. Which is correct?"*

#### Proactively Report Inconsistencies

When you spot one of these, surface it (with the smallest consistent fix you can suggest):

- Implementation exists but interface/contract is missing
- API is defined in the wrong service interface
- XMLDocs / `cref` points to non-existent members
- Duplicated or contradicting behavior across files
- Code that deviates from these instructions (naming, test structure, XMLDoc, layout, …) — report it even when it is
  outside the current task, but fix it only after approval (see [Workflow](#workflow))

---

## Workflow

### Features / Fixes / Test Changes

> [!IMPORTANT]
> When you are asked to implement a feature or fix a bug:
>
> 1. **Think through** the intended implementation first.
> 2. **Communicate a detailed plan:**
>    - What will change
>    - Which files are expected to be touched and why
>    - How correctness will be validated
> 3. **Wait for an explicit start signal** before making any code changes.

This is required so reviewers can influence the approach before code is changed.

- **No unilateral changes or "optimizations".** Refactorings, clean-ups, or improvements outside the agreed plan
  are proposed, not applied. Present them and wait for explicit approval.
- **Exception:** when a task is explicitly about repository instructions, instruction files may be updated
  immediately without waiting for a start signal.

---

## Privacy & Data Minimization

- **Store only what a feature actually needs** (or what the law requires), and only as long as it is needed.
- Design deletion and anonymization flows accordingly — data the user deletes is really gone.

---

## Logging & Configuration

### Configuration Discoverability

- Surface new configuration keys in `appsettings.json`.
- Secrets (API keys, passwords, tokens) never reach a log, an exception message or diagnostic output — note that a
  record's generated `ToString()` prints every property, so a secret-holding type is a class or overrides
  `PrintMembers()`. Secrets never live in committed configuration files; supply them via environment variables or
  user secrets. A secret the application cannot run without is validated at startup (`ValidateOnStart()`).

### Logging Guidance

- Startup log messages (not noisy warnings) for important configuration trade-offs.
- Automatic cleanup or repair jobs log **Information** when there was nothing to do, **Warning** when they delete
  or change data.
- **Structured logging:** use message templates with named placeholders (`"User {UserId} logged in"`), never string
  interpolation — the placeholders become queryable properties.
- **Log an exception once, where it is handled** (the boundary where it becomes visible: background worker loop,
  endpoint, UI action), with the exception object and context values. Do not log-and-rethrow at every layer.
- **Never swallow exceptions silently.** A `catch` that neither rethrows nor logs must state in a comment why the
  error is fully handled.
- Never log secrets, tokens, or personal data (see [Privacy](#privacy--data-minimization)).

---

## Code Layout & Naming

### Terminology

- Use the domain terms defined in the repo instructions consistently (e.g. never mix two words for the same concept).
- Rename methods promptly when semantics change.

### File Headers

New and modified source files carry the repository's file header (copyright line, SPDX license identifier, project
URL — the exact template is defined in the repo instructions / ReSharper settings). The copyright year range follows
the file's history:

| File | Copyright years |
|---------|--------|
| New file | Current year only (e.g. `Copyright (c) 2026 ...`) |
| Existing file from an earlier year, modified | First year – current year (e.g. `Copyright (c) 2025-2026 ...`) |

### Method Name References

- Method names in **all textual references** (code comments, XMLDoc prose, Markdown documentation, and commit messages)
  always have parentheses: `HandleConnectionFailure()`, not `HandleConnectionFailure`. This makes them instantly
  distinguishable from type names and properties.
- This does **not** apply to machine-readable references like `<see cref="..."/>` or `nameof(...)` — those follow
  compiler syntax.

---

## Commit Messages

> [!IMPORTANT]
> Follow the **Conventional Commits** specification for all commit messages. A commit message must be understandable
> **without inspecting the diff**.

### Format

```text
<type>(<scope>): <subject>

<body>
```

- **Header line is mandatory** (max 72 characters).
- **Body is optional** but required for non-trivial changes (see [Body](#body)).
- Separate header and body with a blank line.

### Commit Types

| Type | Meaning |
|------|---------|
| `feat` | New user-visible feature |
| `fix` | Bug fix |
| `docs` | Documentation only |
| `refactor` | Code restructuring without behavior change |
| `test` | Adding/modifying tests |
| `perf` | Performance improvement |
| `style` | Formatting only |
| `chore` | Tooling, build, CI, dependencies |
| `revert` | Reverts a previous commit (reference the original in the body) |

- Exactly **one type**, reflecting the **primary intent** of the change.
- Avoid `chore` if a more specific type applies.

### Scopes

The scope names the **affected feature or domain area** — stable and domain-oriented, never a file or class name.

**Core:** `api`, `auth`, `core`, `data`, `health`, `openapi`  
**Infrastructure:** `build`, `ci`, `deps`, `docker`, `github`, `tools`  
**Docs:** `architecture`, `deployment`, `development`, `docs`, `features`, `guides`, `roadmap`  
**Features:** `cors`, `errors`, `https`, `logging`, `proxy`, `security`, `system`, `ui`, `validation`

> [!TIP]
> Repositories extend this list with product scopes in their repo instructions. Introduce a new scope only for a new
> major feature area or a clear domain boundary — and add it to the repo instructions before using it.

### Subject

- Type and scope MUST be **lowercase**.
- Subject MUST **start with a capital letter**, use **imperative mood**, and MUST NOT end with punctuation.
- Subject describes **the change**, not the implementation.

```text
✅ fix(auth): Correct token signature calculation
✅ refactor(core): Simplify configuration resolution
❌ fix(auth): fixed token bug               (lowercase, past tense)
❌ feat: added new endpoint                 (no scope, past tense)
❌ refactor(core): TokenSigner.cs cleanup   (file name instead of intent)
```

### Body

Describe **what** changed at a high level and **why** — reasoning and impact, not details the diff already shows.
Wrap at **72 characters**, use full sentences, reference issues or decisions where applicable.

A body is **required** when:

- the change is not self-explanatory,
- behavior differs from previous expectations,
- a workaround or limitation is introduced, or
- the change affects correctness, security, or compatibility.

```text
test(core): Add unit tests for ExceptionHelpers.PrepareForRethrow()

Introduces comprehensive test coverage for the PrepareForRethrow
method, including verification of stack trace preservation and proper
exception unwrapping from AggregateException.
```

### Breaking Changes

A breaking change is anything that may cause existing consumers to fail: removed/renamed endpoints, changed response
structures or status codes, renamed configuration keys, new mandatory settings, schema changes requiring migration.
Mark it with the `!` suffix **and** a `BREAKING CHANGE:` footer that describes the impact and the migration path:

```text
feat(auth)!: Change token response structure

BREAKING CHANGE: Response field "token" renamed to "accessToken".
Clients must update their JSON parsing logic.
```

### Granularity

- One commit = **one logical change**. Do not mix unrelated changes; do not bundle refactorings with behavior changes.
- Messages such as `fix`, `wip`, `stuff`, `minor changes`, or `cleanup` are not acceptable.
