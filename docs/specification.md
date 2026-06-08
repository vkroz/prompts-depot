# agn: Vision, Specification and Requirements

## Vision


### agn goals:
- Provide a framework and mechanisms for accelerated software development using AI coding tools (Claude Code).
- Establish AI-assisted SDLC where repeatable steps in the lifecycle are automated or assisted using AI agents, accelerating overall delivery.
- Stretch goal: establish a feedback loop for incremental improvements of agn artifacts (skills, rules, agents, hooks, tools/MCPs) — the more users use it, the better it serves their projects (tracked by backlog task `feedback_loop_infrastructure`).

### SDLC model

agn's SDLC is a **recursive decomposition** pattern. Each tier (product → epic → feature → task) runs the same five stages at a different level of abstraction, with a user gate between stages:

```
define → design → plan → implement → validate
```

Decomposition recurses: a stage at one tier produces the work items that are detailed one tier down.

Each stage owns one responsibility — and one thing it must not do:

| Stage | Owns | Produces | Does NOT do |
|---|---|---|---|
| **Define** | WHAT and WHY: problem, objective, scope, acceptance criteria | Unit body (or product docs) | Solution shape; decomposition |
| **Design** | HOW-shape: solution structure, technology choices, contracts | Architecture (product); design delta (epic, feature) | Implementation detail; child work items |
| **Plan** | Decomposition and ordering | Child work items + ordered list in the parent unit | Body revision; design decisions |
| **Implement** | Execution | Code, artifacts, detailed design (task level) | Independent verification |
| **Validate** | Independent verification against spec | QA report, verdict | Redesign; fixes beyond glue-level |

**Stage boundaries:**

- **Define stops at WHAT.** If the dialog drifts into how to build it, Define records the question and moves on — the answer belongs to Design or Implement.
- **Design stops at shape.** Function-level design, schemas, and signatures belong to Implement (task level, inline).
- **Plan owns all decomposition.** Define never creates child units. Epic → features and feature → tasks happen in Plan; product → top-level work items happens in Plan at product level.
- **Implement owns its exit check.** A task's `## Quality gates` run as Implement's exit criterion — a task does not move to `done` with failing gates. This is builder self-verification, not QA.
- **Validate runs fresh-context at feature level and above.** The QA sub-agent (`rules/qa.md`) sees spec + result only. Task-level validate is lightweight — it runs the task's own `## Quality gates` in the main session, not a fresh-context QA pass.

Every stage skill is **mode-aware**: it composes on the first pass and refines in place on later invocations. Creating and revising are the same stage, not different verbs.

#### Stage × level matrix

Which stage produces what artifact at which level:

| Level | Define | Design | Plan | Implement | Validate |
|---|---|---|---|---|---|
| **Product** | `docs/vision.md`, `docs/spec.md`, `docs/requirements.md` | `docs/architecture.md` | Ordered set of top-level work items under `tasks/` — epics by default; tier per item by effort estimate | — (via work items) | `/agn:validate product` |
| **Epic** | Epic body: problem, objective, scope, acceptance criteria | Design delta — required by default | `## Linked features` list + feature files | — (via features) | `/agn:validate epic` |
| **Feature** | Feature body: problem, objective, acceptance criteria | Design delta — optional; skip requires recorded rationale | `## Tasks` list + task files | — (via tasks) | `/agn:validate feature` |
| **Task** | Task body: problem, scope, acceptance criteria, quality gates; design decisions inline | Folded into Define | None — the task is the plan | Execution: code, artifacts | `/agn:validate task` (lightweight gates) |

Per-level notes:

- **Product.** Define, Design, and Plan always run, in order, each behind a user gate. The product plan has no artifact of its own — the ordered set of work-item files is the plan. Plan produces top-level work items, epics by default; for each, pick the smallest tier that fits the estimated effort. Product has no direct implementation; it expands into its epics over time.
- **Epic.** Design and Plan are separate, gated stages. Define produces the epic body only — decomposition into features belongs to Plan. Validate is integration / system test at the epic boundary.
- **Feature.** Design runs when complexity warrants it. Skipping is a recorded decision (`Design: not needed — <reason>`), not an omission. Plan (task decomposition) always runs.
- **Task.** Define and Design combine — design decisions are documented inline in the task body. A task-level decision that contradicts or extends the parent feature's design escalates via the gap protocol (`tasks/gaps/`); it is not resolved in the task.

Documentation evolves continuously. Each completed unit triggers automatic review of upstream artifacts (architecture, spec, requirements) for drift.

## Current Implementation State

