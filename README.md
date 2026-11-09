---
type: knowledge_bundle_index
title: Browser Use Skill
description: An Agent Skill that gives AI coding agents local, open-source browser automation with Google Gemini and TypeSafe Jev Ultrafast via macOS Keychain API keys.
tags:
  [
    agent-skills,
    browser-use,
    browser-automation,
    jev-ultrafast,
    typesafe,
    gemini,
    web-agents,
  ]
timestamp: 2026-11-09T00:00:00-03:00
---

# Browser Use Skill

This repository publishes the `browser-use` Agent Skill. It equips AI coding agents with local open-source browser automation supporting two high-performance backends:

1. **Google Gemini (`google`)**: Multimodal Playwright agent using Gemini Flash (`gemini-flash-lite-latest`) for visual inspection, multi-step navigation, and structured DOM reasoning.
2. **TypeSafe Jev Ultrafast (`jev`)**: Ultra-low-latency agent powered by [browser-use/jev-ultrafast](https://github.com/browser-use/jev-ultrafast) with dynamic indexed action spaces, speculative decision heads, and Chrome CDP execution via Browser Harness.

It works with **OpenCode**, **Claude Code**, **Codex**, **Hermes**, and any agent that loads standard `SKILL.md` definitions and can execute shell commands.

## Installation

Review `SKILL.md` and `run.py` first. Browser automation agents interact with web pages and run with your local permissions.

### 1. Install Skill into Your Agent

Clone the repository and copy the skill into your agent's directory:

```bash
git clone --depth 1 https://github.com/israelsaba/browser-use-skill.git
mkdir -p ~/.config/opencode/skills/browser-use
cp browser-use-skill/SKILL.md browser-use-skill/run.py ~/.config/opencode/skills/browser-use/
```

Other agents load the same files from their respective user directories:

| Agent | User Skill Directory | Project Skill Directory |
| --- | --- | --- |
| **OpenCode** | `~/.config/opencode/skills/browser-use/` | `.opencode/skills/browser-use/` |
| **Claude Code** | `~/.claude/skills/browser-use/` | `.claude/skills/browser-use/` |
| **Codex** | `~/.agents/skills/browser-use/` | `.agents/skills/browser-use/` |
| **Hermes** | `~/.hermes/skills/browser-use/` | `.hermes/skills/browser-use/` |

> *Tip: Restart your agent session after installing so the new skill definition is loaded.*

### 2. Install Python Dependencies

In your active Python environment:

```bash
# Core Browser Use and Playwright browser
uv pip install browser-use
uvx browser-use install

# Jev Ultrafast and Browser Harness
uv pip install "git+https://github.com/browser-use/jev-ultrafast.git" "browser-harness==0.1.13"
```

---

## Keychain Setup

API credentials are read securely from the macOS Keychain (`/usr/bin/security`) so secrets never land in shell history, logs, or repository commits.

### TypeSafe Jev (`typesafe_ai_api`)

Store your TypeSafe API key under the generic password service `typesafe_ai_api`:

```bash
security add-generic-password \
  -a "$USER" \
  -s "typesafe_ai_api" \
  -w
```
*(The command prompts securely for the key).*

To verify the item exists without revealing the secret:
```bash
security find-generic-password \
  -a "$USER" \
  -s "typesafe_ai_api" \
  >/dev/null && echo "TypeSafe API key is stored"
```

#### Jev Text Helper Model
When Jev encounters a text input field (`TYPE_TEXT`), an OpenAI-compatible helper model supplies the field value. If `TEXT_MODEL_API_KEY` is not set in the environment, the runner automatically bridges the helper to Google Gemini's OpenAI-compatible endpoint using your stored `browser-use-google-api-key`, eliminating the need for an extra subscription.

### Google Gemini (`browser-use-google-api-key`)

For the standard Google Gemini provider:

```bash
security add-generic-password \
  -a "$USER" \
  -s "browser-use-google-api-key" \
  -w
```

Verify:
```bash
security find-generic-password \
  -a "$USER" \
  -s "browser-use-google-api-key" \
  >/dev/null && echo "Google API key is stored"
```

---

## Provider Selection

The runner allows seamless switching between Google and JEV.

### Precedence:
$$\text{CLI Flag } (`--provider`) \longrightarrow \text{Shell Env } (`BROWSER_USE_PROVIDER`) \longrightarrow \text{.env File } \longrightarrow \text{Default } (`google`)$$

### 1. Selected by the Calling AI Assistant
The assistant or user can pass `--provider jev` or `--provider google`:
```bash
# Run with Jev Ultrafast
python ~/.config/opencode/skills/browser-use/run.py \
  --provider jev \
  "Open https://en.wikipedia.org/wiki/Main_Page and find the article on Gödel's incompleteness theorems."

# Run with Google Gemini
python ~/.config/opencode/skills/browser-use/run.py \
  --provider google \
  "Search Google for clothing marketplaces and list the top visible names."
```

### 2. Forced via Environment Variable or `.env` File
To set the default provider across all runs:
```bash
export BROWSER_USE_PROVIDER=jev
```

Or place it in a `.env` file in your working directory, skill directory, or `~/.config/opencode/.env`:
```dotenv
BROWSER_USE_PROVIDER=jev
```

---

## Browser Controls & Diagnostics

The runner passes standardized controls across providers:

- `--headed`: Show the browser window instead of running headless.
- `--manual-captcha`: Stop and alert for manual intervention if a CAPTCHA or bot check appears.
- `--max-steps N`: Limit agent actions (default: 100).
- `--url URL`: Specify starting URL directly (inferred from task if omitted).
- `--screenshots-dir DIR`: Save screenshots as numbered image files.
- `--record-video-dir DIR`: Record the session as an MP4 video.
- `--traces-dir DIR`: Export execution trace files.
- `--viewport WIDTHxHEIGHT`: Set custom viewport (e.g. `1440x900`).
- `--allowed-domain DOMAIN`: Whitelist navigation domains.
- `--prohibited-domain DOMAIN`: Blacklist navigation domains.

Example diagnostic run:
```bash
python ~/.config/opencode/skills/browser-use/run.py \
  --provider jev \
  --headed \
  --screenshots-dir ./artifacts/screenshots \
  "Open https://example.com and extract header links."
```

---

## Architecture & Abstractions

### Evaluation of LiteLLM, LangGraph, and Core Design

1. **LiteLLM**:
   - Standardizes `/chat/completions` across LLM providers.
   - **Why LiteLLM cannot run Jev's loop**: Jev does not use chat completions to drive browser actions. Instead, `snapshot.js` injects an atomic DOM reader that extracts an indexed control table. Jev then calls TypeSafe's SystemOne API (`POST /v1/systemone`), which executes speculative classification over discrete heads (`operation`, `click_target`, `type_text_target`). SystemOne is an action-space classifier, not a text chat API.
   - **Where LiteLLM applies**: LiteLLM/OpenAI-compatible format is used for Jev's `TYPE_TEXT` helper model.

2. **LangGraph**:
   - Provides cyclic graph orchestration. Both `browser-use` and `jev-ultrafast` implement native, highly optimized execution runtimes with DOM staleness checking and CDP event loops. Wrapping them in an external LangGraph state graph would introduce unnecessary state channel overhead and latency without adding reuse.

3. **Pluggable Provider Architecture**:
   - Shared concerns (CLI parsing, macOS Keychain secrets, screenshot decoding, `.env` resolution, exit codes) are unified in `run.py`.
   - Providers implement a minimal `BrowserProvider` protocol and register in `PROVIDERS`. New engines can be added with a single class without modifying the execution harness.

---

## Security Model

- **Local-first execution**: All automation runs locally in your browser.
- **Keychain protection**: Secrets stay in the macOS Keychain under `$USER`.
- **Human-in-the-loop safety**: CAPTCHA and anti-bot challenges halt execution when `--manual-captcha` is set. The runner does not attempt stealth evasion or automated challenge bypass.

---

## Sources & Provenance

- [Browser Use (Official Repository)](https://github.com/browser-use/browser-use)
- [Jev Ultrafast (Official Repository)](https://github.com/browser-use/jev-ultrafast)
- [TypeSafe AI Documentation](https://docs.typesafe.ai/)
- [Browser Harness](https://github.com/browser-use/browser-harness)

---

## License & Contributing

Licensed under the [MIT License](LICENSE). Contributions welcome via [pull requests](https://github.com/israelsaba/browser-use-skill/pulls).
Please review [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md).
