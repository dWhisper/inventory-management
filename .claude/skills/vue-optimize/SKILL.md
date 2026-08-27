---
name: vue-optimize
description: Analyze Vue component structure across client/src for performance and code-reuse opportunities, report findings, then apply approved fixes via vue-expert. Use when asked to audit, optimize, or review Vue component structure or reuse.
---

# Vue Component Structure Analysis

Structural audit of the Vue frontend — distinct from `/optimize` (whole-codebase dead-code cleanup) and `code-review`/`simplify` (diff-scoped correctness/simplification). This skill specifically targets component architecture: performance patterns and cross-file duplication that a generic diff review won't catch because the duplication spans files that weren't all touched in the same change.

## Scope

Analyze the whole frontend on every run, not just recently changed files:

- `client/src/views/*.vue`
- `client/src/components/*.vue`
- `client/src/composables/*.js`
- `client/src/utils/*.js` (to check whether views/components should be using these instead of reinventing them)

Skip `node_modules`, `dist`, build artifacts.

## Procedure

1. Read `client/src/composables/useFilters.js`, `useAuth.js`, `useI18n.js`, and `client/src/utils/currency.js` first — these are the shared primitives everything else should be reusing. Knowing what already exists is required before flagging something as "should be extracted."
2. Read every file in the scope above.
3. For each file, note structural issues per the two categories below with `file:line`.
4. Explicitly cross-reference across files for the reuse category — duplication findings are inherently multi-file; a pattern that appears in only one file is not a reuse finding.
5. Report findings with the `ReportFindings` tool. Use `category: "performance"` or `category: "reuse"`. Rank most-impactful first.
6. Ask the user which findings to act on — do not assume all findings should be fixed.
7. For approved findings, delegate implementation to the **vue-expert** subagent. Do not edit `.vue` files or `client/src/composables/*.js` directly from this skill — CLAUDE.md mandates vue-expert for any creation or significant modification of `.vue` files, and composables are also in vue-expert's scope. Give vue-expert the specific finding(s) (file:line + intended fix), batched by logical change so related edits land together.
8. After vue-expert completes, briefly summarize what changed and which findings were left unaddressed.

## What to look for: Performance

- `v-for` using array `index` as `:key` instead of a stable id (`sku`, `month`, order id, etc.)
- Derived values computed via a method call in the template (`{{ calculateTotal() }}`) instead of a `computed()` — recalculates on every render instead of caching
- A `watch()` doing what a `computed()` would do more simply and cheaply
- The same filter/reduce/aggregation logic recomputed by multiple separate `computed()` properties in one file instead of one shared computed that the others derive from
- Inline object/array/function literals passed as props in the template (e.g. `:options="{foo: 1}"`, `@click="() => doThing(x)"`) — creates a new reference every render, defeating child memoization
- `reactive()` or deep `ref()` wrapping a large read-only API response where a `shallowRef` would do
- Date parsing done inline at multiple call sites instead of validated once (per the "validate dates before `.getMonth()`" pattern in `vue-expert.md`)
- Chart/SVG coordinate data built inline in the template rather than via `computed()`

## What to look for: Code reuse

- Repeated `v-if="loading"` / `v-else-if="error"` / `v-else` markup copy-pasted across views instead of a shared loading/error wrapper component
- A view re-implementing filter logic that duplicates what `useFilters` already provides, instead of extending/reusing it
- Manual currency/number formatting instead of `client/src/utils/currency.js`
- Near-identical `*Modal.vue` open/close/prop scaffolding across `client/src/components/*Modal.vue` that could share a base pattern
- Repeated `loading.value = true / try / catch / finally` API-call scaffolding across views' `loadData()` functions — candidate for a shared composable
- Duplicate SVG chart-building logic across two or more views that could become one shared chart component

## Constraints

- Read-only except for `ReportFindings` and delegating to `vue-expert` — never `Edit`/`Write` `.vue` files or `client/src/composables/*.js` yourself from within this skill.
- Don't re-flag things `vue-expert.md`'s "Must-Know Patterns" already cover as house style (unique `v-for` keys, computed-over-method, never mutating props, date validation, no month filter on inventory) unless you find an actual violation — the goal is genuinely new findings, not restating existing rules.
- No emojis in findings or summaries (project convention).
