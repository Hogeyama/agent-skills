---
name: distill-to-adr
description: Distill an implementation plan/result into an Architecture Decision Record (ADR) after the work is done. Use this whenever the user asks to "turn this plan into an ADR", "write an ADR for what we just did", "計画をADR化", or similar phrasing that signals a completed implementation whose decisions deserve a durable record. The skill produces a concise ADR focused on decisions and rationale.
---

# Distill plan to ADR

Implementation plans contain two kinds of content mixed together:

1. **Execution scaffolding** — todos, step ordering, file paths, test checklists. Valuable while implementing, worthless afterward. The code and git history answer these questions better than any document.
2. **Decisions and rationale** — architectural boundaries chosen, alternatives rejected, non-obvious constraints, invariants that must be preserved. Valuable forever. Not recoverable from code alone.

The job of this skill is to extract (2), discard (1), and leave behind a durable record that a teammate reading it in six months can actually use.

## When to run this skill

Only when the user explicitly asks for distillation after implementation is done. Typical triggers:

- "Turn this plan into an ADR"
- "ADR作って"
- "Write a decision record for what we just built"
- User points at a plan file and asks for a summary/decision doc

Do not run this skill proactively during planning or mid-implementation. The whole point is that distillation happens once the dust has settled and you know which decisions actually mattered.

## Inputs to gather

Before writing, make sure you have:

