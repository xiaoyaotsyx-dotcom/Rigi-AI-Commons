# Security Policy

## Reporting a Vulnerability

If you discover a security vulnerability in Rigi AI Commons, please **DO NOT** open a public issue.

Email: **xiaoyao@aibook.online**

We will respond within 48 hours and work with you to assess and fix the issue.

## Scope

This policy covers:

- Skill files that could cause unintended behavior
- Scripts that could expose user data
- CDP/browser automation security issues
- Hardcoded credentials or API keys accidentally committed

## What We Ask

- Give us reasonable time to fix before public disclosure
- Don't exploit the vulnerability beyond what's needed to demonstrate it
- Don't access or modify other users' data

## Our Commitment

- We'll acknowledge your report within 48 hours
- We'll keep you updated on progress
- We'll credit you in the fix (unless you prefer anonymity)
- We won't pursue legal action for responsible disclosure

---

## Security Design Principles

1. **Local-first:** All workflows run on the user's own machine. No data leaves their computer.
2. **No cloud dependency:** Skills don't call home. No telemetry.
3. **User's browser, user's login:** We never ask for or store passwords, cookies, or API keys.
4. **Open source:** Every line of code is visible and auditable.

## What to Check When Submitting a Skill

- [ ] No hardcoded API keys or tokens
- [ ] No `eval()` or `exec()` on untrusted input
- [ ] File paths use proper sanitization
- [ ] CDP connections only to user's own Chrome
- [ ] No data exfiltration to external servers
