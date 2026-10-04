# URL Shortener - Testing & Load Testing

This document describes the testing strategy for verifying the correctness, reliability, and performance of the URL Shortener system.

## 1. Testing Strategy

We need tests to verify that the application behaves as expected during development.

Manual testing is time-consuming and can be error-prone, so automated tests help us verify the system consistently after making changes.

Automated tests can also be integrated into the CI/CD pipeline to detect problems before changes are deployed.

## 2. Unit Testing

Unit tests verify small and isolated parts of the application, such as business logic and utility functions.

We write unit tests for important application logic and cover happy paths, edge cases, and error cases.

For example, unit tests can be used for:

- URL validation and normalization
- Short-code generation
- Rate-limiting logic
- Other isolated business rules

Unit tests should not require external dependencies such as PostgreSQL or Redis.

## 3. Integration Testing

Different parts of the application need to work together, so integration tests verify the interaction between application components and external dependencies.

Integration tests can use real test dependencies such as PostgreSQL and Redis.

For example, we can test:

- User registration with PostgreSQL
- User login and session creation in Redis
- URL creation and storage in PostgreSQL
- URL lookup from PostgreSQL
- Rate limiting with Redis

## 4. End-to-End Testing

End-to-end tests verify the application from the user's perspective.

These tests are based on the business requirements and verify complete user flows through the system.

Important E2E scenarios include:

- A user registers or logs in and creates a short URL.
- An authenticated user retrieves their created URLs.
- An authenticated user deletes one of their URL associations.
- An unauthenticated user opens a short URL created by another user and is redirected to the original URL.
- An unauthenticated user cannot access protected endpoints.
- A user cannot retrieve or delete URL associations owned by another user.

## 5. Load Testing

Load testing helps us understand how the system behaves under expected traffic and identify performance bottlenecks.

### Workloads

The system has two main workloads that should be tested:

- URL creation
- URL redirect

Redirect is the main workload because it receives significantly more traffic than URL creation.

We gradually increase the request rate for these operations and observe how the system behaves under load.

### Scenarios

The target load is based on the traffic estimated in the Capacity Estimation.

For URL creation:

- Average load: ~1.16 requests per second
- Expected peak load: ~12 requests per second

For URL redirects:

- Average load: ~579 requests per second
- Expected peak load: ~5,790 requests per second

During load testing, the request rate should be increased gradually instead of immediately sending the expected peak traffic.

For example, redirect traffic can be increased through several stages:

```text
100 req/s
↓
500 req/s
↓
1,000 req/s
↓
3,000 req/s
↓
5,790 req/s
```

This helps us identify when performance starts to degrade and which part of the system becomes the bottleneck.

### Metrics

During load testing, we collect metrics to understand how the system performs under increasing traffic and identify possible bottlenecks.

#### Request Metrics

We measure:

- Throughput (requests per second)
- Request latency (p50, p95, p99)
- Error rate

These metrics help us understand whether the application can handle the generated traffic while maintaining acceptable response times and error rates.

#### Resource and Dependency Metrics

We also monitor the resources and dependencies used by the application:

**Application**

- CPU usage
- Memory usage

**PostgreSQL**

- Query latency
- Active connections
- Connection pool usage
- Database errors

**Redis**

- Operation latency
- Redis errors

These metrics help us identify which part of the system becomes a bottleneck when performance starts to degrade.

### Success Criteria

A load test is considered successful when the system can handle the expected traffic while maintaining acceptable performance and reliability.

We evaluate:

- Whether the system can process the expected request rate
- Whether request latency remains within an acceptable range
- Whether the error rate remains within an acceptable range
- Whether the application and its dependencies remain stable during sustained load

Exact latency and error-rate thresholds will be defined and adjusted based on performance requirements and load-testing results.

## 6. Failure Testing

Failure testing verifies how the system behaves when an application component or dependency becomes unavailable or slow.

The goal is to ensure that dependency failures do not cause unexpected application behavior and that the system fails in a controlled and observable way.

Important failure scenarios include:

- PostgreSQL becomes unavailable
- PostgreSQL operations timeout
- Redis becomes unavailable
- Redis operations timeout
- An application instance becomes unavailable

During failure testing, we verify that:

- The application does not crash because of a dependency failure.
- Requests fail within configured timeouts instead of waiting indefinitely.
- Appropriate HTTP errors are returned.
- Failures are recorded through logs and metrics.
- Health checks reflect the state of the application and its dependencies.
- The application can recover when the dependency becomes available again.

Public redirects should continue to work when Redis is unavailable because redirects in V1 depend on PostgreSQL rather than Redis.
