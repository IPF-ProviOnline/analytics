# Implementation Plan: @providirect/analytics (GTM Rewrite)

## Overview

Implement a lightweight TypeScript analytics package that replaces the existing GTM-based tracking with a clean dispatch pipeline. The package provides a singleton config store, unified event dispatch (dataLayer push + HTTP POST), and framework adapters for React, Next.js App Router, and server-side runtimes. Implementation uses tsup for bundling, vitest + jsdom for testing, and fast-check for property-based tests.

## Tasks

- [ ] 1. Set up project structure, types, and build configuration
  - [ ] 1.1 Create core type definitions and package configuration
    - Create `src/types.ts` with `AnalyticsConfig`, `AnalyticsEvent` (discriminated union: `PageViewEvent | CustomEvent`), `ServerTrackOptions`, and dataLayer shape types
    - Update `package.json`: rename to `@providirect/analytics`, set `"sideEffects": false`, remove unused framework peer deps (nuxt, svelte, vue, remix, astro), keep only react and next as optional peer deps, add `fast-check` as devDependency
    - Update `exports` map to expose `.`, `./react`, `./nextjs`, `./server` entry points with browser/import/require conditions
    - Update `tsup.config.js` to build only the four entry points: `src/index.ts`, `src/react/index.tsx`, `src/nextjs/index.tsx`, `src/server/index.ts`
    - Update `typesVersions` to match the new entry points
    - _Requirements: 13.1, 13.2, 13.4, 13.5_

- [ ] 2. Implement core state and pipeline modules
  - [ ] 2.1 Implement singleton config store (`src/state.ts`)
    - Implement `setConfig()`, `getConfig()`, and `_resetConfig()` at module scope
    - `setConfig()` stores config on first call; subsequent calls are ignored with dev-mode warning
    - `getConfig()` returns stored config or `null`
    - `_resetConfig()` clears the store for test isolation
    - _Requirements: 1.1, 1.6, 2.1, 2.2, 2.3, 2.4, 2.5_

  - [ ]* 2.2 Write property test: Idempotent initialisation (Property 1)
    - **Property 1: Idempotent initialisation**
    - For any two config objects, calling `setConfig(config1)` then `setConfig(config2)` results in `getConfig()` returning config1
    - **Validates: Requirements 2.1, 2.3**

  - [ ] 2.3 Implement dispatch pipeline (`src/pipeline.ts`)
    - Implement `dispatch()` function: applies `beforeSend`, pushes to `window.dataLayer`, logs in dev mode, fires HTTP POST
    - Implement `toDataLayerShape()`: converts `AnalyticsEvent` to flat GTM-compatible object
    - Implement `postEvent()`: fire-and-forget HTTP POST with 5s AbortController timeout
    - All errors are caught internally — `dispatch()` never throws
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5, 3.6, 3.7, 3.8, 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7, 5.1, 5.2, 5.3, 5.4, 5.5, 5.6, 11.1, 11.2, 11.3, 11.4, 11.5, 12.1, 12.2, 12.3_

  - [ ]* 2.4 Write property test: beforeSend drop semantics (Property 3)
    - **Property 3: beforeSend drop semantics**
    - For any event, if `beforeSend` returns null/undefined or throws, dataLayer length is unchanged and no HTTP POST fires
    - **Validates: Requirements 4.2, 4.3, 4.4**

  - [ ]* 2.5 Write property test: beforeSend transformation (Property 4)
    - **Property 4: beforeSend transformation**
    - For any event and any `beforeSend` that returns a modified event, the dataLayer contains the transformed event (not the original)
    - **Validates: Requirements 4.1**

  - [ ]* 2.6 Write property test: Event name pass-through (Property 5)
    - **Property 5: Event name pass-through**
    - For any event dispatched, the dataLayer object's `event` field equals the event name directly without prefix
    - **Validates: Requirements 3.4, 11.1**

  - [ ]* 2.7 Write property test: Flat properties invariant (Property 6)
    - **Property 6: Flat properties invariant**
    - For any CustomEvent with any properties dispatched, all values in the dataLayer object are primitives (no nested objects)
    - **Validates: Requirements 3.5, 11.4**

  - [ ]* 2.8 Write property test: Custom event dataLayer shape (Property 12)
    - **Property 12: Custom event dataLayer shape**
    - For any CustomEvent with name N and properties P, the dataLayer object has `event === N` and contains every key from P as a top-level field
    - **Validates: Requirements 11.2, 11.3**

  - [ ]* 2.9 Write property test: Production silence (Property 9)
    - **Property 9: Production silence**
    - For any error condition occurring while mode is `production`, zero console output is produced
    - **Validates: Requirements 4.5, 5.4, 10.5, 12.1**

  - [ ]* 2.10 Write property test: Dispatch-without-config is no-op (Property 11)
    - **Property 11: Dispatch-without-config is no-op**
    - For any valid AnalyticsEvent, calling `dispatch()` when `getConfig()` returns null produces no side effects
    - **Validates: Requirements 3.3**

