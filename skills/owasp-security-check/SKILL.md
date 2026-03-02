---
name: owasp-security-check
description: Security audit guidelines for web applications and REST APIs based on OWASP Top 10 and web security best practices. Use when checking code for vulnerabilities, reviewing auth/authz, auditing APIs, or before production deployment.
---

# OWASP Security Check

Comprehensive security audit patterns for web applications and REST APIs. Contains 20 rules across 5 categories covering OWASP Top 10 and common web vulnerabilities.

## When to Apply

Use this skill when:

- Auditing a codebase for security vulnerabilities
- Reviewing user-provided file or folder for security issues
- Checking authentication/authorization implementations
- Evaluating REST API security
- Assessing data protection measures
- Reviewing configuration and deployment settings
- Before production deployment
- After adding new features that handle sensitive data

## How to Use This Skill

1. **Identify application type** - Web app, REST API, SPA, SSR, or mixed
2. **Scan by priority** - Start with CRITICAL rules, then HIGH, then MEDIUM
3. **Review relevant rule files** - Load specific rules from @rules/ directory
4. **Report findings** - Note severity, file location, and impact
5. **Provide remediation** - Give concrete code examples for fixes

## Audit Workflow

### Step 1: Systematic Review by Priority

Work through categories by priority:

1. **CRITICAL**: Authentication & Authorization, Data Protection, Input/Output Security
2. **HIGH**: Configuration & Headers
3. **MEDIUM**: API & Monitoring

### Step 2: Generate Report

Format findings as:

- **Severity**: CRITICAL | HIGH | MEDIUM | LOW
- **Category**: Rule name
- **File**: Path and line number
- **Issue**: What's wrong
- **Impact**: Security consequence
- **Fix**: Code example of remediation

## Rules Summary

### Authentication & Authorization (CRITICAL)

#### broken-access-control - @rules/broken-access-control.md

Check for missing authorization, IDOR, privilege escalation.

```typescript
// Bad: No authorization check
async function getUser(req: Request): Promise<Response> {
  let url = new URL(req.url);
  let userId = url.searchParams.get("id");
  let user = await db.user.findUnique({ where: { id: userId } });
  return new Response(JSON.stringify(user));
}

// Good: Verify ownership
async function getUser(req: Request): Promise<Response> {
  let session = await getSession(req);
  let url = new URL(req.url);
  let userId = url.searchParams.get("id");

  if (session.userId !== userId && !session.isAdmin) {
    return new Response("Forbidden", { status: 403 });
  }

  let user = await db.user.findUnique({ where: { id: userId } });
  return new Response(JSON.stringify(user));
}
```

#### authentication-failures - @rules/authentication-failures.md

Check for weak authentication, missing MFA, session issues.

```typescript
// Bad: Weak password check
if (password.length >= 6) {
  /* allow */
}

// Good: Strong password requirements
function validatePassword(password: string) {
  if (password.length < 12) return false;
  if (!/[A-Z]/.test(password)) return false;
  if (!/[a-z]/.test(password)) return false;
  if (!/[0-9]/.test(password)) return false;
  if (!/[^A-Za-z0-9]/.test(password)) return false;
  return true;
}
```

### Data Protection (CRITICAL)

#### cryptographic-failures - @rules/cryptographic-failures.md

Check for weak encryption, plaintext storage, bad hashing.

```typescript
// Bad: MD5 for passwords
let hash = crypto.createHash("md5").update(password).digest("hex");

// Good: bcrypt with salt
let hash = await bcrypt(password, 12);
```

#### sensitive-data-exposure - @rules/sensitive-data-exposure.md

Check for PII in logs/responses, error messages leaking info.

```typescript
// Bad: Exposing sensitive data
return new Response(JSON.stringify(user)); // Contains password hash, email, etc.

// Good: Return only needed fields
return new Response(
  JSON.stringify({
    id: user.id,
    username: user.username,
    displayName: user.displayName,
  }),
);
```

#### data-integrity-failures - @rules/data-integrity-failures.md

Check for unsigned data, insecure deserialization.

```typescript
// Bad: Trusting unsigned JWT
let decoded = JSON.parse(atob(token.split(".")[1]));
if (decoded.isAdmin) {
  /* grant access */
}

// Good: Verify signature
let payload = await verifyJWT(token, secret);
```

