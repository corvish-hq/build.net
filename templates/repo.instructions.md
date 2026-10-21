---
applyTo: "**"
---

# Repo-specific Copilot Instructions — `__PRODUCT__`

<!-- Repository-specific rules. They extend base.instructions.md and win where they contradict it. -->
<!-- base.instructions.md is managed by the build.net submodule — do not edit it manually. -->
<!-- Hint: Template from build.net/templates/. Copy to .github/instructions/repo.instructions.md once, then own it. -->
<!-- Hint: Replace every __PLACEHOLDER__ and delete sections that do not apply (e.g. no UI, no data layer). -->
<!-- Hint: Finally delete every "Hint:" comment (search for "Hint:"); keep the two comments above. -->

## Placeholder Values

The shared baseline uses placeholders. For this repository:

| Placeholder | Value |
|---|---|
| `<Product>` | `__PRODUCT__` (namespace root `__PRODUCT__.*`) |
| `<UiPrefix>` | `__UI_PREFIX__` (e.g. `__UI_PREFIX__Dialog`) |

<!--
  Hint: `__PRODUCT__` is the namespace root and product name, e.g. `Contoso`. `__UI_PREFIX__` is the short prefix of
  your own UI components, e.g. `Ui` → `UiDialog`. Delete the `<UiPrefix>` row if the repository has no UI.
-->

## Target Framework

> [!IMPORTANT]
> **Target Framework:** `__TARGET_FRAMEWORK__`

<!-- Hint: The .NET version all projects target, e.g. `.NET 10`. -->

## Project Layout

<!--
  Hint: Paths the base rules refer to.
  - Build files: directory holding `Directory.Build.*` and `Directory.Packages.props`, e.g. `src/`.
  - UI primitives: reusable, product-neutral components (buttons, dialogs), e.g. `src/Contoso.Ui.Components/`.
  - UI composites: app-specific components built from primitives, e.g. `src/Contoso.Ui.Shared/`.
  Delete the UI entries if there is no UI. Add further entries if your agents need to know where other code lives.
-->

- **Build files:** `__BUILD_FILES_DIR__`
- **UI primitives:** `__UI_PRIMITIVES_PROJECT__`
- **UI composites:** `__UI_COMPOSITES_PROJECT__`

## Data Layer

<!--
  Hint: Tells the agent where the persistence code lives, so the rules in `data.instructions.md` apply.
  Delete this whole section if the repository has no data layer. Delete single bullets that do not apply.
-->

- **Data project:** `__DATA_PROJECT__`
  <!-- Hint: Path of the project containing the EF Core DbContext, e.g. `src/Contoso.Data/`. -->
- **DbContext:** `__DBCONTEXT__`
  <!-- Hint: Class name of the EF Core DbContext, e.g. `ContosoDbContext`. -->
- **Migrations folder:** `__MIGRATIONS_FOLDER__`
  <!-- Hint: Folder created by `dotnet ef migrations add`, relative to the data project, e.g. `Migrations/`. -->
- **Data service implementations:** `__DATA_SERVICE_IMPLEMENTATIONS__`
  <!--
    Hint: Classes implementing the data service interfaces (`IFooDataService`), one per storage backend, with their
    location, e.g. `SqlServerDataService` in `src/Contoso.Data/`. List several if the app has several backends.
  -->
- **Compiled queries:** `__COMPILED_QUERIES_NAMESPACE__`
  <!--
    Hint: Namespace holding EF Core compiled queries (`EF.CompileAsyncQuery`), e.g. `Contoso.Data.Queries`.
    Delete this bullet if the repository uses none. If a configuration switch toggles between compiled and regular
    queries, name the option here.
  -->
- **Migration phase:** `__MIGRATION_PHASE__`
  <!--
    Hint: Choose one (see "Migrations" in `data.instructions.md`):
    - **Pre-MVP** — the database has not shipped yet; editing existing migrations is allowed. Add the line
      `TODO(MVP): Consolidate migrations` as a reminder to consolidate before the first release.
    - **Post-MVP** — the database has shipped; every schema change needs a new migration.
  -->

## File Header

New and modified source files use this header (year range per the base rules):

<!--
  Hint: `__YEAR__` is the year the file was created, e.g. `2026`. `__COPYRIGHT_HOLDER__` is a person or company.
  `__SPDX_LICENSE__` is an SPDX identifier, e.g. `MIT` (see https://spdx.org/licenses/). Keep the header in sync
  with the ReSharper file-header template, which lives in the repository's own `*.sln.DotSettings`
  (`FileHeaderText` plus the `ProjectURL` variable) — the shared `ReSharper.DotSettings` deliberately has none.
-->

```csharp
// Copyright (c) __YEAR__ __COPYRIGHT_HOLDER__
// SPDX-License-Identifier: __SPDX_LICENSE__
// Project: https://github.com/__GITHUB_OWNER__/__GITHUB_REPO__
```

## Terminology

<!-- Hint: Domain terms that must be used consistently, e.g. "Use **Conversation** (not Chat)." -->

- Use **`__TERM__`** consistently (not `__AVOIDED_TERM__`).

## Commit Scopes

<!--
  Hint: Optional: Conventional-Commit scopes for product areas, in addition to the common scopes in the base
  instructions, e.g. `billing`, `checkout`. Delete the section if the common scopes suffice.
-->

**Product:** `__SCOPE__`, `__SCOPE__`

## Product Principles

<!--
  Hint: Optional: domain rules the agent must respect (product philosophy, threat models, target platforms, ...).
  Write them as short subsections (`### Name`). Delete the section if there are none.
-->

## Colors — Theme Tokens

<!--
  Hint: Optional: delete this section if the repository has no UI.
  - Theme files: where theme tokens are defined, e.g. `src/Contoso.Ui.Components/wwwroot/themes/*/theme.css`.
  - Token families: the token names agents may use, e.g. "Borders → `--border-*`, surfaces → `--bg-*`".
  A verification script that checks used tokens against the theme files is recommended.
-->

The general theme-token rules live in `ui.instructions.md`. `__PRODUCT__`-specific values:

- **Theme files:** `__THEME_FILES__`
- **Token families:** `__TOKEN_FAMILIES__`
