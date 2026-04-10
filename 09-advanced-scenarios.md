# 09 — Advanced Scenarios

> **Questions 71–90** | Real-world edge cases, integrations, and operational challenges

---

## Question 71 — Handle Duplicate Webhook Deliveries from Stripe/PayPal
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API receives webhooks from Stripe for payment events. Sometimes Stripe retries and sends the same webhook 2-3 times. How do you prevent processing duplicates?"*

### The Answer

**Step 1: Understand why duplicates happen**
- Webhook provider retries if it doesn't receive a timely 200 response
- Network issues can cause your server to process the request but fail to send the 200 back
- Provider may have multiple delivery attempts from different servers

**Step 2: Implement idempotent webhook handling**

| Strategy | How It Works |
|----------|-------------|
| **Store event ID** | Save webhook event ID in DB before processing; skip if exists |
| **Acknowledge first** | Return 200 immediately, process asynchronously |
| **Idempotent operations** | Make your processing logic safe to run multiple times |

```
DUPLICATE WEBHOOK FLOW:

Stripe ──webhook (event_id: evt_001)──► Your API
                                         │
                                    Check: evt_001 in DB?
                                         │
                                    ┌─────┴─────┐
                                    │ No         │ Yes
                                    ▼            ▼
                              Save evt_001   Return 200
                              Process event  (already handled)
                              Return 200
```

```mermaid
flowchart TD
    A[Webhook Received] --> B[Verify Signature\nHMAC-SHA256]
    B -->|Invalid| C[Return 400 ❌]
    B -->|Valid| D{Event ID in DB?}
    D -->|Yes| E[Return 200\nAlready processed ✅]
    D -->|No| F[Insert event ID\nstatus = received]
    F --> G[Return 200 immediately]
    G --> H[Process event\nasynchronously]
    H --> I[Update status = processed]
```

### Code Example — Idempotent Webhook Handler

```python
import hmac
import hashlib
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from datetime import datetime

app = FastAPI()

WEBHOOK_SECRET = "whsec_your_stripe_secret"

# In production: use PostgreSQL with UNIQUE constraint on event_id
processed_events: dict[str, dict] = {}

def verify_stripe_signature(payload: bytes, sig_header: str, secret: str) -> bool:
    """Verify the webhook came from Stripe."""
    parts = dict(item.split("=", 1) for item in sig_header.split(","))
    timestamp = parts.get("t", "")
    signature = parts.get("v1", "")
    signed_payload = f"{timestamp}.{payload.decode()}".encode()
    expected = hmac.new(secret.encode(), signed_payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

async def process_webhook_event(event_id: str, event_type: str, data: dict):
    """Process the webhook event asynchronously."""
    try:
        if event_type == "payment_intent.succeeded":
            # Update order status, send confirmation email, etc.
            pass
        elif event_type == "payment_intent.payment_failed":
            # Notify user, update order status
            pass
        processed_events[event_id]["status"] = "processed"
        processed_events[event_id]["processed_at"] = datetime.utcnow().isoformat()
    except Exception as e:
        processed_events[event_id]["status"] = "failed"
        processed_events[event_id]["error"] = str(e)

@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request, background_tasks: BackgroundTasks):
    payload = await request.body()
    sig_header = request.headers.get("stripe-signature", "")

    # Step 1: Verify signature
    if not verify_stripe_signature(payload, sig_header, WEBHOOK_SECRET):
        raise HTTPException(400, "Invalid signature")

    event = await request.json()
    event_id = event["id"]

    # Step 2: Check for duplicate (idempotency)
    if event_id in processed_events:
        return {"status": "already_processed"}

    # Step 3: Record immediately (before processing)
    processed_events[event_id] = {
        "status": "received",
        "type": event["type"],
        "received_at": datetime.utcnow().isoformat()
    }

    # Step 4: Return 200 fast, process in background
    background_tasks.add_task(
        process_webhook_event, event_id, event["type"], event["data"]
    )

    return {"status": "accepted"}
```

### Key Takeaways
> - 💡 **Always store the event ID** before processing — this is your deduplication key
> - 💡 **Return 200 immediately** and process asynchronously — Stripe expects fast responses
> - 💡 **Verify the signature** (HMAC) to ensure the webhook is authentic
> - 💡 Use a **database with UNIQUE constraint** on event_id for production reliability
> - 💡 Design your processing to be **idempotent** — safe to run multiple times

---

## Question 72 — Users Complain Images Load Slowly — CDN Not Being Used
🟢 Junior | ★★☆ Common

### The Scenario
> *"Users report that images and static assets in your API responses load slowly. You have a CDN (CloudFront/Cloudflare) set up, but it's not being utilized. What's wrong and how do you fix it?"*

### The Answer

**Step 1: Common reasons CDN is bypassed**

| Problem | Symptom | Fix |
|---------|---------|-----|
| API returns direct S3/server URLs | URLs point to origin, not CDN | Generate CDN URLs instead |
| Cache-Control headers missing | CDN doesn't cache without them | Add proper cache headers |
| Query strings vary per request | Each unique URL = cache miss | Normalize or strip unnecessary params |
| POST requests for assets | POST is not cacheable | Use GET for static resources |
| Cookies included | CDN won't cache responses with cookies | Separate static from dynamic domains |

```
BEFORE (CDN bypassed):
Client ──GET image──► API Server ──fetch──► S3
   (slow, API server bottleneck)

AFTER (CDN used):
Client ──GET image──► CDN (CloudFront)
   │                     │
   │                  ┌──┴──┐
   │                  │Cache│ → Cache HIT: return immediately (fast!)
   │                  │Miss │ → Fetch from S3, cache, return
   │                  └─────┘
```

```mermaid
flowchart TD
    A[Client requests image] --> B{CDN has it cached?}
    B -->|Cache HIT| C[Return from CDN edge\n< 50ms ✅]
    B -->|Cache MISS| D[CDN fetches from origin S3]
    D --> E[CDN caches for TTL]
    E --> C
```

### Code Example — Proper CDN Integration

```python
from fastapi import FastAPI
from fastapi.responses import JSONResponse
import os

app = FastAPI()

CDN_BASE_URL = os.getenv("CDN_URL", "https://cdn.example.com")
S3_BUCKET = "my-assets-bucket"

@app.get("/products/{product_id}")
async def get_product(product_id: int):
    product = {
        "id": product_id,
        "name": "Widget Pro",
        # ❌ WRONG: Direct S3 URL (bypasses CDN)
        # "image_url": f"https://{S3_BUCKET}.s3.amazonaws.com/products/{product_id}.jpg",

        # ✅ CORRECT: CDN URL
        "image_url": f"{CDN_BASE_URL}/products/{product_id}.jpg",
        "thumbnail_url": f"{CDN_BASE_URL}/products/{product_id}_thumb.jpg",
    }
    return product

@app.get("/assets/{file_path:path}")
async def serve_asset(file_path: str):
    """Serve assets with proper cache headers for CDN."""
    content = await fetch_from_storage(file_path)

    return JSONResponse(
        content=content,
        headers={
            # Tell CDN to cache for 1 year (immutable assets with hash in filename)
            "Cache-Control": "public, max-age=31536000, immutable",
            # Or for mutable assets: cache 1 hour, revalidate
            # "Cache-Control": "public, max-age=3600, must-revalidate",
            "CDN-Cache-Control": "max-age=86400",  # CDN-specific TTL
        }
    )

async def fetch_from_storage(path: str):
    return {"url": f"{CDN_BASE_URL}/{path}"}
```

### Key Takeaways
> - 💡 Always return **CDN URLs** in API responses, never direct S3/origin URLs
> - 💡 Set proper `Cache-Control` headers — without them, CDN won't cache
> - 💡 Use **immutable filenames** (hash in URL) for long cache TTLs
> - 💡 Separate static asset domains from API domains to avoid cookie issues
> - 💡 Monitor CDN **cache hit ratio** — should be >90% for static assets

---

## Question 73 — API Needs to Support Both JSON and XML Response Formats
🟢 Junior | ★★☆ Common

### The Scenario
> *"Your API returns JSON, but a legacy enterprise client requires XML responses. How do you support multiple response formats without duplicating your endpoints?"*

### The Answer

**Step 1: Use content negotiation via the Accept header**

| Client Sends | Server Returns |
|-------------|---------------|
| `Accept: application/json` | JSON response |
| `Accept: application/xml` | XML response |
| `Accept: */*` or no header | JSON (default) |
| `Accept: text/html` | 406 Not Acceptable |

```
REQUEST:
GET /users/1 HTTP/1.1
Accept: application/xml

RESPONSE:
HTTP/1.1 200 OK
Content-Type: application/xml

<?xml version="1.0"?>
<user>
  <id>1</id>
  <name>Alice</name>
</user>
```

```mermaid
flowchart TD
    A[Client Request] --> B{Accept header?}
    B -->|application/json| C[Serialize to JSON]
    B -->|application/xml| D[Serialize to XML]
    B -->|none or */*| C
    B -->|unsupported| E[406 Not Acceptable]
    C --> F[Return with Content-Type: application/json]
    D --> G[Return with Content-Type: application/xml]
```

### Code Example — Multi-Format API Response

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse, Response
import dicttoxml
from pydantic import BaseModel

app = FastAPI()

class User(BaseModel):
    id: int
    name: str
    email: str

def to_xml_response(data: dict) -> Response:
    """Convert dict to XML response."""
    xml_bytes = dicttoxml.dicttoxml(data, custom_root="response", attr_type=False)
    return Response(content=xml_bytes, media_type="application/xml")

def negotiate_response(request: Request, data: dict) -> Response:
    """Return response in format requested by Accept header."""
    accept = request.headers.get("accept", "application/json")
    if "application/xml" in accept:
        return to_xml_response(data)
    elif "application/json" in accept or "*/*" in accept:
        return JSONResponse(content=data)
    else:
        return JSONResponse(
            status_code=406,
            content={"error": "Not Acceptable", "supported": ["application/json", "application/xml"]}
        )

@app.get("/users/{user_id}")
async def get_user(user_id: int, request: Request):
    user = {"id": user_id, "name": "Alice", "email": "alice@example.com"}
    return negotiate_response(request, user)

@app.get("/products")
async def list_products(request: Request):
    products = [
        {"id": 1, "name": "Widget", "price": 9.99},
        {"id": 2, "name": "Gadget", "price": 19.99},
    ]
    return negotiate_response(request, {"products": products})
