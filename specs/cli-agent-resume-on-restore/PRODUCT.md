# Resume third-party CLI agents on session restore — Product Spec

Ticket: none yet (kebab-name dir; rename to `specs/GH<issue-number>/` if filed upstream).
Figma: none provided.

## Summary

When Warp restores a previous session on relaunch, panes that were running a third-party CLI agent (e.g. Claude Code, Codex) automatically re-launch that agent against its own saved native session, so the agent comes back with its prior conversation already loaded instead of starting from scratch. Resuming reopens the saved session and returns control to the user; it does not submit a new prompt/turn to the model on the user's behalf.

## Problem

Warp already restores a great deal on relaunch — pane layout, working directory, scrollback blocks, and Warp's *own* agent conversations. But a third-party CLI agent running inside a pane is just a process; closing Warp kills it, and on restore the pane gets a fresh shell at the saved cwd with no connection to the agent's prior session. The agent's conversation still exists on disk (Claude writes `~/.claude/projects/<encoded_cwd>/<uuid>.jsonl` locally; Codex keeps its own local session store), and these CLIs already support resuming by id (`claude --resume <session_id>`). Warp also already captures the agent's native `session_id` from its hook events. The missing piece is wiring that captured id through restore so the user can resume the agent's prior context in one click rather than re-establishing it manually.

## Goals

- After restore, a pane that hosted a resumable CLI agent automatically relaunches that agent against its prior native session — the user gets their agent back where they left it with no manual step (herdr-style).
- Resume reuses the agent's own local transcript on this machine — no transcript download or rehydration (that is REMOTE-1373's cloud concern, out of scope here).
- The captured native `session_id` is passed to the agent CLI as argument data, never interpolated into a shell string.
- Resuming reopens the saved session without auto-submitting a new prompt/turn to the model, keeping the behavior consistent with GH703's prohibition on silently sending agent input.
- Automatic resume is gated behind a feature flag and can be turned off with a setting (default on), so users and reviewers have a clear escape hatch from auto-execution on startup.

## Non-goals

- Keeping the agent process *alive* across close/reopen (that needs a headless PTY daemon; explicitly out of scope).
- Transcript rehydration / re-download for a fresh machine or cloud sandbox (covered by REMOTE-1373).
- Resuming agents that do not expose a native resume CLI or whose `session_id` Warp does not capture (those panes restore as today, with no resume affordance).
- Restoring an in-flight tool call mid-execution. Resume restores conversation *context*, not a partially-completed turn.
- Reviving the prior OS process; resume always launches a new process.

## Behavior

### What is captured and persisted

1. While a CLI-agent session is tracked in a terminal pane and Warp learns the agent's native `session_id` (today, from the agent's structured hook events), Warp associates that `session_id` and the agent kind with the pane.
2. When the session is persisted (the same save path that already persists the pane's cwd, shell launch data, and Warp conversation ids), the captured native `session_id` and agent kind are persisted alongside, scoped to that pane.
3. A pane with no tracked CLI-agent session, or a tracked agent for which no native `session_id` was ever captured (e.g. an un-instrumented agent, or an agent launched without the Warp plugin), persists no resume data and is unaffected by this feature.
4. The persisted `session_id` is treated as opaque data. It is never expanded, split, or evaluated by a shell.

### Automatic resume on restore

5. When session restore is enabled and Warp relaunches, a restored pane that has persisted resume data (agent kind + native `session_id`) and whose agent kind supports native resume automatically launches that agent in the restored pane on startup — at the restored working directory, using the agent's native resume flag with the persisted `session_id` as argument data — without any manual user step.
6. The resume command Warp runs is the agent's standard resume invocation (e.g. `claude --resume <id>`) and appears in the pane exactly as if the user had typed it, so the action is transparent, inspectable, and re-runnable. It is run only after the restored shell is ready to accept a command.
7. Resuming reopens the prior session's context inside the agent and returns control to the user for the next turn. Warp does not synthesize or auto-submit a new prompt/turn on the user's behalf — this is the distinction from GH703, which prohibits auto-submitting prompt *content* into the model.
8. Automatic resume can be disabled by a setting (default on). With it off, the pane restores as a normal shell at the saved cwd, no resume command is run, and the persisted `session_id` remains available for a future manual resume. With the feature flag off entirely, the setting is absent and no resume command is ever run.
9. Each restored agent pane resumes independently and exactly once per restore; if several panes hosted resumable agents, each re-launches its own agent. A user-visible consequence is that restoring a session with N resumable agent panes starts N agents on launch.

### Resume outcome and failures

10. On successful resume, the agent starts in the pane already aware of its prior conversation: subsequent turns reference earlier context, and the agent's tracked status (working / blocked / done) updates through the existing CLI-agent status pipeline as normal.
11. If the agent CLI is not found on `PATH` at restore time, the affordance either is not offered or, if activated, surfaces a clear error in the pane rather than failing silently. The pane remains usable as a normal shell.
12. If the restored working directory no longer exists, resume falls back to the user's home directory (or the shell's default) and still attempts resume; if the agent then cannot locate its session, see (13).
13. If the agent CLI cannot resume the given `session_id` (expired, pruned, or desynced local session index), the error is surfaced to the user and the agent exits non-zero. Resume must not silently start a brand-new empty session disguised as a resume (consistent with REMOTE-1373).
14. Resume operates per pane and independently: multiple restored panes each offer and perform resume for their own agent without interfering with one another.
15. Resuming the same `session_id` is best-effort idempotent from Warp's side — Warp issues the resume command; any constraint on the same session being resumed in two places simultaneously is the agent CLI's behavior, not enforced by Warp, and any resulting agent-side message is shown as-is.

### Interaction with adjacent features

16. Warp's own (non-CLI) agent conversations continue to restore through their existing path and are unaffected; this feature only adds resume for third-party CLI-agent panes.
17. The feature is gated behind a feature flag. With the flag off, no resume command is ever run and restored panes behave exactly as they do today. Persisting the `session_id` may occur regardless of the flag (a harmless extra column), but automatic resume happens only when the flag is on (and the opt-out setting from (8) is on).
18. Undo-close and other in-session pane restoration paths are out of scope for v1: automatic resume targets cross-relaunch restore. If a closed agent pane is reopened via undo-close within the same Warp session, no regression is introduced (it behaves as today).
19. A restored pane that is a shared-session viewer does not auto-resume an agent.
