---
name: cqs
description: "Use cqs for call graph analysis: callers, callees, impact of changing a function, dead functions, refactoring safety, diff risk. Requires a callable function name — does not find text, types, or identifiers. Triggers: who calls this, what breaks if I change, impact analysis, callers, callees, dead code, refactor function, rename function, review diff."
license: MIT
compatibility: opencode
metadata:
  category: code-intelligence
  complements: ck
---

# cqs — Code Intelligence CLI

## Setup (required before first use in any project)

```bash
cqs init        # Download ML model (~547MB, one-time per machine)
cqs index       # Index the codebase (builds embeddings + call graph + HNSW)
```

Indexing is slow on large codebases (5–15 min for 500K LOC). Run `cqs watch` afterward
to keep the index fresh automatically. Check status with `cqs -q stats --json`.

**Re-index when:** new files were added outside of `watch`, or call graph shows 0 entries.

## Git Worktrees (zproj)

Projects use a bare repo structure where worktrees are siblings in a parent directory:

```
project-name/          ← bare repo root (contains .bare/)
  main/                ← main worktree  (has .cqs/ index)
  feature-foo/         ← feature worktree (new, no index yet)
  feature-bar/         ← another worktree
```

**Never cold-index a new worktree.** `cqs` stores relative paths and uses blake3
content hashing — not mtime. Copying an index from a sibling worktree is safe and
reduces indexing from ~15 minutes to ~27 seconds, because all unchanged files are
recognised as cached regardless of their checkout timestamps. Only files that
actually differ on the feature branch get re-embedded.

**Workflow for a new worktree:**

```bash
# 1. Find the most recently indexed sibling
ls -dt ../*/. | head -5                          # list siblings by recency

# 2. Copy its index into the new worktree
cp -r ../main/.cqs ./.cqs

# 3. Delta-index — only changed files are re-embedded (~27s even if all mtimes differ)
cqs index
```

**What happens during delta index on a copied index (verified):**
- `cqs` scans all files, computes blake3 hashes
- Files with identical hashes → embeddings reused from copied index (0 re-embedded)
- Files changed on the feature branch → re-embedded only for those files
- Call graph, type graph, and HNSW rebuilt from the updated chunk set

A typical feature branch touching 10–20 files completes in well under a minute.

## When to Use cqs vs ck

| Task | Tool |
|---|---|
| Find code by concept or pattern | `ck` or `cqs` |
| Who calls this function? | `cqs callers` |
| What breaks if I change this? | `cqs impact` |
| Is it safe to rename/move/delete this? | `cqs impact` + `cqs callers` |
| Find dead code | `cqs dead` |
| Understand a whole subsystem | `cqs gather` or `cqs onboard` |
| Prepare an implementation brief | `cqs task` or `cqs scout` |
| Review a diff before committing | `cqs review` |
| Search config/JSON/YAML files | `ck` is excluded by `.ckignore` — use `grep`/`rg` |

## Critical Syntax Rules

**`-q` is a global flag — it must come before the subcommand:**
```
cqs -q <command> <args> --json     ✓
cqs <command> <args> --json -q     ✗ (error: unexpected argument)
```

**`notes add/remove/update` do not accept `-q` at all.**

**`trace`, `impact`, `review`, `ci` use `--format json`, not `--json`:**
```
cqs impact "encode" --format json     ✓
cqs -q impact "encode" --json         ✗
```

## Recommended Workflows

### New task / feature implementation
```
cqs -q scout "<task description>" --json        # 1. Orient: search + callers + tests + notes
cqs -q task "<task description>" --json         # 2. Full brief: scout + gather + impact + placement
```

### Understanding unfamiliar code
```
cqs -q onboard "<concept>" --json               # Entry point → call chain → types → tests
cqs -q gather "<concept>" --json                # Everything related, call-graph expanded
cqs explain "<function>" --json                 # Single function deep dive
```

### Before refactoring (rename / move / change signature)
```
cqs -q callers "<function>" --json              # Who calls it?
cqs impact "<function>" --format json           # What breaks transitively?
cqs -q deps "<type>" --json                     # What uses this type?
cqs trace "<fn-a>" "<fn-b>" --format json       # Is there a call path between them?
```

