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

`zproj integrate` installs tmux keybindings for worktree management and asks
your coding agent to wire up your editor with the review workflow.

## Requirements

| Tool | Version | Purpose |
|------|---------|---------|
| bash | 3.2+ | Runtime (bash 4+ preferred) |
| git | 2.5+ | Worktree support |
| tmux | 3.0+ | Named pane support |

One of the following coding agents is auto-discovered (or set `$CODING_AGENT`):
`opencode`, `claude`, `codex`, `amp`, `aider`, `goose`, `gemini`, `agent` (Cursor)

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
cd my-project
zproj                        # opens tmux session with main worktree

# Clone an existing repo
zproj clone git@github.com:you/my-project.git
cd my-project
zproj

# Work on a feature
zproj feature-auth           # creates worktree + branch + tmux window

# Try an experiment (fork from main, work on it, merge back)
zproj fork main experiment   # fork main → experiment at same commit
# ... make changes in experiment, commit ...
zproj join experiment main   # merge experiment → main, delete experiment

# Convert an existing directory or git repo
cd ~/my-existing-project
zproj init .                 # upgrades in place (preserves history)

# Done with a feature
zproj delete feature-auth    # removes worktree, branch, and tmux window
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

## Fork and join

Fork creates a new worktree at the same commit as the source. Join merges a
source worktree's branch into a target and deletes the source on success.

```bash
# Fork from CWD worktree
zproj fork experiment        # CWD → experiment

# Fork from a named source
zproj fork main hotfix       # main → hotfix

# Join back (merge + delete source)
zproj join hotfix main       # hotfix → main

# Join from CWD
zproj join main              # CWD → main
```

Only committed state is forked/joined. Uncommitted changes produce a warning
but don't block the operation. If the source has uncommitted changes at join
time, it is preserved (not deleted) even on a successful merge.

If a join has merge conflicts, the merge is aborted and the diff is sent to
the coding agent in the target worktree with instructions. After resolving
the conflicts and committing, re-run the same `zproj join` command to
complete the join and clean up the source.

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

`zproj integrate` bootstraps a new machine in one step:

1. **Installs tmux keybindings** for worktree management (create, delete,
   fork, join) into your tmux config
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
aider, codex, gemini). Agents without a notification API (amp, goose, agent)
use a `monitor-silence` fallback.

**tmux layer**: installs `alert-bell` and `alert-silence` hooks that call
`zproj notify` with session and window context.

**Terminal layer**: add these to `~/.config/ghostty/config` for rich desktop
notifications (not auto-configured):

```
desktop-notifications = true
bell-features = title,attention,audio
bell-audio-path = /System/Library/Sounds/Glass.aiff
```

tmux 3.3+ delivers full OSC 777 desktop notifications. tmux 3.0–3.2 delivers
bell forwarding only (dock bounce + audio still work via `bell-features`).

Set `$ZPROJ_NOTIFY_COOLDOWN` to override the 5-second per-pane debounce.

## Command reference

```
zproj                                       Open project session (from repo root)
zproj <worktree-dir>                        Create (if needed) and launch
zproj init <dir> [--main <branch>]          Init, convert, or upgrade to bare worktree repo
zproj clone <git-url> [dir]                 Clone remote repo as bare worktree structure
zproj create <worktree-dir> [--from ref]    Create a new worktree
zproj delete <worktree-dir> [--force]       Remove worktree, branch, and window
zproj fork [<source>] <new-worktree>        Fork worktree at same commit
zproj join [<source>] <target>              Merge source into target, delete source
zproj launch <worktree-dir>                 Start or switch to tmux window
zproj list [dir]                            Show worktrees with status
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

363 tests covering init, clone, upgrade, worktree management, fork/join,
review workflow, tmux binding installation, diagnostics, integrate, and tool
detection. Requires tmux, git, and bash in PATH.

## License

MIT — see [LICENSE](LICENSE).
