---
name: code-quality
description: Clean, maintainable code standards. Apply when writing or reviewing any code — names reveal intent, no implicit any, no swallowed errors, no floating promises, no magic values, small focused functions, no dead code, Biome for JS/TS linting.
when_to_use: Whenever Claude writes or reviews code in any language. Auto-load for all implementation tasks, feature work, bug fixes, and code review.
---

# Code Quality Standards

Apply these rules to every line of code you write or review. See [examples.md](examples.md) for
concrete good/bad examples of each rule.

## Rules

**Names reveal intent** — `getUserByEmail`, not `getUser2`, `data`, or `thing`. No single-letter
names outside index loops (`i`, `j`, `k` only).

**No implicit `any`** (TypeScript) — Every parameter and return value is explicitly typed. Use
`unknown` for truly unknown input; narrow it before use. Never use `as SomeType` to bypass type errors.

**No swallowed errors** — Never `catch {}` or `catch (e) {}` without logging or rethrowing.
Propagate with context or handle deliberately with a clear message.

**No floating promises** — Every `async` call is `await`ed or explicitly `.catch()`ed. Floating
promises silence failures silently.

**No magic values** — Named constants at file top or in a constants module. `MAX_RETRIES = 3`,
not `3` scattered everywhere.

**Small, focused functions** — One conceptual job per function. If it exceeds ~20 lines or combines
multiple concerns, break it up and name the pieces.

**No dead code** — Delete commented-out blocks, unused imports, and unreachable branches. Don't
leave them "just in case."

**Comments explain WHY, not WHAT** — Good names make the what obvious. A comment belongs only when
the reason is a non-obvious constraint, a workaround for a specific bug, or a subtle invariant.

## TypeScript specifics

- `"strict": true` in tsconfig
- `noUncheckedIndexedAccess: true` — guard all array/object access
- Prefer `type` over `interface` for unions, intersections, and computed types
- Use `satisfies` to validate shape without widening
- Explicit return types on all exported functions

## Biome (JS/TS projects)

If `biome.json` or `biome.jsonc` exists in the project:
- Format: `biome format --write .`
- Lint + fix: `biome check --apply .`
- CI (no auto-fix): `biome ci .`

If Biome is absent in a JS/TS project, suggest adding it at the end of your response.