#### secrets-management - @rules/secrets-management.md

Check for hardcoded secrets, exposed env vars.

```typescript
// Bad: Hardcoded secret
const API_KEY = "<HARDCODED_SECRET_DO_NOT_USE>";

// Good: Environment variables
let API_KEY = process.env.API_KEY;
if (!API_KEY) throw new Error("API_KEY not configured");
```

### Input/Output Security (CRITICAL)

#### injection-attacks - @rules/injection-attacks.md

Check for SQL, XSS, NoSQL, Command, Path Traversal injection.

```typescript
// Bad: SQL injection
let query = `SELECT * FROM users WHERE email = '${email}'`;

// Good: Parameterized query
let user = await db.user.findUnique({ where: { email } });
```

#### ssrf-attacks - @rules/ssrf-attacks.md

Check for unvalidated URLs, internal network access.

```typescript
// Bad: Fetching user-provided URL
let url = await req.json().then((d) => d.url);
let response = await fetch(url);

// Good: Validate against allowlist
const ALLOWED_DOMAINS = ["api.example.com", "cdn.example.com"];
let url = new URL(await req.json().then((d) => d.url));
if (!ALLOWED_DOMAINS.includes(url.hostname)) {
  return new Response("Invalid URL", { status: 400 });
}
```

#### file-upload-security - @rules/file-upload-security.md

Check for unrestricted uploads, MIME validation.

```typescript
// Bad: No file type validation
let file = await req.formData().then((fd) => fd.get("file"));
await writeFile(`./uploads/${file.name}`, file);

// Good: Validate type and extension
const ALLOWED_TYPES = ["image/jpeg", "image/png", "image/webp"];
const ALLOWED_EXTS = [".jpg", ".jpeg", ".png", ".webp"];
let file = await req.formData().then((fd) => fd.get("file") as File);

if (!ALLOWED_TYPES.includes(file.type)) {
  return new Response("Invalid file type", { status: 400 });
}
```

#### redirect-validation - @rules/redirect-validation.md

Check for open redirects, unvalidated redirect URLs.

```typescript
// Bad: Unvalidated redirect
let returnUrl = new URL(req.url).searchParams.get("return");
return Response.redirect(returnUrl);

// Good: Validate redirect URL
let returnUrl = new URL(req.url).searchParams.get("return");
let allowed = ["/dashboard", "/profile", "/settings"];
if (!allowed.includes(returnUrl)) {
  return Response.redirect("/");
}
```

### Configuration & Headers (HIGH)

#### insecure-design - @rules/insecure-design.md

Check for security anti-patterns in architecture.

```typescript
// Bad: Security by obscurity
let isAdmin = req.headers.get("x-admin-secret") === "admin123";

// Good: Proper role-based access control
let session = await getSession(req);
let isAdmin = await db.user
  .findUnique({
    where: { id: session.userId },
  })
  .then((u) => u.role === "ADMIN");
```

#### security-misconfiguration - @rules/security-misconfiguration.md

Check for default configs, debug mode, error handling.

```typescript
// Bad: Exposing stack traces
catch (error) {
  return new Response(error.stack, { status: 500 });
}

// Good: Generic error message
catch (error) {
  console.error(error); // Log server-side only
  return new Response("Internal server error", { status: 500 });
}
```

#### security-headers - @rules/security-headers.md

Check for CSP, HSTS, X-Frame-Options, etc.

```typescript
// Bad: No security headers
return new Response(html);

// Good: Security headers set
return new Response(html, {
  headers: {
    "Content-Security-Policy": "default-src 'self'",
    "X-Frame-Options": "DENY",
    "X-Content-Type-Options": "nosniff",
    "Strict-Transport-Security": "max-age=31536000; includeSubDomains",
  },
});
```

#### cors-configuration - @rules/cors-configuration.md

Check for overly permissive CORS.

```typescript
// Bad: Wildcard with credentials
headers.set("Access-Control-Allow-Origin", "*");
headers.set("Access-Control-Allow-Credentials", "true");

// Good: Specific origin
let allowedOrigins = ["https://app.example.com"];
let origin = req.headers.get("origin");
if (origin && allowedOrigins.includes(origin)) {
  headers.set("Access-Control-Allow-Origin", origin);
}
```

