# Grok Register Security Notes

Date: 2026-04-12

## Scope
Focused review of `grok-register/` only.

## Summary
The main risk in this subproject is weak handling of secrets. I did not find obvious command injection, unsafe `eval/exec`, or disabled TLS verification. The most important issues are plaintext credential storage, OTP logging, and over-trusting operator-supplied service endpoints.

## Findings

### 1. Plaintext credential storage is the highest-risk behavior
- `grok.py:183` writes raw SSO tokens to `keys/grok.txt`.
- `grok.py:184` writes `email:password:sso` to `keys/accounts.txt`.
- Impact: anyone with access to the working directory, backups, synced folders, or copied artifacts can reuse valid credentials directly.

### 2. OTP values are printed to stdout
- `grok.py:73` logs the email verification code before validation.
- Impact: OTP exposure is shorter-lived than token exposure, but still expands the leak surface through terminal history, recordings, and log capture.

### 3. Trust boundary for mail-provider configuration is too loose
- `email_service.py:33-39` accepts `LUCKMAIL_BASE_URL`, API key, and secret directly from environment variables.
- `email_service.py:112-118` initializes the LuckMail client with those values without validation.
- Impact: if `.env` or runtime configuration is tampered with, high-value API credentials can be sent to an arbitrary remote endpoint.

### 4. GPTMail token handling depends on scraping a third-party page
- `email_service.py:60-67` fetches the provider homepage, extracts a JWT-like string from HTML, and uses it as `x-inbox-token`.
- Impact: this is not an immediate exploit by itself, but it is a fragile trust model and can behave unpredictably if the provider changes response format.

## Practical Severity For This Subproject

### Should be fixed first
1. Plaintext storage of `email:password:sso`.
2. Plaintext storage of raw SSO tokens.

### Should be improved by default
1. Mask OTPs in logs.
2. Validate `LUCKMAIL_BASE_URL` at least for scheme and obvious unsafe targets.

### Lower concern for now
- Browser-like request impersonation.
- Vendored `luckmail` HTTP client. No obvious `verify=False`-style issue was found in the current code review.

## Recommended Minimum Improvements
1. Add a masked logging mode as the default.
2. Require an explicit opt-in to save full credentials locally.
3. Restrict or validate remote mail-provider base URLs.
4. Separate account metadata from bearer-equivalent tokens where possible.
