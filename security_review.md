# Security Review & Fix Plan

## Executive Summary

A security review was conducted on the git diff between `main` and branch `feature/lead-search`. The diff introduces a new search endpoint (`GET /api/leads/search`) in `server.ts`. 

The review identified **1 Critical** security vulnerability and **2 Low-to-Medium** security/hardening issues.

---

## Detailed Findings

### 1. SQL Injection (Critical)
* **Location:** [server.ts](file:///home/gurun_setiawan/leads-crm-app-demo/server.ts#L237)
* **Vulnerability Type:** OWASP A03:2021 – Injection
* **Description:** 
  The search endpoint constructs a raw SQL query string by concatenating `req.query.q` directly into the `SELECT` statement:
  ```typescript
  const sql = `SELECT * FROM leads WHERE name LIKE '%${query}%' OR company LIKE '%${query}%' OR email LIKE '%${query}%'`;
  const results = db.prepare(sql).all();
  ```
  An attacker can inject arbitrary SQLite syntax (e.g. using `' UNION SELECT ...` or SQL comments `--`), bypassing application logic and accessing or altering sensitive database content.

### 2. Information Disclosure in Error Responses (Medium)
* **Location:** [server.ts](file:///home/gurun_setiawan/leads-crm-app-demo/server.ts#L248-L251)
* **Vulnerability Type:** OWASP A01:2021 – Broken Access Control / Insecure Error Handling
* **Description:**
  When a database error occurs, internal error details (`error.message`) are returned in the JSON HTTP 500 response:
  ```typescript
  res.status(500).json({
    error: "Search failed",
    details: error.message,
  });
  ```
  Exposing raw error details reveals database table schemas, SQL dialect details, and stack traces to end users or potential attackers.

### 3. Missing Input Validation & Unsanitized Logging (Low)
* **Location:** [server.ts](file:///home/gurun_setiawan/leads-crm-app-demo/server.ts#L230-L240)
* **Vulnerability Type:** OWASP A04:2021 – Insecure Design / Data Validation
* **Description:**
  1. `req.query.q` is type-asserted as `as string` without runtime validation. Express query parameters can be arrays or objects if multiple parameters are passed (e.g., `?q[]=a&q[]=b`).
  2. Raw user input and un-sanitized SQL query strings are logged directly to standard output via `console.log`.

---

## Remediation Plan

Below is the proposed fix to remediate all identified security issues without altering expected behavior:

### Remediation Code Snippet

```typescript
app.get("/api/leads/search", (req, res) => {
  const query = req.query.q;

  // 1. Strict input type & presence check
  if (typeof query !== "string" || !query.trim()) {
    return res.status(400).json({ error: "Search query must be a non-empty string" });
  }

  const searchTerm = query.trim();

  try {
    // 2. Use parameterized queries with better-sqlite3 placeholders (?)
    const searchPattern = `%${searchTerm}%`;
    const stmt = db.prepare(`
      SELECT * FROM leads 
      WHERE name LIKE ? OR company LIKE ? OR email LIKE ?
      ORDER BY updated_at DESC
    `);

    const results = stmt.all(searchPattern, searchPattern, searchPattern);

    res.json(results);
  } catch (error: any) {
    // 3. Log detailed error internally, sanitize API response
    console.error("[search] Database error during search execution:", error);
    res.status(500).json({
      error: "Search failed due to an internal server error",
    });
  }
});
```

---

## Action Items Checklist (Pending Approval)

- [ ] Replace raw string concatenation in `server.ts` with parameterized SQLite bindings (`?`).
- [ ] Add runtime validation for `req.query.q` to handle non-string or whitespace-only inputs cleanly.
- [ ] Omit `error.message` from public HTTP 500 JSON response payloads.
- [ ] Remove raw SQL string logging that includes un-sanitized input.