#### csrf-protection - @rules/csrf-protection.md

Check for CSRF tokens, SameSite cookies.

```typescript
// Bad: No CSRF protection
let cookies = parseCookies(req.headers.get("cookie"));
let session = await getSession(cookies.sessionId);

// Good: SameSite cookie + token validation
return new Response("OK", {
  headers: {
    "Set-Cookie": "session=abc; SameSite=Strict; Secure; HttpOnly",
  },
});
```

#### session-security - @rules/session-security.md

Check for cookie flags, JWT issues, token storage.

```typescript
// Bad: Insecure cookie
return new Response("OK", {
  headers: { "Set-Cookie": "session=abc123" },
});

// Good: Secure cookie with all flags
return new Response("OK", {
  headers: {
    "Set-Cookie":
      "session=abc123; Secure; HttpOnly; SameSite=Strict; Path=/; Max-Age=3600",
  },
});
```

### API & Monitoring (MEDIUM-HIGH)

#### api-security - @rules/api-security.md

Check for REST API vulnerabilities, mass assignment.

```typescript
// Bad: Mass assignment vulnerability
let userData = await req.json();
await db.user.update({ where: { id }, data: userData });

// Good: Explicitly allow fields
let { displayName, bio } = await req.json();
await db.user.update({
  where: { id },
  data: { displayName, bio }, // Only allowed fields
});
```

#### rate-limiting - @rules/rate-limiting.md

Check for missing rate limits, brute force prevention.

```typescript
// Bad: No rate limiting
async function login(req: Request): Promise<Response> {
  let { email, password } = await req.json();
  // Allows unlimited login attempts
}

// Good: Rate limiting
let ip = req.headers.get("x-forwarded-for");
let { success } = await ratelimit.limit(ip);
if (!success) {
  return new Response("Too many requests", { status: 429 });
}
```

#### logging-monitoring - @rules/logging-monitoring.md

Check for insufficient logging, sensitive data in logs.

```typescript
// Bad: Logging sensitive data
console.log("User login:", { email, password, ssn });

// Good: Log events without sensitive data
console.log("User login attempt", {
  email,
  ip: req.headers.get("x-forwarded-for"),
  timestamp: new Date().toISOString(),
});
```

#### vulnerable-dependencies - @rules/vulnerable-dependencies.md

Check for outdated packages, known CVEs.

```bash
# Bad: No dependency checking
npm install

# Good: Regular audits
npm audit
npm audit fix
```

## Common Vulnerability Patterns

Quick reference of patterns to look for:

- **User input without validation**: `req.json()` → immediate use
- **Missing auth checks**: Routes without authorization middleware
- **Hardcoded secrets**: Strings containing "password", "secret", "key"
- **SQL injection**: String concatenation in queries
- **XSS**: `dangerouslySetInnerHTML`, `.innerHTML`
- **Weak crypto**: `md5`, `sha1` for passwords
- **Missing headers**: No CSP, HSTS, or security headers
- **CORS wildcards**: `Access-Control-Allow-Origin: *` with credentials
- **Insecure cookies**: Missing Secure, HttpOnly, SameSite flags
- **Path traversal**: User input in file paths without validation

## Severity Quick Reference

**Fix Immediately (CRITICAL):**

- SQL/XSS/Command Injection
- Missing authentication on sensitive endpoints
- Hardcoded secrets in code
- Plaintext password storage
- IDOR vulnerabilities

**Fix Soon (HIGH):**

- Missing CSRF protection
- Weak password requirements
- Missing security headers
- Overly permissive CORS
- Insecure session management

**Fix When Possible (MEDIUM):**

- Missing rate limiting
- Incomplete logging
- Outdated dependencies (no known exploits)
- Missing input validation on non-critical fields

**Improve (LOW):**

- Missing optional security headers
- Verbose error messages (non-production)
- Suboptimal crypto parameters

## Complete Sub-Rules and Purposes

This catalog lists every sub-rule from `@rules/*.md` with its purpose.

