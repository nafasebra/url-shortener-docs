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

### Workloads

### Scenarios

### Metrics

### Success Criteria

## 6. Failure Testing