```

### Key Takeaways
> - 💡 Use the `Accept` header for **content negotiation** — don't create separate endpoints
> - 💡 Default to **JSON** when no Accept header is provided
> - 💡 Return `406 Not Acceptable` for unsupported formats
> - 💡 Always set the `Content-Type` response header to match the actual format
> - 💡 Use libraries like `dicttoxml` for XML serialization — don't build XML strings manually

---

## Question 74 — Throttle a Specific Abusive Client Without Blocking Others
🟡 Mid | ★★☆ Common

### The Scenario
> *"One client is making 10x more requests than normal, degrading performance for everyone. You need to throttle just this client without affecting other users. How?"*

### The Answer

**Step 1: Identify the abusive client**

| Identification Method | Example |
|----------------------|---------|
| API key | `X-API-Key: abc123` |
| JWT user ID | `sub` claim in token |
| IP address | `request.client.host` |
| Client fingerprint | Combination of headers + IP |

**Step 2: Apply per-client rate limits**

```
NORMAL CLIENTS:     100 req/min → ✅ All fine
ABUSIVE CLIENT:     1000 req/min → ❌ Throttle to 100 req/min

STRATEGY:
┌──────────────────────────────────────────────┐
│  Request arrives                              │
│  │                                           │
│  ├─ Extract client ID (API key / user ID)    │
│  ├─ Check client-specific counter in Redis   │
│  │                                           │
│  ├─ Under limit? → Process normally          │
│  ├─ Over limit?  → 429 Too Many Requests     │
│  │                                           │
│  └─ Override list? → Apply custom limit      │
└──────────────────────────────────────────────┘
```

```mermaid
flowchart TD
    A[Request Received] --> B[Extract Client ID\nAPI key or JWT sub]
    B --> C{On blocklist/throttle list?}
    C -->|Yes| D[Apply custom rate limit\ne.g., 10 req/min]
    C -->|No| E[Apply default rate limit\ne.g., 100 req/min]
    D --> F{Under limit?}
    E --> F
    F -->|Yes| G[Process Request ✅]
    F -->|No| H[429 Too Many Requests\nRetry-After header]
```

### Code Example — Per-Client Throttling

```python
import time
from fastapi import FastAPI, Request, HTTPException
from collections import defaultdict

app = FastAPI()

# Per-client request tracking (use Redis in production)
client_requests: dict[str, list[float]] = defaultdict(list)

# Custom limits for specific clients (configurable)
CUSTOM_LIMITS = {
    "client_abc123": 10,     # Throttled: 10 req/min
    "client_known_bot": 5,   # Heavily throttled
}
DEFAULT_LIMIT = 100  # requests per minute

def get_client_id(request: Request) -> str:
    """Extract client identifier from request."""
    api_key = request.headers.get("X-API-Key", "")
    if api_key:
        return f"key_{api_key}"
    return f"ip_{request.client.host}"

def get_rate_limit(client_id: str) -> int:
    """Get rate limit for this client (custom or default)."""
    return CUSTOM_LIMITS.get(client_id, DEFAULT_LIMIT)

@app.middleware("http")
async def per_client_rate_limit(request: Request, call_next):
    client_id = get_client_id(request)
    limit = get_rate_limit(client_id)
    now = time.time()
    window = 60  # 1 minute window

    # Clean old entries
    client_requests[client_id] = [
        t for t in client_requests[client_id] if t > now - window
    ]

    if len(client_requests[client_id]) >= limit:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded. Limit: {limit} requests per minute.",
            headers={
                "Retry-After": "60",
                "X-RateLimit-Limit": str(limit),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(now + window)),
            }
        )

    client_requests[client_id].append(now)
    response = await call_next(request)
    remaining = limit - len(client_requests[client_id])
    response.headers["X-RateLimit-Limit"] = str(limit)
    response.headers["X-RateLimit-Remaining"] = str(max(0, remaining))
    return response
```

### Key Takeaways
> - 💡 Identify clients by **API key**, **JWT user ID**, or **IP address**
> - 💡 Use **per-client counters** in Redis with sliding window algorithm
> - 💡 Maintain a **configurable override list** for throttling specific clients
> - 💡 Always return `429` with `Retry-After` and `X-RateLimit-*` headers
> - 💡 Consider a **tiered system**: warn at 80%, throttle at 100%, block at 200%

---

## Question 75 — Build an Audit Log for Every Data Change in Your API
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your company needs to track every data change (who changed what, when, and the before/after values) for compliance and debugging. How do you build an audit log system?"*

### The Answer

**Step 1: Decide what to log**

| Field | Purpose | Example |
|-------|---------|---------|
| `timestamp` | When the change happened | `2024-01-15T10:30:00Z` |
| `user_id` | Who made the change | `user_456` |
| `action` | What was done | `UPDATE`, `CREATE`, `DELETE` |
| `resource_type` | What type of resource | `order`, `user`, `product` |
| `resource_id` | Which specific resource | `order_123` |
| `changes` | Before/after values | `{"name": {"old": "Alice", "new": "Bob"}}` |
| `ip_address` | Where the request came from | `192.168.1.1` |
| `request_id` | Correlation ID | `req_abc123` |

```
AUDIT LOG ENTRY EXAMPLE:
┌───────────────────────────────────────────────────┐
│  timestamp:     2024-01-15T10:30:00Z              │
│  user_id:       user_456                          │
│  action:        UPDATE                            │
│  resource:      users/user_123                    │
│  changes:                                         │
│    name:        "Alice" → "Alice Smith"           │
│    email:       "alice@old.com" → "alice@new.com" │
│  ip_address:    203.0.113.42                      │
│  request_id:    req_abc123                        │
└───────────────────────────────────────────────────┘
```

```mermaid
flowchart TD
    A[API Request\nPUT /users/123] --> B[Middleware captures\nuser, IP, request_id]
    B --> C[Load current state\nfrom DB before]
    C --> D[Apply changes\nto database]
    D --> E[Compute diff\nold vs new values]
    E --> F[Write audit log\nasynchronously]
    F --> G[Audit Log Store\nPostgreSQL / Elasticsearch]
```

### Code Example — Audit Log Middleware in FastAPI

```python
import uuid
import json
from datetime import datetime, timezone
from fastapi import FastAPI, Request, Depends
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# Audit log storage (use PostgreSQL/Elasticsearch in production)
audit_logs: list[dict] = []

class AuditEntry(BaseModel):
    timestamp: str
    request_id: str
    user_id: str
    action: str
    resource_type: str
    resource_id: str
    changes: dict
    ip_address: str

def compute_diff(old: dict, new: dict) -> dict:
    """Compute the difference between old and new states."""
    changes = {}
    all_keys = set(list(old.keys()) + list(new.keys()))
    for key in all_keys:
        old_val = old.get(key)
        new_val = new.get(key)
        if old_val != new_val:
            changes[key] = {"old": old_val, "new": new_val}
    return changes

def log_audit(
    request_id: str, user_id: str, action: str,
    resource_type: str, resource_id: str,
    changes: dict, ip_address: str
):
    """Record an audit log entry."""
    entry = AuditEntry(
        timestamp=datetime.now(timezone.utc).isoformat(),
        request_id=request_id,
        user_id=user_id,
        action=action,
        resource_type=resource_type,
        resource_id=resource_id,
        changes=changes,
        ip_address=ip_address,
    )
    audit_logs.append(entry.model_dump())

# In-memory database
users_db = {
    "user_123": {"name": "Alice", "email": "alice@example.com", "role": "admin"}
}

@app.put("/users/{user_id}")
async def update_user(user_id: str, request: Request):
    body = await request.json()
    request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
    auth_user = request.headers.get("X-User-ID", "anonymous")

    if user_id not in users_db:
        return {"error": "User not found"}, 404

    # Capture before state
    old_state = users_db[user_id].copy()

    # Apply changes
    users_db[user_id].update(body)

    # Compute and log diff
    changes = compute_diff(old_state, users_db[user_id])
    if changes:
        log_audit(
            request_id=request_id,
            user_id=auth_user,
            action="UPDATE",
            resource_type="user",
            resource_id=user_id,
            changes=changes,
            ip_address=request.client.host,
        )

    return {"user": users_db[user_id], "audit": {"changes_logged": len(changes)}}

@app.get("/audit-logs")
async def get_audit_logs(resource_type: str = None, resource_id: str = None):
    """Query audit logs with optional filters."""
    results = audit_logs
    if resource_type:
        results = [l for l in results if l["resource_type"] == resource_type]
    if resource_id:
        results = [l for l in results if l["resource_id"] == resource_id]
    return {"logs": results, "total": len(results)}
```

### Key Takeaways
> - 💡 Audit logs should capture **who, what, when, and before/after values**
> - 💡 Write audit logs **asynchronously** to avoid slowing down the main request
> - 💡 Store in **append-only** storage — audit logs should never be editable or deletable
> - 💡 Include a **request_id** for correlation with application logs
> - 💡 For compliance (HIPAA, SOX), audit logs may need tamper-proof storage (e.g., AWS QLDB)

---

## Question 76 — Manage API Keys for 3rd Party Developers
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API is now used by third-party developers. You need to implement API key management — issuing, revoking, rotating, and rate limiting per key. How do you design this?"*

### The Answer

**Step 1: API key lifecycle**

```
CREATE → ACTIVE → ROTATE → REVOKE → EXPIRED

