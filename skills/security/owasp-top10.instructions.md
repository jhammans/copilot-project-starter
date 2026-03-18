---
applyTo: "**"
---

# OWASP Top 10 — Controls & Code Patterns

Apply these controls when implementing any feature that handles user input, authentication, data storage, or external integrations.

---

## A01: Broken Access Control

**Risk:** Users can access resources or perform actions they are not authorized for.

### Controls

**Server-side enforcement only:**
```typescript
// ✅ Good — authorization checked server-side on every request
app.get('/api/documents/:id', authenticate, authorize('documents:read'), async (req, res) => {
  const doc = await documentService.getById(req.params.id, req.user.id);
  if (!doc) return res.status(404).json({ error: 'Not found' });
  res.json(doc);
});

// ❌ Bad — trusting client-supplied role
app.get('/api/admin/users', async (req, res) => {
  if (req.headers['x-user-role'] === 'admin') { // NEVER trust client headers for authz
    ...
  }
});
```

**Resource-level ownership check:**
```typescript
// ✅ Good — user can only access their own document
async getDocument(docId: string, requestingUserId: string): Promise<Document> {
  const doc = await db.documents.findById(docId);
  if (!doc || doc.ownerId !== requestingUserId) {
    throw new ForbiddenError('Access denied');
  }
  return doc;
}
```

---

## A02: Cryptographic Failures

**Risk:** Sensitive data exposed due to missing or weak encryption.

### Controls

**Always use HTTPS:**
- Enforce HTTPS with HSTS header: `Strict-Transport-Security: max-age=31536000; includeSubDomains`
- Redirect HTTP to HTTPS at the load balancer

**Password hashing:**
```typescript
// ✅ Good — bcrypt with appropriate work factor
import bcrypt from 'bcrypt';
const SALT_ROUNDS = 12;
const hash = await bcrypt.hash(rawPassword, SALT_ROUNDS);
const valid = await bcrypt.compare(rawPassword, storedHash);

// ❌ Bad — MD5, SHA1, or SHA256 are NOT suitable for password hashing
const hash = crypto.createHash('md5').update(password).digest('hex'); // NEVER
```

**Sensitive data at rest:**
```typescript
// ✅ Use field-level encryption for PII columns
// ✅ Enable encryption at rest on all databases and storage buckets
// ✅ Never log sensitive fields
```

**Never use:**
- MD5 or SHA1 for security purposes
- DES, 3DES, RC4
- RSA with PKCS#1 v1.5 padding
- Random number generators not designed for cryptography (Math.random())

---

## A03: Injection

**Risk:** Attacker-supplied data is interpreted as code or commands.

### SQL Injection Prevention

```typescript
// ✅ Good — parameterized query
const user = await db.query(
  'SELECT * FROM users WHERE email = $1 AND active = TRUE',
  [req.body.email]
);

// ✅ Good — ORM
const user = await User.findOne({ where: { email: req.body.email } });

// ❌ Bad — string concatenation
const user = await db.query(
  `SELECT * FROM users WHERE email = '${req.body.email}'` // SQL INJECTION
);
```

### Command Injection Prevention

```typescript
// ✅ Good — use library APIs instead of shell commands
import { readdir } from 'fs/promises';
const files = await readdir(sanitizedPath);

// ✅ If shell is unavoidable, use execFile (not exec) with explicit args
import { execFile } from 'child_process';
execFile('convert', ['-resize', '800x600', inputPath, outputPath], callback);

// ❌ Bad — user input in shell command
exec(`convert ${req.body.filename} output.jpg`); // COMMAND INJECTION
```

### XSS Prevention

```typescript
// ✅ Good — use framework's built-in escaping (React, Angular auto-escape)
function UserName({ name }: { name: string }) {
  return <span>{name}</span>; // React automatically escapes
}

// ❌ Bad — bypasses framework escaping
function UserBio({ bio }: { bio: string }) {
  return <div dangerouslySetInnerHTML={{ __html: bio }} />; // XSS if bio contains <script>
}

// ✅ If HTML must be rendered, sanitize first:
import DOMPurify from 'dompurify';
const cleanHtml = DOMPurify.sanitize(userHtml, { USE_PROFILES: { html: true } });
```

---

## A04: Insecure Design

**Risk:** Security was not considered in the design, creating structural vulnerabilities.

### Controls

- Threat model every feature before implementation
- Define trust boundaries between components
- Apply defense-in-depth: don't rely on a single control
- Rate limit all sensitive operations at design time
- Design audit logging into every sensitive operation from the start

---

## A05: Security Misconfiguration

**Risk:** Default configurations, debug features, or overly permissive settings create vulnerabilities.

### Security Headers

