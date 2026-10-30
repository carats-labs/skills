---
name: carats-hooks
description: Use this skill when coding components that need state management or a client-side lifecycle callback
---

Dependency: @carats/hooks

Carats exports named functions for state management and the client-side lifecycle via @carats/hooks.

A component function runs on both the server and the client, so a hook body must never touch the DOM or the network. When a side effect is needed, put it inside a lifecycle callback: **afterMount** or **beforeMount**. Both run only in the browser and both may return a clear callback for cleanup.

The two differ only in when they run:

- **afterMount** runs after the `load` event has been dispatched, so the document is complete. Use it to reach DOM elements the same component created.
- **beforeMount** runs immediately, without waiting for `load`. Use it for work that does not need the component's own markup, such as timers or key listeners.

Prefer **afterMount** for anything that reads the DOM, because waiting for `load` guarantees the markup exists. Reach for **beforeMount** when the work is independent of the markup and should start sooner.

Keep a reference to the exact handler that was registered. Passing a fresh arrow function to `removeEventListener` creates a new function identity, so the listener is never actually removed.
Call `subscribe` from a lifecycle callback and return its unsubscribe function as the clear callback, otherwise the subscriber stays registered after the page is replaced.

For state, use `use`. It is not reactive like a React hook: setting a value re-renders nothing, so nothing on the page updates by itself. Read the value with `get()` while rendering, write the DOM from a `subscribe` callback, and change the value from anywhere. Keep the handler that changes state separate from the subscriber that writes the DOM, or the element ends up with two sources of truth.
`set` returns the new value, so the result is available immediately when a caller needs it.
There is no `useEffect` and no dependency array. A callback runs on every render, so it must never touch the DOM or the network directly.
`use` is also callable outside a component, which is the idiomatic way to share one piece of state between pages.

Consider below interface:

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
type Subscribe<T> = (subscriber: Subscriber<T>) => () => void;
type State<T> = {
  get: Getter<T>;
  set: Setter<T>;
  subscribe: Subscribe<T>;
};
declare function use<T>(): State<T>;
declare function use<T>(initialState: T): State<T>;

export { type HydrationCallback, afterMount, beforeMount, use };
```

A working state example, where the subscriber callback is what replaces a React re-render:

```tsx
import { use, afterMount } from '@carats/hooks';

export default function Counter() {
    const count = use(0);

    afterMount(() => {
        const inc = document.getElementById('inc');
        const label = document.getElementById('count');

        const unsub = count.subscribe((value) => {
            if (label) label.textContent = String(value);
        });

        const increment = () => count.set((n) => n + 1);
        inc?.addEventListener('click', increment);

        return () => {
            unsub();
            inc?.removeEventListener('click', increment);
        };
    });

    return (
        <div>
            <p id="count">{count.get()}</p>
            <button id="inc">+</button>
        </div>
    );
}
```
