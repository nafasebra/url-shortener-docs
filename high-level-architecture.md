# URL Shortener - High-Level Architecture

This document explains how the system is structured and how its main components interact.

## Overview

The system will use a Modular Monolith architecture for the initial version.

The backend will run as a single application with separate modules for authentication, URL management, and rate limiting. PostgreSQL will be used as the primary database, while Redis will be used for session storage and rate-limiting state.

This architecture keeps the initial system simple while maintaining clear module boundaries that can be separated into independent services if the system needs to scale in the future.

## Diagram

```text
Client
   |
   v
API Server
   |
   +-- Auth Module
   |
   +-- URL Module
   |
   +-- Rate Limiter
   |
   +------ Redis
   |        |-- Sessions
   |        `-- Rate-limiting state
   |
   +------ PostgreSQL
            |-- Users
            `-- URLs
```

### Client

The client evolves with the system. Users can sign up, log in, enter a URL, and generate a short URL. They can share the short URL with others.

When another user clicks on the short URL, the request is sent to the system, which redirects the user to the original URL.

### API Server

The API Server receives requests from the client and returns appropriate responses. It contains the Auth, URL, and Rate Limiter modules.

The Auth and URL modules handle authentication and URL-related operations, while the Rate Limiter controls request frequency.

The API Server communicates with Redis for temporary data such as sessions and rate-limiting information. It communicates with PostgreSQL for persistent data such as users and URLs.

### Redis

In V1, Redis has two responsibilities:

1. Session storage
2. Rate-limiting state

After a successful login, the API Server generates a random `session_id`, stores it in Redis, and maps it to the authenticated `user_id`. The `session_id` is sent to the client using a secure `HttpOnly` cookie. Sessions have a TTL so they expire automatically.

For authenticated URL creation, Redis also stores rate-limiting state based on `user_id`.

```text
                API Server
             /      |       \
            /       |        \
       Auth     URL Module   Rate Limiter
         |                        |
         |                        |
         +---------- Redis -------+
                    /     \
             Sessions   Rate-limiting state
```

### Database (PostgreSQL)

PostgreSQL is the primary persistent database for the application. Based on the [Data Model](data-model.md), it stores permanent data such as users and URLs.

Session IDs are not stored in PostgreSQL.

## Request Flow

### Create Short URL

```text
Client
  |
  v
API Server
  |
  v
Authentication
  |
  v
Rate Limiter
  |
  v
URL Module
  |
  v
PostgreSQL
  |
  v
Client
```

The client signs up or logs in and is redirected to the homepage. The user enters a URL and submits it to generate a short URL.
The client must be authenticated to create a short URL. If the client is not authenticated, the API returns a `401 Unauthorized` error.
The API Server reads the session cookie, looks up the session in Redis, and determines the authenticated `user_id`.
The authenticated user can make a maximum of `X` requests within one second. If the rate limit is exceeded, the API returns a `429 Too Many Requests` error.
If the request is allowed, the URL Module validates the original URL, generates a short code, and stores the URL in PostgreSQL. The generated short URL is then returned to the client.

### Redirect

```text
Client
  |
  v
API Server
  |
  v
URL Module
  |
  v
PostgreSQL
  |
  v
301 Redirect
  |
  v
Original URL
```

Anyone can access a generated short URL. Authentication is not required for redirects.
The URL Module looks up the short code in PostgreSQL. If the short URL exists, the API returns a `301` redirect to the original URL.
Public redirects are not subject to the authenticated user-based URL creation rate limiter.

### Authentication Session Flow

```text
Client
  |
  v
API Server
  |
  v
Auth Module
  |
  v
Redis
  |
  v
Secure HttpOnly cookie
  |
  v
Client
```

After successful login, the API Server creates a random `session_id`, stores it in Redis with the authenticated `user_id`, and sends the `session_id` to the client in a secure `HttpOnly` cookie.

For logout, the API Server deletes the session from Redis and clears the session cookie.

## Data Flow

### Login / Session Data Flow

```text
Client
  │ email + password
  ↓
API Server / Auth Module
  │ query user
  ↓
PostgreSQL
  │ user data
  ↓
Auth Module
  │ create session_id
  │
  ├── Store session_id → user_id in Redis
  │
  └── Set session_id as HttpOnly cookie
                    ↓
                  Client
```


### Create Short URL Data Flow

```
Client
  │ session cookie + original_url
  ↓
API Server
  │
  ├── session_id → Redis
  │                 ↓
  │               user_id
  ↓
Rate Limiter
  │ user_id
  ↔ Redis
  ↓
URL Module
  │ user_id + original_url + short_code
  ↓
PostgreSQL
  ↓
API Server
  │ short_url
  ↓
Client
```


### Redirect Data Flow

```text
Client
  │ short_code
  ↓
API Server / URL Module
  │ short_code
  ↓
PostgreSQL
  │ original_url
  ↓
API Server
  │ 301 Location: original_url
  ↓
Client
```

## Architectural Decisions

### Why Modular Monolith?

Our application is relatively simple and currently has two main entities: User and URL. A modular monolith keeps the architecture simple while allowing us to separate responsibilities into modules such as Auth and URL management.

At this stage, we do not need the additional complexity of microservices. If the system grows in the future, the clear module boundaries can make it easier to separate parts of the application into independent services.

### Why Redis?

Redis is an in-memory key-value data store. We use it for temporary data such as sessions and rate-limiting state.

Keeping this state outside the API Server also helps us scale the application horizontally in the future. If we run multiple API Server instances behind a load balancer, they can share the same session and rate-limiting state through Redis.

### Why PostgreSQL?

We need persistent storage for users and URLs. PostgreSQL is a relational database and is a good fit because our data has a clear relationship: a user can own multiple URLs.

It also provides features such as constraints, indexes, and transactions that are useful for maintaining consistent application data.

## Scalability Considerations

The initial version runs as a single backend application, but the system can be scaled horizontally by running multiple API Server instances behind a load balancer.

Redis stores shared session and rate-limiting state, allowing all API Server instances to access the same temporary data.

PostgreSQL remains the primary persistent database. As traffic grows, we can introduce additional scaling strategies based on the system's bottlenecks.

Session-based authentication can still be used when the application scales because sessions are stored centrally in Redis and shared between API Server instances.