Developer Portal:
┌─────────────────────────────────────────────┐
│  My API Keys                                │
│                                             │
│  Production: sk_live_abc123... (Active)     │
│  Created: Jan 15, 2024                      │
│  Last used: 2 hours ago                     │
│  Rate limit: 1000 req/min                   │
│  [Rotate] [Revoke]                          │
│                                             │
│  Staging: sk_test_def456... (Active)        │
│  Created: Jan 10, 2024                      │
│  [Rotate] [Revoke]                          │
│                                             │
│  [+ Create New Key]                         │
└─────────────────────────────────────────────┘
```

```mermaid
flowchart TD
    A[Developer Registers] --> B[Generate API Key\nsk_live_ + random]
    B --> C[Store hashed key in DB]
    C --> D[Developer uses key\nin X-API-Key header]
    D --> E[API validates key\nhash and compare]
    E --> F{Key valid?}
    F -->|Yes| G[Check rate limit\nfor this key's tier]
    F -->|No| H[401 Unauthorized]
    G -->|Under limit| I[Process request ✅]
    G -->|Over limit| J[429 Rate Limited]
```

### Code Example — API Key Management System

```python
import secrets
import hashlib
from datetime import datetime, timezone
from fastapi import FastAPI, Request, HTTPException, Depends
from pydantic import BaseModel

app = FastAPI()

# Database (use PostgreSQL in production)
api_keys_db: dict[str, dict] = {}

class APIKeyCreate(BaseModel):
    name: str
    tier: str = "free"  # free, pro, enterprise

def generate_api_key(prefix: str = "sk_live") -> str:
    """Generate a secure API key with prefix."""
    random_part = secrets.token_urlsafe(32)
    return f"{prefix}_{random_part}"

def hash_key(key: str) -> str:
    """Hash API key for storage (never store raw keys)."""
    return hashlib.sha256(key.encode()).hexdigest()

TIER_LIMITS = {"free": 100, "pro": 1000, "enterprise": 10000}

@app.post("/developer/api-keys")
async def create_api_key(key_request: APIKeyCreate):
    raw_key = generate_api_key()
    key_hash = hash_key(raw_key)

    api_keys_db[key_hash] = {
        "name": key_request.name,
        "tier": key_request.tier,
        "created_at": datetime.now(timezone.utc).isoformat(),
        "is_active": True,
        "rate_limit": TIER_LIMITS.get(key_request.tier, 100),
        "last_used": None,
    }

    # Return raw key ONLY on creation — never again
    return {
        "api_key": raw_key,
        "message": "Save this key securely. It won't be shown again.",
        "tier": key_request.tier,
        "rate_limit": f"{TIER_LIMITS[key_request.tier]} req/min"
    }

async def validate_api_key(request: Request) -> dict:
    """Dependency to validate API key from header."""
    raw_key = request.headers.get("X-API-Key")
    if not raw_key:
        raise HTTPException(401, "X-API-Key header required")

    key_hash = hash_key(raw_key)
    key_data = api_keys_db.get(key_hash)

    if not key_data:
        raise HTTPException(401, "Invalid API key")
    if not key_data["is_active"]:
        raise HTTPException(403, "API key has been revoked")

    key_data["last_used"] = datetime.now(timezone.utc).isoformat()
    return key_data

@app.get("/data")
async def get_data(key_info: dict = Depends(validate_api_key)):
    return {"data": "protected content", "your_tier": key_info["tier"]}

@app.post("/developer/api-keys/{key_prefix}/revoke")
async def revoke_key(key_prefix: str):
    """Revoke an API key by marking it inactive."""
    for key_hash, data in api_keys_db.items():
        if data["name"] == key_prefix:
            data["is_active"] = False
            return {"status": "revoked"}
    raise HTTPException(404, "Key not found")
```

### Key Takeaways
> - 💡 **Never store raw API keys** — store hashed values, show raw key only once at creation
> - 💡 Use **prefixes** (`sk_live_`, `sk_test_`) to distinguish key types and environments
> - 💡 Implement **per-key rate limits** based on tier (free/pro/enterprise)
> - 💡 Support **key rotation** — allow two active keys during transition period
> - 💡 Log all API key usage for **abuse detection** and billing

---

## Question 77 — Isolate a Slow Downstream Microservice During Black Friday
🔴 Senior | ★★★ Very Common

### The Scenario
> *"During Black Friday, your recommendation service responds in 5 seconds instead of 200ms. This is slowing down your entire checkout flow. How do you isolate this service to prevent it from bringing everything down?"*

### The Answer

**Step 1: Apply the Bulkhead Pattern**

The **bulkhead pattern** isolates components so a failure in one doesn't cascade. Named after watertight compartments in ships.

```
WITHOUT BULKHEAD:
┌───────────────────────────────────────┐
│           Shared Thread Pool          │
│  [payment] [payment] [recommend]     │
│  [recommend] [recommend] [payment]   │
│  [recommend] [recommend] [recommend] │ ← All threads waiting on slow service!
│  No threads left for payments! 💥     │
└───────────────────────────────────────┘

WITH BULKHEAD:
┌────────────────┐  ┌────────────────────┐
│ Payment Pool   │  │ Recommendation Pool│
│ [pay] [pay]    │  │ [rec] [rec] [rec]  │
│ [pay] [pay]    │  │ (all stuck, but    │
│ (works fine!)  │  │  isolated to 5     │
│                │  │  threads max)      │
└────────────────┘  └────────────────────┘
Payments unaffected ✅  Recommendations degrade gracefully
```

```mermaid
flowchart TD
    A[Checkout Request] --> B{Parallel calls with timeouts}
    B --> C[Payment Service\n200ms timeout\nCRITICAL]
    B --> D[Recommendation Service\n500ms timeout\nOPTIONAL]
    B --> E[Inventory Service\n300ms timeout\nCRITICAL]
    C -->|Success| F[Continue checkout]
    D -->|Timeout!| G[Use cached recommendations\nFallback ✅]
    E -->|Success| F
    F --> H[Checkout complete\nwithout recommendations]
```

### Code Example — Bulkhead + Circuit Breaker + Timeout

```python
import asyncio
from fastapi import FastAPI
from enum import Enum

app = FastAPI()

class ServicePriority(Enum):
    CRITICAL = "critical"   # Must succeed (payment, inventory)
    OPTIONAL = "optional"   # Can skip (recommendations, analytics)

# Isolated semaphores per service (Bulkhead)
service_semaphores = {
    "payment": asyncio.Semaphore(20),        # Max 20 concurrent
    "inventory": asyncio.Semaphore(15),       # Max 15 concurrent
    "recommendation": asyncio.Semaphore(5),   # Max 5 concurrent (isolated!)
}

async def call_with_bulkhead(
    service_name: str,
    coro,
    timeout: float,
    priority: ServicePriority,
    fallback=None
):
    """Call service with bulkhead isolation, timeout, and fallback."""
    semaphore = service_semaphores.get(service_name)

    try:
        # Bulkhead: limit concurrent calls to this service
        async with semaphore:
            # Timeout: don't wait forever
            result = await asyncio.wait_for(coro, timeout=timeout)
            return result
    except asyncio.TimeoutError:
        if priority == ServicePriority.OPTIONAL and fallback is not None:
            return fallback  # Graceful degradation
        raise
    except Exception:
        if priority == ServicePriority.OPTIONAL and fallback is not None:
            return fallback
        raise

@app.post("/checkout")
async def checkout(order: dict):
    """Checkout with isolated service calls."""
    payment_task = call_with_bulkhead(
        "payment",
        process_payment(order),
        timeout=2.0,
        priority=ServicePriority.CRITICAL
    )
    inventory_task = call_with_bulkhead(
        "inventory",
        reserve_inventory(order),
        timeout=1.0,
        priority=ServicePriority.CRITICAL
    )
    recommendation_task = call_with_bulkhead(
        "recommendation",
        get_recommendations(order),
        timeout=0.5,
        priority=ServicePriority.OPTIONAL,
        fallback=[]  # Empty recommendations if service is slow
    )

    payment, inventory, recommendations = await asyncio.gather(
        payment_task, inventory_task, recommendation_task,
        return_exceptions=True
    )

    # Check critical services
    for result in [payment, inventory]:
        if isinstance(result, Exception):
            raise result  # Critical failure — abort checkout

    return {
        "order_status": "confirmed",
        "payment": payment,
        "recommendations": recommendations if not isinstance(recommendations, Exception) else []
    }

# Service stubs
async def process_payment(order):
    await asyncio.sleep(0.1)
    return {"status": "charged"}

async def reserve_inventory(order):
    await asyncio.sleep(0.05)
    return {"status": "reserved"}

async def get_recommendations(order):
    await asyncio.sleep(5)  # Slow during Black Friday!
    return [{"product": "Widget Pro"}]
```

### Key Takeaways
> - 💡 **Bulkhead pattern** = isolate service calls into separate thread/connection pools
> - 💡 Classify services as **CRITICAL** (must succeed) vs **OPTIONAL** (can degrade)
> - 💡 Set **aggressive timeouts** for optional services (500ms max during peak)
> - 💡 Always have **fallback responses** for optional services (cached data, empty list)
> - 💡 Combine with **circuit breaker** — after N timeouts, stop calling the slow service entirely

---

## Question 78 — Search API Returns Different Results for Same Query Intermittently
🟡 Mid | ★★☆ Common

### The Scenario
> *"Users report that searching for the same term sometimes returns different results. The results seem random — sometimes all results appear, sometimes only half. What could cause this?"*

### The Answer

**Step 1: Common causes of inconsistent search results**

| Cause | Symptom | Fix |
|-------|---------|-----|
| **Multiple replicas out of sync** | Different servers return different data | Check replication lag; use `preference=_primary` |
| **Caching inconsistency** | Stale cached results mixed with fresh | Normalize cache keys; invalidate properly |
| **Elasticsearch refresh interval** | Recently indexed docs not searchable | Reduce refresh interval or force refresh |
| **Race condition in indexing** | Data written to DB but not yet indexed | Use CDC (Change Data Capture) with guaranteed delivery |
| **Random scoring / tie-breaking** | Equal-score results ordered randomly | Add deterministic tie-breaker (e.g., `_id`) |

```
TYPICAL CAUSE — Replication Lag:

Write goes to: Primary DB ──(sync)──► Replica 1
                           ──(async)──► Replica 2 (2 seconds behind!)

Request 1 → Replica 1 → Returns updated results ✅
Request 2 → Replica 2 → Returns stale results ❌
Request 3 → Replica 1 → Returns updated results ✅
```

```mermaid
flowchart TD
    A[Search Request] --> B[Load Balancer]
    B --> C[Replica 1\nUp to date ✅]
    B --> D[Replica 2\n2 seconds behind ❌]
    C --> E[Returns 50 results]
    D --> F[Returns 48 results\nMissing latest 2]
    E --> G[User sees 50 results]
    F --> H[User sees 48 results\nInconsistent!]
```

### Code Example — Ensuring Consistent Search Results

```python
from fastapi import FastAPI, Query
from elasticsearch import AsyncElasticsearch
import hashlib

app = FastAPI()
es = AsyncElasticsearch(["http://localhost:9200"])

@app.get("/search")
async def search(
    q: str = Query(..., min_length=1),
    page: int = Query(1, ge=1),
    size: int = Query(20, ge=1, le=100),
):
    """Search with consistent results."""
    body = {
        "query": {
            "multi_match": {
                "query": q,
                "fields": ["title^3", "description", "tags"],
                "type": "best_fields"
            }
        },
        # Fix 1: Deterministic sort to prevent random ordering of equal scores
        "sort": [
            {"_score": "desc"},
            {"created_at": "desc"},  # Tie-breaker: consistent ordering
            {"_id": "asc"}           # Final tie-breaker: document ID
        ],
        "from": (page - 1) * size,
        "size": size,
    }

    result = await es.search(
        index="products",
        body=body,
        # Fix 2: Read from primary shard to avoid replica lag
        preference="_primary_first",
        # Fix 3: Force refresh to see latest writes (use sparingly!)
        # refresh=True,  # Only for critical searches, impacts performance
    )

    return {
        "query": q,
        "total": result["hits"]["total"]["value"],
        "results": [hit["_source"] for hit in result["hits"]["hits"]],
        "page": page,
    }

# Fix 4: Normalize cache keys to prevent cache inconsistency
def get_cache_key(query: str, page: int, size: int) -> str:
    """Normalized cache key — same query always hits same cache."""
    normalized = query.lower().strip()
    return hashlib.md5(f"{normalized}:{page}:{size}".encode()).hexdigest()
```

### Key Takeaways
> - 💡 **Replication lag** is the #1 cause — read replicas can be seconds behind
> - 💡 Add **deterministic tie-breakers** in sort (timestamp + ID) to prevent random ordering
> - 💡 Use `preference=_primary_first` in Elasticsearch for consistent reads when needed
> - 💡 **Normalize cache keys** — different casing or whitespace shouldn't produce different caches
> - 💡 Monitor **replication lag** with alerts when it exceeds acceptable thresholds

---

## Question 79 — Users Get Logged Out Randomly — Session/Token Expiry Bug
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Users report getting logged out randomly throughout the day. Some sessions last 5 minutes, others last hours. There's no pattern. How do you debug and fix this?"*

### The Answer

**Step 1: Common causes of random logouts**

| Cause | How to Detect | Fix |
|-------|--------------|-----|
| **Clock skew between servers** | Token exp claim fails on some servers | Sync clocks with NTP; add 30s buffer |
| **Token stored in memory only** | App restart = all sessions lost | Store in Redis/cookie with expiry |
| **Multiple server instances** | Session on server A, request hits server B | Use JWT (stateless) or Redis sessions |
| **Short token TTL** | Access token expires too quickly | Use refresh token rotation |
| **Cookie misconfiguration** | SameSite, Secure, Domain issues | Fix cookie attributes |

```
CLOCK SKEW PROBLEM:
Server A clock: 10:00:00 → Issues token: exp = 10:30:00
Server B clock: 10:00:05 → At "10:30:02" (really 10:29:57) → token seems expired!

FIX: Add 30-second tolerance
Server B: token.exp (10:30:00) + 30s buffer = 10:30:30
Server B at 10:30:02: 10:30:02 < 10:30:30 → token still valid ✅
```

```mermaid
flowchart TD
    A[User reports random logout] --> B{Check server logs}
    B --> C{Token expired error?}
    C -->|Yes| D{Clock skew?\nToken TTL too short?}
    C -->|No| E{Session not found error?}
    D -->|Clock skew| F[Sync NTP + add buffer]
    D -->|Short TTL| G[Implement refresh tokens]
    E -->|Yes| H{Server restarted?\nMulti-instance?}
    H -->|Restart| I[Move sessions to Redis]
    H -->|Multi-instance| J[Use JWT or shared Redis]
```

### Code Example — Robust Token Management

```python
import jwt
import time
from datetime import datetime, timedelta, timezone
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import HTTPBearer

app = FastAPI()
security = HTTPBearer()

SECRET_KEY = "your-secret-key"
ACCESS_TOKEN_EXPIRE = 15 * 60      # 15 minutes
REFRESH_TOKEN_EXPIRE = 7 * 24 * 3600  # 7 days
CLOCK_SKEW_TOLERANCE = 30          # 30 seconds tolerance

def create_tokens(user_id: str) -> dict:
    """Create access + refresh token pair."""
    now = datetime.now(timezone.utc)

    access_token = jwt.encode({
        "sub": user_id,
        "type": "access",
        "iat": now,
        "exp": now + timedelta(seconds=ACCESS_TOKEN_EXPIRE),
    }, SECRET_KEY, algorithm="HS256")

    refresh_token = jwt.encode({
        "sub": user_id,
        "type": "refresh",
        "iat": now,
        "exp": now + timedelta(seconds=REFRESH_TOKEN_EXPIRE),
    }, SECRET_KEY, algorithm="HS256")

    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "expires_in": ACCESS_TOKEN_EXPIRE,
    }

