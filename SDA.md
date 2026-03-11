Reflow v0.1 — Architectural Specification

1. Purpose and normative status

This document is the normative architecture for Reflow v0.1.

It defines the module boundaries, runtime responsibilities, data contracts, persistence rules, provider integration contracts, retry and routing semantics, nested-workflow behavior, resume behavior, testing strategy, and implementation order required to implement Reflow without ambiguity.

Unless a statement is explicitly labeled guidance or future work, it is normative.

Reflow v0.1 is a single-process, single-threaded CLI program. It reads workflow definitions from disk, invokes provider CLIs or local commands as subprocesses, evaluates step outcomes through one semantic evaluation pass per attempt, and writes canonical run artifacts under .ai/runs/. There is no daemon, no service backend, no database, and no Reflow-owned network API.

Normative keywords mean:
	•	MUST = required for conformance
	•	SHOULD = recommended default
	•	MAY = optional

2. Architectural principles

Reflow v0.1 MUST preserve these principles:
	1.	Kernel determinism
The orchestrator owns run state, counters, transitions, and terminal states. Providers perform bounded step work.
	2.	File-first state
Every meaningful result is persisted as files and append-only logs.
	3.	Visible persisted context for orchestrator correctness
Reflow correctness depends on messages.jsonl, on-disk artifacts, workflow/config/skill files, and explicit runtime state. Reflow does not depend on hidden provider chain-of-thought or hidden provider session state.
	4.	Immutable attempts
Every retry and every semantic revisit creates a new attempt folder. Attempts are never overwritten.
	5.	Provider-agnostic workflows
Workflow authors do not select providers inside workflow definitions.
	6.	Retry / iteration separation
Retry budget protects semantic iteration budget. These counters are independent.
	7.	Fail loud
Every runtime failure is written to the ledger, the attempt record, and the current run state.
	8.	KISS
v0.1 only implements abstractions with a concrete consumer.
	9.	DRY
Shared constants, failure types, model contracts, transition rules, and file-layout logic each live in one place.

3. Scope

3.1 In scope
	•	workflow execution for skill, workflow, and cli actions
	•	deterministic run / step / semantic-visit / attempt lifecycle
	•	execution-stage and evaluator-stage retry behavior
	•	route evaluation and transition resolution
	•	canonical run artifacts
	•	Codex CLI and Claude Code CLI adapters
	•	nested workflows
	•	Reflow-native resume of interrupted runs

3.2 Out of scope
	•	orchestrator-level human approvals
	•	graph runtime or parallel branches
	•	context pruning or compilation
	•	provider fallback
	•	a central schema registry
	•	web UI or service backend
	•	provider-native skill libraries as a required dependency
	•	provider-native resume as a correctness dependency
	•	hidden-memory dependence
	•	schema validation for general step outputs in v0.1

4. Technology stack

Recommended implementation stack:

Concern	Choice
Language	Python 3.11+
CLI	click
YAML	PyYAML
JSON/JSONL	stdlib json
Process control	stdlib subprocess
Models	dataclasses
Paths	pathlib
Hashing	hashlib
Tests	pytest
Lint/format	ruff
Types	mypy in strict mode

No async framework, ORM, template engine, or HTTP server is required in v0.1.

5. Verified provider contract baseline

Codex CLI documents codex exec as its non-interactive automation surface, including --cd, --json, --output-last-message, --output-schema, --sandbox, --ask-for-approval, --skip-git-repo-check, --ephemeral, and codex exec resume. The docs also state that Codex requires a Git repository by default and can be run outside one with --skip-git-repo-check, and that Codex reads AGENTS.md files as native project instructions.  ￼

Claude Code documents claude -p for print-mode/programmatic execution, --output-format json, --json-schema, --allowedTools, --disallowedTools, --disable-slash-commands, --max-turns, --permission-mode, --resume, and --no-session-persistence. Claude’s memory docs state that auto memory is on by default and can be disabled with CLAUDE_CODE_DISABLE_AUTO_MEMORY=1.  ￼

