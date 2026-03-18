---
mode: agent
agent: solution-architect
description: Run this prompt to design the architecture after requirements are approved
---

# Stage 2: Define Architecture

You are acting as the **solution-architect** agent. Design the system architecture based on the approved requirements document below.

**DO NOT write any implementation code during this stage.** Your outputs are design artifacts, not source code.

## Pre-Flight Check

Before starting, confirm:
- [ ] Requirements document is included below and marked as APPROVED
- [ ] All non-functional requirements have measurable targets

If either condition is not met, stop and ask for the missing items.

## Instructions

1. Review the requirements document carefully
2. Ask clarifying questions if any requirement is architecturally ambiguous (2–3 questions at a time max)
3. Produce the following artifacts in this order:
   - **System Context Diagram** (C4 Level 1) — Mermaid `C4Context` or `graph TD`
   - **Container Diagram** (C4 Level 2) — system components and their technologies
   - **Technology Stack** — with rationale for each choice
   - **Data Model** — Mermaid `erDiagram` with key entities and relationships
   - **API Contract** — high-level resource structure and auth requirements
   - **Security Architecture** — authentication, authorization, secrets, encryption
   - **Architecture Decision Records** — one ADR for every significant technology choice
4. Explicitly state: "Architecture design is complete. Ready for Stage 3: Security Review."

## Approved Requirements Document

<!-- Paste the approved requirements document from Stage 1 here -->

[PASTE APPROVED REQUIREMENTS DOCUMENT HERE]

## Constraints

<!-- List any technology, team, or budget constraints not captured in requirements -->

[PASTE CONSTRAINTS HERE]
