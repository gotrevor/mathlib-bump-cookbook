# mathlib bump cookbook

A symptom → cause → fix catalog of **mathlib breaking changes that produce no deprecation warning** —
the ones that just leave you staring at `unsolved goals` or `made no progress` after a version bump,
with nothing to grep for.

## Why this exists

The official docs cover the *mechanics* of upgrading well
([Updating Mathlib in your project](https://leanprover-community.github.io/install/project.html)):
bump `lean-toolchain`, `lake update`, `lake exe mk_all && lake build`. What they don't cover — by design —
is **what will break and how to fix it**. The canonical answer to "what changed" is "read the
[git tags](https://github.com/leanprover-community/mathlib4/tags)," which are organized by PR, not by the
error in front of you.

For *renamed* declarations that's fine: mathlib's `@[deprecated]` aliases name their replacement in the
warning text, so a rename is self-documenting. But a whole class of breaks has **no name to alias** and so
gets no warning at all:

> **Tactic and elaboration *behavior* changes** — `convert` splitting a goal differently, `ring` now
> erroring on no-progress, `simpa`'s final check going syntactic instead of up-to-defeq, `rw`'s
> auto-closing `rfl` going reducible-transparency-only, `grind` no longer reasoning past a multiplication.

These produce a hard error or an unsolved goal with zero indication of *why*, and they're the most
time-consuming part of any bump. This catalog is aimed squarely at that gap.

**One clarification after the fact:** Lean's *release notes* (distinct from the install docs above) *do*
document several of these behavior changes, with migration hints — see [Primary sources](#primary-sources).
But they're organized by change rather than by symptom, they cover Lean core only (not mathlib's own
tactic-behavior shifts or lemma renames), and you still have to map the error in front of you back to the
note. Two payoffs from reading them are folded in below: (1) the biggest category here — the
transparency/defeq tightening — has **official one-character escape hatches** worth trying before any
hand-rewrite (see [Official escape hatches](#official-escape-hatches-try-before-hand-rewriting)); (2) a few
entries can now be pinned to the 4.30 vs 4.31 step.

## Scope

These notes were collected forward-porting several mathlib projects from the **v4.29.1** toolchain to
**v4.31.0** (skipping v4.30.0). A given change may have landed in either the 4.30 or 4.31 step.
Cross-referencing the Lean release notes (see [Primary sources](#primary-sources)) now pins a few - e.g.
entry I's app-elaborator beta-reduction and the entry-F defeq/transparency tightening are 4.31 changes - but
most are still un-bisected, and I'd welcome anyone who pins down the rest. Each entry is empirically
grounded: the error
was hit in a real proof, the fix made it compile. Corrections, sharper explanations, and additions for
other version windows are very welcome — **issues and PRs open**.

## How to use it

Find your symptom in [`v4.29.1-to-v4.31.0.md`](v4.29.1-to-v4.31.0.md). Each entry gives the error you see,
what changed underneath, and a minimal fix. The patterns are grouped:

- **Behavior changes (no warning)** — the high-value ones; A–K, M, P.
- **Renames / signature changes that hard-error** — L, N, O, Q, R (a deprecation alias was missing or only
  *warned*, `@[to_additive]` didn't propagate, or an argument went implicit — so you get `unknown constant` /
  `Function expected` / `Invalid field`, not a friendly warning).

The single most useful debugging technique (drop a `trace_state` and read the goal v4.31 actually produced)
is described at the top of that file.

## Official escape hatches (try before hand-rewriting)

The largest category below - `simpa` / `convert` / `exact` / `rw` no longer closing a goal that's
definitionally equal but not *syntactically* equal (entries F, G, and the `convert` family A/C/K/P) - is the
Lean 4.31 **defeq-discipline** change: defeq checking now respects transparency, and these tactics' final
check runs at *reducible* transparency instead of ambient. The release notes ship one-character fixes for
exactly this. Try them **before** the per-entry manual rewrites, which become the explicit fallback:

- `simpa using h` → **`simpa using! h`** - the `!` restores ambient (default/semireducible) transparency.
- `convert h using N` → **`convert! h using N`** - same idea, for the over-split / spurious-instance-goal family.
- A whole declaration breaks on transparency → scope **`set_option backward.defeqAttrib.useBackward true in`** over it.
- A specific `def` must unfold under `simp` / `dsimp` → mark it **`@[implicit_reducible]`**, or add its
  projection explicitly to the `simp` set.

These restore the *looser* transparency rather than re-proving anything, so they can paper over a defeq you'd
rather make explicit - see [the faithfulness note](#a-note-on-faithfulness). For speed, reach for the `!`
form first, then spot-check headline theorems with `#print axioms`; where the footprint shifts, switch that
one site to the explicit `simp only [...]; exact h` form the catalog gives.

> ✅ **`simpa using!` / `convert!` confirmed in practice** porting the Foundation logic library to v4.31:
> they are the faithful one-character fix for the defeq cluster, not just a release-note suggestion. The
> `set_option backward.defeqAttrib.useBackward` / `@[implicit_reducible]` hatches come straight from the 4.31
> notes; verify their exact spelling against your build. Note the per-entry catalog below (and the Foundation
> Part-3 notes) still spells out the *manual* `simp only […]; exact h` ladder - those entries predate this
> section and haven't been retrofitted to lead with the `!` form yet.

## Where this fits in a bump

A mathlib bump has two phases. The first is **mechanical and scriptable**: point
`lean-toolchain` and the lakefile's mathlib rev at the new tag, refresh
`lake-manifest.json`, fetch the prebuilt oleans (`lake exe cache get`), and `lake build`.
That part is the same every time — automate it.

**Before the second phase, read the new version's release notes** ([Primary
sources](#primary-sources)). This is the cheapest win in the whole bump and the easiest step to skip: Lean's
notes document the Lean-core behavior changes *with the official migration fixes*, so you patch the biggest
category — the defeq/transparency cluster — in one character (`simpa using!` / `convert!`) instead of
re-deriving it by hand across dozens of sites. (This catalog was first built without reading them, and the
fix it hand-rolls for that cluster is exactly the one the 4.31 notes state outright. Don't repeat that.)

The second phase is **this catalog**: the cascade of source breaks the new mathlib
introduces, which no script can fix for you. So an automated bump that ends in a *red
build* isn't a failure of the automation — it's the normal hand-off point, and that red
build is exactly where the symptom → cause → fix entries below come in. Automate phase
one; keep this open for phase two. (And a green build still owes you the check below.)

## A note on faithfulness

A bump that compiles is not necessarily a bump that preserved your theorems. After porting, check your
headline results' axiom footprint with `#print axioms <thm>` against a pre-bump baseline — a green build
can still hide a proof whose footprint silently changed. "Green ≠ faithful" is the one habit worth keeping.
This applies doubly to the `using!` / `backward.defeqAttrib.useBackward` escape hatches above: they win back
the old transparency instead of re-proving anything, so they're the most likely to mask a meaning-shift.

## Primary sources

The official release notes that pin and explain the Lean-core behavior changes (read these alongside the catalog):

- [Lean 4.31.0 release notes](https://lean-lang.org/doc/reference/latest/releases/v4.31.0/) — the
  defeq/transparency tightening (entries F/G + the `convert` family), app-elaborator beta-reduction
  (entry I), and the `simpa using!` / `convert!` / `@[implicit_reducible]` fixes.
- [Lean 4.30.0 release notes](https://lean-lang.org/doc/reference/latest/releases/v4.30.0/) —
  `inferInstanceAs` no longer a synonym for `inferInstance`, typeless inductive-constructor binders needing
  a type, and the removal of `change … with`.

mathlib has **no single changelog**: its own tactic-behavior shifts (`ring` erroring on no-progress, the
`convert` over-split) and lemma renames (entries D, L, N, O, Q here) live in the bump PRs and
`Mathlib/Deprecated/`, surfaced as deprecation messages at build time.

## License

[Apache License 2.0](LICENSE), Copyright 2026 Trevor Morris — matching mathlib, Lean, and the rest of the
ecosystem, so snippets copy into your mathlib project with zero license friction.
