# bedit-agent

Coding agents in [bedit](https://github.com/broodlang/bedit), as a plugin package
(`:enhances bedit`). The first backend is **Claude Code on your subscription**. It never
uses an API key: the `claude` CLI runs with every API-key variable removed, the login is
checked in the session's own project before anything is sent (so a project's Claude settings
cannot switch the provider), and the session's first event must say no API key is in use
([agentkit](https://github.com/broodlang/agentkit)).

## Two front ends, one set of profiles and tools

| | `M-x claude` (chat) | `M-x claude-tui` |
|---|---|---|
| What you see | a bedit buffer: the transcript, your prompt below it | Claude Code's own interface, in a terminal buffer |
| The agent's tools | **bedit's only**: every call goes through bedit | Claude Code's own, plus bedit's |
| Permission | bedit asks in `*agent: approvals*` | Claude Code asks for its tools; bedit asks for its own |
| What ran | written under each call in the transcript, with the diff | an edit is said in the echo area |

Each opens this project's session of that kind, or starts one. `M-x agent-new` starts another
under any profile; `M-x agent-switch` goes to any running session in any project.

### bedit's tools

| tool | what it does | asks unless the profile `:allow`s |
|---|---|---|
| `read` | a file as bedit has it: an open buffer's unsaved text, else the disk (decoded as bedit decodes it) | `:read` |
| `list_buffers` | what is open, what is unsaved, what you are looking at | `:read` |
| `edit` / `write` | change a file through its bedit buffer (one undo step there, your windows left where they are) and save it | `:edit` |
| `bash` | `sh -c` in the project root | `:run` (or `:allow-commands`) |
| `grep` / `glob` | search the project (ripgrep) | `:read` |
| `diagnostics` | what bedit's language servers and checkers show, current only | `:read` |
| `show` | open a file for you, at a line, in another window | `:read` |
| `show_changes` | show you a diff of what **the agent** changed this session (against each file before its first edit) | `:read` |
| `show_diff` | show you the uncommitted changes against git's HEAD | `:read` |

Anything that reaches **outside the project** (and the profile's `:add-dirs`) asks, whatever the
profile says. Ask "what did you change? show me" and the agent calls `show_changes`.

## The chat buffer

```
Claude Code · claude-opus-5-5 · subscription · profile dev · ~/.claude-work
» make the tests pass
I'll run them first.
  ⏺ bash nest test
    ⎿ exit 1
      3 failed
  ⏺ edit src/pool.blsp
    ✓ allowed
    ⎿ edited src/pool.blsp:713 (1 replacement)
      - (defn- ed-ensure-file-buffer (m path)
      + (defn ed-ensure-file-buffer (m path)
── 41k context · 12s ──

❯ _
```

- **RET** at the prompt sends. While the agent is still working, your message is queued (`» … (queued)`) and goes the moment the turn ends. On a `path:line` in the transcript it opens the file there. Anywhere else it returns to the prompt.
- **C-j** adds a newline. **C-a** goes to the start of your message, after the prompt.
- **M-p** / **M-n** / **C-r** recall what you sent.
- **C-c C-c** interrupts the turn. **C-c C-k** ends the session. Write in an ended chat and it goes on: its tools and agent start again, the conversation resumed.
- **C-c C-a** goes to the approvals. **C-c C-o** opens the session menu.
- **M-x agent-resume** goes on with an earlier chat of this project. Each one is listed by the first message you sent it and when you last used it, and its transcript is replayed from Claude Code's own log.
- The cursor sticks to the prompt as text streams in above it, and a window at the prompt follows the tail as a compile buffer does, whether or not you are in it. Move up into the transcript and you stay where you are.
- The transcript is bedit's: only your message is editable.
- The agent writes Markdown, and it is coloured as Markdown, fenced code included. The transcript's own lines have their own faces.
- Each call's result, and your answer to it, is written under its own row, even when the agent makes several calls before any answers.
- A turn that ends while no window shows its chat is said in the echo area.

### Approvals

A call the profile doesn't allow, and anything that reaches outside the project, waits in
`*agent: approvals*`. It opens beside you without taking the cursor; the echo area gets one line
and the chip turns amber. Each request shows its session, the tool and file, a command in full,
and an edit or write as the diff it would make to the file as bedit has it, with the lines
around it. Go there with **C-c C-a** in a chat, a click on the chip, or `M-x agent-approvals`.
A key answers the request at point, else the first:

- **y** allows the call. **n** declines it.
- **a** allows it for the rest of the session. For `bash` that covers only this exact command.
- **o** opens the file it is about, at the text an edit would change.
- **RET** / **TAB** go to the next request. **q** puts the buffer away; the requests keep waiting.

A key typed anywhere else never answers a request. Once nothing waits, the window goes away the
way Emacs's quit-window takes a popup away: a window split off for it is deleted, and a window it
borrowed shows what it showed before. Ending or interrupting a session declines whatever it has
waiting.

### The session menu (C-c C-o, `M-x agent-menu`)

A transient over the session as it is now:

- **Without asking:** `e` edit files and `r` run commands, toggled. `c` forgets the "always" answers.
- **Model:** `m` cycles opus, sonnet, haiku and fable. `f` cycles the effort. A change applies at your next message: the agent restarts with the conversation resumed.
- **Session:** approvals, interrupt, end, new, switch, account, profiles, resume.

### The mode-line chip

In any buffer of a project with a session:

- `✦ dev · working 14s` while a turn runs, in the accent colour.
- `✦ dev · allow? 2 waiting` while requests wait, in amber. A click opens the approvals.
- `✦ dev · 41k · 5h 62%` when idle: the context used and the plan window.

Click it to go to the session. Hover it for the model, effort, account, plan windows and "always" answers.

## Handing the agent context

`M-x agent-send-region` (quoted with its `file:line`), `agent-send-buffer` (unsaved text
included), `agent-send-file` (`@path`) and `agent-send-diagnostics`. Each goes to the session
you are in, else the project's latest. In a chat it lands in your message for you to send. In
a TUI session it is pasted into Claude Code.

## Profiles, per project

`.bedit-agent.blsp` at a project's root, over `agent-profiles.blsp` in bedit's config dir,
over the built-in `claude` and `claude-tui`. Data, never evaluated:

```clojure
{:default "dev"
 :profiles {"dev"    {:extends "claude" :effort "high"
                      :allow [:read :edit]                  ; what never asks
                      :allow-commands ["nest test"]          ; bash commands that never ask
                      :instructions "Run the tests after a change."}
            "review" {:extends "claude" :model "sonnet"
                      :tools ["read" "grep" "glob" "diagnostics" "show" "show_changes"]
                      :instructions "Review only: never edit."}
            "tui"    {:extends "claude-tui" :permission-mode "acceptEdits"}}}
```

Keys:
- `:frontend` (`:chat` / `:tui`)
- `:model`, `:effort`
- `:instructions`
- `:tools`
- `:allow`, `:allow-commands`, `:add-dirs`
- `:config-dir` (another Claude account)
- for the TUI only: `:permission-mode` and `:claude-tools`

**A project's file may only narrow** until you trust it. The keys that let an agent do more,
reach further or run as someone else are left out, and said:
- `:allow`, `:allow-commands`, `:add-dirs`
- `:config-dir`
- `:permission-mode`, `:claude-tools`

`M-x agent-trust-project` trusts the project. Your own `agent-profiles.blsp` may always set them.

`M-x agent-profiles` opens the project's file, writing a starting one if missing.

## Which Claude account

`M-x agent-account` lists the Claude config directories under your home (`~/.claude`,
`~/.claude-*`), each with its account's email and organisation. The choice is saved per
project in bedit's state, never in the project. The first session in a project asks once, when
you have more than one account.

## Feedback is a registry

```clojure
(defn my-note (m request) ;; request: {:session :tool :arguments :result}
  (when (= (:tool request) "bash") "remember: CI runs on Linux only"))
(bedit-agent/bridge/register-feedback :my-note 'my-package/my-note)
```

Each row's answer is added to what the agent reads after a call. The built-in row tells it
what bedit already shows in a file it just edited.

## Lifecycle

- A session's tool server and agent belong to the editor loop that started them, so a loop that restarts leaves nothing running.
- Killing a session's buffer ends it.
- The MCP token is passed in a file only you can read, never on a command line.
- Quitting asks about sessions still working.

## Install

Both packages go in bedit's package dir (`~/.config/bedit/packages/`), with `agentkit` beside
`bedit-agent`. bedit puts a plugin's `:dependencies` that are installed beside it on the load
path first. Needs a brood with `mcp-server` (ADR-711).

```bash
nest test                                                     # 58 tests, no network, no claude
AGENT_LIVE_ROOT=/tmp/scratch nest run slow-tests/claude_live.blsp   # one real haiku turn
```