async def get_current_user(credentials=Depends(security)) -> dict:
    """Validate access token with clock skew tolerance."""
    try:
        payload = jwt.decode(
            credentials.credentials,
            SECRET_KEY,
            algorithms=["HS256"],
            leeway=CLOCK_SKEW_TOLERANCE,  # ← FIX: 30s tolerance for clock skew
        )
        if payload.get("type") != "access":
            raise HTTPException(401, "Invalid token type")
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Token expired. Use refresh token.",
                          headers={"X-Token-Expired": "true"})
    except jwt.InvalidTokenError:
        raise HTTPException(401, "Invalid token")

@app.post("/auth/login")
async def login(username: str, password: str):
    # Validate credentials...
    return create_tokens(user_id=username)

@app.post("/auth/refresh")
async def refresh_token(credentials=Depends(security)):
    """Exchange refresh token for new access + refresh token pair."""
    try:
        payload = jwt.decode(
            credentials.credentials, SECRET_KEY,
            algorithms=["HS256"], leeway=CLOCK_SKEW_TOLERANCE
        )
        if payload.get("type") != "refresh":
            raise HTTPException(400, "Provide a refresh token")
        return create_tokens(user_id=payload["sub"])
    except jwt.ExpiredSignatureError:
        raise HTTPException(401, "Refresh token expired. Please log in again.")

@app.get("/profile")
async def get_profile(user=Depends(get_current_user)):
    return {"user_id": user["sub"]}
```

### Key Takeaways
> - 💡 **Clock skew** between servers is the most common cause — add `leeway` parameter in JWT decode
> - 💡 Use **short access tokens** (15 min) + **long refresh tokens** (7 days) pattern
> - 💡 Store sessions in **Redis** if using server-side sessions (survives restarts)
> - 💡 Send `X-Token-Expired: true` header so clients know to use refresh token
> - 💡 Always **sync server clocks** with NTP in production environments

---

## Question 80 — Design a REST API for a Notification System (Push, Email, SMS)
🔴 Senior | ★★☆ Common

### The Scenario
> *"Design a notification system REST API that can send push notifications, emails, and SMS. It needs to handle templates, user preferences, and high volume."*

### The Answer

**Step 1: API design**

```
ENDPOINTS:
POST   /notifications          → Send a notification
GET    /notifications          → List sent notifications
GET    /notifications/{id}     → Get notification status
POST   /notifications/bulk     → Send to multiple users
GET    /users/{id}/preferences → Get notification preferences
PUT    /users/{id}/preferences → Update preferences
POST   /templates              → Create notification template
```

**Step 2: Architecture**

```
┌──────────┐     ┌───────────────┐     ┌─────────────────────┐
│  API     │────►│  Message      │────►│  Channel Workers    │
│  Server  │     │  Queue        │     │                     │
│          │     │  (RabbitMQ)   │     │  ┌── Email Worker   │
└──────────┘     └───────────────┘     │  ├── SMS Worker     │
                                       │  ├── Push Worker    │
                                       │  └── In-App Worker  │
                                       └─────────────────────┘
```

```mermaid
flowchart TD
    A[POST /notifications] --> B[Validate request]
    B --> C[Check user preferences\nDo they want this channel?]
    C --> D[Render template\nwith user data]
    D --> E[Enqueue per channel]
    E --> F[Email Worker\nSendGrid/SES]
    E --> G[SMS Worker\nTwilio]
    E --> H[Push Worker\nFCM/APNs]
    F --> I[Update delivery status]
    G --> I
    H --> I
```

### Code Example — Notification System API

```python
from fastapi import FastAPI, HTTPException, BackgroundTasks
from pydantic import BaseModel
from enum import Enum
from datetime import datetime, timezone
import uuid

app = FastAPI()

class Channel(str, Enum):
    EMAIL = "email"
    SMS = "sms"
    PUSH = "push"
    IN_APP = "in_app"

class Priority(str, Enum):
    LOW = "low"
    NORMAL = "normal"
    HIGH = "high"
    CRITICAL = "critical"

class NotificationRequest(BaseModel):
    user_id: str
    template_id: str
    channels: list[Channel]
    priority: Priority = Priority.NORMAL
    data: dict = {}  # Template variables

class UserPreferences(BaseModel):
    email_enabled: bool = True
    sms_enabled: bool = True
    push_enabled: bool = True
    quiet_hours_start: str = "22:00"
    quiet_hours_end: str = "08:00"

# Storage
notifications_db: dict[str, dict] = {}
preferences_db: dict[str, dict] = {}

# Channel processors
async def send_email(notification_id: str, user_id: str, content: dict):
    """Send email via SendGrid/SES."""
    await update_status(notification_id, "email", "delivered")

async def send_sms(notification_id: str, user_id: str, content: dict):
    """Send SMS via Twilio."""
    await update_status(notification_id, "sms", "delivered")

async def send_push(notification_id: str, user_id: str, content: dict):
    """Send push via FCM/APNs."""
    await update_status(notification_id, "push", "delivered")

CHANNEL_HANDLERS = {
    Channel.EMAIL: send_email,
    Channel.SMS: send_sms,
    Channel.PUSH: send_push,
}

async def update_status(notif_id: str, channel: str, status: str):
    if notif_id in notifications_db:
        notifications_db[notif_id]["channel_status"][channel] = status

@app.post("/notifications", status_code=202)
async def send_notification(req: NotificationRequest, background_tasks: BackgroundTasks):
    notif_id = str(uuid.uuid4())

    # Check user preferences
    prefs = preferences_db.get(req.user_id, UserPreferences().model_dump())
    active_channels = []
    for ch in req.channels:
        if ch == Channel.EMAIL and prefs.get("email_enabled", True):
            active_channels.append(ch)
        elif ch == Channel.SMS and prefs.get("sms_enabled", True):
            active_channels.append(ch)
        elif ch == Channel.PUSH and prefs.get("push_enabled", True):
            active_channels.append(ch)

    notifications_db[notif_id] = {
        "id": notif_id,
        "user_id": req.user_id,
        "template_id": req.template_id,
        "channels": [ch.value for ch in active_channels],
        "priority": req.priority.value,
        "status": "queued",
        "channel_status": {ch.value: "queued" for ch in active_channels},
        "created_at": datetime.now(timezone.utc).isoformat(),
    }

    # Dispatch to channel workers
    for channel in active_channels:
        handler = CHANNEL_HANDLERS.get(channel)
        if handler:
            background_tasks.add_task(handler, notif_id, req.user_id, req.data)

    return {"notification_id": notif_id, "channels": [ch.value for ch in active_channels]}

@app.get("/notifications/{notif_id}")
async def get_notification_status(notif_id: str):
    if notif_id not in notifications_db:
        raise HTTPException(404, "Notification not found")
    return notifications_db[notif_id]

@app.put("/users/{user_id}/preferences")
async def update_preferences(user_id: str, prefs: UserPreferences):
    preferences_db[user_id] = prefs.model_dump()
    return {"user_id": user_id, "preferences": prefs}
