# Response Audit

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![CI](https://github.com/mindroom-ai/response-audit-plugin/actions/workflows/plugin-ci.yml/badge.svg)](https://github.com/mindroom-ai/response-audit-plugin/actions/workflows/plugin-ci.yml)
[![Docs](https://img.shields.io/badge/docs-plugins-blue)](https://docs.mindroom.chat/plugins/)
[![Hooks](https://img.shields.io/badge/docs-hooks-blue)](https://docs.mindroom.chat/hooks/)

<picture>
  <source media="(prefers-reduced-motion: no-preference)" srcset="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark-animated.svg" />
  <img src="https://raw.githubusercontent.com/mindroom-ai/mindroom/main/assets/logo/logo-mark.svg" alt="MindRoom Logo" align="right" width="120" />
</picture>

A [MindRoom](https://github.com/mindroom-ai/mindroom) hook plugin that audits an agent's reply after delivery and, when a claim isn't backed by any tool lookup, posts a follow-up asking the agent to double-check it.

- **Opt-in per agent:** only agents listed in `agents:` are audited; teams and non-human turns are skipped.
- **Checks real tool activity:** inspects recorded tool calls and shared file names rather than taking the agent's prose at face value.
- **One targeted follow-up:** tags the agent once in the same thread, quotes the start of the audited reply, and skips audit-triggered turns to prevent loops.
- **Configurable checks and backends:** ships with a `research` check on OpenAI Decisions by default, and supports custom checks, **System One JEV** (`typesafe`), or any MindRoom `llm` model alias.

```text
@research — response audit flagged the following for review:
- research: Verify the specific facts with your tools, such as search, browsing, or documents you can read; correct or withdraw anything that does not hold up, name the sources you checked, and say which facts remain unverified.
Please verify these findings in your reply that starts "Cafe Sol opens at 8 AM on Sundays and charges $18 for brunch." and correct that reply where needed; later replies are not part of this audit.
```

## Quick start

```bash
mindroom plugins install response-audit-plugin
```

```yaml
plugins:
  - path: plugins/response-audit-plugin
    settings:
      agents: [research]
      judgment:
        provider: openai_decisions  # default; needs OPENAI_API_KEY (or use typesafe / llm)
        timeout_seconds: 5
```

No agent tools are needed; this plugin is hooks-only. `agents` defaults to `[]`, so nothing is audited until you add agent names.

Update later with `mindroom plugins update response-audit-plugin`.
Needs MindRoom v2026.10.258 or later.

## Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `agents` | `[]` | Agent names to audit (up to 100). Teams are excluded. |
| `judgment` | `openai_decisions`, threshold `0.8`, timeout `5 s` | MindRoom [judgment backend](https://docs.mindroom.chat/configuration/threads/#judgment-backends): `openai_decisions` (`OPENAI_API_KEY`), `typesafe` (System One JEV; `TYPESAFE_API_KEY`), or `llm` (a model alias from `models:`). |
| `checks` | `[research]` | 1 to 8 checks (`id`, `instructions`, optional `when_true` / `when_false`, `feedback`). Setting `checks` replaces the default `research` check. |

See [`docs/configuration.md`](docs/configuration.md) for custom checks, backend options, upgrade notes, and benchmarks.

<details>
<summary><b>How it works</b></summary>

1. When an enabled agent receives a person's prompt (skipping teams, other agents, and audit-triggered follow-ups), `begin-response-audit` begins capturing the turn without modifying its prompt.
2. `record-audit-tool` records each completed tool call's name, arguments, result, and status (`succeeded`, `failed`, or `blocked`).
3. If the reply is cancelled or fails, `discard-response-audit` drops the capture.
4. After a non-empty AI reply is delivered, `audit-response` serializes the request, reply, recorded tool calls, and non-audio room-scoped shared file names, claims `(agent, room, response_event_id)` in a local SQLite ledger (`audit-attempts.sqlite3`) so each reply is audited at most once across replays and restarts, and runs each check sequentially.
5. If any check flags a violation within the 30-second hook deadline, the plugin posts one combined follow-up in the same thread mentioning `@<agent>`, quoting up to 80 characters of the audited reply, and attaching `com.mindroom.response_audit` metadata (`response_event_id`, `checks`).

</details>

<details>
<summary><b>Privacy and limits</b></summary>

- **External provider transfer:** Request, reply, tool arguments/results, and shared file names are sent to the configured judgment provider. Secrets matched by MindRoom's redaction rules skip the entire audit rather than sending altered evidence, though redaction is not a general PII filter.
- **Current-turn scope only:** The judge sees only the audited message, room-scoped shared file names, and instrumented tool calls from that turn — not earlier history, system prompts, memory, provider-native tools, or delegated work. Audits request verification; they do not prove an answer wrong.
- **Bounded, fail-quiet, best-effort:** Audits are skipped on missing/evicted captures (cap: 128 captures, 1-hour TTL), non-JSON tool results, more than 32 tool calls, inputs over 16 KB, restarts/reloads, timeouts, or backend errors. Follow-ups are not retried on send failure.
- **Internal MindRoom imports:** `hooks.py`, `audit.py`, and `state.py` import internal MindRoom helpers outside `mindroom.plugin_api` (`mindroom.attachments.storage`, `mindroom.config.judgment`, `mindroom.hooks`, `mindroom.judgment.*`, `mindroom.tool_system.events`), so `mindroom plugins check` warns about them; re-run it after upgrades.

See [Evidence and Limits](docs/configuration.md#evidence-and-limits) for full details.

</details>

## Hooks

| Hook | Event | Timeout | Purpose |
|------|-------|---------|---------|
| `begin-response-audit` | `message:enrich` | `1000 ms` | Start observing an eligible turn without changing its prompt |
| `record-audit-tool` | `tool:after_call` | `300 ms` | Capture completed tool calls, arguments, results, and statuses |
| `audit-response` | `message:after_response` | `30000 ms` | Evaluate configured checks on the delivered reply and post one follow-up if flagged |
| `discard-response-audit` | `message:cancelled` | `1000 ms` | Discard captured evidence for cancelled or failed replies |
