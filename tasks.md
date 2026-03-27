# Implementation Tasks

## Task List

- [ ] 1. Scaffold the HTML file skeleton
  - [ ] 1.1 Create `bedrockchat.html` with `<!DOCTYPE html>`, `<html lang="en">`, `<head>`, and `<body>` tags
  - [ ] 1.2 Add `<meta charset="UTF-8">` and `<meta name="viewport">` tags
  - [ ] 1.3 Add the CSP `<meta>` tag with `default-src 'none'`, inline script/style, `connect-src` for all 7 Bedrock regions, `img-src data:`, `font-src 'self'`
  - [ ] 1.4 Add `<meta http-equiv="X-Content-Type-Options" content="nosniff">` and `<meta name="referrer" content="no-referrer">`
  - [ ] 1.5 Add `<title>Bedrock ChatBot</title>`
  - [ ] 1.6 Add `<link rel="icon" type="image/svg+xml">` with the base64-encoded AWS Bedrock SVG favicon as a `data:` URI

- [ ] 2. Write all CSS inside a `<style>` block
  - [ ] 2.1 Global reset (`box-sizing`, `margin`, `padding`) and `body` flex-column dark theme (`#0f1117` background, full viewport height)
  - [ ] 2.2 Header styles: flex row, dark background (`#1a1d27`), bottom border, title `h1` in purple (`#a78bfa`)
  - [ ] 2.3 `#region-select` and `#model-select` styles: dark background, border, border-radius, focus ring
  - [ ] 2.4 Token bar styles: flex row, dark background (`#13151f`), `#token-input` monospace flex-grow, `#mask-btn` borderless
  - [ ] 2.5 Chat area styles: flex-1, overflow-y auto, gap between messages, scrollbar styling
  - [ ] 2.6 Message bubble styles: `.msg.user` right-aligned purple, `.msg.assistant` left-aligned dark blue, avatar circles
  - [ ] 2.7 Typing indicator animation (`bounce` keyframes on three `<span>` dots)
  - [ ] 2.8 Streaming cursor styles: `.cursor::after` and `.streaming-tail::after` with `blink` keyframe animation
  - [ ] 2.9 Input bar styles: `#user-input` textarea with max-height 140px, `#send-btn` purple button with hover/disabled states
  - [ ] 2.10 Error toast styles: fixed bottom-center, dark red background, hidden by default
  - [ ] 2.11 Token usage pill styles: `.token-usage` with `.in` blue, `.out` green, total grey
  - [ ] 2.12 Markdown element styles inside `.bubble.assistant`: headings, paragraphs, lists, code, pre, blockquote, hr, links, tables

- [ ] 3. Build the HTML body structure
  - [ ] 3.1 Add `<header>` with `<h1>⚡ Bedrock ChatBot</h1>`, `#region-select`, and `#model-select`
  - [ ] 3.2 Populate `#region-select` with 7 region options: `us-east-1`, `us-east-2`, `us-west-2`, `eu-west-1`, `eu-central-1`, `ap-northeast-1`, `ap-southeast-2`
  - [ ] 3.3 Populate `#model-select` with 12 `<optgroup>` elements covering all providers and model IDs from the design document; set Claude Sonnet 4.5 as the default selected option; use `us.` prefix for all Claude and Nova model values
  - [ ] 3.4 Add `<div id="token-bar">` with `<label>`, `#token-input` (`type="text"`, `spellcheck="false"`), and `#mask-btn` (`👁` emoji)
  - [ ] 3.5 Add `<div id="chat"></div>` and `<div id="error-toast"></div>`
  - [ ] 3.6 Add `<div id="input-bar">` with `#user-input` textarea and `#send-btn`

- [ ] 4. Implement security scripts
  - [ ] 4.1 Add the frame-buster inline script as the first `<script>` tag in `<body>`: `if (top !== self) top.location = self.location;`

- [ ] 5. Implement utility functions
  - [ ] 5.1 Implement `escHtml(s)` — replaces `&`, `<`, `>` with HTML entities
  - [ ] 5.2 Implement `showError(msg)` — sets toast text, shows it, hides after 6 s
  - [ ] 5.3 Implement `REGION()` — returns current value of `#region-select`

- [ ] 6. Implement UI helper functions
  - [ ] 6.1 Implement `appendUserMsg(text)` — creates `.msg.user` div with avatar and HTML-escaped bubble, appends to `#chat`, scrolls to bottom
  - [ ] 6.2 Implement `createAssistantBubble()` — creates `.msg.assistant` div with avatar and `.bubble.cursor`, appends to `#chat`; also creates hidden `.token-usage` div; returns `{ bubble, finalize(fullText), showUsage(in, out, total) }`
  - [ ] 6.3 Wire `#mask-btn` click handler to toggle `#token-input` type between `"text"` and `"password"` and swap emoji between `👁` and `🙈`

