# Wren — Project Plan v2 (Revised)

> **This is a revision.** The original `Flux_Project_Plan.docx` is preserved untouched.
> This version bakes in the decisions made during the May 21, 2026 planning interview and
> reorganizes the build into **independently testable Features (A–K)** so you build and
> validate one thing at a time before moving on.
>
> Owner: Ting Geidel · Cadence: 3–5 hrs/week · **Realistic full-v1 target: 12–18 months**
> (the original 10–12 mo is optimistic for a near-beginner). **But a Coach prototype you
> use daily is reachable in weeks** — and that early win is your best defense against
> burnout, the #1 risk to this project.

---

## 0. What changed from v1, and why

| Topic | v1 plan said | v2 decision | Why |
|---|---|---|---|
| **Cycle science framing** | "Follicular = strength/PRs, luteal = endurance" as fact | **Feel-based: cycle phase is a *prior* your logged energy/symptoms correct over time.** Language is "many women find…" | Performance-periodization evidence is weak/individual (McNulty et al. 2020, *Sports Medicine*, rated low quality). Feel-based is honest, defensible, harder to debunk, and makes the "personalized from real data" claim literally true. Also lowers legal exposure. |
| **Build order** | Stand up Supabase + RevenueCat early | **Prototype-first. Defer auth, cloud DB, payments, and push.** Build a single-user local app you live in. | You're a near-beginner solo-testing yourself first. Learning auth/payments on spec is wasted effort before the product feels right. |
| **First feature** | All 5 tabs in parallel-ish | **Coach first (Feature A).** Everything else hangs off it. | Least code for a beginner to make magic (Claude does the heavy lifting), it's the differentiator, and it can *absorb the other features conversationally* until you build real screens. |
| **Body photos** | Required at onboarding | **Optional + skippable**, prompted gently later. | Heavy, sensitive first-run ask → drop-off. Photos are nearly as sensitive as cycle data. |
| **Food photo logging** | Implies precise calories | **Estimate + always confirm/edit. Numbers framed as approximate.** | Vision can't see grams, oils, hidden ingredients. Honesty preserves trust. |
| **Birth control** | Engine assumes natural cycle | **HC is a logged option in onboarding;** these users get a non-phase-synced (or pill-pack) experience, not fake phases. | Large share of 22–45 women are on HC, which suppresses the natural cycle. |
| **ED safety** | Basic "not medical advice" | **Strict + protective guardrails** baked into every Coach prompt. | Calorie tracking + body photos + AI coach for women sits close to disordered-eating risk. Non-negotiable. |
| **Pricing / RevenueCat** | Decide early | **Defer entirely** until there's a working app + real beta users. | Don't build payment plumbing on spec. |
| **Push / proactive nudges** | In v1 (Phase 3) | **Defer to beta (Feature J).** | Real complexity (scheduling, tone tuning) for little prototype value. |

**Goal of the project (your words): all three** — learn modern mobile dev, build something you
personally use daily, and *maybe* grow it into a side business. The build order below serves
all three by front-loading the thing you'd use every day.

---

## 1. Vision (unchanged in spirit, sharpened in claim)

Wren builds a personalized training and nutrition plan from where you actually are — your body,
your cycle, your data, what you want — and keeps it tuned as your body and cycle change.

**Positioning (revised):**
- Tagline candidates: *"Train with your cycle, not against it."* (marketing) — but **inside the
  app the Coach speaks in feel-based, "many women find…" language, never prescriptions.**
- The honest engine: **cycle phase is a starting prior; your logged energy, symptoms, and
  results correct it.** That *is* the personalization.
- Primary audience: women 22–45 who feel generic apps don't fit them.

**What Wren is NOT:** a body-composition scanner, a medical app, a social network, a general
fitness app. (Same as v1.)

---

## 2. The Cycle Engine (reframed)

Same phase map as v1 — but treated as **defaults to be personalized**, not rules.

| Phase | Days (28-day) | Typical training lean | Typical nutrition lean |
|---|---|---|---|
| Menstrual | 1–5 | Lower intensity, mobility, recovery | Iron-rich foods |
| Follicular | 6–13 | Many feel strongest; strength/intensity | Higher carbs for output |
| Ovulatory | 14–16 | Often peak energy | Sustained calories, hydration |
| Luteal | 17–28 | Endurance, steady-state, lower volume | More protein/fat, magnesium for symptoms |

