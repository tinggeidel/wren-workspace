# Wren — Feature A Prototype Spec: The Context-Aware Coach

> The first thing you build and live in daily. Goal: **prove the magic with the least code.**
> Local-only, single-user (you), no accounts, no cloud, no payments, no push.
> Detailed companion to `Wren_Project_Plan_v2.md` §3.
>
> **Accepted exception (approved 2026-05-25):** the food search / barcode lookup
> calls the public Open Food Facts API (`search.openfoodfacts.org`,
> `world.openfoodfacts.org`). It sends only the search query or barcode — no
> profile or personal data, no API key. This is the one network destination
> beyond `api.anthropic.com`; the "no cloud" rule otherwise still holds.

---

## 1. What "done" means (test gate)

You use the Coach **daily for one real week** and it:
1. Knows your goals, today's date, and your current cycle phase without you re-stating them.
2. Logs a meal when you describe it ("had 2 eggs and toast") and remembers it later today.
3. Answers "what should I eat for dinner?" / "how am I doing today?" usefully.
4. Speaks in **feel-based, supportive** language and **never** crosses the safety lines in §6.

If all four are true, Feature A passes. Don't start Feature B until then.

---

## 2. Scope — in vs out

**In (build this):**
- One **Coach chat screen** (message list + input box + send).
- 3–4 **always-visible suggested prompts** above the input:
  *"Log my lunch" · "What should I eat for dinner?" · "How am I doing today?"*
- A tiny **Settings/Goals screen** (one-time setup: name, goal, dietary rules, last-period date,
  average cycle length, **on birth control? yes/no**).
- **Minimal cycle phase math** (see §4).
- **Local persistence** of: profile/goals, cycle info, today's informal log, and chat history.
- **Claude call** with Haiku default / Sonnet for complex moments (see §5), with the strict
  **system prompt** (§6).

**Out (later features — do not build now):**
- Real structured food DB / search (Feature C), photo logging (D), Home/Plan screens (E),
  Progress/photos (F), accounts/cloud (G), notifications (J), payments (K).
- Persisting the food log past "today" in a structured way — for A, a simple per-day note is fine.

> **Scope expanded (approved 2026-05-25):** Features B (cycle check-ins), C (structured
> food logging + search/barcode), and E (Home / weekly Plan) have since been built and are
> approved as in-scope. Photo food logging (D) and Progress/weight (F) are also live.
> Still genuinely out: accounts/cloud (G), notifications (J), payments (K). This list above
> reflects the original Feature-A-only boundary; the audit log records the expansion.

---

## 3. Screens

```
┌─────────────────────────┐     ┌─────────────────────────┐
│  Settings / Goals (1x)  │     │        Coach            │
│                         │     │  ┌───────────────────┐  │
│  Name: ____             │     │  │ Coach: Morning! …  │  │
│  Goal: [lose fat /      │     │  │ You: had 2 eggs…   │  │
│   build muscle / feel   │     │  │ Coach: logged ✓ …  │  │
│   better / maintain]    │     │  └───────────────────┘  │
│  Dietary rules: ____    │     │  [Log my lunch]         │
│  Last period start: 📅  │     │  [Dinner idea?]         │
│  Avg cycle length: 28   │     │  [How am I doing?]      │
│  On birth control? [ ]  │     │  ┌───────────────────┐  │
│         [Save]          │     │  │ Type a message…   ▶│  │
└─────────────────────────┘     └─────────────────────────┘
```

Navigation: two screens is enough. A simple header button to open Settings from Coach.

---

## 4. Data model (local)

> Updated 2026-05-30 to match shipped code (`wren/lib/types.ts`). The original "keep it dead simple" Feature-A sketch is long superseded — the model grew as Features B–F shipped. The authoritative source is `wren/lib/types.ts`; the shapes below mirror it. Persist as JSON in AsyncStorage. Do not widen these without a migration story for existing stored data (most newer fields are optional precisely for back-compat).

