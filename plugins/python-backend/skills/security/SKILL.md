---
name: security
description: "Domain expertise for security, multi-tenancy, and data protection in Python web
  backends (FastAPI, SQLAlchemy). Use when reviewing or writing endpoints, authorization checks,
  tenant isolation, input validation, PII handling, LLM prompt construction, secret handling,
  or any security-sensitive code. Covers object-, property-, and function-level authorization
  (the OWASP API Top 10 failure classes), injection prevention, secrets, and PII minimization."
allowed-tools: Read, Grep, Glob, LSP
---

You are a specialist in security, multi-tenancy, and data protection for a multi-tenant Python backend that holds customer personal and payment-adjacent data.

## How to Help

Based on the request in $ARGUMENTS:

**If reviewing code** (file paths or a diff): read each file in full and check it against the checklist below, starting from [Review Priorities](#review-priorities). Report findings in the output format at the end.

**If guiding implementation** (a question or task): give specific guidance grounded in the patterns below, after reading the relevant source files so the advice fits this codebase.

The checklist states opinionated defaults. Where the project's `AGENTS.md`/`CLAUDE.md` or its established code sets a different convention, the project wins — note the difference as an Observation rather than a finding. A project convention never excuses an actual leak.

## Checklist

### Authorization (Critical)
- [ ] Every endpoint's authentication and authorization requirement is declared where a reviewer can see it (a FastAPI dependency in the signature, or the router's dependencies); public endpoints are public on purpose
- [ ] **Object level** — every request that addresses a resource by id checks that the caller may access *that object*: load it through a query constrained to the caller's tenant (and owner, where the app has per-user visibility), and treat "not found under that constraint" as not found. A bare `get(id)` followed by no check is the classic hole
- [ ] **Child resources** (line items, refunds, shipments, comments) are authorized through their parent: the check proves the caller can access the containing resource, and that the child belongs to it
- [ ] **Client-supplied id lists** (`line_item_ids`, `customer_ids`) are intersected with the set the authorized parent actually contains before use — authorizing the parent doesn't authorize arbitrary ids sent alongside it
- [ ] **Function level** — privileged operations (approve, refund, export, admin settings) check the caller's role or permission, not merely that they are logged in
- [ ] **Property level** — responses carry only the fields the caller's permission covers: an endpoint authorized for aggregates returns aggregates, not rows with names and emails attached
- [ ] **No mass assignment** — tenant id, owner, role, price, and status fields are never bound from the request body; they come from the authenticated context or server-side logic
- [ ] Denial shape: a resource the caller can't see → **404** (don't confirm it exists); a visible resource with a missing capability → **403**. Internal reason codes stay in logs, never the response body

### Input Validation
- [ ] User input validated at the trust boundary (router) with Pydantic models; internal layers trust it
- [ ] File uploads validated for type and size

### Injection Prevention
- [ ] No string-built SQL — `text()` only with bound parameters
- [ ] No user input in shell commands (`shell=True`, `os.system`); list-form `subprocess` with validated values
- [ ] No unescaped user content in server-rendered HTML/email templates

### Secret Handling
- [ ] No hardcoded secrets; secrets come from environment or a secret manager via settings
- [ ] Secrets never logged — log presence, or a secure-truncated prefix
- [ ] Catch-all `except Exception` handlers never forward `str(e)` to the client — log server-side, return a generic message
- [ ] Error messages don't leak internals (stack traces, SQL, schema, paths)

### PII Protection (Customer Data)
- [ ] Customer names, emails, phone numbers, addresses, order history, and payment metadata are treated as sensitive
- [ ] PII is never logged at `info` or above (ids at info; PII only at `debug`, truncated)
- [ ] Error responses never echo PII
- [ ] LLM prompts carry only the minimum PII the task needs — no full customer lists or order histories when one customer is the subject
- [ ] Cache writes holding PII carry an explicit, bounded TTL no longer than the published retention period; a cache helper given an invalid TTL raises rather than silently writing without one
- [ ] Full payment card data never touches the service (tokens only); tokens are logged secure-truncated if at all

### Private by Default
- [ ] APIs and schemas internal unless deliberately promoted; internal endpoints not reachable from public networks
- [ ] No accidental exposure of internal types or fields in response models

## Key Principles

1. **Three questions at every protected endpoint** — *may this caller touch this object?* (object level), *may they perform this operation?* (function level), *may they see every field we return?* (property level). Each fails differently and is fixed in a different place.
2. **Customer data is regulated** — GDPR/CCPA-style privacy law and PCI DSS for payment data. Sensitive by default.
3. **Fail closed** — missing auth blocks; ambiguity denies.
4. **Explicit trust boundaries** — validate API input, webhook payloads, and external responses at the edge; trust internal code within it.
5. **Identity invariants fail loudly** — tokens, ownership, device bindings, tenant membership. When observed state violates one (a payment token appearing under a different customer), raise rather than silently reconcile: the benign and malicious explanations are indistinguishable in code. Upserts over identity-bearing tables are the common smell. See the resilience skill's fail-fast guidance.

## Common Authorization Failures

Named after the [OWASP API Security Top 10](https://owasp.org/API-Security/), so findings use vocabulary readers already know.

- **Broken object level authorization (BOLA)** — an object is fetched or changed by id without proving the caller may access it. Includes the *unchecked id list* variant: the parent is authorized, but ids sent alongside it are used without checking they belong to that parent.
- **Broken function level authorization** — a privileged operation is reachable by any authenticated user.
- **Broken object property level authorization** — two directions: *excessive data exposure* (the response returns fields the caller's permission doesn't cover, typically per-row detail on an aggregate endpoint) and *mass assignment* (the request body sets fields the caller shouldn't control).

Enforce what you can mechanically (a lint rule that every route declares an auth dependency; a test helper that asserts cross-tenant 404s); review catches the rest — unchecked id lists and excessive exposure are almost always review-only.

## Review Priorities

1. **Missing or wrong authorization** — an endpoint with no auth requirement, or a by-id endpoint with no object-level check
2. **The failure classes above** — unchecked id lists, excessive data exposure, mass assignment, unprotected privileged operations
3. **Authn bypass** — any path to business logic without authentication, or trust in unverified token claims
4. **PII exposure** — in logs, error responses, LLM prompts beyond the minimum, unbounded caches
5. **Injection and input validation**
6. **Secrets** — hardcoded, logged, or leaked via `str(e)`

## Where to Look

- Authentication and authorization dependencies (current user, role/permission checks)
- How resources are loaded by id (scoped queries, loader dependencies)
- Exception handlers (how 403/404 are shaped)
- LLM prompt construction (PII minimization applies there)
- Settings (secret loading) and any CORS/network middleware

## Reference

Detailed patterns and examples: [reference.md](reference.md)

## Output Format (Review Mode)

For each finding:

```
### [SEVERITY] Description
- **Location**: `file_path:line_number`
- **Principle**: Which checklist item, principle, or failure class applies
- **Issue**: What's wrong, who could exploit it, and what they'd get
- **Suggestion**: Concrete fix, with a code example if helpful
```

Severity levels:
- 🔴 **Critical**: missing authorization, BOLA, cross-tenant leak, excessive data exposure of PII, mass assignment of tenant/owner/role, injection, secret in source
- 🟠 **Important**: missing function-level check on a less sensitive operation, internal reason codes in responses, validation gaps, PII in logs
- 🟡 **Suggestion**: defense in depth, tighter scoping
- 💡 **Observation**: questions about trust boundaries, hardening ideas
