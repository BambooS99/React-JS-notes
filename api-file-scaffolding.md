---
title: API file scaffolding (fetch + ok-check)
tags: [api, fetch, async-await, error-handling, scaffolding]
source: C:\Users\Owais\Documents\JS\promise-practice\src\api.ts
---

## What it does
One file holds the base URL and one small function per endpoint. Each function fetches, throws if the response isn't OK, and returns parsed JSON. Components import these functions and never touch `fetch`. TypeScript types and app-specific shapes (`Product`, etc.) are removed here; the pattern is the same with or without them.

## Snippet
Top of the file, base URL once:
```javascript
const BASE = "https://dummyjson.com";
```

Endpoint that takes a value and the response wraps the data (`data.<key>` unwraps it):
```javascript
export async function searchThings(query) {
  const res = await fetch(`${BASE}/things/search?q=${query}`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  const data = await res.json();
  return data.things;
}
```

Endpoint where the response is already the data you want (return the parsed JSON directly):
```javascript
export async function getThing(id) {
  const res = await fetch(`${BASE}/things/${id}`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  return res.json();
}
```

Endpoint with no arguments:
```javascript
export async function getCategories() {
  const res = await fetch(`${BASE}/things/category-list`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  return res.json();
}
```

Helper for simulating slow or hung requests:
```javascript
export const sleep = (ms) =>
  new Promise((resolve) => setTimeout(resolve, ms));
```

## Why it works / gotchas
- `fetch` only rejects on network failure. A 404 or 500 still resolves, so the `if (!res.ok) throw` is what turns HTTP errors into rejections the caller's `catch` can handle.
- Throwing inside an `async` function returns a rejected promise, so callers use `try/catch` (or `.catch`) around `await searchThings(...)`.
- `return res.json()` and `return await res.json()` behave the same for callers; both return a promise of the parsed data.
- The `!res.ok` block is repeated in every function. A shared `request(path)` helper would remove the repetition (not part of the original file).
- The query is interpolated into the URL as-is. Wrap it in `encodeURIComponent(query)` if it can contain spaces or special characters (not in the original file).
- The error message is the same generic string everywhere; it doesn't include the HTTP status.
- Names `things`/`searchThings`/`getThing` are placeholders; swap in your own resource names and endpoint paths.

## Whole file, assembled
How `api.js` looks with the pieces above together (types removed):
```javascript
const BASE = "https://dummyjson.com";

export async function searchThings(query) {
  const res = await fetch(`${BASE}/things/search?q=${query}`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  const data = await res.json();
  return data.things;
}

export async function getThing(id) {
  const res = await fetch(`${BASE}/things/${id}`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  return res.json();
}

export async function getCategories() {
  const res = await fetch(`${BASE}/things/category-list`);
  if (!res.ok) {
    throw new Error("Failed to fetch results!");
  }
  return res.json();
}

export const sleep = (ms) =>
  new Promise((resolve) => setTimeout(resolve, ms));
```

## Consuming it in a component
From `App.tsx` (types removed). Import the function, hold the result in state, and call it from an effect:
```javascript
import { useEffect, useState } from "react";
import { searchThings } from "./api";

export default function App() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);

  useEffect(() => {
    searchThings(query).then(setResults);
  }, [query]);

  return (
    <div>
      <input
        value={query}
        onChange={(e) => {
          setQuery(e.target.value);
        }}
      />
      <ul>
        {results.map((p) => (
          <li key={p.id}>{p.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

Consumer gotcha: since `api.js` throws on `!res.ok`, the call above has no `.catch`, so a failure becomes an unhandled promise rejection and the UI shows nothing. The original `App.tsx` has this same gap. Handling it (not in the original file):
```javascript
useEffect(() => {
  searchThings(query).then(setResults).catch((err) => console.error(err));
}, [query]);
```
Next step from there is `error`/`loading` state (see `PRACTICE.md`, exercise 1).
