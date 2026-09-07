---
name: smux
description: Control tmux panes and communicate between AI agents. Use this skill whenever the user mentions tmux panes, cross-pane communication, sending messages to other agents, reading other panes, managing tmux sessions, or interacting with processes running in tmux. Includes tmux-bridge CLI for agent-to-agent messaging and raw tmux commands for direct session control.
metadata:
  { "openclaw": { "emoji": "🖥️", "os": ["darwin", "linux"], "requires": { "bins": ["tmux", "tmux-bridge"] } } }
---

# smux

Tmux pane control and cross-pane agent communication. Use `tmux-bridge` (the high-level CLI) for all cross-pane interactions. Fall back to raw tmux commands only when you need low-level control.

## tmux-bridge — Cross-Pane Communication

A CLI that lets any AI agent interact with any other tmux pane. Works via plain bash. Use `send` for complete agent messages: it frames, types, and normally submits the message. Use `type`, `message`, and `keys` only when staged interaction or the documented submit recovery is required.

### Agent messages: classify before sending

Before touching the target, classify the message:

- **Urgent steer:** a correction that changes the active task, a blocker,
  approval request, safety issue, or an explicit request to interrupt now.
- **Routine report:** progress, completion/build/test details, checksums,
  acknowledgments, or anything that can wait until the active turn finishes.

Read the target first. If it is idle with an empty composer, use this mandatory
delivery protocol. Run each command as a separate tool/shell invocation so the
verification result is actually inspected:

```bash
tmux-bridge read <target> 20
tmux-bridge send <target> 'message text'
sleep 0.5
tmux-bridge read <target> 20
```

`read` preserves the read-before-act safety check. `send` uses bracketed paste,
waits for the target pane to stop redrawing, and then presses Enter. The
length-aware wait is inside `send`; the external 0.5-second wait is only for
post-submit verification. Always perform the single verification below.

Do not use `message` for a complete agent message. It is a draft-only command that deliberately does not press Enter and prints a warning.

### BUSY TARGETS: ROUTINE REPORTS WAIT; URGENT CORRECTIONS STEER

Do **not** inject a routine report while the target shows `Working`. Enter or
Escape can intentionally steer the active turn and surface `Conversation
interrupted`; Tab can submit the report at a later tool boundary and abort the
rest of that turn. For a routine report:

1. Do not call `send`, `message`, or `keys`, and do not queue it with Tab.
2. Wait until a fresh `read` shows the target is idle and its composer is
   empty, then use read → send → wait → verify.
3. Short readiness rechecks every 2–5 seconds are allowed here. They only
   determine when delivery is safe; they are not polling for a reply.
4. Prefix pure completion reports with `[REPORT — no reply needed]` when an
   acknowledgment would add no value. Receivers must not create acknowledgment
   ping-pong.

An urgent steer may be sent while the target is `Working`. If verification
proves that this exact outbound message remains in its active composer, press
**Enter once**. If the exact message was placed under `Messages to be submitted
after next tool call` and the TUI offers `press esc to interrupt and send
immediately`, press **Escape once**. The interruption is intentional only for
this urgent lane.

Never press Tab unless the caller explicitly asks for deferred/queued delivery.
For normal routine reports, waiting for an idle target is safer than queueing
because it prevents both mid-turn interruption and stale late reports.

### PROTECT AN EXISTING COMPOSER BEFORE SENDING

The pre-send `read` is also a composer safety check. If the target already has
typed text in its active input composer, stop. Do not call `send`, do not append
the new message, and do not press Enter or Tab: those actions could submit the
user's unfinished draft. Ask the user to submit/cancel it, or wait for the
composer to become empty.

When scrollback is ambiguous, inspect the active cursor row; it is stronger
evidence than a matching message elsewhere in the transcript:

```bash
row=$(tmux display-message -t <target> -p '#{cursor_y}')
tmux capture-pane -t <target> -p -J -S "$row" -E "$row"
```

