# zproj

```
███████╗██████╗ ██████╗  ██████╗      ██╗
╚══███╔╝██╔══██╗██╔══██╗██╔═══██╗     ██║
  ███╔╝ ██████╔╝██████╔╝██║   ██║     ██║
 ███╔╝  ██╔═══╝ ██╔══██╗██║   ██║██   ██║
███████╗██║     ██║  ██║╚██████╔╝╚█████╔╝
╚══════╝╚═╝     ╚═╝  ╚═╝ ╚═════╝  ╚════╝
```

**Parallel workspaces with bare git worktrees and tmux.**

Manage parallel feature branches as first-class workspaces. Each branch gets
its own directory and a tmux window with a 3-pane layout — coding agent,
editor, and shell — all opened in the right place automatically. Worktrees,
branches, and windows are created and torn down together as a unit.

A built-in review workflow lets you annotate lines during a diff review and
dispatch the collected notes to your coding agent for implementation.

Run `zproj menu` when you want a context-aware command picker instead of
remembering subcommands.

`zproj integrate` installs tmux keybindings for worktree management and asks
your coding agent to wire up your editor with the review workflow.

## Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| bash | 3.2+ | Runtime (bash 4+ preferred) |
| git | 2.5+ | Worktree support |
| tmux | 3.0+ | Named pane support |
| [Lima](https://github.com/lima-vm/lima) | (any recent) | Optional: `limactl` on `PATH` for `--sandbox` (guest coding agent in a Linux VM) |

One of the following coding agents is auto-discovered (or set `$CODING_AGENT`):
`opencode`, `claude`, `codex`, `amp`, `aider`, `goose`, `gemini`, `pi`, `hermes`, `agent` (Cursor)

> **Cursor CLI Agent:** In repositories containing `.cursor/`, `.cursorrules`,
> or `.cursorignore`, the Cursor CLI agent (`agent`) is automatically preferred
> over the default discovery order.

Editor resolution (first match wins):
`$ZPROJ_EDITOR` → `$EDITOR` → `nvim` → `vim`

## Installation

```bash
# Clone the repo
git clone https://github.com/jdegoes/zproj ~/zproj

# Symlink the script into your PATH
ln -s ~/zproj/zproj ~/.local/bin/zproj

# Verify
zproj --version
```

## Quick start

```bash
# Start a new project
zproj init my-project
zproj my-project             # opens tmux session with main worktree
# (or: cd my-project; zproj)

# Clone an existing repo
zproj clone git@github.com:you/my-project.git
cd my-project
zproj

# Work on a feature
zproj feature-auth           # creates worktree + branch + tmux window

# Try an experiment (fork from main, work on it, merge back)
zproj fork main experiment   # fork main → experiment at same commit
# ... make changes in experiment, commit ...
# Pull main's latest commits into experiment as you go:
cd experiment && zproj update    # merge latest main into experiment
zproj join experiment main   # merge experiment → main, delete experiment

# Convert an existing directory or git repo
cd ~/my-existing-project
zproj init .                 # upgrades in place (preserves history)

# Done with a feature
zproj delete feature-auth    # removes worktree, branch, and tmux window

# Not sure what to run next?
zproj menu                   # status-aware command menu with exact commands
```

## How it works

### Directory structure

`zproj init` or `zproj clone` creates a bare repo with one worktree per branch:

```
my-project/
  .bare/              bare git repository
  .git                pointer: "gitdir: ./.bare"
  main/               worktree for the main branch
  feature-auth/       worktree for feature-auth branch
  fix-bug/            worktree for fix-bug branch
```

Each worktree directory is a fully independent working tree. You can have
multiple branches open simultaneously without stashing or switching.

> **Invariant**: each worktree directory holds the branch of the same name
> (`feature-auth/` is on branch `feature-auth`, `main/` is on branch `main`).
> zproj enforces this on `create`/`fork`. If you manually `git checkout` a
> different branch inside a worktree, `zproj launch`, `fork`, `join`, and
> `update` will warn — operations like `update` resolve the source by branch
> name and will silently target the wrong branch otherwise.

### Tmux layout

One tmux **session** per project, one **window** per worktree. Each window has
three panes, all opened in the worktree directory:

```
┌─────────────────┬──────────────────┐
│                 │                  │
│  coding agent   │     editor       │
│  (pane 1)       │     (pane 2)     │
│                 ├──────────────────┤
│                 │                  │
│                 │     shell        │
│                 │     (pane 3)     │
└─────────────────┴──────────────────┘
```

Panes are named after the tool they run (e.g. `opencode`, `nvim`, `shell`)
so dispatch commands always reach the right pane regardless of which window
is active.

### Tmux keybindings

`zproj integrate` installs these keybindings into your tmux config:

| Binding | Action |
|---------|--------|
| `<prefix> N` | Create a new worktree (prompts for name) |
| `<prefix> X` | Delete the current worktree (with confirmation) |
| `<prefix> F` | Fork the current worktree (prompts for new name) |
| `<prefix> J` | Join the current worktree into another (prompts for target) |
| `<prefix> U` | Update the current worktree from its forked source (with confirmation) |

All bindings work from any pane — they derive the project root automatically.

### Tool resolution

```bash
zproj --env          # show which editor and agent will be used
zproj --diagnostics  # check the full environment for problems
```

| Env var | Purpose |
|---------|---------|
| `CODING_AGENT` | Override coding agent (e.g. `export CODING_AGENT=claude`) |
| `ZPROJ_EDITOR` | Override editor (e.g. `export ZPROJ_EDITOR=nvim`) |

## Interactive menu

`zproj menu [dir]` opens a terminal menu that checks where you are:
outside zproj, at a project root, or inside a worktree. It shows current
status, suggests useful next steps, and displays the exact command before
asking whether to run it.

```bash
zproj menu                 # menu for current directory
zproj menu ~/repo/main     # menu for a specific project/worktree path
```

Optional fish shortcut:

```fish
function zp
    zproj menu $argv
end
```

## Lima sandbox (optional)

With [Lima](https://lima-vm.github.io/) installed (`limactl` on your `PATH`), you can run the **coding agent** in a persistent Linux VM while the **editor** stays on the host. The project directory is mounted at `/mnt/zproj` in the guest (see `$ZPROJ_LIMA_MOUNT`).

**Canonical argv (v1):** put global sandbox flags **before** the subcommand.
`--sandbox` and `--plan` may appear in either order:

```bash
zproj --sandbox --plan launch main    # dry-run: show instance name and mounts; no VM/tmux
zproj --plan --sandbox launch main    # equivalent — order does not matter
zproj --sandbox launch main           # create/start VM if needed, open tmux layout with guest agent pane
```

`zproj integrate` already has `--plan` for a **host-side** editor-integration dry run; that is unchanged. Combining **`--sandbox`** with **`integrate`** is not supported in v1 — use `zproj integrate --plan` on the host.

Handoff prompts (review dispatch, merge-conflict context) are written under `<project-root>/.zproj/` **only** when `--sandbox` is active, so the Lima guest can read them via the bind mount. Without `--sandbox`, prompts go to `$TMPDIR` as before and never appear in `git status`.

Unset `ZPROJ_DISABLE_LIMA` if you set it for diagnostics and expect `limactl` to be discovered.

## Fork, update, and join

Fork creates a new worktree at the same commit as the source. Update pulls
the source's latest commits into a fork as it progresses. Join merges a
source worktree's branch into a target and deletes the source on success.

```bash
# Fork from CWD worktree
zproj fork experiment        # CWD → experiment

# Fork from a named source
zproj fork main hotfix       # main → hotfix

# Pull latest source commits into a fork
cd experiment && zproj update           # CWD ← recorded source (e.g. main)
zproj update main                       # CWD ← main (also self-records)
zproj update experiment main            # experiment ← main (from anywhere)

# Join back (merge + delete source)
zproj join hotfix main       # hotfix → main

# Join from CWD
zproj join main              # CWD → main
```

Only committed state is forked/joined/updated. Uncommitted changes produce a
warning but don't block the operation. If the source has uncommitted changes
at join time, it is preserved (not deleted) even on a successful merge.

If a `join` or `update` hits merge conflicts, the merge is aborted and the
diff is sent to the coding agent in the target worktree with instructions.
After resolving the conflicts and committing, the merge is complete (for
`update`) or you re-run the same `zproj join` to clean up the source.

`zproj fork` records the source branch in `branch.<n>.zprojSource` so that
subsequent `zproj update` calls (or `<prefix> U`) need no arguments. For
worktrees forked before this feature existed, the source is auto-detected
from local-branch topology (the branch with the most-recent merge-base with
your fork; the bare repo's default branch wins on ties) and recorded on
first successful update. Disambiguate auto-detection failures by passing the
source explicitly: `zproj update <source>`.

Fork/join identity: `zproj fork old new && zproj join new old` is a no-op.

## Review workflow

During development, annotate lines of interest while reviewing a diff. Notes
accumulate in `.agents/plans/review-notes.md` inside the worktree. When ready,
dispatch all notes to the coding agent in one step.

```bash
zproj review path      # print the notes file path
zproj review view      # display accumulated notes
zproj review dispatch  # send notes to coding agent, delete file
zproj review clear     # discard notes
```

The dispatch command assembles a prompt from the notes, saves it to a temp
file, and sends a one-line handoff message to the coding agent pane via tmux.
The notes file is deleted after the temp file is safely written.

## Machine integration

`zproj integrate` bootstraps a new machine in one step. **Re-running it is safe:**
worktree keybindings and desktop-notification hooks are rewritten from a clean
slate (no duplicate `bind` lines or stacked `set-hook` entries in `tmux.conf`).

1. **Installs tmux keybindings** for worktree management (create, delete,
   fork, join, update) into your tmux config
2. **Asks your coding agent** to implement the review workflow in your editor
   (add note / view / dispatch / clear), using the Neovim reference
   implementation as a concrete example

```bash
zproj integrate              # full bootstrap
zproj integrate --plan       # dry run: show what would be done
```

## Notifications

`zproj integrate` configures a three-layer notification pipeline so that when
any coding agent finishes working in a tmux pane, you get a desktop
notification with worktree context.

**Agent layer**: configures each installed agent to emit BEL on idle (claude,
aider, codex, gemini). Agents without a notification API (amp, goose, hermes,
agent) use a `monitor-silence` fallback.

**tmux layer**: installs `alert-bell` and `alert-silence` hooks that call
`zproj notify`. Installs use **`set-hook -g`** (replace one handler) so each
`tmux source-file …` does **not** multiply copies. **`zproj integrate`** also
rewrites any older **`-ga`** notifier lines in your tmux conf to **`-g`** so
reloads stop stacking handlers. If your **server** already has a long stacked
chain from previous reloads, reset once, then reload:

```bash
tmux set-hook -ug alert-bell
tmux set-hook -ug alert-silence
tmux source-file ~/.config/tmux/tmux.conf   # or ~/.tmux.conf
```

**Terminal layer**: add these to `~/.config/ghostty/config` for rich desktop
notifications (not auto-configured):

```
desktop-notifications = true
bell-features = title,attention,audio
bell-audio-path = /System/Library/Sounds/Glass.aiff
```

tmux 3.3+ delivers full OSC 777 desktop notifications. tmux 3.0–3.2 delivers
bell forwarding only (dock bounce + audio still work via `bell-features`).
If a session or window name contains a semicolon, `zproj notify` rewrites it before
emitting OSC 777 so the terminal does not mis-parse the sequence (which would
otherwise show as junk in apps like Neovim until you redraw).

Set `$ZPROJ_NOTIFY_COOLDOWN` to override the 5-second per-pane debounce.

## Command reference

```
zproj                                       Open project session (from repo root)
zproj <worktree-dir>                        Create (if needed) and launch
zproj <project-root>                        Open project session (shortcut; from parent)
zproj init <dir> [--main <branch>]          Init, convert, or upgrade to bare worktree repo
zproj clone <git-url> [dir]                 Clone remote repo as bare worktree structure
zproj create <worktree-dir> [--from ref]    Create a new worktree
zproj delete <worktree-dir> [--force]       Remove worktree, branch, and window
zproj fork [<source>] <new-worktree>        Fork worktree at same commit
zproj update [<target>] [<source>]          Merge forked source branch into target
zproj join [<source>] <target>              Merge source into target, delete source
zproj launch <worktree-dir>                 Start or switch to tmux window
zproj --sandbox [--plan] launch <worktree-dir>
                                            Guest coding agent via Lima (project at /mnt/zproj)
zproj list [dir]                            Show worktrees with status
zproj menu [dir]                            Context-aware interactive command menu
zproj review <subcommand>                   Manage review notes (path/view/dispatch/clear)
zproj integrate [--plan]                    tmux bindings + editor integration
zproj notify [session] [window] [pane]      Emit desktop notification (used by tmux hooks)
zproj --env                                 Show resolved editor and coding agent
zproj --diagnostics                         Check environment for problems
zproj --test                                Run the built-in test suite
```

Run `zproj <command> --help` for details on any command.

## Self-test

```bash
zproj --test
```

437 tests pass in the default suite. Some checks are skipped when optional
tools or agent binaries are absent; the one slow Lima VM test (`LM5`) is
skipped unless you set `ZPROJ_TEST_LIMA_VM=1` and have `limactl` installed.
Coverage includes init, clone, upgrade, worktree management, fork/update/join,
interactive menu, review workflow, tmux binding installation, diagnostics,
integrate, notify, Lima sandbox flags, and tool detection.
Requires tmux, git, and bash in `PATH`. The suite uses a dedicated tmux server
socket (`ZPROJ_TEST_TMUX_SOCKET`) so it does not attach to or mutate your
normal tmux sessions.

## License

MIT — see [LICENSE](LICENSE).