- [ ] 3. Implement navigation listeners and entry points
  - [ ] 3.1 Implement History API listeners (`src/listeners.ts`)
    - Implement `registerListeners()` that monkey-patches `history.pushState`, `history.replaceState`, and adds `popstate` listener
    - Deduplicate by comparing `window.location.pathname` to stored `_previousPath`
    - Return a cleanup function that restores originals
    - Guard against double-registration (return existing cleanup if already active)
    - _Requirements: 1.3, 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [ ]* 3.2 Write property test: Path deduplication (Property 7)
    - **Property 7: Path deduplication**
    - For any sequence of navigations where consecutive pathnames are identical, no Page_View_Event is dispatched for the repeated path
    - **Validates: Requirements 6.4, 9.3**

  - [ ]* 3.3 Write property test: Cleanup restores History API (Property 8)
    - **Property 8: Cleanup restores History API**
    - After calling cleanup, `history.pushState` and `history.replaceState` equal their original references and popstate listener is removed
    - **Validates: Requirements 6.5**

  - [ ] 3.4 Implement root entry point (`src/inject.ts`)
    - Implement `inject()`: SSR guard, call `setConfig()`, dispatch initial `page_view`, call `registerListeners()`, return cleanup
    - If SSR: return no-op function without side effects
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_

  - [ ]* 3.5 Write property test: SSR safety (Property 2)
    - **Property 2: SSR safety**
    - For any config and event, calling `inject()` or `track()` when `typeof window === 'undefined'` is a no-op that never throws
    - **Validates: Requirements 1.5, 7.4**

  - [ ] 3.6 Implement custom event API (`src/track.ts`)
    - Implement `track()`: SSR guard, verify config exists (warn in dev if not), dispatch custom event through pipeline
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

  - [ ]* 3.7 Write property test: Track-before-inject safety (Property 10)
    - **Property 10: Track-before-inject safety**
    - For any event name and properties, calling `track()` before `inject()` drops the event without throwing
    - **Validates: Requirements 7.2, 3.3**

  - [ ] 3.8 Create barrel export (`src/index.ts`)
    - Export `inject` and `track` from the root entry point
    - _Requirements: 13.4_

- [ ] 4. Checkpoint - Ensure all core tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 5. Implement framework adapters
  - [ ] 5.1 Implement React adapter (`src/react/index.tsx`)
    - Create `Analytics` component that calls `inject()` in `useEffect([], [])` and returns cleanup on unmount
    - Re-export `track` from `../track`
    - Component renders `null`
    - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5, 8.6_

  - [ ] 5.2 Implement Next.js App Router adapter (`src/nextjs/index.tsx`)
    - Create `Analytics` component using `'use client'` directive
    - Call `setConfig()` directly (NOT `inject()` — avoids History API registration)
    - Dispatch initial `page_view` on mount
    - React to `usePathname()` changes via `useRef` for deduplication
    - Re-export `track` from `../track`
    - _Requirements: 9.1, 9.2, 9.3, 9.4, 9.5, 9.6_

  - [ ] 5.3 Implement server export (`src/server/index.ts`)
    - Implement `trackServer()`: fire HTTP POST with 5s timeout, fire-and-forget, dev-mode console.warn on failure
    - Guard against missing/empty endpoint or name (no-op)
    - _Requirements: 10.1, 10.2, 10.3, 10.4, 10.5, 10.6, 10.7_

  - [ ]* 5.4 Write unit tests for framework adapters
    - Test React adapter mounts/unmounts correctly and calls inject() once
    - Test Next.js adapter dispatches page_view on pathname change and does NOT register History listeners
    - Test server trackServer() fires HTTP POST with correct payload and headers
    - Test server trackServer() is silent on failure in production mode
    - _Requirements: 8.1, 8.2, 8.3, 9.1, 9.2, 9.4, 10.1, 10.5_

- [ ] 6. Checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 7. Final wiring, build verification, and bundle size check
  - [ ] 7.1 Wire all entry points and verify build
    - Run `tsup` build and verify all four entry points produce ESM + CJS + `.d.ts` output
    - Verify `exports` map resolves correctly for each entry point
    - Ensure no circular dependencies between modules
    - _Requirements: 13.4, 13.5_

  - [ ]* 7.2 Write integration tests for full pipeline
    - Test full flow: inject → track → verify dataLayer contains both page_view and custom event
    - Test HTTP delivery: mock fetch, verify payload structure matches design
    - Test cleanup: after calling cleanup function, navigation events no longer fire
    - _Requirements: 1.1, 1.2, 1.4, 3.1, 3.2, 7.1_

- [ ] 8. Final checkpoint - Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Checkpoints ensure incremental validation
- Property tests validate universal correctness properties from the design document using fast-check
- Unit tests validate specific examples and edge cases using vitest + jsdom
- The existing vitest.config.mts already configures jsdom environment — tests can use the existing setup
- The `_resetConfig()` function is test-only and should be used in `beforeEach` blocks for isolation

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["2.1"] },
    { "id": 2, "tasks": ["2.2", "2.3"] },
    { "id": 3, "tasks": ["2.4", "2.5", "2.6", "2.7", "2.8", "2.9", "2.10", "3.1", "3.6"] },
    { "id": 4, "tasks": ["3.2", "3.3", "3.4", "3.7", "3.8"] },
    { "id": 5, "tasks": ["3.5", "5.1", "5.2", "5.3"] },
    { "id": 6, "tasks": ["5.4", "7.1"] },
    { "id": 7, "tasks": ["7.2"] }
  ]
}
```
