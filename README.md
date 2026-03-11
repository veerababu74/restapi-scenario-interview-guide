# 🚀 Ultimate REST API Scenario-Based Interview Guide

> **FastAPI & Flask | 70 Real-World Scenarios | Visual Diagrams | Working Code**

[![Difficulty](https://img.shields.io/badge/Level-Junior%20to%20Senior-blue)](.)
[![Questions](https://img.shields.io/badge/Questions-70-green)](.)
[![Framework](https://img.shields.io/badge/Framework-FastAPI%20%7C%20Flask-orange)](.)
[![Language](https://img.shields.io/badge/Language-Python-yellow)](.)

---

## 📖 Introduction

This guide was created to help you **crack any REST API interview** — from junior to senior level. After struggling with scenario-based questions in real interviews, this comprehensive resource covers **70 realistic scenarios** that interviewers actually ask.

Each question mirrors what you'd face at companies like:
- 🏢 Product companies (startups to FAANG)
- 💼 Service-based companies
- 🤖 AI/ML product companies

### What Makes This Guide Different?
- ✅ **Real scenarios** — Not textbook definitions
- ✅ **Step-by-step answers** — How to think through problems live
- ✅ **Visual diagrams** — ASCII art + Mermaid flowcharts
- ✅ **Working code** — Copy-paste ready FastAPI & Flask examples
- ✅ **Difficulty levels** — Know what's expected at each level

---

## 🗂️ How to Use This Guide

### For Interview Prep (Recommended Order)
1. Start with **QUICK_REFERENCE.md** to get a bird's-eye view
2. Go through each category file (01 → 07) systematically
3. For each question: **read the scenario → try to answer → check the solution**
4. Practice explaining diagrams out loud
5. Type out (don't copy) the code examples to build muscle memory
6. Review **INTERVIEW_TIPS.md** before your actual interview

### Difficulty Levels
| Badge | Level | Who Should Know |
|-------|-------|-----------------|
| 🟢 | Junior (0-2 years) | Must know |
| 🟡 | Mid-level (2-5 years) | Should know |
| 🔴 | Senior (5+ years) | Expected to know deeply |

### Interview Frequency
| Stars | Meaning |
|-------|---------|
| ★★★ | Very common — asked in most interviews |
| ★★☆ | Common — asked in many interviews |
| ★☆☆ | Rare — asked in senior/specialist roles |

---

## 📚 Table of Contents

### [01 — Performance & Optimization](./01-performance-optimization.md) (Q1–Q10)
> Making APIs fast, efficient, and scalable

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 1 | Handle 10MB+ JSON payloads in POST | 🟡 Mid | ★★★ |
| 2 | AI chatbot takes 10s, reduce to 2s | 🔴 Senior | ★★★ |
| 3 | API drops from 1000 to 100 req/sec | 🔴 Senior | ★★☆ |
| 4 | Endpoint returns 1 million records | 🟡 Mid | ★★★ |
| 5 | Multiple clients call same expensive query | 🟡 Mid | ★★★ |
| 6 | 5 sequential external API calls (5 seconds) | 🟡 Mid | ★★★ |
| 7 | File upload crashes with 500MB+ files | 🟡 Mid | ★★☆ |
| 8 | Database queries getting slower over time | 🟡 Mid | ★★★ |
| 9 | Process CSV with 1 million rows | 🔴 Senior | ★★☆ |
| 10 | API response payload is 5MB JSON | 🟢 Junior | ★★★ |

### [02 — Error Handling & Debugging](./02-error-handling-debugging.md) (Q11–Q20)
> Finding and fixing bugs, handling failures gracefully

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 11 | Intermittent 500 errors you can't reproduce locally | 🟡 Mid | ★★★ |
| 12 | Client reports timeouts but server logs show success | 🟡 Mid | ★★☆ |
| 13 | API sometimes returns partial JSON responses | 🔴 Senior | ★★☆ |
| 14 | New version causes 5% of requests to fail with 422 | 🟡 Mid | ★★★ |
| 15 | API crashes with specific Unicode characters | 🟢 Junior | ★★☆ |
| 16 | "Connection pool exhausted" during peak traffic | 🟡 Mid | ★★★ |
| 17 | API chain A→B→C has cascading failures | 🔴 Senior | ★★★ |
| 18 | Users report data inconsistency / stale data | 🟡 Mid | ★★★ |
| 19 | API logs are 50GB/day — impossible to search | 🟡 Mid | ★★☆ |
| 20 | Memory errors after API runs for hours | 🔴 Senior | ★★☆ |

### [03 — Security](./03-security.md) (Q21–Q30)
> Protecting your API from attacks and vulnerabilities

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 21 | API stores passwords in plain text | 🟢 Junior | ★★★ |
| 22 | Brute force attack on login endpoint | 🟡 Mid | ★★★ |
| 23 | SQL injection vulnerability in entire codebase | 🟡 Mid | ★★★ |
| 24 | JWT tokens being stolen and reused | 🟡 Mid | ★★★ |
| 25 | Serve mobile and web clients with different auth | 🟡 Mid | ★★☆ |
| 26 | Employee accessing other users' data | 🔴 Senior | ★★★ |
| 27 | Malicious script uploaded through file upload | 🟡 Mid | ★★☆ |
| 28 | Public API being scraped by competitors | 🟡 Mid | ★★☆ |
| 29 | Secure service-to-service communication | 🔴 Senior | ★★☆ |
| 30 | CORS misconfiguration allows any website | 🟢 Junior | ★★★ |

### [04 — Architecture & Design](./04-architecture-design.md) (Q31–Q40)
> Designing scalable, maintainable API architectures

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 31 | Monolithic Flask API has 200+ endpoints | 🔴 Senior | ★★☆ |
| 32 | Design API for real-time chat + REST | 🔴 Senior | ★★☆ |
| 33 | Support multiple API versions simultaneously | 🟡 Mid | ★★★ |
| 34 | Design e-commerce checkout REST API | 🔴 Senior | ★★☆ |
| 35 | Notify clients when data changes | 🟡 Mid | ★★★ |
| 36 | Multi-tenant SaaS API data isolation | 🔴 Senior | ★★☆ |
| 37 | API needs both sync and async operations | 🟡 Mid | ★★★ |
| 38 | Search across multiple entities | 🟡 Mid | ★★☆ |
| 39 | Handle file processing without blocking | 🟡 Mid | ★★☆ |
| 40 | Design API gateway for 20 microservices | 🔴 Senior | ★★☆ |

### [05 — Database & Data Management](./05-database-data-management.md) (Q41–Q50)
> Handling data correctly, efficiently, and reliably

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 41 | Two users simultaneously update same resource | 🟡 Mid | ★★★ |
| 42 | Write to database and message queue atomically | 🔴 Senior | ★★☆ |
| 43 | Soft deletes vs GDPR right to be forgotten | 🟡 Mid | ★★☆ |
| 44 | Database migration breaks API in production | 🔴 Senior | ★★★ |
| 45 | Aggregate data from SQL + MongoDB + Redis | 🔴 Senior | ★★☆ |
| 46 | Read-heavy API overwhelming primary database | 🟡 Mid | ★★★ |
| 47 | Full-text search with autocomplete | 🟡 Mid | ★★☆ |
| 48 | Handle high-volume IoT time-series data | 🔴 Senior | ★☆☆ |
| 49 | N+1 query problem causing severe slowdowns | 🟡 Mid | ★★★ |
| 50 | Validate uniqueness against database state | 🟢 Junior | ★★★ |

### [06 — Testing & Deployment](./06-testing-deployment.md) (Q51–Q60)
> Ensuring quality and smooth deployments

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 51 | 200 endpoints, 0 tests — where to start? | 🟡 Mid | ★★★ |
| 52 | Tests pass locally but fail in CI/CD | 🟡 Mid | ★★★ |
| 53 | Test endpoint depending on payment API | 🟡 Mid | ★★☆ |
| 54 | Deployment causes 30 seconds downtime | 🟡 Mid | ★★★ |
| 55 | Manual verification after every deployment | 🟡 Mid | ★★☆ |
| 56 | Configuration for dev/staging/prod environments | 🟢 Junior | ★★★ |
| 57 | Test suite takes 45 minutes | 🟡 Mid | ★★☆ |
| 58 | Test WebSocket endpoints in FastAPI | 🟡 Mid | ★★☆ |
| 59 | Containerize and deploy to Kubernetes | 🔴 Senior | ★★☆ |
| 60 | Implement A/B testing for new API response format | 🔴 Senior | ★☆☆ |

### [07 — Real-World Integration](./07-real-world-integration.md) (Q61–Q70)
> Handling complex real-world integration scenarios

| Q# | Scenario | Difficulty | Frequency |
|----|----------|------------|-----------|
| 61 | Payment webhook arrives out of order | 🔴 Senior | ★★☆ |
| 62 | Generate PDF reports taking 30 seconds | 🟡 Mid | ★★☆ |
| 63 | API needs multi-language content | 🟢 Junior | ★★☆ |
| 64 | Rate limiting with different tiers (free/pro/enterprise) | 🟡 Mid | ★★☆ |
| 65 | Handle idempotent POST requests | 🟡 Mid | ★★★ |
| 66 | API serves mobile app — minimize battery/data | 🟡 Mid | ★★☆ |
| 67 | Support real-time collaborative editing | 🔴 Senior | ★☆☆ |
| 68 | Implement multi-level approval workflow | 🔴 Senior | ★★☆ |
| 69 | Migrate Flask API to FastAPI without downtime | 🔴 Senior | ★★☆ |
| 70 | Graceful degradation when downstream services fail | 🔴 Senior | ★★★ |

---

### [📝 Interview Tips](./INTERVIEW_TIPS.md)
> STAR method, communication strategies, common mistakes

### [⚡ Quick Reference](./QUICK_REFERENCE.md)
> One-line answers to all 70 questions — perfect for last-minute review

---

## 🎯 Quick Start — Top 10 Most Asked Questions

If you have limited time, focus on these (asked in almost every interview):

1. **Q4** — Pagination for large datasets
2. **Q5** — Caching repeated expensive queries
3. **Q6** — Parallel async API calls
4. **Q16** — Database connection pool exhaustion
5. **Q17** — Cascading failures in API chains
6. **Q23** — SQL injection prevention
7. **Q24** — JWT security
8. **Q33** — API versioning
9. **Q49** — N+1 query problem
10. **Q65** — Idempotent POST requests

---

## 🛠️ Tech Stack Covered

```
FastAPI  ▓▓▓▓▓▓▓▓▓▓  Primary
Flask    ▓▓▓▓▓▓▓░░░  Secondary / Alternatives
Redis    ▓▓▓▓▓▓░░░░  Caching & Rate Limiting
Celery   ▓▓▓▓░░░░░░  Background Tasks
SQLAlchemy ▓▓▓▓▓░░░░  ORM / Database
Docker   ▓▓▓░░░░░░░  Containerization
Kubernetes ▓▓░░░░░░░  Orchestration
```

---

## 💡 How to Answer Scenario Questions in Interviews

```
1. CLARIFY   → Ask 1-2 clarifying questions (scale? data size? SLA?)
2. THINK     → Pause 10-15 seconds, think out loud
3. STRUCTURE → "I'd approach this in 3 steps..."
4. EXPLAIN   → Walk through your solution clearly
5. CODE      → Write pseudocode or real code if asked
6. TRADEOFFS → Mention what you'd choose and WHY
```

---

*Happy interviewing! 🎉 Star this repo if it helps you land your dream job.*
