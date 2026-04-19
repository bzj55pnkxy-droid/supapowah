---
name: ui-lifecycle-check
description: Use during planning or brainstorming when the work touches React/Next.js UI — before writing tasks or committing to an approach. Forces enumeration of lifecycle failure modes that narrow-context implementations miss.
---

# UI Lifecycle Check

## Why This Exists

React/Next.js implementations that work "on this page, in this state, once" routinely break the moment the system grows around them. The code reads locally correct — it just assumes a mount context that doesn't hold.

The three failure families you keep producing:

- **State-capture bugs** — handlers close over stale state; resets fire only on the route they were designed for; global state persists across navigations that the local component doesn't know about.
- **Replay bugs** — entrance animations, `useEffect` initializations, and SSR→CSR hydration flicker re-run on every mount, which is every route change, not every page load.
- **Invocation-context bugs** — a component/handler is designed against one caller, but lives somewhere (provider, layout, shared sidebar) that invokes it from N callers. The other N−1 get wrong behavior.

These are not coding-skill bugs. They are **planning-context** bugs. The implementer didn't enumerate where the component mounts, who invokes it, or what state outlives a navigation.

This skill forces that enumeration *before* the plan locks in.

## When To Invoke

Invoke at planning/brainstorming time, before tasks are written, if the work involves any of:

- New or modified React component that renders above the route tree (layout, provider, sidebar, header, modal portal)
- Navigation-triggered behavior (route change, redirect, reset-on-home)
- State that should clear, reset, or persist across routes (form state, active selection, scroll position, playback)
- Entrance animations, loading states, skeletons, or any `animate-in` / `transition` on mount
- Effects that fetch, subscribe, initialize, or set up listeners
- Any use of `useEffect`, `useState`, refs, context, or `usePathname`/`useRouter`

Skip only if the work is pure styling with no lifecycle surface (e.g. "change this button's color").

**Announce at start:** "Using ui-lifecycle-check before finalizing the plan."

## The Discipline: Four Enumerations

You must produce all four enumerations in writing. Each must pass the "bad enumeration" filter below. No skipping, no merging, no "N/A" without justification.

### 1. Mount Matrix

List every place the affected component(s) actually mount in the app. Not "where the feature lives" — where the React tree instantiates them.

For each mount site, note:
- Which layout/provider wraps it
- Whether it mounts once per session or once per navigation
- Whether it unmounts and remounts on route change

**Bad enumeration (rejected):**
> "The component mounts on the home page."

**Why bad:** Treats "page" as the unit. React mounts per tree position, not per URL. A provider above the route tree mounts once; a component inside the route tree remounts per navigation. The distinction is the whole point.

**Good enumeration:**
> "`SidebarProvider` mounts once in `layout.tsx` above the route tree — state survives navigation.
> `Sidebar` (the UI) mounts inside the layout slot — unmounts/remounts on any route change.
> `HintCTA` mounts inside `app/page.tsx` — only exists while on `/`, remounts every time the user returns to `/`."

### 2. State Persistence Audit

For every piece of state the plan touches, answer: **does it survive a route change, and should it?**

Enumerate:
- State held in providers above the route tree (survives)
- State held in route-tree components (dies on navigation away)
- State in the URL / search params (survives, shareable)
- State in localStorage / sessionStorage / cookies (survives sessions)
- Derived state computed from props (recomputes on every render — risk of stale capture)

Then: for each "should reset on X" behavior in the plan, trace what actually triggers the reset. If the reset is wired to a route-tree component but the state lives in a provider, the reset only fires when the user happens to be on that route — which is the bug.

**Bad enumeration:**
> "Clicking the logo resets the page."

**Why bad:** Doesn't say where the reset handler lives, where the state lives, or whether the handler is mounted when the user clicks. If the logo's handler only fires `triggerReset()` when `pathname === '/'`, the reset dies silently on every other route.

**Good enumeration:**
> "Active summary selection lives in `SidebarProvider` (above route tree, persists).
> Logo click handler lives in `Header` (above route tree, always mounted).
> Reset trigger: must fire `setActiveSummary(null)` AND `router.push('/')` unconditionally. Do not gate on `pathname === '/'` — the whole point is that the user might not be on `/`."

### 3. Replay Audit

List every effect, animation, or initialization that runs on mount, and decide whether it should run:
- Once per session
- Once per route visit (every time the user returns)
- Never (should be SSR-rendered or hydration-stable)

**Replay surfaces to enumerate:**
- `useEffect(() => {...}, [])` — runs on every mount
- Tailwind `animate-in`, `fade-in`, `slide-in` classes — CSS runs on every mount
- Framer Motion `initial`/`animate` — runs on every mount
- Suspense boundaries revealing content — plays entrance on every reveal
- `useState` initializer functions with side effects
- Data fetching in effects vs. in server components

If an entrance animation is on a component that remounts per navigation, it *will* replay on every navigation. That's usually not what anyone wants.

**Bad enumeration:**
> "The hint has a fade-in animation."

