---
name: laravel-best-practices
description: Laravel best practices and architecture patterns for building production-ready applications. Use when writing, reviewing, or refactoring Laravel code to ensure proper patterns for service providers, dependency injection, security, Eloquent, and performance.
license: MIT
metadata:
  author: dev_ai_chat
  version: "1.0.0"
---

# Laravel Best Practices

Comprehensive best practices guide for Laravel applications. Contains 40 rules across 10 categories, prioritized by impact to guide automated refactoring and code generation.

## When to Apply

Reference these guidelines when:

- Writing new Laravel controllers, services, or models
- Implementing authentication and authorization
- Reviewing code for architecture and security issues
- Refactoring existing Laravel codebases
- Optimizing performance or database queries
- Building microservices or queue-based architectures

## Rule Categories by Priority

| Priority | Category | Impact | Prefix |
|----------|----------|--------|--------|
| 1 | Architecture | CRITICAL | `arch-` |
| 2 | Dependency Injection | CRITICAL | `di-` |
| 3 | Error Handling | HIGH | `error-` |
| 4 | Security | HIGH | `security-` |
| 5 | Performance | HIGH | `perf-` |
| 6 | Testing | MEDIUM-HIGH | `test-` |
| 7 | Database & ORM | MEDIUM-HIGH | `db-` |
| 8 | API Design | MEDIUM | `api-` |
| 9 | Microservices | MEDIUM | `micro-` |
| 10 | DevOps & Deployment | LOW-MEDIUM | `devops-` |

## Quick Reference

### 1. Architecture (CRITICAL)

- `arch-avoid-circular-deps` - Avoid circular service provider dependencies
- `arch-feature-modules` - Organize by domain/feature, not technical layer
- `arch-single-responsibility` - Focused services over "god services"
- `arch-use-repository-pattern` - Abstract database logic for testability
- `arch-use-events` - Event-driven architecture for decoupling

### 2. Dependency Injection (CRITICAL)

- `di-avoid-service-locator` - Avoid app() helper as service locator
- `di-interface-segregation` - Interface Segregation Principle (ISP)
- `di-liskov-substitution` - Liskov Substitution Principle (LSP)
- `di-prefer-constructor-injection` - Constructor over facade/helper injection
- `di-scope-awareness` - Understand singleton/bind/scoped bindings
- `di-use-interfaces-tokens` - Bind interfaces to implementations

### 3. Error Handling (HIGH)

- `error-use-exception-handler` - Centralized exception handling
- `error-throw-http-exceptions` - Use proper HTTP exceptions
- `error-handle-queue-errors` - Handle queue and job errors properly

### 4. Security (HIGH)

- `security-auth-jwt` - Secure authentication with Sanctum/Passport
- `security-validate-all-input` - Validate with Form Requests
- `security-use-guards` - Authentication guards, gates, and policies
- `security-sanitize-output` - Prevent XSS attacks
- `security-rate-limiting` - Implement rate limiting

### 5. Performance (HIGH)

- `perf-service-provider-lifecycle` - Proper service provider boot/register lifecycle
- `perf-use-caching` - Implement caching strategies
- `perf-optimize-database` - Optimize database queries
- `perf-lazy-loading` - Lazy collections and route caching

### 6. Testing (MEDIUM-HIGH)

- `test-use-testcase-refresh-database` - Use Laravel TestCase and RefreshDatabase
- `test-e2e-http` - HTTP/feature testing with Laravel
- `test-mock-external-services` - Mock external dependencies with facades

### 7. Database & ORM (MEDIUM-HIGH)

- `db-use-transactions` - Transaction management
- `db-avoid-n-plus-one` - Avoid N+1 query problems with eager loading
- `db-use-migrations` - Use migrations for schema changes

### 8. API Design (MEDIUM)

- `api-use-dto-serialization` - API Resources and serialization
- `api-use-interceptors` - Middleware for cross-cutting concerns
- `api-versioning` - API versioning strategies
- `api-use-pipes` - Form Requests for input transformation

### 9. Microservices (MEDIUM)

- `micro-use-patterns` - Message and event patterns
- `micro-use-health-checks` - Health checks for orchestration
- `micro-use-queues` - Background job processing with queues

### 10. DevOps & Deployment (LOW-MEDIUM)

- `devops-use-config-module` - Environment configuration with config()
- `devops-use-logging` - Structured logging with channels
- `devops-graceful-shutdown` - Zero-downtime deployments

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/arch-avoid-circular-deps.md
rules/security-validate-all-input.md
```

Each rule file contains:
- Brief explanation of why it matters
- Incorrect code example with explanation
- Correct code example with explanation
- Additional context and references

> Only read `AGENTS.md` or individual `rules/*.md` files when you need 
> full implementation detail for a specific rule. Do not load them proactively.

## Full Compiled Document

For the complete guide with all rules expanded: `AGENTS.md`

## Laravel Boost Compatibility

If Laravel Boost is installed, the following rules are already covered 
by its core guidelines and can be skipped:

`security-validate-all-input`, `security-auth-jwt`, `db-avoid-n-plus-one`,
`db-use-migrations`, `perf-use-caching`, `test-use-testcase-refresh-database`,
`devops-use-logging`, `api-versioning`