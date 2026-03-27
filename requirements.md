# Requirements Document

## Introduction

A self-contained, single HTML file chatbot that connects directly to Amazon Bedrock from the browser using a visitor-supplied Bearer token. The page requires no backend, no build step, and no external dependencies. It provides a dark-themed chat interface with streaming responses, multi-turn conversation history, markdown rendering, and token usage display.

## Glossary

- **Page**: The single HTML file delivered to the browser.
- **Chat_UI**: The visual interface composed of the header, token bar, chat area, and input bar.
- **Header**: The top bar containing the page title, region selector, and model selector.
- **Token_Bar**: The full-width bar below the header containing the Bearer token input and mask toggle.
- **Chat_Area**: The scrollable region that displays the conversation history.
- **Input_Bar**: The bottom bar containing the message textarea and send button.
- **Bearer_Token**: An AWS temporary credential token supplied by the visitor and used as the HTTP Authorization header value.
- **Region_Selector**: A `<select>` element in the Header that controls the AWS region used for API calls.
- **Model_Selector**: A `<select>` element in the Header that controls which Bedrock model is invoked.
- **Conversation_History**: The ordered list of all user and assistant messages accumulated during the session.
- **Bedrock_API**: The Amazon Bedrock Runtime `converse-stream` HTTP endpoint.
- **Event_Stream_Parser**: The client-side binary frame parser that decodes AWS event-stream frames from the response body.
- **Markdown_Renderer**: The client-side function that converts Markdown text to safe HTML.
- **Token_Usage**: The `inputTokens`, `outputTokens`, and total token counts returned by the Bedrock API.
- **Inference_Profile_ID**: A model identifier prefixed with `us.` that routes through cross-region inference.
- **CSP**: Content Security Policy, declared via a `<meta>` tag.

---

## Requirements

### Requirement 1: Single-File Delivery

**User Story:** As a developer, I want a single HTML file with no external dependencies, so that I can share or deploy the chatbot by copying one file.

#### Acceptance Criteria

1. THE Page SHALL contain all CSS, JavaScript, and markup inline within a single `.html` file.
2. THE Page SHALL NOT load any resources from external URLs (fonts, scripts, stylesheets, images).
3. THE Page SHALL embed the AWS Bedrock favicon as a base64-encoded SVG data URI in a `<link rel="icon">` element.

---

### Requirement 2: Dark-Themed Chat UI Layout

**User Story:** As a user, I want a clean dark-themed interface, so that the chatbot is comfortable to use in low-light environments.

#### Acceptance Criteria

1. THE Chat_UI SHALL render a Header, Token_Bar, Chat_Area, and Input_Bar stacked vertically to fill the full viewport height.
2. THE Chat_UI SHALL use a dark background color (`#0f1117`) for the page body.
3. THE Chat_UI SHALL display user messages right-aligned with a purple bubble (`#7c3aed`) and assistant messages left-aligned with a dark blue bubble (`#1e2235`).
4. THE Chat_UI SHALL display a bouncing three-dot typing indicator inside the assistant bubble while a response is being streamed.
5. THE Chat_UI SHALL display a blinking block cursor (`▋`) at the end of the streaming text while a response is in progress.
6. THE Chat_UI SHALL auto-scroll the Chat_Area to the bottom after each new message or streamed chunk is appended.
7. THE Chat_UI SHALL display a fixed-position error toast at the bottom center of the viewport when an error occurs, and SHALL hide it automatically after 6 seconds.

---

### Requirement 3: Region Selector

**User Story:** As a user, I want to choose the AWS region, so that I can connect to the Bedrock endpoint closest to me or required by my credentials.

#### Acceptance Criteria

1. THE Header SHALL contain a `<select>` element with the id `region-select` listing the following regions: `us-east-1`, `us-east-2`, `us-west-2`, `eu-west-1`, `eu-central-1`, `ap-northeast-1`, `ap-southeast-2`.
2. WHEN the user changes the Region_Selector, THE Bedrock_API endpoint URL SHALL use the newly selected region for all subsequent requests.

---

### Requirement 4: Model Selector

**User Story:** As a user, I want to switch between Bedrock models, so that I can compare responses or use the model best suited to my task.

#### Acceptance Criteria

