# 01 — Performance & Optimization

> **Questions 1–10** | Making APIs fast, efficient, and scalable

---

## Question 1 — Handle Large POST Payloads (10MB+)
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your POST endpoint receives 10MB+ JSON payloads and times out. Clients report request failures. How do you handle large data in POST requests?"*

### The Answer

**Step 1: Diagnose the problem**
- Default server limits (nginx: 1MB, uvicorn: no limit but memory issues)
- JSON parsing blocks the event loop for large payloads
- Single request can exhaust memory

**Step 2: Choose the right strategy based on data type**

| Data Type | Best Strategy |
|-----------|---------------|
| JSON records | Pagination / batch endpoints |
| Files | Multipart upload or pre-signed URLs |
| Streaming data | Chunked transfer encoding |
| Binary data | gzip compression + streaming |

**Step 3: Implement the solution**

```
┌─────────────────────────────────────────────────────────────┐
│              LARGE PAYLOAD HANDLING STRATEGIES              │
├─────────────────────────┬───────────────────────────────────┤
│   SINGLE LARGE UPLOAD   │      CHUNKED UPLOAD               │
│                         │                                   │
│  Client ──10MB──► API   │  Client ──2MB──► API (chunk 1)   │
│         (timeout!)      │  Client ──2MB──► API (chunk 2)   │
│                         │  Client ──2MB──► API (chunk 3)   │
│                         │  Client ──2MB──► API (chunk 4)   │
│                         │  Client ──2MB──► API (chunk 5)   │
│                         │         (no timeout!)             │
└─────────────────────────┴───────────────────────────────────┘
```

```mermaid
flowchart TD
    A[Client sends large payload] --> B{What type of data?}
    B -->|JSON records| C[Split into batches\n max 1000 records/request]
    B -->|File upload| D[Multipart upload\nor pre-signed S3 URL]
    B -->|Streaming data| E[Chunked transfer\nwith streaming endpoint]
    C --> F[Process each batch\nasync with job ID]
    D --> G[Upload directly to S3\nAPI only gets URL]
    E --> H[Stream process\nwithout buffering full body]
    F --> I[Return job_id\nClient polls /jobs/status]
    G --> J[Trigger background job\nfor processing]
    H --> K[Acknowledge receipt\nProcess in background]
```

### Code Example — FastAPI Streaming Upload + Background Task

```python
import asyncio
from fastapi import FastAPI, Request, BackgroundTasks, HTTPException
from fastapi.responses import JSONResponse
import uuid
import json

app = FastAPI()

# In-memory job store (use Redis in production)
job_store = {}

async def process_large_data(job_id: str, data: list):
    """Background task to process large data"""
    job_store[job_id] = {"status": "processing", "progress": 0}
    
    total = len(data)
    for i, record in enumerate(data):
        # Simulate processing each record
        await asyncio.sleep(0.001)
        if i % 100 == 0:
            job_store[job_id]["progress"] = int((i / total) * 100)
    
    job_store[job_id] = {"status": "completed", "progress": 100, "processed": total}

@app.post("/upload/large-json")
async def upload_large_json(request: Request, background_tasks: BackgroundTasks):
    """
    Handle large JSON payloads by:
    1. Streaming the body (no buffering)
    2. Parsing in chunks
    3. Processing in background
    """
    content_length = request.headers.get("content-length")
    if content_length and int(content_length) > 50 * 1024 * 1024:  # 50MB limit
        raise HTTPException(413, "Payload too large. Max 50MB.")
    
    # Stream the body instead of loading all at once
    body_bytes = b""
    async for chunk in request.stream():
        body_bytes += chunk
        if len(body_bytes) > 50 * 1024 * 1024:
            raise HTTPException(413, "Payload exceeded 50MB limit")
    
    try:
        data = json.loads(body_bytes)
    except json.JSONDecodeError:
        raise HTTPException(400, "Invalid JSON")
    
    if not isinstance(data, list):
        raise HTTPException(400, "Expected a JSON array")
    
    # Generate job ID and process in background
    job_id = str(uuid.uuid4())
    job_store[job_id] = {"status": "queued", "progress": 0}
    background_tasks.add_task(process_large_data, job_id, data)
    
    return {
        "job_id": job_id,
        "status": "queued",
        "records_received": len(data),
        "poll_url": f"/jobs/{job_id}"
    }

@app.get("/jobs/{job_id}")
async def get_job_status(job_id: str):
    """Poll endpoint to check processing status"""
    if job_id not in job_store:
        raise HTTPException(404, "Job not found")
    return job_store[job_id]

# Alternative: Chunked batch endpoint
@app.post("/upload/batch")
async def upload_batch(
    records: list[dict],
    batch_number: int = 1,
    total_batches: int = 1,
    upload_session: str = None
):
    """
    Clients split large data into batches and send separately.
    Each batch: max 1000 records
    """
    if len(records) > 1000:
        raise HTTPException(400, "Max 1000 records per batch")
    
    session_id = upload_session or str(uuid.uuid4())
    
    # Store batch and check if all batches received
    # In production, use Redis to aggregate batches
    return {
        "session_id": session_id,
        "batch_received": batch_number,
        "total_batches": total_batches,
        "records_in_batch": len(records),
        "all_batches_received": batch_number == total_batches
    }
```

### Flask Alternative