### Code review / pre-commit
```
cqs review --format json                        # Diff impact + risk scoring
cqs ci --format json                            # Same + gate (exit 3 on high risk)
```

### Codebase health
```
cqs -q dead --json                              # Find dead code
cqs -q health --json                            # Full snapshot
cqs -q stale --json                             # What needs re-indexing?
```

## Do / Don't

**Do:**
- Use single quotes for queries containing `$` — in double quotes, `$letter` is silently expanded by the shell, changing the query without any error
- Run `cqs init` + `cqs index` at the start of every new project
- Start with `scout` or `task` before implementing — they assemble context in one call
- Use `impact` before any refactor — even "small" changes can have transitive callers
- Use `gather` with `--tokens N` to stay within context window on large codebases
- Use `--format json` for `trace`, `impact`, `review`, and `ci`
- Use `-q` before the subcommand, never after
- Use `cqs watch` in the background during long coding sessions to keep the index fresh

**Don't:**
- Skip `cqs index` — without it, call graph and type graph will be empty
- Use `cqs` for searching `.json`, `.yaml`, or config files — those are excluded; use `grep`/`rg`
- Use `--json` with `trace` or `impact` — it will error; use `--format json`
- Append `-q` after the subcommand for most commands — it will error
- Run `cqs index` repeatedly without need — it's expensive; `watch` handles incremental updates

---

## Search & Discovery

### Semantic search — `cqs -q "<query>" [flags] --json`

**There is no `search` subcommand.** The query is a bare positional argument
passed directly to `cqs`. Do not add `search` before the query — it will fail
with "unexpected argument".

```bash
cqs -q "error handling" --json          # correct
cqs -q search "error handling" --json   # WRONG — search is not a subcommand
```

Response: `{ query, results: [{chunk_type, content, file, has_parent, language, line_start, line_end, name, score, signature, type}], total }`

| Flag | Description |
|------|-------------|
| `--lang <L>` | Filter by language (rust, python, typescript, javascript, go, c, java, sql, markdown, scala) |
| `-n/--limit <N>` | Max results (default 5, max 20) |
| `-t/--threshold <N>` | Similarity threshold (default 0.3) |
| `--name-only` | Definition lookup, skips embedding |
| `--semantic-only` | Pure vector similarity, no hybrid RRF |
| `--rerank` | Cross-encoder re-ranking (slower, more accurate) |
| `-p/--path <glob>` | Path pattern filter (e.g., `src/cli/**`) |
| `--chunk-type <T>` | Filter: function, method, class, struct, enum, trait, interface, constant, section |
| `--pattern <P>` | Pattern: builder, error_swallow, async, mutex, unsafe, recursion |
| `--note-only` | Return only notes |
| `--note-weight <F>` | Note score weight 0.0-1.0 (default 1.0) |
| `--ref <name>` | Search only this reference index |
| `--tokens <N>` | Token budget (greedy knapsack by score) |
| `--expand` | Include parent context (boolean flag, no value) |
| `-C/--context <N>` | Lines of context around chunk |
| `--no-content` | Omit code content from display output (field still present in JSON) |
| `--no-stale-check` | Skip per-file staleness checks |

### similar `<function>` — Find similar code
Find code similar to a given function. Refactoring discovery, duplicates.

```
cqs -q similar "<name>" --json
```

Response: `{ target, results: [...], total }`

### gather `<query>` — Smart context assembly
Seed search + BFS call graph expansion. "Show me everything related to X."

```
cqs -q gather "<query>" [flags] --json
```

Response: `{ query, chunks: [...], expansion_capped, search_degraded }`

| Flag | Description |
|------|-------------|
| `--expand <N>` | BFS depth (default 1, max 5) |
| `--direction <D>` | both, callers, callees (default both) |
| `-n/--limit <N>` | Max results (default 10) |
| `--tokens <N>` | Token budget |
| `--ref <name>` | Cross-index: seed from reference, bridge into project |

### where `<description>` — Placement suggestion
Where to add new code based on semantic similarity and local patterns.

