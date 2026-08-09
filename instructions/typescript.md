# TypeScript / React Coding Standards

See [file-headers.md](file-headers.md) for mandatory version header rules.

## File headers

TypeScript/TSX files use HTML comment format:

```typescript
<!-- file: web/src/components/MyComponent.tsx -->
<!-- version: 1.2.3 -->
<!-- guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx -->
<!-- last-edited: YYYY-MM-DD -->
```

Bump version and update `last-edited` on every change.

## General rules

- Prefer `const` over `let`; never use `var`.
- Use explicit return types on exported functions and React components.
- No `any` — use `unknown` + type narrowing or proper interfaces.
- Use `AbortController` for fetch cleanup in `useEffect`.

## React components

- Functional components only; no class components.
- Clean up subscriptions, timers, and event listeners in `useEffect` return.
- Use `useMemo` / `useCallback` only when profiling shows a measurable benefit.
- Keep components focused — split into smaller pieces if a component exceeds ~150 lines.

## Imports

Group in three blocks:

1. React and third-party libraries
2. Internal components and hooks
3. Types and utilities

## Error handling

- Never swallow errors silently — surface them to the user or log them.
- Use React error boundaries for component-level failures.
