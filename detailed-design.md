# URL Shortener - Detailed Design

This document describes the internal design of the URL Shortener system. It expands the high-level architecture, data model, and API behavior without changing the existing modular-monolith architecture.

## 1. Short Code Generation

Each globally unique URL receives one 7-character short code. The application generates the code with a cryptographically secure random-number generator and the following Base62 character set:

```text
abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789
```

```text
Cryptographically secure random generator
        |
        v
Select 7 characters from the Base62 alphabet
        |
        v
Attempt to insert short_code into PostgreSQL
        |
        +-- Insert succeeds: use the new code
        |
        +-- Unique conflict: generate another code and retry
```

The short code is independent of `urls.id`. The system does not Base62-encode a sequential database ID because sequential IDs would make codes predictable.

PostgreSQL enforces `UNIQUE(short_code)`. The application relies on this constraint instead of running a separate existence query before insertion. This avoids an extra database round trip and prevents a check-then-insert race condition.

### Collision Handling

1. Generate a candidate code with the secure random generator.
2. Attempt to insert the new `urls` row.
3. If `short_code` violates its unique constraint, generate another candidate and retry.
4. Stop after a small configured maximum, such as five attempts, and return an internal server error if every attempt collides.

A short code is generated only when the submitted `original_url` has no global row. When the URL already exists, its current short code is reused.

## 2. URL Creation

The authenticated user submits an `original_url`. The API authenticates the session, applies the per-user creation rate limit, validates the URL, performs global deduplication, and creates the user's ownership association.

```text
User submits original_url
        |
        v
Authenticate user from session cookie
        |
        v
Apply user-based rate limit
        |
        v
Validate and normalize URL
        |
        v
Look up urls row by original_url
        |
        +-- Found: reuse existing url_id and short_code
        |
        +-- Not found: generate code and insert global urls row
        |
        v
Create or restore active user_urls association
        |
        v
Return short_url
```

### URL Validation

Before accessing the URL tables, the application:

- requires a non-empty URL;
- accepts only supported schemes, normally `http` and `https`;
- rejects malformed URLs;
- enforces the configured maximum URL length; and
- applies the same deterministic normalization rules to every request.

Normalization must be conservative because it defines URL identity. Safe rules may include lowercasing the scheme and host and removing a default port. The system must not reorder query parameters, remove fragments, or otherwise change URL meaning unless that behavior is explicitly adopted for all clients.

The normalized value is stored as `original_url` and used for deduplication. Identical normalized values therefore produce the same short URL.

### Global Deduplication

The `urls` table represents global URL identity:

```sql
urls
-------------------------
id            BIGINT PRIMARY KEY
original_url  TEXT NOT NULL UNIQUE
short_code    VARCHAR(7) NOT NULL UNIQUE
created_at    TIMESTAMP NOT NULL
```

It does not contain `user_id`. If different users submit the same `original_url`, they reference the same `urls` row and receive the same `short_code`.

The creation service performs the lookup and insert inside a transaction. PostgreSQL's `UNIQUE(original_url)` constraint is the final concurrency guard. If two requests concurrently find no row and attempt to insert the same URL, one insert succeeds. The other receives an `original_url` uniqueness conflict, reads the winning row, and continues with its existing `url_id` and `short_code`.

A conflict on `short_code` has a different meaning: the generated code collided with another URL. The service generates a new code and retries the insert.

### User Ownership

The `user_urls` table records which global URLs appear in each user's account:

```sql
user_urls
-------------------------
user_id       BIGINT NOT NULL REFERENCES users(id)
url_id        BIGINT NOT NULL REFERENCES urls(id)
created_at    TIMESTAMP NOT NULL
deleted_at    TIMESTAMP NULL
```

PostgreSQL prevents duplicate active ownership associations with a partial unique index:

```sql
CREATE UNIQUE INDEX user_urls_active_unique
ON user_urls (user_id, url_id)
WHERE deleted_at IS NULL;
```

This PostgreSQL-specific index preserves soft-delete history while allowing a deleted association to be restored or recreated. The service should prefer restoring the most recent deleted association by setting `deleted_at` to `NULL`. If another concurrent request already created an active association, the partial-index conflict is treated as success and the active row is returned.

Deleting a URL from a user's account sets `user_urls.deleted_at`. It does not delete the global `urls` row, change the short code, or affect another user's association.

### Creation Response

On success, the service returns the same response shape whether it created a global URL, reused one, or restored a user association. The response includes the global URL identifier, original URL, short code, and complete short URL.

Expected failures are:

- `400 Bad Request` for invalid input;
- `401 Unauthorized` for a missing, invalid, or expired session;
- `429 Too Many Requests` when the creation rate limit is exceeded; and
- `500 Internal Server Error` after an unexpected database error or exhausted collision retries.

## 3. Redirect

Redirects are public and depend only on the global `urls` row. They do not require authentication and do not query `user_urls`.

```text
GET /{short_code}
        |
        v
Validate short-code format
        |
        v
Look up urls row by short_code
        |
        +-- Found: return redirect to original_url
        |
        +-- Not found: return 404 Not Found
```