```python
from flask import Flask, request, jsonify
import threading
import uuid

app = Flask(__name__)
job_store = {}

def process_in_background(job_id, data):
    job_store[job_id] = {"status": "processing"}
    # Process data...
    job_store[job_id] = {"status": "completed"}

@app.route("/upload/large-json", methods=["POST"])
def upload_large_json():
    # Flask streams request by default when using request.stream
    data = request.get_json(force=True)
    
    job_id = str(uuid.uuid4())
    thread = threading.Thread(target=process_in_background, args=(job_id, data))
    thread.start()
    
    return jsonify({"job_id": job_id, "status": "queued"})

# Set max content length in config
app.config['MAX_CONTENT_LENGTH'] = 50 * 1024 * 1024  # 50MB
```

### Key Takeaways
> - 💡 **Never load the entire body into memory** — stream it
> - 💡 **Return a job_id immediately** — process asynchronously
> - 💡 **Set explicit size limits** to prevent abuse
> - 💡 **For files**: use pre-signed S3 URLs so the file bypasses your API entirely
> - 💡 **For JSON arrays**: implement batch endpoints (max N records per request)

---

## Question 2 — Debug Slow AI Chatbot Response (10s → 2s)
🔴 Senior | ★★★ Very Common

### The Scenario
> *"Your AI chatbot API response time is 10 seconds but should be 2 seconds. Users are dropping off. How do you debug and fix this?"*

### The Answer

**Step 1: Profile to find the bottleneck**

```
DEBUGGING TIMELINE (find where time is spent):

Request  ──────────────────────────────────────► Response
         │         │           │          │
         │         │           │          │
      0ms │      50ms        500ms      10000ms
         │         │           │          │
    Input │   Auth  │     DB    │     LLM  │
  Parsing │  Check  │  Queries  │    Call  │
   (fast) │  (fast) │   (slow?) │  (SLOW!) │
```

**Step 2: Common causes and fixes**

| Cause | Fix | Time Saved |
|-------|-----|-----------|
| LLM API call (synchronous) | Stream response with SSE | 8s perceived |
| No caching of common queries | Redis cache | 2-5s |
| Database queries per request | Connection pool + query opt | 1-2s |
| No async | async/await + asyncio.gather | 1-3s |
| Response serialization | Optimize pydantic models | 0.1-0.5s |

```mermaid
flowchart LR
    A[Request arrives] --> B[Profile with middleware]
    B --> C{Where is time spent?}
    C -->|LLM API call| D[Implement SSE streaming\nUser sees response immediately]
    C -->|Database| E[Add Redis cache\nConnection pooling]
    C -->|Sequential operations| F[Use asyncio.gather\nParallelize independent calls]
    C -->|Serialization| G[Optimize Pydantic models\nUse response_model_exclude_unset]
    D --> H[Perceived latency: 0.3s]
    E --> I[Actual latency: -2s]
    F --> J[Actual latency: -3s]
```

### Code Example — FastAPI SSE Streaming + Redis Caching

```python
import asyncio
import hashlib
import json
import time
from typing import AsyncGenerator

import redis.asyncio as redis
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI  # or any LLM client

app = FastAPI()
redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)
openai_client = AsyncOpenAI()

# Middleware to time every request
@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    response.headers["X-Process-Time"] = str(duration)
    if duration > 2.0:
        print(f"SLOW REQUEST: {request.url.path} took {duration:.2f}s")
    return response

def get_cache_key(user_message: str) -> str:
    """Create a cache key from the user message"""
    return f"chat:{hashlib.md5(user_message.encode()).hexdigest()}"

async def stream_llm_response(messages: list) -> AsyncGenerator[str, None]:
    """Stream LLM response using Server-Sent Events"""
    async with openai_client.chat.completions.stream(
        model="gpt-4",
        messages=messages,
        max_tokens=500
    ) as stream:
        async for text in stream.text_stream:
            # SSE format: "data: {content}\n\n"
            yield f"data: {json.dumps({'content': text})}\n\n"
    yield "data: [DONE]\n\n"

@app.post("/chat/stream")
async def chat_stream(request: Request):
    """
    Streaming chat endpoint - user sees response immediately
    instead of waiting 10 seconds for full response
    """
    body = await request.json()
    user_message = body.get("message", "")
    
    # Check cache for common questions
    cache_key = get_cache_key(user_message)
    cached = await redis_client.get(cache_key)
    
    if cached:
        # Return cached response instantly
        async def cached_stream():
            data = json.loads(cached)
            for chunk in data["chunks"]:
                yield f"data: {json.dumps({'content': chunk})}\n\n"
                await asyncio.sleep(0.01)  # Simulate streaming
            yield "data: [DONE]\n\n"
        return StreamingResponse(cached_stream(), media_type="text/event-stream")
    
    # Stream from LLM
    messages = [{"role": "user", "content": user_message}]
    return StreamingResponse(
        stream_llm_response(messages),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
        }
    )

@app.post("/chat/optimized")
async def chat_optimized(request: Request):
    """
    Non-streaming with all optimizations applied:
    - Caching
    - Parallel operations
    - Connection pooling
    """
    body = await request.json()
    user_message = body.get("message", "")
    user_id = body.get("user_id", "anonymous")
    
    # Run cache check and user context fetch in parallel
    cache_key = get_cache_key(user_message)
    cached_response, user_context = await asyncio.gather(
        redis_client.get(cache_key),
        fetch_user_context(user_id)  # DB call
    )
    
    if cached_response:
        return json.loads(cached_response)
    
    # Call LLM
    messages = build_messages(user_message, user_context)
    response = await openai_client.chat.completions.create(
        model="gpt-3.5-turbo",  # Faster than gpt-4
        messages=messages,
        max_tokens=500
    )
    
    result = {"response": response.choices[0].message.content}
    
    # Cache for 1 hour
    await redis_client.setex(cache_key, 3600, json.dumps(result))
    
    return result

async def fetch_user_context(user_id: str) -> dict:
    """Fetch user context from database"""
    await asyncio.sleep(0.1)  # Simulated DB call
    return {"user_id": user_id, "preferences": []}

def build_messages(user_message: str, context: dict) -> list:
    return [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": user_message}
    ]
```

