# ⚡ Quick Reference — All 70 Questions

> One-line answers to all 70 questions. Perfect for last-minute review.

---

## 🏎️ Performance & Optimization (Q1–Q10)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 1 | Handle 10MB+ POST payloads | Stream body in chunks + return job_id immediately, process async | 🟡 | ★★★ |
| 2 | AI chatbot takes 10s (should be 2s) | Stream response with SSE + Redis cache for common questions | 🔴 | ★★★ |
| 3 | API drops 1000 → 100 req/sec suddenly | Profile: CPU/memory → DB pool exhaustion → external deps → memory leak | 🔴 | ★★☆ |
| 4 | Endpoint returns 1 million records | Cursor-based pagination (not offset) + field filtering + async export | 🟡 | ★★★ |
| 5 | Multiple clients hit same expensive query | Redis cache-aside pattern + request coalescing to prevent stampede | 🟡 | ★★★ |
| 6 | 5 sequential external API calls = 5 seconds | `asyncio.gather()` to parallelize + circuit breaker per service | 🟡 | ★★★ |
| 7 | File upload crashes with 500MB files | Stream to S3 via presigned URL (server never touches bytes) | 🟡 | ★★☆ |
| 8 | DB queries getting slower as data grows | EXPLAIN ANALYZE → add indexes → read replicas → query caching | 🟡 | ★★★ |
| 9 | Process CSV with 1 million rows | Return job_id immediately, stream+chunk CSV in Celery background task | 🔴 | ★★☆ |
| 10 | API response payload is 5MB JSON | GZipMiddleware + field filtering + pagination + ETags | 🟢 | ★★★ |

---

## 🐛 Error Handling & Debugging (Q11–Q20)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 11 | Intermittent 500s can't reproduce locally | Add correlation IDs + structured JSON logging + Sentry for production | 🟡 | ★★★ |
| 12 | Client timeouts but server logs show success | Proxy/LB timeout < response time; check nginx timeout configs | 🟡 | ★★☆ |
| 13 | Sometimes returns partial JSON | Use Pydantic response_model to validate before sending + check nginx timeouts | 🔴 | ★★☆ |
| 14 | New version causes 5% requests → 422 | Accept both old and new field names; log all 422s to find breaking change | 🟡 | ★★★ |
| 15 | API crashes with Unicode/emoji characters | Strip null bytes + normalize unicode + validate byte length not char length | 🟢 | ★★☆ |
| 16 | Connection pool exhausted during peak | pool_size = max_db_connections/instances - buffer; use PgBouncer | 🟡 | ★★★ |
| 17 | Cascading failures in API chain A→B→C | Circuit breaker per downstream service + fallback responses + timeouts | 🔴 | ★★★ |
| 18 | Users see stale/inconsistent data | Invalidate cache on write + read from primary after writes + use ETags | 🟡 | ★★★ |
| 19 | 50GB/day of API logs | Structured JSON logging + sample 10% of INFO + ELK stack + log rotation | 🟡 | ★★☆ |
| 20 | Memory errors after hours of running | tracemalloc to find leak + fix global list growth + use `deque(maxlen=N)` | 🔴 | ★★☆ |

---

## 🔒 Security (Q21–Q30)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 21 | API stores passwords in plain text | Hash with bcrypt (cost=12) + gradual migration: hash on next login | 🟢 | ★★★ |
| 22 | Brute force attack on login | IP rate limit + per-account limit + progressive delays + temporary lockout | 🟡 | ★★★ |
| 23 | SQL injection throughout codebase | Use parameterized queries/ORM everywhere; scan with `bandit -r .` | 🟡 | ★★★ |
| 24 | JWT tokens being stolen and reused | 15-min access tokens + refresh token rotation + JTI blacklist in Redis | 🟡 | ★★★ |
| 25 | Different auth for mobile and web | Authorization Code (web) + PKCE (mobile) + API keys (B2B) + multi-auth middleware | 🟡 | ★★☆ |
| 26 | Employee accessing other users' data | Check object ownership on every endpoint (BOLA prevention) + audit logging | 🔴 | ★★★ |
| 27 | Malicious script via file upload | Validate magic bytes (not extension) + store outside web root + serve via API | 🟡 | ★★☆ |
| 28 | Public API being scraped | API keys + request fingerprinting + honeypot endpoints + tiered rate limits | 🟡 | ★★☆ |
| 29 | Secure service-to-service communication | mTLS (Istio/Linkerd) or short-lived service JWTs with iss/aud claims | 🔴 | ★★☆ |
| 30 | CORS misconfiguration allows any origin | Whitelist specific origins from env vars; never combine `*` with credentials | 🟢 | ★★★ |

---

