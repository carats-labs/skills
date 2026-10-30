---
name: carats-detailed-types
description: Use when you need exact signatures for @carats/* APIs, or when a type error points into a Carats package. This is the single copy of the exported declarations; other skills link here.
---

These blocks are copied from the published `dist/index.d.ts` of each package, with parameter names normalized for spelling. This file is the only copy of the package types in this bundle; the other skills link here instead of repeating them.

When a `@carats/*` version changes, run `npm pack @carats/<name>` and diff `package/dist/index.d.ts` against the block below. Do not paste declarations into other skills.

## @carats/core
```typescript
interface CaratsRequest<T = any> {
    url: string;
    headers: Record<string, string>;
    cookies: Record<string, string>;
    method: string;
    data: T;
}
declare function findClosest(fileName: string): string | undefined;

export { type CaratsRequest, findClosest };
```

## @carats/csr
```typescript
import { Facets } from '@carats/render';

declare global {
    interface HTMLAnchorElement {
        _isHandled: boolean;
    }
    interface Window {
        carats: {
            ssp: {
                for: string | undefined;
                data: any;
            };
        };
    }
}
declare function clientRender(): Promise<void>;
declare function goTo(url: string): void;
declare function mount(facets: Facets): Facets;

export { clientRender, goTo, mount };
```

## @carats/express
```typescript
import { Router } from 'express';

declare const carats: () => Router;

export { carats };
```

## @carats/hooks
```typescript
type MaybePromise<T> = T | Promise<T>;
type ClearCallback = () => MaybePromise<void>;
type HydrationCallback = () => MaybePromise<ClearCallback | void>;
declare function afterMount(callback: HydrationCallback): void;
declare function clearHydrations(): Promise<void>;
declare function beforeMount(callback: HydrationCallback): void;
type Getter<T> = () => T;
type Factory<T> = (currentValue: T) => T;
type Setter<T> = {
    (fn: Factory<T>): T;
    (value: T): T;
};
type Subscriber<T> = (value: T) => void;
type Subscribe<T> = (subscriber: Subscriber<T>) => () => void;
type State<T> = {
    get: Getter<T>;
    set: Setter<T>;
    subscribe: Subscribe<T>;
};
declare function use<T>(): State<T>;
declare function use<T>(initialState: T): State<T>;

export { type HydrationCallback, type State, afterMount, beforeMount, clearHydrations, use };
```

One spelling normalization: the published `Subscribe` declaration names its parameter `subsriber`; the block above uses `subscriber`. Parameter names do not affect type compatibility.

## @carats/render
```typescript
type CaratsFunction<T> = (this: CaratsComponent, ...args: T[]) => JSX.Element | Promise<JSX.Element>;
interface CaratsComponent<T = any> extends CaratsFunction<T> {
    defaultProps?: T;
    head?: JSX.Element;
    burnished?: boolean;
    recast?: boolean;
    frame?: () => JSX.Element;
    status?: number;
}
type CaratsComponentWithThis<T = any> = ((this: CaratsComponent<T>, props: T) => JSX.Element) & CaratsComponent<T>;
interface Facets {
    inAppRouting?: boolean;
    routes: Record<string, CaratsComponent>;
    suspense: {
        loading: () => JSX.Element;
        error: (error: Error) => JSX.Element;
        notFound: CaratsComponent;
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
    recast?: boolean;
}
declare function Burnish<T = any>(component: CaratsComponentWithThis<T>, options?: BurnishOptions): CaratsComponentWithThis<T>;
declare function Burnish<T = any>(component: CaratsComponent<T>, options?: BurnishOptions): CaratsComponent<T>;
declare function Status<T = any>(status: number): (component: CaratsComponent<T>) => CaratsComponent<T>;
declare function Status<T = any>(status: number): (component: CaratsComponentWithThis<T>) => CaratsComponentWithThis<T>;

export { Burnish, type CaratsComponent, type CaratsComponentWithThis, type Facets, type PageComponentResult, Status, defineFacets, getPageComponent };
```

## @carats/ssg
`@carats/ssg` ships a command-line binary and no runtime exports, so it has no importable surface. It is referenced by the `build:static` script and nowhere in application code.

## @carats/ssr
```typescript
import { CaratsRequest } from '@carats/core';
import { Facets } from '@carats/render';

interface CaratsServerEntry {
    render: (req: CaratsRequest<never>) => Promise<{
        html?: string;
        head?: string;
        status?: number;
    }>;
    getServerProps: <T = any>(req: CaratsRequest<never>) => Promise<T> | T;
    facets: Facets;
    culets: Record<string, Culet>;
}
type CuletArgs = Omit<CaratsRequest, 'data'> & {
    params: Record<string, string>;
};
type Culet<T extends Object = any> = (request: CuletArgs) => T;
declare const seat: <T extends Culet>(f: T) => T;
declare function culet<T extends Object = any>(route: string, culet: Culet<T>): Culet<T>;
declare function defineServerEntry(facets: Facets): CaratsServerEntry;

export { type CaratsServerEntry, type Culet, type CuletArgs, culet, defineServerEntry, seat };
```

## Notes that matter when reading these declarations
- `CaratsComponent.head` is a `JSX.Element`, so a fragment of tags is assigned to it. It is not a string.
- `defineFacets` accepts a `PartialFacets` and returns a fully populated `Facets`. Every input key is optional, and `Facets['suspense']` lists three states while the input allows declaring only a subset of them.
- `CuletArgs` is `CaratsRequest` without `data`, plus `params`. A culet is only ever reached by a GET, so there is no request body and no `req.query`. Parse `req.url` with `new URL(req.url, 'http://localhost').searchParams` for a query string.
- `Culet<T>` returns `T`, not `Promise<T>`. The runtime awaits a returned promise, but `culet<User>(..., async ...)` does not type-check; omit the explicit generic for an async handler.
- A culet's return value is serialized into `window.carats.ssp`, so it must be JSON-serializable. `undefined` and functions are dropped, `Date` values become strings, and circular references or `BigInt` throw.
- If a culet throws, `suspense.error` renders with status 500. If it returns `undefined`, the page receives `undefined` props.
- `seat(f)` is an identity helper: it returns `f` unchanged and does not attach a route. The route is attached by `culet(route, handler)` when the module that calls it is imported.
- `mount`, `clientRender`, and `goTo` are browser only. `goTo` navigates the way an intercepted link click does, so it depends on in-app routing being enabled.
- `clearHydrations()` runs every registered clear callback and empties the registry. `clientRender()` calls it before rendering the next page, so lifecycle cleanup also runs on in-app navigation. It is public for tests and manual teardown.
- `window.carats.ssp.data` is typed as `any` because it is whatever the culet returned. `ssp.for` is the route the payload belongs to, which is how a shared component can check that the data was resolved for the page it is rendered on.
- `window.carats.crown` is not part of the current `@carats/csr` declarations. Older material mentioned it as the serialized `<head>` HTML; do not rely on it. Set page head tags with `this.head` on a component.
- `HTMLAnchorElement._isHandled` is set by the runtime to mark a link it has taken over. Never set it manually.
- `Status(status)` sets the HTTP status for a page component. The same value can be assigned directly to `this.status`.
