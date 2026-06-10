# Requirements Document

## Introduction

`@providirect/analytics` is a lightweight TypeScript analytics package that replaces the existing GTM-based tracking setup with a clean, framework-agnostic dispatch pipeline. The package provides a singleton configuration store, a unified event pipeline (dataLayer push + HTTP POST), and framework adapters for React, Next.js App Router, and server-side Node.js/Edge runtimes. GDPR consent is explicitly a consuming-app responsibility — the package provides no built-in consent mechanism.

## Glossary

- **Analytics_Package**: The `@providirect/analytics` npm package providing event dispatch capabilities
- **Config_Store**: Module-level singleton that holds the `AnalyticsConfig` object, set once via `inject()`
- **Dispatch_Pipeline**: The central `dispatch()` function through which all client-side events flow
- **DataLayer**: The `window.dataLayer` array used by Google Tag Manager to receive event objects
- **History_Listeners**: Monkey-patched `history.pushState`, `history.replaceState`, and `popstate` event listener for SPA navigation tracking
- **React_Adapter**: The `<Analytics />` component exported from `/react` that uses History API listeners
- **NextJS_Adapter**: The `<Analytics />` component exported from `/nextjs` that uses `usePathname()` as the sole navigation signal
- **Server_Export**: The `trackServer()` function exported from `/server` for Node.js/Edge runtime event dispatch
- **beforeSend_Hook**: A synchronous user-provided function that can transform or drop events before dispatch
- **Page_View_Event**: An event of type `page_view` containing url, path, referrer, and timestamp
- **Custom_Event**: An event of type `custom` containing name, properties, and timestamp
- **SSR**: Server-Side Rendering — an environment where `typeof window === 'undefined'`

## Requirements

### Requirement 1: Client-side Initialisation

**User Story:** As a developer, I want to initialise the analytics package with a single function call, so that page view tracking begins automatically without additional setup.

#### Acceptance Criteria

1. WHEN `inject()` is called in a browser environment, THE Config_Store SHALL store the provided configuration such that `getConfig()` returns the stored configuration object for all subsequent operations within the same page lifecycle
2. WHEN `inject()` is called in a browser environment, THE Dispatch_Pipeline SHALL fire an initial `page_view` event containing the current `window.location.href`, `window.location.pathname`, and `document.referrer`
3. WHEN `inject()` is called in a browser environment, THE History_Listeners SHALL monkey-patch `history.pushState` and `history.replaceState` and add a `popstate` event listener to track subsequent SPA navigations
4. WHEN `inject()` completes, THE Analytics_Package SHALL return a cleanup function that restores the original `history.pushState` and `history.replaceState` methods and removes the `popstate` event listener
5. WHEN `inject()` is called in an SSR environment where `typeof window === 'undefined'`, THE Analytics_Package SHALL return a no-op cleanup function without throwing or producing console output, dataLayer mutations, or HTTP requests
6. IF `inject()` has already been called successfully in the current page lifecycle, THEN THE Analytics_Package SHALL ignore the subsequent call, preserve the original configuration unchanged, skip duplicate listener registration, and log a warning in development mode

### Requirement 2: Singleton Initialisation Guard

**User Story:** As a developer, I want the analytics package to prevent duplicate initialisation, so that multiple mount points or accidental double-calls do not create duplicate event streams.

#### Acceptance Criteria

1. WHEN `inject()` is called a second time after successful initialisation, THE Config_Store SHALL ignore the second configuration and preserve the original such that `getConfig()` continues to return the first configuration object
2. WHEN `inject()` is called a second time WHILE the configured mode is not `production`, THE Config_Store SHALL emit a `console.warn` indicating the duplicate call was ignored
3. WHEN `inject()` is called a second time WHILE the configured mode is `production`, THE Config_Store SHALL silently ignore the call without any console output
4. WHEN `inject()` is called a second time, THE History_Listeners SHALL NOT register a second set of monkey-patches or event listeners
5. WHEN `inject()` is called a second time, THE Analytics_Package SHALL return a cleanup function (the same cleanup function from the first registration) without creating a new one

### Requirement 3: Event Dispatch Pipeline

**User Story:** As a developer, I want all events to flow through a single dispatch pipeline, so that dataLayer pushes and HTTP delivery are handled consistently for every event type.

#### Acceptance Criteria