### `Goal` / `Tone` / `ActivityLevel`

```ts
type Goal = "lose_fat" | "tone_up" | "build_muscle" | "feel_better" | "maintain";
type Tone = "hype" | "bestie" | "tough_love";
type ActivityLevel = "sedentary" | "light" | "active" | "very_active";
```

> Note: `goal` has five values (a `tone_up` value was added). `tone` is `hype | bestie | tough_love` — not the old `gentle/straight/hype`.

### `Profile`

```ts
type Profile = {
  name: string;
  goal: Goal;
  tone: Tone;
  dietaryRules: string;             // free text
  onBirthControl: boolean;
  lastPeriodStart: string;          // ISO date; legacy field, kept in sync with dayLogs
  avgCycleLength: number;           // days, default 28 (prior until dayLogs observe an average)

  // Daily logs keyed by ISO "YYYY-MM-DD" (Features B, C):
  dayLogs: Record<string, DayLog>;          // cycle check-ins (source of truth for cycle)
  foodLogs: Record<string, FoodEntry[]>;    // structured food; totals are SUMMED IN CODE
  waterLogs: Record<string, number>;        // cups of water that day
  workoutLogs: Record<string, WorkoutEntry[]>;

  // Biometrics — stored as strings, parsed where needed. Drive BMR/targets:
  age: string;
  height: string;
  weight: string;
  goalWeight: string;
  activityLevel: ActivityLevel;
  calorieMode: "static" | "net";            // "net" = burned calories add to today's budget

  // Optional reuse + history:
  savedFoods?: SavedFood[];
  savedMeals?: SavedMeal[];
  weightLog?: WeightEntry[];                // optional weight-over-time (Progress tab)

  // Optional US Navy tape measurements (inches) — onboarding calibration only:
  waistIn?: number;
  neckIn?: number;
  hipIn?: number;

  // Feature E — tailored rolling weekly plan:
  plan?: PlanState;

  // Long-term Coach memory (durable facts, saved/removed via tool):
  coachMemory?: CoachMemory[];

  // Optional onboarding/calibration + progress photos (local URIs only; base64 never persisted):
  currentFrontPhotoUri?: string;
  currentSidePhotoUri?: string;
  goalPhotoUri?: string;
  photoLog?: BodyPhotoEntry[];              // Feature F1 timeline
  photoBannerDismissedAt?: string;

  // Feature F2 — real body-composition scan log (DEXA/InBody/other), numeric only:
  bodyCompLog?: BodyCompEntry[];
  lastBfTrendSurfacedAt?: string;           // gates BF%-trend re-pestering (>=14 days)

  // User-set macro override (may sit below BMR floor — Settings forces explicit confirm;
  // the Coach's set_targets tool still refuses sub-floor):
  customTargets?: {
    calories: number; protein: number; carbs: number; fat: number; fiber: number;
  };
  customTargetsSetAt?: string;
};
```

### `DayLog`

```ts
type DayLog = {
  date: string;                  // "YYYY-MM-DD"
  flow?: "spotting" | "light" | "medium" | "heavy";  // set => this is a period (bleed) day
  energy?: "low" | "medium" | "high";
  moods?: string[];              // multiple moods (plural)
  symptoms?: string[];
  digestion?: string[];
  note?: string;
};
```

> Note: `DayLog` is a **cycle check-in**, not a food container. Food lives in `Profile.foodLogs` as `FoodEntry[]` per day — there is no `DayLog.entries` field, and `mood` is plural (`moods`). The old "informal food notes" idea was replaced by structured `FoodEntry` records when Feature C shipped.

### `ChatMessage`

```ts
type ChatMessage = {
  role: "user" | "assistant";
  content: string;
  date?: string;        // "YYYY-MM-DD" the message belongs to (day dividers + scoping)
  imageUri?: string;    // local uri of an attached meal photo, shown in the bubble
  imageBase64?: string; // transient: JPEG sent to the vision model; stripped before persist
};
```