1. **The source plan** — file path or pasted content. If the user only gestured at it ("the plan we've been working from"), ask which file.
2. **What actually got implemented** — the plan and the final implementation may differ. Skim recent commits, the current state of the relevant files, or ask the user "did anything change vs the original plan?". Decisions that got reversed mid-implementation should be reflected in the ADR, not the stale plan.
3. **Where ADRs live in this repo** — check for `docs/adr/`, `docs/decisions/`, `adr/`, or similar. If none exists, ask the user where to put it (don't silently create a new convention). If one exists, follow its numbering and naming style.

## The distillation

### What to keep

- **The decision itself**, stated plainly. One or two sentences.
- **The reason it was chosen over alternatives.** This is the most valuable part of the whole document — it prevents the same debate from being re-litigated in six months.
- **Rejected alternatives** with brief reasons. Even one line each is enough. The existence of "we considered X and rejected it because Y" stops future contributors from proposing X again without new information.
- **Non-obvious constraints and invariants.** Things like "path must stay at X so mounts don't break", "cleanup order must be A then B", "this matching strategy uses both metadata and probes because neither alone is sufficient". These are the things that look arbitrary in code but have real reasons behind them.
- **Consequences** — what the decision forces elsewhere in the system (new APIs exposed, behaviors now guaranteed, things now possible or impossible).

### What to discard

- **Todo lists and step-by-step plans.** Git history and the current code are the answer.
- **File paths and module names as first-class structure.** Mention them inline where they clarify a decision, but don't organize the ADR around them.
- **Implementation order.** Nobody reading the ADR later cares what was built first.
- **Test plans.** The tests exist; they document themselves.
- **Hedging language from the planning phase** ("likely requires", "probably", "we might"). After implementation these are either facts or they didn't happen. State facts.
- **Restating what the code obviously does.** If a function's name and signature explain it, the ADR doesn't need to.

### Rule of thumb

A good distillation is roughly **30–60% the length of the source plan**. If the result is longer than ~70% of the original, you're still copying execution scaffolding. If it's under ~20%, you're probably dropping rationale that should be preserved.

## ADR format

Follow the existing convention in the repo if there is one. If not, use this default:

```markdown
# ADR NNNN: <Short decision-oriented title>

## Context
<2–5 sentences. What problem does this address? What constraints
or forces made a decision necessary? No narrative of how you got here —
just the situation that demanded a choice.>

## Decision
<The chosen approach, stated plainly. Include the structural boundaries
(what lives where, what does NOT live where) and any invariants the
decision establishes.>

## Consequences
<What this forces or enables elsewhere. New APIs exposed, behaviors
now guaranteed, paths preserved, ordering requirements, etc.
Bullet points are fine here.>

## Rejected alternatives
<Each alternative considered, with a one-line reason it was rejected.
This section is often the most valuable — skipping it is the most
common way an ADR loses long-term value.>
```

Title the ADR around the decision, not the feature. "Display runtime GC architecture" is better than "Add xpra cleanup". The reader wants to know "what did we decide" not "what did we build".

If the repo already has ADRs, match their numbering scheme (zero-padded sequential is typical: `0007-display-runtime-gc.md`). If there's no existing scheme, ask.

## Workflow

1. Read the source plan file.
2. Check git log / current code briefly to catch plan-vs-implementation drift.
3. Locate the ADR directory (or ask where to put one).
4. Draft the ADR following the format above.
5. Show the draft to the user before writing anything to disk. Explicitly call out anything you compressed heavily or omitted, so they can push back if something important was dropped.
6. On user approval: write the ADR file and report the path.

## Example

**Source plan (abbreviated):**

> ## Problem
> nas starts xpra-backed display sandboxes, but does not persist enough runtime metadata to GC orphaned xpra resources after crashes...
>
> ## Proposed approach
> 1. Introduce a display runtime registry...
> 2. Extend the display lifecycle so DisplayService.startXpra() records enough information...
> 3. Wire GC into the requested triggers...
> 4. Add tests around registry GC...
>
> ## Todos
> 1. **display-runtime-registry** — Add src/display/registry.ts ...
> 2. **display-service-bookkeeping** — Update src/services/display.ts ...
> 3. **ui-cleanup-hooks** — Update src/ui/data.ts ...
> 4. **xpra-gc-tests** — Add focused tests ...
>
> ## Notes
> - Preserve the current display runtime location...
> - Prefer killing the auto-attach client before the xpra server...
> - The GC sweep should be conservative about live sessions: metadata alone
>   is not enough, and probes alone are not enough...

**Distilled ADR:**

```markdown
# ADR 0007: Display runtime GC architecture

## Context
xpra-backed display sandboxes leak resources (server/attach processes,
X11 sockets, session dirs) on crashes or partial startup failures.
GC is needed at session startup, stopContainer, and cleanContainers.

## Decision
GC lives behind a display runtime registry, not in DisplayStage.
DisplayStage stays an orchestration boundary; filesystem and process
teardown belong to display-focused modules.

Session matching uses both persisted metadata and live probes
(PID liveness, socket/session-dir existence). Neither alone is
sufficient: metadata-only misses legacy/partial leftovers, and
probe-only risks killing healthy sessions we don't own.

## Consequences
- ProcessService.SpawnHandle exposes `pid` (previously hidden).
- Display runtime path stays at `$XDG_RUNTIME_DIR/nas/display/<sessionId>`
  (fallback `/tmp/nas-<uid>/...`) so mount behavior is unchanged.
- Cleanup order is fixed: attach client → xpra server → session dir.
  Already-exited processes are treated as benign.
- No manual GC command is exposed; GC is implicit in the three triggers.

## Rejected alternatives
- Putting teardown in DisplayStage: breaks the orchestration/implementation
  boundary and couples the stage to filesystem details.
- Metadata-only matching: cannot recover from pre-registry leftovers.
- Probe-only matching: unsafe against concurrent healthy sessions.
```

Note what disappeared: the four-item todo list, file paths as section structure, test enumeration, and hedging ("likely requires"). What stayed: the boundary decision, the "both" rationale, the preserved path, the cleanup order, and the rejected alternatives. Roughly 50% the length of the original.

## Things to watch for

- **Don't invent rationale.** If the plan says "do X" without saying why, and you don't know why either, ask the user rather than making up a justification. A confidently-wrong ADR is worse than a missing one.
- **Don't preserve the plan's structure just because it was there.** A plan organized around todos should become an ADR organized around decisions. Restructure freely.
- **If the plan turned out to be wrong.** Sometimes implementation revealed that the original plan was misguided and a different approach was taken. The ADR should describe what was actually built and why, not what was originally proposed. The original wrong direction can appear as a rejected alternative if it's instructive.
- **If there's genuinely nothing to distill.** Some plans are pure execution checklists with no real decisions (e.g. "rename these 12 variables"). In that case, tell the user honestly: "there's no architectural decision here worth an ADR — the git history covers this." Don't force an ADR just because one was requested.