1. WHEN the Dispatch_Pipeline receives an event and the Config_Store is initialised with a `beforeSend` function, THE Dispatch_Pipeline SHALL apply `beforeSend` synchronously to the event and push the returned (transformed) event object to `window.dataLayer`
2. WHEN the Dispatch_Pipeline receives an event and an `endpoint` is configured, THE Dispatch_Pipeline SHALL fire an HTTP POST request with the event payload serialised as JSON, using a 5-second timeout via AbortController, without awaiting the response before completing the dataLayer push
3. IF the Dispatch_Pipeline receives an event and the Config_Store returns null, THEN THE Dispatch_Pipeline SHALL take no action and produce no side effects
4. WHEN the Dispatch_Pipeline pushes an event to the DataLayer, THE Dispatch_Pipeline SHALL use the event name directly as the `event` field value — `page_view` for page view events, and the custom event's `name` for custom events — without any prefix
5. WHEN the Dispatch_Pipeline pushes a Custom_Event to the DataLayer, THE Dispatch_Pipeline SHALL flatten all properties directly onto the dataLayer object without nesting
6. WHILE the Config_Store mode is not `production`, WHEN the Dispatch_Pipeline receives an event, THE Dispatch_Pipeline SHALL log the event to `console.info`
7. IF the configured `beforeSend` function returns null, returns undefined, or throws an exception, THEN THE Dispatch_Pipeline SHALL drop the event entirely without pushing to `window.dataLayer` and without firing an HTTP POST request
8. IF the HTTP POST request fails due to network error, timeout, or server error, THEN THE Dispatch_Pipeline SHALL suppress the error silently in production mode and log a warning via `console.warn` in non-production mode without affecting the already-completed dataLayer push

### Requirement 4: beforeSend Hook

**User Story:** As a developer, I want to transform or filter events before they are dispatched, so that I can strip PII, enrich events, or conditionally drop events based on application logic.

#### Acceptance Criteria

1. WHEN a `beforeSend` function is configured and returns a valid `AnalyticsEvent` object, THE Dispatch_Pipeline SHALL discard the original event and use the returned event for both the dataLayer push and the HTTP POST delivery
2. WHEN a `beforeSend` function returns `null`, THE Dispatch_Pipeline SHALL drop the event entirely — no dataLayer push and no HTTP POST shall occur
3. WHEN a `beforeSend` function returns `undefined`, THE Dispatch_Pipeline SHALL treat it as a drop — no dataLayer push and no HTTP POST shall occur
4. WHEN a `beforeSend` function throws an exception and the configured mode is not `production`, THE Dispatch_Pipeline SHALL drop the event and log a warning via `console.warn`
5. WHEN a `beforeSend` function throws an exception and the configured mode is `production`, THE Dispatch_Pipeline SHALL drop the event silently with zero console output
6. WHEN a `beforeSend` function throws an exception, THE Dispatch_Pipeline SHALL continue processing subsequent events normally without requiring reinitialisation
7. THE Dispatch_Pipeline SHALL invoke the `beforeSend` function synchronously — if a `beforeSend` function returns a Promise, THE Dispatch_Pipeline SHALL treat the Promise object as a non-null return value and pass it through without awaiting it

### Requirement 5: HTTP Delivery

**User Story:** As a developer, I want events to be sent to an HTTP endpoint with fire-and-forget semantics, so that analytics delivery never blocks UI interactions or adds latency to the application.

#### Acceptance Criteria

1. WHEN an HTTP POST is sent to the configured endpoint, THE Dispatch_Pipeline SHALL include a `Content-Type: application/json` header and merge any key-value pairs from the configured `requestHeaders` into the request headers
2. WHEN an HTTP POST does not complete within 5000 milliseconds, THE Dispatch_Pipeline SHALL abort the request via `AbortController`
3. IF an HTTP POST encounters a network-level failure (including AbortController timeout, DNS failure, or connection refused) WHILE mode is not `production`, THEN THE Dispatch_Pipeline SHALL log a warning via `console.warn` and take no further action for that request
4. IF an HTTP POST encounters a network-level failure WHILE mode is `production`, THEN THE Dispatch_Pipeline SHALL fail silently without any console output
5. THE Dispatch_Pipeline SHALL NOT await or block on HTTP POST completion before returning control to the caller — `dispatch()` returns synchronously
6. IF no `endpoint` is configured in the AnalyticsConfig, THEN THE Dispatch_Pipeline SHALL skip HTTP delivery entirely and only push to the DataLayer

### Requirement 6: SPA Navigation Tracking via History API

**User Story:** As a developer using a non-Next.js SPA, I want page views to be tracked automatically on route changes, so that I don't need to manually fire events on every navigation.

#### Acceptance Criteria

