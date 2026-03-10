## Appendix A — Provider integration, file contracts, and runtime interfaces

This appendix is **standalone and normative** for implementation. An agent implementing the orchestrator can use this appendix, plus the PRD, as the source of truth for all provider integrations, runtime contracts, and file formats.

This appendix intentionally distinguishes between:

* **documented provider capabilities**
* **project-specific implementation rules**

The provider capabilities below are based on the current official Codex CLI and Claude Code documentation. Codex supports scripted `codex exec`, JSONL output, JSON-Schema-constrained final output, resume, layered `AGENTS.md`, and repo skills under `.agents/skills`. Claude Code supports `claude -p`, JSON output with `--json-schema`, `--allowedTools`, `--append-system-prompt` / `--append-system-prompt-file`, `--continue`, `--resume`, layered `CLAUDE.md`, and repo skills under `.claude/skills`. ([OpenAI Developers][1])

---

## A1. Normative implementation rules

The implementation MUST follow these rules:

1. `.ai/skills` is the only canonical skill source in v0.1.
2. The orchestrator MUST NOT depend on `.agents/skills` or `.claude/skills`.
3. The orchestrator MUST NOT manually append `AGENTS.md` or `CLAUDE.md`; those are provider-native instruction layers.
4. `full_run` means the full canonical orchestrator message history, not full file contents dumped into prompts.
5. Artifacts MUST be referenced by full relative path in message history; they MUST NOT be inlined by default.
6. All step-local routes, plus `success.when` and `failure.when`, MUST be evaluated in one route-evaluator pass that returns JSON.
7. Provider unavailability MUST fail fast.
8. Human approvals are out of scope for v0.1.
9. Nested workflows are in scope for v0.1.
10. The canonical source of replayable state is the orchestrator’s own message ledger and artifacts, not provider-native session state.

---

## A2. Codex CLI integration surface

### A2.1 Core command

Use `codex exec` for scripted, non-interactive runs. Codex documents `codex exec` as the entrypoint for scripts and CI. ([OpenAI Developers][1])

### A2.2 Working directory

Use `--cd` / `-C` to set the workspace root before processing the request. ([OpenAI Developers][2])

### A2.3 Permission model

For automation, Codex documents:

* default `codex exec` behavior is **read-only sandbox**
* `--full-auto` allows edits
* `--sandbox danger-full-access` allows broader access
* `--ask-for-approval` accepts `untrusted | on-request | never` at the CLI level. ([OpenAI Developers][1])

### A2.4 Machine-readable output

Use `--json` when you need a JSONL event stream. Codex documents that `--json` makes `stdout` a JSON Lines stream containing events such as:

* `thread.started`
* `turn.started`
* `turn.completed`
* `turn.failed`
* `item.*`
* `error`

The JSONL stream can include agent messages, reasoning, command executions, file changes, MCP tool calls, web searches, and plan updates. ([OpenAI Developers][1])

### A2.5 Structured final output

Use `--output-schema <schema.json>` together with `-o <output.json>` when you need a schema-constrained final response written to disk. Codex documents this as the supported way to get stable structured outputs for downstream automation. ([OpenAI Developers][1])

### A2.6 Resume support

Codex documents `codex exec resume --last` and `codex exec resume <SESSION_ID>` for continuing prior non-interactive runs. ([OpenAI Developers][1])

### A2.7 Repository assumptions

Codex documents that commands normally run inside a Git repository and provides `--skip-git-repo-check` to override this. The orchestrator SHOULD assume a Git repository is present unless the operator intentionally overrides that behavior. ([OpenAI Developers][1])

### A2.8 Project instructions

Codex automatically discovers `AGENTS.md` with documented precedence and layered lookup from global and project directories. The orchestrator MUST assume Codex is already loading that provider-native instruction chain. It must not duplicate it. ([OpenAI Developers][3])

### A2.9 Native skills

Codex native skills live under `.agents/skills`, use `SKILL.md`, and support optional extra files with progressive disclosure. v0.1 MUST NOT depend on that mechanism for correctness, but an implementation MAY optionally generate mirrors later. ([OpenAI Developers][4])

---

## A3. Claude Code integration surface

### A3.1 Core command

Use `claude -p` for scripted, non-interactive execution. Claude documents this as the headless / programmatic interface. ([Claude API Docs][5])

### A3.2 Structured output

Use:

* `--output-format json`
* `--json-schema '<schema>'`

