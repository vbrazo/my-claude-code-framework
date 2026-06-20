---
name: security-hardening
description: Hardening an application — boundary validation, output encoding, auth and tokens, security headers, secrets, and dependency audits
---

# Hardening an application

## Validating input

Validate everything where it enters the system, and treat client-side checks as a convenience, never a guarantee.

```typescript
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email().max(255),
  name: z.string().min(1).max(100).regex(/^[a-zA-Z\s'-]+$/),
  age: z.number().int().min(13).max(150),
});

function createUser(req: Request) {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return { status: 400, errors: result.error.flatten().fieldErrors };
  }
  // result.data is typed and validated
}
```

Guidelines:
- Check type, length, format, and range on every field
- Prefer allowlists to denylists — accept the known-good and reject the rest
- For uploads, verify MIME type, extension, and magic bytes together
- Cap request body size at the server or proxy (a 1MB ceiling, say)

## Encoding output

```typescript
// Prevent XSS: encode output based on context
// HTML context: use framework auto-escaping (React does this by default)
// Never use dangerouslySetInnerHTML with user input

// URL context: encode parameters
const safeUrl = `/search?q=${encodeURIComponent(userInput)}`;

// JSON context: use JSON.stringify (handles escaping)
const safeJson = JSON.stringify({ query: userInput });
```

Don't hand-build HTML from user input — lean on a templating engine with auto-escaping turned on.

## Preventing SQL injection

```python
# NEVER do this
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")

# Always use parameterized queries
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
```

```typescript
// NEVER do this
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// Always use parameterized queries
db.query("SELECT * FROM users WHERE email = $1", [email]);
```

Let an ORM or query builder handle it, and parameterize every time you drop down to raw SQL.

## CSRF protection

```typescript
// Server: generate and validate CSRF tokens
import { randomBytes } from 'crypto';

function generateCsrfToken(): string {
  return randomBytes(32).toString('hex');
}

// Middleware: validate on state-changing requests
function csrfMiddleware(req, res, next) {
  if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(req.method)) {
    const token = req.headers['x-csrf-token'] || req.body._csrf;
    if (!timingSafeEqual(token, req.session.csrfToken)) {
      return res.status(403).json({ error: 'Invalid CSRF token' });
    }
  }
  next();
}
```

Token-based APIs (Bearer tokens) don't need CSRF defenses — browsers don't attach those tokens automatically, so there's nothing to forge.

## Content Security Policy

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{random}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self';
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
  base-uri 'self';
  form-action 'self';
```

Begin locked-down and loosen only where you must. Use a `nonce` for inline scripts rather than `unsafe-inline`, collect violations through `report-uri`, and dry-run with `Content-Security-Policy-Report-Only` before you enforce.

## Security headers

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

Attach these to every response — `helmet` on Node.js, or the equivalent middleware in your stack.

## Rate limiting

```typescript
// Per-user, per-endpoint rate limiting
const rateLimits = {
  'POST /auth/login':    { window: '15m', max: 5 },
  'POST /auth/register': { window: '1h',  max: 3 },
  'POST /api/*':         { window: '1m',  max: 60 },
  'GET /api/*':          { window: '1m',  max: 120 },
};
```

Use a sliding window, keep the counters in Redis, and reject over-limit calls with `429` plus a `Retry-After`. Clamp auth endpoints harder than the rest.

## JWTs, done carefully

- Keep access tokens short-lived (around 15 minutes)
- Back them with refresh tokens (7–30 days) in httpOnly cookies
- Sign with RS256 (asymmetric) across services, HS256 (symmetric) inside a monolith
- Keep nothing sensitive in the payload — it's base64, not encryption
- Check `iss`, `aud`, `exp`, and `nbf` on every request
- Support revocation through a denylist, or short expiry plus rotation

```typescript
// Verify JWT with all checks
const payload = jwt.verify(token, publicKey, {
  algorithms: ['RS256'],
  issuer: 'auth.example.com',
  audience: 'api.example.com',
  clockTolerance: 30,
});
```

## Managing secrets

- Keep secrets out of version control — `.gitignore` your `.env`
- Inject runtime secrets through environment variables
- In production, store them in a secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler)
- Rotate on a schedule — 90 days at most for API keys
- Use a distinct set of secrets per environment (dev/staging/prod)
- Catch leaks in CI with `trufflehog`, `gitleaks`, or `git-secrets`

```bash
# Check for secrets in git history
gitleaks detect --source . --verbose

# Pre-commit hook to prevent secret commits
gitleaks protect --staged
```

## Auditing dependencies

```bash
# Node.js
npm audit --production
npx better-npm-audit audit --level=high

# Python
pip-audit
safety check

# Go
govulncheck ./...
```

Audit on every PR in CI, block merges on critical or high findings, pin your versions, and let Dependabot or Renovate open update PRs weekly.

## Before you deploy

1. Every input passes schema validation
2. SQL queries are parameterized
3. Security headers are in place
4. HTTPS is enforced with HSTS
5. Secrets live outside the code
6. Dependencies are audited and clear of critical issues
7. Public endpoints are rate-limited
8. Auth tokens expire and rotate
9. Error messages reveal nothing about internals
10. Logs record security events without capturing sensitive data
