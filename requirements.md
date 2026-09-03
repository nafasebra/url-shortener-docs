# URL shortener requirements

What is a URL Shortener?

A URL shortener is a piece of software that reduces the length of your current URL. As an internet user, you need this tool to share links across the internet :)

Suppose we want to implement a tool like bit.ly, but very simple and without any authentication. The user can just send their URL and see all the created URLs based on their IP address.

## Functional and non-functional requirements

### functional:

- User can submit a URL.
- System generates a short URL.
- A short URL maps to its original URL.
- User can retrieve all URLs created from their IP address.
- User can delete URLs they created.
- The same original URL should map to the same short URL.
- The system validates submitted URLs.
- URL ownership is determined by IP address.
- No authentication is required.

### non-functional: 
- Each IP can create at most one short URL every 15 seconds.
- The system should be able to handle URL creation and retrieval at scale.
- Do validation based frequently URL, IP and IP rate-limiting (at least 15 seconds).
- Use pagination to retrive URLs


## Capacity Estimation

peak DAU (active user per day) = 10000 user
peak URLs created per user per day = 5
peak URLs created per day = 50000
If 1 created URL → 10 redirects/day
peak Redirect request per day = 500000
