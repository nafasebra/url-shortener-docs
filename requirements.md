# URL shortener requirements

Suppose we want to implement a simple URL shortener like Bitly. Users can register, log in, and manage the URLs created under their accounts.

## Functional and non-functional requirements

### functional:

- User can register, log in and log out.
- User can create URLs under their account.
- Unauthenticated users cannot access protected endpoints.
- The system uses session-based authentication.
- After login, the authenticated session is stored in Redis and sent to the client as a secure `HttpOnly` cookie.
- Logout deletes the session from Redis and clears the session cookie.
- Users can access short URLs and be redirected to the original URLs.
- The system returns an appropriate error when a short URL does not exist.
- User can retrieve only their own URLs.
- User can delete only their own URLs.
- The system generates a short URL.
- A short URL maps to its original URL.
- The same original URL should always map to the same short URL across the system.
- The system validates submitted URLs.

### non-functional:
- Each authenticated user can create at most one short URL every 15 seconds.
- The system should handle URL creation and retrieval at scale.
- Use cursor-based pagination to retrieve URLs.
- Passwords must be securely hashed before storage.
- Authentication endpoints should have rate limiting.
- Sessions should have a TTL so they expire automatically.
- The system should use HTTPS to protect user credentials and session cookies.
- Public short-URL redirects do not require authentication and are not subject to the authenticated user-based URL creation rate limiter.

# What Is Next?

Continue -> [Capacity Estimation](capacity-estimation.md)
