# Design Document

## Overview

The bedrock-chatbot is a single self-contained HTML file. All logic lives in one `<script>` block; all styles live in one `<style>` block. There is no build step, no module system, and no network fetches for assets. The file is opened directly in a browser.

The architecture has three logical layers:

1. **UI layer** — DOM construction helpers, event listeners, CSS animations.
2. **Protocol layer** — `streamBedrock()` function that calls the Bedrock `converse-stream` endpoint and drives the Event_Stream_Parser.
3. **Rendering layer** — `renderMd()` Markdown-to-HTML converter and `escHtml()` sanitiser.

---

## File Structure

```
bedrockchat.html          ← single deliverable file
```

Internal HTML structure:

```
<head>
  <meta charset> <meta viewport> <meta CSP> <meta nosniff> <meta referrer>
  <title>
  <link rel="icon" (base64 SVG)>
  <style> … all CSS … </style>
</head>
<body>
  <header>          ← title h1, #region-select, #model-select
  <div#token-bar>   ← label, #token-input, #mask-btn
  <div#chat>        ← message bubbles appended here at runtime
  <div#error-toast>
  <div#input-bar>   ← #user-input textarea, #send-btn
  <script> if(top!==self)… </script>   ← frame-buster (must be first script)
  <script> … all application JS … </script>
</body>
```

---

## Component Design

### 2.1 Header

- `<h1>` with title text and lightning emoji.
- `#region-select` — `<select>` listing 7 AWS regions.
- `#model-select` — `<select>` with 12 `<optgroup>` elements, one per provider. Model `value` attributes are the exact Bedrock model IDs / inference profile IDs. Claude and Nova entries use the `us.` prefix.

### 2.2 Token Bar

- `<label>` "Bearer token" linked to `#token-input`.
- `#token-input` — `type="text"` by default, monospace font, `spellcheck="false"`.
- `#mask-btn` — plain `<button>` with no border/background. Clicking toggles `token-input.type` between `"text"` and `"password"` and swaps the emoji between `👁` and `🙈`.

### 2.3 Chat Area (`#chat`)

Flex column, `overflow-y: auto`, `gap: 16px`. Messages are appended as `.msg.user` or `.msg.assistant` divs, each containing an `.avatar` div and a `.bubble` div.

User bubbles: right-aligned, purple background, `white-space: pre-wrap`, HTML-escaped text.

Assistant bubbles: left-aligned, dark blue background, innerHTML set by `renderMd()`. During streaming the bubble carries the `cursor` CSS class which appends the blinking `▋` via `::after`. Incomplete trailing lines use a `<span class="streaming-tail">` which also carries the blinking cursor.

Token usage row: a `.token-usage` div appended after the assistant `.msg` div, hidden until usage data arrives.

### 2.4 Input Bar

- `#user-input` — `<textarea rows="1">`, auto-resizes on `input` event up to `max-height: 140px`.
- `#send-btn` — purple button, disabled while a request is in flight.
- Enter (no Shift) submits; Shift+Enter inserts newline.

### 2.5 Error Toast (`#error-toast`)

Fixed position, bottom-center, hidden by default (`display: none`). `showError(msg)` sets `textContent`, shows it, and hides it after 6 s via `setTimeout`.

---

## Data Flow

```
User types → sendMessage()
  → appendUserMsg()          push to DOM
  → history.push(userTurn)   push to Conversation_History
  → createAssistantBubble()  create DOM placeholder
  → streamBedrock()          POST to Bedrock
      → reader.read() loop
          → Event_Stream_Parser
              → onChunk(fullText)  → renderMd() partial → bubble.innerHTML
      → returns { fullText, usage }
  → bubble.innerHTML = renderMd(fullText)   final render
  → history.push(assistantTurn)
  → showUsage(in, out, total)
```

---

## Key Algorithms

### 3.1 Event-Stream Frame Parser

AWS event-stream binary framing:

```
[total_len: 4 bytes big-endian]
[headers_len: 4 bytes big-endian]
[prelude_crc: 4 bytes]
[headers: headers_len bytes]
[payload: (total_len - 12 - headers_len - 4) bytes]
[message_crc: 4 bytes]
```

Implementation:

```javascript
// buf: Uint8Array accumulator
while (buf.length >= 12) {
  const totalLen   = u32(buf, 0);
  const headersLen = u32(buf, 4);
  if (totalLen < 16 || totalLen > 1_000_000) { buf = new Uint8Array(0); break; }
  if (buf.length < totalLen) break;           // wait for more bytes
  const payloadStart = 12 + headersLen;
  const payloadEnd   = totalLen - 4;
  const frameStr     = dec.decode(buf.slice(payloadStart, payloadEnd));
  buf = buf.slice(totalLen);
  for (const evt of extractJSON(frameStr)) {
    if (evt.delta?.text)              fullText += evt.delta.text, onChunk(fullText);
    if (evt.usage?.inputTokens != null) usage = evt.usage;
  }
}
```

`extractJSON(str)` uses brace-depth counting to extract all top-level JSON objects from a string, tolerating non-JSON header bytes that precede the payload JSON.

### 3.2 Markdown Renderer (`renderMd`)

Single-pass regex pipeline operating on HTML-escaped input:

| Step | Pattern | Output |
|------|---------|--------|
| 1 | Fenced code blocks | `<pre><code>` |
| 2 | Inline code | `<code>` |
| 3 | ATX headings h1–h3 | `<h1>`–`<h3>` |
| 4 | Blockquotes | `<blockquote>` |
| 5 | Horizontal rules | `<hr>` |
| 6 | Bold-italic / bold / italic | `<strong><em>` / `<strong>` / `<em>` |
| 7 | Links (http/https only) | `<a href target rel>` |
| 8 | Pipe tables | `<table><tr><th>/<td>` |
| 9 | Unordered lists | `<ul><li>` |
| 10 | Ordered lists | `<ol><li>` |
| 11 | Paragraphs | `<p>` wrapping double-newline blocks |

Link sanitisation rule (Requirement 10):
```javascript
const safe = /^https?:\/\//i.test(url.trim()) ? url.trim() : '#';
```

### 3.3 Streaming Partial Render

While streaming, the bubble is updated on every `onChunk` call:

```javascript
const lastNewline = chunk.lastIndexOf('\n');
if (lastNewline === -1) {
  bubble.innerHTML = escHtml(chunk);   // no complete line yet
} else {
  const complete   = chunk.slice(0, lastNewline);
  const incomplete = chunk.slice(lastNewline + 1);
  bubble.innerHTML = renderMd(complete) +
    (incomplete ? `<span class="streaming-tail">${escHtml(incomplete)}</span>` : '');
}
```

### 3.4 Endpoint Construction

```javascript
const endpoint =
  `https://bedrock-runtime.${REGION()}.amazonaws.com/model/${encodeURIComponent(modelId)}/converse-stream`;
```

`REGION()` reads `#region-select` at call time so region changes take effect immediately.

---

## Security Design

| Control | Implementation |
|---------|---------------|
| CSP | `<meta http-equiv="Content-Security-Policy">` — `default-src 'none'`, `script-src 'unsafe-inline'`, `style-src 'unsafe-inline'`, `connect-src` limited to 7 Bedrock runtime hostnames, `img-src data:`, `font-src 'self'` |
| XSS | `escHtml()` applied to all user text before DOM insertion; `renderMd()` operates on already-escaped input |
| Link injection | URLs not matching `^https?://` are replaced with `#` |
| Clickjacking | Frame-buster: `if (top !== self) top.location = self.location` |
| MIME sniffing | `<meta http-equiv="X-Content-Type-Options" content="nosniff">` |
| Referrer leakage | `<meta name="referrer" content="no-referrer">` |

---

## Model List

Models are grouped by provider. Claude and Nova use `us.` inference profile IDs. All other providers use direct model IDs.

| Provider | Models |
|----------|--------|
| Anthropic Claude | Claude Opus 4.6, Sonnet 4.6, Opus 4.5, Sonnet 4.5 (default), Haiku 4.5, 3.7 Sonnet, 3.5 Sonnet v2, 3.5 Haiku |
| Amazon Nova | Nova Premier, Nova 2 Lite, Nova Pro, Nova Lite, Nova Micro |
| Meta Llama | Llama 4 Maverick 17B, Scout 17B, 3.3 70B, 3.2 90B, 3.2 11B, 3.1 405B |
| DeepSeek | V3.2, R1, V3.1 |
| Mistral AI | Large 3, Magistral Small, Ministral 14B, Ministral 8B, Pixtral Large, Large 24.02 |
| Google | Gemma 3 27B, 12B, 4B |
| Qwen | Qwen3 235B, 32B, Coder 480B |
| Moonshot AI | Kimi K2 Thinking, Kimi K2.5 |
| MiniMax | M2.1, M2 |
| Writer | Palmyra X5, X4 |
| AI21 Labs | Jamba 1.5 Large, Mini |
| Cohere | Command R+, Command R |

---

## Correctness Properties

Based on the prework analysis, the following properties are verifiable:

### P1 — Conversation History Completeness (Req 6.4, 6.5)
For any sequence of N user messages sent in a session, the Nth request body's `messages` array SHALL have length `2N - 1` (N user turns interleaved with N-1 assistant turns).

### P2 — Event-Stream Chunk-Boundary Independence (Req 8.7)
For any valid event-stream byte sequence, splitting the bytes at arbitrary positions and feeding them to the parser in multiple `read()` calls SHALL produce the same accumulated `fullText` as feeding all bytes in a single call.

### P3 — Markdown Renderer Idempotence (Req 9.10)
For any Markdown string `s`, `renderMd(renderMd(s))` SHALL equal `renderMd(s)` — applying the renderer twice produces the same HTML as applying it once.

### P4 — Link Sanitisation Totality (Req 10.3)
For all possible URL strings `u`, the rendered `href` SHALL be either `u` (when `u` matches `^https?://`) or `#` (otherwise). No other value is possible.

### P5 — Inference Profile ID Prefix (Req 4.4)
For all model `<option>` elements within the "Anthropic Claude" and "Amazon Nova" `<optgroup>` elements, the `value` attribute SHALL start with `"us."`.

### P6 — Region-Aware Endpoint (Req 7.1)
For any region string `r` selected in `#region-select`, the fetch URL used by `streamBedrock()` SHALL contain the substring `bedrock-runtime.${r}.amazonaws.com`.

### P7 — XSS Escape Totality (Req 13.5)
For all strings containing `<`, `>`, or `&`, `escHtml()` SHALL replace every `&` with `&amp;`, every `<` with `&lt;`, and every `>` with `&gt;` before the string is inserted into the DOM.
