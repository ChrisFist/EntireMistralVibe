# entire-agent-vibe

Agent integration that connects **Mistral Vibe Code** (VS Code extension / CLI)
with the **[Entire CLI](https://entire.io)** — so Vibe sessions are captured as
Entire checkpoints with full conversation transcripts, per-turn token usage,
and file-change tracking, viewable on the Entire dashboard next to your commits.

This is a standalone implementation of Entire's Agent Integration Protocol
(`protocol_version 1`). Entire discovers it as the external agent **`vibe`**
when the binary is on your `PATH` under the name `entire-agent-vibe`.

## What it captures

| Capability | What it does |
|---|---|
| `hooks` | Vibe's `post_agent` hook maps to Entire's TurnEnd event, so every completed turn becomes a checkpoint opportunity. |
| `transcript_preparer` | Replays Vibe's recovery journal (`projection_delta` append/replace/remove ops) into a readable Markdown transcript — user prompts, assistant responses, reasoning, and tool calls with command and output. Waits briefly for the journal to settle so the turn's final response is always captured. |
| `transcript_analyzer` | Tracks the journal position and extracts modified files (Git-confirmed: only files actually changed in the worktree are reported). |
| `token_calculator` | Reports **per-turn token deltas** from Vibe's cumulative `tokenUsage`, with a fallback that reads usage straight from the journal. |
| `compact_transcript` | Converts the transcript into Entire's compact transcript JSONL (`transcript.jsonl`), which is what the Entire dashboard renders as Prompts / Responses / Tool calls. Merges the stored transcript (full history) with the live journal (complete recent turns), so nothing is lost at either end. |

## How it works

Mistral Vibe persists each session as a recovery journal under
`~/.vibe/logs/session/unified/<session-id>/journal/*.jsonl` — a rolling window
of JSONL records (`core_input`, `projection_delta`, `action_intent`,
`action_result`, `callback_*`). The journal is compacted over time, so this
integration captures the transcript at **turn end**, before older chunks are
removed, and stores a prepared sidecar (`.prepared.md`) next to each chunk.

At checkpoint time, Entire feeds the stored transcript back through
`compact-transcript`, which converts it into the compact JSONL format the
dashboard understands.

### Turn-end timing and how it is handled

Vibe's `post_agent` hook fires **before** the turn's final assistant message
is appended to the journal. Two safeguards make sure that message still ends
up in every checkpoint:

1. **Journal quiescence wait** — `prepare-transcript` polls the journal until
   its size stops growing (a fraction of a second, capped at a few seconds),
   then replays it. Turn end therefore takes roughly one extra second, and
   the final response is always captured.
2. **Two-source merge** — `compact-transcript` merges the stored transcript
   with the live journal: the stored transcript carries the **full session
   history** (the journal is a rolling window — Vibe deletes older chunks),
   while the live journal carries the **complete recent turns** including the
   final response, which the stored snapshot can miss. Older journal chunks
   being deleted over time is exactly why neither source alone is sufficient.

## Requirements

- **Python 3.8+** (standard library only — no third-party dependencies)
- **Mistral Vibe Code** installed and used at least once (the `~/.vibe` directory exists)
- **Entire CLI** installed
- **Git** (used to confirm which files actually changed)

## Installation

### Linux / macOS

```bash
cp entire_agent_vibe.py /usr/local/bin/entire-agent-vibe
chmod +x /usr/local/bin/entire-agent-vibe
```

### Windows (PowerShell)

Create a folder on your `PATH` (e.g. `C:\Tools\entire-agents`), save the
script there, and add a small wrapper so Entire can execute it by protocol
name:

```powershell
mkdir C:\Tools\entire-agents
Copy-Item .\entire_agent_vibe.py C:\Tools\entire-agents\
Set-Content C:\Tools\entire-agents\entire-agent-vibe.cmd '@python "%~dp0entire_agent_vibe.py" %*'
```

(The `.cmd` wrapper is needed because the protocol requires an executable
named exactly `entire-agent-vibe`; Windows resolves `.cmd` files through
`PATHEXT`.)

### Enable external agent discovery

External agent discovery is opt-in. In your repository, add to
`.entire/settings.json`:

```json
{ "external_agents": true }
```

Then enable the agent:

```bash
entire enable --agent vibe
```

### Recommended `.gitignore` entries

Add these lines to your repository's `.gitignore`:

```gitignore
# Entire: agent discovery is configured locally, never committed
.entire/settings.local.json
# Integration working state (session cache, token counters) — local only
.entire/vibe-agent-sessions/
```

Why: Entire requires the `external_agents` setting to live in the **untracked**
`.entire/settings.local.json` on purpose — the committed `.entire/settings.json`
cannot grant it. The integration's working state under
`.entire/vibe-agent-sessions/` (per-session cache, token counters) is local-only
as well and does not belong in commits.

Note that checkpoint data (transcripts, tokens, file changes) is **not**
affected by `.gitignore`: Entire stores it in git refs outside the working
tree, so your captured sessions are safe either way.

## Verify it works

```bash
entire status          # should list the vibe agent
```

Then:

1. Start a Vibe session in your repository and complete at least one turn.
2. Commit your work as usual — Entire will offer to absorb the session into a checkpoint.
3. Push and open the checkpoint on the Entire dashboard: the session shows its
   conversation (prompts, responses, tool calls), token totals, and file changes.

## Troubleshooting

Set the environment variable `VIBE_AGENT_DEBUG=1` to record every subcommand
invocation and result in `~/vibe-agent-debug.log` (off by default):

```powershell
$env:VIBE_AGENT_DEBUG = "1"   # PowerShell
```

Common checks:

- **`entire status` doesn't list the agent** — confirm the binary name is
  exactly `entire-agent-vibe` and it responds to `entire-agent-vibe info`.
- **No transcripts appear** — complete a full turn (the hook fires on turn
  end), and confirm Vibe's journal exists under `~/.vibe/logs/session/unified/`.
- **The last response of a turn is missing or truncated on the dashboard** —
  Vibe's turn-end hook fires before the final response is flushed to the
  journal. The current version waits for the journal to settle and merges the
  live journal into the compact transcript, so new turns are captured
  completely. If you see this, make sure the deployed script is the latest
  version and complete a **new** turn: already-stored checkpoints are not
  rewritten retroactively.
- **Token totals look too high or too low** — token usage is read from the
  journal at turn end; a turn that is still running reports the previous
  cumulative state.

## Compatibility

Tested with:

- Mistral Vibe Code 1.27.0 (`mistral-vibe-vscode`), recovery journal record
  version 1
- Entire CLI Agent Integration Protocol, protocol version 1 (tested with
  Entire CLI 0.12.2)

Vibe's journal format is not a documented public contract; if a future Vibe
release changes its record structure, `prepare-transcript` and
`compact-transcript` are the components to adapt.

## Privacy notes

- Session transcripts are stored in your git repository via Entire's
  per-checkpoint refs; Entire redacts detected secrets before writing
  checkpoints. Everything else about your repository's visibility applies
  (a public repo makes this data public).
- This integration only reads local files under `~/.vibe` and the current
  repository, and makes no network calls.

## Glossary

Terms used above that are common when building AI agent environments, but
not everyone knows.

| Term | Meaning |
|---|---|
| Session | One continuous conversation between you and the agent, tied to a working directory. Identified by a UUID, with its own start time, model, and cumulative token counter. Vibe stores it under `~/.vibe/logs/session/unified/<session-uuid>/`. |
| Turn | One round of a session: a single user prompt plus everything the agent does in response — reasoning, tool calls, and the final answer. The turn ends when the agent finishes responding; that is when Vibe fires the `post_agent` hook. |
| Hook | A command Vibe runs at defined moments, declared in `.vibe/hooks.toml` (`pre_tool`, `post_tool`, `post_agent`). The hook receives a JSON payload on stdin (`session_id`, `transcript_path`, `cwd`, `hook_event_name`, ...). This integration bridges `post_agent` to Entire's TurnEnd event. |
| Recovery journal | Vibe's append-only log of a session: JSONL files at `~/.vibe/logs/session/unified/<id>/journal/*.jsonl`, one JSON record per line (`core_input`, `projection_delta`, `action_intent`, `action_result`, `callback_*`, `receipt_*`). It is the source of truth for what happened in the session. |
| Projection delta | `projection_delta` records that describe updates to the canonical conversation state via ops: `append_entry`, `replace_entry`, `remove_entry`, `set_envelope`. Replaying all deltas in order rebuilds the conversation — that is how `prepare-transcript` produces the transcript. |
| Rolling window (compaction) | The journal is not kept forever: Vibe compacts it and deletes older chunks as the session grows. This is why the transcript is captured at turn end, and why neither the stored transcript nor the live journal is complete on its own. |
| Journal quiescence | `post_agent` fires slightly before the turn's final assistant message is flushed to the journal, so `prepare-transcript` polls the journal's total size until it stops growing before reading it. |
| Transcript | The human-readable Markdown rendering of a session — the `.prepared.md` sidecar written next to the journal chunk, with `## User`, `## Assistant`, and `## Tool: <name>` sections. |
| Compact transcript | Entire's dashboard format (`transcript.jsonl`): one JSON object per message — `type: "user"` or `"assistant"` with `text` and `tool_use` content blocks — produced by `compact-transcript` and base64-encoded in protocol responses. |
| Checkpoint | Entire's snapshot of an agent session, attached to a git commit and stored in Entire's own git refs outside the working tree, so `.gitignore` never affects it. Rendered on the Entire dashboard next to the commit. |
| Agent Integration Protocol | Entire's protocol (here `protocol_version 1`) for external agents: Entire discovers a binary on `PATH` (here `entire-agent-vibe`), reads its `info` capabilities, and drives it through subcommands that pass JSON on stdin and return JSON on stdout. |
| Capability | A feature an integration advertises in `info` — `hooks`, `transcript_preparer`, `transcript_analyzer`, `token_calculator`, `compact_transcript`, ... Entire only calls the subcommands behind advertised capabilities. |
| Lifecycle event | Entire's session event types: 1 SessionStart, 2 TurnStart, 3 TurnEnd, 4 Compaction, 5 SessionEnd. `parse-hook` maps Vibe's `post_agent` to type 3 (TurnEnd). |
| AgentSession | Entire's per-session state object (`session_id`, `repo_path`, `session_ref`, `modified_files`, `new_files`, `deleted_files`, `transcript_position`, ...), persisted by this integration under `.entire/vibe-agent-sessions/`. |
| Session ref / transcript position | Pointers into the session data: a path to a journal chunk or session directory, plus a byte offset into it. The byte offset doubles as the resume position (`vibe --resume <session-id>`). |
| Token usage | Vibe records `tokenUsage` (input/output/total tokens) cumulatively for the whole session. Entire sums per-turn values, so this integration remembers the previous cumulative total and reports only the per-turn delta. |
| Tool call (effect) | One agent action, e.g. running a shell command. In the journal it is an `effect` entry (`detail.input.command`, `state.outputText`); in the compact transcript a `tool_use` block with command and result. |
| Reasoning | The model's internal thinking text (journal `reasoning` entries). Deliberately omitted from the compact transcript. |
| Redaction | Entire strips detected secrets from transcripts before persisting a checkpoint. |
| JSONL | Newline-delimited JSON — one JSON object per line. Used for the journal files and for `transcript.jsonl`. |

## License

Released under the **MIT License** — see [`LICENSE`](LICENSE).
You are free to use, modify, and redistribute this integration, including in
commercial products, provided the copyright notice and license text are
included. Contributions (issues, pull requests) are welcome.