Only proceed when that active row is an empty/fresh prompt. For a label, resolve
it to a pane ID first with `tmux-bridge resolve <label>`.

### DELIVERY IS NOT COMPLETE UNTIL VERIFIED

The stdout line `sent and submitted to ...` only proves that keys were sent to
tmux. It does not prove that the target TUI processed Enter. Never finish the
turn or report that a message was sent based on that line alone.

After `send`, wait 0.5 seconds and perform exactly one verification `read`.
This is a delivery check, not reply polling:

```bash
tmux-bridge read codex 20
tmux-bridge send codex 'Please review src/auth.ts'
sleep 0.5
tmux-bridge read codex 20
```

Classify the verification output in this strict priority order. Higher-priority
message-location evidence always overrides a generic `Working` indicator.
These recovery branches apply after an idle-target send, or after an explicitly
urgent send to a busy target:

1. **This exact message was queued:** the target shows the exact outbound
   message under `Messages to be submitted after next tool call`.
   - For an urgent steer, if the banner offers `press esc to interrupt and
     send immediately`, press **Escape once** to promote it.
   - If deferred delivery was explicitly requested, leave it queued.
   - A routine report should not have been sent while the target was busy.
     Do not promote it and do not add another copy.
   - Do not press Escape for a generic banner or an unknown/pre-existing queued
     message.
2. **Submitted:** the exact framed message is in the transcript and the active
   cursor row is a fresh/empty composer. Stop.
3. **Still in the active composer:** the active cursor row contains the tail of
   the exact outbound message. This means it is not submitted even if the pane
   also says `Working`.
   - Press **Enter once** if the target is idle or the message is an urgent
     steer. If the target unexpectedly became busy and this is a routine
     report, stop rather than interrupting it.
   - Do **not** press Tab merely because the TUI says `tab to queue message`.
     Use Tab only when the caller explicitly requested deferred delivery.
   - Do this only when the pre-send composer was empty and the active composer
     still contains your exact message. Otherwise stop to protect a user draft.
4. **Only generic activity is visible:** `Working`, tool output, or a status
   spinner without states 1–3 is inconclusive. Do not claim delivery and do not
   press a key until the active composer is identified.

The verification `read` satisfies the read guard. Apply at most one corrective
key selected from the proven state, wait 0.5 seconds, and verify once more:

```bash
# Choose exactly one proven-state branch:
# Exact message is queued and the TUI offers immediate interruption:
tmux-bridge keys codex Escape
# OR exact message is still in the busy or idle composer:
# tmux-bridge keys codex Enter
sleep 0.5
tmux-bridge read codex 20
```

If the corrective verification still shows the exact message in the composer
or queue, report the delivery failure instead of claiming success or pressing
another key. Never press Enter, Escape, or Tab when the message has moved into
the transcript or the composer/queue contains pre-existing or unknown text.

Other panes reply by injecting a `[tmux-bridge from:... pane:%N ...]` message
into your pane. A normal assistant response in your own pane does not reach the
sender. For every such inbound message that requires a reply, use the `pane:%N`
target and complete this same read → send → wait → verify protocol before
finishing your turn.

The ONLY time you read a target pane is:
- **Before** interacting with it (enforced by the read guard)
- During short **idle-readiness checks** while holding a routine report for a
  busy target
- **Once after `send`** to verify submission, plus once after one corrective Enter/Escape if needed
- **After typing** to verify your text landed before pressing Enter
- When interacting with a **non-agent pane** (plain shell, running process)

### Read Guard

The CLI enforces read-before-act. You cannot `send`, `type`, or `keys` to a pane unless you have read it first.

1. `tmux-bridge read <target>` marks the pane as "read"
2. `tmux-bridge send/type/keys <target>` checks for that mark — errors if you haven't read
3. After a successful `send`/`type`/`keys`, the mark is cleared — you must read again before the next interaction

