# Ghostty & tmux AI Agent Integration — Hooks, Notifications, Status Bars

AI coding agents run in terminals. They finish tasks, need permissions, hit errors, and wait for input — all while you are looking at another pane, another window, or your phone. The feedback loop between agent and human is broken by default. This article shows how to fix it using Ghostty, tmux, Claude Code hooks, and a macOS notch HUD, turning your terminal into an agent-aware operating surface.

**What you will learn:**

- Every integration point Ghostty exposes: AppleScript, OSC sequences, keybind actions, custom shaders, and notification hooks
- tmux's hook system, `pipe-pane`, `capture-pane`, status bar customization, and control mode
- How Claude Code hooks fire JSON payloads you can route to tmux status bars, desktop notifications, and external dashboards
- How to build a feedback loop between AI agents in tmux and a macOS notch HUD app
- How Ghostty compares to iTerm2, Kitty, WezTerm, and Alacritty for terminal automation
- Production patterns from tmux-agent-indicator, tmux-agent-status, dmux, NTM, and claude-tmux
- Anti-patterns that will waste your time

---

## Table of Contents

1. [The Problem](#the-problem)
2. [Core Concepts](#core-concepts)
   - [Ghostty Integration Points](#ghostty-integration-points)
   - [tmux Hook System](#tmux-hook-system)
   - [Claude Code Hooks](#claude-code-hooks)
   - [OSC Escape Sequences](#osc-escape-sequences)
3. [Patterns](#patterns)
   - [Pattern 1: Agent State in the tmux Status Bar](#pattern-1-agent-state-in-the-tmux-status-bar)
   - [Pattern 2: Desktop Notifications via Ghostty OSC 9](#pattern-2-desktop-notifications-via-ghostty-osc-9)
   - [Pattern 3: Two-Line AI Summary Status Bar](#pattern-3-two-line-ai-summary-status-bar)
   - [Pattern 4: Notch HUD Fed by Agent Hooks](#pattern-4-notch-hud-fed-by-agent-hooks)
   - [Pattern 5: Ghostty AppleScript Automation Layouts](#pattern-5-ghostty-applescript-automation-layouts)
   - [Pattern 6: pipe-pane Agent Output Monitoring](#pattern-6-pipe-pane-agent-output-monitoring)
   - [Pattern 7: Custom Ghostty Shader for Agent State](#pattern-7-custom-ghostty-shader-for-agent-state)
4. [Small Examples](#small-examples)
5. [Comparisons](#comparisons)
   - [Terminal Emulators for Automation](#terminal-emulators-for-automation)
   - [tmux Agent Management Tools](#tmux-agent-management-tools)
6. [Anti-Patterns](#anti-patterns)
7. [References](#references)

---

## The Problem

You are running four Claude Code agents in parallel tmux sessions. One finishes and needs a PR review. Another hits a permission prompt and is blocked. A third is still churning through a refactor. The fourth errored out on a rate limit ten minutes ago.

You know none of this. You are staring at pane 2.

The terminal was designed for a single human typing commands. AI agents broke that model. An agent session can run for minutes without human input, then suddenly need attention. The feedback mechanisms available in a raw terminal — cursor blinking, bell character, scrolling output — were never designed for this.

**What goes wrong without integration:**

- **Missed completions** — Agent finishes, sits idle. You check 20 minutes later. Time wasted.
- **Blocked permissions** — Agent needs approval to run a dangerous command. It waits silently. Your pipeline stalls.
- **Rate limit blindness** — API errors happen in a background pane. You don't see them until you manually check.
- **Context switching tax** — You cycle through 4-8 tmux panes every few minutes just to poll status. Interrupts your own work.
- **No glanceable state** — There is no dashboard, no status bar, no indicator showing which agents are working, waiting, or done.

**What changes if you get this right:**

Your tmux status bar shows `3 working | 1 needs-input` at a glance. Ghostty fires a desktop notification when an agent finishes. A macOS notch HUD shows a live summary of what each agent is doing. You never poll. You respond to events. Your throughput with parallel agents doubles because you eliminate idle time.

---

## Core Concepts

### Ghostty Integration Points

[Ghostty](https://ghostty.org) is a GPU-accelerated terminal emulator by Mitchell Hashimoto. It is fast, minimal in configuration, and increasingly scriptable. Here are the integration surfaces that matter for agent automation.

#### AppleScript (macOS, Ghostty 1.3+)

Ghostty exposes a native [AppleScript dictionary](https://ghostty.org/docs/features/applescript) with a hierarchical object model:

```
application → windows → tabs → terminals
```

```typescript
// The Ghostty AppleScript object model, expressed as TypeScript
interface GhosttyApp {
  windows: GhosttyWindow[];
  terminals: GhosttyTerminal[];  // flat access across all windows
  frontWindow: GhosttyWindow;
}

interface GhosttyWindow {
  id: number;
  name: string;
  selectedTab: GhosttyTab;
  tabs: GhosttyTab[];
  terminals: GhosttyTerminal[];
}

interface GhosttyTab {
  id: number;
  name: string;
  index: number;
  selected: boolean;
  focusedTerminal: GhosttyTerminal;
  terminals: GhosttyTerminal[];
}

interface GhosttyTerminal {
  id: number;
  name: string;
  workingDirectory: string;
}

// Commands available on terminals
type GhosttyCommands = {
  inputText: (text: string, terminal: GhosttyTerminal) => void;
  sendKey: (key: string, modifiers?: ('shift' | 'control' | 'option' | 'command')[]) => void;
  sendMouseButton: (button: number, action: 'press' | 'release') => void;
  sendMousePosition: (x: number, y: number) => void;
  sendMouseScroll: (deltaX: number, deltaY: number) => void;
  performAction: (action: string) => void;
  split: (direction: 'right' | 'left' | 'down' | 'up') => void;
  focus: () => void;
  close: () => void;
};

// Surface configuration for new windows/tabs
interface SurfaceConfig {
  fontSize?: number;
  initialWorkingDirectory?: string;
  command?: string;
  initialInput?: string;
  waitAfterCommand?: boolean;
  environmentVariables?: Record<string, string>;
}
```

A practical AppleScript example — broadcast a command to every terminal:

```applescript
tell application "Ghostty"
    repeat with w in windows
        repeat with t in tabs of w
            repeat with term in terminals of t
                input text "git status\n" to term
            end repeat
        end repeat
    end repeat
end tell
```

> **Key insight:** Ghostty's AppleScript is the *only* way to programmatically control Ghostty from outside the terminal. There is no socket, no HTTP API, no IPC channel. If you need external automation on macOS, AppleScript is the path. On Linux, Ghostty has no equivalent — you must work through tmux or shell scripts running inside the terminal.

#### OSC Sequences

Ghostty supports [OSC 9](https://ghostty.org/docs/vt/osc/9) (iTerm2-style desktop notifications) and OSC 777 (rxvt-unicode-style notifications with title and body). These are escape sequences that any process running inside the terminal can emit:

```bash
# OSC 9 — body only (iTerm2 origin, pre-2010)
printf '\e]9;Agent finished: refactor complete\a'

# OSC 777 — title + body (rxvt-unicode origin)
printf '\e]777;notify;Claude Code;Task complete: PR ready for review\a'
```

Ghostty also supports [notification on command finish](https://ghostty.org/docs/install/release-notes/1-3-0) via configuration:

```
# In ~/.config/ghostty/config
notify-on-command-finish = unfocused
notify-on-command-finish-action = notify
notify-on-command-finish-after = 10
```

This fires a macOS desktop notification if a command runs for more than 10 seconds and the terminal is not focused. It uses OSC 133 (semantic prompts) under the hood, so it requires shell integration or a shell that natively sends OSC 133 (Fish, Nushell).

#### Keybind Actions

Ghostty supports [keybind sequences](https://ghostty.org/docs/config/keybind/sequence) with tmux-like chaining:

```
# tmux-style leader key: ctrl+a then n = new window
keybind = ctrl+a>n=new_window
keybind = ctrl+a>s=new_split:right
keybind = ctrl+a>v=new_split:down
```

Actions relevant to automation include:

| Action | Description |
|--------|-------------|
| `write_scrollback_file:paste` | Dump scrollback to a temp file, paste the path into the terminal |
| `write_screen_file:copy` | Dump visible screen to a temp file, copy path to clipboard |
| `write_selection_file:open` | Write selected text to a temp file, open in default editor |
| `new_split:auto` | Create a split pane, auto-choose direction |
| `close_surface` | Close the current pane/tab/window |

The `write_scrollback_file` action is particularly useful: bind it to a key, then a script can read the temp file to extract the latest agent output without `pipe-pane`.

#### Custom Shaders

Ghostty exposes [uniforms to GLSL shaders](https://github.com/ghostty-org/ghostty/discussions/6901) including focus state:

| Uniform | Type | Description |
|---------|------|-------------|
| `iFocus` | `int` | 0 = blurred, 1 = focused |
| `iTimeFocus` | `float` | `iTime` when last focused |
| `iCursorColor` | `vec4` | Current cursor color |
| `iPalette[256]` | `vec4[256]` | Full 256-color ANSI palette |
| `iBackgroundColor` | `vec4` | Terminal background color |
| `iForegroundColor` | `vec4` | Terminal foreground color |

You can use `iFocus` to dim unfocused panes, creating a visual indicator of which pane is active. More on this in [Pattern 7](#pattern-7-custom-ghostty-shader-for-agent-state).

> **Key insight:** Ghostty's shader system is a visual feedback channel, not a data channel. You cannot pass arbitrary state to a shader. But you *can* change the terminal's cursor color or background color from a shell script (via ANSI escape sequences), and the shader will see those changes through the uniform values. This creates an indirect communication path: hook script sets cursor color based on agent state, shader reacts to cursor color.

---

### tmux Hook System

tmux hooks are commands that fire automatically on specific events. They are the backbone of any agent monitoring setup.

```typescript
// tmux hook model
interface TmuxHook {
  event: TmuxHookEvent;
  index: number;       // hooks are arrays: after-new-window[0], after-new-window[1]
  command: string;      // any tmux command or shell command via run-shell
}

type TmuxHookEvent =
  // Window lifecycle
  | 'after-new-window'
  | 'after-kill-window'
  | 'window-renamed'
  // Pane lifecycle
  | 'after-new-session'
  | 'after-split-window'
  | 'after-kill-pane'
  | 'pane-died'
  | 'pane-exited'
  // Focus events
  | 'pane-focus-in'
  | 'pane-focus-out'
  | 'client-focus-in'
  | 'client-focus-out'
  // Session events
  | 'client-session-changed'
  | 'session-renamed'
  | 'client-attached'
  | 'client-detached'
  // Input events
  | 'after-send-keys'
  | 'after-copy-mode'
  // Layout events
  | 'after-select-layout'
  | 'after-resize-pane'
  | 'window-layout-changed'
  // Alert events
  | 'alert-activity'
  | 'alert-bell'
  | 'alert-silence'
  // Theme events (tmux 3.5+)
  | 'client-light-theme'
  | 'client-dark-theme';
```

Setting a hook:

```bash
# Run a script whenever a pane gains focus
tmux set-hook -g pane-focus-in 'run-shell "~/.tmux/hooks/on-focus.sh #{pane_id}"'

# Run a script when a pane's process exits
tmux set-hook -g pane-exited 'run-shell "~/.tmux/hooks/on-exit.sh #{pane_id} #{pane_dead_status}"'

# Refresh status bar when a window is renamed
tmux set-hook -g window-renamed 'refresh-client -S'

# Chain multiple commands on one hook
tmux set-hook -g pane-focus-in[0] 'select-pane -T "#{pane_current_command}"'
tmux set-hook -g pane-focus-in[1] 'refresh-client -S'
```

> **Key insight:** tmux hooks fire tmux commands, not shell commands. To run a shell script, wrap it in `run-shell`. The `run-shell` command runs asynchronously by default — it does not block tmux. Variables like `#{pane_id}`, `#{session_name}`, and `#{pane_current_path}` are expanded before execution.

#### Status Bar Customization

The tmux status bar is the primary surface for showing agent state. Key options:

```bash
# Enable two-line status bar (tmux 2.9+)
set -g status 2

# Set refresh interval (seconds)
set -g status-interval 5

# Status bar content — can call shell scripts
set -g status-right '#(~/.tmux/scripts/agent-status.sh) | %H:%M'

# Multi-line format (when status = 2)
set -g status-format[0] '#[align=left]#{agent_summary}#[align=right]#{session_name}'
set -g status-format[1] '#[align=left]#{agent_detail}#[align=right]#{git_branch}'

# Status bar styling
set -g status-style 'bg=#1e1e2e,fg=#cdd6f4'
set -g status-right-length 120
set -g status-left-length 40
```

#### pipe-pane and capture-pane

`pipe-pane` sends all output from a pane to an external command in real time. `capture-pane` takes a snapshot.

```bash
# Log all output from a pane to a file
tmux pipe-pane -o "cat >> ~/logs/agent-#{pane_id}.log"

# Pipe output through a filter — notify on specific patterns
tmux pipe-pane -o 'grep --line-buffered "Task complete\|Error\|Permission" >> /tmp/agent-events.log'

# Stop piping
tmux pipe-pane

# Capture current screen contents to stdout
tmux capture-pane -p -S -

# Capture and save to file
tmux capture-pane -p -S - > /tmp/pane-snapshot.txt
```

> **Key insight:** `pipe-pane -o` only captures output (what the program writes to the terminal), not input (what the user types). For agent monitoring, this is exactly what you want — you see the agent's responses without the noise of your prompts. The `-I` flag adds input if you need it.

#### Control Mode

tmux control mode (`tmux -CC`) sends structured protocol messages instead of raw terminal output. [iTerm2 uses this extensively](https://iterm2.com/documentation-tmux-integration.html) to render tmux windows as native tabs and panes.

```bash
# Start control mode (used by iTerm2)
tmux -CC attach

# The protocol output looks like:
# %begin 1234567890 1 0
# %output %0 hello world
# %end 1234567890 1 0
```

No other terminal implements control mode integration besides iTerm2. Ghostty does not support it. This is a significant differentiator — if you rely on `tmux -CC` for native tab integration, iTerm2 is the only option.

#### DCS Passthrough

When running inside tmux, escape sequences from programs do not reach the outer terminal by default. tmux intercepts them. To pass OSC sequences through to Ghostty (or any terminal), enable passthrough:

```bash
# In ~/.tmux.conf
set -g allow-passthrough on
```

The wrapped format doubles `\033` characters and wraps in a DCS sequence:

```bash
# Raw OSC 9 (outside tmux):
printf '\e]9;Notification text\a'

# DCS passthrough (inside tmux):
printf '\ePtmux;\e\e]9;Notification text\a\e\\'
```

Most modern tools handle this wrapping automatically if `allow-passthrough` is enabled.

---

### Claude Code Hooks

[Claude Code hooks](https://code.claude.com/docs/en/hooks) are the primary mechanism for integrating AI agent state with external systems. When a hook event fires, Claude Code passes a JSON payload to your handler via stdin.

```typescript
// Claude Code hook event types
type HookEvent =
  | 'SessionStart'         // Session begins or resumes
  | 'UserPromptSubmit'     // User sends a prompt
  | 'PreToolUse'           // Before tool execution
  | 'PostToolUse'          // After successful tool execution
  | 'PostToolUseFailure'   // After failed tool execution
  | 'PermissionRequest'    // Permission dialog about to show
  | 'Notification'         // Agent sends a notification
  | 'Stop'                 // Agent finishes responding
  | 'StopFailure'          // Turn ends due to API error
  | 'SubagentStart'        // Subagent spawned
  | 'SubagentStop'         // Subagent finishes
  | 'TaskCreated'          // Agent team task created
  | 'TaskCompleted'        // Agent team task completed
  | 'TeammateIdle'         // Team member about to go idle
  | 'SessionEnd';          // Session terminates

// Common JSON payload (received via stdin)
interface HookPayload {
  session_id: string;
  transcript_path: string;
  cwd: string;
  permission_mode: 'default' | 'plan' | 'acceptEdits' | 'auto' | 'dontAsk' | 'bypassPermissions';
  hook_event_name: HookEvent;
  agent_id?: string;        // Present in subagent hooks
  agent_type?: string;      // Present in subagent hooks
}

// Stop hook payload
interface StopPayload extends HookPayload {
  stop_hook_active: boolean;
  last_assistant_message: string;
}

// Notification hook payload
interface NotificationPayload extends HookPayload {
  message: string;
  title: string;
  notification_type: 'permission_prompt' | 'idle_prompt' | 'auth_success' | 'elicitation_dialog';
}

// StopFailure hook payload
interface StopFailurePayload extends HookPayload {
  error: 'rate_limit' | 'authentication_failed' | 'billing_error' | 'server_error' | 'max_output_tokens' | 'unknown';
  error_details: string;
  last_assistant_message: string;
}
```

Hook configuration lives in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/on-stop.sh",
            "timeout": 30
          }
        ]
      }
    ],
    "Notification": [
      {
        "matcher": "permission_prompt",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/on-permission.sh",
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

> **Key insight:** Hook handlers communicate back to Claude Code through exit codes and JSON on stdout. Exit 0 = success (parse stdout for JSON). Exit 2 = blocking error (stderr is fed to Claude as an error message). Any other exit code = non-blocking error (logged but ignored). This means your hook can *control* Claude's behavior — a Stop hook returning `{"decision": "block", "reason": "Tests not passing"}` will tell Claude to keep going.

---

### OSC Escape Sequences

OSC (Operating System Command) sequences are the universal mechanism for terminal-to-emulator communication. Three standards matter for notifications:

```typescript
// OSC notification protocols
interface OSCNotification {
  // OSC 9 — iTerm2 style (simplest, widest support)
  // Format: \e]9;{body}\a
  osc9: {
    body: string;
  };

  // OSC 777 — rxvt-unicode style (title + body)
  // Format: \e]777;notify;{title};{body}\a
  osc777: {
    title: string;
    body: string;
  };

  // OSC 99 — Kitty style (extensible, with focus-on-click)
  // Format: \e]99;i=1:d=0;{body}\a
  osc99: {
    id: number;
    done: boolean;     // d=0 means complete notification
    body: string;
    // Can optionally focus window on click
    // Can send escape code back to application
  };
}

// Terminal support matrix
const oscSupport = {
  'OSC 9':  ['Ghostty', 'iTerm2', 'Windows Terminal', 'WezTerm'],
  'OSC 777': ['Ghostty', 'urxvt', 'foot', 'contour'],
  'OSC 99': ['Kitty'],
} as const;
```

Helper functions for sending notifications:

```bash
#!/usr/bin/env bash
# notify.sh — Send desktop notifications via OSC escape sequences

notify_osc9() {
  local body="$1"
  printf '\e]9;%s\a' "$body"
}

notify_osc777() {
  local title="$1"
  local body="$2"
  printf '\e]777;notify;%s;%s\a' "$title" "$body"
}

# Detect if running inside tmux and wrap in DCS passthrough
notify_passthrough() {
  local title="$1"
  local body="$2"

  if [ -n "$TMUX" ]; then
    # DCS passthrough — double the ESC, wrap in \ePtmux;...\e\\
    printf '\ePtmux;\e\e]777;notify;%s;%s\a\e\\' "$title" "$body"
  else
    printf '\e]777;notify;%s;%s\a' "$title" "$body"
  fi
}
```

> **Key insight:** Ghostty supports OSC 9 and OSC 777 but not OSC 99 (Kitty's protocol). If you are writing a notification script that needs to work across terminals, use OSC 9 as the lowest common denominator. OSC 777 adds a title field but has narrower support. Always check for tmux and wrap in DCS passthrough if `$TMUX` is set.

---

## Patterns

### Pattern 1: Agent State in the tmux Status Bar

**What it does:** Shows live agent state (working/done/waiting) in the tmux status bar using file-based state tracking.

**When to use it:** You are running 1-8 Claude Code agents across tmux sessions and want a glanceable overview without switching panes.

This pattern is the foundation used by both [tmux-agent-indicator](https://github.com/accessd/tmux-agent-indicator) and [tmux-agent-status](https://github.com/samleeney/tmux-agent-status). Here is a minimal implementation from scratch.

#### Step 1: Claude Code hooks write state to files

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/agent-state.sh
# Called by Claude Code hooks to write state to a cache file
# Usage: agent-state.sh <state>
# States: working, done, waiting, error

STATE_DIR="$HOME/.cache/agent-status"
mkdir -p "$STATE_DIR"

# Read the hook payload from stdin
PAYLOAD=$(cat)

# Extract session ID and event
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id // "unknown"')
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name // "unknown"')
CWD=$(echo "$PAYLOAD" | jq -r '.cwd // ""')

# Determine state from event
case "$EVENT" in
  UserPromptSubmit|PreToolUse)
    STATE="working"
    ;;
  Stop)
    STATE="done"
    ;;
  Notification)
    NOTIFICATION_TYPE=$(echo "$PAYLOAD" | jq -r '.notification_type // ""')
    case "$NOTIFICATION_TYPE" in
      permission_prompt) STATE="waiting" ;;
      idle_prompt)       STATE="done" ;;
      *)                 STATE="working" ;;
    esac
    ;;
  StopFailure)
    STATE="error"
    ERROR_TYPE=$(echo "$PAYLOAD" | jq -r '.error // "unknown"')
    ;;
  *)
    STATE="working"
    ;;
esac

# Get the tmux session name if running in tmux
if [ -n "$TMUX" ]; then
  TMUX_SESSION=$(tmux display-message -p '#{session_name}')
else
  TMUX_SESSION="$SESSION_ID"
fi

# Write state file
STATE_FILE="$STATE_DIR/${TMUX_SESSION}.json"
cat > "$STATE_FILE" <<EOF
{
  "state": "$STATE",
  "event": "$EVENT",
  "session": "$TMUX_SESSION",
  "cwd": "$CWD",
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "error": "${ERROR_TYPE:-}"
}
EOF
```

#### Step 2: Configure Claude Code hooks

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/agent-state.sh",
          "timeout": 5
        }]
      }
    ],
    "Stop": [
      {
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/agent-state.sh",
          "timeout": 5
        }]
      }
    ],
    "Notification": [
      {
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/agent-state.sh",
          "timeout": 5
        }]
      }
    ],
    "StopFailure": [
      {
        "hooks": [{
          "type": "command",
          "command": "~/.claude/hooks/agent-state.sh",
          "timeout": 5
        }]
      }
    ]
  }
}
```

#### Step 3: tmux status bar script reads state files

```bash
#!/usr/bin/env bash
# ~/.tmux/scripts/agent-status.sh
# Called by tmux status-right every N seconds

STATE_DIR="$HOME/.cache/agent-status"

working=0
done_count=0
waiting=0
errored=0

if [ -d "$STATE_DIR" ]; then
  for f in "$STATE_DIR"/*.json; do
    [ -f "$f" ] || continue

    state=$(jq -r '.state' "$f" 2>/dev/null)
    updated=$(jq -r '.updated' "$f" 2>/dev/null)

    # Skip stale entries (older than 1 hour)
    if [ -n "$updated" ]; then
      updated_epoch=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$updated" +%s 2>/dev/null || echo 0)
      now_epoch=$(date +%s)
      age=$(( now_epoch - updated_epoch ))
      if [ "$age" -gt 3600 ]; then
        continue
      fi
    fi

    case "$state" in
      working) working=$((working + 1)) ;;
      done)    done_count=$((done_count + 1)) ;;
      waiting) waiting=$((waiting + 1)) ;;
      error)   errored=$((errored + 1)) ;;
    esac
  done
fi

# Build status string
parts=()
[ "$working" -gt 0 ]    && parts+=("#[fg=blue]⚡${working} working#[default]")
[ "$waiting" -gt 0 ]    && parts+=("#[fg=yellow]⏳${waiting} waiting#[default]")
[ "$done_count" -gt 0 ] && parts+=("#[fg=green]✓${done_count} done#[default]")
[ "$errored" -gt 0 ]    && parts+=("#[fg=red]✗${errored} error#[default]")

if [ ${#parts[@]} -eq 0 ]; then
  echo "no agents"
else
  IFS='|'
  echo "${parts[*]}"
fi
```

#### Step 4: tmux.conf wiring

```bash
# In ~/.tmux.conf
set -g status-interval 5
set -g status-right-length 80
set -g status-right '#(~/.tmux/scripts/agent-status.sh) | %H:%M'

# Enable passthrough for notifications
set -g allow-passthrough on

# Enable focus events so pane-focus-in/out hooks fire
set -g focus-events on
```

**Gotchas:**

- `status-interval 5` means the status bar updates every 5 seconds. Set lower for more responsiveness at the cost of running the script more often.
- `jq` must be installed. The state files are JSON so you can extend them with arbitrary metadata.
- State files persist across tmux restarts. Add cleanup logic or use `/tmp` instead of `~/.cache` if you want them to be ephemeral.
- On macOS, `date -j` has different syntax than GNU `date`. The script above uses BSD `date` syntax. For Linux, replace with `date -d "$updated" +%s`.

---

### Pattern 2: Desktop Notifications via Ghostty OSC 9

**What it does:** Sends macOS desktop notifications when an agent finishes, needs permission, or errors out.

**When to use it:** You are working in a different application (browser, editor) and want to be pulled back to the terminal only when an agent needs attention.

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/notify-desktop.sh
# Sends desktop notifications via Ghostty OSC 9/777

PAYLOAD=$(cat)
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name')
CWD=$(echo "$PAYLOAD" | jq -r '.cwd')
PROJECT=$(basename "$CWD")

send_notification() {
  local title="$1"
  local body="$2"

  if [ -n "$TMUX" ]; then
    # DCS passthrough for tmux
    printf '\ePtmux;\e\e]777;notify;%s;%s\a\e\\' "$title" "$body"
  else
    printf '\e]777;notify;%s;%s\a' "$title" "$body"
  fi
}

case "$EVENT" in
  Stop)
    LAST_MSG=$(echo "$PAYLOAD" | jq -r '.last_assistant_message // ""' | head -c 200)
    send_notification "Claude Code [$PROJECT]" "Task complete: ${LAST_MSG:0:100}"
    ;;
  Notification)
    MSG=$(echo "$PAYLOAD" | jq -r '.message // "Needs attention"')
    NTYPE=$(echo "$PAYLOAD" | jq -r '.notification_type // ""')
    case "$NTYPE" in
      permission_prompt)
        send_notification "Claude Code [$PROJECT]" "Permission needed: $MSG"
        ;;
      idle_prompt)
        send_notification "Claude Code [$PROJECT]" "Waiting for input"
        ;;
    esac
    ;;
  StopFailure)
    ERROR=$(echo "$PAYLOAD" | jq -r '.error // "unknown"')
    DETAILS=$(echo "$PAYLOAD" | jq -r '.error_details // ""' | head -c 100)
    send_notification "Claude Code ERROR [$PROJECT]" "$ERROR: $DETAILS"
    ;;
esac
```

Add this alongside the state hook in `~/.claude/settings.json`:

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/agent-state.sh",
            "timeout": 5
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/notify-desktop.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "Notification": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/agent-state.sh",
            "timeout": 5
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/notify-desktop.sh",
            "timeout": 5
          }
        ]
      }
    ],
    "StopFailure": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/agent-state.sh",
            "timeout": 5
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/notify-desktop.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**Gotchas:**

- Ghostty must have notification permissions in macOS System Settings > Notifications.
- `allow-passthrough on` must be set in tmux.conf for notifications to reach Ghostty through tmux.
- OSC 9 notifications are fire-and-forget — there is no callback when the user clicks the notification.
- Notification content should be short. macOS truncates long notification bodies.
- [Ghostty native notifications](https://ghostty.org/docs/install/release-notes/1-3-0) (`notify-on-command-finish`) work independently of these hooks and fire based on command completion time, not agent lifecycle events. Both can coexist.

---

### Pattern 3: Two-Line AI Summary Status Bar

**What it does:** Generates a live, AI-written summary of what each agent session is doing and displays it in a two-line tmux status bar.

**When to use it:** You are running long-running agents and want more context than "working/done" — you want to know *what* they are working on.

This pattern was pioneered by [Quickchat AI](https://quickchat.ai/post/tmux-session-summaries-for-parallel-ai-agents). Here is a clean implementation.

#### The summarization hook

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/summarize.sh
# Called asynchronously after each Claude Code response
# Generates a 2-3 sentence summary of the session's current task

PAYLOAD=$(cat)
TRANSCRIPT=$(echo "$PAYLOAD" | jq -r '.transcript_path')
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id')

# Get tmux session name
if [ -n "$TMUX" ]; then
  SESSION_NAME=$(tmux display-message -p '#{session_name}')
else
  SESSION_NAME="$SESSION_ID"
fi

SUMMARY_DIR="/tmp/claude-summaries"
mkdir -p "$SUMMARY_DIR"
SUMMARY_FILE="$SUMMARY_DIR/${SESSION_NAME}.txt"
PREV_SUMMARY=""
[ -f "$SUMMARY_FILE" ] && PREV_SUMMARY=$(cat "$SUMMARY_FILE")

# Extract last 4 meaningful exchanges from the transcript
# Filter out tool-use entries, keep only text messages
CONTEXT=$(jq -r '
  select(.type == "message") |
  if .role == "user" then
    if (.content | type) == "string" then
      "USER: " + (.content | .[0:300])
    else
      empty
    end
  elif .role == "assistant" then
    if (.content | type) == "array" then
      .content[] | select(.type == "text") | "ASSISTANT: " + (.text | .[0:300])
    else
      empty
    end
  else
    empty
  end
' "$TRANSCRIPT" 2>/dev/null | tail -8)

# Skip if no meaningful context
[ -z "$CONTEXT" ] && exit 0

# Build the summarization prompt
PROMPT="You are a status line generator for a developer dashboard.

Given the conversation below, produce a factual, consolidated description of the session's core task.
Lead with the main goal. Follow with 1-2 sentences on current progress. Maximum 2-3 sentences total.
Do not use markdown. Do not use quotes. Plain text only. Keep under 200 characters.

Previous status line (keep first sentence stable if goal unchanged):
${PREV_SUMMARY}

Recent conversation:
${CONTEXT}"

# Call a fast model for summarization
# IMPORTANT: unset CLAUDECODE to avoid blocking nested calls
SUMMARY=$(env -u CLAUDECODE claude -p --model haiku "$PROMPT" 2>/dev/null)

if [ -n "$SUMMARY" ]; then
  echo "$SUMMARY" > "$SUMMARY_FILE"
fi
```

#### The status bar renderer

```bash
#!/usr/bin/env bash
# ~/.tmux/scripts/summary-line.sh
# Reads the AI-generated summary and wraps it for the tmux status bar

SESSION_NAME="$1"
SUMMARY_FILE="/tmp/claude-summaries/${SESSION_NAME}.txt"

if [ -f "$SUMMARY_FILE" ]; then
  # Get terminal width for wrapping
  WIDTH=$(tmux display-message -p '#{client_width}' 2>/dev/null || echo 120)
  MAX_LEN=$(( WIDTH * 6 / 10 ))  # Use 60% of width

  SUMMARY=$(cat "$SUMMARY_FILE")

  # Truncate at word boundary
  if [ ${#SUMMARY} -gt "$MAX_LEN" ]; then
    SUMMARY="${SUMMARY:0:$MAX_LEN}"
    SUMMARY="${SUMMARY% *}..."
  fi

  echo "$SUMMARY"
else
  echo "No summary yet"
fi
```

#### tmux.conf for two-line status

```bash
# Enable two-line status bar
set -g status 2

# Line 0: AI summary (left) + session info (right)
set -g status-format[0] '#[align=left,fg=cyan]#(~/.tmux/scripts/summary-line.sh "#{session_name}")#[align=right,fg=white]#{session_name} #[fg=gray]%H:%M'

# Line 1: Agent counts (left) + git branch (right)
set -g status-format[1] '#[align=left]#(~/.tmux/scripts/agent-status.sh)#[align=right,fg=magenta]#(cd #{pane_current_path} && git branch --show-current 2>/dev/null || echo "no git")'

# Refresh every 5 seconds
set -g status-interval 5
set -g status-right-length 120
```

#### Hook configuration

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/summarize.sh",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**Gotchas:**

- The `env -u CLAUDECODE` is critical. Inside an active Claude Code session, the `CLAUDECODE` environment variable blocks nested `claude` invocations. Unsetting it allows the summarization call to work.
- `claude -p` ignores stdin when given a positional argument. The prompt must be passed as the argument, not piped.
- The previous summary is fed back to maintain stability. Without this, the summary flickers between different phrasings of the same task on every refresh.
- This calls Haiku on every Stop event. At ~$0.001 per call, it costs roughly $0.10/day for a heavy user. Use `--model haiku` explicitly — do not let it default to a more expensive model.
- tmux `status-format` requires tmux 2.9+. Older versions do not support multi-line status bars.

---

### Pattern 4: Notch HUD Fed by Agent Hooks

**What it does:** Writes agent state to `~/.atlas/status.json`, which a macOS notch HUD app reads via file watching. The HUD displays agent state in the MacBook notch area.

**When to use it:** You want a persistent, always-visible agent dashboard that is visible regardless of which application is in the foreground.

This pattern uses the architecture from the [Atlas HUD](https://github.com/garywu/atlas), a macOS app that watches `~/.atlas/status.json` using `DispatchSource.makeFileSystemObjectSource` (kernel-level file events, no polling).

#### The status file format

```typescript
// ~/.atlas/status.json schema
interface AtlasStatus {
  status: 'green' | 'yellow' | 'red';   // Severity — drives HUD layout
  source: string;                         // Which system wrote this
  message: string;                        // Shown in hover panel
  banner?: string;                        // Marquee text in the notch
  bannerStyle?: 'scroll' | 'typewriter' | 'flash' | 'slide' | 'split-flap';
  updated: string;                        // ISO timestamp
  details: string[];                      // Expandable detail lines
  slots?: Record<string, SlotData>;       // Named data slots
}

interface SlotData {
  label: string;
  value: string;
  style?: 'normal' | 'warning' | 'critical';
}
```

The HUD app uses a severity-driven layout:

| Severity | HUD State | Description |
|----------|-----------|-------------|
| `green` | Collapsed | Just dots flanking the notch. All agents nominal. |
| `yellow` | Expanded (small) | 200pt left ear. An agent needs attention. |
| `red` | Expanded (large) | 380pt left ear. Error or urgent state. |

#### The hook that writes to the HUD

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/hud-status.sh
# Aggregates agent states and writes to ~/.atlas/status.json

STATE_DIR="$HOME/.cache/agent-status"
STATUS_FILE="$HOME/.atlas/status.json"

mkdir -p "$(dirname "$STATUS_FILE")"

# First, run the agent-state.sh logic to update this session's state
PAYLOAD=$(cat)
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name')
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id')
CWD=$(echo "$PAYLOAD" | jq -r '.cwd')

# Write per-session state (same as Pattern 1)
# ... (reuse agent-state.sh logic) ...

# Now aggregate all sessions into a single HUD status
working=0
done_count=0
waiting=0
errored=0
details=()
slots="{}"

for f in "$STATE_DIR"/*.json; do
  [ -f "$f" ] || continue

  state=$(jq -r '.state' "$f" 2>/dev/null)
  session=$(jq -r '.session' "$f" 2>/dev/null)
  cwd=$(jq -r '.cwd' "$f" 2>/dev/null)
  project=$(basename "$cwd" 2>/dev/null)

  case "$state" in
    working) working=$((working + 1)); details+=("⚡ $session ($project): working") ;;
    done)    done_count=$((done_count + 1)); details+=("✓ $session ($project): done") ;;
    waiting) waiting=$((waiting + 1)); details+=("⏳ $session ($project): needs input") ;;
    error)   errored=$((errored + 1)); details+=("✗ $session ($project): error") ;;
  esac
done

total=$((working + done_count + waiting + errored))

# Determine severity
if [ "$errored" -gt 0 ] || [ "$waiting" -gt 0 ]; then
  if [ "$errored" -gt 0 ]; then
    severity="red"
    message="$errored agent(s) in error state"
    banner="ERROR: Check agent sessions"
    banner_style="flash"
  else
    severity="yellow"
    message="$waiting agent(s) waiting for input"
    banner="$waiting agent(s) need attention"
    banner_style="typewriter"
  fi
elif [ "$working" -gt 0 ]; then
  severity="green"
  message="$working agent(s) working, $done_count done"
  banner=""
  banner_style=""
else
  severity="green"
  message="All agents idle"
  banner=""
  banner_style=""
fi

# Build details JSON array
DETAILS_JSON=$(printf '%s\n' "${details[@]}" | jq -R . | jq -s .)

# Write the HUD status file (atomic write via temp file)
TEMP_FILE=$(mktemp)
cat > "$TEMP_FILE" <<EOF
{
  "status": "$severity",
  "source": "claude-hooks",
  "message": "$message",
  "banner": $([ -n "$banner" ] && echo "\"$banner\"" || echo "null"),
  "bannerStyle": $([ -n "$banner_style" ] && echo "\"$banner_style\"" || echo "null"),
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "details": $DETAILS_JSON,
  "slots": {
    "agents": {"label": "Active", "value": "$total"},
    "working": {"label": "Working", "value": "$working"},
    "waiting": {"label": "Waiting", "value": "$waiting", "style": $([ "$waiting" -gt 0 ] && echo '"warning"' || echo '"normal"')},
    "errors": {"label": "Errors", "value": "$errored", "style": $([ "$errored" -gt 0 ] && echo '"critical"' || echo '"normal"')}
  }
}
EOF
mv "$TEMP_FILE" "$STATUS_FILE"
```

#### How the HUD reads it (Swift, from AtlasHUD)

```swift
// StatusWatcher.swift — file watcher using GCD DispatchSource
@Observable
class StatusWatcher {
    static let shared = StatusWatcher()

    var currentStatus: AtlasStatus = AtlasStatus(
        status: "green", source: "jane",
        message: "Connecting...", updated: "", details: []
    )

    private var statusMonitor: DispatchSourceFileSystemObject?
    private let statusPath = NSString("~/.atlas/status.json").expandingTildeInPath

    func startWatching() {
        readStatus()
        // Watch for file writes — kernel-level, no polling
        let fd = open(statusPath, O_EVTONLY)
        guard fd >= 0 else { return }
        let source = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: fd,
            eventMask: [.write, .rename],
            queue: .main
        )
        source.setEventHandler { [weak self] in self?.readStatus() }
        source.setCancelHandler { close(fd) }
        source.resume()
        statusMonitor = source
    }

    private func readStatus() {
        guard let data = FileManager.default.contents(atPath: statusPath),
              let status = try? JSONDecoder().decode(AtlasStatus.self, from: data) else {
            return
        }
        currentStatus = status
    }
}
```

The full feedback loop:

```
Claude Code agent (in tmux pane)
  → fires Stop/Notification/StopFailure hook
    → hook script writes per-session state to ~/.cache/agent-status/
    → hook script aggregates all sessions → writes ~/.atlas/status.json
      → macOS HUD app detects file change (DispatchSource, ~instant)
        → HUD updates: notch expands, banner animates, severity colors change
```

> **Key insight:** The file-based approach is the right abstraction. It decouples the agent (which runs in a terminal) from the HUD (which is a native macOS app). No sockets, no HTTP servers, no IPC complexity. The kernel's file event system (`kqueue` on macOS, `inotify` on Linux) makes it effectively instant. The HUD app does not poll — it reacts to file changes.

**Gotchas:**

- Use atomic writes (write to temp file, then `mv`). If the HUD reads a partially-written JSON file, it will fail silently (the `try?` in Swift swallows the error).
- The `~/.atlas/` directory must exist. The hook should `mkdir -p` it.
- Multiple agents writing to the same status file will race. The aggregation approach (read all per-session files, then write one aggregate) avoids this because each agent writes its own session file.
- The HUD watches one file. If you want per-agent detail, put it in the `details` array or `slots` map, not separate files.

---

### Pattern 5: Ghostty AppleScript Automation Layouts

**What it does:** Uses Ghostty's AppleScript API to create multi-pane agent layouts programmatically.

**When to use it:** You want to spin up a consistent agent workspace with one command — four panes, each running an agent in a specific directory.

```applescript
-- agent-layout.applescript
-- Creates a 2x2 grid of agent sessions in Ghostty

tell application "Ghostty"
    -- Create a new window with specific config
    set cfg to new surface configuration
    set initial working directory of cfg to "/Users/admin/Work/project-a"

    set win to new window with cfg
    set mainTab to selected tab of win

    -- Get the initial terminal
    set t1 to focused terminal of mainTab

    -- Split right for second agent
    split t1 direction right
    set t2 to focused terminal of mainTab

    -- Split t1 down for third agent
    focus t1
    split t1 direction down
    set t3 to focused terminal of mainTab

    -- Split t2 down for fourth agent
    focus t2
    split t2 direction down
    set t4 to focused terminal of mainTab

    -- Start agents in each pane
    input text "cd /Users/admin/Work/project-a && claude\n" to t1
    input text "cd /Users/admin/Work/project-b && claude\n" to t2
    input text "cd /Users/admin/Work/project-c && claude\n" to t3
    input text "cd /Users/admin/Work/project-d && claude\n" to t4

    -- Name each pane for identification
    input text "/clear\n" to t1
    input text "/clear\n" to t2
    input text "/clear\n" to t3
    input text "/clear\n" to t4
end tell
```

Run it from the command line:

```bash
osascript agent-layout.applescript
```

Or wrap it in a shell function:

```bash
#!/usr/bin/env bash
# agent-workspace.sh — Create a multi-agent Ghostty workspace

PROJECTS=("$@")

if [ ${#PROJECTS[@]} -lt 1 ]; then
  echo "Usage: agent-workspace.sh <project-dir> [project-dir] ..."
  exit 1
fi

# Generate AppleScript dynamically
SCRIPT='tell application "Ghostty"
  set cfg to new surface configuration
  set initial working directory of cfg to "'"${PROJECTS[0]}"'"
  set win to new window with cfg
  set mainTab to selected tab of win
  set t1 to focused terminal of mainTab
  input text "claude\n" to t1'

for i in $(seq 1 $((${#PROJECTS[@]} - 1))); do
  dir="${PROJECTS[$i]}"
  if [ $((i % 2)) -eq 1 ]; then
    direction="right"
  else
    direction="down"
  fi
  SCRIPT+="
  split t1 direction $direction
  set t$((i + 1)) to focused terminal of mainTab
  input text \"cd $dir && claude\n\" to t$((i + 1))"
done

SCRIPT+='
end tell'

osascript -e "$SCRIPT"
```

**Gotchas:**

- AppleScript is macOS-only. Ghostty on Linux has no equivalent automation API.
- The AppleScript API is labeled "preview" as of Ghostty 1.3. The API may change in future releases.
- `macOS-applescript = false` in Ghostty config disables AppleScript entirely. Check if the user has disabled it.
- macOS TCC (Transparency, Consent, and Control) will prompt for Automation permission the first time another app tries to control Ghostty.
- Unlike tmux splits, Ghostty splits do not persist across application restarts. If Ghostty crashes, your layout is gone. Use tmux inside Ghostty for persistence.

> **Key insight:** Ghostty AppleScript is for *initial setup* — creating layouts, launching agents, and one-time commands. For *ongoing monitoring*, use tmux hooks and status bars. The two complement each other: Ghostty gives you the window management layer, tmux gives you the session persistence and state tracking layer.

---

### Pattern 6: pipe-pane Agent Output Monitoring

**What it does:** Uses tmux's `pipe-pane` to stream agent output to an external processor that detects events and triggers actions.

**When to use it:** You want to monitor agent output in real time without modifying the agent's hook configuration. Useful for agents that do not support hooks (Codex, Aider, custom scripts).

```bash
#!/usr/bin/env bash
# ~/.tmux/scripts/monitor-agent.sh
# Processes piped output from a tmux pane to detect agent events

STATE_DIR="$HOME/.cache/agent-status"
mkdir -p "$STATE_DIR"

PANE_ID="$1"
SESSION_NAME="$2"

STATE_FILE="$STATE_DIR/${SESSION_NAME}.json"

write_state() {
  local state="$1"
  local detail="$2"
  cat > "$STATE_FILE" <<EOF
{
  "state": "$state",
  "event": "pipe-pane-detect",
  "session": "$SESSION_NAME",
  "detail": "$detail",
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
}
EOF
}

# Read piped output line by line
while IFS= read -r line; do
  # Strip ANSI escape codes for pattern matching
  clean=$(echo "$line" | sed 's/\x1b\[[0-9;]*m//g; s/\x1b\][^\\]*\\//g')

  # Detect common agent completion patterns
  case "$clean" in
    *"Task complete"*|*"I've finished"*|*"Done!"*|*"Changes applied"*)
      write_state "done" "$(echo "$clean" | head -c 100)"
      ;;
    *"Permission"*|*"approve"*|*"Allow"*|*"deny"*|*"Y/n"*)
      write_state "waiting" "$(echo "$clean" | head -c 100)"
      ;;
    *"Error"*|*"error"*|*"FAILED"*|*"rate limit"*|*"429"*)
      write_state "error" "$(echo "$clean" | head -c 100)"
      ;;
    *"Thinking"*|*"Working"*|*"Analyzing"*|*"Reading"*|*"Writing"*)
      write_state "working" "$(echo "$clean" | head -c 100)"
      ;;
  esac
done
```

Activate monitoring for a pane:

```bash
# Start monitoring the current pane
tmux pipe-pane -o "~/.tmux/scripts/monitor-agent.sh '#{pane_id}' '#{session_name}'"

# Or bind it to a key
# In ~/.tmux.conf:
bind-key M pipe-pane -o "~/.tmux/scripts/monitor-agent.sh '#{pane_id}' '#{session_name}'" \; display-message "Agent monitoring ON"
bind-key m pipe-pane \; display-message "Agent monitoring OFF"
```

**Gotchas:**

- Pattern matching on output is fragile. Agents change their output format between versions. Hook-based integration (Pattern 1) is always preferable when available.
- `pipe-pane` runs the command in a subshell. The command receives raw terminal output including ANSI escape codes. You must strip them before pattern matching.
- Only one `pipe-pane` handler can be active per pane at a time. If you call `pipe-pane` again, the previous handler is replaced.
- The `read -r line` approach buffers by newline. If the agent writes partial lines (e.g., a progress spinner), the handler will not see them until the line is complete.
- The `tmux-logging` plugin (`tmux-plugins/tmux-logging`) strips ANSI codes from logged output, which is cleaner than raw `pipe-pane`. Consider using it if you need logging alongside monitoring.

---

### Pattern 7: Custom Ghostty Shader for Agent State

**What it does:** Uses a Ghostty custom shader to visually indicate agent state — dim unfocused panes, add a colored border based on agent state, or animate the background.

**When to use it:** You want visual differentiation between active and inactive agent panes without relying on tmux borders.

```glsl
// ~/.config/ghostty/shaders/agent-state.glsl
// Dims unfocused terminals and adds a colored left-edge indicator
// The indicator color is controlled by setting the cursor color via escape sequences

// Ghostty provides these uniforms
uniform int iFocus;           // 0 = blurred, 1 = focused
uniform float iTime;
uniform float iTimeFocus;     // iTime when last focused
uniform vec4 iCursorColor;    // Current cursor color — we use this as state signal
uniform sampler2D iChannel0;  // Terminal texture
uniform vec2 iResolution;     // Terminal size in pixels

void mainImage(out vec4 fragColor, in vec2 fragCoord) {
    vec2 uv = fragCoord / iResolution.xy;

    // Sample the terminal texture
    vec4 color = texture(iChannel0, uv);

    // Dim unfocused panes
    if (iFocus == 0) {
        color.rgb *= 0.6;  // 60% brightness when not focused
    }

    // Left edge indicator bar (4 pixels wide)
    float barWidth = 4.0 / iResolution.x;
    if (uv.x < barWidth) {
        // Use cursor color as the indicator
        // Hook scripts set cursor color to signal state:
        //   Green (#00ff00) = done
        //   Yellow (#ffff00) = waiting
        //   Blue (#0088ff) = working
        //   Red (#ff0000) = error
        color = iCursorColor;

        // Pulse animation for non-green states
        if (iCursorColor.g < 0.9 || iCursorColor.r > 0.1) {
            float pulse = 0.7 + 0.3 * sin(iTime * 3.0);
            color.rgb *= pulse;
        }
    }

    fragColor = color;
}
```

Enable the shader in Ghostty config:

```
# ~/.config/ghostty/config
custom-shader = ~/.config/ghostty/shaders/agent-state.glsl
custom-shader-animation = always
```

Set cursor color from a hook to signal state:

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/set-cursor-color.sh
# Changes the terminal cursor color to signal agent state via Ghostty shader

PAYLOAD=$(cat)
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name')

set_cursor_color() {
  local color="$1"
  # OSC 12 sets cursor color
  if [ -n "$TMUX" ]; then
    printf '\ePtmux;\e\e]12;%s\a\e\\' "$color"
  else
    printf '\e]12;%s\a' "$color"
  fi
}

case "$EVENT" in
  UserPromptSubmit|PreToolUse)
    set_cursor_color "#0088ff"  # Blue = working
    ;;
  Stop)
    set_cursor_color "#00ff00"  # Green = done
    ;;
  Notification)
    NTYPE=$(echo "$PAYLOAD" | jq -r '.notification_type // ""')
    case "$NTYPE" in
      permission_prompt) set_cursor_color "#ffff00" ;;  # Yellow = waiting
      *)                 set_cursor_color "#0088ff" ;;  # Blue = working
    esac
    ;;
  StopFailure)
    set_cursor_color "#ff0000"  # Red = error
    ;;
esac
```

> **Key insight:** This is a creative hack, not a supported integration. Ghostty does not have an API to pass arbitrary data to shaders. The cursor color is the only dynamic value a shell script can control (via OSC 12) that is visible to the shader (via `iCursorColor`). It works, but it also changes the actual cursor color. If you use cursor color for other purposes, this will conflict.

**Gotchas:**

- `custom-shader-animation = always` keeps the animation loop running even when idle. This uses more GPU/CPU. Consider `custom-shader-animation = true` (only when focused) if you have many terminals open.
- The shader runs on every frame. Keep it simple — complex shaders will impact performance.
- Cursor color changes via OSC 12 persist until explicitly reset. If the agent crashes without firing a hook, the cursor stays the wrong color.
- This approach only works in Ghostty. iTerm2, Kitty, and other terminals have different shader/styling mechanisms.

---

## Small Examples

### Example 1: tmux hook that auto-focuses a pane when its agent needs input

```bash
# In ~/.tmux.conf
# Watch for the "waiting" state file and focus that pane
set-hook -g alert-activity 'run-shell "
  for f in ~/.cache/agent-status/*.json; do
    state=$(jq -r .state \"$f\" 2>/dev/null)
    session=$(jq -r .session \"$f\" 2>/dev/null)
    if [ \"$state\" = \"waiting\" ]; then
      tmux select-window -t \"$session\"
      break
    fi
  done
"'
```

### Example 2: Ghostty AppleScript to find a terminal by working directory

```applescript
-- Find and focus the terminal working on a specific project
tell application "Ghostty"
    repeat with w in windows
        repeat with t in tabs of w
            repeat with term in terminals of t
                if working directory of term contains "project-alpha" then
                    focus term
                    activate window w
                    return
                end if
            end repeat
        end repeat
    end repeat
end tell
```

### Example 3: Send a tmux notification with a sound

```bash
#!/usr/bin/env bash
# Play a sound when an agent finishes (macOS)
PAYLOAD=$(cat)
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name')

if [ "$EVENT" = "Stop" ]; then
  # Play the system "Glass" sound
  afplay /System/Library/Sounds/Glass.aiff &
  # Also update tmux display
  tmux display-message "Agent finished in #{session_name}"
fi
```

### Example 4: tmux capture-pane to extract the last agent response

```bash
#!/usr/bin/env bash
# Capture the last N lines of a pane and extract meaningful content
PANE_TARGET="${1:-}"

if [ -z "$PANE_TARGET" ]; then
  echo "Usage: capture-response.sh <pane-target>"
  exit 1
fi

# Capture visible pane content
CONTENT=$(tmux capture-pane -t "$PANE_TARGET" -p -S -50)

# Extract text between the last two prompts (Claude Code uses > as prompt)
RESPONSE=$(echo "$CONTENT" | awk '
  /^>/ { buf=""; collecting=1; next }
  collecting { buf = buf "\n" $0 }
  END { print buf }
')

echo "$RESPONSE"
```

### Example 5: Reset all agent state files (cleanup script)

```bash
#!/usr/bin/env bash
# ~/.tmux/scripts/reset-agents.sh
# Clear all agent state — useful after a session crash

STATE_DIR="$HOME/.cache/agent-status"

if [ -d "$STATE_DIR" ]; then
  rm -f "$STATE_DIR"/*.json
  echo "Cleared $(ls "$STATE_DIR" 2>/dev/null | wc -l) agent state files"
else
  echo "No state directory found"
fi

# Reset HUD to green
cat > ~/.atlas/status.json <<EOF
{
  "status": "green",
  "source": "manual-reset",
  "message": "All agents cleared",
  "updated": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "details": []
}
EOF

echo "HUD reset to green"
```

### Example 6: tmux key binding to cycle through agent panes

```bash
# In ~/.tmux.conf
# Prefix + n: jump to next pane with an active agent
bind-key n run-shell '
  CURRENT=$(tmux display-message -p "#{pane_id}")
  FOUND=0
  for pane in $(tmux list-panes -a -F "#{pane_id}"); do
    if [ "$FOUND" = "1" ]; then
      tmux select-pane -t "$pane"
      exit 0
    fi
    if [ "$pane" = "$CURRENT" ]; then
      FOUND=1
    fi
  done
  # Wrap around to first pane
  FIRST=$(tmux list-panes -a -F "#{pane_id}" | head -1)
  tmux select-pane -t "$FIRST"
'
```

### Example 7: Write context window usage to tmux status

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/context-usage.sh
# Extract context window usage from the hook payload

PAYLOAD=$(cat)
SESSION_ID=$(echo "$PAYLOAD" | jq -r '.session_id')

# Read from the transcript to find context usage
# Claude Code includes context info in its internal state
TRANSCRIPT=$(echo "$PAYLOAD" | jq -r '.transcript_path')

if [ -n "$TMUX" ]; then
  SESSION_NAME=$(tmux display-message -p '#{session_name}')
  CTX_FILE="/tmp/claude-ctx/${SESSION_NAME}.txt"
  mkdir -p /tmp/claude-ctx

  # Extract the last context percentage from transcript metadata
  CTX_PCT=$(jq -r '
    select(.type == "system" and .context_window) |
    .context_window.used_percentage // empty
  ' "$TRANSCRIPT" 2>/dev/null | tail -1)

  if [ -n "$CTX_PCT" ]; then
    echo "${CTX_PCT}%" > "$CTX_FILE"
  fi
fi
```

### Example 8: Ghostty config for agent-friendly defaults

```
# ~/.config/ghostty/config — optimized for AI agent workflows

# Performance
font-size = 13
theme = catppuccin-mocha

# Notifications (critical for agent feedback)
desktop-notifications = true
notify-on-command-finish = unfocused
notify-on-command-finish-action = notify
notify-on-command-finish-after = 10

# Shell integration (required for notify-on-command-finish)
shell-integration = detect

# Window management
window-save-state = always
window-padding-x = 4
window-padding-y = 4

# Key tables for tmux-like modal workflow
keybind = ctrl+a>n=new_window
keybind = ctrl+a>s=new_split:right
keybind = ctrl+a>v=new_split:down
keybind = ctrl+a>w=close_surface
keybind = ctrl+a>1=goto_tab:1
keybind = ctrl+a>2=goto_tab:2
keybind = ctrl+a>3=goto_tab:3
keybind = ctrl+a>4=goto_tab:4

# Scrollback dump for debugging
keybind = ctrl+a>d=write_scrollback_file:open

# AppleScript (macOS) — enabled by default, disable if not needed
# macos-applescript = false

# Custom shader for visual agent state
# custom-shader = ~/.config/ghostty/shaders/agent-state.glsl
# custom-shader-animation = always
```

### Example 9: Combined hook handler (all events, one script)

```bash
#!/usr/bin/env bash
# ~/.claude/hooks/unified-handler.sh
# Single script handling all hook events
# Delegates to sub-functions for state, notifications, HUD, and summaries

PAYLOAD=$(cat)
EVENT=$(echo "$PAYLOAD" | jq -r '.hook_event_name')

# Source sub-handlers
HOOKS_DIR="$HOME/.claude/hooks"

# Always update state
echo "$PAYLOAD" | "$HOOKS_DIR/agent-state.sh" 2>/dev/null &

# Always update HUD
echo "$PAYLOAD" | "$HOOKS_DIR/hud-status.sh" 2>/dev/null &

# Notify on terminal events
case "$EVENT" in
  Stop|StopFailure|Notification)
    echo "$PAYLOAD" | "$HOOKS_DIR/notify-desktop.sh" 2>/dev/null &
    ;;
esac

# Summarize on Stop
if [ "$EVENT" = "Stop" ]; then
  echo "$PAYLOAD" | "$HOOKS_DIR/summarize.sh" 2>/dev/null &
fi

# Update cursor color for shader
echo "$PAYLOAD" | "$HOOKS_DIR/set-cursor-color.sh" 2>/dev/null &

# Wait for background jobs (within timeout)
wait
```

Configure once in settings:

```json
{
  "hooks": {
    "UserPromptSubmit": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/unified-handler.sh", "timeout": 30}]}],
    "Stop": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/unified-handler.sh", "timeout": 30}]}],
    "Notification": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/unified-handler.sh", "timeout": 30}]}],
    "StopFailure": [{"hooks": [{"type": "command", "command": "~/.claude/hooks/unified-handler.sh", "timeout": 30}]}]
  }
}
```

---

## Comparisons

### Terminal Emulators for Automation

| Feature | Ghostty | iTerm2 | Kitty | WezTerm | Alacritty |
|---------|---------|--------|-------|---------|-----------|
| **Automation API** | AppleScript (macOS, preview) | AppleScript + Python API | Remote control protocol + kittens | Lua scripting API | None |
| **tmux integration** | Standard (run tmux inside) | Control mode (`-CC`) — native tabs | Standard | Standard (mux server built-in) | Standard |
| **OSC 9 notifications** | Yes | Yes | No (uses OSC 99) | Yes | No |
| **OSC 777 notifications** | Yes | No | No | No | No |
| **OSC 99 notifications** | No | No | Yes | No | No |
| **Custom shaders** | GLSL with rich uniforms | No | Yes (GLSL) | GLSL (WebGPU) | No |
| **Shell integration** | OSC 133 (detect) | Proprietary + OSC 133 | No | OSC 133 | No |
| **Focus events** | Yes | Yes | Yes | Yes | No |
| **GPU acceleration** | Yes (Metal/Vulkan) | Metal | OpenGL | WebGPU/Metal | OpenGL |
| **Config format** | Key-value file | GUI + plist | Key-value (kitty.conf) | Lua | YAML/TOML |
| **Cross-platform** | macOS, Linux | macOS only | macOS, Linux, BSD | macOS, Linux, Windows | macOS, Linux, Windows, BSD |
| **External command trigger** | Via keybind actions | Via scripts/triggers | Via `kitten @` | Via Lua events | No |

**Recommendations:**

- **Best for tmux automation:** iTerm2 wins with control mode (`-CC`), which renders tmux panes as native tabs. No other terminal does this. If your workflow is tmux-centric and you are on macOS, iTerm2's integration is unmatched.
- **Best for visual feedback:** Ghostty wins with custom shaders that can react to focus state and cursor color. Kitty also supports GLSL but with fewer uniforms exposed.
- **Best for scripting breadth:** Kitty wins with its `kitten` system — Python-based extensions that can do file transfer, diff viewing, SSH with automatic terminfo, and custom tools. iTerm2's Python API is also powerful but macOS-only.
- **Best for cross-platform simplicity:** WezTerm with its built-in multiplexer and Lua scripting. No tmux needed — WezTerm has its own mux protocol.
- **Best for speed with no frills:** Alacritty. Zero automation features, fastest rendering. Use it if you run tmux for everything and need nothing from the terminal.
- **Best all-around for AI agents on macOS:** Ghostty + tmux. Fast rendering, native notifications that work out of the box, AppleScript for initial setup, and clean passthrough for agent hooks. The lack of control mode is the main trade-off vs iTerm2.

### tmux Agent Management Tools

| Tool | Type | Agents Supported | Key Feature | Status Tracking | Multi-machine |
|------|------|-----------------|-------------|----------------|---------------|
| [tmux-agent-indicator](https://github.com/accessd/tmux-agent-indicator) | tmux plugin | Claude, Codex, OpenCode, custom | Pane border colors + status icons | Hook-based + process detection | No |
| [tmux-agent-status](https://github.com/samleeney/tmux-agent-status) | tmux plugin | Claude, Codex, any custom agent | Session switcher with grouping | Hook-based + file-based | Yes (SSH) |
| [dmux](https://dmux.ai) | CLI/TUI | 11+ agents (Claude, Codex, Gemini, etc.) | Git worktree isolation per task | Built-in | No |
| [NTM](https://github.com/Dicklesworthstone/ntm) | CLI/TUI | Claude, Codex, Gemini | Broadcast prompts, conflict detection | Dashboard | No |
| [claude-tmux](https://github.com/nielsgroen/claude-tmux) | TUI (Rust) | Claude Code | Popup session manager with live preview | Process detection | No |
| [agent-deck](https://github.com/asheshgoplani/agent-deck) | TUI | Claude, Gemini, OpenCode, Codex + more | Terminal session management | Process-based | No |
| [agtx](https://github.com/fynnfluegge/agtx) | CLI/TUI | Claude, Codex, Gemini, OpenCode, Cursor | Kanban board + orchestrator agent | Spec-driven | No |
| [Codeman](https://github.com/Ark0N/Codeman) | Web UI | Claude, OpenCode | Browser-based tmux session manager | WebSocket | No |
| DIY (this article) | Scripts | Any | Full control, no dependencies | File-based | Yes (file sync) |

**Recommendations:**

- **Quickest setup:** `tmux-agent-indicator` — one install command, hook auto-configuration, visual feedback in seconds.
- **Best for remote agents:** `tmux-agent-status` — SSH sync for GPU servers and cloud VMs. Groups sessions by state with a switcher.
- **Best for parallel development:** `dmux` — git worktree isolation means agents never step on each other's files. Supports 11 agents.
- **Best for large teams:** `NTM` — broadcast prompts to multiple agents, conflict detection when two agents edit the same file.
- **Best for Claude-only workflows:** `claude-tmux` — Rust-based, fast popup, integrates with git worktrees and PRs.
- **Best for learning/customization:** DIY with the patterns in this article. You understand every line, can extend freely, no plugin dependencies.

---

## Anti-Patterns

| Don't | Do Instead | Why |
|-------|-----------|-----|
| Poll panes with `tmux capture-pane` in a loop | Use Claude Code hooks to write state files | Polling wastes CPU, misses events between polls, and scales badly with pane count |
| Use `tmux -CC` control mode with Ghostty | Use standard tmux inside Ghostty | Ghostty does not support control mode. Only iTerm2 does. Your tmux sessions will not render correctly |
| Send notifications without checking `$TMUX` | Always check for tmux and wrap in DCS passthrough | Raw OSC sequences are intercepted by tmux and never reach the outer terminal |
| Run `claude -p` inside a Claude Code hook without unsetting `CLAUDECODE` | Use `env -u CLAUDECODE claude -p ...` | The `CLAUDECODE` environment variable blocks nested invocations, causing silent failures |
| Use `pipe-pane` for agents that support hooks | Use hooks for hook-capable agents, `pipe-pane` for those that do not | Hook payloads have structured JSON with session ID, event type, and transcript path. `pipe-pane` gives you raw ANSI-cluttered output you must parse |
| Write the HUD status file directly (non-atomic) | Write to a temp file, then `mv` it atomically | The HUD app may read a half-written file, parse it as invalid JSON, and silently show stale data |
| Rely on Ghostty AppleScript for session persistence | Use tmux for persistence, Ghostty AppleScript for layout setup | Ghostty AppleScript windows do not survive application restarts. tmux sessions persist |
| Set `custom-shader-animation = always` with many terminals | Use `true` (focused-only) or limit to specific profiles | `always` runs the shader animation loop on every terminal surface, even unfocused ones. With 8+ terminals this is measurable GPU load |
| Use different state file formats per agent | Use a single JSON schema for all agent state files | Multiple tools reading the same state directory need a consistent format. Define it once and stick to it |
| Nest `jq` calls inside `jq` filters | Use a single `jq` pipeline with proper type checking | Nested calls are slower and fail silently. Check types (`type == "string"`, `type == "array"`) within one pipeline |
| Set `status-interval 1` for real-time status | Use `status-interval 5` and accept ~5s latency | 1-second intervals run your status scripts 5x more often. For most agent workflows, 5-second updates are perfectly fine |
| Hardcode paths in hook scripts | Use `$HOME`, `$TMUX`, `$CLAUDE_PROJECT_DIR` | Hardcoded paths break when used by other users, in CI, or on different machines |
| Use OSC 99 (Kitty protocol) for cross-terminal compatibility | Use OSC 9 as the lowest common denominator | OSC 99 only works in Kitty. OSC 9 works in Ghostty, iTerm2, WezTerm, and Windows Terminal |

---

## References

### Official Documentation

- [Ghostty Documentation](https://ghostty.org/docs) — Complete docs for configuration, keybindings, and features
- [Ghostty AppleScript Documentation](https://ghostty.org/docs/features/applescript) — AppleScript API reference with object model and commands
- [Ghostty OSC 9 Documentation](https://ghostty.org/docs/vt/osc/9) — Desktop notification escape sequence format
- [Ghostty Keybind Action Reference](https://ghostty.org/docs/config/keybind/reference) — All available keybind actions
- [Ghostty Keybind Sequence Triggers](https://ghostty.org/docs/config/keybind/sequence) — Chained keybind syntax
- [Ghostty Configuration Reference](https://ghostty.org/docs/config/reference) — Full option reference including notifications and shaders
- [Ghostty 1.3.0 Release Notes](https://ghostty.org/docs/install/release-notes/1-3-0) — AppleScript, notify-on-command-finish, key tables
- [Ghostty 1.2.0 Release Notes](https://ghostty.org/docs/install/release-notes/1-2-0) — Custom shader cursor uniforms, write_screen_file
- [Claude Code Hooks Reference](https://code.claude.com/docs/en/hooks) — Complete hook event types, payloads, and configuration
- [Claude Code Terminal Configuration](https://code.claude.com/docs/en/terminal-config) — tmux passthrough, notification setup, terminal recommendations
- [tmux Manual Page](https://man7.org/linux/man-pages/man1/tmux.1.html) — Authoritative reference for all tmux commands and options
- [tmux Control Mode Wiki](https://github.com/tmux/tmux/wiki/Control-Mode) — Protocol format for control mode (-CC)
- [tmux Advanced Use Wiki](https://github.com/tmux/tmux/wiki/Advanced-Use) — pipe-pane, capture-pane, and hook examples
- [iTerm2 tmux Integration](https://iterm2.com/documentation-tmux-integration.html) — Control mode (-CC) native tab integration
- [Kitty Remote Control](https://sw.kovidgoyal.net/kitty/remote-control/) — kitten @ commands and automation
- [Kitty Desktop Notifications (OSC 99)](https://sw.kovidgoyal.net/kitty/desktop-notifications/) — Kitty's extensible notification protocol

### Tools and Plugins

- [tmux-agent-indicator](https://github.com/accessd/tmux-agent-indicator) — tmux plugin for AI agent visual state tracking (pane borders, status icons)
- [tmux-agent-status](https://github.com/samleeney/tmux-agent-status) — Session-level agent status with remote machine support
- [dmux](https://dmux.ai) — Dev agent multiplexer with git worktree isolation, supports 11+ agents
- [NTM (Named Tmux Manager)](https://github.com/Dicklesworthstone/ntm) — Multi-agent coordination with broadcast prompts and conflict detection
- [claude-tmux](https://github.com/nielsgroen/claude-tmux) — Rust-based tmux popup for managing Claude Code sessions
- [agent-deck](https://github.com/asheshgoplani/agent-deck) — Terminal session manager TUI for multi-agent workflows
- [agtx](https://github.com/fynnfluegge/agtx) — Kanban-style multi-agent tmux orchestrator
- [Codeman](https://github.com/Ark0N/Codeman) — Web UI for managing Claude Code and OpenCode in tmux
- [agent-viewer](https://github.com/hallucinogen/agent-viewer) — Kanban board for managing Claude Code agents in tmux
- [tmux-logging](https://github.com/tmux-plugins/tmux-logging) — tmux plugin for session logging with ANSI stripping
- [awesome-tmux](https://github.com/rothgar/awesome-tmux) — Curated list of tmux resources
- [awesome-claude-code](https://github.com/hesreallyhim/awesome-claude-code) — Curated list of Claude Code plugins, hooks, and integrations
- [Ghostty GitHub Repository](https://github.com/ghostty-org/ghostty) — Ghostty source code and discussions
- [ghostty-cursor-shaders](https://github.com/sahaj-b/ghostty-cursor-shaders) — Custom cursor trail and pulse shaders for Ghostty

### Blog Posts and Articles

- [Live AI Session Summaries in a Two-Line tmux Status Bar (Quickchat AI)](https://quickchat.ai/post/tmux-session-summaries-for-parallel-ai-agents) — Detailed walkthrough of hook-based AI summaries in tmux
- [Notification System for Tmux and Claude Code (Alexandre Quemy)](https://quemy.info/2025-08-04-notification-system-tmux-claude.html) — Notification architecture for tmux + Claude
- [Desktop Notifications for Claude Code (kane.mx)](https://kane.mx/posts/2025/claude-code-notification-hooks/) — OSC-based notification hooks for Claude Code
- [Claude Code + Tmux: How I Got Notifications Working (software-dc.com)](https://software-dc.com/blog/4-claude-code-tmux-how-i-got-notifications-working) — Practical notification setup guide
- [Claude Code from the Beach: Remote Setup with mosh, tmux, and ntfy (rogs)](https://rogs.me/2026/02/claude-code-from-the-beach-my-remote-coding-setup-with-mosh-tmux-and-ntfy/) — Remote agent workflow with push notifications
- [Automating 4 macOS Terminals for Claude Code (dev.to)](https://dev.to/thinkerjack/automating-4-macos-terminals-for-claude-code-applescript-ghosttys-e-trap-and-warps-missing-3nk1) — AppleScript automation across terminal emulators
- [Choosing a Terminal on macOS 2025 (Chris Evans)](https://medium.com/@dynamicy/choosing-a-terminal-on-macos-2025-iterm2-vs-ghostty-vs-wezterm-vs-kitty-vs-alacritty-d6a5e42fd8b3) — Feature comparison of modern terminals
- [Ghostty vs iTerm2 (soloterm.com)](https://soloterm.com/ghostty-vs-iterm) — Detailed comparison of automation capabilities
- [Switching from Ghostty to Kitty (linkarzu)](https://linkarzu.com/posts/terminals/ghostty-to-kitty/) — Honest comparison of missing features
- [Ghostty Focus and Blur Shaders (Martin Emde)](https://martinemde.com/blog/ghostty-focus-shaders) — Using iFocus and iTimeFocus uniforms
- [Watch Claude Code Agents Work Side by Side (Karan Singh)](https://ksingh7.medium.com/watch-claude-code-agents-work-side-by-side-a-tmux-setup-guide-1ef3ba1531c4) — tmux split-pane agent monitoring setup
- [Claude Code Multi-Agent tmux Setup (Dariusz Parys)](https://www.dariuszparys.com/claude-code-multi-agent-tmux-setup/) — Production multi-agent workflow
- [Agent Forking in AI Coding Sessions with tmux (kau.sh)](https://kau.sh/blog/agent-forking/) — Forking subagents in tmux for parallel work
- [I Built a Desktop App to Supercharge My TMUX + Claude Code Workflow (dev.to)](https://dev.to/joe-re/i-built-a-desktop-app-to-supercharge-my-tmux-claude-code-workflow-521m) — Tauri-based desktop agent dashboard
- [Claude Code Hooks + tmux Pane Auto-Focus (GitHub Gist)](https://gist.github.com/grmkris/85a9b8b0cbdffaa752d2fcc4ae619dcd) — Hook configuration for automatic pane switching
- [Show HN: Visual State Tracking for AI Agents in tmux](https://news.ycombinator.com/item?id=47022633) — Hacker News discussion on tmux-agent-indicator
- [tmux Hooks Mintlify Documentation](https://tmux-tmux.mintlify.app/configuration/hooks) — Clean reference for all tmux hook events
- [Claude Code Hooks Mastery (GitHub)](https://github.com/disler/claude-code-hooks-mastery) — Comprehensive examples of hook patterns
- [Claude Code Hooks Multi-Agent Observability (GitHub)](https://github.com/disler/claude-code-hooks-multi-agent-observability) — Real-time monitoring through hook event tracking

### Discussions and Issues

- [Ghostty OSC 99 Desktop Notifications Discussion](https://github.com/ghostty-org/ghostty/discussions/4405) — Request for Kitty-style notification support
- [Ghostty AppleScript Discussion](https://github.com/ghostty-org/ghostty/discussions/10201) — Community discussion on automation use cases
- [Ghostty Command Finished Notifications Discussion](https://github.com/ghostty-org/ghostty/discussions/3555) — Original request for notification on command finish
- [Ghostty Custom Shader Cursor Uniforms Discussion](https://github.com/ghostty-org/ghostty/discussions/6901) — Discussion of cursor-related shader uniforms
- [tmux Complete List of Hooks (Issue #1083)](https://github.com/tmux/tmux/issues/1083) — Comprehensive hook enumeration
- [Claude Code Terminal Notifications Feature Request](https://github.com/anthropics/claude-code/issues/19976) — Discussion on notification support inside tmux
- [Claude Code OSC Escape Sequences Bug](https://github.com/anthropics/claude-code/issues/28338) — VSCode terminal notification issues
- [Claude Code Agent Teams tmux Pane Border Bug](https://github.com/anthropics/claude-code/issues/30145) — Pane border color issues on tmux < 3.3
