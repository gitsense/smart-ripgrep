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

## Optional GitSense Chat UI integration

The local Brain build does not require GitSense Chat. If you also want GitSense Chat to discover the Analyzer definition for its metadata UI, run this from the `smart-ripgrep` repository:

```bash
cd ~/smart-ripgrep
REPO_ROOT="$(git rev-parse --show-toplevel)"
ANALYZER_ID="risk-session-review"
GSC_ANALYZER_DIR="${GSC_HOME:?Set GSC_HOME}/data/analyzers/$ANALYZER_ID"
mkdir -p "$GSC_ANALYZER_DIR"
cp -R "$REPO_ROOT/.gitsense/analyzers/$ANALYZER_ID/." "$GSC_ANALYZER_DIR/"
```

The `/.` source suffix copies the Analyzer contents into the target directory and makes the command safe to rerun without creating a nested Analyzer directory. GitSense Chat discovers the Analyzer from this directory without requiring a restart.

## Required Source Analysis

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
