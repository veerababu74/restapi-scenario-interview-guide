# 02 — Error Handling & Debugging

> **Questions 11–20** | Finding and fixing bugs, handling failures gracefully

---

## Question 11 — Intermittent 500 Errors You Can't Reproduce Locally
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API returns 500 errors intermittently — maybe 2% of requests. You can't reproduce it locally. Logs are minimal. How do you debug this?"*

### The Answer

**Step 1: Implement structured logging with correlation IDs**

```
WITHOUT CORRELATION IDs (impossible to trace):

[ERROR] Database error
[ERROR] Database error
[INFO]  Request completed
[ERROR] Null pointer exception

WITH CORRELATION IDs (trace each request end-to-end):

[ERROR] req_id=abc123 user=john@ex.com endpoint=/orders Database error: timeout
[ERROR] req_id=def456 user=jane@ex.com endpoint=/orders Null pointer: order.items is None
[INFO]  req_id=ghi789 user=bob@ex.com endpoint=/orders completed in 245ms
```

```mermaid
flowchart LR
    A[Client Request] --> B[Load Balancer]
    B --> C[API Server]
    C --> D[Database]
    C --> E[External Service]
    
    A -- "X-Request-ID: abc123" --> B
    B -- "X-Request-ID: abc123" --> C
    C -- "SELECT ... correlation_id=abc123" --> D
    C -- "POST ... X-Request-ID: abc123" --> E
    
    style A fill:#e1f5fe
    style C fill:#f3e5f5
```

### Code Example — Structured Logging + Correlation IDs

```python
import logging
import json
import uuid
import sys
import traceback
from contextvars import ContextVar
from typing import Callable

from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse

app = FastAPI()

# Context variable to store request ID across async calls
request_id_var: ContextVar[str] = ContextVar("request_id", default="")

# --- Structured JSON Logger ---
class StructuredLogger:
    def __init__(self, name: str):
        self.logger = logging.getLogger(name)
        handler = logging.StreamHandler(sys.stdout)
        handler.setFormatter(self._json_formatter())
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.DEBUG)
    
    def _json_formatter(self):
        class JSONFormatter(logging.Formatter):
            def format(self, record):
                log_data = {
                    "timestamp": self.formatTime(record),
                    "level": record.levelname,
                    "message": record.getMessage(),
                    "request_id": request_id_var.get(""),
                    "module": record.module,
                    "function": record.funcName,
                    "line": record.lineno,
                }
                if record.exc_info:
                    log_data["exception"] = self.formatException(record.exc_info)
                return json.dumps(log_data)
        return JSONFormatter()
    
    def info(self, msg, **kwargs):
        self.logger.info(msg, extra=kwargs)
    
    def error(self, msg, **kwargs):
        self.logger.error(msg, extra=kwargs)
    
    def warning(self, msg, **kwargs):
        self.logger.warning(msg, extra=kwargs)

logger = StructuredLogger("api")

# --- Correlation ID Middleware ---
@app.middleware("http")
async def correlation_id_middleware(request: Request, call_next: Callable):
    # Use existing ID from headers or generate new one
    correlation_id = (
        request.headers.get("X-Request-ID") or
        request.headers.get("X-Correlation-ID") or
        str(uuid.uuid4())
    )
    
    # Store in context variable (available throughout async call chain)
    request_id_var.set(correlation_id)
    
    # Log request
    logger.info(
        f"Request started",
        path=str(request.url.path),
        method=request.method,
        user_agent=request.headers.get("user-agent"),
    )
    
    try:
        response = await call_next(request)
        response.headers["X-Request-ID"] = correlation_id
        return response
    except Exception as e:
        # Log unhandled exception with full context
        logger.error(
            f"Unhandled exception: {type(e).__name__}: {str(e)}",
            traceback=traceback.format_exc(),
            path=str(request.url.path),
        )
        return JSONResponse(
            status_code=500,
            content={
                "error": "Internal server error",
                "request_id": correlation_id,  # Client can use this to report issues
                "message": "Please contact support with the request_id"
            }
        )

# --- Global Exception Handler ---
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    correlation_id = request_id_var.get("unknown")
    
    logger.error(
        "Unhandled exception",
        error_type=type(exc).__name__,
        error_message=str(exc),
        traceback=traceback.format_exc(),
        request_path=request.url.path,
        request_method=request.method,
    )
    
    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "request_id": correlation_id
        }
    )

# --- Sentry Integration (for production) ---
def setup_sentry():
    import sentry_sdk
    from sentry_sdk.integrations.fastapi import FastApiIntegration
    
    sentry_sdk.init(
        dsn="your-sentry-dsn",
        integrations=[FastApiIntegration()],
        traces_sample_rate=0.1,  # Sample 10% for performance monitoring
        environment="production",
    )
    
    # Add user context to every Sentry error
    @app.middleware("http")
    async def sentry_context_middleware(request: Request, call_next):
        with sentry_sdk.configure_scope() as scope:
            scope.set_tag("request_id", request_id_var.get(""))
            scope.set_context("request", {
                "url": str(request.url),
                "method": request.method,
            })
        return await call_next(request)

# Flask equivalent
"""
from flask import Flask, g, request
import uuid

app = Flask(__name__)

@app.before_request
def before_request():
    g.request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    app.logger.info(json.dumps({
        "event": "request_started",
        "request_id": g.request_id,
        "path": request.path
    }))

@app.after_request
def after_request(response):
    response.headers['X-Request-ID'] = g.request_id
    return response
"""
```

