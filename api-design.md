# URL Shortener - API Design Documentation

In this document, we define how the client communicates with the system.

## 1. Create Short URL

**URL:** `POST /urls`

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
    "message": "URL was created successfully!",
    "data": {
        "id": "123",
        "short_url": "https://shrtnr.xyz/ab23c5e",
        "url": "https://example.com/blah"
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

**429 Too Many Requests** — Rate limit exceeded

```json
{
    "code": 429,
    "message": "Rate limit exceeded. Try again later."
}
```

---

## 2. Get All URLs

The backend identifies the user based on their IP address.

**URL:** `GET /urls`

The endpoint uses **cursor-based pagination**.

### Query Parameters

```text
limit
cursor
```

- `limit` — Maximum number of URLs to return.
- `cursor` — A cursor pointing to the position from which the next page should be retrieved.
- If `cursor` is not provided, the first page is returned.

### Example Request

```http
GET /urls?limit=20&cursor=eyJpZCI6MTIzfQ==
```

### Success Response

**Status:** `200 OK`

```json
{
    "code": 200,
    "message": "URLs retrieved successfully!",
    "data": [
        {
            "id": "123",
            "short_url": "https://shrtnr.xyz/ab23c5e",
            "url": "https://example.com/blah"
        },
        {
            "id": "122",
            "short_url": "https://shrtnr.xyz/xy91ka",
            "url": "https://example.com/another-url"
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
    "pagination": {
        "limit": 20,
        "next_cursor": null,
        "has_more": false
    }
}
```

### Error Responses

**400 Bad Request** — Invalid cursor or pagination parameters

```json
{
    "code": 400,
    "message": "Invalid pagination parameters."
}
```

---

## 3. Delete a Specific URL

**URL:** `DELETE /urls/{url_id}`

The backend must verify that the requested URL belongs to the user's IP address before deleting it.

### Success Response

**Status:** `204 No Content`

No response body is returned.

### Error Responses

**404 Not Found** — URL does not exist

```json
{
    "code": 404,
    "message": "URL not found."
}
```

**403 Forbidden** — URL does not belong to the requesting IP address

```json
{
    "code": 403,
    "message": "You do not have permission to delete this URL."
}
```

---

## 4. Redirect

**URL:** `GET /{shortCode}`

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
