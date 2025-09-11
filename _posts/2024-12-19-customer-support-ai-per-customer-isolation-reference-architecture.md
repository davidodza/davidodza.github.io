---
layout: post
published: true
categories:
  - AI
  - Security
  - Architecture
  - DevSecOps
mathjax: false
featured: false
comments: true
title: Customer-Support AI with Per-Customer Isolation Reference Architecture
---

# Reference architecture — Customer-support AI with per-customer isolation

_This document sketches a secure, production-ready reference architecture for giving an LLM-based support assistant access to customer data while ensuring strict per-customer isolation and auditability._

---

## High-level goals
- **Least privilege:** AI can only read/write data for the specific customer in the current session.
- **Defense-in-depth:** multiple layers (gateway, tokens, RLS, auditing) enforce constraints.
- **Ephemeral context:** session state isn't reused across customers and is short-lived.
- **Auditability & observability:** every AI access is logged, traceable, and reviewable.

---

## Core components
1. **User / Agent UI** — chat widget or support console used by agent/customer.
2. **Session Manager** — backend component that creates a per-interaction session and issues ephemeral scoped credentials (access token tied to `customer_id` + session id).
3. **API Gateway / Mediator (Policy Enforcer)** — single place the LLM uses to fetch data or call actions. Enforces filters (adds `WHERE customer_id = X`), rate limits, and templates.
4. **MCP / Tool Adapter / LLM Tooling** — the model-facing adapter that exposes actions the LLM may call (e.g., `get_customer_profile`, `list_tickets`, `update_ticket`). These endpoints accept the ephemeral token.
5. **Data Stores** — CRM, ticketing DB, analytics (e.g., Postgres, BigQuery, S3). Each store enforces RLS where possible.
6. **Secrets & Token Service** — short-lived signing keys, vault for service credentials.
7. **Audit Log & SIEM** — centralized event store for request/response logs and DLP scanning.
8. **Human Review Layer** (optional) — for high-risk operations, require agent approval.

---

## Sequence flow (simplified)

![Customer Support AI Sequence Diagram]({{site.baseurl}}/images/customer-support-ai/sequence-diagram.png)

---

## Key implementation patterns (concrete)

### 1) Ephemeral scoped tokens
- Issue JWTs scoped to a single `customer_id`, `session_id`, and short TTL (minutes).
- Token claims example:
```json
{
  "iss": "auth.mycompany.com",
  "sub": "mediator-service",
  "aud": "mcp-gateway",
  "iat": 1690000000,
  "exp": 1690000600,
  "customer_id": "cust_123",
  "session_id": "sess_abcd",
  "scopes": ["read:profile","read:tickets"]
}
```
- Private signing key stored in a vault; public key used to verify in gateway.

### 2) Mediator / Policy Gateway responsibilities
- Validate token and claims.
- Map high-level tool calls to safe queries or action templates.
- Insert enforced filters (e.g., `WHERE customer_id = :customer_id`).
- Rate-limit and sanitize inputs (prevent `*` or arbitrary SQL passage).
- Log full request metadata (tool call, caller session, resolved SQL/operation hash, response size).
- Return only fields allowed by scope (mask PII if necessary).

### 3) Data-layer enforcement (RLS / Authorized Views)
- **Postgres RLS example:**
```sql
ALTER TABLE tickets ENABLE ROW LEVEL SECURITY;

CREATE POLICY customer_isolation ON tickets
  USING (customer_id = current_setting('app.customer_id')::text);
```
Before executing queries, the mediator sets the Postgres session var:
```sql
SET app.customer_id = 'cust_123';
```
- **BigQuery authorized view pattern:** expose a view that filters by `customer_id` and grant the mediator only view access.

### 4) Tool interface design (explicit actions)
- Expose **whitelisted** tool functions (no arbitrary SQL). Example endpoints:
  - `GET /tools/get_customer_profile?customer_id=`
  - `POST /tools/search_tickets` (templated search fields)
  - `POST /tools/perform_billing_action` (requires extra approval)
- Each endpoint verifies token `customer_id` == requested `customer_id`.

### 5) Ephemeral context & cache
- Keep chat history and retrieved docs tied to `session_id`.
- TTL and immediate invalidation after session end.
- If storing embeddings, prefix keys with `cust_{id}` and enforce access control on retrieval.

### 6) Auditing & DLP
- Log: tool call, token claims, resolved query, response summary, and LLM prompt.
- Send logs to SIEM and run automated DLP checks to detect data exfil patterns.

---

## Example code snippets (pseudo)

**Mediator: validate token and enforce customer**
```python
claims = verify_jwt(token)
if claims['customer_id'] != request.params.get('customer_id'):
    raise Unauthorized()
# Build safe query with param binding
rows = db.query("SELECT id,subject FROM tickets WHERE customer_id = %s AND status=%s",
                (claims['customer_id'], 'open'))
```

**Postgres session var approach**
```python
conn.execute("SET app.customer_id = %s", (claims['customer_id'],))
conn.execute("SELECT * FROM tickets WHERE id = %s", (ticket_id,))
```

---

## Additional controls (recommended for higher security)
- **Break-glass & human approval** for destructive operations.
- **Data minimization**: return only the minimal fields needed by the LLM.
- **Differential access**: separate read-only vs write-capable mediator tokens.
- **Periodic secret rotation** and short TTLs for tokens.
- **Mutation hashing / idempotency**: require operation confirmations from agents.

---

## Quick integration checklist
- [ ] Issue ephemeral JWTs scoped to `customer_id` + `session_id`.
- [ ] Implement policy gateway that validates tokens and enforces `customer_id` filter.
- [ ] Add RLS or authorized views on all sensitive tables.
- [ ] Whitelist tool endpoints; disallow arbitrary query injection.
- [ ] Centralize audit logs and DLP scanning.
- [ ] Ensure ephemeral context (session cache) is namespaced and TTL'd.
- [ ] Add human approval for high-risk actions.

---

## Tradeoffs & notes
- RLS gives strong protection but increases DB complexity.
- Mediator simplifies policy enforcement but is a single point to secure and scale.
- Ephemeral tokens reduce blast radius but require a reliable token service.

---
