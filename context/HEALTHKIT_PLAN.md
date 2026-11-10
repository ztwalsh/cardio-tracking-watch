# Apple Health / HealthKit sync — plan

Forward-looking design doc. Nothing here is built yet. Park until after
the core app is in TestFlight.

Trigger: making the app feel native on the Apple ecosystem so Watch-logged
workouts show up here, and entries logged here show up in Fitness/Health.

---

## Scope decision: HealthKit, not a watchOS app

The Apple Watch's built-in Workout app writes to HealthKit. Reading from
HealthKit covers ~95% of what "syncs with my Watch" actually means. A
standalone watchOS companion app (start workouts from the wrist, voice-log
via dictation) is a separate project — not worth it until product-market
fit with iPhone users.

If the watchOS app ever lands, it's a new Xcode target. Big enough that
this doc explicitly scopes it out.

---

## Phasing

Each phase ends in something usable.

### Phase 1 — One-way import from HealthKit

User taps "Import from Apple Health" in Settings → Health. We pull past
workouts and create `RunEntry` rows for each.

- Add HealthKit capability to the Xcode target (paid Dev account required)
- Add `NSHealthShareUsageDescription` to Info.plist with a specific
  reason (App Store reviews this — "Used to import your runs and walks
  from the Health app so they appear alongside entries you log here")
- Request authorization for `HKWorkoutType` read on these activity types:
  - `.running` → `ActivityType.running`
  - `.walking` → `.walking`
  - `.cycling` → `.biking`
  - `.swimming` → `.swimming`
- One-shot `HKSampleQuery` ordered by `endDate` descending
- Map each `HKWorkout` to `RunEntry`:
  - `date` = workout.startDate
  - `durationSeconds` = workout.duration
  - `distanceMiles` = workout.totalDistance.doubleValue(for: .mile()) if available, else 0
  - `notes` = nil (workouts don't have free-text)
  - `city` = optional (not present on the workout; skip)
  - `activityType` = mapped from `workoutActivityType`
- Skip workouts we already imported (see dedup below)
- Show progress UI: "Imported N of M workouts"

Estimate: 1 evening.

### Phase 2 — Two-way: write our entries back to HealthKit

When the user logs a manual or voice entry, also write an `HKWorkout` so
it appears in Apple Fitness and on the Watch's history.

- Add `NSHealthUpdateUsageDescription` to Info.plist
- Request authorization for `HKWorkoutType` write
- On every successful `RunRepository.addRun`, also create an `HKWorkout`
  via `HKWorkoutBuilder` (or the modern `HKWorkout(activityType:start:end:duration:totalDistance:...)`)
- Store the resulting workout UUID on the `RunEntry` so a future
  HealthKit re-import doesn't create a duplicate

Estimate: 1 evening.

### Phase 3 — Live background sync

A workout finished on the Watch should appear in our app without the
user opening Health and tapping import.

- `HKObserverQuery` registered at app launch
- `enableBackgroundDelivery(for: .workoutType(), frequency: .immediate)`
- On observer fire, kick off an `HKAnchoredObjectQuery` so we only pull
  the delta since last sync (store the `HKQueryAnchor` in UserDefaults)
- Same mapping + dedup as Phase 1

Estimate: 1 evening. Real complexity is testing — the simulator's
HealthKit doesn't fire background delivery; need a real device.

### Phase 4 — watchOS companion app

Genuinely separate project. Scoped out for now. Re-evaluate once iPhone
app has real users.

- New Xcode target (watchOS app, paired with the iPhone app)
- SwiftUI views sized for the Watch
- `HKWorkoutSession` + `HKLiveWorkoutBuilder` for active workouts
- Voice via on-device dictation (Watch has no `SFSpeechRecognizer`)
- IPC: probably nothing custom — both targets read/write to HealthKit
  and to Supabase via shared keychain session

Estimate: 1–2 weeks of focused work.

---

## Schema implications

`RunEntry` and the Supabase `run_entries` table each need a column to
track which workouts came from HealthKit, so re-import is idempotent and
so writes-back-to-HK don't loop.

```swift
// RunEntry
var healthKitUUIDRaw: String?  // matches HKWorkout.uuid.uuidString
```

```sql
alter table public.run_entries
    add column health_kit_uuid text;
create unique index if not exists run_entries_user_hk_uuid
    on public.run_entries (user_id, health_kit_uuid)
    where health_kit_uuid is not null;
```

Partial unique index — null values don't conflict, so manually-logged
entries (no HK source) aren't constrained.

Dedup rules during import:
1. If incoming `HKWorkout.uuid` matches an existing `RunEntry.healthKitUUIDRaw`
   for the current user → skip.
2. Otherwise insert as new, stamping the UUID.

---

## Activity-type mapping

`HKWorkoutActivityType` has dozens of cases. We only ship 4. Mapping:

| HKWorkoutActivityType                        | ActivityType  |
|----------------------------------------------|---------------|
| `.running`, `.crossCountrySkiing`, `.hiking` | `.running` (?)|
| `.walking`                                    | `.walking`    |
| `.cycling`, `.handCycling`                    | `.biking`     |
| `.swimming`, `.waterFitness`, `.waterSports`  | `.swimming`   |
| _everything else_                             | skip          |

Open question: should we add `.hiking` as a distinct activity type
before HealthKit lands, or roll it into `.walking`? Defer until users
ask.

---

## Privacy + App Store

HealthKit usage strings must be specific enough that Apple's reviewers
understand what's being read/written and why. Vague strings get
rejected. Draft:

> NSHealthShareUsageDescription
> Reads your running, walking, cycling, and swimming workouts so they
> appear in Cardio Tracking alongside the entries you log manually.

> NSHealthUpdateUsageDescription
> Writes entries you log in Cardio Tracking back to Apple Health so
> they show up in Fitness and on your Watch's history.

Privacy policy (`PRIVACY.md`) needs a new section disclosing that
HealthKit-sourced data is also stored in Supabase. App Store Review is
strict about this — HealthKit data leaving the device without explicit
disclosure is a rejection trigger.

---

## Risks / unknowns

- **Background delivery is real-device-only.** Phase 3 can't be tested
  in the simulator. Plan a TestFlight build specifically for HealthKit
  verification.
- **HealthKit data volumes.** A user with years of Workout-app history
  could have thousands of workouts. Phase 1 import needs batching,
  progress UI, and cancellation.
- **Time zones.** `HKWorkout.startDate` is a wall-clock `Date`. Our
  week-bucketing logic assumes the user's current calendar. Should be
  fine but worth re-verifying when importing old data.
- **HKWorkoutRoute / heart rate / pace samples.** Apple has them, we
  don't model them. Out of scope for v1; could be additive later.

---

## Build order recap

1. Ship TestFlight beta with current scope (auth, sync, goals, voice).
2. Phase 1 — HealthKit one-way import.
3. Phase 2 — Write-back.
4. Phase 3 — Live observer (only if friends-as-beta-users actually ask).
5. Phase 4 — watchOS app (only if there's a real reason to start workouts
   from the wrist, which there usually isn't if you're already wearing
   the Watch + carrying the phone).
