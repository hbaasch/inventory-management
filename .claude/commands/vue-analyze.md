---
description: Analyze Vue component structure and suggest performance and code-reuse optimizations
---

Analyze Vue 3 component(s) for performance issues and code-reuse opportunities.

**Target:** `$ARGUMENTS` (a `.vue` file path, a directory, or empty to analyze all `client/src/views/*.vue`)

## Step 1 — Discover files

If `$ARGUMENTS` is a specific `.vue` file, analyze only that file.
If `$ARGUMENTS` is a directory, find all `.vue` files inside it.
If `$ARGUMENTS` is empty, analyze all files matching `client/src/views/*.vue` and `client/src/components/**/*.vue`.

Use `find` or `glob` to collect the file list.

## Step 2 — Read and analyze each component

For each file, read the full contents and check for the following:

### Performance issues

| ID | Check | What to look for |
|----|-------|-----------------|
| P1 | `v-for` key quality | `v-for` without `:key`, or `:key="index"` — should use a stable unique id |
| P2 | `v-if` + `v-for` on same element | Should split onto a wrapper element; `v-for` always wins and wastes `v-if` checks |
| P3 | Template expressions that should be `computed` | Expressions with method calls, filters, or array operations (`.filter`, `.map`, `.sort`) directly in `{{ }}` or `:bind` |
| P4 | Inline arrow functions on repeated elements | `@click="() => handler(item)"` inside a `v-for` — creates a new function per render |
| P5 | Watchers vs computed | `watch` that only derives a value from another ref — should be `computed` |
| P6 | Missing `shallowRef` for large static data | Large arrays/objects assigned to `ref()` that are never mutated deeply |
| P7 | Synchronous expensive watchers | `watch` without `{ lazy: true }` or debounce on frequently-changing refs |
| P8 | Missing `defineAsyncComponent` | Large components imported statically that are only sometimes rendered (e.g., modals, detail panels) |

### Code-reuse opportunities

| ID | Check | What to look for |
|----|-------|-----------------|
| R1 | Duplicated `ref` / `reactive` state | Same state shape (e.g., `loading`, `error`, `data` trio) declared in multiple components |
| R2 | Duplicated API call patterns | `onMounted` + `api.getSomething()` + error handling repeated across files |
| R3 | Duplicate `computed` logic | Same derivation (status badge color, currency formatting, date parsing) copy-pasted |
| R4 | Repeated template blocks | Structurally identical template sections that could be extracted to a sub-component or slot |
| R5 | Options API in a `<script setup>` codebase | Inconsistent API style — should migrate to Composition API |
| R6 | Missing prop validation | Props without `type`, `required`, or `default` |
| R7 | Undeclared emits | `$emit('event')` calls not listed in `defineEmits` |

## Step 3 — Cross-component analysis

After reading all files, compare them to find:
- State or logic patterns that appear in 2+ components (candidates for composables in `client/src/composables/`)
- Template patterns repeated across views (candidates for shared components)

## Step 4 — Output a structured report

Format the report as follows:

```
## Vue Component Analysis

### Summary
- Files analyzed: N
- Issues found: N (P: N performance, R: N reuse)

### Per-file findings

#### <filename>
- [P1] <line number or template location>: <specific problem> → <suggested fix>
- [R2] <description>

### Cross-component findings
- <pattern> found in: <file1>, <file2> → extract to `useXxx` composable in `client/src/composables/useXxx.js`

### Recommended composables to create
| Composable | Encapsulates | Used by |
|------------|-------------|---------|

### Quick wins (fixes that take < 10 min each)
1. ...

### Larger refactors (prioritized by impact)
1. ...
```

Focus on findings that have clear, actionable fixes. Skip issues that are subjective style preferences. If a file has no issues, say so explicitly.
