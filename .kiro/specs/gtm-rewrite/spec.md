# `@providirect/analytics` — v1 Implementation Spec

**Status:** Ready for implementation
**Date:** 2026-06-08

---

## 0. Prerequisites & Constraints

**GDPR requirement — consuming app responsibility:**
This package provides no built-in consent mechanism. The consuming app MUST obtain user consent before calling `inject()`. Do not call `inject()` on page load unconditionally in EU-facing apps. A typical integration gates `inject()` on a consent callback. This is explicitly a consuming-app concern; the package does not enforce it.

---

## 1. Package Structure

```
packages/analytics/
├── src/
│   ├── state.ts         # Module-level singleton config store
│   ├── pipeline.ts      # Shared dispatch pipeline (dataLayer + HTTP)
│   ├── listeners.ts     # History API listener registration + cleanup
│   ├── inject.ts        # inject() — client initialisation
│   ├── track.ts         # track() — custom event dispatch
│   ├── types.ts         # All TypeScript interfaces + global augmentations
│   ├── react/
│   │   └── index.tsx    # <Analytics /> — generic React (History API mode)
│   ├── nextjs/
│   │   └── index.tsx    # <Analytics /> — Next.js (usePathname mode only)
│   └── server/
│       └── index.ts     # trackServer() — Node.js/Edge fetch
├── package.json         # exports map below
└── tsconfig.json        # target: ES2020, moduleResolution: bundler, strict: true
```

**Build tool:** `tsup`. Emits ESM + CJS + `.d.ts` declarations for every entry point.

**`package.json` exports:**

```json
{
  "name": "@providirect/analytics",
  "version": "1.0.0",
  "exports": {
    ".":          { "import": "./dist/inject.js",         "types": "./dist/inject.d.ts" },
    "./react":    { "import": "./dist/react/index.js",    "types": "./dist/react/index.d.ts" },
    "./nextjs":   { "import": "./dist/nextjs/index.js",   "types": "./dist/nextjs/index.d.ts" },
    "./server":   { "import": "./dist/server/index.js",   "types": "./dist/server/index.d.ts" }
  },
  "sideEffects": false
}
```

**Install (git dependency):**

```json
"@providirect/analytics": "git+https://github.com/providirect/analytics.git#semver:^1.0.0"
```

**Bundle size target:** < 10KB gzipped for the full client bundle. Apps using only `/nextjs` should tree-shake to < 6KB.

---

## 2. Types (`types.ts`)

```typescript
export interface AnalyticsConfig {
  /** POST endpoint for HTTP delivery. Omit for dataLayer-only mode. */
  endpoint?: string;
  /** Additional headers sent with every client-side HTTP POST (e.g. API key). */
  requestHeaders?: Record<string, string>;
  /**
   * 'production' | 'development'.
   * Defaults to: typeof process !== 'undefined' && process.env.NODE_ENV === 'production'
   *   ? 'production' : 'development'
   * Pass explicitly if your bundler does not inject process.env.NODE_ENV.
   */
  mode?: 'production' | 'development';
  /**
   * Synchronous only. Return a modified event to change it, return null to drop it.
   * Throwing is treated as a drop + console.warn in dev.
   * Async functions are NOT supported — return value must be AnalyticsEvent | null.
   */
  beforeSend?: (event: AnalyticsEvent) => AnalyticsEvent | null;
}

export type AnalyticsEvent = PageViewEvent | CustomEvent;

export interface PageViewEvent {
  type: 'page_view';
  url: string;       // window.location.href
  path: string;      // window.location.pathname
  referrer: string;  // previous path, or document.referrer on first load, or ''
  ts: number;        // Date.now()
}

export interface CustomEvent {
  type: 'custom';
  name: string;
  properties: Record<string, string | number | boolean | null>;
  ts: number;
}

// Global augmentation — avoids TypeScript errors on window.dataLayer
declare global {
  interface Window {
    dataLayer?: Record<string, unknown>[];
  }
}
```

---

## 3. Singleton Config Store (`state.ts`)

```typescript
import type { AnalyticsConfig } from './types';

let _config: AnalyticsConfig | null = null;

export function setConfig(config: AnalyticsConfig): void {
  if (_config !== null) {
    if (_config.mode !== 'production') {
      console.warn('[analytics] inject() called more than once. Subsequent calls are ignored.');
    }
    return;
  }
  _config = config;
}

export function getConfig(): AnalyticsConfig | null {
  return _config;
}

/** For testing only — resets singleton between test cases. */
export function _resetConfig(): void {
  _config = null;
}
```

