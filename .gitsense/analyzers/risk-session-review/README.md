# Risk Session Review Analyzer

`risk-session-review` stores deterministic change-risk review items for file-read, targeted-edit, and complete-file-write events in a coding-agent session.

This smart-ripgrep version is tuned for change review: it combines file purpose, agent triage, Rust blast radius, test-coverage metadata, and repository lessons so a reviewer can quickly see what a read or edit brought into play. Each item has a group, stable key, short title, concise authored review summary, and complete Markdown detail, allowing a session metadata index to group matching concerns without interpreting prose.

The Analyzer is a data contract. Its values are populated by [build-risk-session-review](../../bin/build-risk-session-review), which combines existing local Brains without making another LLM call.

## Compose a Brain from Existing Brains

This Analyzer demonstrates a deterministic composition pattern. Existing Brains provide focused repository knowledge—file purpose, triage, dependency risk, test coverage, and lessons. The builder joins that knowledge by file path, turns it into structured review items, and creates a new `risk-session-review` Brain.

The derived Brain can then enrich coding-agent sessions with repository context at the moment a file is read, edited, or written. AI-generated metadata and deterministic analysis can be combined in the same review surface while remaining independently inspectable.

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
  "short_markdown": "Review the impact of this high-risk change before continuing.",
  "long_markdown": "Downstream tools parse this output.",
  "markdown": "Downstream tools parse this output."
}
```

`short_markdown` is authored for high-information-density views and should quickly tell a reviewer what to check. It must not be produced by mechanically truncating the full detail. `long_markdown` is the authoritative explanation and should retain lesson details, review checks, and supporting context for deeper investigation. Existing consumers can continue reading `markdown`, which is kept as a compatibility alias for `long_markdown`.

The two representations let the same repository context be useful at different
levels of attention:

- `short_markdown` keeps a list of matching items scannable when many files or
  metadata groups are present.
- `long_markdown` preserves the complete reasoning, lesson content, and review
  checks when a reviewer needs to understand why the item matters.

The metadata describes repository context that applies to the file. It does not
prove that the agent saw, understood, or followed that context. Reviewers should
use the matching item as a prompt for what to compare against the agent's work.

## Source Brains

Before running the builder, create the local source Brains from the committed manifests:

```bash
for manifest in \
  code-intent \
  agent-file-triage \
  rust-blast-radius \
  rust-test-coverage-intent
do
  gsc manifest import ".gitsense/manifests/$manifest.json"
done
```

Run this setup on a fresh checkout. If a Brain already exists, inspect it with `gsc brains --summary`; `gsc manifest import` will not overwrite it unless `--force` is supplied.

The deterministic builder reads these local Brains:

* `code-intent`
* `agent-file-triage`
* `rust-blast-radius`
* `rust-test-coverage-intent`

The builder also reads repository-local `.gitsense/lessons/records.jsonl`. Lesson paths are matched deterministically against both `applies_to.files` and `applies_to.linked_files`. A missing lesson file is valid and produces no repository-memory items.

The source repository does not need to be imported into GitSense Chat for the local build. Missing metadata for an individual file is allowed; the corresponding Markdown section is omitted. The source Brain databases are the only analysis input required.

## Build and Review

Build the local Brain and generate reviewable JSONL:

```bash
.gitsense/bin/build-risk-session-review
```

The builder reads the local Brains, writes reviewable JSONL to `/tmp/risk-session-review.jsonl`, generates `.gitsense/manifests/risk-session-review.json`, and creates or refreshes the local `.gitsense/risk-session-review.db` Brain. GitSense Chat is not required.

The generated manifest is deterministic for the same source metadata apart from its `generated_at` timestamp. The local Brain is refreshed only when the manifest changes.

## Adapt the Pattern

The review items are produced by deterministic functions in `.gitsense/bin/build-risk-session-review`. Another repository can keep the same Analyzer shape, then change the source Analyzer list and the `buildReadItems`, `buildEditItems`, or `buildWriteItems` mappings to match what its reviewers need.

The Analyzer contract remains the same even when a repository chooses different source metadata or presentation rules.

## Enrich a Pi Session Export

After the builder creates the local `risk-session-review` Brain:

```bash
gsc pi sessions export \
  --format gsc-json \
  --metadata 'read::risk-session-review::read_items' \
  --metadata 'edit::risk-session-review::edit_items' \
  --metadata 'write::risk-session-review::write_items'
```

`write_items` tells the reviewer that the agent supplied complete file contents. That may mean a new file or a replacement of an existing one, so the Analyzer does not guess which happened without separate session or filesystem evidence.
