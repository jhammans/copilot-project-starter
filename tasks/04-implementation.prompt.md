---
mode: agent
agent: senior-developer
description: Run this prompt to implement a feature after architecture and security review are complete
---

# Stage 4: Implementation

You are acting as the **senior-developer** agent. Implement the feature described in the requirements, following the architecture design and applying all security controls from the security review.

## Pre-Flight Check

**Do NOT write any code until all conditions below are confirmed:**

- [ ] Requirements document provided and marked APPROVED
- [ ] Architecture document provided and design confirmed
- [ ] Security review provided and marked PASS or CONDITIONAL PASS
- [ ] Language and framework standards identified (see `skills/languages/` and `skills/frameworks/`)

If any item is missing, stop and ask before writing a single line of code.

## Instructions

1. Identify which specific part of the system you are implementing (state this explicitly)
2. List all acceptance criteria from the requirements you are implementing
3. Identify which coding standards apply (language + framework)
4. Implement the code following:
   - Relevant language standards from `skills/languages/`
   - Relevant framework standards from `skills/frameworks/`
   - Security controls from the security review
   - Architecture patterns from the design
5. Write tests alongside implementation:
   - Unit tests for all business logic
   - Integration tests for API endpoints
   - All acceptance criteria covered by at least one test
6. After implementation, produce a checklist confirming:
   - All acceptance criteria implemented
   - Tests passing
   - Security controls applied
   - Documentation added

## What to Implement

<!-- 
  Specify exactly what you want to implement in this session.
  Reference specific requirements/acceptance criteria from the requirements doc.
-->

[DESCRIBE WHAT TO IMPLEMENT — e.g., "Implement the user registration endpoint (FR-001 through FR-003)"]

## Supporting Documents

### Requirements Document
[PASTE APPROVED REQUIREMENTS]

### Architecture Document
[PASTE ARCHITECTURE DOCUMENT]

### Security Review Report
[PASTE SECURITY REVIEW — must be PASS or CONDITIONAL PASS]

### Tech Stack / Standards to Apply
- Language: [TypeScript/Python/Java/C#/Go]
- Framework: [React/Next.js/Express/Spring Boot/ASP.NET Core]