### Key Takeaways
> - 💡 **Add correlation IDs** as first step — you can't debug without them
> - 💡 **Structured JSON logging** makes log queries possible (vs plain text)
> - 💡 **Use Sentry** in production — it groups similar errors and tracks frequency
> - 💡 **Return request_id in error responses** — clients can report it to support
> - 💡 **Log at request start AND end** — if only start is logged, it crashed mid-request

---

## Question 12 — Client Reports Timeouts but Server Logs Show Success
🟡 Mid | ★★☆ Common

### The Scenario
> *"A client reports they keep getting timeout errors, but your server logs show all requests completing successfully in under 500ms. What's happening?"*

### The Answer

**The network has layers between client and server:**

```
CLIENT ──► DNS ──► CDN ──► Load Balancer ──► Nginx ──► App Server ──► Database

Each layer has its OWN timeout setting!
If ANY layer times out, client gets error even if app server succeeds.

Common culprits:
┌─────────────────────────────────────────────────────────┐
│  Layer          │ Default Timeout │ Who Configures      │
├─────────────────────────────────────────────────────────┤
│ Client SDK      │ 30s             │ Client developer    │
│ Load Balancer   │ 60s             │ DevOps              │
│ Nginx proxy     │ 60s / 300s      │ nginx.conf          │
│ Your app server │ No limit        │ Your code           │
│ Your response   │ 5MB default     │ Your code           │
└─────────────────────────────────────────────────────────┘
```

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant N as Nginx
    participant A as App Server

    C->>LB: Request (client timeout: 30s)
    LB->>N: Forward (LB timeout: 60s)
    N->>A: Forward (nginx timeout: 60s)
    A->>A: Process (takes 25s)
    A->>N: Response (success! 200)
    N->>LB: Forward response
    Note over C,LB: Client already timed out at 30s!
    LB--xC: Connection closed by client
    Note over C: Client sees: "Request timed out"
    Note over A: Server sees: "200 OK sent successfully"
```

### Code Example — Debugging Timeout Issues

```python
import time
import asyncio
from fastapi import FastAPI, Request, Response
from fastapi.middleware.gzip import GZipMiddleware

app = FastAPI()

@app.middleware("http")
async def timeout_debugging_middleware(request: Request, call_next):
    start = time.time()
    
    try:
        # Set server-side timeout to catch slow requests before proxy does
        response = await asyncio.wait_for(
            call_next(request),
            timeout=25.0  # Should be less than nginx/LB timeout
        )
        
        duration = time.time() - start
        
        # Log slow requests for analysis
        if duration > 5.0:
            print(f"SLOW: {request.method} {request.url.path} took {duration:.2f}s")
        
        response.headers["X-Response-Time"] = f"{duration:.3f}s"
        return response
        
    except asyncio.TimeoutError:
        duration = time.time() - start
        print(f"TIMEOUT: {request.method} {request.url.path} after {duration:.2f}s")
        return Response(
            content='{"error": "Request timed out"}',
            status_code=504,
            media_type="application/json"
        )

# Nginx configuration to check:
"""
# /etc/nginx/nginx.conf

upstream app {
    server localhost:8000;
    keepalive 32;
}

server {
    location /api/ {
        proxy_pass http://app;
        
        # These need to match your app's max processing time
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;    # ← Check this!
        proxy_send_timeout 30s;    # ← And this!
        
        # For large responses
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # For streaming responses (SSE, WebSocket)
        # proxy_buffering off;
    }
}
"""

# Check: Is response size causing the delay?
@app.get("/debug/response-size")
async def check_response_size(request: Request):
    """Returns info about response size limitations"""
    return {
        "client_ip": request.client.host,
        "headers": dict(request.headers),
        "note": "Check if response body > nginx proxy_buffer_size"
    }