**Contract:** `inject()` calls `setConfig()` once. `track()`, `pipeline`, and listeners all read via `getConfig()`. If `getConfig()` returns `null`, callers are no-ops.

---

## 4. Dispatch Pipeline (`pipeline.ts`)

Every client-side event flows through this function:

```typescript
import type { AnalyticsEvent } from './types';
import { getConfig } from './state';

export function dispatch(raw: AnalyticsEvent): void {
  const config = getConfig();
  if (!config) return;

  // 1. beforeSend — synchronous, return null drops the event
  let event: AnalyticsEvent | null = raw;
  if (config.beforeSend) {
    try {
      event = config.beforeSend(raw);
    } catch (err) {
      if (config.mode !== 'production') {
        console.warn('[analytics] beforeSend threw, event dropped:', err);
      }
      return;
    }
  }
  if (event === null || event === undefined) return;

  // 2. dataLayer push — always, client-side only
  window.dataLayer = window.dataLayer ?? [];
  try {
    window.dataLayer.push(toDataLayerShape(event));
  } catch (err) {
    if (config.mode !== 'production') {
      console.warn('[analytics] dataLayer.push failed:', err);
    }
  }

  // 3. Dev console logging
  if (config.mode !== 'production') {
    console.info('[analytics]', event);
  }

  // 4. HTTP POST — fire-and-forget, additive
  if (config.endpoint) {
    postEvent(config.endpoint, config.requestHeaders ?? {}, event, config.mode);
  }
}

function toDataLayerShape(event: AnalyticsEvent): Record<string, unknown> {
  if (event.type === 'page_view') {
    return {
      event: 'page_view',
      page_url: event.url,
      page_path: event.path,
      page_referrer: event.referrer,
    };
  }
  // Custom events: name becomes the GTM event trigger directly, properties flattened onto object
  // GTM variables are {{DLV - price}} not {{DLV - event_properties.price}}
  return {
    event: event.name,
    ...event.properties,
  };
}

async function postEvent(
  endpoint: string,
  headers: Record<string, string>,
  event: AnalyticsEvent,
  mode: AnalyticsConfig['mode'],
): Promise<void> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 5000);
  try {
    await fetch(endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', ...headers },
      body: JSON.stringify(event),
      signal: controller.signal,
    });
  } catch (err) {
    if (mode !== 'production') {
      console.warn('[analytics] POST failed:', err);
    }
  } finally {
    clearTimeout(timeout);
  }
}
```

---

## 5. History API Listeners (`listeners.ts`)

```typescript
import { dispatch } from './pipeline';
import type { PageViewEvent } from './types';

let _previousPath = '';
let _removeListeners: (() => void) | null = null;

export function registerListeners(): () => void {
  if (_removeListeners) return _removeListeners;

  _previousPath = window.location.pathname;

  function handleNavigation(): void {
    const path = window.location.pathname;
    if (path === _previousPath) return; // hash-only change, skip
    const event: PageViewEvent = {
      type: 'page_view',
      url: window.location.href,
      path,
      referrer: _previousPath,
      ts: Date.now(),
    };
    _previousPath = path;
    dispatch(event);
  }

  const originalPushState = history.pushState.bind(history);
  const originalReplaceState = history.replaceState.bind(history);

  history.pushState = (...args) => { originalPushState(...args); handleNavigation(); };
  history.replaceState = (...args) => { originalReplaceState(...args); handleNavigation(); };
  window.addEventListener('popstate', handleNavigation);

  _removeListeners = () => {
    history.pushState = originalPushState;
    history.replaceState = originalReplaceState;
    window.removeEventListener('popstate', handleNavigation);
    _removeListeners = null;
    _previousPath = '';
  };

  return _removeListeners;
}
```

---

## 6. `inject()` (`inject.ts`)

```typescript
import { setConfig } from './state';
import { registerListeners } from './listeners';
import { dispatch } from './pipeline';
import type { AnalyticsConfig, PageViewEvent } from './types';

export function inject(config: AnalyticsConfig = {}): () => void {
  if (typeof window === 'undefined') return () => {};

  setConfig(config);

  dispatch({
    type: 'page_view',
    url: window.location.href,
    path: window.location.pathname,
    referrer: document.referrer ?? '',
    ts: Date.now(),
  } satisfies PageViewEvent);

  return registerListeners();
}
```

Returns a cleanup function so `useEffect` can deregister listeners on unmount.

---

## 7. `track()` (`track.ts`)

