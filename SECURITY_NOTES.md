# Security Notes

Date: 2026-04-12

## Scope
Repository-wide review of local registration scripts under `exa-register/`, `grok-register/`, `openai-register/`, and `tavily-register/`.

## Summary
This repository appears to be designed for local, operator-run automation rather than a deployed multi-user service. That lowers the severity of some findings, but it does not make all current behaviors reasonable. The main risks are secret hygiene issues, not remote code execution.

## Findings

### 1. Fixed default password is not reasonable even for local use
- `tavily-register/batch_signup.py:27` hard-codes `PASSWORD = "Tavily@2024Test"`.
- The same value is exposed as the CLI default at `tavily-register/batch_signup.py:683`.
- Risk: all created accounts may share a predictable password. This affects the security of the registered accounts themselves, not just the local machine.

### 2. Sensitive values are printed or stored in plaintext
- `grok-register/grok.py:183-184` writes SSO tokens and `email:password:SSO` records to disk.
- `openai-register/openai_register.py:1744,1747,1753` prints credentials and writes account/password plus token JSON to `tokens/`.
- `exa-register/exa_browser_solver.py:552-556` prints the API key after success.
- `openai-register/openai_register.py:1420,1530` logs OTP codes.
- Assessment: this may be acceptable for a strictly local, single-user workflow, but only as an explicit tradeoff. It is still weak secret handling.

### 3. High-value credentials are sent to operator-supplied external base URLs without validation
- `tavily-register/gptmail_client.py:73-125` sends `X-API-Key` to arbitrary configured `base_url`.
- `openai-register/openai_register.py:616-634,747-753,828-835` sends Sub2API admin credentials, bearer tokens, and account tokens to configured `SUB2API_BASE_URL`.
- Risk: an unsafe `.env` or CLI parameter can exfiltrate secrets. This is closer to misconfiguration risk than classic SSRF, but it is still a trust-boundary weakness.

## Practical Severity For This Repo

### Acceptable with explicit risk acceptance
- Local plaintext runtime artifacts.
- Local token files under ignored directories.

### Should be improved by default
- Printing full secrets to stdout instead of masking.
- Sending secrets to arbitrary configured URLs without scheme/host validation.

### Should be fixed
- Fixed default password for created accounts.

## Recommended Minimum Improvements
1. Replace fixed default passwords with per-account random passwords.
2. Mask API keys, tokens, passwords, and OTPs in normal logs.
3. Require an explicit debug flag for full secret output.
4. Validate configured remote base URLs at minimum for scheme and obvious private/loopback targets where inappropriate.
