# Session Review Analyzer

`session-review` stores deterministic Markdown designed to appear beneath file-read and file-edit events in a coding-agent session.

This smart-ripgrep version is tuned for change review: it combines file purpose, agent triage, Rust blast radius, and test-coverage metadata so a reviewer can quickly see what a read or edit brought into play.

The Analyzer is a data contract. Its values are populated by [build-session-review](../../bin/build-session-review), which combines existing GitSense analysis without making another LLM call.

## Fields

| Field | Purpose |
| :--- | :--- |
| `read_md` | Explains what a file does, where it fits, and when it is useful to inspect. |
| `edit_md` | Explains change risk, dependency or public API exposure, and existing test safeguards. |
| `source_analyzers` | Records which Analyzers contributed metadata. |
| `source_fingerprint` | Detects when the source metadata changed. |

## Install

Copy the Analyzer into the GitSense Chat Analyzer directory:

```bash
GSC_ANALYZERS_DIR="${GSC_HOME:?Set GSC_HOME}/data/analyzers"
mkdir -p "$GSC_ANALYZERS_DIR"
cp -R .gitsense/analyzers/session-review "$GSC_ANALYZERS_DIR/"
```

Restart GitSense Chat after installing a new Analyzer.

## Required Source Analysis

The deterministic builder reads these Analyzers:

* `code-intent`
* `agent-file-triage`
* `rust-blast-radius`
* `rust-test-coverage-intent`

The source repository and branch must already be imported into GitSense Chat, and the source analysis must be available there. Missing metadata for an individual file is allowed; the corresponding Markdown section is omitted.

## Build and Review

Generate reviewable JSONL without writing analysis:

```bash
.gitsense/bin/build-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --output /tmp/session-review.jsonl
```

Validate the import path without writing:

```bash
.gitsense/bin/build-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --import \
  --dry-run
```

Populate the Analyzer after reviewing the output:

```bash
.gitsense/bin/build-session-review \
  --owner gitsense \
  --repo smart-ripgrep \
  --branch master \
  --import
```

The builder hashes canonical source metadata and skips unchanged records.

## Adapt the Pattern

The Markdown is produced by deterministic functions in `.gitsense/bin/build-session-review`. Another repository can keep the same Analyzer shape, then change the source Analyzer list and the `renderReadMarkdown` or `renderEditMarkdown` templates to match what its reviewers need.

The Analyzer contract remains the same even when a repository chooses different source metadata or presentation rules.

## Enrich a Pi Session Export

After packaging or loading the `session-review` results as a Brain:

```bash
gsc pi sessions export \
  --format gsc-json \
  --metadata 'read::session-review::read_md' \
  --metadata 'edit::session-review::edit_md'
```