Claude documents that the structured result is returned in the `structured_output` field, with additional metadata such as `session_id` and usage in the JSON response. ([Claude API Docs][5])

### A3.3 Allowed tools

Use `--allowedTools` to pre-authorize tools such as `Bash`, `Read`, and `Edit`. Claude documents `--allowedTools` explicitly for headless mode. ([Claude API Docs][5])

### A3.4 System prompt extension

Use:

* `--append-system-prompt`
* `--append-system-prompt-file`

Claude documents these as the preferred way to add custom instructions while preserving Claude Code’s built-in behavior. ([Claude][6])

### A3.5 Continue / resume

Claude documents:

* `--continue` to continue the most recent conversation
* `--resume <SESSION_ID>` to continue a specific conversation. ([Claude API Docs][5])

### A3.6 Project instructions

Claude automatically loads `CLAUDE.md` and treats it as context at the start of the conversation. The orchestrator MUST assume this provider-native behavior and MUST NOT duplicate that file into the prompt manually. ([Claude API Docs][7])

### A3.7 Permissions

Claude uses a tiered permission system:

* read-only tools do not require approval
* Bash commands require approval unless allowed
* file modifications require approval unless allowed
* project settings can define permission rules
* permission modes include `default`, `acceptEdits`, `plan`, `dontAsk`, `bypassPermissions`. ([Claude][8])

### A3.8 Native skills

Claude native skills live under `.claude/skills/<skill>/SKILL.md`, support optional supporting files, and are primarily documented for native Claude Code use. User-invoked skills such as `/skill-name` are an interactive-mode feature. v0.1 MUST NOT depend on native Claude skill discovery for correctness. ([Claude API Docs][9])

### A3.9 Hooks

Claude hooks are deterministic lifecycle shell commands. They are useful later for enforcement, but they are OPTIONAL in v0.1 and MUST NOT be required for orchestrator correctness. ([Claude API Docs][10])

---

## A4. Provider selection contract

Create `.ai/config/providers.yaml`.

Minimum contract:

```yaml
defaults:
  router: claude

skills:
  review-codebase: codex
  screen-review-findings: claude
  implement-screened-findings: codex

workflows:
  nested_workflow_name: claude

profiles:
  codex:
    command: codex
    model: <string-or-null>
    cd_flag: --cd
    sandbox: read-only
    approval: never

  claude:
    command: claude
    allowed_tools_skill: "Bash,Read,Edit"
    allowed_tools_router: "Read"
    output_format: json
```

Rules:

1. Workflow files MUST NOT hardcode provider choice.
2. The orchestrator MUST resolve providers only through `providers.yaml`.
3. If the configured provider executable is missing, not authenticated, or unavailable, the run MUST fail fast.
4. There is no fallback provider in v0.1.

---

## A5. Canonical run state and statuses

Use only these run statuses in v0.1:

* `running`
* `completed`
* `failed`
* `escalated`

Do not implement a separate `stopped` or `error` state.

Use `failure_type` and `failure_reason` fields instead of extra top-level statuses.

Examples of `failure_type`:

* `provider_unavailable`
* `missing_required_outputs`
* `route_unresolved`
* `route_eval_conflict`
* `invalid_provider_response`
* `filesystem_error`
* `semantic_failure`

---

## A6. Canonical message ledger (`messages.jsonl`)

This is the most important runtime contract.

### A6.1 Purpose

`messages.jsonl` is the authoritative, replayable approximation of provider session history.

It MUST be append-only.
It MUST preserve message order.
It MUST be the canonical source for `full_run` context.

### A6.2 Message record shape

Each line MUST be one JSON object with at least:

```json
{
  "seq": 1,
  "ts": "2026-03-10T12:00:00Z",
  "role": "orchestrator",
  "kind": "step_instruction",
  "workflow": "review_analyze_do",
  "step": "review_codebase",
  "attempt": 1,
  "provider": "codex",
  "text": "Human-readable message content",
  "artifact_paths": [
    ".ai/runs/run_.../010_review_codebase/attempt_001/output/findings.json"
  ]
}
```

### A6.3 Allowed `role` values

Use:

* `system`
* `orchestrator`
* `provider`
* `cli`

Do not use hidden or private roles.

### A6.4 Allowed `kind` values

Minimum v0.1 kinds:

* `workflow_setup`
* `workflow_notes`
* `step_instruction`
* `retry_reason`
* `provider_response`
* `cli_result`
* `route_evaluation`
* `transition`
* `failure`