```

### Key Takeaways
> - 💡 **"Server succeeded" doesn't mean client received it** — there are multiple layers
> - 💡 **Check nginx/LB timeout configs** — often set to 60s by default, clients may be 30s
> - 💡 **Large responses get buffered** — a 10MB response may stream slowly through proxies
> - 💡 **Add X-Response-Time header** to responses so clients can see actual server time
> - 💡 **For streaming endpoints**: set `proxy_buffering off` in nginx

---

## Question 13 — API Sometimes Returns Partial JSON Responses
🔴 Senior | ★★☆ Common

### The Scenario
> *"Users report that sometimes API responses are cut off in the middle — they get partial JSON like `{"users": [{"id": 1, "name": "Jo`. How do you investigate and fix this?"*

### The Answer

**Possible causes:**

```
PARTIAL JSON CAUSES:

1. Connection dropped mid-stream
   Client ──► Server [generating response...]
   ╳ Network drops here
   Client gets: {"users": [{"id": 1...  (incomplete)

2. Memory error during serialization
   Large object → JSONEncoder → Out of Memory
   Result: truncated output

3. Timeout during response streaming
   Nginx timeout fires at 30s
   Response was 60% generated
   
4. Response buffer issue
   Uvicorn/Gunicorn flushes partial buffer on error
```

### Code Example — Response Validation Middleware

```python
import json
from fastapi import FastAPI, Request, Response
from fastapi.responses import JSONResponse

app = FastAPI()

@app.middleware("http")
async def response_validation_middleware(request: Request, call_next):
    """
    Intercept response and validate it's complete JSON before sending.
    WARNING: This buffers entire response — only use for debugging!
    """
    response = await call_next(request)
    
    if response.headers.get("content-type", "").startswith("application/json"):
        # Buffer the response body
        body = b""
        async for chunk in response.body_iterator:
            body += chunk
        
        # Validate JSON is complete
        try:
            json.loads(body)  # Will raise if incomplete
        except json.JSONDecodeError as e:
            print(f"INVALID JSON RESPONSE: {e}")
            print(f"Partial body: {body[:200]}")
            return JSONResponse(
                status_code=500,
                content={"error": "Response serialization failed"}
            )
        
        # Return validated response
        return Response(
            content=body,
            status_code=response.status_code,
            headers=dict(response.headers),
            media_type=response.media_type
        )
    
    return response

# Safer approach: Use Pydantic response models to catch serialization errors
from pydantic import BaseModel, ValidationError
from typing import List

class UserResponse(BaseModel):
    id: int
    name: str
    email: str
    
    class Config:
        # Raise on serialization errors instead of silently omitting
        populate_by_name = True

class UsersListResponse(BaseModel):
    users: List[UserResponse]
    total: int

@app.get("/users", response_model=UsersListResponse)
async def list_users():
    """FastAPI validates response against model before sending"""
    try:
        users = await fetch_users_from_db()
        return UsersListResponse(users=users, total=len(users))
    except ValidationError as e:
        # Log the specific field that failed serialization
        print(f"Serialization error: {e}")
        raise HTTPException(500, "Data serialization failed")

async def fetch_users_from_db():
    # Return sample data
    return [{"id": 1, "name": "John", "email": "john@example.com"}]
```

### Key Takeaways
> - 💡 **Use Pydantic response_model** — FastAPI validates before sending
> - 💡 **Check nginx timeout** vs your actual response generation time
> - 💡 **Monitor memory usage** — OOM during serialization truncates response
> - 💡 **Add Content-Length header** so clients can detect truncation
> - 💡 **For debugging**: temporarily log response size for every request

---

## Question 14 — New Version Causes 5% of Requests to Fail with 422
🟡 Mid | ★★★ Very Common

### The Scenario
> *"After deploying a new API version, 5% of requests start failing with 422 Unprocessable Entity. The other 95% work fine. How do you handle this?"*

### The Answer

**The 422 error means request validation failed — a breaking change was introduced:**

```
BREAKING CHANGE EXAMPLE:

Old API (v1):   POST /users  { "name": "John" }           ✅ Works
New API (v2):   POST /users  { "full_name": "John" }      ← renamed field!

Old clients sending { "name": "John" } → 422 Unprocessable Entity
```

```mermaid
flowchart TD
    A[422 errors detected in monitoring] --> B{What changed?}
    B -->|Field renamed| C[Make both fields valid\nDeprecate old one with warning]
    B -->|Field type changed| D[Accept both types\nCoerce automatically]
    B -->|Required field added| E[Make it optional with default]
    B -->|New validation rule| F[Validate and log\nbefore enforcing]
    C --> G[Deploy fix]
    D --> G
    E --> G
    F --> G
    G --> H[Monitor 422 rate drops to 0%]
```

### Code Example — Backward-Compatible API with Validation Logging

```python
from fastapi import FastAPI, Request
from pydantic import BaseModel, field_validator, model_validator
from typing import Optional
import logging

app = FastAPI()
logger = logging.getLogger(__name__)

# PROBLEM: This breaking change causes 422 for old clients
class UserCreateV2_BREAKING(BaseModel):
    full_name: str  # renamed from 'name' — breaks old clients!
    email: str

# SOLUTION: Accept both old and new field names
class UserCreate(BaseModel):
    # New field name
    full_name: Optional[str] = None
    # Old field name (deprecated but still accepted)
    name: Optional[str] = None
    email: str
    
    @model_validator(mode="after")
    def accept_old_field_names(self):
        """Handle renamed fields for backward compatibility"""
        if self.name and not self.full_name:
            # Accept old 'name' field, map to 'full_name'
            self.full_name = self.name
            logger.warning(
                "Deprecated field 'name' used. Please migrate to 'full_name'",
                extra={"field": "name", "new_field": "full_name"}
            )
        
        if not self.full_name:
            raise ValueError("Either 'full_name' or 'name' is required")
        
        return self

@app.post("/users")
async def create_user(user: UserCreate, request: Request):
    return {"id": 1, "full_name": user.full_name, "email": user.email}

# Log ALL 422 errors to understand what's failing
@app.exception_handler(422)
async def validation_error_handler(request: Request, exc):
    from fastapi.exceptions import RequestValidationError
    
    body = await request.body()
    logger.error(
        "Validation error",
        extra={
            "path": request.url.path,
            "method": request.method,
            "body": body.decode("utf-8", errors="replace")[:500],
            "errors": str(exc),
        }
    )
    
    # Return helpful error message
    from fastapi.responses import JSONResponse
    return JSONResponse(
        status_code=422,
        content={
            "error": "Validation failed",
            "details": str(exc),
            "hint": "Check API docs for required fields",
            "docs_url": "/docs"
        }
    )

# Gradual rollout with feature flags
VALIDATION_STRICT_MODE = False  # Enable gradually

class UserCreateWithGradualRollout(BaseModel):
    full_name: Optional[str] = None
    name: Optional[str] = None  # deprecated
    email: str
    
    @model_validator(mode="after")
    def validate_name_fields(self):
        if self.name and not self.full_name:
            self.full_name = self.name
            
            if VALIDATION_STRICT_MODE:
                raise ValueError("'name' is deprecated, use 'full_name'")
        
        if not self.full_name:
            raise ValueError("'full_name' is required")
        
        return self
```

### Key Takeaways
> - 💡 **Never rename required fields without deprecation period**
> - 💡 **Accept both old and new field names** during migration
> - 💡 **Log all 422 errors** to understand what clients are sending
> - 💡 **Use feature flags** for gradual enforcement of new validation rules
> - 💡 **Return helpful error messages** with migration hints in 422 responses

---

## Question 15 — API Crashes with Specific Unicode Characters
🟢 Junior | ★★☆ Common

### The Scenario
> *"Your API works perfectly with regular text but crashes or returns 500 errors when users enter emojis, Arabic text, or special characters like `\u0000`. How do you fix this?"*

### The Answer

### Code Example — Unicode Handling and Input Sanitization

```python
from fastapi import FastAPI
from pydantic import BaseModel, field_validator
import re
import unicodedata

app = FastAPI()

class UserInput(BaseModel):
    username: str
    bio: Optional[str] = None
    
    @field_validator("username")
    @classmethod
    def validate_username(cls, v: str) -> str:
        # Remove null bytes (crashes many databases)
        v = v.replace("\x00", "").replace("\u0000", "")
        
        # Normalize unicode (é can be represented 2 ways in unicode)
        v = unicodedata.normalize("NFC", v)
        
        # Remove control characters
        v = "".join(c for c in v if unicodedata.category(c) not in ["Cc", "Cf"])
        
        # Length check after cleaning
        if not v or len(v) < 2:
            raise ValueError("Username must be at least 2 characters")
        
        if len(v) > 50:
            raise ValueError("Username must be 50 characters or less")
        
        return v
    
    @field_validator("bio")
    @classmethod
    def validate_bio(cls, v: Optional[str]) -> Optional[str]:
        if v is None:
            return v
        
        # Clean null bytes
        v = v.replace("\x00", "")
        
        # Normalize
        v = unicodedata.normalize("NFC", v)
        
        # Allow emojis but limit length
        if len(v.encode("utf-8")) > 5000:  # 5000 bytes, not characters!
            raise ValueError("Bio too long (max 5000 bytes)")
        
        return v

@app.post("/users")
async def create_user(user: UserInput):
    return {"username": user.username, "bio": user.bio}

# Database configuration to support full Unicode
"""
# PostgreSQL: Use UTF-8 encoding (default in modern versions)
# MySQL/MariaDB: Use utf8mb4 (not utf8 which is incomplete!)

# SQLAlchemy connection string:
engine = create_engine(
    "mysql+pymysql://user:pass@host/db?charset=utf8mb4"
)

# For SQLite:
engine = create_engine("sqlite:///app.db")
# SQLite always uses UTF-8, no configuration needed
"""
```

### Key Takeaways
> - 💡 **Null bytes (`\x00`)** crash PostgreSQL — always strip them
> - 💡 **Use `utf8mb4` in MySQL** — regular `utf8` doesn't support emojis
> - 💡 **Normalize unicode** with `unicodedata.normalize("NFC", text)`
> - 💡 **Measure length in bytes, not characters** — "😀" is 4 bytes but 1 character
> - 💡 **Test with**: `\x00`, Arabic, Chinese, emojis, RTL text

---

## Question 16 — Connection Pool Exhausted During Peak Traffic
🟡 Mid | ★★★ Very Common

### The Scenario
> *"During peak traffic, your FastAPI app raises `asyncpg.exceptions.TooManyConnectionsError` or `sqlalchemy.exc.TimeoutError: QueuePool limit exceeded`. Users get 500 errors. How do you solve this?"*

### The Answer

```
CONNECTION POOL EXHAUSTION:

DB Max Connections: 100
Your app instances: 4
Pool size per instance: 30
Total possible connections: 4 × 30 = 120  ← EXCEEDS 100!

When peak traffic hits:
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│App 1    │  │App 2    │  │App 3    │  │App 4    │
│30 conns │  │30 conns │  │30 conns │  │30 conns │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
     └──────────────────┬──────────────────┘
                        │ 120 connections
                        ▼
                  ┌──────────┐
                  │PostgreSQL│  ← Max: 100 connections!
                  │ OVERLOAD │
                  └──────────┘
                  
SOLUTION: Pool size per instance = (DB max connections) / (num instances) - 5 buffer
= (100 / 4) - 5 = 20 per instance
```

```mermaid
stateDiagram-v2
    [*] --> Idle: Connection released
    Idle --> InUse: Request needs connection
    InUse --> Idle: Query complete, connection returned
    InUse --> Overflow: All pool connections busy
    Overflow --> Waiting: Queue request
    Waiting --> InUse: Connection becomes available
    Waiting --> Error: Queue timeout exceeded
    Error --> [*]: 503 returned to client
```

### Code Example — Connection Pool Configuration and Monitoring

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
from sqlalchemy import event, pool
from fastapi import FastAPI, Depends
import asyncio
import logging

logger = logging.getLogger(__name__)

# CORRECT pool configuration
engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    
    # Pool settings — tune based on your setup
    pool_size=10,           # Base pool size (permanent connections)
    max_overflow=20,        # Extra connections allowed during spikes
    pool_timeout=30,        # Wait up to 30s for a connection (then raise error)
    pool_recycle=3600,      # Recycle connections after 1 hour (prevents stale)
    pool_pre_ping=True,     # Test connection before using it (detects broken connections)
    
    echo=False,             # Set True only for debugging (logs all SQL)
)

AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

# Monitor pool usage
@event.listens_for(engine.sync_engine, "checkout")
def on_checkout(dbapi_conn, conn_record, conn_proxy):
    pool_status = engine.pool.status()
    logger.debug(f"Connection checkout: {pool_status}")

@event.listens_for(engine.sync_engine, "checkin")  
def on_checkin(dbapi_conn, conn_record):
    pass

async def get_db() -> AsyncSession:
    """Dependency: properly manage session lifecycle"""
    async with AsyncSessionLocal() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
        # Session is automatically closed on exit

app = FastAPI()

@app.get("/db/pool-status")
async def pool_status():
    """Monitor pool health"""
    sync_pool = engine.pool
    return {
        "pool_size": sync_pool.size(),
        "checked_out": sync_pool.checkedout(),
        "overflow": sync_pool.overflow(),
        "checked_in": sync_pool.checkedin(),
    }

# Add PgBouncer for production (connection pooler at DB level)
"""
# pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction  # Share connections across requests
max_client_conn = 1000   # PgBouncer accepts 1000 connections
default_pool_size = 20   # But only uses 20 actual DB connections
"""

# Handle pool exhaustion gracefully
from fastapi import HTTPException

@app.get("/users")
async def list_users(db: AsyncSession = Depends(get_db)):
    try:
        result = await db.execute(text("SELECT * FROM users LIMIT 100"))
        return result.fetchall()
    except Exception as e:
        if "QueuePool limit" in str(e) or "too many connections" in str(e).lower():
            raise HTTPException(
                status_code=503,
                detail="Service temporarily unavailable. Please retry.",
                headers={"Retry-After": "5"}
            )
        raise
```

### Key Takeaways
> - 💡 **Formula**: `pool_size = max_db_connections / num_app_instances - buffer`
> - 💡 **Use PgBouncer** in production — multiplexes thousands of app connections into few DB connections
> - 💡 **Always use `pool_pre_ping=True`** — detects broken connections before query fails
> - 💡 **Set `pool_timeout`** — better to get a fast error than hang forever
> - 💡 **Return 503 with Retry-After** when pool is exhausted

---

## Question 17 — Cascading Failures in API Chain A→B→C
🔴 Senior | ★★★ Very Common

### The Scenario
> *"Your API chain is: API-A calls API-B which calls API-C. API-C goes down. Instead of graceful degradation, everything crashes. How do you prevent cascading failures?"*

### The Answer

```
WITHOUT CIRCUIT BREAKER (cascade failure):

API-C goes down
    ↑
API-B waits 30s, then fails  ← thread is blocked for 30s!
    ↑                           All API-B threads get stuck
API-A waits 30s, then fails  ← API-A threads now all blocked
    ↑
ALL USERS: 500 errors after 30 second hang

WITH CIRCUIT BREAKER:

API-C goes down
    ↑
Circuit OPENS after 5 failures
    ↑
API-B: Circuit is open → returns fallback in <1ms  ← no wait!
    ↑
API-A: Gets fast response → returns degraded result
    ↑
Users: See degraded but working response
```

```mermaid
stateDiagram-v2
    [*] --> Closed
    Closed --> Open: failures >= threshold\n(5 failures in 60s)
    Open --> HalfOpen: reset_timeout elapsed\n(30 seconds)
    HalfOpen --> Closed: test request succeeds
    HalfOpen --> Open: test request fails
    
    Closed: CLOSED\nNormal operation
    Open: OPEN\nReturn fallback immediately
    HalfOpen: HALF-OPEN\nTest one request
```

### Code Example — Circuit Breaker Implementation

```python
import asyncio
import time
from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, Any, Optional
import httpx
from fastapi import FastAPI

app = FastAPI()

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

@dataclass
class CircuitBreaker:
    """Production-ready circuit breaker"""
    name: str
    failure_threshold: int = 5      # Open after 5 failures
    success_threshold: int = 2      # Close after 2 successes in half-open
    timeout: float = 30.0           # Seconds before trying again
    
    _failures: int = field(default=0, init=False)
    _successes: int = field(default=0, init=False)
    _state: CircuitState = field(default=CircuitState.CLOSED, init=False)
    _last_failure_time: float = field(default=0.0, init=False)
    
    @property
    def state(self) -> CircuitState:
        if self._state == CircuitState.OPEN:
            if time.time() - self._last_failure_time >= self.timeout:
                self._state = CircuitState.HALF_OPEN
                self._successes = 0
        return self._state
    
    def call_succeeded(self):
        self._failures = 0
        if self._state == CircuitState.HALF_OPEN:
            self._successes += 1
            if self._successes >= self.success_threshold:
                self._state = CircuitState.CLOSED
                print(f"Circuit {self.name}: CLOSED (recovered)")
    
    def call_failed(self):
        self._failures += 1
        self._last_failure_time = time.time()
        if self._failures >= self.failure_threshold:
            if self._state != CircuitState.OPEN:
                print(f"Circuit {self.name}: OPENED (too many failures)")
            self._state = CircuitState.OPEN
    
    def can_execute(self) -> bool:
        return self.state != CircuitState.OPEN
    
    async def execute(self, func: Callable, *args, fallback=None, **kwargs) -> Any:
        """Execute function with circuit breaker protection"""
        if not self.can_execute():
            print(f"Circuit {self.name} is OPEN — using fallback")
            if fallback is not None:
                return fallback() if callable(fallback) else fallback
            raise Exception(f"Circuit breaker {self.name} is open")
        
        try:
            result = await func(*args, **kwargs)
            self.call_succeeded()
            return result
        except Exception as e:
            self.call_failed()
            if fallback is not None:
                return fallback() if callable(fallback) else fallback
            raise

# Register circuit breakers
breakers = {
    "service_b": CircuitBreaker("service_b", failure_threshold=5, timeout=30),
    "service_c": CircuitBreaker("service_c", failure_threshold=3, timeout=60),
}

async def call_service_b(data: dict) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://service-b/process",
            json=data,
            timeout=5.0  # Always set timeout!
        )
        response.raise_for_status()
        return response.json()

async def call_service_c(data: dict) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            "http://service-c/enrich",
            json=data,
            timeout=3.0
        )
        response.raise_for_status()
        return response.json()

