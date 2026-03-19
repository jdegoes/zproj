## 2026-03-19
- `cmd_notify()` in tmux `run-shell` hook context should treat all failures as silent no-ops (`return 0`) to avoid invisible stderr/non-zero behavior.
- Broadcast desktop notifications to every attached tmux client by iterating `tmux list-clients -F '#{client_tty}'` and calling `_notify_osc777` per tty.
- Debounce should use `_notify_debounce "$pane" "${ZPROJ_NOTIFY_COOLDOWN:-5}"` so cooldown remains environment-configurable.
