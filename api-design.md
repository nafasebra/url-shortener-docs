# URL Shortener - API Design documentation

In this document, we want to determine how the client communicates with the system.

1. Create **short-URL**

URL: `POST /urls`

Request:

```json
{
    url: "https//example.com/blah"
}
```

Response

```json
{
    "code": 201,
    "message": "URL was created successfuly!"
}
```

2. Get all URLs 

Send it. in backend side, we'll recognize user IP.

url: `GET /urls`

Response: 

```json
[
    {
        "id": "123",
        "short_url": "https://shrtnr.xyz/ab23c5e",
        "url": "https://example.com/blah",
    }
]
```