`agn` ships as a Claude Code plugin (`agn`) in the `agenture` marketplace; install via `/plugin install agn@agenture`. It implements this specification in full. For the shipped-component inventory see `CLAUDE.md`; for the user-facing skill reference see `README.md`. Component contracts are specified under **Plugin Specification** below.

# Specification and Requirements

## Workflows: User Experience

agn supports four workflows. Each is a sequence of stages driven by skill invocations. At every stage the user invokes a skill, the agent executes it (often via a sub-agent), and the user approves the output before moving forward.

### New Product Development

Starting point: the user has an idea for a new product and no existing documents.

**Definition** — `/agn:define product`

The Planner sub-agent guides the user through producing three documents:

1. *Vision* (`docs/vision.md`): the agent interviews the user about the problem, target users, and key capabilities. It drafts a one-page vision. The user reviews and iterates.
2. *Specification* (`docs/spec.md`): using the approved vision and any additional inputs (reference products, online docs, domain knowledge), the agent drafts a product specification. The user reviews and iterates.
3. *Requirements* (`docs/requirements.md`): the agent drafts requirements that disambiguate and complement the spec with formal detail. The user reviews and iterates.

After all three are drafted, the agent validates them against business-case and functional-completeness rubrics, produces a report, and addresses findings until all critical issues are resolved or the user approves moving forward.

**Design (Architecture)** — `/agn:design product`

The Planner sub-agent drafts `docs/architecture.md` covering technology choices, system architecture, domain dictionary (key terms only), workflows, key APIs, and security mechanisms. Detailed design (API signatures, schemas) is out of scope.

Review cycle: user feedback → agent updates → QA review → consistency check against specs → user approval.

**Plan (Decomposition)** — `/agn:define epic`, `/agn:define feature`, `/agn:define task`

The agent decomposes the product into a four-tier hierarchy. There is no single implementation plan document — the epic, feature, and task files together are the plan.

- Large functional blocks become epics via `/agn:define epic`. The epic file lists linked features (optionally pre-created in the same Planner session).
- Coherent slices become features via `/agn:define feature`. The feature file lists linked tasks. A feature can be attached to an epic via `--epic <slug>` or stand alone.
- One-off work or units inside a feature become tasks via `/agn:define task`. A task can be attached to a feature via `--feature <slug>` or be ad-hoc.

The same Planner sub-agent handles Design + Plan at every tier. Each tier is optional one level up. Most product work is a feature; epics are for genuinely larger blocks.

Review cycle: user reviews each definition → Planner self-validates against rubrics → specs/requirements updated if needed → user approves.

**Refinement** — `/agn:design <level>`, `/agn:plan <level>`

For revising existing work units rather than creating new ones, `/agn:design <product|epic|feature>` and `/agn:plan <epic|feature>` invoke the same Planner sub-agent in focused mode against an existing file.

**Implementation** — `/agn:implement epic`, `/agn:implement feature`, `/agn:implement task`

- `/agn:implement task <path>` — single task execution.
- `/agn:implement feature <slug>` — every open task of a feature in order, stop-per-task for review.
- `/agn:implement epic <slug>` — every linked feature of an epic in order, stop-per-feature for review.

Per task, the agent: (1) cross-checks upstream design, halting via the escalation protocol if gaps surface; (2) produces detailed design (if not already locked upstream); (3) implements; (4) writes and runs tests. Blockers and ambiguities are surfaced via the escalation protocol.

At feature and epic boundaries, the QA sub-agent runs integration tests via `/agn:validate feature` or `/agn:validate epic`. User reviews before advancing. Documents are updated if implementation reveals necessary changes (PostClose hook automates the check).

**Validation** — `/agn:validate task | feature | epic | product`

- `/agn:validate task` — task-level quality gates run in the main session (lightweight).
- `/agn:validate feature` and `/agn:validate epic` — integration tests run by the QA sub-agent (fresh context, sees spec + result only).
- `/agn:validate product` — full system test by the QA sub-agent. Produces a system test report (critical/major/minor). Fixes what it can; escalates the rest. Re-tests until all critical and major issues are cleared.

---

### Bug Fix

Precondition: existing product with documents in `docs/`.

1. User creates a bug ticket: `/agn:define task --kind bug` — observed problem, expected result, actual result, reproduction steps.
2. User runs `/agn:implement task <defect-task>`.
3. Agent investigates root cause. Produces root cause analysis and fix plan.
4. **Architecture gate**: if the fix requires architectural changes, the agent stops and discusses with the user before proceeding.
5. Agent implements: detailed design → code → tests.
6. `/agn:validate task` — task-level quality gates.
7. `/agn:validate feature` (or `/agn:validate epic` if the touched scope crosses features) — integration tests via QA sub-agent.
8. `/agn:validate product` — full regression via QA sub-agent.
9. User approves.

