# zproj — Agent Instructions

## Project

Single-file bash script (`zproj`) managing parallel git worktrees with tmux.
Each worktree gets a tmux window with a 3-pane layout (coding agent, editor,
shell). Includes a review workflow for annotating diffs and dispatching notes.

Also in the repo: `skills/` and `rules/` (installed by `zproj integrate`),
`README.md` (user-facing docs).

## Build & Test

```bash
zproj --test       # full test suite — MUST pass before any commit
zproj --version    # current version
zproj --help       # all subcommands
```

Before every commit:
1. Bump `readonly VERSION=` — patch (x.y.Z) for fixes, minor (x.Y.0) for features
2. Run `zproj --test` — done when output says `All N tests passed (0 skipped)`
3. Update `README.md` to reflect any user-facing changes (new commands, changed options, modified behaviour)

## Code Conventions

- Single file `zproj`. Never create additional source files.
- Subcommands: `cmd_<name>`. Helpers: `_<name>`. Usage: `usage_<name>`.
- `die "msg"` for fatal errors (exits 1). `_err "msg"` + `return 1` when caller needs cleanup.
- `warn`, `info` (green checkmark), `dim` for output.
- `set -euo pipefail` — quote all expansions, use `"${var:-}"` for optionals.
- `((count++)) || true` to protect arithmetic from `set -e`.
- `local` for all function variables; declare loop variables before the loop.

## Architecture (top to bottom)

1. **Constants** — `VERSION`, `BARE_DIR`, color vars, `_KNOWN_AGENTS`
2. **Helpers** — `die`, `warn`, `_version_gte`, `_dotglob_expand`, `_git_is_registered_worktree`, `_git_default_branch`
3. **Dependency checks** — `check_git`, `check_session_deps`
4. **Tool resolution** — `_resolve_editor`, `_resolve_ai`, `_agent_skills_dir`, `_agent_instructions_file`
5. **Directory resolution** — `resolve_project_dir`
6. **Init helpers** — `_init_bare_structure`, `_case1_plain_dir`, `_case2_upgrade_repo`
7. **Subcommands** — `cmd_init`, `cmd_clone`, `cmd_create`, `cmd_delete`, `cmd_launch`, `cmd_list`, `cmd_open`, `cmd_default`, `cmd_review`, `cmd_diagnostics`, `cmd_integrate`
8. **Usage functions** — `usage_main`, `usage_init`, etc.
9. **Self-test** — `cmd_test()`
10. **Main dispatch** — `main()` case statement

## Tests

Every feature, behaviour change, or bug fix requires tests. No exceptions.

Helpers: `_t_check "desc" <cmd>` (pass if exit 0), `_t_grep "desc" <pattern> <cmd>` (pass if output matches), `_t_pass`, `_t_fail`, `_t_skip`, `_section "XX1 — desc"`.

Each section uses a unique 1-3 letter prefix: R (init), C (plain dir), U (upgrade), E (editor), A (agent), RP (cursor), P/V/RC/D (review), I (integrate), X (diagnostics), T (tmux/launch), B (regressions), TM (session names), S (safety/meta), N (edge cases), RT (runtime), SK (skills), IN (instructions), RU (rules), OL (--only), TX (tmux config).

To add a test: find the prefix, use the next number (e.g. `T12` exists, add `T13`), place adjacent to related tests. Tests requiring tmux are gated behind `command -v tmux`; tests requiring specific binaries use `_t_skip`.

## Common Tasks

### Adding a subcommand

1. Add `cmd_<name>()` in the subcommands section
2. Add `usage_<name>()` in the usage section
3. Add case in `main()` dispatch
4. Add to `usage_main()` listing
5. Add tests with a new prefix
6. Run `zproj --test`

### Adding a coding agent

1. Append to `_KNOWN_AGENTS` (order = discovery priority)
2. Add case in `_agent_skills_dir()` if it supports skills
3. Add case in `_agent_instructions_file()` if it has global instructions
4. Add tests in SK and IN sections
5. Update `usage_launch` help text

### Fixing a bug

1. Reproduce the bug
2. Write tests that fail — confirm with `zproj --test`
3. Fix the bug
4. Run `zproj --test` — all tests must pass

For true GUI/UI bugs (tmux pane visuals, rendering) where no automated test
is possible, guide the user through manual repro before and verification after.
Most tmux behaviours *can* be tested via `tmux list-panes`, `tmux list-windows`, etc. — prefer automated tests.

### Modifying the review workflow

1. Edit `cmd_review()` for logic changes
2. Update the heredoc in `_integrate_build_prompt()` — source of truth for
   what `zproj integrate` tells the agent to implement in the editor
3. Add tests in P/V/RC/D sections
4. Run `zproj --test`

### Shipping a change ("ship it")

1. Bump `VERSION` if not already bumped
2. Run `zproj --test`, fix failures
3. Update `README.md` for user-facing changes
4. Verify `AGENTS.md` is factually correct
5. `git add && git commit` with clear message
6. `git push` and open PR with good description
7. Fix CI failures, address valid feedback

## Do Not

- Create additional source files
- Commit with failing or skipped tests
- Use `die` inside a function that needs cleanup — use `_err` + `return 1`
- Edit `zproj` via a symlink — edit in the worktree directory
- Add comments restating what code does — only explain *why*
- Modify `skills/` or `rules/` unless asked

## When Stuck

- `zproj --diagnostics` to check the environment
- `zproj --test` to see failures — tests are the best docs for expected behaviour
- Ask the user before architectural changes or adding dependencies
