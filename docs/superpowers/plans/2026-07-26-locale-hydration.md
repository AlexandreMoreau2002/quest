# Locale Hydration Mismatch Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prevent the Next.js hydration mismatch caused by reading the persisted locale before the first client render.

**Architecture:** Keep `useLocale` on `useSyncExternalStore`, but provide the same French snapshot to SSR and the first client render. Let the browser snapshot switch to the validated `localStorage` value after hydration, then keep the existing explicit locale mutation flow.

**Tech Stack:** Next.js 16, React 19, TypeScript, i18next, Vitest, Testing Library.

## Global Constraints

- Preserve the existing `fr` default and `en` persistence behavior.
- Do not change translation dictionaries or the `I18nProvider` public API.
- Keep imports grouped and ordered from smallest to largest visually.
- Run web tests, lint, and build before claiming completion.
- Do not revert unrelated working-tree changes in `api`, `web`, or the repository root.

---

### Task 1: Make locale snapshots hydration-safe

**Files:**
- Modify: `web/src/hooks/use-locale.ts`
- Modify: `web/src/hooks/use-locale.test.ts`

**Interfaces:**
- Consumes: `SupportedLocale`, `DEFAULT_LOCALE`, and the existing locale store API.
- Produces: `useLocale()` with the same `{ locale, setLocale }` return shape; SSR and first client snapshot both return `DEFAULT_LOCALE`.

- [ ] **Step 1: Write the failing regression test**

Add a test that stores `en`, renders the hook with a server snapshot override, and asserts the server snapshot is `fr` while the hydrated client eventually reflects `en`. Keep the existing persistence and invalid-value tests unchanged.

```typescript
it('keeps the server snapshot in French when English is persisted', () => {
  localStorage.setItem('quest-locale', 'en');
  const { result } = renderHook(() => useLocale(), {
    serverHydration: true,
  });

  expect(result.current.locale).toBe('fr');
});
```

If the installed Testing Library version cannot expose a server snapshot through `renderHook`, test the store contract through a small exported test-only snapshot helper instead; do not weaken the assertion to only test the post-hydration value.

- [ ] **Step 2: Run the focused test and verify it fails for the current implementation**

Run: `npm test -- --run web/src/hooks/use-locale.test.ts`

Expected: the new regression test fails because `readStoredLocale()` currently returns `en` on the client render.

- [ ] **Step 3: Implement the minimal snapshot change**

Split the current browser read into a client snapshot function and make the `useSyncExternalStore` third argument the server snapshot already used by the theme hook. The browser snapshot must continue validating `localStorage` and the setter must continue notifying listeners.

```typescript
function getServerLocale(): SupportedLocale {
  return DEFAULT_LOCALE;
}

const locale = useSyncExternalStore(subscribe, readStoredLocale, getServerLocale);
```

If React requires a stable first client snapshot for hydration in the current implementation, introduce a `hasHydrated` store flag and set it from the subscription callback, while keeping the first client value equal to `DEFAULT_LOCALE`; do not read `window` from render-time branching outside the store contract.

- [ ] **Step 4: Run focused tests and type/lint checks**

Run: `npm test -- --run src/hooks/use-locale.test.ts`

Expected: all locale tests pass.

Run: `npm run lint`

Expected: ESLint exits with code 0.

- [ ] **Step 5: Review the diff and commit the web fix**

Run: `git diff --check && git diff -- src/hooks/use-locale.ts src/hooks/use-locale.test.ts`

Then commit only the web files:

```bash
git add src/hooks/use-locale.ts src/hooks/use-locale.test.ts
git commit -m "fix(web): avoid locale hydration mismatch"
```

### Task 2: Add delivery documentation for the fix

**Files:**
- Create: `docs/locale-hydration-mismatch/fonctionnement.md`
- Create: `docs/locale-hydration-mismatch/guide-test.md`
- Create: `http/locale-hydration-mismatch.http`

**Interfaces:**
- Consumes: The implemented `useLocale` behavior and the web validation commands.
- Produces: A child-friendly technical explanation, a manual browser checklist, and a REST Client file documenting that this UI-only fix has no HTTP endpoint.

- [ ] **Step 1: Write the feature explanation**

Document the server/client flow with an ASCII diagram, the root cause, the corrected snapshot sequence, affected files, and the fact that no API endpoint changes.

- [ ] **Step 2: Write the manual test guide**

Include prerequisites, clearing and setting `quest-locale`, a French first-load scenario, an English persisted-locale scenario, language toggle behavior, refresh behavior, and a final checklist that explicitly checks the browser console for hydration warnings.

- [ ] **Step 3: Write the HTTP file**

Include a comment explaining that the fix is frontend-only and therefore has no endpoint to call, plus placeholder environment variables and a health request for the existing local API as an optional prerequisite.

- [ ] **Step 4: Verify documentation files**

Run: `git diff --check`

Expected: no whitespace errors and all three files exist at the exact paths above.

### Task 3: Whole-branch verification

**Files:**
- Verify: `web/src/hooks/use-locale.ts`
- Verify: `web/src/hooks/use-locale.test.ts`
- Verify: `docs/locale-hydration-mismatch/fonctionnement.md`
- Verify: `docs/locale-hydration-mismatch/guide-test.md`
- Verify: `http/locale-hydration-mismatch.http`

- [ ] **Step 1: Run the complete web validation**

Run from `web/`: `npm run validate`

Expected: Vitest, ESLint, and `next build` all exit with code 0.

- [ ] **Step 2: Confirm only intended changes are present**

Run from the repository root: `git diff --check` and `git status --short`.

Expected: the web submodule contains only the locale fix commit beyond its previous state, and the root contains only the three documentation files plus the already-existing unrelated user changes.
