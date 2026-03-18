---
applyTo: "**"
---

# Skill: Requirements Gathering & SDLC Gating

This skill defines the requirements elicitation process and enforces gate-based progression through the SDLC. It applies to all interactions.

## Core Rule

**Never write code, propose a database schema, select a technology, or produce a system design until requirements have been gathered, clarified, and explicitly approved.**

If you are asked to "build X" or "create Y" without an accompanying requirements document, your first response must be to begin the requirements gathering process using the questions below.

## Requirements Gathering Workflow

### Phase 1 — Discovery (4–6 questions max per round)

Ask questions in this order of priority:

1. **What problem are you solving?** (Not "what do you want to build" — "what problem needs solving?")
2. **Who are the users?** (Roles, technical level, volume of users)
3. **What does success look like?** (How will you know this is done and working?)
4. **What constraints exist?** (Must use X tech, must integrate with Y system, $Z budget, N-week timeline)
5. **What is the risk of getting this wrong?** (Informs how thorough we need to be)
6. **What already exists?** (Existing systems, code, or data we need to work with?)

### Phase 2 — Functional Deep Dive

For each feature or capability mentioned:
- What exactly does the user do? (Action)
- What does the system respond with? (Result)
- What can go wrong? (Error cases)
- How does success look? (Acceptance criterion)

### Phase 3 — Non-Functional Requirements

Always ask about:

| Category | Key Questions |
|----------|--------------|
| **Performance** | How many concurrent users? What is the acceptable response time? |
| **Availability** | What's the acceptable downtime? Do you need 99.9%, 99.99%? |
| **Security** | Who should have access? Any compliance requirements (HIPAA, GDPR, PCI, SOC2)? |
| **Scalability** | Expected growth over 12–24 months? |
| **Observability** | What does ops need to monitor? Who gets paged when it breaks? |
| **Data** | How much data? How sensitive? How long is it retained? |

### Phase 4 — Confirm & Document

After gathering information:
1. Produce a structured requirements summary
2. Ask: _"Does this accurately capture what you need? Anything missing or incorrect?"_
3. Iterate until the stakeholder confirms the requirements are complete
4. Explicitly state: _"Requirements confirmed. Moving to Architecture stage."_

## Ambiguity Resolution Rules

Any of these patterns signals an ambiguity that must be resolved before proceeding:

| Vague Phrase | Required Clarification |
|-------------|------------------------|
| "It should be fast" | Define p50/p95/p99 latency in ms |
| "It should be secure" | Define authentication method, data classification, compliance requirements |
| "It should scale" | Define expected load and growth projections |
| "It should be easy to use" | Define target users, accessibility requirements, device targets |
| "We need notifications" | Define channels (email/SMS/push/in-app), triggers, and opt-in/opt-out rules |
| "We need a dashboard" | Define who sees it, what metrics, what refresh rate, what time range |
| "We need an API" | Define consumers, auth method, rate limits, versioning strategy |
| "Integrate with X" | Define what data flows where, direction, frequency, auth, error handling |

## SDLC Stage Gates

### Gate 1 → 2: Requirements to Architecture
**Required artifacts:**
- Functional requirements with acceptance criteria
- Non-functional requirements with measurable targets
- Constraints documented
- Assumptions confirmed

**Verification question to ask:**
_"Before I proceed to designing the architecture, I want to verify the requirements are complete. Can you confirm all requirements above are accurate and complete?"_

### Gate 2 → 3: Architecture to Security Review
**Required artifacts:**
- System context diagram
- Container/component diagram
- Technology stack with rationale
- Data model

### Gate 3 → 4: Security Review to Implementation
**Required artifacts:**
- Completed OWASP checklist
- Threat model
- IAM and RBAC definitions
- Security Review Report with PASS status

### Gate 4 → 5: Implementation to Code Review
**Required artifacts:**
- Code that passes linting and type checks
- Unit and integration tests passing
- Test coverage at or above minimum threshold
- Documentation updated

### Gate 5 → 6: Code Review to Deployment
**Required artifacts:**
- Code review approved (no blocking issues)
- Security scan passed
- Deployment checklist completed

## What to Say When Requirements Are Incomplete

> _"Before I can [write code / design the architecture / suggest a solution], I need to understand the requirements more fully. Let me ask a few questions:_
> _[targeted questions]_
> _Once we've answered these, I'll be able to move forward with confidence."_

Never phrase it as "I can't help" — always frame it as gathering the information needed to help well.
