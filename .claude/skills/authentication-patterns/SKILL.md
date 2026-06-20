---
name: authentication-patterns
description: Auth done right — JWT access/refresh tokens, OAuth2 with PKCE, role-based access control, sessions, and the pitfalls to dodge
---

# Authentication and authorization

## JWT access and refresh tokens

```typescript
import jwt from "jsonwebtoken";

interface TokenPayload {
  sub: string;
  email: string;
  roles: string[];
}

function generateTokens(user: User) {
  const accessToken = jwt.sign(
    { sub: user.id, email: user.email, roles: user.roles },
    process.env.JWT_SECRET!,
    { expiresIn: "15m", issuer: "auth-service" }
  );

  const refreshToken = jwt.sign(
    { sub: user.id, tokenVersion: user.tokenVersion },
    process.env.REFRESH_SECRET!,
    { expiresIn: "7d", issuer: "auth-service" }
  );

  return { accessToken, refreshToken };
}

function verifyAccessToken(token: string): TokenPayload {
  return jwt.verify(token, process.env.JWT_SECRET!, {
    issuer: "auth-service",
  }) as TokenPayload;
}
```

Keep access tokens short (around 15 minutes) and let refresh tokens live longer (a week or so). Park refresh tokens in HTTP-only cookies, never in JavaScript-reachable storage.

## Middleware: authenticate, then authorize

```typescript
function authenticate(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith("Bearer ")) {
    return res.status(401).json({ error: "Missing authorization header" });
  }

  try {
    const payload = verifyAccessToken(header.slice(7));
    req.user = payload;
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      return res.status(401).json({ error: "Token expired" });
    }
    return res.status(401).json({ error: "Invalid token" });
  }
}

function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!req.user) return res.status(401).json({ error: "Not authenticated" });
    if (!roles.some(role => req.user.roles.includes(role))) {
      return res.status(403).json({ error: "Insufficient permissions" });
    }
    next();
  };
}

app.get("/admin/users", authenticate, authorize("admin"), listUsers);
```

## OAuth2 authorization-code flow with PKCE

```typescript
import crypto from "crypto";

function generatePKCE() {
  const verifier = crypto.randomBytes(32).toString("base64url");
  const challenge = crypto
    .createHash("sha256")
    .update(verifier)
    .digest("base64url");
  return { verifier, challenge };
}

app.get("/auth/login", (req, res) => {
  const { verifier, challenge } = generatePKCE();
  req.session.codeVerifier = verifier;

  const params = new URLSearchParams({
    response_type: "code",
    client_id: process.env.OAUTH_CLIENT_ID!,
    redirect_uri: `${process.env.APP_URL}/auth/callback`,
    scope: "openid profile email",
    code_challenge: challenge,
    code_challenge_method: "S256",
    state: crypto.randomBytes(16).toString("hex"),
  });

  res.redirect(`${process.env.OAUTH_AUTHORIZE_URL}?${params}`);
});

app.get("/auth/callback", async (req, res) => {
  const { code } = req.query;

  const tokenResponse = await fetch(process.env.OAUTH_TOKEN_URL!, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded" },
    body: new URLSearchParams({
      grant_type: "authorization_code",
      code: code as string,
      redirect_uri: `${process.env.APP_URL}/auth/callback`,
      client_id: process.env.OAUTH_CLIENT_ID!,
      code_verifier: req.session.codeVerifier,
    }),
  });

  const tokens = await tokenResponse.json();
  const userInfo = jwt.decode(tokens.id_token);

  req.session.user = { id: userInfo.sub, email: userInfo.email };
  res.redirect("/dashboard");
});
```

## A role-based access model

```typescript
interface Permission {
  resource: string;
  action: "create" | "read" | "update" | "delete";
}

const ROLE_PERMISSIONS: Record<string, Permission[]> = {
  viewer: [
    { resource: "posts", action: "read" },
    { resource: "comments", action: "read" },
  ],
  editor: [
    { resource: "posts", action: "create" },
    { resource: "posts", action: "read" },
    { resource: "posts", action: "update" },
    { resource: "comments", action: "create" },
    { resource: "comments", action: "read" },
  ],
  admin: [
    { resource: "*", action: "create" },
    { resource: "*", action: "read" },
    { resource: "*", action: "update" },
    { resource: "*", action: "delete" },
  ],
};

function hasPermission(roles: string[], resource: string, action: string): boolean {
  return roles.some(role =>
    ROLE_PERMISSIONS[role]?.some(
      p => (p.resource === resource || p.resource === "*") && p.action === action
    )
  );
}
```

## What to avoid

- Keeping JWTs in `localStorage`, where any XSS payload can read them — use HTTP-only cookies
- Sharing one symmetric secret across services; sign with RS256 key pairs instead
- Skipping `iss`, `aud`, and `exp` checks when you verify a token
- Rolling your own password hashing rather than leaning on bcrypt or argon2
- Cookie-based auth with no CSRF defense
- Distinct messages for "no such user" and "wrong password" — that hands attackers a user-enumeration oracle

## Before you ship

- [ ] Access tokens expire fast (15 minutes or less)
- [ ] Refresh tokens live in HTTP-only, Secure, SameSite cookies
- [ ] Passwords go through bcrypt or argon2 — never MD5 or raw SHA
- [ ] Public clients use the OAuth2 PKCE flow
- [ ] Permission checks happen at both the route and the data layer
- [ ] Tokens can be revoked via a version counter or a blocklist
- [ ] CSRF protection is on wherever cookies carry the session
- [ ] Auth failures never reveal whether the account exists
