---
name: carats-client
description: Use this skill when coding client side logic, pages or components for carats framework
---

Prerequisites:
- carats-essentials skill
Dependencies:
- "@carats/hooks": to use hooks
- "@carats/render": to define facets
- "@carats/core": to handle core functionality
- "@carats/csr": to handle client side rendering
- "jjsx": to handle jsx (peer dependency)
- "vite": to handle build & resolution (peer dependency)


Client coding is very crucial for carats.
- There must be a proper folder at exact location src/client
- There must be a facets.tsx file in src/client folder
- There must be a entrypoint.ts file in src/client folder
- There must be a index.html file in src/client folder which is compatible with vite html template
- HTML template must contain <!--app-head--> inside head tag and <div id="app"><!--app-html--></div> inside body tag (including html comments)
- HTML template must contain <script type="module" src="/entrypoint.ts"></script> before closing body tag
- HTML template can contain static head tags but try to avoid conflict with components head property. A page component can provide dynamic head tags via this.head assignment (never works with arrow functions)
- Sample component that provides dynamic tags:
```tsx
import { CaratsComponent } from '@carats/render';
export default function Home(this: CaratsComponent) {
    this.head = <>
        <title>Home</title>
        <meta name="description" content="Home page for carats" />
    </>;
    return <div>Home</div>;
}
```
- The `CaratsComponent` type must be imported from `@carats/render`, it is not global.
- There must be a vite-env.d.ts file with exact content: "/// <reference types="vite/client" />". It lives at **src/vite-env.d.ts**, one level above src/client, not inside src/client.
- `this.head` requires a component declared with the `function` keyword, because an arrow function has no own `this`. For a component that is a plain function expression or an arrow and needs the annotation, type it as `CaratsComponentWithThis` instead.

Facets are defined in facets.tsx file by defineFacets function from @carats/render and must be default exported.
Sample facet definition:
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
        loading: () => 'Loading...',
        error: ErrorPage,
        notFound: NotFound
    },
    inAppRouting: true
});
```

routes is a map of routes to carats page components. Each route can have parameters like :id.
suspense is a map of suspense states to carats components. `defineFacets` accepts a **partial** facets object, so `routes` and `suspense` are both optional and every suspense state can be omitted. Declare only what the app actually needs; `loading` is usually unnecessary.
A culet is matched to a page purely by its route string, so the route of the culet and the route in `routes` must be character for character identical, including parameter names.
inAppRouting is a boolean value that indicates whether the app should use in-app routing or not. in app routing provides a navigation similar to SPA. It requires clientRender() in the entrypoint.
after defining facets, you must mount them in src/client/entrypoint.ts file. Sample entrypoint file:
```typescript
import { mount, clientRender } from '@carats/csr';
import facets from './facets.cara';

mount(facets);
clientRender(); // required when in-app routing is enabled
```

Carats components are simplified react components:
- Only pure html attributes are supported. That means you can not pass an arrow function to onclick attribute. or onsubmit or a similar attribute.
- Html attributes considered as strings generally
- Props, .defaultProps, children are supported
- There is no context, no memo and no forwardRef
- A component can import any file Vite understands, which is how SASS is attached to a component
A sample carats component looks like this:
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
Carats components must be .tsx files.
Carats components can import other components and files supported by Vite (like sass, css, json, etc.)
Carats components can be burnished when wrapped by Burnish function from @carats/render.
Burnished means this component needs server side props. And carats framework will automatically fetch it from its culet. (Culets are defined by carats-server skill and they are matched with burnished components if their routes are equal)
Sample burnished component:
```tsx
import { Burnish } from '@carats/render';
export default Burnish<User>((user) => <h1>Hello {user.name}</h1>);
```
Burnished components needs to be defined with function keyword only when this.head assignment is needed. This type is automatically inferred as CaratsComponent for Burnished function components (not the arrow ones).
```tsx
import { Burnish } from '@carats/render';
export default Burnish<User>(function (user){
    this.head = <title>{user.name} - Profile Page</title>;
    return <h1>Hello {user.name}</h1>
});
```
Carats framework automatically caches the recent culet. If a component needs to fetch fresh data on every render, Burnish function accepts a second argument BurnishOptions. Setting `recast` property of options to true will trigger refetch on every render.
Use `goTo` from @carats/csr for programmatic navigation such as submitting a search form. Build a query string with the standard `URLSearchParams`, and read a route's parameters by asking a culet for them: a culet receives the matched parameters in `req.params`.
The props the server already resolved are exposed on `window.carats.ssp`, where `ssp.for` is the route they belong to and `ssp.data` is the value. Read them from a plain component instead of issuing a second request when the data is already there.
A burnished component still accepts its culet's return value as ordinary props, so it renders in a test without a server.
Here is the full @carats/render interface for better understanding:
```typescript
interface CaratsComponent<T = any> extends JSX.FunctionComponent<T> {
    defaultProps?: T;
    head?: JSX.Element;
    burnished?: boolean;
    recast?: boolean;
}
type CaratsComponentWithThis<T = any> = ((this: CaratsComponent<T>, props: T) => JSX.Element) & CaratsComponent<T>;
interface Facets {
    inAppRouting?: boolean;
    routes: Record<string, CaratsComponent>;
    suspense: {
        loading: () => JSX.Element;
        error: (error: Error) => JSX.Element;
        notFound: () => JSX.Element;
    };
}
interface PartialFacets {
    inAppRouting?: boolean;
    routes?: Record<string, CaratsComponent>;
    suspense?: Partial<Facets['suspense']>;
}
declare function defineFacets(facets: PartialFacets): Facets;
interface PageComponentResult {
    component: CaratsComponent<any>;
    params: Record<string, string>;
    route: string;
}
declare function getPageComponent(this: Facets, url: string): PageComponentResult;
interface BurnishOptions {
    recast?: boolean; // prevents props being cached
}
declare function Burnish<T = any>(component: CaratsComponentWithThis<T>, options?: BurnishOptions): CaratsComponentWithThis<T>;
declare function Burnish<T = any>(component: CaratsComponent<T>, options?: BurnishOptions): CaratsComponent<T>;

export { Burnish, type CaratsComponent, type CaratsComponentWithThis, type Facets, type PageComponentResult, defineFacets, getPageComponent };
```
`head` is a JSX.Element, not a string. Assign a fragment of tags to it.
`getPageComponent` resolves a url to a component, its extracted params and the matched route. It is the function the server plugin is built on, and it is enough to work out what a request would render.

Page and folder conventions:
- Each route lives in its own folder under `src\client\pages`, exporting a default component from `index.tsx` next to a stylesheet of the same name.
- A folder prefixed with an underscore is not a route, it is a shared building block. `_layout`, `_error` and `_not_found` are the conventional names.
- There is no file convention layout. A `_layout` component is imported by each page that needs it, and renders `props.children` where the page belongs. Composition is explicit, so a page can opt out.
- `suspense.error` receives the thrown `Error`, so it can read `message` and `stack`. `suspense.notFound` receives nothing. Wrap both in the shared layout so a failure still renders the site chrome.

Facets are so crucial, they will be used on both client and server entrypoints.
There must be a vite.config.client.ts at root, sample vite config for client:
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
  }
})
```

- You can use version "latest" for all dependencies if no version is specified (e.g. you cant use latest if express 5 is required)