### Authentication & Authorization (CRITICAL)
#### broken-access-control - `@rules/broken-access-control.md`
- Purpose: Check for missing authorization checks, insecure direct object references (IDOR), privilege escalation, and path traversal.
- Sub-rules:
  - **Never trust user input for authorization** - Verify against server-side session
  - **Check ownership on every resource access** - Don't assume URL ID is valid
  - **Implement deny-by-default** - Require explicit permission grants
  - **Use role-based access control** - Define clear roles and check them
  - **Validate file paths** - Never construct paths directly from user input
  - **Log authorization failures** - Track denied access for monitoring
  - **Test with different roles** - Verify unprivileged users can't access privileged resources

#### authentication-failures - `@rules/authentication-failures.md`
- Purpose: Check for weak authentication mechanisms, missing MFA, session management issues, and credential handling vulnerabilities.
- Sub-rules:
  - **Require strong passwords** - Minimum 12 characters with complexity
  - **Hash passwords properly** - Use bcrypt, argon2, or scrypt (never MD5/SHA1)
  - **Implement rate limiting** - Limit authentication attempts per IP/account
  - **Use secure session tokens** - Cryptographically random tokens
  - **Set session expiration** - Both absolute and idle timeout
  - **Regenerate session on login** - Prevent session fixation attacks
  - **Implement account lockout** - Temporarily lock after multiple failures
  - **Support MFA** - Especially for privileged accounts
  - **Never log credentials** - Don't log passwords, tokens, or reset links

### Data Protection (CRITICAL)
#### cryptographic-failures - `@rules/cryptographic-failures.md`
- Purpose: Check for weak encryption, improper key management, plaintext storage of sensitive data, and missing encryption in transit.
- Sub-rules:
  - **Use strong password hashing** - bcrypt, argon2, or scrypt (never MD5/SHA1)
  - **Use modern encryption** - AES-256-GCM or ChaCha20-Poly1305
  - **Never hardcode keys** - Use environment variables or key management systems
  - **Encrypt sensitive data at rest** - PII, credentials, financial data
  - **Enforce HTTPS/TLS** - All data in transit must be encrypted
  - **Use sufficient key lengths** - RSA ≥ 2048 bits, symmetric ≥ 256 bits
  - **Generate random IVs** - New random IV for each encryption operation
  - **Rotate keys regularly** - Implement key rotation policies

#### sensitive-data-exposure - `@rules/sensitive-data-exposure.md`
- Purpose: Check for PII, credentials, and sensitive data exposed in API responses, error messages, logs, or client-side code.
- Sub-rules:
  - **Never return password hashes** - Even hashed, they can be cracked
  - **Use explicit field selection** - Don't return entire database records
  - **Create DTOs for responses** - Define exactly what fields are public
  - **Generic error messages** - Don't expose system details to users
  - **Log full errors server-side** - Return generic messages to clients
  - **Sanitize logs** - Redact passwords, tokens, PII before logging
  - **Different views for different users** - Own profile vs others' profiles
  - **Disable debug in production** - No verbose errors or stack traces

#### data-integrity-failures - `@rules/data-integrity-failures.md`
- Purpose: Check for unsigned data, insecure deserialization, and lack of integrity verification in code and data.
- Sub-rules:
  - **Always verify JWT signatures** - Never decode without verification
  - **Never trust client data** - Look up prices, roles, permissions server-side
  - **Use JSON.parse, never eval** - Safe deserialization only
  - **Use Subresource Integrity** - For all CDN-loaded scripts/styles
  - **Sign cookies** - Use HMAC for tamper detection
  - **Verify checksums** - For downloaded code and updates
  - **Lock dependency versions** - Use lockfiles to ensure integrity
  - **Sign code in CI/CD** - Verify builds haven't been tampered with

#### secrets-management - `@rules/secrets-management.md`
- Purpose: Check for hardcoded secrets, exposed API keys, and improper credential management.
- Sub-rules:
  - **Never hardcode secrets** - Use environment variables or secret managers
  - **Add .env to .gitignore** - Never commit secret files
  - **Rotate secrets regularly** - Implement expiration and rotation
  - **Validate env vars at startup** - Fail fast if secrets missing
  - **Don't log secrets** - Sanitize logs to remove sensitive values
  - **No secrets in client code** - Keep API keys server-side only
  - **Use secret management services** - For production (AWS Secrets Manager, Vault, etc.)
  - **Scan Git history** - Use tools to find accidentally committed secrets

