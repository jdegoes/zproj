---
name: ck
description: "Use ck to find anything in source code: identifiers, types, patterns, concepts. Replaces grep, rg, and ripgrep — never use those on source files. Triggers: find, search, locate, where is, usages, references, grep, rg, ripgrep."
license: MIT
compatibility: opencode
metadata:
  category: search
  replaces: grep, rg, Glob
---

# ck — Semantic Code Search

## Tool Override

Use `ck` instead of `grep`, `rg`, or Glob for code search on source files. `ck`
understands meaning — searching "error handling" finds try/catch blocks, error
returns, and exception handling even if those exact words don't appear.

**Critical exception:** `.ckignore` excludes files from ALL search modes — not
just indexing. Regex, semantic, lexical, and hybrid searches all silently skip
excluded files. The default `.ckignore` (auto-created on first `--index`) excludes
`.json`, `.yaml`, `.yml`, and other non-source file types. If you need to search
excluded files (config files, data files, lockfiles, etc.), use `grep` or `rg`
directly — `ck` will never find them. When in doubt, check `.ckignore` first.

## When to Use ck vs cqs

| Task | Tool |
|---|---|
| Find code by concept, pattern, or meaning | `ck --sem` |
| Find exact identifiers or regex patterns | `ck` (default) |
| Search config, JSON, YAML, lockfiles | `grep` or `rg` (excluded by `.ckignore`) |
| Who calls this function? | `cqs callers` |
| What breaks if I change this? | `cqs impact` |
| Refactoring safety check | `cqs impact` + `cqs callers` |
| Understand a whole subsystem | `cqs gather` or `cqs onboard` |
| Find dead code | `cqs dead` |

## Search Modes

| Mode | Flag | Use for |
|---|---|---|
| Semantic | `--sem` | Concepts, behavior, intent — "retry logic", "input validation" |
| Lexical | `--lex` | Ranked full-text — requires a prior `--index` run; unreliable without it |
| Hybrid | `--hybrid` | Balance precision and recall — "JWT token validation" |
| Regex | _(default)_ | Exact identifiers and patterns — `fn authenticate`, `class.*Handler` |

**Quoting:** Use single quotes for regex patterns containing `$` or `[`. In double quotes, `$letter` and `$[...]` are expanded by the shell before `ck` sees them, silently corrupting the pattern.

```bash
ck 'Scope\.$'   # correct — $ preserved
ck "Scope\.$"   # wrong   — $ may be expanded by the shell
```

**`--lex` caveat:** BM25 lexical search requires embeddings to be present. Always run
`ck --index .` before using `--lex`. Auto-indexing on first `--sem` does not guarantee
the BM25 index is ready for `--lex`.

## Index Management

`ck` auto-indexes on first `--sem` search. Run `--index` explicitly to ensure
embeddings are present before using `--lex` or `--hybrid`. Indexing costs ~10s
per 100K LOC; searches are <500ms once indexed.

On first index, `ck` auto-creates a `.ckignore` with default exclusions (including
`.json` and `.yaml`). Check this file if expected files aren't being found.

**Gitignore:** `ck` does not auto-create a `.gitignore` inside `.ck/`. Always run
`echo "*" > .ck/.gitignore` after first indexing or after copying from a sibling
worktree, or index files will appear as untracked in `git status`.

```bash
ck --index .            # Build index with embeddings (required for --sem, --lex, --hybrid)
ck --status .           # Check index status (shows file count and embedded chunk count)
ck --status-verbose .   # Detailed index statistics
ck --clean-orphans .    # Remove stale entries only
ck --clean .            # Remove entire index
```

## Git Worktrees (zproj)

Projects use a bare repo structure where worktrees are siblings in a parent directory:

```
project-name/          ← bare repo root (contains .bare/)
  main/                ← main worktree  (has .ck/ index)
  feature-foo/         ← feature worktree (new, no index yet)
  feature-bar/         ← another worktree
```

**Never cold-index a new worktree.** Both `ck` and `cqs` use relative paths and
content-hash caching (not mtime). Copying an index from a sibling worktree is safe
and reduces indexing time from minutes to under a second.

**Workflow for a new worktree:**

```bash
# 1. Find the most recently indexed sibling
ls -dt ../*/. | head -5                          # list siblings by recency

# 2. Copy its index into the new worktree
cp -r ../main/.ck ./.ck

# 3. Delta-index — only changed files are re-embedded (typically <1s)
ck --index .
```

`ck` re-indexes in ~0.2s on a copied worktree even when all file mtimes differ,
because it detects identical content by hash and reuses cached embeddings.
Only files that actually changed on the feature branch get re-processed.