---

### Maintenance

Precondition: existing product with documents in `docs/`.

**Ad-hoc task:**

1. User creates a task: `/agn:define task` (kind: `refactor`, `upgrade`, `chore`, etc.).
2. User runs `/agn:implement task <task>`. Architecture gate applies.
3. `/agn:validate task` → `/agn:validate feature` → user approves.

**Codebase optimization:**

1. User invokes `/agn:code-review`.
2. Agent audits the full codebase. Produces a report: technical debt, fragmentation, inconsistency, duplication, performance issues.
3. User reviews and prioritizes findings.
4. If structural changes are needed: agent updates architecture via `/agn:design product`, user approves.
5. Agent decomposes the work via `/agn:define feature` (or `/agn:define epic` for larger scope). User approves.
6. Execution follows the standard `/agn:implement feature` → `/agn:validate feature` → `/agn:validate product` flow.

---

### Incremental Feature

Precondition: existing product with documents in `docs/`.

Same stages as new product development, but the agent operates on existing documents:

1. `/agn:define product` (or focused `/agn:design product`) — agent drafts additions/amendments to existing vision, spec, requirements, and architecture as needed. After this stage, the existing docs describe the product including the new feature — no separate "feature docs."
2. `/agn:define feature` (or `/agn:define epic` if the work spans multiple features) — scoped to the new feature but accounts for dependencies on existing code.
3. `/agn:implement feature` (or `/agn:implement epic`), then `/agn:validate feature`, `/agn:validate product` — same as new product development.

---

## Plugin Specification

Derived from the workflows above. Everything below exists to support the user experience described in the previous section.

### Skills

agn is a Claude Code plugin named `agn`. All skills are namespaced as `/agn:<skill>`. Lifecycle skills follow the verb-noun pattern (`<verb> <level>`); tool skills are level-agnostic.

#### Lifecycle skills

| Verb | Levels | Phase(s) driven | Mechanism |
|------|--------|-----------------|-----------|
| `define` | `product\|epic\|feature\|task` | Full creation: Requirements + Spec + Design + Plan at the chosen level | Delegates to Planner sub-agent |
| `design` | `product\|epic\|feature` | Focused revision of an existing unit's design | Delegates to Planner sub-agent |
| `plan` | `epic\|feature` | Focused revision of an existing unit's decomposition | Delegates to Planner sub-agent |
| `implement` | `epic\|feature\|task` | Implementation. At task level: cross-check + code + unit tests. Halts on design gaps via escalation protocol | Main session (orchestration); recursive descent at higher levels |
| `validate` | `task\|feature\|epic\|product` | Validation. Task level: lightweight quality gates in main session. Higher levels: QA sub-agent | Mixed |

#### Tool skills

| Skill | Drives | Input | Output |
|-------|--------|-------|--------|
| `/agn:code-review` | Codebase audit | Full codebase | Audit report, backlog tasks for findings |
| `/agn:code-comment` | Code commenting | Source files | Commented source files |
| `/agn:code-commit` | Version control | Staged changes | Git commit |
| `/agn:docs-sync` | Doc maintenance on closure | Closed work unit + linked spec | Proposed diffs to `docs/architecture.md`, `docs/spec.md`, `docs/requirements.md` |

#### Skill behavior contract

Every lifecycle skill must:
1. **Validate preconditions** before starting. Skills tell the user what is missing if preconditions are not met.
2. **Produce artifacts** as specified by the verb.
3. **Run a validation cycle** after producing artifacts: self-review against rubrics, produce a report, address findings with user input.
4. **Maintain document consistency**: changes flow forward along `vision → spec → requirements → architecture → epics → features → tasks`. Upstream documents are updated only when downstream work reveals the need.
5. **Get user approval** before the stage is considered complete.

### Sub-agents

agn uses two sub-agents to enforce role separation and isolate rules from the main session.

#### Planner

- **Owns**: `rules/task-composition.md` (and `rules/writing-guideline.md` for doc-producing tiers).
- **Invoked by**: `/agn:define <level>`, `/agn:design <level>`, `/agn:plan <level>`.
- **Behavior**: receives the user's context from the parent session, dialogs with the user via the parent for clarifications, drives Design + Plan phases. Persists outputs via `taskman.sh`.
- **Why a sub-agent**: composition rules stay out of the parent session's context; the same agent can be reused across all four tiers (it is level-aware).

