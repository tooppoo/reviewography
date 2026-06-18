# kogoto-review-log

`kogoto-review-log` is a small review-policy log for AI-assisted development workflows.

It records review findings after human triage and turns them into reusable project-specific review guidance.

The tool is designed to work both as:

* a standalone skill for review workflows, and
* a Kogoto plugin for issue-driven implementation / review loops.

## Motivation

AI reviewers often produce useful findings, but not every finding should become future guidance.

Some findings are accepted because they reveal real project concerns.
Some findings are rejected because they are false positives, out of scope, or simply not important for the project.

`kogoto-review-log` exists to preserve that human judgment.

It does not treat AI review output as authoritative.
Instead, it records what the human accepted or rejected, and uses those decisions to build a reusable review policy for the project.

## What this tool is

`kogoto-review-log` is a curated review policy log.

It helps answer questions such as:

* What kinds of review findings does this project care about?
* What kinds of review findings are repeatedly accepted?
* What kinds of review findings are intentionally not prioritized?
* What guidance should be passed to the next AI reviewer?

## What this tool is not

`kogoto-review-log` is not:

* an AI reviewer
* an automatic code fixer
* a replacement for human review
* a raw archive of every AI review message
* a general knowledge base
* a workflow orchestrator

Kogoto may orchestrate the workflow.
`kogoto-review-log` records the review judgments that should persist across review sessions.

## Core workflow

The intended workflow is:

1. Run an AI review.
2. List review findings.
3. Let a human decide whether each finding should be accepted or rejected.
4. Compare accepted or rejected findings with existing review-policy entries.
5. Update the review log:

   * If an accepted finding matches an existing emphasized entry, increment its occurrence count.
   * If it does not match an existing entry, add a new emphasized entry.
   * If a rejected finding represents something the project does not want to prioritize, record it as a deemphasized entry.
   * If a matching deemphasized entry already exists, increment its occurrence count.
6. Generate review guidance for future AI review sessions.

## Key concepts

### Review finding

A `ReviewFinding` is a concrete finding from a review session.

It represents what the reviewer pointed out, before it is turned into a reusable project policy.

Example:

```json
{
  "id": "finding-001",
  "summary": "The schema allows duplicate evidence IDs.",
  "rationale": "Duplicate IDs make evidence references ambiguous.",
  "category": "correctness",
  "severity": "major",
  "target": {
    "path": "schemas/evidence/evidence.v0.json"
  }
}
```

### Review decision

A `ReviewDecision` is the human judgment attached to a finding.

Supported decision types may include:

```text
accept
reject_as_false_positive
reject_as_out_of_scope
reject_as_not_important
defer
```

Only some decisions should normally update the reusable policy log.

In v0, the main policy-producing decisions are:

* `accept`
* `reject_as_not_important`

### Review policy entry

A `ReviewPolicyEntry` is a reusable review guidance item derived from human-triaged findings.

It can be either:

* `emphasized`: this project cares about this review concern
* `deemphasized`: this project does not want to prioritize this kind of concern

Example:

```json
{
  "id": "review-policy-001",
  "status": "emphasized",
  "summary": "Evidence references must point to existing IDs.",
  "rationale": "Broken evidence references make review results unverifiable.",
  "category": "correctness",
  "occurrence_count": 3,
  "examples": [
    {
      "review_id": "review-2026-06-18-001",
      "finding_id": "finding-002",
      "decision": "accept"
    }
  ]
}
```

Example of a deemphasized entry:

```json
{
  "id": "review-policy-002",
  "status": "deemphasized",
  "summary": "Do not prioritize purely stylistic rewrites when behavior and interface are unchanged.",
  "rationale": "This project prefers minimal diffs unless clarity, correctness, or maintainability is materially improved.",
  "category": "style",
  "occurrence_count": 2,
  "examples": [
    {
      "review_id": "review-2026-06-18-002",
      "finding_id": "finding-004",
      "decision": "reject_as_not_important"
    }
  ]
}
```

## Emphasized and deemphasized entries

`kogoto-review-log` records both positive and negative review preferences.

### Emphasized entries

An emphasized entry means:

> This project wants future reviewers to pay attention to this concern.

Examples:

* Schema IDs must be unique.
* Evidence references must not be broken.
* Dry-run commands must not perform destructive operations.
* JSON output must be validated against the current schema version.

### Deemphasized entries

A deemphasized entry means:

> This project intentionally does not want future reviewers to prioritize this concern.

Examples:

* Do not request stylistic rewrites when behavior and public interface are unchanged.
* Do not require abstraction before duplication has become a demonstrated maintenance problem.
* Do not treat missing future extensibility as a blocker for v0 work.

Deemphasized entries are not ignored facts.
They are explicit review preferences.

## Occurrence count, weight, and priority

