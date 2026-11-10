# Apple Watch companion — implementation plan

Status: proposed, not started. Written 2026-11-10 against
`ztwalsh/cardio-tracking` at commit `a845e43`. The plan is based on reading
the code; nothing has been built or run.

## Goal

A watchOS companion to the move/think iPhone app that:

- tracks a workout while it happens (walk, run, bike, swim)
- plots the route as you move
- lets you speak into the Watch and saves the result as an entry

## Product direction

The full principles are in [context/PRODUCT.md](context/PRODUCT.md). The
ones that shape the Watch app:

- **A running partner, not a fitness dashboard.** The Watch is where
  dashboards usually creep in. The workout screen stays small: time,
  distance, and the route.
- **Notes are first-class data.** The spoken reflection is the point of the
  Watch app, not an add-on after the metrics. The full transcript is kept,
  as it is on the phone.
- **Metrics and streaks are opt-in.** No rings to close, no pace alerts, no
  nudges by default.
- **Mood is context, not a score.** Mood comes out of the spoken note, the
  same way the phone's voice flow derives it. The Watch does not ask for a
  rating.
- **Language matters everywhere.** Watch copy follows the same voice: warm,
  direct, no fitness clichés.

Visual design follows [context/DESIGN.md](context/DESIGN.md) and motion
follows [MOTION.md](MOTION.md).

## Approach

The Watch does the tracking. The iPhone stays the only thing that talks to
Supabase and the language model. That means no login on the Watch and no
changes to the existing sync or auth code.

1. **Watch tracks the workout** with `HKWorkoutSession` and
   `HKLiveWorkoutBuilder`. This keeps the app alive in the background and
   saves the workout to Apple Health.
2. **Watch records the route** with `HKWorkoutRouteBuilder` and draws it
   live on a SwiftUI `Map` with a polyline.
3. **Watch records the voice note as an audio file** and hands it to the
   phone over WatchConnectivity.
4. **Phone turns it into an entry** with code that already exists:
   `VoiceInputManager.transcribeFile()` → `VoiceParser` →
   `RunRepository.addRun`.

If the phone is not nearby, the workout summary and the audio queue on the
Watch and are delivered when it reconnects.

```
Watch                                   iPhone                       Cloud
─────                                   ──────                       ─────
HKWorkoutSession ──► Apple Health ◄──── (dedupe by workout UUID)
route + metrics ───► WatchConnectivity ─► WatchSessionReceiver
audio file ────────► WatchConnectivity ─► transcribeFile()
                                          VoiceParser ─────────────► LLM proxy
                                          RunRepository.addRun ────► Supabase
```

## Phases

Each phase ends in something usable. Sizes are rough estimates.

| Phase | What you get | Size |
|---|---|---|
| 0. Foundations | Watch target, shared models, HealthKit capability, schema migration | 1–2 days |
| 1. Track a workout | Pick activity, start/pause/end, live time and distance; the finished workout appears as an entry on the phone | 3–4 days |
| 2. Routes | Live map on the wrist; recorded route shown in the phone's read-only route viewer | 2–3 days |
| 3. Voice | Speak a reflection after a workout, or a voice-only entry with no tracking | 3–4 days |
| 4. Polish | Complication and Smart Stack widget, always-on display, water lock for swims | 2–3 days |

### Phase 0 — Foundations

- Add a watchOS app target to `Cardio Tracking.xcodeproj`, paired with the
  iPhone app. Done in Xcode under the existing team.
- Add the HealthKit capability to both targets, and the workout-processing
  background mode to the Watch target.
- Add usage strings: Health read and write, location while in use, and
  microphone on the Watch.
- Share `ActivityType`, `RunDraft`, the duration formatting and the `RT`
  design tokens with the Watch target.
- Run the schema migration described under "Changes to the existing app".

### Phase 1 — Track a workout