## 🏗️ Architecture & Design (Q31–Q40)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 31 | 200+ endpoints in one file | Domain-driven modules: each domain gets its own router + service + schema | 🔴 | ★★☆ |
| 32 | Design API for real-time chat + REST | REST for CRUD/history + WebSocket for real-time delivery + Redis Pub/Sub | 🔴 | ★★☆ |
| 33 | Support multiple API versions at once | URL versioning (/v1/, /v2/) + shared service layer + Deprecation headers | 🟡 | ★★★ |
| 34 | E-commerce checkout API design | Saga pattern + state machine (cart→address→payment→confirmed) + idempotency | 🔴 | ★★☆ |
| 35 | Notify clients when data changes | SSE for dashboards + WebSocket for chat + webhooks for B2B + Redis Pub/Sub | 🟡 | ★★★ |
| 36 | Multi-tenant SaaS data isolation | Row-level security (PostgreSQL) + tenant_id on every table + tenant middleware | 🔴 | ★★☆ |
| 37 | API needs both sync and async ops | 202 Accepted + job_id + polling endpoint + Celery for long-running tasks | 🟡 | ★★★ |
| 38 | Search across multiple entities | Elasticsearch + parallel search with asyncio.gather + faceted results | 🟡 | ★★☆ |
| 39 | File processing without blocking | Presigned S3 URL upload → Celery worker → CDN for processed results | 🟡 | ★★☆ |
| 40 | API gateway for 20 microservices | Single entry point + auth at gateway + circuit breaker per service + BFF | 🔴 | ★★☆ |

---

