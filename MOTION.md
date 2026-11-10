# Motion guidance

The app repo has no standalone motion document. Part 1 collects the rules
the iPhone app already follows, compiled from `RT.Animation` in
`Cardio Tracking/DesignSystem/RT.swift`, the notes in
[context/DESIGN.md](context/DESIGN.md), and the animation call sites in the
views (commit `a845e43`). Part 2 is a proposal for the Watch and has not
been reviewed.

## Part 1 — What the iPhone app does today

### Tokens

Defined in `RT.Animation`. Use the token; do not restate the curve inline.

| Token | Curve | Duration | Used for |
|---|---|---|---|
| `sheetPresent` | cubic bezier (0.32, 0.72, 0, 1) | 0.36s | Full-screen cards sliding up from the bottom |
| `sheetDismiss` | cubic bezier (0.32, 0.72, 0, 1) | 0.32s | The same cards leaving |
| `ringCounter` | cubic bezier (0, 0, 0.2, 1) | 0.9s | Progress rings and counting numbers |
| `rowEntrance` | ease out | 0.5s | Rows appearing |
| `rowHighlight` | ease out | 1.6s | The fading highlight on a newly added row |
| `calloutPop` | ease out | 0.2s | Chart callouts and picker selection |

### Conventions seen in the views

These are not tokens, but they are used consistently.

- **Tab switches:** ease out, 0.15s.
- **State swaps inside a screen:** ease in-out, 0.2s, with an opacity
  transition on the content that changes. The recording screen's
  idle → recording → done change is the reference case.
- **Text that changes in place:** `contentTransition(.opacity)` with ease
  in-out, 0.35s.
- **Home screen entrance:** ease out, 0.3–0.4s per element, staggered by
  0.05s steps.
- **Numbers that change:** interpolate through `AnimatedNumber`; do not
  swap the text.

### Rules

- **Fade; do not scale or slide small things.** Callouts and disclosure
  content use plain `.opacity`. Opacity combined with scale, and opacity
  combined with move, were both tried and reverted because they read as a
  "chunky, claymation" unfold.
- **Layout changes must animate.** State changes that pop instantly were
  flagged in review as jarring.
- **Movement is reserved for full-screen cards.** They enter and leave with
  `.move(edge: .bottom)` on the sheet tokens. Nothing else slides.
- **Nothing synthetic.** The waveform draws real microphone levels, oldest
  on the left and newest on the right. A fake sine ripple was removed and
  should not come back.
- **Smooth in one place, by elapsed time.** The waveform applies its attack
  (about 50ms) and release (about 150ms) envelope once, in the view, using
  real time between frames. Smoothing in two places made it laggy, and a
  fixed per-frame factor ties speed to the display's refresh rate.
- **Ambient signals stay small.** While recording, the transcript is the
  main content and the waveform is a 36pt strip in a quiet color.
- **Respect Reduce Motion.** `ReviewView` reads
  `accessibilityReduceMotion` and drops the animation when it is on. Only
  that view does so today; new work should do it everywhere.

## Part 2 — Applying this on the Watch (proposed)

The Watch is glanced at mid-stride, on a small screen, on a tight battery
budget. The same rules apply, turned down.

- **Reuse the tokens.** Share `RT.Animation` with the Watch target so both
  apps move the same way. Add Watch-specific tokens only when a real need
  appears.
- **Fades only for state changes.** Starting, pausing, resuming and ending
  a workout swap content with a 0.2s opacity transition. No scale, no
  slide.
- **Metrics tick; they do not animate.** Elapsed time and distance update
  as plain text. `AnimatedNumber` and `ringCounter` are for the summary
  screen after the workout, not the live screen.
- **The route grows; the camera stays calm.** New points extend the
  polyline without animating the whole line. Camera moves are infrequent
  and eased, not a re-center on every fix.
- **The recording indicator shows real audio.** A reduced version of the
  waveform driven by actual levels, with fewer samples than the phone's 64.
- **Haptics carry what motion cannot.** Start, pause, resume and end each
  get a system haptic, since the wrist is often not being looked at.
- **Always-on display is still.** When the wrist is down, drop to roughly
  once-a-second updates, hide the waveform and any pulsing element, and dim
  the route.
- **Reduce Motion is honoured everywhere** from the first build.
- **No celebration animations.** No confetti, bursts or rings closing. The
  product principles rule out gamification by default.