- Activity picker with the four existing types.
- Map each to a HealthKit activity: running, walking, cycling, swimming.
  Swimming also needs pool or open water, and the pool length for pool
  swims.
- Start, pause, resume, end. Live elapsed time and distance.
- On end, save to Apple Health and send a summary to the phone: activity,
  start date, duration, distance, HealthKit workout UUID.
- Phone creates the `RunEntry` through `RunRepository.addRun`, which syncs
  to Supabase as it does today.

### Phase 2 — Routes

- Request location permission on the Watch and feed locations to
  `HKWorkoutRouteBuilder`.
- Draw the live track on a SwiftUI `Map` during the workout.
- On end, simplify the track to a few hundred points and include it in the
  summary sent to the phone.
- Show it on the phone in the existing read-only route viewer, marked as
  recorded so it is not offered for tap editing.

### Phase 3 — Voice

- After a workout ends, offer a "say something about it" step. Record with
  `AVAudioRecorder`, with a simple level indicator.
- Also allow a voice-only entry with no tracked workout, for logging after
  the fact.
- Transfer the audio file to the phone, tagged with the entry ID.
- Phone transcribes it with the existing batch path, parses it with
  `VoiceParser`, and updates the entry with notes, transcript, mood and
  icon.
- For a tracked workout, measured distance, duration and activity win over
  anything spoken.

### Phase 4 — Polish

- Complication and Smart Stack widget to start a workout or a voice note.
- Always-on display state with reduced detail.
- Water lock during swims.
- Haptics for start, pause and end.

## Changes to the existing app

- **Shared code.** The project uses file-system-synchronized groups, so
  sharing files with the Watch target is a membership change, not a
  restructure.
- **Schema.** Add `health_kit_uuid` (already drafted in
  `HEALTHKIT_PLAN.md`) and a `route_source` column (`drawn` or `recorded`)
  to `RunEntry` and the `run_entries` table. Both are additive, which is
  the safe SwiftData migration path the repo already follows.
- **Route storage.** The simplified track goes in the existing `waypoints`
  column. The full-fidelity track stays in Apple Health.
- **Phone receiver.** A new `WatchSessionReceiver` accepts workout
  summaries and audio files and deduplicates on the HealthKit UUID.
- **Voice pipeline.** `VoiceParser` gains a mode that takes known metrics
  and returns only notes, mood and icon.
- **Docs.** This plan replaces Phase 4 of `HEALTHKIT_PLAN.md` and pulls its
  schema work forward. `PRIVACY.md` needs a section on HealthKit and
  location data.

## Limits

- **Swimming routes.** Pool swims have no GPS, so they get laps and
  distance only. Open-water swims get a route, but it is patchy because GPS
  only fixes when the wrist is above water.
- **Voice is not instant.** The existing transcription uses
  `SFSpeechRecognizer`, which watchOS does not have, so the transcript and
  summary appear once the phone has processed the audio. The entry itself
  saves immediately with its metrics.
- **Testing needs a real Watch.** Workout sessions, GPS and Watch-to-phone
  transfer do not behave realistically in the simulator.
- **Xcode steps are manual.** Creating the Watch target and enabling
  capabilities are done in Xcode under the developer team.

## Open decisions

1. **Phone as hub, or fully standalone Watch?** Recommended: hub for the
   first version. Standalone (a cellular Watch syncing with no phone at
   all) means passing Supabase tokens to the Watch and duplicating the sync
   layer, roughly another week.
2. **Voice capture: recorded audio or system dictation?** Recommended:
   recorded audio. It reuses the phone pipeline and keeps the long-form
   transcript. Dictation into a text field is about a day less work and
   shows text on the wrist immediately, but suits short notes only.
3. **How much to show mid-workout?** Recommended: time, distance and the
   map only, with heart rate and pace left out or behind a swipe.
4. **Minimum watchOS version.** The iPhone app targets iOS 18; the natural
   match is watchOS 11. Not yet decided.
