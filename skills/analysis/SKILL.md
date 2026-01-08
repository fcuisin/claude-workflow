# Mandatory Analysis Workflow

## Core Principle

**BEFORE any code modification, a thorough analysis phase is MANDATORY.**

This rule is CRITICAL and NON-NEGOTIABLE. It prevents:

- Regressions
- Unexpected side effects
- Production bugs

---

## 4-Step Process

### Step 1: Understand the Request

**Questions to ask:**

1. What is the precise objective?
2. What are the acceptance criteria?
3. Are there any constraints (performance, security, compliance)?
4. What is the user impact?

**Actions:**

- Rephrase the request for validation
- Identify affected use cases
- Verify alignment with business objectives

### Step 2: Analyze Existing Code

**Files to read MANDATORILY:**

1. Files directly affected by the modification
2. Dependent files (that use the modified code)
3. Existing tests (to understand expected behavior)
4. Schema migrations (if database impact)

**Key considerations:**

- Will any tests break?
- Are there other modules depending on this code?
- Does the code respect the project architecture?
- Are there sensitive data involved?

### Step 3: Document the Analysis

**Mandatory content:**

1. **Objective**: Clear description of the modification
2. **Impacted files**: Exhaustive list with justification
3. **Impacts**:
   - Breaking changes: yes/no
   - DB migration required: yes/no
   - Performance impact: yes/no
   - Sensitive data: yes/no
4. **Risks**: List + mitigations
5. **Approach**: Implementation strategy
6. **TDD Tests**: List of tests to write BEFORE implementation

**Example:**

```markdown
## Analysis: Adding User Activity Audit Logging

### Objective

Track and persist all user authentication events (login, logout, password changes)
for security compliance and fraud detection.

### Impacted Files

- AuthenticationService (add audit event dispatch)
- AuditLogger (new service)
- AuditLogRepository (new repository)
- User entity (optional: add lastLoginAt field)
- Authentication middleware (inject audit logger)
- Tests for AuditLogger and integration tests

### Impacts

- Breaking change: NO
- DB migration: YES (new audit_logs table)
- Performance: Medium (async writes recommended for high-traffic)
- Sensitive data: IP addresses, user agents (requires GDPR compliance)

### Risks

1. High write volume in audit table → Mitigation: async queue + table partitioning
2. PII storage concerns → Mitigation: data retention policy (90 days) + anonymization
3. Audit failure blocks login → Mitigation: fire-and-forget pattern with error logging

### Approach

1. TDD: Write tests for AuditLogger service
2. Create migration for audit_logs table
3. Implement AuditLogger service
4. Dispatch audit events from AuthenticationService
5. Add integration tests
6. Implement data retention job

### TDD Tests

1. test_should_log_successful_login_with_ip_and_user_agent()
2. test_should_log_failed_login_attempt()
3. test_should_log_password_change_event()
4. test_should_not_block_authentication_if_audit_fails()
5. test_should_respect_data_retention_policy()
```

### Step 4: Validation

**Validation questions:**

- Does the approach respect the project architecture?
- Are the TDD tests sufficient?
- Is there a simpler alternative (KISS)?

---

## Anti-Patterns to Avoid

### Coding without reading existing code

```typescript
// BAD: Modification without understanding impact
function updateUserStatus(user: User) {
  user.status = "active"; // Impact on other modules?
}
```

### Ignoring dependencies

```typescript
// BAD: Modification without checking who uses this method
function calculateDiscount() {
  return this.price * 0.5; // Who calls calculateDiscount()?
}
```

### Forgetting tests

```typescript
// BAD: No verification of existing tests
// If I modify UserRepository, which tests will break?
```

### Ignoring security

```typescript
// BAD: Adding sensitive field without protection
class User {
  creditCardNumber: string; // Sensitive data!
  cvv: string; // PCI-DSS violation!
}
```

---

## Quick Checklist

Before any modification:

- [ ] I have read and understood the request
- [ ] I have read the affected files
- [ ] I have identified dependencies
- [ ] I have documented the analysis
- [ ] I have defined TDD tests
- [ ] I have verified architecture + SOLID compliance
- [ ] I have verified security if sensitive data involved
