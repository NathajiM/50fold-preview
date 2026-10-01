# Claudius: model-tier routing for coding work (condensed design)

Claudius is the execution layer of 50fold. It is a cost-smart dispatcher for Claude Code: the main chat sizes each coding task, hands the writing to the cheapest capable model, and requires a read-only skeptic to approve the result before it counts as done. This is a condensed design document. The implementation (agent definitions, skills, hooks, installer, health check) is in the private repo.

## Goal

An always-on orchestration layer for code writing. The main chat becomes a dispatcher that classifies each task, routes it to the right model tier, and pushes substantial tasks into isolated parallel venues so the main chat stays free.

The design fuses two patterns:

- **Advisor:** a cheap session escalates up to an expensive model at verification points. Strong for coding.
- **Orchestrator:** an expensive lead decomposes work down to cheap workers. Strong for decomposable work.

Cheap model on the many small edits, expensive model on the few hard ones. The smartest model plans and judges; it does not type boilerplate.

## Model tiers

| Tier | Role |
| --- | --- |
| Top tier (Fable) | Planner for architectural or ambiguous work; verification gate for high-stakes changes only |
| Mid tier (Opus) | Senior writer for complex code; default verification gate |
| Fast tier (Sonnet) | Writer for standard code; default host chat |

Routing uses **subagents pinned to a model**. This is the load-bearing mechanism: tier selection works regardless of which model the host chat runs on, so a cheap-hosted chat can still dispatch a strong writer. A subagent never inherits the host model.

## Complexity rubric

Every code task is classified into one of four bands.

| Band | Signals | Plan | Write | Gate |
| --- | --- | --- | --- | --- |
| Trivial | typo, rename, dependency bump, one-liner | none | inline, cheapest tier | none |
| Standard | one file, well specified, follows an existing pattern | skip or brief | fast writer | mid-tier skeptic |
| Complex | multi-file, new abstraction, tricky logic, security-sensitive | writer plans inline | senior writer | mid-tier skeptic |
| Architectural | schema change, new subsystem, cross-cutting, ambiguous spec | top-tier planner | senior writer (may split) | top-tier skeptic |

**Escalators** raise a task a band regardless of size: security, authentication, money, data migrations, row-level access policies, anything hard to reverse.

## The verification gate

Every non-trivial change needs a PASS from a read-only skeptic before it is done or merged.

- **Read-only.** The reviewer can read code and run commands. It cannot edit.
- **Adversarial.** Its instructions are to find what is broken, not confirm success. It judges the diff against the spec's done-criteria and the surrounding ground truth.
- **PASS or BLOCK.** BLOCK sends the change back to the writer. It does not merge.
- **Criteria, not volume.** A one-liner that meets every done-criterion is a PASS. This keeps the gate compatible with a least-code policy applied to writers: one stops over-building, the other stops under-building.
- **Escalation is one-way.** The standing gate model is a floor. High-stakes tasks escalate to the top tier automatically; Claudius never quietly drops down.
- **Cost-shaped.** A gate is expensive because it rebuilds a cold context, not because of what it writes. So gates run once per unit of work, scoped to the diff and the specific claims to refute, and a re-gate after a BLOCK re-checks only the changed hunks. Shortening the gate's prompt saves almost nothing; cutting runs does.

## Venues

- **In-session subagents:** parallel fan-out for work wanted now, results land in the current chat.
- **Chips:** a substantial, independent task gets its own chat and its own git worktree, so it never collides with the main chat or other chips. Several run in parallel while the main chat keeps assigning.

### Chip lifecycle

1. Create an isolated worktree off the current working branch.
2. Plan (top or mid tier, by band).
3. Write, via pinned-model writers; architectural work may fan out.
4. Run the repo's tests and type checks.
5. Run the verification gate. A BLOCK stops the worker.
6. On all green, merge into the working branch. Never the main branch unattended. Remove the worktree.
7. Report what merged and the gate results, ending with an archive verdict. The chip archives its own chat only when the verdict is clean; otherwise it stays open and says why.

## Token discipline beyond routing

Routing only governs dispatched work. The host chat bills at its own model for the whole session.

- **Model-fit nudge.** A pre-prompt hook warns once per session per model when the chat's model looks oversized for the prompt (premium model on a plain lookup, for example). Advisory only: no model call, no blocking. The motivation was an audit in which 54% of a premium model's spend was ordinary chat.
- **Context-size nudge.** The same hook flags when a conversation is large or stale enough that every new turn re-bills a lot of history, and suggests compacting or starting fresh.
- **Efficiency audit.** A read-only local report of tools and plugins that load into every session but go unused, duplicated skills, and which models and oversized conversations drove the cost.
- **Noisy-output compression** for test runs, builds, and diffs, with a deliberate exception: verification evidence stays raw so a PASS or BLOCK rests on real output.

## Constraints and risks

| Risk | Mitigation |
| --- | --- |
| Runaway spend on strong models | Rubric defaults most edits to the fast tier; strong models plan and gate only; always-on cost is classification |
| Bad auto-merge | Gate plus green tests, working branch only, worktree isolation, easy revert |
| Clobbering the user's global instructions | Marker-delimited, idempotent, reversible install; one backed-up settings edit |
| A chip loses the main chat's context | Chip prompts are fully self-contained specs |
| Spawning a chip needs one human click | Kept deliberately: it caps parallel spend |

Non-goals: no cross-vendor routing in the core design, no re-implementing third-party tools the system can call.