```

### Key Takeaways
> - 💡 Use **async processing** (202 + queue) — never send notifications synchronously
> - 💡 Respect **user preferences** — always check before sending to any channel
> - 💡 Design with **templates** — separate content from delivery logic
> - 💡 Track **per-channel delivery status** — one notification can succeed on email but fail on SMS
> - 💡 Implement **quiet hours** and **priority levels** to avoid notification fatigue

---

## Question 81 — Long-Running API Endpoint Holds DB Connection for 30 Seconds
🟡 Mid | ★★★ Very Common

### The Scenario
> *"You have an endpoint that does heavy processing and holds a database connection for 30 seconds. During peak traffic, you run out of connections. How do you fix this?"*

### The Answer

**Step 1: Identify the problem**

```
PROBLEM:
10 concurrent requests × 30 seconds each = 10 connections held
Max pool = 20 connections
→ Only 10 connections left for the rest of the app!
→ At 50 concurrent requests → Pool exhausted → 500 errors
```

**Step 2: Solutions**

| Solution | When to Use |
|----------|------------|
| **Minimize connection hold time** | Always — fetch data early, release, then process |
| **Background processing** | If processing doesn't need real-time response |
| **Streaming/chunked processing** | For large data sets |
| **Connection pool tuning** | Quick fix but has limits |
| **Read replicas** | For read-heavy long queries |

```mermaid
flowchart TD
    A[Long-running endpoint] --> B{Does processing\nneed DB the whole time?}
    B -->|No| C[Fetch data → Release connection\n→ Process in memory → Write back]
    B -->|Yes| D{Must be synchronous?}
    D -->|No| E[Return 202 + job_id\nProcess in background worker]
    D -->|Yes| F[Use read replica for queries\nOnly write to primary briefly]
```

### Code Example — Release DB Connection During Processing

```python
from fastapi import FastAPI, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine, async_sessionmaker
import uuid

app = FastAPI()

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/db",
    pool_size=20,
    max_overflow=10,
    pool_timeout=10,  # Fail fast if pool is exhausted
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)

# ❌ BAD: Holds connection for entire processing time
@app.get("/reports/bad/{report_id}")
async def generate_report_bad(report_id: str):
    async with SessionLocal() as session:
        # Fetch data (1 second)
        data = await session.execute("SELECT * FROM large_table")
        rows = data.fetchall()

        # Process data — 30 SECONDS with DB connection held!
        result = heavy_computation(rows)

        # Write result (1 second)
        await session.execute(
            "INSERT INTO reports (id, data) VALUES (:id, :data)",
            {"id": report_id, "data": str(result)}
        )
        await session.commit()
    return result

# ✅ GOOD: Minimize connection hold time
@app.get("/reports/good/{report_id}")
async def generate_report_good(report_id: str):
    # Step 1: Fetch data quickly, release connection
    async with SessionLocal() as session:
        data = await session.execute("SELECT * FROM large_table")
        rows = data.fetchall()
    # ← Connection released here!

    # Step 2: Process in memory (no DB connection held)
    result = heavy_computation(rows)  # 30 seconds, but no connection held

    # Step 3: Write result briefly
    async with SessionLocal() as session:
        await session.execute(
            "INSERT INTO reports (id, data) VALUES (:id, :data)",
            {"id": report_id, "data": str(result)}
        )
        await session.commit()
    # ← Connection released here!

    return result

# ✅ BEST: Background processing for very long tasks
@app.post("/reports", status_code=202)
async def create_report(background_tasks: BackgroundTasks):
    job_id = str(uuid.uuid4())
    background_tasks.add_task(process_report_async, job_id)
    return {"job_id": job_id, "poll_url": f"/reports/{job_id}/status"}

async def process_report_async(job_id: str):
    """Background task — doesn't tie up API connection pool."""
    async with SessionLocal() as session:
        data = await session.execute("SELECT * FROM large_table")
        rows = data.fetchall()

    result = heavy_computation(rows)

    async with SessionLocal() as session:
        await session.execute(
            "INSERT INTO reports (id, data, status) VALUES (:id, :data, :s)",
            {"id": job_id, "data": str(result), "s": "completed"}
        )
        await session.commit()

def heavy_computation(rows):
    """Simulate heavy processing."""
    import time
    time.sleep(0.1)  # In reality: complex calculations
    return {"processed": len(rows)}
```

### Key Takeaways
> - 💡 **Fetch data → release connection → process → reconnect briefly to write** — minimize hold time
> - 💡 For tasks over 5 seconds, use **background processing** (Celery, BackgroundTasks)
> - 💡 Set `pool_timeout` to **fail fast** instead of hanging when pool is exhausted
> - 💡 Use `pool_size` = total_connections / app_instances - buffer
> - 💡 Consider **PgBouncer** as a connection pooler between your app and PostgreSQL

---

## Question 82 — Implement Health Check Endpoints for Kubernetes Liveness/Readiness
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Your FastAPI app runs in Kubernetes. You need liveness and readiness probe endpoints. What's the difference and how do you implement them?"*

### The Answer

| Probe | Purpose | What It Checks | On Failure |
|-------|---------|---------------|------------|
| **Liveness** | "Is the app alive?" | App is running, not deadlocked | K8s **restarts** the pod |
| **Readiness** | "Can the app serve traffic?" | DB connected, dependencies ready | K8s **stops routing** traffic to pod |
| **Startup** | "Has the app started?" | Initial setup complete | K8s waits before checking liveness |

```
KUBERNETES PROBES:

Startup Probe ──► Liveness Probe ──► Readiness Probe
   │                  │                   │
   │ "Is the app     │ "Is the app      │ "Can the app
   │  booted yet?"   │  still running?" │  handle requests?"
   │                  │                   │
   │ Fail → Wait     │ Fail → Restart   │ Fail → Stop traffic
   │ more time        │ the pod          │ to this pod
```

```mermaid
flowchart TD
    A[Pod Starts] --> B[Startup Probe\n/health/startup]
    B -->|Success| C[Liveness Probe\n/health/live every 10s]
    B -->|Failure for 5min| D[Kill Pod ❌]
    C -->|Success| E[Readiness Probe\n/health/ready every 5s]
    C -->|3 failures| F[Restart Pod 🔄]
    E -->|Success| G[Receive Traffic ✅]
    E -->|Failure| H[Remove from Service\nNo traffic 🚫]
```

### Code Example — Health Check Endpoints

```python
from fastapi import FastAPI
from datetime import datetime, timezone
import asyncio

app = FastAPI()

# Track app state
app_state = {
    "started": False,
    "ready": False,
    "start_time": None,
}

@app.on_event("startup")
async def startup():
    app_state["start_time"] = datetime.now(timezone.utc)
    # Simulate initialization (DB migrations, cache warmup, etc.)
    await asyncio.sleep(0.1)
    app_state["started"] = True
    app_state["ready"] = True

# === LIVENESS: Is the app alive? (simple check) ===
@app.get("/health/live")
async def liveness():
    """Liveness probe — fails only if app is deadlocked or crashed."""
    return {"status": "alive", "timestamp": datetime.now(timezone.utc).isoformat()}

# === READINESS: Can the app serve traffic? (deep check) ===
@app.get("/health/ready")
async def readiness():
    """Readiness probe — checks all dependencies."""
    checks = {}

    # Check database
    try:
        # In production: await db.execute("SELECT 1")
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {str(e)}"

    # Check Redis
    try:
        # In production: await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {str(e)}"

    # Check if app is ready
    all_ok = all(v == "ok" for v in checks.values()) and app_state["ready"]

    if all_ok:
        return {"status": "ready", "checks": checks}
    else:
        from fastapi.responses import JSONResponse
        return JSONResponse(
            status_code=503,
            content={"status": "not_ready", "checks": checks}
        )

# === STARTUP: Has initial setup completed? ===
@app.get("/health/startup")
async def startup_check():
    if app_state["started"]:
        return {"status": "started"}
    from fastapi.responses import JSONResponse
    return JSONResponse(status_code=503, content={"status": "starting"})
```

```yaml
# Kubernetes deployment configuration
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: api
          image: myapp:latest
          ports:
            - containerPort: 8000
          startupProbe:
            httpGet:
              path: /health/startup
              port: 8000
            failureThreshold: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            periodSeconds: 5
            failureThreshold: 3
```

### Key Takeaways
> - 💡 **Liveness** = "is the process alive?" → Keep it simple (just return 200)
> - 💡 **Readiness** = "can it handle traffic?" → Check DB, Redis, and other dependencies
> - 💡 **Startup** = "has it finished initializing?" → Prevents premature liveness checks
> - 💡 Liveness failure → pod **restarts**; Readiness failure → pod **stops receiving traffic**
> - 💡 Don't make liveness checks too complex — a false failure restarts your pod unnecessarily

---

## Question 83 — Handle Time Zones Correctly Across Global Users
🟢 Junior | ★★☆ Common

### The Scenario
> *"Your API serves users across different time zones. Users in Tokyo see different event times than users in New York. Timestamps are inconsistent. How do you handle time zones correctly?"*

### The Answer

**The golden rule: Store in UTC, display in local time.**

| Layer | What to Do |
|-------|-----------|
| **Database** | Store all timestamps in UTC |
| **API** | Accept and return ISO 8601 with timezone info |
| **Client** | Convert UTC to user's local timezone for display |

```
❌ WRONG: Store local times
  User in Tokyo creates event at "2024-01-15 10:00" (JST)
  Stored as: "2024-01-15 10:00" (no timezone!)
  User in NY sees: "2024-01-15 10:00" ← thinks it's 10 AM EST!

✅ CORRECT: Store UTC
  User in Tokyo creates event at "2024-01-15 10:00" (JST = UTC+9)
  Stored as: "2024-01-15T01:00:00Z" (UTC)
  User in NY sees: "2024-01-14T20:00:00-05:00" (EST) ← correct!
```

```mermaid
flowchart LR
    A[Client Tokyo\n10:00 JST] -->|Send UTC\n01:00Z| B[API Server\nStore UTC]
    B -->|Return UTC\n01:00Z| C[Client New York\nDisplay 20:00 EST]
    B -->|Return UTC\n01:00Z| D[Client London\nDisplay 01:00 GMT]
```

### Code Example — Proper Timezone Handling

```python
from datetime import datetime, timezone, timedelta
from fastapi import FastAPI, Query
from pydantic import BaseModel, field_validator
from zoneinfo import ZoneInfo

app = FastAPI()

class EventCreate(BaseModel):
    title: str
    event_time: datetime  # Must include timezone info

    @field_validator("event_time")
    @classmethod
    def validate_timezone(cls, v):
        if v.tzinfo is None:
            raise ValueError("Timestamp must include timezone (e.g., 2024-01-15T10:00:00+09:00)")
        return v

class EventResponse(BaseModel):
    id: int
    title: str
    event_time_utc: str      # Always UTC
    event_time_local: str     # In requested timezone

