---
name: carats-server
description: Use this skill when coding backend side logic for carats framework
---

Prerequisites:
- Carats framework essentials
- Server side rendering knowledge
- Node.js knowledge
- Bun package manager knowledge

Dependencies:
- "@carats/ssr": to handle server side rendering
- "@carats/core": core logic (peer dependency)
- "jjsx": jsx runtime (peer dependency)
- "vite": build tool (peer dependency)
- related backend plugin (e.g. @carats/express) and its peer dependencies (e.g. express, sirv, compression)

Setup:
- Can be integrated to any project but needs proper folder structure
- Needs proper server folder at exact location: src\server
- Needs proper client folder at exact location: src\client
- Needs server entrypoint file at exact location: src\server\entrypoint.ts
- Needs proper facets file at exact location: src\client\facets.tsx
- Culets live at the exact location src\server\culets.ts, imported by the server entrypoint for their side effects.
- Needs vite.config.server.ts at root, sample vite config for server:
```typescript
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
  publicDir: false,
  build: {
    ssr: path.resolve(__dirname, 'src/server/entrypoint.ts'),
    outDir: path.resolve(__dirname, 'dist/server'),
    emptyOutDir: true,
    minify: true,
    rollupOptions: {
      treeshake: true
    }
  }
})
```

- The entrypoint file is crucial for server side rendering.
- The entrypoint file should export a default function that is defined using `defineServerEntry` from `@carats/ssr`.
- `defineServerEntry` function expects to receive `facets` as parameter.
- `facets` is a file that exports all the facets of the application. Server coder doesnt have to learn how to create facets, just import them '../client/facets.cara';
- If facets (the frontend routes) needs to be changed, learn carats-client skill.
- The entrypoint file may contain culets (server side functions) that are used to fetch data from the database or other sources.
- Culets are defined using `culet` function from `@carats/ssr`.
- You should only define culets if a server side prop is needed for a facet route & if its feasible to retrieve by a single GET request.
- The route string of a culet must be character for character identical to the route of the facet it serves, including parameter names. There is no implicit matching by file name or component name. A mismatch fails silently: the page renders with `undefined` props and the error surfaces as a crash inside the component.
- A culet declared inline in the entrypoint registers itself and needs no `seat` call.
- If a culet is defined in another file, it must be seated by `seat` function from `@carats/ssr` in the entrypoint file. `seat` returns the culet unchanged, so it is a pure registration step. Do NOT also wrap a seated culet in another `culet()` call, because that registers the same handler under a second route. A minimal sample server entrypoint file:
```typescript
import { culet, defineServerEntry, seat } from '@carats/ssr';
import facets from '../client/facets.cara';
import getTradeData from './culets';

seat(getTradeData)
culet<User>('/profile', () => {
  return {
    id: '1',
    name: 'Alexander Whitmore',
    username: 'awhitmore',
    email: 'a.whitmore@vault.io',
    phone: '+1 (212) 555-0193',
    website: 'whitmore.capital',
  };
});

export default defineServerEntry(facets)
```
- The @carats/ssr type definitions for better understanding:
```typescript
import { CaratsRequest } from '@carats/core';
import { Facets } from '@carats/render';

interface CaratsServerEntry {
    render: (req: CaratsRequest<never>) => Promise<{
        html?: string;
        head?: string;
    }>;
    getServerProps: <T = any>(req: CaratsRequest<never>) => Promise<T> | T;
    facets: Facets;
    culets: Record<string, Culet>;
}
type CuletArgs = Omit<CaratsRequest, 'data'> & {
    params: Record<string, string>;
};
type Culet<T = any> = (request: CuletArgs) => T;
declare const seat: <T extends Culet>(f: T) => T;
declare function culet<T = any>(route: string, culet: Culet<T>): Culet<T>;
declare function defineServerEntry(facets: Facets): CaratsServerEntry;

export { type CaratsServerEntry, type Culet, type CuletArgs, culet, defineServerEntry, seat };

```

- server entrypoint's default export is crucial to make server plugins like @carats/express work.
- The object returned by `defineServerEntry` is what the server plugin drives. `render(req)` resolves the facet for the requested url, runs the page component and returns the rendered `html` together with the collected `head` tags. `getServerProps(req)` resolves the culet for the requested url and returns its value, or `undefined` when the route has no culet. `facets` is the shared facets object and `culets` is the registry built by `culet` and `seat`.
- server plugins are used to integrate carats with existing backend frameworks like express.
- server plugins export named function `carats` which can be used as an app middleware when invoked. It returns an express `Router`, so mount it with `app.use`.
- server plugins are needed to make carats work with existing backend frameworks.
- order the middleware so that `carats()` runs after the routes and static handlers it should not swallow.
- a minimal express server example: (app.ts in /src directory) which requires @types/node and @types/express peer dependencies too:
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
- note that @carats/express works with express v5, so the major version is not optional. A culet may also read `req.cookies`, `req.headers`, `req.query`, `req.url` and `req.method`, but it is only ever reached by a GET, which is why `CuletArgs` omits `data`.
- a culet handler can be async; awaiting inside it is the normal way to talk to a database or a remote API. Returning a promise is enough, there is no need to await it at the call site.
- @carats/ssg prerenders the app to static HTML and is used through the `build:static` script (`cross-env NODE_ENV=production carats-ssg`). It ships a command line binary and no runtime exports, so it is never imported in application code.