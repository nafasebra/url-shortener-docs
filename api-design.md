# URL Shortener - API Design Documentation

This document defines how the client communicates with the system.

## Authentication

Protected endpoints require authentication.

```http
Authorization: Bearer <access_token>
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

```json
{
  "code": 200,
  "message": "Login successful.",
  "data": {
    "access_token": "<access_token>"
  }
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

### Success Response

**Status:** `204 No Content`

No response body is returned.

---

## 4. Create Short URL

**URL:** `POST /urls`

**Authentication:** Required

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
  "message": "URL created successfully.",
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

The backend identifies the user using their authenticated `user_id`.

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
Authorization: Bearer <access_token>
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
      "short_code": "xy91ka",
      "short_url": "https://shrtnr.xyz/xy91ka",
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

The backend verifies that the URL belongs to the authenticated user before deleting it.

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

**410 Gone** — Short URL was deleted

```json
{
  "code": 410,
  "message": "Short URL has been deleted."
}
```

---

## Client Behavior

The client handles API responses as follows:

- After successful registration, the client redirects the user to the `/login` page.
- The user can then log in using the `/auth/login` endpoint.
- If a protected endpoint returns `401 Unauthorized`, the client redirects the user to the `/login` page.
- After successful login, the client stores the access token securely and uses it for subsequent protected requests.

---

# What Is Next?

Continue → [Data Model](data-model.md)