**Why bad:** Doesn't say when it runs. "Has an animation" is a styling fact; "replays on every navigation back to `/`" is a behavioral fact, and the one that matters.

**Good enumeration:**
> "Hint CTA uses `animate-in fade-in-0 duration-700 delay-[600ms]`. Mounts on `/` only.
> User flow: land on `/` (plays) → navigate to `/summary/abc` (unmounts) → back to `/` (**replays**).
> Decision: either (a) lift the hint above the route tree so it mounts once, (b) gate the animation behind a sessionStorage flag, or (c) accept the replay as intentional. Pick one in the plan, don't leave it implicit."

### 4. Invocation Matrix

For every handler, effect, or component method the plan introduces, list **all** call sites — not just the one you're designing for.

- Who calls it today?
- Who will call it after this change?
- Is the component rendered in multiple places simultaneously?
- Does a global listener or context subscriber invoke it?

The failure mode: you design `loadSummary(id)` assuming it's called from the home-page list, so you don't `router.push()` inside it. Then the sidebar — which also calls `loadSummary(id)` and is visible on every route — calls it from `/account`, and the user stays stuck on `/account` with the summary "loaded" but invisible.

**Bad enumeration:**
> "`loadSummary` is called when the user clicks a summary."

**Why bad:** "The user clicks a summary" conflates call sites. There are (at least) two: the home-page list and the sidebar. They have different surrounding contexts. The handler must work for both or have different handlers.

**Good enumeration:**
> "`loadSummary(id)` call sites:
> 1. `app/page.tsx` summary list — caller is already on `/`, no navigation needed.
> 2. `components/sidebar.tsx` — caller can be on ANY route; handler must `router.push('/')` after setting state, or the summary loads invisibly.
> Decision: put `router.push('/')` inside `loadSummary` itself. Caller on `/` gets a no-op push; caller elsewhere gets navigated. One handler, correct for all call sites."

## Reference Library (Open, Judgment-Driven)

After completing the four enumerations, consult the Vercel React best-practices rules to pressure-test your plan. Do **not** treat this as a checklist. Read the titles, pick the ones that match the lifecycle concerns you just enumerated, and fetch only those.

Canonical source: `vercel-labs/agent-skills` on GitHub. Rule files live under:

```
https://raw.githubusercontent.com/vercel-labs/agent-skills/main/skills/react-best-practices/rules/<rule-name>.md
```

Rules most often relevant to the three failure families above (starting floor, not ceiling — fetch more as the plan demands):

| Concern | Rule file |
|---|---|
| Stale closures in handlers / effects | `advanced-event-handler-refs.md`, `advanced-use-latest.md` |
| State that should survive or reset across navigations | `rerender-derived-state.md`, `advanced-init-once.md` |
| Entrance animations / flicker on mount | `rendering-hydration-no-flicker.md`, `react-view-transitions.md` |
| Effects running in the wrong component | `rerender-effect-dependencies.md`, `advanced-init-once.md` |
| Handlers invoked from multiple call sites | (no dedicated rule — use Invocation Matrix above) |

Full rule index (fetch to browse):
`https://raw.githubusercontent.com/vercel-labs/agent-skills/main/skills/react-best-practices/SKILL.md`

**Judgment call:** If the enumeration surfaces a concern that none of the rules cover, name the concern in the plan anyway. The rules are a library, not an exhaustive catalog.

## Output

Emit the four enumerations as a **Lifecycle Check** section in the plan or brainstorming doc, before tasks are written. Format:

```markdown
## Lifecycle Check

### Mount Matrix
- <component>: mounts at <location>, <once-per-session | remounts-per-nav>
- ...

### State Persistence
- <state>: lives in <location>, <survives | dies> on nav
- Reset triggers: <handler> in <location>, fires <conditions>
- ...

### Replay Audit
- <effect/animation>: runs on <mount frequency>, decision: <keep | gate | lift>
- ...

### Invocation Matrix
- <handler>: called from <sites>, needs <behavior> to be correct for all
- ...

### Rules Consulted
- <rule-name.md>: <what it flagged or confirmed>
- ...
```

If the plan changes after this section is written, update the section. A Lifecycle Check that doesn't match the final tasks is worse than none.

## Red Flags — Stop and Redo

You're producing a bad enumeration if you catch yourself:

- Treating "page" as the unit instead of "mount site"
- Writing "the user clicks X" without saying *from where*
- Saying "state resets" without naming where the state lives AND where the reset fires
- Describing an animation as a visual fact instead of a per-mount behavior
- Enumerating one call site when you know or suspect there are more
- Using "should" language ("the handler should only fire on home") without enforcing it in code

If any of these appear, the enumeration isn't done. Redo the affected section.

## What This Skill Is Not

- **Not a linter.** It runs at planning time, not edit time. The point is to think about the tree before you write code against it.
- **Not a replacement for the Vercel rules.** It's the discipline that makes you consult them at the right moment with the right context.
- **Not a general "think harder" prompt.** It demands four specific artifacts. If you produce three, you skipped one — that's the one most likely to bite.