events_db = {}

@app.post("/events")
async def create_event(event: EventCreate):
    # Convert to UTC for storage
    utc_time = event.event_time.astimezone(timezone.utc)

    event_id = len(events_db) + 1
    events_db[event_id] = {
        "id": event_id,
        "title": event.title,
        "event_time_utc": utc_time,
    }

    return {
        "id": event_id,
        "title": event.title,
        "event_time_utc": utc_time.isoformat(),
    }

@app.get("/events/{event_id}")
async def get_event(event_id: int, tz: str = Query("UTC", description="Timezone (e.g., America/New_York)")):
    event = events_db.get(event_id)
    if not event:
        return {"error": "Not found"}, 404

    utc_time = event["event_time_utc"]

    # Convert to requested timezone
    try:
        target_tz = ZoneInfo(tz)
        local_time = utc_time.astimezone(target_tz)
    except KeyError:
        local_time = utc_time  # Fall back to UTC if invalid timezone

    return {
        "id": event["id"],
        "title": event["title"],
        "event_time_utc": utc_time.isoformat(),
        "event_time_local": local_time.isoformat(),
        "timezone": tz,
    }

# Always use timezone-aware datetime
@app.get("/now")
async def get_current_time(tz: str = "UTC"):
    utc_now = datetime.now(timezone.utc)
    try:
        local_now = utc_now.astimezone(ZoneInfo(tz))
    except KeyError:
        local_now = utc_now
    return {
        "utc": utc_now.isoformat(),
        "local": local_now.isoformat(),
        "timezone": tz,
    }
```

### Key Takeaways
> - 💡 **Always store timestamps in UTC** — convert to local time only for display
> - 💡 Require **timezone info** in all input timestamps (reject naive datetimes)
> - 💡 Use **ISO 8601 format** (`2024-01-15T01:00:00Z`) — it's unambiguous
> - 💡 Let the **client** handle display conversion, or accept a `tz` query parameter
> - 💡 Use Python's `zoneinfo` module (built-in since 3.9) instead of pytz

---

## Question 84 — API Returns 200 OK Even for Errors — Legacy System
🟡 Mid | ★★☆ Common

### The Scenario
> *"You inherited a legacy API that returns 200 OK for everything — even errors. The error details are buried inside the JSON body. Clients can't distinguish success from failure. How do you fix this without breaking existing clients?"*

### The Answer

**Step 1: Understand the problem**

```
CURRENT (WRONG):
POST /orders → 200 {"success": false, "error": "Out of stock"}
POST /orders → 200 {"success": true, "data": {...}}

Both return 200! Load balancers, monitoring, and retry logic
all think every request succeeded.

CORRECT:
POST /orders → 409 {"error": "Out of stock"}
POST /orders → 201 {"data": {...}}
```

**Step 2: Gradual migration strategy**

```mermaid
flowchart TD
    A[Phase 1: Add header\nX-Use-Proper-Status: true] --> B[Clients opt-in\nto proper status codes]
    B --> C[Phase 2: New API version\n/v2/ uses correct codes]
    C --> D[Phase 3: Deprecate v1\nSunset header]
    D --> E[Phase 4: Remove v1\nAll clients on v2]
```

### Code Example — Gradual Status Code Migration

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import JSONResponse

app = FastAPI()

# Legacy error mapping
ERROR_STATUS_MAP = {
    "not_found": 404,
    "validation_error": 422,
    "conflict": 409,
    "unauthorized": 401,
    "forbidden": 403,
    "internal_error": 500,
    "out_of_stock": 409,
}

def make_response(
    request: Request,
    success: bool,
    data: dict = None,
    error: str = None,
    error_code: str = None
) -> JSONResponse:
    """Return proper status code or legacy 200 based on client preference."""

    # Check if client opted into proper status codes
    use_proper_status = (
        request.headers.get("X-Use-Proper-Status") == "true"
        or request.url.path.startswith("/v2")
    )

    if success:
        body = {"success": True, "data": data}
        status = 200
    else:
        body = {"success": False, "error": error, "error_code": error_code}
        if use_proper_status:
            status = ERROR_STATUS_MAP.get(error_code, 400)
        else:
            status = 200  # Legacy behavior

    return JSONResponse(content=body, status_code=status)

# Legacy endpoint (v1) — returns 200 for everything by default
@app.post("/v1/orders")
async def create_order_v1(request: Request):
    order_data = await request.json()
    if not order_data.get("items"):
        return make_response(request, success=False, error="No items", error_code="validation_error")
    return make_response(request, success=True, data={"order_id": "ord_123"})

# New endpoint (v2) — always uses proper status codes
@app.post("/v2/orders")
async def create_order_v2(request: Request):
    order_data = await request.json()
    if not order_data.get("items"):
        raise HTTPException(422, detail={"error": "No items provided"})
    return JSONResponse(
        status_code=201,
        content={"data": {"order_id": "ord_123"}}
    )

# Add deprecation headers for v1
@app.middleware("http")
async def add_deprecation_headers(request: Request, call_next):
    response = await call_next(request)
    if request.url.path.startswith("/v1"):
        response.headers["Deprecation"] = "true"
        response.headers["Sunset"] = "2025-06-01"
        response.headers["Link"] = '</v2>; rel="successor-version"'
    return response
```

### Key Takeaways
> - 💡 Returning 200 for errors breaks **monitoring**, **load balancers**, and **client error handling**
> - 💡 Migrate gradually: add an **opt-in header** → create **v2** → deprecate **v1**
> - 💡 Use `Deprecation` and `Sunset` headers to communicate the timeline to clients
> - 💡 In v2, use **proper HTTP status codes**: 201, 400, 404, 409, 422, 500
> - 💡 Document the migration clearly and give clients **at least 6 months** to migrate

---

## Question 85 — Implement an Undo/Rollback Feature for REST API Operations
🔴 Senior | ★☆☆ Rare

### The Scenario
> *"Users want an 'undo' button that reverses their last API action. How would you design an undo/rollback feature for a REST API?"*

### The Answer

**Step 1: Choose an undo strategy**

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Command pattern** | Store each action as a reversible command | Complex actions with known inverses |
| **Event sourcing** | Store all events, rebuild state at any point | Full audit trail + time travel |
| **Soft delete + snapshots** | Mark as deleted, keep previous versions | Simple CRUD operations |
| **Compensation transactions** | Execute reverse action | Distributed systems |

```
COMMAND PATTERN:
Action: Update user name "Alice" → "Bob"
Stored command: {action: "update", field: "name", old: "Alice", new: "Bob"}
Undo: Update user name "Bob" → "Alice" (reverse the command)

EVENT SOURCING:
Event 1: UserCreated {name: "Alice"}
Event 2: NameChanged {old: "Alice", new: "Bob"}
Event 3: EmailChanged {old: "a@x.com", new: "b@x.com"}
Undo Event 3: Replay events 1+2 only → State: {name: "Bob", email: "a@x.com"}
```

```mermaid
flowchart TD
    A[User performs action] --> B[Store action + before state]
    B --> C[Execute action]
    C --> D[Return action_id]
    D --> E[User clicks Undo]
    E --> F[POST /actions/action_id/undo]
    F --> G{Action undoable?}
    G -->|Yes| H[Apply reverse operation]
    G -->|No expired/already undone| I[Return 409 Cannot undo]
    H --> J[Mark action as undone]
```

### Code Example — Undo System with Command Pattern

```python
from fastapi import FastAPI, HTTPException
from datetime import datetime, timezone, timedelta
from pydantic import BaseModel
import uuid
import copy

app = FastAPI()

# Storage
resources_db: dict[str, dict] = {
    "user_1": {"id": "user_1", "name": "Alice", "email": "alice@example.com"}
}
actions_db: dict[str, dict] = {}

UNDO_WINDOW = timedelta(minutes=30)  # Can only undo within 30 minutes

class UndoableAction:
    def __init__(self, resource_type: str, resource_id: str, action: str,
                 before_state: dict, after_state: dict):
        self.id = str(uuid.uuid4())
        self.resource_type = resource_type
        self.resource_id = resource_id
        self.action = action
        self.before_state = before_state
        self.after_state = after_state
        self.created_at = datetime.now(timezone.utc)
        self.undone = False

def record_action(resource_type: str, resource_id: str, action: str,
                  before: dict, after: dict) -> str:
    """Record an undoable action."""
    act = UndoableAction(resource_type, resource_id, action, before, after)
    actions_db[act.id] = {
        "id": act.id,
        "resource_type": resource_type,
        "resource_id": resource_id,
        "action": action,
        "before_state": before,
        "after_state": after,
        "created_at": act.created_at.isoformat(),
        "undone": False,
    }
    return act.id

@app.put("/users/{user_id}")
async def update_user(user_id: str, updates: dict):
    if user_id not in resources_db:
        raise HTTPException(404, "User not found")

    # Capture before state
    before = copy.deepcopy(resources_db[user_id])

    # Apply updates
    resources_db[user_id].update(updates)
    after = copy.deepcopy(resources_db[user_id])

    # Record undoable action
    action_id = record_action("user", user_id, "update", before, after)

    return {
        "user": resources_db[user_id],
        "action_id": action_id,
        "undo_url": f"/actions/{action_id}/undo",
        "undo_expires": (datetime.now(timezone.utc) + UNDO_WINDOW).isoformat()
    }

@app.post("/actions/{action_id}/undo")
async def undo_action(action_id: str):
    action = actions_db.get(action_id)
    if not action:
        raise HTTPException(404, "Action not found")
    if action["undone"]:
        raise HTTPException(409, "Action already undone")

    created = datetime.fromisoformat(action["created_at"])
    if datetime.now(timezone.utc) - created > UNDO_WINDOW:
        raise HTTPException(410, "Undo window has expired (30 minutes)")

    # Restore before state
    resource_id = action["resource_id"]
    resources_db[resource_id] = copy.deepcopy(action["before_state"])
    action["undone"] = True

    return {
        "status": "undone",
        "resource": resources_db[resource_id],
        "message": f"Action {action_id} has been reversed"
    }
```

### Key Takeaways
> - 💡 Store **before state** with every mutation — this enables reversal
> - 💡 Set an **undo window** (e.g., 30 minutes) — indefinite undo is impractical
> - 💡 Return `action_id` and `undo_url` in mutation responses for easy client integration
> - 💡 For full undo capability at scale, consider **event sourcing**
> - 💡 Some actions are **not reversible** (e.g., sent email) — communicate this clearly

---

## Question 86 — API Documentation Is Outdated — Clients Are Breaking
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Your API documentation is outdated — it doesn't match the actual API behavior. Clients are building integrations based on wrong docs and breaking. How do you fix this permanently?"*

