# URL Shortener - API Design Documentation

This document defines how the client communicates with the system.

## Authentication

Protected endpoints require authentication.

The system uses session-based authentication.

After a successful login, the API Server generates a random `session_id`, stores it in Redis with the authenticated `user_id`, and sends it to the client using a secure `HttpOnly` cookie.

The client automatically sends the session cookie with authenticated requests.
For protected endpoints, the API Server reads the session cookie, looks up the session in Redis, and determines the authenticated `user_id`.

```http
Cookie: session_id=<session_id>
```

---

## 1. Register

**URL:** `POST /auth/register`

### Request

```json
{
  "email": "user@example.com",
  "password": "StrongPassword123!"
}
```

### Success Response

**Status:** `201 Created`

```json
{
  "code": 201,
  "message": "User registered successfully."
}
```

### Error Responses

**400 Bad Request** — Invalid input

```json
{
  "code": 400,
  "message": "Invalid email or password."
}
```

**409 Conflict** — Email already exists

```json
{
  "code": 409,
  "message": "Email is already registered."
}
```

---

## 2. Login

**URL:** `POST /auth/login`

### Request

```json
{
  "email": "user@example.com",
  "password": "StrongPassword123!"
}
```

### Success Response

**Status:** `200 OK`

The response sets a secure `HttpOnly` session cookie.

```http
Set-Cookie: session_id=<session_id>; HttpOnly; Secure; SameSite=Lax; Path=/
```

```json
{
  "code": 200,
  "message": "Login successful."
}
```

### Error Responses

**401 Unauthorized** — Invalid credentials

```json
{
  "code": 401,
  "message": "Invalid email or password."
}
```

**429 Too Many Requests** — Rate limit exceeded

```json
{
  "code": 429,
  "message": "Too many login attempts. Try again later."
}
```

---

## 3. Logout

**URL:** `POST /auth/logout`

**Authentication:** Required

Logout deletes the session from Redis and clears the session cookie.

### Success Response

**Status:** `204 No Content`

No response body is returned.

---

## 4. Create Short URL

**URL:** `POST /urls`

**Authentication:** Required

The client sends the session cookie automatically. The backend resolves the authenticated `user_id` from Redis before applying the URL creation rate limit.

The same original URL always maps to the same short URL across the system. If the submitted URL already exists in the global `urls` table, the backend reuses the existing `short_code` and creates or restores the authenticated user's `user_urls` ownership association instead of generating a new short code.

### Request

```json
{
  "url": "https://example.com/blah"
}
```

### Success Response

**Status:** `201 Created`

```json
{
  "code": 201,
  "message": "URL created or retrieved successfully.",
  "data": {
    "id": "123",
    "short_code": "ab23c5e",
    "short_url": "https://shrtnr.xyz/ab23c5e",
    "url": "https://example.com/blah",
    "created_at": "2026-09-21T10:00:00Z"
  }
}
```

### Error Responses

**400 Bad Request** — Invalid URL

```json
{
  "code": 400,
  "message": "Invalid URL."
}
```

**401 Unauthorized** — Authentication required

```json
{
  "code": 401,
  "message": "Authentication required."
}
```

**429 Too Many Requests** — Rate limit exceeded

```json
{
  "code": 429,
  "message": "Rate limit exceeded. Try again later."
}
```

---

## 5. Get User's URLs

**URL:** `GET /urls`

**Authentication:** Required

The backend identifies the user by reading the session cookie and resolving the authenticated `user_id` from Redis.

The endpoint uses cursor-based pagination.

### Query Parameters

```text
limit
cursor
```

- `limit` — Maximum number of URLs to return.
- `cursor` — Cursor pointing to the next page.
- If `cursor` is not provided, the first page is returned.

### Example Request

```http
GET /urls?limit=20&cursor=eyJpZCI6MTIzfQ==
Cookie: session_id=<session_id>
```

### Success Response

**Status:** `200 OK`

```json
{
  "code": 200,
  "message": "URLs retrieved successfully.",
  "data": [
    {
      "id": "123",
      "short_code": "ab23c5e",
      "short_url": "https://shrtnr.xyz/ab23c5e",
      "url": "https://example.com/blah",
      "created_at": "2026-09-21T10:00:00Z"
    },
    {
      "id": "122",
      "short_code": "Q7mK2xP",
      "short_url": "https://shrtnr.xyz/Q7mK2xP",
      "url": "https://example.com/another-url",
      "created_at": "2026-09-20T10:00:00Z"
    }
  ],
  "pagination": {
    "limit": 20,
    "next_cursor": "eyJpZCI6MTAyfQ==",
    "has_more": true
  }
}
```

When there are no more URLs:

```json
{
  "code": 200,
  "message": "URLs retrieved successfully.",
  "data": [],
  "pagination": {
    "limit": 20,
    "next_cursor": null,
    "has_more": false
  }
}
```

### Error Responses

**400 Bad Request** — Invalid pagination parameters

```json
{
  "code": 400,
  "message": "Invalid pagination parameters."
}
```

**401 Unauthorized** — Authentication required

```json
{
  "code": 401,
  "message": "Authentication required."
}
```

---

## 6. Delete a Specific URL

**URL:** `DELETE /urls/{url_id}`

**Authentication:** Required

The backend verifies that the URL belongs to the authenticated user before deleting it from that user's account.
Deletion soft-deletes the `user_urls` ownership association by setting `deleted_at`; it does not delete the global `urls` row because other users may still own the same short URL.

### Success Response

**Status:** `204 No Content`

No response body is returned.

### Error Responses

**401 Unauthorized** — Authentication required

```json
{
  "code": 401,
  "message": "Authentication required."
}
```

**404 Not Found** — URL does not exist or does not belong to the user

```json
{
  "code": 404,
  "message": "URL not found."
}
```

---

## 7. Redirect

**URL:** `GET /{short_code}`

**Authentication:** Not required

The system redirects the client to the original URL.

Public short-URL redirects do not require a session cookie and are not subject to the authenticated user-based URL creation rate limiter.

### Success Response

**Status:** `301 Moved Permanently`

```http
HTTP/1.1 301 Moved Permanently
Location: https://example.com/blah
```

### Error Responses

**404 Not Found** — Short code does not exist

```json
{
  "code": 404,
  "message": "Short URL not found."
}
```

---

## Client Behavior

The client handles API responses as follows:

- After successful registration, the client redirects the user to the `/login` page.
- The user can then log in using the `/auth/login` endpoint.
- If a protected endpoint returns `401 Unauthorized`, the client redirects the user to the `/login` page.
- After successful login, the browser stores the secure `HttpOnly` session cookie set by the API Server.
- For subsequent protected requests, the browser sends the session cookie automatically.

---

# What Is Next?

Continue -> [Data Model](data-model.md)
