---
name: carats-hooks
description: Use when a Carats component needs state (`use`) or a client-side lifecycle callback (`afterMount`, `beforeMount`).
---

Dependency: `@carats/hooks`. Read `carats-essentials` first.

`@carats/hooks` exports state management and the client-side lifecycle.

## Lifecycle
A component function runs on both the server and the client, so a hook body must never touch the DOM or the network. When a side effect is needed, put it inside a lifecycle callback: **afterMount** or **beforeMount**. Both accept a callback that runs only in the browser; the callback may return a clear callback for cleanup.

The two differ only in when they run:

- **afterMount** runs after the `load` event has been dispatched, so the document is complete. Use it to reach DOM elements the same component created.
- **beforeMount** runs immediately, without waiting for `load`. Use it for work that does not need the component's own markup, such as timers or key listeners.

Prefer **afterMount** for anything that reads the DOM, because waiting for `load` guarantees the markup exists. Reach for **beforeMount** when the work is independent of the markup and should start sooner.

Keep a reference to the exact handler that was registered. Passing a fresh arrow function to `removeEventListener` creates a new function identity, so the listener is never actually removed.

Call `subscribe` from a lifecycle callback and return its unsubscribe function as the clear callback; otherwise the subscriber stays registered after the page is replaced.

### Cleanup on navigation
`clientRender()` calls `clearHydrations()` before rendering the next page. That runs every registered clear callback and empties the registry, so clear callbacks run on in-app navigation. A full page unload does not call them; the browser simply discards the page. `clearHydrations()` is public so tests and manual teardown can release the same registrations; application code rarely calls it directly.

There is no `useEffect` and no dependency array. Because the component body runs on every render, the `use`, `afterMount`, and `beforeMount` calls inside it run again too. Keep that setup idempotent and put the real work inside the callback.

## State
For state, use `use`. It is not reactive like a React hook: setting a value re-renders nothing, so nothing on the page updates by itself. Read the value with `get()` while rendering, write the DOM from a `subscribe` callback, and change the value from anywhere. Keep the handler that changes state separate from the subscriber that writes the DOM, or the element ends up with two sources of truth.

`set` returns the new value, so the result is available immediately when a caller needs it.

`use(0)` inside a component does not persist across renders. Every call creates a fresh state object, so a re-run starts again at `0`. If state must survive a re-render or be shared between pages, create it at module scope:

```typescript
// src/client/state/cart.ts
import { use } from '@carats/hooks';

interface Cart {
    items: string[];
}

export const cart = use<Cart>({ items: [] });
```

```tsx
import { cart } from '../state/cart';

export default function CartButton() {
    return <button id="cart-count">Cart: {cart.get().items.length}</button>;
}
```

The module-level object is created once and shared by every component that imports it. During server-side rendering it is also shared by every request in that server process, so use module-level state for UI state, not per-user data.

The exact `@carats/hooks` signatures are documented once in [carats-detailed-types](../carats-detailed-types/SKILL.md).

## Counter example
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