### The Answer

**Step 1: Root cause — docs are separate from code**

```
❌ PROBLEM: Manual docs (separate wiki/Notion)
  Code changes → Developer forgets to update docs → Docs drift

✅ SOLUTION: Auto-generated docs from code
  Code changes → Docs update automatically → Always in sync
```

| Approach | How | Tools |
|----------|-----|-------|
| **Code-first (recommended)** | FastAPI/Pydantic auto-generates OpenAPI spec | FastAPI, Swagger UI |
| **Spec-first** | Write OpenAPI spec first, validate code against it | Schemathesis, Dredd |
| **Contract testing** | Tests fail if API doesn't match spec | Schemathesis, Pact |

```mermaid
flowchart TD
    A[Developer writes FastAPI code\nwith type hints + docstrings] --> B[FastAPI auto-generates\nOpenAPI spec]
    B --> C[Swagger UI /docs\nalways up to date]
    B --> D[CI: Schemathesis tests\nvalidate spec matches reality]
    D -->|Mismatch| E[Build fails ❌\nDeveloper must fix]
    D -->|Match| F[Deploy ✅\nDocs guaranteed accurate]
```

### Code Example — Self-Documenting API

```python
from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel, Field
from enum import Enum

app = FastAPI(
    title="Product API",
    description="API for managing products. Auto-generated docs always match the code.",
    version="2.0.0",
    docs_url="/docs",       # Swagger UI
    redoc_url="/redoc",     # ReDoc
)

class ProductStatus(str, Enum):
    ACTIVE = "active"
    DRAFT = "draft"
    ARCHIVED = "archived"

class ProductCreate(BaseModel):
    """Request body for creating a product."""
    name: str = Field(..., min_length=1, max_length=200, description="Product name")
    price: float = Field(..., gt=0, description="Price in USD, must be positive")
    status: ProductStatus = Field(ProductStatus.DRAFT, description="Product status")

    model_config = {
        "json_schema_extra": {
            "examples": [
                {"name": "Widget Pro", "price": 29.99, "status": "active"}
            ]
        }
    }

class ProductResponse(BaseModel):
    """Response body for product operations."""
    id: int
    name: str
    price: float
    status: ProductStatus

@app.post(
    "/products",
    response_model=ProductResponse,
    status_code=201,
    summary="Create a new product",
    description="Creates a new product in the catalog. Returns the created product with its ID.",
    responses={
        201: {"description": "Product created successfully"},
        422: {"description": "Validation error — check field requirements"},
    }
)
async def create_product(product: ProductCreate):
    """
    Create a product with the following rules:
    - **name** must be 1-200 characters
    - **price** must be positive
    - **status** defaults to 'draft'
    """
    return ProductResponse(id=1, **product.model_dump())

@app.get(
    "/products",
    response_model=list[ProductResponse],
    summary="List all products",
    description="Returns paginated list of products with optional status filter.",
)
async def list_products(
    status: ProductStatus = Query(None, description="Filter by product status"),
    page: int = Query(1, ge=1, description="Page number"),
    size: int = Query(20, ge=1, le=100, description="Items per page"),
):
    return []
```

```bash
# CI pipeline: validate docs match reality
# pip install schemathesis
# schemathesis run http://localhost:8000/openapi.json --validate-schema=true
```

### Key Takeaways
> - 💡 Use **code-first** approach (FastAPI) — docs are auto-generated and always accurate
> - 💡 Add **Pydantic models** with Field descriptions and examples for rich documentation
> - 💡 Run **contract tests** (Schemathesis) in CI to catch spec/implementation mismatches
> - 💡 Include **response examples** and **error descriptions** in your schema
> - 💡 Treat docs as **code** — version them, review them in PRs, test them in CI

---

## Question 87 — Mask/Redact Sensitive Fields (PII) in API Logs
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API logs contain sensitive data — credit card numbers, passwords, SSNs, email addresses. Compliance requires you to mask PII in all logs. How do you implement this?"*

### The Answer

**Step 1: Identify PII fields**

| Field Type | Example | Masking Strategy |
|-----------|---------|-----------------|
| Credit card | `4111111111111111` | `****-****-****-1111` (show last 4) |
| SSN | `123-45-6789` | `***-**-6789` (show last 4) |
| Password | `mypassword` | `[REDACTED]` (never log) |
| Email | `alice@example.com` | `a***@example.com` |
| Phone | `+1-555-123-4567` | `+1-555-***-4567` |
| API key | `sk_live_abc123xyz` | `sk_live_***xyz` |

```mermaid
flowchart TD
    A[API Request/Response] --> B[Logging Middleware]
    B --> C[PII Scanner\nRegex + field names]
    C --> D{Contains PII?}
    D -->|Yes| E[Mask sensitive fields]
    D -->|No| F[Log as-is]
    E --> G[Write masked log]
    F --> G
```

### Code Example — PII Masking Middleware

```python
import re
import json
import logging
import copy
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

app = FastAPI()

logger = logging.getLogger("api")

# Fields to always redact
SENSITIVE_FIELDS = {
    "password", "secret", "token", "api_key", "authorization",
    "credit_card", "card_number", "cvv", "ssn", "social_security",
}

# Regex patterns for PII detection
PII_PATTERNS = {
    "credit_card": re.compile(r"\b\d{4}[-\s]?\d{4}[-\s]?\d{4}[-\s]?\d{4}\b"),
    "ssn": re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
    "email": re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"),
}

def mask_value(key: str, value: str) -> str:
    """Mask a sensitive value based on its type."""
    if key.lower() in {"password", "secret", "cvv", "authorization"}:
        return "[REDACTED]"
    if key.lower() in {"credit_card", "card_number"}:
        return f"****-****-****-{str(value)[-4:]}" if len(str(value)) >= 4 else "[REDACTED]"
    if key.lower() == "ssn":
        return f"***-**-{str(value)[-4:]}" if len(str(value)) >= 4 else "[REDACTED]"
    if key.lower() in {"email"}:
        parts = str(value).split("@")
        if len(parts) == 2:
            return f"{parts[0][0]}***@{parts[1]}"
    if key.lower() in {"api_key", "token"}:
        s = str(value)
        if len(s) > 8:
            return f"{s[:6]}***{s[-3:]}"
    return "[REDACTED]"

def mask_dict(data: dict) -> dict:
    """Recursively mask sensitive fields in a dictionary."""
    masked = {}
    for key, value in data.items():
        if key.lower() in SENSITIVE_FIELDS:
            masked[key] = mask_value(key, str(value))
        elif isinstance(value, dict):
            masked[key] = mask_dict(value)
        elif isinstance(value, list):
            masked[key] = [mask_dict(v) if isinstance(v, dict) else v for v in value]
        else:
            # Check value against regex patterns
            str_val = str(value)
            masked_val = str_val
            for pattern_name, pattern in PII_PATTERNS.items():
                if pattern.search(str_val):
                    masked_val = mask_value(pattern_name, str_val)
                    break
            masked[key] = masked_val
    return masked

class PIIMaskingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Log request (masked)
        try:
            body = await request.body()
            if body:
                body_dict = json.loads(body)
                masked_body = mask_dict(body_dict)
                logger.info(f"Request: {request.method} {request.url} body={json.dumps(masked_body)}")
        except (json.JSONDecodeError, Exception):
            logger.info(f"Request: {request.method} {request.url}")

        response = await call_next(request)
        logger.info(f"Response: {response.status_code}")
        return response

app.add_middleware(PIIMaskingMiddleware)

@app.post("/users")
async def create_user(user: dict):
    # Even in application logs, use masked versions
    logger.info(f"Creating user: {mask_dict(user)}")
    return {"id": 1, "name": user.get("name")}
```

### Key Takeaways
> - 💡 **Never log** passwords, credit cards, SSNs, or API keys in plain text
> - 💡 Use a **middleware** that masks PII before writing to logs
> - 💡 Combine **field name matching** and **regex pattern matching** for thorough detection
> - 💡 Show **partial values** where useful (last 4 of card) — helps debugging without exposing PII
> - 💡 Compliance standards (GDPR, PCI-DSS, HIPAA) **require** PII masking in logs

---

## Question 88 — Handle Concurrent File Uploads to Same Resource
🟡 Mid | ★★☆ Common

### The Scenario
> *"Two users upload a profile picture at the same time for the same account. One upload overwrites the other silently. How do you handle concurrent file uploads safely?"*

### The Answer

**Step 1: The concurrency problem**

```
User A: Upload photo_A.jpg for user_123 → starts at T=0
User B: Upload photo_B.jpg for user_123 → starts at T=1
User A: Completes at T=5 → user_123.avatar = photo_A.jpg
User B: Completes at T=3 → user_123.avatar = photo_B.jpg (overwrites A!)
Final result: photo_B.jpg (User A's upload was silently lost)
```

| Strategy | How It Works | Best For |
|----------|-------------|----------|
| **Optimistic locking (ETag)** | Check version before write; reject if changed | Most cases |
| **Last-write-wins** | Accept it, latest upload wins | Profile pics, avatars |
| **Versioned storage** | Keep all versions (S3 versioning) | Documents, important files |

```mermaid
flowchart TD
    A[User uploads file] --> B[Include ETag/version\nin If-Match header]
    B --> C{Version matches\ncurrent?}
    C -->|Yes| D[Accept upload ✅]
    C -->|No| E[412 Precondition Failed\nSomeone else uploaded first]
    D --> F[Update ETag/version]
    E --> G[Client refreshes\nand retries]
```

### Code Example — Concurrent Upload with Optimistic Locking

```python
import hashlib
import uuid
from fastapi import FastAPI, UploadFile, File, Request, HTTPException
from datetime import datetime, timezone

app = FastAPI()

# Simulated file storage
files_db: dict[str, dict] = {}

def generate_etag(content: bytes) -> str:
    return hashlib.md5(content).hexdigest()

@app.post("/users/{user_id}/avatar")
async def upload_avatar(
    user_id: str,
    request: Request,
    file: UploadFile = File(...)
):
    content = await file.read()
    new_etag = generate_etag(content)

    current = files_db.get(f"{user_id}_avatar")
    if_match = request.headers.get("If-Match")

    # Optimistic locking: check ETag if provided
    if if_match and current:
        current_etag = current.get("etag", "")
        if if_match != f'"{current_etag}"':
            raise HTTPException(
                status_code=412,
                detail="File was modified by another user. Refresh and try again.",
                headers={"ETag": f'"{current_etag}"'}
            )

    # Store new version
    file_id = str(uuid.uuid4())
    files_db[f"{user_id}_avatar"] = {
        "file_id": file_id,
        "filename": file.filename,
        "etag": new_etag,
        "uploaded_at": datetime.now(timezone.utc).isoformat(),
        "content_length": len(content),
    }

    # Keep previous version for history
    history_key = f"{user_id}_avatar_history"
    if history_key not in files_db:
        files_db[history_key] = []
    if current:
        files_db[history_key].append(current)

    return {
        "file_id": file_id,
        "etag": f'"{new_etag}"',
        "message": "Avatar uploaded successfully"
    }

@app.get("/users/{user_id}/avatar")
async def get_avatar_info(user_id: str):
    current = files_db.get(f"{user_id}_avatar")
    if not current:
        raise HTTPException(404, "No avatar found")

    from fastapi.responses import JSONResponse
    response = JSONResponse(content=current)
    response.headers["ETag"] = f'"{current["etag"]}"'
    return response
```

