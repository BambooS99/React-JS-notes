---
title: React search input (fetch on query change, with cancellation)
tags: [react, typescript, search, useEffect, useState, input, abort-controller]
---

## What it does
A controlled search input that runs a request whenever the query changes and stores the results in state. The second version cancels the previous request so a slow, outdated response can't overwrite newer results.

Assumes an async function `searchProducts(query)` that returns a promise of an array. Swap in any fetching function.

## Snippet: basic version
State, in the component:
```tsx
const [query, setQuery] = useState("");
const [results, setResults] = useState<Product[]>([]);
```

In the effect:
```tsx
useEffect(() => {
  searchProducts(query).then(setResults);
}, [query]);
```

Then in the JSX:
```tsx
<input
  value={query}
  onChange={(e) => setQuery(e.target.value)}
  placeholder="Search products"
/>
```

## Snippet: with cancellation (AbortController)
In the effect, replacing the basic one:
```tsx
useEffect(() => {
  const controller = new AbortController();

  searchProducts(query, controller.signal)
    .then(setResults)
    .catch((err) => {
      if (err.name === "AbortError") return; // expected: query changed, request cancelled
      console.error(err);
    });

  return () => controller.abort(); // runs before the next effect and on unmount
}, [query]);
```

The search function accepts the signal and passes it to `fetch`, merged with a timeout:
```ts
export async function searchProducts(
  query: string,
  signal?: AbortSignal,
): Promise<Product[]> {
  const res = await fetch(`${BASE}/products/search?q=${encodeURIComponent(query)}`, {
    signal: signal
      ? AbortSignal.any([signal, AbortSignal.timeout(5000)])
      : AbortSignal.timeout(5000),
  });
  if (!res.ok) {
    throw new Error(`Failed to fetch results! (${res.status})`);
  }
  const data = await res.json();
  return data.products;
}
```

## Why it works / gotchas
- The `[query]` dependency re-runs the effect on every keystroke; setting state re-renders the controlled input.
- `.then(setResults)` passes the resolved array straight to the state setter. `results` starts as `[]` and is filled in when the promise resolves.
- **The race in the basic version:** type "phone" then "laptop" quickly. If the "phone" response is slower, it arrives last and overwrites the "laptop" results. The basic version has no cancellation and no debounce, so every keystroke fires a request.
- **The fix:** the caller owns the `AbortController` and passes `controller.signal` down. The effect cleanup calls `abort()` when `query` changes or the component unmounts. A cancelled `fetch` rejects with `AbortError`, so `.then(setResults)` never runs for the old request; the `.catch` ignores that error and logs anything else.
- `AbortSignal.any([a, b])` combines the caller's signal with the timeout; either one stops the request.
- `AbortSignal.timeout` rejects with `TimeoutError`, not `AbortError`, so it falls through to `console.error`. That is where a "timed out" message would go.
- A simpler alternative to `AbortController`: an `ignore` flag set to `true` in the cleanup, checked before calling `setResults`. It prevents the overwrite but doesn't cancel the request.
- In development with React StrictMode, effects run twice on mount, so one aborted request at startup is normal.
- To test, throttle the network to Slow 3G in DevTools; cancelled requests show as "(canceled)" in the Network tab.
