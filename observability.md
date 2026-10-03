# URL Shortener - Observability & Monitoring

This document describes how we monitor the health and performance of the URL Shortener system.

## 1. Metrics

We collect metrics to understand the traffic, performance, and errors of the system.

### API Metrics

The API metrics measure traffic, latency, and failures.

The system tracks:

- Request count by endpoint
- Request rate (requests per second)
- Error count by status code
- Error rate
- Request latency (p50, p95, p99)


### PostgreSQL Metrics

The database might has high latency, failure or high connection usage and we need to measure these.

These metrics help us understand whether PostgreSQL can handle the current load and identify database bottlenecks when query latency increases or connection usage becomes high.

- Query latency
- Slow query count
- Database errors
- Active connections
- Connection pool usage

### Redis Metrics

Redis is used for session storage and rate limiting. As the number of users and requests increases, we monitor Redis to detect performance or availability problems.

- Operation latency
- Redis errors
- Session lookup failures
- Missing or expired sessions
- Rate-limit operations

## 2. Logging

Logs are used to investigate errors, debug unexpected behavior, and understand important events inside the system.

The application should log enough information to investigate a problem without exposing sensitive user or authentication data.

### Log Levels

The application uses three main log levels:

- `INFO`: Normal and important system events.
- `WARN`: Unexpected or unusual events that the system can recover from.
- `ERROR`: Events where an operation fails or an important dependency is unavailable.

Not every successful request needs to be logged. High-volume information such as request counts and successful redirects should primarily be tracked through metrics.

### Logged Events

#### INFO

Examples of normal but useful system events:

- Application started or stopped
- Important application configuration or lifecycle events
- Successful completion of important background or maintenance operations, if introduced later

High-volume events such as every successful redirect or URL creation are not logged individually in V1.

#### WARN

Recoverable or unusual events include:

- Rate limit exceeded
- Short-code collision detected and retried
- Invalid or expired session
- Invalid request rejected by validation
- Expected database constraint conflict that can be recovered from

For example, a short-code collision is logged as a warning because the application can generate another code and retry the operation.

#### ERROR

Errors are logged when an operation cannot complete successfully:

- PostgreSQL query or connection failure
- PostgreSQL timeout
- Redis connection or operation failure
- URL creation failure
- Unexpected internal server error
- Dependency failure that causes a request to return a `5xx` response

### Log Context

Each log should contain enough context to identify and investigate the event.

Depending on the event, logs may include:

- Timestamp
- Log level
- Request ID
- HTTP method
- Endpoint
- HTTP status code
- Operation name
- Request duration
- Error type or error message
- Relevant non-sensitive identifiers such as `url_id` or `short_code`

The `request_id` is used to correlate multiple log entries generated while processing the same request.

For example:

```text
timestamp=2026-10-03T12:30:10Z
level=ERROR
request_id=req_7281
method=POST
endpoint=/urls
operation=create_url
error="database query timeout"
status=503
duration_ms=2050
```

### Sensitive Data

Logs must not contain sensitive authentication or user data.

The application must not log:

- Passwords
- Password hashes
- Session IDs
- Authentication cookies
- Authentication secrets
- Database credentials
- Full request bodies when they may contain sensitive data
- Full original URLs when they may contain tokens, credentials, or private information

When possible, non-sensitive identifiers such as `url_id`, `short_code`, or `request_id` should be used instead of sensitive values.

### Logging Failures

Logging should not cause the main request to fail.

If the logging system becomes unavailable, the application should continue serving requests when possible. Logging failures should therefore be handled separately from failures in critical dependencies such as PostgreSQL or Redis.

## 3. Distributed Tracing

## 4. Profiling

## 5. Alerts

## 6. Health Checks
