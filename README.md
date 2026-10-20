# stackmap-check

[![test](https://github.com/linkaiheng2233-cyber/stackmap-check/actions/workflows/test.yml/badge.svg)](https://github.com/linkaiheng2233-cyber/stackmap-check/actions/workflows/test.yml)

Read-only documentation navigation for coding agents.

Agents often answer "where is this documented?" by reading a lot of files. This tool turns that into two steps: **locate** a few candidate documents, then **read the right section of the original**. It never edits, never starts a service and never touches the network.

```
stackmap query --query <term>          # one line per document, up to 20
stackmap query --query <term> --count  # how many documents match, and what listing them costs
stackmap query --query <term> --all    # every matching document, one line each
stackmap query --query <term> --select entries   # every heading with a short quote
stackmap boot --task <tag>             # curated entry points for a task, when a catalog exists
stackmap check                         # verify curated records against the current commit
```

## Why

Reading a documentation tree is the expensive way to find one paragraph. Measured on a 267-document repository: the whole tree is ~2.9 MB (~900k tokens, impossible in one context); a typical "where is this?" query returns 20 documents in ~1.3k tokens and one second. The agent then reads one or two sections instead of the tree.

It is a locator, not an authority: it tells you **where** to look, not **whether** the text is correct, current or complete.

## Install

No dependencies. Node.js ≥ 22 and `git`.

```sh
git clone <this repo> stackmap
node stackmap/cli.mjs query --query <term> --root /path/to/project
```

Or copy `cli.mjs` into any checkout and run `node cli.mjs ...`. Nothing else is required.

## How it reads a repository

- Scans **git-tracked Markdown** only; untracked notes and other files are ignored.
- Excludes any directory named `archive` and `handoff/debt-marathon/waves/` (historical material).
- Matches literally (case-insensitive) in paths, headings and body text — it is not a semantic search.
- Reads documents from the git index or a commit, never from the working tree, so line endings and unstaged edits cannot corrupt the result. Git blob IDs are the source of identity.
- Reports `staged_change` / `unstaged_change` separately; an editable working tree does not silently change what a query returns.

## Modes and cost

| Mode | Unit | Cost (measured) |
|---|---|---|
| default (`--limit 20`) | documents | 20 documents ≈ 1.3k tokens |
| `--count` | — | a few tokens |
| `--all` | documents | 112 documents ≈ 4.5k tokens |
| `--select entries` | headings + 240-char quote | ~400 chars per entry |

`--limit` counts documents by default; `--select entries` makes it count headings. Truncation prints `Showing 20/N` and the estimated cost of `--all`.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | the listed navigation found no problems |
| `1` | a listed record needs attention (stale reference, working-tree change, unreadable document) |
| `2` | invalid arguments, unreadable catalog, git failure |

Exit codes describe this run, not document correctness.

## Optional curated catalog

A repository may register a few high-traffic documents with `catalog/records.json` (or `handoff/catalog/records.json`, or `--catalog PATH`):

```json
{
  "schema_version": 1,
  "roots": ["docs", "README.md"],
  "records": [
    {
      "id": "guide",
      "path": "docs/guide.md",
      "purpose": "How to install",
      "tags": ["docs"],
      "source_ref": { "scheme": "git-blob", "object_format": "sha1", "oid": "<blob id>" },
      "entries": [
        { "key": "install", "label": "Install", "locator": { "kind": "heading-path", "value": ["Install"] } }
      ]
    }
  ]
}
```

Without a catalog the tool still runs: discovery only, no curated entries.

## Evaluation

`eval.mjs` measures whether the navigator actually finds what a task needs. Cases are repository-owned:

```json
{
  "locate": [{ "q": "build_prompt", "expect": ["docs/prompt.md"] }],
  "coupling": ["module-compat"]
}
```

```sh
node eval.mjs --cases eval.cases.json          # report only
node eval.mjs --cases eval.cases.json --gate   # exit 1 when thresholds fail
```

`locate` reports recall at 8/12/20 documents; `coupling` compares `--all` against a full-text baseline and reports completeness. See `eval.cases.example.json`.

## Limits (read these before trusting it)

- **Literal matching**: synonyms and paraphrases are missed. Use identifiers, command names and IDs.
- **Window**: more than 20 matching documents need `--count` first, then `--all`.
- **Coverage, not meaning**: it cannot find contradictions between documents, implicit conventions or indirect impact. Those need review rules, not search.
- **No relevance ranking**: order is curated first, then path and line. Treat the list as a candidate set.

## Development

```sh
node --test cli.test.mjs
```

The test suite builds throwaway git repositories and checks discovery, quoting, catalog validation, commit isolation, line endings and exit codes.

## License

Apache-2.0. See `LICENSE`.
