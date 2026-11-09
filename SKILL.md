---
name: browser-use
description: Local open-source Browser Use automation with Google Gemini and TypeSafe Jev Ultrafast via macOS Keychain API keys. Use for multi-step interactive pages, authenticated UI workflows, visual inspection, or ultrafast browser tasks.
license: MIT
compatibility: opencode
metadata:
  audience: developers
  category: browser-automation
---

# Browser Use

Use the local open-source Python runner at `skills/browser-use/run.py` for
browser tasks that need Gemini or TypeSafe Jev Ultrafast. Do not use Browser
Use Cloud, the hosted SDK, or the old Browser Use MCP server. The MCP
registration and launcher have been removed from this installation.

Prefer simpler tools for routine retrieval:

- Use `gh` for GitHub repositories, issues, pull requests, and releases.
- Use `curl` for straightforward web pages and endpoints.
- Read local files directly when the content is already available.

## Providers

The runner supports two local engines:

1. **`google`** (default): Multimodal Browser Use agent using Google Gemini (`gemini-flash-lite-latest`) with Playwright.
2. **`jev`**: Ultrafast agent using TypeSafe Jev (`https://github.com/browser-use/jev-ultrafast`) with dynamic indexed action spaces and Chrome CDP over Browser Harness.

### Provider Selection

The active provider can be chosen by the AI assistant or user in two ways:

1. **CLI Flag (`--provider`)**:
   Pass `--provider jev` or `--provider google` when invoking the runner:
   ```bash
   python ~/.config/opencode/skills/browser-use/run.py --provider jev \
     "Open https://en.wikipedia.org/wiki/Main_Page and find the article on Gödel's incompleteness theorems."
   ```

2. **Environment Variable / `.env` file (`BROWSER_USE_PROVIDER`)**:
   Force the provider for all runs in a shell or project:
   ```bash
   export BROWSER_USE_PROVIDER=jev
   # or
   export BROWSER_USE_PROVIDER=google
   ```
   You can also put this in a `.env` file in your working directory, skill directory, or `~/.config/opencode/.env`:
   ```bash
   BROWSER_USE_PROVIDER=jev
   ```

Precedence: CLI `--provider` flag > Active Shell Environment > `.env` file > Default (`google`).

## Keychain Setup

API keys are read securely from the macOS Keychain. Storing keys via `/usr/bin/security` prevents secrets from entering shell history or `.env` files.

### 1. TypeSafe API Key (`typesafe_ai_api`)

For Jev Ultrafast, store your TypeSafe API key under the generic password service `typesafe_ai_api`. Omitting the final `-w` value prompts securely:

```bash
security add-generic-password \
  -a "$USER" \
  -s "typesafe_ai_api" \
  -w
```

To update an existing entry, add `-U`:

```bash
security add-generic-password \
  -U \
  -a "$USER" \
  -s "typesafe_ai_api" \
  -w
```

Verify that the item is stored without printing its secret:

```bash
security find-generic-password \
  -a "$USER" \
  -s "typesafe_ai_api" \
  >/dev/null && echo "TypeSafe API key is stored"
```

#### Jev Text Helper Model

When Jev performs a text entry (`TYPE_TEXT`), it queries an OpenAI-compatible helper model. If `TEXT_MODEL_API_KEY` is not explicitly set in the environment, the runner automatically reuses the stored `browser-use-google-api-key` via Google's OpenAI-compatible endpoint (`gemini-2.5-flash`), eliminating the need for an additional third-party text model subscription.

### 2. Google Gemini API Key (`browser-use-google-api-key`)

For the standard Google Gemini provider, store the Gemini key:

```bash
security add-generic-password \
  -a "$USER" \
  -s "browser-use-google-api-key" \
  -w
```

Verify the Google item:

```bash
security find-generic-password \
  -a "$USER" \
  -s "browser-use-google-api-key" \
  >/dev/null && echo "Google API key is stored"
```

## Installation

Install the required packages in your Python environment:

