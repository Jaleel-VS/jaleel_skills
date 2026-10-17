---
name: tern-panes
description: "Use to drive Tern panes like tmux from a shell: open tabs/splits, send keys or text, run a command and wait for it to finish, read a pane's screen, close it. For agents running inside a Tern pane on the Mac or a dev desk."
---

# Driving Tern panes like tmux

The `tern` CLI scripts the window you run it from. Verified on Tern 0.9.0 (macOS), 2026-10-17.

## Which window and which pane

- Commands act on the window of the pane they run in (`$TERN_WINDOW_KEY`); `--window KEY` picks another one.
- **Remote hosts:** `tern ls` from a Mac pane lists only that window's local sessions, not dev-desk ones. To drive panes on a dev desk, run `tern` inside a Tern pane on that host. This wasn't tested from the Mac side.
- `BLOCK` = numeric id (most reliable), `@focused`, a tab title, or a program name (`omp`); `SESSION/BLOCK` narrows it to one session.
- **Never send keys to `@focused` blindly.** That's usually the user's own pane, or yours. Create a pane, keep its id, and drive that id.

```sh
tern ls                 # sessions › tabs › blocks: id, title, cwd, size
tern inspect --json     # every pane and client
tern whoami             # the pane this shell runs in
```

## Open

```sh
B=$(tern split @focused down --keep-open --json | python3 -c 'import json,sys; print(json.load(sys.stdin)["block"])')
tern new tab --cwd /path --keep-open --json        # a tab in the shown session (JSON has "block")
tern new session NAME --cwd /path                  # a new session
tern split $B right --keep-open -- htop            # a pane running a program
```

- `--keep-open`: without it, a pane closes when its program succeeds, and you can't read its output after.
- `-- COMMAND…` runs a program instead of a shell.

## Send

```sh
tern run  $B 'make test'            # types the command + Enter
tern send $B text 'partial input'   # types, no Enter
tern send $B keys Enter             # keys: Enter, Escape, Tab, Up, C-c, ...
tern send $B paste "$BIG"           # bracketed paste (multi-line safe)
```

## Wait

| Situation | Use | Notes |
|---|---|---|
| The pane runs **one program** (`split … -- cmd`) | `tern wait $B --until exit --timeout 600` | exits with the **program's exit status**; 124 = timed out |
| A **command typed into a shell** | the end marker below | reliable; also gives the exit status |
| Output that should simply stop changing | `--until idle=MS` | ⚠️ "idle" = no output, so a silent command (`sleep 3`) counts as idle at once |
| `--until prompt` | **avoid** | in testing it timed out after 20 s although the command had finished |

End marker for shell commands:

```sh
tern run $B 'make test; echo "__done:$?__"'
S=$(date +%s)
until tern capture $B | grep -qE '^__done:[0-9]+__'; do
  sleep 0.5; (( $(date +%s) - S > 600 )) && { echo timeout; break; }
done
rc=$(tern capture $B | grep -oE '^__done:[0-9]+__' | tail -1 | tr -dc 0-9)
```

`^` matches the marker only at the start of a line, so the echoed command line doesn't count. This assumes your prompt prints something before the command, like `❯ `.

`tern process $B --json` → `.foreground.name` is the program in front: the shell (`zsh`) when idle, otherwise the running command. Useful as a second check.

## Read

```sh
tern capture $B                  # visible screen as text
tern capture $B --scrollback     # include scrollback
tern capture $B --ansi           # with colors (raw)
tern events --filter KIND        # JSON lines until interrupted (live events)
```

Strip blank lines before reading: `tern capture $B | grep -v '^\s*$' | tail -20`.

## Arrange and clean up

```sh
tern rename $B "build"               # tab title
tern move $B right-of OTHER          # or: tab N | new-tab
tern focus $B                        # show it in every window. Don't steal the user's focus unless asked
tern keep-open $B on|off
tern pip / tern dock                 # floating picture-in-picture pane / back into the layout
tern close $B                        # close a pane you created
tern kill session NAME               # whole session
```

Always close the panes you opened when done. Leave the user's own panes alone.

## Compared with tmux

`split-window` → `split`, `new-window` → `new tab`, `send-keys` → `send … keys`, `capture-pane` → `capture`, `list-panes` → `ls`/`inspect`, `kill-pane` → `close`. tmux has no `tern wait --until exit` (an exit status is passed on).
