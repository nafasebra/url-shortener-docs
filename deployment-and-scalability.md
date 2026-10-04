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



## 3. PostgreSQL Scaling

## 4. Redis Scaling

## 5. Scaling Strategy

## 6. Deployment Considerations and Process

We use GitHub Actions to automate the deployment process.

When changes are pushed to the deployment branch, a GitHub Actions workflow is triggered to deploy the new version of the application.