```typescript
import { getConfig } from './state';
import { dispatch } from './pipeline';
import type { CustomEvent } from './types';

export function track(
  name: string,
  properties: Record<string, string | number | boolean | null> = {},
): void {
  if (typeof window === 'undefined') return;
  if (!getConfig()) {
    if (typeof process === 'undefined' || process.env.NODE_ENV !== 'production') {
      console.warn('[analytics] track() called before inject(). Event dropped:', name);
    }
    return;
  }
  dispatch({ type: 'custom', name, properties, ts: Date.now() } satisfies CustomEvent);
}
```

---

## 8. Framework Adapters

### 8.1 Generic React (`react/index.tsx`)

Uses History API listeners. Suitable for React apps without Next.js App Router.

```tsx
'use client';
import { useEffect } from 'react';
import { inject } from '../inject';
import type { AnalyticsConfig } from '../types';

export type { AnalyticsConfig };
export { track } from '../track';

export function Analytics(props: AnalyticsConfig): null {
  // Destructure at mount time — avoids re-running effect on referentially
  // unstable beforeSend prop (new function instance on every parent render)
  const { endpoint, requestHeaders, mode, beforeSend } = props;

  useEffect(() => {
    const cleanup = inject({ endpoint, requestHeaders, mode, beforeSend });
    return cleanup;
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  return null;
}
```

### 8.2 Next.js App Router (`nextjs/index.tsx`)

**Critical design decision:** This adapter uses `usePathname()` as the sole navigation signal. It does NOT register History API listeners via `inject()`. Next.js App Router intercepts `pushState` internally, making `usePathname()` the only reliable navigation signal. Using both mechanisms would fire two `page_view` events per navigation.

```tsx
'use client';
import { useEffect, useRef } from 'react';
import { usePathname } from 'next/navigation';
import { setConfig } from '../state';
import { dispatch } from '../pipeline';
import type { AnalyticsConfig, PageViewEvent } from '../types';

export type { AnalyticsConfig };
export { track } from '../track';

export function Analytics(props: AnalyticsConfig): null {
  const { endpoint, requestHeaders, mode, beforeSend } = props;
  const pathname = usePathname();
  const previousPath = useRef<string>('');
  const initialised = useRef(false);

  useEffect(() => {
    if (!initialised.current) {
      initialised.current = true;
      setConfig({ endpoint, requestHeaders, mode, beforeSend });
      dispatch({
        type: 'page_view',
        url: window.location.href,
        path: pathname,
        referrer: document.referrer ?? '',
        ts: Date.now(),
      } satisfies PageViewEvent);
      previousPath.current = pathname;
    }
  }, []); // eslint-disable-line react-hooks/exhaustive-deps

  useEffect(() => {
    if (!initialised.current) return;
    if (pathname === previousPath.current) return;
    dispatch({
      type: 'page_view',
      url: window.location.href,
      path: pathname,
      referrer: previousPath.current,
      ts: Date.now(),
    } satisfies PageViewEvent);
    previousPath.current = pathname;
  }, [pathname]);

  return null;
}
```

> **Note:** Any env var passed as `endpoint` from a Server Component must use the `NEXT_PUBLIC_` prefix to be available client-side. Example: `endpoint={process.env.NEXT_PUBLIC_ANALYTICS_ENDPOINT}`.

---

## 9. Server Export (`server/index.ts`)

```typescript
export interface ServerTrackOptions {
  /** Required — server-side events POST to HTTP only, never to dataLayer. */
  endpoint: string;
  name: string;
  properties?: Record<string, string | number | boolean | null>;
  /** Forward request headers (user-agent, x-forwarded-for, etc.) */
  headers?: Record<string, string>;
  mode?: 'production' | 'development';
}

/** Fire a custom event from Node.js or Edge runtime. Do NOT await — fire-and-forget. */
export function trackServer(options: ServerTrackOptions): void {
  const { endpoint, name, properties = {}, headers = {}, mode } = options;
  const payload = { type: 'custom' as const, name, properties, ts: Date.now() };
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 5000);

  fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', ...headers },
    body: JSON.stringify(payload),
    signal: controller.signal,
  })
    .catch((err) => {
      if (mode !== 'production') console.warn('[analytics/server] POST failed:', err);
    })
    .finally(() => clearTimeout(timeout));
}
```

---

## 10. DataLayer Event Reference

All client events push to `window.dataLayer`. GTM triggers on the `event` field.

### Page view

