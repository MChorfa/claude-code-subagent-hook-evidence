# Claude Code subagent and hook behaviour on 2.1.330: evidence for upstream issues

Measured 2026-11-23 for anthropics/claude-code #78063, #100198, #79953 and #76726. Data for claude-agent-sdk-typescript#172 is included but not posted. `scripts/upstream-verify.sh .` recomputes every per-run `summary.json` from the records beside it and checks `SHA256SUMS`. The cross-run totals below are sums over those summaries, or counts taken directly from the bundled `hooks.jsonl` and `subagent-tool-calls/` files.

## Setup

- Claude Code 2.1.330 on macOS 27.3 arm64 (Darwin 27.3.0), headless `claude -p --output-format stream-json --include-hook-events`.
- Models `claude-sonnet-5-5` (24 experiments × 3 runs) and `claude-haiku-5-5` (4 experiments × 3 runs): 84 runs, 72 of which spawned subagents (90 subagents in all).
- User settings are not loaded (`--setting-sources project,local`), and MCP is off. Every run's init line lists only Claude Code's 3 built-in plugins. In all 84 runs, the number of `hook_started` events equals the number of hook records this harness wrote, so no other hook ran. Every `hook_started` has a matching `hook_response` with the hook's exit code, and `upstream-verify.sh` checks that each non-zero exit is the exit 2 the run's hook mode calls for.
- Each run happens in a fresh temp project that holds one `file.txt`. Permission mode is `bypassPermissions` unless an experiment's name ends in `default-mode`.
- `scripts/upstream-hook.sh` writes one record per hook event: the raw input, the time, and the subagent's `.meta.json` sidecar as it exists at that moment. Depending on the experiment, it also denies one tool by exit 2 or by JSON `permissionDecision: "deny"`.

**Counting.**
- "Ran Bash": a `PostToolUse` for Bash with `agent_id` set, whose command is the probe and whose response contains the expected output. The probe is `printf dg-<random> | shasum -a 256 | cut -c1-16`, with a fresh random value per run.
- "Hook-visible attempt": a `PreToolUse` for the tool.
- "Transcript call": a `tool_use` in a subagent transcript, by unique id. Across all runs the subagent transcripts hold 76 tool calls. 75 have a matching `PreToolUse` with `agent_id`. The other one was refused by Claude Code as unavailable before any hook ran.
- A denial is attributed to a subagent when the parent's `permission_denials` lists a `tool_use_id` that belongs to that subagent's `PreToolUse`.
- Children ran in the foreground (`run_in_background: false`, `Agent` result `status: completed`) except in `78063-background` and in haiku run-2 of `78063-agentdef-deny`, where the model left `run_in_background` out and the child started in the background.

Every result below held in all 3 runs of its experiment unless a row says otherwise.

## claude-code#78063: agent-level restrictions and subagents (sonnet)

| Configuration (each its own experiment) | Child ran Bash |
| --- | --- |
| `--agent conf`, no restriction (control) | 3/3 |
| `--agent conf`, `disallowedTools: ["Bash"]` (Bash absent from the main thread's init tool list) | 3/3, haiku 3/3 (haiku run-2's child ran in the background) |
| `--agent conf`, `tools: [Read, Glob, Grep, Agent]` | 3/3 |
| `--agent conf` with Bash denied, custom child with no `tools` field | 3/3 |
| `--agent conf` with Bash denied, general-purpose child started with `run_in_background: true` | 3/3 |
| `--agent conf` with Bash denied, `--permission-mode default` with `--allowedTools` for the probe, general-purpose child | 3/3 |
| `--disallowedTools Bash` on the session (no `--agent`) | 0/3, haiku 0/3. One child (haiku run-2) called Bash and got "No such tool available: Bash. Bash is disabled for this session, in subagents as well as here."; three (haiku run-1 and run-3, sonnet run-2) looked for it with ToolSearch `select:Bash` and found nothing; two (sonnet run-1 and run-3) made no tool call |
| `--agent conf` unrestricted, settings `permissions.deny: ["Bash(printf *)"]` | 0/3. The child's `PreToolUse` fired, then the call was denied |
| main, subagent `reviewer` with `tools: [Read, Agent]` delegates to general-purpose | grandchild 3/3 |
| main, subagent `reviewer` with `tools: [Read, "Agent(Explore)"]` asked for general-purpose | general-purpose spawned, grandchild 3/3. This matches the docs, which say a type list in `Agent(...)` is ignored in a subagent definition |

## claude-code#76726: hook denials inside subagents (sonnet, foreground, no worktrees)

