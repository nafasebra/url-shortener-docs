# URL Shortener - Data model documentation

At this file, We are going to determine what data models do you have in our system and how to generate short_code (It is the important)

## Data Models

We have URL entity and we should have a table to store all urls:

| Field        | Type | Required? | Purpose |
|--------------|------|-----------|---------|
| id           | Integer (BIGINT)   | Yes         | Primary Key (URL Identify)      |
| short_code   | String (VARCHAR(7))    | Yes         | short code for short URL       |
| original_url | String (TEXT)    | Yes         | Store original URL to redirect       |
| ip_address   | String (VARCHAR(15))    | Yes         | who       |
| created_at   | DateTime    | Yes         | ?       |
| deleted_at   | DateTime    | No         | ?       |


## How to generate __short_code__

As we have `id`, We can use `Base64()` function to Convert it to __short_code__. 

For example:

```
ID: 123456789
     ↓
Base62
     ↓
short_code: 8M0kX
```

This option is appropriate for this state of design system right now. When we Add authentication system to this system, We can use `random` method and manage collision when created a new short_code.


