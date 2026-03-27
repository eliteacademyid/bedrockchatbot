Build a clean, minimal, single-file AI chatbot page (index.html) that talks directly to Amazon Bedrock from the browser using a Bearer token API. No backend, no dependencies, no npm — just one self-contained HTML file ready to drop on any static host.

UI

Dark theme (#0f1117 background, purple accent #7c3aed)

Header with app title, region selector, and model switcher dropdown

Dedicated token bar below the header — full-width input for pasting the Bearer token, with a 👁 toggle to mask/unmask it

Chat area with user bubbles (right, purple) and assistant bubbles (left, dark)

Auto-resizing textarea input, Enter to send, Shift+Enter for newline

Token usage stats shown below each assistant reply (input tokens in indigo, output in green)

Official AWS Bedrock favicon embedded as base64 SVG inline

Models Include all currently available Bedrock text models grouped by provider: Anthropic Claude (Opus/Sonnet/Haiku 4.x, 3.7, 3.5), Amazon Nova (Premier, 2 Lite, Pro, Lite, Micro), Meta Llama (4 Maverick/Scout, 3.3, 3.2, 3.1), DeepSeek (V3.2, R1, V3.1), Mistral AI (Large 3, Magistral Small, Ministral 14B/8B, Pixtral Large), Google Gemma 3, Qwen3, Moonshot Kimi, MiniMax, Writer Palmyra, AI21 Jamba, Cohere Command. Use us. prefixed cross-region inference profile IDs for Claude and Nova.

Streaming Use the Bedrock converse-stream endpoint. Parse the AWS binary event-stream format incrementally — each frame has a 12-byte prelude (total_len:4, headers_len:4, prelude_crc:4), variable-length headers, a JSON payload, and a 4-byte trailing CRC. Extract JSON objects from each frame payload by brace-depth matching. Call the render callback on every chunk so text appears word by word.

Markdown rendering Render assistant responses as markdown with no external libraries. While streaming, render completed lines as markdown and keep the current incomplete line as plain text with a blinking cursor. On stream completion, do a full markdown render pass. Support: headings, bold/italic, inline code, fenced code blocks, blockquotes, horizontal rules, ordered/unordered lists, tables, and links (http/https only — strip javascript: hrefs).

Security

Content Security Policy meta tag: lock connect-src to explicit Bedrock runtime hostnames per region, block everything else

X-Content-Type-Options: nosniff and Referrer-Policy: no-referrer meta tags

Frame-busting script to prevent clickjacking

Link sanitization in the markdown renderer — only http:// and https:// URLs allowed in hrefs

Regions supported us-east-1, us-east-2, us-west-2, eu-west-1, eu-central-1, ap-northeast-1, ap-southeast-2 — selectable from the header dropdown, used dynamically in both the fetch endpoint and the CSP.