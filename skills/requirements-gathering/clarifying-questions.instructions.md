---
applyTo: "**"
---

# Clarifying Questions Playbook

This instruction file provides structured question sets for every common requirements scenario. Use these to systematically eliminate ambiguity before implementation.

## Universal Opening Questions

Always start with these before domain-specific questions:

1. _"What is the primary goal of this feature/system? What problem does it solve?"_
2. _"Who are the intended users? How technical are they?"_
3. _"What does a successful outcome look like? How will you measure it?"_
4. _"Are there any hard constraints I should know about upfront? (Technology, timeline, budget, existing systems)"_

## Domain-Specific Question Sets

### Authentication & Identity
- What identity providers does this need to support? (Local, Google, Microsoft, SAML, etc.)
- Should there be support for Single Sign-On (SSO)?
- Is Multi-Factor Authentication (MFA) required? For all users or only specific roles?
- What happens when a user forgets their password? (Self-service reset, admin reset, or both)
- Are there session timeout requirements?
- Should sessions be invalidated when password changes?
- Do you need "remember me" / persistent sessions?
- What do you do with accounts that are inactive for X days?

### Authorization & Access Control
- What are the user roles in this system? What can each role do?
- Is there resource-level ownership? (User can only see their own data?)
- Are there org/tenant hierarchies? (Multi-tenant?)
- Who can assign roles? Who can revoke roles?
- Should role assignments be audited?
- Are there time-limited permissions (access grants that expire)?
- What happens to data owned by a deleted user?

### Data & Storage
- What data will be stored? Can you provide a draft of the key entities?
- Which fields are personally identifiable information (PII)?
- What is the data retention policy? When can data be deleted?
- Do you need a soft-delete or hard-delete?
- How will data be accessed — primarily by ID lookup, by search, or by bulk export?
- What are the expected data volumes today and in 12/24 months?
- Are there compliance requirements for data storage location? (Data residency)
- Does data need to be encrypted at rest?

### APIs & Integrations
- Who are the consumers of this API? (Internal services, mobile apps, third parties, partners?)
- What authentication method should the API use? (API key, OAuth2, JWT, mTLS?)
- What are the expected call volumes? (Calls/second, daily active consumers)
- Do you need rate limiting? Per consumer? What are the limits?
- How should the API handle versioning? (URL path `/v1/`, headers, query params?)
- What is the expected response time SLA for this API?
- Should the API be documented with OpenAPI/Swagger?
- Are there any third-party APIs this needs to consume? (What are their SLAs and failure modes?)

### Notifications & Messaging
- What notification channels are needed? (Email, SMS, push, in-app, Slack/Teams?)
- What events trigger notifications?
- Who receives notifications? (One user, a group, an org admin?)
- Can users opt out? Of individual notification types or all notifications?
- What happens if a notification delivery fails? (Retry? Log and move on?)
- Are there notification rate limits? (Prevent spam)
- Do notifications need to be localized/translated?

### Search & Filtering
- What fields should be searchable?
- Should search be full-text or exact match?
- What filters should users be able to apply?
- What should the default sort order be?
- Are there pagination requirements? (Max page size, cursor vs. offset?)
- Should search history be saved?
- Are there results that should be excluded based on the user's permissions?

### File Uploads / Media
- What file types are allowed?
- What is the maximum file size?
- Where should files be stored? (Cloud object storage, database, CDN?)
- Can files be public, or are they always access-controlled?
- Is virus/malware scanning required?
- What happens to files associated with a deleted account?
- Is image resizing or video transcoding needed?

### Payments & Financial
- What payment processor will be used? (Stripe, PayPal, Braintree, etc.)
- What payment methods must be supported? (Cards, ACH, PayPal, crypto?)
- Is PCI DSS compliance required?
- What currency/currencies must be supported?
- Is there a subscription/recurring billing component?
- What is the refund policy? How are refunds handled technically?
- Do you need invoice generation?
- What financial reporting is needed?

### Performance & Scalability
- How many concurrent users do you expect at launch?
- What is the growth projection for 6, 12, 24 months?
- What is the acceptable response time for the most frequently used operations? (p50, p95, p99?)
- Are there predictable traffic peaks? (Scheduled jobs, business hours, seasonal?)
- What should happen during peak load — degrade gracefully? Queue requests? Reject?
- Is there a budget constraint that limits infrastructure scale?

### Reporting & Analytics
- Who needs reports? (Admins, end users, executives, operations?)
- What data should be in the reports?
- What time periods and granularity are needed?
- Should reports be generated on-demand or scheduled?
- What export formats are needed? (CSV, PDF, Excel, API?)
- Is real-time data required, or is a daily batch acceptable?

## Question Anti-Patterns to Avoid

| Bad Question | Why It's Bad | Better Alternative |
|-------------|-------------|-------------------|
| "Do you want it to be fast?" | Leading question — no one says no | "What is the acceptable p95 response time for [operation]?" |
| "Should it be secure?" | Too vague | "What authentication and authorization model is required?" |
| "Do you need logging?" | Closed yes/no | "What events need to be audited, and who reviews those audit logs?" |
| "Is that all?" | Unhelpful | "Are there any edge cases or error scenarios we haven't covered?" |
| 10-question lists | Overwhelming | Group into 2–4 at a time, wait for answers |

## Iteration Protocol

After each round of questions and answers:

1. Summarize what you now understand
2. Identify any remaining ambiguities or follow-up questions
3. If no ambiguities remain: _"I believe I have a complete picture of the requirements. Let me summarize them for your confirmation before we proceed."_
4. Produce the requirements summary (see requirements-analyst agent for format)
5. Get explicit confirmation: _"Does this look correct and complete?"_
