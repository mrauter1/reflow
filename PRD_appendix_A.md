# Reflow — Product Requirements Document

## Reflow v0.1

**Project name:** Reflow
**Category:** Deterministic workflow runtime

## 1. Product summary

Reflow is a file-first, local CLI runtime for reusable AI-assisted workflows.

It executes declarative workflows through a small deterministic kernel and delegates actual work to pluggable executors such as:

* Codex CLI
* Claude Code CLI
* nested workflows
* local shell or CLI commands

Reflow is **not** a planner or autonomous manager. It manages:

* runs
* steps
* semantic iterations
* attempts
* artifacts
* retries
* semantic routing
* terminal states

Domain logic lives in workflows. Role logic lives in skills. Provider-specific execution behavior lives in configuration and adapters.

The first concrete workflow is a review → screen → implement → verify loop, but Reflow is intended to support broader technical and operational workflows over time.

## 2. Problem statement

Advanced coding and analysis agents can already do a large share of expert-level labor, but the bottleneck is still the manual scaffolding around them.

Useful loops are often discovered empirically and then repeated by hand:

* review work
* screen findings
* implement worthwhile changes
* verify
* repeat until convergence

These loops are powerful but fragile. They are hard to scale because they are:

* prompt-driven instead of file-driven
* difficult to inspect after the fact
* hard to reuse consistently
* dependent on manual orchestration
* easy to drift over time

Reflow solves this by turning useful loops into explicit workflow definitions executed by a deterministic runtime that preserves every attempt and every artifact.

## 3. Goals

Reflow must:

* provide a **general workflow runtime**, not a hardcoded code-review bot
* keep authored workflows readable and maintainable in-repo
* support iterative loops with immutable attempt history
* support semantic success, semantic failure, and semantic routing
* support nested workflows in v0.1
* keep provider choice outside workflow definitions
* fail fast if the configured provider is unavailable
* rely only on visible persisted context, not hidden provider memory
* use terminal-native execution in v0.1
* preserve enough history for providers to infer what is current from the run history
* separate execution-stage retry from semantic iteration budgets

## 4. Non-goals

Reflow will not, in v0.1:

* implement orchestrator-level human approvals
* implement a context compiler or pruning engine
* implement parallel branches or a graph runtime
* implement a central policy engine
* implement a global schema registry
* depend on `.agents/skills` or `.claude/skills`
* depend on hidden provider-private chain-of-thought
* provide a web UI or service backend
* perform dynamic provider capability planning beyond config-based mapping

Provider-native tool approvals remain the provider’s responsibility, not Reflow’s.

## 5. Primary users

Primary users are:

* experienced developers using Codex CLI or Claude Code
* technical operators building reusable AI loops
* engineering and operations leads standardizing AI-assisted workflows

Secondary users are:

* analysts or domain experts running terminal-driven workflows under supervision
* teams that want repeatable, inspectable AI-assisted SOP execution

## 6. Current tooling assumptions

Reflow is designed around current local CLI surfaces.

Codex supports non-interactive `codex exec`, structured output constraints, resume, and project guidance through `AGENTS.md`. Claude Code supports programmatic execution with `claude -p`, structured JSON output with a JSON Schema, and project guidance through `CLAUDE.md`. Both tools operate directly against the local repository and workspace, which makes a path-based artifact model workable: Reflow can reference local artifacts by relative path instead of inlining whole files into prompt context. Because provider-native instruction layering already exists, Reflow does **not** manually append `AGENTS.md` or `CLAUDE.md`; it appends only orchestration-owned messages.

## 7. Product principles

### 7.1 Deterministic kernel

Reflow owns state and transitions, not business reasoning.

### 7.2 File-first

Every meaningful step result is persisted as files and append-only logs.

### 7.3 Visible context only

The runtime depends only on persisted visible artifacts and orchestrator messages.

### 7.4 Immutable attempts

Every retry and every loop iteration creates a new attempt folder. Older attempts are never overwritten.

