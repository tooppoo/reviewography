# reviewography command design

This document defines the initial CLI command design for `reviewography` v0.

The canonical command name is:

```bash
reviewography
```

The short alias is:

```bash
revg
```

The alias should behave identically to `reviewography`.

## Design principles

The CLI should follow these principles:

1. Structured JSON data is canonical.
2. Human-readable output is a generated temporary view.
3. Review sessions are worktree-local.
4. `review start` requires a base revision.
5. `review run` performs one AI review pass through one configured agent.
6. Agent output must pass schema validation before it becomes canonical data.
7. Schema-invalid agent output is retried by the tool, not by external skills.
8. Implementation-review loops are out of scope.
9. Multi-agent orchestration is out of scope for v0.
10. Project-local config is the default; user-level config is out of scope for v0.
11. Criteria may exist at repository level or user level.
12. Commands should be composable with external tools, skills, and GitHub Actions.

## Common options

### `-c, --config <path>`

Use a specific config file.

```bash
revg review run -c reviewography.codex.kdl
```

Configuration lookup order:

```text
1. -c / --config
2. ./reviewography.config.kdl
3. built-in defaults
```

v0 does not support user-level config.

A user who wants multiple review agents may run multiple processes with different config files.

```bash
revg review run -c reviewography.codex.kdl
revg review run -c reviewography.claude.kdl
```

reviewography v0 does not merge or deduplicate those outputs automatically.

## Configuration file

The default config file is:

```text
reviewography.config.kdl
```

Example:

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

  criteria {
    local ".reviewography/criteria"
    global "~/.config/reviewography/criteria"
  }

  output {
    default_format "markdown"
  }
}
```

Only one `agent` is active for a single `review run`.

`provider` is required.

`model`, `subagent`, and `effort` are provider-specific and may be optional depending on the selected provider adapter.

## Initialization

### `revg init`

Initialize reviewography for the repository.

```bash
revg init
```

Expected output:

```text
.reviewography/
  criteria/
  proposals/
  records/
    findings/
    runs/
  views/
  tmp/
reviewography.config.kdl
```

Options:

```text
--force
  Overwrite existing local configuration.

--minimal
  Create only the minimum required files.

--gitignore
  Add recommended run-local paths to .gitignore.
```

Recommended `.gitignore` entries:

```gitignore
.reviewography/session.json
.reviewography/records/runs/
.reviewography/views/
.reviewography/tmp/
```

Projects may choose to commit selected files under:

```text
.reviewography/criteria/
```

## Review session commands

### `revg review start`

Start a worktree-local review session.

```bash
revg review start --base main
```

`--base` is required.

Options:

```text
--base <revision>
  Required. Git base revision for the whole review session.

-c, --config <path>
  Config file to use.
```

The session stores:

```json
{
  "session_id": "revg-session-20260622-001",
  "base": "main",
  "current_head": "HEAD",
  "last_reviewed_head": null
}
```

`base` is stable for the whole session.

`last_reviewed_head` is the review cursor used by incremental review.

When using Git worktrees, each worktree should have its own active session file at the worktree root.

### `revg review status`

Show active session status.

```bash
revg review status
```

Example output:

```text
active session: yes
session id: revg-session-20260622-001
base: main
head: HEAD
last reviewed head: abc123
findings: 5
unresolved: 2
criteria proposals: 1
```

### `revg review run`

Run one AI review pass in the active session.

```bash
revg review run
```

Default scope is `incremental`.

```bash
revg review run --scope incremental
```

Full review:

```bash
revg review run --scope full
```

Options:

```text
--scope <incremental|full>
  Review scope.
  Default: config review.default_scope, otherwise incremental.

--format <json|toon|markdown|text>
  Temporary view output format.

--no-view
  Do not generate a human-readable temporary view.

--strict
  Fail on invalid records, ambiguous session state, or invalid config.

--dry-run
  Build the review context and planned agent request without saving findings.

-c, --config <path>
  Config file to use.
```

Scope behavior:

```text
incremental
  Focus on last_reviewed_head..HEAD.
  Include base..HEAD summary context, unresolved findings, and relevant criteria.