1. WHEN `history.pushState` is called and the pathname has changed, THE History_Listeners SHALL dispatch a Page_View_Event with the new path and the previous path as referrer
2. WHEN `history.replaceState` is called and the pathname has changed, THE History_Listeners SHALL dispatch a Page_View_Event with the new path and the previous path as referrer
3. WHEN a `popstate` event fires and the pathname has changed, THE History_Listeners SHALL dispatch a Page_View_Event with the new path and the previous path as referrer
4. WHEN a navigation occurs via pushState, replaceState, or popstate but only the query string or hash has changed and the pathname remains the same, THE History_Listeners SHALL NOT dispatch a Page_View_Event
5. WHEN the cleanup function is called, THE History_Listeners SHALL restore the original `history.pushState` and `history.replaceState` methods and remove the `popstate` listener
6. IF `registerListeners()` is called while listeners are already active, THEN THE History_Listeners SHALL return the existing cleanup function without registering duplicate patches or event listeners

### Requirement 7: Custom Event Tracking

**User Story:** As a developer, I want to fire custom analytics events with arbitrary properties, so that I can track user interactions like button clicks, form submissions, and feature usage.

#### Acceptance Criteria

1. WHEN `track()` is called with a non-empty `name` string and an optional `properties` object after initialisation, THE Dispatch_Pipeline SHALL dispatch a Custom_Event through `beforeSend`, push an object to `window.dataLayer` with the `event` field set to the event name directly and all `properties` keys spread as top-level fields, and fire an HTTP POST to the configured endpoint if one is set
2. IF `track()` is called before `inject()` has been called, THEN THE Analytics_Package SHALL drop the event without throwing, without mutating `window.dataLayer`, and without firing an HTTP POST
3. IF `track()` is called before `inject()` has been called WHILE mode is not `production`, THEN THE Analytics_Package SHALL emit a `console.warn` indicating the event was dropped
4. IF `track()` is called when `typeof window === 'undefined'`, THEN THE Analytics_Package SHALL be a no-op that produces no side effects and does not throw
5. WHEN `track()` is called with a `properties` object, THE Dispatch_Pipeline SHALL only accept values of type `string`, `number`, `boolean`, or `null` — no nested objects — and the resulting `window.dataLayer` entry SHALL contain no nested object values

### Requirement 8: React Adapter

**User Story:** As a React developer using a non-Next.js application, I want a drop-in component that handles analytics lifecycle, so that I can add tracking without manual `useEffect` wiring.

#### Acceptance Criteria

1. WHEN the React_Adapter mounts, THE React_Adapter SHALL call `inject()` with the provided props as configuration inside a `useEffect` hook with an empty dependency array
2. WHEN the React_Adapter unmounts, THE React_Adapter SHALL call the cleanup function returned by `inject()` to remove all History API listeners
3. THE React_Adapter SHALL call `inject()` only once regardless of parent re-renders or prop changes
4. THE React_Adapter SHALL re-export the `track` function from the `@providirect/analytics/react` entry point for convenient single-import usage
5. THE React_Adapter SHALL render `null` — it produces no DOM output
6. WHEN the React_Adapter is rendered in an SSR environment, THE React_Adapter SHALL not throw and SHALL defer all side effects to the `useEffect` hook (which does not run on the server)

### Requirement 9: Next.js App Router Adapter

**User Story:** As a Next.js App Router developer, I want an adapter that uses `usePathname()` for navigation detection, so that page views are tracked reliably without double-firing caused by Next.js internal History API usage.

#### Acceptance Criteria

1. WHEN the NextJS_Adapter mounts, THE NextJS_Adapter SHALL call `setConfig()` with the provided AnalyticsConfig props, then dispatch an initial Page_View_Event where `url` is `window.location.href`, `path` is the current `usePathname()` value, and `referrer` is `document.referrer`
2. WHEN `usePathname()` returns a new value different from the previously stored path, THE NextJS_Adapter SHALL dispatch a Page_View_Event with `path` set to the new pathname, `url` set to `window.location.href`, and `referrer` set to the previous pathname
3. WHEN `usePathname()` returns the same value as the previously stored path, THE NextJS_Adapter SHALL NOT dispatch a Page_View_Event
4. THE NextJS_Adapter SHALL NOT call `registerListeners()` or register any History API monkey-patches
5. THE NextJS_Adapter SHALL re-export the `track` function for single-import usage
6. IF the NextJS_Adapter renders in an SSR environment where `typeof window === 'undefined'`, THEN THE NextJS_Adapter SHALL not dispatch any events and SHALL not throw

### Requirement 10: Server-side Event Tracking

**User Story:** As a back-end developer, I want to fire analytics events from Node.js or Edge runtimes, so that I can track server-side actions like authentication failures or API events.

#### Acceptance Criteria

