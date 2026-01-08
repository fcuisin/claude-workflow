# Security

## Overview

This document presents general security principles applicable to any project.

## Input Validation

### Golden Rule

> **Never trust user input.**
> Always validate on the server side.

### Validation Types

All user inputs must be validated according to multiple criteria:

- **Format validation**: Ensure data matches expected patterns (email, UUID, phone number, etc.)
- **Length validation**: Enforce minimum and maximum length constraints
- **Type validation**: Verify data type matches expectations (string, number, boolean, etc.)
- **Range validation**: Check numeric values fall within acceptable bounds
- **Whitelist validation**: Prefer allowing known-good values over blocking known-bad ones
- **Business logic validation**: Ensure data makes sense in the application context

### Examples

```
// ❌ BAD - No validation
function getUserById(id):
  return prisma.user.findFirst({where: {id}})

// ✅ GOOD - Validation + parameterized query
function getUserById(id):
  if not isValidId(id):
    throw httpError.BadRequest("Invalid user ID")

  return prisma.user.findFirst({where: {id}})
```

### Sanitization vs Validation

```
Validation: Reject invalid data
  → "abc" as numeric ID → ERROR

Sanitization: Clean the data
  → "<script>" in a name → "script"

Prefer VALIDATION (reject) over SANITIZATION (transform)
```

---

## Authentication

### Passwords

```
Rules:
- Minimum 12 characters
- Uppercase, lowercase, numbers, special characters
- Not in compromised password lists
- Hash with bcrypt/Argon2 (NEVER MD5/SHA1)
- Unique salt per user

// ✅ GOOD
hash = bcrypt.hash(password, costFactor=12)

// ❌ BAD
hash = md5(password)
hash = sha1(password + "static_salt")
```

### Sessions

```
Rules:
- Cryptographically secure random token
- Server-side storage (not in cookies)
- Expiration: 15-30 min of inactivity
- Renewal after login
- Invalidation after logout

Session config:
  cookie:
    httpOnly: true     # Not accessible via JS
    secure: true       # HTTPS only
    sameSite: strict   # CSRF protection
```

### JWT

```
Rules:
- Algorithm: RS256 or ES256 (not HS256 with weak secret)
- Short expiration (30 min)
- Verify signature and claims
- Do not store sensitive data in payload

// ❌ BAD
jwt.sign(payload, "secret123", { algorithm: "HS256" })

// ✅ GOOD
jwt.sign(payload, privateKey, {
  algorithm: "RS256",
  expiresIn: "30m"
})
```

### Multi-Factor Authentication (MFA)

```
When to enable MFA:
- Admin access
- Sensitive operations (payment, deletion)
- Password changes
- Login from new device

Methods:
- TOTP (Google Authenticator)
- SMS (less secure)
- Hardware keys (FIDO2)
```

---

## Authorization

### Principle of Least Privilege

```
Rule: Grant ONLY the necessary permissions.

❌ BAD
user.role = "admin"  # Access to everything

✅ GOOD
user.permissions = ["read:users", "write:orders"]
```

### RBAC (Role-Based Access Control)

```
Roles:
- admin: All permissions
- manager: User management, read reports
- user: Access to own data only

Verification:
function deleteUser(userId, currentUser):
  if not currentUser.hasPermission("delete:users"):
    throw Forbidden("Permission denied")

  // ... delete logic
```

### Row-Level Security

```
Rule: Verify the user has access to THAT specific resource.

// ❌ BAD - Only checks authentication
function getOrder(orderId):
  return prisma.order.findFirst({where: {id: orderId}})

// ✅ GOOD - Checks ownership
function getOrder(orderId, currentUser):
  const order = prisma.order.findFirst({where: {id: orderId}})

  if order.userId != currentUser.id:
    throw Forbidden("Not your order")

  return order
```

---

## Sensitive Data

### Classification

- **Internal** : Restricted access (such as emails or address)
- **Secret** : Vault or hash (password)
- **Confidential** : Use encryption

### Storage

```
Passwords:
  → Hash with bcrypt/Argon2
  → NEVER in plain text

Personal data (GDPR/Privacy laws):
  → Encryption at rest
  → Pseudonymization when possible
  → Limited retention

Secrets (API keys, etc.):
  → Environment variables
  → Vault (HashiCorp, AWS Secrets Manager)
  → NEVER in source code
```

### Transmission

```
Rules:
- HTTPS mandatory (TLS 1.3)
- Valid certificates
- HSTS enabled
- No sensitive data in URLs

// ❌ BAD
GET /api/users?password=secret123

// ✅ GOOD
POST /api/auth
Body: { "password": "..." }
```

---

## Security Headers

### Recommended Headers

```http
# XSS Protection
Content-Security-Policy: default-src 'self'; script-src 'self'
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block

# Clickjacking Protection
X-Frame-Options: DENY

# HTTPS
Strict-Transport-Security: max-age=31536000; includeSubDomains

# Referrer
Referrer-Policy: strict-origin-when-cross-origin

# Permissions
Permissions-Policy: geolocation=(), camera=()
```

### Content-Security-Policy (CSP)

```http
# Restrictive (recommended)
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self';
  img-src 'self' data:;
  font-src 'self';
  connect-src 'self' api.example.com;
  frame-ancestors 'none';
```