| Hook | Subagents | Denials listed in the parent and attributed to the subagent |
| --- | --- | --- |
| exit 2 on Bash | 1 | 1, 3/3 |
| JSON deny on Write | 1 | 1, 3/3, no file written |
| JSON deny on Write | 3 in parallel | 3, 3/3, no file written |
| JSON deny on Write, `default` mode, `--allowedTools Agent Write` | 1 | 1, 3/3 |
| exit 2 on Bash / JSON deny on Write, main thread (controls) | 0 | 1 attributed to main, 3/3 each |

- Every hook input in all 84 runs (504 events) carries the main session's `session_id`.
- `agent_id` is on all 90 `SubagentStart`, all 90 `SubagentStop`, and every subagent tool call that reached a hook (75).
- `prompt_id` has one value per run.
- The hook sees `tool_name: "Agent"`, while `permission_denials` names the same call `Task` (`79953-direct-agent-deny`).

## claude-code#79953: workflow agents (sonnet)

| Hook | Main calls | Workflow agent | Its Bash |
| --- | --- | --- | --- |
| denies `Agent`/`Task` | `Workflow` | started, no `PreToolUse(Agent)` | ran 3/3 |
| exit 2 on `SubagentStart` (`hook_response` exit code 2, 3/3) | `Workflow` | started | ran 3/3 |
| denies Bash when `agent_id` is set | `Workflow` | started | denied 3/3, listed in the parent |
| denies `Workflow` | `Workflow` | none, the call is denied | none |
| denies `Agent`/`Task` | `Agent` (control) | none, the call is denied | none |

## claude-code#100198 and #32016: payload fields and sidecars

- `SubagentStart` input keys (90/90): `agent_id, agent_type, cwd, hook_event_name, prompt_id, session_id, transcript_path`. Subagent `PreToolUse` adds `effort, permission_mode, tool_input, tool_name, tool_use_id`. Neither has a parent id or the spawning call's `tool_use_id`.
- Sidecar `<session>/subagents/agent-<id>.meta.json`:
  - absent at `SubagentStart` in 90 of 90 starts;
  - present with `toolUseId` at the first `PreToolUse` of every Agent-spawned child that made one (66; the other 15 of the 81 made no tool call that reached a hook);
  - workflow agents' sidecars (`subagents/workflows/wf_*/`) are present then, but have no `toolUseId` (9/9).
- Final sidecars of the 81 Agent spawns:
  - `toolUseId` names the `Agent` call whose result returned that agent's `agentId` (81/81);
  - `parentAgentId` is set at depth 2 (9/9) and absent at depth 1 (72/72);
  - `description` is always present.
- `PostToolUse(Agent)` carries the child's `agentId` in `tool_response`. For the 77 foreground children it fired after the child's `SubagentStop`. For the 4 background children it fired at launch, 0.9 to 1.6 s before the child's first `PreToolUse`.

## claude-agent-sdk-typescript#172 (not posted)

CLI path only, sonnet and haiku, 3 runs each:
- A subagent with `tools: ["Read"]` made no Bash call and reported `Read` as its only tool.
- A subagent with `tools: ["Read","Write"]` and `disallowedTools: ["Agent"]` spawned nothing and reported `Read, Write`.

This rests on self-report plus the absence of a call. In 9 of the 12 runs, the parent model's prompt to the child matches `another way|other means|other method|substitut|workaround` (0 of 12 in the #78063 session-deny and agent-definition-deny controls). The Agent SDK path was not tested.

In the 3 haiku runs of the second variant (`exit_status: 1`), the parent's final turn failed with an API usage-limit error after the subagent had finished (`SubagentStop` recorded). The subagent results above come from the completed part of those runs. These are the only runs with a non-zero exit status.

## Not tested

- Windows, Linux, VS Code, interactive sessions, worktree isolation.
- Agent SDK callbacks.
- tmux teammates (#87415).
- Forked-skill display names.
- Workflows with more than one `agent()`.
- Models other than sonnet and haiku.

## Layout

- `<model>/<experiment>/run-<n>/` contains:
  - `run.json`: argv, exit status, probe, expected output, files left in the project;
  - `settings.json`: the hook wiring and any `permissions.deny` rule the run used;
  - `hooks.jsonl`;
  - `stream.tool-calls.jsonl`: tool calls, results, hook events, and a reduced init line;
  - `sidecars/`;
  - `subagent-tool-calls/`;
  - `summary.json`.
- `scripts/` holds the harness.
  - `upstream-repro.sh EXPERIMENT OUTDIR [MODEL]` runs one experiment.
  - `upstream-scrub.sh` produced this directory. It rewrites local paths and fails if the user name, host name, uid or a home path survives.
  - `upstream-verify.sh` recomputes the verdict, denial, sidecar and id fields of every `summary.json` from the files beside it, and checks `SHA256SUMS`.
