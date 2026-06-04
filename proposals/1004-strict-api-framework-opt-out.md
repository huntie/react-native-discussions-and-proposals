---
title: Strict API framework opt-out via custom condition
author:
  - Alex Hunt
date: 2026-06-03
---

# RFC1004: Strict API framework opt-out via custom condition

## Summary

Provide a single custom condition — `"react-native-frameworks-private"` — that grants frameworks and libraries continued access to all internal `react-native` subpaths, giving framework authors a clear escape hatch while preserving the strict public API boundary for app developers.

## Basic example

```jsonc
// tsconfig.json (in the framework package)
{
  "compilerOptions": {
    "customConditions": ["react-native-frameworks-private"],
  },
}
```

```js
// With "react-native-frameworks-private" active, frameworks can import
// any internal module:
import HMRClient from "react-native/Libraries/Utilities/HMRClient";
import { PressabilityDebugView } from "react-native/Libraries/Pressability/PressabilityDebug";

// New src/private/ paths are also available as modules migrate:
import HMRClient from "react-native/private/devsupport/HMRClient";
```

```js
// Without the condition:
import HMRClient from "react-native/Libraries/Utilities/HMRClient"; // JS resolves, but TS error
import HMRClient from "react-native/private/devsupport/HMRClient"; // Fails entirely
```

## Motivation