### 7.5 Workflow-owned logic

Process logic belongs to workflows, not the runtime.

### 7.6 Skill-owned role behavior

Method, constraints, and examples belong to skills.

### 7.7 Provider-agnostic workflows

Authored workflows must not hardcode provider choice.

### 7.8 Small v0.1 surface

Only enough abstraction to run real workflows well.

## 8. Repository boundary

The repository root may contain normal project files: code, documents, SOPs, examples, tests, and other domain assets.

The `.ai/` folder contains only orchestration-native assets:

* workflow definitions
* canonical skills
* orchestrator config
* helper scripts
* run outputs

## 9. Canonical v0.1 repository structure

```text
.ai/
  config/
    orchestrator.yaml
    providers.yaml
    route_eval.schema.json

  workflows/
    <workflow_name>/
      workflow.yaml
      notes.md          # optional
      schemas/          # optional, workflow-local only

  skills/
    <skill_name>/
      SKILL.md

  scripts/
    run_workflow.py
    verify.sh

  runs/
    <run_id>/
      state.json
      messages.jsonl
      <step folders>/
```

There is no `.agents/skills` or `.claude/skills` dependency in v0.1.

## 10. Core concepts

### 10.1 Workflow

A reusable process definition composed of named steps and step-local route rules.

### 10.2 Step

A named unit of work with:

* an action
* a maximum semantic iteration count
* an optional retry policy
* required outputs
* semantic success and failure conditions
* optional local routes

### 10.3 Skill

A reusable role package whose primary entrypoint is `SKILL.md`.

### 10.4 Executor

How a step is run. In v0.1:

* `skill`
* `workflow`
* `cli`

### 10.5 Run

One concrete execution of a workflow.

### 10.6 Semantic visit

One workflow-level visit to a step for domain work. `max_iter` counts semantic visits, not retries.

### 10.7 Attempt

One execution attempt of a step. Multiple attempts may belong to the same semantic visit if retries occur.

### 10.8 Route evaluation

A separate semantic evaluation pass that determines:

* whether the step succeeded
* whether the step failed
* which local routes match
* which route to take next

### 10.9 Message ledger

The canonical append-only run message history that mimics provider session context.

## 11. Workflow model

Each workflow lives in its own folder and must contain `workflow.yaml`.

Optional workflow-local files:

* `notes.md`
* `schemas/*`

### 11.1 Required workflow fields

A workflow must contain:

* `name`
* `steps`

A workflow may contain:

* `description`
* `entrypoint`

### 11.2 `entrypoint`

If `entrypoint` is omitted, Reflow defaults to the first defined step.

### 11.3 Step cardinality

A workflow must define at least one step.

### 11.4 `notes.md`

`notes.md` is workflow-local orchestration guidance.

It is appended by Reflow into the workflow’s message history and therefore becomes part of the effective provider session context for steps in that workflow.

## 12. Step contract

Each step must support:

* `name`
* optional `description`
* `action`
* `max_iter`
* optional `retry`
* optional `context_mode`
* `outputs.required`
* `success.when`
* `failure.when`
* optional `routes`

### 12.1 `name`

Stable step identifier.
Slug-like and machine-safe.

### 12.2 `description`

Optional human-readable explanation.

### 12.3 `action`

Must contain:

* `kind`: `skill | workflow | cli`
* `ref`
* optional `with`

Examples:

* `kind: skill, ref: review-codebase`
* `kind: cli, ref: verify.sh`
* `kind: workflow, ref: nested_workflow_name`

### 12.4 `max_iter`

Maximum number of **semantic visits** to the step within one run.

`max_iter` counts only valid workflow-driven revisits where the domain work needs another pass. It does **not** count retries caused by execution-stage noise or evaluator malfunction.

### 12.5 `retry`

Optional step-level retry policy.

Shape:

```yaml
retry:
  max_retries: 2
```

Rules:

* The first execution of a semantic visit is not counted as a retry.
* On eligible execution-stage or evaluator-stage failure, Reflow may retry up to `max_retries` additional times before that semantic visit is considered failed.
* The retry budget resets to its full value at the start of each new semantic visit.
* Each retry creates a separate attempt folder.
* Retry reasons are appended to the canonical message ledger.
* If omitted, the step inherits the default from `orchestrator.yaml`.

Retryable conditions are:

* transient provider execution error such as process crash or timeout
* malformed provider response
* malformed `route_eval.json`
* `route_eval_conflict` (`step_success = true` and `step_failure = true`)
* ambiguous evaluation (`step_success = false` and `step_failure = false`)
* local script crash or timeout for `cli` steps
* missing required outputs
* routes defined but none match

Non-retryable conditions are:

* provider unavailable (fail fast)
* semantic failure (`failure.when = true` from a valid, non-conflicting evaluation)
* `max_iter` exhaustion

### 12.6 `context_mode`

Allowed for any step kind:

* `skill`
* `workflow`
* `cli`

Only one value is supported in v0.1:

* `full_run`

Any other value is invalid in v0.1.

### 12.7 `outputs.required`

List of files that must exist in the attempt output directory after execution.

This is structural, not semantic.

### 12.8 `success.when`

Natural-language semantic condition answering:
“Did this step achieve its local purpose well enough to continue?”

### 12.9 `failure.when`

Natural-language semantic condition answering:
“Did this step hit a terminal or unrecoverable condition?”

Recoverable conditions should be handled by routing or retry, not by `failure.when`.

### 12.10 `routes`

Ordered local route rules attached to the step.

Each route contains:

* `when`
* `then`

`then` may target:

* another step name
* `stop`
* `escalate`

There is no `from` field because routes are defined on the step itself.

## 13. Skill model

Canonical skills live only under `.ai/skills`.

v0.1 convention:

* `SKILL.md` is the canonical skill entrypoint
* extra support files are allowed but not required or assumed by default
* Reflow is only required to load `SKILL.md` in v0.1

Reflow never exposes the full skill library to a single step. It loads only the current step’s selected skill.

### 13.1 Working directory semantics

By default, all `skill` and `workflow` actions run from the directory where Reflow was launched.

`cli` steps may override the working directory through `action.with.cwd`.

Child workflows inherit the parent run’s working directory in v0.1.

This matters because provider-native project instruction discovery is sensitive to working directory and repository hierarchy. Reflow must keep working-directory behavior stable across parent and child runs.

## 14. Provider selection

Provider selection lives in `.ai/config/providers.yaml`.

It maps:

* skill names to providers
* workflow steps to providers, if needed
* router to a provider
* provider profiles and flags

Workflows themselves do not specify provider choice.

### 14.1 Provider failure behavior

If the configured provider for a step or route evaluator is unavailable, Reflow fails fast.

There is no automatic fallback in v0.1.

### 14.2 Failure recording

Provider-unavailable failures must be written to:

* `messages.jsonl`
* `attempt.json`
* `state.json`

### 14.3 Configuration files

#### 14.3.1 `orchestrator.yaml`

`orchestrator.yaml` must support:

* run ID format
* default `context_mode` (`full_run` only in v0.1)
* default `max_retries` when a step omits `retry` (recommended default: `1`)
* placeholder `retry_backoff` field reserved for the future, not implemented in v0.1
* terminal UX settings
* behavior for unresolved routing after retries are exhausted

#### 14.3.2 `providers.yaml`

`providers.yaml` must support:

* provider mapping per skill
* provider mapping per workflow, if needed
* provider mapping for router
* provider-specific command templates and flags

Conceptual example:

```yaml
defaults:
  router: claude

skills:
  review-codebase: codex
  screen-review-findings: claude
  implement-screened-findings: codex

workflows:
  review_analyze_do: local
```

## 15. Provider approvals

Reflow does not implement human approval gates in v0.1.