A finding may appear repeatedly, but frequency alone does not prove importance.

For that reason, the model should distinguish:

* `occurrence_count`: how many times this concern has appeared
* `weight`: how strongly the project cares about it
* `priority`: how prominently it should appear in future review guidance

v0 may start with `occurrence_count` only, but the schema should leave room for weight and priority.

## Planned CLI shape

The CLI is still experimental, but the intended shape is:

```sh
kogoto-review-log init
kogoto-review-log import review.md
kogoto-review-log list
kogoto-review-log decide finding-001 --accept
kogoto-review-log decide finding-002 --reject-as-not-important
kogoto-review-log link finding-001 --entry review-policy-003
kogoto-review-log add-entry --from finding-002 --status deemphasized
kogoto-review-log guidance
```

The exact command names may change before the first stable release.

## Standalone usage

As a standalone skill, `kogoto-review-log` can be used with any AI review workflow.

A typical flow:

```sh
# Import review output
kogoto-review-log import review.md

# Inspect extracted findings
kogoto-review-log list

# Record human decisions
kogoto-review-log decide finding-001 --accept
kogoto-review-log decide finding-002 --reject-as-not-important

# Update the reusable policy log
kogoto-review-log update-log

# Generate guidance for the next review
kogoto-review-log guidance
```

## Kogoto integration

When used with Kogoto, `kogoto-review-log` should act as a plugin-like component.

Kogoto owns the workflow:

```text
issue selection
  -> implementation
  -> review
  -> human triage
  -> fix
  -> re-review
```

`kogoto-review-log` owns the persistent review policy:

```text
review finding
  -> human decision
  -> emphasized / deemphasized policy entry
  -> future review guidance
```

Possible integration points:

```text
after_review_completed
after_finding_triaged
after_fix_completed
after_rereview_completed
before_next_review
```

Kogoto can use `kogoto-review-log guidance` to provide project-specific review guidance to the next AI reviewer.

## Storage model

A standalone project may store review policy data under:

```text
.kogoto-review-log/
  entries/
    review-policy-001.json
    review-policy-002.json
  reviews/
    review-2026-06-18-001.json
  guidance.md
```

When integrated with Kogoto, run-local review facts and project-level policy should remain separate.

Example:

```text
.kogoto/
  runs/
    issue-123/
      run-001/
        review-findings.json

.kogoto-review-log/
  entries/
    review-policy-001.json
    review-policy-002.json
```

The Kogoto run directory records what happened in a specific run.
The `kogoto-review-log` directory records reusable project review policy.

## Generated review guidance

The guidance output should be concise enough to pass to an AI reviewer.

Example:

```md
# Review Guidance

## Emphasize

- Evidence references must point to existing IDs.
  - Seen: 3 times
  - Reason: Broken references make review results unverifiable.

- Dry-run commands must not perform destructive operations.
  - Seen: 2 times
  - Reason: Users rely on dry-run for safe inspection.

## Deemphasize

- Do not prioritize purely stylistic rewrites when behavior and interface are unchanged.
  - Seen: 2 times
  - Reason: This project prefers minimal diffs unless clarity or correctness is materially improved.
```

## Design principles

### Human judgment is required

AI review output is not treated as ground truth.

A finding should only affect reusable project guidance after a human decision.

### Raw findings are not reusable policy

A raw review finding is an event.
A review policy entry is a curated project preference.

The tool should preserve this distinction.

### Rejected findings are not always useless

A rejected finding may still be valuable when it expresses a project preference.

For example:

> This project does not treat purely stylistic rewrites as important unless they improve correctness, clarity, or maintainability.

Such preferences help future reviewers avoid repeating low-value comments.

### False positives should not become policy

A finding rejected as factually wrong should not become a normal deemphasized policy entry.

False positives may be stored separately for reviewer evaluation, but they should not be confused with project preferences.

### Project policy should remain reviewable

Review guidance can become stale.

Policy entries should be editable, removable, and auditable.

## v0 scope

The initial version should focus on:

* importing review findings
* listing findings
* recording human decisions
* adding emphasized entries
* adding deemphasized entries
* incrementing occurrence counts for matching entries
* generating Markdown review guidance

## Out of scope for v0

The following are intentionally out of scope for v0:

* automatic AI review execution
* automatic code modification
* semantic deduplication without human confirmation
* vector database integration
* organization-wide policy sharing
* automatic ADR generation
* complex UI

## Future ideas

Possible future features:

* JSON Schema for review findings and policy entries
* TOON output for AI-friendly context passing
* reviewer-specific false-positive tracking
* Kogoto hook integration
* policy entry expiration or review dates
* ADR candidate generation
* guidance profiles by category or path
* machine-assisted duplicate detection
* project-level and user-level review preferences

## License

Apache License 2.0