### Key Takeaways
> - 💡 **Streaming (SSE) is the #1 fix** — users see response in 0.3s instead of waiting 10s
> - 💡 **Cache common questions** — 80% of questions are repeated
> - 💡 **Use asyncio.gather()** to parallelize independent operations
> - 💡 **Profile first** with timing middleware before optimizing blindly
> - 💡 **Use faster models** (GPT-3.5 vs GPT-4) for non-critical responses

---

## Question 3 — API Drops from 1000 to 100 req/sec
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your API serves 1000 req/sec but suddenly drops to 100 req/sec without any code changes. How do you investigate?"*

### The Answer

**Systematic investigation checklist:**

```
INVESTIGATION FLOW:

   Request drops detected
           │
     ┌─────▼──────┐
     │  Check    │
     │  Metrics  │
     └─────┬─────┘
           │
    ┌──────▼───────┐    ┌──────────────────┐
    │   CPU/Memory  │───►│ If >90%: scale   │
    │    usage?     │    │ horizontally     │
    └──────┬───────┘    └──────────────────┘
           │
    ┌──────▼───────┐    ┌──────────────────┐
    │  DB Connection│───►│ Increase pool    │
    │  Pool full?   │    │ size or fix leaks│
    └──────┬───────┘    └──────────────────┘
           │
    ┌──────▼───────┐    ┌──────────────────┐
    │ External API  │───►│ Circuit breaker  │
    │  slowdown?    │    │ + timeout        │
    └──────┬───────┘    └──────────────────┘
           │
    ┌──────▼───────┐    ┌──────────────────┐
    │  Memory leak? │───►│ Restart + profile│
    │ GC pressure?  │    │ tracemalloc      │
    └──────────────┘    └──────────────────┘
```

### Code Example — Monitoring Middleware

```python
import time
import psutil
import asyncio
from collections import deque
from fastapi import FastAPI, Request
from prometheus_client import Counter, Histogram, Gauge, generate_latest

app = FastAPI()

# Prometheus metrics
REQUEST_COUNT = Counter("api_requests_total", "Total requests", ["endpoint", "status"])
REQUEST_LATENCY = Histogram("api_request_duration_seconds", "Request latency", ["endpoint"])
ACTIVE_REQUESTS = Gauge("api_active_requests", "Active requests")
DB_POOL_SIZE = Gauge("db_pool_size", "Database connection pool size")

# Sliding window for req/sec calculation
request_times = deque(maxlen=1000)

@app.middleware("http")
async def monitoring_middleware(request: Request, call_next):
    start = time.time()
    ACTIVE_REQUESTS.inc()
    request_times.append(start)
    
    try:
        response = await call_next(request)
        status = response.status_code
    except Exception as e:
        status = 500
        raise
    finally:
        duration = time.time() - start
        ACTIVE_REQUESTS.dec()
        REQUEST_COUNT.labels(request.url.path, status).inc()
        REQUEST_LATENCY.labels(request.url.path).observe(duration)
    
    return response

@app.get("/metrics/health")
async def health_check():
    """Detailed health check for debugging performance drops"""
    now = time.time()
    
    # Calculate req/sec over last 60 seconds
    recent_requests = [t for t in request_times if now - t < 60]
    rps = len(recent_requests) / 60
    
    # System metrics
    cpu_percent = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    
    return {
        "requests_per_second": round(rps, 2),
        "system": {
            "cpu_percent": cpu_percent,
            "memory_percent": memory.percent,
            "memory_available_mb": memory.available // 1024 // 1024,
        },
        "warnings": [
            f"High CPU: {cpu_percent}%" if cpu_percent > 80 else None,
            f"Low memory: {memory.percent}%" if memory.percent > 90 else None,
            f"Low RPS: {rps}" if rps < 100 else None,
        ]
    }

@app.get("/metrics/prometheus")
async def prometheus_metrics():
    """Expose metrics for Prometheus scraping"""
    from fastapi.responses import PlainTextResponse
    return PlainTextResponse(generate_latest())
```

### Key Takeaways
> - 💡 **Always have metrics** — you can't debug what you can't measure
> - 💡 **Check in this order**: CPU/Memory → DB connections → External deps → Memory leaks
> - 💡 **Connection pool exhaustion** is the most common cause of sudden drops
> - 💡 **Add Prometheus + Grafana** for production monitoring

---

## Question 4 — Endpoint Returns 1 Million Records
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your endpoint returns 1 million records. Users complain about slow response and the page freezes. How do you optimize?"*

### The Answer

**Pagination strategies comparison:**

```
PAGINATION STRATEGIES:
┌─────────────────┬────────────────────────────────────────────────────┐
│   STRATEGY      │  HOW IT WORKS                                      │
├─────────────────┼────────────────────────────────────────────────────┤
│ Offset/Limit    │  SELECT * FROM t LIMIT 20 OFFSET 1000000           │
│ (AVOID for big) │  Problem: DB scans all 1M rows before skipping!    │
├─────────────────┼────────────────────────────────────────────────────┤
│ Cursor-based    │  SELECT * FROM t WHERE id > {last_id} LIMIT 20     │
│ (RECOMMENDED)   │  Uses index! Fast even at row 999,999              │
├─────────────────┼────────────────────────────────────────────────────┤
│ Keyset          │  SELECT * FROM t WHERE (created_at, id) > (t, id) │
│ (Multi-column)  │  Best for sorted data with ties                    │
└─────────────────┴────────────────────────────────────────────────────┘
```

