# reflow

Deterministic workflow runtime for AI agents.

## The loop you're already running

You open a terminal. You prompt an agent to review your code. You wait. You read the output, then prompt another agent to screen those findings. You wait. You take the worthwhile suggestions and prompt an agent to implement them. You wait. You run verification. You read the results and start over.

Review. Wait. Screen. Wait. Implement. Wait. Verify. Wait. Repeat.

This works. The results are genuinely good. But you're the glue — copying context between prompts, deciding what to feed forward, remembering what was already addressed, babysitting each handoff. The agents do the thinking. You do the scheduling, the plumbing, and the waiting.

reflow runs the loop for you.

```
reflow run review_analyze_do
```

You define the loop once as a workflow file in your repo. reflow executes each step, evaluates whether to continue or stop, routes to the next step, and preserves everything. You come back to a completed run with a full history of every attempt, every finding, every decision.

## What changes

**Before:** You sit in a terminal for 45 minutes, manually prompting agents in sequence, copying context, re-reading outputs, deciding what's next.

**After:** You type one command. The workflow runs the same loop you would have run — review, screen, implement, verify, repeat — but without you in the middle. Each step delegates to the agent you choose. Each handoff is automatic. Each decision point is evaluated against conditions you wrote. When it's done, every attempt and every artifact is sitting in a run folder you can inspect.

## How it looks

A workflow is a YAML file in your repo:

```yaml
name: review_analyze_do
entrypoint: review_codebase

steps:
  - name: review_codebase
    action:
      kind: skill
      ref: review-codebase
    max_iter: 5
    retry:
      max_retries: 2
    outputs:
      required:
        - findings.json
    success:
      when: "Review completed with actionable findings or confirmed no issues remain."
    failure:
      when: "Codebase is inaccessible or review cannot be performed."
    routes:
      - when: "Worthwhile findings exist that have not been addressed."
        then: screen_findings
      - when: "No worthwhile findings remain."
        then: stop

  - name: screen_findings
    action:
      kind: skill
      ref: screen-findings
    # ...
    routes:
      - when: "Screened findings contain items worth implementing."
        then: implement_changes
      - when: "No findings survive screening."
        then: stop

  - name: implement_changes
    action:
      kind: skill
      ref: implement-changes
    # ...
    routes:
      - when: "Changes applied, verification needed."
        then: verify

  - name: verify
    action:
      kind: cli
      ref: verify.sh
    # ...
    routes:
      - when: "Verification passed."
        then: review_codebase
      - when: "Verification failed with fixable issues."
        then: implement_changes
```

You choose which agent runs which step:

```yaml
# providers.yaml
skills:
  review-codebase: codex
  screen-findings: claude
  implement-changes: codex
```

The workflow doesn't know about providers. Swap one line, same loop, different agent.

## What reflow actually does

reflow is not an agent. It doesn't plan, reason, or improvise. It's a small deterministic runtime that:

1. Loads your workflow definition
2. Executes each step by delegating to Codex CLI, Claude Code, a script, or a nested workflow
3. Evaluates semantic conditions to determine success, failure, and routing
4. Follows the route to the next step, loops back, or stops
5. Persists every attempt, every evaluation, every message

The agents do the work. reflow does the scheduling, the handoffs, and the bookkeeping that you were doing manually.

## Why not just script it

You could chain prompts in a bash script. People do. It falls apart when:

- **The API drops a connection** and your script burns a loop iteration on a timeout instead of retrying. reflow separates retry (infrastructure noise) from iteration (real work). A step with `max_iter: 3` and `max_retries: 2` gets 3 real convergence passes, each of which can survive 2 transient failures.

- **You need to debug what happened.** reflow writes an append-only message ledger (`messages.jsonl`), a route evaluation record for every step (`route_eval.json`), and immutable attempt folders. You can reconstruct exactly what every agent saw, what it produced, and why the orchestrator routed where it did.

- **Context gets lost between steps.** reflow maintains a canonical message history that accumulates across the entire run. Each step sees the full conversation — prior findings, prior screens, prior implementations — referenced by file path, not pasted inline.

- **You want to reuse the loop.** The workflow is a YAML file. It works on any repo. Different team members get the same process. The loop doesn't drift because someone forgot a step or changed the prompt.

## Design decisions

**File-first.** Workflows are YAML. Runs are folders. Context is JSONL. You can `cat`, `grep`, `jq`, and `git diff` your way through everything.

**Deterministic shell, agentic interior.** reflow owns state transitions. Agents own the actual work. The orchestrator doesn't know what "review" or "implement" means — your skill files define that.

**Visible context only.** No hidden chain-of-thought. No provider-private memory. If it's not in the run folder, it doesn't exist to the runtime.

**Provider-agnostic workflows.** Provider choice lives in `providers.yaml`, not in workflows. Swap agents without touching the loop.

## Run output

Every run produces a complete, inspectable record:

```
.ai/runs/run_20250310T1423_review_analyze_do/
  state.json              # current run state
  messages.jsonl          # full context ledger
  010_review_codebase/
    attempt_001/
      input.json          # what was sent
      attempt.json        # execution metadata
      route_eval.json     # success/failure/routing decision
      output/             # domain artifacts (findings.json, etc.)
      logs/               # raw provider output
    attempt_002/
      ...
  020_screen_findings/
    attempt_001/
      ...
```

Nothing is overwritten. Earlier attempts stay in the history. The most recent attempt is the current view.

## Current scope

reflow v0.1 works with:

- **Codex CLI** — non-interactive `codex exec` with structured output
- **Claude Code CLI** — programmatic `claude -p` with JSON output
- **Local scripts** — any CLI command
- **Nested workflows** — child runs with full parent linkage

Terminal-native. No daemon, no web UI, no service mode.

## Status

v0.1 — proving the architecture with a real review → screen → implement → verify workflow.

## License

[TBD]