```bash
# Core Browser Use and Playwright browser
uv pip install browser-use
uvx browser-use install

# Jev Ultrafast and Browser Harness
uv pip install "git+https://github.com/browser-use/jev-ultrafast.git" "browser-harness==0.1.13"
```

## Runner Usage

### Running with Jev Ultrafast

```bash
python ~/.config/opencode/skills/browser-use/run.py \
  --provider jev \
  "Open https://en.wikipedia.org/wiki/Main_Page and find the article on Gödel's incompleteness theorems."
```

You can optionally specify `--url` explicitly:

```bash
python ~/.config/opencode/skills/browser-use/run.py \
  --provider jev \
  --url "https://www.google.com/travel/flights?hl=en" \
  "Find one-way flights from Zurich to London for one adult in economy."
```

### Running with Google Gemini

```bash
python ~/.config/opencode/skills/browser-use/run.py \
  --provider google \
  "Open Google, search for clothing marketplaces, and return the main visible marketplace names and URLs."
```

For tasks supplied via stdin:

```bash
printf '%s\n' "Open https://example.com and return the page title." | \
  python ~/.config/opencode/skills/browser-use/run.py --provider jev
```

### Browser Controls

- `--headed`: Show the browser window instead of running headless.
- `--manual-captcha`: Run headed and pause if a CAPTCHA or bot check appears.
- `--max-steps N`: Set step limit (default: 100).
- `--screenshots-dir DIR`: Save page screenshots as numbered image files.
- `--record-video-dir DIR`: Record the browser session as video.
- `--traces-dir DIR`: Write execution trace data.
- `--viewport WIDTHxHEIGHT`: Set page viewport dimensions (e.g. `1440x900`).
- `--allowed-domain DOMAIN`: Restrict navigation to specific domains.
- `--prohibited-domain DOMAIN`: Block navigation to specific domains.
- `--wait-page-load SECONDS`: Minimum wait after page load (default: 0.1).
- `--wait-between-actions SECONDS`: Delay between actions (default: 0.1).

## Architecture & Abstractions

### Why LiteLLM and LangGraph are applied where appropriate

1. **LiteLLM**:
   - **Where it fits**: Standardizing chat completion requests (`/chat/completions`) for text and multimodal LLMs (OpenAI, Anthropic, Gemini, DeepSeek).
   - **Why it cannot abstract Jev's action loop**: Jev Ultrafast does not use chat completions to drive browser actions. Instead, it extracts an indexed element table via in-browser JavaScript (`snapshot.js`) and calls TypeSafe's SystemOne API (`/v1/systemone`), which performs classification across speculative heads (`operation`, `click_target`, `type_text_target`). LiteLLM does not support TypeSafe SystemOne because SystemOne is an action-space classifier rather than a generative chat model.
   - **Where LiteLLM applies to Jev**: For Jev's `TYPE_TEXT` operations, an OpenAI-compatible text completion helper is used. The runner bridges this directly with Gemini or any standard endpoint.

2. **LangGraph**:
   - LangGraph provides state graph orchestration for cyclic multi-agent loops. However, both Browser Use (`Agent.run`) and Jev Ultrafast (`Agent.run`) implement optimized, native execution loops with internal DOM staleness guards and event queues. Wrapping both distinct runtimes into an external LangGraph graph would introduce unnecessary latency, dependencies, and state boilerplate without improving code reuse.

3. **Reusability & Provider Registry (`BrowserProvider`)**:
   - Common cross-cutting concerns (CLI parsing, macOS Keychain resolution, screenshot persistence, options normalization, error formatting) are implemented once in `run.py`.
   - Engine-specific drivers (`GoogleBrowserProvider` and `JevBrowserProvider`) implement a clean `BrowserProvider` protocol and register in `PROVIDERS`.
   - New providers (e.g. `anthropic`, `openai`, `deepseek`) can be added by implementing a single `run(task, options)` method and registering in `PROVIDERS`.
