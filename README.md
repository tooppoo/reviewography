# reviewography

`reviewography` is a local-first review memory tool for AI-assisted software development.

It runs an AI review against the current change, records review findings as structured AI-readable data, and folds useful findings into reusable review criteria for later reviews.

The name combines `review` and `-graphy`: writing, describing, and organizing review knowledge. It is not a visualization tool.

The short CLI name is:

```bash
revg
```

## Purpose

AI-assisted development often produces useful review findings, but those findings are easy to lose.

Common problems include:

- review findings are copied manually between tools
- the same class of issue is rediscovered repeatedly
- accepted, rejected, and out-of-scope review comments are not preserved as reusable knowledge
- human reviewers must inspect large diffs and large test suites without enough structured context
- AI agents lack a stable local memory of past review decisions

`reviewography` addresses these problems by treating review findings as persistent project knowledge.

## Core model

reviewography is organized around a review session.

```text
review session
  -> review run
  -> structured findings
  -> finding resolution
  -> criteria proposal
  -> criteria update
```

A session starts from a required base revision.

```bash
revg review start --base main
```

During the session, implementation and review may repeat outside reviewography:

```text
change
  -> revg review run
  -> inspect findings
  -> change again
  -> revg review run
```

reviewography does not implement the fix loop. It records and manages the review side of that loop.

## Goals

reviewography aims to:

- run one AI review pass for the current change
- validate AI review output against a structured schema
- retry AI generation when the output is schema-invalid
- record findings as canonical AI-readable data
- distinguish concrete findings from reusable review criteria
- preserve resolved, rejected, deferred, duplicate, and negative decisions
- reuse prior criteria in future reviews
- support worktree-local review sessions
- keep human-readable output as a temporary generated view
- operate primarily on local files

## Non-goals

reviewography does not aim to:

- implement code changes
- run an implementation-review-fix loop until findings disappear
- perform multi-agent review orchestration
- merge or deduplicate findings from multiple agents
- replace human review
- manage GitHub Issues or Pull Requests
- create ADRs
- perform full repository architecture audits
- analyze test coverage by itself
- import GitHub PR comments in v0
- become a single integrated development automation platform

Implementation and review loops should be handled by skills, scripts, or external workflow tools.

Repository-wide structure audits should be handled by a separate tool.

Test evidence extraction should be handled by a separate tool such as `testography`.

## Positioning

reviewography is an independent tool.

It may be used together with tools such as:

- `kogoto` for issue refinement and development workflow support
- `testography` for test evidence mapping
- repository audit tools for documentation and implementation consistency checks
- AI coding agents for implementation
- AI review agents for review

But reviewography does not depend on those tools.

The intended dependency direction is:

```text
other workflow tools may call reviewography
reviewography does not depend on those workflow tools
```

## Primary workflow

The normal workflow is:

```bash
revg init
revg review start --base main
revg review run
revg review show --unresolved
revg review resolve <review-id> --as fixed
revg review run --scope full
revg review fold
revg criteria apply --local <proposal-id>
revg review end
```

The key operation is `revg review run`.

It performs one review pass:

```text
load active session
  -> compute review scope
  -> load relevant criteria
  -> build review context
  -> invoke one configured AI review agent
  -> validate structured output
  -> retry if schema-invalid
  -> save valid findings
  -> update review cursor
  -> generate temporary view
```

## Review scope

reviewography supports two review scopes.

```text
incremental
  Focus on last_reviewed_head..HEAD.
  Still include base..HEAD context, unresolved findings, and relevant criteria.

full
  Review base..HEAD as a whole.
  Use this before ending a session, before human review, or before PR creation.
```

`incremental` is the default.

There is no special `final` scope. A full review can still produce findings; if it does, the session is not finished.

## Review cursor

A session separates the stable base revision from the review cursor.

```json
{
  "base": "main",
  "current_head": "HEAD",
  "last_reviewed_head": "abc123"
}
```

`base` defines the whole change under review.

`last_reviewed_head` lets later review runs focus on what changed since the previous run.

## Agent execution

`revg review run` includes AI reviewer execution.

This is intentional. reviewography must control:

- review context generation
- output schema instruction
- JSON extraction
- schema validation
- retry on schema-invalid output
- saving only valid canonical records

Agent support is implemented through an adapter model.

A project config defines one review agent:

```kdl
reviewography {
  review {
    default_scope "incremental"

    schema_retry {
      max_attempts 3
    }

    agent {
      provider "codex"
      model "gpt-5.5-codex"
      effort "high"
    }
  }
}
```

`provider` is required.

`model`, `subagent`, and `effort` are interpreted by the selected provider adapter and may be optional depending on that provider.

## Single-agent v0

v0 intentionally does not orchestrate multi-agent review.

One config means one review agent.

If a user wants multi-agent review, they can run multiple processes with different config files:

