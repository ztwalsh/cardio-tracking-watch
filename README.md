# move/think — Apple Watch companion plan

Planning repo for an Apple Watch companion to the move/think iPhone app
(`ztwalsh/cardio-tracking`). Nothing here is built yet.

The Watch app should track a workout as it happens, plot the route for
walks, runs, rides and swims, and let you speak a note that becomes a saved
entry.

## Contents

| File | What it is |
|---|---|
| [PLAN.md](PLAN.md) | The implementation plan: architecture, phases, code changes, limits, open decisions |
| [MOTION.md](MOTION.md) | Motion guidance: the iPhone app's existing rules, and how they apply on the Watch |
| [context/PRODUCT.md](context/PRODUCT.md) | Product principles, copied unchanged from the app repo |
| [context/DESIGN.md](context/DESIGN.md) | Design system reference, copied unchanged from the app repo |
| [context/HEALTHKIT_PLAN.md](context/HEALTHKIT_PLAN.md) | The earlier HealthKit plan, copied unchanged; PLAN.md replaces its Phase 4 |

## Provenance

The files in `context/` are snapshots of `ztwalsh/cardio-tracking` at commit
`a845e43` (2026-11-10). The app repo stays the source of truth; refresh
these copies if the originals change.

`MOTION.md` is not a copy. The app repo has no standalone motion document,
so it was compiled from the `RT.Animation` tokens in
`Cardio Tracking/DesignSystem/RT.swift`, the motion notes scattered through
`DESIGN.md`, and the animation call sites in the views. Its Watch section is
a proposal, not established guidance.

`DESIGN.md` refers to files that are not copied here, such as
`design-refs/basho-linear-round4/linear.design.md` and the JSX mockups. They
live in the app repo.