React Native is [moving towards a stable JavaScript API](https://reactnative.dev/blog/2025/06/12/moving-towards-a-stable-javascript-api) — deprecating deep imports and introducing the `"react-native-strict-api"` condition ([RFC0894](https://github.com/react-native-community/discussions-and-proposals/pull/894)). Once the Strict TypeScript API is enforced by default, deep imports from `react-native/Libraries/...` and `react-native/src/...` will stop resolving.

Framework authors and ecosystem libraries legitimately need access to internals — HMRClient for custom bundlers, PressabilityDebugView for gesture libraries, renderer shims for navigation libraries. Stabilizing and exposing each API individually doesn't scale: every internal API needs its own subpath export, type surface, and stability commitment.

**Who's affected:**

- [Expo](https://github.com/expo/expo) — Broad access to internals for dev tooling, error handling, and platform integration
- [Re.pack](https://github.com/callstack/repack) — `HMRClient` for Hot Module Replacement
- [React Native Gesture Handler](https://github.com/software-mansion/react-native-gesture-handler) — `PressabilityDebugView`, `customDirectEventTypes`, `findHostInstance_DEPRECATED`
- [React Native Screens](https://github.com/software-mansion/react-native-screens) — `ReactNativeStyleAttributes`, `AppContainer`
- [React Native Reanimated](https://github.com/software-mansion/react-native-reanimated) — `findHostInstance_DEPRECATED`, `getInternalInstanceHandleFromPublicInstance`
- Community tooling — `parseErrorStack`, `symbolicateStackTrace`, `openURLInBrowser`

See the [Community Requested Root Exports spreadsheet](https://docs.google.com/spreadsheets/d/1_bX6Rgz4BgOmpm0XFZPUf306jDVmsFJqcQ4ONYWBKhk/edit?gid=0#gid=0) for the full list.

## Detailed design

### Package layout context

React Native's JavaScript source is being reorganized from `Libraries/` into `src/private/`:

```
packages/react-native/
├── index.js                  # Public entry point
├── Libraries/                # Legacy layout
│   ├── Utilities/
│   ├── Core/Devtools/
│   ├── Pressability/
│   └── ...
└── src/
    └── private/              # Internal modules (new location)
        ├── animated/
        ├── components/
        ├── devsupport/
        └── ...
```

### Custom condition: `"react-native-frameworks-private"`

Add two subpath exports to `react-native/package.json`, gated behind the `"react-native-frameworks-private"` condition:

```jsonc
{
  "exports": {
    // Stable public API (near-future state)
    ".": {
      "types": "./types_generated/index.d.ts",
      "default": "./index.js",
    },
    // NEW: maps to Libraries/ — JS already resolvable, types hidden
    "./Libraries/*": {
      "types": {
        "react-native-frameworks-private": "./types_generated/Libraries/*.d.ts",
        "default": null,
      },
      "default": "./Libraries/*.js",
    },
    // NEW: maps to src/private/ — condition required at all layers
    "./private/*": {
      "react-native-frameworks-private": {
        "types": "./types_generated/src/private/*.d.ts",
        "default": "./src/private/*.js",
      },
      "default": null,
    },
    // ... other existing entries
  },
}
```

The two subpath patterns behave differently:

- **`./Libraries/*`** — Already resolvable at runtime (the JS files exist on disk). The condition only gates type resolution — without it, TypeScript reports errors but the bundler still resolves the JS files. This matches the Strict TypeScript API's enforcement model: types-only gating.
- **`./private/*`** — A new subpath routed to `src/private/`. The `"react-native-frameworks-private"` condition gates both type resolution and runtime resolution. Without the condition, the import fails entirely (resolves to `null`).

**Important:** `types_generated/` only contains `.d.ts` files for modules reachable from the public API's type graph. Many internal modules will have no generated type definition. Frameworks should expect incomplete type coverage and may need to use `@ts-ignore` or author their own type declarations for paths without definitions.

### Contract

1. **No stability guarantees.** Exports under `/private/` and `/Libraries/` can change, move, or be removed in any release.
2. **No type guarantees.** Modules may be untyped or have incomplete type definitions.
3. **Framework responsibility.** Consumers accept responsibility for tracking changes across React Native releases.
4. **Not for app developers.** The condition is intended for frameworks and libraries only.

### How frameworks opt in

Frameworks activate the condition in their `tsconfig.json` and bundler configuration. The `tsconfig.json` condition is scoped to the framework's own package — end users do not inherit it.

```jsonc
// tsconfig.json (in the framework package)
{
  "compilerOptions": {
    "customConditions": ["react-native-frameworks-private"],
  },
}
```

For `./Libraries/*` imports, `tsconfig.json` is sufficient — the JS files already resolve at runtime. For `./private/*` imports, the bundler must also be configured, since `./private/*` is fully gated:

```js
// metro.config.js (only needed for ./private/* imports)
module.exports = {
  resolver: {
    unstable_conditionNames: ["react-native-frameworks-private", "import"],
  },
};
```

### Migration path

1. **Before Strict API enforcement:** Frameworks adopt `"react-native-frameworks-private"`. Existing `Libraries/` deep imports work immediately via the `./Libraries/*` export — same paths, just routed through the exports field.
2. **At enforcement:** `"react-native-strict-api"` blocks all non-public imports. Frameworks with the condition active retain access.
3. **Over time:** As modules move to `src/private/`, frameworks migrate imports from `react-native/Libraries/...` to `react-native/private/...` incrementally.

## Drawbacks

- **Potential for misuse.** App developers may activate the condition to access internals, then file bugs when things break.
- **No incremental stability signal.** Everything behind the escape hatch carries the same "unstable" label, unlike named subpath exports.
- **No type coverage.** Frameworks using these imports lose TypeScript type safety unless they maintain their own declarations.
- **Coupled to internal directory structure.** Import paths mirror the filesystem layout — internal reorganization will break framework imports.

## Alternatives considered

### Individual subpath exports (e.g. `react-native/devsupport`)

Expose curated sets of internal APIs behind named subpath exports. Creating partial boundaries between "public" and "framework-private" APIs adds complexity without meaningful protection — frameworks need broad access, and halfway restrictions just push them back to workarounds.

## Unresolved questions

- Should the condition name include a stability prefix (e.g. `"react-native-unstable-private"`)?
- Linting or tooling to warn app developers who activate the condition without being a framework.
