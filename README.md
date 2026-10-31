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
| `transcript_preparer` | Replays Vibe's recovery journal (`projection_delta` append/replace/remove ops) into a readable Markdown transcript — user prompts, assistant responses, reasoning, and tool calls with command and output. |
| `transcript_analyzer` | Tracks the journal position and extracts modified files (Git-confirmed: only files actually changed in the worktree are reported). |
| `token_calculator` | Reports **per-turn token deltas** from Vibe's cumulative `tokenUsage`, with a fallback that reads usage straight from the journal. |
| `compact_transcript` | Converts the transcript into Entire's compact transcript JSONL (`transcript.jsonl`), which is what the Entire dashboard renders as Prompts / Responses / Tool calls. |

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
- **Token totals look too high or too low** — token usage is read from the
  journal at turn end; a turn that is still running reports the previous
  cumulative state.

## Compatibility

Tested with:

- Mistral Vibe Code 1.27.0 (`mistral-vibe-vscode`), recovery journal record
  version 1
- Entire CLI Agent Integration Protocol, protocol version 1

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

## License

Released under the **MIT License** — see [`LICENSE`](LICENSE).
You are free to use, modify, and redistribute this integration, including in
commercial products, provided the copyright notice and license text are
included. Contributions (issues, pull requests) are welcome.
