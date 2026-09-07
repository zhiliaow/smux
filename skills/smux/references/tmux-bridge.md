---
name: tmux-bridge
description: Agent-agnostic CLI for cross-pane communication — type text, send keys, read output, and interact between tmux panes.
metadata:
  { "openclaw": { "emoji": "🌉", "os": ["darwin", "linux"], "requires": { "bins": ["tmux", "tmux-bridge"] } } }
---

# tmux-bridge

A single CLI that lets any AI agent (Claude Code, Codex, Gemini CLI, etc.) interact with any other tmux pane. Works via plain bash — any tool that can run shell commands can use it.

Use the atomic `send` command for agent messages. After the required read, `send` frames the message, types it, and presses Enter in one command. `type`, `message`, and `keys` remain available for intentionally staged interactions.

## CLASSIFY, DELIVER SAFELY, VERIFY ONCE

**Other panes have agents that will reply to you via tmux-bridge.** When you send a message to another agent, their reply will appear directly in YOUR pane as a `[tmux-bridge from:...]` message. You do NOT need to:

- Poll the target pane for a response
- Read the target pane to check if they replied
- Loop or retry to see output

You MUST, however, wait 0.5 seconds and read the target exactly once after
`send` to verify that the target TUI processed Enter. This delivery
verification is not reply polling. Do not trust the `sent and submitted`
stdout line by itself.

`send` itself uses bracketed paste and waits for the pane to stop redrawing
before it presses Enter. This prevents long UTF-8 or multiline messages from
still being processed when the submit key arrives. Do not add a guessed
pre-submit sleep; the command owns that timing.

Classify every message before sending:

- **Urgent steer:** a correction that changes active work, a blocker, approval
  request, safety issue, or an explicit immediate interruption.
- **Routine report:** progress/completion details, test output, hashes, or an
  acknowledgment.

If the target shows `Working`, hold routine reports until a fresh read shows it
is idle with an empty composer. Do not type them, do not press Tab, and do not
put them into `Messages to be submitted after next tool call`: queued input can
still interrupt the target at the next tool boundary. Short readiness checks
every 2–5 seconds are allowed; they are not reply polling. Once idle, use the
normal read → send → wait → verify cycle.

Only urgent steers may be injected into a busy target. If an urgent message
remains in the active composer, press Enter once. If it is the exact message
shown in a queued banner offering `press esc to interrupt and send
immediately`, press Escape once. The resulting interruption is intentional.

Before `send`, the required read must also confirm that the active input
composer is empty. Never append a bridge message to existing text and never
press Enter/Tab on a user draft. If scrollback is ambiguous, inspect the
target's active cursor row with `#{cursor_y}` plus `capture-pane`; active
composer evidence is stronger than matching text in transcript history.

The ONLY time you need to read a target pane is:
- **Before** interacting with it (enforced — see Read Guard below)
- During short **idle-readiness checks** while holding a routine report
- **Once after `send`** to verify submission, plus once after one corrective Enter/Escape if needed
- **After typing** to verify your text landed correctly before pressing Enter
- When interacting with a **non-agent pane** (plain shell, running process) where there's no agent to reply back

## Read Guard — Enforced by CLI

The CLI **enforces** read-before-act. You cannot `send`, `type`, or `keys` to a pane unless you have read it first.

**How it works:**
1. `tmux-bridge read <target>` marks the pane as "read"
2. `tmux-bridge send/type/keys <target>` checks for that mark — **errors if you haven't read**
3. After a successful `send`/`type`/`keys`, the mark is **cleared** — you must read again before the next interaction

This enforces read-before-act at the CLI level. If you skip the read, the command fails:

```
$ tmux-bridge type codex "hello"
error: must read the pane before interacting. Run: tmux-bridge read codex
```

## When to Use

**USE this skill when:**

- Sending messages to another agent running in a tmux pane
- Reading output from another pane
- Labeling and discovering panes by name
- Any cross-pane interaction between agents

