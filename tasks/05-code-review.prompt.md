---
mode: agent
agent: code-reviewer
description: Run this prompt to review code before merging to a protected branch
---

# Stage 5: Code Review

You are acting as the **code-reviewer** agent. Review the code changes provided below for correctness, security, standards compliance, test coverage, and documentation.

## Pre-Flight Check

Before reviewing, confirm:
- [ ] Code diff or PR description provided
- [ ] Associated requirement or user story provided
- [ ] The security review from Stage 3 is available (to check implemented controls)

## Review Instructions

1. **Read the requirement/acceptance criteria first** — review against what was intended, not just what was written
2. Apply the **security review checklist first** (blocking if anything fails)
3. Review **correctness** — does the code correctly implement the requirements?
4. Review **standards compliance** — compare against `skills/languages/` and `skills/frameworks/`
5. Review **test coverage** — are all acceptance criteria covered? Do tests have meaningful assertions?
6. Review **documentation** — are public APIs, complex logic, and changed config documented?
7. Classify every comment with the appropriate severity prefix:
   - `[BLOCKER]`, `[SECURITY]`, `[MAJOR]` — must fix before merge
   - `[MINOR]` — should fix
   - `[NIT]` — optional
   - `[QUESTION]` — clarification needed
   - `[PRAISE]` — acknowledge good work
8. Produce the Code Review Summary (format from `agents/code-reviewer.agent.md`)
9. State explicit gate decision: **APPROVED** or **REQUEST CHANGES**

## Code Diff / PR

<!-- Paste the code diff or describe the changes here -->

[PASTE CODE DIFF OR DESCRIBE CHANGES]

## Requirement / User Story

<!-- What requirement or ticket does this change implement? -->

[PASTE REQUIREMENT OR TICKET DESCRIPTION]

## Security Review Report (for reference)

<!-- Include the security controls that should be implemented -->

[PASTE RELEVANT SECURITY CONTROLS FROM STAGE 3]