1. THE Header SHALL contain a `<select>` element with the id `model-select` grouping models by provider using `<optgroup>` elements.
2. THE Model_Selector SHALL include models from the following providers: Anthropic Claude, Amazon Nova, Meta Llama, DeepSeek, Mistral AI, Google, Qwen, Moonshot AI, MiniMax, Writer, AI21 Labs, and Cohere.
3. THE Model_Selector SHALL use Inference_Profile_IDs (prefixed with `us.`) for all Anthropic Claude and Amazon Nova models.
4. FOR ALL models in the Anthropic Claude and Amazon Nova optgroups, the model value attribute SHALL start with the prefix `us.`.
5. WHEN the user changes the Model_Selector, THE Bedrock_API endpoint URL SHALL use the newly selected model ID for all subsequent requests.

---

### Requirement 5: Bearer Token Input

**User Story:** As a user, I want to paste my Bearer token into a dedicated input, so that the page can authenticate requests to Bedrock on my behalf.

#### Acceptance Criteria

1. THE Token_Bar SHALL contain a full-width text input with the id `token-input` for entering the Bearer_Token.
2. THE Token_Bar SHALL contain a toggle button with the id `mask-btn` that switches the `token-input` between `type="text"` and `type="password"`.
3. WHEN the mask toggle is in the masked state, THE `mask-btn` SHALL display the `🙈` emoji.
4. WHEN the mask toggle is in the unmasked state, THE `mask-btn` SHALL display the `👁` emoji.
5. WHEN the user attempts to send a message without a Bearer_Token, THE Chat_UI SHALL display an error toast with the message "Paste your Bearer token first."

---

### Requirement 6: Multi-Turn Conversation History

**User Story:** As a user, I want the chatbot to remember the full conversation, so that I can have coherent multi-turn dialogues.

#### Acceptance Criteria

1. THE Page SHALL maintain a Conversation_History array in memory for the duration of the browser session.
2. WHEN the user sends a message, THE Page SHALL append a user turn to the Conversation_History before sending the request.
3. WHEN the Bedrock_API returns a complete assistant response, THE Page SHALL append an assistant turn containing the full response text to the Conversation_History.
4. FOR ALL requests sent to the Bedrock_API, the request body SHALL include the complete Conversation_History accumulated up to that point.
5. WHEN the user sends the Nth message in a session, the request body SHALL contain exactly N user turns and N-1 assistant turns.

---

### Requirement 7: Streaming via Bedrock converse-stream

**User Story:** As a user, I want responses to appear word-by-word as they are generated, so that I can start reading immediately without waiting for the full reply.

#### Acceptance Criteria

1. THE Page SHALL send POST requests to the endpoint `https://bedrock-runtime.{region}.amazonaws.com/model/{modelId}/converse-stream`.
2. THE Page SHALL include the `Authorization: Bearer {token}` header and `Content-Type: application/json` header on every request.
3. THE Page SHALL include `inferenceConfig` with `maxTokens: 4096` and `temperature: 0.7` in every request body.
4. WHEN the Bedrock_API returns a non-2xx HTTP status, THE Page SHALL throw an error containing the HTTP status code and response body text.
5. THE Page SHALL read the response body as a `ReadableStream` and process frames incrementally as bytes arrive.

---

### Requirement 8: AWS Binary Event-Stream Frame Parser

**User Story:** As a developer, I want the page to correctly parse AWS binary event-stream frames, so that streamed response chunks are decoded reliably.

#### Acceptance Criteria

1. THE Event_Stream_Parser SHALL parse each frame using the structure: 4-byte total length, 4-byte headers length, 4-byte prelude CRC, variable-length headers, variable-length payload, 4-byte message CRC.
2. THE Event_Stream_Parser SHALL buffer incoming bytes and SHALL NOT attempt to parse a frame until the full `totalLen` bytes are available in the buffer.
3. THE Event_Stream_Parser SHALL advance the buffer past each fully parsed frame before processing the next frame.
4. THE Event_Stream_Parser SHALL extract JSON objects from the frame payload using brace-depth matching to handle multiple JSON objects in a single payload.
5. WHEN a frame payload contains a `delta.text` field, THE Event_Stream_Parser SHALL append the text to the accumulated full-text string and invoke the streaming callback.
6. WHEN a frame payload contains a `usage.inputTokens` field, THE Event_Stream_Parser SHALL store the usage object for display after streaming completes.
7. FOR ALL valid event-stream byte sequences, parsing the frames SHALL extract the same text content regardless of how the bytes are chunked across `reader.read()` calls (chunk-boundary independence).

---

### Requirement 9: Markdown Rendering

**User Story:** As a user, I want assistant responses rendered as formatted text, so that code blocks, lists, and headings are readable.

#### Acceptance Criteria