> Note: messages carry an optional `date` (the chat spans multiple days), not an epoch `ts`. `imageBase64` is transient and stripped before persistence.

Persist `Profile` (with its keyed `dayLogs`/`foodLogs`/`waterLogs`/`workoutLogs` maps) as one object, plus the chat conversation and a rolling summary (see §5). The API only ever sends the last `MAX_HISTORY` (16) messages verbatim.

### Cycle phase math (shipped — `wren/lib/cycle.ts`)

> Updated 2026-05-30 to match shipped code. The original single-anchor sketch
> (one `lastPeriodStart` modulo `avgCycleLength`) is gone — phase is now driven
> by the user's **logged** period days, with each past cycle measured by its own
> real length.

`PhaseInfo = { phase: string; dayOfCycle: number | null; onBirthControl: boolean }`.

**Source of truth = logged starts, not the legacy anchor.** Period (bleed) days are
every `dayLogs` entry whose `flow` is set, sorted ascending. Consecutive bleed days
are grouped into ranges; the first day of each range is a **cycle start**
(`cycleStarts(p)`). The legacy `lastPeriodStart` is used only as a fallback while no
check-in marks a period, so existing users keep a working phase.

**Observed average (`observedCycleLength`).** Gaps between consecutive cycle starts
are filtered to plausible values (15–60 days) so a single mis-tap can't wreck the
average; the average of the last up to 6 gaps is the observed cycle length. With no
usable history it falls back to `p.avgCycleLength` (or 28).

**`phaseForDate(p, date)` / `currentPhase(p, today = now)`:**
1. **Birth-control branch:** if `p.onBirthControl`, return
   `{ phase: "on birth control", dayOfCycle: null, onBirthControl: true }` — no cycle math runs.
2. With **no logged starts** (and no fallback), return
   `{ phase: "unknown", dayOfCycle: null, onBirthControl: false }`.
3. Find the most recent cycle start on or before `date`. Dates **before the first
   logged period** return `phase: "unknown"` — the app does not claim to know.
4. For a **completed** cycle (a later start exists), the cycle's length is the real
   gap to the next start (floored at 15) and `dayOfCycle = daysSince + 1` exactly.
   For the **ongoing/future** cycle (no next start yet) the projected
   `observedCycleLength` is used and the day wraps by that length so upcoming phases
   still render.
5. Phase windows scale with the cycle's own length (`scale = len / 28`):
   `menstrual` ≤ 5·scale, `follicular` ≤ 13·scale, `ovulatory` ≤ 16·scale, else `luteal`.

Predictions (`nextPredictedPeriod`, `isPredictedPeriodDay`) and the "previous cycle"
calendar tint (`isPreviousCycle`) all derive from the same logged starts + observed
average, and all return null/false on birth control.

---

## 5. Claude integration

> Updated 2026-05-30 to match shipped code (`wren/lib/coach.ts`).

- **Model IDs in use:**
  - `claude-haiku-4-5` (`HAIKU`) — default for routine chat (cheap).
  - `claude-sonnet-4-6` (`SONNET`) — complex coaching, the daily opener, food logging, vision/photo turns, likely durable-fact (memory) disclosures, and plan/summary generation.
  - `claude-opus-4-7` (`OPUS`) — reserved for the ONE-OFF onboarding photo body-composition calibration call only (`calibrateFromPhotos`), never normal chat.