1. WHEN `trackServer()` is called with a non-empty `endpoint` string and a non-empty `name` string, THE Server_Export SHALL fire an HTTP POST to the provided endpoint with a JSON body containing the `name`, `properties` (if provided), and a `ts` timestamp field
2. WHEN `trackServer()` fires an HTTP POST, THE Server_Export SHALL include a `Content-Type: application/json` header and merge any key-value pairs from the provided `headers` option into the request headers
3. WHEN the HTTP POST does not complete within 5000 milliseconds, THE Server_Export SHALL abort the request via `AbortController`
4. IF the HTTP POST encounters a network-level failure or abort timeout WHILE `mode` is not `production`, THEN THE Server_Export SHALL log a warning to `console.warn`
5. IF the HTTP POST encounters a network-level failure or abort timeout WHILE `mode` is `production`, THEN THE Server_Export SHALL produce zero console output
6. THE Server_Export SHALL return `void` synchronously without awaiting the HTTP POST — delivery is fire-and-forget
7. IF `trackServer()` is called with a missing or empty `endpoint` or a missing or empty `name`, THEN THE Server_Export SHALL not fire any HTTP request

### Requirement 11: DataLayer Event Format

**User Story:** As a GTM implementer, I want all dataLayer events to follow a consistent flat format with `pd_` prefixes, so that GTM triggers and variables can be configured reliably across all event types.

#### Acceptance Criteria

1. THE Dispatch_Pipeline SHALL use the event name directly as the dataLayer `event` field value without any prefix (e.g., `page_view`, `button_click`)
2. WHEN a Page_View_Event is pushed to the DataLayer, THE Dispatch_Pipeline SHALL produce an object with exactly the fields `event` (value `page_view`), `page_url` (current full URL), `page_path` (current pathname), and `page_referrer` (previous path for SPA navigation, or `document.referrer` on initial load, or empty string if neither is available)
3. WHEN a Custom_Event with name N and properties P is pushed to the DataLayer, THE Dispatch_Pipeline SHALL produce an object where the `event` field equals N, and every key-value pair from P is included as a top-level field; IF a property key in P equals `event`, THEN THE Dispatch_Pipeline SHALL discard that property key so the event name field is never overwritten
4. THE Dispatch_Pipeline SHALL NOT nest properties inside a sub-object — all values pushed to the DataLayer SHALL be primitives (string, number, boolean, or null) so that each is directly accessible via `{{DLV - key}}` in GTM
5. WHEN a Custom_Event is pushed to the DataLayer, THE Dispatch_Pipeline SHALL include only the `event` field and the caller-supplied property keys in the resulting DataLayer object — internal fields (`type`, `ts`) SHALL NOT appear in the pushed object

### Requirement 12: Production Error Silence

**User Story:** As a production operator, I want the analytics package to fail silently in production, so that analytics errors never surface in the browser console or disrupt the user experience.

#### Acceptance Criteria

1. WHILE the mode is set to `production`, THE Analytics_Package SHALL NOT produce any `console.warn`, `console.info`, or `console.error` output regardless of the error condition encountered (including `beforeSend` throwing, `dataLayer.push()` throwing, HTTP POST failing, or `track()` called before `inject()`)
2. IF any error occurs during event dispatch in production mode (including `beforeSend` throwing an exception or returning an invalid value, HTTP POST network failure, or HTTP POST timeout), THEN THE Dispatch_Pipeline SHALL catch the error without throwing to the caller and SHALL dispatch subsequent events through the full pipeline (beforeSend, dataLayer push, HTTP delivery) without requiring re-initialisation
3. IF `dataLayer.push()` throws in production mode, THEN THE Dispatch_Pipeline SHALL proceed with HTTP delivery for that same event and SHALL produce no console output
4. IF `track()` is called before `inject()` in production mode, THEN THE Analytics_Package SHALL drop the event without throwing and without producing any console output

### Requirement 13: Bundle Size and Tree-shaking

**User Story:** As a developer, I want the analytics package to be lightweight and tree-shakeable, so that my application bundle size stays minimal and unused adapters are eliminated during build.

#### Acceptance Criteria

1. THE Analytics_Package SHALL have zero runtime `dependencies` in `package.json` — only `peerDependencies` (for React/Next.js adapters) and `devDependencies` are permitted
2. THE Analytics_Package SHALL declare `"sideEffects": false` in `package.json` to enable tree-shaking by bundlers (webpack, esbuild, rollup)
3. THE Analytics_Package full client bundle (all entry points combined) SHALL be less than 10KB gzipped; an application importing only from `@providirect/analytics/nextjs` SHALL produce a tree-shaken bundle of less than 6KB gzipped
4. THE Analytics_Package SHALL provide separate entry points (`.`, `./react`, `./nextjs`, `./server`) via the `exports` map in `package.json` so bundlers can eliminate unused adapters via dead code elimination
5. THE Analytics_Package SHALL emit both ESM and CJS formats with TypeScript declaration files (`.d.ts`) for each entry point
