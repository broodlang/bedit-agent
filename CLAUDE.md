# CLAUDE.md: working on bedit-agent

bedit-agent is a bedit plugin package (`:enhances bedit`). It runs coding agents in bedit: a
chat front end where every tool is bedit's, and Claude Code's own TUI with bedit's tools added.
It is built on [agentkit](https://github.com/broodlang/agentkit), which knows nothing about editors.

- **Read `docs/architecture.md` first.** It covers the module layering (each module uses only
  the ones above it), the flow of a tool call, where state lives and how to add a tool, a
  backend or feedback.
- **bedit's conventions apply.** See `../bedit/CLAUDE.md`. In particular:
  - commands are `model -> model`;
  - registries are given SYMBOLS, never functions, so they hot-swap and retire with the package;
  - no abbreviated names;
  - typed parameters rather than `(sig …)` for a single-clause `defn`;
  - every public function has a docstring, and its `;=>` examples run.
- **Subscription only.** Never pass an API key, never weaken the three checks in
  `agentkit/claude-code`, and never put a token on a command line.
- **The editor loop never waits on the agent.** Work that might block (the disk, a process,
  the login check) runs in a handler or a spawned process, which sends the loop an event.
- **Every tool request gets exactly one reply** (`workspace/reply`), failures included.
- **Tests:** `nest test` (no network). `AGENT_LIVE_ROOT=/tmp/x nest run
  slow-tests/claude_live.blsp` spends one small real turn; run it after changing the
  tool/session path.
- **It needs a brood with `mcp-server`** (brood ADR-711). Until that is on brood's `main`, use
  the worktree's nest.