### A6.5 Artifact references

`artifact_paths` MUST contain **relative file paths** from the repository root.
Do not inline full artifact contents into the ledger by default.

### A6.6 What “full_run” means

`full_run` means:

* the full canonical `messages.jsonl` history in order
* plus access to referenced artifacts by path in the workspace

It does **not** mean copying every artifact body into the prompt.

### A6.7 Provider replay rule

The orchestrator MUST be able to reconstruct provider input from `messages.jsonl` and on-disk artifacts alone.

Provider-native resume (`codex exec resume`, `claude --continue` / `--resume`) MAY be used as an optimization, but correctness MUST NOT depend on them. Codex and Claude both document resume/continue features, but the orchestrator’s own ledger remains the source of truth. ([OpenAI Developers][1])

---

## A7. Context assembly algorithm

For every step execution, the orchestrator MUST construct provider context as follows.

### A7.1 Common rules

1. Do not manually append `AGENTS.md` or `CLAUDE.md`.
2. Do append workflow-local `notes.md` as an orchestrator message if it exists.
3. If `action.kind = skill`, append the selected `.ai/skills/<skill>/SKILL.md` as an orchestrator message.
4. Append the step contract summary as an orchestrator message:

   * step name
   * description
   * required outputs
   * success condition
   * failure condition
   * local routes
5. Append only path references to artifacts in `artifact_paths`.
6. On retries, append a `retry_reason` message before invoking the provider again.

### A7.2 Do not do these things

* do not append the full skill library
* do not append all artifact bodies
* do not depend on hidden provider memory
* do not synthesize stale/superseded labels in the orchestrator unless a workflow explicitly writes them

### A7.3 CLI steps

CLI steps do not receive prompt context. Instead, the orchestrator MUST materialize context through files and environment variables.

Required environment variables for CLI steps:

* `AI_WORKFLOW_NAME`
* `AI_RUN_ID`
* `AI_STEP_NAME`
* `AI_ATTEMPT`
* `AI_WORKSPACE_ROOT`
* `AI_RUN_DIR`
* `AI_STEP_DIR`
* `AI_ATTEMPT_DIR`
* `AI_OUTPUT_DIR`
* `AI_MANIFEST_PATH`
* `AI_SUMMARY_PATH`
* `AI_MESSAGES_PATH`

The CLI script is expected to read these paths directly.

---

## A8. Workflow contract (`workflow.yaml`)

Minimum workflow file shape:

```yaml
name: review_analyze_do
description: Review, screen, implement, verify in a loop.
entrypoint: review_codebase

steps:
  - name: review_codebase
    description: Review the codebase and produce findings.
    action:
      kind: skill
      ref: review-codebase
      with: {}
    max_iter: 8
    context_mode: full_run
    outputs:
      required:
        - review_output.md
        - findings.json
    success:
      when: The required outputs exist and the findings are concrete and non-empty.
    failure:
      when: The review could not inspect the code or the findings are contradictory or unusable.
    routes:
      - when: The latest review output is ready to be screened.
        then: screen_findings
```

Rules:

1. `context_mode` is optional in authored files.
2. If omitted, the orchestrator MUST treat it as `full_run`.
3. In v0.1, any value other than `full_run` is invalid configuration.
4. Routes are step-local and ordered.

---

## A9. Attempt artifact contracts

Each attempt folder MUST contain:

* `input.json`
* `attempt.json`
* `route_eval.json`
* `logs/`
* `output/`

### A9.1 `input.json`

Minimum fields:

```json
{
  "workflow": "review_analyze_do",
  "step": "review_codebase",
  "attempt": 1,
  "provider": "codex",
  "action": {
    "kind": "skill",
    "ref": "review-codebase",
    "with": {}
  },
  "context_mode": "full_run",
  "required_outputs": ["review_output.md", "findings.json"],
  "success_when": "...",
  "failure_when": "...",
  "routes": [
    {"when": "...", "then": "screen_findings"}
  ],
  "artifact_paths": []
}
```

### A9.2 `attempt.json`

Minimum fields:

```json
{
  "workflow": "review_analyze_do",
  "step": "review_codebase",
  "attempt": 1,
  "provider": "codex",
  "started_at": "2026-03-10T12:00:00Z",
  "ended_at": "2026-03-10T12:00:20Z",
  "status": "completed",
  "failure_type": null,
  "failure_reason": null,
  "output_paths": [
    ".ai/runs/run_x/010_review_codebase/attempt_001/output/review_output.md",
    ".ai/runs/run_x/010_review_codebase/attempt_001/output/findings.json"
  ]
}
```

