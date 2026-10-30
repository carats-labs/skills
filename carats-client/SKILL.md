---
name: carats-client
description: Use when coding client-side pages, components, facets, or navigation in a Carats project.
---

Read `carats-essentials` first.

Dependencies:
- `@carats/hooks` for state and lifecycle callbacks
- `@carats/render` for facets and `Burnish`
- `@carats/core` for core functionality
- `@carats/csr` for client-side rendering and navigation
- `jjsx` for JSX (peer dependency)
- `vite` for build and resolution (peer dependency)

## Project files
- There must be a proper folder at the exact location `src/client`.
- There must be a `facets.tsx` file in the `src/client` folder.
- There must be an `entrypoint.ts` file in the `src/client` folder.
- There must be an `index.html` file in the `src/client` folder, compatible with the Vite HTML template.
- The HTML template must contain `<!--app-head-->` inside the `<head>` tag and `<div id="app"><!--app-html--></div>` inside the `<body>` tag (including the HTML comments).
- If any component sets `this.head`, the template must wrap `<!--app-head-->` with `<meta id="carats-crown-start" />` before it and `<meta id="carats-crown-end" />` after it. `clientRender` replaces the content between those markers on navigation.
- The HTML template must contain `<script type="module" src="/entrypoint.ts"></script>` before the closing `</body>` tag.
- The HTML template can contain static head tags, but try to avoid conflicts with a component's `head` property. A page component can provide dynamic head tags by assigning to `this.head` (this never works with arrow functions).
- There must be a `vite-env.d.ts` file with the exact content `/// <reference types="vite/client" />`. It lives at `src/vite-env.d.ts`, one level above `src/client`, not inside `src/client`.

Sample component that provides dynamic head tags:

```tsx
import { CaratsComponent } from '@carats/render';

export default function Home(this: CaratsComponent) {
    this.head = <>
        <title>Home</title>
        <meta name="description" content="Home page for Carats" />
    </>;
    return <div>Home</div>;
}
```

- The `CaratsComponent` type must be imported from `@carats/render`; it is not global.
- `this.head` requires a component declared with the `function` keyword, because an arrow function has no own `this`. An arrow function cannot set `this.head`; use a `function` component when that assignment is needed.
- `CaratsComponentWithThis<T>` describes a function component that uses `this` and receives props. `Burnish` infers it, so you rarely name it yourself.

## Facets
Facets are defined in `facets.tsx` with `defineFacets` from `@carats/render` and must be default exported.

```typescript
import { defineFacets } from '@carats/render';
import ErrorPage from './pages/_error';
import NotFound from './pages/_not_found';
import Home from './pages/home';
import Profile from './pages/profile';

export default defineFacets({
    routes: {
        '/': Home,
        '/profile/:id': Profile
    },
    suspense: {
        error: ErrorPage,
        notFound: NotFound
    },
    inAppRouting: true
});
```

- `routes` is a map of route strings to Carats page components. A route can have parameters such as `:id`.
- `suspense` is a map of suspense states to Carats components. `defineFacets` accepts a **partial** facets object, so `routes` and `suspense` are both optional and every suspense state can be omitted. Declare only what the app actually needs; `loading` is usually unnecessary and is omitted above.
- A culet is matched to a page purely by its route string, so the culet route and the route in `routes` must be character-for-character identical, including parameter names. The page above uses `/profile/:id`, so its culet must also use `/profile/:id`. See `carats-server`.
- `inAppRouting` indicates whether the app should use in-app routing. In-app routing provides SPA-like navigation and requires `clientRender()` in the entrypoint.

After defining facets, mount them in `src/client/entrypoint.ts`:

```typescript
import { mount, clientRender } from '@carats/csr';
import './base.sass';
import facets from './facets';

mount(facets);
clientRender(); // renders the first page; also enables in-app routing when configured
```

Import the global stylesheet (`base.sass`) exactly once here. Component stylesheets should only import tokens; see `carats-styling`.

## Components
Carats components are simplified React components:
- Only pure HTML attributes are supported. You cannot pass an arrow function to `onclick`, `onsubmit`, or a similar attribute.
- HTML attributes are generally treated as strings.
- Props, `defaultProps`, and `children` are supported.
- There is no context, no `memo`, and no `forwardRef`.
- A component can import any file Vite understands, which is how SASS is attached to a component.
- Event handling and other browser side effects belong in `afterMount` or `beforeMount`; read `carats-hooks`.

A sample Carats component:

```tsx
import './_layout.sass';

export default function Layout(props: JSX.ComponentProps) {
    return (
        <>
            <header>Hello World</header>
            <main>{props.children}</main>
            <footer>Carats</footer>
        </>
    )
}
```

- Carats components must be `.tsx` files.
- Carats components can import other components and files supported by Vite (SASS, CSS, JSON, and so on).

## Burnished components
A Carats component can be burnished by wrapping it with `Burnish` from `@carats/render`. A burnished component needs server-side props, and Carats automatically fetches them from its culet. Culets are covered in `carats-server`; they are matched to burnished components by equal route strings.

```tsx
import { Burnish } from '@carats/render';

export default Burnish<User>((user) => <h1>Hello {user.name}</h1>);
```

`User` is a global DTO declared in `src/dto/user.d.ts`, so no import is needed. See `carats-essentials` for the DTO convention.

Burnished components need the `function` keyword only when a `this.head` assignment is needed. `Burnish` gives a function component a `this` of type `CaratsComponent`, so no annotation is needed; arrow functions still have no `this`.

```tsx
import { Burnish } from '@carats/render';

export default Burnish<User>(function (user) {
    this.head = <title>{user.name} - Profile Page</title>;
    return <h1>Hello {user.name}</h1>
});
```

- Carats automatically caches the most recent culet result. If a component needs fresh data on every render, pass `BurnishOptions` as the second argument and set `recast` to `true`.
- A burnished component still accepts its culet's return value as ordinary props, so it can render in a test without a server.
- The props the server already resolved are exposed on `window.carats.ssp`, where `ssp.for` is the route they belong to and `ssp.data` is the value. Read them from a plain component instead of issuing a second request when the data is already there.
- The exact `@carats/render` signatures are documented once in [carats-detailed-types](../carats-detailed-types/SKILL.md).

## Navigation
Use `goTo` from `@carats/csr` for programmatic navigation, such as submitting a search form:

```typescript
import { goTo } from '@carats/csr';

function openSearch(query: string) {
    const params = new URLSearchParams({ q: query });
    goTo(`/search?${params}`);
}
```

`goTo` depends on in-app routing being enabled and on `clientRender()` running in the entrypoint. Call it from a real event handler registered in `afterMount` or `beforeMount`, not from an `onclick` attribute.

Route parameters are matched by the facet route. When a page is burnished, the same parameters reach its culet in `req.params`, and the culet's return value becomes the page props. Query strings are not exposed as `req.query`; parse `req.url` with `new URL(req.url, 'http://localhost').searchParams` inside the culet.

`getPageComponent` resolves a URL to a component, its extracted params, and the matched route. The server entrypoint uses it, and it is enough to work out what a request would render.

## Page and folder conventions
Each route lives in its own folder under `src/client/pages`, exporting a default component from `index.tsx` next to a stylesheet named after the folder:

```text
src/client/pages/
  home/
    index.tsx      <- default export of the Home component
    home.sass      <- stylesheet named after the folder, not "index.sass"
  profile/
    index.tsx
    profile.sass
  _layout/
    index.tsx
    _layout.sass
```

- A folder prefixed with an underscore is not a route; it is a shared building block. `_layout`, `_error`, and `_not_found` are the conventional names.
- There is no file-convention layout. A `_layout` component is imported by each page that needs it and renders `props.children` where the page belongs. Composition is explicit, so a page can opt out.
- `suspense.error` receives the thrown `Error`, so it can read `message` and `stack`. `suspense.notFound` receives nothing. Wrap both in the shared layout so a failure still renders the site chrome.

## Vite config
There must be a `vite.config.client.ts` at the root. Sample client Vite config:

```typescript
import { defineConfig } from 'vite'
import path from 'path'

export default defineConfig({
  publicDir: path.resolve(import.meta.dirname, 'public'),
  root: path.resolve(import.meta.dirname, 'src/client'),
  base: '/',
  appType: 'custom',
  server: {
    middlewareMode: true
  },
  build: {
    outDir: path.resolve(import.meta.dirname, 'dist/client'),
    emptyOutDir: true,
    manifest: true,
    minify: true,
    rollupOptions: {
      treeshake: true
    }
  },
  css: {
    modules: {
      localsConvention: 'dashes'
    }
  }
})
```

The same facets object is used by both the client and server entrypoints.

Pin dependency versions that are current when you install rather than using `latest`; a major release can break the framework. `@carats/express` requires Express 5, which is the current major version on npm.