```
cqs -q where "<description>" --json
```

Response: `{ description, suggestions: [...] }`

### scout `<task>` — Pre-investigation dashboard
Search + callers/tests + staleness + notes in one call. First step for new tasks.

```
cqs -q scout "<task>" [-n N] [--tokens N] --json
```

Response: `{ summary, file_groups: [...], relevant_notes: [...] }`

### task `<description>` — Implementation brief
Scout + gather + impact + placement + notes. Waterfall token budgeting (scout 15%, code 50%, impact 15%, placement 10%, notes 10%).

```
cqs -q task "<description>" [-n N] [--tokens N] --json
```

Response: `{ description, summary, scout, code, risk, placement, tests, token_budget, token_count }`

### onboard `<concept>` — Guided codebase tour
Entry point + call chain + callers + types + tests. Ordered reading list.

```
cqs -q onboard "<concept>" [-d/--depth N] [--tokens N] --json
```

Response: `{ concept, summary, entry_point, call_chain, callers, key_types, tests }`

### read `<path>` — File with contextual notes
File contents with notes injected as comments. Richer than raw Read.

```
cqs read <path> [--focus <function>] --json
```

Response: `{ path, content }` or `{ focus, content }` with `--focus`

`--focus <function>` returns only the target function + type dependencies.

### context `<file>` — Module overview
Chunks, callers, callees, notes for a file.

```
cqs context <path> [--compact] [--summary] --json
```

Response: `{ file, chunks, dependent_files, external_callers, external_callees, ... }`
`--compact` response: `{ file, chunks, chunk_count }`

### explain `<function>` — Function card
Signature, docs, callers, callees, similar in one call.

```
cqs explain "<name>" --json
```

Response: `{ name, chunk_type, file, language, lines, signature, doc, callers, callees, similar }`

---

## Call Graph

### callers `<function>` — Who calls this?
Returns a JSON **array** of caller objects.

```
cqs -q callers "<name>" --json
```

Response: `[{ name, file, line }, ...]`

### callees `<function>` — What does this call?
```
cqs -q callees "<name>" --json
```

Response: `{ function, count, calls: [{ callee, file, line }, ...] }`

### trace `<source>` `<target>` — Shortest call path
BFS shortest path between two functions. Uses `--format json`, not `--json`.

```
cqs trace "<source>" "<target>" --format json
```

Response: `{ source, target, path, message }`

### deps `<name>` — Type dependencies
Forward (default): who uses this type? Returns a JSON **array**.
Reverse (`--reverse`): what types does this function use? Returns a dict.

```
cqs -q deps "<name>" --json                  # forward: array of { name, chunk_type, file, line_start }
cqs -q deps --reverse "<name>" --json        # reverse: { function, count, types: [...] }
```

### related `<function>` — Co-occurrence
Functions sharing callers, callees, or types.

```
cqs -q related "<name>" --json
```

Response: `{ target, shared_callers, shared_callees, shared_types }`

### impact `<function>` — What breaks if you change X?
Uses `--format json`, not `--json`.

```
cqs impact "<name>" [--depth N] [--suggest-tests] [--include-types] --format json
```

Response: `{ function, caller_count, callers, test_count, tests, type_impacted, type_impacted_count }`

### impact-diff — Diff-aware impact analysis
Changed functions, affected callers, tests to re-run.

```
cqs impact-diff [--base <ref>] [--stdin] [--json] [--tokens N]
```

Response: `{ changed_functions, callers, tests, summary }`

### test-map `<function>` — Map function to tests
Tests that exercise a function via reverse call graph.

```
cqs -q test-map "<name>" --json
```

Response: `{ function, test_count, tests: [...] }`

### blame `<function>` — Semantic git blame
Who changed a function, when, and why.

```
cqs blame "<name>" [--callers]
```

---

## Quality & Review

### dead — Find dead code
Functions/methods with no callers.

```
cqs -q dead [--include-pub] [--min-confidence low|medium|high] --json
```

Response: `{ dead: [...], possibly_dead_pub: [...], total_dead, total_possibly_dead_pub }`

