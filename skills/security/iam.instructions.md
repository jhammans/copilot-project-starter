---
applyTo: "**"
---

# Identity and Access Management (IAM) Standards

Apply these standards when designing or implementing any identity, authentication, or cloud IAM configuration.

---

## Principles

### 1. Least Privilege
Every identity (human, service, application) should have **only the permissions required to perform its function** — nothing more, nothing less.

- Start with zero permissions and grant only what is explicitly needed
- Avoid wildcard (`*`) actions or resources in production
- Review and revoke unused permissions on a defined schedule

### 2. Separation of Duties
No single identity should be able to both initiate and approve a critical action:
- Code author ≠ code merger (require PR review)
- Developer ≠ production deployer (require pipeline approval)
- Application service account ≠ deployment automation account

### 3. Defense in Depth
Do not rely on a single authentication control. Layer:
- Network controls (VPC, firewall rules)
- Application authentication (JWT, OAuth2)
- Infrastructure IAM (cloud provider IAM roles)
- Audit logging of all access

---

## Human Identity Standards

### Authentication
- **SSO required** for all human users in production environments — no local username/password-only accounts
- **MFA mandatory** for:
  - All users accessing production environments
  - All admin/privileged accounts
  - All users with access to sensitive data (PII, financial, credentials)
- **Password policy** (when local accounts exist):
  - Minimum 12 characters
  - No complexity requirements that cause predictable patterns (NIST SP 800-63 guidance)
  - Check against known-breached password lists (HaveIBeenPwned API)
  - Prohibit reuse of last 10 passwords
- **Session management:**
  - Inactivity timeout: 30 minutes for standard users, 15 minutes for admins
  - Absolute session timeout: 8 hours
  - Invalidate all sessions on password change
  - Single active session per user (optional — depends on requirement)

### Privileged Access
- Privileged access (admin, production) should be time-limited (just-in-time access)
- All privileged access must be requested with justification and approved
- All privileged sessions must be logged and auditable
- Break-glass accounts: documented, secured, and access to them immediately triggers an alert

### Account Lifecycle
- Provision new accounts only through an approved provisioning process
- Disable accounts immediately on employee offboarding — do not wait
- Remove all access tokens, API keys, and session tokens for disabled accounts
- Review all user accounts and permissions quarterly

---

## Service Identity Standards

### Prefer Managed Identities Over Static Credentials

| Platform | Preferred Method |
|----------|-----------------|
| AWS | IAM Roles for ECS/EKS (IRSA), EC2 Instance Profiles |
| GCP | Workload Identity Federation, Service Account with Workload Identity |
| Azure | Managed Identity (system-assigned or user-assigned) |
| Kubernetes | ServiceAccount bound to cloud provider identity |

```yaml
# ✅ Good — AWS IRSA (IAM Roles for Service Accounts)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-service-role

# ❌ Bad — static credentials in a Kubernetes Secret
apiVersion: v1
kind: Secret
data:
  AWS_ACCESS_KEY_ID: <base64>       # Avoid static creds
  AWS_SECRET_ACCESS_KEY: <base64>   # Rotate regularly if you must use them
```

### Static Credential Requirements (When Unavoidable)
- Store in a secrets manager (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault)
- Rotate every 90 days maximum
- Never store in environment files (`.env`) committed to source control
- Each service has its own credentials — never share credentials between services
- Monitor for credential usage anomalies

### IAM Policy Design

```json
// ✅ Good — scoped to specific actions and resources
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/uploads/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}

// ❌ Bad — wildcard permissions
{
  "Statement": [{
    "Effect": "Allow",
    "Action": "*",         // Never use * in production
    "Resource": "*"        // Never use * in production
  }]
}
```

---

## OAuth2 / OIDC Implementation Standards

### Token Types and Lifetimes

| Token Type | Recommended Lifetime | Storage |
|-----------|---------------------|---------|
| Access Token | 15 minutes | Memory only (never localStorage) |
| Refresh Token | 7–30 days (sliding) | HttpOnly cookie or secure storage |
| ID Token | Same as access token | Memory; used for identity claim verification only |

### Scopes

- Request only the scopes you need
- Never request `openid profile email` if you only need `openid email`
- Document every scope your application requests and why

### PKCE (Required for all public clients)

```typescript
// ✅ Always use PKCE for browser-based and mobile OAuth flows
const codeVerifier = generateRandomString(128);
const codeChallenge = base64UrlEncode(sha256(codeVerifier));

// Include in authorization request:
const params = new URLSearchParams({
  response_type: 'code',
  client_id: CLIENT_ID,
  redirect_uri: REDIRECT_URI,
  scope: 'openid email',
  code_challenge: codeChallenge,
  code_challenge_method: 'S256',
  state: generateRandomString(32), // CSRF protection
});
```

### Token Storage (Web Applications)

```typescript
// ✅ Good — access token in memory, refresh token in HttpOnly cookie
// In-memory token store for access tokens
let accessToken: string | null = null;

function setAccessToken(token: string) {
  accessToken = token; // in memory, not localStorage
}

// ❌ Bad — tokens in localStorage are accessible to any JS code (XSS risk)
localStorage.setItem('access_token', token); // NEVER for security tokens
sessionStorage.setItem('access_token', token); // Also avoid
```

---

## Secrets Management

### Hierarchy of Preference

1. **Managed identity / workload identity** (no credentials at all) ← Best
2. **Secrets manager with dynamic secrets** (auto-rotated) ← Preferred
3. **Secrets manager with static secrets** (manually rotated) ← Acceptable
4. **Environment variables from CI/CD vault integration** ← Acceptable for CI
5. **Encrypted secrets in Kubernetes** ← Only with external secret operator
6. **`.env` file** ← Development only, never committed to source control

### Secret Rotation

Every static secret must have:
- An owner responsible for rotation
- A rotation schedule (maximum 90 days)
- An automated rotation mechanism where possible
- Application must handle rotation without downtime (graceful credential reload)

### Audit All Secret Access

```hcl
# Example: AWS Secrets Manager policy with logging
resource "aws_secretsmanager_secret_policy" "example" {
  secret_arn = aws_secretsmanager_secret.example.arn
  policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = aws_iam_role.my_service.arn }
      Action    = ["secretsmanager:GetSecretValue"]
      Resource  = "*"
      Condition = {
        StringEquals = { "aws:RequestedRegion" = "us-east-1" }
      }
    }]
  })
}
# Enable CloudTrail to log all GetSecretValue calls
```

---

## IAM Audit Checklist

Run this review quarterly or before any major release:

- [ ] No wildcard (`*`) permissions in production IAM policies
- [ ] No shared service accounts between services
- [ ] All human accounts use SSO (no local-only accounts)
- [ ] MFA enforced for all admin and production-access users
- [ ] All static credentials stored in a secrets manager
- [ ] No active credentials for disabled/departed employees
- [ ] Unused permissions removed from all roles (last reviewed: DATE)
- [ ] All privileged access is time-limited and audited
- [ ] Break-glass account usage is alerted and reviewed
- [ ] No secrets in source code, Dockerfiles, or CI/CD config
- [ ] Secret rotation schedule is documented and enforced