### Input/Output Security (CRITICAL)
#### injection-attacks - `@rules/injection-attacks.md`
- Purpose: Check for SQL injection, XSS, NoSQL injection, Command injection, and Path Traversal through proper input validation and output encoding.
- Sub-rules:
  - **Always use parameterized queries** - Never concatenate user input into SQL
  - **Validate all input** - Use type checks and format validation
  - **Escape output by context** - HTML, JavaScript, SQL require different escaping
  - **Use allowlists over denylists** - Explicitly allow known-good values
  - **Never use eval()** - Find safe alternatives for dynamic execution
  - **Avoid shell commands** - Use libraries or built-in APIs instead
  - **Validate file paths** - Prevent directory traversal with strict validation

#### ssrf-attacks - `@rules/ssrf-attacks.md`
- Purpose: Check for unvalidated URLs that allow attackers to make requests to internal services or arbitrary external URLs.
- Sub-rules:
  - **Validate URLs against allowlist** - Never trust user URLs
  - **Block internal IP ranges** - 127.0.0.1, 10.x, 192.168.x, etc.
  - **Enforce HTTPS** - No HTTP or other protocols
  - **Disable redirects** - Or validate redirect targets
  - **Block cloud metadata** - 169.254.169.254 (AWS/GCP/Azure)

#### file-upload-security - `@rules/file-upload-security.md`
- Purpose: Check for secure file upload handling including type validation, size limits, and safe storage.
- Sub-rules:
  - **Validate MIME type** - Check file.type
  - **Validate extension** - Check file extension
  - **Enforce size limits** - Prevent huge uploads
  - **Generate random filenames** - Don't use user input
  - **Store outside web root** - Not in public/
  - **Validate both MIME and extension** - Double check

#### redirect-validation - `@rules/redirect-validation.md`
- Purpose: Check for unvalidated redirect and forward URLs that could be used for phishing attacks.
- Sub-rules:
  - **Validate redirect URLs** - Use allowlist
  - **Only allow relative URLs** - Starts with / not //
  - **Never trust user input** - For redirect targets
  - **Validate OAuth redirects** - Pre-registered URIs only
  - **Default to safe redirect** - Home page if invalid

### Configuration & Headers (HIGH)
#### insecure-design - `@rules/insecure-design.md`
- Purpose: Check for security anti-patterns and flaws in application architecture that can't be fixed by implementation alone.
- Sub-rules:
  - **Don't rely on security by obscurity** - Use proper authentication
  - **Use transactions for atomic operations** - Prevent race conditions
  - **Rate limit expensive operations** - Prevent resource exhaustion
  - **Verify privileges server-side** - Never trust client data
  - **Implement defense in depth** - Multiple layers of security
  - **Perform threat modeling** - Identify risks in design phase
  - **Define trust boundaries** - Know what to validate
  - **Fail securely** - Default deny, not default allow

#### security-misconfiguration - `@rules/security-misconfiguration.md`
- Purpose: Check for insecure default configurations, unnecessary features enabled, verbose error messages, and missing security patches.
- Sub-rules:
  - **Disable debug mode in production** - No verbose logging or errors
  - **Change default credentials** - Require strong passwords
  - **Disable unnecessary features** - Minimize attack surface
  - **Generic error messages** - Don't reveal system details
  - **Keep dependencies updated** - Regularly patch vulnerabilities
  - **Remove development endpoints** - No debug/admin routes in production
  - **Secure default configurations** - Fail securely by default
  - **Regular security audits** - npm audit, dependency checks

#### security-headers - `@rules/security-headers.md`
- Purpose: Check for proper HTTP security headers that protect against XSS, clickjacking, MIME sniffing, and downgrade attacks.
- Sub-rules:
  - **Always set CSP** - Strict policy without `unsafe-inline`/`unsafe-eval`
  - **Enable HSTS** - Minimum 1 year, include subdomains
  - **Set X-Frame-Options** - Use `DENY` or `SAMEORIGIN`
  - **Set X-Content-Type-Options** - Always `nosniff`
  - **Configure Referrer-Policy** - `strict-origin-when-cross-origin`
  - **Use nonces for inline scripts** - When inline scripts are needed
  - **Set Permissions-Policy** - Restrict unnecessary browser features

