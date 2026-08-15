# claude2tmux

Move [Claude Code](https://claude.com/claude-code) sessions that you started in a
plain terminal tab into tmux — so you can detach, close the laptop, and pick them
back up over SSH from another machine.

For the times you get three sessions deep and then realize none of them are
reachable from anywhere but that one window.

## The catch, up front

It does not move the running process. **A live process's controlling terminal
cannot be reassigned on macOS** — the Linux tool for that (`reptyr`) works by
`ptrace`-ing the target and swapping its TTY, and there is no working macOS
equivalent.

So `claude2tmux` migrates the *session*, not the process. Claude Code appends
every message to `~/.claude/projects/<slug>/<session-id>.jsonl` as it happens, so
killing a session and running `claude --resume <id>` inside tmux restores the
full conversation, working directory, and history.

What survives: everything already written to the transcript.
What is lost: a tool call that was in flight at the moment of the kill.

If that trade isn't acceptable for a given session, `--no-kill` builds its tmux
window and stages the resume command without touching the original, so you can
quit it yourself when it's idle.

## Install

```sh
git clone https://github.com/daksh-gargas/claude2tmux.git
cd claude2tmux
install -m 755 claude2tmux /usr/local/bin/    # or anywhere on your PATH
```

Optionally add a `c2t` shell function to `~/.zshrc`:

```sh
claude2tmux --install-c2t && exec zsh
```

That writes a marked block, so re-running updates it in place rather than
appending a second copy, it backs up your rc file first, and it refuses to
install if you already define `c2t` somewhere else. Remove it with
`--uninstall-c2t`, which restores the file byte-for-byte.

Requires macOS, tmux, and Python 3 (used to read session transcripts).

## Use

```sh
claude2tmux            # list what would move — the default, changes nothing
claude2tmux --go       # do it
```

The listing states a disposition per session, so what actually moves is never
something you have to infer:

```
  Sessions that will move into tmux session 'claude':

  MOVE   ttys000  pid 39902  -> claude:D4G4
         ~/GitHub/D4G4  People don't realize how crappy their sound is...
  STAGE  ttys001  pid 36339  -> claude:dg
         ~  give me a flag which lists the sessions that will be moved
  MOVE   ttys002  pid 30842  -> claude:Wildr
         ~/GitHub/Wildr  Theme set to auto
  MOVE   ttys003  pid 30259  -> claude:First-Ink
         ~/GitHub/D4G4/First-Ink  lets make a new build
  MOVE   ttys005  pid 57289  -> claude:First-Ink-2
         ~/GitHub/D4G4/First-Ink  commit your work to main

  MOVE  (4) SIGTERMed, then relaunched as claude --resume <id>.
  STAGE (1) window built, command typed but not entered — you quit the
        original and press Enter. Never killed for you.
```

| Disposition | Meaning |
|---|---|
| `MOVE` | SIGTERMed, then relaunched as `claude --resume <id>` in a tmux window at the same cwd |
| `STAGE` | Window built, resume command typed but **not** entered. Never killed for you |
| `SKIP` | Already inside a tmux pane, or has no controlling terminal. Counted, not listed — nothing to do |

A session with no controlling TTY is not a terminal session at all: the VS Code
or JetBrains extension host, a `claude -p` run, a hook. There is nothing to
migrate, and SIGTERMing one would kill a session you cannot see in any tab — so
they are excluded before anything else happens.

The session you run the command *from* is always `STAGE`, never `MOVE` — killing
it would abort the migration mid-run. Quit that tab yourself, switch to its
window, press Enter.

## Flags

```
-l, --list        list every session and its disposition; changes nothing
    --plain       with --list: tab-separated for scripting
                  DISPOSITION <TAB> tty <TAB> pid <TAB> sid <TAB> cwd
    --go          perform the migration
    --no-kill     with --go: build the windows, leave the originals running
-s, --session N   target tmux session name (default: claude)
    --install-c2t     add a `c2t` shell function to ~/.zshrc
    --uninstall-c2t   remove it again
-h, --help        usage
```

Bare `claude2tmux` is identical to `--list`, and `--list` overrides `--go` —
asking to see the plan never executes it. An unrecognized flag exits 2 rather
than being ignored, since a typo next to `--go` would otherwise be a bad
surprise.

Env equivalents: `CLAUDE2TMUX_SESSION` (for `-s`), `CLAUDE_BIN` (path to the
`claude` binary), `CLAUDE2TMUX_RC` (rc file for `--install-c2t`).

## How a session is detected

A candidate has to survive four filters. Three are structural facts; the fourth
is a heuristic, and it's worth knowing which is which.

1. **`argv[0]` basename is exactly `claude`.** Not a substring match on the
   command line — a running Claude session surrounds itself with shell wrappers
   whose command lines *contain* "claude" (snapshot sourcing, `/tmp/claude-*-cwd`
   redirects), and grepping would sweep all of them in. This uses shell
   parameter expansion rather than `basename`, because login shells are named
   `-zsh` and `basename` parses that as a flag.

2. **No `claude` ancestor** (ppid walk, 25 levels). Subagents and teammates are
   always spawned under a parent Claude, so this leaves only top-level sessions.
   It also identifies "self": the nearest `claude` ancestor of `$$`.

3. **Not on a tmux pane TTY.** This cannot be inferred from ancestry — a pane's
   shell is parented to the tmux *server*, never to a Claude process — so it's
   checked against `tmux list-panes -a -F '#{pane_tty}'` directly. Without it,
   sessions already safely in tmux get killed and rebuilt for no reason.

4. **A cwd and a session ID both resolve.** cwd via `lsof` (macOS has no
   `/proc`). Session ID from `--resume <uuid>` in argv when present — definitive,
   since resuming appends to the same transcript rather than forking a new one.

Filter 4's fallback is the soft spot: for a session started *without* `--resume`,
the ID is recovered by matching the recorded `cwd` across transcripts and taking
the most recently written. If you ran two sessions in one directory and killed
one, the survivor could in principle be handed the dead one's ID. In practice the
live session is always the most recently written — but it is inference, not a
fact read off the process. `--list` shows you the resolved ID and the last user
message of each session, so you can check before committing.

## License

MIT
