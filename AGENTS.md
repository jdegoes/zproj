# zproj — Agent Instructions

## What this repository is

`zproj` is a single-file bash script (`zproj`) that manages parallel git
worktrees with tmux sessions. Each worktree gets its own tmux window with a
3-pane layout (coding agent, editor, shell). It also provides a review
workflow for annotating diffs and dispatching notes to a coding agent.

The repository also contains:
- `skills/` — bundled skill files installed by `zproj integrate` into the
  coding agent's global skills directory
- `rules/` — rule files installed by `zproj integrate` into the agent's
  global instructions file (e.g. `~/.config/opencode/AGENTS.md`)
- `README.md` — user-facing documentation

## Commands

```bash
zproj --test       # run the full test suite (must pass before committing)
zproj --version    # print the current version
zproj --help       # list all subcommands
```

## Working with the codebase

The entire implementation lives in the single file `zproj`. It is a bash
script — read and edit it directly. Do not create additional source files.

**Before committing:**
- Edit `zproj` in this directory (not via any symlink)
- Bump `readonly VERSION=` — patch (x.y.Z) for fixes, minor (x.Y.0) for features
- Run `zproj --test` and confirm: `All N tests passed (0 skipped)`

**Do not:**
- Create additional source files — everything lives in `zproj`
- Commit with failing or skipped tests

## Architecture

`zproj` is organized into these layers, top to bottom:

1. **Constants and helpers** — `VERSION`, color vars, `die`/`warn`/`info`, git and tmux utilities
2. **Tool/editor/agent resolution** — `_resolve_editor`, `_resolve_ai`, `_agent_skills_dir`, etc.
3. **Subcommands** — `cmd_<name>` functions, one per subcommand (`init`, `clone`, `create`, `delete`, `launch`, `list`, `review`, `integrate`, `env`, `diagnostics`)
4. **Usage functions** — `usage_<name>`, one per subcommand
5. **Self-test** — `cmd_test()`, containing all tests

Key areas most likely to need changes:
- `cmd_review` — review workflow (`path`, `view`, `dispatch`, `clear`)
- `_integrate_build_prompt` — generates the editor integration prompt sent to the agent; contains the bundled reference Neovim+diffview.nvim implementation as a literal heredoc
- `cmd_test` — all tests; sections named by prefix (e.g. `OL1`, `SK5`, `IN2`)

## Code conventions

- Subcommand functions: `cmd_<subcommand>`
- Helper functions: `_<name>` (underscore prefix)
- Fatal errors: `die`; non-fatal: `warn`; success: `info`
- Tests: `_t_check <desc> <cmd>` (pass if exit 0), `_t_grep <desc> <pattern> <cmd>` (pass if output matches)
- Test section headers: `_section "XX1 — short description"` where `XX` is a 2-letter prefix unique to that section

## Tests

Every new feature or behaviour change must be accompanied by tests in
`cmd_test()` — no exceptions. Add tests in a new or existing `_section` block
using the appropriate prefix. Choose the next available number within the
prefix (e.g. if `OL7` exists, add `OL8`).

## Editor integration sync

The bundled Neovim reference implementation lives as a literal heredoc inside
`_integrate_build_prompt` in `zproj`. It is the source of truth for what
`zproj integrate` hands to a new user's coding agent.

If the review workflow changes (e.g. new subcommands, changed note format,
different key bindings), the heredoc must be updated to reflect the new
workflow so that new users get correct integration instructions.
