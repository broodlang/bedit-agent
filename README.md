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
| Permission | bedit asks in the minibuffer, the request shown above the prompt | Claude Code asks for its tools; bedit asks for its own |
| What ran | written under each call in the transcript, with the diff | an edit is said in the echo area |

Each starts a new session of that kind in this project; one already running stays a buffer to
switch to. `C-c a` (`M-x agent`) goes to this project's latest session, or starts one; it is the
package's key, so bind it elsewhere, or away, in init.blsp's `:keys` (`"C-c a" nil`).
`M-x agent-new` starts one under any profile; `M-x agent-switch` goes to any running session in
any project.

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
- **C-/** undoes your typing and never what the agent wrote meanwhile: its text goes in as program output, which undo steps over.
- **C-c C-c** interrupts the turn. **C-c C-k** ends the session. Write in an ended chat and it goes on: its tools and agent start again, the conversation resumed.
- **C-x k** on a session's buffer asks first while its agent is working or waiting for your answer, or while Claude Code runs in a TUI session's terminal.
- A request is asked in the minibuffer (see Approvals). **C-c C-a** asks it now, put off or not. **C-c C-o** opens the session menu.
- **M-x agent-resume** goes on with an earlier chat of this project. Each one is listed by the first message you sent it and when you last used it, and its transcript is replayed from Claude Code's own log.
- The cursor sticks to the prompt as text streams in above it, and a window at the prompt follows the tail as a compile buffer does, whether or not you are in it. Move up into the transcript and you stay where you are.
- The transcript reads like a compile buffer: nothing is typed there and the cursor stays where it is. **n** / **p** go to the next and previous turn or tool call, and a letter is refused. At the prompt they are letters as usual.
- The agent writes Markdown, and it is coloured as Markdown, fenced code included. The transcript's own lines have their own faces.
- Each call's result, and your answer to it, is written under its own row, even when the agent makes several calls before any answers.
- A turn that ends while no window shows its chat is said in the echo area.

### Approvals

A call the profile doesn't allow, and anything that reaches outside the project, waits for your
answer, and the chip turns amber.

**A chat asks in the minibuffer**, one key, as Emacs's y-or-n questions do:
`bedit/dev: allow edit a.txt? (y, n, a always, o the file)`. No window opens. Meanwhile the
request is shown above the prompt: the tool and file, a command in full, or an edit or write as
the diff it would make to the file as bedit has it, with the lines around it.

- **y** allows the call. **n** declines it.
- **a** allows it for the rest of the session. For `bash` that covers only this exact command.
- **o** opens the file it is about, at the text an edit would change; it asks again when you come back.
- Any other key asks again. **C-g** puts it off: it waits, shown above the prompt, until **C-c C-a** asks it again.

It is asked only while you are in that chat with nothing typed at its prompt, and never in the
middle of a key sequence (`C-x …`, `C-u …`), so it never takes a key meant for your message or
another buffer. A request that arrives while you are elsewhere, up in the transcript or
mid-message is asked when you come back to the prompt or clear it, and the echo area says so.
The line under the request says whether it is being asked now. The answer is written under the
call's row and the next request, if any, is asked. A request settled while its question stands
(it expired, or its session ended) takes the question down with it.

**Answering never needs a buffer.** Wherever you are, a click on the amber chip asks the request
in the minibuffer: `bedit/dev: allow bash `make test`? (y, n, a always)`. A command is shown
itself, cut to one line. It is the oldest request of the session you are in, else the oldest of
all. **C-c C-a** in a chat does the same. A TUI session's request opens nothing; the echo area
names the command and the chip. While a question stands the caret sits after it in the echo
area and the buffer's caret is hollow, so it is clear where your keys go; once answered, the
prompt leaves the echo area.

**`M-x agent-approvals` lists every request**, from every session, with its whole command or
diff, in `*agent: approvals*` (the session menu has it too). There **y** / **n** / **a** / **o**
answer the request at point, else the first; **RET** / **TAB** go to the next request, and **q**
puts the buffer away (the requests keep waiting). Once nothing waits, its window goes away the
way Emacs's quit-window takes a popup away.

Ending or interrupting a session declines whatever it has waiting. A request nobody answers for
30 minutes is declined, said under its call, and leaves the queue; the agent is told nobody
answered.

### The session menu (C-c C-o, `M-x agent-menu`)

A transient over the session as it is now:

- **Without asking:** `e` edit files and `r` run commands, toggled. `c` forgets the "always" answers.
- **Model:** `m` cycles opus, sonnet, haiku and fable. `f` cycles the effort. A change applies at your next message: the agent restarts with the conversation resumed.
- **Session:** approvals, interrupt, end, new, switch, account, profiles, resume.

### The mode-line chip

In any buffer of a project with a session:

- `✦ dev · working 14s` while a turn runs, in the accent colour.
- `✦ dev · allow? 2 waiting` while requests wait, in amber. A click asks the oldest in the minibuffer, here.
- `✦ dev · 41k · 5h 62%` when idle: the context used and the plan window.

Otherwise a click goes to the session. Hover it for the model, effort, account, plan windows and "always" answers.

## Handing the agent context

`M-x agent-send-region` (quoted with its `file:line`), `agent-send-buffer` (unsaved text
included), `agent-send-file` (`@path`) and `agent-send-diagnostics`. Each goes to the session
you are in, else the project's latest, which opens beside you. In a chat it lands in your message for you to send. In
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

In bedit, `M-x package-install RET bedit-agent` (or `i` on it in `M-x package-list`). bedit
installs `agentkit`, the library it uses, from the hive beside it, loads it first, and binds
`C-c a`. `M-x package-remove` takes both away again, and the key with them. Needs brood
0.51.0 or later (`mcp-server`, ADR-711; `insert-program-output`, ADR-712).

Developing it, with bedit checked out beside it:

```bash
nest test                                                     # 58 tests, no network, no claude
AGENT_LIVE_ROOT=/tmp/scratch nest run slow-tests/claude_live.blsp   # one real haiku turn
```
