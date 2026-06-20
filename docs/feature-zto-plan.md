# Feature Plan: Zero to One (`zto`)

Status: Proposal — not yet implemented
Branch: `feature/zto`
Owner: TBD
Last updated: 2026-06-20

---

## 1. Summary

**Zero to One (`zto`)** is a new top-level GSD command — `/gsd-zto` — that takes a project from "mapped but empty roadmap" to "rough first cut shipped" with a **single interview** at the start and **no further human intervention** until the build is done (or it must stop for safety).

The user provides one short spec — either typed inline or a path to a file — and after one focused interview, GSD autonomously walks through whatever milestones and phases are required to turn that spec into working code.

`zto` is explicitly for **rough cuts**: fast, opinionated, iterable. Polish and refinement happen in subsequent normal GSD cycles.

---

## 2. Motivation

Today, GSD's strength — its rigor (questioning, research, per-phase discussion, plan checking, verification, UAT) — is also its highest-friction surface for the "I just want to see this thing exist" moment:

- `/gsd-new-project` requires an open-ended conversation, then a roadmap approval, then a `/clear`, then per-phase `discuss → plan → execute → verify`, *for every phase*.
- For a 5-phase rough cut, that is **~15–25 separate slash commands** and several human approval gates.
- A solo developer with a clear vision for a v0 spends their best energy answering the same kinds of clarifying questions over and over instead of looking at running code.

`zto` collapses that to: **one command, one interview, one "go."**

The principle: *the most leveraged moment is the first interview, not the per-phase ones.* If we extract enough at the top, downstream phases can run on rails.

---

## 3. User Experience

### 3.1 Invocation

```
/gsd-zto                                 # interactive: prompts for spec inline
/gsd-zto "<inline spec text>"            # one-line spec
/gsd-zto path/to/spec.md                 # file path (any text format)
/gsd-zto --from STATE                    # resume an interrupted zto run
```

Argument resolution rule:
1. If the argument is an existing readable file path, load its contents as the spec.
2. Otherwise treat the argument as the spec itself.
3. If no argument, ask the user for the spec inline (one freeform question).

### 3.2 Preconditions

`zto` requires that the project has been mapped:

- **Greenfield** (`/gsd-new-project` already run): `.planning/PROJECT.md` exists.
- **Brownfield** (`/gsd-map-codebase` already run): `.planning/codebase/` exists.

If neither is present, `zto` refuses with a clear message and points the user at `/gsd-new-project` or `/gsd-map-codebase`. There is no implicit mapping inside `zto` — mapping is a separate, conscious step and we keep it that way.

### 3.3 The one interview

After loading the spec, `zto` runs a single, focused interview session designed to extract everything downstream agents will need so they never have to ask the user again. The interview has three parts:

1. **Vision lock** — confirm what the user wants to build (the spec, restated in GSD's voice) and what is explicitly out of scope.
2. **Gray-area resolution** — `zto` enumerates the gray areas it sees across the *entire* spec (not per phase): look-and-feel, key UX flows, data model assumptions, integrations, auth posture, deployment target, tech preferences. The user resolves each — or explicitly delegates to `zto` ("you decide, rough is fine").
3. **Autonomy contract** — the user confirms the autonomy bounds: secrets allowed, network/install allowed, destructive commands allowed, max wall-clock, max milestones, and which (if any) hard stop conditions apply.

The interview always ends with a single approval gate: **"Go."** Once approved, GSD runs to completion without further prompts unless a hard stop fires.

### 3.4 The "go" loop

After approval, `zto`:

1. Writes `.planning/ZTO.md` capturing the spec, decisions, and autonomy contract.
2. Generates `REQUIREMENTS.md` and `ROADMAP.md` directly from the interview (no second user approval).
3. For each phase in order:
   - Auto-runs the equivalent of `discuss-phase` using ZTO.md as the source of truth (no human questions).
   - Runs `plan-phase`, `execute-phase`, and `verify-work` in their existing forms but in **non-interactive ("zto") mode**.
   - On a passing verification, advances. On gaps, auto-runs `plan-phase --gaps` and re-executes — up to a per-phase gap-retry cap.
4. When the last phase verifies, marks the milestone complete.
5. If the autonomy contract allows multiple milestones and the spec implies more than one, starts the next milestone the same way and continues until the spec is satisfied.

### 3.5 Completion / interruption

At terminal states `zto` prints:

- **DONE** — milestone(s) complete, summary of what shipped, suggested follow-ups.
- **HARD STOP** — what triggered it, current state, and `/gsd-zto --resume` to continue after the user resolves it.
- **BUDGET EXHAUSTED** — wall-clock, retries, or token budget reached. Same `--resume` path.

---

## 4. Scope & non-goals

**In scope:**
- New command: `/gsd-zto`.
- New workflow file: `workflows/zto.md`.
- New template: `templates/zto.md` for `ZTO.md`.
- Reusing existing agents (`gsd-planner`, `gsd-executor`, `gsd-verifier`, `gsd-roadmapper`, etc.) with a "zto" mode flag rather than forking them.
- A small set of new orchestrator-only knobs in `config.json` (see §6.4).

**Out of scope (v1):**
- Replacing or deprecating any existing command. `zto` lives next to them.
- Polish-level UX work, deep research, or comprehensive UAT in the auto-loop. Those remain on the manual path.
- Multi-repo orchestration.
- Cloud/remote execution. `zto` runs in the user's Cursor session.
- Self-modifying the autonomy contract mid-run. Once set in the interview, it's fixed for the run.

---

## 5. Design

### 5.1 Where `zto` fits

`zto` is an orchestrator over the existing primitives, **not** a parallel system. The mental model:

```
existing manual path:
  /gsd-new-project  → /clear → /gsd-discuss-phase N → /gsd-plan-phase N
                    → /gsd-execute-phase N → /gsd-verify-work N → repeat

zto path:
  /gsd-zto          → one interview → ZTO.md → REQUIREMENTS.md → ROADMAP.md
                    → loop { discuss(auto) → plan → execute → verify → gaps? }
                    → milestone done → next milestone (if any) → DONE
```

Every box in the `zto` row is implemented by calling the existing workflow/agent with a `"mode": "zto"` flag in its prompt and config. We **do not** copy logic.

### 5.2 New artifacts

| Artifact | Purpose |
|---|---|
| `.planning/ZTO.md` | Source-of-truth for the run. Holds the spec, all interview decisions, the autonomy contract, and a state machine snapshot. |
| `.planning/zto/RUN.log` | Append-only event log for the run (phase started, verifier result, gap retry, hard stop). Cheap, human-readable. |
| `.planning/zto/CHECKPOINT.json` | Latest resumable state: current milestone, current phase, retry counters, budgets remaining. |

`REQUIREMENTS.md`, `ROADMAP.md`, `STATE.md`, and `phases/*` are the same artifacts the manual path produces — `zto` writes to them through the same agents.

### 5.3 Agent reuse vs. new agents

No new agents. Each existing agent that today branches on `mode: "interactive" | "yolo"` gets a third branch `mode: "zto"`. The differences in `zto` mode:

- `gsd-planner` — never asks the user; if it would, it picks the rough-cut default and notes the choice in the plan.
- `gsd-roadmapper` — auto-approves its own output; logs the proposed roadmap to RUN.log instead of asking.
- `gsd-verifier` — same logic, but `human_needed` collapses to `gaps_found` (humans are not in the loop).
- `gsd-executor` — `autonomous: true` on every generated plan; checkpoints are disabled.

This is a deliberate constraint: if an agent needs a code change for `zto` mode, the change must be a small switch, not a rewrite. If it cannot be a small switch, that is signal that we should redesign rather than fork.

### 5.4 The single interview, in detail

The interview is a new sub-workflow `workflows/zto-interview.md`. It is the only place `zto` ever blocks on the user. It must extract enough to (a) write a full REQUIREMENTS.md and ROADMAP.md *and* (b) supply per-phase CONTEXT.md content for every phase before any phase starts.

Interview structure:

1. **Restate the spec.** Read the spec (inline or file), summarize in 5–10 bullets, ask "Is this what you mean?" — only ask one follow-up clarification round.
2. **Detect gray areas across the whole spec.** Use the same gray-area taxonomy as `discuss-phase.md` but applied globally:
   - Users SEE → look-and-feel anchor (1 question), layout density (1), key state coverage (1)
   - Users CALL → response shape, errors, auth, versioning
   - Users RUN → I/O format, flags, exit codes
   - Data → entities, lifetimes, persistence
   - Integration → which third parties, with what credentials posture
   - Deployment → local-only vs. hosted, target runtime
   Only ask about categories the spec implies. Maximum **12 questions total**, hard cap, to keep the interview tight.
3. **Autonomy contract.** AskUserQuestion blocks for:
   - Network/install allowed? (yes/no)
   - Destructive shell allowed? (yes/no — defaults to no)
   - Secrets the run may use? (list or "none")
   - Wall-clock budget? (default: 2h)
   - Max gap-retries per phase? (default: 2)
   - Max milestones in this run? (default: 1)
   - Stop on first failing verifier? (default: no, try gap closure first)
4. **Final gate.** Present the synthesized ZTO.md inline; one AskUserQuestion: "Approve and go / Edit / Cancel."

After approval the interview never reopens within the same run.

### 5.5 The autonomous loop

Pseudocode in `workflows/zto.md`:

```
load ZTO.md, CHECKPOINT.json
while milestones_remaining and budgets_ok:
  ensure REQUIREMENTS.md, ROADMAP.md exist for current milestone
    # generated from ZTO.md on first entry, via gsd-roadmapper in zto mode
  for each phase in ROADMAP.md not yet complete:
    write CONTEXT.md from ZTO.md (deterministic, no agent call)
    run gsd-plan-phase (mode=zto)
    run gsd-execute-phase (mode=zto, autonomous plans)
    run gsd-verifier (mode=zto)
    if verifier=passed: continue
    if verifier=gaps_found and retries_left:
      run gsd-plan-phase --gaps; loop
    else:
      hard_stop("verification failed after retries")
  mark milestone complete
  if more milestones in spec: advance; else: done
emit DONE or HARD_STOP
```

The loop is a flat `while`, not recursion, so resume is trivial: re-enter the loop with the checkpoint.

### 5.6 Hard stops

`zto` immediately halts and writes a HARD_STOP event for:

- Verifier returns `gaps_found` more than `max_gap_retries` in a row on the same phase.
- Executor reports a deviation that requires user input (rule 4 in `<deviation_rules>`).
- An executor would run a destructive command but `destructive: false` is set.
- A required secret is missing.
- Wall-clock or token budget hit.
- Git working tree becomes dirty in a way the orchestrator cannot resolve.

Each hard stop writes the reason to RUN.log and CHECKPOINT.json; the user resolves out-of-band, then runs `/gsd-zto --resume`.

### 5.7 Resume semantics

`--resume` reads CHECKPOINT.json and re-enters the loop. It never re-runs the interview. The CHECKPOINT is updated atomically after every phase boundary so a crash mid-phase only loses partial executor work — which existing per-task commits already protect.

---

## 6. Implementation phasing

This is itself a multi-phase build that, ironically, is a strong candidate for a manual GSD project on this repo. Suggested phases:

### Phase 1 — Spec and skeleton
- Add `src/commands/gsd/zto.md` with frontmatter, objective, and a stub `<process>` that prints "not yet implemented."
- Add `src/workflows/zto.md` and `src/workflows/zto-interview.md` skeletons.
- Add `src/templates/zto.md` template for `.planning/ZTO.md`.
- Update `src/commands/gsd/help.md` to list the new command.
- Update `scripts/install.sh` / `scripts/install.ps1` if they enumerate files explicitly.

Risk: minimal. No behavior change for existing users.

### Phase 2 — Interview-only end-to-end
- Implement `zto-interview.md` fully: load spec, restate, gray areas, autonomy contract, approval gate.
- Write `.planning/ZTO.md` and `.planning/zto/CHECKPOINT.json`.
- After approval, do **not** loop yet — just print the synthesized roadmap that *would* run and exit.

This lets us validate the interview UX in isolation. It also gives users a useful tool on day one: "show me what zto thinks I'm asking for."

### Phase 3 — Roadmap + first phase autonomy
- Add `mode: "zto"` branches to `gsd-roadmapper` and `gsd-planner` agent prompts.
- Implement the loop in `workflows/zto.md` but cap at one phase.
- Run end-to-end on a tiny sample spec ("a CLI that prints the date in ISO format"). Verify artifacts, commits, and verifier all behave.

### Phase 4 — Multi-phase loop + gap retries
- Remove the one-phase cap.
- Wire `verify → gaps → plan-phase --gaps → execute-phase --gaps-only` cycle with retry counters.
- Add hard-stop branches.

### Phase 5 — Multi-milestone + resume
- Detect when the spec implies more than one milestone (interview asks; roadmapper can also signal).
- Implement `/gsd-zto --resume`.
- Stress-test interrupt/resume: kill mid-execute, mid-plan, mid-verify.

### Phase 6 — Polish & docs
- Update `README.md`, `docs/GSD-CURSOR-ADAPTATION.md`, and `CHANGELOG.md`.
- Add a worked example: a small spec, the resulting ZTO.md, and the artifacts produced.
- Add `references/zto.md` with the autonomy-contract glossary and hard-stop catalog.

Each phase ends in a working `/gsd-zto` — features compound, nothing is half-wired.

---

## 7. Configuration changes

Add to `.planning/config.json` schema (all optional, with defaults):

```json
{
  "zto": {
    "default_wall_clock_minutes": 120,
    "default_max_gap_retries": 2,
    "default_max_milestones": 1,
    "allow_destructive_default": false,
    "allow_network_default": true
  }
}
```

These are *defaults* the interview presents; the user can override per run. They never bypass the interview.

The existing `mode: "interactive" | "yolo"` field is unchanged. `zto` is not a third mode — it is an orchestrator that overrides mode-locally for the duration of the run.

---

## 8. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Agents silently degrade in `zto` mode because nobody is watching. | RUN.log captures every decision; `gsd-verifier` still runs at phase boundaries; ZTO.md records every auto-resolved gray area so the user can audit after. |
| Roadmaps generated from a 12-question interview are too shallow. | `zto` is explicitly for rough cuts; we set expectations in the command description and in `ZTO.md`'s opening sentence. The fix for "not good enough" is the manual path, not deeper auto-interviewing. |
| Long unattended runs burn tokens on a wrong path. | Wall-clock and gap-retry caps. Default 2h. RUN.log lets the user diagnose without re-running. |
| `mode: "zto"` becomes a giant special case in every agent. | Hard rule: if a `zto` branch in an agent exceeds ~15 lines or duplicates logic, redesign instead of forking. |
| Resume corrupts state. | CHECKPOINT.json is written atomically (write-temp + rename) only at safe boundaries (between phases). Mid-phase crashes are recoverable via the per-task git commits the executor already makes. |
| Brownfield: `zto` overwrites useful existing planning files. | `zto` refuses to start if `ROADMAP.md` already has incomplete phases. The user must complete or archive them first, or pass `--force` (Phase 5+). |

---

## 9. Open questions

1. **Spec format.** Do we require any structure (e.g., bullets under headings) or accept truly freeform prose? Proposal: accept anything, but the interview restates it into a structured form before the user approves.
2. **Single-milestone default.** Should multi-milestone be opt-in (`--milestones N`) or auto-detected? Proposal: opt-in for v1 to keep blast radius small.
3. **UAT placement.** Do we ever run `/gsd-verify-work` (conversational UAT) inside the loop? Proposal: no — it requires the user. `zto` ends with a *suggestion* to run UAT, not a call to it.
4. **What counts as "rough enough"?** `gsd-verifier` already has `must_haves` per phase. We rely on those. The interview is the only place "rough" is encoded — by which gray areas the user delegates.
5. **Concurrency with `/gsd-quick`.** Should `zto` block quick tasks while running? Proposal: yes — a lock file in `.planning/zto/` that quick mode checks.

---

## 10. Success criteria

A working `zto` v1 is one where:

- [ ] A user with a mapped project can run `/gsd-zto path/to/spec.md`, answer ≤12 questions, hit "Go," and walk away.
- [ ] On a sample spec ("a CLI that prints the date in ISO format with `--utc` and `--unix` flags"), `zto` produces ROADMAP.md → executed phases → passing verifier → green commit history, without further input.
- [ ] On a deliberately ambiguous spec, `zto` still completes a defensible rough cut and logs every assumption it made in ZTO.md and RUN.log.
- [ ] Killing the session mid-phase and running `/gsd-zto --resume` continues from the last phase boundary.
- [ ] No existing command's behavior changes for users who never invoke `/gsd-zto`.
- [ ] Adding `zto` mode to each agent costs ≤15 lines per agent.
- [ ] `help.md`, `README.md`, and `CHANGELOG.md` describe `zto` accurately enough that a new user could try it without reading the workflow files.

---

## 11. Testing approach

Because GSD is documentation-as-code, "tests" are reproducible runs against fixture specs:

- `tests/zto/specs/` — small spec files of varying ambiguity (`tiny-cli.md`, `crud-app.md`, `ambiguous.md`).
- For each fixture: a recorded transcript of the interview, the produced ZTO.md, and the final artifact tree.
- A CI-friendly check that, for the `tiny-cli` fixture, the produced commits compile and run.

In Phase 2 we ship the interview-only path, which is easy to snapshot-test: same spec in → same ZTO.md out (modulo dates). That snapshot is the first regression net.

---

## 12. Rollout

1. Land Phases 1–2 behind no feature flag — they are inert without user invocation.
2. Land Phases 3–4 with a `## Experimental` note in `help.md`.
3. Promote to general availability after Phase 6, with an entry in `CHANGELOG.md` and an example walk-through in `README.md`.

No migration is required for existing users. `zto` is purely additive.