@app.post("/api-a/process")
async def api_a_process(data: dict):
    """API-A with circuit breakers and fallbacks for B and C"""
    
    # Call Service B with circuit breaker
    b_result = await breakers["service_b"].execute(
        call_service_b,
        data,
        fallback={"processed": False, "reason": "service_b_unavailable"}
    )
    
    # Call Service C with circuit breaker
    c_result = await breakers["service_c"].execute(
        call_service_c,
        b_result,
        fallback={"enriched": False, "data": b_result}
    )
    
    return {
        "result": c_result,
        "degraded": not b_result.get("processed") or not c_result.get("enriched"),
        "circuit_states": {
            name: cb.state.value for name, cb in breakers.items()
        }
    }

@app.get("/circuit-breakers/status")
async def circuit_status():
    """Monitor circuit breaker states"""
    return {
        name: {
            "state": cb.state.value,
            "failures": cb._failures,
        }
        for name, cb in breakers.items()
    }
```

### Key Takeaways
> - 💡 **Circuit breaker prevents thread exhaustion** — fail fast instead of waiting
> - 💡 **Three states**: Closed (normal) → Open (failing, use fallback) → Half-Open (testing)
> - 💡 **Always provide fallbacks** for non-critical downstream services
> - 💡 **Always set HTTP timeouts** — never call external services without timeout
> - 💡 **Expose circuit state** via health endpoint for monitoring

---

## Question 18 — Users Report Data Inconsistency / Stale Data
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Users report they update their profile, but when they refresh, they see the old data. This happens randomly. What's wrong and how do you fix it?"*

### The Answer

**Most common causes:**

```
CAUSE 1: Read replica lag
  Write → Primary DB ──(replication lag 100ms-2s)──► Replica DB
  User writes to primary, immediately reads from replica → stale!