## When NOT to Use

**DON'T use this skill when:**

- Running one-off shell commands in the current pane
- Tasks that don't involve other tmux panes
- You need raw tmux commands → use the `tmux` skill directly

## Command Reference

| Command | Description | Example |
|---|---|---|
| `tmux-bridge list` | Show all panes with target, pid, command, size, label | `tmux-bridge list` |
| `tmux-bridge send <target> <text>` | Frame, type, and submit an agent message | `tmux-bridge send codex "review this"` |
| `tmux-bridge message <target> <text>` | Draft a framed message without Enter | `tmux-bridge message codex "draft"` |
| `tmux-bridge type <target> <text>` | Type text without pressing Enter | `tmux-bridge type codex "hello"` |
| `tmux-bridge read <target> [lines]` | Read last N lines (default 50) | `tmux-bridge read codex 100` |
| `tmux-bridge keys <target> <key>...` | Send special keys | `tmux-bridge keys codex Enter` |
| `tmux-bridge name <target> <label>` | Label a pane (visible in tmux border) | `tmux-bridge name %3 codex` |
| `tmux-bridge resolve <label>` | Print pane target for a label | `tmux-bridge resolve codex` |
| `tmux-bridge id` | Print this pane's ID | `tmux-bridge id` |

## Target Resolution

Targets can be:
- **tmux native**: `session:window.pane` (e.g. `shared:0.1`), pane ID (`%3`), or window index (`0`)
- **label**: Any string set via `tmux-bridge name` — resolved automatically

This means `tmux-bridge type codex "hello"` works directly if the pane was labeled `codex`.

## Messaging Convention

The `send` command automatically frames agent messages with sender and reply information:

```
[tmux-bridge from:claude pane:%4 at:3:0.0 — load the smux skill to reply] Please review src/auth.ts
```

This lets the receiving agent know who sent it and how to reply.

### Receiving messages — IMPORTANT

**When you see a message prefixed with `[tmux-bridge from:<sender>]` that
requires a reply, you MUST reply using `send`:**

```bash
tmux-bridge read <sender> 20
tmux-bridge send <sender> "your response here"
sleep 0.5
tmux-bridge read <sender> 20
```

This sends your reply directly into the sender's pane so they see it immediately. **Do not just respond in your own pane** — the sender won't see it unless you send it back via tmux-bridge.

Keep replies concise (1-3 sentences). They will be typed into the sender's terminal as a single line.
Do not acknowledge a message prefixed `[REPORT — no reply needed]`; that avoids
cross-pane acknowledgment loops.

### Example conversation

**Agent A (claude) sends:**
```bash
tmux-bridge read codex 20       # 1. READ — satisfy read guard
tmux-bridge send codex 'What is the test coverage for src/auth.ts?'
                                 # 2. SEND — frame, type, and submit
sleep 0.5
tmux-bridge read codex 20       # 3. VERIFY — delivery only, not reply polling
# Done. Do NOT read codex again for the response.
# Agent B will reply via tmux-bridge and it will appear in your pane.
```

**Agent B (codex) sees in their prompt:**
```
[tmux-bridge from:claude] What is the test coverage for src/auth.ts?
```

**Agent B replies:**
```bash
tmux-bridge read claude 20      # 1. READ — satisfy read guard
tmux-bridge send claude 'src/auth.ts has 87% line coverage. Missing coverage on the OAuth refresh token path (lines 142-168).'
                                 # 2. SEND — frame, type, and submit
sleep 0.5
tmux-bridge read claude 20      # 3. VERIFY — delivery only
# Done. The reply appears in Agent A's pane automatically.
```

## Read-Send-Verify Cycle

Every delivered agent message MUST follow **read → send → wait 0.5s → verify**.
Before this cycle, hold routine reports while the target is working; only an
urgent steer may enter a busy target. The CLI enforces read-before-act, while
the final read verifies that the target TUI actually processed Enter.

The full cycle for sending a message:

1. **Read** the target pane (satisfies read guard)
2. If it is busy and the message is routine, wait and repeat step 1 after
   2–5 seconds; otherwise **send** it
3. **Wait 0.5 seconds** for the interactive TUI to render the submitted turn
4. **Verify once** in this priority order:
   - If this exact outbound message is shown in the queued banner and the TUI
     offers immediate interruption, press Escape once only when it is an
     urgent steer. Leave it queued only when deferred delivery was explicitly
     requested. A routine report should never be queued while busy.
   - The exact message in transcript plus a fresh active composer means
     submitted.
   - The exact message tail in the active composer means unsubmitted. Press
     Enter once when the target is idle, or when it is an urgent steer. Do not
     submit a routine report if the target unexpectedly became busy.
   - `Working` or tool output alone is inconclusive.

Only apply a corrective key when the pre-send composer was empty and the exact
outbound message is proven to be in the queue or active composer: Escape for a
known queued urgent steer, Enter for a known urgent or idle-target composer
state.
After one corrective key, wait 0.5 seconds and verify once. Never loop or apply
both keys.

### Example: sending a message to an agent

```bash
# 1. READ — check the pane and satisfy read guard
tmux-bridge read codex 20

# 2. SEND — frame, type, and submit atomically
tmux-bridge send codex 'Please review the changes in src/auth.ts'

# 3–4. WAIT, THEN VERIFY DELIVERY
sleep 0.5
tmux-bridge read codex 20

# STOP. Do NOT read codex again to check for a reply.
# The other agent will reply via tmux-bridge into YOUR pane.
```

### Example: approving a prompt (non-agent pane)

```bash
# 1. READ — see what the prompt is asking
tmux-bridge read worker 10

# 2. TYPE — type the answer
tmux-bridge type worker "y"

# 3. READ — verify it landed
tmux-bridge read worker 10

# 4. KEYS — press Enter to submit
tmux-bridge keys worker Enter

# 5. READ — for non-agent panes, you DO need to read to see the result
tmux-bridge read worker 20
```

## Agent-to-Agent Workflow

### Step 1: Label yourself

```bash
tmux-bridge name "$(tmux-bridge id)" claude
```

### Step 2: Discover other panes

```bash
tmux-bridge list
```

### Step 3: Read, then send

```bash
tmux-bridge read codex 20
tmux-bridge send codex 'Please review the changes in src/auth.ts and suggest improvements'
sleep 0.5
tmux-bridge read codex 20
# Done. Wait for the reply to appear in your pane.
```

## Tips

- **Use read → send → wait 0.5s → verify for every agent message.**
- **Classify first** — routine reports wait for idle; urgent corrections may steer.
- **Do not queue routine reports** — queued input can interrupt at a later tool boundary.
- **Do not trust `sent and submitted` alone** — it confirms tmux key delivery, not target TUI acceptance.
- **Never send into a non-empty composer** — stop rather than submitting a user draft.
- **Composer evidence overrides `Working`** — `Working` alone never proves delivery.
- **Use Escape on a busy target only to promote an urgent steer.**
- **Use Enter on a busy target only for an urgent steer.**
- **Never use Tab by default** — queue only when deferred delivery was explicitly requested.
- **`message` is draft-only** — it deliberately leaves text unsubmitted and prints a warning.
- **Read guard is enforced** — you MUST read before every `send`/`type`/`keys`. The CLI will error otherwise.
- **Every action clears the read mark** — after `type`, you must `read` again before `keys`.
- **Never poll for replies** — short reads are allowed only to wait for safe routine-report delivery.
- **Label panes early** — it makes cross-agent communication much easier than using `%N` IDs
- **`type` uses literal mode** — it uses `-l` so special characters are typed as-is
- **`read` defaults to 50 lines** — pass a higher number for more context
- **Non-agent panes** (shells, processes) are the exception — you DO need to read them to see output
- **Framing is automatic** — `send` and `message` add sender pane and reply information.
