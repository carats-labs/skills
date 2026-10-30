---
name: carats-essentials
description: Use when working in a Carats project (it has @carats/* dependencies or src/client/facets.tsx). Read before any other carats-* skill.
---

Carats is a full-stack framework similar to Next.js that does not use React.

## Core concepts
- It uses JJSX. JJSX gives a pure HTML and JavaScript experience similar to PHP in a modern way; it still looks similar to React.
- JavaScript and TypeScript code can be placed inside JSX/TSX files, similar to React.
- Carats uses Vite for bundling and development, so SASS files can be imported directly from components.
- Carats uses Bun for package management.
- Carats does not use React hooks or React DOM.
- Carats provides server-side and client-side rendering automatically.
- A component is a plain function that returns HTML. Its body runs on the server and in the browser, so anything that touches `document`, reads `window`, or has a side effect must move into `afterMount` or `beforeMount` from `@carats/hooks`. Read `carats-hooks` for state and lifecycle details.
- Frontend routes are defined in `src/client/facets.tsx`. Read `carats-client` for pages, components, facets, and navigation.
- Burnished components fetch data from culets. Culets are defined in `src/server/culets.ts` and registered by importing that file from `src/server/entrypoint.ts`. Read `carats-server` for culets and server-side rendering.
- HTML attributes are preserved as-is, so passing an arrow function to an `onclick` attribute will not work like React. Use `addEventListener` inside `afterMount` or `beforeMount`. Read `carats-hooks`.
- Carats works best with TypeScript and needs a proper `tsconfig.json` at the root.
- Exact package signatures live in [carats-detailed-types](../carats-detailed-types/SKILL.md); the other skills link there instead of copying declarations.

## Global DTOs
Global DTO interfaces live in `src/dto/*.d.ts`. They have no `export` keyword, which is what makes them global, so `culet<User>(...)` and `Burnish<User>(...)` work in any file without an import. Keep them free of imports and runtime code.

`src/dto` is part of the TypeScript program because `include` covers `src`. The template also lists `./src/dto` in `typeRoots`, but that entry is unnecessary: `typeRoots` controls type-package lookup, not project files. `types: ["jjsx"]` is what pulls in the global JJSX declarations. In this setup `typeRoots` can be omitted; if your TypeScript setup needs it for `jjsx`, use `["./node_modules"]`, never `src/dto`.

```typescript
// src/dto/user.d.ts
interface User {
  id: string;
  name: string;
  username: string;
  email: string;
  phone: string;
  website: string;
}
```

## tsconfig.json
```jsonc
{
  "compilerOptions": {
    "target": "es2022",
    "useDefineForClassFields": true,
    "module": "esnext",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["jjsx"], // pulls in JJSX's global JSX declarations
    "skipLibCheck": true,
    "jsx": "react", // required for JJSX
    "jsxFactory": "JJSX.jsxFactory", // required for JJSX
    "jsxFragmentFactory": "JJSX.fragmentFactory", // required for JJSX

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true
  },
  "include": ["src"] // also brings src/dto/*.d.ts into the program
}
```

## package.json scripts
Carats uses Bun and Vite for development and building. A sample `package.json` script block:

```json
{
  "scripts": {
    "dev": "bun --inspect=6499 src/app.ts",
    "build": "bun run build:server && bun run build:client",
    "build:client": "vite build --config vite.config.client.ts",
    "build:server": "vite build --config vite.config.server.ts",
    "build:static": "cross-env NODE_ENV=production carats-ssg",
    "preview": "cross-env NODE_ENV=production bun src/app.ts",
    "test": "vitest --run --coverage"
  }
}
```

## Dependencies
- Runtime packages: `@carats/core`, `@carats/render`, `@carats/hooks`, `@carats/csr`, `@carats/ssr`, `@carats/express`, and `jjsx`.
- The `@carats/express` plugin also needs `express` (v5), `compression`, and `sirv`; `carats()` mounts them for you.
- Dev packages: `vite`, `sass` (or `sass-embedded`), `typescript`, `cross-env`, the `@types/*` packages, and `vitest` plus `@vitest/coverage-v8` if you use the test script.
- `build:static` needs the `@carats/ssg` CLI and `cross-env`.

## Project layout
- The app main file must be `src/app.ts`.
- A page folder exports a default component from `index.tsx` and keeps its stylesheet next to it, named after the folder: `src/client/pages/home/index.tsx` plus `src/client/pages/home/home.sass`.
- Folders prefixed with an underscore are not routes; they are shared building blocks such as `_layout`, `_error`, and `_not_found`.
- There is no file-convention layout. A shared layout is imported explicitly by the pages that need it.
- Define facets before the server entrypoint and culets.

## Reference template
`bun create carats <directory>` runs the published `create-carats` package, which clones [SSR repo](https://github.com/carats-labs/template-ssr) or [CSR only repo](https://github.com/carats-labs/template). That repository is the reference template for these skills; this skills repository does not contain the template. Read it before writing new code and prefer mirroring its structure over inventing a new layout.

You do not need to search the web to implement a Carats app. Everything is covered by `carats-client`, `carats-server`, `carats-hooks`, `carats-styling`, and `carats-detailed-types` skills.