- **Model routing (`pickModel`):** default Haiku; escalate a turn to Sonnet when the message is long (> 200 chars), trips the memory-trigger regex (allergies/intolerances, dietary identity like vegetarian/vegan/dairy-free, explicit remember/forget, injury phrasing), or contains coaching/logging signals (why, plan, macro, calorie, protein, target, feel, stress, anxious, sad, depress, tired, exhaust, advice, should i, explain, struggle, ate/eat/had, breakfast/lunch/dinner/snack, drank, water, log). Any message with a **photo always goes to Sonnet** (vision + better food estimates).
- **Where the call runs (prototype):** directly from the app with the key in local config (`EXPO_PUBLIC_ANTHROPIC_API_KEY`) — *fine for solo testing on your own phone.* **Graduation trigger:** before anyone else installs it, move to a Supabase Edge Function (Feature H). Keep the **$20/mo cap** set in the Anthropic console.
- **Prompt caching:** the (large, static) system prompt is sent as a **cached** block (`cache_control: ephemeral`); the volatile per-request context block is sent uncached after it.
- **Context window + continuity:** the API only ever sends the last `MAX_HISTORY` (16) messages verbatim. Older messages that age out are folded into a single bounded **rolling summary** (`SUMMARY_BATCH` = 8; summarized by a cheap Haiku call) that travels in the volatile context for continuity — the chat UI still renders the full history.
- **Context assembled and sent every call** (this IS the "scoped memory"):
  - her name; today's date/time
  - computed cycle phase/day (or "on hormonal birth control — no natural cycle"), and a gentle, non-medical next-period prediction when known
  - today's check-in (flow / energy / moods / symptoms / digestion / note)
  - her goal, activity level, body fields, and dietary rules/preferences
  - code-summed daily targets, consumed totals, and remaining (never recomputed by the model)
  - today's food log entries, water, and workouts; this week's plan by day
  - long-term memory facts; body-composition context (and, when gated, the F2 trend "ARMED" block)
  - on the daily opener only: the deterministic "WHAT TO NOTICE TODAY" block
  - the rolling summary of aged-out messages, when present
  - *(nothing else — no other days' raw logs dumped, no other users)*

---

## 6. Coach system prompt

> Updated 2026-05-30 to match shipped code (`wren/lib/coach.ts`). This is the load-bearing safety surface. Do not soften without flagging the change for review. The original "draft" prompt below has been superseded — the authoritative prompt lives in code. The full persona prompt is long; this section documents the **safety contract that actually ships**, not the whole prompt verbatim.

The shipped chat system prompt is a long persona + behavior block. Its safety-critical pieces are kept in **shared constants** so they cannot drift between surfaces:

- **`ED_SAFETY_RULES`** — the hard ED/medical guardrails (verbatim below). Appended to the **end of every LLM system surface that produces free text the user reads**:
  1. the chat `SYSTEM_PROMPT`,
  2. the plan generator `PLAN_SYSTEM`,
  3. the photo/vision `PHOTO_SYSTEM`,
  4. the rolling-summary `SUMMARY_SYSTEM` (so the summary that feeds back into context is governed too),
  5. the onboarding `CALIBRATION_SYSTEM`.
  Editing the one constant changes all five at once.
- **`PROACTIVE_ED_SAFETY`** — voice/tone calibration for proactive noticing. Honest noticing (low protein, weight trends she's engaged with, missed workouts) IS coaching; what's restricted is shame, moralizing food as good/bad/clean/dirty, and recommending any sub-floor / restrictive / compensatory intake. Composed alongside `ED_SAFETY_RULES` in the chat prompt.

There is also a **deterministic, non-model backstop** in `wren/lib/safety.ts`: if the user's own words trip a high-confidence crisis pattern (`detectCrisisLanguage`), a warm crisis-resources message is appended to the reply regardless of what the model said. Belt and suspenders — the LLM care-mode and the deterministic surface both fire.

### `ED_SAFETY_RULES` (verbatim from `wren/lib/coach.ts`)

```
SAFETY (overrides everything, but does not make you timid):
- Never recommend calories below her estimated BMR. Never endorse starving,
  purging, fasting for weight loss, earning or compensating for food, or any
  sub-healthy target. If she asks for one, say no plainly and give the safe
  version instead. That is good coaching, not hedging.
- No medical advice, diagnosis, or treatment. You are not a medical provider.
  If something sounds medical, say so in one line and point her to her own
  doctor or provider.
- No supplement-by-name recommendations.
- Never body-shame, attack her body, her weight, or her worth, and never
  moralize about food being "good" or "bad."
- Care mode, only on genuine red flags (language about restricting, purging,
  self-harm, or real distress): stop the coaching push, respond with genuine
  warmth, and gently point her to real support. For eating-disorder concerns,
  mention the National Eating Disorders Association (NEDA) at
  nationaleatingdisorders.org or texting "NEDA" to 741741. If she mentions
  self-harm or suicidal thoughts, gently point her to the 988 Suicide & Crisis
  Lifeline (call or text 988). Do not trigger this for a normal bad day or an
  off-hand comment.
```

### Crisis resources (current, accurate)

- Eating-disorder concerns: NEDA at **nationaleatingdisorders.org**, or text **"NEDA" to 741741** (Crisis Text Line).
- Self-harm / suicidal thoughts: **988** Suicide & Crisis Lifeline (call or text).

The discontinued **NEDA phone helpline** (shut down 2023) is **not** used and must not be reintroduced anywhere — neither in the prompt nor in the deterministic backstop. (The old draft prompt above referenced a generic "NEDA Helpline"; that has been removed.)

### Cycle voice (flat declarative)

The shipped prompt speaks about phases with confidence while staying honest that bodies vary, and ties advice to how she actually feels and what she logs. The old **"many women find…"** hedging framing was **dropped 2026-05-28** for cycle copy; the default voice is flat declarative. Cycle suggestions stay gentle, feel-based, and food-first — never medical advice, never naming specific supplements or doses.

> Voice update (2026-05-30): the "many women find" population hedge is now fully removed from the Coach prompts in `wren/lib/coach.ts` — the chat `SYSTEM_PROMPT` (cycle **and** workout sections), the welcome opener built inline in `coachKickoff`, and the `PLAN_SYSTEM` prompt all use plain declarative phrasing. Flat declarative is the voice across the UI and the prompts; there is no longer a "many women find" inconsistency between surfaces.

---

## 7. Beginner-friendly build steps

Each step ends with something you can *see* working. Lean on Claude Code in Cursor/VS Code.

1. **Tooling (Week 1 of original plan):** Node LTS, `npx create-expo-app wren`, run it on your
   phone via Expo Go. Get Anthropic API key, set $20/mo cap. *See: default app on your phone.*
2. **Coach screen, hardcoded:** build the chat UI with a fake hardcoded reply. *See: you can type
   and a bubble appears.*
3. **Wire Claude:** send the message to Claude (Haiku) with a *simple* system prompt, show the
   real reply. *See: a real AI answer.*
4. **Settings/Goals screen + local save:** the `Profile` form, persisted. *See: it remembers your
   goal after restart.*
5. **Cycle math:** compute phase from profile; show it in the Coach header. *See: "Follicular,
   day 8" up top.*
6. **Assemble real context:** send date + phase + goal + rules into the system prompt (§5/§6).
   *See: Coach references your phase/goal unprompted.*
7. **Today's log:** when you describe food, append to `DayLog`; pass today's entries into context.
   *See: "how am I doing today?" recalls what you ate.*
8. **Suggested prompts + Sonnet routing + full safety prompt.** *See: tapping a prompt works;
   "why does my plan look like this?" gets a thoughtful answer.*
9. **Live in it for a week.** Note what's annoying in your build log → that's your Feature B input.

---

## 8. Test checklist before calling A done

- [ ] Coach knows my goal, today's date, and cycle phase without me restating them.
- [ ] "Log my lunch: chicken and rice" → confirmed + recalled later today.
- [ ] "What should I eat for dinner?" respects my dietary rules.
- [ ] "How am I doing today?" summarizes today's log sensibly.
- [ ] Birth-control mode: no fake cycle phase claims.
- [ ] Safety: a message implying restriction/distress triggers the caring response, NOT a diet plan.
- [ ] Safety: it never suggests calories below BMR and never shames.
- [ ] I used it daily for 7 days and would miss it if it disappeared.
```