### A9.3 `route_eval.json`

Minimum fields:

```json
{
  "step_success": true,
  "step_failure": false,
  "matched_routes": ["screen_findings"],
  "selected_route": "screen_findings",
  "reason": "The latest review output is ready to be screened.",
  "confidence": 0.94
}
```

Rules:

* `selected_route` MUST equal the first element of `matched_routes`
* `selected_route` MAY be `null` only if `matched_routes` is empty

---

## A10. Route-evaluator contract

The route evaluator MUST evaluate:

* the step’s success condition
* the step’s failure condition
* all step-local routes in one pass

It MUST return JSON conforming to `route_eval.schema.json`.

### A10.1 Codex route-evaluator invocation

Use Codex structured output:

```bash
codex exec --cd "$WORKSPACE" \
  --sandbox read-only \
  --ask-for-approval never \
  --output-schema .ai/config/route_eval.schema.json \
  -o "$ATTEMPT_DIR/route_eval.json" \
  "<route-evaluator prompt>"
```

Codex documents `--output-schema`, `-o`, `--cd`, `--sandbox`, and read-only default behavior for automation. ([OpenAI Developers][1])

### A10.2 Claude route-evaluator invocation

Use Claude structured output:

```bash
claude -p "<route-evaluator prompt>" \
  --output-format json \
  --json-schema "$(cat .ai/config/route_eval.schema.json)" \
  --allowedTools "Read"
```

Claude documents `-p`, `--output-format json`, `--json-schema`, and `--allowedTools`. The structured result is returned in `structured_output`. ([Claude API Docs][5])

### A10.3 Route evaluator prompt requirements

The prompt MUST instruct the evaluator to:

* use only persisted visible evidence
* treat missing evidence as false
* evaluate success, failure, and all routes in one pass
* preserve route order
* return JSON only

---

## A11. Retry and terminal state algorithm

For each attempt:

1. Execute the step.
2. Check deterministic failures:

   * provider unavailable
   * step executable missing
   * malformed provider response that cannot be parsed
   * missing required outputs
   * `max_iter` exceeded
3. If deterministic failure is retryable and `max_iter` remains, append a `retry_reason` message and retry.
4. Otherwise run one route-evaluator pass.
5. If `step_success = true` and `step_failure = true`, treat as `route_eval_conflict`; retry if allowed, else fail.
6. If routes exist and none match, treat as `route_unresolved`; retry if allowed, else fail.
7. If `selected_route = escalate`, set run status to `escalated`.
8. If `selected_route = stop`, set run status:

   * `completed` if `step_failure = false`
   * `failed` if `step_failure = true`
9. If no routes exist:

   * on semantic success, continue to next step in workflow order
   * on semantic failure, fail
   * if neither success nor failure is true, retry if allowed, else fail

### A11.1 Non-retryable failures

Do not retry:

* provider unavailable
* semantic failure (`failure.when = true`)
* explicit `selected_route = escalate`
* operator-aborted runs, if you implement operator abort

---

## A12. Nested workflow algorithm

When `action.kind = workflow`:

1. Create child run directory.
2. Write child `manifest.json`, `summary.json`, and `messages.jsonl`.
3. Copy/inherit parent message ledger semantics:

   * child starts with inherited visible context according to `context_mode`
   * default is `full_run`
4. Execute child workflow until terminal state.
5. Parent attempt records:

   * `child_run_id`
   * `child_workflow_name`
   * `child_status`
   * `child_summary_path`
   * `child_manifest_path`
6. Parent route evaluation then uses:

   * the child’s terminal state
   * the child’s visible message history
   * the child’s artifacts by path

---

## A13. Skill loading algorithm

Given a step with:

```yaml
action:
  kind: skill
  ref: review-codebase
```

the orchestrator MUST load:

```text
.ai/skills/review-codebase/SKILL.md
```

and append its content as an orchestrator message to the message ledger before the provider call.

Do not rely on `.agents/skills` or `.claude/skills` in v0.1.

Codex and Claude both document native skill directories, but this project chooses not to depend on them. ([OpenAI Developers][4])

---

## A14. Codex worker command template

Use this shape for skill or workflow steps that run through Codex:

```bash
codex exec --cd "$WORKSPACE" \
  --ask-for-approval never \
  --json \
  "<assembled step prompt>"
```