### Key Takeaways
> - 💡 Use **ETags + If-Match** for optimistic locking on file uploads
> - 💡 Return `412 Precondition Failed` when concurrent modification is detected
> - 💡 Keep **version history** — don't permanently lose overwritten files
> - 💡 For non-critical files (avatars), **last-write-wins** may be acceptable
> - 💡 Upload to **S3 with versioning enabled** for automatic version tracking in production

---

## Question 89 — Design API Endpoint for Bulk Operations (Create/Update/Delete)
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API processes items one at a time, but clients need to create/update/delete hundreds of items at once. 100 individual API calls is too slow. How do you design a bulk operations endpoint?"*

### The Answer

**Step 1: Bulk endpoint design**

```
INSTEAD OF:
POST /products  (×100 calls = 100 HTTP round trips)

USE:
POST /products/bulk  (1 call with 100 items)

RESPONSE — Per-item status (some may fail, some may succeed):
{
  "total": 100,
  "succeeded": 97,
  "failed": 3,
  "results": [
    {"index": 0, "status": "created", "id": "prod_1"},
    {"index": 1, "status": "created", "id": "prod_2"},
    {"index": 2, "status": "error", "error": "Duplicate SKU"},
    ...
  ]
}
```

```mermaid
flowchart TD
    A[POST /products/bulk\n100 items] --> B[Validate all items]
    B --> C{All valid?}
    C -->|Yes| D[Process in batch\nDB bulk insert]
    C -->|Some invalid| E[Process valid items\nReport errors for invalid]
    D --> F[Return 200\nAll succeeded]
    E --> G[Return 207 Multi-Status\nPer-item results]
```

### Code Example — Bulk Operations Endpoint

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
from enum import Enum

app = FastAPI()

class BulkAction(str, Enum):
    CREATE = "create"
    UPDATE = "update"
    DELETE = "delete"

class BulkItem(BaseModel):
    action: BulkAction
    data: dict = {}
    id: str = None  # Required for update/delete

class BulkRequest(BaseModel):
    items: list[BulkItem] = Field(..., max_length=1000)  # Max 1000 per batch

class ItemResult(BaseModel):
    index: int
    action: str
    status: str  # "success" or "error"
    id: str = None
    error: str = None

# In-memory database
products_db: dict[str, dict] = {}

@app.post("/products/bulk")
async def bulk_operations(request: BulkRequest):
    """Process multiple create/update/delete operations in one request."""
    if len(request.items) > 1000:
        raise HTTPException(400, "Maximum 1000 items per bulk request")

    results = []
    succeeded = 0
    failed = 0

    for idx, item in enumerate(request.items):
        try:
            if item.action == BulkAction.CREATE:
                product_id = f"prod_{len(products_db) + 1}"
                products_db[product_id] = {**item.data, "id": product_id}
                results.append(ItemResult(
                    index=idx, action="create", status="success", id=product_id
                ))
                succeeded += 1

            elif item.action == BulkAction.UPDATE:
                if not item.id or item.id not in products_db:
                    results.append(ItemResult(
                        index=idx, action="update", status="error",
                        id=item.id, error="Product not found"
                    ))
                    failed += 1
                    continue
                products_db[item.id].update(item.data)
                results.append(ItemResult(
                    index=idx, action="update", status="success", id=item.id
                ))
                succeeded += 1

            elif item.action == BulkAction.DELETE:
                if not item.id or item.id not in products_db:
                    results.append(ItemResult(
                        index=idx, action="delete", status="error",
                        id=item.id, error="Product not found"
                    ))
                    failed += 1
                    continue
                del products_db[item.id]
                results.append(ItemResult(
                    index=idx, action="delete", status="success", id=item.id
                ))
                succeeded += 1

        except Exception as e:
            results.append(ItemResult(
                index=idx, action=item.action.value, status="error",
                error=str(e)
            ))
            failed += 1

    # Use 207 Multi-Status when there's a mix of success and failure
    from fastapi.responses import JSONResponse
    status_code = 200 if failed == 0 else 207

    return JSONResponse(
        status_code=status_code,
        content={
            "total": len(request.items),
            "succeeded": succeeded,
            "failed": failed,
            "results": [r.model_dump() for r in results],
        }
    )
```

### Key Takeaways
> - 💡 Limit batch size (e.g., **1000 items**) to prevent timeouts and memory issues
> - 💡 Return **per-item status** — don't fail the entire batch if one item fails
> - 💡 Use HTTP `207 Multi-Status` when some items succeed and some fail
> - 💡 Process in **database batches** (bulk insert) for performance, not one-by-one
> - 💡 For very large batches (10K+), use **async processing** (202 + job_id)

---

## Question 90 — API Needs Feature Flags — Enable Features Per User/Tenant
🔴 Senior | ★★☆ Common

### The Scenario
> *"You want to roll out a new API feature gradually — first to internal users, then to beta testers, then to everyone. How do you implement feature flags in a REST API?"*

### The Answer

**Step 1: Feature flag strategies**

| Strategy | How | Use Case |
|----------|-----|----------|
| **Boolean flag** | On/off for everyone | Kill switch, maintenance mode |
| **Percentage rollout** | Enable for N% of users | Gradual rollout |
| **User allowlist** | Enable for specific user IDs | Beta testers, internal users |
| **Tenant-based** | Enable per tenant/organization | Enterprise feature gating |
| **Datetime-based** | Enable after specific date/time | Scheduled launches |

```
FEATURE FLAG EVALUATION:

Request from user_123 → Check feature "new_search_v2"
│
├─ Is user in allowlist? → Yes → Feature ON ✅
├─ Is user's tenant enabled? → Check tenant flags
├─ Is percentage rollout active? → hash(user_id) % 100 < 25? → 25% rollout
└─ Default → Feature OFF
```

```mermaid
flowchart TD
    A[Request arrives] --> B[Extract user context\nuser_id, tenant, role]
    B --> C{Feature flag check}
    C --> D{In allowlist?}
    D -->|Yes| E[Feature ON ✅]
    D -->|No| F{Tenant enabled?}
    F -->|Yes| E
    F -->|No| G{Percentage rollout?}
    G -->|hash user_id in range| E
    G -->|Not in range| H[Feature OFF ❌]
    E --> I[Execute new code path]
    H --> J[Execute old code path]
```

### Code Example — Feature Flag System

```python
import hashlib
from datetime import datetime, timezone
from fastapi import FastAPI, Request, Depends

app = FastAPI()

# Feature flag configuration (in production: use LaunchDarkly/Unleash/database)
FEATURE_FLAGS = {
    "new_search_v2": {
        "enabled": True,
        "rollout_percentage": 25,      # 25% of users
        "allowlist": ["user_admin", "user_beta1", "user_beta2"],
        "tenant_allowlist": ["acme_corp"],
        "start_date": "2024-01-15T00:00:00Z",
    },
    "bulk_export": {
        "enabled": True,
        "rollout_percentage": 100,     # Everyone
        "allowlist": [],
        "tenant_allowlist": [],
    },
    "ai_recommendations": {
        "enabled": False,              # Kill switch — off for everyone
        "rollout_percentage": 0,
        "allowlist": ["user_admin"],   # Except admin for testing
        "tenant_allowlist": [],
    },
}

def is_feature_enabled(feature: str, user_id: str, tenant_id: str = None) -> bool:
    """Evaluate if a feature flag is enabled for this user."""
    flag = FEATURE_FLAGS.get(feature)
    if not flag or not flag.get("enabled"):
        return False

    # Check allowlist (explicit opt-in)
    if user_id in flag.get("allowlist", []):
        return True

    # Check tenant allowlist
    if tenant_id and tenant_id in flag.get("tenant_allowlist", []):
        return True

    # Check date gate
    start_date = flag.get("start_date")
    if start_date:
        if datetime.now(timezone.utc) < datetime.fromisoformat(start_date):
            return False

    # Percentage rollout (deterministic — same user always gets same result)
    percentage = flag.get("rollout_percentage", 0)
    if percentage >= 100:
        return True
    if percentage <= 0:
        return False

    # Deterministic hash ensures same user always sees same variant
    hash_value = int(hashlib.md5(f"{feature}:{user_id}".encode()).hexdigest(), 16)
    return (hash_value % 100) < percentage

def get_user_context(request: Request) -> dict:
    return {
        "user_id": request.headers.get("X-User-ID", "anonymous"),
        "tenant_id": request.headers.get("X-Tenant-ID"),
    }

@app.get("/search")
async def search(q: str, request: Request):
    ctx = get_user_context(request)

    if is_feature_enabled("new_search_v2", ctx["user_id"], ctx["tenant_id"]):
        # New search implementation
        return {
            "results": [{"title": "Result (v2 algorithm)", "score": 0.95}],
            "feature": "new_search_v2",
            "version": "v2"
        }
    else:
        # Old search implementation
        return {
            "results": [{"title": "Result (v1 algorithm)"}],
            "version": "v1"
        }

@app.get("/features")
async def get_features(request: Request):
    """Return which features are enabled for this user."""
    ctx = get_user_context(request)
    features = {}
    for feature_name in FEATURE_FLAGS:
        features[feature_name] = is_feature_enabled(
            feature_name, ctx["user_id"], ctx["tenant_id"]
        )
    return {"user": ctx["user_id"], "features": features}
```

### Key Takeaways
> - 💡 Use **deterministic hashing** (not random) — same user always sees the same variant
> - 💡 Support multiple strategies: **allowlist**, **percentage rollout**, **tenant-based**, **date-based**
> - 💡 Always have a **kill switch** — be able to turn off any feature instantly
> - 💡 In production, use dedicated services: **LaunchDarkly**, **Unleash**, **Flagsmith**
> - 💡 Clean up old feature flags — they become **tech debt** if left indefinitely

---

*⬅️ Previous: [08 — Theoretical & Conceptual](./08-theoretical-conceptual.md)*

*📝 Next: [Interview Tips →](./INTERVIEW_TIPS.md)*

*⚡ Quick Reference: [All 113 Questions →](./QUICK_REFERENCE.md)*
