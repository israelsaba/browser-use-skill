# Contributing

Contributions are welcome through pull requests.

### Development Guidelines

1. Keep `SKILL.md` as the main skill entry point. The skill must be usable by copying the directory alone.
2. Verify code quality:
   - Run `python3 -m py_compile run.py`
   - Run `python3 run.py --help`
   - Verify that both `google` and `jev` providers resolve correctly.
3. Discuss new provider additions or major architectural changes in an issue first.
4. Keep dependencies minimal and use standard library facilities where feasible.

### Submitting Issues

Include:
- Agent name and version (OpenCode, Claude Code, Codex, Hermes)
- Operating system and Python version
- Provider used (`google` or `jev`)
- Minimal reproducible example
- Remove all credentials, cookies, and personal data before posting.

Security vulnerabilities belong in [SECURITY.md](SECURITY.md), not public issues.
