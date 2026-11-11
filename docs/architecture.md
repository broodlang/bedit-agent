# bedit-agent: how it is built

bedit-agent runs coding agents inside bedit. It has three parts:

- [agentkit](../../agentkit), which knows nothing about editors: how to talk to an agent and
  what it reports;
- bedit's own registries, which bedit-agent plugs into;
- the modules below, kept small and layered. Each module uses only the ones above it in the
  list, so there are no import cycles. The entry module (`src/bedit-agent.blsp`) loads them in
  that order.

## The flow of one tool call (chat front end)

```
 claude CLI ──MCP/HTTP──▶ tool server (mcp-server, one per session, loopback + token)
                              │  handler process: tools.blsp's tool, wrapped by sessions/guard
                              │
                              ├─ ask-loop {:op :permission}  ─▶ bridge ─▶ permission
                              │      allowed now? answer at once : queue, ask you, answer later
                              ├─ the tool runs (in the handler process; asks the loop for buffers)
                              │      ask-loop {:op :read/:edit/…} ─▶ bridge ─▶ edits / workspace
                              └─ ask-loop {:op :ran} ─▶ bridge ─▶ feedback
                                     written under the call in the transcript; feedback rows'
                                     notes appended to what the agent reads
 claude CLI ──stream-json──▶ agentkit worker ──[:agent worker kind payload]──▶ chat/ed-ev-agent
                                                                         ─▶ transcript
```

The editor loop never waits on the agent. Every request a handler makes gets exactly one reply
(`workspace/reply`), and that includes a request that fails.

## The modules, lowest first

| module | what it owns |
|---|---|
| `profiles` | profile layers (built-in < user < project), `:extends`, the keys a project may only narrow |
| `accounts` | the Claude config dirs under home, labelled with their account |
| `project-state` | what is kept per project in bedit's state: the account, trust, past sessions |
| `tools` | the tool catalogue (data: name, schema, risk, `:paths`, handler); `ask-loop`; reading the disk off the loop |
| `sessions` | the session records on the model; the tool server; the guard; the agent's process (start, restart, stop) |
| `workspace` | the loop's view of files: `reply`, `relative`, `buffer-index-for`, `current-text`, `diagnostics-of` |
| `diffs` | diff lines and unified diffs, coloured, shown beside you |
| `transcript` | writing a chat's transcript: inserting above the prompt, events, a call's details under its row, carrying windows |
| `edits` | `edit` / `write` through buffers (splices, saved on the loop), what changed (`show_changes`, `show_diff`) |
| `permission` | the question queue: decided at once, or asked after a pause; `y n a d C-g`; declining a session's questions |
| `feedback` | `:ran`: the call's details into the transcript; the feedback registry |
| `bridge` | the one event handler for tool requests; the read-only requests |
| `chat` | the chat mode: sending (queued behind a turn), the session's events, its commands and keymap |
| `start` | starting a session in either front end; the account question; a project's profiles and live sessions |
| `commands` | `M-x claude`, `claude-tui`, `agent*`; what ends a session when its buffer goes; the exit query |
| `resume` | `M-x agent-resume`: a past chat replayed from Claude Code's log and continued |
| `context` | `agent-send-region`/`-buffer`/`-file`/`-diagnostics` |
| `menu` | the session menu (transient): permission toggles, model, effort, session commands |
| `chip` | the mode-line chip and its one-second beat while a session works |

## The state, and where it lives

- **On the model.** `:agent-sessions` (id → session), `:agent-session-count`, `:agent-asks` (the
  queued questions), `:agent-question` (the one shown), `:agent-last-key` (for the typing pause).
  A session record holds:
  - `:id :root :profile :frontend :buffer :status :busy`;
  - `:server` (`{:pid :url :token}`), `:config-file`, `:config-dir`;
  - `:worker` (agentkit's pid) and `:claude-session`;
  - `:always` (`{:tools :commands}`), `:originals` (path → text before the first edit), `:queued`;
  - `:title :turn-started :last-turn :limits :model :restart-pending`.
- **On a chat buffer.** `:agent-session` (its id), `:transcript-end`, `:input-start`,
  `:agent-pending` (calls awaiting details), `:agent-detailed`, `:agent-streamed`, the
  `:agent-face` text property, and the `[:tool id]` markers.
- **On disk.**
  - `state/agent-projects.blsp`: per project, the account, trust and past sessions.
  - `state/agent/mcp-*.json`: one per live session, mode 0600, removed when the session ends.
  - The profiles files.
  - Claude Code's own session logs, which `resume` reads.

## bedit's registries it plugs into

All of these are claimed, so removing the package takes them all back:

- event handlers: `:agent` `:agent-tool` `:agent-tui-ready` `:agent-accounts` `:agent-history`;
- the `:agent-asks` post-step and timer, and the `:agent-keys` input step;
- the `:agent-sessions` slot keeper and the `:agent` exit query;
- the `:agent-chat` text faces and buffer type;
- the `:agent` mode-line segment and the `:agent-tick` timer;
- M-x commands (`defcommand`);
- `:bedit-agent/feedback` rows.

## Adding things

- **A tool:** add an entry to `tools/all-tools`, with a `:risk` and, if it takes a path, its
  `:paths`. If it needs the editor, add an `:op` to `bridge/answer-request`, answered by the
  module that owns that concern. Add a line to the instructions in `sessions/instructions` if
  the agent should know when to use it.
- **A backend:** it reports `agentkit/events`. The chat folds those and never names the
  backend. Starting one is `sessions/start-worker`'s job.
- **A front end:** start it from `start/start-session` (`:frontend`); the tools, permission
  and feedback are already shared.
- **Feedback:** `(feedback/register-feedback :name 'pkg/fn)`. Nothing else changes.

## Tests

`nest test` runs 58 tests with no network and no `claude`. Builds go through a fake CLI
(`../agentkit/tests/fixtures/fake-claude`) and models through `ed-init`.
`slow-tests/claude_live.blsp` runs one real haiku turn on your subscription, end to end.
