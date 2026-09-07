# URL Shortener - Data Model Documentation

In this document, we define the data models used in the system and how we generate the `short_code`.

## Data Models

We have a `URL` entity and a table to store all created URLs.

| Field | Type | Required? | Purpose |
|---|---|---|---|
| `id` | BIGINT | Yes | Primary key and internal URL identifier |
| `short_code` | VARCHAR(7) | Yes | Short code used in the short URL |
| `original_url` | TEXT | Yes | Stores the original URL for redirection |
| `ip_address` | VARCHAR(45) | Yes | Identifies the client that created the URL |
| `created_at` | DateTime | Yes | Used to track when the URL was created and for sorting/pagination |
| `deleted_at` | DateTime | No | Used for soft deletion |

## How to Generate `short_code`

Since we already have a unique `BIGINT` ID, we can use **Base62 encoding** to generate the `short_code`.

Base62 uses 62 characters:

- A-Z: 26 characters
- a-z: 26 characters
- 0-9: 10 characters

For example:

```text
ID: 123456789
       ↓
   Base62 Encode
       ↓
short_code: 8M0kX
```

Using the database ID as the source of the `short_code` guarantees uniqueness because the ID is a unique primary key.

This approach is appropriate for the current stage of the system because it is simple, fast, and does not require collision handling.

However, the generated short codes are predictable and can potentially be enumerated. If privacy becomes an important requirement, we can switch to a random or non-sequential short-code generation strategy and handle collisions using a unique constraint on `short_code`.
