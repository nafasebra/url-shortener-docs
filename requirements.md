# URL shortener requirements

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


## Note

IP address is used as a temporary identity mechanism for the MVP. This approach does not provide reliable user identity because multiple users may share an IP address and a single user may use multiple IP addresses. A future version should introduce authentication and a persistent user ID.

## Capacity Estimation

Continue -> [Capacity Estimation](capacity-estimation.md)