CAUSE 2: Cache not invalidated
  User updates DB → Cache still has old value
  Next request hits cache → returns old data

CAUSE 3: Load balancer sending requests to different nodes
  Node 1: has cache {"user": "John"}
  Node 2: has cache {"user": "Jane"} (just updated)
  User gets different responses depending on which node!
```

### Code Example — Cache Invalidation and Read-After-Write Consistency

```python
import asyncio
import redis.asyncio as redis_async
from fastapi import FastAPI, Depends

app = FastAPI()
redis_client = redis_async.Redis(host="localhost", port=6379, decode_responses=True)

class UserRepository:
    
    async def get_user(self, user_id: int, use_cache: bool = True) -> dict:
        cache_key = f"user:{user_id}"
        
        if use_cache:
            cached = await redis_client.get(cache_key)
            if cached:
                import json
                return json.loads(cached)
        
        # Fetch from database (primary, not replica!)
        user = await self._fetch_from_primary_db(user_id)
        
        # Cache result
        import json
        await redis_client.setex(cache_key, 300, json.dumps(user))
        return user
    
    async def update_user(self, user_id: int, data: dict) -> dict:
        # 1. Write to primary DB
        updated_user = await self._update_in_db(user_id, data)
        
        # 2. IMMEDIATELY invalidate cache
        cache_key = f"user:{user_id}"
        await redis_client.delete(cache_key)
        
        # 3. For read-after-write consistency:
        #    Write the new value directly to cache
        import json
        await redis_client.setex(cache_key, 300, json.dumps(updated_user))
        
        return updated_user
    
    async def _fetch_from_primary_db(self, user_id: int) -> dict:
        # IMPORTANT: Always read from PRIMARY after write, not replica
        # In SQLAlchemy: use the write engine for immediate reads
        return {"id": user_id, "name": "John", "email": "john@example.com"}
    
    async def _update_in_db(self, user_id: int, data: dict) -> dict:
        return {"id": user_id, **data}