**Rules of the reframe:**
- Windows scale proportionally to the user's stated cycle length.
- Every suggestion is phrased "many women find…" and **always overridable**.
- The app **learns the user's actual pattern** from logged energy/symptoms and shifts the prior.
- **Birth-control users**: onboarding asks. If on HC, no fake phase syncing — they get a steady
  plan (optionally aligned to pill-pack week if they want), and the Coach knows not to attribute
  things to a natural cycle they don't have.

---

## 3. Feature breakdown — build & test independently

Each feature is a self-contained slice you can finish, *use*, and validate before starting the
next. Features A–F are the local prototype. G–K are the "make it a real product" phase.

> **Test gate for every feature:** "I (or a tester) used this for a real week and it did the job
> without me touching code." Don't start the next feature until the current one passes.

### 🟢 Local prototype (no accounts, no cloud, no payments)

**Feature A — Context-aware Coach (THE WEDGE).** *Detailed spec in `Wren_Prototype_Spec_FeatureA_Coach.md`.*
A chat screen powered by Claude that knows your goals, today, and your cycle phase, with strict
ED-safe guardrails. Minimal cycle math + a simple local store so the Coach actually knows things.
You log meals and ask questions by *talking to it*.
*Test gate: you use the Coach daily for a week and it feels genuinely helpful.*

**Feature B — Proper cycle tracking + phase math.** Onboarding for last-period date, average
length, **and the birth-control option**. Compute current phase/day, store history, "many women
find…" explanations. Coach reads this instead of the placeholder from A.
*Test gate: phase shown matches a manual calc across a full cycle; HC path works.*

**Feature C — Structured logging (food / workout / water).** Manual food search seeded from
**Open Food Facts**; workout log (sets/reps/weight, mark complete); water tracking. Replaces
"tell the Coach" with real structured data the Coach can read.
*Test gate: a day's meals + a workout logged and recalled accurately.*

**Feature D — Photo food logging.** Snap a meal → Claude vision best-guess → **you confirm/edit**
→ save into C's data model. Numbers framed as approximate.
*Test gate: 10 real meals logged; edits are fast; you trust the flow.*

**Feature E — Home + weekly Plan.** Today's plan card, phase indicator, "why today looks like
this" note, progress rings (calories/protein/workouts). Claude generates a 4–5 workout + nutrition
target week tuned to phase, refreshed weekly. (Home + Plan tabs.) **✅ Deterministic macros were
pulled forward into Feature A** (`wren/lib/targets.ts` — Mifflin-St Jeor BMR → TDEE, goal-adjusted,
hard BMR floor in code; identical every run, fixed a trust bug where tone changed the numbers).
Feature E now just *refines* them: explicit unit pickers for height/weight (replace the tolerant
text parsing), optional body-fat %, cycle-phase calorie cycling, and surfacing targets as a card with progress rings.
*Test gate: a generated week makes sense and you'd actually follow it.*

**Feature F — Progress.** **Optional/skippable** body photo + new-photo prompts every 4 weeks,
cycle calendar, workout consistency, biometric input (DEXA/InBody/blood/RHR — all optional).
*Test gate: 4 weeks of history renders correctly; photos truly optional.*

### 🟡 Make it a real product (infra + business — deferred until A–F feel right)

**Feature G — Accounts + cloud sync (Supabase).** Move local store → Supabase auth + Postgres +
storage. **This is where you learn auth/DB.** Encryption at rest for cycle data, working
delete-everything button, strict per-user context isolation.
*Test gate: two separate accounts never see each other's data; delete actually deletes.*

**Feature H — Coach hardening for multi-user.** Move Claude calls behind a server-side function
(Supabase Edge Function) so your API key never ships in the app. Re-verify context isolation.
*Test gate: key not present in the shipped bundle; each call sends only that user's context.*

**Feature I — Beta polish.** Visual polish, bug fixing, onboarding refinement, TestFlight build.
*Test gate: 10–20 beta users onboard without you on a call.*

**Feature J — Notifications / proactive nudge.** Expo Notifications, **one** opt-in daily nudge at
user-chosen time, "helpful neighbor not drill sergeant" tone templates, cycle-aware care (no
"you've been undereating protein" during PMS).
*Test gate: nudge fires on time, feels supportive, easy to turn off.*