1. THE Markdown_Renderer SHALL convert fenced code blocks (triple backtick) to `<pre><code>` elements.
2. THE Markdown_Renderer SHALL convert inline code (single backtick) to `<code>` elements.
3. THE Markdown_Renderer SHALL convert ATX headings (`#`, `##`, `###`) to `<h1>`, `<h2>`, `<h3>` elements.
4. THE Markdown_Renderer SHALL convert blockquotes (`>`) to `<blockquote>` elements.
5. THE Markdown_Renderer SHALL convert horizontal rules (`---`) to `<hr>` elements.
6. THE Markdown_Renderer SHALL convert bold (`**text**`), italic (`*text*`), and bold-italic (`***text***`) to `<strong>`, `<em>`, and `<strong><em>` elements respectively.
7. THE Markdown_Renderer SHALL convert Markdown links to `<a>` elements with `target="_blank"` and `rel="noopener noreferrer"`.
8. THE Markdown_Renderer SHALL convert simple pipe-delimited tables to `<table>` elements with `<th>` for the header row and `<td>` for data rows.
9. THE Markdown_Renderer SHALL convert unordered lists (`-`, `*`, `+`) to `<ul><li>` elements and ordered lists (`1.`) to `<ol><li>` elements.
10. WHEN the Markdown_Renderer is applied to the same input twice, THE output SHALL be identical to applying it once (idempotence of the rendered HTML structure).
11. WHILE streaming is in progress, THE Page SHALL render all complete lines (up to the last newline) as Markdown HTML and SHALL render the incomplete trailing line as escaped plain text.
12. WHEN streaming completes, THE Page SHALL render the full accumulated text through the Markdown_Renderer and replace the bubble content.

---

### Requirement 10: XSS-Safe Link Sanitization

**User Story:** As a user, I want links in assistant responses to be safe, so that malicious URLs cannot execute scripts or redirect to dangerous pages.

#### Acceptance Criteria

1. THE Markdown_Renderer SHALL only render links whose URL begins with `http://` or `https://` as clickable `<a>` elements.
2. WHEN a Markdown link URL does not begin with `http://` or `https://`, THE Markdown_Renderer SHALL replace the `href` attribute value with `#`.
3. FOR ALL possible Markdown link URL strings, the rendered `href` SHALL either be the original URL (if it starts with `http://` or `https://`) or `#` (otherwise).

---

### Requirement 11: Token Usage Display

**User Story:** As a user, I want to see how many tokens each exchange consumed, so that I can monitor API usage.

#### Acceptance Criteria

1. THE Chat_UI SHALL render a token usage row below each assistant message bubble after streaming completes.
2. THE token usage row SHALL display input token count in blue (`#818cf8`), output token count in green (`#34d399`), and total token count in grey.
3. WHEN the Bedrock_API does not return usage data, THE token usage row SHALL remain hidden.

---

### Requirement 12: Input Bar Behaviour

**User Story:** As a user, I want a comfortable message input area, so that I can type and send messages efficiently.

#### Acceptance Criteria

1. THE Input_Bar SHALL contain a `<textarea>` with the id `user-input` that auto-resizes vertically up to a maximum height of 140px as the user types.
2. WHEN the user presses Enter without Shift, THE Page SHALL submit the message.
3. WHEN the user presses Shift+Enter, THE Page SHALL insert a newline in the textarea without submitting.
4. WHEN a request is in flight, THE send button SHALL be disabled and SHALL NOT accept further clicks.
5. WHEN a request completes or fails, THE send button SHALL be re-enabled and focus SHALL return to the textarea.

---

### Requirement 13: Security Controls

**User Story:** As a security-conscious developer, I want the page to apply browser security controls, so that it is safe to open from any origin.

#### Acceptance Criteria

1. THE Page SHALL include a `<meta http-equiv="Content-Security-Policy">` tag that sets `default-src 'none'`, allows only inline scripts and styles, restricts `connect-src` to the specific Bedrock runtime hostnames for the supported regions, allows `img-src data:`, and allows `font-src 'self'`.
2. THE Page SHALL include a `<meta http-equiv="X-Content-Type-Options" content="nosniff">` tag.
3. THE Page SHALL include a `<meta name="referrer" content="no-referrer">` tag.
4. THE Page SHALL include an inline script that redirects the top-level frame to the page URL if the page is loaded inside an iframe (`if (top !== self) top.location = self.location`).
5. THE Page SHALL escape all user-supplied text using HTML entity encoding before inserting it into the DOM.
