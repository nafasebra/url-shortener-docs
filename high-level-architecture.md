# URL Shortener - High-Level Architecture

This document explains how the system is structured and how its main components interact.

## Overview

The system will use a Modular Monolith architecture for the initial version.

The backend will run as a single application with separate modules for authentication, URL management, and rate limiting. PostgreSQL will be used as the primary database, while Redis will be used for rate limiting.

This architecture keeps the initial system simple while maintaining clear module boundaries that can be separated into independent services if the system needs to scale in the future.

## Diagram

```text
                 Client
                    |
                    v
             ┌──────────────┐
             │  API Server  │
             │              │
             │ Auth Module  │
             │ URL Module   │
             │ Rate Limiter │
             └──────┬───────┘
                    |
             +------+------+
             |             |
             v             v
          Redis       PostgreSQL
```

### Client

The client evolves with the system. Users can sign up, log in, enter a URL, and generate a short URL. They can share the short URL with others.

When another user clicks on the short URL, the request is sent to the system, which redirects the user to the original URL.

### API Server

The API Server receives requests from the client and returns appropriate responses. It contains the Auth, URL, and Rate Limiter modules.

The Auth and URL modules handle authentication and URL-related operations, while the Rate Limiter controls request frequency.

The API Server communicates with Redis for temporary data such as rate-limiting information and with PostgreSQL for persistent data such as users and URLs.

### Redis

In V1, Redis is used only to store rate-limiting state.

```text
                API Server
             /      |       \
            /       |        \
       Auth     URL Module   Rate Limiter
                                  |
                                  ↓
                                Redis
```

### Database (PostgreSQL)

PostgreSQL is the primary persistent database for the application. Based on the [Data Model](data-model.md), it stores permanent data such as users and URLs.

## Request Flow


### Create Short URL

```

Client
  ↓
API Server
  ↓
Authentication
  ↓
Rate Limiter
  ↓
URL Module
  ↓
PostgreSQL
  ↓
Client

```

### Redirect

## Data Flow

```
User
 ↓
Auth Module
 ↓
PostgreSQL

URL
 ↓
URL Module
 ↓
PostgreSQL

Rate Limit
 ↓
Redis
```

## Architectural Decisions

## Scalability Considerations