full
  Review base..HEAD as a whole.
  Intended before human review, PR creation, or session end.
```

There is no `final` scope.

A full review may still produce findings. If it does, the session should continue.

`review run` performs:

```text
load active session
  -> compute review scope
  -> load relevant criteria
  -> load unresolved findings
  -> generate review context
  -> invoke the configured single agent
  -> extract JSON
  -> validate schema
  -> retry if schema-invalid
  -> save valid findings
  -> generate temporary view
  -> update last_reviewed_head
```

#### Agent schema retry

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

If every attempt fails, the review run fails.

Invalid JSON or schema-invalid JSON is not saved as canonical finding data.

### `revg review show`

Show findings in the active session.

```bash
revg review show
```

Show only unresolved findings:

```bash
revg review show --unresolved
```

Options:

```text
--unresolved
  Show only unresolved findings.

--format <json|toon|markdown|text>
  Output format.

-c, --config <path>
  Config file to use.
```

### `revg review resolve`

Mark a finding as resolved by a specific resolution state.

```bash
revg review resolve <review-id> --as fixed
```

Options:

```text
--as <state>
  Required resolution state.

--reason <text>
  Reason for the resolution.

-c, --config <path>
  Config file to use.
```

Suggested v0 resolution states:

```text
fixed
rejected
deferred
accepted-negative
superseded
duplicate
```

Examples:

```bash
revg review resolve finding-20260622-001 --as fixed
```

```bash
revg review resolve finding-20260622-002 \
  --as accepted-negative \
  --reason "Before v0, backward compatibility should not block simplification."
```

### `revg review fold`

Create criteria update proposals from findings in the active session.

```bash
revg review fold
```

`fold` does not directly update criteria.

It writes proposal JSON, for example:

```text
.reviewography/proposals/proposal-20260622-001.json
```

Options:

```text
--format <json|toon|markdown|text>
  Temporary view output format.

-c, --config <path>
  Config file to use.
```

### `revg review end`

End the active review session.

```bash
revg review end
```

Ending a session should remove or clear the active session pointer.

It should not delete canonical findings, run records, or criteria proposals.

If unresolved findings remain, the command should warn.

Options:

```text
--force
  End the session even if unresolved findings remain.

-c, --config <path>
  Config file to use.
```

## Criteria commands

Criteria are reusable review standards.

The canonical criteria representation is JSON.

Human-readable Markdown, text, or TOON output is a generated view.

### `revg criteria list`

List criteria IDs, titles, weights, and status.

```bash
revg criteria list
```

Options:

```text
--format <json|toon|markdown|text>
  Output format.

-c, --config <path>
  Config file to use.
```

### `revg criteria show`

Show criteria.

```bash
revg criteria show --format markdown
revg criteria show --format json
revg criteria show --format toon
```

Options:

```text
--format <json|toon|markdown|text>
  Output format.

--id <criteria-id>
  Show only one criterion.

-c, --config <path>
  Config file to use.
```

`--format` is the canonical form. Short flags such as `--json` or `--toon` may be added later as aliases, but the internal representation should normalize to `--format`.

### `revg criteria search`

Search criteria.

```bash
revg criteria search "hidden command"
```

Options:

```text
--category <category>
  Filter by category.

--path <path>
  Search criteria relevant to a path.

--format <json|toon|markdown|text>
  Output format.

-c, --config <path>
  Config file to use.
```

### `revg criteria weight up`

Increase criteria weight.

```bash
revg criteria weight up <criteria-id>
revg criteria weight up <criteria-id> 2
```

The default amount is `1`.

`weight` means review priority. It should not be used as occurrence count.

### `revg criteria weight down`

Decrease criteria weight.

```bash
revg criteria weight down <criteria-id>
revg criteria weight down <criteria-id> 2
```

The default amount is `1`.

### `revg criteria edit`

Edit a criteria field through an editor buffer.

```bash
revg criteria edit <criteria-id>
revg criteria edit --body <criteria-id>
revg criteria edit --title <criteria-id>
```

`revg criteria edit <criteria-id>` is equivalent to:

```bash
revg criteria edit --body <criteria-id>
```

v0 supports editing only:

```text
title
body
```

The command should not expose the full JSON record for direct editing.

It should:

```text
1. read the criteria JSON
2. extract the selected field
3. open the field content in $EDITOR
4. write the updated field back to JSON
5. validate the updated criteria record
```

### `revg criteria deprecate`

Mark criteria as deprecated without deleting its history.

```bash
revg criteria deprecate <criteria-id> --reason "covered by broader CLI public surface criterion"
```

### `revg criteria apply`

Apply a criteria proposal.

```bash
revg criteria apply --local <proposal-id>
revg criteria apply --global <proposal-id>
```

Options:

```text
--local <proposal-id>
  Apply the proposal to repository-level criteria.

