---
mode: agent
agent: devops-engineer
description: Run this prompt to prepare and execute a deployment
---

# Stage 6: Deployment

You are acting as the **devops-engineer** agent. Prepare and execute the deployment of the approved code changes.

## Pre-Flight Check

**Do NOT deploy until all conditions below are confirmed:**

- [ ] Code review approved (from Stage 5) with no blocking issues
- [ ] All CI/CD pipeline stages passing (lint, tests, security scan, build)
- [ ] Deployment checklist reviewed
- [ ] Target environment confirmed
- [ ] Rollback plan documented

## Instructions

1. Review the change being deployed and identify any deployment-specific concerns
2. Identify if there are database migrations — if yes, confirm they are backwards-compatible
3. Confirm feature flags configuration (if applicable)
4. Walk through the deployment checklist item by item
5. Generate the deployment runbook for this specific release
6. After deployment: verify health checks and smoke test checklist
7. Confirm: "Deployment to [environment] complete. Health check: PASS."

## Deployment Checklist

Walk through each item explicitly:

### Pre-Deployment
- [ ] CI/CD pipeline: all stages green
- [ ] Code review: approved, no blocking issues
- [ ] Security scan: passed
- [ ] Database migrations: backwards-compatible / no migrations in this release
- [ ] Environment-specific config: validated in staging
- [ ] Rollback plan: documented below
- [ ] On-call engineer: notified

### Deployment
- [ ] Deploy to staging: complete
- [ ] Smoke test staging: pass
- [ ] Approval gate: obtained
- [ ] Deploy to production: execute

### Post-Deployment
- [ ] Health check endpoint: responding with 2xx
- [ ] Key user flows: verified
- [ ] Error rate: baseline (no spike)
- [ ] Latency: baseline (no degradation)
- [ ] Alerts: no new alerts firing
- [ ] Monitoring dashboard: reviewed

## Change Description

<!-- Describe what is being deployed -->

[DESCRIBE WHAT IS BEING DEPLOYED — feature name, ticket number, what changed]

## Rollback Plan

<!-- Document the rollback procedure for this specific deployment -->

[DESCRIBE HOW TO ROLL BACK IF THIS DEPLOYMENT FAILS]

## Target Environment

- **Environment:** [dev / staging / production]
- **Cloud Provider:** [AWS / GCP / Azure / on-prem]
- **Deployment Method:** [Kubernetes / ECS / App Service / EC2 / etc.]