Provider-native execution approvals remain external to Reflow:

* Codex uses whatever sandbox and approval settings are configured for that invocation
* Claude Code uses its own permission model and prompts for additional actions as needed

Reflow records execution results, but it does not mediate or store business approvals in v0.1.

## 16. Runtime statuses

Canonical run statuses in v0.1:

* `running`
* `completed`
* `failed`
* `escalated`

### 16.1 Meaning of statuses

* `completed`: workflow reached its intended successful terminal condition
* `failed`: workflow or runtime terminated unsuccessfully
* `escalated`: workflow intentionally handed off because automated continuation is not appropriate

Failure diagnostics are captured through `failure_type` and `failure_reason` fields in runtime artifacts.

Examples of `failure_type`:

* `semantic_failure`
* `provider_unavailable`
* `route_unresolved`
* `route_eval_conflict`
* `ambiguous_evaluation`
* `missing_required_outputs`
* `retries_exhausted`
* `invalid_provider_response`
* `filesystem_error`

## 17. Full-run context model

This is the most important runtime rule.

`full_run` means Reflow maintains a **canonical append-only message ledger** that mimics provider session context history.

Each orchestrator instruction appends a message.
Each provider output appends a message.
Future steps continue from that accumulated history.

Artifacts stay on disk and are referenced by full relative file path.
Reflow does **not** inline whole artifact files into prompt context by default.

### 17.1 Canonical source of truth

The source of truth for v0.1 context is:

* `messages.jsonl`
* referenced artifacts in the workspace

Provider-native session continuation may be used as an implementation optimization, but it is **not** the source of truth.

### 17.2 What gets appended as messages

Reflow appends messages such as:

* workflow setup message
* workflow notes message
* step instruction message
* retry reason message
* provider response message
* CLI result summary message
* route evaluation result message
* transition message
* failure message

### 17.3 Step message contents

A step instruction message should include:

* current workflow name
* current step name
* step description
* required outputs
* semantic success condition
* semantic failure condition
* local routes
* current skill text if the step uses `action.kind: skill`
* relative paths to relevant artifacts

### 17.4 Artifact references

Generated artifacts must be referenced in message history by **full relative file path**.

Reflow must not inline full artifact bodies into context by default.

### 17.5 Historical outputs

Earlier attempts remain available as history even if no longer current.
The provider is expected to infer current relevance from message order, recent step outputs, and artifact references.

## 18. Nested workflows

Nested workflows are fully in scope for v0.1.

A step with `action.kind: workflow` launches a child run.

### 18.1 Child run behavior

The child run:

* has its own run folder
* has its own `state.json` and message ledger
* records its parent run and parent step reference

### 18.2 Parent/child linking

The parent attempt records:

* `child_run_id`
* `child_workflow_name`

### 18.3 Child context

Subworkflows support `context_mode`.

In v0.1, `context_mode` defaults to `full_run`, meaning the child inherits the full visible parent message history at launch and then appends its own messages during execution.

### 18.4 Parent completion rule

A workflow step completes when its child run reaches a terminal state.

The parent sees:

* the child run’s full visible message history
* child run id
* child terminal status
* child state reference

### 18.5 Child semantics

A nested workflow step supports:

* `success.when`
* `failure.when`
* `routes`

exactly like any other step.

### 18.6 Child retry behavior

If a parent step invoking a child workflow is retried, the child workflow is executed again as a new child run. No special retry semantics exist for nested workflows in v0.1.

## 19. Route evaluation model

Route evaluation is externalized.

After each attempt, Reflow must run **one** semantic evaluation pass that evaluates:

* local step success
* local step failure
* all local routes in order

The evaluator must return JSON.

### 19.1 `route_eval.json`

Each attempt produces a `route_eval.json` artifact with at least:

* `step_success`
* `step_failure`
* `matched_routes`
* `selected_route`
* `reason`
* `confidence`

### 19.2 Contract notes

