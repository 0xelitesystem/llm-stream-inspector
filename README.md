# LLM Stream Inspector

Paste a raw Server-Sent Events dump from a streaming LLM response and get it parsed, reassembled, and diagnosed in your browser.

**Live demo:** https://0xelitesystem.github.io/llm-stream-inspector/

## Use

1. Paste a raw Server-Sent Events dump from a streaming LLM response, or press **Load sample (Anthropic)** or **Load sample (OpenAI)**.
2. Press **Analyze**. The wire format is detected for you, and you can override it.
3. Read the findings, then check the reassembled text, the rebuilt tool-call arguments and the event timeline.
4. Press **Copy report** for a plain-text report you can drop into a ticket.

## Why this exists

When a streamed response stops mid-sentence or a tool call arrives as invalid JSON, the SDK hides the reason and the raw event stream shows it. This tool parses that stream and names the failure. A stream dump carries prompts and tool arguments, so it runs as one HTML file in your browser, with no tracking and no server, under the MIT license.

## Features

- **Real SSE parsing.** Handles CRLF and LF line endings, blank-line event separation, multi-line `data:` fields, `event:` names, `id:` and `retry:` fields, `:` comment keep-alives, and a trailing `data: [DONE]` sentinel. A dump that was cut off mid-line is expected input, not an error.
- **Wire format auto-detection.** Tells Anthropic Messages streaming (`message_start`, `content_block_start`, `content_block_delta` with `text_delta` / `input_json_delta` / `thinking_delta`, `content_block_stop`, `message_delta`, `message_stop`, `ping`, `error`) from OpenAI chat completions streaming (`choices[].delta.content`, `delta.tool_calls[].function.arguments`, `finish_reason`, `[DONE]`) from plain unrecognised SSE. It shows the evidence it used and the signal score for each shape, and you can override the result.
- **Reassembly.** Concatenates the assistant text in block order, and rebuilds every tool call by joining its streamed JSON argument fragments, flagging loudly when the joined fragments do not parse as valid JSON.
- **Named findings, each with a plain explanation.** Stream ended without a terminal event; `stop_reason` or `finish_reason` of `max_tokens` or `length` (the output cap); content block opened but never closed; tool argument JSON incomplete; malformed event that failed to parse; an error event inside the stream; refusal, paused turn, and context-window stops; deltas for a block that was never opened; and reported token usage.
- **Event timeline.** Every event in order with its index, type, byte size, a bar sized by payload bytes, and the delta text or argument fragment it carried.
- **Stats.** Event count, total bytes, count by event type, content blocks (or choices), characters of text reassembled, and tool calls found.
- **Two samples, one click.** An Anthropic-shaped stream cut mid tool call, so the truncation findings visibly fire, and a healthy OpenAI-shaped stream for comparison.
- Dark theme by default with a light toggle that persists, keyboard accessible, works down to a 360px screen, copy buttons on every output, and a copyable plain-text report.

## How it works

This is wire-level debugging for streaming responses: the layer where "the model stopped mid-sentence" and "my tool call arguments are invalid JSON" actually get diagnosed. From inside an SDK those two failures look like a short string and a `JSON.parse` error. In the raw event stream they are obvious, and they are usually the same bug.

1. The dump is framed into events using the event-stream field rules, so a killed connection leaves a partial trailing event rather than breaking the parse.
2. Each event's `data` payload is parsed as JSON. Failures are kept and reported rather than thrown away, because a payload that fails to parse is itself a finding.
3. The format is scored from the shapes actually present in the events, not from a header or a guess.
4. The events are replayed through the reassembly rules for that format: text accumulates per content block, argument fragments accumulate per tool call, and block open and close events are tracked.
5. The resulting model is checked for the specific failure modes above, and each check produces a named finding with the reasoning spelled out.

The core is a set of pure functions (`parseSseDump`, `detectWireFormat`, `reassembleAnthropic`, `reassembleOpenAI`, `diagnoseStream`, `analyzeSseDump`) that take input and return a result object, kept separate from all DOM code, so they can be lifted out of the file and run headless.

One thing worth internalising: a streamed tool-argument fragment is almost never valid JSON on its own. It can end mid-key, mid-string, or mid-number. Only the concatenation of every fragment for that tool call parses, and only if the stream ran to completion. Buffer per tool call index, parse once at the end.

## Privacy

Everything runs in your browser. A streamed response carries the user prompt, retrieved documents, and tool arguments, which is exactly the data that should not be pasted into a stranger's parser, so nothing you paste here is uploaded, logged, or sent anywhere. The page has no analytics, no external dependencies, and makes no network requests of any kind. Verify by viewing the page source, or by opening DevTools and watching the network tab while you use it. For anything sensitive, save the file and open it offline.

The only thing written to storage is your light or dark theme choice, saved in `localStorage` under the key `llm-stream-inspector-theme`.

## Run locally

```bash
git clone https://github.com/0xelitesystem/llm-stream-inspector
cd llm-stream-inspector
```

Open `index.html` in any modern browser. Or serve the folder with `python -m http.server 8000` and visit http://localhost:8000/.

## Build

No build step. The whole tool is one `index.html` file with its CSS and JavaScript inline, and nothing to install.

## License

MIT. See [LICENSE](LICENSE).

## More

- Full catalog of these tools: https://0xelitesystem.github.io/
- https://elitesystem.ai