--global <proposal-id>
  Apply the proposal to user-level criteria.

-c, --config <path>
  Config file to use.
```

v0 excludes user-level config, but user-level criteria may still be supported as a criteria store.

Because user-level criteria are environment-dependent, repository-level criteria should be preferred when review reproducibility matters.

### `revg criteria export`

Export criteria.

```bash
revg criteria export --format json --out criteria.json
revg criteria export --format markdown --out criteria.md
```

Options:

```text
--format <json|toon|markdown|text>
  Export format.

--out <path>
  Output path.

--scope <local|global|all>
  Criteria scope to export.

-c, --config <path>
  Config file to use.
```

v0 does not need a dedicated `criteria import` command.

A user may place exported criteria files in the expected criteria location and then run validation.

### `revg criteria validate`

Validate criteria records.

```bash
revg criteria validate
```

Validation should check:

- required fields
- artifact type
- schema version
- unique IDs
- valid status
- valid weight
- valid proposal references
- malformed body/title content

Options:

```text
--strict
  Treat warnings as errors.

--format <json|toon|markdown|text>
  Output format.

-c, --config <path>
  Config file to use.
```

## Tools commands

### `revg tools install`

Install helper integration material.

```bash
revg tools install <tool-name>
```

Initial tool names:

```text
codex-skill
claude-skill
```

Examples:

```bash
revg tools install codex-skill
revg tools install claude-skill
```

These helpers should teach the target AI environment how to use reviewography, but they should not replace reviewography's schema validation and retry mechanism.

## Suggested v0 command set

```text
revg init

revg review start --base <revision>
revg review status
revg review run
revg review run --scope incremental
revg review run --scope full
revg review show
revg review show --unresolved
revg review resolve <review-id> --as <state>
revg review fold
revg review end

revg criteria list
revg criteria show --format <json|toon|markdown|text>
revg criteria search <query>
revg criteria weight up <criteria-id> [num]
revg criteria weight down <criteria-id> [num]
revg criteria edit <criteria-id>
revg criteria edit --body <criteria-id>
revg criteria edit --title <criteria-id>
revg criteria deprecate <criteria-id>
revg criteria apply --local <proposal-id>
revg criteria apply --global <proposal-id>
revg criteria export --format <json|toon|markdown|text>
revg criteria validate

revg tools install <tool-name>
```

## Exit codes

Suggested exit codes:

```text
0  command completed successfully
1  review completed with unresolved findings
2  invalid input or configuration
3  agent execution failed
4  schema validation failed after max_attempts
5  record validation failed
6  no active session
```

A finding is not necessarily a command failure.

However, exit code `1` is useful for scripts that want to stop when unresolved findings are produced.

## Command scope boundaries

### In scope

```text
- worktree-local session management
- base and review cursor management
- single-agent AI review execution
- review context generation
- criteria lookup
- schema validation
- retry on schema-invalid agent output
- finding persistence
- unresolved finding display
- explicit finding resolution
- criteria proposal generation
- criteria application
- criteria display, search, weighting, editing, deprecation, export, and validation
- temporary human-readable views
```

### Out of scope

```text
- implementation changes
- automatic fix loops
- running review until findings disappear
- multi-agent orchestration
- finding merge or deduplication across agents
- GitHub Issue management
- GitHub Pull Request creation or merge
- GitHub PR comment ingestion
- ADR creation
- repository-wide audits
- test evidence extraction from source code
- user-level config
```

External tools, skills, scripts, or future GitHub Actions may compose reviewography with those workflows.
