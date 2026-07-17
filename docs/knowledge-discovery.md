# Knowledge Discovery and Topics

Lessons, notes, and rules are organized around topics such as `binary-detection`, `walk-parallelism`, and `file-type-matching`. Every knowledge record has one primary topic and may have related topics.

## Find Knowledge

Search notes, lessons, and rules together when you do not know which type contains the answer:

```bash
gsc knowledge search "binary detection"
```

Browse a known topic:

```bash
gsc knowledge list --topic binary-detection
```

Or limit the search to one kind of record:

```bash
gsc knowledge search "walk" --type lessons
```

## Browse Topics

```bash
gsc topics list
gsc topics show binary-detection
gsc topics search binary
```

## Migrate Older Records

If the repository has lessons created before topics were required, preview and apply the migration:

```bash
gsc topics migrate --dry-run
gsc topics migrate
```
