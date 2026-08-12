# Wren — orchestration rules for subagents

This repo has two layers: the Expo app code lives in `wren/` (with its own `wren/CLAUDE.md` and `wren/AGENTS.md`), and project-wide specs (`Wren_Project_Plan_v2.md`, `Wren_Prototype_Spec_FeatureA_Coach.md`, `Wren_Trademark_Knockout.md`) live at this root. `Trademark_Knockout_priorname_Flux.md` is the historical knockout for the prior name "Flux" — kept for reference, not active spec.

Four subagents live in `.claude/agents/`:

- **wren-designer** (Opus) — matches submitted mockups pixel-faithfully; the mockup is its spec, brand tokens are its vocabulary. Edits presentation only.
- **wren-implementer** (Opus) — writes and edits app code
- **wren-researcher** (Sonnet) — compares Wren against MyFitnessPal, Cal AI, Flow, Clue, Flo, Noom, Lifesum, Whoop, Oura on UX and feature mechanics
- **wren-auditor** (Opus, read-only) — flags safety, scope, drift, and ED-safety risks

## Delegation defaults

- Any task whose success is "make this screen match the mockup" → delegate to **wren-designer** (see the design-phase pipeline below).
- Any task that produces or changes code (behavior, logic, data, Coach, wiring) → delegate to **wren-implementer**.
- Any task that asks "how does X app do Y" or "what's the standard pattern for Z" → delegate to **wren-researcher**.
- Risk review, code audit, "is this safe" → delegate to **wren-auditor**.

## Design-phase pipeline (mockup → match → integrity → safety)

When Ting submits a mockup and wants a screen to match it, run the three agents in this fixed order:

1. **wren-designer** does whatever it takes to match the mockup. The mockup rules all aesthetic decisions — layout, spacing, color, type, hierarchy. The one thing it cannot override is ED-safety copy and its legibility; if the mockup conflicts with safety copy, the designer flags it rather than complying. The designer edits presentation only and hands any required behavior/wiring to the implementer.
2. **wren-implementer** then verifies the design pass didn't break the app — that wiring, navigation, state, data flow, and all processes still work, and completes any behavior the designer left as a TODO.
3. **wren-auditor** then runs the mandatory audit (below) to confirm nothing is broken and no safety/scope/contrast guardrail was crossed.

Do not skip step 2 or 3. A pixel-perfect screen that broke a handler or buried the disclaimer is not done.

## Mandatory audit after implementation

**After every change from wren-implementer or wren-designer, automatically invoke wren-auditor before reporting work as done.** This is not optional and not conditional on the size of the change. (In the design-phase pipeline the order is designer → implementer → auditor, with the audit running last.) The reason: the audit checklist covers ED-safety, birth-control branching, and Coach safety-prompt integrity — all of which can be silently broken by a one-line change.

The audit produces two outputs:
1. A report in chat
2. An `APPEND_TO .claude/audit-log.md:` block that should be appended to that file before reporting work complete

If the audit returns any **BLOCK** finding, do not report the implementation as done. Surface the blockers, ask Ting whether to address them, and only proceed after she decides.

## When auditor is not needed

- Pure research tasks from wren-researcher (no code changed)
- Conversational questions answered directly without delegating to implementer
- Reading files for context without modifying them
