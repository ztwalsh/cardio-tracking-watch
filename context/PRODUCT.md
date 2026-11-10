# Product Principles

## The core idea

This app is a trusted running partner, not a fitness dashboard. It remembers
what you were thinking about three weeks ago. It notices when you seem off. It
celebrates the run you almost didn't take. The relationship between the user
and this app should feel like being known — not being measured.

---

## Principles

### 1. The run that almost didn't happen matters most
Metrics can't capture meaning. A 1.8 mile run logged on an exhausted Tuesday
is often more significant than an easy 4 miler. The app should never make a
user feel like they underperformed. Effort and consistency are honored over
output.

### 2. Notes are first-class data
Free-form notes are not an afterthought field. They are the richest signal the
app collects — more meaningful than pace, distance, or any structured metric.
They should be stored, surfaced, and treated with the same weight as
performance data. The LLM layer in particular should treat notes as the
primary lens.

### 3. The LLM layer leads with reflection, not analysis
When the app or an AI layer surfaces patterns, it should speak like a
thoughtful friend, not a data analyst. "You've been carrying a lot this month"
is the right register. "Your average pace dropped 12% in high-stress weeks" is
not — even if it's true and interesting. Insight should land emotionally before
it lands analytically.

### 4. Metrics and streaks are opt-in
Performance data is available but not the default view. The app does not guilt,
nudge, or gamify by default. Users who want a dashboard can have one. Users who
just want to log and reflect should never feel like they're falling behind.

### 5. The app should know you over time
The goal of every data point collected — location, pace, mood, notes — is to
build a longitudinal picture of a person. Not to benchmark them against others
or against their past self, but to help them understand their own patterns.
The MCP layer exists so that this knowledge can travel with the user into
whatever AI tools they use — this data should be portable and personal.

### 6. Mood is context, not a score
Mood capture exists to give meaning to the run, not to produce a wellness
metric. The two-axis model (energy + pleasantness) is intentionally simple.
The derived quadrant label (e.g. "Calm," "Stressed") is a human-readable
tag for memory and reflection — not a grade.

### 7. Language matters everywhere
Every string in this app is an opportunity to sound like a person. Default
copy should be warm, direct, and free of fitness-app clichés. No "crushing
it," no "beast mode," no empty affirmations. Just honest, human language that
respects the user's intelligence and emotional state.

---

## What this app is not

- A performance tracker optimizing for PRs
- A social platform for comparing runs
- A gamified streak machine
- A clinical wellness tool

---

## The test

Before shipping any feature, ask: does this make the app feel more like it
knows you, or more like it's watching you? If it's the latter, reconsider.