```mermaid
flowchart TD
    A[Client requests data] --> B{How many records?}
    B -->|< 1000| C[Return all at once\nwith field filtering]
    B -->|1000 - 100k| D[Cursor-based pagination\n20-100 records per page]
    B -->|> 100k| E[Async export endpoint\nReturn download link]
    D --> F[Include next_cursor in response]
    F --> G[Client uses cursor for next page]
    E --> H[Background job generates file]
    H --> I[Notify client via webhook/email]
```

### Code Example — Cursor-Based Pagination

```python
from fastapi import FastAPI, Query, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import select, Integer, String, DateTime
from datetime import datetime
from typing import Optional
import base64
import json

app = FastAPI()
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String)
    created_at: Mapped[datetime] = mapped_column(DateTime)

def encode_cursor(last_id: int, last_created_at: str) -> str:
    """Encode cursor as base64 to hide implementation details"""
    data = {"id": last_id, "created_at": last_created_at}
    return base64.b64encode(json.dumps(data).encode()).decode()

def decode_cursor(cursor: str) -> dict:
    """Decode cursor back to query parameters"""
    try:
        return json.loads(base64.b64decode(cursor).decode())
    except Exception:
        raise HTTPException(400, "Invalid cursor")

@app.get("/products")
async def list_products(
    limit: int = Query(default=20, ge=1, le=100),
    cursor: Optional[str] = None,
    fields: Optional[str] = Query(default=None, description="Comma-separated fields to return")
):
    """
    Cursor-based pagination - fast even for 1M+ records
    
    First page: GET /products?limit=20
    Next page:  GET /products?limit=20&cursor={next_cursor_from_previous_response}
    """
    async with AsyncSession(engine) as session:
        query = select(Product).order_by(Product.id).limit(limit + 1)  # +1 to check if more pages exist
        
        if cursor:
            cursor_data = decode_cursor(cursor)
            # Use index-friendly comparison
            query = query.where(Product.id > cursor_data["id"])
        
        result = await session.execute(query)
        products = result.scalars().all()
        
        has_next = len(products) > limit
        if has_next:
            products = products[:limit]
        
        # Build response
        items = []
        for p in products:
            item = {"id": p.id, "name": p.name, "created_at": p.created_at.isoformat()}
            
            # Field filtering — return only requested fields
            if fields:
                requested = set(fields.split(","))
                item = {k: v for k, v in item.items() if k in requested}
            
            items.append(item)
        
        next_cursor = None
        if has_next and products:
            last = products[-1]
            next_cursor = encode_cursor(last.id, last.created_at.isoformat())
        
        return {
            "items": items,
            "pagination": {
                "limit": limit,
                "has_next": has_next,
                "next_cursor": next_cursor,
                "count": len(items)
            }
        }

# For massive exports: async endpoint
@app.post("/products/export")
async def export_products(background_tasks):
    """For 1M+ records: async CSV/JSON export"""
    export_id = "export_123"
    background_tasks.add_task(generate_export, export_id)
    return {
        "export_id": export_id,
        "status": "generating",
        "download_url_available_at": f"/exports/{export_id}"
    }
```

### Key Takeaways
> - 💡 **Never use OFFSET for large tables** — it's O(n) not O(1)
> - 💡 **Cursor pagination is O(log n)** using the index
> - 💡 **Add field filtering** — clients often only need 3 fields from 30
> - 💡 **For exports > 100k rows** — use async job + file download pattern
> - 💡 **Always add a maximum limit** — prevent `?limit=1000000`

---

## Question 5 — Multiple Clients Call Same Expensive Query
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Multiple clients call your API with the same expensive query (5 seconds) within seconds of each other. How do you avoid running it 100 times?"*

### The Answer

```
WITHOUT CACHING (100 clients × 5 seconds each = 500 CPU seconds):

Client 1 ──► API ──► DB (5s query) ──► Response
Client 2 ──► API ──► DB (5s query) ──► Response   ← Same query!
Client 3 ──► API ──► DB (5s query) ──► Response   ← Same query!
...×100

WITH CACHING (first client pays 5s, rest get instant response):

Client 1 ──► API ──► MISS ──► DB (5s) ──► Cache SET ──► Response
Client 2 ──► API ──► HIT ──► Cache GET (1ms) ──────────► Response
Client 3 ──► API ──► HIT ──► Cache GET (1ms) ──────────► Response
```

### Code Example — Redis Cache-Aside Pattern

