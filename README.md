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

It walks the tabs of the iTerm2 window you ran it from, in order, and says what
it found in each — so the listing reads like the window in front of you, and you
can see that every tab was looked at:

```
  iTerm2 window 1, 5 tab(s) -> tmux session 'claude': (1 other iTerm2 window(s) not touched)

  MOVE   tab 1    ttys000  pid 39902  -> claude:D4G4
           ~/GitHub/D4G4  People don't realize how crappy their sound is...
  STAGE  tab 2    ttys001  pid 36339  -> claude:dg
           ~  this session
  --     tab 3    ttys014  zsh    no Claude session
  MOVE   tab 4    ttys003  pid 30259  -> claude:First-Ink
           ~/GitHub/D4G4/First-Ink  lets make a new build
  --     tab 5    ttys015  tmux   running tmux — its sessions are already in tmux

  MOVE  (2) SIGTERMed, then relaunched as claude --resume <id>.
        The iTerm2 tab stays open — only the Claude process is stopped.
  STAGE (1) window built, command typed but not entered — you quit the
        original and press Enter. Never killed for you.
```

| Disposition | Meaning |
|---|---|
| `MOVE` | SIGTERMed, then relaunched as `claude --resume <id>` in a tmux window at the same cwd |
| `STAGE` | Window built, resume command typed but **not** entered. Never killed for you |
| `--` | A tab with no Claude session in it — a shell, tmux, an editor. Listed so you can see it was checked |
| `SKIP` | A session that exists but is not safely movable. Each reason is named |

Split panes are listed as `tab 2.1`, `tab 2.2`. **The iTerm2 tab is left open** —
only the Claude process inside it is stopped, so the tab drops back to its shell
prompt with its scrollback intact.

Two whole classes of mistake are impossible by construction rather than by
filtering. A **Claude desktop app** or **VS Code** session runs on a pty that is
not an iTerm2 tab, so it can never be picked up. And a tab running **tmux** holds
its Claude sessions on *pane* ttys rather than the tab's own tty, so sessions
already in tmux are never touched.

`SKIP` is what's left: a session with no transcript on disk to resume from, two
processes sharing one id, or a stale registry entry. A session whose ID cannot be
*proven* is never guessed at and never killed — it is `STAGE`d with a bare
`claude --resume`, which opens Claude Code's own interactive picker.

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
    --system      ignore iTerm2; scan every CLI session on the machine
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

## How a session is identified

The whole chain is four hops, and every hop is a fact rather than a guess:

```
tab  ->  tty  ->  the claude on that tty  ->  ~/.claude/sessions/<pid>.json  ->  id
```

iTerm2 is asked for its tabs over AppleScript, and hands back each tab's `tty`
directly. That tty is the honest link between a tab you can see and a process you
can signal: the session-level `claude` for a tab is the one whose *controlling
terminal is exactly that tty*, excluding any process with a `claude` ancestor
(those are subagents and teammates).

Which window counts as "yours" is resolved by looking for the tab whose tty
matches this process's own; when you run it from inside tmux there is no such tab,
so it falls back to iTerm2's current (frontmost) window.

A trap worth naming: `ITERM_SESSION_ID` looks like it would answer this and does
not. It is inherited, so every process spawned under a tmux server still carries
the ID of whichever tab that server was first launched from — years of tabs later.
The tty is checked instead.

The last hop is the pid-keyed registry Claude Code maintains at
`~/.claude/sessions/<pid>.json`, in which each running session records its own
facts.

```json
{ "pid": 74217, "sessionId": "c90402ec-…", "cwd": "/Users/dg/GitHub/First-Ink",
  "entrypoint": "cli", "kind": "interactive", "tmux": "claude:@6.%6",
  "procStart": "Mon Aug 17 21:00:23 2026" }
```

That turns what used to be inference into a lookup:

1. **`sessionId`** — the session's own ID, not a guess. Nothing is derived from
   the working directory (see below).
2. **`entrypoint`** — `cli`, `claude-desktop`, or `claude-vscode`. Used by the
   `--system` fallback, where there are no tabs to scope to; in iTerm2 mode the
   non-`cli` sessions are already unreachable, since they own no tab.
3. **`tmux`** — set when the session already lives in a pane. Also a fallback
   concern, cross-checked against `tmux list-panes -a -F '#{pane_tty}'`, since a
   pane's shell is parented to the tmux *server* and no ancestry walk can see it.
4. **`procStart`** — compared against the process's real start time (UTC vs
   `ps -o lstart=` local, ±5s). This is what stops a leftover record from a
   crashed session, or a recycled pid, from being resurrected.

Liveness is then confirmed with `ps -o comm=` — the executable path, *not*
`command`, because argv cannot be split on whitespace: the desktop app ships
`claude` under `~/Library/Application Support/…`, so splitting argv on the first
space yields `/Users/you/Library/Application` and the process stops looking like
Claude at all.

Finally, `--resume <id>` is only offered for an ID whose transcript actually
exists on disk, and two live processes claiming the same ID are reported rather
than silently duplicated into two windows of one conversation.

### Why not match on the working directory

Earlier versions recovered the ID by matching a process's `cwd` against the
recorded `cwd` in `~/.claude/projects/*/*.jsonl` and taking the most recently
written. A directory is not unique to a session, so that heuristic fails in
every direction at once. One repo can hold, simultaneously: several live CLI
sessions, a desktop-app session, an IDE session, and the transcripts of sessions
you exited hours ago — all writing into the same project folder. Whichever one
typed last wins the match, so the tool could open a desktop-app conversation you
never asked about, resurrect an exited one, and leave the CLI session you
actually wanted sitting in its bare terminal — the last of those silently, since
a session that lost the match simply vanished from the output.

Every live `claude` process is now accounted for in the listing, including the
skips. A session can be skipped, but it can no longer disappear.

## License

MIT
