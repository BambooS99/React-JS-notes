---
title: React search input (effect on query)
tags: [react, typescript, search, useEffect, useState, input]
source: C:\Users\Owais\Documents\JS\promise-practice\src\App.tsx
---

## What it does
Runs a search whenever the query changes and stores results in state; the input is a controlled component bound to `query`.

## Snippet
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
  placeholder="Search products (try: phone, laptop, mascara)"
/>
```

## Why it works / gotchas
- `[query]` dependency re-runs the search on every keystroke; the state update re-renders the input (controlled).
- `.then(setResults)` passes the resolved array straight to the state setter.
- The source file marks this version as deliberately naive: no cancellation, so a slow earlier response can overwrite a newer one (race). Add an `ignore` flag in the effect cleanup if it matters.
- No debouncing and no error handling; every keystroke fires a request.