```python
import hashlib
import json
import time
from typing import Optional, Any
from functools import wraps

import redis.asyncio as redis
from fastapi import FastAPI, Query

app = FastAPI()
redis_client = redis.Redis(host="localhost", port=6379, decode_responses=True)

# ---- Cache decorator ----
def cache(expire: int = 300, key_prefix: str = ""):
    """Decorator for automatic caching of endpoint responses"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Build cache key from function name + arguments
            key_parts = [key_prefix or func.__name__] + [str(v) for v in kwargs.values()]
            cache_key = hashlib.md5(":".join(key_parts).encode()).hexdigest()
            
            # Try cache first
            cached = await redis_client.get(cache_key)
            if cached:
                result = json.loads(cached)
                result["_cached"] = True
                return result
            
            # Cache miss — run actual function
            result = await func(*args, **kwargs)
            await redis_client.setex(cache_key, expire, json.dumps(result))
            return result
        return wrapper
    return decorator

# ---- Request coalescing (prevent cache stampede) ----
pending_requests: dict[str, asyncio.Future] = {}

async def get_with_coalescing(cache_key: str, fetch_func) -> Any:
    """
    If multiple requests come in for the same uncached key simultaneously,
    only ONE goes to the database. Others wait for that one to complete.
    This prevents cache stampede.
    """
    import asyncio
    
    # Check cache first
    cached = await redis_client.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Check if another request is already fetching this key
    if cache_key in pending_requests:
        # Wait for the first request to complete
        return await pending_requests[cache_key]
    
    # We are the first request — create a future others will wait on
    future = asyncio.get_event_loop().create_future()
    pending_requests[cache_key] = future
    
    try:
        result = await fetch_func()
        await redis_client.setex(cache_key, 300, json.dumps(result))
        future.set_result(result)
        return result
    except Exception as e:
        future.set_exception(e)
        raise
    finally:
        pending_requests.pop(cache_key, None)

@app.get("/reports/expensive")
@cache(expire=300, key_prefix="report")
async def get_expensive_report(
    date_from: str = Query(...),
    date_to: str = Query(...)
):
    """Expensive report — results cached for 5 minutes"""
    # Simulate expensive DB query
    import asyncio
    await asyncio.sleep(5)  
    return {
        "data": [{"metric": "sales", "value": 42000}],
        "generated_at": time.time()
    }

@app.delete("/cache/invalidate/{key_prefix}")
async def invalidate_cache(key_prefix: str):
    """Manually invalidate cache when data changes"""
    # Delete all keys matching pattern
    keys = await redis_client.keys(f"*{key_prefix}*")
    if keys:
        await redis_client.delete(*keys)
    return {"invalidated": len(keys), "keys": keys}
```

### Key Takeaways
> - 💡 **Cache-aside pattern**: app manages cache, DB is source of truth
> - 💡 **Request coalescing**: prevent N identical DB queries when cache is cold
> - 💡 **Set appropriate TTL**: balance freshness vs performance
> - 💡 **Invalidate cache on writes**: `DELETE /cache/invalidate/report`
> - 💡 **Consider stale-while-revalidate**: serve stale data while refreshing in background

---

## Question 6 — 5 Sequential External API Calls (5 seconds)
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API makes 5 external API calls sequentially per request, each taking 1 second. Total response time: 5 seconds. How do you reduce this to ~1 second?"*

### The Answer

```
SEQUENTIAL (current — 5 seconds):
─────────────────────────────────────────────────────────

0s    1s    2s    3s    4s    5s
│     │     │     │     │     │
▓▓▓▓▓▓│▓▓▓▓▓▓│▓▓▓▓▓▓│▓▓▓▓▓▓│▓▓▓▓▓▓│
Call1  Call2  Call3  Call4  Call5

PARALLEL (asyncio.gather — ~1 second):
─────────────────────────────────────────────────────────

0s    1s
│     │
▓▓▓▓▓▓│ Call1 (1s)
▓▓▓▓▓▓│ Call2 (1s)
▓▓▓▓▓▓│ Call3 (1s)
▓▓▓▓▓▓│ Call4 (1s)
▓▓▓▓▓▓│ Call5 (1s)
      │
      └─── All done! Total: 1s
```

### Code Example — asyncio.gather + Circuit Breaker

```python
import asyncio
import httpx
import time
from dataclasses import dataclass
from enum import Enum
from typing import Optional

from fastapi import FastAPI, HTTPException

app = FastAPI()

# ---- Circuit Breaker ----
class CircuitState(Enum):
    CLOSED = "closed"      # Normal — requests pass through
    OPEN = "open"          # Failed — requests blocked
    HALF_OPEN = "half_open"  # Testing — one request allowed

@dataclass
class CircuitBreaker:
    name: str
    failure_threshold: int = 5
    reset_timeout: int = 60  # seconds
    
    failures: int = 0
    state: CircuitState = CircuitState.CLOSED
    last_failure_time: float = 0
    
    def can_execute(self) -> bool:
        if self.state == CircuitState.CLOSED:
            return True
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.reset_timeout:
                self.state = CircuitState.HALF_OPEN
                return True
            return False
        return True  # HALF_OPEN: allow one request
    
    def record_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED
    
    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Circuit breakers per external service
circuit_breakers = {
    "weather": CircuitBreaker("weather"),
    "pricing": CircuitBreaker("pricing"),
    "inventory": CircuitBreaker("inventory"),
    "user_profile": CircuitBreaker("user_profile"),
    "recommendations": CircuitBreaker("recommendations"),
}

async def call_external_api(
    client: httpx.AsyncClient,
    service_name: str,
    url: str,
    fallback: dict = None
) -> dict:
    """Call external API with circuit breaker and fallback"""
    cb = circuit_breakers[service_name]
    
    if not cb.can_execute():
        print(f"Circuit OPEN for {service_name} — using fallback")
        return fallback or {"error": f"{service_name} unavailable"}
    
    try:
        response = await client.get(url, timeout=3.0)  # 3s timeout
        response.raise_for_status()
        cb.record_success()
        return response.json()
    except (httpx.TimeoutException, httpx.HTTPStatusError) as e:
        cb.record_failure()
        print(f"Error calling {service_name}: {e}")
        return fallback or {"error": str(e)}

@app.get("/product/{product_id}/details")
async def get_product_details(product_id: str):
    """
    BEFORE: 5 sequential calls = 5 seconds
    AFTER: 5 parallel calls = ~1 second
    """
    async with httpx.AsyncClient() as client:
        # Run all 5 API calls in PARALLEL
        results = await asyncio.gather(
            call_external_api(
                client, "weather",
                f"http://weather-api/current",
                fallback={"temp": "N/A"}
            ),
            call_external_api(
                client, "pricing",
                f"http://pricing-api/product/{product_id}",
                fallback={"price": 0}
            ),
            call_external_api(
                client, "inventory",
                f"http://inventory-api/product/{product_id}",
                fallback={"stock": "unknown"}
            ),
            call_external_api(
                client, "user_profile",
                f"http://user-api/profile",
                fallback={"preferences": []}
            ),
            call_external_api(
                client, "recommendations",
                f"http://rec-api/for/{product_id}",
                fallback={"items": []}
            ),
            return_exceptions=True  # Don't fail all if one fails
        )
    
    weather, pricing, inventory, profile, recommendations = results
    
    return {
        "product_id": product_id,
        "weather": weather if not isinstance(weather, Exception) else None,
        "pricing": pricing if not isinstance(pricing, Exception) else None,
        "inventory": inventory if not isinstance(inventory, Exception) else None,
        "profile": profile if not isinstance(profile, Exception) else None,
        "recommendations": recommendations if not isinstance(recommendations, Exception) else None,
    }
```