user_repo = UserRepository()

@app.get("/users/{user_id}")
async def get_user(user_id: int, fresh: bool = False):
    """
    fresh=true forces reading from primary DB (bypasses cache)
    Use after write operations to ensure read-after-write consistency
    """
    return await user_repo.get_user(user_id, use_cache=not fresh)

@app.put("/users/{user_id}")
async def update_user(user_id: int, data: dict):
    updated = await user_repo.update_user(user_id, data)
    # Return the updated data directly so client doesn't need to re-fetch
    return updated
```

### Key Takeaways
> - 💡 **Invalidate cache on write** — the most common cause of stale data
> - 💡 **Write the new value to cache** after invalidating (avoid stampede)
> - 💡 **Read from primary after writes** — replicas can lag 100ms-2s
> - 💡 **Use ETag/If-None-Match** for optimistic concurrency
> - 💡 **Return updated data in PUT response** — saves a round trip

---

## Question 19 — API Logs Are 50GB/Day
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your API generates 50GB of logs per day, making it impossible to search through them when debugging. How do you improve your logging strategy?"*

### The Answer

```
LOGGING BEST PRACTICES:

BEFORE (unstructured, verbose):
[2024-01-01 10:00:00] INFO Starting request
[2024-01-01 10:00:00] DEBUG DB query: SELECT * FROM users WHERE id = 1
[2024-01-01 10:00:00] DEBUG DB result: (1, 'John', 'john@example.com', ...)
[2024-01-01 10:00:00] INFO Request completed

