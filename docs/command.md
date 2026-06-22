# reviewography command design

This document defines the initial CLI command design for `reviewography`.

The canonical command name is:

```bash
reviewography
```

A short alias may be provided:

```bash
revg
```

The alias should behave identically to `reviewography`.

## Design principles

The CLI should follow these principles:

1. Structured data is canonical.
2. Human-readable output is a generated view.
3. One command should support the normal one-step workflow.
4. Implementation-review loops are out of scope.
5. Local files are the default storage.
6. Commands should be composable with external tools.
7. Review records should be stable enough for AI agents to consume.

## Primary command

### `reviewography review`

Run one review pass against a change.

```bash
reviewography review --base main --head HEAD
```

Alias:

```bash
revg review --base main --head HEAD
```

This command performs the core workflow:

```text
load change
  → load relevant prior review records
  → review change
  → record findings
  → generate output
```

#### Options

```text
--base <ref>
  Git base revision.

--head <ref>
  Git head revision.

--issue <path>
  Optional issue or task description file.

--adr <path>
  Optional ADR or ADR plan file.

--input-review <path>
  Optional external review output to ingest during the review.

--source <name>
  Source name for external review input.
  Example: codex, chatgpt, manual.

--out <path>
  Output directory.
  Default: .reviewography/runs/<run-id>/

--format <format>
  Human-facing output format.
  Supported initial values: text, markdown, json.

--no-view
  Do not generate a human-readable temporary view.

--config <path>
  Path to reviewography config file.

--profile <name>
  Review profile to use.

--strict
  Fail when review records are invalid or ambiguous.

--dry-run
  Print planned actions without writing records.
```

#### Output

The command should write structured records.

Example:

```text
.reviewography/runs/20260622-001/
  review-run.json
  relevant-patterns.json
  findings.json
  view.md
```

`view.md` is temporary.

The canonical outputs are:

```text
review-run.json
relevant-patterns.json
findings.json
```

#### Exit codes

Suggested exit codes:

```text
0  review completed without findings
1  review completed with findings
2  invalid input or configuration
3  review execution failed
4  record validation failed
```

A finding is not necessarily a command failure.
However, exit code `1` is useful for scripts that want to stop on review findings.

## Initialization

### `reviewography init`

Create initial local configuration.

```bash
reviewography init
```

Expected output:

```text
.reviewography/
  config.json
  records/
    findings/
    patterns/
    decisions/
  runs/
```

Options:

```text
--force
  Overwrite existing configuration.

--minimal
  Create only the minimum required files.

--gitignore
  Add recommended run-local paths to .gitignore.
```

Recommended `.gitignore` entries:

```gitignore
.reviewography/runs/
.reviewography/tmp/
```

Projects may choose to commit selected files under:

```text
.reviewography/records/patterns/
```

or export stable review patterns into a project documentation directory.

## Ingesting external review output

### `reviewography ingest`

Convert external review output into structured findings.

```bash
reviewography ingest codex-review.md --source codex
```

Options:

```text
--source <name>
  Review source name.
  Example: codex, chatgpt, manual.

--issue <id-or-path>
  Optional issue identifier or issue file.

--run <run-id>
  Attach ingested findings to an existing run.

--out <path>
  Output path.

--decision <state>
  Initial decision state.
  Default: proposed.
```

Example:

```bash
reviewography ingest codex-review.md \
  --source codex \
  --issue 51 \
  --out .reviewography/runs/20260622-001/findings.json
```

The command should not automatically accept findings.

Default decision state is:

```text
proposed
```

## Deciding findings

### `reviewography decide`

Record a decision for a finding or pattern.

```bash
reviewography decide finding-20260622-001 --decision accepted
```

Options:

```text
--decision <state>
  One of:
  proposed
  accepted
  rejected
  deferred
  needs-human-decision
  accepted-negative
  superseded

--reason <text>
  Reason for the decision.

--by <name>
  Optional decision maker label.

--out <path>
  Optional output path for the decision record.
```