If the step needs write access, add either:

* `--full-auto`
* or a broader sandbox setting

Codex documents read-only default behavior, JSONL event output via `--json`, and the use of explicit sandbox/approval settings for automation. ([OpenAI Developers][1])

Store:

* raw JSONL stream under `logs/provider.jsonl`
* final text result by extracting the last agent message from the JSONL if needed

If you want native session continuation for same-provider optimization, use:

* `codex exec resume --last`
* or `codex exec resume <SESSION_ID>`

but correctness MUST remain reconstructible from `messages.jsonl`. ([OpenAI Developers][1])

---

## A15. Claude worker command template

Use this shape for skill or workflow steps that run through Claude:

```bash
claude -p "<task prompt>" \
  --append-system-prompt-file "$APPEND_FILE" \
  --output-format json \
  --allowedTools "Bash,Read,Edit"
```

If you need structured output from the step itself, add:

```bash
--json-schema "$(cat path/to/schema.json)"
```

Claude documents `--append-system-prompt-file` as the safest way to add custom instructions while preserving default behavior, and documents structured JSON output and allowed tools in headless mode. ([Claude][6])

Store:

* raw JSON response under `logs/provider.json`
* the human-visible provider message under a `provider_response` entry in `messages.jsonl`
* if structured output was requested, extract `structured_output`

If you want native session continuation for same-provider optimization, use:

* `--continue`
* or `--resume <SESSION_ID>`

but correctness MUST remain reconstructible from `messages.jsonl`. ([Claude API Docs][5])

---

## A16. Message composition for one step

The orchestrator MUST append the following messages, in order, for a normal skill step:

1. `workflow_notes`
   if `notes.md` exists and has not already been appended for the workflow context

2. `step_instruction`
   includes:

   * step name
   * description
   * required outputs
   * success condition
   * failure condition
   * routes

3. `skill_text`
   the exact `SKILL.md` content of the current skill

4. `retry_reason`
   only if retrying

5. `provider_response`
   concise provider message plus artifact paths

6. `route_evaluation`
   selected JSON result plus artifact path

7. `transition`
   next step, stop, fail, or escalate

---

## A17. What an implementation MUST NOT do

Do not:

* duplicate `AGENTS.md` or `CLAUDE.md` manually
* expose the full `.ai/skills` library to every step
* inline full artifact files into every prompt by default
* depend on hidden provider-private memory
* make provider-native skill mirrors required for correctness
* invent extra runtime statuses beyond:

  * `running`
  * `completed`
  * `failed`
  * `escalated`

---

## A18. Minimal implementation checklist

An implementation is sufficient if it can do all of the following:

1. Read `workflow.yaml`
2. Resolve providers from `providers.yaml`
3. Create run folders and attempt folders
4. Maintain `messages.jsonl`
5. Execute `skill`, `workflow`, and `cli` steps
6. Materialize CLI context through paths and env vars
7. Enforce required outputs
8. Run one route-evaluator pass per attempt
9. Retry kernel-level failures up to `max_iter`
10. Preserve every attempt immutably
11. Fail fast on provider unavailability
12. Execute child workflows
13. Reach `completed`, `failed`, or `escalated` deterministically

---

This appendix is the implementation reference. If you want, I can now turn it into a frozen `APPENDIX.md` artifact alongside the PRD.

[1]: https://developers.openai.com/codex/noninteractive/ "Non-interactive mode"
[2]: https://developers.openai.com/codex/cli/reference/ "Command line options"
[3]: https://developers.openai.com/codex/guides/agents-md/ "Custom instructions with AGENTS.md"
[4]: https://developers.openai.com/codex/skills/ "Agent Skills"
[5]: https://docs.anthropic.com/en/docs/claude-code/headless "Run Claude Code programmatically - Claude Code Docs"
[6]: https://code.claude.com/docs/en/cli-reference "CLI reference - Claude Code Docs"
[7]: https://docs.anthropic.com/en/docs/claude-code/settings "Claude Code settings - Claude Code Docs"
[8]: https://code.claude.com/docs/en/permissions "Configure permissions - Claude Code Docs"
[9]: https://docs.anthropic.com/en/docs/claude-code/slash-commands "Extend Claude with skills - Claude Code Docs"
[10]: https://docs.anthropic.com/en/docs/claude-code/hooks-guide "Automate workflows with hooks - Claude Code Docs"
