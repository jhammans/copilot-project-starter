# Tasks — SDLC Prompt Templates

This directory contains reusable `.prompt.md` files for each stage of the SDLC. These are designed to be used with GitHub Copilot Chat's prompt file feature.

## Usage

In VS Code with GitHub Copilot Chat:
1. Open a prompt file or reference it with `#file:tasks/01-gather-requirements.prompt.md`
2. Or run it directly from the Copilot Chat prompt file selector

## Task Sequence

Run these in order — each task's output is input to the next:

| # | File | Stage | Uses Agent |
|---|------|-------|-----------|
| 1 | `01-gather-requirements.prompt.md` | Requirements | requirements-analyst |
| 2 | `02-define-architecture.prompt.md` | Architecture | solution-architect |
| 3 | `03-security-review.prompt.md` | Security Review | security-engineer |
| 4 | `04-implementation.prompt.md` | Implementation | senior-developer |
| 5 | `05-code-review.prompt.md` | Code Review | code-reviewer |
| 6 | `06-deployment.prompt.md` | Deployment | devops-engineer |
