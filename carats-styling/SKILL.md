---
name: carats-styling
description: Use when styling a Carats frontend and the user has not specified a ruleset. Default rules; indented Sass, scoped selectors, semantic HTML, and design tokens.
---

Default to indented `.sass` syntax, not `.scss`. If the existing project already standardizes on `.scss` modules, follow that convention. Install `sass` (or `sass-embedded`) as a dev dependency; Vite compiles it.

Prefer semantic HTML and as few classes as possible.

## What "scoped Sass" means
Sass does not scope selectors by itself. In Carats, scoped Sass is a convention:

- Give each component one root element or class (`#profile`, `.home`, `.trade-page`).
- Nest every rule for that component under that root.
- Do not style bare element selectors from a component stylesheet; global element styles belong in `base.sass`.

```sass
.home
  padding: 4rem 0

  .title
    font-size: $font-size-base
```

Everything stays under `.home`, so the stylesheet cannot leak into another page.

## File layout
- `src/client/_tokens.sass` holds design tokens only: top-level variables, no selectors, no CSS output. Every component stylesheet `@use`s it.
- `src/client/base.sass` holds resets and global element styles. Import it exactly once from `src/client/entrypoint.ts`.
- Do not `@use` `base.sass` from component stylesheets. Vite compiles each Sass entry separately, so any real CSS in a shared `@use`d file is emitted into every component's output.

```typescript
// src/client/entrypoint.ts
import './base.sass';
```

## Tokens
Colors, spacing, typography, breakpoints, and other design tokens belong in `_tokens.sass`.

Tokens must be top-level. `@use ... as *` exposes a module's top-level members; a variable declared inside a selector is local to that block and is not visible through `as *`.

```sass
// src/client/_tokens.sass
$color-primary: #000
$spacing-xs: 4px
$font-size-base: 16px
$breakpoint-md: 768px
```

## Component stylesheet
Every component stylesheet starts by loading the tokens:

```sass
// src/client/pages/home/home.sass
@use '../../tokens' as *

.home
  padding: 4rem 0
  color: $color-primary

  .title
    font-size: $font-size-base
```

Sass resolves `_tokens.sass` from `@use '../../tokens'`; the underscore is optional in the import path.

The reference template currently uses this line in component stylesheets (page stylesheets use `@use '../../base' as *`):

```sass
@use '../base' as *
```

That works for variables but emits the base CSS into every component's output. Treat it as the existing pattern, not the target: put tokens in `_tokens.sass`, import `base.sass` once from the entrypoint, and use `@use '../../tokens' as *` (shown above) in components.
