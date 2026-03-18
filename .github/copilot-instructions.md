# GitHub Copilot — Global Project Instructions

These instructions apply to **every** Copilot interaction across the project. They establish baseline behavior, quality gates, and workflow expectations.

---

## Core Principles

1. **Clarify before coding.** Never write implementation code until requirements are fully understood and documented. If anything is ambiguous, ask a targeted clarifying question.
2. **Gate your work.** Follow the SDLC stages below in order. Do not advance to the next stage until the current stage is complete and approved.
3. **Security first.** Every design decision, code suggestion, and architecture choice must account for security. Reference the security skills before implementing anything that touches auth, data storage, networking, or user input.
4. **Standards always.** Apply the relevant language and framework standards from `skills/languages/` and `skills/frameworks/` whenever generating or reviewing code.
5. **Never guess intent.** If a requirement, constraint, or decision is unclear, surface it as a question rather than making an assumption.

---

## SDLC Stages & Gates

Work through these stages **in order**. Each stage has an explicit gate condition. Do not proceed until the gate is met.

### Stage 1 — Requirements Gathering
- **Goal:** Capture functional requirements, non-functional requirements, constraints, and acceptance criteria.
- **Process:** Use the `requirements-analyst` agent or the `01-gather-requirements.prompt.md` task.
- **Gate:** All ambiguities resolved. Requirements documented and approved by stakeholder.
- **Blocked action:** Writing any code, defining data models, or selecting technology stack.

### Stage 2 — Architecture & Design
- **Goal:** Define system components, data flows, API contracts, and technology stack.
- **Process:** Use the `solution-architect` agent or the `02-define-architecture.prompt.md` task.
- **Gate:** Architecture document reviewed. No unresolved trade-offs remain.
- **Blocked action:** Writing implementation code.

### Stage 3 — Security Review
- **Goal:** Threat model the design, identify attack surfaces, and define security controls.
- **Process:** Use the `security-engineer` agent or the `03-security-review.prompt.md` task.
- **Gate:** Threat model complete. OWASP Top 10 addressed. IAM and RBAC defined.
- **Blocked action:** Writing implementation code.

### Stage 4 — Implementation
- **Goal:** Write code that satisfies requirements, follows standards, and passes security controls.
- **Process:** Use the `senior-developer` agent or the `04-implementation.prompt.md` task.
- **Gate:** All acceptance criteria met. Tests written and passing. No critical lint errors.
- **Blocked action:** Merging code without review.

### Stage 5 — Code Review
- **Goal:** Validate code quality, security, standards compliance, and test coverage.
- **Process:** Use the `code-reviewer` agent or the `05-code-review.prompt.md` task.
- **Gate:** All review comments resolved. Security scan passing.
- **Blocked action:** Deploying to any environment.

### Stage 6 — Deployment
- **Goal:** Release to target environment safely with rollback capability.
- **Process:** Use the `devops-engineer` agent or the `06-deployment.prompt.md` task.
- **Gate:** Deployment checklist complete. Monitoring confirmed active.

---

## Clarifying Questions Protocol

When requirements or instructions are ambiguous:

1. **Identify** the ambiguity explicitly (e.g., _"It's unclear whether X means A or B"_).
2. **Ask one or two focused questions** — do not overwhelm with a long list.
3. **Propose a default** if you have a strong reasonable assumption (e.g., _"I'll assume X unless you tell me otherwise"_).
4. **Wait for confirmation** before proceeding.
5. **Document the decision** in the requirements or design artifact.

---

## Security Baseline

All code must:

- Validate and sanitize all user input at system boundaries
- Never store secrets, credentials, or PII in source code or logs
- Use parameterized queries / ORMs — never string-interpolated SQL
- Apply least-privilege IAM roles and service accounts
- Enforce RBAC on every API endpoint
- Return generic error messages to clients (no stack traces in production)
- Depend on actively maintained packages — flag known CVEs before accepting a dependency
- Use HTTPS/TLS for all external communication

---

## Language & Framework Standards

When generating code:

- **TypeScript/JavaScript:** See `skills/languages/typescript.instructions.md`
- **Python:** See `skills/languages/python.instructions.md`
- **Java:** See `skills/languages/java.instructions.md`
- **C#/.NET:** See `skills/languages/csharp.instructions.md`
- **Go:** See `skills/languages/go.instructions.md`
- **React:** See `skills/frameworks/react.instructions.md`
- **Next.js:** See `skills/frameworks/nextjs.instructions.md`
- **Node/Express:** See `skills/frameworks/nodejs-express.instructions.md`
- **Spring Boot:** See `skills/frameworks/spring-boot.instructions.md`
- **ASP.NET Core:** See `skills/frameworks/dotnet-aspnet.instructions.md`

---

## Documentation Standards

- Every public API must have an OpenAPI / Swagger spec.
- Every public function/method must have a doc comment (JSDoc, docstring, Javadoc, XML doc, or GoDoc).
- Every README must explain: purpose, prerequisites, quick-start, configuration, and contributing guidelines.
- Architecture Decision Records (ADRs) must be created for every non-trivial design decision.

---

## What Copilot Should Never Do

- Generate code before requirements are confirmed
- Assume missing security controls are "someone else's responsibility"
- Use deprecated APIs, insecure defaults, or known-vulnerable dependencies
- Skip error handling or input validation "for brevity"
- Hard-code credentials, tokens, secrets, or environment-specific values
- Produce placeholder comments like `// TODO: implement this` without flagging them as blockers
