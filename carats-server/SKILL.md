---
name: carats-server
description: Use when coding server-side logic in a Carats project; culets, the server entrypoint, and server-side rendering props.
---

Read `carats-essentials` first.

Dependencies:
- `@carats/ssr` for server-side rendering and culets
- `@carats/core` for core logic (peer dependency)
- `jjsx` for the JSX runtime (peer dependency)
- `vite` for the build tool (peer dependency)
- a backend plugin such as `@carats/express`, plus its peer dependencies (`express`, `sirv`, `compression`)

## Setup
- Carats can be integrated into any project, but it needs the proper folder structure.
- Server folder: `src/server`.
- Client folder: `src/client`.
- Server entrypoint: `src/server/entrypoint.ts`.
- Facets file: `src/client/facets.tsx`.
- Culets: `src/server/culets.ts`, imported by the server entrypoint for its side effects.
- `vite.config.server.ts` at the root. Sample server Vite config:

```typescript
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
  publicDir: false,
  build: {
    ssr: path.resolve(import.meta.dirname, 'src/server/entrypoint.ts'),
    outDir: path.resolve(import.meta.dirname, 'dist/server'),
    emptyOutDir: true,
    minify: true,
    rollupOptions: {
      treeshake: true
    }
  }
})
```

Use `import.meta.dirname` in both Vite configs; the project is ESM and runs on Bun.

## Entrypoint
The entrypoint default-exports the object returned by `defineServerEntry` from `@carats/ssr`. `defineServerEntry` receives the application facets.

`facets` is the same object the client uses. You do not need to learn how to create facets to work on the server; import them from `../client/facets`. If the frontend routes need to change, read `carats-client`.

```typescript
import { defineServerEntry } from '@carats/ssr';
import facets from '../client/facets';
import './culets'; // registers every culet() call in that file

export default defineServerEntry(facets);
```

The entrypoint may also contain culets inline. An inline culet registers itself and needs no import.

## Culets
Culets are server-side functions that fetch data for burnished pages. Define one only when a server-side prop is needed for a facet route and it is feasible to retrieve with a single GET request.

- The culet route string must be character-for-character identical to the facet route it serves, including parameter names. There is no implicit matching by file name or component name.
- A mismatch fails silently: the page renders without the culet's return value, so the props it expects are missing and the error surfaces inside the component.
- A culet declared inline in the entrypoint registers itself.
- A culet defined in `culets.ts` registers when that module runs `culet(route, handler)`. Importing the module from the entrypoint is what registers it.
- `seat(f)` is an identity helper: it returns `f` unchanged and does not attach a route. It is kept for API compatibility; registering a culet still requires `culet(route, handler)`.

Complete `culets.ts` example:

```typescript
// src/server/culets.ts
import { culet } from '@carats/ssr';

async function getProfile(id: string): Promise<User> {
    return {
        id,
        name: 'Alexander Whitmore',
        username: 'awhitmore',
        email: 'a.whitmore@vault.io',
        phone: '+1 (212) 555-0193',
        website: 'whitmore.capital',
    };
}

culet('/profile/:id', (req) => getProfile(req.params.id));
```

`User` is a global DTO declared in `src/dto/user.d.ts`, so no import is needed. See `carats-essentials` for the DTO convention.

Reading route parameters:

```typescript
culet<{ id: string }>('/users/:id', (req) => {
    return { id: req.params.id };
});
```

`CuletArgs` contains `url`, `headers`, `cookies`, `method`, and `params`. It does not contain `query` and it omits `data`, because a culet is only ever reached by a GET. For a query string, parse `req.url`:

```typescript
culet<{ q: string }>('/search', (req) => {
    const url = new URL(req.url, 'http://localhost');
    return { q: url.searchParams.get('q') ?? '' };
});
```

Error and empty behavior:
- If a culet throws or rejects, rendering fails and the server renders `suspense.error` with status 500. A `/culet/...` request also responds with 500.
- If a culet returns `undefined`, the page receives `undefined` as props. There is no fallback; the component must tolerate it or the render fails.
- A culet return value is serialized into `window.carats.ssp`, so it must be JSON-serializable. Return plain objects and arrays. `Date` values become strings, `undefined` and functions are dropped, and `BigInt` or circular references throw.
- A culet handler can be async; the runtime awaits the returned promise. The published `Culet<T>` type returns `T`, not `Promise<T>`, so `culet<User>(..., async ...)` fails type checking. Omit the explicit generic for an async handler, as in the example above.

The exact `@carats/ssr` signatures are documented once in [carats-detailed-types](../carats-detailed-types/SKILL.md).

## Server entry behavior
The object returned by `defineServerEntry` is what the server plugin drives:
- `render(req)` resolves the facet for the requested URL, runs the page component, and returns the rendered `html` together with the collected `head` tags.
- `getServerProps(req)` resolves the culet for the requested URL. If the route has no culet, it returns the route params merged with the component's `defaultProps`; if a culet returns `undefined`, the result is `undefined`.
- `facets` is the shared facets object.
- `culets` is the registry that `culet` populates. `seat` does not add to it.

## Express integration
Server plugins integrate Carats with an existing backend framework. A plugin exports a named `carats` function that returns an Express `Router`, so mount it with `app.use`.

Order the middleware so `carats()` runs after the routes and static handlers it should not swallow.

A minimal Express server (`src/app.ts`) also needs the `@types/node` and `@types/express` dev dependencies:

```typescript
import { carats } from '@carats/express'
import express from 'express'

const app = express()
const port = process.env.PORT || 5173

app.use(carats())

const server = app.listen(port, () => {
  console.log(`Server started at http://localhost:${port}`)
})

export default server
```

You do not import `sirv` or `compression` yourself. Inside `carats()`, `@carats/express` adds compression and serves the built client assets with `sirv` in production, and uses Vite middleware in development. They are peer dependencies of the plugin.

`@carats/express` requires Express 5, so the major version is not optional.

`@carats/ssg` prerenders the app to static HTML and is used through the `build:static` script (`cross-env NODE_ENV=production carats-ssg`). It ships a command-line binary and no runtime exports, so it is never imported in application code.