### Key Takeaways
> - 💡 **asyncio.gather()** runs coroutines in parallel — biggest win
> - 💡 **return_exceptions=True** so one failure doesn't cancel all others
> - 💡 **Circuit breaker** prevents one slow service from slowing down your API
> - 💡 **Always set timeouts** on external HTTP calls (never wait forever)
> - 💡 **Provide fallbacks** for non-critical data (recommendations can be empty)

---

## Question 7 — File Upload Crashes with 500MB+ Files
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your file upload endpoint crashes with large files (500MB+). Memory usage spikes to 100% and the server dies. How do you handle this?"*

### The Answer

```
PROBLEM — Loading entire file into memory:

Client ──500MB──► Server RAM [██████████ 500MB!] ──► CRASH

SOLUTION — Stream directly to storage:

Client ──chunks──► Server [10MB buffer] ──chunks──► S3/Storage
                   (never holds full file in memory)

BEST SOLUTION — Pre-signed URL (bypass server entirely):

Client ──1──► API: "I want to upload a file"
API    ──2──► S3: "Generate upload URL"
API    ──3──► Client: Here's your pre-signed URL
Client ──4──► S3: (uploads 500MB directly, API not involved!)
S3     ──5──► Webhook: "File uploaded, key=abc123"
API    ──6──► Process file from S3
```

### Code Example — Streaming Upload + Pre-signed URLs

```python
import boto3
import hashlib
import os
import aiofiles
from pathlib import Path
from fastapi import FastAPI, UploadFile, File, HTTPException, BackgroundTasks
from fastapi.responses import JSONResponse

app = FastAPI()
s3_client = boto3.client("s3", region_name="us-east-1")
BUCKET_NAME = "my-upload-bucket"
UPLOAD_DIR = Path("/tmp/uploads")
UPLOAD_DIR.mkdir(exist_ok=True)

# Method 1: Streaming upload (handles large files without OOM)
@app.post("/upload/stream")
async def stream_upload(
    file: UploadFile = File(...),
    background_tasks: BackgroundTasks = None
):
    """Stream file directly to disk without loading into memory"""
    
    # Validate file type using magic bytes (not extension!)
    first_chunk = await file.read(8)
    if not is_valid_file_type(first_chunk):
        raise HTTPException(400, "Invalid file type")
    await file.seek(0)
    
    # Validate file size
    MAX_SIZE = 500 * 1024 * 1024  # 500MB
    file_path = UPLOAD_DIR / f"{file.filename}"
    total_size = 0
    
    async with aiofiles.open(file_path, "wb") as f:
        while chunk := await file.read(1024 * 1024):  # Read 1MB at a time
            total_size += len(chunk)
            if total_size > MAX_SIZE:
                await aiofiles.os.remove(file_path)
                raise HTTPException(413, f"File exceeds {MAX_SIZE // 1024 // 1024}MB limit")
            await f.write(chunk)
    
    # Upload to S3 in background
    job_id = hashlib.md5(file.filename.encode()).hexdigest()[:8]
    background_tasks.add_task(upload_to_s3, str(file_path), file.filename, job_id)
    
    return {"job_id": job_id, "status": "uploading", "filename": file.filename}

# Method 2: Pre-signed URL (BEST for large files — server not involved in upload)
@app.post("/upload/presigned-url")
async def get_presigned_url(filename: str, content_type: str, file_size_bytes: int):
    """
    Generate a pre-signed S3 URL.
    Client uploads DIRECTLY to S3 — API never sees the file bytes!
    """
    if file_size_bytes > 5 * 1024 * 1024 * 1024:  # 5GB limit
        raise HTTPException(400, "File too large (max 5GB)")
    
    # Sanitize filename
    safe_filename = "".join(c for c in filename if c.isalnum() or c in ".-_")
    s3_key = f"uploads/{safe_filename}"
    
    # Generate pre-signed URL (valid for 15 minutes)
    presigned_url = s3_client.generate_presigned_url(
        "put_object",
        Params={
            "Bucket": BUCKET_NAME,
            "Key": s3_key,
            "ContentType": content_type,
            "ContentLength": file_size_bytes,
        },
        ExpiresIn=900  # 15 minutes
    )
    
    return {
        "upload_url": presigned_url,
        "s3_key": s3_key,
        "instructions": "PUT your file directly to upload_url",
        "notify_url": f"/upload/complete?key={s3_key}"
    }

@app.post("/upload/complete")
async def upload_complete(key: str, background_tasks: BackgroundTasks):
    """Called by client after S3 upload completes to trigger processing"""
    # Verify file actually exists in S3
    try:
        s3_client.head_object(Bucket=BUCKET_NAME, Key=key)
    except Exception:
        raise HTTPException(404, "File not found in S3")
    
    job_id = hashlib.md5(key.encode()).hexdigest()[:8]
    background_tasks.add_task(process_file_from_s3, key, job_id)
    
    return {"job_id": job_id, "status": "processing"}

def is_valid_file_type(magic_bytes: bytes) -> bool:
    """Validate file type by magic bytes, not extension"""
    signatures = {
        b"\xff\xd8\xff": "JPEG",
        b"\x89PNG": "PNG",
        b"GIF8": "GIF",
        b"%PDF": "PDF",
        b"PK\x03\x04": "ZIP/DOCX/XLSX",
    }
    return any(magic_bytes.startswith(sig) for sig in signatures)

async def upload_to_s3(file_path: str, filename: str, job_id: str):
    """Background task: upload file to S3 and cleanup"""
    try:
        s3_client.upload_file(file_path, BUCKET_NAME, f"uploads/{filename}")
    finally:
        os.remove(file_path)  # Always cleanup temp file

async def process_file_from_s3(s3_key: str, job_id: str):
    """Background task: process file from S3"""
    print(f"Processing {s3_key} for job {job_id}")
```

