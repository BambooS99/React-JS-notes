---
title: API file scaffolding (fetch wrapper, basic to standard)
tags: [api, fetch, async-await, error-handling, scaffolding, abort-signal]
---

## What it does
One file holds the base URL and one small function per endpoint. Components import these functions and never call `fetch` directly. Two levels are shown: a basic version, and the shared-helper version most real codebases use. Examples are plain JavaScript; `things` and `/things/...` are placeholder names to replace with your own resource.

## Level 1: basic
Base URL once at the top of the file:
```javascript
const BASE = "https://api.example.com";
```

Endpoint that takes a value, where the response wraps the data (`data.things` unwraps it):
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

Endpoint where the response is already the data you want:
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

Helper for simulating slow or hung requests while testing:
```javascript
export const sleep = (ms) =>
  new Promise((resolve) => setTimeout(resolve, ms));
```

Using it from a React component:
```javascript
const [query, setQuery] = useState("");
const [results, setResults] = useState([]);

useEffect(() => {
  searchThings(query).then(setResults).catch((err) => console.error(err));
}, [query]);
```

### Level 1 gotchas
- `fetch` only rejects on network failure. A 404 or 500 still resolves, so `if (!res.ok) throw` is what turns HTTP errors into rejections that a `catch` can handle.
- Throwing inside an `async` function returns a rejected promise, so callers need `try/catch` or `.catch`. Without it, a failure is an unhandled promise rejection and the UI just shows nothing.
- `return res.json()` and `return await res.json()` behave the same for callers.
- The `!res.ok` block is repeated in every function, there is no timeout or cancellation, the query isn't URL-encoded, and the error message doesn't include the HTTP status. Level 2 fixes all of these.

## Level 2: shared `request` helper
```javascript
const BASE = import.meta.env.VITE_API_URL;

class ApiError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
  }
}

async function request(path, { signal, ...options } = {}) {
  const res = await fetch(`${BASE}${path}`, {
    ...options,
    signal: signal
      ? AbortSignal.any([signal, AbortSignal.timeout(5000)])
      : AbortSignal.timeout(5000),
  });
  if (!res.ok) {
    throw new ApiError(res.status, `Request failed (${res.status})`);
  }
  return res.json();
}

export async function searchThings(query, signal) {
  const data = await request(`/things/search?q=${encodeURIComponent(query)}`, { signal });
  return data.things;
}

export const getThing = (id, signal) => request(`/things/${id}`, { signal });
export const getCategories = (signal) => request(`/things/category-list`, { signal });
```

Using the signal from a component: create an `AbortController` in the effect, pass `controller.signal` as the last argument, call `controller.abort()` in the cleanup, and ignore `AbortError` in the `catch`:
```javascript
useEffect(() => {
  const controller = new AbortController();

  searchThings(query, controller.signal)
    .then(setResults)
    .catch((err) => {
      if (err.name === "AbortError") return;
      console.error(err);
    });

  return () => controller.abort();
}, [query]);
```

### What changed from Level 1
| Level 1 | Level 2 |
|---|---|
| `fetch` + `ok` check copied into every function | Done once in `request`; endpoints are one or two lines |
| No timeout | `AbortSignal.timeout(5000)` applied in one place |
| No way to cancel | Every endpoint takes an optional `signal`, merged with the timeout via `AbortSignal.any` |
| Query put into the URL raw | `encodeURIComponent(query)` |
| Generic `Error("Failed to fetch results!")` | `ApiError` carrying `status`, so callers can tell 404 from 500 |
| Hardcoded `BASE` string | Read from an env variable (`VITE_API_URL` in Vite; set it in `.env`) |

### Level 2 gotchas
- `request` always parses JSON. An endpoint that returns no body (e.g. a 204 on DELETE) throws on `res.json()`; handle that case before parsing.
- Extra `options` (method, headers, body) pass straight through to `fetch`, so POST/PUT work: `request(path, { method: "POST", headers: {...}, body: JSON.stringify(x) })`. Set `Content-Type: application/json` for JSON bodies.
- `AbortSignal.timeout` rejects with `TimeoutError`, while a manual `abort()` rejects with `AbortError`. Treat them differently in the UI.
- `AbortSignal.any` needs a recent browser or Node version; check support if you target older ones.
- Auth tokens belong in a header added inside `request`, not in each endpoint function.
- `import.meta.env` is Vite-specific; other bundlers use `process.env.<NAME>` or their own mechanism.
