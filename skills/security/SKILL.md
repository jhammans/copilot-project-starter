---
applyTo: "**"
---

# Security Skill Index

This skill provides security engineering guidance across all stages of the SDLC. It covers OWASP Top 10, IAM, RBAC, threat modeling, and secure coding patterns.

## When to Apply

- **Architecture stage:** Use threat modeling and security architecture sections
- **Implementation stage:** Use secure coding patterns and OWASP controls
- **Code review stage:** Use the review checklist
- **Pre-deployment:** Use the deployment security checklist

## Sub-Skills

| File | Purpose |
|------|---------|
| `owasp-top10.instructions.md` | OWASP Top 10 controls and code patterns |
| `iam.instructions.md` | Identity and Access Management standards |
| `rbac.instructions.md` | Role-Based Access Control design patterns |
| `threat-modeling.instructions.md` | STRIDE threat modeling process |

## Security is Non-Negotiable

Security is a **first-class requirement**, not an afterthought. Every agent and skill in this project applies security controls at every stage. These are not optional.

If a user or stakeholder asks to skip or defer a security control:
1. Explain the specific risk of doing so
2. Document the decision and the person who approved it
3. Create a tracked issue to remediate it
4. Never silently skip a security control

## Quick Reference — Security Red Flags

When you see any of the following, treat it as a blocking security issue:

- Passwords or tokens in source code, config files, or environment variables checked into source control
- `SELECT * FROM users WHERE username='` + userInput (SQL injection)
- `eval(userInput)`, `exec(userInput)`, `new Function(userInput)` (code injection)
- `response.setHeader('Access-Control-Allow-Origin', '*')` on authenticated endpoints (CORS misconfiguration)
- User role/permissions stored in a JWT or cookie that the server doesn't validate server-side
- `http://` instead of `https://` for any external call
- `console.log(user)`, `logger.info(request.body)` where body may contain PII or credentials
- Unhandled promise rejections or exceptions that expose stack traces to the client
- Hard-coded API keys: `const API_KEY = "sk-..."` or `AWS_SECRET_ACCESS_KEY = "..."`
- Missing rate limiting on authentication endpoints
- Missing CSRF protection on state-changing form endpoints