```bash
revg review run -c reviewography.codex.kdl
revg review run -c reviewography.claude.kdl
```

reviewography v0 does not merge or deduplicate those outputs automatically.

## Schema retry

AI review output must validate against reviewography's review finding schema before it becomes canonical data.

If generated JSON is schema-invalid, reviewography asks the agent to regenerate the output.

`max_attempts` includes the initial generation.

```text
max_attempts = 1
  initial generation only

max_attempts = 3
  initial generation + 2 regenerations
```

v0 constraints:

```text
default: 3
minimum: 1
maximum: 10
```

Schema-invalid output is not saved as a canonical review record.

Raw invalid output may be saved only as run-local debug data.

## Review records

reviewography distinguishes several record types.

### Review session

A review session represents one worktree-local review procedure.

It records:

- session ID
- base revision
- current head
- last reviewed head
- review runs
- unresolved findings
- generated proposals

### Review run

A review run represents one execution of `revg review run`.

It records:

- scope
- base revision
- head revision
- review cursor before and after the run
- criteria used as review context
- agent configuration metadata
- structured findings
- validation attempts
- generated temporary views

### Review finding

A review finding is a concrete issue found in a specific review run.

Example:

```json
{
  "artifact_type": "review_finding",
  "schema_version": "0",
  "id": "finding-20260622-001",
  "category": "test-gap",
  "severity": "medium",
  "target": {
    "kind": "module",
    "path": "cmd/tools/run"
  },
  "claim": "Hidden command visibility is not directly tested through user-facing help output.",
  "evidence": [
    {
      "kind": "file",
      "path": "cmd/tools/run_test.go",
      "note": "No test asserts that the hidden command is excluded from normal help output."
    }
  ],
  "recommended_action": "Add a help output test for hidden command visibility.",
  "resolution": "unresolved"
}
```

### Review criteria

A review criterion is reusable review knowledge.

Example:

```json
{
  "artifact_type": "review_criteria",
  "schema_version": "0",
  "id": "criteria-hidden-command-help-visibility",
  "title": "Hidden command visibility should be tested through help output",
  "body": "When a CLI command is hidden, its visibility should be verified through user-facing help output rather than only through implementation flags.",
  "weight": 5,
  "status": "active"
}
```

The canonical criteria data is JSON.

Markdown, text, and TOON outputs are generated views.

## Criteria proposals

`revg review fold` does not directly update criteria.

It creates criteria update proposals from findings produced in the current session.

```bash
revg review fold
```

A proposal is applied explicitly:

```bash
revg criteria apply --local <proposal-id>
revg criteria apply --global <proposal-id>
```

`--local` applies the proposal to repository-level criteria.

`--global` applies the proposal to user-level criteria.

v0 excludes user-level config, but user-level criteria may still exist as a separate criteria store. Because user-level criteria are environment-dependent, repository-level criteria should be preferred when review reproducibility matters.

## Configuration

v0 uses project-local config by default.

Default config path:

```text
reviewography.config.kdl
```

A different config file may be specified with:

```bash
-c, --config <path>
```

Example:

```bash
revg review run -c reviewography.codex.kdl
```

v0 does not support user-level config.

Configuration lookup order:

```text
1. -c / --config
2. ./reviewography.config.kdl
3. built-in defaults
```

## Local-first storage

A typical project may store reviewography data under:

```text
.reviewography/
  session.json
  criteria/
  proposals/
  records/
    findings/
    runs/
  views/
  tmp/
```

The exact storage layout may evolve, but the principle is stable:

- structured JSON data is canonical
- local files are the primary storage
- generated human-readable views are secondary
- worktree-local session state is temporary

A conservative default is:

```text
commit:
  shared criteria
  explicit project review policies

do not commit by default:
  active session files
  raw AI review output
  temporary views
  run-local metadata
  sensitive findings
```

## Human review

reviewography does not replace human review.

It changes the unit of review.

Instead of asking humans to inspect every changed line first, it helps them inspect:

- unresolved findings
- relevant criteria
- evidence for each finding
- repeated issue classes
- rejected or out-of-scope concerns
- criteria proposals derived from the session

Human review remains responsible for final judgment.

Manual GitHub PR review is expected to happen through GitHub PR comments or reviews. GitHub Action integration for importing those comments into reviewography records is outside v0 and should be designed together with that action.

## Status

reviewography v0 should focus on:

1. worktree-local review sessions
2. single-agent review execution
3. schema-valid review finding records
4. incremental and full review scopes
5. unresolved finding management
6. criteria display, search, weighting, editing, and export
7. criteria proposal generation and explicit application
8. temporary human-readable views

Implementation-review loops, multi-agent orchestration, GitHub PR comment ingestion, repository audits, and test evidence extraction are intentionally out of scope for the core v0 tool.
