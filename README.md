# Serverless REST API with Cognito Auth, DynamoDB & WAF

## Overview

This project demonstrates a fully serverless REST API architecture on AWS, built for a to-do / customer records application. The solution uses API Gateway, Lambda, and DynamoDB as the core compute and data layer, with Amazon Cognito for authentication, WAF for protection, X-Ray for distributed tracing, and CloudFront for global edge delivery.

---

## 🏗️ Project Architecture

![Architecture Diagram](manara-digram.png)

---

## Architecture Components

### 1. Users
End users access the application over HTTPS. Route 53 resolves the domain name to the CloudFront distribution.

### 2. Route 53
- AWS managed DNS service
- Routes user requests to the CloudFront distribution via Alias record
- Supports health checks for high availability

### 3. CloudFront
- CDN distribution sitting in front of both the API Gateway and the S3-hosted frontend
- Caches static assets at 400+ global edge locations to reduce latency
- Enforces HTTPS and forwards dynamic API requests to API Gateway

### 4. WAF (Web Application Firewall)
- Attached to CloudFront distribution
- Enforces OWASP Top 10 managed rule set (SQLi, XSS, etc.)
- Rate-based rules to block abusive IPs
- Geo-blocking support for restricted regions

### 5. S3 (Static Frontend Hosting)
- Hosts the static React/Vue frontend build artifacts
- Configured as a static website with bucket policy restricting access to CloudFront only (Origin Access Control)
- CloudFront serves the frontend files from S3 with HTTPS

### 6. API Gateway (REST API)
- REST API with resource and method definitions for CRUD operations
- Cognito JWT authorizer validates every incoming request before invoking Lambda
- Response caching enabled per endpoint to reduce Lambda invocations and cut cost
- Usage plans and API keys for rate limiting per consumer
- Integrated with CloudFront for global edge caching

### 7. Amazon Cognito (User Pool)
- Handles user sign-up, sign-in, and MFA
- Issues JWT access tokens and ID tokens upon successful authentication
- API Gateway validates the JWT token on every request using the Cognito authorizer
- Hosted UI available for quick frontend integration

### 8. AWS Lambda (CRUD Handlers)
- Separate Lambda functions for each CRUD operation (Create, Read, Update, Delete)
- Stateless, auto-scales to zero when idle
- Each function assigned its own IAM execution role following least-privilege principles
- Environment variables used for configuration (table names, region, etc.)
- Integrated with X-Ray for distributed tracing

### 9. AWS X-Ray
- End-to-end distributed tracing across API Gateway → Lambda → DynamoDB
- Service map visualization to identify latency bottlenecks
- Trace sampling and annotation for debugging production issues

### 10. DynamoDB
- NoSQL table with a well-designed primary key and GSIs (Global Secondary Indexes) for flexible access patterns
- On-demand capacity mode — no provisioning required, scales automatically
- Point-in-time recovery (PITR) enabled for data protection
- DynamoDB Streams enabled to capture item-level changes

### 11. DynamoDB Streams
- Captures every write event (INSERT, MODIFY, REMOVE)
- Triggers downstream Lambda functions for async processing (e.g., notifications, cache invalidation, audit logs)

### 12. CloudWatch
- Monitors Lambda invocation count, error rate, duration, and throttles
- Monitors API Gateway latency, 4xx/5xx error rates
- Monitors DynamoDB read/write capacity and throttling
- Alarms configured to notify via SNS on threshold breaches

### 13. SNS (Simple Notification Service)
- Receives CloudWatch alarm notifications
- Sends email/SMS alerts to administrators on errors or scaling events

### 14. IAM (Identity and Access Management)
- Each Lambda function has its own IAM execution role
- Least-privilege: only access to its specific DynamoDB table and CloudWatch Logs
- No wildcard permissions (`*`) used in any policy

### 15. KMS (Key Management Service)
- Customer-managed keys (CMKs) used for encrypting DynamoDB table at rest
- Encryption of S3 bucket objects
- Full key rotation enabled

---

## Traffic Flow