### stale — Index freshness
Files modified since last index.

```
cqs -q stale --json
```

Response: `{ stale: [...], stale_count, missing: [...], missing_count, total_indexed }`

### health — Codebase quality snapshot
Stats + dead code + staleness + hotspots + untested hotspots + notes.

```
cqs -q health --json
```

Response: `{ stats, dead_confident, dead_possible, stale_count, missing_count, hotspots, untested_hotspots, note_count, note_warnings, warnings, hnsw_vectors }`

### ci — CI pipeline analysis
Review + dead code + gate logic. Exit 3 on gate fail. Uses `--format json`.

```
cqs ci [--base <ref>] [--stdin] [--gate high|medium|off] [--format json] [--tokens N]
```

### review — Comprehensive diff review
Impact + notes + risk scoring + staleness. Uses `--format json`.

```
cqs review [--base <ref>] [--stdin] [--format json] [--tokens N]
```

Response: `{ changed_functions, affected_callers, affected_tests, relevant_notes, risk_summary, stale_warning }`

### suggest — Auto-suggest notes
Scan for dead code clusters, untested hotspots, high-risk functions. Returns a JSON array.

```
cqs -q suggest [--apply] --json
```

### gc — Report staleness
Report/clean stale index entries.

```
cqs -q gc --json
```

Response: `{ stale_files, missing_files, pruned_chunks, pruned_calls, pruned_type_edges, hnsw_rebuilt, hnsw_vectors }`

### stats — Index statistics
Chunk counts, languages, last update.

```
cqs -q stats --json
```

Response: `{ total_chunks, total_files, by_language, by_type, model, schema_version, created_at, call_graph, type_graph, hnsw_vectors, notes, stale_files, missing_files }`

---

## Notes

Notes do not accept `-q`. Omit it entirely for notes commands.

### notes add `<text>` — Add a note
```
cqs notes add "<text>" [--sentiment N] [--mentions a,b,c]
```

Sentiment: -1 (serious pain), -0.5 (notable pain), 0 (neutral), 0.5 (notable gain), 1 (major win).

### notes list — List all notes
```
cqs notes list [--json] [--warnings] [--patterns]
```

Response: array of `{ id, text, sentiment, mentions, type }`

### notes update `<exact text>` — Update a note
```
cqs notes update "<exact text>" [--new-text "..."] [--new-sentiment N] [--new-mentions a,b,c]
```

### notes remove `<exact text>` — Remove a note
```
cqs notes remove "<exact text>"
```

### audit-mode — Toggle audit mode
Excludes notes from search/read for unbiased review.

```
cqs audit-mode [on|off] [--expires 30m] --json
```

---

## Infrastructure

### ref — Reference index management
Manage external codebases for multi-index search.

| Subcommand | Usage |
|------------|-------|
| `ref add <name> <path> [--weight 0.8]` | Index external codebase (weight 0.0-1.0) |
| `ref list` | Show all references |
| `ref update <name>` | Re-index reference |
| `ref remove <name>` | Delete reference |

### watch — File watcher
Keep index fresh automatically.

```
cqs watch [--debounce <ms>] [--no-ignore]
```

### convert `<path>` — Document conversion
PDF/HTML/CHM/MD to cleaned Markdown.

```
cqs convert <path> [-o/--output <dir>] [--overwrite] [--dry-run] [--clean-tags <tags>]
```

Tags: `aveva`, `pdf`, `generic`, `all` (default).

### diff `<source>` — Semantic diff
Compare indexed snapshots. Requires a reference via `ref add` first.

```
cqs -q diff "<source>" [target] [--threshold F] [--lang L] --json
```

### drift `<reference>` — Semantic drift detection
Embedding distance between same-named functions across snapshots. Requires a reference.

```
cqs -q drift "<ref>" [--min-drift 0.1] [--lang L] [--limit N] --json
```

### chat — Interactive REPL
```
cqs chat
```

### batch — Batch mode (JSONL pipeline)
Read commands from stdin, output JSONL. Supports pipeline syntax: `search "error" | callers | test-map`

```
cqs batch
```