* `selected_route` must equal the first entry in `matched_routes`
* `selected_route` may be `null` only when no route matched
* `matched_routes` must preserve route order
* `reason` should reference current evidence from persisted artifacts
* `confidence` is advisory only and not used for routing in v0.1

### 19.3 Evaluation order

For each attempt:

1. Execute the step
2. Check deterministic failures
3. Run one semantic evaluation pass
4. Persist `route_eval.json`
5. Resolve next transition

### 19.4 Conflict rule

If `step_success = true` and `step_failure = true`, treat the result as `route_eval_conflict`.

Retry using the **retry budget** for the current semantic visit.
If retries are exhausted, fail the run.

This conflict does **not** consume `max_iter`.

## 20. Deterministic checks before semantic routing

Reflow must first evaluate deterministic runtime facts.

The evaluation sequence is:

1. Attempt step execution
2. If execution-stage failure occurred, apply retry policy directly and skip semantic evaluation
3. Check whether required outputs exist
4. If required outputs are missing, apply retry policy directly and skip semantic evaluation
5. Check whether the next semantic visit would exceed `max_iter`
6. Then proceed to semantic evaluation

The key rule is: if the provider process crashed, timed out, or failed to produce the required outputs, there is nothing meaningful to semantically evaluate.

### 20.1 Semantic visit increment rule

When workflow routing selects a step, Reflow begins a new **semantic visit**.

Before executing that visit, it checks whether `semantic_iteration_number + 1 > max_iter`.

* If yes, the run fails with `max iteration exceeded`
* If no, it increments `semantic_iteration_number` and starts attempt `retry_number = 0`

Retries within that visit do not increment `semantic_iteration_number`.

### 20.2 Error and diagnostic messaging

On every runtime failure Reflow detects, it must append a descriptive error message to all three of:

* `messages.jsonl`
* `attempt.json`
* `state.json`

Error messages must state:

* what failed, using a specific `failure_type`
* whether a retry will occur
* remaining retry count for this semantic visit
* current semantic iteration usage (for example, `semantic visit 2 of 3`)

Reflow must never silently swallow a failure. If something goes wrong, there must be a written record in all three locations.

## 21. Retry policy

### 21.1 Scope

Retry is for **execution-stage and evaluator-stage failures that prevent a clean semantic outcome**.

It protects the semantic iteration budget (`max_iter`) from being consumed by infrastructure noise or evaluator malfunction.

### 21.2 Independence

`retry` and `max_iter` are independent counters.

* `max_iter` counts how many times the workflow semantically routes to a step
* `retry.max_retries` counts how many times one semantic visit can recover from execution-stage or evaluator-stage failure

The retry counter resets to its full budget at the start of each new semantic visit.

Worst-case attempt count for one step:

`max_iter × (1 + max_retries)`

### 21.3 Provider unavailable

Provider unavailable bypasses retry and fails fast.

### 21.4 Retryable conditions

The following conditions consume from the retry budget:

* missing required outputs
* malformed structured output from provider
* malformed `route_eval.json`
* `route_eval_conflict` (`step_success` and `step_failure` both true)
* ambiguous evaluation (`step_success` and `step_failure` both false)
* routes defined but none match
* provider process crash or timeout
* local script crash or timeout

### 21.5 Non-retryable conditions

The following are not retryable by the kernel:

* provider unavailable → fail fast
* semantic failure (`failure.when = true` from a valid, non-conflicting evaluation)
* `max_iter` exhaustion → fail

### 21.6 Retry artifacts

Each retry creates:

* a new attempt folder
* a new `attempt.json`
* a new error entry
* a new retry-reason message in `messages.jsonl`

### 21.7 Retry exhaustion

If retries are exhausted for a semantic visit and the condition persists, the run fails.

`failure_type` should distinguish the cause, for example:

* `retries_exhausted_missing_outputs`
* `retries_exhausted_route_conflict`
* `retries_exhausted_ambiguous_evaluation`

