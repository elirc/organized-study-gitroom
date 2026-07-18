# Frontend Framework Question Cards

Twelve cards on the repo's React 19 / Next 16 / SWR / zustand stack. Answer aloud, lead with the repo example.

---

## Q1: "When do you use SWR/React Query vs a store like zustand vs useState?"
Testing: server-state vs client-state distinction.
Anchor: SWR for server data ([calendar.context.tsx](../../../apps/frontend/src/components/launches/calendar.context.tsx#L1-L50)); zustand for the composer ([store.ts](../../../apps/frontend/src/components/new-launch/store.ts#L28-L60)); `useState` in leaves.
Junior: "SWR for fetching, zustand for global state."
Mid: server state is *owned by the server* — you cache and revalidate it (SWR), you don't "set" it; client state you own (zustand for complex interlocked, useState for local). This repo splits them cleanly.
Senior: names the failure of conflating them (putting server data in zustand → manual cache invalidation hell) and the cost of the module-level zustand store (singleton — two composers can't coexist, [store.ts](../../../apps/frontend/src/components/new-launch/store.ts#L1-L5)).
Follow-up: "how does SWR revalidate?" (focus, interval, `mutate`).
Drill: point at one `mutate()` call after a write ([launches.component L374-L394](../../../apps/frontend/src/components/launches/launches.component.tsx#L353-L394)) and explain what it re-fetches.

## Q2: "This app is Next.js but almost every component is 'use client'. Is that wrong?"
Testing: RSC understanding + judgment.
Anchor: [(site)/layout.tsx](../../../apps/frontend/src/app/(app)/(site)/layout.tsx) server component wrapping a client tree; data via SWR not RSC.
Junior: "they should use server components."
Mid: RSC shines for static/server-fetched content; this is a highly interactive, auth-gated dashboard where nearly everything needs interactivity and talks to a *separate* NestJS API — RSC buys little. Legitimate choice; the cost is bundle size.
Senior: what they'd still push server-side (initial calendar payload for first paint, auth redirect) and the migration cost; notes the app treats Next as a shell, not a data layer.
Drill: find one thing that *could* be a server component and one that can't.

## Q3: "Explain the rules of hooks. Show me where they shaped this code."
Testing: hooks mental model.
Anchor: the staged-mount composer ([add.edit.modal.tsx L36-L107](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L107)) — three nested components, each gating on store readiness; [CLAUDE.md's one-SWR-hook-per-function rule](../../../CLAUDE.md#L43-L58).
Junior: "don't call hooks conditionally."
Mid: hooks rely on stable call order; conditional/looped calls break the fiber's hook list. The composer can't conditionally run the hooks for a not-yet-ready stage, so it splits into components that mount only when ready — **the rule shaped the component tree**.
Senior: contrasts with the anti-pattern CLAUDE.md forbids (returning an object of `() => useSWR(...)`), and explains why an early `return null` before hooks is itself a violation risk.
Drill: explain why the composer uses three components instead of `if (!ready) return null` then hooks.

## Q4: "What's a stale closure and where's the risk here?"
Testing: effects + closures.
Anchor: empty-dep init effects in [add.edit.modal.tsx L48-L53, L135-L212](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L48-L212) — capture props once at mount.
Junior: "old value captured in a closure."
Mid: `useEffect(fn, [])` closes over the first render's props/state; if a prop changes later, the effect won't see it (here: a late-arriving `set` prop is ignored). Zustand mitigates by reading via `getState`/selectors rather than closed-over values.
Senior: when this is a bug vs intentional (init-only is deliberate here) and how to fix real cases (deps + cleanup, or refs).
Drill: predict what happens if `props.date` changes after mount.

## Q5: "How does this app keep re-renders cheap in a big form?"
Testing: render performance.
Anchor: `useShallow` selectors ([add.edit.modal L37-L47](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L36-L47)) subscribe to slices, not the whole store.
Junior: "memoization."
Mid: a zustand selector re-renders only when its selected slice changes; `useShallow` prevents re-render when a new object with equal fields is returned. Without it, every keystroke updating the store would re-render every subscriber.
Senior: measures with the Profiler; notes `React.memo`/`useMemo` are complementary; the calendar month view's row keying as a separate concern.
Drill: what re-renders when the user types in one channel's editor? (Ideally just that channel's slice.)

## Q6: "How is auth carried on the frontend?"
Testing: cookies, credentials, SSR boundaries.
Anchor: [custom.fetch.func.ts](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L16-L74) — `credentials: 'include'`, cookie fallback to header, `typeof document` guards.
Junior: "cookies."
Mid: the wrapper sends the `auth` cookie (or reads it into a header for non-secured/http self-host), plus `showorg`/`impersonate`; SSR path skips document access. One configured fetch in React context ([custom.fetch.tsx](../../../libraries/helpers/src/utils/custom.fetch.tsx#L14-L46)).
Senior: the never-resolving-promise on 401 ([L83-L85](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L76-L85)) and its trade-off (clean redirect vs hung callers); why cookie flags depend on `NOT_SECURED`.
Drill: trace one request's headers from `useFetch()` to the wire.

## Q7: "Controlled vs uncontrolled, and how does a rich-text editor fit?"
Testing: form state.
Anchor: tiptap editor producing HTML into the zustand store ([store values are HTML strings](../../../apps/frontend/src/components/new-launch/store.ts#L9-L14)); react-hook-form is a dep used elsewhere.
Mid: the editor is effectively controlled via the store; content is HTML, later stripped server-side per provider ([stripHtmlValidation](../../../apps/orchestrator/src/activities/post.activity.ts#L229-L235)).
Senior: the coupling — storing HTML means the server must sanitize + strip; an alternative is storing a structured doc (tiptap JSON) and rendering per-platform. Trade-off: portability vs simplicity.
Drill: where is the content sanitized, and is it before or after it hits the store?

## Q8: "How would you add optimistic UI to scheduling a post?"
Testing: SWR mutate patterns.
Anchor: current flow re-fetches via `mutate` after the POST ([launches.component](../../../apps/frontend/src/components/launches/launches.component.tsx#L353-L394)).
Mid: `mutate(optimisticData, { revalidate: true })` — show the post immediately, roll back on error; the calendar key must match.
Senior: reconciliation with the server's assigned id/group; the failure case (workflow start swallowed server-side means "scheduled" might not be true — optimistic UI can *over*-promise here).
Drill: what does the optimistic entry look like before the server returns `postId`/`group`?

## Q9: "Accessibility: what would you check in the composer/calendar?"
Testing: a11y baseline.
Anchor: `eslint-plugin-jsx-a11y` is configured ([package.json L292](../../../package.json#L292)); `Media.alt` exists but surfacing is thin (ticket J14).
Mid: keyboard nav for the calendar grid, focus management in modals, alt text for media, ARIA on custom controls, color contrast (the app has a dark theme).
Senior: names automated (axe/eslint) vs manual (screen reader) gaps; ties alt-text to a real ticket.
Drill: pick the modal and list three a11y checks.

## Q10: "Internationalization — how's it done and what breaks?"
Testing: i18n awareness.
Anchor: `i18next`/`react-i18next` ([package.json L174-L212](../../../package.json#L173-L215)), `i18n.json`/`i18n.lock`, `tailwindcss-rtl` for RTL.
Mid: keys + resource bundles + language detection; RTL support present.
Senior: pluralization/interpolation pitfalls, date/number locale vs the fixed-offset timezone issue, translation drift (i18n.lock hints at a sync process).
Drill: how would you add a language, and what could go stale?

## Q11: "The custom fetch returns a never-resolving Promise on auth failure. Frontend consequences?"
Testing: promise lifetime in UI.
Anchor: [custom.fetch.func.ts L76-L85](../../../libraries/helpers/src/utils/custom.fetch.func.ts#L76-L85).
Mid: callers `await` forever during a redirect (intentional — no error flash), but any `.finally` loading-reset or SWR error state won't fire; if it triggers on a non-redirect path, the UI hangs silently.
Senior: the alternative (throw a typed redirect error, catch once) and why the current design trades correctness-everywhere for simplicity-at-the-redirect.
Drill: describe the user-visible symptom if this fires unexpectedly.

## Q12: "How do you share types/logic between frontend and backend here?"
Testing: monorepo boundaries.
Anchor: `@gitroom/*` aliases; shared codec [posts.list.minify.ts](../../../libraries/helpers/src/utils/posts.list.minify.ts) imported by both; frontend importing `makeId` from nestjs-libraries ([add.edit.modal L6](../../../apps/frontend/src/components/new-launch/add.edit.modal.tsx#L1-L11)).
Mid: shared pure modules avoid drift (one codec, both sides); but nothing prevents importing server-only code into the client.
Senior: enforcement (eslint import zones, `server-only`, exports maps) and the single-dep-universe trade-off.
Drill: is importing `makeId` from `nestjs-libraries` into the frontend safe? Why? (It's pure — yes.)

---

Practice protocol: for each, give the junior answer, then immediately upgrade it to mid, then senior — out loud. The upgrade *is* the interview skill. Cross-reference the deeper mechanics in [02-stack-and-language-mastery](../02-stack-and-language-mastery/02-framework-mental-models.md).