## 🗄️ Database & Data Management (Q41–Q50)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 41 | Two users update same resource simultaneously | Optimistic locking (version field) → 409 Conflict if mismatch; pessimistic for finance | 🟡 | ★★★ |
| 42 | Write DB and message queue atomically | Outbox pattern: write to DB + outbox table in same transaction, poller publishes | 🔴 | ★★☆ |
| 43 | Soft deletes vs GDPR deletion | Anonymize PII fields (don't delete records) + purge actual PII columns | 🟡 | ★★☆ |
| 44 | Zero-downtime DB migration | Expand-Contract: add column → backfill in batches → switch reads → drop old | 🔴 | ★★★ |
| 45 | Aggregate SQL + MongoDB + Redis data | Repository pattern per data source + asyncio.gather for parallel fetching | 🔴 | ★★☆ |
| 46 | Read-heavy API overwhelming primary DB | Read replicas for reads + Redis cache + materialized views | 🟡 | ★★★ |
| 47 | Full-text search with autocomplete | Elasticsearch with fuzzy matching + completion suggester + CDC sync | 🟡 | ★★☆ |
| 48 | High-volume IoT time-series data | TimescaleDB + Kafka buffer + batch inserts + continuous aggregates + retention | 🔴 | ★☆☆ |
| 49 | N+1 query problem (101 queries for 100 items) | SQLAlchemy `selectinload()` = 2 queries; or JOIN = 1 query | 🟡 | ★★★ |
| 50 | Validate uniqueness against DB state | DB UNIQUE constraint (mandatory) + catch IntegrityError → return 409 | 🟢 | ★★★ |

---

## 🧪 Testing & Deployment (Q51–Q60)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 51 | 200 endpoints, 0 tests | Start with critical paths (auth/payments) + testing pyramid + in-memory DB | 🟡 | ★★★ |
| 52 | Tests pass locally but fail in CI | Pin dependencies + Docker Compose for CI environment + mock time/env vars | 🟡 | ★★★ |
| 53 | Test endpoint calling payment API | pytest-httpx to mock HTTP calls + test all scenarios (success/decline/timeout) | 🟡 | ★★☆ |
| 54 | 30 seconds downtime on every deploy | Blue-green deployment + readiness probes + graceful shutdown + preStop hook | 🟡 | ★★★ |
| 55 | Manual verification after deployment | Automated smoke tests + auto-rollback on failure + health check endpoint | 🟡 | ★★☆ |
| 56 | Config management for dev/staging/prod | Pydantic Settings + environment variables + CI/CD secrets + .env.example | 🟢 | ★★★ |
| 57 | Test suite takes 45 minutes | pytest-xdist parallel execution + mark slow tests + session-scoped fixtures | 🟡 | ★★☆ |
| 58 | Test WebSocket endpoints | FastAPI TestClient.websocket_connect() + test full lifecycle | 🟡 | ★★☆ |
| 59 | Containerize and deploy to Kubernetes | Multi-stage Dockerfile + non-root user + resource limits + HPA + probes | 🔴 | ★★☆ |
| 60 | A/B test new API response format | Feature flags + deterministic hash for consistent assignment + track metrics | 🔴 | ★☆☆ |

---

## 🌐 Real-World Integration (Q61–Q70)

| Q# | Scenario | One-Line Answer | Diff | Freq |
|----|----------|-----------------|------|------|
| 61 | Payment webhook arrives out of order | Verify HMAC → store raw event first → acknowledge 200 → process async with retry | 🔴 | ★★☆ |
| 62 | PDF reports taking 30 seconds | 202 Accepted + job_id + polling endpoint + S3 storage + pre-signed download URL | 🟡 | ★★☆ |
| 63 | Multi-language API content | Accept-Language header negotiation + translations in DB + Content-Language response | 🟢 | ★★☆ |
| 64 | Rate limiting with free/pro/enterprise tiers | Token bucket in Redis per tier + X-RateLimit headers + 429 with upgrade URL | 🟡 | ★★☆ |
| 65 | Idempotent POST requests (double payment) | Client sends Idempotency-Key header + server caches result for 24h by key | 🟡 | ★★★ |
| 66 | Optimize API for mobile clients | Gzip + field filtering + ETags + batch endpoint + delta updates | 🟡 | ★★☆ |
| 67 | Real-time collaborative editing | OT or CRDT for conflict resolution + WebSocket + use Yjs library in production | 🔴 | ★☆☆ |
| 68 | Multi-level approval workflow | State machine with valid transitions + audit history + role-based approval | 🔴 | ★★☆ |
| 69 | Migrate Flask to FastAPI without downtime | Strangler Fig: proxy unknown endpoints to Flask while migrating domain by domain | 🔴 | ★★☆ |
| 70 | Graceful degradation when services fail | Classify critical vs optional services + fallbacks + circuit breakers + degrade gracefully | 🔴 | ★★★ |

---

## 🔑 Most Important Concepts to Remember

### Top 10 Patterns Every Interview Tests

1. **Async/await + asyncio.gather()** — Parallel operations, not sequential
2. **Circuit breaker** — Fail fast, use fallbacks
3. **Cache-aside pattern** — Check cache, miss → DB → store in cache
4. **Idempotency keys** — Safe retries for POST operations
5. **Cursor-based pagination** — Scalable alternative to OFFSET
6. **Outbox pattern** — Atomic DB write + event publishing
7. **Optimistic locking** — Version field, 409 on conflict
8. **JWT refresh rotation** — Short-lived access + rotating refresh tokens
9. **Expand-Contract migration** — Zero-downtime DB schema changes
10. **Strangler Fig** — Incremental migration without big bang rewrite

---

## 💻 Code Patterns to Memorize

### Parallel API Calls
```python
results = await asyncio.gather(
    call_service_a(), call_service_b(), call_service_c(),
    return_exceptions=True
)
```

### Cache-Aside
```python
cached = await redis.get(key)
if cached:
    return json.loads(cached)
result = await db.query(...)
await redis.setex(key, 300, json.dumps(result))
return result
```

### Circuit Breaker State Check
```python
if not circuit_breaker.can_execute():
    return fallback_response()
try:
    result = await call_service()
    circuit_breaker.record_success()
    return result
except Exception:
    circuit_breaker.record_failure()
    raise
```

### Idempotency
```python
cached = await redis.get(f"idempotency:{key}")
if cached:
    return json.loads(cached)  # Return same response
result = await process()
await redis.setex(f"idempotency:{key}", 86400, json.dumps(result))
return result
```

### Optimistic Lock Update
```python
result = await db.execute(
    "UPDATE table SET data=:data, version=:new_v WHERE id=:id AND version=:old_v",
    {"data": data, "new_v": old_version+1, "id": id, "old_v": old_version}
)
if result.rowcount == 0:
    raise HTTPException(409, "Concurrent modification detected")
```

---

## 📊 Technology Quick Reference

| Problem | Technology | Why |
|---------|-----------|-----|
| Caching | Redis | In-memory, fast, TTL support, pub/sub |
| Background tasks | Celery + Redis | Reliable queuing, retry support |
| Search | Elasticsearch | Full-text, fuzzy, facets, aggregations |
| File storage | S3 | Scalable, cheap, presigned URLs |
| Message queue | Kafka / RabbitMQ | Durability, ordering, replay |
| Metrics | Prometheus + Grafana | Industry standard |
| Error tracking | Sentry | Error grouping, alerts |
| Service mesh | Istio / Linkerd | mTLS, circuit breaking, observability |
| Load balancing | Nginx / AWS ALB | High performance, health checks |
| Secrets | HashiCorp Vault / AWS Secrets Manager | Rotation, audit logging |

---

## 🎓 Difficulty Reference

| 🟢 Junior Level | 🟡 Mid Level | 🔴 Senior Level |
|----------------|-------------|-----------------|
| Password hashing | Connection pool | Circuit breakers |
| CORS config | Cursor pagination | Outbox pattern |
| Response compression | Optimistic locking | Strangler fig |
| HTTP status codes | JWT security | CRDT/OT |
| Environment variables | Rate limiting | Zero-downtime migration |
| Basic caching | N+1 queries | Multi-tenant architecture |

---

*✅ You've reviewed all 70 questions! Now go ace that interview!*

*📚 For detailed answers and code examples, see the individual category files:*
- *[01 - Performance](./01-performance-optimization.md)*
- *[02 - Error Handling](./02-error-handling-debugging.md)*
- *[03 - Security](./03-security.md)*
- *[04 - Architecture](./04-architecture-design.md)*
- *[05 - Database](./05-database-data-management.md)*
- *[06 - Testing](./06-testing-deployment.md)*
- *[07 - Integration](./07-real-world-integration.md)*