```
Users
  │
  ▼
Route 53 (DNS resolution)
  │
  ▼
CloudFront (CDN + HTTPS enforcement)
  ├──► S3 (static frontend assets)
  └──► WAF (filter malicious traffic)
         │
         ▼
      API Gateway (REST API + Cognito JWT authorizer + caching)
         ├──► Cognito (JWT token validation)
         └──► Lambda (CRUD handler invocation)
                ├──► X-Ray (distributed trace)
                └──► DynamoDB (read / write)
                       └──► DynamoDB Streams (async event triggers)

Lambda / API Gateway ──► CloudWatch ──► SNS ──► Admin notifications
```

---

## Security Considerations

| Layer | Control |
|---|---|
| Edge | CloudFront enforces HTTPS; WAF blocks OWASP Top 10 |
| Authentication | Cognito User Pool with JWT tokens |
| Authorization | API Gateway Cognito authorizer on every endpoint |
| Compute | Lambda IAM roles with least-privilege policies |
| Data | DynamoDB encrypted at rest with KMS CMK |
| Secrets | No hardcoded credentials; environment variables via Lambda config |
| Audit | CloudTrail logs all API calls across the account |

---

## Scalability and High Availability

- **CloudFront** serves content from edge locations globally — no single point of failure at the CDN layer
- **API Gateway** is a fully managed, multi-AZ service with built-in auto scaling
- **Lambda** scales automatically per request with no infrastructure management
- **DynamoDB** on-demand mode scales read/write capacity instantly with no throttling under normal load
- **Cognito** is a fully managed, highly available identity service

---

## Learning Outcomes

- Implement token-based authentication with Cognito and API Gateway JWT authorizers
- Design DynamoDB table schemas and GSIs for efficient access patterns
- Apply WAF rules to protect APIs from abuse and injection attacks
- Use X-Ray to trace and debug latency across a serverless call chain
- Enable API Gateway caching to reduce Lambda invocations and cut cost
- Understand DynamoDB Streams for triggering downstream event processing

---

## Key AWS Services Summary

| Service | Purpose |
|---|---|
| Route 53 | DNS resolution + health checks |
| CloudFront | CDN + edge caching + HTTPS |
| WAF | OWASP protection + rate limiting |
| S3 | Static frontend hosting |
| API Gateway | REST API + JWT auth + caching |
| Cognito | User Pool + JWT token issuing |
| Lambda | Serverless CRUD handlers |
| X-Ray | Distributed tracing |
| DynamoDB | NoSQL data store with GSIs |
| DynamoDB Streams | Event-driven async triggers |
| CloudWatch | Monitoring + alarms |
| SNS | Alert notifications |
| IAM | Least-privilege roles |
| KMS | Encryption at rest |

---

## Conceptual Setup Steps

1. **Route 53** — Create a hosted zone and Alias record pointing to CloudFront distribution
2. **S3** — Create bucket, enable static website hosting, upload frontend build, apply bucket policy for CloudFront OAC
3. **Cognito** — Create User Pool, configure app client, enable hosted UI, note User Pool ID and App Client ID
4. **DynamoDB** — Create table with partition key and sort key, define GSIs, enable Streams and PITR, attach KMS CMK
5. **Lambda** — Create CRUD functions, attach IAM execution roles, set environment variables (table name, region), enable X-Ray active tracing
6. **API Gateway** — Create REST API, define resources and methods, attach Cognito authorizer, integrate with Lambda functions, enable response caching, deploy to stage
7. **WAF** — Create Web ACL, attach OWASP managed rule group, add rate-based rule, associate with CloudFront distribution
8. **CloudFront** — Create distribution, set S3 as origin for frontend and API Gateway as origin for API paths, attach WAF Web ACL, enforce HTTPS
9. **CloudWatch** — Create dashboards and alarms for Lambda errors, API Gateway 5xx, DynamoDB throttling
10. **SNS** — Create topic, subscribe admin email, attach to CloudWatch alarms

---

*Author: Mohammed Hamdy*
*Program: Manara — AWS Solutions Architect Associate*
