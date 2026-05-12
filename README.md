# <img src="icon.png" width="45" align="center" alt="icon"> Claude Alfred Workflow

Streaming chat with [Anthropic's Claude](https://www.anthropic.com/claude) inside Alfred 5. Reply types out live in a Text View. Modifier shortcuts for new chat, copy last/full, and interrupt — same UX as the official ChatGPT workflow.

> [!NOTE]
> Architecture, streaming logic, and Text View graph ported from the [official OpenAI Alfred workflow](https://github.com/alfredapp/openai-workflow) by [Vítor Galvão](https://github.com/vitorgalvao). Huge thanks — this workflow would not exist without his work.

## Install

1. Download `Claude.alfredworkflow` from the [latest release](https://github.com/vinnie291/claude-alfred-workflow/releases/latest).
2. Double-click — Alfred imports.
3. In Alfred → Workflows → Claude → gear → **Configure Workflow…** — paste your API key from [console.anthropic.com](https://console.anthropic.com/settings/keys).

## Setup

1. Sign in to [console.anthropic.com](https://console.anthropic.com/login).
2. On the [API keys page](https://console.anthropic.com/settings/keys), click `+ Create Key`.
3. Name the key and click `Create Key`.
4. Copy the key into the [workflow's Configuration](https://www.alfredapp.com/help/workflows/user-configuration/).

## Usage

Query Claude via the `ask` keyword. Reply streams live in Alfred's Text View.

| Key | Action |
|---|---|
| <kbd>↩</kbd> | Ask a new question |
| <kbd>⌘</kbd><kbd>↩</kbd> | Clear and start new chat |
| <kbd>⌥</kbd><kbd>↩</kbd> | Copy last answer |
| <kbd>⌃</kbd><kbd>↩</kbd> | Copy full chat as markdown |
| <kbd>⇧</kbd><kbd>↩</kbd> | Stop generating answer |

Type `ask` with no query to reopen the prior chat.

## Configuration

Open **Configure Workflow…** to set:

- **Anthropic API Key** — required. Not exported when sharing the workflow.
- **Claude Keyword** — default `ask`.
- **Model** — Opus 4.7, Sonnet 4.6, or Haiku 4.5.
- **Max Tokens** — cap on reply length (256 → 8192).
- **Context** — how many recent messages to send each request (4 → 40).
- **Timeout** — seconds without stream activity before considering the connection stalled (3 → 30).
- **System Prompt** — sent with every request.

## Architecture

- **`claude`** (JXA, in workflow bundle root) — invoked by Text View on every keystroke. On first call: spawns background `curl` writing Anthropic SSE to `$alfred_workflow_cache/stream.txt`, records PID in `pid.txt`, returns Text View JSON with `rerun: 0.1`. Subsequent calls (`streaming_now=1`) parse the growing SSE file (`content_block_delta` events), emit incremental `response` updates with `behaviour.response=replacelast`. When `message_stop` arrives, appends final reply to `chat.json` and stops the rerun loop.
- **`info.plist`** — workflow graph: keyword → Text View → modifier-keyed actions (stop / clear / copy-last / copy-all). After stop/clear, control returns to the Text View via `callexternaltrigger` → matching `trigger.external` objects (`new_chat`, `view_chat`), mirroring the OpenAI workflow's wiring.
- **Stall detection** — if `stream.txt` mtime is older than `timeout_seconds`, partial reply is written to history with `[Connection Stalled]` footer and curl is abandoned.
- **Resume mid-stream** — if the Text View closes while streaming, reopening detects an orphaned `stream.txt` (with `streaming_now` unset) and resumes the rerun loop.

## Differences vs OpenAI workflow

Deliberately **not** included (kept simple — can add later):

- Chat-history archive view (`⌥↩` on the keyword in OpenAI version lists prior chats).
- Universal Action integration.
- Fallback Search.
- DALL·E equivalent (no Anthropic image-generation product).

Protocol swaps:

- Auth: `x-api-key` header (no `Bearer` prefix) + `anthropic-version: 2023-06-01`.
- Endpoint: `https://api.anthropic.com/v1/messages`.
- Body: `system` is a top-level field, not a message with `role: "system"`. `max_tokens` is required.
- SSE: events tagged `content_block_delta` carry `delta.text`. Finish via `message_stop`; reason via `message_delta.delta.stop_reason`.

## Build the bundle yourself

```bash
git clone https://github.com/vinnie291/claude-alfred-workflow.git
cd claude-alfred-workflow
zip Claude.alfredworkflow info.plist claude icon.png README.md
open Claude.alfredworkflow
```

## State paths

- Chat history: `~/Library/Application Support/Alfred/Workflow Data/com.vinhle.alfred-claude/chat.json`
- Stream temp: `~/Library/Caches/com.runningwithcrayons.Alfred/Workflow Data/com.vinhle.alfred-claude/stream.txt` + `pid.txt`

Delete `chat.json` (or hit <kbd>⌘</kbd><kbd>↩</kbd> in the Text View) to reset.

## Requirements

- macOS 12+ (osascript ships built-in).
- Alfred 5 with [Powerpack](https://www.alfredapp.com/powerpack/).
- Anthropic API key.

## Troubleshooting

- **"ANTHROPIC API key not set"** — open Configure Workflow… and paste your key.
- **API error returned in Text View** — Anthropic's error message is rendered verbatim. Common: bad model name, malformed system prompt, key revoked.
- **Stream stalls indefinitely** — increase `timeout_seconds`, or check network.
- **No streaming, just dumped final** — open Alfred's workflow debugger (bug icon on workflow) to see what JSON each invocation returns.

## License

BSD 3-Clause. Same as the upstream OpenAI workflow this is derived from. See [LICENSE](LICENSE).