### Key Takeaways
> - 💡 **Stream files in chunks** — never load the entire file into memory
> - 💡 **Pre-signed S3 URLs are the gold standard** for large file uploads
> - 💡 **Validate by magic bytes** not file extension (`.jpg` could be a PHP script)
> - 💡 **Always set size limits** before starting to write to disk
> - 💡 **Clean up temp files** even on failure (use try/finally)

---

## Question 8 — Database Queries Getting Slower Over Time
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API's database queries are getting slower as the table grows from 100K to 10M rows. Page loads went from 200ms to 8 seconds. How do you handle this?"*

### The Answer

```
QUERY OPTIMIZATION DECISION TREE:

    Slow Query
        │
   EXPLAIN ANALYZE
        │
   ┌────▼────────┐
   │  Seq Scan   │──►  Add index on WHERE columns
   │  on large   │
   │  table?     │
   └────┬────────┘
        │
   ┌────▼────────┐
   │  Index      │──►  Multi-column index? Composite index
   │  exists but │     covering query
   │  not used?  │
   └────┬────────┘
        │
   ┌────▼────────┐
   │  Too many   │──►  Read replica
   │  concurrent │     Connection pooling
   │  queries?   │     Caching
   └─────────────┘
```

### Code Example — Query Optimization

```python
from sqlalchemy import text, Index
from sqlalchemy.ext.asyncio import AsyncSession

# Add missing indexes
class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String, index=True)  # index!
    created_at: Mapped[datetime] = mapped_column(DateTime)
    status: Mapped[str] = mapped_column(String)
    
    # Composite index for common query: WHERE status = ? ORDER BY created_at
    __table_args__ = (
        Index("ix_users_status_created", "status", "created_at"),
    )

@app.get("/debug/query-analysis")
async def analyze_query(db: AsyncSession):
    """Run EXPLAIN ANALYZE to see query plan"""
    result = await db.execute(
        text("EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active' ORDER BY created_at DESC LIMIT 20")
    )
    plan = result.fetchall()
    return {"query_plan": [str(row) for row in plan]}
```

### Key Takeaways
> - 💡 **EXPLAIN ANALYZE** before adding any index — find the actual bottleneck
> - 💡 **Index columns used in WHERE, JOIN, ORDER BY** clauses
> - 💡 **Composite indexes** for multi-column queries (order matters!)
> - 💡 **Read replicas** for read-heavy workloads
> - 💡 **Partition large tables** by date/range for time-series data

---

## Question 9 — Process CSV with 1 Million Rows
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your API needs to process a CSV file with 1 million rows uploaded by users. Processing takes 30 minutes. How do you design this?"*

### The Answer

```
ASYNC PROCESSING PIPELINE:

1. Client uploads CSV ──► API returns job_id immediately (< 1 second)
2. Background worker picks up job
3. Worker streams CSV (not load all into memory)
4. Processes in chunks (1000 rows at a time)
5. Updates progress in Redis
6. Client polls /jobs/{id} for status
7. On completion: webhook notification or email

┌──────────┐     ┌─────────┐     ┌──────────┐     ┌──────────┐
│  Client  │────►│   API   │────►│  Queue   │────►│  Worker  │
│          │◄────│         │     │ (Redis)  │     │ (Celery) │
│ job_id   │     │ job_id  │     │          │     │          │
│ returned │     │ stored  │     │          │     │ processes│
│ in <1s   │     │ in DB   │     │          │     │ CSV      │
└──────────┘     └─────────┘     └──────────┘     └──────────┘
      │                                                  │
      │ GET /jobs/{id}                                   │
      └──────────────────────────────────────────────────┘
              Client polls for progress
```

### Code Example — Async CSV Processing Pipeline

