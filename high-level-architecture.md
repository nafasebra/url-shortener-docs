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

Create short URL:

```text
Client
  | original_url + session cookie
  v
API Server
  | session_id
  v
Redis
  | user_id
  v
Rate Limiter
  | user_id-based rate-limit state
  <-> Redis
  |
  v
URL Module
  | user_id + original_url + short_code
  v
PostgreSQL
  |
  v
short_url
  |
  v
Client
```

## Architectural Decisions

<!-- Placeholder: document the major decisions here, including the modular monolith, PostgreSQL for persistent users/URLs, Redis for sessions and rate-limiting state, secure HttpOnly cookies for sessions, and user_id-based rate limiting for authenticated URL creation. -->

## Scalability Considerations

<!-- Placeholder: document scalability considerations here. -->
