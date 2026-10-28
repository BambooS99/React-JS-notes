---
title: Parallel search with Promise.all (own file + component)
tags: [promise, promise-all, fetch, abort-controller, react, search, github-api]
---

## What it does
One query, several independent requests fired at once, one combined result. `Promise.all` takes the array of promises and resolves to an array of results in the same order. The fetching lives in its own file; the component only calls one function.

Two examples with the same shape:
1. **Pretend API** (customers, orders, products) to show the pattern with data that belongs together.
2. **Real API** (GitHub issue search) across 3 repos. No key needed.

Each file below is standalone JavaScript, and the component uses React hooks. The pattern also works outside React.

## Part 1: pretend API (customers, orders, products)

### The parallel file: `searchAll.js`
```javascript
import { searchCustomers, searchOrders, searchProducts } from "./api";

export async function searchAll(query, signal) {
  const [customers, orders, products] = await Promise.all([
    searchCustomers(query, signal),
    searchOrders(query, signal),
    searchProducts(query, signal),
  ]);
  return { customers, orders, products };
}
```

### The endpoints it calls (in `api.js`)
A small wrapper over `fetch` that every endpoint shares: it applies the cancel signal and a timeout, and throws on HTTP errors.
```javascript
const BASE = "https://api.example.com";

async function request(path, { signal } = {}) {
  const res = await fetch(`${BASE}${path}`, {
    signal: signal
      ? AbortSignal.any([signal, AbortSignal.timeout(5000)])
      : AbortSignal.timeout(5000),
  });
  if (!res.ok) {
    throw new Error(`Request failed (${res.status})`);
  }
  return res.json();
}

export const searchCustomers = (q, signal) =>
  request(`/customers/search?q=${encodeURIComponent(q)}`, { signal });
export const searchOrders = (q, signal) =>
  request(`/orders/search?q=${encodeURIComponent(q)}`, { signal });
export const searchProducts = (q, signal) =>
  request(`/products/search?q=${encodeURIComponent(q)}`, { signal });
```

### In the component
```javascript
const [query, setQuery] = useState("");
const [results, setResults] = useState({ customers: [], orders: [], products: [] });

useEffect(() => {
  const controller = new AbortController();

  searchAll(query, controller.signal)
    .then(setResults)
    .catch((err) => {
      if (err.name === "AbortError") return;
      console.error(err);
    });

  return () => controller.abort();
}, [query]);
```

Render each group from `results.customers`, `results.orders`, `results.products`.

## Part 2: real API, searching issues across 3 GitHub repos

GitHub's search API is public and needs no key. The repo is part of the query (`repo:owner/name`), so the same keyword goes to three repos in parallel. Checked against the live API: `vuejs/core`, `sveltejs/svelte` and `vitejs/vite` all returned 200.

### The endpoint, in `github.js`
```javascript
const GITHUB = "https://api.github.com";

export async function searchRepoIssues(repo, query, signal) {
  const q = encodeURIComponent(`${query} repo:${repo}`);
  const res = await fetch(`${GITHUB}/search/issues?q=${q}&per_page=5`, {
    signal: signal
      ? AbortSignal.any([signal, AbortSignal.timeout(5000)])
      : AbortSignal.timeout(5000),
  });
  if (!res.ok) {
    throw new Error(`GitHub search failed for ${repo} (${res.status})`);
  }
  const data = await res.json();
  return data.items;
}
```

### The parallel file: `searchRepos.js`
```javascript
import { searchRepoIssues } from "./github";

const REPOS = ["vuejs/core", "sveltejs/svelte", "vitejs/vite"];

export async function searchAcrossRepos(query, signal) {
  const lists = await Promise.all(
    REPOS.map((repo) => searchRepoIssues(repo, query, signal)),
  );
  return REPOS.map((repo, i) => ({ repo, issues: lists[i] }));
}
```

### In the component
```javascript
const [query, setQuery] = useState("");
const [groups, setGroups] = useState([]);

useEffect(() => {
  if (!query.trim()) return;
  const controller = new AbortController();

  searchAcrossRepos(query, controller.signal)
    .then(setGroups)
    .catch((err) => {
      if (err.name === "AbortError") return;
      console.error(err);
    });

  return () => controller.abort();
}, [query]);
```

Render one section per `{ repo, issues }` and list `issue.title` and `issue.html_url`.

## Why it works / gotchas
- All requests start immediately inside the array, so total time is about the slowest one, not the sum. Writing `await` on each one in turn would run them one after another.
- `Promise.all` keeps order: the result array lines up with the input array. That is why `REPOS.map(...)` and `lists[i]` match up, and why destructuring works in Part 1.
- **Fail-fast:** if one request rejects, `Promise.all` rejects and you lose the others. Swap to `Promise.allSettled` when partial results are fine; then each entry is `{ status, value }` or `{ status, reason }` and you handle them yourself.
- One shared `signal` cancels every request in the group when the query changes. Pass it to every call; a call without it is not cancelled.
- `Promise.all` accepts any array, so `REPOS.map(...)` scales to as many repos as you list. Adding a repo is one string.
- **Rate limit (observed):** unauthenticated GitHub search is limited to about 10 requests per minute. While testing, a repo returned 403 after a handful of calls. Three repos per keystroke burns the limit fast, so debounce and skip empty queries (`if (!query.trim()) return`). A personal token in an `Authorization` header raises the limit.
- Some repos can't be searched: `facebook/react` returned 422 ("cannot be searched") at the time of writing, even though the repo exists. A 422 or 403 on one repo would fail the whole `Promise.all`, another reason to prefer `allSettled` here.
- Part 1 endpoints are made up; only Part 2 is a live API.