#### cors-configuration - `@rules/cors-configuration.md`
- Purpose: Check for overly permissive Cross-Origin Resource Sharing (CORS) policies that allow unauthorized cross-origin requests.
- Sub-rules:
  - **Never use `Access-Control-Allow-Origin: *` with credentials** - Pick one or the other
  - **Use strict origin allowlist** - Explicitly list allowed origins
  - **Validate origin before reflecting** - Don't blindly reflect request origin
  - **Separate dev and prod origins** - Don't allow localhost in production
  - **Limit allowed methods** - Only necessary HTTP methods
  - **Limit allowed headers** - Only required headers
  - **Handle preflight requests** - Respond to OPTIONS correctly

#### csrf-protection - `@rules/csrf-protection.md`
- Purpose: Check for Cross-Site Request Forgery protection on state-changing operations.
- Sub-rules:
  - **Use SameSite=Strict or Lax** - On all session cookies
  - **No state changes via GET** - Use POST/PUT/DELETE
  - **Implement CSRF tokens** - For session-based auth
  - **Double-submit cookie** - Alternative to tokens
  - **Validate Origin header** - Additional protection layer

#### session-security - `@rules/session-security.md`
- Purpose: Check for secure session management including cookie flags, token storage, and session lifecycle.
- Sub-rules:
  - **Set HttpOnly flag** - Prevent XSS token theft
  - **Set Secure flag** - HTTPS only
  - **Set SameSite=Strict** - CSRF protection
  - **Use cryptographically random IDs** - crypto.randomBytes
  - **Set expiration** - Both absolute and idle timeout
  - **Regenerate on login** - Prevent session fixation
  - **Don't store in localStorage** - Use HttpOnly cookies
  - **Validate on every request** - Check expiry and validity

### API & Monitoring (MEDIUM-HIGH)
#### api-security - `@rules/api-security.md`
- Purpose: Check for REST API vulnerabilities including mass assignment, lack of validation, and missing resource limits.
- Sub-rules:
  - **Prevent mass assignment** - Explicitly define allowed fields
  - **Always paginate lists** - Enforce maximum page size
  - **Validate input types** - Check types and constraints
  - **Version your API** - Use `/api/v1/` prefix for versioning
  - **Limit response data** - Return only necessary fields
  - **Validate Content-Type** - Ensure correct headers

#### rate-limiting - `@rules/rate-limiting.md`
- Purpose: Check for rate limiting on authentication endpoints, APIs, and resource-intensive operations to prevent abuse and denial of service.
- Sub-rules:
  - **Rate limit auth endpoints** - Prevent brute force
  - **Per-IP and per-user limits** - Multiple layers; trust proxy headers only when configured
  - **Return 429 status** - Standard rate limit response
  - **Include retry headers** - Retry-After, X-RateLimit-\*
  - **Different limits for tiers** - Free vs paid users
  - **Rate limit expensive operations** - Reports, exports, search

#### logging-monitoring - `@rules/logging-monitoring.md`
- Purpose: Check for insufficient logging of security events, missing monitoring, and lack of incident response capabilities.
- Sub-rules:
  - **Log all authentication events** - Successes and failures
  - **Log authorization failures** - When access is denied
  - **Don't log sensitive data** - Sanitize passwords, tokens, PII
  - **Use structured logging** - JSON format for parsing
  - **Include context** - User ID, IP, timestamp, request ID
  - **Monitor and alert** - Set up alerts for suspicious patterns
  - **Retain logs appropriately** - Balance storage and compliance
  - **Protect log integrity** - Prevent tampering

#### vulnerable-dependencies - `@rules/vulnerable-dependencies.md`
- Purpose: Check for outdated packages with known security vulnerabilities and supply chain risks.
- Sub-rules:
  - **Always use lockfiles** - Commit dependency lockfiles for reproducible builds
  - **Pin production versions** - Use exact versions for production dependencies
  - **Audit regularly** - Run security audits in CI/CD and before deployments
  - **Keep dependencies updated** - Use automated update tools
  - **Separate dev dependencies** - Keep development tools separate from production
  - **Remove unused packages** - Regularly clean up unused dependencies
  - **Review before adding** - Check package age, maintainers, and reputation
  - **Monitor advisories** - Subscribe to security advisories for critical dependencies

