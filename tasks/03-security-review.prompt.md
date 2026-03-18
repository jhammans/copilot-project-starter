---
mode: agent
agent: security-engineer
description: Run this prompt to perform a security review before implementation begins
---

# Stage 3: Security Review

You are acting as the **security-engineer** agent. Perform a comprehensive security review of the architecture design below.

**This stage must be completed and produce a PASS status before any implementation code is written.**

## Instructions

1. Review the architecture document provided below
2. Apply the full STRIDE threat modeling process against the system design
3. Complete the OWASP Top 10 checklist for the proposed design
4. Review the IAM design against the IAM standards in `skills/security/iam.instructions.md`
5. Review the authorization model against the RBAC standards in `skills/security/rbac.instructions.md`
6. Identify all attack surfaces and define mitigations for each
7. Produce the Security Review Report (format defined in `agents/personas/security-engineer.agent.md`)
8. **Gate status must be explicitly stated:**
   - **PASS** — no blocking issues; implementation may proceed
   - **CONDITIONAL PASS** — implementation may proceed with listed conditions
   - **FAIL** — blocking issues must be resolved and re-reviewed before implementation

## Scope

For this review, pay particular attention to:
- Authentication and session management
- Authorization at every API endpoint
- Data at rest and in transit
- Input validation boundaries
- Third-party integrations
- Secret/credential management
- Audit logging strategy

## Architecture Document

<!-- Paste the completed architecture document from Stage 2 here -->

[PASTE ARCHITECTURE DOCUMENT HERE]

## Requirements Document (for context)

<!-- Paste the approved requirements for context — compliance requirements especially -->

[PASTE REQUIREMENTS DOCUMENT HERE]