```
$ tmux-bridge type codex "hello"
error: must read the pane before interacting. Run: tmux-bridge read codex
```

### Command Reference

| Command | Description | Example |
|---|---|---|
| `tmux-bridge list` | Show all panes with target, pid, command, size, label | `tmux-bridge list` |
| `tmux-bridge type <target> <text>` | Type text without pressing Enter | `tmux-bridge type codex "hello"` |
| `tmux-bridge send <target> <text>` | Frame, type, and submit an agent message (preferred) | `tmux-bridge send codex "review src/auth.ts"` |
| `tmux-bridge message <target> <text>` | Draft a framed message without Enter | `tmux-bridge message codex "draft only"` |
| `tmux-bridge read <target> [lines]` | Read last N lines (default 50) | `tmux-bridge read codex 100` |
| `tmux-bridge keys <target> <key>...` | Send special keys | `tmux-bridge keys codex Enter` |
| `tmux-bridge name <target> <label>` | Label a pane (visible in tmux border) | `tmux-bridge name %3 codex` |
| `tmux-bridge resolve <label>` | Print pane target for a label | `tmux-bridge resolve codex` |
| `tmux-bridge id` | Print this pane's ID | `tmux-bridge id` |

### Target Resolution

Targets can be:
- **tmux native**: `session:window.pane` (e.g. `shared:0.1`), pane ID (`%3`), or window index (`0`)
- **label**: Any string set via `tmux-bridge name` — resolved automatically

### Read-Act Cycle

Every delivered agent message follows **read → send → wait 0.5s → verify**.
Before that cycle, hold routine reports until the target is idle; only urgent
steers may enter a busy target. The CLI enforces read-before-act. Treat `send`
as the preferred submit action, but verify the terminal UI state once.

**Sending a message to an agent:**
```bash
tmux-bridge read codex 20                    # 1. READ — inspect and satisfy read guard
tmux-bridge send codex 'Please review src/auth.ts'
                                              # 2. SEND — frame, type, and submit
sleep 0.5                                    # 3. LET THE TUI RENDER SUBMISSION
tmux-bridge read codex 20                    # 4. VERIFY — classify delivery state
# If this urgent message is queued with an immediate-send hint: Escape once.
# If still in composer: Enter once when idle, or when it is an urgent steer.
# Never Tab unless deferred delivery was explicitly requested.
# STOP. Do NOT read codex again for its eventual reply.
```

**Approving a prompt (non-agent pane):**
```bash
tmux-bridge read worker 10                   # 1. READ — see the prompt
tmux-bridge type worker "y"                  # 2. TYPE
tmux-bridge read worker 10                   # 3. READ — verify
tmux-bridge keys worker Enter                # 4. KEYS — submit
tmux-bridge read worker 20                   # 5. READ — see the result
```

### Messaging Convention

The `send` and `message` commands auto-prepend sender info and location:

```
[tmux-bridge from:claude pane:%4 at:3:0.0] Please review src/auth.ts
```

The receiver gets: who sent it (`from`), the exact pane to reply to (`pane`),
and the session/window location (`at`). When the message requires a reply, use
tmux-bridge with the pane ID from the header. Do not reply to a
`[REPORT — no reply needed]` completion report.

### Agent-to-Agent Workflow

```bash
# 1. Label yourself
tmux-bridge name "$(tmux-bridge id)" claude

# 2. Discover other panes
tmux-bridge list

# 3. Send, then verify that the TUI actually submitted it
tmux-bridge read codex 20
tmux-bridge send codex 'Please review the changes in src/auth.ts'
sleep 0.5
tmux-bridge read codex 20
```

### Example Conversation

**Agent A (claude) sends:**
```bash
tmux-bridge read codex 20
tmux-bridge send codex 'What is the test coverage for src/auth.ts?'
sleep 0.5
tmux-bridge read codex 20
```

