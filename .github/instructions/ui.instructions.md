---
applyTo: "**/*.razor,**/*.razor.cs,**/*.css,**/*.xaml,**/*.xaml.cs"
---

# UI Rules

<!-- Shared baseline, owned by the build.net repository (https://github.com/corvish-hq/build.net). -->
<!-- Consumer repos receive a copy on every build — never edit the copy, change build.net instead. -->

> [!NOTE]
> UI rules for every UI technology (Blazor, MAUI/XAML, web front-ends).
>
> Placeholders (`<Product>`, `<UiPrefix>`) and general rules are explained in
> [base.instructions.md](base.instructions.md); the concrete placeholder values come from the repo instructions,
> which also win where they contradict this file.

## UI Discipline

> [!IMPORTANT]
> These rules keep the UI **update-safe** and maintainable: against third-party component libraries (whether one is
> in use today or introduced later) and against our own components drifting apart through outside overrides. UI
> primitives should be project-owned (`<UiPrefix>*` components) wherever possible.

- **Never style or manipulate third-party internals.** Reaching into the private layout tree, undocumented style
  classes/keys, or control templates of an external component library is **forbidden** — it breaks silently on every
  library update. See the styling order below for what is allowed.
- **Styling third-party components, in this order:** (1) component parameters, (2) documented CSS custom
  properties / theme resources, (3) a class or style key the library's documentation explicitly names as a styling
  hook — with a link to that documentation in a comment. If none of these covers the requirement, **stop** and
  propose a `<UiPrefix>*` wrapper component that solves the problem on an element we own — never escalate against
  internals.
- **Never override code we own from the outside.** If an own component lacks a variant (surface, spacing, …), add
  a parameter or modifier class to that component — never force it from a consumer with `!important`, deep
  selectors, or higher specificity. The same applies to own style classes: use a modifier class instead of
  overriding the base rule.
- **No override escalation without a justifying comment.** Every remaining specificity/priority override (e.g. CSS
  `!important`) must have a comment on the same line or directly above explaining why the normal mechanism is
  insufficient. "To override a third-party library" or "to override our own component" is never a valid reason —
  see the rules above.
- **Blazor specifics (scoped CSS, `*.razor.css`):**
  - **Allowed:** `::deep` on markup that has no `.razor` file of its own and therefore no scope attribute — e.g.
    HTML rendered from Markdown, a `MarkupString`, or JS-generated elements.
  - **Allowed:** `::deep` on a documented styling hook of a third-party component (step 3 above), without
    `!important`.
  - **Allowed:** `::deep .own-class` where `.own-class` is a class the consumer itself passed to an own child via
    its `Class` parameter — Blazor does not stamp the parent's scope attribute on a child's root, so this is the
    only way to reach it. Use it for **placement only** (margin, flex/grid position, width), never for the
    child's look.
  - **Forbidden:** `::deep` on the internal classes of own child components (e.g. `.ui-btn`, `.ui-avatar-img`).
    Appearance and variants (size, surface, spacing, trigger style) belong to the child: add a parameter and let
    the child style itself (see "Never override code we own from the outside").
  - **Forbidden:** `::deep` on internal layout/composition classes of any third-party library.
- **Component naming:** Own UI components are prefixed with `<UiPrefix>` (e.g., `<UiPrefix>StretchTextArea`,
  `<UiPrefix>IconButton`, `<UiPrefix>Dialog`). Reusable primitives live in a shared component library project;
  composites live in a shared composite project (when several apps use them) or in the app project. The repo
  instructions name the concrete projects. Together they form a stable abstraction layer so a component library
  can be plugged in without touching consumers.
- **TODO markers for existing overrides:** Overrides of third-party internals (e.g. a `::deep` + `!important` block)
  are hacks. If one already exists and cannot be removed immediately, or the user explicitly approves one as an
  interim step, mark it with `TODO(<UiPrefix>ComponentName): ...` describing what will replace it (see
  [Architectural Integrity](base.instructions.md#architectural-integrity-binding-agreement)). Overrides of **own**
  components are never tolerated with a TODO — fix the component directly.

> [!CAUTION]
> **Know the rendered elements before styling.** Before writing or changing any style rule or layout fix, determine
> which elements are actually rendered (HTML DOM, XAML visual tree, …):
>
> 1. Read the component's own markup — which elements does it render itself, which come from child components?
> 2. Read those child components' markup to see which root element each one renders.
> 3. Only then choose the selector. Prefer layout on the container (`display: flex; gap`, `display: grid`, layout
>    panels) over sibling selectors and deep selector chains.
>
> Guessed styling fixes ("this should work") are a defect. If the rendered output cannot be determined from source
> (e.g. dynamic third-party output), say so and ask the user for it before proposing a selector.
>
> **Blazor:** only elements written in the component's own `.razor` file receive its scoped `b-xyz` attribute — the
> root elements of child components do **not**, so scoped CSS does not reach them.

## Localization

- **No hardcoded user-facing text.** Every visible string comes from the localization resources — including
  accessible names (`aria-label`, `AutomationProperties.Name`), tooltips (`title`), placeholders, and
  validation/empty-state messages. Brand names and technical identifiers are the only exceptions.
- **Missing translations are errors, not fallbacks.** Reusable primitives that do not localize themselves take their
  texts as parameters and have **no** built-in default text. Always-needed texts are `[EditorRequired]` (a missing
  one fails the build via RZ2012) and guarded at runtime in `OnParametersSet()`; texts needed only together with an
  optional feature are bundled into a parameter object with `required` members (e.g. a discard prompt), so the
  feature cannot be enabled without them. Only the localizer itself marks a missing **resource key** visibly (e.g.
  `?? key ??`), because that gap cannot be caught at compile time.

## Accessibility

- **Visual casing only via styles.** Text that should appear in all caps (wordmarks, labels) is written in normal
  casing in the source and transformed visually (CSS `text-transform`, …). Screen readers may spell out all-caps
  strings letter by letter.
- **Every interactive element has an accessible name.** Icon-only buttons and controls need a label (e.g.
  `aria-label`, `AutomationProperties.Name`) — localized like any other user-facing text.
- **Never convey meaning by color alone.** Status, errors, and selection also need text, an icon, or a shape.
- **Decorative content is hidden from assistive technology** (e.g. `aria-hidden="true"` on purely decorative icons);
  meaningful images get a text alternative.

## Colors — Theme Tokens Only

> [!IMPORTANT]
> Every color value in component styles and theme files **must** come from a theme-defined design token (CSS custom
> property, XAML theme resource, …). The rules below use CSS syntax; apply them analogously to other technologies
> (e.g. `{DynamicResource ...}` instead of `var(--...)`). This is non-negotiable — hardcoded colors break
> theme-awareness (e.g. a `rgba(255,255,255,0.08)` divider is invisible on light themes). The repo instructions list
> the concrete token families and the theme files that define them.

### The Base Theme Contract

- **The base theme declares every token.** Every token any component or theme uses is declared in the base theme —
  derived themes only **override** existing tokens and never introduce new ones.
- **Theme-owned palette tokens get a sentinel, never a functional default.** Surface/text/border/accent colors and
  direction-dependent hover overlays differ per theme, so the base theme sets them to a loud sentinel color (e.g.
  magenta `#ff00ff`) instead of a dark- or light-biased literal. An incomplete theme then screams on screen instead
  of silently borrowing a wrong color. New palette tokens go into the base theme as the sentinel; then every
  official theme gets an explicit value.
- **Invariants keep real defaults — and only invariants.** Layout metrics, transitions, semantic status/error/
  warning/success/info colors, fixed values such as a modal scrim, and derived tokens that reference a root token
  stay functional in the base theme. Everything whose right value depends on the theme (light/dark surfaces, the
  accent's lightness) is palette and gets the sentinel — including syntax colors, shadows, and text on accent. The
  only exception are non-color values that cannot show the sentinel (e.g. an icon `filter`): they get a neutral
  no-op default.
- **Seeing the sentinel color in the running UI is a bug**: a theme is missing an override. Fix the theme, never
  suppress the sentinel.

### Using Tokens

- **Never invent tokens.** Before using `var(--foo)`, confirm `--foo` is declared in the base theme. An undefined
  custom property silently falls back to its initial value (often transparent or inherited) — nothing warns you.
  If a token is missing, **add it to the base theme**.
- **Never use hardcoded color fallbacks.** `var(--x, #fff)` / `var(--x, rgba(...))` is forbidden — the fallback
  masks a missing token and is not theme-aware.
- **Comment non-obvious token choices only.** A token whose name matches its use (`--text-muted` for muted text)
  needs no comment. Comment when the choice is not self-explanatory — e.g. a border token used as a hover surface,
  or a status token used outside its semantic meaning.
- **Genuinely theme-independent literals are the only exception.** Mark them with a comment starting with
  `theme-independent:` that explains why (e.g. a startup error page rendered before theme CSS loads, or a viewer
  surface that is always dark by design). If the same fixed value is needed in more than one place (e.g. the dark
  scrim behind every modal), declare it once in the base theme as an invariant token (e.g. `--scrim`) instead of
  repeating the literal.
- **Verify before finishing.** A task that adds or edits CSS is only done after every used `var(--…)` has been
  checked against the tokens declared in the base theme and no unmarked hardcoded color remains. If the repo
  instructions provide a verification script, run it.
