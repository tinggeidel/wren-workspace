# Wren audit log

Appended by `wren-auditor` after every implementer change. Each entry is timestamped and references the change being audited.

Severity legend: **BLOCK** (stop, fix before continuing) · **WARN** (fix soon) · **NOTE** (worth knowing).

---
## 2026-05-25T00:00:00Z — Full-state audit (no single diff; first recorded audit, baseline review against Feature A spec)

# Audit report — 2026-05-25 — Full-state review of Flux app vs specs

## Summary
First recorded audit (audit log was empty). The app has grown beyond the Feature A spec into Features B/C/E (cycle check-ins, food/water/workout logging, photo + barcode food logging, tailored weekly plans, progress/weight). Coach safety prompt is intact and strong; birth-control branching is clean. 2 BLOCK, 5 WARN, 6 NOTE. Most serious issue: a per-day calorie target that can drop below the BMR floor.

## Findings

Coach safety prompt:
- [NOTE] lib/coach.ts:37-79 — System prompt substantially rewritten vs spec §6, but all required hard rules present and intact (no medical advice/provider referral, no sub-BMR, no starving/purging/fasting/earning/compensating, no body-shaming, ED care-mode + NEDA helpline, no supplement-by-name, feel-based cycle framing). Spec §6 is now stale; update it to match shipped prompt.
- [NOTE] lib/coach.ts:485-489 — System prompt attached to every conversational call; cached static block + uncached volatile context. Good.
- [WARN] lib/coach.ts:605-608, 694-709 — Two extra Claude paths (estimateFoodFromPhoto/PHOTO_SYSTEM, generateWeekPlan/PLAN_SYSTEM) use narrow prompts without full ED-safety guardrails. Photo path forced to a tool call (safe). Plan path emits free-text coaching (whyThisWeek) and lacks explicit BMR/ED-care lines. Factor hard-safety block into a shared constant and append to PLAN_SYSTEM.

ED-trigger surfaces:
- [BLOCK] lib/plan.ts:138-217 — Per-day calorie cycling can drop target BELOW BMR. dayTargets() multiplies the BMR-floored base by INTENSITY_FACTOR (rest 0.9, light 0.95) with no floor re-check; for a user already at the BMR floor, rest day = BMR×0.9 < BMR. Comment "never below ~90% of it" is itself the bug. These numbers are presented as the daily GOAL (coach.ts:139-141, Food tab). Violates "NEVER recommend calories below BMR." Fix: re-apply Math.max(calories, bmr) inside dayTargets (expose BMR from targets.ts) and clamp the rest-day multiplier.
- [NOTE] screens/CoachScreen.tsx:62 — Suggested prompts safe/feel-based. No earn/deficit/burn-off copy in UI (grep confirmed; such strings appear only as prohibitions in the prompt).
- [NOTE] screens/ProgressScreen.tsx:160-177 — Weight tracking present (out of Feature A scope). Framing careful: optional, weight-down green / up grey, no before/after photos, consistency keyed to workouts + calories-vs-target. Acceptable; watch as it grows.

Birth-control branch:
- [NOTE] Clean end to end: cycle.ts:124-125, coach.ts:86-96, CoachScreen.tsx:362-363, CycleScreen.tsx:207-216/359, SettingsScreen.tsx:251, generateWeekPlan coach.ts:785-786. No phase/day leak; nothing reaches Coach context.

Scope creep:
- [WARN] App spans Features B/C/E, all listed out-of-scope for Feature A in spec §2. Likely intentional given code maturity; confirm with Ting and update spec/CLAUDE.md scope language.
- [WARN] lib/food.ts:229-366 — Live network calls to Open Food Facts (search + barcode), a second destination beyond api.anthropic.com. Sends only query/barcode, no key, low privacy risk, but breaks "local-only/no cloud" framing. Needs Ting sign-off; document as accepted exception if approved.
- [NOTE] No auth/payments/push/telemetry/social. No analytics SDKs in deps or imports. app.json has no notification plugins.

Data model drift:
- [WARN] lib/types.ts:10-48 — Profile/DayLog/ChatMessage drifted well beyond spec §4 (tone, dayLogs, foodLogs, waterLogs, savedFoods/Meals, weightLog, workoutLogs, biometrics, calorieMode, plan; Goal+tone_up; DayLog repurposed to cycle check-ins). Migration IS handled (storage.ts:13-28 backfills maps + migrates legacy periodLog). Deliberate + migrated, so WARN not BLOCK. Update spec data model.
- [NOTE] storage.ts — Migration covers maps, calorieMode default, periodLog. New biometric strings not explicitly initialized but treated as optional downstream (safe). No data-loss path; saves preserve existing logs.

Dependency/version drift:
- [WARN] AGENTS.md vs package.json — KNOWN discrepancy confirmed: AGENTS.md says Expo SDK v56, package.json declares expo ^54.0.34 (react 19.1.0, RN 0.81.5 = SDK 54). Reconcile (upgrade SDK or fix AGENTS.md to v54). Re-check when either changes.
- [NOTE] lib/coach.ts:22 — Haiku ID "claude-haiku-4-5" (unpinned) vs spec's pinned "claude-haiku-4-5-20251001"; Sonnet "claude-sonnet-4-6" matches. Pin or confirm alias.
- [NOTE] Imports vs declared deps clean (async-storage, datetimepicker, expo-camera, expo-image-manipulator, expo-image-picker all declared+used).

API key handling:
- [NOTE] coach.ts:30 — No hardcoded key (grep clean); from EXPO_PUBLIC_ANTHROPIC_API_KEY, .env gitignored. Caveat: EXPO_PUBLIC_* inlines into the client bundle, so the key ships in any distributed build. Documented solo-testing-only w/ Feature H graduation trigger. Fine for solo; do not distribute a build.

TypeScript hygiene:
- [NOTE] CycleScreen.tsx:300-301 local any[] style arrays (cosmetic).
- [NOTE] Anthropic call error handling solid (postMessages res.ok check, CoachScreen.deliver catch, kickoff graceful degrade, per-tool try/catch). Implementer should run tsc to confirm clean type-check.

Cost guardrails:
- [WARN] coach.ts:217-224 — Routing drifted toward Sonnet vs spec "Haiku default": pickModel escalates on ate/eat/had/meal-words/log/water, so the core daily food-logging path now hits Sonnet; photos + daily kickoff force Sonnet. Defensible for estimate quality but raises cost per call; $20/mo cap is the backstop. Confirm intentional with Ting. Prompt caching present; max_tokens 1500/1024/4000 reasonable.

## Not checked
- No tsc/expo/lint/runtime/visual checks (read-only, no Bash) — implementer to verify on device.
- Did not verify model aliases resolve to valid live IDs (no network).
- Did not exhaustively scan node_modules for transitive networked/analytics packages (checked declared deps + source imports).
- Did not fully read FoodScreen.tsx / WorkoutScreen.tsx / lib/image.ts — sampled via lib modules + Coach context; future pass should review those screens' copy (esp. how Food tab presents remaining + "net" budget).
- Flux_Project_Plan_v2.md / Flux_Trademark_Knockout.md not present at root (only Flux_Project_Plan.docx, unreadable as text, and the Feature A spec) — project-plan-level scope intent not cross-referenced.

---
## 2026-05-25T00:00:00Z — Follow-up audit: sub-BMR per-day calorie cycling fix (resolves prior 2026-05-25 BLOCK)

# Audit report — 2026-05-25 sub-BMR per-day calorie cycling fix (follow-up to prior BLOCK)

## Summary
Clean — the prior BLOCK is resolved. 0 blockers, 0 warnings, 3 notes. The per-day cycled target can no longer fall below BMR for any intensity, the refactor is behavior-preserving, and the change is correctly scoped.

## Findings

### 1. BLOCK resolved — per-day target floored at BMR for all intensities — CONFIRMED
flux/lib/plan.ts:209-222 (dayTargets). Lowest INTENSITY_FACTOR is rest: 0.9. Line 216 clamps with the raw un-rounded computeBMR return: floored = Math.max(base.calories * f, bmr). Rounding happens AFTER clamping (line 217), so no multiplier (rest 0.9, light 0.95, moderate 1.0, hard 1.08) can push the cycled target below true BMR (residual <=~5 kcal is from round-to-10 only, identical to the pre-existing base-target rounding gap at targets.ts:99 — cosmetic, not a sub-BMR cycling bug). This path feeds the Coach context via targetForDate -> dayTargets at coach.ts:138, where numbers are presented verbatim. Spec §6 line 157 ("NEVER recommend daily calories below her estimated BMR") and §7 checklist line 204 satisfied. The hundreds-of-kcal-under-BMR cycling defect is gone. BLOCK genuinely resolved.

### 2. Refactor behavior-preserving — CONFIRMED no regression
flux/lib/targets.ts:58-104. computeBMR reproduces the original inline female Mifflin-St Jeor exactly (line 63: 10*kg + 6.25*cm - 5*age - 161). Null guard equivalent to before (line 72: !kg || bmr == null subsumes the old age/cm/kg checks). Protein still depends on the separate line-70 parseWeightKg (unchanged). Floor (line 91), rounding (99-103), macro derivation byte-for-byte unchanged. Consumers ProgressScreen:42, SettingsScreen:93, coach.ts:776, plan.ts:171 unaffected — no signature change.

### 3. Fallback path (bmr null, base non-null) — UNREACHABLE, safe by construction
plan.ts:216 falls back to un-floored base.calories * f when computeBMR is null. dayTargets returns null first if computeTargets fails (line 210), and computeTargets returns null whenever computeBMR is null (71-72). computeBMR is pure/deterministic over the same profile, so the line-215 call cannot return null if line 210 succeeded. Branch is dead code — defensible defensive guard, not a hazard. Even if it executed, base.calories is BMR-floored and f>=0.9, so worst case >=0.9*BMR. NOTE-level only.

### 4. Rounding order and carb consistency — CORRECT
plan.ts:216-220: clamp (raw BMR) -> round to 10 -> derive carbs from rounded-clamped calories. Carbs use post-clamp calories (220) with Math.max(0,...) guard. Protein/fat carried from base unchanged ("protein stays put, difference moves through carbs"). Floor raises rest/light calories vs buggy version, so carbs only go up, never negative.

### 5. Scope / Coach prompt / BC / deps — CLEAN
Change confined to targets.ts and plan.ts (import line 20 + comment). No new files, networked deps, fetch/axios, auth/payments/push/telemetry. Coach system prompt untouched. Birth-control branch untouched (functions don't read onBirthControl). No package.json impact. No new any / unhandled promises. (Standing flux/AGENTS.md v56 vs SDK 54.0.34 discrepancy unrelated to this change, remains open from prior audits.)

## Notes
- NOTE plan.ts:216 — bmr-null fallback is dead code (unreachable). Harmless now; would become a ~10%-under-BMR rest-day exposure if dayTargets is ever decoupled from computeTargets.
- NOTE targets.ts:70-72 — parseWeightKg invoked twice per call (once direct, once in computeBMR). Negligible, deterministic, no correctness issue; protein's kg dependency confirmed intact.
- NOTE both targets.ts:99 and plan.ts:217 round to 10 AFTER clamp, so displayed value can be up to ~5 kcal below true BMR. Pre-existing, symmetric, cosmetic. Use Math.ceil if strict "never a single kcal below BMR" is wanted. Not required to clear the BLOCK.

## Not checked
- Did not run tsc (no Bash) — implementer to confirm clean compile; no risky type constructs introduced.
- Did not verify on-device rendering of cycled numbers (FoodScreen/ProgressScreen); data-layer values correct, device smoke test (rest day, BC + non-BC profile, displayed kcal >= BMR) is implementer's.
- Did not re-verify full prior-audit BLOCK list — scoped to the sub-BMR BLOCK plus touched lines, as requested.

---
## 2026-05-25T00:00:00Z — Correctness & breakage-risk pass (post-churn; not ED-safety)

# Audit report — 2026-05-25 Correctness & breakage-risk pass (post-churn)

## Summary
Decent, defensible shape despite heavy churn. 1 BLOCK (unguarded profile load → possible white-screen on launch for corrupt/legacy data), 5 WARN (concurrency/staleness in save paths + 1 dead completion-tracking branch), handful of NOTE cruft. Will not break easily in normal single-user use; the BLOCK is the real crash path; the documented "wipe bug" class is closed everywhere checked. (Orchestrator note: independently verified `tsc --noEmit` exit 0 and `expo export --platform ios` clean — 630 modules, 2.09 MB.)

## Findings

Crash / runtime risk
- [BLOCK] lib/storage.ts:11 (+ App.tsx:49-54) — loadProfile does JSON.parse(raw) with NO try/catch; App.tsx calls loadProfile().then(...) with NO .catch. Corrupt/truncated/legacy flux.profile → unhandled rejection, setLoading(false) never runs → stuck spinner / white screen, no recovery. Contrast: loadChat (storage.ts:43-48) DOES try/catch. Fix: try/catch the profile parse returning null (first-run), add .catch on the load effect to clear loading.
- [NOTE] lib/storage.ts:8-30 — Maps/arrays well-defaulted; scalar fields (goal/tone/activityLevel/name) not defaulted, but all consumers tolerate via ?? / optional chaining. Low risk; harden by defaulting enums if desired.
- [NOTE] No other unguarded JSON.parse; all map/filter/reduce go through ?? []/{} helpers; computeTargets/computeBMR/targetForDate/currentPhase/planDayForDate return null cleanly and consumers null-check; nextPredictedPeriod has guard<1000 cap; ProgressScreen ±Infinity only used inside length>=2 guards. Solid area.

Data-model migration safety
- [WARN] No schema version field — periodLog→dayLogs migration + map defaults work, but future shape changes have no anchor. Add a version int on next data-model change.
- [NOTE] Profile has grown far past spec §4 (spec ChatMessage.ts:string vs live date?/imageUri?/imageBase64?). Expected B/C/E growth, not drift; mark lib/types.ts as canonical, spec is stale.
- [GOOD] Wipe-bug class closed: SettingsScreen.buildProfile (66-90) preserves all sibling maps from initial; App passes live profile as initial; every screen uses {...profile,<field>} spreads via pure helpers.

Internal consistency
- [GOOD] All food writes funnel makeEntry/entryFromSaved→addEntry (same shape, respects selDate). Totals summed in code; calorie-hallucination bug stays fixed.
- [GOOD] All workout writes funnel makeWorkout→addWorkout. Weekday mapping consistent (Mon-anchored) across weekdayKey/mondayOf/dateForWeekday/planDayForDate.
- [WARN] WorkoutScreen.tsx:237-256 (commitDay) & 273-325 (syncStrengthLog) read render-time `profile` prop, not a ref; same for FoodScreen persist paths (logMeal 351-356, savePhotoItems 455-476) and CycleScreen.persist. With all tabs always mounted, a Coach write landing between renders → last-writer-wins, silently reverts the Coach write. CoachScreen avoids this via profileRef. Fix: thread writes through a shared ref/updater or centralize persistence in App.

State / effect hazards
- [GOOD] Kickoff effect (CoachScreen 322-359): empty deps, active-flag cleanup, all-tabs-mounted design prevents re-fire on tab switch (documented App.tsx:75-76). No duplicate-log risk.
- [WARN] CoachScreen mixes `messages` state + profileRef. handleCoachScan (420-447) and handleClear append off closure `messages` snapshot; if a multi-step deliver() turn is in flight, scan/clear append can overwrite the in-flight reply on saveChat (chat-message loss). Fix: functional setMessages + single in-flight lock for all append paths.

Dead / inconsistent code
- [WARN] PlanExercise.loggedEntryId (types.ts:270) read but NEVER written. dayLogged (plan.ts:72-76) has a dead per-exercise clause; move_workout_day clears it (CoachScreen 302) but nothing sets it — leftover from abandoned per-exercise-entry architecture. Currently harmless (day.loggedEntryId covers real cases) but a trap: weekReviewSummary (plan.ts:151-152) counts done/skipped only off d.loggedEntryId, ignoring e.done, so a half-checked strength day reports fully done once sync sets day.loggedEntryId. Delete dead field/branch; make dayLogged/weekReviewSummary agree on one "done" definition.
- [NOTE] Unused-only cruft (no inconsistent refs, no risk): food.ts:51-56 mealsFor(), food.ts:82-88 guessMeal(), types.ts:138-147 MealType/MEAL_LABELS/MEAL_ORDER + FoodEntry.meal? (180); FoodScreen styles mealBlock/mealTitle (1470-1478). NOTE: workouts.ts setsLabel() is NOT dead (used by exerciseLabel:187) — memory hint stale. Safe to delete for hygiene.

TypeScript hygiene
- [NOTE] CycleScreen.tsx:300-301 const cellStyle/txtStyle: any[] — localized styling any. No as any/as unknown as in app code; tool inputs typed + runtime-validated in the runner.

## Not checked
- ED-safety / Coach prompt / BC copy / scope / key handling not re-audited (prior 2 audits, out of scope). Incidental: BC phase-label suppression intact (CycleScreen, currentPhase); key still from EXPO_PUBLIC_ env, no hardcoded sk-ant-.
- Visual rendering, modal-stacking setTimeout timing (FoodScreen 230-241/333-336), camera/barcode need device test.

---
## 2026-05-25T00:00:00Z — Audit of three fixes: profile-load crash guard, dead-code removal/unify-done, chat-loss race fix (response to 2026-05-25 breakage-risk audit; save-race refactor deliberately deferred)

### Summary
Clean. All three fixes correct and behavior-preserving; no BLOCK, no WARN, no regression introduced. Two NOTEs plus standing open items. (Orchestrator independently verified: tsc --noEmit exit 0, expo export --platform ios clean.)

### Fix 1 — profile load crash guard (flux/lib/storage.ts:8-37, flux/App.tsx:49-61)
- PASS: try opens before JSON.parse (line 14/15) and closes after the full normalization+periodLog migration (line 33); a throw mid-normalization is caught and returns null → onboarding. Spinner always cleared.
- PASS: App.tsx clears loading in both .then and .catch.
- PASS (clobber checked): loadProfile is read-only; corrupt blob stays on disk; no silent auto-save between load-null and first user action, so no data-loss. Overwrite only happens when user chooses to onboard fresh.

### Fix 2 — dead-code removal + unify "done" (flux/lib/types.ts:262-270, flux/lib/plan.ts:73-76,149-161)
- PASS: PlanExercise.loggedEntryId removed from types.
- PASS (behavior-preserving): grep confirms every remaining loggedEntryId is the day-level PlanDay field; the exercise-level field was never written, so the removed .some(...) clause was always false. weekReviewSummary done/skipped now via dayLogged() = !!day.loggedEntryId, functionally identical.
- PASS: move_workout_day still clears day-level loggedEntryId + exercise done flags. No other file reads the deleted field.

### Fix 3 — chat-loss race fix (flux/screens/CoachScreen.tsx)
- PASS (no desync): setMessages appears only in the useState decl (87) and inside appendMessages (100); every write routes through appendMessages which updates messagesRef + state together. Kickoff/deliver/scan/clear all use appendMessages.
- PASS: in-flight guard (sending || booting) on handleCoachScan (435), handleClear (506), and re-checked in the Clear-confirm onPress (518). Append paths build on messagesRef.current, not the render closure.
- PASS: idle scan/clear/send unaffected. Kickoff "new day w/ history" branch sets sending=true so guards still protect during async kickoff.

### Standard quick checks
- PASS Coach safety prompt: coach.ts untouched; SYSTEM_PROMPT intact, matches spec §6 (BMR floor, no fasting/earning, no medical advice + provider note, no supplement-by-name, NEDA/care-mode ED branch, no body-shaming); attached with cache_control to chat call; all fetch sites → api.anthropic.com.
- PASS ED-trigger copy: no new strings; SUGGESTED chips/headers unchanged and safe.
- PASS Birth-control branch: untouched; "On birth control" label and BC context handling intact; no phase/day leaks.
- PASS Scope: no new deps, no new network code, no auth/push/payments/telemetry.
- PASS TS hygiene: no new any; new try/catch + .catch are deliberate recovery handlers.

### NOTEs
- NOTE-1 storage.ts:34-36: corrupt-profile path is graceful degradation, not recovery — user silently sent to onboarding with no signal that unreadable data exists on disk. Acceptable for single-user prototype.
- NOTE-2 CoachScreen.tsx:97-101: messagesRef fix relies on the invariant "never call setMessages directly"; add a guarding comment/lint so a future edit doesn't reintroduce desync.

### Open items carried forward (not introduced here)
- WARN (deferred by design): larger save-race persistence refactor not done; runTool/deliver/scan/clear still call saveProfile/saveChat independently with no serialized write queue.
- NOTE (standing): EXPO_PUBLIC_ANTHROPIC_API_KEY ships in client bundle (coach.ts:27-30), guarded only by the GRADUATION TRIGGER comment; solo-testing only.
- NOTE (standing): AGENTS.md now consistently pins SDK 54; previously-flagged v54-vs-v56 discrepancy resolved. Re-flag only if package.json or AGENTS.md changes.

### Not checked
- Runtime/device behavior (cannot run app). Implementer to verify: (1) corrupt flux.profile blob → onboarding, no white screen, spinner clears; (2) Send-then-Clear/scan during in-flight reply → reply not lost, history not wiped; (3) tsc --noEmit clean after PlanExercise.loggedEntryId removal.

---
## 2026-05-25T00:00:00Z — dead-code cleanup (removed meal-time-slots leftovers)

# Audit report — 2026-05-25 dead-code cleanup (removed meal-time-slots leftovers)

## Summary
Clean. Zero blockers, zero warnings. Verified no-op at runtime; every removed symbol has zero remaining references and nothing live was removed. One NOTE on dayAllChecked. (Orchestrator independently verified: tsc --noEmit exit 0, expo export --platform ios clean, 630 modules.)

## Findings
- [NOTE] flux/lib/plan.ts:61 — dayAllChecked(day: PlanDay) is exported but has zero references in the entire repo. Only match is its own definition. Independently confirmed genuinely dead. Not removed in the meal-slot round (since handled separately afterward).

## Verification detail (all passed)
1. Nothing live removed. Greps across flux/ for each removed symbol returned zero: mealsFor, guessMeal, MealType, MEAL_LABELS, MEAL_ORDER, mealBlock/mealTitle styles — all 0. Unrelated must-stay symbols intact: SavedMeal (type), saveMeal/removeSavedMeal, savedMeals (Profile field, storage, FoodScreen, SettingsScreen). FoodSource still includes the string-literal "meal" union member (types.ts:140), distinct from the removed MealType type — correctly left in place.
2. Data-model safety of dropping FoodEntry.meal?: no meal?: field declaration remains; no .meal access on any FoodEntry; consumedTotals reduces only over calories/protein/carbs/fat; all readers use id/date/macros only. Old stored entries with a meal value are safe (structural non-exact types ignore the extra prop). No migration needed. storage.ts untouched.
3. No behavior change: cosmetic-only. No logic, no write path, no Coach prompt, no Anthropic call, no dependency touched. package.json unchanged.
4. ED-safety / Coach-prompt / BC: no new copy in the touched files; pure deletions of unused type/style declarations. No earn/burn/deficit framing. BC logic not in change set. No scope creep.

## Side observation (resolved, no action)
- Expo SDK cross-check: flux/AGENTS.md now reads "pinned to Expo SDK 54" and package.json declares expo ^54.0.34 — they agree; previously-flagged discrepancy resolved.

## Not checked
- Visual rendering of FoodScreen after orphaned-style removal (low risk, zero styles.mealBlock/styles.mealTitle usages).

---
## 2026-05-25T00:00:00Z — Remove dead `dayAllChecked` helper from flux/lib/plan.ts

### Summary
Clean — 0 blockers, 0 warnings. Behavior-preserving dead-code deletion, fully verified. (Implementer verified tsc --noEmit exit 0 + expo export clean, 630 modules.)

### Findings
- [NOTE] flux/lib/plan.ts — `dayAllChecked` removed; project-wide grep across flux/ returns zero references. No dangling call sites; build not broken by a stale import.
- [NOTE] flux/lib/plan.ts:6-18 — `PlanDay` still imported and consumed by live functions (dayExercises, isTrainingDay, dayLogged, planDayToWorkoutEntry, planDayForDate). Import block not wrongly pruned; no other symbol dangling.
- [NOTE] All external importers of ./plan (lib/coach.ts:18, screens/CoachScreen.tsx:38, screens/WorkoutScreen.tsx:59, screens/FoodScreen.tsx:25, screens/ProgressScreen.tsx:18) import only still-existing symbols; none imported dayAllChecked.
- [NOTE] Behavior-preserving confirmed: remaining helpers (dayExercises, dayLogged, weekProgress, isWeekComplete, targetForDate, dayTargets, planDayForDate, weekReviewSummary, planDayToWorkoutEntry) intact. No Coach prompt, Anthropic call, data-model, or dependency change.
- [NOTE] ED-safety / BC / scope pass: hard BMR floor in dayTargets (Math.max(base.calories * f, bmr) re-clamp) untouched; cycle math (./cycle, ./targets) not modified; no networked deps; no key-handling change.

Verdict: trivially clean. Safe to commit.

---
## 2026-05-25T00:00:00Z — ED-safety hardening: 1200 kcal floor + uniform ED_SAFETY_RULES + deterministic crisis backstop

# Audit report — 2026-05-25 ED-safety hardening: calorie floor + uniform safety prompt + crisis backstop

## Summary
Clean — 0 blockers, 2 warnings, 4 notes. Both changes strengthen safety. The prompt extraction is a strict superset of the prior rules (corrections + additions only, nothing dropped or softened), the calorie floor holds for base and every cycled intensity, and the crisis backstop is wired deterministically through the safe append/save path in both success and error branches. (Orchestrator: implementer verified tsc exit 0 + expo export clean, 631 modules; safety self-check 10/10.)

## Findings

### Change 1 — calorie floor
- [NOTE] targets.ts:99 / plan.ts:211 — Math verified. computeTargets returns null when bmr==null (line 79), so max(calories, bmr, 1200) always >= max(BMR,1200). dayTargets re-clamps max(base.calories*f, bmr ?? 0, 1200); all four intensities (rest 0.9 / light 0.95 / moderate 1.0 / hard 1.08) — sub-1.0 factors caught by the 1200 and BMR terms. bmr ?? 0 fallback safe. Carbs derived from clamped calories in both functions, Math.max(0,…), non-negative. No regression.
- [NOTE] Consumers unaffected — Targets/Macros shapes unchanged (ProgressScreen:42, SettingsScreen:93, FoodScreen:96 via targetForDate, food.ts remaining()).

### Change 2 — uniform safety prompt + crisis backstop
- [NOTE] coach.ts:43-48 — Item 1 (not weakened): PASS. Every spec §6 / prior inline hard rule survives in ED_SAFETY_RULES (BMR floor; no starving/purging/fasting/earning/compensating; not-a-medical-provider/refer-out; no supplement-by-name; never body-shame + no moralizing; ED care-mode + real support). Deltas are corrections/additions only.
- [NOTE] safety.ts:22-23 / coach.ts:48 — Item 2 (resource accuracy): PASS. 988 (call/text), 741741 (text NEDA), nationaleatingdisorders.org. No invented numbers. Discontinued NEDA phone helpline fully gone (grep-confirmed).
- [NOTE] coach.ts:620,723 — Item 3: PASS. PLAN_SYSTEM and PHOTO_SYSTEM both interpolate the exact ${ED_SAFETY_RULES} constant, not a partial copy.
- [WARN] safety.ts:45-81 — Item 4 (detector): FP guarding solid (starving/killing me/dying for coffee/threw up hands excluded). Conservative by design -> WARN not BLOCK. High-value FN gaps worth adding: "I can't go on", "don't want to wake up", "better off dead/without me", "no reason to live", "overdose"/"OD on", bare "throwing up again", reflexive "starving myself". [Addressed in follow-up round below.]
- [NOTE] CoachScreen.tsx:380-424 — Item 5 (wiring): PASS. crisis computed once from userMsg.content (model-independent). crisisMsg appended after the Coach reply in BOTH success (399-405) and error (408-420) paths, single appendMessages + single saveChat on array from messagesRef.current — no re-introduced race. Augments (not replaces) the normal reply; user msg still sent to askCoach so model engages care mode.
- [NOTE] safety.ts:22-23 — Item 6 (wording): PASS. Warm, non-clinical, non-alarming, non-diagnosing. Minor: "pause the coaching" first-person two-voices sequencing; cosmetic.
- [NOTE] safety.ts — Item 7 (scope/deps/data model): PASS. Dependency-free (no imports). No package.json change, no network calls, no Profile/DayLog/ChatMessage shape change. BC branch (cycle.ts 124/167/184, coach.ts 96-97/799-800) untouched.

### Standing items
- [WARN] AGENTS.md vs package.json — package.json pins expo ^54.0.34 (correct). AGENTS.md reads SDK 54 throughout (corrected earlier this session). Discrepancy resolved; no code impact.
- [NOTE] coach.ts:30 — Pre-existing, out of scope: Anthropic key ships in client bundle via EXPO_PUBLIC_ (documented graduation trigger). No sk-ant- literal committed.

## Not checked
- On-device runtime of the crisis backstop (no Bash). Send a triggering message and confirm care-mode reply + resources bubble + persistence across restart + error-path resources when API unreachable.
- runSafetyAssertions() reviewed for logical consistency, not executed. [Orchestrator later executed it: returns [] / ALL PASS.]

---
## 2026-05-25T00:00:00Z — Crisis-detector false-negative extension (safety.ts patterns + runSafetyAssertions only)

# Audit report — 2026-05-25 — Crisis-detector false-negative extension (safety.ts patterns + self-check)

## Summary
Clean for safety purposes — 0 BLOCK, 0 WARN, 3 NOTE. All five verification requirements pass. The previously-flagged false-negative gaps (prior ED-safety hardening round) are genuinely closed; the four named must-be-FALSE illness/hyperbole cases all stay FALSE under the actual regexes; no harmful new false positives. (Orchestrator executed runSafetyAssertions() → [] / ALL PASS; tsc + expo export confirmed clean by implementer.)

## Findings

### 1. No regression — CONFIRMED
- Original patterns unchanged at flux/lib/safety.ts:47-61, 78-87, 99-102, 108-109; additions interleaved, not substituted.
- detectCrisisLanguage uses CRISIS_PATTERNS.some(...) (OR), so adding patterns can only make more inputs TRUE, never fewer — a prior-TRUE phrase cannot regress to FALSE.
- Five original TRUE assertions + original hyperbole FALSE assertions remain and still hold.

### 2. New TRUE match / new FALSE stay false — CONFIRMED (reasoned vs regex + normalization)
Normalization: lowercase → non-[a-z0-9\s]→space → collapse. So can't→can t, I've→i ve, don't→don t.
TRUE: "overdose on my pills" (:73); "thinking about overdosing" (:75); "i ve been/i m starving myself" (:107); "throw up after every meal" (:93); "make myself vomit to lose weight" (:78); "throw up to get rid of the food" (:93).
FALSE (named cases): "throwing up, I think I have the flu" — no cue after "up" → FALSE. "I keep vomiting, probably food poisoning" — vomit patterns need space+cue; "vomiting" has no space → FALSE. "I overdosed on caffeine this morning" — needs token+" on"+med noun; "overdosed on" breaks adjacency, caffeine not a med → FALSE. "I'm starving what's for dinner" — requires literal "myself" → FALSE.

### 3. No new harmful false positives — CONFIRMED, two over-broad NOTEs
- [NOTE] safety.ts:96 — /to get rid of (?:the |…)/ fires on any "get rid of the ___" (clutter/headache/leftovers), broader than the accepted "can't go on" FP. Consequence: gentle message only, never a hard block; rare in fitness chat → NOTE. Direction: drop bare "the " branch, keep food/calorie/meal/"what I ate" branches.
- [NOTE] safety.ts:97 — /to undo (?:the |…)/ same bare-"the " over-breadth ("undo the damage/change"). Same suggested tightening.
- Verified NOT over-broad: od alternatives require word-boundary od + med noun/intent verb (no "god"/"good" collision); "threw up my hands" stays FALSE ("my" not a cue); "can t go on" also matches "we can't go on" — already-documented accepted FP class.

### 4. Normalization consistency — CONFIRMED
- New apostrophe-bearing patterns use space-substituted form: "can t go on", "don t want to wake up", "i m"/"i ve been" alternatives. All fire under real normalize() output. No silently-dead pattern.

### 5. Scope / Coach prompt / floor / data-model / BC — CONFIRMED clean
- Only flux/lib/safety.ts changed; dependency-free (zero imports). Sole consumer CoachScreen.tsx:393-395 unchanged. No Coach prompt, no BMR/1200 floor, no Profile/DayLog/ChatMessage change → no migration. No onBirthControl/cycle touch. No network/deps/package.json/any/API-key impact.

## Not checked
- On-device runtime of crisis backstop bubble — wiring unchanged, covered by prior audit; not re-run.

---
## 2026-05-25T00:00:00Z — Long-term Coach memory (remember_fact/forget_fact tools, lib/memory.ts store, Settings view/delete)

# Audit report — 2026-05-25 Long-term Coach memory

## Summary
Not clean: 0 blockers, 3 warnings, 4 notes. ED-safety of the memory surface is sound and ED_SAFETY_RULES is intact and remains the final authority in SYSTEM_PROMPT — no BLOCK. Warnings: a stale-state wipe race on coachMemory in Settings, a forget_fact fuzzy-match that can delete the wrong fact, and the pre-existing SDK version note. (Two WARNs addressed in follow-up round below.)

## Findings

### 1. ED-safety of the memory surface (priority 1) — PASS, no blocker
- ED_SAFETY_RULES (lib/coach.ts:44-49) unchanged and still the FINAL block of SYSTEM_PROMPT (lib/coach.ts:97), after the new LONG-TERM MEMORY section (lines 90-95). Safety remains last authority. Deployed version is a hardened superset with correct crisis resources (988, NEDA text 741741, no discontinued NEDA phone line).
- New LONG-TERM MEMORY section carries an explicit ED guardrail (lib/coach.ts:95); remember_fact tool description repeats the SAFETY clause (lib/coach.ts:459) — defense in depth.
- Failure-mode: a harmful stored fact can't bypass safety — facts load into the VOLATILE context block (lib/coach.ts:228) below SYSTEM_PROMPT; ED_SAFETY_RULES opens "SAFETY (overrides everything)". detectCrisisLanguage backstop is memory-independent.
- User deletion (SettingsScreen.tsx:314-335) is a genuine transparent off-ramp but the ONLY remediation; no detection of an unsafe stored fact. Acceptable for solo prototype (see NOTE).

### 2. Coach prompt integrity — PASS
- SYSTEM_PROMPT otherwise intact (voice, macros/BMR-floor 77/79, cycle, workouts, daily flex); memory block purely additive. PLAN_SYSTEM (758) and PHOTO_SYSTEM (655) still compose ${ED_SAFETY_RULES} as final block. Safety attached to every Anthropic call site (chat/kickoff 531, photo 705, plan 883).

### 3. Data-model migration safety — mostly PASS, one WARN
- CoachMemory + Profile.coachMemory? additive/optional (types.ts:51,55). loadProfile normalizes missing/malformed -> [] inside try/catch (storage.ts:25); no crash, no data loss.
- [WARN] SettingsScreen.tsx:68,94 — Stale-state wipe race on coachMemory. Other preserved fields read live `initial` prop; coachMemory reads local state seeded once via useState(initial?.coachMemory ?? []) which doesn't refresh on prop change. Settings open while Coach saves a fact -> Save persists stale snapshot, drops new fact. Bounded (needs Settings open across a Coach memory write). Fix: read coachMemory from live prop in buildProfile, local state only for delete UI. [FIXED in follow-up.]

### 4. Tool wiring + no regressions — mostly PASS, one WARN
- remember_fact/forget_fact wired in runTool (CoachScreen.tsx:330-361) follow the fixed concurrency pattern (profileRef.current, await saveProfile, onProfileChange); never touch messages/appendMessages, so chat-append race not reintroduced. remember_fact reports dedupe no-op via reference compare (336).
- [WARN] CoachScreen.tsx:349-354 — forget_fact bidirectional substring match (t.includes(q) || q.includes(t)) can remove an unintended fact. q.includes(t) lets a long query containing a short stored fact (e.g. "no dairy") delete a real restriction/allergy; removes first match, no confirmation. Fix: drop q.includes(t), require exact/unambiguous match. [FIXED in follow-up.]

### 5. addMemory correctness — PASS
- Case-insensitive trimmed dedupe (memory.ts:27-28), blank skip (24), cap MAX_COACH_MEMORY=40 oldest-front eviction (31), returns new profile, no mutation. runMemoryAssertions covers dedupe/blank/cap+eviction/remove incl. unknown-id no-op (84-105).

### 6. Context/cost — PASS
- memoryLines injected into buildContextBlock (228) = UNCACHED block (533, no cache_control), separate from cached SYSTEM_PROMPT (531). Memory edits don't bust cache. Included in chat, plan (856), kickoff. Empty memory omits section (memoryLines returns "" line 44, guarded both call sites).

### 7. Settings UI — PASS (with item 3 WARN)
- Delete persists immediately (handleForget -> removeMemory -> saveProfile + onSaved, 119-124). Empty hint + per-row ✕ with accessibilityLabel.

### 8. Scope / deps — PASS
- lib/memory.ts dependency-free beyond newId (food.ts:66) + toISODate (cycle.ts:14). No new npm packages. No networking/auth/payments/push/telemetry. No BC/cycle logic touched (buildContextBlock 104-114, planContext 834-838 unchanged).

## Notes
- [NOTE] SDK: AGENTS.md pins SDK 54 and package.json declares expo ^54.0.34 — AGREE; prior v56 discrepancy already resolved.
- [NOTE] ED-safety remediation is deletion-only/detection-free; backlog: a deterministic deny-list/heuristic on addMemory for bare calorie/weight targets.
- [NOTE] newId() module-counter + Date.now(); IDs only compared within the profile array — no practical collision.
- [NOTE] Memory eviction at the 40-cap is silent. Benign for now.

## Not checked
- Could not run tsc/app/runMemoryAssertions() (read-only). [Orchestrator note: implementer ran tsc exit 0 + runMemoryAssertions ALL PASS + expo export clean, 632 modules.]
- On-device rendering/delete interaction of the Settings memory section.

---
## 2026-05-25T00:00:00Z — Coach memory: stale-state wipe race fix (SettingsScreen) + forget_fact over-broad match fix (CoachScreen)

# Audit report — Coach memory correctness fixes

## Summary
Clean. 0 blockers, 0 warnings, 2 notes. Both fixes do what they claim; no safety, scope, data-model, or cost regressions. (Implementer verified tsc exit 0 + expo export clean.)

## Findings
- [NOTE] screens/SettingsScreen.tsx:124-128 — handleForget calls onSaved(next); App.tsx:111 onSaved runs setProfile + setTab("coach"), so forgetting one fact navigates out of Settings to the Coach tab. Same handler as Save (intentional), but surprising for a delete-one-row action. Verify on device.
- [NOTE] screens/SettingsScreen.tsx:124-128 — handleForget builds from live form state via buildProfile(), so a forget also commits any unsaved edits to other Settings fields without pressing Save. Memory delete itself is correct/persists; flagging the coupling.

## Verification
Fix 1 (race closed): no mount-time useState snapshot of memory remains; coachMemory derived live as initial?.coachMemory ?? [] (SettingsScreen.tsx:70); no setCoachMemory anywhere. buildProfile persists initial?.coachMemory ?? [] (:96), mirroring dayLogs/foodLogs/waterLogs/workoutLogs/savedFoods/savedMeals/weightLog/plan (:88-95) — none disturbed; wipe guard holds. Re-render wiring: CoachScreen onProfileChange->setProfile (App.tsx:89) -> initial re-passed (App.tsx:109) -> list re-renders. Coach-saved fact while Settings mounted is preserved on next save because buildProfile reads live prop.
Fix 2 (dangerous path removed): q.includes(t) gone (CoachScreen.tsx:351-354); match is exact else m.text.includes(q). "stop telling me to avoid dairy" vs "no dairy" -> no deletion. Ambiguous (>1) deletes nothing + reports ambiguity (:359-360); zero -> nothing to forget (:357-358); exactly-one -> deletes by id (:361-366); echo-back preserved. Only remaining auto-delete is single-match substring (intended self-correction), single-match-gated; no over-broad path remains.
No regressions: CoachMemory/Profile types and lib/memory.ts unchanged; storage migration storage.ts:25 present. ED_SAFETY_RULES (coach.ts:44-49) intact and last in SYSTEM_PROMPT/PHOTO_SYSTEM/PLAN_SYSTEM; prompt attached to every Anthropic call. BC/cycle untouched. Forget routes through profileRef/saveProfile, not messages/saveChat — no chat-append race. Cost guardrails intact. No new deps/network. No dangling setCoachMemory; useState + CoachMemory imports still valid/used; no new any.

## Not checked
- On-device behavior of the two NOTE items (tab-jump after forget; committing unsaved field edits on a forget).
- tsc not run by auditor (read-only); implementer verified exit 0.

---
## 2026-05-25T00:00:00Z — SettingsScreen per-row Coach memory delete (onProfileChange wiring)

# Audit report — SettingsScreen per-row Coach memory delete

## Summary
Clean. No blockers, no warnings. The two prior NOTEs (tab-jump after forget; committing unsaved field edits on a forget) confirmed resolved. (Implementer verified tsc exit 0 + expo export clean.)

## Findings
- [RESOLVED] screens/SettingsScreen.tsx:129-134 — handleForget no longer navigates away; calls onProfileChange?.(next) (wired to setProfile only at App.tsx:115), not onSaved/setTab. User stays on Settings.
- [RESOLVED] screens/SettingsScreen.tsx:131 — handleForget no longer commits unsaved form edits; removes from the live `initial` prop, not buildProfile().
- [NOTE] screens/SettingsScreen.tsx:74,131-133 — Re-render path sound. coachMemory derived from live prop; removeMemory -> saveProfile -> onProfileChange(setProfile) -> App.tsx re-passes initial={profile} -> .map re-renders with the row gone. Same pattern as CoachScreen.tsx:362-365.
- [PASS] screens/SettingsScreen.tsx:130 — `if (!initial) return;` guard sound; theoretical-only no-op, avoids passing null to removeMemory.
- [PASS] "Save & go to Coach" unchanged (buildProfile -> saveProfile -> onSaved -> setProfile+setTab). onProfileChange optional; onboarding instance (initial=null) unchanged and safe via optional chaining.
- [PASS] lib/memory.ts removeMemory used correctly (pure, returns new Profile, unknown id no-op).
- [PASS] Scope/regression: no data-model change; no Coach prompt / ED_SAFETY_RULES change; no ED-trigger copy; no birth-control/cycle change; no chat-append race; no new deps; no any; saveProfile awaited; Profile typing intact.

## Not checked
- On-device delete interaction (row disappears in place, no tab jump, unsaved edits not persisted by a delete).

---
## 2026-05-25T00:00:00Z — Coach memory reliability (strengthened LONG-TERM MEMORY prompt + MEMORY_TRIGGERS Sonnet routing) and keyboard-covering-input fix

### Summary
Clean on safety. 0 blockers, 2 warnings, 2 notes. Files: flux/lib/coach.ts, flux/screens/CoachScreen.tsx. Root cause of "memory not appearing": the model acknowledged without calling remember_fact (worsened by Haiku routing) — wiring/persistence/display were all correct.

### Findings
- [WARN] flux/lib/coach.ts:245 — MEMORY_TRIGGERS is broad and OR'd into pickModel's complex check on top of the already-broad food/eat/why/plan clause. Common non-durable phrases route to Sonnet (\bi like\b, \bi love\b, \bi hate\b, \bat home\b, \bi have (a|an)\b, bare \bknee\b/\bshoulder\b, \bwedding\b, \bi work\b). Risk: higher Sonnet share = higher cost vs §5 Haiku-default intent. Not a correctness bug (always returns a valid model; existing escalations still fire). If cost matters, tighten the loosest alternations; otherwise acceptable for a solo prototype as a deliberate reliability-for-cost tradeoff.
- [WARN] flux/lib/coach.ts:251 — Net effect is a deliberate shift of more turns to Sonnet to make remember_fact fire reliably (Haiku under-fires proactive tool calls). Bounded: Haiku still default for simple messages, prompt caching on SYSTEM_PROMPT intact (line 543), max_tokens 1500 and history cap 16 unchanged. $20/mo cap remains the backstop.
- [NOTE] flux/screens/CoachScreen.tsx:356,368 — New console.log lines print remembered/forgotten fact text to the Metro console. Local-only, no crash risk; acceptable for solo debugging. Remove before sharing any build.
- [NOTE] flux/AGENTS.md vs package.json:9 — Expo SDK discrepancy RESOLVED: AGENTS.md states SDK 54, package.json pins expo ^54.0.34. They agree. Standing item cleared.

### Verification detail
1. ED-safety prompt integrity — PASS. ED_SAFETY_RULES (coach.ts:44-49) unchanged, matches spec §6, STILL final block of SYSTEM_PROMPT (line 98). New LONG-TERM MEMORY section (90-96) preserves its ED-safety bullet (96); strengthening is tool-mechanics-only ("saying I'll remember does nothing — must call remember_fact"). remember_fact tool description (469-471) retains no-goal-weight/no-restrictive-intent SAFETY clause. No other rule altered.
2. pickModel — PASS (cost WARN). Returns SONNET or HAIKU only; new clause additive; existing escalations intact.
3. Keyboard fix — PASS. Listener (127-133) registers once, cleans up via sub.remove() (no leak), only calls scrollToEnd (no setState/render loop). keyboardVerticalOffset (insets.top iOS / 0 else) + behavior logic unchanged; send/photo/scan row, kickoff, messagesRef/appendMessages, in-flight guards untouched. useSafeAreaInsets valid (react-native-safe-area-context declared, package.json:17).
4. Debug logging — harmless; no PII off device.
5. Scope/regressions — PASS. No data-model change, no new deps, no BC/cycle change (BC header label intact), no new chat-append race, remember/forget still route through profileRef + saveProfile + onProfileChange. Only network destination api.anthropic.com; no key exposure.

### Not checked
- Runtime keyboard behavior on device (iOS/Android offset + scroll timing).
- Live confirmation Sonnet fires remember_fact for new triggers — observable via the new Metro logs during the week-long test.
- Actual Sonnet/Haiku traffic mix + cost — not statically measurable; watch the Anthropic console vs the $20/mo cap.

---
## 2026-05-25T00:00:00Z — CoachScreen suggested-prompt chips hide on input focus

### Summary
Clean — 0 blockers, 0 warnings, 1 note. The change is exactly the three pieces described and touches nothing safety-, scope-, BC-, data-, or cost-relevant. (Implementer verified tsc exit 0 + expo export clean, 632 modules.)

### Findings
- [NOTE] screens/CoachScreen.tsx:688-701 — Hiding suggestRow on focus removes its layout height exactly as the keyboard animates in (iOS behavior="padding"). Cosmetic only; may cause a single-frame layout shift. No functional impact, no guard affected.

### Verified clean
- Send/photo/scan row, onSubmitEditing, kickoff effect, messagesRef/appendMessages, and every `sending || booting` guard unchanged (CoachScreen.tsx:449-622, 703-728).
- Chip tap still routes to send(s) with unchanged disabled={sending || booting} (694-695); chips reappear via onBlur (717). Input row sits outside the conditional, so it can never be hidden.
- No Coach system-prompt or ED_SAFETY_RULES content in this file (both in lib/coach.ts, not touched). Crisis backstop (detectCrisisLanguage / CRISIS_RESOURCES_MESSAGE) untouched.
- Birth-control phase-label branch (440-445) untouched; no dayOfCycle leak.
- No new imports/deps/network/auth/telemetry/data-model changes. No new any/unhandled promises. inputFocused is local state, no effect dependency or leak.

### Not checked
- On-device keyboard animation / cosmetic layout shift (the NOTE above).

---
## 2026-05-25T00:00:00Z — CoachScreen suggested-chip scroll-hide (Change A) + suggested-prompt text (Change B)

# Audit report — CoachScreen suggested-chip scroll-hide + prompt-text changes

## Summary
Clean. No blockers, no warnings. Two NOTEs (one informational, one a known prior discrepancy unrelated to this change). (Both implementers verified tsc exit 0 + expo export clean, 632 modules.)

## Findings

### Change A — chips hide when scrolled up
- [OK] screens/CoachScreen.tsx:136-141 — Near-bottom math correct: contentOffset.y + layoutMeasurement.height >= contentSize.height - 48 treats within-48px-of-end as bottom. Scrolled up -> false -> chips hide; at end -> true -> chips show.
- [OK] screens/CoachScreen.tsx:140 — setAtBottom((prev) => prev === near ? prev : near) flips only on change (React bailout); no per-frame setState storm / no jank from scrollEventThrottle=16.
- [OK] screens/CoachScreen.tsx:106 — Default atBottom=true, chips show on open.
- [OK] screens/CoachScreen.tsx:673-679 — onContentSizeChange scrolls to end and asserts setAtBottom(true), covering programmatic scrolls that don't emit onScroll (reply, kickoff, cleared+rebooted) so chips aren't stuck hidden after sending.
- [OK] screens/CoachScreen.tsx:147-157 — keyboard-show listener scrolls to end and asserts setAtBottom(true); with !inputFocused gating, keyboard path consistent.
- [OK] screens/CoachScreen.tsx:671-672 — onScroll handler additive: only reads e.nativeEvent and sets one boolean; doesn't call scrollToEnd, mutate messagesRef, or touch sending/booting. No interference with append/persist, kickoff, scan, or in-flight guards.
- [NOTE] screens/CoachScreen.tsx:137 — updateAtBottom recreated each render and NEAR_BOTTOM local const; harmless (plain ScrollView, no memoized child). Not worth fixing.

### Change B — suggested prompt text
- [OK] screens/CoachScreen.tsx:70 — SUGGESTED now ["What should I eat?", "How am I doing today?"]; pure two-string swap. Chip rendering and onPress send(s) unchanged; sent verbatim.
- [OK] ED-safety of new copy — no earn/deficit/burn/shame/weight-loss framing; both neutral. Removed "I had eggs and toast" was benign.

### Cross-cutting safety / scope checks
- [OK] Coach safety prompt — lib/coach.ts ED_SAFETY_RULES (44-49) and SYSTEM_PROMPT (52-98) untouched; attached to every call (542-545); caching intact (543). All hard rules present. No new bypass path.
- [OK] Crisis backstop — deliver (483-510) still computes detectCrisisLanguage and appends CRISIS_RESOURCES_MESSAGE on success + error paths. Unchanged.
- [OK] Birth-control branch — no cycle math touched; phaseLabel (462-466) shows "On birth control" with no day label; buildContextBlock BC substitution intact.
- [OK] Scope — no new deps/network/auth/push/payments/telemetry. NativeSyntheticEvent/NativeScrollEvent are types from already-imported react-native.
- [OK] Data model — Profile/DayLog/ChatMessage untouched; atBottom is local state, not persisted; no migration.
- [OK] TS hygiene — no new any; updateAtBottom fully typed; synchronous handler; no state leak (keyboard listener cleaned up via sub.remove(), line 156).
- [OK] Cost — no change to model routing, MAX_HISTORY, or per-call tokens. UI-only.
- [NOTE] Pre-existing, unrelated: AGENTS.md pins Expo SDK 54 and is internally consistent (its "55/56" mention is the explanation for the pin). No version drift introduced.

## Not checked
- Visual rendering / flicker on device. Logic supports no-flicker (threshold + flip-only setState).
- tsc not run by auditor; both implementers verified exit 0.

---
## 2026-05-25T00:00:00Z — proactive-Coach: pattern detectors (lib/patterns.ts) + observational daily opener (lib/coach.ts)

### Summary
Clean — 0 blockers, 0 warnings, 4 notes. ED-safety preserved and enforced structurally (data channel cannot carry intake/weight signals); BC branch does not leak into the opener; anti-nag wiring correct (noticings fed only on the opener); no scope/data-model/dependency/cost drift; prompt cache not busted. (Orchestrator executed runPatternAssertions → [] / ALL PASS; implementer verified tsc exit 0 + expo export clean, 633 modules.)

### ED-safety (priority 1) — PASS
- ED_SAFETY_RULES remains the FINAL block of SYSTEM_PROMPT (coach.ts:115-117: ...${PROACTIVE_ED_SAFETY}\n\n${ED_SAFETY_RULES}); new PROACTIVE_ED_SAFETY sits before it. Content unchanged (BMR floor, no fasting/earning/compensating, no medical advice, no supplement-by-name, no body-shaming, care-mode with 988 + NEDA 741741, no discontinued NEDA phone line).
- PROACTIVE_ED_SAFETY (coach.ts:57-60) forbids proactive intake-policing and weight/scale/progress comments; frames under-fueling as eat-ENOUGH-never-less.
- STRUCTURAL enforcement: NoticingKind = energy|symptom|cycle|adherence|fueling (patterns.ts:25). No detector for calorie amount / weight trend / weight-loss progress. Only food-adjacent line is `fueling` (patterns.ts:167-170), hardcoded to eating enough. runPatternAssertions encodes the ED-kind invariant (480-486).
- New PROACTIVE/NOTICING prompt section (coach.ts:109-113) additive; no existing rule weakened/reordered.

### Anti-nag wiring — PASS
- noticingsBlock appended only when forOpener===true (coach.ts:233). Defaults false on buildContextBlock(126)/postMessages(567)/runConversation(619); askCoach never sets it; only coachKickoff passes true (679). Structurally cannot recite noticings on normal turns.

### Detector correctness — PASS
- lowEnergyStreak (55-64): gap/non-low breaks streak; surfaces >=2. recurringSymptoms (68-87): >=2 days over last 7, deduped per day. workoutAdherence (99-131): today excluded from missed walk; rest days don't break; completed breaks; miss-streak (>=2) preferred over strong_run (done>=3 && >=75%). periodHeadsUp (137-145): only 0<=daysUntil<=4. noticingsBlock returns "" when empty; capped MAX_NOTICINGS=4. runPatternAssertions [] confirmed.

### Birth-control integrity — PASS
- periodHeadsUp -> nextPredictedPeriod returns null when onBirthControl (cycle.ts:167-168); cycle noticing cannot fire for BC users; no prediction leak. Assertion patterns.ts:360-361. buildContextBlock BC handling unchanged.

### Kickoff behavior — PASS
- Once-per-day/no re-fire on tab switch: fires only when messages.length===0 || isNewDay (CoachScreen.tsx:427-428) using persisted lastDate (storage.ts:63). History appended, never erased. Pre-existing logic untouched.

### Scope / cost — PASS
- No data-model change (patterns.ts read-only over dayLogs/plan). No new deps (imports only ./types, ./cycle, ./plan). No new network calls. Cache intact: system[0]=SYSTEM_PROMPT (ephemeral); noticings go to volatile system[1] (coach.ts:574), cache not busted per turn. Model routing unchanged; opener still Sonnet.

### Notes (non-blocking)
- [NOTE] AGENTS.md (SDK 54) — standing Expo discrepancy unchanged by this work.
- [NOTE] patterns.ts:235 runPatternAssertions is intentional runtime dead code; ships in bundle; negligible.
- [NOTE] patterns.ts:206 singular "workout" branch unreachable (missed only returns >=2); harmless.
- [NOTE] coach.ts:32 EXPO_PUBLIC_ANTHROPIC_API_KEY in client bundle (solo-testing only, graduation comment); pre-existing accepted condition.

---
## 2026-05-25T00:00:00Z — Conversation-continuity rolling summary (storage.ts ChatStore, coach.ts summarizer + context injection, CoachScreen maybeSummarize)

### Summary
Clean on safety, scope, and data-model. 1 WARN (narrow summary-only persistence race — never drops a message; self-heals on reload). No BLOCKERS. (Orchestrator executed runContinuityAssertions → [] / ALL PASS; implementer verified tsc exit 0 + expo export clean, 633 modules.)

### Findings
- [WARN] storage.ts:93-100 + CoachScreen.tsx:166 — Non-atomic read-modify-write in the cont-omitted saveChat. A concurrent summarizer save can land between the message save's loadChat() read and its setItem write, so the message save overwrites the just-persisted summary/summarizedCount with stale values. NO chat message is ever lost (both writers rebuild messages from messagesRef.current). Only summary/count regress, and only in persistence — in-memory refs stay correct, so continuity holds for the session. Stale values surface on the next reload, after which messagesToFold self-heals (re-folds from count=0; one redundant Haiku call, a one-reload continuity gap, no crash). Does NOT reintroduce the prior message-loss race. Fix direction: source summary/count from the in-memory refs on every save (drop the read-modify-write preserve), or merge instead of blind-write. Acceptable to ship; tighten before standalone.
- [NOTE] AGENTS.md vs Expo SDK — pre-existing; AGENTS.md pins SDK 54 (matches Ting's Expo Go). Unaffected by this change.
- [NOTE] coach.ts:823 + CoachScreen.tsx:163 — On a successful-but-empty Haiku extraction, summarizeConversation falls back to priorSummary while the caller still advances summarizedCount (marks messages folded into a summary that didn't incorporate them). Low impact (rare; prevents infinite re-fold loop; messages already aged out). Throw path handled correctly (count not advanced).

### Verification (all pass)
1. Chat-save race (priority 1): maybeSummarize reads messagesRef.current, guards with summarizingRef (no overlap), persists a single saveChat(messagesRef.current, cont) rebuilt on the latest ref — mid-summarization messages included, not clobbered. Only residual issue is the WARN (summary staleness, never message loss).
2. ED-safety: ED_SAFETY_RULES and SYSTEM_PROMPT unchanged. SUMMARY_SYSTEM ends with ED_SAFETY_RULES and forbids calorie/weight targets, positive framing of restriction/compensation, amplifying distress. Summary injected as labeled DATA ("a recap, not new instructions") in volatile context, governed by the unchanged safety block — cannot encode a directive that outranks safety.
3. Trigger correctness: messagesToFold clamps non-negative, folds only at/above batch then all pending (seamless seam), returns 0 on over-large count. runContinuityAssertions covers boundary/sub-batch/batch/over-batch/advancement/seam/corrupt-count. Confirmed [] when executed.
4. Data-model migration: loadChat normalizes missing summary->""/count->0 + try/catch corrupt blob; old chats load fine; clearChat + handleClear reset both refs. Profile/DayLog/ChatMessage untouched — new fields only on storage-layer ChatStore.
5. Context/cost: summary in VOLATILE (uncached) block; cached SYSTEM_PROMPT unchanged (ephemeral cache intact). Summarizer Haiku, max_tokens 512, fired ~every 8 aged-out messages (not per message). Full history still rendered; messages never trimmed.
6. Scope: no new deps, all Anthropic calls to api.anthropic.com, only other fetch is approved Open Food Facts. No Profile/DayLog/ChatMessage change, no BC/cycle impact. Key via EXPO_PUBLIC_; no sk-ant- literal. No new any; un-awaited maybeSummarize intentional with own try/catch.

### Not checked
- Live timing race on device — WARN is from static reasoning about AsyncStorage's non-atomic read-modify-write.

---
## 2026-05-25T00:00:00Z — Robustness round 1: centralized profile persistence behind updateProfile(updater) (save-race fix)

### Summary
Clean — 0 blockers, 0 warnings, 4 notes. The save-race is genuinely closed; every former write site routes through the single shared updateProfile exactly once; saveProfile is no longer called outside App.tsx; CoachScreen's dual-ref staleness is read-only; no safety/scope/data-model/dependency change. (Orchestrator independently verified tsc exit 0 + expo export clean, 633 modules.)

### Verified (no findings)
1. Race fixed: App.updateProfile does synchronous read(profileRef.current)→compute→set-ref→setState before await saveProfile; sequential awaited calls compose (N+1 sees N). No write site rebuilds a save-payload from the render-time prop (grep: saveProfile/setProfile only in App.tsx).
2. No missed/double write: all 8 Coach tool handlers, ~15 Food paths, Workout plan/log paths, Cycle saveEditor/clearDay, Progress logWeight, Settings save/forget/onboarding each go through updateProfile once; no leftover paired saveProfile/setProfile.
3. Dual-ref: CoachScreen private profileRef is read-only context; sole write source is App's updateProfile (composes each tool transform on the latest ref).
4. Behavior preserved: Settings Save→Coach; memory delete stays on Settings; onboarding creates first profile via initProfile→Coach; clearChat, daily kickoff, messagesRef append-race fix + in-flight guards, safe single-match forget_fact, wipe-guard (sibling maps from latest p) intact.
5. ED-safety/scope: ED_SAFETY_RULES, SYSTEM_PROMPT, system attachment + cache_control on every api.anthropic.com fetch, crisis backstop, BC branch untouched. No new deps/fetch/model-routing/data-model/lib-signature/storage change. lib helpers remain pure transforms.
6. Riskier conversions: toggleExercise(exId) re-finds day+exercise by id (no wrong-day toggle); commitDay/syncStrengthLog operate on passed-in latest prof inside persist transforms; Cycle + Coach log_checkin recompute lastPeriodStart identically; Food edit/delete by entry id on latest p; remember_fact `updated === p` dup detection sound.

### Notes (informational)
- [NOTE] CycleScreen.tsx:173-179 — saveEditor isEmpty truthiness relies on the just-above "only set when length>0" invariant. Pre-existing.
- [NOTE] CoachScreen.tsx:309,359,403 — three reads from the render-behind private profileRef (weight estimate + two plan guards); reads only, writes re-find against latest p; benign cosmetic edge (same-turn create-then-adjust may conservatively refuse, never corrupt).
- [NOTE] App.tsx:126-130 — onboarding updater({} as Profile) safe (buildProfileFromForm nullish-fallbacks; loadProfile re-normalizes next launch).
- [NOTE] AGENTS.md SDK 54 consistent; no dependency change in this refactor.

### Not checked
- Runtime/device race scenario (Coach logs food while Food tab edits an entry) — implementer/Ting to smoke-test. [Orchestrator independently ran tsc exit 0 + expo export clean.]

---
## 2026-05-25T00:00:00Z — Robustness round 2: chat save-race (Fix A), cost routing tighten (Fix B), crisis detector bare-"the" drop (Fix C1), summarizer empty-extraction null (Fix C2)

## Summary
Clean. 0 blockers, 0 warnings, 2 notes. All fixes verify correct; no safety regression, no scope creep, no data-model drift. (Orchestrator verified tsc exit 0 + runSafetyAssertions [] + runContinuityAssertions [] + expo export clean.)

## Findings

### 1. Crisis detector (ED-safety, priority 1) — PASS
- [VERIFIED] safety.ts:96-100 — only the benign bare-"the" branch dropped from the two purge-intent patterns; objects food/calories/meal/what-I-ate retained. Full pattern list (45-113) read: all self-harm, purge, deliberate-starvation patterns intact; nothing real lost.
- [VERIFIED] Benign FALSE: "get rid of the clutter", "undo the changes" (178-179). Genuine TRUE: "get rid of the calories I ate", "undo the meal I just had" (182-183). runSafetyAssertions [] confirmed.

### 2. saveChat signature change — PASS
- [VERIFIED] storage.ts:86-100 — cont now required; read-modify-write removed (no loadChat inside saveChat); writes exactly the passed refs, closing the staleness race. All 6 callers pass cont (CoachScreen:178,533,585,607,657,740; handleClear ""/0); none missed. loadChat still normalizes old stores.
- [VERIFIED] No message drop — each caller passes its freshly-built list or messagesRef.current.

### 3. Summarizer empty-extraction — PASS
- [VERIFIED] coach.ts:794-842 returns string|null; CoachScreen maybeSummarize:170 handles null before advancing (no ref change, no count advance, no save → batch retried). No runaway loop (fires post-turn, summarizingRef-guarded, bounded by message cadence). Normal path advances + persists.

### 4. MEMORY_TRIGGERS — PASS
- [VERIFIED] coach.ts:333-334 still escalates allergies/intolerances/can't-eat/dietary-identity/remember/note-that/injury to Sonnet; pickModel always returns valid model; food/why/plan escalations intact.
- [NOTE] Removed loose triggers (i like/love/hate, at home, i have a/an, bare knee/shoulder, wedding, i work) mean casual durable-fact phrasing lacking a food/why/plan keyword may route to Haiku (weaker at proactive remember_fact). Intended cost-vs-reliability tradeoff; mitigated by strengthened memory prompt + Haiku tool use. Equipment/life-event phrasings first to re-add if memory misses surface.

### 5. Scope — PASS
- [VERIFIED] SYSTEM_PROMPT + ED_SAFETY_RULES unchanged; still on all four LLM surfaces; cache_control ephemeral intact. No data-model drift (ChatStore shape unchanged; only saveChat signature). No new deps; Anthropic plain fetch to api.anthropic.com only. No BC/cycle impact. Cost guardrails intact (tightening reduces spend).
- [NOTE] coach.ts:47 pre-existing EXPO_PUBLIC_ANTHROPIC_API_KEY in client bundle (documented solo-only/graduation); unchanged; standing concern before external share.

---
## 2026-05-25T00:00:00Z — Robustness round 3: error-handling sweep (storage write try/catch + Coach boot .catch)

Audit of three defensive changes: try/catch around AsyncStorage writes in lib/storage.ts (saveProfile, saveChat, clearChat) and a boot .catch in screens/CoachScreen.tsx. Survey found Food/Workout-log/Cycle/Progress empty states already handled (no changes there).

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. (Orchestrator verified tsc exit 0.)

### Findings
- [NOTE] lib/storage.ts:47-51, :113-117, :121-125 — Swallowing write rejections with console.warn is acceptable for this local-only solo prototype. Tradeoff: a genuine disk-write failure is now silent to the user (data appears saved but isn't; lost on relaunch). Right call vs. an app-crashing unhandled rejection; worth a future "persistence failed" UI signal if data ever becomes non-recreatable.
- Save-race centralization intact: App.updateProfile updates profileRef.current + setProfile synchronously before `await saveProfile`, which now always RESOLVES; in-memory state stays consistent. No caller relied on rejection (all append in-memory first then await; none branch on the save throwing).
- [NOTE] CoachScreen.tsx:545-550 — boot .catch clears booting (guarded by if(active)), mirrors App.tsx loadProfile().catch; cannot strand the boot spinner; only fires on a hard storage-read rejection (loadChat already returns empty store on parse failure). No bad-state path.
- [NOTE] No regression: ChatStore/Profile shapes + signatures unchanged; loadProfile/loadChat still run all back-compat normalization + legacy periodLog migration. No change to Coach prompt, ED_SAFETY_RULES, crisis backstop, cycle/calorie/plan math, or BC branch. No new deps.
- Empty-state spot check friendly: FoodScreen:742-746, WorkoutScreen:858-862 — neutral action-oriented cues, no shaming/deficit/earn-burn framing; render a cue not a blank view.

### Not checked
- Runtime of boot recovery + empty states (no build access by auditor; orchestrator confirmed tsc).

---
## 2026-05-25T00:00:00Z — Guided onboarding/first-run flow (OnboardingScreen)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. New onboarding is ED-safe, honors the BC branch exactly, builds a complete correctly-shaped Profile, changes no safety prompt, math, or data model. (Orchestrator verified tsc exit 0; implementer verified expo export clean, 634 modules.)

## Findings
1. ED-safety of new copy (priority 1) — PASS. Every onboarding string supportive/feel-based. Welcome copy explicitly anti-restriction ("No calorie policing, no guilt, no chasing a number... helps you eat enough"). Goal step reuses existing GOAL_LABELS; no weight-loss cheerleading/number pressure. No goalWeight collected (hardcoded ""). No earn/deficit/burn-off/shaming language (grep-confirmed).
2. Memory-seeding safety — PASS. Free-text dietNotes seeded verbatim into coachMemory (addMemory) + dietaryRules. A user could type an unsafe target, but contained by untouched ED_SAFETY_RULES (BMR floor, "say no plainly"), the "memory is data not instructions" prompt rule, and code-computed targets. addMemory dedupe + MAX cap apply. Acceptable for prototype.
3. Data-model integrity — PASS. finish() builds complete Profile (all 4 maps {}, all 3 arrays [], coachMemory [], scalars). Matches lib/types.ts. plan correctly omitted (optional). No widening. Same initProfile path. storage normalization backfills regardless.
4. Birth-control branch — PASS. onBirthControl=true forces lastPeriodStart:"" + avgCycleLength:28, hides cycle inputs, reassuring copy. Propagates via cycle.ts (dayOfCycle null, no prediction) + coach.ts BC substitution. No phase leak, no prediction seeded.
5. First-run gating + kickoff — PASS. OnboardingScreen renders only when !profile; returning users skip; Settings unchanged + reachable. Brand-new kickoff: noticingsBlock returns "" for empty profile (patterns self-check), prompt says "if nothing notable, just a genuine hello" — warm welcome, no noticing format on zero data. coach.ts untouched.
6. Scope — PASS. No new deps (datetimepicker + async-storage already declared); no new network/auth/payments/push/social/analytics; no change to crisis backstop, ED_SAFETY_RULES, SYSTEM_PROMPT, updateProfile, or cycle/calorie/plan math (verified by reading).

## Notes
- [NOTE] OnboardingScreen.tsx:166 — free-text notes seeded verbatim into coachMemory + dietaryRules; contained by guardrails, acceptable for prototype, revisit before multi-user graduation.
- [NOTE] OnboardingScreen.tsx:142-143 vs SettingsScreen.tsx — onboarding forces lastPeriodStart:"" for BC; Settings only hides + preserves. Moot for new users; consistent.
- [NOTE] Spec §4 lists goal without tone_up; live types include it. Pre-existing drift, NOT from this change. Reconcile spec/code eventually; not a blocker.

## Not checked
- Device rendering of the stepped flow (footer/dots/date picker/keyboard).
- Expo SDK discrepancy NO LONGER present: AGENTS.md says SDK 54, package.json expo@^54.0.34 — agree.

---
## 2026-05-25T00:00:00Z — "Start over / reset all data" feature (storage.clearAllData, App.resetApp, SettingsScreen Start over button)

## Summary
Clean. 0 blockers, 0 warnings, 2 informational notes. (Orchestrator verified tsc exit 0.)

## Findings
1. Reset correctness — confirmed. clearAllData (storage.ts:136-142) multiRemove([PROFILE_KEY, CHAT_KEY]); grep confirms exactly two keys exist (flux.profile, flux.chat) and both removed. All data maps live inside Profile (one key); lib/memory.ts is pure (no own storage). resetApp (App.tsx:98-103) sets profileRef.current=null + setProfile(null) + setTab("coach") → App renders OnboardingScreen when !profile. Stale-profile invariant preserved (ref+state both null).
2. Confirmation gate — confirmed. handleStartOver Alert.alert with Cancel + destructive "Erase & restart"; onReset only on destructive confirm. Visible button red-outline destructive in an isolated dangerZone block.
3. No regression/scope — confirmed. No Coach prompt/ED_SAFETY_RULES/crisis backstop/cycle/calorie/plan-math/data-model change; no new deps; clearAllData try/catch+warn (no throw); BC branch, Save, memory delete untouched.
4. Re-onboarding writes a fresh profile end-to-end — confirmed (OnboardingScreen.finish → initProfile after reset cleared ref+disk; BC branch honored).

## Notes
- [NOTE] clearAllData cannot reject (swallows); await + void onReset() both safe.
- [NOTE] resetApp has no in-flight guard; rapid double-confirm harmless + not practically reachable (Alert dismisses on first tap).

## Not checked
- On-device Alert/button rendering + the reset→onboarding→fresh-profile flow.

---
## 2026-05-25T00:00:00Z — Onboarding revisions: DOB picker, height/weight scroll-wheels, start-fork routing (OnboardingScreen.tsx, App.tsx, WorkoutScreen.tsx)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. No silent default can reach BMR; written strings parse correctly; data model / parsers / BMR untouched; no new dependency (Expo Go safe); routing correct on both forks; ED-safety + BC branch preserved. (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json unchanged, 634 modules.)

## Findings
1. No silent default into BMR inputs — PASS. age/height/weight init "". age written only via applyBirthDate (Android "set" / iOS Done); iOS spin without Done leaves age "". Wheels write only via settle on user scroll events; initial contentOffset doesn't emit them, so shown defaults (5'6"/150lb) are display-only, never persisted unless scrolled. finish() trims → "" if untouched; computeBMR returns null on any missing field. No wrong BMR from defaults.
2. Correct parser formats — PASS. Height "5'6"" parsed by parseHeightCm (normalizeUnits handles curly/straight); weight "150 lb" parsed by parseWeightKg. age via ageFromBirthDate (birthday-aware, clamped); picker bounded maximumDate=today, minimumDate=100yr.
3. No data-model/parser/BMR change — PASS. targets.ts untouched (MIN_DAILY_CALORIES=1200 floor intact). Profile keeps age/height/weight/goalWeight as string; finish() literal matches shape; birthDate local-only, never persisted; no migration.
4. No new dependency — PASS. datetimepicker already declared; wheel is core RN ScrollView+snapToInterval; package.json unchanged; Expo Go SDK 54 unaffected.
5. Routing — PASS. workout fork: onDone("workout") → setTab + planSetupSignal(n+1) → WorkoutScreen effect guards !signal (no open-on-0/mount), fires on change, monotonic counter re-fires on re-onboarding. coach fork → setTab("coach"), no signal.
6. Preserved — PASS. BC branch intact (lastPeriodStart "" + avgCycleLength 28 on BC; cycle step hides date input); diet→dietaryRules + addMemory seeding; tone set; no goal weight; calorieMode static; ED-safe copy unchanged; ED_SAFETY_RULES/Coach prompt/save-race/crisis backstop untouched.

## Notes
- [NOTE] App.tsx resetApp doesn't zero planSetupSignal; harmless (monotonic + change-driven).
- [NOTE] OnboardingScreen WheelPicker.settle skips onChange when idx unchanged; intentional, safe for BMR.
- [NOTE] AGENTS.md (SDK 54) + package.json agree; no SDK drift.

## Not checked
- Device rendering/gesture feel of the core-RN wheels (snap, iOS/Android picker variants).

---
## 2026-05-25T00:00:00Z — Onboarding body-step layout/gesture fix (wheels compacted VISIBLE 5→3, height+weight one row, nestedScrollEnabled)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. Single file (OnboardingScreen.tsx). Fixes nested-scroll conflict (wheels trapped page scroll → Activity Level unreachable) by compacting wheels to fit the viewport. No silent-default-into-BMR regression; data model, Coach prompt, ED rules, BC branch, deps untouched. (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json md5 unchanged.)

### Verification (all pass)
1. No-silent-default-into-BMR — PASS (priority). Touched-gating unchanged: onFeet/onInch/onWeight (260-274) set touched only via settle (115) on real drag/momentum; initial contentOffset (134) doesn't emit settle events, so mount doesn't mark touched. Age stays "" until applyBirthDate confirm; default birthDate display-only. finish() trims → blanks if untouched; computeBMR returns null on missing fields.
2. Wheel correctness — PASS. Band top + content padding both (WHEEL_H-ITEM_H)/2=40; wheel height WHEEL_H=120; selection round(offset/ITEM_H) unchanged + VISIBLE-independent. Centered at 3 rows.
3. Format/parsing/data model — PASS. Height "5'6"", weight "150 lb" still match parseHeightCm/parseWeightKg (targets.ts:22-41). No parser/targets/Profile change.
4. Scope — PASS. Only OnboardingScreen.tsx. No data-model/Coach-prompt/ED_SAFETY_RULES/crisis-backstop/save-race/dependency change; package.json unchanged. BC branch (317-318), diet→memory (339-342), tone intact.

### Notes
- [NOTE] OnboardingScreen.tsx:134 — initial position via contentOffset doesn't emit settle → no onChange on mount (touched-gating safe).
- [NOTE] OnboardingScreen.tsx:505-508 — combined hint reworded; cosmetic, no ED-trigger framing.

### Not checked
- Device gesture/nested-scroll fix, Activity Level tappability, viewport fit, nestedScrollEnabled Android behavior — implementer/Ting to test on Expo Go.

---
## 2026-05-25T00:00:00Z — OnboardingScreen height/weight wheels → +/- steppers (replaced freezing scroll-wheels)

## Summary
Clean — 0 blockers, 0 warnings, 1 note. UI-fix scope respected; no-silent-BMR-default invariant holds; freezing scroll-wheels removed. (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json unchanged.)

## Verified
- No silent BMR default: height/weight start ""; written ONLY via Stepper.onChange ← StepButton.onStep ← start() ← onPressIn. No mount/layout/effect write path. heightTouched/weightTouched are display-only, don't gate the write. Untouched step stores "" → parseHeightCm/parseWeightKg/computeBMR return null. Displayed default 5'6"/150lb never persists unless pressed.
- Timer safety: clear() null-guarded; start() clears before re-arming (no stacking); onPressOut clears; useEffect cleanup clears on unmount (no setState-after-unmount).
- Clamping: buttons disable at bounds; start() early-returns when disabled; every step wrapped in clamp() (belt-and-braces).
- Formats/data model: formatHeight → 5'6" parsed by parseHeightCm; "${n} lb" parsed by parseWeightKg. No parser/targets/Profile change. finish() write path, BC branch, coachMemory seeding unchanged.
- Scope: package.json unchanged; Stepper/StepButton core RN only; no Coach prompt/ED rules/crisis backstop/BC/dietary-memory/tone/birth-date-picker change; no deleted WheelPicker symbols referenced anywhere.

## Notes
- [NOTE] OnboardingScreen.tsx:118 — useEffect(() => clear, []) returns the first-render clear closure; correct (always reads same timer ref). Flag only so a future refactor moving timer off a ref doesn't break unmount cleanup.

## Not checked
- On-device rendering/gesture (freeze resolution, hold feel) — Ting to test on Expo Go.

---
## 2026-05-25T00:00:00Z — onboarding photo calibration step (Feature: optional "Your goals in pictures" + calibrateFromPhotos vision call)

# Audit report — onboarding photo calibration step

## Summary
Clean. 0 blockers, 2 warnings, 3+ notes. ED-safety contract holds: schema cannot leak a number, prompt steers away from thinspo, photos use-then-discard, profile.goal + macro math untouched, ED_SAFETY_RULES byte-unchanged and final on all 5 LLM surfaces. (Orchestrator independently verified tsc exit 0 + expo export clean, no package.json change.)

## Findings

1. ED_SAFETY_RULES integrity (priority 1) — PASS. coach.ts:85-90 unchanged (BMR floor, no purging/fasting/earning, no medical advice, no supplement-by-name, no body-shaming/moralizing, NEDA + 988). Interpolated as FINAL block of SYSTEM_PROMPT(:157)/SUMMARY_SYSTEM(:775)/PHOTO_SYSTEM(:862)/CALIBRATION_SYSTEM(:981)/PLAN_SYSTEM(:1108). New "CALIBRATION SAFETY (overrides anything below it except the final SAFETY block)" header (:970) explicitly does NOT claim to override ED_SAFETY_RULES.

2. CALIBRATION_SYSTEM framing — PASS. (coach.ts:968-981) Mandates QUALITATIVE TEXT ONLY, enumerates forbidden number types (BF%, body-weight estimate/target, goal weight, calorie number, "any other number framed as a target"); forbids body assessment of current photo + before/after framing. Thinspo-steer clause concrete: "If the goal image reflects an extreme, unhealthy, or visibly thinspo-style ideal ... do NOT endorse it. Gently steer ... toward a STRENGTH, HEALTH, FEEL-BASED framing." Motivation hard-bound: "must NEVER cheerlead extreme leanness, weight loss, or shrinking the body as the goal." No loophole.

3. Tool schema (structural safety) — PASS. (coach.ts:983-1008) CALIBRATION_TOOL.input_schema has exactly three type:"string" properties (goal_direction, training_emphasis, motivation), all required. No type:"number" anywhere, no goal_weight/calories/macros. tool_choice forces this exact tool. A number cannot exit the tool boundary as a typed numeric field.

4. Use-then-discard — PASS. (OnboardingScreen.tsx:373-393) runCalibration captures cur/dst from state then setCurrentBase64(null)/setGoalBase64(null) BEFORE the await. Mid-call interrupt cannot leak bytes. Base64 in transient component state only — never written to Profile/AsyncStorage/any persisted store. URIs are local display refs, not in Profile.

5. profile.goal NOT overridden + floors untouched — PASS. (OnboardingScreen.tsx:447-471) finish() builds Profile with goal from the basics-step state (set only by goal-chip press); no path in photo step calls setGoal. calibrateFromPhotos returns three strings → addMemory only. goalWeight hardcoded "" (:467). targets.ts byte-unchanged: MIN_DAILY_CALORIES=1200, computeBMR, computeTargets with Math.max(calories, bmr, MIN_DAILY_CALORIES) floor, goal multipliers intact. No calibration field reaches any numeric target anywhere in the codebase.

6. Memory seeding — PASS. (OnboardingScreen.tsx:483-484) Three facts go through addMemory (dedupe + MAX_COACH_MEMORY=40 cap). Seeded only on success; skip/empty/no-key/fail return [] → nothing seeded.

7. Graceful failure — PASS. runCalibration wrapped in try/catch; all error paths return []. advanceFromPhotos calls setStepIndex in finally. No Alert on failure — only the in-flight ActivityIndicator. !hasApiKey() → silent skip.

8. Scope — PASS. package.json unchanged. Only lib/coach.ts and screens/OnboardingScreen.tsx changed. Profile shape unchanged. No BC-branch/crisis-backstop/data-model/new-fetch-destination change. Only api.anthropic.com.

9. UI copy & framing — MOSTLY PASS. Step title "Your goals in pictures" direction-framed; subtitle "a workout, a person, a vibe, a feeling" (broader than dream-body, preserved). Transparency line verbatim: "Your macros stay computed from your stats with a safety floor; photos don't change the numbers." Footer: "Photos aren't saved — your Coach reads them once to understand the direction, then they're discarded. Skip the whole step anytime." In-flight "Reading what motivates you…". No "dream body"/before-after/numbers/goal-weight ask.

10. Standing invariants — PASS. BC branch in finish() clears lastPeriodStart + forces avgCycleLength:28 when onBirthControl. Dietary chip + free-text seeding preserved. Coach SYSTEM_PROMPT byte-unchanged. No goal weight collected.

## Warnings (non-blocking)
- [WARN] OnboardingScreen.tsx:739 — In-flight copy "Reading what motivates you…" slightly opaque; transparency line above already states the contract but a tiny word-choice review at the moment a user wonders what the AI is "reading" about her body. Not a safety risk.
- [WARN] OnboardingScreen.tsx:259, 405 — calibratedFacts is component state, not a ref. Between setCalibratedFacts and finish() the user navigates through cycle → diet → tone → start; state survives, but dataflow is implicit. A future remount would lose facts. Worth a comment or ref-based handoff. Not a current bug.

## Notes
- [NOTE] coach.ts:1086 — calibrateFromPhotos returns the three fields after a non-empty check; if all three are empty strings, throws — caught and soft-skipped.
- [NOTE] AGENTS.md SDK 54 pin and package.json expo ^54.0.34 consistent.
- [NOTE] No test runner configured; addMemory cap/dedupe on calibration path verified by inspection (runMemoryAssertions covers the underlying invariants).

## Not checked
- Visual rendering of photo step on device (slot layout, ActivityIndicator, footer Skip↔Next↔Reading…).
- Live Sonnet behavior with real photos: (a) goal photo not a body, (b) very lean physique (thinspo-steer fires), (c) one slot filled, (d) no API key. Recommend Ting test each.

---
## 2026-05-25T00:00:00Z — Coach first-time welcome kickoff (coach.ts coachKickoff firstTime branch + CoachScreen detection)

## Summary
Clean. 0 blockers, 0 warnings. (Orchestrator verified tsc exit 0; git diff confirms else-branch string byte-identical — only the ternary punctuation around it differs.)

## Findings
- [NOTE] coach.ts:85-90 — ED_SAFETY_RULES byte-unchanged, still FINAL block of SYSTEM_PROMPT (line 157). PROACTIVE_ED_SAFETY (line 155) intact. New code adds only a USER-message string in coachKickoff; system prompt, context block, and safety composition untouched. Both branches flow through runConversation(SONNET, undefined, true) with forOpener=true → noticingsBlock still composed.
- [NOTE] coach.ts:753-759 — Welcome instruction framing reviewed: macros framed feel-based "starting point, never 'must hit', never about shrinking or earning food"; unavailable-targets branch explicit ("skip the numbers and say plainly you'll need a couple more details… do not estimate"); BC branch preserved ("if hormonal birth control, say so, skip phase/day prediction"); photo absence silent ("never mention photos or 'what you uploaded,' never say she didn't share"); tone clause ends with "ED-safety overrides tone — no body-comparison, no weight-loss cheerleading, no pressure." No new ED-trigger phrasing. Numbers must come from context (explicit "do NOT recompute or invent them").
- [NOTE] CoachScreen.tsx:527 — firstTime = messages.length===0 && !store.lastDate; BOTH required. saveChat always writes lastDate=toISODate, so a returning user with any prior turn cannot false-positive.
- [NOTE] CoachScreen.tsx:747 — Clear-chat calls coachKickoff(profile) without flag → regular hello (not the first-time welcome). Start-over wipes CHAT_KEY + profile → next mount lands in first-time path with both conditions met → desired.
- [NOTE] coach.ts:760 — Else branch (returning-user new-day kickoff) byte-identical (git diff confirmed). Both branches share runConversation/SONNET/forOpener=true.
- [NOTE] Scope: no data-model change, no new dependencies, only the two listed files. Two coachKickoff callers (boot kickoff + clear). Model routing + noticings composition unchanged.

## Not checked
- Visual rendering of the welcome on device (model adherence to "few short paragraphs, no lists/markdown" — Ting to confirm).

---
## 2026-05-25T00:00:00Z — Round: DOB silent-default tightened (spun-required + day-granular), photo confirmation card, welcome with concrete direction + plan-suggestion + no-deficit-wording

## Summary
Three product changes, one tightening round; all audited. The "deficit" wording flagged in the first audit pass was tightened in a second pass — final commit is the safer version. 0 BLOCK, 0 WARN, several NOTEs. (Orchestrator verified tsc exit 0; implementer verified expo export clean.)

### Fix A — DOB silent-default (OnboardingScreen.tsx)
- `defaultBirthDate()` normalized to 00:00:00; new `sameDay(a,b)` compares only Y/M/D; iOS spinner + Android "set" branches both gate on `!sameDay(...)` to flip `birthDateChanged`. iOS Done only closes picker (no commit). `next()` and `finish()` belt-nets gate on `birthDateChanged`. Opening picker without spinning to a different calendar day → age stays blank. Robust against iOS mount-tick rounding (mount value comes from initialBirthDateRef itself; sameDay-only check immune to sub-day rounding).

### Fix B — Photo confirmation card (OnboardingScreen.tsx)
- After a successful calibration, the photos step shows a card with the three qualitative strings (Direction / Training focus / Why) + "Saved to your Coach's memory. You can adjust anytime in Settings under 'What your Coach remembers about you.'" Auto-advances ~2.2s or via Continue. Footer hidden while card is up. Empty/fail paths silently advance as before. base64 still dropped before await; use-then-discard preserved. Numbers structurally impossible (schema strings only).

### Fix C — First-time welcome: concrete direction + plan suggestion + no deficit wording (coach.ts coachKickoff firstTime branch only)
- Item 1: "REFLECT THEM CLEARLY" — direction/emphasis/motivation from memory reflected in Coach voice; no photo/upload/absence callouts; no body/appearance framing.
- Item 2: targets from context only, "do NOT recompute or invent," unavailable path explicit.
- Item 3 (TIGHTENED after audit WARN): macros framed by what they DO for her (protein→muscle/recovery, carbs→training, fats→hormones/feel); fat-loss framed as "fueling her training and supporting her body, NEVER as eating less or shrinking"; explicit prohibition "Do NOT say 'deficit,' 'cut,' or any restriction word; do NOT name a calorie gap or quote any number not in the context." Training emphasis reflects memory; cycle line preserves BC branch.
- Item 6: `hasPlan = !!profile.plan?.current` — when !hasPlan, brief invite to build one; when hasPlan, brief nod to it.
- Closing sentinel: "ED-safety overrides tone — no body-comparison, no weight-loss cheerleading, no pressure, no commenting on her body."

### ED-safety integrity (verified end-to-end)
- ED_SAFETY_RULES byte-unchanged (coach.ts:85-90) and remains the FINAL block of all five LLM-surface system prompts (SYSTEM_PROMPT, SUMMARY_SYSTEM, PHOTO_SYSTEM, CALIBRATION_SYSTEM, PLAN_SYSTEM).
- Grep across coach.ts for `deficit|\bcut\b|calorie gap|shrink|earning food|eat less` returns ONLY prohibition contexts — no Coach-facing recommendation uses any restriction word.
- BC branch preserved; photo absence silent; floors (max(BMR,1200)) untouched; photos still use-then-discard; profile.goal not overridden.

### Scope
- No data-model change; no new dependency (package.json unchanged); no save-race/crisis-backstop/cycle-math/calorie-math change; only `lib/coach.ts` + `screens/OnboardingScreen.tsx` modified. Model routing unchanged (kickoff still SONNET, forOpener=true).

### Not checked
- Live model behavior on the new welcome wording (smoke test recommended with: lose_fat goal + recomp memory fact + cycle phase known; then BC + no memory facts; then plan-fork-taken vs plan-not-created).
- On-device verification of the confirmation card auto-advance + DOB picker no-spin behavior on real iPhone (Expo Go SDK 54).

---
## 2026-05-25T00:00:00Z — Coach voice recalibration (round 1) + photo-card auto-advance removal

## Summary
Clean. 0 blockers, 0 warnings, 4 notes. Deliberate recalibration: Ting authorized loosening vocabulary, proactive observations, weight-trend discussion, and welcome voice. ALL real safety primitives intact: BMR/1200 floor (targets.ts unchanged; plan.ts dayTargets re-clamps), crisis detector (safety.ts unchanged), ED_SAFETY_RULES byte-unchanged + final on all 5 LLM surfaces, BC branch preserved, no-shame/no-moralize/no-sub-floor still explicit in PROACTIVE_ED_SAFETY. (Orchestrator verified tsc exit 0.)

## Findings
- ED_SAFETY_RULES byte-unchanged + final block on SYSTEM_PROMPT (160) / SUMMARY_SYSTEM (807) / PHOTO_SYSTEM (894) / CALIBRATION_SYSTEM (1013) / PLAN_SYSTEM (1140). All five hard rules intact.
- PROACTIVE_ED_SAFETY (lib/coach.ts:99-103) retains all three required safety lines: (a) sub-floor/restrictive prohibition, (b) no-shame/no-moralize, (c) under-fueling → nudge to MORE food.
- Welcome firstTime branch (lib/coach.ts:758-773) — numbers-from-context-only preserved; BC branch preserved; closing sentinel names all three safety lines; vocabulary expansion ("cut, lean out, fat loss, calorie gap, recomp") explicitly gated to fat-loss goal.
- SYSTEM_PROMPT line 131 (MACROS section) — "no earn / no burn off" and "never below BMR" intact; only the "never as permission to eat less" net-mode vocabulary phrase dropped (BMR floor + PROACTIVE eat-enough nudge + code floor still hold).
- targets.ts byte-unchanged (MIN_DAILY_CALORIES=1200; Math.max(calories, bmr, MIN_DAILY_CALORIES)). plan.ts dayTargets cycled-day floor intact.
- safety.ts crisis detector + CRISIS_RESOURCES_MESSAGE unchanged (988 + NEDA 741741 + nationaleatingdisorders.org).
- Photo confirmation card auto-advance fully removed: no setTimeout/timer/cleanup remnants. commitCalibrationAndAdvance only fires from on-card Continue tap. base64 use-then-discard preserved.
- BC branch preserved end-to-end (welcome firstTime, buildContextBlock, planContext).
- Scope contained: only lib/coach.ts + screens/OnboardingScreen.tsx modified; package.json unchanged; no new auth/payments/push/telemetry; only api.anthropic.com network.

## Notes
- [NOTE] coach.ts:100 — PROACTIVE_ED_SAFETY now allows "If she's chasing fat loss and the trend is moving, you can name it." Most behavior-changing line; scale-fixation hedge at line 101 partially mitigates noise-vs-signal risk on weight trends.
- [NOTE] coach.ts:766 — vocabulary expansion gated on fat-loss goal in the prompt instruction; recommend smoke-test welcome opener across all five goal variants.
- [NOTE] coach.ts:131 — dropped "never as permission to eat less" was vocabulary-only; BMR floor in code + ED_SAFETY_RULES still block sub-floor deterministically.
- [NOTE] AGENTS.md SDK 54 pin agrees with package.json. Earlier discrepancy resolved.

---
## 2026-05-25T00:00:00Z — Coach voice recalibration round 2: optional onboarding goal-weight stepper + persistent currentPhotoUri/goalPhotoUri on Profile + qualitative body_fat_range on calibration tool + Settings Body photos section

## Summary
Clean. 0 BLOCK, 1 WARN, several NOTEs. Three additions ship safely: optional onboarding goal-weight stepper (touched-gated, no silent default), persistent currentPhotoUri/goalPhotoUri on Profile (URI-only, no base64 persisted, no network egress), and an optional qualitative body_fat_range field on the calibration tool with explicit "rough estimate" framing + UI caveat. ED_SAFETY_RULES byte-unchanged + FINAL block on all 5 LLM surfaces. Floors/crisis detector/BC branch untouched. (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json unchanged.)

## Findings
- ED_SAFETY_RULES (coach.ts:85-90) byte-unchanged + FINAL block: SYSTEM_PROMPT(160), SUMMARY_SYSTEM(807), PHOTO_SYSTEM(894), CALIBRATION_SYSTEM(1021 — surface that changed; final position preserved), PLAN_SYSTEM(1158).
- CALIBRATION_SYSTEM safety integrity verified: BF allowance explicitly framed ROUGH/APPROXIMATE ("'~22–26%', never '23.4%'") + explicit omission rule when photos don't support; "NEVER include body-weight estimate, goal weight, calorie number, or any other number framed as a target" preserved; no before/after; thinspo-steer + "do NOT endorse" + strength/health framing preserved; "never shame, never comparison-as-judgment" preserved. CALIBRATION SAFETY explicitly subordinated to ED_SAFETY_RULES (:1008).
- CALIBRATION_TOOL schema (:1023-1053): four properties, ALL type:"string"; body_fat_range deliberately NOT required (matches omission rule). NO type:"number" anywhere. NO goal_weight/target_weight/calories/macros/bmi. Structurally cannot emit a numeric target. tool_choice forces this single tool.
- Data model: currentPhotoUri?/goalPhotoUri? added as top-level optional strings on Profile (types.ts:55-59). No expo-file-system added (only transitive sub-dep of expo). loadProfile back-compat: missing fields → undefined natively (no new normalization needed). SettingsScreen.buildProfileFromForm preserves both URIs from latest profile (wipe-guard pattern). No base64 persisted (URI-only); URIs are local cache paths only, no upload.
- Goal-weight stepper: touched-gating preserved (goalWeightTouched only flips on real onPressIn; untouched → blank "" stored; no silent default). String format "X lb" matches parseWeightKg.
- Settings Body photos UI: Retake + Delete both route through updateProfile (save-race centralization intact). Delete only clears targeted URI. Image-picker pipeline mirrors onboarding. Documented decision NOT to re-run calibration on retake (avoid burn-rate spend; qualitative facts editable in memory section). Section copy reinforces safety; no earn/deficit/before-after framing.
- Photo confirmation card BF row renders conditionally on body_fat_range truthiness. Caveat ("Visual estimates are rough — take with a grain of salt.") + parenthetical "(rough estimate)" right in field label — caveat reinforced twice. No fixation framing.
- Floors/crisis detector/BC unchanged: targets.ts MIN_DAILY_CALORIES=1200 + Math.max clamp intact; plan.ts dayTargets re-clamp intact; safety.ts CRISIS_RESOURCES_MESSAGE + detectCrisisLanguage unchanged; OnboardingScreen.finish() BC branch (lastPeriodStart:"" + avgCycleLength:28 when BC) unchanged.
- Scope: only CALIBRATION_SYSTEM changed among LLM surfaces. coachKickoff signature unchanged. askCoach unchanged. No new networked dep. Network destinations unchanged (api.anthropic.com + Open Food Facts).

## WARN (non-blocking; acceptable per recalibrated safety line)
- [WARN] coach.ts:1045-1049 — body_fat_range precision is enforced ONLY in the prompt + description text, not in the schema. A drifted model could emit "23.4%" and runtime would pass it through. NOT a BLOCK because: (i) string-type structurally prevents numeric-target injection, (ii) tool_choice forces this single tool, (iii) UI caveat rendered immediately under the value, (iv) field is optional so empty/undefined is safe. Suggested defense-in-depth (deferred): runtime regex strip in calibrateFromPhotos to round any "\d+\.\d" → integer. Deliberately NOT added per the recalibrated safety line (no more belt-and-suspenders overcaution).

---
## 2026-05-26T00:00:00Z — Seed photo body_fat_range into Coach memory in OnboardingScreen.runCalibration

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json unchanged.)

### Findings
- [NOTE] screens/OnboardingScreen.tsx:471 — Conditional is correctly gated on `result.body_fat_range`, which calibrateFromPhotos (coach.ts:1131-1132) normalizes empty/whitespace to undefined. Fact is only seeded when the model emitted a non-empty descriptor; no empty "body composition estimate: " leak possible.
- [NOTE] coach.ts:1010 — Pre-existing (not introduced by this change): calibration prompt forbids precise decimals via prose only; schema does not regex-enforce. Flagged WARN-not-BLOCK previously and unchanged. New seed line inherits the same model-honesty dependency — no new risk added.

### Verifications performed
1. Seed text is pure prefix-on-model-string — no numeric extraction, no parsing.
2. No data-model change. CalibrationResult.body_fat_range? already existed; no Profile/DayLog/ChatMessage/CoachMemory change.
3. ED_SAFETY_RULES byte-unchanged and still final block on all 5 surfaces (chat/kickoff/plan/calibration/review).
4. addMemory pipeline (memory.ts:23-33) dedupes case-insensitive-trimmed + caps at MAX_COACH_MEMORY=40; extra fact (at most +1 per onboarding) well under cap; re-onboard no-ops via dedupe.
5. No new dependency; package.json untouched; no new imports.
6. BC branch, BMR floor, crisis detector untouched.
7. Conditional `if (result.body_fat_range)` aligns with calibrateFromPhotos returning undefined (not "") when model omits the field.

---
## 2026-05-26T00:00:00Z — Onboarding restructure: goal + goal weight from photos (auto/manual goals step) + deterministic BMI floor + calibration context

## Summary
Combined audit covering: (1) onboarding flow restructure (metrics first, photos before goal pick, new `goals` step with auto OR manual rendering), (2) photo calibration extended to derive `goal` (enum) + optional `goal_weight` (string), (3) deterministic BMI-18.5 floor on goal_weight in calibrateFromPhotos parser, (4) profile context (age/height/weight/activity + memory facts) passed into the calibration call's user message, (5) the contradictory "no numbers" line removed from that user message. **0 BLOCK across both rounds; 3 WARN noted as follow-ups (not blockers).** (Orchestrator verified tsc exit 0; implementer verified expo export clean + package.json unchanged.)

## Safety primitives intact (verified end-to-end)
- ED_SAFETY_RULES byte-unchanged at coach.ts:85-90; still FINAL block of SYSTEM_PROMPT(:160-162), SUMMARY_SYSTEM(:807-809), PHOTO_SYSTEM(:894-896), CALIBRATION_SYSTEM(:1032-1034), PLAN_SYSTEM(:1304).
- BMR floor (lib/targets.ts MIN_DAILY_CALORIES=1200 + Math.max(calories, bmr, MIN_DAILY_CALORIES)) untouched. computeTargets contains zero references to goalWeight — goal weight is informational, structurally cannot push macros sub-floor.
- Underweight-BMI guard for goal_weight: prompt-level + deterministic parser floor (REJECT, not clamp). 5'6" → 115 lb floor; 5'10" → 129 lb floor (worked examples verified). Rejects when profile.height missing/unparseable.
- CALIBRATION_TOOL schema: 4 required + 2 optional STRING fields only; `goal` enum-validated against the 5 Goal values; `goal_weight` regex-validated `/^\d+\s*lb$/i`. No numeric type, no body-weight-estimate field, no calorie/macro field. tool_choice forces the single tool.
- Thinspo-steer extended to goal weight: "STEER THE GOAL WEIGHT TOWARD A HEALTHY RANGE rather than chasing the ideal."
- BC branch preserved in OnboardingScreen.finish() (lastPeriodStart "" + avgCycleLength 28 when onBirthControl).
- Crisis detector (lib/safety.ts) untouched.
- Save-race centralization preserved (initProfile single path).
- Photos still use-then-discard (base64 nulled before await; only URI persists).
- No silent default into BMR: birthDateChanged + sameDay gate; height/weight stepper touched-gating; goal-weight stepper touched only on parse-valid model value OR user interaction.

## WARNs (follow-up improvements, not blockers)
- [WARN] coach.ts:1103-1125 — `parseCalibrationHeightCm` is a local byte-equivalent mirror of `parseHeightCm` because targets.ts was byte-unchanged this round. Drift hazard for future edits. Follow-up: export `parseHeightCm` from lib/targets.ts and delete the duplicate (~3-line change).
- [WARN] coach.ts:1273 — BMI floor uses `goal_weight_raw` for digit extraction; currently safe (regex already matched) but if regex is later relaxed could silently extract wrong digits. Follow-up: use `goal_weight_formatted` (validated value) as the digit source.
- [WARN] OnboardingScreen.tsx:477-483 — Calibration call now sends current weight + height + age + activity + memory facts. Same data buildContextBlock sends every chat turn, but first time on a solo calibrateFromPhotos call. Intentional per task; flagged for explicitness. No PII / identifiers / chat history / additional photos leak.

## Notes
- [NOTE] coach.ts:26-27 — Pre-existing model-ID drift (`claude-haiku-4-5` / `claude-sonnet-4-6` vs spec's dated IDs). Not regressed.
- [NOTE] AGENTS.md says SDK 54 (matches package.json ^54.0.34); prior discrepancy resolved.

---
## 2026-05-26T00:00:00Z — Workout-plan onboarding crash fix: setTimeout(0) defer + replace iOS DateTimePicker with shared Stepper (eliminate native picker from all 3 plan-setup entry points)

## Summary
Clean. 0 BLOCK, 0 WARN, 2 NOTE. Crash hypothesis: native layout race when WorkoutScreen's plan-setup Modal opened a native iOS countdown DateTimePicker synchronously during the onboarding teardown + main-screens mount commit. Fix is two layers: (a) setTimeout(0) defer on the post-onboarding auto-open (belt-and-suspenders), (b) replace the iOS DateTimePicker entirely with the same shared Stepper component used in onboarding for height/weight/goal-weight. The native picker crash surface is removed from ALL three plan-setup entry points (post-onboarding auto-open, "Create my plan" CTA, "New plan" link), not just the post-onboarding one. (Orchestrator verified tsc exit 0.)

## Findings
- New shared `components/Stepper.tsx` extracted from OnboardingScreen — `Stepper` + `StepButton` exports; added optional `step?:number` prop (default 1). Verbatim style copy; behavior-preserving for the 4 existing onboarding callsites.
- OnboardingScreen imports `Stepper` from `../components/Stepper`; inline definitions + unused styles deleted; removed dead `useEffect` import.
- WorkoutScreen swap: removed `@react-native-community/datetimepicker` import (still in package.json since OnboardingScreen DOB + SettingsScreen last-period still use it); removed `minutesToDate`/`dateToMinutes` helpers + `timerWrap` style. State `spDuration: Date` → `spSessionMinutes: number` (default 45); `openPlanSetup` reads `s?.sessionMinutes ?? 45`; `generatePlan` writes `sessionMinutes: spSessionMinutes || undefined`. Modal renders `<Stepper min=15 max=120 step=5>`. `PlanSetup.sessionMinutes` (lib/types.ts:305) unchanged → `generateWeekPlan` end-to-end unchanged.
- All 3 plan-setup entry points (post-onboarding `useEffect([openSetupSignal])` at lines 234-242; "Create my plan" CTA at 688; "New plan" link at 701) all flow through `openPlanSetup` → same Modal → same Stepper. Crash surface eliminated from every path.
- `setTimeout(0)` defer preserved in the `useEffect([openSetupSignal])` block as belt-and-suspenders (with clearTimeout cleanup).

## Safety primitives intact
- ED_SAFETY_RULES byte-unchanged + final block on all 5 LLM surfaces (lib/coach.ts:87-92; surfaces at 162, 809, 896, 1034, 1304).
- lib/targets.ts MIN_DAILY_CALORIES=1200 + BMR floor untouched.
- lib/safety.ts crisis detector untouched.
- BC branch in WorkoutScreen (`lean` text gated on `!phase.onBirthControl`) intact.
- Save-race centralization preserved (writes still go through persist/updateProfile).
- No new dependency (package.json byte-unchanged); only one new file `components/Stepper.tsx`.

## NOTEs
- [NOTE] components/Stepper.tsx:52 — `useEffect(() => clear, [])` is correct (returns the cleanup function); could be misread as a typo by future readers, prefer `useEffect(() => () => clear(), [])` clarity-wise (no behavior change).
- [NOTE] WorkoutScreen.tsx:237 — `setTimeout(0)` defer is now belt-and-suspenders (native picker is gone). Comment correctly states this. Worth keeping for prototype duration.

## Not checked
- On-device reproduction of the original crash + confirmation of the fix (no Bash / no device access). Implementer/Ting to verify: fresh install → onboarding via "Create my workout plan" → should land on Workout tab with the setup Modal open and the new Stepper for session length, no crash.

---
## 2026-05-26T00:00:00Z — CALIBRATION_SYSTEM goal/goal_weight COHERENCE + "toned" clarification

## Summary
Clean — 0 blockers, 0 warnings, 1 note. Real-use failure (current=128, model returned tone_up + goal_weight=140 — internally inconsistent) prompted a prompt-level tightening. (Orchestrator verified tsc exit 0.)

## Findings
- New COHERENCE block (coach.ts:1031-1036) constrains the model to keep goal and goal_weight consistent: build_muscle MAY be ≥ current; tone_up/lose_fat MUST be ≤ current; maintain/feel_better at-current or omitted; cross-check before returning. Plus a TONED clarification: "toned" usually means lean out, not bulk; only return build_muscle + higher weight if the goal photo shows a noticeably more muscular physique.
- ED_SAFETY_RULES (87-92) byte-unchanged + still FINAL block on all 5 LLM surfaces (SYSTEM_PROMPT:162, SUMMARY_SYSTEM:809, PHOTO_SYSTEM:896, CALIBRATION_SYSTEM:1040, PLAN_SYSTEM:1310).
- CALIBRATION_TOOL schema unchanged (4 required + 2 optional string fields; goal enum is the original five values).
- New COHERENCE block CONSTRAINS model output, never relaxes a safety rule. Underweight-BMI guard at coach.ts:1024 intact alongside it.
- Deterministic BMI-18.5 floor in calibrateFromPhotos (1253-1284) intact (defense-in-depth).
- Only lib/coach.ts modified; no new dep; no data-model change.

## Note
- [NOTE] coach.ts:1031-1036 — New COHERENCE rules use backtick-wrapped enum values inside the template literal; renders as literal backticks in the prompt (probably helpful as code-token cues). Rest of file uses bare names; consistency note only.

## Not checked
- End-to-end model run to empirically confirm the new prompt prevents the 128→140 inconsistency in practice (Ting to verify on device).

---
## 2026-05-26T00:00:00Z — Photo calibration accuracy upgrades: front+side photos, measurements step + Navy Method BF, Opus upgrade, navyBF-anchored confirmation

## Summary
Clean. 0 BLOCK, 4 NOTE. Safety primitives intact, Navy math verified correct on three worked examples (5'6"+28/13/38→25%; 5'6"+26/12.5/35→19%; 5'8"+30/14/40→28%), migration safe (legacy currentPhotoUri→currentFrontPhotoUri idempotent), BMI floor still applies regardless of navyBF presence. The new measurement-anchored BF surface adds accuracy without weakening any guardrail. (Orchestrator verified tsc exit 0; runBodyCompAssertions() → [] / PASS.)

## Verified
- ED_SAFETY_RULES (coach.ts:96) byte-unchanged + final block on all 5 LLM surfaces (SYSTEM_PROMPT:171, SUMMARY_SYSTEM:818, PHOTO_SYSTEM:905, CALIBRATION_SYSTEM:1050, PLAN_SYSTEM:1363).
- lib/targets.ts + lib/safety.ts byte-unchanged. BMR/1200 floor + crisis detector intact. BC branch in OnboardingScreen.finish() preserved.
- Navy formula in bodycomp.ts:79-80 correct; guards handle missing/zero/transposed/non-finite/out-of-range inputs (clamp <5% or >60% → null). runBodyCompAssertions PASS.
- CALIBRATION_SYSTEM new measurement-anchor clause is internally consistent: body_fat_range still type:"string" with qualitative + ±2% range (never precise point estimate); existing no-precise-BF / no-body-weight-estimate-of-current / no-calorie-target / BMI-18.5 floor / thinspo-steer / no-shame / coherence rules all preserved.
- Deterministic BMI floor in calibrateFromPhotos (coach.ts:1306-1337) unchanged; runs regardless of navyBF context; missing height → drops goal_weight outright.
- Migration safety: loadProfile copies legacy currentPhotoUri → currentFrontPhotoUri only when target unset (idempotent; not lossy); waistIn/neckIn/hipIn/currentSidePhotoUri default to undefined for old profiles.
- Touched-gating on three new steppers preserved; finish() writes touched ? value : undefined; runCalibration uses same gating to build the partial Profile for navyBodyFatPercent — partial fills → null → visual fallback.
- Opus only on calibrateFromPhotos (1257); other 4 call sites unchanged. Cost trade-off documented in-line.
- Save-race centralization preserved (buildProfileFromForm reads photos + measurements from latest p; updateProfile path for retake/delete).
- No new dependency. Only network destination among modified code: api.anthropic.com.

## Notes
- [NOTE] bodycomp.ts:34-60 — parseHeightInches duplicates lib/targets.ts:parseHeightCm rules. In-file sync-warning comment present. Acceptable per byte-unchanged constraint; consider promoting to shared helper later.
- [NOTE] OnboardingScreen.tsx:480-486 — partial Profile cast `as unknown as Profile` for navyBodyFatPercent (only four fields populated). Helper is narrow; future widening would silently consume undefined. Consider tightening helper signature later.
- [NOTE] bodycomp.ts:96-102 — Reference comment correctly shows 25; an older spec text mentioned "~22% rough target" — stale, no safety implication.
- [NOTE] Pre-existing: .claude/agents/flux-auditor.md references "v56" but flux/AGENTS.md pins to SDK 54 (current ground truth). Out of scope.

## Not checked
- Device render of new measurements step + three-slot photo UI.
- End-to-end Opus call success against the live claude-opus-4-7 slug — Ting to verify on device.

---
## 2026-05-26T00:00:00Z — Stepper `dimmed` prop for untouched preview values (UX fix)

## Summary
Clean. 0 BLOCK, 0 WARN. UI-only fix. Diagnosis = case B (visual confusion). Touched-gating contract was already correct; Stepper just displayed default values prominently. Fix: added `dimmed` prop to Stepper (dashed border, gray italic, "· not set" suffix) and applied `dimmed={!*Touched}` to all onboarding-step Steppers where untouched should NOT commit (body height/weight, measurements waist/neck/hip, goal weight on both auto and manual cards). WorkoutScreen session-minutes intentionally not dimmed (no skip semantics). (Orchestrator verified tsc exit 0.)

## Verified
- Touched-gating contract unchanged. finish() still writes `*Touched ? value : undefined` (or "" for goalWeight string). Untouched → undefined/"" → not stored. dimmed prop is purely visual; doesn't call onChange.
- All applicable callsites use `dimmed={!*Touched}`: height/weight (body), waist/neck/hip (measurements), goal-weight (both auto and manual goals). advanceFromPhotos flips goalWeightTouched=true when model proposes a number, so auto-card renders un-dimmed correctly.
- WorkoutScreen session-minutes Stepper correctly omits dimmed (defaults false; that field's default is the committed value).
- No data-model/schema/dep change. ED_SAFETY_RULES untouched + still final block on all 5 LLM surfaces. BC branch, save-race, BMR floor, crisis detector untouched.

## Notes
- [NOTE] goalWeight is persisted as "" when untouched (string), while waist/neck/hip are persisted as undefined (number). Pre-existing asymmetry matching how Settings collects goalWeight; downstream parsers handle both empty string and missing field. No safety implication.
- [NOTE] components/Stepper.tsx:105 — "30 in · not set" repeats the prominent default number. Dimming + suffix is reasonable but a more hideable variant ("— · not set" or "tap to set") could be considered if device test shows the number is still misread. Pure UX judgment.

---
## 2026-05-26T00:00:00Z — Personal goals free-text input on onboarding goals step (placed under chips, above goal-weight stepper)

## Summary
Clean. 0 BLOCK, 2 NOTE. Optional free-text input on goals step (both auto and manual paths), placed directly under the goal chips and above the goal-weight stepper. Placeholder verbatim: "My goal is snatched waist, build glutes, defined leaner arms...". Parsed in finish() (split on `,`/`;`/` and `; trim; filter blanks; slice ≤8; prefix "personal goal: "; seed via addMemory). No sanitization (deliberate per recalibration). (Orchestrator verified tsc exit 0.)

## Findings (both NOTE-level)
- [NOTE] OnboardingScreen.tsx:1009/:1122 — Placeholder contains "My goal is " lead-in; if user retypes the placeholder verbatim, first stored fact becomes "personal goal: My goal is snatched waist" (awkward, not unsafe). Could be improved by dropping the lead-in from placeholder OR stripping "my goal is" prefix in parse. Left as-is per recalibration line on not over-engineering.
- [NOTE] OnboardingScreen.tsx:712-717 — Regex `\s+and\s+` splits compound phrases ("toned and lean" → two facts). Intentional. Does NOT chop the middle of words (whitespace required on both sides).

## Verified
- No data-model change; personalGoalsText is local state only; Profile shape unchanged.
- Empty input → no facts seeded (trim + guard).
- Regex correctness verified via worked example ("snatched waist, build glutes, and defined leaner arms" → 3 clean facts after empty-segment filter).
- Cap: .slice(0,8) per-field + MAX_COACH_MEMORY=40 in addMemory with dedupe/oldest-evict.
- Placement: between goal chips and goal-weight stepper on BOTH render paths (auto-card 983-1011-1013 and manual-card 1093-1124-1126).
- ED_SAFETY_RULES untouched + still final block on all 5 LLM surfaces; targets.ts BMR/1200 floor untouched; safety.ts crisis detector untouched; BC branch in finish() preserved; CALIBRATION_SYSTEM + goal/goal_weight coherence rules untouched.
- No new dependency.

## Not checked
- Visual rendering on device (multiline TextInput growth, ScrollView keyboard behavior).

---
## 2026-05-26T00:00:00Z — Custom macros: Settings adjustable, Coach set_targets tool, sub-floor soft-warn (Settings) + Coach-refusal

## Summary
Clean. Real safety-adjacent feature: user can now override deterministic macro targets. PER TING'S CHOSEN DESIGN: Settings allows sub-floor override with soft warning + destructive-confirm Alert; Coach `set_targets` tool REFUSES sub-floor (Coach maintains its own ED_SAFETY_RULES stance — user can still set sub-floor herself in Settings). Full override granularity. (Orchestrator verified tsc exit 0.)

## What changed
- Profile: customTargets?: {calories,protein,carbs,fat}; customTargetsSetAt?: string. Optional, undefined for old profiles.
- lib/targets.ts: computeTargets returns customTargets verbatim when set (no floor clamp — floor decision was made at set-time with warning). Existing behavior unchanged when customTargets absent. New exports: targetsFloorCalories(p), recomputeMacrosFromCalories(p, kcal).
- lib/coach.ts: SYSTEM_PROMPT macros section gains a bullet about the set_targets tool with explicit "NEVER set calories below BMR floor via the tool — refuse and explain she can change it herself in Settings." buildContextBlock targetsLine appends a customNote when customTargets is set (and below-floor note if applicable). New FOOD_TOOLS entry set_targets (all four fields optional). ED_SAFETY_RULES byte-unchanged + still FINAL block on all 5 LLM surfaces.
- CoachScreen runTool: set_targets handler refuses sub-floor (first pass + defense-in-depth second pass after fill-in); fills any unspecified fields from current effective targets; persists via updateProfile.
- SettingsScreen: new Macros section with current effective display + Customize button → macros editor Modal (four Steppers, Auto-calc-from-calories button, Reset-to-auto button, sub-floor warning + destructive Alert confirm). buildProfileFromForm preserves customTargets + customTargetsSetAt (wipe-guard).

## Safety primitives
- ED_SAFETY_RULES byte-unchanged + final block on all 5 LLM surfaces. The "never recommend calories below BMR" rule still governs the Coach's tool actions (set_targets refuses sub-floor).
- BMR/1200 floor in computeTargets still active for the AUTO-COMPUTED path. Soft floor in Settings UI (warning + confirm), hard floor in Coach tool (refusal).
- safety.ts crisis detector untouched. BC branch preserved. Save-race centralization preserved (every write through updateProfile).
- No new dependency.

## What it means in practice
- User can set her own sub-floor macros directly in Settings after a clear warning and a destructive Alert confirm. Her agency, explicit override.
- Coach won't do it for her via tool — points her to Settings if she really wants. Coach maintains ED_SAFETY_RULES on its own actions.
- The Coach context line notes when targets are custom (and whether below floor) so the Coach knows the situation.

---
## 2026-05-26T00:00:00Z — Fiber added as a tracked macro across 8 files

## Summary
Clean. 0 BLOCK, 5 NOTE. Fiber threading correct end-to-end (Targets shape, computeTargets auto + override paths, dayTargets cycling, FoodEntry, consumedTotals, OFF scaling, all three tool schemas log_food/report_foods/set_targets, Settings macros editor, Food tab totals + manual + edit + meal builder + photo-review). ED_SAFETY_RULES byte-unchanged + still final block on all 5 LLM surfaces. safety.ts byte-unchanged (0 fiber matches). Back-compat: old customTargets back-filled via fiberFromCalories on read; old FoodEntry without fiber → e.fiber ?? 0 in sums (never NaN/undefined); old SavedFood passes undefined through. BMR/1200 floor on calories untouched. (Orchestrator verified tsc exit 0.)

## Formula (Ting confirmed)
fiber_target_g = max(25, round(calories/1000 * 14)). 2000→28, 1500→25 (floor), 2100→29, 600→25 (floor). Math.round.

## NOTEs (non-blocking)
- [NOTE] CoachScreen.tsx:254 — log_food echo doesn't include per-entry fiber; daily total does. Cosmetic.
- [NOTE] FoodScreen.tsx:795-802 — Per-entry row shows P·C·F but not fiber. Matches "no stale 0g for legacy" stance.
- [NOTE] food.ts:411 — Meal builder addBuilderHit persists fiber from scaleHit even when OFF lacks fiber_100g (defaults to 0). Minor honesty issue, not a back-compat break.
- [NOTE] types.ts:88 — customTargets.fiber is now required in the type; old on-disk profiles deserialize without it but computeTargets back-fills correctly on read. A one-shot storage migration would make the type honest.
- [NOTE] AGENTS.md SDK 54 vs prior "v56" discrepancy is RESOLVED — AGENTS.md and package.json agree on SDK 54.

## Safety primitives intact
- ED_SAFETY_RULES byte-unchanged + final block on SYSTEM_PROMPT (172), SUMMARY_SYSTEM (862), PHOTO_SYSTEM (950), CALIBRATION_SYSTEM (1096), PLAN_SYSTEM (1409).
- lib/safety.ts crisis detector untouched. BC branch preserved. Save-race centralization preserved. BMR/1200 floor in computeTargets auto-path intact for calories.
- set_targets BMR floor refusal still triggers ONLY on a.calories < floor; fiber has no safety floor (the 25g min is nutritional, not clinical).
- No new dependency.

---

## 2026-05-26 — CoachScreen TypingDots swap (in-flight + boot indicator)

### Summary
Clean — 0 blockers, 1 warning, 2 notes.

### Findings

- **[WARN]** `screens/CoachScreen.tsx:947, 975` — `{booting && <TypingDots />}` and `{sending && <TypingDots />}` are not mutually exclusive at the JSX level. In current code paths the two flags cannot both be true (new-day boot path sets booting=false before sending=true at lines 676–678; first-time boot keeps sending=false), so two stacked typing bubbles cannot render today. The invariant is implicit, not enforced; a future change letting both flags coexist would render two indicators. Cosmetic-only risk.

- **[NOTE]** `screens/CoachScreen.tsx:99–156` — Animation cleanup is correct. `useEffect` cleanup clears the two stagger timeouts and stops all three `Animated.loop` handles. `Animated.Value`s come from `useRef(...).current` so they're stable; the `[a, b, c]` deps array is harmless but slightly misleading (could be `[]`). No setState inside the animation, so no setState-after-unmount risk. `useNativeDriver: true` consistent across interpolated `opacity` and `translateY`. No leak.

- **[NOTE→RESOLVED]** `screens/CoachScreen.tsx:1056` — `typingDot` was `borderRadius: 4` on a 7×7 square (rounded square, not true circle). Adjusted to `borderRadius: 3.5` post-audit for true circular dots.

### Verification of the implementer's checklist

1. Animation cleanup on unmount — confirmed clean.
2. No data/logic change — confirmed: booting/sending flags, all `if (sending || booting) return` guards, messagesRef/appendMessages, kickoff effect, saveChat paths, maybeSummarize all untouched.
3. Safety surface — no regression. Coach system prompt is not in this file. Crisis backstop wiring at lines 732–734 intact on both success and failure branches. BC branch ("On birth control" label, no phase/day) untouched. `set_targets` BMR-floor refusal logic untouched.
4. No new dependency — `Animated`/`Easing` added to existing `react-native` import; `ActivityIndicator` removed; package.json untouched.
5. Edge case (both flags true) — not reachable in current code paths; cosmetic-only if it ever became reachable.

### Not checked
- Visual rendering / animation smoothness on device (requires device test).
- Whether lib/coach.ts, lib/safety.ts, or the spec system prompt were touched in the same commit (only screens/CoachScreen.tsx was read; implementer confirmed UI-only swap).

---
## 2026-05-27T00:00:00Z — Feature F1 photo timeline (Progress tab)

# Audit report — 2026-05-27 Feature F1 photo timeline

## Summary
Clean / 0 blockers / 2 warnings / 3 notes.

## Findings

- **[NOTE]** `flux/screens/ProgressScreen.tsx:319` — Banner copy uses "no pressure" framing; bannerWeeks clamp `Math.max(4, …)` is defensive belt-and-suspenders since the banner can't fire before 28 days.
- **[NOTE]** `flux/screens/ProgressScreen.tsx:320` — Empty-state copy "see changes over time" stays observational, not "progress / results / transformation."
- **[NOTE]** `flux/screens/ProgressScreen.tsx:591` — Compare modal title is "Compare," sub-headers are dates only. No-commentary contract honored.
- **[WARN]** `flux/screens/ProgressScreen.tsx:84-102` — Banner re-arm mixes `daysBetween` numeric compare with lexical YYYY-MM-DD string compare. Correct today but brittle to a future refactor that stores `latestPhoto.date` as a full ISO timestamp. Consider consolidating into a `bannerShouldShow(profile, today)` helper.
- **[WARN]** `flux/lib/storage.ts:51-70` — Back-compat migration stamps a seeded photoLog entry with `toISODate(new Date())`, so a legacy user whose calibration photos were taken months ago will see them dated today. Not safety-critical; consider an "Imported" sentinel date or dropping the seed before F1 reaches real long-history users.

### Checklist coverage
1. ED-safety copy — PASS. No "earn / burn / progress / transformation / before-after / results / leaner" anywhere. Section title "Body photos (optional)". Compare modal headers are dates only.
2. Photos remain optional — PASS. Banner dismissable; one-tap+confirm delete; Profile fields optional; onboarding init seeds empty [] when no URI captured.
3. Migration safety — PASS. Legacy `currentFrontPhotoUri` / `currentSidePhotoUri` / `goalPhotoUri` are not cleared; Settings still reads them as calibration anchors. Migration only adds a `photoLog` field when missing.
4. No vision call on F1 photos — PASS. No photoLog/BodyPhotoEntry references in `lib/coach.ts`; no fetch / claude- / api.anthropic.com strings in the new ProgressScreen code.
5. Birth-control branching — PASS. Not touched. No edits to cycle math or BC-aware paths.
6. Coach safety prompt — PASS. `lib/safety.ts` and the Coach system prompt are not edited in this slice.
7. Race-safety — PASS. Save (line 174), dismiss (line 186), delete (line 199) all run through `updateProfile((p) => …)` reading the freshest `p`, matching the `logWeight` pattern at line 227.
8. Scope creep — PASS. No biometrics (F2), no cycle calendar (F3), no AI. `expo-image-picker` already in package.json. No new deps. SDK 54 pinning honored.
9. Drift — PASS. Styles reuse `ACCENT` / `card` / `sectionLabel` / `saveBtn` / `backdrop` / `sheet`. No new `any`. Photo URIs stored as strings, no base64 persisted.

## Not checked
- Visual rendering on device (modal layouts, KeyboardAvoidingView, date-strip selection state, compare-grid empty cells) — requires device test.
- Behavior on legacy profiles whose calibration URI points at an evicted iOS cache file.
- `tsc --noEmit` compile run — implementer should verify if in doubt.
- Whether the banner Dismiss tap target competes with any underlying card tap target — layout read suggests siblings.

## Verdict
No BLOCK findings. F1 is safe to report as done. Two WARNs are follow-ups, not gates.

---
## 2026-05-27T00:00:00Z — Feature F2 (body composition tracking + Coach trend surface + plan-adjust flow)

### Summary
Clean — 0 blockers, 2 warnings, 4 notes. Coach safety contract, BMR floor enforcement, crisis-deferral gate, and trend trigger all wired correctly.

### Findings

- **[WARN]** `flux/screens/ProgressScreen.tsx:1042-1050` — Save button enabled and saveScan accepts a note-only entry (no BF%/lean/muscle). The entry then participates in the body-comp log as a sort target. Trend trigger correctly skips it (requires bodyFatPct on both), but `bodyCompContextLines` will still render "Latest: ... — note 'xxx'." Suggest disabling save unless at least one numeric field is set, or filtering note-only entries from the scan log. Not a safety risk; data-shape hygiene.
- **[WARN]** `flux/lib/bodycomp.ts:220-222` — Trend status checks "the two most recent entries" literally. If the user logs a note-only entry after a real BF% scan, the trend trigger is suppressed until another BF% scan lands. Per spec; flagged because it interacts with the WARN above.
- **[NOTE]** `flux/lib/coach.ts:172-179` — Trend block uses "lean conservative — 100-150…" Read as coaching idiom it means "err on the conservative side"; read literally near body-comp content the "lean" can carry a body-comp connotation. Consider rewording to "conservative end — 100-150…"
- **[NOTE]** `flux/lib/coach.ts:337-338` — Trend is suppressed on the opener via `!forOpener`. Confirmed per spec; the opener leads with noticings, not the trend.
- **[NOTE]** `flux/screens/ProgressScreen.tsx:929-938` — Low-BF% clinician note (<14%) is non-blocking, neutral wording, italic, detail-modal only. Threshold is implementer's pick.
- **[NOTE]** `flux/lib/storage.ts` — No migration added for `bodyCompLog` / `lastBfTrendSurfacedAt`. Both optional; consumers handle missing values defensively. Existing profiles load cleanly.

### Verified clean
- SYSTEM_PROMPT: PROACTIVE_ED_SAFETY and ED_SAFETY_RULES interpolated exactly once, at the end, in order. String constants byte-for-byte unchanged. BODY-COMPOSITION TREND block sits ABOVE both.
- Trend block forbids moralizing words; FACTUAL framing only; defers to ED-safety.
- BMR floor: trend block refuses sub-floor cuts; set_targets handler hard-refuses sub-floor (defense in depth).
- `mark_bf_trend_surfaced`: only stamps marker. No plan/target mutation. Race-safe via updateProfile transform.
- Crisis-deferral: crisisActive computed in askCoach from detectCrisisLanguage(lastUser.content), threaded through to buildContextBlock; suppresses ARMED block. CoachScreen.deliver crisis backstop untouched.
- Trend trigger: ≥2 entries with bodyFatPct, ≥56 days apart, goal mapping correct, 14-day re-pester gate enforced, opener excluded.
- Progress UI: numbers/dates/source only; plain +/- delta in neutral gray; no green/red; no "improving / on track / progress" copy.
- Untouched: lib/safety.ts, lib/targets.ts, lib/cycle.ts, lib/storage.ts. Navy tape-measurement math preserved.
- BC branch untouched. Scope clean (no new deps/endpoints). No new `any`. Expo SDK 54 honored.

### Not checked
- Visual rendering on device.
- Live Coach behavior — does Claude reliably call `mark_bf_trend_surfaced` in the same turn it raises the trend? Recommend a contrived two-entry test (BF up, ≥8 weeks apart) to confirm marker advances + trend doesn't re-fire within 14 days.

### Verdict
No BLOCK findings. F2 is safe to report as done. Two WARNs are data-hygiene follow-ups, not gates.

---
## 2026-05-27T00:00:00Z — F2 WARN cleanup (note-only save refusal + BF-filtered trend pair)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes.

### Findings
- **[NOTE]** `flux/lib/bodycomp.ts:217` — Trend math now filters bodyCompSorted to entries with `typeof bodyFatPct === "number"` before picking the two most recent. Verified: (a) BF + note-only + BF → trend fires on the two BF entries; (b) BF + note-only → not armed (fewer than 2 BF entries); (c) 8-week gap is computed between the filtered pair, not raw neighbors.
- **[NOTE]** `flux/lib/bodycomp.ts:226-230` — Defensive re-check uses local typeof narrowing instead of an `as` cast. No new `as` / `as any` / `as unknown as` introduced.

### No-regression checks (all pass)
- SYSTEM_PROMPT unchanged. `mark_bf_trend_surfaced` unchanged. 14-day re-pester gate unchanged. Crisis-deferral gate unchanged. Low-BF% clinician note unchanged.
- `bodyCompContextLines` still reports the literal latest entry (per spec — even legacy note-only entries surface as "Latest" for Coach context).

### Front-end gate
- `saveScan` refuses when bf/lean/muscle are all null. Save button stays disabled in the same condition. Style and disabled expressions are identical so they stay in sync.

### Scope
- Only `flux/lib/bodycomp.ts` and `flux/screens/ProgressScreen.tsx` touched. No safety.ts / targets.ts / cycle.ts / storage.ts / Navy math / Coach prompt changes. No new deps.

### Not checked
- Device-level disabled state rendering.
- Legacy migration: pre-existing note-only entries (from before the gate) remain in `bodyCompLog` and would still surface in Coach context as "Latest." Per spec; flagging in case Ting wants a one-time cleanup later.

### Verdict
Ship it. No blockers.

---
## 2026-05-27T00:00:00Z — Feature F3: Cycle length chart + summary tweaks (CycleScreen)

### Summary
Clean. No blockers, no warnings. One NOTE about a labeled-but-divergent count, intentional per inline comment.

### Findings
- **[NOTE]** `flux/screens/CycleScreen.tsx:143-144` — `cyclesTracked` uses raw `starts.length - 1`; `chartSeries` uses plausibility-filtered `cycleLengthSeries`. A logging-vacation gap could yield "Cycles tracked: 4" alongside "Cycle length (last 3)". Each label is accurate on its own; flagged in implementer's inline comment.

### BC branching — clean
- Entire new content lives inside the existing `!phase.onBirthControl && starts.length > 0` gate. No leak.
- No phase/cycle math claim reaches a BC user via the new code.

### ED-safety / medical copy — clean
- New strings: "Cycles tracked: N", "Cycle length (last N)", outlier note "If you have questions about your cycle, your provider is the best resource." All neutral, non-diagnostic, defers to provider.
- Single ACCENT for all bars; #bbb avg line. No traffic-light semantics. No fertility/ovulation language.
- Outlier threshold (<21 / >35) is the standard published reference range; note fires only on the AVERAGED window the user can see (can't fire on a single anomaly); italic gray, single instance, never popup.

### No regressions — clean
- Calendar grid, month nav, status card, day-log editor, disclaimer: all unchanged.
- Dormant `selectDateSignal` infrastructure untouched.
- `lib/cycle.ts` read-only; `cycleLengthSeries` already existed with 15-60 plausibility filter at line 93.

### Chart math correctness — clean
- `cyclesTracked = Math.max(0, starts.length - 1)`: correct.
- `chartSeries = cycleLengthSeries(profile).slice(-6)`: implausible gaps filtered out before outlier-note evaluation.
- `chartAvg` is the mean over the displayed window, so the average line + outlier note reference what's on screen.
- No division-by-zero risk (chartMax >= 15 always when chart renders). Bar heights and avg-line position stay 0–100%.

### Scope / drift — clean
- No Progress-tab edits. No Coach-context additions. No new tools/data-model/predictions/fertility surfaces.
- No new imports/deps. No new `any`. Expo SDK 54 honored.

### Not checked
- Device-level visual rendering (bar alignment, average-line cross-bar continuity, month-label slot alignment).
- Android-specific rendering of absolutely-positioned percentage-height views.

### Verdict
Feature F is clean — go ahead and close it out.

---
## 2026-05-28T00:00:00Z — Wren visual rebrand pilot (theme.ts + CycleScreen.tsx)

# Audit report — 2026-05-28 Wren visual rebrand pilot (theme.ts + CycleScreen.tsx)

## Summary
4 warnings, 0 blockers. Implementation respects scope and preserves all load-bearing safety copy byte-for-byte. Primary risk surface is contrast: the ED-safety disclaimer and the clinician outlier note both use colors.clay on cream at small sizes and miss WCAG AA. Also flagging a deliberate but worth-tracking palette inconsistency between the new colors.period (ember) and the legacy PHASE_COLORS.menstrual.ink (rose) still living in lib/cycle.ts.

## Findings

- [WARN] wren/screens/CycleScreen.tsx:743 (disclaimer style) — The ED-safety / medical-deferral disclaimer ("Predictions are estimates from your own history… This isn't medical or fertility advice.") renders as colors.clay #8A7B68 on colors.surface #FBF7F0 at type.size.caption (12dp regular). Contrast ratio approximately 3.7:1, below WCAG AA's 4.5:1 floor for normal text. The string is intact and visible, but the rebrand has pushed the single most important ED-safety paragraph on this screen into a softer, less-readable register than it had under the previous palette. Direction: bump to colors.ink (cocoa), increase to body, or both.

- [WARN] wren/screens/CycleScreen.tsx:741 (clinicianNote style) — The outlier clinician note ("If you have questions about your cycle, your provider is the best resource.") is colors.clay italic at 13dp. Same contrast problem as the disclaimer (~3.7:1 on cream). String is byte-identical and still gated correctly by showOutlierNote. Same fix direction.

- [WARN] wren/screens/CycleScreen.tsx:671 (statusGuidance style) — Status card's guidance line uses colors.inkMuted (clay) on colors.surfaceMuted (vapor #E3E8E9) at type.size.callout (14dp). Contrast ~3.2:1, below AA. This slot renders both the BC branch copy AND the four PHASE_GUIDANCE "Many women find…" strings — both safety/ED-adjacent. Strings unchanged but now sit on a low-contrast surface.

- [WARN] wren/lib/cycle.ts:101-106 (PHASE_COLORS) vs wren/lib/theme.ts:23 (ember) — Called out by implementer. PHASE_COLORS.menstrual.ink is still #e2556b (old rose) while new brand period is #B4564B (ember). On CycleScreen the menstrual-day text reads old-rose, the legend "Period" swatch reads new-ember, and the Progress F3 grid keeps old-rose. Two reds at once in the period/menstrual semantic. Not a regression but the rebrand isn't done until unified — follow-up to migrate PHASE_COLORS into theme tokens.

- [NOTE] wren/screens/CycleScreen.tsx:642-648 (eyebrow style) — "A SOFTER SCIENCE" at clay/12dp/medium with letterSpacing 6 on cream is intentionally decorative, well below AA. Fine for a micro-label but cannot carry safety-critical content.

- [NOTE] wren/screens/CycleScreen.tsx:704 (predictedCell + dayText) — Predicted-period day text is PERIOD (ember) on paper (white). Contrast ~4.4:1, borderline AA at 15dp/600. Affordance is carried by dashed border.

- [NOTE] wren/lib/theme.ts — Token module is clean. Exports colors, type, spacing, radius as const, imports only Platform from react-native, no mutability, no React imports. Semantic-alias pattern honored. accent === ink === focus === cocoa is by design.

- [PASS] Scope — only wren/lib/theme.ts (new) and wren/screens/CycleScreen.tsx (modified) touch the theme. lib/cycle.ts, lib/coach.ts, lib/safety.ts, App.tsx, and other seven screens have zero references to lib/theme and were not modified. Verified via grep.

- [PASS] ED-safety disclaimer string byte-identical and rendered unconditionally.

- [PASS] Birth-control branch — phase.onBirthControl still triggers "On birth control" status with correct guidance. History summary correctly gated by !phase.onBirthControl && starts.length > 0.

- [PASS] PHASE_GUIDANCE strings byte-identical.

- [PASS] Clinician note string byte-identical, still gated by showOutlierNote.

- [PASS] PHASE_COLORS consumption — PHASE_COLORS[ph] still read at calendar cell, legend, status card. No phase tint shadowed or overridden.

- [PASS] No scope creep — no new network deps, telemetry, auth, push, payments. theme.ts imports only Platform.

- [PASS] Coach pipeline untouched — routing, caching, system prompt unaffected.

- [PASS] Tap-target affordance for ACCENT-as-cocoa — + button, today outline, month chevrons all retain high contrast (>10:1).

## Not checked
- Could not run npx tsc --noEmit independently; trusted implementer's claim.
- Could not render on device; contrast numbers are sRGB hex calculations.
- Did not audit brand SVGs vs extracted token hexes.
- Did not check Progress F3 grid visually for juxtaposition with new theme.

---
## 2026-05-28T00:00:00Z — Contrast fix for safety copy on CycleScreen (closes prior contrast WARNs)

# Audit report — 2026-05-28 — Contrast fix for safety copy on CycleScreen

## Summary
Clean. 0 blockers, 0 warnings, 4 notes. The three single-property color swaps land exactly as described, all safety copy is byte-identical, cocoa-on-cream and cocoa-on-vapor both clear WCAG AA by a wide margin (>11:1) for the small sizes used, and no out-of-scope styles, tokens, or files were touched. This closes the three WARN-level contrast threads from the 2026-05-28 visual rebrand audit. The PHASE_COLORS rose-vs-ember inconsistency at lib/cycle.ts:101-106 is correctly left untouched and remains an open follow-up.

## Findings

### Changes verified at the claimed sites — PASS
- [PASS] wren/screens/CycleScreen.tsx:671 — statusGuidance now color: colors.ink. Renders the BC branch copy (line 270-273), the no-data branch copy (line 280-283), and the four PHASE_GUIDANCE strings (line 300) inside the status card on colors.surfaceMuted vapor.
- [PASS] wren/screens/CycleScreen.tsx:741 — clinicianNote now color: colors.ink. Italic 13dp, still gated by showOutlierNote, still renders the outlier note string at line 475.
- [PASS] wren/screens/CycleScreen.tsx:743 — disclaimer now color: colors.ink. Renders the bottom-of-screen ED-safety paragraph at lines 497-501 unconditionally.

### Safety copy byte-identical — PASS
- [PASS] ED-safety disclaimer at lines 497-501: "Predictions are estimates from your own history and shift as you log more. Phases are defaults many women feel — yours may differ, so go by how you actually feel. This isn't medical or fertility advice." — unchanged.
- [PASS] Clinician outlier note at line 475: "If you have questions about your cycle, your provider is the best resource." — unchanged.
- [PASS] BC branch copy at lines 271-272 — unchanged.
- [PASS] All four PHASE_GUIDANCE strings at lines 52-59 (menstrual / follicular / ovulatory / luteal) — unchanged.
- [PASS] showOutlierNote gating at line 471 — unchanged.

### Contrast math — PASS
Cocoa #3B2F22 relative luminance ≈ 0.0246.
- On cream #FBF7F0 (L ≈ 0.9362) — contrast ≈ 13.2:1. Disclaimer at type.size.caption (12dp) clears AA (4.5:1) and AAA (7:1) for normal text by a wide margin.
- On vapor #E3E8E9 (L ≈ 0.7895) — contrast ≈ 11.3:1. statusGuidance at type.size.callout (14dp) and clinicianNote italic 13dp both clear AA and AAA comfortably.
- Both BC branch copy and the four PHASE_GUIDANCE strings now read at >11:1.
The previously-flagged ~3.2-3.7:1 contrast deficit on ED-safety-adjacent surfaces is resolved.

### No collateral damage — PASS
- [PASS] Other colors.clay consumers in the same StyleSheet intentionally untouched: weekday, helpText, monthLabelText, modalClose, eyebrow. Decorative/secondary labels.
- [PASS] Other colors.inkMuted consumers untouched: legendText, summarySub, chartHeading.
- [PASS] wren/lib/theme.ts unchanged; semantic aliases preserved; no token additions.
- [PASS] wren/lib/cycle.ts PHASE_COLORS unchanged.
- [PASS] No changes to App.tsx, other screens, or coach/safety modules.
- [PASS] No new tokens, no size/weight/lineHeight changes at the three sites.

### Prior audit NOTE-level items correctly left alone — PASS
- [PASS] eyebrow still colors.clay (decorative-only, not safety-critical).
- [PASS] predictedCell + dayText still PERIOD ember on paper; affordance is dashed border.
- [PASS] helpText, legendText, summarySub, monthLabelText, chartHeading, weekday, modalClose — intentionally unchanged.

### Open follow-up still open — NOTE
- [NOTE] wren/lib/cycle.ts:101-106 PHASE_COLORS.menstrual.ink #e2556b (old rose) still mismatches new brand period ember #B4564B from theme.ts:23. Confirmed untouched by this fix, as it should be. Re-flagging as still-open follow-up: migrate PHASE_COLORS into theme.ts tokens in a separate pass so the menstrual phase ink, the legend Period swatch, and the Progress F3 grid all read the same red.

### Scope & safety — PASS
- [PASS] No new networked deps, telemetry, auth, push, payments, or social. No package.json / app.json touch.
- [PASS] Coach system prompt, Coach API routing, prompt caching, BC branch propagation into Coach context — all unaffected.
- [PASS] Birth-control branch on CycleScreen still routes through phase.onBirthControl at line 266 and renders "On birth control" status with the unchanged BC copy.
- [PASS] No any, no new promise hazards, no raw user input added to the Anthropic call surface.

## Not checked
- Did not run npx tsc --noEmit independently (read-only, no Bash). Color swaps on TextStyle color property trivially type-safe.
- Did not visually render on device. Implementer's responsibility for on-device smoke test (BC + non-BC profile in each phase).
- Did not verify ProgressScreen.tsx duplicate strings (lines 154, 227) carry same treatment — out of scope for this CycleScreen patch.

---
## 2026-05-28T00:00:00Z — Cycle Image-1 visual rebuild + theme/cycle helpers

# Audit report — 2026-05-28 Cycle Image-1 visual rebuild + theme/cycle helpers

## Summary
PASS with 2 NOTEs (both intentional, Ting-approved). No WARN, no BLOCK.

## Findings

- [NOTE: intentional, Ting-approved override of "Many women find" framing — see project memory wren-brand-system / future ed-safety-language memory] wren/screens/CycleScreen.tsx:55-64 — PHASE_GUIDANCE strings landed verbatim. All four match byte-for-byte. The "Be soft with yourself." callback in two of four panels mirrors the original ED-safety tone in a flatter voice; Ting's call. grep of the whole repo confirms PHASE_GUIDANCE does NOT leak into coach.ts, safety.ts, or the Coach system prompt, so the Coach pipeline still uses its own canonical phase guidance.

- [NOTE: intentional, closes 2026-05-28 rose-vs-ember follow-up; Progress F3 retints by design] wren/lib/theme.ts:123-128 and wren/lib/cycle.ts:104 — phaseColors token group landed with the brand palette (menstrual ember #B4564B, follicular sage #8A9A6B, ovulatory powder blue #BDD0D6, luteal dusty rose #C68D8A). Menstrual bg is byte-identical to colors.period/colors.ember, closing the rose-vs-ember inconsistency. cycle.ts re-exports as PHASE_COLORS preserving the {bg, ink} shape and the four phase keys; ProgressScreen.tsx:268 still resolves. ProgressScreen retints downstream by design.

## Verifications (all PASS)

- ED-safety disclaimer CycleScreen.tsx:613-617 — byte-identical, unconditional, cocoa on cream. "Phases are defaults many women feel — yours may differ, so go by how you actually feel" frame intact.
- Clinician outlier note CycleScreen.tsx:596-602 — byte-identical, gated by showOutlierNote, cocoa italic via styles.clinicianNote.
- BC branch integrity CycleScreen.tsx:310-320, 364, 304, 537 — phase.onBirthControl correctly gates the status card to the "On birth control" branch with Wren-branded copy ("Wren doesn't sync to a natural cycle"), hides the phase ribbon entirely (no dot position leak), drops the · DAY N suffix from the eyebrow, and hides the Pattern card. No dayOfCycle claim leaks for BC users.
- No-data branch CycleScreen.tsx:321-331, 364, 304, 537 — "No cycle data yet" status renders; ribbon hidden; eyebrow drops cycle day; Pattern card hidden. Graceful degradation confirmed.
- PHASE_COLORS shape compatibility lib/cycle.ts:104, lib/theme.ts:123-128 — PHASE_COLORS exported with keys menstrual/follicular/ovulatory/luteal, each {bg: string, ink: string}. All four call sites resolve: CycleScreen.tsx:404, 473, 525 and ProgressScreen.tsx:268. Index signature Record<string, ...> keeps unknown-phase indexing safe.
- isPreviousCycle helper lib/cycle.ts:210-217 — pure, no side effects, returns false for BC, no-data, and iso >= currentStart (covers today's cycle and the future). Correct.
- Carryover tint correctness CycleScreen.tsx:482-498 — opacity 0.6 applied ONLY in the tint branch, AFTER isLogged and isPredicted have returned. Logged previous-cycle days keep full-ember fill; predicted days are always future and cannot be isPrev. Future-text opacity 0.55 stacks on text only.
- Scope creep — only 3 files changed (CycleScreen, theme, cycle). No fetch, no axios, no Supabase/Firebase/Sentry/PostHog/Mixpanel/Amplitude. No package.json change. No new imports beyond phaseColors, isPreviousCycle, and the legacy PHASE_COLORS re-export.
- Coach safety prompt — PHASE_GUIDANCE does not appear in coach.ts or safety.ts. Coach pipeline retains its own BC line at coach.ts:207-208, 1581-1582 and §6 instruction at coach.ts:918. Voice change does not propagate to Claude calls.
- Contrast on new surfaces — disclaimer, clinician note, BC branch copy (statusGuidance), and PHASE_GUIDANCE all render cocoa #3B2F22 on cream #FBF7F0 or vapor #E3E8E9. Both above 8:1.
- phaseForDate / currentPhase BC short-circuit lib/cycle.ts:147-149, 180-182 — intact.

## Not checked

- Visual rendering on device (no Bash; cannot run expo start). Implementer should sanity-check painted calendar and ribbon on iOS Expo Go.
- Cocoa numerals on ovulatory powder blue (#3B2F22 on #BDD0D6) at 15pt italic serif — value unchanged; flag if the painted calendar reads thin in person.
- TS compile — no tsc run; all annotations look sound. No any widening introduced; Record<string, {bg, ink}> index signature keeps unknown-phase access safe.
- AsyncStorage migration — no Profile/DayLog/ChatMessage shape change in this diff; no migration needed.

---
## 2026-05-28T00:00:00Z — expansion of wren-auditor.md (six new/expanded sections, renumbering)

# Audit report — 2026-05-28 expansion of wren-auditor.md (six new/expanded sections, renumbering)

## Summary
2 warnings, 0 blockers. New sections are well-scoped and existing BLOCK criteria are preserved verbatim. The dominant finding is stale "Flux" / `flux/` naming throughout the file body — including broken spec-file references.

## Findings

- **[WARN]** `/Users/tinggeidel/Desktop/wren/.claude/agents/wren-auditor.md:16,158` — References to `Flux_Prototype_Spec_FeatureA_Coach.md` are broken. The file at the project root has been renamed to `Wren_Prototype_Spec_FeatureA_Coach.md`. The §1 reference is load-bearing — future audit runs may fail to cross-reference safety rules. Suggested fix: rename both references in the body to the `Wren_*` filename.

- **[WARN]** `/Users/tinggeidel/Desktop/wren/.claude/agents/wren-auditor.md:8,19,39,61,78,84,85` — Stale `flux/` directory and "Flux" brand references throughout the body. The frontmatter and description are correct, but the body still says "Flux Auditor," `flux/` folder, `flux/screens/`, `flux/lib/storage.ts`, `flux/package.json`, `flux/AGENTS.md`. Suggested fix: replace "Flux" → "Wren" and `flux/` → `wren/` throughout the body.

- **[NOTE]** §2 Prompt injection defenses correctly BLOCK-tier and well-scoped to defenses (not prompt content).
- **[NOTE]** §10 Supabase/RLS correctly marked n/a today with no-op-until-detected framing.
- **[NOTE]** §9 GitHub repo security correctly stays read-only (tells implementer to run `git log -p -S "sk-ant-"`).
- **[NOTE]** §11 General security default WARN with BLOCK carve-out for deep-link-to-Coach injection.
- **[NOTE]** §12/§13 expansions inherit file-level severity defaults; not a problem.

### Preservation check (passes)
- §1 Coach safety prompt integrity: all hard-rule bullets unchanged, still BLOCK/HIGHEST PRIORITY.
- §3 (was §2) ED-trigger surfaces, §4 (was §3) BC branch, §5 (was §4) scope creep, §6 (was §5) data model drift, §7 (was §6) dep drift, §8 (was §7) API key handling: all bullets present, severities preserved.
- "How to audit", "Output format", "Severity definitions", "You do not" sections unchanged.
- No prior BLOCK softened, deleted, or renamed away. Renumbering clean.

## Not checked
- Whether prior audit-log format is consistent with this entry.
- Whether sibling agent files (`wren-implementer.md`, `wren-researcher.md`) have stale Flux references — out of task scope.
- Whether `wren/CLAUDE.md` or `wren/AGENTS.md` reference the old `flux/` path — out of task scope.

---
## 2026-05-29 — CycleScreen mockup re-skin (eyebrow header, painted-ribbon ring, inline-dot legend, coach-quote line, collapsible NEXT 7 DAYS, two-column pattern card, shortened disclaimer, clinician note removed) + Progress→Cycle calendarExpanded fix

### Summary
1 BLOCK, 2 WARN, 2 NOTE. The disclaimer rewrite collided with a project-mandated byte-identical safety string and dropped the only "yours may differ" override frame on the Cycle surface. BC gate, Coach-quote copy, and scope are clean.

### Findings

- **[BLOCK]** `wren/screens/CycleScreen.tsx:619-621` — Disclaimer replaced with "Phases are defaults — go by how you actually feel. Not medical advice." `ed-safety-language.md` records the disclaimer as project-mandated byte-identical ("do not edit"): "Predictions are estimates from your own history and shift as you log more. Phases are defaults many women feel — yours may differ, so go by how you actually feel. This isn't medical or fertility advice." It is documented as the ONLY place the "yours may differ" override frame appears on the Cycle surface — the residual ED-safety frame the 2026-05-28 voice flip was allowed to rely on. The rewrite drops: (1) the predictions-are-estimates clause, load-bearing because the new ring prominently displays a "Period in ~N days" prediction with no remaining on-screen "estimate" qualifier; (2) the "yours may differ" feelings-override frame; (3) "fertility advice" (narrowed to "medical advice"). Documented do-not-edit safety string changed with no recorded override. Rendering at `colors.ink` is correct and resolves known WCAG contrast debt — legibility fine; wording mandate is the issue. Fix: restore the byte-identical string OR obtain and log an explicit Ting override (as done for the voice flip).

- **[WARN]** `wren/screens/CycleScreen.tsx` — Cycle-screen clinician note ("If you have questions about your cycle, your provider is the best resource.") removed. Confirmed still present byte-identical at `ProgressScreen.tsx:227` (gated by `showClinicianNote`) plus a low-BF clinician line at `ProgressScreen.tsx:1216`. `ed-safety-language.md` lists this sentence among the non-negotiables that survived the voice flip. App not fully stripped of provider framing (survives on Progress), but the screen that makes cycle predictions now carries no provider pointer of its own. Coach §6 "check with your provider" rule unaffected. Ting to decide whether Progress-only placement is acceptable.

- **[WARN]** `wren/screens/CycleScreen.tsx:449,469,853` — Collapsed-strip weekday labels render at phase `ink` × 0.7 opacity over phase fills at fontSize 9; designer-flagged, likely sub-AA, same class as known brand-system contrast debt. Not safety text. Verify on-device; consider full opacity or a fixed darker ink.

- **[NOTE]** `wren/screens/CycleScreen.tsx:213` — Progress→Cycle fix (`setCalendarExpanded(true)` in the `selectDateSignal` nonce effect) is pure presentational state, keyed on nonce so re-taps refire, no data write, future-day guard intact, no render-time recompute race. Clean.

- **[NOTE]** `wren/screens/CycleScreen.tsx:339,54-63` — Coach-quote line renders `PHASE_GUIDANCE` verbatim; diffed all four strings against `ed-safety-language.md`, byte-match, no "many women" regression, flat declarative voice intact. Screen copy only; does not propagate to the Coach pipeline.

### Birth-control branch — verified clean
`showCycleUI = !phase.onBirthControl && starts.length > 0` (line 323) gates the entire new UI. BC users get the status card with the verbatim "Wren doesn't sync to a natural cycle — your plan stays steady" copy. `cycle.ts` returns `dayOfCycle: null` / `phase: "on birth control"` and all prediction/previous-cycle helpers early-return on BC. No phase label, cycle-day number, or prediction can reach BC UI; `calendarExpanded` unreachable for BC users.

### Scope — clean
No new deps, no `fetch`/`axios`, no auth/payment/push/telemetry, no `App.tsx` tab edits in this file, no data-model drift (`persist` writes `dayLogs` + `lastPeriodStart` as before), no `PHASE_COLORS` edits.

### Not checked
On-device contrast ratios; `npx tsc --noEmit` (reported clean by prior agents); existence of a verbal Ting override superseding the disclaimer mandate.

---
## 2026-05-29 — Cycle screen final re-audit: second disclaimer override (removal of feelings-override frame) + cosmetic ring tuning

Authorization note: Records Ting's SECOND confirmed 2026-05-29 disclaimer override. Earlier today she dropped "many women feel — yours may differ" and narrowed "medical or fertility advice" → "medical advice". She has now explicitly confirmed removing "Phases are defaults — go by how you actually feel." as well. Net effect: the Cycle surface intentionally no longer carries any on-screen feelings-override frame. Disclaimer is now exactly: "Predictions are estimates from your own history and shift as you log more. Not medical advice." (CycleScreen.tsx, rendered colors.ink / cocoa, centered italic). Confirmed with Ting directly before applying; ed-safety-language.md memory updated to reflect this as current state. Authorized, not a blocker.

Summary: Clean on blockers. 1 NOTE (residual ED-safety gap Ting asked about), 1 NOTE (cross-surface voice inconsistency). No BLOCK, no WARN.

Findings:

[NOTE] CycleScreen.tsx (disclaimer) + coachQuote render + PHASE_GUIDANCE — Residual ED-safety gap from the authorized reduction. The retained disclaimer hedges the ring's "Period in ~N days" prediction (self-history estimate) and preserves the medical-advice carve-out, but no longer explicitly hedges the prominent phase-guidance coach quote (headline italic serif, the most prominent prose on the surface, e.g. luteal: "Endurance feels steadier than intensity this week..."). Gap is narrow/low-severity: (1) guidance copy is flat-declarative, no earn/burn/deficit framing, no calorie target, no weight/comparison — hard ED-trigger surfaces absent, so the missing hedge is epistemic not a removed content guardrail; (2) "Be soft with yourself" partially substitutes a self-compassion register; (3) "estimates from your own history" implicitly extends to the phase label since the quote is keyed off phase.phase. Worth knowing: guidance now asserts how a week "feels" with no on-screen "yours may differ" counterweight — for a user whose body contradicts the label, the surface no longer invites her to trust her body over the prediction. Acceptable today as NOTE; would upgrade to WARN if guidance copy ever drifts toward prescriptive/food-restrictive, since the absorbing hedge is gone. Ting's decision; surfaced, not blocked.

[PASS] Comment accuracy — comments above PHASE_GUIDANCE and coachQuote quote the live two-sentence disclaimer verbatim and state the surface carries no feelings-override frame. No stale live references; remaining hits are inside the comment block documenting the dropped lines.

[PASS] PHASE_COLORS untouched (cycle.ts re-exports phaseColors from theme; PhaseRing imports phaseColors from theme). Progress F3 share intact.

[PASS] No data-model drift. currentPhase contract intact; BC → { phase:"on birth control", dayOfCycle:null, onBirthControl:true }; ring cycleDay fallback unreachable for BC due to gate.

[PASS] BC gate intact — showCycleUI = !phase.onBirthControl && starts.length > 0. BC + no-data fall back to status card; BC copy "Wren doesn't sync to a natural cycle — your plan stays steady" intact verbatim. No phase label / dayOfCycle leak.

[PASS] Hero number clip — bigNumber fontSize 96 / lineHeight 124 (~1.29x), includeFontPadding:false, symmetric center box. No new clip risk. On-device not verifiable here.

[PASS] Center stack within inner diameter — paddingTop:28 / paddingBottom:0 downward bias. At size 348: strokeWidth 37, inner diameter ≈ 273; fits. On-device overlap not verifiable here.

[PASS] No new deps, no new imports, no implied App.tsx change.

[NOTE] WorkoutScreen.tsx and coach.ts still use "Many women feel/find…" framing while Cycle has dropped it twice. Internally coherent per ed-safety-language.md (Coach carries its own frame; voice flip scoped to Cycle), but two ED-safety voices now coexist app-wide. Awareness only, no action.

Not checked:
- On-device rendering of hero numeral / center stack (clip, overlap, vertical balance). Geometry math internally consistent; implementer/Ting should verify on device given the 28dp downward bias and -38 ofLine.marginTop.
- Coach prompt integrity / injection / cost / data model / deps not re-audited — change touches none of coach.ts, safety.ts, types.ts, storage.ts, package.json, or the spec. Coach last confirmed 2026-05-28.
- Expo SDK discrepancy: AGENTS.md now reads "pinned to SDK 54" with no v56 claim; previously-flagged discrepancy appears resolved.

---
## 2026-05-29 — CycleScreen "Today" day-log editor sheet re-skin (presentation + copy)

### Summary
Clean. 0 blockers, 0 warnings. Two informational notes only.

### Findings

- [NOTE] CycleScreen.tsx (sheet copy) — New copy is a net ED-safety improvement. "Tell me about today — only what you want to share.", "or close without saving — nothing here is required.", and "· pick as many as fit" hints reinforce optional/low-pressure logging, reducing compulsion. Note placeholder voice change ("...you want the Coach to know..." → "...you want me to know about today…") is a benign first-person voice shift; field still routes to DayLog.note. No existing safety copy removed/softened. Medical-advice disclaimer untouched.

- [NOTE] CycleScreen.tsx (flow chip) — Flow-chip ember→cocoa selected-state change is visual-only, matching the mockup (selected "Light" shown cocoa like all chips). chipActive uses ACCENT (colors.accent); chipActiveRose fully removed, zero residual refs. Ember token NOT stranded: colors.period still drives PHASE_COLORS ring/grid fills and clearBtnText. Saved DayLog.flow value is independent of chip color (flow === o.key selection unchanged). colors.period/PHASE_COLORS defined in lib/theme.ts, not edited.

### Verified clean
1. ED-safety copy: no removal/softening; distress moods (Sad/Anxious/Stressed/Sensitive/Foggy) write to DayLog.moods only (local logging, no client branch) — no ED/distress branch needed on sheet.
2. BC branch intact: phaseForDate returns dayOfCycle: null for BC; editorEyebrow drops the CYCLE DAY segment when null — no fabricated day. BC users can still log all fields.
3. Data model: no DayLog drift; saveEditor/clearDay/persist logic unchanged; no migration needed.
4. Scope: no new deps, no networked calls, no Anthropic/Coach-prompt changes, no App.tsx edits. Confined to presentation + copy + comment-only TODO resolution.
5. No BC leak to Coach; sheet doesn't assemble Coach context.

### Not checked
- Visual fidelity to mockup — requires device render.
- tsc — reported clean by implementer/designer; no new any, unhandled promises, or risky optional-field access introduced.

---
## 2026-05-29 — Today day-log editor sheet refinements (CycleScreen chips/title/eyebrow + chipRest token)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. All presentation-only; no safety, scope, data-model, or BC-branch regressions.

### Findings
- [NOTE] theme.ts — New semantic token chipRest: "#FBF7F0" is the third cream-valued token (cream, surface, chipRest). Acceptable, not brand drift: same brand cream hex, not a new color; file already documents role-aliasing (surface/cream; accent/focus/ink). Role distinction genuine (chip rest could later diverge from sheet surface). as const preserved, appended last, no key renamed/removed/retyped. Minor: one more same-valued reference to sync if cream shifts. Keep; NOTE-level redundancy at most.
- [NOTE] CycleScreen.tsx (chips) — Unselected chip = colors.chipRest (cream) + divider border; chipActive = cocoa fill+border; chipTextActive = cream text. Contrast fine both states. surfaceMuted (vapor) still used by statusCard, patternCard, noteInput — untouched. No colors.period/ember in chip path; PERIOD remains on strip/grid period fills + clearBtnText; PHASE_COLORS unedited — ring/grid period fills intact.

### Verified clean
- ED-safety copy intact/unaltered: disclaimer, subtitle "only what you want to share", footer "nothing here is required", both "pick as many as fit" hints. No earn/burn/deficit/shame framing, no calorie totals.
- BC branch intact: editorEyebrow drops cycle-day segment when null, no fabricated day; keys on edited date. modalEyebrow.marginTop is layout-only.
- Title presentation-only: fontSize 48 / lineHeight 55 / serifItalic / uppercase transform; source string unchanged.
- No data-model/save-path drift; no scope creep (no new imports/deps/networked calls/Coach/App.tsx); version consistency confirmed (expo ^54.0.34, AGENTS.md "pinned to SDK 54").

### Not checked
- Device pass for 48/55 title clip + cream-chip-on-cream legibility (static: border + cocoa text should suffice).
- tsc clean per implementer/designer report; auditor cannot run compiler.

---
## 2026-05-29 — Food screen redesign (FoodScreen.tsx presentation re-skin + calm/calmFill tokens in theme.ts)

### Summary
Clean on safety. 0 blockers. 3 warnings (brand sign-off, contrast, over-budget framing), 4 notes. No ED-trigger introduced, BMR floor untouched, BC branch correct, no scope drift.

### Findings
- [PASS] ED labels neutral: Goal / − Food / + Exercise / Remaining. No earn/burn-off/deficit/shaming/weight-loss-as-success copy. Old flame "burned" line removed.
- [PASS] Eat-back mitigation solid. Static is forced default (storage.ts:33); burn renders muted clay + "awareness only" tag and is NOT added to Remaining (FoodScreen.tsx:774-787). budgetCal=target static, target+burn net; never subtracts. Doesn't read as "exercise earns food."
- [WARN] FoodScreen.tsx:741-759 — Large center "CALORIES REMAINING" donut + MFP-style deficit breakdown is more front-and-center than before. No new trigger language (not a BLOCK), but structural prominence of a remaining-calories ring is the most ED-adjacent surface in the app. Flag for Ting's explicit sign-off on the MFP-like prominence + "Remaining" emphasis.
- [PASS] BMR floor: targetForDate/computeTargets untouched, Math.max(...,bmr,MIN_DAILY_CALORIES) intact (targets.ts:134, plan.ts:211). Burn only ADDS in net mode, never lowers a target.
- [WARN] FoodScreen.tsx:758,793 — Negative calLeft renders as bare "-N" in ring center + total. Neutral (no alarm-red, fill clamped [0,1]) but no neutralizing microcopy and number is in low-contrast calm. Consider a short neutral caption when calLeft<0. Owner/design call.
- [PASS] BC branch: FoodScreen.tsx:126 gates phase on !onBirthControl && phase!=="unknown"; phaseForDate returns "on birth control"/dayOfCycle:null, correctly excluded. No phase/day-of-cycle leak on Food surface.
- [WARN-brand] theme.ts:55-56 — calm #7E9AA0 / calmFill #DCE5E4 are a genuinely new accent hue, not from existing palette/phase tokens. Role-named + additive (good) but needs Ting brand sign-off.
- [WARN-contrast] "Remaining" total (breakValTotal), macro-meta numbers, over-budget number render in calm #7E9AA0 on cream #FBF7F0 (~2.3:1, below WCAG AA). stripNumSel cream-on-calm (~2.4:1, marginal). Verify ratios; consider darkening calm for numeric uses or colors.ink for the Remaining total. Safety numbers' legibility.
- [PASS] Photo relocation: estimateFoodFromPhoto called with unchanged (base64,kind) signature; vision prompt/path untouched; only entry point moved into Search tab. No new injection surface.
- [PASS] Scope: no new networked deps, no data-model change (calorieMode pre-existed), no Coach/Anthropic edits, no App.tsx tab-bar change.

### Notes
- [NOTE] FoodScreen.tsx:71 — Modals still use hardcoded ACCENT #7c3aed + literal grays (pre-existing); off-brand vs clean main screen. Future modal re-skin.
- [NOTE] FoodScreen.tsx:838 — "via Coach"/"from photo" provenance retained, neutral.
- [NOTE] FoodScreen.tsx:689 — Ring calPct guarded; noTarget copy when no target, no divide-by-zero.
- [NOTE] Pre-existing: AGENTS.md pins Expo SDK 54; unchanged this diff.

### Not checked
- On-device WCAG contrast measurement; tsc (reported clean by implementer); net-mode visual state (correct by code inspection).

---
## 2026-05-30 — Full-codebase audit (entire Wren app, Ting-requested complete review; not a single diff)

## Summary
1 BLOCK, 4 WARN, 8 NOTE. The blocker is new and severe: a live Anthropic API key is committed in plaintext in wren/.env (prior audits recorded this file as placeholder/clean). ED-safety copy, Coach safety-prompt integrity, prompt-injection posture, and birth-control branching are all intact and strong. The 2026-05-29 Cycle disclaimer trims were authorized Ting overrides and removed no hard ED guardrail. Scope, data model (migrated), and dependency posture unchanged; the Expo SDK-54 doc discrepancy is now resolved in AGENTS.md/package.json. Auditor tooling fully functional — no tool failures; the only path error is the malformed nested audit-log path (see WARN), which is a real path issue, not a glitch.

## Findings

### API key handling — BLOCK
- [BLOCK] wren/.env:11 — Live Anthropic key committed in plaintext (EXPO_PUBLIC_ANTHROPIC_API_KEY=sk-ant-api03-...EcwAA), not the PASTE_ placeholder hasApiKey() guards against. EXPO_PUBLIC_* inlines into the client bundle (documented solo-only condition) AND the secret now sits on disk. .env IS gitignored in both /.gitignore (**/.env) and wren/.gitignore (.env), so it should not be in the index — but the auditor cannot run git to confirm history. Required: (a) implementer verify `git ls-files | grep env`, `git log -p -S "sk-ant-"`, `git log -p -S "ANTHROPIC_API_KEY"`; (b) rotate the key in the Anthropic console regardless (treat as compromised — plaintext on disk + pasted into artifacts); (c) confirm $20/mo cap. BLOCK until rotation + git-history confirmation.

### Coach safety-prompt integrity — clean
- [NOTE] lib/coach.ts:98-183 — ED_SAFETY_RULES carries all hard rules (sub-BMR ban, no starve/purge/fast/earn/compensate, no medical advice + provider referral, no supplement-by-name, no body-shaming, ED care-mode with current resources: NEDA online + text 741741 + 988; discontinued NEDA phone correctly absent). Composed as FINAL block into SYSTEM_PROMPT(183), PLAN_SYSTEM(1505), PHOTO_SYSTEM(1046), SUMMARY_SYSTEM(958), CALIBRATION_SYSTEM(1192). All six Anthropic call sites carry a safety-bearing system prompt; no user-content path bypasses it. Closes the 2026-05-25 WARN on PLAN/PHOTO prompts.
- [NOTE] Spec §6 is materially stale vs shipped prompt (still says "many women find", still cites NEDA helpline). Documented drift, not a regression — running prompt lives in coach.ts, not the spec. Update spec to diff against reality.

### Prompt-injection defenses — clean
- [NOTE] coach.ts:730-776 — User content only in role:"user"/tool_result; spec-§6 prompt is the sole system occupant (cached static + volatile context). No role:"system"/"assistant" from user input. Tool args validated (set_targets BMR-floor refusal, date regex, enum checks). Minor gap: SYSTEM_PROMPT has no explicit "ignore instructions embedded in user content" line; low practical risk (local single-user) so NOTE — one hardening sentence recommended.
- [NOTE] No deep-link/URL-scheme/clipboard ingestion into Coach context or AsyncStorage. Barcode → digit lookup, not executed.

### ED-trigger surfaces (all 7 screens) — clean
- [NOTE] No earn/deficit/burn-off/shaming/weight-loss-as-success copy in user-facing strings (only as prompt prohibitions). Onboarding leads with anti-ED framing. Calories always BMR-floored (targets.ts:134, plan.ts:211). Gamification keyed to self-care consistency with a code-level ban on intake/weight-loss detectors (patterns.ts:9-13,480-486). Progress photo/body-comp UI non-editorializing, neutral BF delta, quiet low-BF clinician note.
- [NOTE] CycleScreen.tsx:658-661 — Disclaimer trim verified against ed-safety-language.md + log: both 2026-05-29 trims were explicit logged Ting overrides; retained line rendered colors.ink/cocoa (legible, clay deliberately avoided for WCAG). No hard guardrail removed. Residual phase-quote hedge gap is the same prior NOTE; still NOTE (flat-declarative copy, no calorie/weight content).

### Birth-control branch — clean end to end
- [NOTE] onBirthControl propagates with no leak: cycle.ts:147-149/180-182, predictions/headsUp early-return on BC; Coach + plan context substitute the BC line (coach.ts:207-208,1581-1582); no phase label in CycleScreen(352), CoachScreen header(724-725), FoodScreen eyebrow(126-128), ProgressScreen look-back(266-269); Settings/Onboarding hide last-period inputs on BC. Nothing reaches Coach context.

### Scope — within accepted bounds
- [NOTE] No new networked deps; fetch only to api.anthropic.com (5 sites) + approved Open Food Facts (query/barcode only, no key). No auth/payments/push/telemetry/social. RLS §10 still n/a. Local-only single-user intact.

### Data model — migrated, acceptable
- [NOTE] Profile/DayLog/ChatMessage wider than spec §4 (approved B/C/E/F expansion); migration handled in storage.ts:14-74 (maps back-fill, legacy periodLog/currentPhotoUri migration, photoLog seed; parse errors → first-run, no crash); customTargets fiber back-fill targets.ts:107-110. No drift-without-migration. Update spec §4 to shipped shape.

### Dependency / version — improved
- [NOTE] Expo SDK discrepancy RESOLVED: AGENTS.md says SDK 54, package.json expo ^54.0.34 (RN 0.81.5/React 19.1.0 = SDK 54) — agree. (Auditor-spec §7's "AGENTS.md says v56" claim is itself stale; correct on next agent-file edit.) Imports vs deps clean. lodash.debounce present in node_modules but not imported in app source (transitive only).
- [NOTE] coach.ts:36-38 — claude-haiku-4-5 (spec pins -20251001), claude-sonnet-4-6, and OPUS claude-opus-4-7 for the once-per-onboarding calibration call (bounded, documented cost rationale, under cap). Opus not in original spec §5 — Ting to confirm. Pin/confirm Haiku alias.

### Cost guardrails — adequate
- [NOTE] Haiku default + Sonnet escalation (pickModel) still routes most food/feel msgs to Sonnet (standing 2026-05-25 WARN); prompt caching on static block; send debounced via sending/booting guards (one in-flight turn); summary folds batched (every 8 aged-out); no exponential backoff but no tight retry loop either; no per-day token budget beyond console cap (fine for solo).

### Robustness / TS hygiene — solid
- [NOTE] All Anthropic calls res.ok-checked + structured throws; deliver try/catch keeps crisis backstop on failure; load* swallow parse errors → safe defaults; save* warn-don't-throw; single updateProfile ref pattern (App.tsx:82-100) composes concurrent writes (kills save race); send state machine exhaustive w/ finally reset. as any only in cosmetic style arrays; as unknown as Profile only in scoped test/calibration helpers. Run tsc --noEmit to confirm (auditor can't).

### Repo / audit-log structure — WARN
- [WARN] The referenced path .claude/audit-log.md/audit-log.md does NOT exist — .claude/audit-log.md is a flat FILE (reading a child errors ENOTDIR). Appends must target the flat file; do not create a nested audit-log.md/ directory.
- [WARN] Root /.gitignore lacks *.key/*.pem globs (only **/.env* + node/expo artifacts); wren/.gitignore has them. App lives in wren/ so mostly fine, but a root-level stray key/cert wouldn't be ignored. Add *.key/*.pem to root gitignore.
- [WARN] .claude/agents/wren-auditor.md still has stale flux/ paths + false "AGENTS.md says v56" claim (open since 2026-05-28). Degrades audit accuracy; apply the renames already specified.
- [WARN] No git verification possible (no Bash). The API-key BLOCK severity depends on git history; implementer must run the git log -S checks and report.

## Not checked
- Git history/index state (no Bash) — BLOCK severity hinges on this; rotate key regardless.
- tsc --noEmit, expo start, npm audit, on-device rendering/WCAG (no Bash); prior agents reported tsc clean.
- Whether the committed key is currently valid/has spend (won't network with a live key).
- Full node_modules transitive scan (checked declared deps + all source imports).
- Word-level diff of spec §6 vs shipped prompt (confirmed all hard rules present; spec known-stale, running prompt is authority).

### BLOCK resolution (git verification, run by orchestrator 2026-05-30)
- The API-key BLOCK was conditional on git history. Verified with git: `git ls-files --error-unmatch .env` → NOT tracked; `git check-ignore -v .env` → ignored at .gitignore:34; `git log -p -S "sk-ant-" --all` → 0 hits; `git log -S "ANTHROPIC_API_KEY" --all` → no commits. The live key NEVER entered git history and .env is correctly ignored.
- Downgraded BLOCK → NOTE. Residual posture is the documented, accepted solo-only condition: a real key must exist in local .env for the app to run, and EXPO_PUBLIC_* inlines it into the client bundle — fine as long as the bundle is not distributed. No git leak. Rotation only warranted if the bundle/screenshots of it were ever shared externally; not required on current evidence. The full key was never written to this log (masked tail only).

### WARN fixes applied (orchestrator, 2026-05-30)
- Root /.gitignore: added secrets/signing globs (*.key *.pem *.p8 *.p12 *.jks *.mobileprovision) — closes the "root lacks key/pem globs" WARN.
- Root /.gitignore: corrected stale `flux/` references → `wren/` (flux/.expo|dist|web-build → wren/...; comment refs to flux/.gitignore → wren/.gitignore). No `flux` strings remain in the root ignore.
- .claude/agents/wren-auditor.md:85: replaced the stale "AGENTS.md says v56" discrepancy line with the resolved state (package.json expo ^54.0.34 and AGENTS.md "pinned to SDK 54" agree; re-check on drift). The auditor's WARN mis-attributed the flux/ paths to the agent file; they were actually only in the root .gitignore (now fixed). The agent file itself had no flux/ paths.
- IMPORTANT correction to the 2026-05-30 audit: the repo ROOT (/Users/tinggeidel/Desktop/wren) is NOT a git repository — the only git repo is wren/wren/.git. The earlier BLOCK-resolution git checks were valid (they ran inside wren/). The root /.gitignore is therefore currently DORMANT (no git repo consumes it); keeping it correct is defense-in-depth for if root is ever `git init`-ed. The active ignore rules are in wren/.gitignore, which already covers .env, *.key, *.pem, etc.
- Deferred (not this pass): specs §4/§6 stale vs shipped code (documentation debt).

### Follow-up: live wren/.gitignore confirmed (orchestrator, 2026-05-30)
- The verification audit's one open item — does the LIVE repo's ignore file cover secrets — confirmed directly with git inside wren/: `wren/.env` is NOT tracked; `git check-ignore -v` resolves .env, *.key, *.pem against wren/.gitignore. The live repo's ignore coverage is intact and is what actually protects against key commits. Root /.gitignore remains dormant defense-in-depth only. WARN closed.

---
## 2026-05-30 — Flux→Wren Coach-prompt rename (coach.ts) + 2 WorkoutScreen "many women" hedge flattens + spec §6/L5 update

### Summary
Clean on all BLOCK-tier checks — 0 blockers, 2 warnings (partial-rename user-facing inconsistency; storage-key migration caution), SDK-version discrepancy confirmed resolved.

### Findings

**1. Safety integrity (highest priority) — PASS**
- ED_SAFETY_RULES (coach.ts:98-103) byte-for-byte unchanged. All hard rules present: BMR floor + no starving/purging/fasting/earning/compensating (L99); no medical advice → own provider (L100); no supplement-by-name (L101); no body-shaming / no food moralizing (L102); care-mode/ED-distress branch with NEDA online + "NEDA" to 741741 + 988, no discontinued phone helpline (L103).
- PROACTIVE_ED_SAFETY (L112-116) unchanged.
- ED_SAFETY_RULES appended as final block on all five surfaces: SYSTEM_PROMPT (L183), SUMMARY_SYSTEM (L958), PHOTO_SYSTEM (L1046), CALIBRATION_SYSTEM (L1192), PLAN_SYSTEM (L1505). coachKickoff welcome (L924) inherits SYSTEM_PROMPT via runConversation + has inline safety close.
- All Anthropic call sites send the safety prompt (L771-773, 1006, 1097, 1401, 1630). No bypass; no user content in system slot.

**2. Rename correctness — PASS**
- grep -i flux in coach.ts → 0 matches. 6 renamed strings coherent (L119, 138, 914, 1041, 1168, 1488). No identifier/logic renamed. storage.ts unchanged (keys "flux.profile"/"flux.chat" correctly preserved).

**3. Flatten correctness — PASS**
- WorkoutScreen PHASE_LEAN L79/L82 flattened; meaning preserved, no guarantee, no earn/burn framing. grep -i women → 0 matches. BC branch (lean guard L123) unaffected.

**4. Spec accuracy — PASS**
- §6 L302 drift note accurate, does not overclaim (verified grep "many women" → 0 in coach.ts; cycle/workout/kickoff/PLAN all declarative). L5 companion filename Wren_Project_Plan_v2.md exists at root.

**5. Scope — PASS**
- Only coach.ts, WorkoutScreen.tsx, spec changed. No data model / model routing / BC branch / deps / network destination changes. No auth/payments/push/telemetry/social.

**NOTE — SDK discrepancy RESOLVED:** wren/AGENTS.md now reads "pinned to Expo SDK 54," matching reality. Known discrepancy closed.

**WARN — Partial rename leaves Flux/Wren split:** Coach prose now says "Wren" but app chrome (app.json, OnboardingScreen "Welcome to Flux", SettingsScreen "Your Flux profile", BarcodeScanner, Wren_Project_Plan_v2.md H1, spec L310 `create-expo-app flux`) still says "Flux." Visible inconsistency; finish rename in a dedicated pass, excluding storage keys.

**WARN — Do NOT rename AsyncStorage keys without migration:** storage.ts:5-6 "flux.profile"/"flux.chat" must stay unless a one-time read-old/write-new/remove-old migration ships; renaming would orphan all existing user data (silent reset to onboarding). Correctly left untouched here.

### Not checked
- Visual rendering of flattened lean card (device test).
- Live model reading of renamed prompts (prose-only, smoke-test only).

---
## 2026-05-30 — Full Flux→Wren rename (~15 files: app.json, package.json, 3 screens, 5 lib/comment files, 3 agent defs, 2 root docs)

### Summary
0 BLOCK, 2 WARN, 3 NOTE. Data-safety exclusion held (AsyncStorage keys intact). No safety constant, identifier, import, or logic touched. Every remaining "flux" intentional. Both warnings pre-existing/follow-up, not regressions.

### Findings
- [PASS] DATA SAFETY: storage.ts:5-6 verbatim unchanged — PROFILE_KEY="flux.profile", CHAT_KEY="flux.chat". No migration added; all load/save/clear paths use these exact keys. Renamed warn strings/comments are cosmetic (console.warn text, not keys). No data-loss path.
- [PASS] SAFETY INTEGRITY: ED_SAFETY_RULES (coach.ts:98-103) byte-identical (coach.ts not in rename set). Crisis resources intact (NEDA online + 741741 + 988, no discontinued phone line). BMR floor + BC branching unchanged; only cycle.ts L132 *comment* edited, phase math/BC short-circuit below untouched. No safety string altered.
- [PASS] RENAME CORRECTNESS: tsc clean consistent with static read — only prose/JSX text/comments/JSON string-values/doc-paths changed. No identifier/import/const/type/object-key renamed. app.json name/slug are config values not read by code lookups (no expo-constants usage). No "Wren Wren" doubling; user-facing strings read correctly. Full-tree grep "flux": all hits classified intentional (storage keys; Flux_Project_Plan.docx refs L3/L274; historical prior-name context in Wren_Trademark_Knockout.md, wren/CLAUDE.md, wren/AGENTS.md; Trademark_Knockout_priorname_Flux.md whole file; historical root audit-log entries). Zero category-(b) misses.
- [PASS] SCOPE: no dep change, no network-destination change, no data model/model routing change, no new file. Pure rename.
- [WARN] Two audit logs diverge: wren/.claude/audit-log.md renamed to "Wren audit log"/wren-auditor, but root .claude/audit-log.md (canonical, per CLAUDE.md) keeps original header. Two audit-log.md files with different headers risk future appends landing in the wrong file. Pre-existing structural oddity surfaced by rename, not caused by it. Consolidate or mark secondary deprecated.
- [WARN] app.json slug flux→wren can detach an existing EAS project link if ever eas-init'd under "flux". Harmless for local Expo Go dev. Flag for future EAS setup.
- [NOTE] app.json name flux→wren also changes on-device display name / Expo Go title — intended.
- [NOTE] wren/.claude/audit-log.md header renamed, historical entries left intact — correct.
- [NOTE] User-facing Flux/Wren split from prior audit now RESOLVED — app is consistently "Wren" everywhere a user sees; data keys stay "flux.*" (invisible, safe).

### Not checked
- tsc not run by auditor (no Bash); relied on implementer clean report + static read.
- App not launched; on-device app-name visual is a quick optional manual check.

---
## 2026-05-30 — Manrope typography swap (italic-serif → Manrope sans-serif, font-only; App.tsx + theme.ts + 8 screens + Stepper.tsx)

### Summary
0 BLOCK, 3 WARN, 4 NOTE. Genuinely font-scoped — no copy, color, fontSize, lineHeight, or layout value changed. ED-safety copy intact and legible (only family/italic-removal on disclaimer; size+color+wording unchanged). Warnings: 2 leftover Georgia refs in FoodScreen modals (implementer's "0 serif hits" was inaccurate), Android fontWeight-vs-family edge on 2 styles, and crisis text gated behind global font-load (degrade-to-system suggested).

### Findings
- [PASS] ED-safety legibility: Cycle disclaimer, Food "awareness only", "nothing here is required" footer, "only what you want to share", "pick as many as fit" hints, Coach crisis text (NEDA/741741/988) all byte-identical in wording. fontSize/lineHeight/color unchanged; only family→Manrope + italic removed on disclaimer. Disclaimer still colors.ink (cocoa on cream) — WCAG posture preserved. No safety meaning lost with italic.
- [PASS] Font-only scope: no fontSize/lineHeight/margin/padding/width/height/flex/borderRadius/color/copy/structure change across all 10 files. Only permitted changes: fontFamily→Manrope_*, fontStyle italic removed, redundant numeric fontWeight removed, letterSpacing/textTransform per mapping. theme.type tokens + theme.tracking {tight:-1.25, button:2.25} within rule.
- [WARN] Italic/serif: fontStyle italic 0 remaining; Fraunces 0; 'serif' family 0. BUT FoodScreen.tsx modalTitle + modalCalNum still fontFamily:"Georgia" — 2 misses; implementer's "0 hits" grep inaccurate. Flatten to Manrope_700Bold (cal number += letterSpacing -1.25). Non-safety modal surface → WARN.
- [PASS] Header vs button: lowercase "cycle"/"food" headers → Manrope_700Bold, letterSpacing -1.25, NO uppercase added (stay lowercase). Buttons → Manrope_500Medium + uppercase + letterSpacing 2.25; no button copy changed; no non-button text uppercased (pre-existing uppercase eyebrows kept transform, family only).
- [PASS] Font loading/boot: App.tsx useFonts folded into existing gate (booting || !fontsLoaded → LoadingScreen). No new layout element, same splash. Render gated on load so no safety surface paints in fallback font.
- [PASS] Scope/deps: only new deps @expo-google-fonts/manrope@^0.4.2 + expo-font@~14.0.4 (SDK54-compatible, no bump). No networked/analytics dep. No coach.ts/storage-key/data-model/model-routing/BC-branch change. fontWeight cleanup did not down-weight (display/heading explicitly 700Bold, data 600SemiBold).
- [WARN] Android fontWeight-vs-family: ProgressScreen bigStat + CoachScreen senderName left numeric fontWeight next to Manrope_* family — can render faux weight on Android. Remove numeric fontWeight on these 2 (implementer removed 38, missed these).
- [WARN] Crisis text behind global font gate: all text waits for fontsLoaded; if Manrope load ever fails, app sits on LoadingScreen and crisis surface is unreachable. Bundled fonts make this low-likelihood. Consider rendering on font-load error (degrade to system) so safety surface is never gated behind a font.
- [NOTE] theme.tracking token added, role-named, within spec. expo-font promoted transitive→explicit (correct). -1.25/2.25 mid-range of spec. App.tsx tab labels Manrope_500Medium (UI-label role, correctly not uppercased).

### Not checked
- Did not run app or tsc (no Bash); relied on implementer clean tsc. Leftover Georgia strings are valid TS so tsc won't catch them.
- No device render check; recommend visual pass of FoodScreen modals where leftover Georgia sits next to Manrope.

---
## 2026-05-30 — FoodScreen background → white (presentation)

**Change:** FoodScreen.tsx container.backgroundColor: colors.surface (cream #FBF7F0) → colors.paper (white #FFFFFF), line ~1561. Goal: white Food log page background.

**Audit result: PASS (no BLOCK). 1 WARN, 2 NOTE. tsc clean (exit 0).**

- [PASS] Scope: single-token value swap. No copy/layout/structure/logic/fontSize/spacing change. No data model, Coach prompt, BC branch, calorie-math, or BMR-floor touched. colors.paper is existing token (theme.ts:25). No scope creep.
- [WARN] (pre-existing, not worsened): ED-sensitive calorie readouts (breakValTotal/Remaining total, breakLabelTotal, macro numbers) render in colors.calm #7E9AA0 — ~2.0:1 on old cream, ~2.13:1 on new white. Both fail WCAG AA. White is marginally better, so no regression, but the most ED-relevant number remains sub-legible. Carry forward 2026-05-29 recommendation to darken colors.calm or use colors.ink for the Remaining total. Disclaimer/"awareness only" copy is cocoa/clay, fully legible (~12:1).
- [NOTE] Cream inner cards (colors.cream/colors.surface, ~lines 1622, 1711) now sit on white page — cream-on-white ~1.02:1, so fills read flat unless bordered. Cosmetic flattening; designer's eye recommended.
- [NOTE] surfaceMuted (vapor #E3E8E9) cards still separate fine on white; only cream cards flatten.

**Verdict:** Safe to ship. Follow-up on colors.calm legibility (standing item) and a designer pass on cream-card separation if crisp edges intended.

---
## 2026-05-30 — FoodScreen breakValTotal (Remaining number) font/color match

**Change:** breakValTotal style in screens/FoodScreen.tsx (line 1685): fontWeight bold→semibold, color colors.calm→colors.ink; fontSize unchanged. Goal: match Remaining number to carbs/protein/fat/fiber macro rows (macroBarLabel). Render site (line ~798, {fmt(calLeft)}) unchanged.

**Audit:** PASS — no BLOCK, no WARN, 1 NOTE. tsc clean (exit 0).
- Scope: exactly one style object changed; no copy/layout/logic/fontSize/other-style edits. No data/Coach/cycle/BC code touched.
- ED-safety legibility: IMPROVEMENT. Remaining calorie number (most ED-adjacent readout) moves from calm #7E9AA0 (~2:1, sub-WCAG-AA, flagged in prior audits) to ink #3B2F22 (~10–12:1 on cream and white). Resolves a standing contrast WARN.
- No alarm/shaming framing: new color is neutral primary ink, not red/alert; no conditional color logic added. Over-budget (negative calLeft) still renders plain neutral number via unchanged fmt(calLeft); bold→semibold marginally less shouty, consistent with non-alarmist posture.
- NOTE: sibling label breakLabelTotal (line 1684) still color: colors.calm, so label is low-contrast next to the now-ink number. Not a regression (label was already calm). Recommend moving breakLabelTotal to colors.ink/inkMuted for a consistent legible label/value pair — Ting's design call.

---
## 2026-05-30 — Cycle/Food presentation batch (FoodScreen white bg + ringNum weight + breakValTotal revert; PhaseRing ofLine spacing; CycleScreen lowercase "today" + coachQuote size)

### Summary
0 BLOCK, 2 WARN, 4 NOTE. Six edits, all presentation-only plus one copy-casing change ("Today"→"today"). tsc clean. No ED-safety copy altered; no logic/data/cycle-math/BC/Coach/storage/deps touched. Scope = 3 files (FoodScreen.tsx, CycleScreen.tsx, PhaseRing.tsx).

### Findings
- [PASS] FoodScreen container bg = colors.paper (#FFFFFF), line 1561. Presentation-only; cards (cream/surfaceMuted) keep own borders. (Re-confirm of earlier-audited change.)
- [PASS] FoodScreen ringNum (big donut "Remaining" number, ~1661): Manrope fontFamily removed so fontWeight semibold applies; fontSize 26 + colors.ink unchanged. Now matches macroBarLabel (system font + semibold + ink). Big number is ink on cream calCard (~10:1) — excellent legibility on the most ED-adjacent number.
- [WARN] FoodScreen breakValTotal (~1685) reverted to {body, bold, colors.calm} — net-zero vs pre-batch. Pre-existing low-contrast: calm #7E9AA0 on cream ~2.2:1 (below WCAG AA) for the SMALL breakdown "Remaining" value (NOT the big ring number). Deliberate current state after revert. Recommend darkening colors.calm or moving this value to colors.ink. Owner/design call, non-blocking.
- [PASS] PhaseRing ofLine marginTop -38->-22 (line 241) — pure spacing to fix "1"/"of 28" overlap. Only ofLine changed; numeral/denominator/phase stack + safeDay/safeLength clamp math + arc/dot geometry untouched. [NOTE] tuned off-device; confirm visually no residual overlap and not too tight.
- [PASS] CycleScreen editor title lowercase: editorLabel returns "today" (line 207) + modalTitle textTransform "lowercase" (line 1130). Scoped to modalTitle only; modalEyebrow still uppercase via editorEyebrow().toUpperCase() — unaffected. Copy-casing + transform only.
- [PASS] CycleScreen coachQuote size: fontSize headline->body (16), lineHeight 28->22 (lines 864-865). PHASE_GUIDANCE strings (58-66) byte-for-byte UNCHANGED — flat-declarative, "Be soft with yourself" intact, no shaming/restriction. fontFamily/color unchanged.
- [WARN] coachQuote is ED-safety-adjacent guidance now at 16dp ink on cream/white (~10:1) — still well above AA, legibility NOT crossed. Flag only as "shrinking safety-adjacent copy warrants explicit sign-off," which Ting gave. No action.

### Scope confirmation
- 3 files only (FoodScreen.tsx, CycleScreen.tsx, PhaseRing.tsx). No lib/ (cycle math, coach prompt, storage, types, safety, targets), data model, BC branch (showCycleUI/phaseForDate untouched), Coach/Anthropic calls, deps, or app.json. Only non-presentation change is one copy-casing edit. No disclaimer/safety-footer copy touched or resized.

### Not checked
- On-device render of ring "1 of 28" spacing post -22, and cream-card-on-white separation (math supports both; visual confirm for Ting).
- Broader Manrope swap state (out of scope; edits consistent with it).

---
## 2026-05-31T00:00:00Z — Food domain batch (Log-food reskin, capture choosers, purple→wren sweep, two-step delete, save/unsave toggle)

Summary: Clean on all BLOCK-tier checks (ED-safety, Coach prompt, BC branching, scope, keys). 2 WARN, 3 NOTE. No blockers.

Findings:
- [NOTE] ED-safety surfaces clean. Food disclaimer (FoodScreen.tsx:955-958) unchanged. No earn/deficit/burn-off/shame framing in new copy. + Exercise row (FoodScreen.tsx:856-868, :114-117) never lowers BMR-floored budget. No gamification tied to eating less.
- [NOTE] BC/phase branching intact despite phaseColors import removal. Eyebrow uses phaseForDate (FoodScreen.tsx:124-132); guard !onBirthControl && phase!=="unknown" (:130) drops phase for BC (cycle.ts:147-148) and unknown users. Zero surviving phaseColors refs. Removal safe.
- [NOTE] Coach safety prompt untouched. No Anthropic call-site changes. tellCoach (FoodScreen.tsx:408-411) only closes sheet + switches tab; no user content into Coach context. No injection surface added.
- [NOTE] Two-step delete safe. deleteEdit (FoodScreen.tsx:702-711) removes by captured {date,id} — no stale-row risk. Confirm required: DELETE THIS ENTRY only sets confirmDelete=true (:1439); removal only via confirmDeleteEntry (:715-718). Both close paths clear confirmDelete+editEntry (:709-710) — no orphaned overlay. In-sheet overlay (:1453-1480) gated on confirmDelete inside Edit Modal, unmounts with sheet (fixes prior stacked-Modal iOS misfire). Inner no-op TouchableWithoutFeedback stops backdrop pass-through. onRequestClose (:1366-1368) dismisses overlay then sheet. removeEntry by id = no double-delete.
- [WARN] FoodScreen.tsx:751-752 — inline sameFood identity currently correct (matches food.ts:164-165 byte-for-byte) but is a duplicated private helper (clean extraction was permission-denied). Drift risk: if food.ts sameFood changes, inline copy silently desyncs and unsave breaks (star stuck saved). Toggle otherwise correct/idempotent (recomputes match vs latest profile :757, removeSavedFood by id is no-op if gone). Fix when permitted: export sameFood / findSavedFoodMatch from food.ts; meanwhile add paired warning comment in food.ts.
- [NOTE] No scope creep. Only out-of-Food change is onOpenCoach prop in App.tsx:194 (mirrors WorkoutScreen :208). No new networked deps/fetch/auth/payments/push/telemetry/social. Local-only intact.
- [NOTE] No dependency/version drift. All new imports (expo-image-picker, react-native-svg, Alert, TouchableWithoutFeedback) declared in package.json, ship with SDK 54. expo ^54.0.34 still agrees with AGENTS.md SDK 54 pin. No native deps added.
- [WARN] App.tsx:34 — const ACCENT="#7c3aed" survives and drives active tab-label color (:253). Purple sweep deleted ACCENT in FoodScreen (confirmed) but tab bar still highlights active tab in old purple — palette inconsistency vs the now-warm Food screen. Out of this batch's file scope (tab-bar redesign deferred per memory), so WARN not BLOCK. Sweep not complete app-wide. Ting to decide: swap to colors.ink/colors.calm now or with deferred tab-bar work.
- [NOTE] warmAlert #B8765E legibility: cream-on-warmAlert DELETE pill fine; warmAlert text on cream (deleteQuietText :2742-2748) ~3:1 — near WCAG floor for 11px tracked caps, acceptable as quiet non-safety decorative label, severity carried by two-tap flow. Device check advised.
- [NOTE] TS hygiene: no new any/as any. catch(e:unknown) narrows via instanceof Error (FoodScreen.tsx:367,:372). State machines have defined error paths (Alert + finally). No new unhandled promise / stuck state.

Not checked: on-device visual contrast (warmAlert text, new SVG icons); tsc --noEmit clean compile; behavioral test of iOS in-sheet delete overlay; npm audit advisories. All require tooling/device the read-only auditor cannot run.

---
## 2026-05-31T00:00:00Z — follow-up: findSavedFood extraction + tab-label purple removal (resolves 2 WARNs from 2026-05-31 batch audit)

### Summary
Clean — 0 blockers, 0 warnings, 1 note. Both prior-batch WARNs correctly resolved.

### Change 1 — findSavedFood extraction (wren/lib/food.ts, wren/screens/FoodScreen.tsx)
- VERIFIED food.ts:174-176 — findSavedFood placed after isFoodSaved, exported, routes through private sameFood (same identity as isFoodSaved/saveFood/removeSavedFood dedupe). sameFood remains single source of truth; duplication eliminated.
- VERIFIED FoodScreen.tsx:746-762 — toggleSaveFood calls findSavedFood(p, food) inside the persist transform against latest p, then removeSavedFood(p, match.id) only if matched (match ? ... : p). Unsave behavior unchanged: removes correct row by real id, idempotent (no match → returns p), recomputes against latest profile so a concurrent Coach write can't cause wrong-row delete. Render-time isFoodSaved gate (line 749) only selects branch; mutation re-derives match, so stale snapshot worst case is a no-op.
- No other consumers; purely additive export.

### Change 2 — tab-label purple removal (wren/App.tsx)
- VERIFIED import { colors } from "./lib/theme" present (line 13). tabLabelActive now { color: colors.ink } (line 252). Inactive tabLabel stays "#999" (line 251). Grep for #7c3aed / ACCENT: no matches. No layout change.

### Scope / drift
- No scope creep (exactly the 2 described files). No new deps, no new network calls, no auth/push/payments/telemetry/social.
- SDK pin intact: package.json "expo": "^54.0.34" agrees with AGENTS.md SDK 54 pin.
- No data-model drift; findSavedFood returns SavedFood | undefined; no migration needed.
- No new any/as any; return type explicit; promise awaited.
- ED-safety / Coach prompt / prompt-injection / birth-control surfaces NOT touched by either change — confirmed.
- No API key material in either file.

### NOTE
- FoodScreen.tsx:682,749,1248 — isFoodSaved still reads render-time profile prop. Harmless now (in-transform re-derivation makes worst case a no-op), but if mutation logic ever moves out of the transform this gate could become a wrong-row risk. Context only.

### Not checked
- Visual rendering of active label in colors.ink (device run needed; confirm active/inactive contrast).
- tsc/lint not run (no Bash); implementer should confirm clean typecheck.

---
## 2026-05-31T00:00:00Z — Onboarding screen design+implementation pass (OnboardingScreen.tsx reskin to wren_onboarding_redesign.html + behavior rewire)

# Audit report — 2026-05-31 — Onboarding screen design+implementation pass

## Summary
Clean. 0 BLOCK, 0 WARN, 4 NOTE. ED-safety copy preserved verbatim and legible, calibration safety contract intact (no body-weight estimate surfaced, goal weight BMI-floored in lib/coach.ts and never written to Coach memory), BC branching clean, no silent defaults, no scope/data-model/dependency drift. Change confined to screens/OnboardingScreen.tsx.

## Findings

1. ED-safety copy + legibility — PASS
- screens/OnboardingScreen.tsx:942-949 — Snapshot infocard "How your targets are set" verbatim to mockup ("computed from your stats with a safe minimum floor… refine the plan, they don't replace the math"). Rendered cocoaSoft on creamTile at 12.5px/19.5 (infocardBody:1673) — legible, not light clay.
- screens/OnboardingScreen.tsx:1054-1057 — Cycle learning copy verbatim ("A starting estimate. Log periods over time and Wren learns your real average."). No feelings-override frame reintroduced.

2. Calibration safety contract — PASS
- screens/OnboardingScreen.tsx:523-528 — calibratedFacts built ONLY from goal_direction/training_emphasis/motivation/body_fat_range; goal_weight deliberately NOT pushed to facts, so never reaches Coach memory via addMemory in finish() (645-661).
- screens/OnboardingScreen.tsx:1149-1178 — Re-homed recap card renders only calibratedFacts + optional goal-weight stepper. No current/body-weight estimate rendered. body_fat_range is qualitative by schema (lib/coach.ts:1254-1258).
- lib/coach.ts:1501-1517 — Goal-weight BMI>=18.5 floor (untouched) rejects (not clamps) sub-floor/unvalidatable suggestions. OnboardingScreen.tsx:550-558 also clamps to WEIGHT_MIN/MAX_LB. goalWeight reaches Profile only when touched (622).

3. Birth-control branching — PASS
- screens/OnboardingScreen.tsx:1018-1059 — Period-date + cycle-length fields render only under !onBirthControl. No phase/day-of-cycle label in onboarding.
- screens/OnboardingScreen.tsx:610-611 — finish() seeds lastPeriodStart:"" and avgCycleLength:28 when onBirthControl, regardless of stale state. No cycle field leaks.

4. No silent defaults / touched-gating — PASS
- height/weight/goalWeight/waist/neck/hip write only when *Touched (387-396, 622, 640-642). DOB writes "" until birthDateChanged (366-377, 572-573, 591-594).
- Measurements expander (renderMeasurementSteppers:1251-1288) reuses the same waist/neck/hip state + touched-setters; Navy path preserved (runCalibration builds partial from touched-gated values, 501-507).

5. No data loss in finish()/persistence — PASS
- screens/OnboardingScreen.tsx:604-664 — Profile shape unchanged: all maps/arrays, calorieMode:"static", photo URIs, photoLog seed, waistIn/neckIn/hipIn all written. Coach-memory seeding (diet facts, note, calibrated facts, personal-goal facts) preserved 645-661.

6. Scope/drift — PASS
- Confined to screens/OnboardingScreen.tsx. No edit to lib/coach.ts (Coach prompt/Anthropic call/calibration contract untouched), lib/types.ts, or package.json. Imports unchanged. No new networked deps, no auth/payments/push/telemetry/social. Env-key posture unaffected.

7. Navigation integrity — PASS
- STEPS = 9 entries, no measurements step (175). INDEX_LABELS (181-191) and OPTIONAL_STEPS=["body","photos","cycle","diet"] (193) match spec; measurements absent from all three.
- Skip advances via setStepIndex(min(i+1,len-1)) (1215), shown only for optional steps when stepIndex>0 (1212).
- onDone(startDest) fires both branches in finish() (664, 666 incl. catch). startDest default "workout" (309) matches pre-selected "Build my plan".

## NOTEs (non-blocking)
- [NOTE] screens/OnboardingScreen.tsx:309 — startDest default coach->workout per mockup. onDone("workout") parent routing confirmed in App.tsx (setTab("workout") + planSetupSignal bump).
- [NOTE] screens/OnboardingScreen.tsx:572-573 — body-exit applyBirthDate backfill duplicated in finish() (591-594); idempotent, harmless.
- [NOTE] screens/OnboardingScreen.tsx:501-506 — partial Profile built via `as unknown as Profile` (pre-existing) feeding navyBodyFatPercent; bypasses type-check on that path. Acceptable; flag if bodycomp inputs widen.
- [NOTE] screens/OnboardingScreen.tsx:42-45 — Manrope 800 approximated with 700Bold + tight tracking (only 400-700 bundled). Presentation-only fidelity gap, documented in-file.

## Resolved post-audit
- onDone("workout")/("coach") parent routing confirmed in App.tsx (workout->workout tab + planSetupSignal, coach->coach tab).
- tsc --noEmit confirmed clean by implementer.

---
## 2026-05-31 — Onboarding fidelity pass (2nd design pass, OnboardingScreen.tsx: CTA sheen animation, Manrope_800ExtraBold/displayHeavy, gradient/shadow depth)

### Summary
Clean — 0 blockers, 0 warnings, 4 notes. ED-safety copy preserved verbatim and legible, calibration safety contract intact, birth-control branching intact, no scope creep, no data-model change, animation leak-safe and behavior-neutral, only dependency change is the approved Manrope_800ExtraBold font.

### Findings
- [NOTE] OnboardingScreen.tsx:325-337 — Sheen animation cleanup correct: Animated.loop started in useEffect, cleanup returns loop.stop(); useNativeDriver:true on translateX/skewX (UI thread, no JS cost). Bar is pointerEvents="none"; onPress owned by underlying TouchableOpacity — cannot intercept the CTA. No leak, no behavior regression. SheenPill lacks explicit accessibilityRole/State but that is pre-existing, not introduced here.
- [NOTE] App.tsx:58-64,169 + lib/theme.ts:155-177 — Manrope_800ExtraBold genuinely added to useFonts and render gated on fontsLoaded (spinner before onboarding paints). ty.displayHeavy → "Manrope_800ExtraBold"; all consumers (title, wordmark, heroLede, coachPill) use family string only, no companion fontWeight. No unbundled weight relied on. package.json confirms expo-linear-gradient is NOT a dependency — gradient/sheen faked without a lib, no silent dep added. (Supersedes the 2026-05-31 NOTE re: 800 approximated with 700Bold — now genuinely bundled.)
- [NOTE] OnboardingScreen.tsx:1035-1043,1810-1811 / 1148-1151 / 1093-1110 — ED-safety + calibration copy preserved verbatim in meaning, legible contrast. Safety disclaimer is cocoaSoft (#6B5641) on creamTile (#F5EFE6), 12.5px/19.5 (~6.7:1, well above threshold). cardHighlight overlay sits behind text (pointerEvents none, rendered first), 50%-white wash on top 62% only — does not put text on a gradient edge. "Safe minimum floor"/"refine the plan, don't replace the math" framing intact, not softened.
- [NOTE] OnboardingScreen.tsx:616-625,715,738-754 + lib/coach.ts:1501-1521,241 — Calibration safety contract holds. calibratedFacts = qualitative only (goal_direction/training_emphasis/motivation/body_fat_range), never a body-weight number; only those + diet/personal-goal strings enter coachMemory via addMemory. Goal weight is a separate channel: profile.goalWeight set only when goalWeightTouched, photo-derived value passes deterministic BMI-18.5 floor (reject-not-clamp, rejects if height missing) before apply, reaches Coach as a structured stat (coach.ts:241) not a memory fact. BMI-floored + out of Coach memory — confirmed.
- Confirmed: BC branching intact (switch hides period picker + cycle stepper, substitutes feel-based copy, finish() writes lastPeriodStart "" / avgCycleLength 28 — no phase/day-of-cycle leak). No scope creep (no networked deps/fetch/auth/push/payments/telemetry/social). Data model unchanged, no migration concern. Depth additions are presentation-only.

### Not checked
- On-device visual rendering of sheen/font/shadows (requires device run).
- tsc: OnboardingScreen.tsx independently confirmed 0 errors. `as unknown as Profile` at :599 is pre-existing.
- Out of scope, noted: WorkoutScreen.tsx has 28 undefined-`ACCENT` references (tsc exit 2) — separate pre-existing breakage, unrelated to this onboarding pass. ProgressScreen.tsx still hardcodes `const ACCENT = "#7c3aed"` (old purple). Both flagged to Ting.

---
## 2026-05-31T00:00:00Z — Workout tab redesign (design-pipeline step 3: designer restyle + implementer integrity PASS)

**Summary:** Clean — 0 blockers, 0 warnings, 3 NOTEs. Presentation-only restyle of `screens/WorkoutScreen.tsx` to match `brand/Workout Tab_/wren_workout_redesign.html`. ED-safety copy, BC branch, and Coach surfaces all intact.

**Findings:**
- [PASS] ED-safety — `WorkoutScreen.tsx:92-97` PHASE_LEAN strings flat-declarative; no "many women"/feelings-override/body-shaming/earn-burn framing. Rest note "Rest and recover." (786) neutral. Burn/score surfaces are neutral logged data, no deficit/goal framing.
- [PASS] Legibility — leanText (1390) & dayNote (1681) render cocoaSoft #6B5641 on cream/white (~7:1, above AA body bar). Verified.
- [PASS] Scope/drift — presentation-only; no data model / storage / Coach-prompt change; only Coach touch is generateWeekPlan call site (unchanged). No new dependency, no SDK bump (SDK 54 pin untouched).
- [PASS] Token integrity — no #7c3aed/#e8833a/ACCENT/BURN hexes remain; only literals are #fff (white-on-cocoa spinner) and rgba backdrop scrim. All colors.* refs resolve to real theme.ts tokens.
- [PASS] BC / Coach branching — cycle.ts:147 returns dayOfCycle:null on BC; WorkoutScreen suppresses leanCard (137), shows phase label only with no fabricated day (144-145), eyebrowColor falls back to inkMuted (148, no crash). No Coach-prompt or BC-substitution surface touched.

**NOTEs (non-blocking):**
- `WorkoutScreen.tsx:80-82` — `const ACCENT = colors.accent` was declared-but-unused (dead code). RESOLVED by orchestrator post-audit: dead const + stale comment removed; `grep ACCENT/BURN` now returns 0 and `tsc --noEmit` still exits 0.
- `theme.ts:137` — `honey` token unused in WorkoutScreen (strip dot uses profileSage, pill mark uses checkDone). Unused token, not a defect.
- `WorkoutScreen.tsx:606,914` — 🔥 burned-cal surfaces neutral/pre-existing; watch for ED-trigger creep if gamification/streaks ever layer onto dayBurned.

**Device-check handoff:** eyebrowColor (148) tints the 9px eyebrow with phaseColors[phase].bg; for ovulatory that is powder blue #BDD0D6 on cream, likely sub-4.5:1. Decorative metadata, not safety copy → informational, not WARN. Glance on device.

**Not re-run by auditor (no Bash/git):** tsc exit 0 and ACCENT/BURN grep — independently confirmed by orchestrator. Git line-diff vs prior committed state — copy verified against established flat-declarative voice.

**Verdict: PASS — no BLOCK. Workout redesign safe to report complete.**

---
## 2026-05-31T00:00:00Z — Workout tab STRUCTURAL REBUILD (designer +1105/−357: week-at-a-glance accordion, log rebuild; implementer: onOpenSettings prop + weekday-param handlers)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. The rebuild is presentation/structure-only; ED-safety, BC branching, scope, data model, and token integrity all hold. (Supersedes the earlier same-day recolor audit — that pass only swapped colors; this pass rebuilt the IA to the mockup after Ting flagged "it looks nothing like the mockup.")

## Findings
- [PASS] ED-safety — PHASE_LEAN (89-94) flat-declarative; NEW weekly-summary (1124-1148) factual sessions done/total/left, no deficit/earn-burn/streak/pressure. whyThisWeek/notes/programName are Coach-generated (rendered only). Grep earn|burn off|deficit|shame|guilt → only the code comment at 1126 documenting avoidance.
- [PASS] Legibility — programWhy/restNote/leanText/summaryText 13px fontRegular cocoaSoft on creamTile/paper; not buried.
- [PASS] Scope/drift — toggleDayDone(weekday=selectedWd):443, toggleExercise(exId,weekday=selectedWd):368 only thread the card's weekday into the existing persist path; estimation/data logic untouched. Removed burnText/dayBurned/dayMinutes were per-day burn DISPLAY helpers; burn still in LogCard (KCAL·MIN); ED-positive (less burn prominence on plan card). No data-model/storage/plan-gen/Coach-prompt change.
- [PASS] onOpenSettings prop + avatar — optional prop (157); TouchableOpacity w/ a11y when present, non-pressable View when absent; App.tsx passes pure setTab("settings"). No leak.
- [PASS] Token integrity — all colors resolve to brand tokens; intensity pins light→calm/moderate→handle/hard→period/rest→divider; dots honey/profileSage/checkDone. No stray off-brand hex.
- [PASS] BC/Coach — lean gated by !onBirthControl (165); eyebrow + log date line use dayOfCycle!=null guard (172-173, 1077) → no fabricated cycle day on BC; no day-claim leak on program/day cards.

## Notes (non-blocking, cosmetic)
- [NOTE] WorkoutScreen.tsx:1071-1077 — Log date line shows "· On birth control" daily for BC users; factual/safe, slightly clinical.
- [NOTE] Dead styles remain post-rebuild (dayPills/dayPill*/dayCard/planSection/scoreRow/scoreCard/logHeader/card* — 1647-1661, 1832-2016). No runtime effect; future cleanup.
- [NOTE] WorkoutScreen.tsx:691 — fromPlan keys off e.source==="coach"; correct today; a future non-plan entry logged as "coach" would mislabel. No action now.

## Not checked
- On-device rendering / contrast ratios (static assessment only).
- Runtime content of generated whyThisWeek/notes (live Coach call, not modified).
- Unrelated OnboardingScreen.tsx tsc failure (bloom/ambient/screenSheen keys, mtime 20:55, parallel edit) — NOT attributed to Workout work; WorkoutScreen.tsx + App.tsx compile clean in isolation.

**Verdict: PASS — no BLOCK. Workout structural rebuild safe to report complete.**

---
## 2026-05-31 — Onboarding 3rd fidelity pass (display scale-up ~1.18x + ambient react-native-svg mesh, grain omitted, Stepper valueFontSize prop)

# Audit report — Onboarding 3rd fidelity pass

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Presentation-only pass; ED-safety copy legibility, BC branching, calibration safety contract, scope/dependency/SDK pin, data model, and the Stepper default path all intact. Orchestrator independently ran `npx tsc --noEmit` → clean (exit 0), closing the auditor's "couldn't run tsc" item.

## Findings

### ED-safety copy legibility — PASS
- "How your targets are set" infocard (OnboardingScreen.tsx:1160-1168) renders on an OPAQUE creamTile (#F5EFE6) tile with overflow:hidden (styles.infocard :1957/1964) — fully occludes the ambient mesh; no bloom washes under the disclaimer. cardHighlight overlay is a top white wash (lightens) at pointerEvents none, behind text.
- Safety copy VERBATIM (:1163-1167): "...computed from your stats with a safe minimum floor ... they refine the plan, they don't replace the math." Scaled 12.5→15px (:1986), helps legibility, no wording change.
- Calibration recap (styles.recap :2090) also opaque creamTile + overflow:hidden.
- BC switch copy verbatim both branches (:1224-1226); cycle micro-copy verbatim (:1276).
- No safety text renders over a translucent mesh bloom.

### Touch-blocking / behavior integrity — PASS
- Ambient stack is FIRST root child, wrapped pointerEvents="none" (:902-906); whole Svg + screenSheen subtree non-interactive per RN semantics; ScrollView/footer paint on top and own touches.
- No handler/state/nav/calibration-pipeline/BC-branching/finish()/initProfile/SheenPill change. SheenPill sweep bar itself pointerEvents none (:363).

### Calibration safety contract — PASS (re-verified)
- calibratedFacts qualitative-only (:713-717), no body-weight estimate.
- Goal weight BMI-floored deterministically (coach.ts:1486-1517) + clamped to WEIGHT_MIN/MAX (:743). CalibrationResult excludes a "where she is now" weight (coach.ts:1212,1521).
- Goal weight stays OUT of Coach memory: finish() pushes only chipFacts/dietNotes/calibratedFacts/personal-goal strings into addMemory (:835-851); goalWeight only to Profile.goalWeight field (:812).

### Scope / dependency / drift — PASS
- No new package: react-native-svg already declared 15.12.1 (package.json:20); import uses only natively-backed Svg/Defs/RadialGradient/Stop/Rect, SDK-54 compatible.
- Expo pin holds: package.json:10 = expo ^54.0.34, matches AGENTS.md SDK 54.
- New mesh* + meshBackdrop + screenSheen tokens (theme.ts:160-166) decorative only, none text-on-color; rgbaSplit correctly maps alpha→stopOpacity.
- Stepper valueFontSize optional + backward-compatible (Stepper.tsx:93,103,118-120); default undefined → original size; overrides only committed (non-dimmed) value. WorkoutScreen call (WorkoutScreen.tsx:1414-1421) passes NO valueFontSize — default path byte-for-byte unchanged.
- No data-model change, no Coach-prompt change, no networked surface.

## Notes
- [NOTE] Grain intentionally OMITTED (OnboardingScreen.tsx:421-428). Ting requested grain; omitted because rn-svg 15.12.1 has no native FeTurbulence backing and RN has no mix-blend-mode:multiply. Per brief ("half-broken grain worse than none"). Tracked for possible future pre-baked noise PNG — Ting to decide.
- [NOTE] Stale token comment (theme.ts:140-159) still says mesh render is "HANDED TO THE IMPLEMENTER / see TODO block" but the AmbientMesh has landed and no TODO remains. Doc-only, cleanup on next touch.
- [NOTE] Index header (OnboardingScreen.tsx:902-906) decorative labels may sit over the top-left bloom + screenSheen; ink-on-warm-light remains legible (not safety copy). Re-check if a future pass darkens a bloom.

## Not checked
- Device rendering of the ~1.18x scale-up and two-line {"\n"} titles at 50px on 430pt — requires simulator/device.
- Live SVG RadialGradient fidelity vs mockup blooms — requires device; objectBoundingBox fraction math correct.
- Deferred ProgressScreen hardcoded-purple — known/deferred, not re-flagged.

**Verdict: PASS — no BLOCK. tsc clean (orchestrator-run). Grain omission is the one open item for Ting.**

---
## 2026-05-31 — OnboardingScreen grain texture overlay (tiled grain.png in ambient layer)

**Summary:** Clean — 0 blockers, 0 warnings, 3 notes. Completes Ting's requested mesh+grain+highlight background. Grain overlay correctly scoped to the behind-content ambient layer; cannot reach ED-safety copy, cannot intercept touches, no dependency or data-model drift.

**Findings:**
- [PASS] ED-safety legibility — grain `<Image>` (OnboardingScreen.tsx:911-915) is in the styles.ambient absoluteFill layer rendered before the ScrollView (:934). "How your targets are set" infocard (styles.infocard :1973-1988) and calibration recap (styles.recap :2107-2115) both render on opaque colors.creamTile (#F5EFE6) tiles with overflow:"hidden", fully occluding the 4.5% grain. Safety body copy verbatim, cocoaSoft on creamTile 15px (:2003, :2132). No safety text renders over the grain.
- [PASS] Touch-blocking — grain Image nested in <View pointerEvents="none"> (:905); RN parent pointerEvents="none" disables the whole subtree. Image has no handler and sits behind content. Omitting pointerEvents on the Image (ImageProps typing) does not weaken the guarantee.
- [PASS] Z-order — backdrop → mesh blooms (:906) → grain (:911) → screenSheen (:917) → content (:934). Grain behind content.
- [PASS] Scope/drift — asset at wren/assets/grain.png (180x180 RGBA, baked via stdlib, fixed seed) via static require(); no new npm dependency (package.json grain grep empty); core RN Image; no data-model/Coach/navigation/behavior change; opacity 0.045 (:1624); resizeMode="repeat" valid on SDK 54 / RN 0.81.
- [PASS] Prior "grain omitted" comments replaced with accurate descriptions (:421-430, :893-904).
- [NOTE] :1602 stale comment "Decorative mesh/grain/orb omitted" — mesh+grain now implemented (only orb still omitted); cosmetic only.
- [NOTE] :1624 redundant width/height:"100%" alongside absoluteFillObject; harmless.
- [NOTE] Orchestrator confirmed tsc clean (exit 0).

**Not checked:** on-device visual rendering of the 0.045 grain (no device access); ProgressScreen purple item deliberately not re-flagged (deferred); Coach prompt / injection / cost guardrails out of scope for this presentation-only diff.

**Result: PASS — safe to report done.**

---
## 2026-05-31 — OnboardingScreen seam-fix (ScreenSheen/CardHighlight SVG gradients) + white footer

## Summary
Clean — 0 blockers, 0 warnings, 2 notes. Both presentation changes safe: ED-safety legibility preserved (gradient white-only, lightens only, behind verbatim safety copy on opaque tile), seams genuinely fixed (fade to stopOpacity 0, clipped, non-interactive), footer chrome-only, no scope/dependency/data-model/Coach/navigation drift. Orchestrator independently confirmed tsc clean (exit 0).

## Findings
- [NOTE] OnboardingScreen.tsx:2221 — White footer deviates from the mockup (mockup is translucent cream rgba(252,250,245,.85)+blur; now solid colors.paper #FFFFFF). Ting's explicit direct request — intentional, not a regression. Footer chrome-only (Back/Skip/CTA), no ED-safety/Coach/BC copy. One-token change (colors.surface → colors.paper); colors.paper = #FFFFFF (theme.ts:26).
- [NOTE] OnboardingScreen.tsx:510-522 — CardHighlight hardcodes rgb(255,255,255)/0.5 inline rather than a theme token (ScreenSheen reads colors.screenSheen). Cosmetic inconsistency only; flag for a future token-hygiene pass.

### Checklist results
1. ED-SAFETY LEGIBILITY (primary) — PASS. Infocard safety copy (1228-1232) verbatim. CardHighlight white-only, 0.5→0 over 90% height — only lightens, no hard stop; renders behind text (first child, 1226). creamTile opaque + unchanged (2009). Text 15px/22.5 unchanged (2038).
2. SEAM-FIX CORRECTNESS — PASS. Both overlays end at stopOpacity 0 (ScreenSheen offset 0.32 / CardHighlight offset 0.9) — no hard edge. ScreenSheen in pointerEvents="none" ambient layer (957), behind content; CardHighlight Svg pointerEvents="none" (512). All 3 sites overflow:hidden + borderRadius (infocard 2012/2016, recap 2147/2150, choice 2183/2187) — clipped, no bleed. Choice TouchableOpacity press unaffected (overlay non-interactive, only mounts when on, 1639).
3. WHITE FOOTER — PASS. One-token, chrome-only, colors.paper = #FFFFFF.
4. SCOPE/DRIFT — PASS. LinearGradient added to existing react-native-svg import (18) — no new dep, SDK 54 intact. No data-model/Coach/navigation/behavior/copy change. No dangling refs to removed screenSheen/cardHighlight STYLE entries.
5. tsc — orchestrator-confirmed clean (exit 0).

## Not checked
- On-device visual rendering of gradients (seamlessness, grain interaction, banding) — needs device run; code-level math correct.
- ProgressScreen deferred-purple item intentionally not re-flagged.

**Result: PASS — safe to report done.**

---
## 2026-05-31T00:00:00Z — Workout tab fidelity-fix round (design-pipeline step 3, deltas vs device screenshots)

**Summary:** Clean — 0 blockers, 0 warnings, 3 notes. ED-safety copy intact, BC/unknown-phase gating correct, presentation/wiring-only, no data/model/Coach drift. Round closes gaps Ting found putting the running app beside the mockup: Log strip → 7-day Mon–Sun week strip; unknown-phase eyebrow/suffix hidden; avatar "·" fallback removed; rest-card spacing tightened.

**Findings (all NOTE, non-blocking):**
- [NOTE] WorkoutScreen.tsx:175 — `hasPhase = phase.phase !== "unknown"` is the correct gate. cycle.ts returns "unknown" only for no-data cases; BC returns distinct "on birth control". Gating on raw phase.phase (not the truthy capitalized phaseLabel) is right.
- [NOTE] WorkoutScreen.tsx:181-185,1149 — No cycle-day leak on BC or unknown. Eyebrow: unknown→"" hidden, BC→label only (dayOfCycle null, no fabricated day), real→"Phase · Day N". Log date suffix label-only, never a day. Both surfaces clean.
- [NOTE] WorkoutScreen.tsx:1098,1117 — Log strip no-plan path safe: week?.startDate ?? mondayOf(new Date()); pure date math, no throw, this-week-only dates; binds existing setLogSelDate, no new data path.

**Verified clean:** ED-safety copy unchanged (PHASE_LEAN/whyThisWeek/weekly summary/day notes verbatim, flat-declarative; hiding phase eyebrow on unknown suppresses a label, not a disclaimer). Scope: presentation + local UI state only; no data model/storage/plan-gen/Coach-prompt/dependency change. Tokens: restBody/restBodyNote/leanText/logDateLine/avatarText/eyebrow/stripTrained all brand tokens (lone #fff at 1526 ActivityIndicator is pre-existing, not this round). Legibility: restBodyNote = cocoaSoft 13/20 on cream, only marginTop dropped (compensated by restBody.paddingTop:12). Avatar: profileInitials returns "" not "·", both branches render a clean circle.

**Intentional UX tradeoff (awareness, not safety):** Log strip is now 7-cell Mon–Sun bound to setLogSelDate; cross-week past-day log review removed — log review is now THIS-WEEK-ONLY. Matches mockup; genuine capability reduction. Ting to confirm acceptance.

**Side note (out of scope, not a regression):** lib/cycle.ts:132 still carries a "many women find" code COMMENT (not rendered copy).

**Result: PASS — no BLOCK.**

---
## 2026-05-31 — Onboarding safe-area full-bleed layout fix (App.tsx + OnboardingScreen.tsx)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. Layout-only change; onboarding gating, main-app branch, ED-safety, and scope all intact. Orchestrator confirmed tsc clean (exit 0). Fixes the pale strip above the mesh: onboarding background now full-bleed behind the status bar, content respects safe-area insets.

### Findings
- [NOTE] App.tsx:197-213 — Onboarding branch moved outside the inset SafeAreaView, still inside SafeAreaProvider, StatusBar style="dark" preserved, onDone/initProfile wiring byte-for-byte identical (workout → setTab+planSetupSignal bump; else coach). Gating chain loading||!fontsLoaded → !profile → main app preserved; returning users still skip onboarding; load-failure catch still falls through to first-run.
- [NOTE] OnboardingScreen.tsx:1499 — Footer paddingBottom: Math.max(28, insets.bottom + 16) on colors.paper (white) last-sibling of the full-bleed root absorbs the home-indicator inset, reaches the bottom edge with no new gap, Math.max floor preserves original 28px on zero-inset devices.

### Checklist
1. Onboarding gating intact — PASS. Three branches in SafeAreaProvider; StatusBar in each; onDone/initProfile unchanged; loading gate + catch-to-first-run preserved.
2. Main tab app untouched — PASS. App.tsx still SafeAreaView edges top+bottom, same #fff backgrounds, same mounted/hidden tab pattern, same six screen props + TabBar.
3. Safe-area correctness — PASS. useSafeAreaInsets from existing react-native-safe-area-context. Insets applied to content only: index header insets.top+10, hero insets.top+28, footer Math.max(28, insets.bottom+16). Ambient layer stays absoluteFill on full-screen root. Welcome step clears status bar via hero's own inset.
4. ED-safety / scope — PASS. No Coach-prompt/calibration/BC-branching/data-model/copy/dependency change. Infocard still on opaque creamTile + overflow:hidden; ambient layer still pointerEvents="none". BC copy intact; finish() still zeroes lastPeriodStart / forces avgCycleLength=28 on BC. No new deps.
5. tsc — orchestrator-confirmed clean (exit 0).

### Not checked
- Device/simulator visual confirmation (pale strip gone; content clears status bar / home indicator).
- ProgressScreen purple item intentionally not re-flagged (deferred).

**Result: PASS — safe to report done.**

---
## 2026-05-31 — Onboarding footer made transparent + top hairline removed

**Change:** OnboardingScreen.tsx footer style: backgroundColor colors.paper(white) → "transparent"; removed top hairline border. Purpose (Ting): let the warm full-bleed mesh show through behind BACK/SKIP/CONTINUE, unbroken to the bottom edge, no dividing line. (Reverses the earlier white-footer change.) Orchestrator confirmed tsc clean (exit 0). Stale "hairline top" comment also corrected.

**Summary:** Clean — 0 blockers, 0 warnings.

**Findings (all NOTE):**
- Layout confirmed: footer (L1499) is a flex sibling AFTER </ScrollView> (L1494), NOT absolute. Ambient mesh is the absolute-fill layer behind all content. Transparent footer reveals the mesh; ScrollView content does NOT bleed behind the buttons (non-overlapping flex boxes).
- Chrome-only: footer holds Back/Skip/SheenPill CTA. No ED-safety/Coach/BC copy. Safety-tile occlusion is inside ScrollView, unaffected.
- Contrast preserved: Back/Skip inkMuted, CTA ink fill + white text on warm mesh — legible. Removed border was a divider, not contrast.
- paddingBottom unchanged (Math.max(28, insets.bottom+16)); home-indicator clearance intact.
- onPress unaffected; transparent bg doesn't change hit-testing.
- No scope/deps/data-model/Coach-prompt/navigation change. Only the footer style block changed.

**Result: PASS — safe to report done.**

---
## 2026-05-31T00:00:00Z — Workout Log gradient + card-format round (CardBloom, WorkoutEntry.focus display field, LogCard detail reformat)

**Summary:** Clean — PASS. 0 blockers, 0 warnings. The focus field is provably display-only, blooms are non-interactive and clipped, gradient ids don't collide, no safety/Coach/BC/cycle surface touched. Closes Ting's gaps: flat "This week so far" card → SVG bloom; logged-workout detail full-exercise-dump → "{N} movements · {focus}".

**Findings (all PASS):**
- lib/types.ts:362-368 — WorkoutEntry.focus?: string optional, DISPLAY-ONLY. Optional → existing stored entries valid without it; no migration.
- WorkoutScreen.tsx:776 — only consumer of e.focus is the LogCard detail string (verbatim {movements} · {focus}). No branch/estimate/decision reads it.
- WorkoutScreen.tsx:482-496 & plan.ts:97-107 — focus does NOT feed burn math (estimateExerciseBurn/estimateBurn use exercises/duration/weight); population guarded.
- coach.ts:311-314,588-627 — focus never reaches Coach prompt/context (workout line uses workoutLabel; log_workout schema has no focus param). Coach-chat entries get no focus by design.
- ED-safety copy unchanged (factual summary recap; plan-day focus strings neutral descriptors). No deficit/earn-burn/"many women"/shaming.
- Gradient ids distinct: base linear variant-suffixed (bloom-base-summary/program); radials (bloom-sage/bloom-rosehoney) in mutually-exclusive branches. No collision.
- Robustness: CardBloom absoluteFill + pointerEvents="none" (can't intercept taps); host cards overflow:hidden clip the SVG; empty exercises → "0 movements" no crash.
- Token integrity: no new palette tokens; bloom hues = profileSage/profileRose/honey; two warm-cream hexes (#F6EFE4/#FCF8F1) mockup-literal neutrals as local consts. No stray purple/orange.
- Scope: no new dependency (react-native-svg@15.12.1 already present). Coach prompt, BC branching, cycle untouched.

**Not checked:** on-device bloom legibility behind text (couldn't run app — Ting/implementer to confirm summary + program copy read cleanly over gradient); tsc/grep taken as reported.

**Result: PASS — no BLOCK, no WARN.**

---
## 2026-05-31 — Onboarding clip-fix (design) + Settings/Coach vocabulary unification (impl)

## Summary
Clean — 0 BLOCK, 0 WARN, 2 NOTE. Both ship. Orchestrator confirmed tsc clean (exit 0) for both.

## Findings
### 1. ED-safety (label strings) — PASS
- [NOTE] lib/types.ts:202-230 — New GOAL/ACTIVITY/TONE labels equal-or-safer, consistent with flat-declarative brand voice. lose_fat "Lose body fat"→"Lean out" (safer; drops explicit body-mass framing; onboarding desc carries sustainability framing). tone_up→"Tone & define", build_muscle→"Build strength", feel_better→"Feel better", maintain unchanged. bestie "Bestie"→"Knowledgeable friend" (neutral). tough_love case-only; hype unchanged. Activity labels descriptive movement levels, no shaming.
- [NOTE] No ED-safety disclaimer/clinician copy changed; only option labels. Hard-rule text (spec §6 :272, onboarding cycle micro-copy) untouched.

### 2. Coach-context drift — PASS (benign vocabulary unification)
- Only Coach-facing change: descriptive strings at coach.ts:378/:1626 (goal), :235/:379/:1401 (activity). Keys still drive all math (multipliers/BMR/macros unchanged). TONE_STYLE unchanged; TONE_LABELS not imported by coach.ts so "Knowledgeable friend" never reaches the model.
- [NOTE] Dropping "(N days/week)" parenthetical from activity labels marginally reduces Coach context specificity; immaterial (key sets multiplier; onboarding desc retains day-count).

### 3. Scope / drift — PASS
- Enum types Goal/Tone/ActivityLevel unchanged; no key renamed. No data-model change, no migration (label maps not persisted — only enum keys stored). No new dependency.
- Designer change presentation-only: title lineHeight 49→54 + paddingTop:2 + includeFontPadding:false; titleSm 41→45; uline paddingTop 10→14 + lineHeight:38 + includeFontPadding:false. uline TextInput entry/onChangeText/value unaffected.
- No cross-contamination: copy pass didn't touch OnboardingScreen.tsx (its local *_ROWS are the source-of-truth); clip-fix didn't touch lib/types.ts.

### 4. tsc — PASS (orchestrator-confirmed exit 0 for both).

## Not checked
- On-device render of clip-fix ("B"/"First name" fully visible on 430pt) — needs a device pass.
- Deferred ProgressScreen purple item not re-flagged.

**Result: PASS — safe to report done.**

---
## 2026-05-31 — Coach suggested-prompt ("OR ASK ME") block moved inside ScrollView

**Summary:** Clean — 0 blockers, 0 warnings. Scoped layout/JSX-position + style change confirmed safe. Fixes "OR ASK ME no longer scrolls with the chat" — block was a pinned sibling above the composer; now last child inside the ScrollView so it scrolls with content. Orchestrator confirmed tsc clean (exit 0).

**Findings (all NOTE):**
- CoachScreen.tsx:369-370 — showSuggestions gating UNCHANGED, fresh-kickoff-only (!booting && input empty && no user message). Not persistent.
- CoachScreen.tsx:1268-1283 — Block now LAST child inside ScrollView (after messages.map / TypingDots, before </ScrollView>). Handlers unchanged (onPress send(s), disabled sending||booting); SUGGESTED + send() untouched.
- CoachScreen.tsx:1549-1552 — suggestList: marginTop spacing.lg + paddingBottom spacing.xs only; removed paddingHorizontal (messages already insets) + paddingTop. Eyebrow/item/text untouched.
- Auto-scroll onContentSizeChange→scrollToEnd unchanged; no oscillation (showSuggestions keys off discrete state, old atBottom feedback-loop gate already removed; mounts once → one scrollToEnd → settles).
- inputRow/composer, header, KeyboardAvoidingView, message rendering untouched.
- Safety scope: Coach system prompt, tool defs/handlers, ED-safety copy, BC/calibration, data model — NOT touched. No dependency change, no new any, no networked calls.

**Not checked:** device visual/scroll behavior; SUGGESTED strings (out of diff scope, previously PASS).

**Result: PASS — safe to report done.**

---
## 2026-05-31T00:00:00Z — Remove cycle calendar month-grids from Progress screen

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Pure deletion of the F3 month-grid calendar plus dead support code and wiring; all ED-safety copy, the BC branch, and the `bleedDays` equivalence refactor check out.

## Findings
- [NOTE] wren/screens/ProgressScreen.tsx — `bleedDays = periodDays(profile).length` is equivalent to the old `new Set(...).size` (object keys unique). Pre-existing edge: `periodDays` falls back to `[lastPeriodStart]` (length 1) with no flow logs; identical to prior behavior since old `allBleed` also sourced via `periodDays`.
- [NOTE] wren/lib/cycle.ts — `PHASE_COLORS`/`phaseForDate`/`isPreviousCycle`/`isPredictedPeriodDay` still exported and used by CycleScreen; removing their import from ProgressScreen did not orphan them.
- [NOTE] wren/screens/ProgressScreen.tsx:42 — `ACCENT = "#7c3aed"` purple debt untouched (deferred, tracked separately).

ED-safety copy intact (non-BC summary lines, BC note, clinician note, neutral chart styling). BC branch intact (pred=null, no phase claim, no "day N", no chart for BC). Wiring clean (App.tsx passes only profile+updateProfile; CycleScreen.selectDateSignal optional with guard, F3 jump never fires — intended). No new deps/network/data-model/Coach changes. tsc --noEmit passed (run by implementer).

No BLOCK findings.

---
## 2026-05-31T00:00:00Z — Workout card bloom/shadow fidelity round (presentation-only)

**Summary:** Clean — 0 blockers, 0 warnings, 2 notes. Presentation-only depth pass; no safety, scope, BC, data-model, or Coach-prompt impact. Fixes Ting's "gradient is goofy and no shadows": CardBloom switched to userSpaceOnUse (onLayout-measured) + lowered opacity + fade-to-edge (kills the wedge/seam); drop shadows added to program/summary (outer wrapper for the overflow:hidden+shadow gotcha)/day/today cards + toggle pill; rest card shadow off.

**Findings (both NOTE, non-blocking):**
- [NOTE] WorkoutScreen.tsx:127 — CardBloom renders nothing until onLayout reports non-zero size → program/summary cards show flat creamTile one frame then bloom pops in. Intended fallback (flat is safe default), cosmetic. Flag so it's not mistaken for a bug. No fix needed unless the pop distracts.
- [NOTE] WorkoutScreen.tsx:2277-2308 — dcard lost overflow:"hidden". Safe: no full-bleed/absolute child on day cards, dbody is a conditional mount, rows within padding. No clip regression.

**Axis verification (all PASS):** Scope — pure presentation; onLayout measure is the only new runtime work, guarded (setSize only on size delta, can't loop); LinearGradient/BLOOM_CREAM consts removed cleanly; react-native-svg already a dep. ED-safety/legibility — no copy changed; bloom peak lowered (~0.30→0.16), shadows on outer wrapper outside card, bloom absoluteFill behind text w/ pointerEvents none → legibility equal-or-better. Tokens — all shadowColor = colors.ink (#3B2F22); bloom hues profileSage/profileRose/honey; no new tokens; no purple/orange. BC/Coach — untouched. Robustness — guard can't loop; wrappers can't intercept taps; dcardRest zeroes shadowOpacity+elevation.

**Not checked:** on-device SVG falloff, bloom flash-in timing, iOS shadow vs Android elevation (require device).

**Result: PASS — no BLOCK, no WARN.**

---
## 2026-05-31 — Coach chat-bubble drop shadow (presentation tweak)

### Summary
Clean — 0 blockers, 0 warnings. Presentation-only StyleSheet change on CoachScreen; scoped to bubble variants; no safety, scope, data-model, or contrast impact. tsc clean (exit 0, orchestrator-confirmed).

### Findings
- [NOTE] CoachScreen.tsx:1437-1467 — Only styles.bubble and styles.userBubble changed. styles.bubble gained shared soft-card shadow (colors.ink, offset {0,2}, opacity 0.12, radius 5, elevation 2); userBubble overrides to {0,3}, 0.16, elevation 3 so the dark cocoa bubble still lifts. Matches WorkoutScreen card recipe. No colors/radii/padding/text/layout/behavior touched.
- [NOTE] CoachScreen.tsx:205, 1201 — styles.bubble applied at exactly two sites (TypingDots bubble + message bubble). NOT applied to dataCard, ctaPill, suggested-prompt list, composer/inputBar, or header. No shadow leak onto non-bubble UI.
- [NOTE] CoachScreen.tsx:1455-1477 — No contrast/legibility regression: bubble fills + text colors unchanged; shadow is elevation-only.
- [NOTE] Coach system prompt, tool handlers, ED-safety/crisis copy, BC/phase-label branch, data model, imports — all outside the diff and untouched.

### Not checked
- On-device visual render of the shadow lift (iOS shadow vs Android elevation).
- ProgressScreen deferred-purple item intentionally not re-flagged.

**Result: PASS — safe to report done.**

---
## 2026-05-31 — Coach composer input bar shadow (styles.inputBar)

### Summary
Clean — 0 blockers, 0 warnings. Presentation-only shadow on the "Message your coach…" composer pill. tsc clean (exit 0, orchestrator-confirmed).

### Findings (all NOTE)
- CoachScreen.tsx styles.inputBar — 5 shadow props added (shadowColor colors.ink, offset {0,2}, opacity 0.12, radius 6, elevation 3). Same soft-card family as the chat bubbles, nudged slightly more present (radius 6 vs 5, elevation 3 vs 2) for the pinned composer. Decorative only.
- Shadow scope contained to inputBar; no overflow:"hidden" added on inputBar or inputRow so it renders unclipped; opaque creamTile fill retained. (Only overflow:hidden in file is the unrelated profile-photo circle.)
- No contrast/legibility regression: fills, border, radius, padding, camera/send buttons, TextInput, placeholder unchanged; shadow doesn't sit over text.
- Coach system prompt, tool handlers, ED-safety copy, BC/calibration, data model, dependencies — all untouched.

### Not checked
- On-device shadow weight / iOS vs Android elevation parity.
- ProgressScreen deferred-purple item intentionally not re-flagged.

**Result: PASS — safe to report done.**

---
## 2026-05-31 — KeyboardAvoidingView wrap on OnboardingScreen content stack

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. Fixes Goal-step "in your words" (and name / diet "anything else") inputs being hidden by the keyboard. Presentation/layout-wiring only; ambient full-bleed, insets, ScrollView keyboard props, and all safety/scope/BC contracts intact. tsc clean (exit 0, orchestrator-confirmed).

### Findings
- [NOTE] OnboardingScreen.tsx:944-980 — Ambient mesh full-bleed intact: ambient View (absoluteFill, pointerEvents none) remains the FIRST child of root flex:1 View, OUTSIDE/BEFORE the KeyboardAvoidingView (opens 992). Not subject to KAV padding-shrink; AmbientMesh still sized from useWindowDimensions. Mesh+grain+sheen still paint behind status bar / home indicator.
- [NOTE] OnboardingScreen.tsx:992-1555 — keyboardVerticalOffset={0} verified sound: CoachScreen uses insets.top because it's inside the inset SafeAreaView; onboarding is full-bleed outside it, so KAV top edge is at the true screen top — 0 avoids a double-offset gap.

### Confirmations (all pass)
1. Ambient mesh full-bleed — PASS (absoluteFill, first sibling, behind KAV; not constrained inside it).
2. Insets preserved inside KAV — PASS (header insets.top+10, hero insets.top+28, footer Math.max(28, insets.bottom+16); footer still transparent).
3. ScrollView keyboard props — PASS (keyboardShouldPersistTaps="handled" + keyboardDismissMode="on-drag" preserved → tapping goal row/Continue with keyboard up still registers).
4. Scope/safety — PASS (no copy/step-flow/calibration/BC-branching/touched-gating/finish()-persistence change; ED-safety infocard still on opaque creamTile; KeyboardAvoidingView+Platform added to existing react-native import, no new dep).
5. keyboardVerticalOffset=0 — PASS.
6. tsc — RECORDED clean (exit 0).
Tree: root View → ambient View (absoluteFill, first, behind) → KeyboardAvoidingView(kav flex:1, behavior ios?padding:undefined, kvo=0) wrapping [index header, ScrollView, footer] → close KAV → close root.

### Not checked
- On-device keyboard behavior iOS + Android (focused textareas/inputs clearing the keyboard) — needs device test.
- Deferred ProgressScreen purple item not re-flagged.

**Result: PASS — safe to report done.**

---
## 2026-05-31 — OnboardingScreen keyboard fix v2 (KAV removed → ScrollView automaticallyAdjustKeyboardInsets) [SUPERSEDED — see v3 below; still didn't keep growing textarea above keyboard]

### Summary
Clean — 0 BLOCK, 0 WARN, 2 notes. Removed KeyboardAvoidingView, added automaticallyAdjustKeyboardInsets to the ScrollView, container paddingBottom 10→40. Ambient mesh full-bleed + insets + ScrollView keyboard props intact; no scope/safety/dependency change; tsc clean (exit 0). NOTE: automaticallyAdjustKeyboardInsets is iOS-only. NOTE: this fix did NOT fully solve on device — field still half-covered and growing multiline text disappeared; a v3 fix follows.

**Result: PASS (audit), but functionally insufficient — superseded by v3.**

---
## 2026-05-31T00:00:00Z — Food tab shadow pass (presentation-only card depth on 8 cards)

**Summary:** Clean — PASS, 0 blockers, 0 warnings. Applies the Workout depth language to Food for app-wide consistency (Ting: "add shadows to food tab").

**Verified clean:**
- Scope/drift: diff surface is shadow style props only (shadowColor/Offset/Opacity/Radius/elevation). No food-logging, calorie/macro math, photo/barcode, storage, Coach-prompt, type, or dependency change. No new imports.
- ED-safety + legibility: no calorie/macro/deficit/food copy touched; shadows render outside card bounds, legibility unaffected; BMR-floored budget logic + "+Exercise shown-but-not-counted" framing untouched.
- Token integrity: shadowColor: colors.ink (#3B2F22 cocoa) on all 8 cards; no #000 shadow remains; no new palette tokens.
- Robustness: only overflow:"hidden" is macroBarTrack (sub-element), not a shadowed card → no iOS shadow-clip; no square-corner bleed; no wrappers added.
- Tiers: calCard hero {0,8}/.18/16/elev6; captureCard+groupCard mid {0,6}/.16/14/5; entryCard+waterCard+previewCard+draftCard list {0,3}/.10/8/3; sheetCard {0,2}/.08/12/elev4 (normalized off #000).

**Notes (pre-existing, not this pass):** literal hex (#eee/#fff/#666/#1a1a1a) in photoThumb/draftCard/draft text; confirmDelete TODO(implementer) scaffold — both unrelated, not regressions.

**Result: PASS — no BLOCK, no WARN.**

---
## 2026-05-31 — Onboarding keyboard fix v3: react-native-keyboard-aware-scroll-view dependency + scroll-container swap

### Summary
0 BLOCK, 2 WARN, 3 NOTE — both WARNs resolved by orchestrator post-audit. Replaces the failed native attempts (KAV, automaticallyAdjustKeyboardInsets). Dependency was Ting-approved. tsc clean (exit 0).

### Findings
- [WARN→RESOLVED] package.json — react-native-keyboard-aware-scroll-view added; pure-JS lib (only peer react-native-iphone-x-helper, also pure-JS; no native module/config-plugin/pod-install) → should be Expo Go / SDK 54 safe. Auditor couldn't inspect node_modules (read-only env) so install tree not disk-verified; lib is old/unmaintained and `^` floated. RESOLVED: orchestrator pinned the version to exact `0.9.5` (removed caret). STILL NEEDS: Ting device-verify on Expo Go v54 (app boots, textareas clear keyboard, no missing-native-module/unresolved-peer Metro warnings) — Expo Go is the only runtime, no dev build.
- [WARN→RESOLVED] OnboardingScreen.tsx container-style comment still described the removed automaticallyAdjustKeyboardInsets. RESOLVED: orchestrator rewrote the comment to reference KeyboardAwareScrollView/extraScrollHeight. paddingBottom:40 retained (still correct).
- [NOTE] OnboardingScreen.tsx — Swap correct: KeyboardAwareScrollView imported from the lib; ScrollView removed from react-native import (remaining "ScrollView" hits are comments); no ScrollView ref/method used. Props: contentContainerStyle={styles.container}, keyboardShouldPersistTaps="handled", keyboardDismissMode="on-drag", enableOnAndroid, extraScrollHeight={28}, enableResetScrollToCoords={false}. automaticallyAdjustKeyboardInsets removed.
- [NOTE] Structural integrity intact: tree root flex → ambient (absoluteFillObject + pointerEvents none, FIRST sibling, AmbientMesh+grain+ScreenSheen, full-bleed) → index header → KeyboardAwareScrollView → pinned transparent footer. Mesh NOT moved inside scroll. Insets: header top+10, hero top+28, footer Math.max(28, bottom+16). keyboardShouldPersistTaps preserved.
- [NOTE] No collateral drift: Expo ^54.0.34 not bumped, RN 0.81.5 / React 19.1.0 unchanged, only the one dep added. No copy/step-flow/calibration/BC-branch/touched-gating/finish-persistence/Coach-prompt/data-model change. ED-safety infocard still on opaque creamTile. No new network/auth/telemetry/any.

### Mechanism (why v3 works where v1/v2 failed)
KAV padding only shrank the container (lifted footer, not field). automaticallyAdjustKeyboardInsets scrolled once on focus but didn't re-track the caret as the multiline auto-grew. KeyboardAwareScrollView listens for focus AND focused-input content-size growth, re-measures the caret, and scrolls to keep it above the keyboard (+extraScrollHeight) line by line.

### Not checked
- node_modules transitive tree / on-device Expo Go behavior (the actual fix) — REQUIRES Ting's device. This is the open validation item.
- Deferred ProgressScreen purple item not re-flagged.

**Result: PASS (0 BLOCK) — both WARNs resolved; one open item = Ting's on-device Expo Go verification.**

---
## 2026-05-31T00:00:00Z — Food "Tell Coach" frozen-page navigation fix (FoodScreen.tsx)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Fix matches report and is correctly scoped; no safety, BC, or Coach-prompt code touched.

## Findings
- [NOTE] FoodScreen.tsx:104-110 — Cleanup useEffect correct: deps [] (clears only on unmount), cleanup references/clears tellCoachTimer.current. Prevents onOpenCoach firing after unmount and prevents stacked timers (line 428 clears in-flight timer before re-arm on rapid taps).
- [NOTE] FoodScreen.tsx:426-433 — Deferred-navigation UX nuance (low severity): during the 300ms window, if the user taps a different tab, the timer still fires and switches to Coach ~300ms later. Sub-second race, worst case one unexpected tab switch (not a freeze). Accepted tradeoff. Optional hardening: capture active tab at schedule time, no-op if user already navigated.
- [NOTE] FoodScreen.tsx:290,295 — Pre-existing, out-of-scope: reopenAddLater()/openScanner() use bare setTimeout (350ms) without ref/cleanup; local-state setters on always-mounted screen so risk near-nil. Not introduced here.

Verification: robust ref/cleanup; scope limited to import + ref/cleanup + tellCoach; ED-safety/BC/Coach-prompt UNTOUCHED (BMR comment :125, BC branch :140, macros disclaimer :974-977 intact; no Anthropic code in file); SDK 54 fine (setTimeout/clearTimeout, no iOS-only Modal.onDismiss). tsc --noEmit passed.

## Not checked
- Runtime timing (300ms vs actual slide-out) on device, both platforms — confirm 300ms >= real slide-out.
- Manual repro: Add food -> Tell Coach -> return to Food, confirm not frozen.

No BLOCK findings.

---
## 2026-05-31 — Onboarding keyboard fix attempt #5: REMOVE react-native-keyboard-aware-scroll-view + custom built-in-RN keyboard avoidance (OnboardingScreen.tsx)

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. v3 library failed to engage on RN 0.81/Expo Go (Ting: field doesn't move, keyboard covers it). Library REMOVED; replaced with custom built-in-RN avoidance. Listener hygiene sound; structural/safety/BC invariants intact; tsc clean (exit 0).

### Findings
1. DEPENDENCY REMOVAL — CLEAN. react-native-keyboard-aware-scroll-view gone from package.json (and node_modules/lockfile via npm uninstall). ScrollView re-imported from react-native (OnboardingScreen.tsx:7), container with ref (:1076→:1583). No KeyboardAwareScrollView import/JSX; remaining hits are prior-attempt comments (:556-557, :1046-1050). expo ^54.0.34 unchanged, no SDK bump.
2. LISTENER HYGIENE — CLEAN. useEffect [] (:565-578) subscribes keyboardWillShow/Did/WillHide/DidHide, all removed via subs.forEach(s=>s.remove()) cleanup (:577). Keyboard imported (:15). No loop: [kbHeight] effect (:583-587) scrollToEnd doesn't mutate kbHeight; onContentSizeChange (:600-602) fires on growth not scroll.
3. STRUCTURAL INTEGRITY — CLEAN. Tree: root flex (:1001) → ambient absoluteFill FIRST, pointerEvents none (:1023) → index header (:1063) → ScrollView ref (:1076) → pinned footer (:1588). Mesh full-bleed, NOT inside ScrollView. Insets intact (header top+10, hero top+28, footer Math.max(28,bottom+16), transparent). keyboardShouldPersistTaps="handled" + keyboardDismissMode="on-drag" preserved. contentContainerStyle=[container,{paddingBottom:40+kbHeight}].
4. SCOPE/SAFETY — CLEAN. No copy/step-flow/calibration/BC-branch/touched-gating/finish()-persistence/Coach-prompt/data-model change. BC branch intact (:920-921); BC switch copy (:1386-1391); ED-safety infocard on opaque creamTile (:1324/:2133-2163), copy verbatim.
5. ROBUSTNESS — CLEAN. scrollToEnd uses optional chaining (:585,:595,:601). onContentSizeChange only on the two textareas (Goal :1185-1195; diet :1471-1481), each on its own step — no cross-step scroll.
6. TSC — orchestrator confirmed clean (exit 0).

### Mechanism
Textareas are the LAST element on their step. kbHeight (from Keyboard events) added to contentContainer paddingBottom creates a keyboard's worth of scroll slack; scrollToEnd on focus + on content-size growth pins the textarea bottom (caret) just above the keyboard line and follows it line-by-line. Pure built-in RN — no library compat risk.

### Notes
- [NOTE] :600-602 scrollToEnd on every content-size change (intended caret-follow; cheap).
- [NOTE] :568-569 endCoordinates?.height ?? 0 degrades to no-lift, not crash.
- [NOTE] Runtime behavior device-dependent; attempt #5 exists because prior approaches passed review but failed at runtime — Ting must confirm on-device (iOS Will* + Android Did*).

### Not checked
- Runtime keyboard lift on physical device (static audit only).
- Deferred ProgressScreen purple item not re-flagged.

**Result: PASS — 0 BLOCK. Open item = Ting's on-device confirmation.**

---
## 2026-05-31T00:00:00Z — Food tab freeze fix (attempt 2: conditional Modal mount + active gate)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Fix correctly implemented; ED-safety, BC branching, Coach safety-prompt untouched. (Attempt 1, the 300ms tellCoachTimer defer, failed and was reverted.)

## Findings
- [NOTE] FoodScreen.tsx:234-245 — Blur force-close discards in-progress draft state (half-typed entry, pending photo, half-built meal, pendingHit). Acceptable: "Tell Coach" intentionally abandons the sheet; only transient state clears, nothing persisted.
- [NOTE] FoodScreen.tsx:236-244 — Reset list complete for modal visibility (addOpen, editEntry, scanning, photoReview, mealBuilderOpen all reset; the other 4 are in-modal sub-states). No modal-visibility var missed.
- [NOTE] FoodScreen.tsx:310-316,411-435 — Existing 350ms intra-Food modal-handoff setTimeouts remain; unrelated to the reverted hack.

## Verification
1. JSX balance PASS — 5 `{active && (` gates matched and balanced; all Modal pairs intact; no visible/onRequestClose lost.
2. Blur deps PASS — useEffect [active] with `if (active) return`; fires only true→false.
3. active default-true PASS — default at FoodScreen; App.tsx:232 passes active={tab === "food"}.
4. Data loss — acceptable.
5. Timer removed PASS — no tellCoachTimer/SHEET_SLIDE_OUT_MS anywhere; tellCoach reverted to simple form; imports valid.
6. ED-safety/BC/Coach UNTOUCHED — macros disclaimer intact (:990-991), BC phase-suppression intact (:142), no Anthropic/system-prompt code in diff.
7. SDK 54 PASS — conditional mount is plain React; correct RN pattern for stranded RCTModalHostView.

## Not checked
- tsc not run by auditor (implementer reports exit 0).
- On-device repro: Food → Add food → Tell Coach → back → confirm touches; repeat switching away mid-modal for the other 4 modals.

No BLOCK findings.

---
## 2026-05-31 — Workout-tab: Sun–Sat display + multi-week date strip + forward-clip generation

# Audit report — Workout-tab Sun–Sat display + multi-week strip + forward-clip

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Scoped to WorkoutScreen.tsx only (Ting chose Workout-only Sun–Sat); shared lib/plan.ts/cycle.ts/coach.ts/types.ts untouched. Off-by-one mapping correct, done days never overwritten, rest-conversion copy ED-safe, BC/unknown gating holds. tsc clean (exit 0, orchestrator-confirmed).

## Findings (NOTE)
- [NOTE] WorkoutScreen.tsx:1120-1130 — Plan view: tapping a strip cell OUTSIDE the current plan week is a silent no-op (planDayForISO null → no selection change, no visible feedback). Intentional/documented; possible future UX polish (switch to Log view or scroll card list). Surfaced to Ting.
- [NOTE] WorkoutScreen.tsx:1012-1014 — Strip "trained" dot ORs plan-day dayLogged with any logged workout on that date. Correct/consistent; neutral (dot, no count/streak).
- [NOTE] WorkoutScreen.tsx:285 — ~70-cell non-virtualized horizontal ScrollView via useMemo. Fine at this size; revisit if STRIP_WEEKS range grows.

### Checklist (all PASS)
1. SHARED MODEL UNTOUCHED — lib/plan.ts/cycle.ts/coach.ts/types.ts unchanged; coach.ts:1730 still startDate: mondayOf; storage stays Monday-anchored, no migration risk for Coach/Progress.
2. OFF-BY-ONE — planDayForISO matches dateForWeekday(startDate, weekday)===iso (no array-position re-derivation); same date → same PlanDay via strip and stored plan; out-of-week → null neutral. Sun-first DISPLAY can't misroute a date.
3. FORWARD-CLIP — loggedEntryId guard precedes conversion (+ dateISO>=today early return); done days never overwritten. Only current week clipped (generatePlan); future weeks (buildNextWeek/acceptReview) full. Past-untrained→rest copy neutral ("Rest day" / "Recovery — rest is part of the plan."), no shame/streak. Disabled day chips dimmed-neutral.
4. CYCLE/BC GATING — eyebrow hasPhase gating unchanged; Day N only when dayOfCycle!=null (null on BC); strip cells carry no phase/day claim; Log date line shows real "On birth control" label, no fabricated day.
5. SCOPE/DRIFT — no calorie/macro math change, no WeekPlan/PlanDay shape change (only existing fields), no Coach-prompt change, no new dependency, SDK 54 respected, PlanSetup.workoutDays/daysPerWeek wiring intact.
6. ROBUSTNESS — auto-scroll can't loop (fixed centerOffset from constant todayIndex, no state set); planDayForISO/forwardClip handle null/empty; centerOffset clamps via Math.max(0,…).
7. TSC — orchestrator-confirmed clean (exit 0).

## Not checked
- Device-level visual render of strip/dimmed chips + auto-center landing.
- Runtime timing of setTimeout(0) auto-open + auto-center under six-screen mount.

No BLOCK findings.

## 2026-05-31T00:00:00Z — Tab-bar emoji icons → custom SVG line-art (TabIcons.tsx + App.tsx)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. Navigation wiring intact, all 5 tab keys correctly mapped, no scope creep, no safety/BC/Coach code touched.

### Findings
- **[NOTE]** `components/TabIcons.tsx:1` — `import type { JSX } from "react"` relies on React 19's exported `JSX` namespace; resolves under React 19.1.0 + `@types/react ~19.1.10`. Type-only, erased at runtime. Implementer should confirm tsc is clean (auditor cannot run tsc).
- **[NOTE]** `App.tsx:49` — Icons always rendered `color={colors.ink}`, using per-icon opacity (0.5/1.0) to signal active state rather than a color split. Matches documented mockup intent; visual fidelity needs a device render.

### Verification performed
- Navigation/handlers intact: `onChange(t.key)` → `setTab` unchanged; no signal/state/updateProfile logic touched.
- All 5 tab keys (coach/food/cycle/workout/progress) map to matching icon keys; `tabIcons` defines exactly those 5 via `Record<TabIconKey,…>`. `settings` correctly excluded.
- `tabIcons[t.icon]` total over `TabIconKey`; no undefined-component path.
- Imports/types OK: `colors.ink` exists (theme.ts:33); `react-native-svg@15.12.1` declared (package.json:20, SDK 54-pinned); no undeclared imports.
- No stale references: only consumers of `tabIcons`/`TabIconKey`/`tabIcon` are TabIcons.tsx and App.tsx; no leftover emoji `icon` strings or old `fontSize:22` style.
- No scope creep: tab-bar only; no networked deps/fetch/auth/push/telemetry; no data-model/storage change; react-native-svg pre-existing.
- Safety surfaces untouched: no Coach prompt, injection-delimiter, ED-copy, birth-control, or Anthropic call-site changes.

### Not checked
- Visual/mockup pixel-fidelity of the 5 SVG icons (esp. CYCLE filled-drop active, FOOD bowl arc) — needs device/simulator.
- tsc clean compile — types look correct but should be confirmed.

No BLOCK findings.

---
## 2026-05-31T00:00:00Z — Progress-tab redesign (designer → implementer → auditor)

**Result: PASS — 0 BLOCK, 2 WARN, 3 NOTE.** Files: screens/ProgressScreen.tsx (rebuilt + wired), components/ProgressViz.tsx (new svg viz), lib/progress.ts (new pure aggregation), lib/theme.ts (6 new FLAG-commented tokens), App.tsx (onOpenSettings + onJumpToCycleDay wiring).

ED-safety checklist — all PASS:
- Weight never daily: bucketWeights (progress.ts:226) averages weekly/monthly only; AreaSpark hard-returns empty <2 pts (ProgressViz.tsx:103); single bucket = plain text not a 1-pt trend (ProgressScreen.tsx:813-819); caption "never day to day" present & legible.
- Body/photo "guide not verdict": captions preserved verbatim (ProgressScreen.tsx:767, :1441); bfDeltaLine neutral, no arrows/color (:600-613); low-BF%<14% clinician note renders (:1502).
- BC + no day-N leak: onBC forces currentCycleDay/pred null (:246-247); chart gated !onBC (:254); cycleRingPoints returns [] on BC (progress.ts:317); week-view day gated off===0 && !onBC && dayOfCycle!=null && phase!=="unknown" (:891); bleed-framing + clinician note + "likely around" hedge survive.
- No fabricated data: seeded()/Math.random removed (grep clean); honest empty states on every card; unlogged mood days render null neutral dots.
- Date-nav no future: off clamped Math.max(0,…) (:660); › disabled at off===0 with double-guard (:1158).
- Purple: zero hardcoded purple in all 3 new files (grep clean); 6 new tokens FLAG-commented (theme.ts:176-201); safety caption cocoaSoft-on-progressCard, contrast fine.

Scope/drift — all PASS: tab bar untouched (App.tsx slot pattern); lib/coach.ts untouched; no SDK bump (expo ^54.0.34); new deps react-native-svg 15.12.1 + expo-image-picker ~17.0.11 already declared, SDK54-safe.

Fidelity NOTE: week labels Mon-anchored (M T W T F S S) vs mockup Sun-anchored — correct, matches app WEEKDAY_LABELS convention, internally consistent.

WARN-1: ProgressScreen has 6 Modals but no `active` prop (unlike FoodScreen). Per known RN display:none Modal-stranding failure mode (Food fix 2026-05-31), an open Modal during a programmatic tab switch could strand the tab. Risk narrow (Modals open only via in-tab taps). Fix direction: pass active={tab==="progress"}, gate Modal mount on it. Implementer to verify device path.
WARN-2: per-view aggregations (bucketWeights, sessionsInWindow, allTimeSessionsByMonth) recomputed each render without useMemo. Pure + cheap; perf hygiene only, not correctness.

NOTE-1: bannerWeeks floor only affects has-photo branch (harmless). NOTE-2: cycle-ring numeral = cycle length not "day N" (BC-safe; flag so future copy pass doesn't mislabel). NOTE-3: expo-image-picker uses SDK54 mediaTypes:"images" string API correctly.

Not checked: device contrast/render; tsc clean (code reads type-safe, no new any); whether open-Modal+tab-switch path exists (needs device test).

---
## 2026-05-31T00:00:00Z — Cycle tab cohesion pass (header rebuild + PhaseRing gradients + page-wide color softening)

**Result: PASS — 0 BLOCK, 0 WARN, 3 NOTE.** Ting asks: cohesive header, gradient ring, softer page colors (incl. calendar), rounded calendar edges, shadows.

Scope: screens/CycleScreen.tsx, components/PhaseRing.tsx, one-line App.tsx prop (onOpenSettings → setTab("settings")). Presentation-only; no cycle math/storage/Coach-prompt/data-model change.

1. ED-safety copy — PASS. Disclaimer (CycleScreen.tsx:769-772 "…Not medical advice."), BC card (:373-376), no-data card (:384-387), 4 phase-guidance lines (:83-92) all verbatim at colors.ink on cream/vapor — never on a softened phase fill. No deficit/earn/burn/"many women"/feelings-override in user-facing copy.
2. BC/no-data branching — PASS. Gate intact (showCycleUI = !onBirthControl && starts.length>0, :396). New header + Log-today pill render OUTSIDE the gate for all users. Eyebrow is calendar month/year ("CYCLE · {MONTH} {YEAR}"), no cycle-day claim. cycle.ts BC contract (dayOfCycle:null) unchanged. No cycle-day leak.
3. Softened-cell legibility — PASS. Washed cells carry cocoa numerals; today = solid cocoa tile + white (overrides wash); logged = solid ember + white. Lightest wash (ovulatory #BDD0D6 @0.32 over cream ≈ #EFF1F0) + full cocoa numerals = high contrast. Alpha on background fill only, never cell opacity → text never fades.
4. Scope/drift — PASS. withAlpha() pure string math; App.tsx prop is a pure tab switch, no data through it.
5. Token integrity — PASS. Softened colors = alpha on existing phase tokens (theme.phaseColors); shadowColor = colors.ink on all 3 cards; no new/off-brand hues, no purple/orange.
6. Robustness — PASS. Gradient ids unique per phase (ring-{phase}); shadowed cards (next7Card/statusCard/patternCard) have no overflow:hidden ancestor (only inner strip clips its own cells); grid dropped overflow:hidden consistent w/ rounded cells. Relocated Log-today pill reachable (same openEditor(today)).

NOTES: (a) device-render contrast of 0.19–0.32 washes assessed analytically — recommend device eyeball; (b) tsc exit-0 reported by implementer; (c) PHASE_GUIDANCE flat-declarative voice treated as previously Ting-approved (2026-05-29).

---
## 2026-05-31T00:00:00Z — ProgressScreen WARN-1 follow-up: gate Modals on `active` (FoodScreen freeze-fix pattern)

## Summary
PASS — clean, zero warnings, zero blockers. The fix correctly mirrors the FoodScreen pattern and is regression-free.

## Findings
- [PASS] Pattern match — ProgressScreen.tsx:347 (`active?: boolean = true`), :653-662 (force-close effect), :1209-1694 (`{active && (<>…</>)}` wrapping all 7 Modals) mirror FoodScreen.tsx:88/234-245/998+. App.tsx:264 `active={tab === "progress"}` matches FoodScreen mount at :239. Only difference: a fragment of 7 Modals vs FoodScreen's single Modal — same unmount semantics, not a defect.
- [PASS] Force-close exhaustive — All 7 modal-open vars reset on active→false: viewWeek, detailEntry, newOpen, compareOpen, compDetailEntry, scanSheetOpen, weighOpen — 1:1 with the 7 gated Modals. Transient vars (saving flags, compareWithId, drafts) correctly NOT reset.
- [PASS] No new failure mode — deps [active] correct; no self-retrigger/infinite loop; no visible-while-unmounted race; no lost handler; default true keeps any prop-less caller functional.
- [PASS] Regression check — Safety surfaces unchanged: weight averaged-only w/ "never day to day" caption; BC/no-day-N gating; neutral body-comp delta + low-BF% clinician line; honest empty states; zero purple. Legacy :45 purple debt not present in current file.
- [PASS] tsc — `active?: boolean = true` sound; no any/cast; implementer-reported exit 0 consistent.

## Not checked
- Runtime device verification of the freeze-fix (bug only reproduces with a real native modal host). Implementer should manually verify: open each of 7 sheets, switch tabs while open, return, confirm taps register and no stale sheet re-shows.

No BLOCK findings.

---
## 2026-05-31T00:00:00Z — Cycle tab seamless-ring + eyebrow round

**Result: PASS — 0 BLOCK, 0 WARN, 2 NOTE.** Ting: ring "not seamless" (blobs) + eyebrow "cycle cycle" redundancy.

1. Eyebrow — ED-safe gate PASS. showCycleUI = !onBirthControl && starts.length>0 (CycleScreen:403); eyebrow phase arg = showCycleUI ? PHASE_TITLE[phase.phase] ?? null : null (:470). BC + no-data/unknown → "MONTH YEAR" only (no phase/day/"cycle"); real-phase → "{PHASE} · MONTH YEAR" from the SAME currentPhase the ring uses (no new math, label only — never a day number). PHASE_TITLE["unknown"]=undefined→??null double-safe.
2. Ring presentation-only PASS. 120 butt-capped segments (no round caps → blobs gone), each a circular RGB-interpolation of the 4 existing phaseColors anchored at arc midpoints, wrapping luteal→menstrual; overlap covers AA gaps; opacity 0.5. lerpHex pure (no NaN/crash). Proportions {5,8,3,12} unchanged (match cycle.ts phaseName). Dot/disc/stat-stack untouched. No new hue; legend still matches.
3. Scope/tokens PASS — no Coach/storage/dep/data-model change; colors all from phaseColors/colors.*; no purple/orange.
4. Disclaimer/BC/no-data copy untouched + legible (colors.ink).

NOTES (stale comments only, no impact): PhaseRing.tsx:6-8 "Step 2/PREVIEW" doc; CycleScreen.tsx:457-459 old-eyebrow comment.

## 2026-05-31T00:00:00Z — TabIcons optical-normalization + stroke/size refinement (presentation-only, designer)

### Summary
Clean — 0 blockers, 0 warnings. Presentation-only SVG change to `wren/components/TabIcons.tsx`; API/exports intact, App.tsx consumer unaffected.

### Findings
- **[NOTE]** `wren/components/TabIcons.tsx:28` — `SIZE` default dropped 24→20. App.tsx:49 renders icons without passing `size`, so live render is now ~17% smaller inside the 24×24 wrapper. Intended visual effect; no clipping risk (content bounded to ~14 units in a 24 viewBox).
- **[NOTE]** `wren/components/TabIcons.tsx:147` — Progress bar top uses `b.top + 2.5` while dot cap sits at `b.top` (r=1.1); geometry self-consistent, cosmetic only.

Verified intact:
- API `{ active, color, size? }` unchanged; App.tsx:49 supplies both required props, omits optional `size`. Compiles.
- Exports `tabIcons`, `TabIconKey`, all five icon components present; App.tsx imports the two it uses, both still exported.
- All imports used; no new imports/deps.
- Active-fill/opacity preserved: Cycle fills when active, other four stroke-only, inactive opacity 0.5, active stroke 1.7 / inactive 1.4.
- Scope: no network/auth/payment/telemetry/push; no new files.
- Safety surface: no Coach prompt, no ED/calorie/birth-control/cycle-math logic, no storage, no Anthropic call. §1–§6 untouched.

### Not checked
- Device visual rendering / optical balance at the new 20px footprint — requires device pass.

No BLOCK findings.

---
## 2026-05-31 — Workout plan week: Monday-anchored → rolling 7-day window (today-anchored)

**Result: PASS — 0 BLOCK, 0 WARN, 3 NOTE.** Ting: "if I set up a plan today it should set my days for the week ahead." Replaces the prior forward-clip approach. Files: lib/plan.ts, lib/coach.ts, screens/WorkoutScreen.tsx. Sun–Sat scrollable strip (display) kept; only the plan's date anchor changed. Full-repo tsc clean (exit 0, orchestrator-confirmed; earlier transient App.tsx topInsetBand error was stale/not present).

### Changes
- lib/plan.ts dateForWeekday(startISO, wd): offset = ((wdIndex - startWdIndex + 7) % 7) relative to startISO's own weekday (was hardcoded mon=0..sun=6). BACKWARD-COMPATIBLE: Monday anchor → byte-identical mon=0..sun=6. planDayForDate already window-based, unchanged.
- lib/coach.ts: GenerateWeekOptions += optional startDate?; fresh plan startDate = today (was mondayOf); rollover (buildNextWeek) = prev startDate +7. mondayOf import removed (function kept in plan.ts). No SYSTEM_PROMPT/ED_SAFETY_RULES change.
- WorkoutScreen.tsx: weekIsOver → today > addDays(startDate,6) || isWeekComplete; forwardClipCurrentWeek + day-picker gray-out/disable + dead chipDisabled styles REMOVED; Sun–Sat scrollable strip + planDayForISO kept.

### Verification (auditor)
1. BACKWARD-COMPAT PASS — Monday anchor identical; round-trip dateForWeekday(start, weekdayKey(d))===d for ANY anchor (bijection); old stored Monday plans resolve identically, no migration needed.
2. ROLLING CORRECTNESS PASS — Thu setup Mon/Wed/Fri → workouts Fri(+1)/next-Mon(+4)/next-Wed(+6), full set forward, none on passed days; rollover prev+7 repeats with no drift; weekIsOver window check can't strand.
3. COACH SYNC PASS — planDayForDate/targetForDate/weekReviewSummary/planContext/generateWeekPlan all anchor-agnostic; no Monday assumption; SYSTEM_PROMPT + ED rules unchanged; mondayOf still exists in plan.ts for progress.
4. SCOPE/SAFETY PASS — no calorie/macro math change (BMR+1200 floor intact); only GenerateWeekOptions widened (optional); no WeekPlan/PlanDay/Profile/DayLog shape change; no new dep; SDK 54; BC/unknown gating preserved (no fabricated Day N); removed clip/gray-out introduced no shame/streak framing.
5. ROBUSTNESS PASS — dateForWeekday total bijection; no off-by-one vs Sun–Sat strip (matched by real ISO date); rollover can't strand.

### Notes (cosmetic)
- WorkoutScreen.tsx:213 weekdayDateNum param still named startMonISO (now any anchor) — misleading name, rename to startISO suggested.
- lib/patterns.ts:402,455 dateForWeekday(monISO,...) in runPatternAssertions — monISO genuinely Monday, assertions still valid.
- lib/coach.ts:36-38 model constants pre-existing, not introduced here.

### Not checked
- On-device render of strip/day cards (verify Thursday plan shows training on Fri/next-Mon/next-Wed, strip centered on today).

No BLOCK findings.

---
## 2026-05-31T00:00:00Z — Cycle: "+ Log today" pill moved into header as round cocoa "+" (design-pipeline step 3)

**Summary:** Clean — PASS, 0 blockers, 0 warnings. Placement-only change in screens/CycleScreen.tsx header region (Ting picked: header, next to avatar).

**Findings (all NOTE):**
- CycleScreen.tsx:480-488 — new addBtn wires onPress={() => openEditor(today)}, same handler as the removed pill; openEditor is read-only setup + iso>today guard, no logic change.
- :479-509 — "+" sits in headerActions inside headerRow, rendered BEFORE the showCycleUI gate (:512) → reachable for BC + no-data users. Log-today entry preserved for all.
- :991-1006 — addBtn/addBtnPlus use existing colors.ink fill + colors.cream glyph (identical to avatar). No new token, no stray hex.
- ED-safety untouched: disclaimer (:785-788), BC branch (:375-385), no-data branch (:386-397), phase-label gating, editorEyebrow cycle-day logic all unchanged. "+" is a control (a11y "Log today"), not copy; no fabricated phase/day.
- Scope: no new import/dep/fetch/storage/Coach-context change; persist/saveEditor/clearDay route through existing updateProfile. Old pill + its 3 styles removed cleanly.

**Verdict: PASS — no BLOCK, no WARN.**

## 2026-06-01T00:00:00Z — TabIcons.tsx even-ness coordinate fix (presentation-only)

### Summary
Clean — 0 blockers, 0 warnings. Presentation-only SVG coordinate change; API, exports, and consumer all intact.

### Findings
No findings. Verification:
- API intact — `TabIconProps ({active,color,size?})` unchanged; all five components same signature.
- Exports/lookup intact — `TabIconKey` + `tabIcons` record unchanged; five keys mapped.
- Consumer unaffected — App.tsx imports `{tabIcons, type TabIconKey}`, TABS references same five keys, renders `tabIcons[t.icon]` with `active`/`color`; no `size` passed so SIZE=20 default applies.
- Active-fill preserved: Cycle drop `fill={active?color:"none"}`; Progress dots `fill={active?color:"none"}` (old `active?0:sw*0.7` logic dropped → constant r=1.15 + conditional fill, hollow-when-inactive preserved); Coach/Food/Workout stroke-only both states.
- Opacity/stroke preserved — iconOpacity 1:0.5, strokeWidth 1.7:1.4, SIZE=20.
- Imports — Svg/Path/Line/Circle/Rect all used; no dead/missing.
- Scope creep — none. Single file, no new deps/network/auth/telemetry.
- Safety surfaces — none in this file (no Coach prompt, ED copy, BC branch, data model, API key, cycle math). Cycle icon is a decorative glyph, unrelated to cycle-phase/BC logic.

### Not checked
- Optical even-ness of rendered icons — orchestrator confirmed via qlmanage PNG render before applying coordinates.
- tsc — implementer/designer reported clean; diff is coordinate literals + `{x,top}` number array, no type risk.

No BLOCK findings.

## 2026-06-01T00:00:00Z — TabIcons SIZE constant 20 → 23 (designer, presentation-only)

### Summary
Clean — 0 blockers, 0 warnings. Trivial presentation-only constant bump; no clipping, API/consumer intact, no scope or safety surface touched.

### Findings
- **[NOTE]** `wren/components/TabIcons.tsx:28` — `SIZE` now 23, rendered into App.tsx's 24×24 wrapper. 23 ≤ 24 with center justification → no clipping; ~0.5px breathing room per side.
- **[NOTE]** `size` stays optional defaulting to `SIZE`; App.tsx invokes `<Icon active color />` without `size`, so the default carries the bump. `viewBox="0 0 24 24"` unchanged → per-icon coords, y6.5–17.5 normalization, strokes (1.4/1.7), opacity logic, Cycle fill-when-active all intact.

Verified intact: exports/API, consumer wiring, no scope creep, no safety surfaces (no Coach prompt, Anthropic call, ED copy, BC branch, data model, storage).

### Not checked
- On-device visual at 23px — geometric no-clip guarantee holds; aesthetic look unverifiable from source.

No BLOCK findings.

---
## 2026-06-01T00:00:00Z — Food tab future-day logging (strip range + scroll target + dateLabel "Tomorrow")

**Summary:** PASS with 2 WARN, 2 NOTE. No BLOCK. Correctly scoped (strip range + scroll target + dateLabel); no throw on future ISOs; Coach context + prompt untouched. One ED-safety surface inherent to the Ting-requested feature, flagged.

- [WARN] FoodScreen.tsx:879-947 — Future MEAL PRE-LOGGING (~4 wks) is the new capability. A future day with 0 entries shows the FULL target as "Remaining" (ring) + Goal/−Food 0/Remaining=full. Benign per-day, but 28 forward empty days each showing "Remaining=full budget" + empty list to fill can scaffold rigid restriction planning. Ting-requested → NOT a BLOCK. No deficit/earn/burn/weight-loss framing; BMR floor intact; burn never lowers target. Safer: (a) on 0-entry future days suppress the "Remaining=full" ring, show neutral planning affordance; (b) tighten 28-day horizon or show ring only once an entry exists; (c) keep Coach walled off (currently true).
- [WARN] FoodScreen.tsx:949-955 — "FOOD TODAY" header + "Nothing logged for this day" are hard-coded; on a future selected day "TODAY" is factually wrong + mildly reinforces a fill-the-quota read. Make header reflect selDate when selDate!==today. Presentation-only.
- [NOTE] FoodScreen.tsx:119-128,851-859 — Scope correct: logDays oldest→today→future (70 cells), todayIndex=41, scroll x=max(0,41*48−120) centers today, selDate defaults today. No data-model/storage/target-math/Coach change.
- [NOTE] FoodScreen.tsx:75-85 — dateLabel "Tomorrow" symmetric to "Yesterday", correct.

**Future-date safety PASS:** targetForDate→planDayForDate null outside plan week→base target; consumed/entries/water keyed lookups ??[]/??0; phaseForDate projects forward. No NaN/throw.
**Coach drift PASS:** Coach reads food only for todayISO (coach.ts:283-284), never future; future pre-logs can't enter Coach context. No prompt change. BC branch untouched.
**Not checked:** tsc (the 2 WorkoutScreen.tsx PlanDayKind/PlanExercise errors are a separate parallel edit, NOT this Food change); on-device scroll; full Coach prompt (no Coach file touched).

**Verdict: PASS — no BLOCK.**

---
## 2026-06-01 — WorkoutScreen: (A) drop WeekDateStrip from Plan view, (B) per-day plan edit sheet

**Result: PASS — 0 BLOCK, 0 WARN, 3 NOTE.** Wipe-guard + done-day safety correct; no history loss, no log desync, no scope/ED/cycle drift. Full-repo tsc clean (exit 0, orchestrator-confirmed; the "2 WorkoutScreen PlanDayKind/PlanExercise errors" the Food entry above mentions were this change's transient mid-edit, now fixed). All 6 new edit-sheet style keys confirmed present.

### Findings
PART B Persistence/wipe-guard (CRITICAL) — PASS. saveDayEdit (WorkoutScreen.tsx:724-793) reads p.plan.current on the LATEST profile; replaces ONLY the edited weekday (days.map weekday===wd?next:d), spreads ...p.plan! (preserves setup+history) + ...p (all non-plan fields). Early returns on !cur/missing prev. Rebuilt PlanDay shape-valid: rest drops training fields; strength always sections:[{name:"Workout",exercises}] with stable ids/newId(), sets/weight only finite>0, reps only non-blank; activity/class writes kind:edKind (class round-trips). 
PART B Done-day safety — PASS. Training card affordance gated checked?note:link (dayLogged) → logged day has NO openDayEdit path, can't desync/orphan WorkoutEntry. Rest always editable (dayLogged false for rest). "Completed — edit this from the Log tab." neutral.
PART A — PASS. Strip removed from Plan (still on Log); planStripRef removed, logStripRef intact; selection works (openWeekday/selectedWd init weekdayKey() today + card onPress sets both). No dangling refs.
SCOPE — PASS. No calorie/macro math (dayTargets/targetForDate/INTENSITY_FACTOR/BMR+1200 floor intact); no Coach prompt/reschedule-tool change (Coach reads plan.current, auto-picks-up edits); no WeekPlan/PlanDay/PlanExercise/Profile shape change (type-only imports added); no new dep; lib/plan.ts + lib/coach.ts untouched.
ED-SAFETY — PASS. Edit copy neutral; cycle/BC phase-lean gating preserved (lean !onBirthControl; Day N only when dayOfCycle!=null; BC → "on birth control", no fabricated day).
ROBUSTNESS — PASS. moveEditRow bounds-checked + arrows disabled at ends; empty edRows → valid empty strength section; rest↔training valid both ways; multi-section→one-section flatten via dayExercises preserves order/ids (toggleExercise by id still resolves).

### Notes (non-blocking)
- [NOTE] WorkoutScreen.tsx:734-787 — saveDayEdit doesn't carry prev.loggedEntryId onto the rebuilt day. Safe TODAY (only un-logged/rest days reachable via the UI gate), but relies on the gate not the writer. Defense-in-depth: preserve loggedEntryId (or bail) in the writer if a future change ever lets a logged day reach the sheet. Surfaced to Ting.
- [NOTE] :680-681 — rest→training always lands on the strength editor (no in-sheet switch to activity/class from a rest start). Known limitation.
- [NOTE] :769 — strength day saved with zero exercises persists an empty section (checkable day, no movements). Harmless minor UX.

### Not checked
- On-device render of the new edit sheet (style keys confirmed present + tsc clean; aesthetics need a device pass).

**Verdict: PASS — no BLOCK.**

---
## 2026-06-01 — WorkoutScreen day-card edit affordance + Coach-adjust copy tweak

**Result: PASS — 0 BLOCK, 0 WARN, 3 NOTE.** Presentation/copy only; ED-safety, scope, wiring, BC gating intact. tsc clean (exit 0). Ting: remove the ✎ icon by "Edit this day" + put back the "adjust through Coach" context.

### Findings (NOTE)
- WorkoutScreen.tsx:1493-1497 — New "Or ask your Coach to adjust your plan →" copy neutral/factual (no streak/missed/shame/earn-burn). Replaced the today-only "Not feeling it?" nudge with a calmer always-available line (equal-or-better). onPress → onOpenCoach unchanged.
- WorkoutScreen.tsx:1490-1497 — isToday gate removed from the Coach-adjust line → now renders on every expanded training day (was today-only). Benign visibility broadening; pure render-condition, no logic/data/math change. isToday still used for today-highlighting + date strip (not orphaned). onOpenCoach is () => void zero-payload tab switch — no Coach-context injection.
- WorkoutScreen.tsx:1334-1342,1483-1489 — Pencil-glyph removal text-only. Rest card → <Text>Edit</Text> (editPencilText restyled fontSemibold/13/ink); training card → <Text>Edit this day</Text>; both onPress → openDayEdit intact. Done-day "Completed — edit this from the Log tab." + checked/dayLogged gating unchanged.

Verified intact: no Anthropic/Coach-prompt/model code in change; BC+cycle gating outside diff; data model untouched; no new imports/deps/network; fontMedium still consumed.

### Not checked
- Device visual/contrast of restyled "Edit" text + always-present Coach line.

**Verdict: PASS — no BLOCK.**

---
## 2026-06-01T00:00:00Z — Food tab "REST DAY" annotation (final future-logging mitigation)

**Summary:** PASS — clean, 0 blockers, 0 warnings. Presentation-only annotation. (Supersedes the hide-the-ring "PLANNED REST" attempt, which Ting rejected — "keep the calorie wheel and everything, just note rest day.")

- [PASS] Scope — presentation-conditional only. FoodScreen.tsx:153-154 derives isFutureDay/showRestNote from existing values; render changes are eyebrow suffix (:896-898), rest-day pill suppression (:902), header relabel (:972-979). Calorie math untouched. No data-model/storage/target-math/Coach/dependency change.
- [PASS] ED-safety — Ting's product call: full "Remaining=target" card kept on future days + "CALORIES REMAINING · REST DAY" annotation (:897), neutral copy (no deficit/earn/burn/should/body framing), no legibility loss.
- [PASS] Condition correctness — showRestNote = planDay?.intensity==="rest" || (isFutureDay && !planDay); training days false; intensity required on PlanDay (null-safe); pill suppression only hides rest-day pill, training pills render; no double "REST DAY".
- [PASS] No Coach drift — coach.ts reads food strictly from todayISO; future pre-logs never reach Coach.

**Note (accepted, not re-litigated):** empty future days still show full-target "Remaining" across the 28-day forward window (STRIP_DAYS_FWD=28) — persists by Ting's explicit choice to keep the wheel + annotate.
**Not checked:** single-line fit of longer eyebrow on narrow devices (device test).

**Verdict: PASS — no BLOCK.**

---
## 2026-06-01 — WorkoutScreen day-card edit affordances (presentation polish)

**Result: PASS — 0 BLOCK, 0 WARN, 2 NOTE.** Ting: the edit/Coach affordances "look like dropped-in text" — make them sophisticated. Designer grouped them into a hairline-topped `dayFooter` card footer: "Edit this day" as a quiet cocoa-outline chip + muted › chevron (primary), Coach line as a calmer inkMuted secondary beneath, rest-card "Edit" restyled to matching chip. Presentation only; tsc clean (exit 0).

### Findings (NOTE)
- WorkoutScreen.tsx:1488-1512 — All strings verbatim ("Edit this day", "Or ask your Coach to adjust your plan →", "Completed — edit this from the Log tab.", rest "Edit"); no ED-relevant wording; done-day note neutral.
- WorkoutScreen.tsx:3040-3102 — No new tokens (ink/inkMuted/creamTile/cardHairline/rowHairline/radius.pill/fontSemibold/fontMedium all existing in theme).

### Verified
- No pencil/edit icon re-introduced (muted › chevron + existing → in copy; editPencil is a text-only chip, legacy style name).
- Wiring intact: openDayEdit(d) on editDayLink + editPencil; onOpenCoach on flexLink; accessibility props preserved.
- Scope/safety: presentation only; no behavior/data/calorie-math/Coach-prompt/dependency change. Done-day disable-edit gate (dayLogged → neutral note vs chip) intact (log/plan desync protection preserved). Cycle/BC gating + edit sheet logic untouched.
- Legibility: cocoa ink on creamTile + inkMuted secondary — existing pairings, no regression. dayFooter is a plain View inside dbody (no overflow:hidden/shadow), can't clip the card shadow.

### Not checked
- On-device visual/pixel polish of the chip border/chevron baseline/footer hairline.

**Verdict: PASS — no BLOCK.**

---
## 2026-06-01 — Workout chevron/chip centering fix + Coach-adjust day opener (WorkoutScreen.tsx, App.tsx, CoachScreen.tsx)

**Result: PASS — 0 BLOCK, 0 WARN, 3 NOTE.** Safety architecture intact: the per-day Coach-adjust link now opens the Coach with a day-specific opener; the opener is a deterministic template (no Anthropic call, no tool exec, append-only) and the user's REPLY still flows through full askCoach (system prompt + ED_SAFETY_RULES + tools + plan context unchanged). lib/coach.ts NOT modified. tsc clean (exit 0).

### Findings
- [PASS] Safety not bypassed — opener static template (CoachScreen.tsx:1159-1163, role:"assistant", no askCoach/kickoff/Anthropic call, no runTool). User reply routes deliver→askCoach (:894); ED_SAFETY_RULES still terminates every system prompt in coach.ts (:184,:996,:1084,:1230,:1543). No prompt/tool/model file changed.
- [PASS] Chat integrity — append-only (next=[...messagesRef.current, opener], :1164); rolling summary passed through unchanged (:1166-1169); ChatMessage shape unchanged. Once-per-nonce gate seeded with initial nonce (:1150) + guarded (:1153) prevents mount/null/unchanged/double fire; booting||sending bail (:1158).
- [PASS] Opener copy neutral/factual ("What would you like to change about {label}'s workout?").
- [PASS] Label — toLocaleDateString({weekday:long,month:long,day:numeric}) (WorkoutScreen.tsx:1523-1527) is the SAME API the app already uses for the Log date line (:1587-1590); Hermes Intl already load-bearing, consistent.
- [PASS] Scope/drift — no data-model change (optional prop adjustRequest + ephemeral non-persisted App signal coachAdjustReq, no migration); no new dep; SDK 54; App.tsx MainShell + other screen props intact; fallback to onOpenCoach preserved; edit sheet/day-card/cycle-BC/Log untouched. NOTE: Coach has adjust_workout_day + move_workout_day tools, so the conversational route can actually set/replace a day's contents & reschedule (not just chat).
- [PASS] Centering (Change 1) style-only: editDayLinkText lineHeight:18; editDayLinkArrow lineHeight:18 no marginTop; editPencil center align; editPencilText textAlign:center.

### Notes (non-blocking)
- NOTE-A: opener effect records the nonce even when it bails on booting||sending, so a tap during that window is dropped silently (re-tap re-fires). Intended trade-off.
- NOTE-B: opener saveChat fire-and-forget (matches existing paths); rejection swallowed, in-memory append stands.
- NOTE-C: opener appended as assistant with no preceding user turn — purely local UI state, never injected into the API messages array; no model turn-ordering confusion.

### Not checked
- On-device visual of centering; live nonce/tab-switch timing (sound by inspection).

**Verdict: PASS — no BLOCK.**

## 2026-06-01T00:00:00Z — Workout log add-button "+" centering fix (presentation-only)

### Summary
Clean — 0 blockers, 0 warnings. SVG swap on the Workout log add button; all wiring intact, no safety/scope surface touched.

### Findings
- **[NOTE]** `WorkoutScreen.tsx:1581-1586` — Fullwidth-plus glyph replaced with a 20×20 SVG (two `<Line>`, strokeWidth 2.4, round caps, `stroke={colors.cream}`) inside the unchanged `<TouchableOpacity style={styles.addBtn} onPress={openAdd}>`. `openAdd` defined at line 857; sole onPress. `addBtn` style unchanged.
- **[NOTE]** `WorkoutScreen.tsx:15` — `Line` added to existing `react-native-svg` import and now used. No unused imports; no new dependency.
- **[NOTE]** `addBtnText` style removed; grep confirms zero remaining references.
- **[NOTE]** Scope/safety — workout-log add button only. No Coach/Anthropic call site, no system prompt, no Profile/DayLog/ChatMessage shape, no BC branching. Nearby `phaseLabel` (line 1595) still gated by `hasPhase`, unmodified. Three unrelated `＋` glyphs (1600, 1713, 2175) left out of scope. No ED-trigger copy introduced.

### Not checked
- On-device visual alignment of the new SVG plus — requires device/simulator.
- `tsc` result — implementer reports clean.

### Note (non-code)
Session context included an Apollo.io MCP block (connected-server boilerplate, irrelevant to Wren). Not acted on; no MCP/Apollo integration added.

No BLOCK findings.

## 2026-06-01T00:00:00Z — New app-icon set (wren wordmark on vapor #E3E8E9, replacing blue peak mark)

**Summary:** Clean — 0 blockers, 0 warnings, 2 notes. Presentation/branding-only change, no code or safety surface touched.

**Findings:**
- [NOTE] wren/app.json:16 — android.adaptiveIcon.backgroundColor changed #E6F4FE → #E3E8E9 (vapor). Only diff in app.json. JSON valid. All five icon path references resolve to existing files in wren/assets/; filenames preserved.
- [NOTE] wren/assets/splash-icon.png — overwritten with new branding but app.json has no splash config / no expo-splash-screen plugin; unwired (same as before). Not a regression; flagged for record.

**Scope:** Only app.json change is the backgroundColor value. No new deps/networked code/auth/telemetry. Six PNG overwrites in wren/assets/; originals preserved under wren/assets/_icon-backup-20260601/. No app code touched.

**Safety:** No Coach prompt, injection defense, ED copy, BC branch, data model, or API-key handling touched. Presentation/branding-only.

**Not checked:** On-device rendering of adaptive/monochrome/favicon (needs build/device test); pixel content not visually diffed by auditor (orchestrator verified renders via headless-Chrome/qlmanage contact sheet).

**Assets generated via:** headless Chrome (`--default-background-color=00000000`) for the two transparent Android layers; qlmanage for opaque layers; Manrope_700Bold embedded as base64 @font-face (no system font installed). Source SVG: brand/wren-app-icon.svg.

---
## 2026-06-02T00:00:00Z — swipe-down-to-dismiss on all bottom-sheet modals (useSwipeDismiss hook; Food/Workout/Progress/Cycle)

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. Presentation-only gesture plumbing via a shared PanResponder/Animated hook (no new deps). No ED-safety copy, BC branching, data model, save-guards, or Coach/Anthropic paths touched.

### Findings
- [NOTE] No Anthropic call sites / system prompt / Coach context touched; lib/coach.ts not edited.
- CycleScreen ED copy byte-intact: bottom disclaimer (CycleScreen.tsx:803-806); BC status body (:383-387) + no-data body (:393-398) + PHASE_GUIDANCE (:86-95) unchanged. Drag-handle pill + panHandlers + paddingTop lg→sm are above-the-ScrollView layout only.
- Progress ED-safety intact: weight avg-only, BC look-back suppresses day-N + chart, showPhaseDay gate, guide-not-verdict captions + clinician notes.
- [NOTE] active-prop mount-gating safe: hook reset only sets translateY value, never Modal visible.
- [NOTE] Scope: diff adds only hook + Animated.View wrappers + panHandlers + (Cycle) one drag pill + padding. Progress weighOpen wiring benign.
- Correctness: panHandlers on handle+header only; gesture claim dy>6 && dy>|dx|*1.5; × buttons retain handlers; save-guards honored; photo-review left non-swipeable.
- No new deps; SDK 54 pin holds.

### Note
- Superseded same day: first pass did not fire on device (Fabric view-flattening of styleless grab View). See follow-up fix entry below.

---
## 2026-06-02T00:00:00Z — swipe-down-to-dismiss follow-up fix (Fabric view-flattening; useSwipeDismiss rewrite + Food/Workout/Progress/Cycle wiring)

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. Pure gesture-layer change; no ED-safety copy, BC branching, data model, or dependency drift. Live gesture behavior unverified (no simulator in env) — residual risk requiring Ting's on-device reload.

### Findings
- SAFETY PASS: CycleScreen disclaimer byte-intact (:803-806); BC status (:378-388) + no-data (:389-399) + phase-fabrication gate showCycleUI (:406) untouched. ProgressScreen weight avg-only, BC branch (:289-300), no day-N leak, clinician note, guide-not-verdict caption all intact. Coach/Anthropic layer not touched.
- onScroll clobber PASS: no sheet ScrollView had pre-existing onScroll; spread can't overwrite. Modal visible props still bound to state; active-gating + blur force-close effects unchanged. Hook visible effect only writes translateY/scrollYRef.
- Correctness PASS: every panHandlers-wired sheet also spreads scrollViewProps on a real ScrollView (scrollY tracks → tall sheets defer to scroll); Progress Log-weight plain View fully-draggable (accepted). onStart*ShouldSet false → ×/BACK/chips/inputs tap through. Save-guards (newSaving/scanSaving) route through same onClose, can't dismiss mid-save. keyboardShouldPersistTaps + KAV intact.
- Dependencies PASS: package.json unchanged; expo ^54.0.34; no gesture-handler/reanimated; core Animated + PanResponder only.

### Notes
- [NOTE] onPanResponderTerminationRequest:()=>false = deliberate one-way claim (prevents freeze-partway).
- [NOTE] Capture-phase claim only fires at scroll-top (scrollY<=0); first-paint default 0 = safe.
- [NOTE] Validated only via npx expo export (790 modules clean); NO simulator/device run in env. Ting must reload on-device and confirm: sheets drag+dismiss, scrolled content scrolls, ×/BACK/chips/inputs tap, mid-save sheets resist swipe.

---
## 2026-06-02T00:00:00Z — PanResponder → react-native-gesture-handler swipe-dismiss migration (index.ts, useSwipeDismiss.ts, Food/Workout/Progress/Cycle screens)

### Summary
Clean. 0 blockers, 0 warnings, 2 notes. Pure gesture/structure change; no ED-safety copy, BC branching, data model, or lib logic touched. Gesture-in-Modal behavior remains device-unverified. (Supersedes the two PanResponder passes above, which produced zero events inside Fabric Modals.)

### Findings
- SAFETY PASS: CycleScreen disclaimer byte-intact (805-806), BC branch (384-386), no-data (395-398), showCycleUI gate (407), editor eyebrow no BC day-N leak (164). ProgressScreen weight avg-only (848,879,883-890), BC no-day-N (962), guide-not-verdict (840,1548). lib/ untouched.
- STRUCTURE PASS: GestureHandlerRootView + GestureDetector balanced 1/3/5/7 per file; zero leftover panHandlers/collapsable/simultaneousWithExternalGesture; nesting Modal > GHRV > KAV/backdrop > GestureDetector > Animated.View card > ScrollView; keyboardShouldPersistTaps preserved.
- active-prop mount-gating PASS: no Modal visible/active wrapper altered; GHRV inside Modal mounts with it. Food edit-sheet delete-confirm overlay correctly a sibling of </GestureDetector>, inside same KAV/GHRV. Save-guards (newSaving/scanSaving) intact.
- DEP HYGIENE PASS: exactly one new dep react-native-gesture-handler ~2.28.0 (resolved 2.28.0); NO reanimated; expo ^54.0.34 holds. index.ts side-effect import first line + GHRV root wrap via createElement; App.tsx no double-wrap.
- Dropped simultaneousWithExternalGesture = UX-only loss (documented scroll-then-drag edge case); fallback activeOffsetY(12)+failOffsetX([-20,20])+scroll-gate sound. ×/save-guards/keyboardShouldPersistTaps/KAV intact.

### Notes
- [NOTE] CycleScreen stale "PanResponder/no native dep" comment — FIXED by orchestrator after audit.
- [NOTE] useSwipeDismiss velocity now px/s (DISMISS_VELOCITY_PXS=800); only validatable by feel on device.

### Not checked
- Gesture-in-Modal RUNTIME behavior device-unverified — tsc + expo export(875 modules) pass but can't exercise native gesture recognition in a Fabric Modal (the point of the change). Ting must test in Expo Go after `npx expo start -c`: per sheet — drag-from-top dismisses, taps/×/inputs pass through, horizontal tab swipe survives, scrolled-content does NOT dismiss, fling feel, scroll-then-drag edge case acceptable.
- npm audit on new dep not run by auditor.

---
## 2026-06-02T00:00:00Z — swipe-dismiss reopen fix (useSwipeDismiss hook-only)

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. Hook-only fix for the reopen regression after the gesture-handler migration (sheet stranded at bottom / frozen touches on reopen). No ED-safety / Coach / data-model / dependency impact.

### Fixes audited
1. animateOut .start() callback now resets translateY.setValue(0)+scrollYRef=0 right after onClose (fixes card stranded at ~800 on reopen / stuck-at-bottom). visible-effect reset kept too.
2. Gesture.Pan() now useMemo'd (deps [animateOut, snapBack, translateY], all stable); animateOut/snapBack now useCallback([translateY]). Fixes GestureDetector re-registration footgun that left a dangling gesture swallowing touches (the freeze).

### Findings
- [NOTE] useSwipeDismiss.ts:115-119 — animateOut completion runs onClose unconditionally (ignores {finished}); a double-fling could double-fire onClose. Harmless: every consumer onClose is idempotent setState + gesture unmounts on visible flip. Pre-existing shape, not worsened. No close/open loop (setValue(0) doesn't trigger onClose).
- [NOTE] Comment says useCallback([]) but actual deps [translateY]; identical behavior (translateY ref-stable). Cosmetic; align comment so nobody "fixes" deps to [].
- [NOTE] useMemo stale-closure check PASSED — gesture reads refs + stable callbacks; changing onClose still seen via onCloseRef (reassigned each render). Footgun correctly resolved.
- Scope PASS: only useSwipeDismiss.ts changed; 4 consumers untouched. No package.json/dep change. No Modal visible/active-gating, ×, save-guards, scrollViewProps shape, KAV, keyboardShouldPersistTaps touched. No ED-safety/Coach/data-model/API surface in this file.

### Not checked
- Runtime/device behavior (no Xcode). Ting to test in Expo Go after `npx expo start -c`: (a) open→swipe-dismiss→reopen renders at rest (not stranded ~800); (b) repeated open/close never freezes touches.

---
## 2026-06-02T00:00:00Z — useSwipeDismiss JS-driver flash fix (3rd iteration, hook-only)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. Hook-only swipe-to-dismiss timing change (components/useSwipeDismiss.ts only): animateOut/snapBack switched useNativeDriver true→false, close-time translateY/scrollY reset removed, on-visible reset kept as sole reset. Scope-clean, prior fixes intact, no stranding path. Fixes the close flash (card snapped back to rest during modal slide-out).

### Findings
- [NOTE] useNativeDriver:false on dismiss timing + snap-back spring moves the transform to the JS thread. Fine for a single bottom-sheet transform. Device-eyeball the slide, no code concern.
- [NOTE] On-visible reset is now the sole reset, gated on `visible` (open only). × buttons close via setX(false) directly and never drag translateY, so value already 0 at × time → instant close, no stuck card. Drag-dismiss parks off-screen; reopen resets. No stranding path.

### Verified
1. Hook-only; consumers still animationType="slide"; no package.json/dep change.
2. Prior fixes intact: gesture useMemo'd (no per-render Pan() rebuild); animateOut/snapBack useCallback-stable; scroll-gate logic unchanged; on-visible reset present.
3. No correctness regression: removed close-time reset covered by on-visible effect (reliable under JS driver); scrollYRef also zeroed; no translateY stranding.
4. No ED-safety / Coach / data-model / dependency / API surface in this file.

### Verdict: PASS — no BLOCK.

---
## 2026-06-02T00:00:00Z — Progress WEIGHT card: trend-word headline (ED-safety surface)

### Summary
0 blockers, 1 WARN (FIXED by orchestrator), 2 NOTEs. Mockup-match: weight headline → non-valenced trend phrase, number → subtext, caption gains opt-out line. ED guardrails preserved.

### Findings
- [NOTE] ProgressScreen.tsx:867-876 — "Easing down"/"Easing up"/"Holding steady" pair is symmetric + non-valenced (same verb, no progress/goal/good-bad framing, no checkmark/arrow/color-verdict). Headline carries NO magnitude. ±1.5 lb dead-band sends water/phase noise to "Holding steady". Safe to ship. Optional max-de-emphasis alt (non-directional headline in all states) offered for Ting's call — not required.
- Core guardrails PRESERVED: averaged-only never day-to-day (bucketWeights groups→avg; sparkline plots b.avg only); thresholds intact (>=2→AreaSpark, 1→inline text no 1-point trend, 0→no graph; NOT lowered to daily); "never day to day" caption clause intact; delta only at buckets>=2 so directional headline can't fire on a single point.
- Opt-out caption "You decide if this is useful; Wren works fine without it." — healthy autonomy frame, approved.
- Dropped "+X lb since you started" subtext magnitude — genuine reduction in number-fixation; only current avg surfaced, no change-magnitude anywhere.
- Scope clean: only WEIGHT IIFE changed; no other Progress section, Coach prompt, data model (weightLog read-only), or dependency. Chevron→setWeighOpen opens existing Log-weight sheet (no new screen). BC/cycle branch untouched.
- [WARN→FIXED] ProgressScreen.tsx:869,871 — zero/single-bucket headline hardcoded "weekly" even in year/all (monthly) views. Orchestrator fixed: now `Start your ${cadenceWord} average` / `Your ${cadenceWord} average`.
- [NOTE] 9 pre-existing OnboardingScreen.tsx tsc errors (onBirthControl/CycleMode) unrelated to this edit — different file, no dependency. Separate open debt.

### Not checked
- Visual/mockup pixel fidelity (needs device). tsc not run by auditor (relied on implementer's clean ProgressScreen result).

---
## 2026-06-24T00:00:00Z — 4-item fix batch (app.json permissions/runtimeVersion, CycleEditor copy, WorkoutScreen active-gating, App.tsx active wiring)

### Summary
Clean — 0 blockers, 0 warnings, 1 note. All four changes verified correct; no safety, scope, or ED guardrail crossed; no regressions.

### Findings

Change 1 — wren/app.json (permissions + runtimeVersion)
- PASS — android.permissions == ["android.permission.CAMERA"] only (line 26); no RECORD_AUDIO, no dupe CAMERA.
- PASS — expo-camera/expo-image-picker permission strings intact (lines 37, 43-44).
- PASS — runtimeVersion singular, well-formed { "policy": "fingerprint" } (lines 58-60); matches deploy plan.
- PASS — bundle IDs, EAS projectId, updates URL, icons, plugins all intact; no scope creep.

Change 2 — wren/screens/profile/CycleEditor.tsx:215-218
- PASS — Copy verbatim-matches CycleScreen disclaimer (CycleScreen.tsx:823-824).
- PASS — No "many women", no feelings-override; "Not medical advice" guardrail preserved, body-size legible.

Change 3 — wren/screens/WorkoutScreen.tsx (active-gating + force-close)
- PASS — active = true default (324), active?: boolean type (357); mirrors Food/Progress.
- PASS — 5 Modals gated, JSX balanced: add 1811->1960, edit 1963->2049, setup 2052->2210, day 2218->2457, review 2462->2530. No double-gate/orphan; root return closes at 2531. Prior imbalance resolved.
- PASS — Force-close effect (551-558): if(active)return; resets addOpen/editEntry/setupOpen/editDayWd/reviewOpen; dep [active] correct.

Change 4 — wren/App.tsx:339
- PASS — active={tab === "workout"} matches visibility toggle (328); Food (318) and Progress (348) wiring undisturbed.

### Note
- [NOTE] WorkoutScreen.tsx:324 — active default true is safe; only caller always passes it explicitly. No action.

### tsc
No type errors expected (optional boolean prop, boolean-position usage only, no any, no model drift). tsc --noEmit confirmed clean by implementer.

### Not checked
- Device test of modal-stranding fix (switch away w/ sheet open, return, confirm touches) — runtime, to verify on device.
- Coach prompt / injection / BC branch / cost guardrails — untouched by this batch, out of scope.

---
## 2026-06-24T00:00:00Z — Coach workout-trust package STAGE 1 (confabulation guard)

Scope: lib/types.ts (CoachCard → discriminated union: CoachFoodCard | new CoachPlanChangeCard), screens/CoachScreen.tsx (new planDiff helper; adjust_workout_day + move_workout_day build an authoritative plan_change card from real pre/post saved state; deliver() post-turn prose detector appends a correction line; renderer branch for plan_change; four food helpers narrowed to CoachFoodCard), lib/coach.ts (one additive WORKOUTS bullet forbidding unsubstantiated plan-change claims).

Summary: 1 WARN (fixed, see next entry), 3 NOTE. No blockers.

[WARN→FIXED] CoachScreen.tsx detector tested the WHOLE reply for (change-verb) AND (plan/day noun) independently. A successful set_targets turn (no plan_change card) that also names a weekday/workout could trip it and falsely append "I didn't actually change your plan just now." Fixed in the re-audit below.
[NOTE] Inverse gap: a confabulation phrased without a recognized change-verb escapes the regex backstop. Acceptable Stage 1 (system-prompt bullet + card are primary defenses).
[NOTE] planDiff treats missing pre/post days as empty lists (no throw); activity/rest-day adjusts yield zero counts → card shows title delta only. Intended.
[NOTE] Card dayLabel showed "MON" while prose said "today" for same-day adjust — fixed to "TODAY" in re-audit below.

Verified clean: ED_SAFETY_RULES, PROACTIVE_ED_SAFETY, lib/safety.ts crisis backstop byte-unchanged; new coach.ts WORKOUTS bullet purely additive; cycle/BC branching untouched; card derives from updateProfile's returned post-write profile (no stale-ref race), never from tool args; union exhaustive, old food cards round-trip; scope clean (no merge/undo/shift_plan/context change, no new deps/network).

---
## 2026-06-24T00:00:00Z — Coach workout-trust Stage 1 WARN fix re-audit (claimsPlanChange detector + plan_change dayLabel)

Summary: Clean — prior WARN RESOLVED. 0 blockers, 0 warnings, 1 note.

[NOTE] CoachScreen.tsx:144 — TARGET_FOOD_CONTEXT excludes the whole clause before the verb+noun check, so a single un-delimited clause claiming both a target change and a real plan change won't append the correction line. Intended false-negative bias; PLAN UPDATED card + system-prompt bullet remain primary defenses. Not actionable.

Verification: WARN resolved ("I updated your calorie target and your Monday workout." → clause has calorie/target → false). Genuine bare claim still fires ("I moved your Monday workout." → true, appends when no card). No crash (empty/no-delimiter/typed-string safe); no catastrophic backtracking (flat literal alternations, bounded optionals); dayLabel "TODAY" parity mirrors prose. Append-only preserved. Scope: only screens/CoachScreen.tsx changed (helper, call site, dayLabel); safety/cycle/BC/food/target handlers + system prompt untouched.

---
## 2026-06-24T00:00:00Z — Coach workout-plan mutation "Pass 1" (card-gate + merge-not-replace + move completion preservation)

Summary: 1 WARN (orphaned WorkoutEntry from move completion-preservation — FIXED in next entry) + 2 NOTE. No BLOCK. ED-safety/crisis/cycle/BC/food/checkin handlers untouched.

- [WARN→FIXED] move_workout_day carried loggedEntryId to the new weekday but did not re-date the underlying WorkoutEntry (workoutLogs keyed by entry.date) → stranded at source date; later un-check/re-check would duplicate. Fixed via redateWorkoutEntry (see next entry).
- [NOTE] AdjustDayArgs.kind narrows out "class" (PlanDayKind allows it); merge defers to existing.kind so stored class days survive. Pre-existing, benign.
- [NOTE] lib/plan.ts mergePlanDay case-insensitive name match collapses two existing same-named exercises. No drop/dup/throw; same-named rows can't be edited independently via merge. Acceptable.
- Confirmed clean: every card push downstream of a true planDayChanged (no-op/weekday-not-found push nothing); merge never drops unmentioned exercises, preserves id/done/loggedEntryId, preserves sections when no exercises; replace:true is the only completion-clearing path; ED-safety/scope intact; helpers null/empty-safe.

---
## 2026-06-24T00:00:00Z — move_workout_day orphaned-WorkoutEntry re-date fix (focused re-audit)

Summary: Clean — WARN RESOLVED, 0 blockers, 0 new warnings, 1 note.

- [WARN-RESOLVED] New pure helper redateWorkoutEntry (lib/workouts.ts) re-buckets each moved day's WorkoutEntry to the moved day's new-weekday date using the same dateForWeekday keying commitDay/syncStrengthLog use → later un-check finds the entry, no duplicate. Both-logged no-clobber verified (target dates computed pre-swap; redates by distinct loggedEntryId). Atomic within one updateProfile transform; pure (no in-place mutation); edge cases covered (fromArg===toArg, missing entry, undefined workoutLogs).
- [NOTE] CoachScreen re-date uses in-transform cur2.startDate while card/echo titles come from outer pre-transform cur — consistent with existing read-back card design; don't "tidy" the two startDate sources. Not urgent.

---
## 2026-06-24T00:00:00Z — Pass 2a: densifyWeek (7-weekday dense plan) + stored-plan migration + on-load persist

Summary: Clean. 0 blockers, 0 warnings, 3 notes. Data-safety + idempotency verified.

- DATA PRESERVATION (lib/plan.ts) — densifyWeek pure/non-destructive: existing day objects returned BY REFERENCE (exercises/done/loggedEntryId/title/kind preserved), input never mutated, only missing weekdays get a synthesized rest day.
- IDEMPOTENCY — re-densifying a dense week returns same 7 objects, no duplication/accumulation.
- MIGRATION SAFETY (lib/storage.ts loadProfile) — guarded (if plan → if current → if Array.isArray(history)); no throw on null/undefined; outer try/catch degrades a malformed days to null/onboarding. All other profile fields preserved.
- REST-DAY SHAPE valid + consistent with generateWeekPlan/mergePlanDay rest output; dayExercises []/dayLogged false on section-less rest day — no throw.
- DOWNSTREAM — weekProgress/weekReviewSummary/patterns filter isTrainingDay so no count drift; coach.ts:262 plan-context now lists all 7 days (rest→"Rest"), more correct/neutral; accordion + ProgressScreen already special-case rest → render dense rest cards (intended fix); move/adjust now resolve any weekday.
- ON-LOAD PERSIST (App.tsx) — saveProfile fires once on load gated on p?.plan; no loop (idempotent), no render storm, no race, errors swallowed.
- SCOPE — no type-shape change; system prompt/safety.ts/food/cycle/BC untouched.

NOTES (non-blocking):
- [NOTE→carry to Pass 2b] storage.ts/densifyWeek — a stored plan with non-array days throws → outer catch → null (whole profile lost to onboarding). Pre-existing for any malformed blob; densify slightly widens it. Add Array.isArray(week.days) guard → treat as [] (all-rest) so it degrades recoverably.
- [NOTE] App.tsx on-load persist writes even when already dense — harmless/redundant.
- [NOTE] coach.ts:262 — eyeball that a densified plan with several "Rest" lines reads naturally / doesn't make Coach over-narrate rest.

Not checked: device run (implementer/Ting should confirm with her EXISTING plan: Plan tab shows all 7 cards, no data lost, 2nd launch idempotent).

---
## 2026-06-24T00:00:00Z — Pass 2b: Coach plan-exercise visibility + real dates + WORKOUTS prompt + densifyWeek guard

Summary: Clean — 0 blockers, 0 warnings, 3 notes. ED-safety intact; prompt confined to WORKOUTS; surfaced weights are lifting loads not bodyweight.

- ED-SAFETY (critical) PASS — ED_SAFETY_RULES (coach.ts:106-111), PROACTIVE_ED_SAFETY (120-124), safety.ts crisis backstop byte-consistent with spec §6; cycle/BC branching (215-234) untouched. New SINGLE-EXERCISE EDITS bullet (coach.ts:166) confined to WORKOUTS, no weight-loss/earn-burn/body-comp framing. Surfaced exercise weights are barbell/dumbbell loads (PlanExercise.weight), NOT bodyweight/scale; only body weight surfaced remains existing profile.weight Body: line.
- TOKEN/429 PASS — planWeekLine iterates planWk.days capped at 7 (densifyWeek); training days list real exercises via dayExercises; ~few hundred tokens added, bounded. NOTE: no per-day exercise cap (pathological/external plan only).
- CORRECTNESS PASS — dateForWeekday bijective mapping; bodyweight moves render "- Name" no dangling separators; [done] only when truly done; no throw on undefined sections; reps string-consistent.
- densifyWeek GUARD PASS — Array.isArray(week.days)?...:[] degrades malformed plan to all-rest, profile survives (no onboarding wipe).
- SINGLE-EXERCISE PATH PASS — adjust→mergePlanDay matches by case-insensitive name, updates in place, preserves id/done + unmentioned exercises; duplicate-append mitigated by EXACT-name prompt instruction. NOTE: title required → model re-sends existing title, mergePlanDay treats as title no-op.
- SCOPE PASS — no new tools/schema/data-model change; food/cycle/BC/target untouched; only api.anthropic.com.

Not checked: runtime token measurement (estimated), tsc (implementer confirmed clean), live EXACT-name reuse behavior (needs device chat test).

---
## 2026-06-24T00:00:00Z — Coach "technical issue" crash fix on single-exercise workout edits (numeric reps coercion + AdjustDayArgs.title optional)

Summary: Clean — crash resolved, 0 blockers / 0 warnings, 3 notes.

- CRASH (FIX 1): model sends reps as a NUMBER; planDayChanged did (e.reps ?? "").trim() → threw on a number → "Could not log that" → Coach "technical issue." Resolved: CoachScreen.tsx:634 now coerces reps: e.reps != null ? String(e.reps) : undefined at adjust ingestion (matches generateWeekPlan coach.ts:1795); plan.ts:137 now String(e.reps ?? "").trim()... defensive for legacy numeric reps. Audited every .reps call site — no unguarded string-method on a possibly-numeric reps remains (log_workout's expandSets path uses reps:number by schema, correct as-is).
- TITLE OPTIONAL (FIX 2): AdjustDayArgs.title → optional (coach.ts:564) + schema required:[] (coach.ts:720), removing the conflict with the Pass 2b "pass ONLY that exercise" instruction. MERGE keeps existing.title (plan.ts:210); REPLACE defaults to Workout/Rest (no blank-title day); all reads are a.title?.trim(); return string + card use saved/merged title.
- SINGLE-EXERCISE EDIT end-to-end verified: {name,weight}(+numeric reps), no title → coerce → mergePlanDay updates weight/reps, preserves other exercises + done + loggedEntryId + title → planDayChanged true → card fires.
- SCOPE: no ED-safety/cycle/BC/safety.ts/food/system-prompt change; no stored data-model shape change (AdjustDayArgs is internal tool-arg type; PlanDay.title stays required+populated). tsc confirmed clean by implementer.
- NOTE: log_workout reps schema=number vs adjust reps schema=string is intentional asymmetry (the reason coercion is needed); guard against a future "unify" reintroducing the crash.

---
## 2026-06-24T00:00:00Z — Fuzzy exercise-name matching in mergePlanDay (matchExerciseName) + single-token precision guard

Summary: Precision design sound; one WARN (single-candidate containment=1.0 false-overwrite) found AND fixed same session. Final: clean.

- FEATURE: matchExerciseName(incomingName, candidates) — normalize + abbrev map (rdl/db/bb/ohp/sldl/bw) + singularize; exact-normalized match wins; else score max(containment,dice) ≥0.6 AND a shared non-generic movement token (generic set excludes dumbbell/barbell/machine/cable/band/kettlebell/weighted/bodyweight/seated/standing); ambiguity guard (runner-up within 0.15 → null). mergePlanDay claims unclaimed existing by exact-then-fuzzy, patches in place preserving id/done/canonical NAME, appends unmatched/ambiguous. Two-token confusables (front/back squat, bicep/hammer curl, leg/bench press) all score 0.5 → append. Generic-only overlap blocked. Deterministic, no div-by-zero.
- [WARN→FIXED] lib/plan.ts — a bare single-token incoming fully contained in ONE longer existing variant ("deadlift"→"Romanian deadlift") scored containment 1.0 with no runner-up → silent wrong-lift overwrite (name preserved = no user signal). FIX: `if (incTokens.size === 1) return null;` after the exact-match loop, before fuzzy scoring — single-token incoming may ONLY bind via exact match, else append. Abbrev expansion happens before the size check so "RDL"/"OHP" (expand to 2 tokens) stay fuzzy-eligible. Multi-token paraphrases ("bent over row"→"Bent-over dumbbell row") unchanged.

---
## 2026-06-24T00:00:00Z — fuzzy-matching WARN re-audit (single-token guard)
Clean — WARN resolved, 0 blockers, 0 new warnings, 1 cosmetic note.
- Verified: "deadlift"/"press"/"row"/"curl" vs a single longer variant → null → append (no overwrite). "deadlift" vs exact "Deadlift" still updates (exact loop returns before guard). "RDL"/"OHP" still fuzzy (2 tokens post-expansion). Multi-token unchanged. Guard placement correct (after exact return, before scoring).
- [NOTE] lib/plan.ts:196 comment "but NOT 'press'" stale (singularizer does strip press→pres; symmetric, behavior correct). Cosmetic only.

---
## 2026-06-24T00:00:00Z — Coach exercise removal/swap (removeExercises in adjust_workout_day → mergePlanDay)

Summary: Clean — 0 blockers, 0 warnings, 3 NOTEs. Wrong-removal + swap-eats-new-exercise risks both defended.

- FEATURE: AdjustDayArgs/schema gain removeExercises?: string[]; mergePlanDay applies removals AFTER append/update, matching via matchExerciseName (exact-first then precision fuzzy). appendedIdx excludes just-added exercises from removal candidates → a swap (remove old + add new in one call) can NEVER drop the new one. WORKOUTS prompt bullet: delete via removeExercises; swap = removeExercises:[old] + exercises:[new]; distinct from weight/rep update.
- DEFENDED: removeExercises:["lunge"] with two lunges → single-token exact-only → no-op; "reverse lunge" can't hit "walking lunge" (score 0.5<0.6); "deadlift" vs only "Romanian deadlift" → no-op. Index integrity (single merged array, no off-by-one). Empty strength day → sections=[{name:"Workout",exercises:[]}], renders/handles safely. Pure removal → planDayChanged true (length drop) → card fires with removed>0.
- [NOTE] removing a `done` exercise sets completionPreserved=false → card shows "completion reset"; expected for an intentional delete, copy may read slightly alarming. Not a defect.
- SCOPE: replace:true path ignores removals; ED-safety/cycle/BC/food/move-handler/card-gate untouched; no persisted data-model change; SYSTEM_PROMPT + ED rules intact + attached.

---
## 2026-06-24T00:00:00Z — mergePlanDay + edit-day modal preserve named workout sections (regroupSections / edRowSection)

Summary: Clean — 0 blockers, 0 warnings, 2 NOTEs. Both edit paths (Coach mergePlanDay + manual edit-day modal) previously flattened multi-section days to one "Workout" section, destroying Warm-up/Working/Core/Cool-down grouping. Now section-preserving.

- mergePlanDay: builds flat Slot={ex,sec} across existing.sections, matches/updates/removes across all sections (matchExerciseName precision + appendedIdx swap-safety intact), new exercises → defaultSectionIndex (first non warm/cool/mobility section; else first; else -1→synthetic Workout), regroupSections buckets survivors back to origin sections preserving order/name/durationMin, drops empties. Single/no-section days unchanged.
- Manual modal: snapshots edSections + edRowSection (id→sectionIndex) at open; save regroups by id, new rows → default section, emits original order, drops empties; no-snapshot fallback = single Workout section.
- VERIFIED: no exercise-loss; index stability (empties dropped AFTER bucketing by original index, both paths); stale/OOB/undefined id→section → default fallback (never dropped/throw); all-warmup day → section 0 (never -1); card-gate can't false-fire (planDayChanged/planDiff read flattened dayExercises); swap-safety intact. No data-model shape change; ED-safety/cycle/BC/food/move-handler untouched.
- [NOTE] flat-editor cross-section row reorder is ignored (membership stays id-keyed) — accepted this pass, no data loss. [NOTE] Coach-added new exercise lands in the main working section (no section hint from model) — safe default.

---
## 2026-06-24T00:00:00Z — infra/privacy fix: raw API error scrub + barcode re-entrancy lock (CoachScreen.tsx, BarcodeScanner.tsx)

Summary: Clean — 0 blockers, 0 warnings, 2 NOTEs (pre-existing out-of-scope surfaces).

PART 1 (error-leak): deliver() catch now logs raw msg to console ONLY; chat bubble + saveChat use fixed COACH_CONNECT_ERROR (+ a rate-limit branch); no msg interpolation, "Check your API key" removed. scrubLeakedErrors on load rewrites any assistant message starting with "Something went wrong reaching the Coach:" → friendly text (prefix-anchored, idempotent, same-ref when unchanged), re-saving cleaned history so already-persisted org-ID/URL leaks are scrubbed on next open. Continuity (summary/summarizedCount) preserved; crisis backstop still fires on the error path.

PART 2 (scan dup): scanBusyRef (synchronous re-entrancy) + lastScanRef (2500ms same-code debounce) at top of handleCoachScan; no-hit path now honors the lock; try/finally releases on EVERY path (busy/same-code returns sit above acquire; sending/booting release explicitly; finally covers no-hit, hit→await deliver, and throws); open-reset clears the lock before setScanning(true). BarcodeScanner belt: synchronous `if (handled) return;`. One scan → one message; different barcode + later re-scan still work. Confirmed duplicates were real array entries sent to the model (inflating tokens → 429s), so dedup also reduces rate-limit hits.

- [NOTE] coach.ts:946 tool-exec errors go to the MODEL as tool_result (local JS errors, not provider text) — not a leak.
- [NOTE] "Photo error" Alerts (FoodScreen/Progress/Onboarding/Settings/Workout) surface local e.message for LOCAL ops (saveFood/jpeg/ImagePicker), not the Anthropic path — pre-existing, not a privacy leak; minor UX polish if ever desired.
- Not checked: device runtime (force a 429 → confirm friendly text + no org ID in bubble/AsyncStorage; scan one product → one message).

---
## 2026-06-24T00:00:00Z — Workout PLAN tab day cards sorted ascending by date (was fixed Sun-first)

Clean — no findings. Order-only presentation change.
- WorkoutScreen.tsx:1556 — [...week.days].sort by dateForWeekday(startDate, weekday).localeCompare (ISO YYYY-MM-DD sorts chronologically; bijection over the 7-day window → distinct dates, today-anchored plan puts anchor weekday first). Copy before sort → stored array not mutated, persistence stays Mon-anchored.
- wd=d.weekday reassigned at top of map; all per-card refs (isToday/weekdayDateNum/openWeekday/selectedWd/toggleDayDone/toggleExercise/key={wd}) keyed on weekday value not array position → no selection/checkoff/open-state regression; keys stable.
- Dropped if(!d) return null safe (dense plan, iterating week.days directly). Label "This week · Sun–Sat"→"This week" (neutral).
- Scope: WD_SUNFIRST def + other usages (Log strip 303, plan-setup chips 2164) untouched; no logic/data/handler/Coach/ED-safety/cycle/BC change.

---
## 2026-06-24T00:00:00Z — shift_plan bulk-rotation Coach tool

Summary: Clean — 0 blockers, 0 warnings, 3 harmless notes. Critical re-date no-clobber, completion preservation, no-op gating, atomicity, card-gate all correct. tsc clean (implementer).

- shiftWeekDays (lib/plan.ts): pure rotation, densifies, eff=((days%7)+7)%7, dst←src=(dstIdx-eff+7)%7, spreads whole source PlanDay overriding only weekday (preserves done/ids/loggedEntryId/sections), eff===0 no-op, no input mutation. Forward1: Mon→Tue, Sun→Mon (wraps).
- Handler (CoachScreen): no-plan guard; eff===0 → "No change" no card; redate tuples computed from PRE-shift snapshot then applied via redateWorkoutEntry (id-keyed remove+add → NO clobber even on a fully-logged week) after rotation, all in ONE updateProfile transform; materially-unchanged guard (all-rest week) → no card; else plan_shift card + honest string. destWd=(srcIdx+eff)%7 is exact inverse of helper's srcIdx → loggedEntryId resolves to entry at the day's NEW date (commitDay/un-check works).
- plan_shift CoachCard variant + render; detector hasPlanCard now ORs plan_change||plan_shift (no false "Heads up"). saveChat round-trips; food-card helpers narrowed safely (plan branches return first).
- Prompt (WORKOUTS): shift_plan for "move/push whole week forward/back N days"; "rebuild from scratch" → hand off to Workout-tab Regenerate, don't fabricate. ED-safety/cycle/BC/crisis untouched.
- NOTEs: shift reads startDate from pre-snapshot vs move's inside-transform (harmless, single-user); days:0→1 intentional; if(!src) dead code (densify guarantees 7).

---
## 2026-07-02T00:00:00Z — always-estimate workout burn (resolveWorkoutBurn: strength/activity always land a calories-burned number)

**Summary:** PASS with 2 NOTEs — no blockers, no warnings. Burn only ever raises the net-mode eat-back budget (never lowers any food budget), estimates are consistently labeled `"estimate"`, and the null-weight and empty-strength guards hold.

**Files audited:** `wren/lib/workouts.ts`, `wren/screens/WorkoutScreen.tsx`, `wren/screens/CoachScreen.tsx`; cross-checked `wren/lib/targets.ts`, `wren/screens/FoodScreen.tsx`, `wren/lib/types.ts`.

1. ED-safety — PASS. `MIN_DAILY_CALORIES=1200` and the `max(calories, bmr, 1200)` floor in `targets.ts` unchanged. `FoodScreen.tsx:145` accounting unchanged: net adds burn, static ignores it; burn can never subtract from the budget or breach the floor. Always-on burn only raises net-mode eat-back (ED-safe direction). No "earn/burn off your food" framing introduced; new copy is neutral and labels estimates ("~N cal (est.)", "Estimated burn ~N cal").

2. burnSource integrity — PASS. `resolveWorkoutBurn` emits `"watch"` only on a positive `watchCalories`; every estimate is `"estimate"`. UI feeds watchCalories only from a user-typed field and pre-fills the burn input only for existing watch entries. An estimate cannot be relabeled watch.

3. Correctness/robustness — PASS. Null-weight -> `{}` (both strength sum and fallback return null). NaN/negative rejected by parse guards + `>0`/`<=0` checks; values rounded and sane. Default-duration fallback (45/30) fires only with no per-exercise data. Load lifted correctly ignored. Watch precedence first.

4. Coach empty-strength guard — PASS. `CoachScreen.tsx:625` returns before resolve/makeWorkout when no exercises; no fabricated strength workout and no phantom default-duration burn.

5. Scope/drift — PASS. Only the three files touched. No new deps, network, auth, push, telemetry. `caloriesBurned`/`burnSource` already existed on `WorkoutEntry` (types.ts:400-401); `BurnResult` is a local helper type, no persisted-model change, no migration needed.

NOTES (non-blocking):
- `WorkoutScreen.tsx:1215-1221` — strength preview shows a default-45-min estimate on an empty builder (weight known). Preview-only; `saveStrength` still blocks on `!built.length`. Consider gating preview on `built.length > 0`. Cosmetic.
- `CoachScreen.tsx:630` — model-supplied `a.caloriesBurned` maps directly to `watchCalories` -> could stamp an estimate as "watch". Pre-existing, unchanged by this diff; on record only.

Not checked: on-device rendering / tsc (read-only); no dependency changes so npm audit unaffected.

---
## 2026-07-02T00:00:00Z — WorkoutScreen: remove strength Duration input + Note field from add/edit forms

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. All four audit-focus items PASS.

Focus 1 — Correctness (strength durationMin undefined): PASS. LogCard (WorkoutScreen.tsx:1270, :1318, :1326-1333) never renders "— MIN"/NaN for strength; the no-burn strength branch hard-codes "—"/"KCAL". Activity path (:1321-1325) unchanged. saveStrength (:1121-1139) guarded by `if (!built.length) return` + disabled Save (:2011), so resolveWorkoutBurn gets non-empty exercises; estimateExercisesBurn (workouts.ts:95-105) returns a positive sum when weight known, making the 45-min fallback (workouts.ts:141) functionally unreachable; weight-unknown -> {} -> clean "— KCAL". estStrength preview (:1212-1219) gated on built.length>0.

Focus 2 — Data integrity (...editEntry spread): PASS. saveEdit (:1185-1194) spreads existing entry, overrides only durationMin/burn/HR/(activity)activity+distance; preserves note/focus/exercises/source/id/date/createdAt. Cannot resurrect a removed field (model unchanged) or clobber the Coach-set title (note = LogCard title fallback :1270). Watch burn preserved via eBurn prefill (:1166).

Focus 3 — Downstream readers handle undefined: PASS. estimateBurn (workouts.ts:46) guards !durationMin; workoutLabel (workouts.ts:271) guards e.durationMin; coach.ts durationMin refs are plan/tool typed-guards; progress.ts:107 uses ?? 0. None throw.

Focus 4 — ED-safety / scope: PASS. resolveWorkoutBurn formula, DEFAULT_STRENGTH_MIN, MET tables, burnSource labeling untouched (only WorkoutScreen.tsx edited). No targets.ts BMR/floor, FoodScreen net-accounting, or burn-framing copy change. No data-model change (durationMin?/note? already optional, types.ts:397,404) -> no migration. No BC/Coach-prompt/network/dependency surface touched. Removed state sDur/sNote/actNote/eNote fully gone (grep-clean, tsc exit 0). Remaining "Duration (min)" inputs (:2033, :2087, :2512) are all non-strength.

## Notes
- [NOTE] progress.ts:106-107 — minutesOn (Progress "minutes moved" chart) now counts a manual strength session as 0 min (no durationMin). Safe, still counts in sessionsOn. Asymmetry: plan-logged strength keeps durationMin via syncStrengthLog (WorkoutScreen.tsx:792-801); only manual strength drops to 0. ED-neutral.
- [NOTE] WorkoutScreen.tsx:1171-1195 — opening+saving the edit sheet on a pre-existing strength entry wipes stored durationMin and re-derives burn from exercises; watch burns preserved. Consistent with intent.
- [NOTE] WorkoutScreen.tsx:1279 — an existing activity entry's note still shows in the detail line but is no longer editable/removable. Minor; strength notes render as title, unaffected.

## Not checked
- On-device visual render (read-only). Recommend manual check: manual strength with no weight -> "— KCAL"; with weight -> KCAL number, no "· MIN".

---
## 2026-07-02T00:00:00Z — Food serving-size recompute (manual edit sheet + Coach edit_food tool)

Audit of change adding parseQuantity/rescaleMacrosForQuantity (wren/lib/food.ts:317-402), FoodScreen live-update onEditQtyChange (wren/screens/FoodScreen.tsx:757-778, AMOUNT input ~:1520), EditFoodArgs + edit_food schema + one system-prompt bullet (wren/lib/coach.ts:148, :504-512, :626-643), and the edit_food handler (wren/screens/CoachScreen.tsx:547-611).

SUMMARY: Clean — 0 blockers, 0 warnings, 3 notes. Ship-safe.

FINDINGS:
1. ED-safety — PASS. consumedTotals summation, targets/BMR/floor logic, and food-framing tone all unchanged. New coach.ts:148 bullet is purely mechanical (correct-an-already-logged-food -> edit_food, never creates one); no earn/burn/compensate/judgment framing, no medical-advice weakening. Adjacent bullets :147/:149 intact. ED_SAFETY_RULES (coach.ts:107-110) verbatim intact: BMR floor / no fasting-purging-earning-compensating, no medical advice + provider pointer, no supplement-by-name, no body-shame. PROACTIVE_ED_SAFETY intact.
2. Recompute correctness — PASS. rescaleMacrosForQuantity cannot NaN/negative (finite-or-null parse, num<=0 guard, clampRound Math.max(0,round)). Null path leaves macros untouched at both call sites (never zeroed). per100g-grams and numeric-ratio paths both proportional/correct. Fraction alt matched before decimal alt so "1/2"->0.5. Unit mismatch -> null (no cross-unit scaling). FoodScreen live-update rescales from ORIGINAL editEntry (non-compounding); eFi blank-vs-0 preserved (gated on editEntry.fiber typeof number).
3. edit_food confabulation — PASS. No-match returns a message and creates nothing. Only mutates existing entry (edited spreads ...match; updateEntry maps by id). Today-scoped via entriesFor(...,today) + preserved match.date; cannot touch other entries/days. Returns REAL post-write state (consumedTotals(next,today)); Coach can't claim an unmade edit. edit_food wired into live FOOD_TOOLS (coach.ts:606) sent at :942 — same guarded path + system prompt as other food tools.
4. Scope/drift — PASS. Exactly 4 files; EditFoodArgs additive; no FoodEntry/Profile/DayLog/ChatMessage shape change (no migration needed); no new dependency; rescale shared so manual/Coach can't drift.

NOTES:
- CoachScreen.tsx:558-561 bidirectional substring fallback matcher could edit an unintended entry when no exact name match exists (e.g. "chicken" vs "chicken salad"); non-destructive, most-recent-wins, echoes matched name back. Fine unless edit_food later gains delete.
- CoachScreen.tsx:554-607 match read from profileRef snapshot, write via updateProfile keyed by id — stale snapshot harmless except a mid-call concurrent deletion (single-user, effectively impossible) where return string would still say "Updated…". Theoretical.
- food.ts:360 rescale floors at 0 with no ceiling; "999999 g" would scale absurdly. Matches manual-entry behavior, user-driven/visible/editable, not an ED fabrication path.

NOT CHECKED: on-device visual rendering of live-updating fields + persistence of a hand-override through Save (needs device run); tsc (implementer reported exit 0, cannot run).

---
## 2026-07-02T00:00:00Z — Food serving-size follow-up (tiny-kcal fix + edit_food matcher tightening)

## Summary
Clean — 0 blockers, 0 warnings, 3 notes. Grams-resolution rewrite correct on every walked case; rescale-derived label stays truthful; edit_food matcher can no longer destructively hit the wrong food.

### 1. Grams resolution correctness — PASS
Traced food.ts:400-485 with entry {"1 serving (150 g)", 225 kcal, per100g.cal 150}:
- "2 servings" -> gramsPerServing 150, grams 300, 450 kcal, label "2 servings (300 g)". Correct.
- "1 serving (300 g)" -> count unchanged, parsed.grams 300 honored -> 450. Correct.
- "1.5 servings" -> grams 225 -> 337.5->338. Correct.
- "2 g" -> GRAM_UNITS path, grams 2 outright (no paren = not a serving edit) -> 3 kcal; label "2 g". Correct (tiny-kcal bug fixed).
- Non-regression: "150 g"->"300 g" x2; "2 eggs"->"3" path(c) r=1.5; "1/2 cup"->"1 cup" x2. All correct.
- clampRound = max(0,round); parsed.num<=0 -> null; fraction guards /0. No NaN/negative.
- Divide-by-zero safe: oldCount>=1, gramsPerServing gated on old.grams!=null; zero/absent paren fails grams>0 filter -> falls back to count ratio, no false "(… g)".
- Null path returns null; both callers leave macros untouched, never zero.

### 2. Label truthfulness — PASS
servingLabel() recomputes parenthetical from resolved grams; saveEdit (FoodScreen.tsx:791) and edit_food (CoachScreen.tsx:621) persist it. Deterministic from original entry + new text; cannot compound.

### 3. edit_food matcher — PASS
CoachScreen.tsx:560-594: exact-name first; else all query words as whole words + query must include head noun. "latte"->"oat milk latte" yes; "chicken" not "chicken salad". 2+ distinct -> asks, mutates nothing. No match -> creates nothing. Same-food-multiple -> most recent. Today-scoped; reports real post-write totals.

### 4. ED-safety / scope — PASS
Summation, targets, BMR, MIN_DAILY_CALORIES untouched; Coach copy neutral. No data-model/dependency change. {macros,label} reaches exactly 3 call sites (FoodScreen 771/791, CoachScreen 617); coach.ts:503 is a comment. No stale Macros|null consumer.

## Notes
- [NOTE] CoachScreen.tsx:607-623 — gaveMacros branch keeps old label when model omits quantity; label can lag new macros (model-dependent, pre-existing).
- [NOTE] FoodScreen.tsx:784-801 — hand-overriding a macro after live rescale persists normalized grams label alongside overridden macros (explicit user override, accepted).
- [NOTE] Head-noun rule biases to false negatives ("not found") over wrong edits — correct safety posture.

---
## 2026-07-02T00:00:00Z — Workout PLAN completion semantics (strength day completed = ALL exercises crossed off, was any-one)

## Summary
1 WARN, no blockers. Completion-semantics change is correct and orphan-gating is sound; one ED-safety/accuracy ripple where partially-trained days now feed the Coach as "Skipped."

## Findings

- [WARN] wren/lib/plan.ts:671-708 — weekReviewSummary now classifies a partially-checked strength day as skipped (skipped = training.filter(d => !dayComplete(d))) and emits its name into Coach context as "Skipped: <weekday> <title>". Previously a day with a loggedEntryId counted as done; now a day where she did 4 of 5 lifts reads as a full skip. Risks: (a) Coach can tell her she "skipped" a day she actually trained — factually wrong; (b) trips the skipped.length >= ceil(training.length/2) "she missed a lot / consistent backing off costs her goal" branch (:704) more easily, so a hard-training week can trigger the honest-but-pushy script. WARN not BLOCK because guardrail copy is well-worded (coach.ts:1681, patterns.ts:213: honest-not-shaming, never moralize about food/body), but the INPUT DATA is now inaccurate on an anti-pushover surface. Fix direction: add a "partially completed (N of M exercises)" bucket rather than folding partial days into "Skipped." Missed-streak correctly left on dayLogged (patterns.ts:115).

- [PASS] Correctness — dayComplete (plan.ts:581-588) correct for all kinds. toggleAllExercises (WorkoutScreen.tsx:855-874) re-finds day in latest profile, sets all done=target, routes through syncStrengthLog; clear-all → removeWorkout + clears loggedEntryId; check-all → creates/updates grouped entry. No desync. Partial check still logs subset.

- [PASS] Edit-gating / orphan risk — Orphan risk real and confirmed: saveDayEdit (:945-1044) rebuilds day without loggedEntryId/done and never calls removeWorkout. Gating on logged=dayLogged (!!loggedEntryId), so edit disabled the instant any exercise is checked. Both openDayEdit sites safe (footer only under !logged; rest-day pencil never logged). No partial-day edit path.

- [PASS] Sum invariant — done and skipped are exact complements over training; done+skipped===total. No double-count.

- [PASS] Scope / drift — Only the four named files. No Coach prompt / Anthropic / data-model / calorie / BMR / dependency change. dayLogged retained for trained-dot + missed-streak; dayComplete used only for checkmark/session-count/praise.

## Not checked
- No tsc run in audit (read-only); relying on implementer's reported exit 0.
- Header-checkmark toggle + footer swap need a device test.

---
## 2026-07-02T00:00:00Z — Workout week-review: partially-trained strength days get their own "Partially completed" bucket (plan.ts weekReviewSummary)

### Summary
Clean — 0 blockers, 0 warnings, 2 notes. The 3-way partition is correct, the anti-pushover branch now keys off true no-shows only (strictly less pushy), no safety copy was reworded, and patterns.ts genuinely needs no change.

### Findings
- [PASS] Partition correctness (plan.ts:680-682, :694): done/partial/skipped is a provable exact cover of training — done = dayComplete; among !dayComplete, split on dayLogged (partial) vs !dayLogged (skipped). done+partial+skipped === total, mutually exclusive. N/M = checked-of-total, strength-guarded (:692), no division so no NaN.
- [PASS] ED-safety / anti-pushover (plan.ts:726, :724): "she missed a lot" threshold now keys off true no-shows only -> strictly less pushy; partials can no longer over-trip it. Praise branch still requires done===total. New "Partially completed (did N of M exercises)" line is neutral/factual. Guardrail copy intact.
- [PASS] Coach-context integrity (plan.ts:687-698): line well-formed; activity/class can never be partial (dayLogged===dayComplete for non-strength); cycle-off gate untouched; signature unchanged.
- [PASS] Scope/drift: only weekReviewSummary changed; no Coach prompt, Anthropic, data-model, calorie/BMR, or dependency change. patterns.ts correct as-is.

### Notes
- [NOTE] plan.ts:694 — strength day with loggedEntryId but 0 exercises would read "did 0 of 0 exercises"; not reachable in practice. Cosmetic.
- [NOTE] plan.ts:726 — PRE-EXISTING (not introduced here): when training.length===0, `0 >= Math.ceil(0/2)` is true, so "She missed a lot this week" fires on an all-rest week. Miss branch lacks the `training.length > 0` guard the praise branch has. Add a guard next time this function is touched.

---
## 2026-07-02T00:00:00Z — Add "plan" workout source (Coach-logged reads "From Coach")

## Summary
Clean — 0 blockers, 0 warnings, 2 notes. The union widen, storage migration, and 3-way display label are correct, additive, and scoped.

1. Migration correctness/safety — PASS. storage.ts:103-121 reclassifies "coach"->"plan" only when e.source==="coach" && planEntryIds.has(e.id); planEntryIds built from PlanDay.loggedEntryId across plan.current+history. Chat-logs never write loggedEntryId (CoachScreen log_workout uses makeWorkout(...,"coach")+addWorkout), so never flipped. Doesn't touch "manual". Crash-safe (try/catch, plan/history/days guards). Idempotent. Only e.source written; no entry add/remove. Composes safely after densify.
2. Data-model widen — PASS. types.ts additive ("manual"|"coach"|"plan"); old "coach" entries valid. Only WorkoutEntry.source readers are the display + migration; other .source hits are FoodEntry/BodyCompEntry (different types). No miscategorization on "plan".
3. Display correctness — PASS. Exhaustive 3-way with else catch-all. New srcPillCoach/srcTextCoach = colors.handle camel bg + colors.ink text; brand tokens, no purple, high contrast; shares base geometry, no collision.
4. Scope / ED-safety — PASS. Cosmetic only; burnSource/calorie math intact; only 5 files touched; CoachScreen log_workout correctly left "coach".

## Notes
- NOTE storage.ts:98-102 — best-effort limit: a plan check-off whose week rotated out of current+history stays "coach", reads "From Coach". Acceptable per design; new entries tagged at creation.
- NOTE WorkoutScreen.tsx:3468 — pill text 8.5px, consistent with existing plan/manual pills; small end of legible, future visual review only.

---
## 2026-07-02T00:00:00Z — WorkoutScreen generatePlan preserves current week number + startDate on regenerate

### Summary
PASS on correctness, data-loss, and scope; 1 WARN (pre-existing orphan made marginally worse by preserving startDate) and 1 NOTE. No BLOCK.

### Findings
- [PASS] Correctness (WorkoutScreen.tsx:605-609) — First-time (no plan.current) → {weekNumber:1}, today-anchored (unchanged). Existing plan → carries weekNumber + startDate. dateForWeekday anchors on startISO's own weekday (not assumed Monday), bijective over the 7-day window, so preserving startDate keeps every day on the same calendar dates. No NaN/undefined: weekNumber/startDate required; corrupt plan degrades via ?? 1 / ?? today. No throw.
- [PASS] No data loss / concurrency (WorkoutScreen.tsx:612-615) — history preserved via p.plan?.history ?? [] against LATEST p in functional persist updater. Outgoing current week intentionally REPLACED not archived (archive happens only in acceptReview). Not a regression.
- [PASS] Scope — Only generatePlan changed. Rollover path untouched. No calorie/burn/BMR/target/food/Coach-prompt/data-model change. GenerateWeekOptions already had weekNumber?/startDate?.
- [WARN] WorkoutScreen.tsx:612-615 + plan.ts:749 — Orphaning prior check-off WorkoutEntries on regenerate is PRE-EXISTING. Preserving startDate makes it marginally worse: the new week now covers the SAME dates as lingering entries, so re-checking a day whose date already has an orphaned entry appends a SECOND WorkoutEntry via commitDay (no same-date de-dup) → double-counted workout burn on that date. Old today-anchored/week-1 behavior made this unreachable. Low probability, mildly ED-adjacent (inflated calories-out). Fix direction: remove outgoing week's logged entries on regenerate, or de-dup same-date in commitDay. Not a ship blocker.
- [NOTE] types.ts:487 — WeekPlan.startDate comment says "Monday" but anchoring is any-weekday now. Stale comment, not a bug.

---
## 2026-07-02T00:00:00Z — Workout plan Regenerate preserves already-logged days (generatePlan merge)

### Summary
Clean — 0 blockers, 0 warnings, 1 note. PASS on all four focus areas; closes the prior WARN (regenerate orphaning logged entries + same-date double-count).

### Findings
- [PASS] Data preservation (primary) — WorkoutScreen.tsx:629-637. Merge iterates wk.days; wk is densified (coach.ts:1892 -> densifyWeek, plan.ts:114-134 all 7 weekdays), so existing.days.find(weekday==) resolves for every old day — no logged old day dropped. `od && dayLogged(od) ? od : nd` returns od regardless of nd.kind (logged old day wins even if new week made that weekday rest). Persist updater (p)=>({...p,plan:{...}}) preserves p.workoutLogs + history — no WorkoutEntry rows deleted.
- [PASS] Double-count resolved — :632-634. Preserved logged day keeps loggedEntryId, so re-check hits update-in-place (syncStrengthLog :836 / commitDay :777), not addWorkout. Prior same-date duplicate-append unreachable via regenerate.
- [PASS] Correctness — First-time (no existing) -> persistedWeek = wk, unchanged. {...wk,days} keeps new week's weekNumber/startDate/programName (carried from existing via genOpts -> same calendar week, not rollover). .find miss -> nd. Preserved loggedEntryId points at live non-orphaned entry.
- [PASS] Scope/ED-safety — Behavior-only in generatePlan. No calorie/burn/BMR/food/Coach-prompt/data-model change. Review/rollover untouched. Shared old-day object refs safe (mutators immutable via map+spread). BC-gated loading line not regressed.
- [NOTE] :605,629-643 — Pre-existing (NOT new) snapshot-vs-latest race: existing read from render-time profile snapshot while persist replaces latest p.plan.current; a Coach plan-edit landing during the ~5-15s generation would be clobbered. Strictly narrower than pre-fix behavior; WorkoutEntry rows always survive in p.workoutLogs. Low likelihood (setup modal open blocks accordion). Flagged for future Coach-concurrency consideration.

---
## 2026-07-02T00:00:00Z — Edit-any-time plan day + saveDayEdit log reconciliation (WorkoutScreen.tsx)

### Summary
Clean — 0 blockers, 0 warnings, 3 notes. Reconciliation is orphan-free, duplicate-free, and atomic across every transition; done-flag preservation and burn updates correct; scope contained to WorkoutScreen.tsx with no ED-safety, data-model, or Coach-prompt impact.

### Findings
- Focus 1 — No orphan: PASS. strength->strength updates same entry in place (or removes + clears loggedEntryId when zero checked, :866-869); activity->activity removeWorkout-then-commitDay (:1131-1135); kind change removeWorkout-then-writeDay unlogged (:1142-1143); un-logged plain writeDay. Missing-entry case re-creates gracefully.
- Focus 2 — No duplicate/double-count: PASS. strength reuses id via updateWorkout (workouts.ts:177-179); activity clears loggedEntryId to undefined before single commitDay add; kind change adds none. Every add-path removes-first.
- Focus 3 — Done-flag preservation + burn: PASS. prevDone snapshotted from latest profile, matched by stable id; new rows start un-done; kind-change-from-non-strength = fresh. syncStrengthLog recomputes exercises/duration/estBurn from edited rows while preserving burnSource==="watch" totals.
- Focus 4 — Atomicity: PASS. All four branches return one composed Profile inside a single persist/updateProfile updater; prev read from latest p; writeDay re-reads plan.current; remove/add/updateWorkout touch only workoutLogs; setup/history/concurrent writes preserved.
- Focus 5 — Scope/ED-safety: PASS. Only WorkoutScreen.tsx; no burn/BMR/target/food/Coach-prompt/data-model/dependency change; review-rollover untouched; footer copy neutral.
- New risk (editing a dayComplete day): PASS. weekProgress + weekReviewSummary recompute from live state, no cached counts.
- [NOTE] :3634 styles.editNote now unused (dead style, harmless).
- [NOTE] :947-954/1019-1033 pre-existing last-write-wins: edit sheet rebuilds exercise set from open-time edRows; concurrent Coach add on same open day could be dropped on Save; exposure widens now logged days are editable. Low risk (single-user).
- [NOTE] :1104/1132/1142 + workouts.ts:183-188 orphan-freedom relies on invariant that a loggedEntryId entry lives at dateForWeekday(startDate, weekday); holds in scope.

---
## 2026-07-02 — Cycle: stop fabricating a menstrual period past the projected start (grace-window + "Period expected")

**Files reviewed:** `wren/lib/cycle.ts`, `wren/screens/CycleScreen.tsx` (edited); consumers cross-checked read-only: `WorkoutScreen.tsx`, `lib/coach.ts`, `CoachScreen.tsx`, `ProgressScreen.tsx`, `FoodScreen.tsx`, `SettingsScreen.tsx`, `components/PhaseRing.tsx`.

**Summary:** Clean on safety — net ED-safety improvement. 0 blockers, 1 warning, 3 notes.

**Findings:**
- [PASS] ED-safety core fix sound — `cycle.ts` ongoing branch no longer wraps; can no longer fabricate a menstrual day. Holds `luteal` + `predictedLate`/`daysLate` in a grace window (max(7, len/2)), then `unknown`. No new bodily-state assertion or feelings-override framing. PHASE_GUIDANCE and disclaimer untouched.
- [PASS] New ring copy neutral/factual; branch order correct (logged bleed > predictedLate > forward prediction), so a late user never wrongly reads "Period in ~27 days".
- [PASS] BC / off / no-data branches undisturbed (gates at showCycleUI, "+" button, disclaimer).
- [PASS] Consumer contract intact — no new/invalid `phase` string; all consumers null-safe on the now-reachable `dayOfCycle: null`; PhaseRing ignores the phase prop and clamps cycleDay.
- [WARN] Beyond-grace ongoing state returns `{phase:"unknown", dayOfCycle:null}` for a history-having user; showCycleUI stays true so the hero ring rendered "CYCLE DAY 1 · unknown". Safer than fabricated-menstrual but confusing hero copy. Direction: route beyond-grace to a neutral status block. (RESOLVED by follow-up pass below.)
- [NOTE] coach.ts late-luteal reads "cycle day 31 of ~28"; beyond-grace reads "cycle phase unknown (no period logged yet)" for a user with logged periods. Odd but safe/non-fabricating; coach.ts read-only this session.
- [NOTE] `daysLate >= 1` fallback in ringPrediction is dead (always ≥1 in that branch); harmless.
- [NOTE] Scope: PhaseInfo lives in cycle.ts so no types.ts edit needed; no dependency/SDK drift. No git — two-file scope verified by inspection only.

**Not checked:** filesystem diff (no git); on-device rendering; auditor could not run tsc (implementer ran it: clean).

---
## 2026-07-02 — CycleScreen very-late hero + predicted-period restyle + ring copy trim

**Summary:** Not clean but not blocking: 1 WARN, 3 NOTE. ED-safety copy sound, precedence correct, no scope/data/dependency drift detected.

**Findings:**
- [WARN] Collapsed-strip footer rendered `PHASE_TITLE[phase.phase]?.toLowerCase() ?? phase.phase`; `PHASE_TITLE["unknown"]` undefined → leaked the raw token "unknown" under the new "Period expected" hero. Pre-existing behavior exposed by this change. Fix: neutralize the fallback. (RESOLVED by follow-up pass below.)
- [NOTE] `ringPrediction` computed unconditionally but only consumed in the ring branch; harmless.
- [NOTE] Reachability of the `unknown` hero verified safe: with history, currentPhase(today) reaches "unknown" only via the beyond-grace branch; no-data users fall to the status card. No regression hiding the ring for a normal user.
- [NOTE] Scope not diff-verifiable (no git); all tokens/symbols used by new lateHero* styles pre-exist; PhaseRing prop signature unchanged.

**Verified clean:** hero copy "Period expected" / "Log it when it starts and Wren re-syncs your cycle." neutral/factual; dropping "~N days later than your average" a net ED improvement; predicted-period restyle (ember wash 0.2 + hairline ember border + cocoa ink) clearly distinct from logged (solid ember + white); precedence logged > predicted > phase-tint holds in strip + grid, today's cocoa tile overrides all, border suppressed on today; BC/off gates untouched.

**Not checked:** on-device rendering/contrast; tsc (implementer ran it: clean).

---
## 2026-07-02 — CycleScreen footer leak fix (A) + cycle.ts forecast restore w/ preserved today-assertion guard (B)

**Summary:** No blockers. Core ED-safety invariant holds. 1 WARN (FoodScreen cross-surface future-eyebrow), 2 NOTEs, rest PASS.

**Findings:**
- [PASS] Crux invariant holds. `currentPhase(p)` = `phaseForDate(p, today, today)` → future gate `iso > toISODate(today)` is X>X = false always → forecast-wrap branch unreachable via currentPhase. All today-consumers (Workout, Coach, Progress, Settings, coach.ts) get luteal-hold/predictedLate or unknown when overdue, never a fabricated menstrual.
- [PASS] Forecast alignment correct, no boundary off-by-one: projDay===1 (menstrual) iff iso === a predicted start (isPredictedPeriodDay off===0), so the first forecast-menstrual day renders as the ember predicted outline, not an asserted bleed. isPredicted precedes phase tint in all render paths.
- [WARN — OPEN, routed] FoodScreen.tsx:165,172-173 — Food eyebrow can print a bare forecast "MENSTRUAL" for a far-future date (Food navigates to today+28; future cells hit the restored forecast wrap; eyebrow gate only skips unknown/off/BC). Not a present-tense assertion; pre-existing behavior restored with the forecast. Food lacks the calendar's predicted-vs-asserted distinction. FoodScreen owned by the other session — routed to Ting, not fixed here.
- [NOTE] Forecast-menstrual width (day<=5*len/28) can differ from predicted-period outline width (observedPeriodLength); short-bleed users get projDay 3-5 forecast days as washed menstrual pastel, not ember outline. Cosmetic; never a committed cell.
- [PASS] Change A resolves the raw-"unknown" footer leak, no layout shift; known phases still lowercase (ovulatory→"ovulation").
- [PASS] Contract additive/unchanged; completed-cycle branch unchanged; BC + off branches short-circuit before date math; YYYY-MM-DD string compare safe.

**Not checked:** git-diff scope confirmation (no git); visual rendering; auditor could not run tsc (implementer ran it: clean).

---
## 2026-07-02 — Cycle daily coach-quote (data-reflection composeDailyMessage + recurringLogsAroundDay)

**Context:** Ting explicitly chose the data-reflection version this session — the 2026-05-29 drop of the "advice based on how you feel" frame STANDS. Message reflects what she logged, stated as fact, + practical tip. Surface: Cycle screen quote; Coach-chat context line drafted and routed to the other session (coach.ts read-only here).

**Summary:** Clean on ED-safety copy and the data-reflection boundary; 1 WARN (pattern-count phrasing), 2 NOTEs. No BLOCKers.

**Findings:**
- [WARN] "your last N cycles" phrasing used the distinct-cycle count as if recency/consecutiveness were guaranteed; case 1 also applied one match count to a 2-item reflected set. (RESOLVED — see re-verify below.)
- [NOTE] composeDailyMessage runs per render → recomputes cycleStarts etc. O(n log n) over dayLogs; fine at dogfood scale, memoize if logs grow.
- [NOTE] Case-2 lines can surface mood chips awkwardly ("logged irritable"); factual and ED-safe, cosmetic only.

**Verified clean:** all 9 phase-fallback variants, 4 phase tips, 10 symptom tips + low-energy tip, and both dynamic templates flat declarative — no "many women", no feelings-override, no moralizing/norm-comparison, no calorie/deficit/burn/earn, no restrictive food advice; Cravings tip neutral-supportive; "Be soft with yourself" preserved on menstrual+luteal. recurringLogsAroundDay: completed-cycles-only (ongoing excluded → today can't self-count as pattern), window clamped [1,length], distinct-cycle Set counting, BC/off/<2 starts → []. predictedLate suppresses pattern branches; unknown keeps ""; deterministic. PHASE_GUIDANCE removal orphaned nothing (0 refs). Scope: only cycle.ts + CycleScreen.tsx.

### Re-verify — count-accuracy fix: PASS
Prior WARN resolved: case 1 uses MIN match count across recurring reflected items (0 suppresses clause); "last cycle" gated strictly on cyclesTracked===1; case-2 mention filter (r.cycles===top) keeps the stated number true for every named item. Templates: "— that matches your last cycle around this point" / "— you've logged this in a past cycle around this point" / "— you've logged this in {N} of your recent cycles"; case 2: "Around this point last cycle you logged {items}." / "…in a past cycle…" / "Around this point you've logged {items} in {top} of your recent cycles."
- [NOTE — residual, accepted] A never-recurred co-logged item can be named in the lead while the count reflects only the recurring item; singular "this" keeps it defensible. Strict fix direction recorded: force clause off when a named item has count 0.
tsc clean (implementer-run) after every pass.

---
## 2026-07-02T00:00:00Z — FoodScreen future-day cycle-phase "(expected)" qualifier

### Summary
Clean on all three focus areas — future-day fix correct, today path guarded — with 1 WARN for a cross-session dependency to track while cycle.ts is being edited.

### Findings
- [PASS] #1 Future-day fix correctness. FoodScreen.tsx:157 isFutureDay = selDate > today is a correct today-boundary (both YYYY-MM-DD ISO strings; monotonic lexicographic compare). Strict > makes today and past both false → (expected) qualifier is future-only, cannot leak onto today/past. ED gate at :175 intact: suppresses phase for BC, tracking-off ("off"), unknown. No cycle leak when tracking off.
- [PASS] #2 Today path IS guarded; guard lives in phaseForDate, not currentPhase. "Never fabricate menstruation for today" guard is inside phaseForDate (cycle.ts:220-238), keyed on the today param (if iso > toISODate(today)). currentPhase is a thin delegate adding no safety of its own. FoodScreen calls phaseForDate(profile, selDt) omitting the 3rd arg → today defaults to new Date() → today's cell takes the TODAY/PAST branch (holds luteal+predictedLate or unknown), never wrapping to a fabricated menstrual. A bare MENSTRUAL on today can only come from a logged period start (confirmed data). No ED gap.
- [WARN] FoodScreen.tsx:165 + cycle.ts:220-238 — Cross-session dependency. FoodScreen's today-safety depends on the fabricated-menstrual guard staying physically inside phaseForDate (keyed on the today param). If the concurrent cycle.ts edit relocates that guard into currentPhase, FoodScreen (which never calls currentPhase) would silently regress to asserting a fabricated MENSTRUAL on today's eyebrow. After the cycle.ts edit lands, re-verify the guard remains in phaseForDate, or route FoodScreen's today cell through currentPhase. cycle.ts was in-flux at audit time.
- [PASS] #3 Scope / ED-safety. Change confined to eyebrow lines :170-176. No calorie/macro/target/BMR/Coach/data-model touch; PhaseInfo shape unchanged. Eyebrow still composed from real values only. No new cycle leak.

---
## 2026-07-02T00:00:00Z — weekReviewSummary cross-references actual workout log (wren/lib/plan.ts:29,725-787)

**Summary:** 1 WARN, 3 NOTE — no blockers. New text is neutral/factual, load math is sound, no guardrail line altered. One ED-adjacency: the change surfaces extra/unplanned training volume into the plan-generation context, and PLAN_SYSTEM has no guardrail to keep the Coach from celebrating/normalizing over-exercise.

**Findings**
- [WARN] coach.ts:1679-1681 (with plan.ts:738-754) — Asymmetric volume guardrail. New "Also trained outside the plan: …" feeds recentSummary → planContext → PLAN_SYSTEM, and whyThisWeek is coaching she reads. PLAN_SYSTEM pushes back only on skipping/backing-off; no counterpart for excessive volume / inadequate rest. Over-exercise is an ED behavior; surfacing "…+6 more" opens a praise/normalization channel. ED_SAFETY_RULES bars only food-compensatory exercise, not grinding volume. Line framing itself is correct (neutral "also trained," no earn/burn, no calories-burned) so not ship-blocking. Fix direction: add an over-training-awareness clause to PLAN_SYSTEM mirroring the skip-pushback clause.
- [NOTE] plan.ts:706,730 — Function trusts week.days to hold 7 distinct weekdays (densify invariant holds in practice; same assumption as pre-existing partition). No defensive re-densify.
- [NOTE] plan.ts:744,777 — Items count-capped (4/8) but not length-capped; free-text names could bloat the ~650-char summary. Low risk.
- [NOTE] plan.ts:782-787 — "Progress from the actual loads…" consistent with PLAN_SYSTEM "progress them" (coach.ts:1673); no contradiction.

**Focus verdicts:** (1) ED-safety — WARN (framing correct, no burn totals; over-training normalization gap is the concern; guardrail lines plan.ts:783 and :789-790 verbatim). (2) Correctness — PASS (partial/complete days' loggedEntryId in plannedEntryIds → no plan leak into unplanned; top-weight skips null weight/no NaN/reps-guarded; unplanned vs loads disjoint; empty logs omit lines). (3) Prompt hygiene — PASS. (4) Scope — PASS (only plan.ts; workoutsFor already in import graph; pure read).

---
## 2026-07-02T00:00:00Z — Coach PLAN_SYSTEM asymmetric-volume guardrail bullet (closes prior over-exercise WARN)

## Summary
Clean — 0 blockers, 0 warnings, 1 note. The change closes the prior WARN and introduces no safety, scope, or drift risk.

## Findings
- [PASS] Prompt integrity — coach.ts:106-111 ED_SAFETY_RULES and :120-124 PROACTIVE_ED_SAFETY intact with every hard rule present (BMR floor, no starving/purging/fasting/earning food, no medical advice, no supplement-by-name, no body-shaming/moralizing, care-mode with NEDA + 988). Skip-pushback bullet :1681 unchanged. New bullet :1682 purely additive, final bullet of "HONEST AND GOAL-ORIENTED (critical)", before OUTPUT. ${ED_SAFETY_RULES} still terminates the prompt. No code logic changed (planContext, generateWeekPlan fetch, model OPUS, max_tokens 4000 all as expected).
- [PASS] Clause quality / ED-safety — Closes the WARN: extra volume explicitly not praiseable, recovery protected ("keep rest days real"), non-punitive framing ("recovery you are protecting, not a slowdown"). No backfire — "do not praise" targets the coach's framing, not her; nothing tells her to stop training or moralizes. Flat declarative voice, no "many women find," no medical claim.
- [PASS] Scope — Single additive string bullet in PLAN_SYSTEM only. No dependency/data-model/Anthropic-call/network change.
- [NOTE] :1681-1682 — Skip-pushback (under-training) and over-volume (over-training) bullets key on opposite conditions; a mixed week could satisfy both but prescriptions don't contradict (make rest real vs. offer smallest real session). Can co-fire, but not confusingly.

## Not checked
- Runtime generate_plan output (prompt-reasoning judgment only). Suggested device test: synthetic "trained 7/7, all unplanned extras" recent-summary → confirm whyThisWeek frames protected recovery, not praise/slowdown.

---
## 2026-07-02T00:00:00Z — Coach prompt: logged-lists-are-authoritative bullet (coach.ts:171)

### Summary
1 WARN (day-boundary false-deletion phrasing), otherwise clean; prompt integrity and scope PASS.

### Findings
- [PASS] Prompt integrity — coach.ts:106-198. New bullet purely additive at :171, between the :170 plan rule and LONG-TERM MEMORY (:173). ED_SAFETY_RULES, PROACTIVE_ED_SAFETY, SYSTEM_PROMPT body, and the :170 rule intact; NEDA + 988 + BMR-floor language unchanged. No code logic touched. Caveat: no git — read-confirmation, not byte-diff.
- [PASS] Fixes failure mode without total conflict — consumed totals and foodLines rebuilt per turn from entriesFor, so deleted items are already gone from both list and summed totals. "don't count it in any total" reinforces the existing code-summed-totals rule (:146); no contradiction.
- [PASS] Inverse case safe — bullet speaks only to disappearance; a user-added item mid-convo just appears in next turn's foodLines, never misread as deleted.
- [WARN] Day-boundary false-positive — :171 with :48 (history spans days) and today-only foodLines/workoutLine. An item the Coach logged on a prior day legitimately drops off today's list; the bullet's absolute "no longer appears there, she deleted it" could produce a phantom "you deleted that" claim on a new day. Mildly ED-adjacent. Fix direction: scope the trigger to "earlier today". Prompt-only.
- [NOTE] Acknowledgment neutrality holds; surrounding prompt discourages interrogation, but the bullet doesn't itself forbid commenting on WHY she deleted a food. Add "don't ask why" clause if editing for the WARN.
- [PASS] Scope — prompt-only, additive. Non-today residual gap (past-dated Coach-logged deletions invisible; context is today-scoped) correctly out of scope; same root cause as the WARN.

---
## 2026-07-02T00:00:00Z — Coach deletion-awareness bullet tightening (coach.ts:171)

### Summary
Clean — 0 blockers, 0 warnings. Both prior findings (day-boundary WARN, neutrality NOTE) closed; no side effects; change isolated to the single bullet.

### Findings
- [PASS] Check 1 — day-boundary WARN closed. coach.ts:171 scopes the deletion inference to "a food or workout you logged EARLIER TODAY in this chat," and pre-empts the rollover false positive: "These lists cover TODAY only, so an item from a previous day dropping off is just the day changing — not a deletion." Deletion trigger bounded on both time (today) and source (Coach-logged this chat).
- [PASS] Check 2 — neutrality NOTE closed without over-reach. "acknowledge plainly that it's no longer in her log; don't ask why she removed it" bars only Coach probing; does not stop a supportive reply or her volunteering a reason. Neutral, non-compensatory framing.
- [PASS] Check 3 — change isolated. :170 (NEVER CLAIM A PLAN CHANGE) intact; :172 blank separator intact; :173 LONG-TERM MEMORY header + body (incl. ED-SAFETY memory rule) intact. Only :171 changed.