Examples:

```bash
reviewography decide finding-20260622-001 \
  --decision accepted \
  --reason "The test gap is in scope for this change."
```

```bash
reviewography decide finding-20260622-002 \
  --decision accepted-negative \
  --reason "Before v0, backward compatibility should not block simplification."
```

## Generalizing findings

### `reviewography generalize`

Create or update a reusable review pattern from a concrete finding.

```bash
reviewography generalize finding-20260622-001
```

Options:

```text
--title <text>
  Explicit title for the generated pattern.

--category <category>
  Pattern category.

--status <state>
  Initial pattern status.
  Default: proposed.

--out <path>
  Output path.

--merge-with <pattern-id>
  Merge the finding into an existing pattern instead of creating a new one.
```

Example:

```bash
reviewography generalize finding-20260622-001 \
  --title "Hidden command visibility should be tested through user-facing help output"
```

## Searching relevant review memory

### `reviewography relevant`

Find review patterns relevant to a change or context.

```bash
reviewography relevant --base main --head HEAD
```

Options:

```text
--base <ref>
  Git base revision.

--head <ref>
  Git head revision.

--changed-files <path>
  File containing changed paths.

--issue <path>
  Issue or task description.

--adr <path>
  ADR or ADR plan.

--category <category>
  Filter by category.

--format <format>
  Output format: json, markdown, text.
```

Example:

```bash
reviewography relevant \
  --changed-files changed-files.txt \
  --issue issue.md \
  --format markdown
```

This command does not perform a review.
It only retrieves relevant prior review memory.

## Generating temporary views

### `reviewography view`

Generate a human-readable view from structured review records.

```bash
reviewography view .reviewography/runs/20260622-001/findings.json
```

Options:

```text
--format <format>
  View format.
  Supported initial values: markdown, text.

--out <path>
  Output file.

--include-patterns
  Include relevant prior patterns.

--include-decisions
  Include decision records.

--include-no-findings
  Include explicit no-finding result when available.
```

Example:

```bash
reviewography view .reviewography/runs/20260622-001/review-run.json \
  --format markdown \
  --out .reviewography/runs/20260622-001/view.md
```

The generated view is not canonical.

## Validating records

### `reviewography validate`

Validate reviewography records.

```bash
reviewography validate .reviewography/
```

Options:

```text
--strict
  Treat warnings as errors.

--schema <path>
  Use a specific schema file.

--format <format>
  Output format: text, json.
```

Validation should check:

* required fields
* artifact type
* schema version
* valid decision states
* references to findings and patterns
* duplicate IDs
* malformed evidence entries

## Exporting records

### `reviewography export`

Export selected review memory.

```bash
reviewography export --patterns --out review-patterns.json
```

Options:

```text
--findings
  Export findings.

--patterns
  Export patterns.

--decisions
  Export decisions.

--since <date>
  Export records since date.

--category <category>
  Filter by category.

--format <format>
  Output format: json, markdown.
```

Exported records may be used by other tools.

## Suggested command set for v0

The minimum useful command set is:

```bash
reviewography init
reviewography review
reviewography ingest
reviewography decide
reviewography generalize
reviewography relevant
reviewography view
reviewography validate
```

The most important v0 command is:

```bash
reviewography review --base main --head HEAD
```

This should support the primary one-step workflow.

## Command scope boundaries

### In scope

```text
- reviewing a change once
- loading relevant prior review memory
- recording new findings
- ingesting external review output
- recording decisions
- generalizing findings into patterns
- generating temporary human-readable views
- validating structured review records
```

### Out of scope

```text
- running Claude Code
- running Codex repeatedly
- automatically fixing findings
- looping until findings disappear
- creating GitHub Issues
- creating Pull Requests
- merging changes
- creating ADRs
- performing repository-wide audits
- extracting test evidence from source code
```

External tools or skills may compose reviewography with those workflows.