**Agent B (codex) sees in their prompt:**
```
[tmux-bridge from:claude pane:%4 at:3:0.0] What is the test coverage for src/auth.ts?
```

**Agent B replies using the pane ID from the header:**
```bash
tmux-bridge read %4 20
tmux-bridge send %4 '87% line coverage. Missing the OAuth refresh token path (lines 142-168).'
sleep 0.5
tmux-bridge read %4 20
```

---

## Raw tmux Commands

Use these when you need direct tmux control beyond what tmux-bridge provides — session management, window navigation, creating panes, or low-level scripting.

### Capture Output

```bash
tmux capture-pane -t shared -p | tail -20    # Last 20 lines
tmux capture-pane -t shared -p -S -          # Entire scrollback
tmux capture-pane -t shared:0.0 -p           # Specific pane
```

### Send Keys

```bash
tmux send-keys -t shared -l -- "text here"   # Type text (literal mode)
tmux send-keys -t shared Enter               # Press Enter
tmux send-keys -t shared Escape              # Press Escape
tmux send-keys -t shared C-c                 # Ctrl+C
tmux send-keys -t shared C-d                 # Ctrl+D (EOF)
```

For interactive TUIs, split text and Enter into separate sends:
```bash
tmux send-keys -t shared -l -- "Please apply the patch"
sleep 0.1
tmux send-keys -t shared Enter
```

### Panes and Windows

```bash
# Create panes (prefer over new windows)
tmux split-window -h -t SESSION              # Horizontal split
tmux split-window -v -t SESSION              # Vertical split
tmux select-layout -t SESSION tiled          # Re-balance

# Navigate
tmux select-window -t shared:0
tmux select-pane -t shared:0.1
tmux list-windows -t shared
```

### Session Management

```bash
tmux list-sessions
tmux new-session -d -s newsession
tmux kill-session -t sessionname
tmux rename-session -t old new
```

### Claude Code Patterns

```bash
# Check if session needs input
tmux capture-pane -t worker-3 -p | tail -10 | grep -E "❯|Yes.*No|proceed|permission"

# Approve a prompt
tmux send-keys -t worker-3 'y' Enter

# Check all sessions
for s in shared worker-2 worker-3 worker-4; do
  echo "=== $s ==="
  tmux capture-pane -t $s -p 2>/dev/null | tail -5
done
```

## Tips

- **Delivery is read → send → wait 0.5s → verify** — never omit the final verification
- **Classify first** — routine reports wait for idle; urgent corrections may steer a busy target
- **Do not queue routine reports** — Tab can still interrupt at a later tool boundary
- **Do not trust `sent and submitted` alone** — it reports key delivery, not TUI acceptance
- **Never append to a non-empty composer** — protect user drafts before sending
- **Exact active-composer evidence overrides `Working`** — `Working` alone is inconclusive
- **Promote an exact queued message with Escape only for an urgent steer**
- **Use Enter on a busy target only for an urgent steer**
- **Never use Tab by default** — queue only when deferred delivery was explicitly requested
- **Reply to inbound bridge messages through their `pane:%N` target** — a local final response is not a cross-pane reply
- **Do not acknowledge `[REPORT — no reply needed]`** — avoid cross-pane ping-pong
- **Apply at most one corrective key, then verify once** — never loop or double-submit
- **Read guard is enforced** — you MUST read before every `send`/`type`/`keys`
- **Every action clears the read mark** — after `type`, read again before `keys`
- **Never poll for replies** — short reads are allowed only to wait for a busy target to become safe for routine-report delivery
- **Label panes early** — easier than using `%N` IDs
- **`type` uses literal mode** — special characters are typed as-is
- **`read` defaults to 50 lines** — pass a higher number for more context
- **Non-agent panes** are the exception — you DO need to read them to see output
- Use `capture-pane -p` to print to stdout (essential for scripting)
- Target format: `session:window.pane` (e.g., `shared:0.0`)