- [ ] 7. Implement the Markdown renderer (`renderMd`)
  - [ ] 7.1 Escape HTML entities on the input string first (call `escHtml`)
  - [ ] 7.2 Replace fenced code blocks with `<pre><code class="lang-{lang}">` elements
  - [ ] 7.3 Replace inline code with `<code>` elements
  - [ ] 7.4 Replace ATX headings (`###`, `##`, `#`) with `<h3>`, `<h2>`, `<h1>`
  - [ ] 7.5 Replace blockquotes (`&gt; `) with `<blockquote>`
  - [ ] 7.6 Replace horizontal rules (`---`) with `<hr>`
  - [ ] 7.7 Replace bold-italic, bold, and italic patterns with `<strong><em>`, `<strong>`, `<em>`
  - [ ] 7.8 Replace Markdown links with `<a>` elements; sanitise href — only allow `http://` or `https://` URLs, otherwise use `#`; add `target="_blank" rel="noopener noreferrer"`
  - [ ] 7.9 Replace pipe-delimited tables with `<table>` elements; first row uses `<th>`, subsequent rows use `<td>`; skip separator rows
  - [ ] 7.10 Replace unordered list blocks with `<ul><li>` elements
  - [ ] 7.11 Replace ordered list blocks with `<ol><li>` elements
  - [ ] 7.12 Wrap remaining double-newline-separated blocks in `<p>` tags, skipping blocks that already start with a block-level HTML tag

- [ ] 8. Implement the Event-Stream frame parser utilities
  - [ ] 8.1 Implement `concat(a, b)` — concatenates two `Uint8Array` instances
  - [ ] 8.2 Implement `u32(arr, off)` — reads a 4-byte big-endian unsigned integer from a `Uint8Array`
  - [ ] 8.3 Implement `extractJSON(str)` — extracts all top-level JSON objects from a string using brace-depth counting; returns an array of parsed objects

- [ ] 9. Implement `streamBedrock(modelId, bearerToken, messages, onChunk)`
  - [ ] 9.1 Construct the endpoint URL as `https://bedrock-runtime.${REGION()}.amazonaws.com/model/${encodeURIComponent(modelId)}/converse-stream`
  - [ ] 9.2 Send a `fetch` POST with `Authorization: Bearer {token}`, `Content-Type: application/json`, and body `{ messages, inferenceConfig: { maxTokens: 4096, temperature: 0.7 } }`
  - [ ] 9.3 On non-2xx response, read the response body text and throw an error with the status code and body
  - [ ] 9.4 Obtain a `ReadableStream` reader from `res.body.getReader()`
  - [ ] 9.5 Implement the read loop: accumulate bytes into `buf`, drain complete frames using the event-stream frame structure, call `onChunk(fullText)` for each `delta.text` event, capture `usage` from `usage.inputTokens` events
  - [ ] 9.6 Return `{ fullText, usage }` when the stream ends

- [ ] 10. Implement `sendMessage()` and wire up event listeners
  - [ ] 10.1 Implement `sendMessage()`: read and validate `#user-input` and `#token-input`; call `appendUserMsg`, push to `history`, call `createAssistantBubble`, call `streamBedrock` with the `onChunk` partial-render callback, call `finalize` and `showUsage` on success, show error on failure, re-enable send button in `finally`
  - [ ] 10.2 Implement the `onChunk` callback: find last newline in accumulated text; render complete lines with `renderMd`, render incomplete trailing line with `escHtml` inside `.streaming-tail` span; scroll chat to bottom
  - [ ] 10.3 Wire `#user-input` `input` event to auto-resize the textarea (set height to `auto` then to `scrollHeight`, capped at 140px)
  - [ ] 10.4 Wire `#user-input` `keydown` event: Enter without Shift calls `sendMessage()` and prevents default; Shift+Enter does nothing
  - [ ] 10.5 Wire `#send-btn` `click` event to `sendMessage()`
  - [ ] 10.6 Initialise the `history` array and declare the `REGION` helper at module scope

- [ ] 11. Verify the complete file
  - [ ] 11.1 Open the file in a browser and confirm the layout matches the design: header with selectors, token bar, empty chat area, input bar
  - [ ] 11.2 Paste a valid Bearer token, send a message, and confirm streaming text appears incrementally with the blinking cursor
  - [ ] 11.3 Confirm that after streaming completes the full response is rendered as Markdown (code blocks, lists, headings visible)
  - [ ] 11.4 Confirm the token usage row appears below the assistant bubble with correct in/out/total counts
  - [ ] 11.5 Send a follow-up message and confirm the assistant's reply reflects the conversation context (multi-turn working)
  - [ ] 11.6 Switch region and model and confirm the next request uses the updated endpoint and model ID
  - [ ] 11.7 Attempt to send a message without a token and confirm the error toast appears with the correct message