```typescript
// ✅ Apply security headers with helmet.js (Express)
import helmet from 'helmet';
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"], // allow only if necessary
    }
  },
  hsts: { maxAge: 31536000, includeSubDomains: true },
  noSniff: true,
  referrerPolicy: { policy: 'same-origin' },
}));
```

### Error Handling

```typescript
// ✅ Good — safe error response
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error({ err, requestId: req.id }, 'Unhandled error'); // full details in logs
  res.status(500).json({ error: 'An internal server error occurred' }); // safe message to client
});

// ❌ Bad — exposes internal details
app.use((err, req, res, next) => {
  res.status(500).json({ error: err.message, stack: err.stack }); // NEVER expose stack traces
});
```

---

## A06: Vulnerable and Outdated Components

**Risk:** Dependencies with known CVEs are exploited.

### Controls

```bash
# Run before every PR merge
npm audit --audit-level=moderate   # Node.js
safety check                        # Python
./gradlew dependencyCheckAnalyze   # Java
dotnet list package --vulnerable   # .NET
```

- Pin dependency versions (avoid `^` or `~` for production)
- Review new dependencies for security reputation before adding
- Automate dependency scanning in CI pipeline (Dependabot, Snyk, Renovate)
- Never use a dependency that has not been updated in 2+ years without justification

---

## A07: Identification and Authentication Failures

**Risk:** Weak authentication allows account takeover.

### Controls

```typescript
// ✅ Rate limiting on auth endpoints
import rateLimit from 'express-rate-limit';
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10, // max 10 attempts per window
  message: { error: 'Too many login attempts, please try again later' },
  standardHeaders: true,
  legacyHeaders: false,
});
app.post('/api/auth/login', authLimiter, loginHandler);

// ✅ Account lockout after N failed attempts
// ✅ JWT expiry — short-lived access tokens (15 min), refresh tokens (7 days)
// ✅ Invalidate all sessions on password change
// ✅ Secure cookie flags
res.cookie('session', token, {
  httpOnly: true,    // inaccessible to JavaScript
  secure: true,      // HTTPS only
  sameSite: 'strict', // CSRF protection
  maxAge: 15 * 60 * 1000,
});
```

---

## A08: Software and Data Integrity Failures

**Risk:** CI/CD pipeline or package integrity can be compromised.

### Controls

- Verify package checksums / integrity hashes
- Sign container images and verify signatures in the pipeline
- Use `npm ci` instead of `npm install` in CI (respects lockfile)
- Lock base image digests in Dockerfiles
- No auto-merge of dependency updates without passing all security scans

---

## A09: Security Logging and Monitoring Failures

**Risk:** Attacks go undetected; incident response is delayed.

### Security Events That MUST Be Logged

```typescript
// These events must always be logged with: timestamp, userId (if known),
// IP address, action, result, and relevant resource identifiers

const SECURITY_EVENTS = [
  'auth.login.success',
  'auth.login.failure',
  'auth.logout',
  'auth.password_changed',
  'auth.mfa_enabled',
  'auth.mfa_disabled',
  'authz.access_denied',
  'user.created',
  'user.deleted',
  'user.role_assigned',
  'user.role_revoked',
  'admin.action',         // any administrative operation
  'data.export',          // bulk data exports
  'secrets.accessed',     // when secrets are retrieved
];
```

**Log format:**
```json
{
  "timestamp": "2026-03-17T10:30:00.000Z",
  "level": "info",
  "event": "auth.login.failure",
  "userId": null,
  "email": "user@example.com",
  "ip": "192.168.1.1",
  "userAgent": "...",
  "reason": "invalid_password",
  "requestId": "req_abc123"
}
```

**What MUST NOT be logged:**
- Passwords (in any form)
- Full credit card numbers
- Social Security numbers
- Auth tokens or session IDs
- Full request/response bodies that may contain sensitive data

---

## A10: Server-Side Request Forgery (SSRF)

**Risk:** Attacker tricks server into making requests to internal services.

### Controls

```typescript
// ✅ Validate URLs against an allowlist before making outbound requests
const ALLOWED_DOMAINS = ['api.partner.com', 'webhooks.authorized.com'];

function validateOutboundUrl(url: string): void {
  const parsed = new URL(url);
  if (!ALLOWED_DOMAINS.includes(parsed.hostname)) {
    throw new ValidationError(`Unauthorized outbound domain: ${parsed.hostname}`);
  }
  // Block private/internal IP ranges
  if (isPrivateIP(parsed.hostname)) {
    throw new ValidationError('Requests to internal addresses are not allowed');
  }
}

// ❌ Bad — making a request to a user-supplied URL without validation
const response = await fetch(req.body.webhookUrl); // SSRF
```

Block requests to:
- `localhost`, `127.0.0.1`, `::1`
- `169.254.169.254` (cloud metadata endpoint)
- RFC-1918 private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- `0.0.0.0/8`, `100.64.0.0/10`
