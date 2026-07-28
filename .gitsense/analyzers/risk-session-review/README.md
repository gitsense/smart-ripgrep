# Risk Session Review Analyzer

`risk-session-review` stores deterministic change-risk review items for file-read, targeted-edit, and complete-file-write events in a coding-agent session.

This smart-ripgrep version is tuned for change review: it combines file purpose, agent triage, Rust blast radius, test-coverage metadata, and repository lessons so a reviewer can quickly see what a read or edit brought into play. Each item has a group, stable key, short title, and Markdown detail, allowing a session metadata index to group matching concerns without interpreting prose.

The Analyzer is a data contract. Its values are populated by [build-risk-session-review](../../bin/build-risk-session-review), which combines existing GitSense analysis without making another LLM call.

## Fields

| Field | Purpose |
| :--- | :--- |
| `read_items` | Classified file-context items explaining what a read brought into context. |
| `edit_items` | Independently reviewable file-context, change-risk, API, dependency, and test items. |
| `write_items` | Marks a complete-file write separately, then includes the same file-context, risk, API, dependency, and test items. |
| `Repository memory` items | One item per repository lesson matched by the file path in `applies_to.files` or `applies_to.linked_files`. These show available guidance; they do not claim that the agent consulted or followed the lesson. |
| `source_analyzers` | Records which Analyzers contributed metadata. |
| `source_fingerprint` | Detects when the source metadata changed. |

Every item contains:

```json
{
  "group": "Change risk",
  "key": "change-risk:high",
  "title": "High-risk change",
  "markdown": "Downstream tools parse this output."
}
```

## Install

Copy the Analyzer into the GitSense Chat Analyzer directory:

```bash
GSC_ANALYZERS_DIR="${GSC_HOME:?Set GSC_HOME}/data/analyzers"
mkdir -p "$GSC_ANALYZERS_DIR"
cp -R .gitsense/analyzers/risk-session-review "$GSC_ANALYZERS_DIR/"
```

Restart GitSense Chat after installing a new Analyzer.

## Required Source Analysis

The deterministic builder reads these Analyzers:

* `code-intent`
* `agent-file-triage`
* `rust-blast-radius`
* `rust-test-coverage-intent`

The builder also reads repository-local `.gitsense/lessons/records.jsonl`. Lesson paths are matched deterministically against both `applies_to.files` and `applies_to.linked_files`. A missing lesson file is valid and produces no repository-memory items.

The source repository and branch must already be imported into GitSense Chat, and the source analysis must be available there. Missing metadata for an individual file is allowed; the corresponding Markdown section is omitted.

## Build and Review

Generate reviewable JSONL without writing analysis:

```bash
.gitsense/bin/build-risk-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --output /tmp/risk-session-review.jsonl
```

Validate the import path without writing:

```bash
.gitsense/bin/build-risk-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --import \
  --dry-run
```

Populate the Analyzer after reviewing the output:

```bash
.gitsense/bin/build-risk-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --import
```

The builder hashes canonical source metadata and skips unchanged records.

## Adapt the Pattern

The review items are produced by deterministic functions in `.gitsense/bin/build-risk-session-review`. Another repository can keep the same Analyzer shape, then change the source Analyzer list and the `buildReadItems`, `buildEditItems`, or `buildWriteItems` mappings to match what its reviewers need.

The Analyzer contract remains the same even when a repository chooses different source metadata or presentation rules.

## Enrich a Pi Session Export

After packaging or loading the `risk-session-review` results as a Brain:

```bash
gsc pi sessions export \
  --format gsc-json \
  --metadata 'read::risk-session-review::read_items' \
  --metadata 'edit::risk-session-review::edit_items' \
  --metadata 'write::risk-session-review::write_items'
```

`write_items` tells the reviewer that the agent supplied complete file contents. That may mean a new file or a replacement of an existing one, so the Analyzer does not guess which happened without separate session or filesystem evidence.