#### QA

- **Owns**: `rules/qa.md`.
- **Invoked by**: `/agn:validate feature`, `/agn:validate epic`, `/agn:validate product`.
- **Behavior**: receives spec + implementation result only (not the implementer's reasoning or decisions). Reviews code against spec, runs tests, identifies coverage gaps, validates end-to-end flows. Produces a verdict + report.
- **Why a sub-agent**: fresh context catches issues the implementer overlooks due to implicit context bias.

`/agn:validate task` runs in the main session (lightweight; just runs the task's quality gates) because the cost of a sub-agent is not warranted for a single task's validation.

### Hooks

- **PostClose** — fires on success of `taskman.sh move <path> done`, `taskman.sh feature close <slug>`, or `taskman.sh epic close <slug>`. Invokes `/agn:docs-sync` to review upstream `docs/` files for drift and propose diffs. User reviews diffs before commit. If no active Claude session at hook fire time, the hook queues a note that surfaces at next session start.

### Escalation protocol

When `/agn:implement task` cross-checks the task body against upstream design and detects missing or ambiguous design decisions, the skill halts immediately and:

1. Writes a gap-log entry to a durable on-disk location (survives compaction; feeds the future feedback loop).
2. The entry contains: gap description, suspected upstream level (task / feature / epic / architecture), implementation context at point of detection.
3. Surfaces routing instructions to the user: `"Gap at <level>. Run /agn:design <level> to address before continuing."`
4. The user manually runs the upstream skill. After it completes, the user re-invokes `/agn:implement task`; the skill re-reads the task body to pick up the revised design.

This protocol prevents silent intermixing of Design and Implementation work. Frequent escalations are a signal of insufficient upstream design and feed into the long-term feedback loop (backlog task `feedback_loop_infrastructure`) to improve prompts and rules.

### Behavioral Guardrails

Rules live in `plugins/agn/rules/`. To activate them in a user's project, the user `@`-imports them in their project's `CLAUDE.md`.

| Guardrail | Content | Source | Loaded by |
|-----------|---------|--------|-----------|
| Core Principles | YAGNI, KISS, DRY, readability over performance, single-task focus, clarify ambiguity, step-by-step approval, root-cause troubleshooting | `rules/first-principles.md` | Always — every session |
| Task Composition | Frontmatter shapes, body section requirements, completion-summary template | `rules/task-composition.md` | Planner sub-agent |
| Task Persistence | Lifecycle (backlog → active → done), CLI surface, validation rules | `taskman.sh help` | Authoritative reference; consulted by skills that touch lifecycle |
| QA | Validation mindset, role separation, integration / system test protocols | `rules/qa.md` | QA sub-agent + `/agn:validate task` |
| Doc Maintenance | What to check on closure (drift in architecture / spec / requirements) | `rules/doc-maintenance.md` | `/agn:docs-sync` |
| Writing Guideline | Crisp, no-fluff document style: short sentences, active voice, absolute dates, no weasel/peacock words | `rules/writing-guideline.md` | Document-producing skills (Planner sub-agent, `/agn:docs-sync`) |

Skills themselves contain only workflow instructions — no inlined rules. They rely on the rules being present in their context (loaded by sub-agent system prompt, by skill `!cat`, or by the user's `CLAUDE.md`).

### User's Project Structure

agn skills create the following layout in the user's project as they progress through SDLC stages:

```
user-project/
├── docs/
│   ├── vision.md
│   ├── spec.md
│   ├── requirements.md
│   ├── architecture.md
│   └── <area>/.../-spec.md       # feature-scoped specs
├── tasks/
│   ├── epics/                     # YYYYMMDD_<slug>.md
│   ├── features/                  # YYYYMMDD_<slug>.md
│   ├── backlog/                   # YYYYMMDD[_NN]_<slug>.md
│   ├── active/
│   └── done/
```

`/agn:define product` creates the `docs/` directory and initial documents if they don't exist. `/agn:define <epic|feature|task>` create files under `tasks/`. The project structure is created incrementally as the user progresses through stages — no upfront scaffolding required.

### Workflow State

Each skill checks its own preconditions by examining which artifacts exist. There is no separate workflow state file. The documents ARE the state:

- Definition complete → `vision.md`, `spec.md`, `requirements.md` exist with validated content.
- Architecture complete → `architecture.md` exists and is consistent with definition docs.
- Decomposition complete → epic / feature / task files exist.
- Feature complete → all member tasks in `tasks/done/`, feature `status: done`.
- Epic complete → all member features `status: done`, epic `status: done`.
- Implementation complete → all tasks in `tasks/done/` and all epics/features closed.
- Validation complete → system test report shows all critical/major issues resolved.

The user can always look at `docs/` and `tasks/` to understand where they are.

---

## Requirements

Cross-cutting concerns that apply across all workflows.

### Document Consistency

When any document changes, the skill that made the change must check dependent documents for inconsistencies and update them. The dependency order is: vision → spec → requirements → architecture → epics → features → tasks. The agent must explicitly state which documents it is updating and why. The PostClose hook automates this check at closure.

### Precondition Enforcement

Every skill validates its preconditions before starting work. Missing preconditions produce a clear message ("Cannot run `/agn:design product` — definition documents not found. Run `/agn:define product` first."). No skill silently proceeds with incomplete inputs.

### User Approval Gates

Stage transitions require explicit user approval. The agent must not proceed to the next stage until the user says to. Within a stage the agent can iterate autonomously (draft → validate → update), but completing a stage and moving forward is always a user decision.

### Architecture Impact Gate

Any change that touches the high-level architecture — whether discovered during bug fixing, maintenance, or implementation — triggers a mandatory discussion with the user before proceeding. The agent presents the architectural impact, proposes options, and waits for explicit approval. The escalation protocol routes architecture-level gaps to `/agn:design product`.

### Task DAG Execution

When executing multiple tasks (`/agn:implement feature` or `/agn:implement epic`), the agent respects feature and epic boundaries. In v1, tasks run sequentially within a feature, and features run sequentially within an epic; parallel execution is deferred. Integration tests (`/agn:validate feature` or `/agn:validate epic`) run at boundaries. The user can interrupt execution at any boundary.

### Validation Rubrics

Two rubrics applied during Definition and reused as quality gates throughout:

**Business case:**
- Do we demonstrate understanding of the business and user problem?
- Does the spec actually solve the business problem? Is there a better way?
- Is the monetization case sound?
- Are risks identified and addressed?

**Functional:**
- Is functionality defined consistently across all documents?
- Are there gaps, ambiguities, inconsistencies, or missing details?
- Is the level of detail sufficient for the next stage to proceed without guesswork?

### Scope Discipline

During implementation, the agent does not modify functionality outside the scope of the current task, feature, or epic. Any scope creep is flagged to the user. This applies to all workflows.

---

## Traceability: Workflows → Plugin Components

Cross-reference confirming that every workflow step maps to a plugin component.

| Workflow Step | Skill (`/agn:*`) | Agent | Artifacts Produced |
|--------------|------------------|-------|--------------------|
| New product → Definition | `define product` | Planner | vision, spec, requirements |
| New product → Architecture | `design product` | Planner | architecture |
| New product → Epic decomposition | `define epic` | Planner | epic file, linked feature files |
| New product → Feature decomposition | `define feature` | Planner | feature file, linked task files |
| New product → Implementation (epic) | `implement epic` | main session | code, tests, integration reports per feature |
| New product → Implementation (feature) | `implement feature` | main session | code, tests, integration report |
| New product → Implementation (task) | `implement task` | main session | code, tests (escalates on design gap) |
| New product → Integration test (feature/epic) | `validate feature` / `validate epic` | QA | integration test report |
| New product → System test | `validate product` | QA | system test report |
| Bug fix → Task creation | `define task --kind bug` | Planner | bug ticket file |
| Bug fix → Fix | `implement task` | main session | code, tests, root cause analysis |
| Bug fix → Integration test | `validate feature` | QA | integration test report |
| Bug fix → Regression | `validate product` | QA | system test report |
| Maintenance → Ad-hoc | `define task`, `implement task` | Planner, main session | task file, code, tests |
| Maintenance → Ad-hoc → Test | `validate task`, `validate feature` | main session, QA | test reports |
| Maintenance → Optimization | `code-review`, `define feature` (or `define epic`), `implement feature` | main session, Planner | audit report, decomposition artifacts, code |
| Incremental feature | `define feature` + `implement feature` + `validate feature` | Planner, main, QA | same as new product (amended) |
| Doc sync after closure | `docs-sync` (PostClose hook) | main session | proposed diffs to `docs/` |

All workflows are fully covered by the lifecycle and tool skills, the Planner and QA sub-agents, and the PostClose hook. Behavioral guardrails are loaded according to the table in **Behavioral Guardrails** above. No workflow step requires a component not listed.
