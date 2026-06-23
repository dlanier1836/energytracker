# AGENTS.md — `energy_tracker.html`

This is a **single-file** task manager. The maintainer is *you, an agent*. The "user" is a
non-technical author who reports bugs as **"I saw this problem"** with no technical detail.
These rules exist so a half-applied change can't garble the page or silently lose data.

## The contract

1. **Mutate state only through `commit` / `commitAdd` / `commitDelete`.**
   Never call `apiPost` / `outreachPost` / `errandsPost` directly, and never hand-write an
   `arr.unshift(...)` + `fetch(...)` pair. `commit*` is the one door: *mutate in-memory →
   render → persist* (the persist is FIFO-queued via `enqueueSync`, so writes can't land out
   of order). Going around it means the three steps can be left half-done. Every mutation
   through this door is also recorded in the event log (see #5) — bypassing it makes bugs
   invisible to the report tool.

2. **Render only through `` html`` `` / `esc` / `raw`.**
   Never interpolate a user value into `innerHTML` unescaped. `` html`<b>${name}</b>` ``
   auto-escapes `${name}`; wrap already-trusted HTML in `raw(...)`. Decompose large templates
   into small `` html`` `` helpers rather than building strings by hand.

3. **After ANY change: open `?selftest=1`, confirm all green, and add an assertion**
   covering the new behavior. The self-test runs offline against the safe-HTML primitives,
   the write queue, the commit choke point, and the event log — it's the regression gate.
   Also run `node --check` on the `<script>` for syntax before you're done.

4. **Bump `BUILD`** (near the top of `<script>`) on every change. It's a plain date string —
   there is no build step — and it's stamped into every bug report so an agent knows which
   version produced a symptom.

5. **On a bug report ("I saw a problem"):** open `?debug` (or have the user click the small
   **⚠ report** button and paste what it copies). Both surface the event log + a state
   snapshot + the most recent error as a paste-ready prompt. **Reproduce via `?selftest`
   before guessing** — trace the recent actions in the log, don't speculate.

## Adding a field

**Adding a field** is now one edit: add a `[name, coercer]` entry to the entity's
`ITEM_FIELDS` / `OUTREACH_FIELDS` / `ERRAND_FIELDS` registry. `normalize*()` and the
new-object defaults derive from it automatically. You still: (a) add the field's UI to
the bespoke template **if it should display/edit**, and (b) ensure a matching Google
Sheet column exists for it to **persist** (the Apps Script backend is external to this
file). Coercers live in `C` — reuse one (`str`, `bool`, `num(d)`, `date`, `pri`, …) or
add a new pure coercer.

## Hard constraints

- **Single file.** No build step, framework, bundler, module split, or external JS dependency.
- The error log summarizes a patch to **field names only, never values** — keep it that way;
  the report may be pasted into a chat.
- The `?selftest` probes must stay **non-destructive**: snapshot any localStorage key / array
  you touch and restore it in a `finally`.