The database lookup is conceptually:

```sql
SELECT original_url
FROM urls
WHERE short_code = $1;
```

The unique index on `short_code` provides an indexed lookup. A successful request returns the redirect status selected in the API design, currently `301 Moved Permanently`, with the original URL in the `Location` header.

User-level deletion does not disable a global redirect. A short code may still be owned by another user, and public links must remain stable. The endpoint returns `404 Not Found` only when no global short-code row exists; it does not return `410 Gone` because a user deleted their association.

## 4. Session Management

Authentication uses server-side sessions stored in Redis.

### Session Creation

After successful login:

1. The application generates a cryptographically secure, unguessable session ID.
2. It stores session data under `session:{session_id}` in Redis.
3. The value contains the minimum required identity data, including `user_id`.
4. Redis assigns the configured session TTL.
5. The API sends the session ID in a cookie.

The cookie uses `HttpOnly`, `Secure`, and an appropriate `SameSite` setting. Its path covers protected API routes, and production deployments transmit it only over HTTPS.

### Session Validation

For each protected request, authentication middleware reads the cookie, retrieves the session from Redis, rejects a missing or expired session with `401 Unauthorized`, and adds the authenticated `user_id` to the request context.

URL creation, listing, and ownership deletion use this context. The public redirect endpoint does not.

### Expiration and Logout

Redis removes sessions when their TTL expires. If sliding expiration is enabled, the application refreshes the TTL only after a valid authenticated request, consistently across all protected endpoints.

Logout deletes the Redis session key and expires the browser cookie. Revocation takes effect on the next request because Redis is the source of truth. Session records are not stored in PostgreSQL in V1.

## 5. Rate Limiting

Redis enforces the URL-creation limit per authenticated user. The existing rule allows one URL-creation request every 15 seconds.

```text
rate_limit:create_url:{user_id}
```

The service implements the rule atomically with:

```text
SET rate_limit:create_url:{user_id} 1 NX EX 15
```

- `OK` allows the request and starts the 15-second window.
- A null result means the key exists, so the API returns `429 Too Many Requests`.

The atomic command handles concurrent requests without a read-then-write race. The limit applies before global URL lookup, so submitting an existing URL still consumes creation capacity and protects the write path consistently.

Login attempts use a separate key and policy, typically based on account identity and client IP. Public redirects are excluded from this authenticated-user limiter. Infrastructure-level redirect protection may be added at the load balancer or edge if traffic requires it.

## 6. Caching Strategy

V1 reads redirects directly from PostgreSQL. The indexed `short_code` lookup is simple, and a separate cache is not required until measured traffic or database latency justifies it.

If redirect reads become a bottleneck, Redis can cache:

```text
redirect:{short_code} -> original_url
```

```text
Redirect request
        |
        v
Read redirect cache
        |
        +-- Hit: return redirect
        |
        +-- Miss: read PostgreSQL
                     |
                     v
                  Cache with TTL and return redirect
```

A finite TTL limits stale data and memory use. PostgreSQL remains the source of truth. If Redis is unavailable, redirect requests fall back to PostgreSQL.

Deleting a `user_urls` association does not require invalidation because it does not alter the global mapping. If the system later supports changing or disabling a global URL, that operation must invalidate its cache key.

Negative caching for unknown codes should use a short TTL, if introduced, so a cached miss cannot hide a newly created code for long.

## 7. Failure Handling

### PostgreSQL Unavailable

URL creation, listing, ownership deletion, and uncached redirects depend on PostgreSQL. When it is unavailable, the API fails quickly with `503 Service Unavailable` or the project's standard dependency-failure response. It must not report success before a transaction commits.

Connection and query timeouts are bounded. The application may retry a transient connection failure once when the operation is safe, but avoids broad retries that could multiply load during an outage.

### Redis Unavailable

Protected endpoints cannot validate server-side sessions when Redis is unavailable, so they fail closed with a temporary service error. They do not treat a session as authenticated without checking Redis.

The URL-creation limiter also fails closed because bypassing it during an outage could overload the database. Public redirects continue through PostgreSQL because Redis is not required for V1 redirect handling. If redirect caching is added later, cache failure falls back to PostgreSQL.

### Constraint Conflicts

- `short_code` conflict: generate a new code and retry.
- `original_url` conflict: read and reuse the existing global URL row.
- Active `(user_id, url_id)` conflict: read and return the existing association.

Other integrity violations are logged and returned as internal errors. Logs include request and constraint context but exclude session IDs, passwords, and other secrets.

### Timeouts and Retries

Every Redis and PostgreSQL operation has a bounded timeout. Retries use short, capped backoff and apply only to transient failures. Validation failures, authentication failures, rate-limit rejections, and permanent constraint errors are not retried.

The API attaches a request ID to logs and error responses so a failed operation can be traced across middleware, Redis, and PostgreSQL. Health checks distinguish application availability from dependency readiness, allowing the load balancer to stop routing traffic to an instance that cannot serve requests safely.

# What Is Next?

Continue to [API Design](api-design.md).