```python
import csv
import io
from celery import Celery
from fastapi import FastAPI, UploadFile, BackgroundTasks

app = FastAPI()
celery_app = Celery("tasks", broker="redis://localhost:6379/0")

@app.post("/upload/csv")
async def upload_csv(file: UploadFile, background_tasks: BackgroundTasks):
    """Accept CSV upload and return job ID immediately"""
    if not file.filename.endswith(".csv"):
        raise HTTPException(400, "Only CSV files accepted")
    
    job_id = str(uuid.uuid4())
    
    # Save file to temp storage
    file_path = f"/tmp/{job_id}.csv"
    async with aiofiles.open(file_path, "wb") as f:
        while chunk := await file.read(1024 * 1024):
            await f.write(chunk)
    
    # Queue background processing
    process_csv_task.delay(job_id, file_path)
    
    return {"job_id": job_id, "status": "queued", "poll_url": f"/jobs/{job_id}"}

@celery_app.task(bind=True)
def process_csv_task(self, job_id: str, file_path: str):
    """Celery task: process CSV in chunks"""
    import redis
    r = redis.Redis()
    
    chunk_size = 1000
    processed = 0
    errors = []
    
    with open(file_path, "r") as f:
        reader = csv.DictReader(f)
        chunk = []
        
        for row in reader:
            chunk.append(row)
            
            if len(chunk) >= chunk_size:
                try:
                    process_chunk(chunk)
                    processed += len(chunk)
                    # Update progress
                    r.hset(f"job:{job_id}", mapping={
                        "status": "processing",
                        "processed": processed,
                        "progress": min(99, int(processed / 1_000_000 * 100))
                    })
                except Exception as e:
                    errors.append(str(e))
                chunk = []
        
        # Process remaining
        if chunk:
            process_chunk(chunk)
            processed += len(chunk)
    
    # Mark complete
    r.hset(f"job:{job_id}", mapping={
        "status": "completed",
        "processed": processed,
        "progress": 100,
        "errors": len(errors)
    })
    r.expire(f"job:{job_id}", 86400)  # Cleanup after 24h
    
    import os
    os.remove(file_path)

def process_chunk(rows: list):
    """Process a chunk of rows"""
    # Bulk insert to database
    pass

@app.get("/jobs/{job_id}")
async def get_job_status(job_id: str):
    import redis
    r = redis.Redis(decode_responses=True)
    job = r.hgetall(f"job:{job_id}")
    if not job:
        raise HTTPException(404, "Job not found")
    return job
```

### Key Takeaways
> - 💡 **Return job_id immediately** — never make users wait 30 minutes
> - 💡 **Stream CSV** don't load 1M rows into memory
> - 💡 **Process in chunks** (1000 rows) for memory efficiency and progress tracking
> - 💡 **Store progress in Redis** for fast polling
> - 💡 **Use Celery + Redis** for production-grade background processing

---

## Question 10 — Reduce 5MB JSON Response Payload
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Your API response payload is 5MB JSON. Mobile users on slow connections are complaining about slow load times. How do you reduce it?"*

### The Answer

```
RESPONSE SIZE OPTIMIZATION LADDER:

5MB JSON (baseline)
  │
  ├─ Field filtering (/users?fields=id,name)     → ~500KB (90% reduction)
  │
  ├─ Pagination (return 20 items, not 10000)      → ~50KB  (99% reduction)
  │
  ├─ gzip compression (Accept-Encoding: gzip)     → ~15KB  (70% of remaining)
  │
  ├─ Remove nulls/empty fields                    → ~12KB
  │
  └─ ETags (304 Not Modified if unchanged)        → 0KB    (if content unchanged)
```

### Code Example — Response Compression + Field Filtering

```python
from fastapi import FastAPI, Query
from fastapi.middleware.gzip import GZipMiddleware
from typing import Optional

app = FastAPI()

# Add gzip compression for responses > 1KB
app.add_middleware(GZipMiddleware, minimum_size=1000)

@app.get("/users")
async def list_users(
    fields: Optional[str] = Query(None, description="Comma-separated: id,name,email"),
    page: int = 1,
    limit: int = Query(20, le=100)
):
    """Return only requested fields, paginated"""
    users = await fetch_users(page=page, limit=limit)
    
    if fields:
        requested = set(fields.split(","))
        users = [{k: v for k, v in u.items() if k in requested} for u in users]
    
    # Remove null/empty values to reduce size
    users = [{k: v for k, v in u.items() if v is not None and v != ""} for u in users]
    
    return {"users": users, "page": page, "limit": limit}

# ETag support
import hashlib
from fastapi import Response, Request

@app.get("/report/heavy")
async def heavy_report(request: Request, response: Response):
    """Return 304 Not Modified if content hasn't changed"""
    data = await generate_report()
    etag = hashlib.md5(str(data).encode()).hexdigest()
    
    if request.headers.get("If-None-Match") == etag:
        return Response(status_code=304)
    
    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "max-age=300"
    return data

async def fetch_users(page: int, limit: int):
    return [{"id": i, "name": f"User {i}", "email": f"user{i}@example.com"} 
            for i in range((page-1)*limit, page*limit)]

async def generate_report():
    return {"data": list(range(100))}
```

### Key Takeaways
> - 💡 **GZipMiddleware** reduces JSON by 60-80% automatically
> - 💡 **Field filtering** is the biggest win (90% reduction possible)
> - 💡 **Paginate** instead of returning all records
> - 💡 **ETags** prevent sending data that hasn't changed
> - 💡 **Remove nulls** — they still take bytes in JSON

---

*Next: [02 — Error Handling & Debugging →](./02-error-handling-debugging.md)*
