# URL shortener requirements

Suppose we want to implement a simple URL shortener like Bitly. Users can register, log in, and manage the URLs created under their accounts.

## Functional and non-functional requirements

### functional:

- User can register, log in and log out.
- User can create URLs under their account.
- Unauthenticated users cannot access protected endpoints.
- Users can access short URLs and be redirected to the original URLs.
- The system returns an appropriate error when a short URL does not exist.
- User can retrieve only their own URLs.
- User can delete only their own URLs.
- The system generates a short URL.
- A short URL maps to its original URL.
- The same original URL should always map to the same short URL across the system.
- The system validates submitted URLs.

### non-functional: 
- Each IP can create at most one short URL every 15 seconds.
- The system should handle URL creation and retrieval at scale.
- Use cursor-based pagination to retrieve URLs.
- Passwords must be securely hashed before storage.
- Authentication endpoints should have rate limiting.
- The system should use HTTPS to protect user credentials and tokens.

## Capacity Estimation

Continue -> [Capacity Estimation](capacity-estimation.md)