AFTER (structured, filtered):
{"level":"INFO","req_id":"abc","path":"/users/1","duration_ms":45,"status":200}
(Only log what matters — filter DEBUG in production)
```

### Code Example — Structured Logging with Sampling

```python
import logging
import json
import random
import time
from fastapi import FastAPI, Request

app = FastAPI()

class SampledLogger:
    """Logger that samples high-frequency events"""
    
    def __init__(self, sample_rate: float = 1.0):
        self.sample_rate = sample_rate
        self.logger = logging.getLogger("api")
    
    def should_log(self) -> bool:
        return random.random() < self.sample_rate
    
    def info(self, event: str, **kwargs):
        if self.should_log():
            self.logger.info(json.dumps({"event": event, **kwargs}))
    
    def error(self, event: str, **kwargs):
        # Always log errors, never sample them
        self.logger.error(json.dumps({"event": event, **kwargs}))
    
    def warning(self, event: str, **kwargs):
        # Always log warnings
        self.logger.warning(json.dumps({"event": event, **kwargs}))

# Sample 10% of INFO logs to reduce volume by 90%
logger = SampledLogger(sample_rate=0.1)

@app.middleware("http")
async def structured_logging_middleware(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration_ms = int((time.time() - start) * 1000)
    
    # Only log slow requests + errors + sampled subset
    should_log = (
        response.status_code >= 400 or  # Always log errors
        duration_ms > 1000 or           # Always log slow requests
        random.random() < 0.1           # Sample 10% of normal requests
    )
    
    if should_log:
        logger.info(
            "request",
            path=request.url.path,
            method=request.method,
            status=response.status_code,
            duration_ms=duration_ms,
        )
    
    return response
```

### Key Takeaways
> - 💡 **Structured JSON logging** makes querying logs 100x faster
> - 💡 **Sample 10% of INFO logs** in production — errors are always logged
> - 💡 **Use log levels properly**: DEBUG (dev only), INFO (sampled), WARNING/ERROR (always)
> - 💡 **ELK Stack (Elasticsearch + Logstash + Kibana)** for log aggregation
> - 💡 **Log rotation**: keep 7 days locally, archive to S3 for longer retention

---

## Question 20 — Memory Leak Causing Crashes After Hours
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your API works fine at startup but after 6-8 hours, memory usage is at 100% and the server crashes. How do you find and fix the memory leak?"*

### The Answer

```
MEMORY LEAK HUNTING APPROACH:

1. Confirm it's a leak (not just normal growth):
   Watch RSS memory over time:
   
   Hour 0:  200MB ─────────────────────────────────
   Hour 2:  350MB ─────────────────────────────────────────
   Hour 4:  500MB ─────────────────────────────────────────────────
   Hour 6:  700MB ──────────────────────────────────────────────────────
   Hour 8: CRASH (OOM)
   
   This pattern = memory leak (consistent growth, never drops)

2. Common Python API memory leak causes:
   - Global lists/dicts that keep growing
   - Event listeners not removed
   - Circular references preventing GC
   - Large objects cached indefinitely
   - Database connections not closed
```

### Code Example — Memory Leak Detection

```python
import tracemalloc
import gc
import asyncio
from fastapi import FastAPI
from typing import Optional

app = FastAPI()

# Global variable that could be a memory leak!
# ❌ BAD: this list grows indefinitely
request_history = []  # Never cleaned up!

# ✅ GOOD: use bounded data structure
from collections import deque
request_history_safe = deque(maxlen=1000)  # Maximum 1000 items

@app.on_event("startup")
async def start_memory_tracking():
    tracemalloc.start(25)  # Keep 25 frames of traceback

@app.get("/debug/memory")
async def memory_snapshot():
    """Take memory snapshot for debugging"""
    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics("lineno")
    
    return {
        "top_memory_consumers": [
            {
                "file": str(stat.traceback),
                "size_kb": stat.size / 1024,
                "count": stat.count
            }
            for stat in top_stats[:10]
        ]
    }

@app.get("/debug/gc")
async def gc_stats():
    """Force garbage collection and report stats"""
    before = sum(len(gc.get_objects()))
    gc.collect()
    after = sum(len(gc.get_objects()))
    
    return {
        "objects_before_gc": before,
        "objects_after_gc": after,
        "collected": before - after,
        "gc_counts": gc.get_count(),
    }

# Common fix: ensure async context managers are used properly
class DatabaseManager:
    def __init__(self):
        self._connections = []  # ❌ Connections never closed!
    
    async def get_connection_LEAKY(self):
        conn = await create_db_connection()
        self._connections.append(conn)  # Grows forever!
        return conn
    
    async def get_connection_FIXED(self):
        """Use context manager to guarantee cleanup"""
        async with create_db_connection_context() as conn:
            yield conn
        # conn is automatically closed here

async def create_db_connection():
    pass  # Placeholder

async def create_db_connection_context():
    pass  # Placeholder
```

### Key Takeaways
> - 💡 **tracemalloc** shows exactly which line of code is allocating memory
> - 💡 **Common leaks**: global lists growing forever, event listeners not removed
> - 💡 **Use bounded structures**: `deque(maxlen=N)` instead of `list`
> - 💡 **Always use context managers** for database connections
> - 💡 **Monitor memory over time** with Prometheus gauge, alert at 80%

---

*Next: [03 — Security →](./03-security.md)*