### 21.8 No-routes-defined rule

If a step defines no routes:

* on semantic success, proceed to the next step in workflow order
* if it is the last step, resolve terminal status as `completed`
* on semantic failure, fail
* if neither success nor failure is true, treat it as ambiguous evaluation and use the retry budget

Ambiguous evaluations are evaluator malfunction. They do **not** consume `max_iter`.

## 22. Transition semantics

`then: stop` is a transition action, not a stored run state.

If `selected_route = stop`:

* and `step_failure = false`, terminal status resolves to `completed`
* and `step_failure = true`, terminal status resolves to `failed`

If `selected_route = escalate`, terminal status resolves to `escalated`.

`stop` must be logged both in:

* `route_eval.json` as `selected_route: "stop"`
* `messages.jsonl` as a transition message that records the terminal status resolution

Failure and route precedence:

* if `step_success = true` and `step_failure = true` → apply retry policy
* if `step_failure = true` and `selected_route = escalate` → `escalated`
* if `step_failure = true` and no escalation route applies → `failed`
* otherwise follow `selected_route`

## 23. Runtime artifact structure

Each run must contain:

* `state.json`
* `messages.jsonl`
* one folder per step

Each step folder contains one folder per attempt.

Each attempt contains:

* `input.json`
* `attempt.json`
* `route_eval.json`
* `logs/`
* `output/`

### 23.1 `state.json`

Contains:

* run id
* workflow name
* workflow version or hash
* current status
* current step
* semantic visit counts per step
* current retry counts per step for the active semantic visit
* start and end timestamps
* workspace path
* parent linkage if child run
* current failure type and reason, if any

`state.json` is the mutable current-state snapshot, not a historical log.

### 23.2 `messages.jsonl`

Canonical append-only message history for the run.

This is the source of truth for `full_run` context and should be replayable into provider calls.

Retry-reason messages are part of the canonical ledger and must be visible to the provider on the next attempt.

Each message record must include at least:

* `seq`
* `ts`
* `role`
* `kind`
* `text`
* optional `artifact_paths`

Recommended additional fields:

* `workflow`
* `step`
* `attempt`

### 23.3 `input.json`

Records:

* action
* provider selected
* context mode
* required outputs
* success and failure conditions
* local routes
* referenced artifact paths

### 23.4 `attempt.json`

Records:

* semantic_iteration_number
* retry_number (`0` for first execution of a semantic visit)
* retry_remaining
* step name
* provider
* timestamps
* execution status
* retry reason if applicable
* failure_type if applicable
* failure_reason if applicable
* output references
* child workflow reference if applicable

### 23.5 `output/`

Contains domain outputs of that attempt.

### 23.6 `logs/`

Contains provider-specific raw output or local execution logs.

## 24. Workflow notes and schemas

### 24.1 `notes.md`

Optional, workflow-local.
Appended to the orchestrator message history for that workflow.

### 24.2 Local schemas

Optional, workflow-local only.
Used only when a workflow needs stronger structured validation.

There is no central schema registry in v0.1.

## 25. CLI actions

`action.kind: cli` invokes a local script or command alias.

`action.with` may include:

* `cwd`
* `timeout_sec`
* small executor-specific parameters

### 25.1 CLI context delivery

For CLI steps, Reflow must materialize context as:

* message ledger path
* state path
* relevant artifact paths
* output directory path
* working directory

This is typically provided through files and environment variables, not prompt context.

## 26. Minimal terminal UX

Reflow is terminal-first in v0.1.

Required terminal capabilities:

* start a workflow run
* show current status
* display failures clearly
* show retry reasons
* inspect run artifact paths
* issue an operator stop command

An operator stop terminates the run with terminal status `escalated`.

### 26.1 CLI exit codes

The implementation must differentiate CLI exit categories.

Recommended default codes:

| Code | Meaning                                                                                               |
| ---- | ----------------------------------------------------------------------------------------------------- |
| 0    | Run completed successfully                                                                            |
| 20   | Provider unavailable (fail fast)                                                                      |
| 21   | Terminal step failure, including semantic failure and execution-stage failure after retries exhausted |
| 22   | Unresolved routing, route conflict, or ambiguous evaluation after retries exhausted                   |
| 23   | Max iteration exceeded                                                                                |
| 24   | Escalated (operator stop or workflow escalation)                                                      |
| 25   | Internal orchestrator or runtime error                                                                |

Exact numeric values are not mandatory, but stable differentiation between these categories **is** mandatory and must be documented per deployment.

## 27. Acceptance criteria

Reflow is acceptable when it can do all of the following.

### 27.1 Repository scaffold

Create and load:

* `.ai/config/`
* `.ai/workflows/`
* `.ai/skills/`
* `.ai/scripts/`
* `.ai/runs/`

### 27.2 Concrete workflow execution

Run the `review_analyze_do` workflow end to end.

### 27.3 Immutable attempts

Produce separate attempt folders for looped steps.

### 27.4 Semantic routing

Evaluate success, failure, and all local routes in one pass and persist `route_eval.json`.

### 27.5 Retry behavior

Retry execution-stage failures (provider timeout, malformed response, malformed evaluator output, route conflicts, ambiguous evaluations, unresolved routes, missing outputs) using the step-level retry budget without consuming `max_iter`. Append retry reasons as messages to the canonical ledger. Verify that `max_iter` is only consumed by valid semantic revisits driven by workflow routing, never by evaluator malfunction.

### 27.6 Provider failure behavior

Fail fast when the configured provider is unavailable.

### 27.7 Nested workflows

Run a child workflow and persist separate parent and child run artifacts.

### 27.8 Context behavior

Use full visible run context by default, with context represented as canonical message history plus artifact path references.

### 27.9 Error diagnostics

Every runtime failure appends descriptive error messages to `messages.jsonl`, `attempt.json`, and `state.json`, including failure type, retry state, and iteration budget usage.

### 27.10 CLI exit codes

Reflow returns differentiated exit codes for success, provider unavailable, terminal step failure, routing failure, max iteration exceeded, escalation, and internal runtime error.

## 28. Risks

### 28.1 Context growth

Full-run message history is intentionally simple but will eventually grow large. This is accepted in v0.1.

### 28.2 Route ambiguity

Natural-language conditions can drift if written vaguely.

Mitigation:
use controlled natural language and preserve route-evaluation artifacts.

### 28.3 Provider mismatch

Codex and Claude do not expose identical native surfaces.

Mitigation:
keep authored workflows provider-agnostic and isolate provider behavior in config and adapters.

### 28.4 Retry budget interaction

The worst-case attempt count for a step is:

`max_iter × (1 + max_retries)`

For a step with `max_iter: 5` and `max_retries: 3`, this is 20 attempts.

Mitigation:

* keep defaults conservative (`max_retries: 1`)
* surface current retry and iteration counts in `state.json`
* document the interaction clearly in workflow authoring guidance

## 29. Roadmap beyond v0.1

Likely future work:

* context pruning and compilation
* richer CLI action contract
* optional orchestrator-level approvals
* optional provider-native skill mirrors
* richer child-output mapping for nested workflows
* additional workflows outside code review
* optional daemon or service mode if local CLI proves too limiting

## 30. Final product position

Reflow v0.1 is a deterministic, file-first, terminal-native runtime for reusable AI workflows. It uses current Codex and Claude CLI capabilities for execution, but keeps workflow state, routing, retries, and attempt history external to the providers. It models full-run context as a canonical message ledger plus artifact path references, not as repeated full-file prompt dumps. It prefers explicit orchestration over ambient agent behavior, preserves every attempt as inspectable history, and intentionally separates execution-stage retry from semantic iteration budgets so infrastructure flakiness does not consume workflow convergence capacity.