**After copying, create the gitignore if not already present:**
```bash
echo "*" > .ck/.gitignore
```

## Recommended Workflow

### Stage 1: Discover with semantic search

```bash
ck --sem "authentication" .
ck --sem "error handling" src/
ck --sem "database connection" .
```

### Stage 2: Refine with hybrid or regex once you have identifiers

```bash
# Found `authenticateUser` in stage 1 — now find all usages
ck --hybrid "authenticateUser" .
ck "fn authenticateUser" src/
```

## Common Invocations

```bash
# Semantic — find by concept
ck --sem "error handling" .
ck --sem "database connection" src/

# Raise precision (fewer, better results)
ck --sem --threshold 0.8 "authentication logic" .

# Lower precision (broader sweep)
ck --sem --threshold 0.5 "retry" .

# Limit result count
ck --sem --limit 10 "caching strategy" .

# Show relevance scores (score appears on its own line before each result block)
ck --sem --scores "retry logic" .

# Rerank results for better relevance ordering
ck --sem --rerank "authentication logic" .

# Specify reranking model (jina or bge, default: jina)
ck --sem --rerank --rerank-model bge "authentication logic" .

# Hybrid — regex + semantic combined (use threshold 0.01–0.05 for RRF scores)
ck --hybrid "async function" .
ck --hybrid --limit 10 --threshold 0.02 "error" .

# Regex — exact identifiers (default mode, no index needed)
ck "fn authenticate" src/
ck -r "AuthController" .

# JSONL output — preferred for agent consumption
# Fields: path, span (byte_start, byte_end, line_start, line_end), language, snippet, score
ck --jsonl --sem "error handling" .
ck --jsonl --sem --limit 10 --threshold 0.7 "auth" .

# Metadata-only JSONL (no snippet — smaller, faster)
# Fields: path, span, language, score
ck --jsonl --no-snippet --sem "error handling" .

# JSON output — single array, fields differ from JSONL: uses "file", "preview", "lang"
ck --json --sem "error handling" .
```

## Adaptive Threshold

Default threshold for `--sem` is `0.6`. Start at `0.7` for focused results. Adjust
based on output:
- Too few results → lower to `0.5`
- Too many results → raise to `0.8`
- Never go above `0.9` (too restrictive) or below `0.3` (too noisy)

**Note:** `--hybrid` uses RRF scores, not cosine similarity. The useful range for
`--threshold` with `--hybrid` is `0.01–0.05`, not `0.0–1.0`.

When no results are found, `ck` shows the nearest match beneath the threshold:

```
No matches found

(nearest match beneath the threshold)
[0.687] src/auth.rs:1:fn authenticate(user: &str, pass: &str) -> Result<(), String> {
```

Use the score shown to decide whether to retry with a lower threshold.

## Output Formats

- **Default** — grep-style, human-readable
- **`--scores`** — adds `[0.950]` score on its own line before each result block
- **`--jsonl`** — one JSON object per line; fields: `path`, `span`, `language`, `snippet`, `score`
- **`--jsonl --no-snippet`** — same without `snippet`; fields: `path`, `span`, `language`, `score`
- **`--json`** — single JSON array; fields: `file`, `span`, `lang`, `preview`, `score`, `signals`

## Do / Don't

**Do:**
- Use single quotes for regex patterns containing `$` or `[`
- After first indexing a project, run `echo "*" > .ck/.gitignore` to prevent index files polluting git
- Run `ck --index .` at the start of a session before using `--sem`, `--lex`, or `--hybrid`
- Use `--sem` for conceptual or behavioral searches
- Use default regex mode for exact identifiers (no index needed)
- Use `--jsonl --limit 20` when results will be processed programmatically
- Use `--no-snippet` when you only need file paths and scores
- Use `--rerank` when result ordering matters
- Start semantic, then refine with hybrid or regex once you have identifiers
- Use `--threshold 0.01–0.05` with `--hybrid`, not the semantic range
- Check `.ckignore` whenever a file you expect isn't showing up in results
- Reach for `cqs` when you need call graph navigation, impact analysis, or refactoring safety

**Don't:**
- Use double quotes for patterns with `$` or `[` — the shell expands them silently before `ck` receives the pattern
- Use `grep`, `rg`, or Glob for source code search — but DO use them for excluded file types
- Use `--threshold` above `0.9` or below `0.3` for semantic search
- Apply semantic threshold values (0.0–1.0) to hybrid searches — hybrid uses RRF (0.01–0.05)
- Re-index unnecessarily — `ck` handles incremental updates automatically
- Use `--lex` without a prior `--index` run — results will be empty or unreliable
- Assume `.ckignore` only affects indexing — it silently excludes files from all search modes