**Feature K — Monetization.** RevenueCat, 7-day trial, $9.99/mo or $59.99/yr. **Revisit price
only after beta feedback.** No freemium.
*Test gate: a real test purchase + restore works on iOS and Android.*

Then: **Launch** — legal/privacy review (see §6), App Store + Play submission, landing page,
marketing (see §7).

---

## 4. Tech stack (with prototype-first nuance)

Same destination as v1, **introduced later**:

| Layer | Tool | When it enters |
|---|---|---|
| Mobile | **React Native + Expo** (you're on a Mac ✓) | Feature A |
| Language | **TypeScript** | Feature A |
| AI | **Anthropic API** — Haiku for routine chat, Sonnet for complex coaching | Feature A |
| Local storage | Expo SQLite or AsyncStorage (no cloud) | Feature A–F |
| Food data | **Open Food Facts** | Feature C |
| Backend/Auth/DB/Storage | **Supabase** | Feature G (not before) |
| Server-side AI proxy | **Supabase Edge Function** (hides API key) | Feature H |
| Push | **Expo Notifications** | Feature J |
| Subscriptions | **RevenueCat** | Feature K |
| Analytics / Errors | PostHog / Sentry (free tiers) | Feature I |
| Design / Editor | Figma · Cursor or VS Code + Claude Code | Throughout |

**API key safety, honestly:** For *solo testing on your own phone in Expo Go*, putting the key in
local config is acceptable risk — it never reaches anyone else. **Graduation trigger:** the moment
a second person installs the app (Feature H/I), move the Claude call behind a Supabase Edge
Function so the key lives server-side. Set a **$20/mo spend cap** in the Anthropic console now.

---

## 5. Roadmap (re-cut around features, near-beginner pace)

| Stage | Rough timeline @ 3–5 hr/wk | Features | Tangible result |
|---|---|---|---|
| Prototype | Months 1–4 | A → (B, C) | A Coach you live in daily that knows your cycle and logs your food |
| Prototype+ | Months 5–8 | D, E, F | The full single-user app, local-only |
| Productize | Months 9–13 | G, H, I | Multi-user, cloud-synced, in beta on TestFlight |
| Launch-ready | Months 14–18 | J, K + legal | Notifications, payments, privacy review, submission |
| Launch & iterate | 18 mo+ | — | Public launch, marketing, v2 planning |

This is honest, not aspirational. If life gets busy, **A alone is still a real, useful thing.**

---

## 6. Legal & Privacy (unchanged priority — still non-negotiable)

Cycle data is sensitive health data. Carried over from v1, with one clarification on local-only:

- **Local-only mode resolution (was an open conflict):** in the **prototype phase the app is
  local-only by default** (everything stays on device), which sidesteps the conflict entirely.
  When you add cloud sync (Feature G), local-only becomes a *choice* — and users who pick it
  accept that **cloud-dependent AI features run on transient, non-persisted context** (or are
  reduced). Decide the exact behavior at Feature G, not now.
- Cycle data **encrypted at rest** in Supabase (Feature G+).
- **Never sell/share/ad-target** cycle data. Ever. (Remember Flo's FTC settlement.)
- **Delete-everything button** that truly works — built *with* Feature G, not bolted on.
- Comply with **Washington's My Health My Data Act** (strictest US law).
- **Privacy policy reviewed by a real attorney before launch** (~$500–1500). The line item I'd
  most caution against skipping.
- Content rules: never diagnose, never give medical advice ("many women find…" not "you should"),
  no supplement-by-name recs in v1, always include a "discuss with your provider" line.

---

## 7. Monetization & Marketing (monetization deferred; **marketing pulled forward**)

- **Monetization (Feature K):** $9.99/mo or $59.99/yr, 7-day full-access trial, **no freemium**,
  target 70% annual. Optional post-launch lifetime deal ($199, first 500). **Revisit price after
  beta** — don't lock it now.
### Marketing track (M0–M3) — pulled forward, runs in parallel with the build

Marketing is the biggest risk after "never finishing," so it is **no longer deferred to launch** —
it runs as a parallel workstream starting now, mapped to the roadmap stages in §5. The asset that
makes this possible early: **the app already works**, so "build-in-public" and "asked my Coach"
content is available *today*, not someday.

| Stage | Maps to roadmap | Deliverables | Effort |
|---|---|---|---|
| **M0 — Plant** (now) | Prototype (A–C done) | Claim handle; start a **content-only TikTok/IG**; stand up a **simple waitlist landing page** with email capture (Carrd/Framer + a form). | ~30 min/wk |
| **M1 — Cadence** | Prototype+ / Productize | 2–3 posts/wk; build the content muscle; first ~1k engaged followers; build-in-public + anonymized "asked my Coach" clips. | ~1–2 hr/wk |
| **M2 — Funnel** | Productize (beta on TestFlight) | Point content → waitlist → beta invites; collect testimonials/retention proof; target **~2–5k engaged followers**. | ~2 hr/wk |
| **M3 — Convert** | Launch-ready | Convert waitlist on launch day; launch posts, relevant communities/Product Hunt; turn the best beta quotes into proof. | launch sprint |

**Content pillars:** phase-specific training, phase-specific nutrition, myth-busting,
build-in-public, "asked my Coach" clips (anonymized).

**Content guardrails (inherit the app's ethos — non-negotiable):** marketing carries the **same
ED-safety and no-medical-claims rules as the Coach.** No before/after body-transformation bait, no
calorie-restriction or weight-loss-number hooks, no diagnosis-flavored claims ("fix your hormones").
Two reasons: it's right for this audience, **and Apple reads your screenshots/marketing during App
Review** — confident medical/weight claims are a top rejection trigger for this category. The
honest, education-first voice is also the safest one.

**Success metric (echoes §10):** ~2–5k engaged followers **and** a warm waitlist by launch day —
so launch isn't shouting into the void.

---

## 8. 🅿️ Parking lot — explicitly NOT in beta (so we don't forget)

Everything here is intentionally out of the prototype/beta. Pull from this list only when the
feature it depends on has passed its test gate.

**Deferred infra/business (have a home above):**
- Cloud accounts & sync → **Feature G**
- Server-side API key proxy → **Feature H**
- Push notifications & the one daily proactive nudge → **Feature J**
- Payments / RevenueCat / trial / pricing → **Feature K**

**Deferred product features (v1.5 / v2):**
- Apple Watch / Oura / Garmin / Whoop auto-sync
- Barcode scanning
- Real-time advisor loop (workout adjusts nutrition mid-day or vice versa)
- Supplement-by-name suggestions (needs legal review first)
- Men's mode / open audience (the "circadian & recovery-aware" reframe)
- Sleep tracking (depends on wearables)
- Social / community features
- Apple Health integration beyond strictly necessary
- Workout *video* library (text + image plans only at first)
- Coach multi-week memory beyond ~4 weeks of cycle history (v1.5)
- Coach noticing complex longitudinal patterns ("you skip Wednesdays in luteal") (v1.5)
- More than one proactive nudge/day (v1.5+, needs careful tuning)

**Deferred decisions (revisit at the noted stage):**
- Final app name + trademark check (before launch)
- Brand palette + typography (Feature I)
- Local-only-mode exact behavior with cloud AI (Feature G)
- Pricing validation / final price (after beta)
- V2 wearable priority — Apple Watch vs Oura first (let beta users decide)

---

## 9. Risk register (carried forward, top 3 for *you*)

1. **Energy bleed / never finishing.** Mitigation: Feature A is small and usable on its own —
   ship that first, keep a build log every session.
2. **Scope creep.** Mitigation: the parking lot in §8 is where ideas go to wait. Don't pull
   forward.
3. **Marketing built too late.** Mitigation: the two cheap early bets in §7.

Plus the Coach-specific risks from v1 (low engagement, nudges feeling naggy, bad advice, memory
leaks across users) — all addressed by the strict guardrails (Feature A spec) and per-user
isolation (Feature G).

---

## 10. Success metrics (unchanged, honest)

- **Prototype success:** *you* use Feature A daily for 4+ weeks and miss it when it's gone.
- **Beta:** 10–20 users active 4+ weeks; ≥5 say they'd pay; Day-7 retention >40%.
- **Year one post-launch:** floor 50 paying users (~$3–6k), real 200–500 (~$12–30k),
  stretch 1000+ ($60k+). Under 50 after a full year of marketing → pivot or sunset gracefully.

---

*This is a living document. Update it as you learn. Original preserved at `Flux_Project_Plan.docx`.*
