---
name: carats-detailed-types
description: Use this skill only if you need to understand low level details about carats framework, this skill will teach you all exported types and interfaces
---

@carats/core:
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

@carats/csr:
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
            crown?: string;
        };
    }
}
declare function clientRender(): Promise<void>;
declare function goTo(url: string): void;
declare function mount(facets: Facets): void;

export { clientRender, goTo, mount };
```

@carats/express:
```typescript
import { Router } from 'express';
declare const carats: () => Router;
export { carats };
```

@carats/hooks:
```typescript
type MaybePromise<T> = T | Promise<T>;
type ClearCallback = () => MaybePromise<void>;
type HydrationCallback = () => MaybePromise<ClearCallback | void>;
declare function afterMount(callback: HydrationCallback): void;
declare function beforeMount(callback: HydrationCallback): void;
declare function clearHydrations(): Promise<void>;
type Getter<T> = () => T;
type Factory<T> = (currentValue: T) => T;
type Setter<T> = {
    (fn: Factory<T>): T;
    (value: T): T;
};
type Subscriber<T> = (value: T) => void;
type Subscribe<T> = (subsriber: Subscriber<T>) => () => void;
type State<T> = {
    get: Getter<T>;
    set: Setter<T>;
    subscribe: Subscribe<T>;
};
declare function use<T>(): State<T>;
declare function use<T>(initialState: T): State<T>;

export { type HydrationCallback, afterMount, beforeMount, clearHydrations, use };

```

@carats/render:
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
    recast?: boolean;
}
declare function Burnish<T = any>(component: CaratsComponentWithThis<T>, options?: BurnishOptions): CaratsComponentWithThis<T>;
declare function Burnish<T = any>(component: CaratsComponent<T>, options?: BurnishOptions): CaratsComponent<T>;

export { Burnish, type CaratsComponent, type CaratsComponentWithThis, type Facets, type PageComponentResult, defineFacets, getPageComponent };
```

@carats/ssg:
```typescript
#!/usr/bin/env bun
```
@carats/ssg ships a command line binary and no runtime exports, so it has no importable surface. It is referenced by the `build:static` script and nowhere in application code.

@carats/ssr:
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
type Culet<T extends Object = any> = (request: CuletArgs) => T;
declare const seat: <T extends Culet>(f: T) => T;
declare function culet<T extends Object = any>(route: string, culet: Culet<T>): Culet<T>;
declare function defineServerEntry(facets: Facets): CaratsServerEntry;

export { type CaratsServerEntry, type Culet, type CuletArgs, culet, defineServerEntry, seat };
```

Notes that matter when reading these declarations:
- `CaratsComponent.head` is a `JSX.Element`, so a fragment of tags is assigned to it. It is not a string.
- `defineFacets` accepts a `PartialFacets` and returns a fully populated `Facets`. Every input key is optional, and `Facets['suspense']` lists three states while the input allows declaring only a subset of them.
- `CuletArgs` is `CaratsRequest` without `data`, plus `params`. A culet is only ever reached by a GET, so there is no request body.
- `mount`, `clientRender` and `goTo` are browser only. `goTo` navigates the way an intercepted link click does, so it depends on in-app routing being enabled.
- `window.carats.ssp.data` is typed as `any` because it is whatever the culet returned. `ssp.for` is the route the payload belongs to, which is how a shared component can check that the data was resolved for the page it is rendered on.
- `HTMLAnchorElement._isHandled` is set by the runtime to mark a link it has taken over. Never set it manually.

