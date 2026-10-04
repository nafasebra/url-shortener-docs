# URL Shortener - Deployment & Scalability

This document describes how the URL Shortener system is deployed
and how the architecture can scale as traffic increases.

## 1. Deployment Architecture

The V1 application is containerized using Docker.

Docker Compose is used to run the main components of the system:

- API Server
- PostgreSQL
- Redis

The API Server connects to PostgreSQL for persistent data storage and Redis for session management and rate limiting.

This provides a simple deployment architecture for the first version of the system.

## 2. Application Scaling

The application initially runs with a single API Server instance.

If monitoring and load testing show that the API Server becomes a bottleneck, we can scale the application horizontally by running multiple API Server instances.

A load balancer distributes incoming requests between the available instances.

```text
                  Load Balancer
                 /      |      \
                ↓       ↓       ↓
              API 1   API 2   API 3
                 \      |      /
                  \     |     /
                PostgreSQL
                    |
                  Redis
```

The API Server is designed to be stateless. Session data is stored in Redis instead of the memory of an individual API instance.

This allows any API instance to handle an authenticated request and makes horizontal scaling easier.

Vertical scaling can also be used by increasing the CPU or memory of an API Server, but horizontal scaling provides a way to increase capacity by adding more application instances.

Application scaling should be introduced when monitoring or load-testing results show that the API layer is a bottleneck.

## 3. PostgreSQL Scaling

PostgreSQL is the primary persistent data store for the system.

As traffic increases, database performance should be monitored to identify bottlenecks such as high query latency, connection pool saturation, or resource usage.

The first step is to optimize queries, indexes, and database connections before introducing additional infrastructure.

Since redirect traffic is expected to be significantly higher than URL creation traffic, read operations may become a bottleneck before writes.

If redirect lookups create significant load on PostgreSQL, frequently accessed URL mappings can be cached in Redis to reduce database reads.

If database read traffic continues to become a bottleneck, read replicas can be considered to distribute read operations.

More complex approaches such as database partitioning or sharding should only be introduced if the system reaches a scale where simpler approaches are no longer sufficient.

## 4. Redis Scaling

Redis is used for temporary and shared application state, including sessions and rate-limiting data.

A single Redis instance is sufficient for the initial version of the system.

Redis performance and memory usage should be monitored as traffic increases.

If Redis availability becomes important, replication and automatic failover can be introduced to reduce the impact of an instance failure.

If a single Redis instance can no longer handle the required throughput or memory usage, Redis Cluster can be considered to distribute data and traffic across multiple nodes.

Redirect caching may also increase Redis usage in the future, but it should only be introduced if load testing shows that PostgreSQL redirect lookups are a bottleneck.

## 5. Scaling Strategy

The system follows an incremental scaling strategy.

We do not introduce additional infrastructure before there is evidence that the current architecture cannot handle the required workload.

Monitoring and load testing are used to identify bottlenecks before making scaling decisions.

Depending on the bottleneck:

- API Server bottleneck → horizontally scale API instances behind a load balancer.
- PostgreSQL redirect-read bottleneck → introduce Redis caching for URL mappings.
- PostgreSQL read bottleneck → consider read replicas.
- Redis capacity or throughput bottleneck → consider Redis replication or clustering.
- Database capacity becomes insufficient at significantly larger scale → evaluate partitioning or sharding.

After each scaling change, the system should be measured again to determine whether the bottleneck has been resolved.

## 6. Deployment Considerations and Process

We use GitHub Actions to automate the deployment process.

When changes are pushed to the deployment branch, a GitHub Actions workflow is triggered to deploy the new version of the application.