Therefore, Reflow v0.1 MUST follow these provider-integration rules:
	1.	Reflow MUST NOT rely on provider-native resume or session continuity for correctness.
	2.	Reflow MUST NOT manually append AGENTS.md, CLAUDE.md, .claude/rules/*, or provider-native skills into prompts; those are provider-native instruction layers.
	3.	Reflow MUST support Reflow-native resume from persisted artifacts.
	4.	Reflow MUST NOT force-disable provider web search by default.
	5.	Reflow MUST NOT force Codex --ephemeral in v0.1.
	6.	Reflow MAY expose provider-native flags that control persistence, memory, tools, or permissions, but those controls are deployment choices, not correctness requirements.

6. Repository layout

Canonical repository layout:

.ai/
  config/
    orchestrator.yaml
    providers.yaml
    route_eval.schema.json

  workflows/
    <workflow_name>/
      workflow.yaml
      notes.md                  # optional
      schemas/                  # optional, workflow-local only

  skills/
    <skill_name>/
      SKILL.md

  scripts/
    verify.sh

  runs/
    <run_id>/
      state.json
      messages.jsonl
      <step folders>/

Rules:
	•	.ai/skills is the only canonical skill source.
	•	Reflow MUST NOT depend on .agents/skills or .claude/skills.
	•	.ai/ contains orchestration-owned assets only.

7. Package architecture

Recommended package layout:

reflow/
  __init__.py
  __main__.py

  contracts.py
  errors.py
  models.py

  config.py
  loader.py

  store.py
  ledger.py
  context.py

  retry.py
  routing.py
  nested.py
  runtime.py

  executors/
    __init__.py
    base.py
    skill.py
    workflow.py
    cli.py

  providers/
    __init__.py
    base.py
    resolver.py
    codex.py
    claude.py

  cli/
    __init__.py
    main.py
    exit_codes.py

7.1 Responsibilities

contracts.py
Canonical enums, message kinds, status values, failure values, and constant strings.

errors.py
Typed exceptions and pure failure helpers.

models.py
All dataclasses and typed contracts.

config.py
Loads and validates orchestrator.yaml and providers.yaml.

loader.py
Loads and validates workflows, notes, local schemas, and skills.

store.py
Creates folders, resolves canonical paths, writes JSON artifacts atomically.

ledger.py
Appends and reads messages.jsonl.

context.py
Builds step instructions, provider context, and evaluator prompts.

retry.py
Retryability classification and retry-counter management.

routing.py
Evaluator invocation, parse/validation, and transition resolution.

nested.py
Child-run creation, parent-child linkage, child-context visibility, and export-dir propagation.

runtime.py
Composition root and main execution loop.

executors/*
Execution strategy by step kind.

providers/*
Provider-neutral to provider-specific invocation mapping.

cli/*
Terminal commands and exit-code mapping.

7.2 Dependency rules
	•	contracts.py, errors.py, and models.py MUST NOT depend on any internal modules.
	•	runtime.py is the composition root.
	•	cli/* depends on runtime and loaders, not vice versa.
	•	executors/* MUST NOT depend on runtime.py.
	•	providers/* MUST NOT depend on runtime.py.
	•	Circular dependencies are forbidden.

Valid dependency shape:

cli/main.py
  -> config.py, loader.py, runtime.py, cli/exit_codes.py

runtime.py
  -> contracts.py, errors.py, models.py
  -> store.py, ledger.py, context.py
  -> retry.py, routing.py, nested.py
  -> executors/base.py

executors/skill.py
  -> providers/resolver.py, context.py

executors/workflow.py
  -> nested.py

executors/cli.py
  -> subprocess + contracts/models/store helpers

routing.py
  -> providers/resolver.py, context.py

8. Canonical constants

contracts.py is the single source of truth for canonical string values.

from enum import StrEnum

class RunStatus(StrEnum):
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    ESCALATED = "escalated"

class ActionKind(StrEnum):
    SKILL = "skill"
    WORKFLOW = "workflow"
    CLI = "cli"

class ContextMode(StrEnum):
    FULL_RUN = "full_run"

class RouteTarget(StrEnum):
    STOP = "stop"
    ESCALATE = "escalate"

class LedgerRole(StrEnum):
    ORCHESTRATOR = "orchestrator"
    PROVIDER = "provider"
    EVALUATOR = "evaluator"

class MessageKind(StrEnum):
    WORKFLOW_SETUP = "workflow_setup"
    WORKFLOW_NOTES = "workflow_notes"
    INHERITED_CONTEXT = "inherited_context"
    STEP_INSTRUCTION = "step_instruction"
    PROVIDER_RESPONSE = "provider_response"
    CLI_RESULT = "cli_result"
    ROUTE_EVAL_RESULT = "route_eval_result"
    RETRY_REASON = "retry_reason"
    TRANSITION = "transition"
    FAILURE = "failure"
    CHILD_CONTEXT_LINK = "child_context_link"
    RUN_COMPLETE = "run_complete"
    RUN_ESCALATED = "run_escalated"

class FailureType(StrEnum):
    PROVIDER_UNAVAILABLE = "provider_unavailable"
    EXECUTION_FAILURE = "execution_failure"
    PROVIDER_TIMEOUT = "provider_timeout"
    MISSING_OUTPUTS = "missing_outputs"
    MALFORMED_RESPONSE = "malformed_response"
    MALFORMED_ROUTE_EVAL = "malformed_route_eval"
    INVALID_ROUTE_EVAL = "invalid_route_eval"
    ROUTE_EVAL_CONFLICT = "route_eval_conflict"
    AMBIGUOUS_EVALUATION = "ambiguous_evaluation"
    ROUTE_UNRESOLVED = "route_unresolved"
    SEMANTIC_FAILURE = "semantic_failure"
    MAX_ITER_EXCEEDED = "max_iter_exceeded"
    CONFIG_ERROR = "config_error"
    WORKFLOW_LOAD_ERROR = "workflow_load_error"
    FILESYSTEM_ERROR = "filesystem_error"
    INTERNAL_ERROR = "internal_error"

STATE_SCHEMA_VERSION = 1

Retry-exhaustion failure types use:

retries_exhausted_<base_cause>

Examples:
	•	retries_exhausted_execution_failure
	•	retries_exhausted_missing_outputs
	•	retries_exhausted_route_unresolved

9. Data models

All models SHOULD use dataclasses. Immutable definitions SHOULD use frozen=True.

9.1 Configuration models

@dataclass(frozen=True)
class OrchestratorConfig:
    run_id_format: str
    default_context_mode: str
    default_max_retries: int
    unresolved_route_behavior: str

@dataclass(frozen=True)
class ProviderMapping:
    router: str
    skills: dict[str, str]
    steps: dict[str, str]
    profiles: dict[str, dict]

@dataclass(frozen=True)
class Config:
    orchestrator: OrchestratorConfig
    providers: ProviderMapping
    config_hash: str

9.2 Workflow models

@dataclass(frozen=True)
class ActionDef:
    kind: str
    ref: str
    with_params: dict | None

@dataclass(frozen=True)
class RouteDef:
    when: str
    then: str

@dataclass(frozen=True)
class RetryPolicy:
    max_retries: int

@dataclass(frozen=True)
class StepDef:
    name: str
    description: str | None
    action: ActionDef
    max_iter: int
    retry: RetryPolicy
    context_mode: str
    required_outputs: list[str]
    success_when: str
    failure_when: str
    routes: list[RouteDef]
    skill_text: str | None

@dataclass(frozen=True)
class WorkflowDef:
    name: str
    description: str | None
    entrypoint: str
    steps: list[StepDef]
    steps_by_name: dict[str, StepDef]
    step_order: list[str]
    notes: str | None
    workflow_hash: str

9.3 Run-state models

@dataclass
class StepState:
    semantic_visit_count: int
    current_retry_number: int
    current_retry_remaining: int
    attempt_count: int

@dataclass
class RunState:
    schema_version: int
    run_id: str
    workflow_name: str
    workflow_hash: str
    config_hash: str
    status: str
    current_step: str | None
    step_states: dict[str, StepState]
    started_at: str
    ended_at: str | None
    workspace_path: str
    parent_run_id: str | None
    parent_step: str | None
    export_dir_path: str | None
    failure_type: str | None
    failure_reason: str | None

9.4 Execution-result models

@dataclass(frozen=True)
class ProviderResult:
    ok: bool
    exit_code: int | None
    timed_out: bool
    unavailable: bool
    final_text: str | None
    structured_output: dict | None
    duration_sec: float
    log_paths: list[str]
    failure_type: str | None
    failure_reason: str | None

@dataclass(frozen=True)
class ExecutionResult:
    executor_kind: str
    stage_ok: bool
    failure_type: str | None
    failure_reason: str | None
    provider_name: str | None
    final_text: str | None
    output_paths: list[str]
    log_paths: list[str]
    timed_out: bool
    child_run_id: str | None
    child_workflow_name: str | None
    child_status: str | None
    child_state_path: str | None
    child_messages_path: str | None

9.5 Artifact models

@dataclass(frozen=True)
class InputRecord:
    action: ActionDef
    provider_or_executor: str
    context_mode: str
    required_outputs: list[str]
    success_when: str
    failure_when: str
    routes: list[RouteDef]
    artifact_paths: list[str]
    export_dir_path: str | None

@dataclass
class AttemptRecord:
    step_name: str
    attempt_number: int
    semantic_iteration_number: int
    retry_number: int
    retry_remaining: int
    executor_kind: str
    provider_name: str | None
    started_at: str
    ended_at: str | None
    execution_status: str          # running | success | execution_failure | timeout | interrupted
    route_eval_status: str         # not_started | skipped | failed | completed
    route_eval_skip_reason: str | None
    transition_status: str         # not_started | resolved | applied
    transition_target: str | None
    failure_type: str | None
    failure_reason: str | None
    retry_reason: str | None
    output_paths: list[str]
    log_paths: list[str]
    child_run_id: str | None
    child_workflow_name: str | None
    child_status: str | None
    child_state_path: str | None
    child_messages_path: str | None

@dataclass(frozen=True)
class RouteEval:
    step_success: bool
    step_failure: bool
    matched_routes: list[str]
    selected_route: str | None
    reason: str
    confidence: float

@dataclass(frozen=True)
class Message:
    seq: int
    ts: str
    role: str
    kind: str
    text: str
    workflow: str | None
    step: str | None
    attempt_num: int | None
    artifact_paths: list[str]
    linked_run_id: str | None

10. Configuration loading and validation

10.1 Responsibilities

config.py MUST:
	•	load .ai/config/orchestrator.yaml
	•	load .ai/config/providers.yaml
	•	apply defaults
	•	validate structure
	•	compute config_hash
	•	return Config

10.2 Defaults

run_id_format: "run_{timestamp}_{workflow}"
default_context_mode: "full_run"
default_max_retries: 1
unresolved_route_behavior: "fail"

10.3 providers.yaml shape

router: claude

skills:
  review-codebase: codex
  screen-review-findings: claude

steps:
  review_analyze_do.implement_screened_findings: codex

profiles:
  codex:
    model: gpt-5.4
    timeout_sec: 1800
    sandbox: workspace-write
    ask_for_approval: never
    skip_git_repo_check: false
    json_events: false
    extra_args: []

  claude:
    model: sonnet
    timeout_sec: 1800
    tools: null
    allowed_tools: []
    disallowed_tools: []
    permission_mode: null
    max_turns: 12
    max_budget_usd: null
    disable_slash_commands: false
    no_session_persistence: false
    disable_auto_memory: false
    extra_args: []

10.4 Provider resolution order

For skill steps:
	1.	providers.steps["<workflow_name>.<step_name>"]
	2.	providers.skills["<skill_name>"]

For route evaluation:
	•	providers.router

10.5 Validation rules

Validation MUST fail before execution if:
	•	config files are missing or malformed
	•	default_context_mode is not full_run
	•	router provider is missing
	•	a referenced skill step has no provider mapping
	•	a mapped provider name is unknown
	•	profile field types are invalid
	•	extra_args contains flags that conflict with adapter-managed flags for that provider

Examples of forbidden conflicting extra_args:
	•	Codex: --cd, --output-last-message, --output-schema
	•	Claude: -p, --print, --output-format, --json-schema

Reflow MUST NOT perform proactive provider-availability probes.

Missing mapping is a config error. Binary absence, authentication failure, or startup failure is an invocation-time provider-unavailable event.

11. Workflow loading and validation

loader.py MUST:
	1.	load workflow.yaml
	2.	load notes.md if present
	3.	load workflow-local schema files if present
	4.	load SKILL.md for each skill step
	5.	apply retry defaults
	6.	validate structure
	7.	compute workflow_hash
	8.	return WorkflowDef

workflow_hash MUST be derived from all workflow-definition inputs that affect behavior:
	•	workflow.yaml
	•	notes.md if present
	•	workflow-local schema files if present
	•	referenced SKILL.md contents for skill steps

Validation MUST fail if:
	•	required workflow fields are missing
	•	steps is empty
	•	duplicate step names exist
	•	entrypoint references an unknown step
	•	action.kind is invalid
	•	action.ref is missing
	•	context_mode is present and not full_run
	•	max_iter < 1
	•	retry.max_retries < 0
	•	a route target is neither a known step nor stop nor escalate
	•	required output paths are absolute
	•	required output paths contain ..
	•	a referenced skill file does not exist
	•	a referenced child workflow does not exist

12. Run storage and artifact semantics

12.1 Run ID generation

Default run IDs use UTC timestamp plus workflow slug:

run_20260310T120000_review_analyze_do

If the path already exists, append a numeric suffix.

12.2 Canonical layout

.ai/runs/{run_id}/
  state.json
  messages.jsonl
  010_review_codebase/
    attempt_001/
      input.json
      attempt.json
      route_eval.json          # only when semantic evaluation ran
      output/
      logs/
    attempt_002/
  020_screen_findings/

12.3 Step folder naming

Step folders are named from workflow declaration order:
	•	first step: 010_<step>
	•	second step: 020_<step>

A step always reuses its existing folder when revisited.

12.4 Attempt folder naming

Attempt numbers are continuous per step across all semantic visits and retries.

12.5 Atomic writes

The runtime MUST write these files atomically:
	•	state.json
	•	input.json
	•	attempt.json
	•	route_eval.json

Write procedure:
	1.	write temp file in same directory
	2.	flush
	3.	fsync
	4.	os.replace()

messages.jsonl MUST be appended line-by-line with flush and fsync after each append.

12.6 state.json

state.json is the mutable run snapshot and MUST be rewritten after:
	•	run creation
	•	step entry
	•	attempt start
	•	attempt completion
	•	retry decision
	•	transition resolution
	•	terminal completion / failure / escalation
	•	resume recovery mutations

12.7 attempt.json

attempt.json MUST be written at attempt start with:
	•	execution_status = "running"
	•	route_eval_status = "not_started"
	•	transition_status = "not_started"

It MUST be updated on every attempt outcome.

12.8 route_eval.json

route_eval.json MUST exist only if semantic evaluation actually ran.

If semantic evaluation was skipped, attempt.json MUST record:
	•	route_eval_status = "skipped"
	•	route_eval_skip_reason

If semantic evaluation ran but the evaluator output was malformed or invalid, attempt.json MUST record:
	•	route_eval_status = "failed"

12.9 Required-output semantics

Reflow v0.1 validates required step outputs by existence only. General step-output schema validation is out of scope in v0.1. route_eval.json validation is separate and required.

For skill and cli steps:
	•	outputs.required paths are relative to the current attempt’s output/ directory.

For workflow steps:
	•	outputs.required paths are also relative to the parent attempt’s output/ directory
	•	the child workflow execution path is responsible for materializing those outputs there before the child reaches terminal state
	•	the parent runtime checks those outputs exactly like any other step

The nested workflow executor MUST still synthesize child_run_ref.json into the parent attempt output directory, whether or not it appears in outputs.required.

If the workflow executor cannot write child_run_ref.json, the failure MUST be classified as filesystem_error, not missing_outputs.

13. Message ledger

messages.jsonl is the canonical append-only run history.

It is the source of truth for full_run context together with on-disk artifacts.

13.1 Required message kinds

The runtime MUST append messages for:
	•	workflow setup
	•	workflow notes
	•	inherited context boundaries
	•	step instructions
	•	provider responses
	•	CLI result summaries
	•	route evaluation results
	•	retry reasons
	•	transitions
	•	failures
	•	child context links
	•	run completion
	•	run escalation

13.2 Record contract

Each record MUST include:
	•	seq
	•	ts
	•	role
	•	kind
	•	text

It SHOULD also include:
	•	workflow
	•	step
	•	attempt_num
	•	artifact_paths
	•	linked_run_id

13.3 Append behavior

Each append MUST:
	1.	assign the next seq
	2.	generate ISO 8601 UTC timestamp
	3.	serialize one JSON object
	4.	append one line
	5.	flush and fsync

No locking is required in v0.1 because Reflow is single-process and single-writer.

14. Context assembly

14.1 full_run

full_run means the provider receives:
	1.	the canonical message history for the current run
	2.	references to relevant artifacts on disk
	3.	the current step instruction

It does not mean auto-inlining every artifact body.

14.2 Step instruction contents

The current step instruction MUST include:
	•	workflow name
	•	step name
	•	step description if present
	•	output directory path
	•	required outputs
	•	success.when
	•	failure.when
	•	ordered local routes
	•	skill text for skill steps
	•	relevant artifact paths
	•	export directory path when run_state.export_dir_path is non-null

14.3 Relevant artifact selection

Artifact references for a step MUST be selected deterministically:
	1.	latest output artifacts from the most recent completed attempt of every previously visited step
	2.	all prior output and route_eval.json paths for the current step
	3.	all prior child_context_link artifacts
	4.	the current run’s state.json
	5.	the current run’s messages.jsonl
	6.	the current attempt’s output/ directory
	7.	if present, the child export directory path

If a prior parent step completed a child workflow, later parent steps under full_run MUST include the child ledger path and child state path among their artifact references.

14.4 External provider context

Reflow does not override provider web-search configuration by default.

If a provider consults external web context or provider-side retrieval during a step, Reflow does not mirror that external material into its own artifact model automatically. Workflows that need durable downstream use of that evidence SHOULD instruct the provider to persist the relevant evidence, summaries, or references into step outputs. Codex documents web-search behavior as provider-side configuration, and Claude documents tool-rich print-mode execution surfaces rather than a Reflow-controlled isolation model.  ￼

14.5 Provider context serialization

context.py MUST serialize the ledger into a stable prompt string for provider worker calls.

A valid format is:

--- [orchestrator | workflow_setup] ---
<text>

--- [provider | provider_response] ---
<text>
Artifacts:
- path/to/file

--- [orchestrator | step_instruction] ---
<current step instruction>

The format MAY evolve, but order and artifact references MUST be preserved.

14.6 Route-evaluation prompt

The evaluator prompt MUST include:
	•	evaluator role statement
	•	step contract
	•	success.when
	•	failure.when
	•	ordered routes
	•	attempt evidence:
	•	provider final text or CLI summary
	•	output paths
	•	child terminal status and child paths for workflow steps
	•	semantic iteration number
	•	retry number
	•	prior evaluations for this step
	•	expected JSON output contract

The evaluator prompt SHOULD ground its request in the run’s recorded evidence, but Reflow does not restrict provider-native retrieval, tools, or external context in v0.1.

14.7 Working-directory semantics

skill and workflow actions run from the directory where Reflow was launched. cli actions may override via action.with.cwd. This working-directory stability matters because Codex documents --cd as setting the working directory and reads AGENTS.md from project scope, while Claude loads CLAUDE.md and memory at session start from project context.  ￼

15. Executors

15.1 Executor protocol

Each executor MUST implement an interface equivalent to:

class Executor(Protocol):
    def execute(
        self,
        step: StepDef,
        workflow: WorkflowDef,
        run_state: RunState,
        config: Config,
        ledger: Ledger,
        store: Store,
        workspace: Path,
        attempt_dir: Path,
        output_dir: Path,
    ) -> ExecutionResult: ...

15.2 Skill executor

The skill executor MUST:
	1.	resolve the provider
	2.	build step instruction and provider context
	3.	invoke the provider adapter
	4.	capture logs
	5.	return ExecutionResult

15.3 CLI executor

The CLI executor MUST:
	•	invoke the command with shell=False
	•	treat action.ref as executable path or PATH name
	•	take arguments only from action.with.args as a list

It MUST pass these environment variables:
	•	REFLOW_WORKFLOW_NAME
	•	REFLOW_RUN_ID
	•	REFLOW_STEP_NAME
	•	REFLOW_SEMANTIC_ITERATION
	•	REFLOW_RETRY_NUMBER
	•	REFLOW_WORKSPACE_ROOT
	•	REFLOW_RUN_DIR
	•	REFLOW_ATTEMPT_DIR
	•	REFLOW_OUTPUT_DIR
	•	REFLOW_STATE_PATH
	•	REFLOW_MESSAGES_PATH
	•	REFLOW_ARTIFACT_PATHS_JSON

If run_state.export_dir_path is non-null, it MUST also pass:
	•	REFLOW_EXPORT_DIR

15.4 Workflow executor

The workflow executor MUST:
	1.	create a child run
	2.	pass the parent ledger snapshot into the child
	3.	pass the parent attempt output directory as the child export directory
	4.	wait for child terminal state
	5.	synthesize output/child_run_ref.json in the parent attempt output directory
	6.	return ExecutionResult

Critical rule:
	•	a child run reaching any terminal state (completed, failed, escalated) counts as a completed execution stage for the parent workflow step
	•	the child terminal state is semantic evidence for the parent step
	•	child semantic failure is not automatically converted into parent execution-stage failure

Only child launch failures, child readback failures, or child runtime-infrastructure failures before a readable terminal state count as parent execution-stage failures.

16. Provider adapters

16.1 Common provider protocol

Each provider adapter MUST implement an interface equivalent to:

class Provider(Protocol):
    def execute(
        self,
        prompt: str,
        working_dir: Path,
        logs_dir: Path,
        timeout_sec: int | None = None,
        schema_path: Path | None = None,
        mode: str = "worker",   # worker | router
    ) -> ProviderResult: ...

There is no check_available() preflight.

16.2 Codex adapter

The Codex adapter MUST use codex exec and the documented automation flags for working directory, structured final output, schema-constrained output, sandboxing, approval behavior, and optional non-git execution. Reflow v0.1 does not use codex exec resume for correctness, does not force --ephemeral, and does not override web-search settings by default. Codex’s AGENTS.md behavior remains provider-native.  ￼

Recommended worker invocation:

codex exec \
  --cd <working_dir> \
  --sandbox <sandbox> \
  --ask-for-approval <approval_mode> \
  [--skip-git-repo-check] \
  [--model <model>] \
  [--output-last-message <logs_dir/final_message.txt>] \
  <prompt>

If json_events: true, the adapter MAY also add --json and preserve event logs. That is for forensics only.

Recommended router invocation:

codex exec \
  --cd <working_dir> \
  [--sandbox <sandbox>] \
  [--ask-for-approval <approval_mode>] \
  [--skip-git-repo-check] \
  [--model <model>] \
  --output-schema <route_eval.schema.json> \
  --output-last-message <logs_dir/route_eval.raw.json> \
  <prompt>

16.3 Claude adapter

The Claude adapter MUST use claude -p and the documented print-mode flags for JSON output, schema-constrained output, tools, permissions, and optional session controls. Reflow v0.1 does not use --continue or --resume for correctness. If a deployment sets no_session_persistence, disable_auto_memory, or disable_slash_commands in the provider profile, the adapter MUST map those to the corresponding Claude CLI behaviors. Claude print mode supports --output-format json, and --json-schema returns schema-validated structured output in print mode.  ￼

Recommended worker invocation:

[CLAUDE_CODE_DISABLE_AUTO_MEMORY=1] \
claude -p "<prompt>" \
  --output-format json \
  [--no-session-persistence] \
  [--disable-slash-commands] \
  [--tools "<tool_csv>"] \
  [--allowedTools "<rule1>" "<rule2>"] \
  [--disallowedTools "<rule1>" "<rule2>"] \
  [--permission-mode <mode>] \
  [--max-turns <n>] \
  [--max-budget-usd <usd>] \
  [--model <model>]

Recommended router invocation:

[CLAUDE_CODE_DISABLE_AUTO_MEMORY=1] \
claude -p "<prompt>" \
  --output-format json \
  --json-schema '<schema_json_or_file_content>' \
  [--no-session-persistence] \
  [--disable-slash-commands] \
  [--tools "<tool_csv>"] \
  [--allowedTools "<rule1>" "<rule2>"] \
  [--disallowedTools "<rule1>" "<rule2>"] \
  [--permission-mode <mode>] \
  [--max-turns <n>] \
  [--max-budget-usd <usd>] \
  [--model <model>]

Parsing rules:
	•	worker mode MUST parse the documented JSON envelope for the ordinary text result
	•	schema-constrained router mode MUST parse the documented structured JSON output returned for --json-schema

16.4 Failure-classification matrix

Adapter classification MUST follow this table.

Condition	Failure type
executable missing	provider_unavailable
provider cannot start because auth / account / CLI initialization is unavailable	provider_unavailable
subprocess exceeds timeout	provider_timeout
subprocess launched and exits nonzero after startup	execution_failure
worker call exits cleanly but response envelope is missing or invalid	malformed_response
router call exits cleanly but JSON cannot be parsed	malformed_route_eval
router call exits cleanly but parsed object violates route-eval contract	invalid_route_eval

Additional required cases:
	•	Codex worker call exits cleanly but the --output-last-message file is missing or unreadable -> malformed_response
	•	Codex router call exits cleanly but the schema-constrained output file is missing or unreadable -> malformed_route_eval
	•	Claude worker call exits cleanly but the JSON envelope is unreadable or lacks the expected text payload -> malformed_response
	•	Claude router call exits cleanly but the structured JSON payload is unreadable or missing -> malformed_route_eval

Examples that MUST classify as execution_failure rather than provider_unavailable:
	•	Codex refuses the directory because repo checks fail and --skip-git-repo-check was not set
	•	Claude exits because --max-turns was reached
	•	tool-permission or sandbox denial after the process started
	•	subprocess returns nonzero after beginning task execution

16.5 Profile extra args

extra_args MAY add provider-specific flags, but MUST NOT override adapter-managed flags.

If a contradiction is detected, execution MUST fail as config_error before subprocess launch.

17. Route evaluation and transitions

17.1 Evaluator contract

After deterministic checks pass, the runtime MUST run exactly one evaluator pass per attempt.

The evaluator MUST:
	•	evaluate success.when
	•	evaluate failure.when
	•	evaluate all local routes in declaration order
	•	return one JSON object

17.2 route_eval.json contract

{
  "step_success": true,
  "step_failure": false,
  "matched_routes": ["screen_findings"],
  "selected_route": "screen_findings",
  "reason": "The latest review output is complete and ready to screen.",
  "confidence": 0.94
}

Validation rules:
	•	step_success and step_failure are booleans
	•	matched_routes is a list of strings
	•	if matched_routes is non-empty, selected_route equals the first entry
	•	if matched_routes is empty, selected_route is null
	•	reason is non-empty
	•	confidence is numeric and between 0 and 1

17.3 Transition resolution rules

routing.py MUST implement pure transition logic in addition to evaluator invocation.

Rules:
	1.	if step_success = true and step_failure = true -> raise route_eval_conflict
	2.	if step_success = false and step_failure = false -> raise ambiguous_evaluation
	3.	if step_failure = true -> raise semantic_failure
	4.	if routes exist and selected_route is null -> raise route_unresolved
	5.	if routes exist and selected_route is non-null -> return it
	6.	if no routes exist and step_success = true -> fall through to next step
	7.	if no routes exist and current step is last and step_success = true -> return stop

Semantic failure is always terminal failure.

escalate is a routing outcome, not a semantic-failure relabel.

17.4 Terminal resolution

Terminal actions resolve as:
	•	stop -> completed
	•	escalate -> escalated

Because semantic failure is terminal before transition resolution, stop and escalate apply only to non-failing semantic outcomes.

18. Retry policy

18.1 Retryable conditions

The retry budget applies to:
	•	execution_failure
	•	provider_timeout
	•	missing_outputs
	•	malformed_response
	•	malformed_route_eval
	•	invalid_route_eval
	•	route_eval_conflict
	•	ambiguous_evaluation
	•	route_unresolved

18.2 Non-retryable conditions

No retry for:
	•	provider_unavailable
	•	semantic_failure
	•	max_iter_exceeded
	•	config / workflow-load / filesystem / internal errors

18.3 Counter independence
	•	max_iter counts semantic visits
	•	max_retries counts retries inside one semantic visit

Retry budget resets to full at the start of each semantic visit.

Worst-case attempts per step:

max_iter × (1 + max_retries)

Retries do not increment semantic_visit_count.

18.4 Retry exhaustion

If a retryable cause persists and no retry remains, fail with:

retries_exhausted_<cause>

19. Runtime execution algorithm

This section is authoritative.

19.1 Main algorithm

For each selected step:
	1.	load current step and step state
	2.	check semantic_visit_count + 1 > max_iter
	3.	if exceeded, fail with max_iter_exceeded
	4.	increment semantic visit count and reset retry counters
	5.	enter attempt loop
	6.	create attempt folder
	7.	write input.json
	8.	append step instruction to ledger
	9.	execute step
	10.	if execution stage failed:

	•	classify failure
	•	mark route eval skipped
	•	apply retry policy

	11.	check required outputs
	12.	if outputs missing:

	•	classify missing_outputs
	•	mark route eval skipped
	•	apply retry policy

	13.	run exactly one semantic evaluation
	14.	validate evaluator output
	15.	if evaluator malfunction:

	•	mark route_eval_status = "failed"
	•	apply retry policy

	16.	resolve transition
	17.	persist the resolved transition target in attempt.json
	18.	if transition resolution raises retryable semantic ambiguity:

	•	apply retry policy

	19.	if semantic failure:

	•	fail immediately

	20.	if stop:

	•	complete run

	21.	if escalate:

	•	escalate run

	22.	otherwise transition to next step

19.2 Triple-write rule

Every runtime failure MUST be written to:
	•	messages.jsonl
	•	attempt.json
	•	state.json

Error content MUST include:
	•	failure_type
	•	failure_reason
	•	whether retry will occur
	•	remaining retries
	•	semantic visit usage

19.3 Reference pseudocode

FUNCTION run_workflow(workflow, config, workspace, parent_run_id=None, parent_step=None, parent_messages=None, export_dir=None):
    run_state = store.create_or_load_run(...)
    ledger = Ledger(run_state.run_path)

    IF new child run AND parent_messages exists AND ledger is empty:
        ledger.append(
            kind=INHERITED_CONTEXT,
            text=f"Inherited {len(parent_messages)} parent messages from run {parent_run_id}. "
                 f"The next {len(parent_messages)} messages are the inherited prefix.",
            linked_run_id=parent_run_id
        )
        FOR msg IN parent_messages:
            ledger.append_copied(msg)

    IF new run:
        ledger.append(WORKFLOW_SETUP, ...)
        IF workflow.notes:
            ledger.append(WORKFLOW_NOTES, workflow.notes)

    current_step = run_state.current_step OR workflow.entrypoint

    WHILE current_step IS NOT None:
        step = workflow.steps_by_name[current_step]
        state = run_state.step_states[step.name]

        IF state.semantic_visit_count + 1 > step.max_iter:
            fail_run(MAX_ITER_EXCEEDED, ...)
            RETURN failed

        state.semantic_visit_count += 1
        state.current_retry_number = 0
        state.current_retry_remaining = step.retry.max_retries
        run_state.current_step = step.name
        store.write_state(run_state)

        WHILE TRUE:
            state.attempt_count += 1
            attempt_num = state.attempt_count
            attempt_dir = store.create_attempt_dir(...)
            output_dir = attempt_dir / "output"

            attempt = store.write_attempt_start(...)
            input_record = build_input_record(...)
            store.write_input(input_record)

            instruction = context.build_step_instruction(...)
            ledger.append(STEP_INSTRUCTION, instruction, ...)

            result = executor.execute(...)

            IF NOT result.stage_ok:
                mark attempt execution failure
                mark route_eval skipped
                diagnose(...)
                IF should_retry(...):
                    consume_retry(...)
                    append retry_reason
                    store.write_state(run_state)
                    CONTINUE
                ELSE:
                    fail_run(retries_exhausted(...), ...)
                    RETURN failed

            missing = check_required_outputs(...)
            IF missing:
                mark attempt missing outputs
                mark route_eval skipped
                diagnose(...)
                IF should_retry(MISSING_OUTPUTS, ...):
                    consume_retry(...)
                    append retry_reason
                    store.write_state(run_state)
                    CONTINUE
                ELSE:
                    fail_run(retries_exhausted(MISSING_OUTPUTS), ...)
                    RETURN failed

            eval_result = routing.evaluate_route(...)

            IF eval_result is error:
                attempt.route_eval_status = "failed"
                diagnose(...)
                IF should_retry(...):
                    consume_retry(...)
                    append retry_reason
                    store.write_state(run_state)
                    CONTINUE
                ELSE:
                    fail_run(retries_exhausted(...), ...)
                    RETURN failed

            store.write_route_eval(eval_result)
            mark attempt route_eval completed
            ledger.append(ROUTE_EVAL_RESULT, summarize(eval_result), ...)

            TRY:
                next_action = routing.resolve_transition(...)
            CATCH retryable semantic errors AS e:
                diagnose(...)
                IF should_retry(e.failure_type, ...):
                    consume_retry(...)
                    append retry_reason
                    store.write_state(run_state)
                    CONTINUE
                ELSE:
                    fail_run(retries_exhausted(e.failure_type), ...)
                    RETURN failed
            CATCH SemanticFailure AS e:
                fail_run(SEMANTIC_FAILURE, e.reason)
                RETURN failed

            attempt.transition_status = "resolved"
            attempt.transition_target = next_action
            store.write_attempt(attempt)

            IF next_action == "stop":
                complete_run(...)
                attempt.transition_status = "applied"
                store.write_attempt(attempt)
                RETURN completed

            IF next_action == "escalate":
                escalate_run(...)
                attempt.transition_status = "applied"
                store.write_attempt(attempt)
                RETURN escalated

            ledger.append(TRANSITION, f"{step.name} -> {next_action}", ...)
            current_step = next_action
            store.write_state(run_state)
            attempt.transition_status = "applied"
            store.write_attempt(attempt)
            BREAK

20. Nested workflows

20.1 Child creation

A workflow step MUST create a new child run with its own:
	•	state.json
	•	messages.jsonl
	•	step folders
	•	attempt folders

20.2 Parent-to-child inheritance

Under full_run, the child MUST inherit the parent’s full visible ledger at launch.

Implementation rule:
	1.	snapshot parent ledger at child launch
	2.	append one inherited_context message to the child ledger that identifies the parent run and the number of inherited parent messages
	3.	copy the parent messages into the child ledger in original order as ordinary child-ledger messages
	4.	do not add per-message provenance fields to those copied messages

This means provenance is recorded once at the inheritance boundary instead of on every copied message.

The child then appends its own messages after the inherited prefix.

20.3 Export directory propagation

When a parent workflow step invokes a child workflow, the parent attempt output directory MUST be passed into the child run as export_dir_path.

The child workflow and its inner steps MAY write parent-required outputs there.

For child CLI steps, REFLOW_EXPORT_DIR MUST be set when export_dir_path is non-null.

For child provider steps, step instructions MUST include the export directory path when present.

20.4 Parent-child linkage

The child state.json MUST record:
	•	parent_run_id
	•	parent_step
	•	export_dir_path

The parent attempt.json MUST record:
	•	child_run_id
	•	child_workflow_name
	•	child_status
	•	child_state_path
	•	child_messages_path

The parent attempt output directory MUST contain child_run_ref.json.

20.5 Parent visibility of child context

The parent has access to the child’s full visible context by default after child completion.

This MUST be implemented by appending a child_context_link message to the parent ledger with artifact references to:
	•	child messages.jsonl
	•	child state.json
	•	parent child_run_ref.json
	•	any exported child outputs in the parent attempt output directory

The parent does not flatten child messages into its own ledger. The child ledger remains canonical for the child run. Parent visibility is provided through linked child artifacts and must be included in later parent-step artifact references under full_run.

20.6 Parent semantics for child terminal status

If the child reaches completed, failed, or escalated, the parent workflow step still counts as a completed execution stage.

The parent step’s own success.when, failure.when, and routes decide what that child outcome means.

20.7 Retry of workflow steps

If the parent retries a workflow step, it MUST launch a fresh child run.

Reflow v0.1 does not resume prior child runs as part of parent retry.

21. Resume semantics

21.1 Command

Reflow MUST support:

reflow resume <run_id> [--workspace <path>]

21.2 Authoritative model

Reflow-native resume is the only authoritative resume model in v0.1.

Resume correctness depends on:
	•	state.json
	•	messages.jsonl
	•	attempt artifacts
	•	config and workflow definitions

Reflow MUST NOT rely on provider-native resume for correctness.

21.3 Resume preconditions

A run is resumable only if:
	•	state.json.status == "running"
	•	the run is not already terminal
	•	current config_hash matches the stored config_hash
	•	current workflow_hash matches the stored workflow_hash

If config or workflow hash changed, reflow resume MUST fail rather than silently continuing with changed semantics.

21.4 Recovery behavior: partial or interrupted attempt

On resume:
	1.	load run state and ledger
	2.	locate current step and latest attempt folder
	3.	if the latest attempt directory exists but attempt.json is missing, unreadable, or structurally incomplete:
	•	synthesize a minimal attempt.json
	•	mark execution_status = "interrupted"
	•	mark route_eval_status = "skipped"
	•	mark route_eval_skip_reason = "attempt record incomplete after interruption"
	•	mark transition_status = "not_started"
	•	triple-write the interruption diagnosis
	4.	if the latest attempt has execution_status = "running" or ended_at = null:
	•	mark execution_status = "interrupted"
	•	mark failure_type = "execution_failure"
	•	mark failure_reason = "attempt interrupted before durable completion"
	•	if route_eval_status != "completed", mark:
	•	route_eval_status = "skipped"
	•	route_eval_skip_reason = "interrupted before semantic evaluation completed"
	•	triple-write the interruption diagnosis
	5.	if the recovered attempt is an interrupted execution-stage failure:
	•	if retry remains for the current semantic visit:
	•	consume one retry
	•	append retry reason
	•	continue with a fresh attempt in the same semantic visit
	•	otherwise fail with retries_exhausted_execution_failure

21.5 Recovery behavior: durable route eval, transition not yet applied

If the latest attempt has:
	•	route_eval_status = "completed"
	•	a durable route_eval.json
	•	transition_status != "applied"

then resume MUST not start a fresh provider call and MUST not consume retry budget unless transition resolution itself produces a retryable outcome.

Instead, Reflow MUST:
	1.	load the saved route_eval.json
	2.	if route_eval.json is missing, truncated, unreadable, or invalid despite route_eval_status = "completed":
	•	reclassify the attempt as route_eval_status = "failed"
	•	classify the cause as malformed_route_eval or invalid_route_eval as appropriate
	•	if retry remains for the current semantic visit:
	•	consume one retry
	•	append retry reason
	•	continue with a fresh attempt
	•	otherwise fail with the corresponding retries_exhausted_<cause>
	3.	recompute next_action = routing.resolve_transition(step, route_eval, workflow) deterministically
	4.	if attempt.transition_status == "resolved" and attempt.transition_target exists but differs from the recomputed next_action, fail with internal_error
	5.	handle routing.resolve_transition(...) results exactly as the live runtime would:
	•	if it raises route_eval_conflict, ambiguous_evaluation, or route_unresolved:
	•	if retry remains, consume one retry, append retry reason, and continue with a fresh attempt
	•	otherwise fail with the corresponding retries_exhausted_<cause>
	•	if it raises semantic_failure, fail immediately without retry
	6.	if no exception is raised:
	•	set or confirm attempt.transition_status = "resolved"
	•	set or confirm attempt.transition_target = <recomputed next_action>
	7.	apply the transition:
	•	stop -> complete run
	•	escalate -> escalate run
	•	step name -> update run state and continue from that next step
	8.	set attempt.transition_status = "applied"
	9.	write attempt.json, state.json, and any missing transition ledger message

This handles the crash window between durable route evaluation and durable transition application.

21.6 Recovery checkpoint rule

Resume occurs from the last durable orchestrator checkpoint, not from inside provider session state.

22. CLI interface and exit codes

22.1 Commands

Required commands:

reflow run <workflow_name> [--workspace <path>]
reflow resume <run_id> [--workspace <path>]
reflow status <run_id> [--workspace <path>]
reflow inspect <run_id> [--workspace <path>]
reflow list [--workspace <path>]

22.2 Output conventions
	•	human-readable status and diagnostics go to stderr
	•	future machine-readable output may use stdout
	•	reflow run and reflow resume SHOULD print the run directory path on start

22.3 Exit codes

Recommended mapping:

Code	Meaning
0	completed
20	provider unavailable
21	terminal step failure
22	routing / evaluator failure after retries exhausted
23	max iteration exceeded
24	escalated
25	config / workflow-load / filesystem / internal error

Mapping rules:
	•	provider_unavailable -> 20
	•	semantic_failure -> 21
	•	retries_exhausted_execution_failure and retries_exhausted_missing_outputs -> 21
	•	routing/evaluator failures and their exhaustion variants -> 22
	•	max_iter_exceeded -> 23
	•	escalated -> 24
	•	config/workflow/filesystem/internal -> 25

23. Diagnostics and signal handling

23.1 Raw logs

Each attempt MUST preserve raw logs under logs/.

Suggested filenames:
	•	provider.stdout.txt
	•	provider.stderr.txt
	•	provider.final_message.txt
	•	provider.events.jsonl
	•	router.raw.json
	•	cli.stdout.txt
	•	cli.stderr.txt

Only relevant files need to be written.

23.2 Signal handling

Reflow MUST trap SIGINT and SIGTERM.

On signal:
	1.	set an interrupt flag
	2.	terminate current subprocess if one exists
	3.	wait briefly
	4.	force-kill if needed
	5.	mark run as escalated
	6.	append escalation message
	7.	write final state.json
	8.	exit 24

Operator stop is escalation, not failure.

23.3 Failure message format

Every failure message MUST include:
	•	failure_type
	•	failure_reason
	•	whether retry will occur
	•	retries remaining
	•	semantic visit usage

Example:

Retry triggered: missing_outputs.
Semantic visit 2 of 5. Retry 1 of 2 (1 remaining).
Missing files: findings.json

24. Testing strategy

24.1 Unit tests

Unit tests MUST cover:
	•	config loading and defaults
	•	provider mapping validation
	•	workflow loading and skill loading
	•	workflow hash and config hash stability
	•	path naming and atomic writes
	•	ledger append ordering
	•	retry classification and budget management
	•	route-eval validation
	•	transition resolution
	•	nested linkage records
	•	inherited-context boundary behavior
	•	resume recovery logic
	•	exit-code mapping

24.2 Integration tests

Integration tests with mock providers MUST cover:
	•	happy-path workflow
	•	execution failure retry
	•	missing outputs retry
	•	malformed response retry
	•	malformed route eval retry
	•	invalid route eval retry
	•	route conflict retry
	•	ambiguous evaluation retry
	•	unresolved route retry
	•	semantic failure with no retry
	•	max_iter exceeded before execution
	•	provider unavailable fail-fast
	•	missing provider mapping as config error
	•	child workflow completed
	•	child workflow failed but parent semantically evaluates child result
	•	workflow-step required outputs created through child export path
	•	child context link visible in parent artifacts
	•	interrupted run resume with retry remaining
	•	interrupted run resume with no retry remaining
	•	resume from durable route-eval / unapplied-transition checkpoint
	•	corrupt or missing route_eval.json during resume recovery
	•	retry budget reset on semantic revisit
	•	operator stop escalation

24.3 Provider-contract regression tests

The test suite SHOULD include golden parse fixtures for:
	•	Codex worker final-message capture
	•	Codex router schema output
	•	Claude worker JSON envelope parsing
	•	Claude router structured-output parsing

These tests SHOULD use captured fixtures, not live provider calls.

25. Implementation sequence

Phase 1 — Foundation
	1.	contracts.py
	2.	errors.py
	3.	models.py
	4.	store.py
	5.	ledger.py

Phase 2 — Loading
	6.	config.py
	7.	loader.py

Phase 3 — Policy
	8.	retry.py
	9.	routing.py

Phase 4 — Context and providers
	10.	context.py
	11.	providers/base.py
	12.	providers/codex.py
	13.	providers/claude.py
	14.	providers/resolver.py

Phase 5 — Executors
	15.	executors/base.py
	16.	executors/skill.py
	17.	executors/cli.py
	18.	nested.py
	19.	executors/workflow.py

Phase 6 — Kernel
	20.	runtime.py

Phase 7 — CLI
	21.	cli/exit_codes.py
	22.	cli/main.py
	23.	__main__.py

Phase 8 — End-to-end
	24.	author first workflow
	25.	author first skills
	26.	configure providers
	27.	run end-to-end validation

26. Consolidated clarifications

This ADS deliberately resolves the main ambiguities from the earlier drafts.
	1.	Required outputs are existence-only in v0.1.
General schema validation is out of scope; route-eval validation remains required.
	2.	Workflow steps can have required outputs.
They are checked like any other step, but their materialization path is the parent attempt’s output directory via child export behavior.
	3.	route_eval.json exists only when semantic evaluation runs.
Skips and evaluator failures are represented explicitly in attempt.json.
	4.	Parent access to child context is explicit and durable.
The parent gets child state and child ledger visibility via linked artifacts, and later parent steps must receive those references.
	5.	Child inheritance provenance is recorded once, not on every copied message.
The child ledger begins with one inherited-context boundary record, followed by the copied parent-message prefix.
	6.	Reflow-native resume is required.
Provider-native resume exists but is not authoritative for correctness.
	7.	Provider behavior is not forcibly constrained by default.
Reflow does not override provider web-search configuration by default and does not require provider persistence or memory controls to be disabled. Provider environment and provider configuration may influence execution; Reflow correctness still depends on its own persisted artifacts and state. Codex documents web-search and resume as provider-side features, and Claude documents separate provider-side controls for persistence, memory, tools, and permissions.  ￼
	8.	Codex --ephemeral is intentionally not required in v0.1.
Codex documents that flag as suppressing persisted local session rollout files, but Reflow v0.1 relies on Reflow-native artifacts for correctness instead.  ￼

27. Final architectural position

Reflow v0.1 is a deterministic, file-first, terminal-native runtime for reusable AI workflows.

Its kernel owns:
	•	run state
	•	semantic visits
	•	retry counters
	•	transitions
	•	terminal states
	•	resume behavior
	•	persistence

Its executors own:
	•	how a step is executed

Its provider adapters own:
	•	how Codex or Claude are invoked

Its workflows own:
	•	domain logic

Its skills own:
	•	reusable role behavior

Its ledgers and artifacts own:
	•	the correctness-critical visible history

That separation is the core contract of the system.
