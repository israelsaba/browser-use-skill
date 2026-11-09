# Security Policy

Do not report key leaks, anti-bot bypasses, or security vulnerabilities in public issues. Never post API keys, cookies, browser storage state, or private URLs.

Use GitHub private vulnerability reporting or contact the repository maintainer through GitHub. Include the affected commit, safe reproduction steps, impact, and a remediation suggestion when possible.

### Trust Boundaries and Security Model

- **Keychain Storage**: API keys are read from the macOS Keychain (`security find-generic-password`). Never store API keys in plain text files or version control.
- **Remote Execution**: Browser actions interact with the public web. Do not pass untrusted instructions that automate financial transactions or confidential workflows without independent confirmation.
- **Artifacts**: HAR files, videos, traces, and screenshots can capture sensitive DOM content and headers. Treat generated artifacts in `artifacts/` as confidential.