```js
{
  event: 'page_view',
  page_url: 'https://app.example.com/dashboard',
  page_path: '/dashboard',
  page_referrer: '/login',   // '' on first load
}
```

GTM trigger: Custom Event, Event Name = `page_view`

### Custom event

```js
// track('button_click', { label: 'sign_up', location: 'hero' })
{
  event: 'button_click',
  label: 'sign_up',
  location: 'hero',
}
```

GTM trigger: Custom Event, Event Name = `button_click`, or a regex pattern matching your naming convention.

**GTM variable lookups:** `{{DLV - label}}`, `{{DLV - location}}` — flat, no nesting.

---

## 11. HTTP POST Payload Contract

```
POST {endpoint}
Content-Type: application/json
{...requestHeaders}

// Page view
{ "type": "page_view", "url": "...", "path": "...", "referrer": "...", "ts": 1234567890 }

// Custom event
{ "type": "custom", "name": "button_click", "properties": { "label": "sign_up" }, "ts": 1234567890 }
```

All field names are camelCase in both TypeScript types and HTTP payload. Response body is ignored. Any 2xx is success.

---

## 12. Failure Handling

| Scenario | Dev | Prod |
|---|---|---|
| `endpoint` 4xx / 5xx / timeout | `console.warn` | Silent |
| `window.dataLayer.push` throws | `console.warn` | Silent |
| `beforeSend` throws | `console.warn`, event dropped | Silent, event dropped |
| `beforeSend` returns `undefined` or non-event | `console.warn`, event dropped | Silent, event dropped |
| `track()` before `inject()` | `console.warn`, event dropped | Silent, event dropped |
| `inject()` in SSR | No-op, no log | No-op, no log |
| `inject()` called twice | `console.warn` | Silent |
| GTM not loaded | Safe — GTM reads array retroactively | Same |
| Component unmounts | Listeners removed via cleanup fn | Same |

---

## 13. Usage Examples

### Next.js 16 App Router

```tsx
// app/layout.tsx — Server Component; Analytics renders as a Client Component
import { Analytics } from '@providirect/analytics/nextjs';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        {children}
        {/* NEXT_PUBLIC_ prefix required — value is read client-side */}
        <Analytics endpoint={process.env.NEXT_PUBLIC_ANALYTICS_ENDPOINT} />
      </body>
    </html>
  );
}
```

### Custom event

```tsx
import { track } from '@providirect/analytics/nextjs';

<button onClick={() => track('button_click', { label: 'sign_up', location: 'hero' })}>
  Sign up
</button>
```

### Server-side from `proxy.ts`

```typescript
import { trackServer } from '@providirect/analytics/server';

// Do NOT await — fire-and-forget to avoid adding latency to every request
trackServer({
  endpoint: process.env.ANALYTICS_ENDPOINT!,
  name: 'auth_failure',
  properties: { reason: 'invalid_token', app_id: appId },
  headers: Object.fromEntries(request.headers),
  mode: process.env.NODE_ENV === 'production' ? 'production' : 'development',
});
```

### Consent-gated initialisation

```tsx
// Only call inject() after consent has been granted
onConsentGranted(() => {
  inject({ endpoint: process.env.NEXT_PUBLIC_ANALYTICS_ENDPOINT });
});
```

---

## 14. Testing Requirements

Minimum test surface (use `vitest` + `jsdom`):

```
✓ inject() fires page_view on call
✓ navigation (pushState) fires page_view with correct referrer
✓ navigation to same path does not fire page_view
✓ inject() twice: second call is no-op, no duplicate listeners
✓ cleanup fn from inject() removes all History API listeners
✓ track() pushes correct flat dataLayer shape
✓ track() before inject() is a no-op
✓ beforeSend returning null drops event from both dataLayer and HTTP
✓ beforeSend throwing drops event, does not crash
✓ beforeSend returning undefined drops event, does not crash
✓ typeof window === 'undefined' — inject(), track() are no-ops, no throw
✓ Next.js adapter: usePathname change fires page_view
✓ Next.js adapter: does NOT register History API listeners
✓ _resetConfig() clears singleton between tests
```

---

## 15. Out of Scope — v1

- Route parameterisation (`/products/123` → `/products/[id]`)
- Built-in consent / GDPR opt-out UI (consuming app responsibility — see §0)
- Event batching or debouncing
- Retry logic for failed POSTs
- Offline queue (localStorage / IndexedDB)
- Session or user identity tracking
- SvelteKit, Nuxt, Remix, Vue adapters
- Multiple simultaneous endpoints
- `beforeSend` async support
