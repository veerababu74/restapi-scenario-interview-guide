# 04 — Architecture & Design

> **Questions 31–40** | Designing scalable, maintainable API architectures

---

## Question 31 — Monolithic Flask API with 200+ Endpoints
🔴 Senior | ★★☆ Common

### The Scenario
> *"You inherit a Flask app with 200+ endpoints all in one `app.py` file. It's impossible to navigate, test, or maintain. How do you restructure it?"*

### The Answer

```
MONOLITH STRUCTURE (current — problematic):
app.py (5000 lines)
├── User endpoints (50 routes)
├── Order endpoints (40 routes)
├── Product endpoints (60 routes)
├── Payment endpoints (30 routes)
└── Admin endpoints (20 routes)
     ↓ Everything mixed together

MODULAR STRUCTURE (target — maintainable):
app/
├── __init__.py
├── main.py (just creates app, registers blueprints)
├── domains/
│   ├── users/
│   │   ├── router.py      (routes)
│   │   ├── models.py      (DB models)
│   │   ├── schemas.py     (Pydantic schemas)
│   │   ├── services.py    (business logic)
│   │   └── tests/
│   ├── orders/
│   │   ├── router.py
│   │   └── ...
│   └── products/
│       └── ...
└── core/
    ├── auth.py
    ├── database.py
    └── config.py
```

### Code Example — Flask Blueprints + FastAPI Routers

```python
# FastAPI Router approach (recommended)
# File: app/domains/users/router.py
from fastapi import APIRouter, Depends, HTTPException
from .schemas import UserCreate, UserResponse, UserUpdate
from .services import UserService
from app.core.auth import get_current_user
from app.core.database import get_db

router = APIRouter(
    prefix="/users",
    tags=["users"],
    responses={404: {"description": "User not found"}},
)

@router.get("/", response_model=list[UserResponse])
async def list_users(
    skip: int = 0,
    limit: int = 100,
    db=Depends(get_db),
    current_user=Depends(get_current_user)
):
    service = UserService(db)
    return await service.list_users(skip=skip, limit=limit)

@router.post("/", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate, db=Depends(get_db)):
    service = UserService(db)
    return await service.create_user(user)

@router.get("/{user_id}", response_model=UserResponse)
async def get_user(user_id: int, db=Depends(get_db)):
    service = UserService(db)
    user = await service.get_user(user_id)
    if not user:
        raise HTTPException(404, "User not found")
    return user

@router.put("/{user_id}", response_model=UserResponse)
async def update_user(user_id: int, update: UserUpdate, db=Depends(get_db)):
    service = UserService(db)
    return await service.update_user(user_id, update)

@router.delete("/{user_id}", status_code=204)
async def delete_user(user_id: int, db=Depends(get_db)):
    service = UserService(db)
    await service.delete_user(user_id)

# File: app/main.py
from fastapi import FastAPI
from app.domains.users.router import router as users_router
from app.domains.orders.router import router as orders_router
from app.domains.products.router import router as products_router
from app.domains.payments.router import router as payments_router
from app.core.config import settings
from app.core.middleware import setup_middleware

def create_app() -> FastAPI:
    app = FastAPI(
        title="My API",
        version="1.0.0",
        docs_url="/docs" if settings.DEBUG else None,  # Disable docs in prod
    )
    
    # Register routers with versioning
    api_v1_prefix = "/api/v1"
    app.include_router(users_router, prefix=api_v1_prefix)
    app.include_router(orders_router, prefix=api_v1_prefix)
    app.include_router(products_router, prefix=api_v1_prefix)
    app.include_router(payments_router, prefix=api_v1_prefix)
    
    setup_middleware(app)
    return app

app = create_app()

# Flask Blueprint equivalent:
"""
# app/blueprints/users.py
from flask import Blueprint, jsonify, request

users_bp = Blueprint('users', __name__, url_prefix='/api/v1/users')

@users_bp.route('/', methods=['GET'])
def list_users():
    return jsonify(UserService.list_users())

@users_bp.route('/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = UserService.get_user(user_id)
    if not user:
        return jsonify({'error': 'Not found'}), 404
    return jsonify(user)

# app/__init__.py
from flask import Flask
from .blueprints.users import users_bp
from .blueprints.orders import orders_bp

def create_app():
    app = Flask(__name__)
    app.register_blueprint(users_bp)
    app.register_blueprint(orders_bp)
    return app
"""
```

### Key Takeaways
> - 💡 **Domain-driven structure** groups related code (routes + models + services)
> - 💡 **One router/blueprint per domain** — max 20-30 routes per file
> - 💡 **Separate business logic** into `services.py` — routes just call services
> - 💡 **Shared code** goes in `core/` (auth, db, config)
> - 💡 **Do this incrementally** — migrate one domain at a time, not all at once

---

## Question 32 — Design API for Real-Time Chat + REST
🔴 Senior | ★★☆ Common

### The Scenario
> *"You need to build a chat application with a REST API for CRUD operations AND real-time message delivery. How do you design this hybrid architecture?"*

### The Answer

```
HYBRID ARCHITECTURE:

REST API (for CRUD):          WebSocket (for real-time):
├── POST /rooms               ├── ws://api/ws/{room_id}
├── GET /rooms/{id}/messages  ├── Send: { type: "message", content: "..." }
├── PUT /messages/{id}        └── Receive: { type: "new_message", data: {...} }
└── DELETE /messages/{id}

USE REST WHEN:                USE WEBSOCKET WHEN:
- Fetching message history    - Delivering new messages instantly
- Creating/editing rooms      - Presence (who's online)
- User management             - Typing indicators
- File uploads                - Read receipts
```

```mermaid
flowchart TB
    subgraph Client
        A[Web App]
    end
    
    subgraph "API Layer"
        B[REST API\nFastAPI]
        C[WebSocket Handler\nFastAPI]
    end
    
    subgraph "Data Layer"
        D[(PostgreSQL\nMessages)]
        E[(Redis\nPub/Sub\n& Presence)]
    end
    
    A -->|GET/POST/PUT\nMessage history\nRoom management| B
    A <-->|ws:// connection\nReal-time messages\nTyping indicators| C
    B --> D
    C --> E
    E -->|Pub/Sub| C
    C --> D
```

### Code Example — WebSocket + REST Hybrid

```python
import asyncio
import json
from typing import Dict, Set
from fastapi import FastAPI, WebSocket, WebSocketDisconnect, Depends, HTTPException
from fastapi.routing import APIRouter
import redis.asyncio as redis_async

app = FastAPI()

# ---- Connection Manager ----
class ConnectionManager:
    """Manages WebSocket connections across rooms"""
    
    def __init__(self):
        # room_id -> set of websocket connections
        self.rooms: Dict[str, Set[WebSocket]] = {}
    
    async def connect(self, room_id: str, websocket: WebSocket, user_id: int):
        await websocket.accept()
        if room_id not in self.rooms:
            self.rooms[room_id] = set()
        self.rooms[room_id].add(websocket)
        websocket.state.user_id = user_id
        
        # Notify others of new user
        await self.broadcast_to_room(
            room_id,
            {"type": "user_joined", "user_id": user_id},
            exclude=websocket
        )
    
    def disconnect(self, room_id: str, websocket: WebSocket):
        if room_id in self.rooms:
            self.rooms[room_id].discard(websocket)
            if not self.rooms[room_id]:
                del self.rooms[room_id]
    
    async def broadcast_to_room(
        self,
        room_id: str,
        message: dict,
        exclude: WebSocket = None
    ):
        """Send message to all connections in a room"""
        if room_id not in self.rooms:
            return
        
        disconnected = []
        for connection in self.rooms[room_id]:
            if connection == exclude:
                continue
            try:
                await connection.send_json(message)
            except Exception:
                disconnected.append(connection)
        
        # Cleanup disconnected clients
        for conn in disconnected:
            self.rooms[room_id].discard(conn)
    
    def get_room_users(self, room_id: str) -> list[int]:
        if room_id not in self.rooms:
            return []
        return [
            ws.state.user_id
            for ws in self.rooms[room_id]
            if hasattr(ws.state, "user_id")
        ]

manager = ConnectionManager()

# ---- REST endpoints ----
rest_router = APIRouter(prefix="/api/v1", tags=["rooms"])

@rest_router.post("/rooms/{room_id}/messages")
async def send_message(room_id: str, content: str, user_id: int):
    """
    REST: Save message to DB + broadcast via WebSocket
    """
    # Save to database
    message = {
        "id": "msg_123",
        "room_id": room_id,
        "user_id": user_id,
        "content": content,
        "created_at": "2024-01-01T12:00:00Z"
    }
    # await db.save_message(message)
    
    # Broadcast to WebSocket connections in real-time
    await manager.broadcast_to_room(
        room_id,
        {"type": "new_message", "data": message}
    )
    
    return message

@rest_router.get("/rooms/{room_id}/messages")
async def get_message_history(room_id: str, limit: int = 50, before_id: str = None):
    """REST: Get paginated message history"""
    # Return last N messages before the given message ID
    return {"messages": [], "has_more": False}

@rest_router.get("/rooms/{room_id}/presence")
async def get_presence(room_id: str):
    """Who is currently connected to this room?"""
    return {"online_users": manager.get_room_users(room_id)}

app.include_router(rest_router)

# ---- WebSocket endpoint ----
@app.websocket("/ws/{room_id}")
async def websocket_endpoint(websocket: WebSocket, room_id: str, user_id: int = 1):
    """
    WebSocket for real-time features:
    - New messages (instant delivery)
    - Typing indicators
    - Presence (online/offline)
    """
    await manager.connect(room_id, websocket, user_id)
    
    try:
        while True:
            data = await websocket.receive_json()
            message_type = data.get("type")
            
            if message_type == "message":
                # Save message and broadcast
                message = {
                    "id": "msg_456",
                    "user_id": user_id,
                    "content": data.get("content"),
                    "room_id": room_id
                }
                await manager.broadcast_to_room(
                    room_id,
                    {"type": "new_message", "data": message}
                )
            
            elif message_type == "typing":
                # Broadcast typing indicator (don't save to DB)
                await manager.broadcast_to_room(
                    room_id,
                    {"type": "user_typing", "user_id": user_id},
                    exclude=websocket  # Don't send back to the typer
                )
            
            elif message_type == "read_receipt":
                # Mark messages as read
                await manager.broadcast_to_room(
                    room_id,
                    {"type": "messages_read", "user_id": user_id, "up_to": data.get("message_id")}
                )
    
    except WebSocketDisconnect:
        manager.disconnect(room_id, websocket)
        await manager.broadcast_to_room(
            room_id,
            {"type": "user_left", "user_id": user_id}
        )
```

### Key Takeaways
> - 💡 **REST for persistence** (CRUD), **WebSocket for real-time** (instant delivery)
> - 💡 **ConnectionManager** tracks who's connected to each room
> - 💡 **For multi-server**: use Redis Pub/Sub so a WebSocket server can broadcast to connections on other servers
> - 💡 **Heartbeat/ping-pong**: detect dead connections (client disconnects without sending close frame)
> - 💡 **SSE is simpler** if you only need server-to-client (no client-to-server messages)

---

## Question 33 — Support Multiple API Versions Simultaneously
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API has v1 users with mobile apps that can't be force-updated. You need to release v2 with breaking changes. How do you support both versions simultaneously?"*

### The Answer

```
API VERSIONING STRATEGIES COMPARISON:

Strategy        │ Example                    │ Pros           │ Cons
────────────────┼────────────────────────────┼────────────────┼─────────────────
URL Path        │ /api/v1/users              │ Obvious,       │ URL changes,
(Recommended)   │ /api/v2/users              │ cacheable      │ code duplication
────────────────┼────────────────────────────┼────────────────┼─────────────────
Header          │ Accept: application/vnd    │ Clean URLs     │ Less visible,
                │ .api.v2+json               │                │ harder to test
────────────────┼────────────────────────────┼────────────────┼─────────────────
Query Param     │ /users?version=2           │ Easy to test   │ Can be cached
                │                            │                │ incorrectly
────────────────┼────────────────────────────┼────────────────┼─────────────────
Custom Header   │ X-API-Version: 2           │ Flexible       │ Non-standard
```

### Code Example — URL-Based Versioning with Shared Logic

```python
from fastapi import FastAPI, APIRouter, Depends
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

# ---- Shared service layer (no duplication) ----
class UserService:
    async def get_user(self, user_id: int) -> dict:
        """Core business logic — shared between all versions"""
        return {
            "id": user_id,
            "first_name": "John",
            "last_name": "Doe",
            "email": "john@example.com",
            "phone": "+1234567890",
            "created_at": "2024-01-01T00:00:00Z"
        }

# ---- V1 Schema (old format) ----
class UserResponseV1(BaseModel):
    id: int
    name: str  # Old: combined name
    email: str

# ---- V2 Schema (new format — breaking change) ----
class UserResponseV2(BaseModel):
    id: int
    first_name: str   # New: split into first/last
    last_name: str    # New: split into first/last
    email: str
    phone: Optional[str] = None  # New field

# ---- V1 Router ----
router_v1 = APIRouter(prefix="/api/v1", tags=["v1"])

@router_v1.get("/users/{user_id}", response_model=UserResponseV1)
async def get_user_v1(user_id: int):
    """V1: Returns combined name (old format)"""
    service = UserService()
    user = await service.get_user(user_id)
    
    # Transform to V1 format
    return UserResponseV1(
        id=user["id"],
        name=f"{user['first_name']} {user['last_name']}",  # Combine for V1
        email=user["email"]
    )

# ---- V2 Router ----
router_v2 = APIRouter(prefix="/api/v2", tags=["v2"])

@router_v2.get("/users/{user_id}", response_model=UserResponseV2)
async def get_user_v2(user_id: int):
    """V2: Returns split first/last name (new format)"""
    service = UserService()
    user = await service.get_user(user_id)
    
    return UserResponseV2(**user)  # Direct mapping

# Register both versions
app.include_router(router_v1)
app.include_router(router_v2)

# Add deprecation warning for V1
@app.middleware("http")
async def version_deprecation_middleware(request, call_next):
    response = await call_next(request)
    
    if "/api/v1/" in str(request.url):
        response.headers["Deprecation"] = "true"
        response.headers["Sunset"] = "2025-12-31"
        response.headers["Link"] = "</api/v2/>; rel=\"successor-version\""
    
    return response
```

### Key Takeaways
> - 💡 **URL versioning** is the most practical choice for most APIs
> - 💡 **Share the service layer** — only transform at the router layer
> - 💡 **Add `Deprecation` and `Sunset` headers** to v1 so clients know to migrate
> - 💡 **Keep v1 alive for 6-12 months** after v2 launch
> - 💡 **Never break a version** — add new fields, but don't remove or rename

---

## Question 34 — Design E-Commerce Checkout REST API
🔴 Senior | ★★☆ Common

### The Scenario
> *"Design the REST API for an e-commerce checkout flow: cart → shipping → payment → confirmation. Each step depends on the previous. How do you design this?"*

### The Answer

```
CHECKOUT SAGA PATTERN:

State Machine:
CART ──► ADDRESS ──► PAYMENT ──► CONFIRMED ──► FULFILLED
                                     │
                            PAYMENT_FAILED ──► (retry)

Each step creates a resource. Steps are idempotent.

API Design:
POST /orders                    → Create order from cart (status: pending)
PUT  /orders/{id}/address       → Add shipping address (status: address_added)
POST /orders/{id}/payment       → Process payment (status: payment_processing)
GET  /orders/{id}               → Poll status
```

### Code Example — Order State Machine

```python
from enum import Enum
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class OrderStatus(Enum):
    PENDING = "pending"
    ADDRESS_ADDED = "address_added"
    PAYMENT_PROCESSING = "payment_processing"
    PAYMENT_FAILED = "payment_failed"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

# Valid status transitions
VALID_TRANSITIONS = {
    OrderStatus.PENDING: [OrderStatus.ADDRESS_ADDED, OrderStatus.CANCELLED],
    OrderStatus.ADDRESS_ADDED: [OrderStatus.PAYMENT_PROCESSING, OrderStatus.CANCELLED],
    OrderStatus.PAYMENT_PROCESSING: [OrderStatus.CONFIRMED, OrderStatus.PAYMENT_FAILED],
    OrderStatus.PAYMENT_FAILED: [OrderStatus.PAYMENT_PROCESSING, OrderStatus.CANCELLED],
    OrderStatus.CONFIRMED: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
    OrderStatus.SHIPPED: [OrderStatus.DELIVERED],
    OrderStatus.DELIVERED: [],
    OrderStatus.CANCELLED: [],
}

class Order(BaseModel):
    id: str
    status: OrderStatus
    cart_items: list[dict]
    address: Optional[dict] = None
    payment: Optional[dict] = None
    total_amount: float

class AddressInput(BaseModel):
    street: str
    city: str
    country: str
    postal_code: str

class PaymentInput(BaseModel):
    payment_method: str  # "card", "paypal", etc.
    idempotency_key: str  # Client provides for retry safety

# In-memory order store (use database in production)
orders_db: dict[str, Order] = {}

def transition_order(order: Order, new_status: OrderStatus) -> Order:
    """Validate and apply state transition"""
    if new_status not in VALID_TRANSITIONS[order.status]:
        raise HTTPException(
            400,
            f"Cannot transition from {order.status.value} to {new_status.value}. "
            f"Valid transitions: {[s.value for s in VALID_TRANSITIONS[order.status]]}"
        )
    order.status = new_status
    return order

@app.post("/orders", status_code=201)
async def create_order(cart_id: str):
    """Step 1: Create order from cart"""
    # Fetch cart, validate items, reserve inventory
    order = Order(
        id="order_123",
        status=OrderStatus.PENDING,
        cart_items=[{"product_id": "prod_1", "qty": 2, "price": 29.99}],
        total_amount=59.98
    )
    orders_db[order.id] = order
    return order

@app.put("/orders/{order_id}/address")
async def add_address(order_id: str, address: AddressInput):
    """Step 2: Add shipping address"""
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(404, "Order not found")
    
    order = transition_order(order, OrderStatus.ADDRESS_ADDED)
    order.address = address.model_dump()
    orders_db[order_id] = order
    
    return order

@app.post("/orders/{order_id}/payment")
async def process_payment(order_id: str, payment: PaymentInput):
    """
    Step 3: Process payment
    IDEMPOTENT: Same idempotency_key returns same result
    """
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(404, "Order not found")
    
    # Check idempotency (if this payment was already processed)
    if order.status == OrderStatus.CONFIRMED:
        return order  # Already processed — return existing result
    
    order = transition_order(order, OrderStatus.PAYMENT_PROCESSING)
    orders_db[order_id] = order
    
    # Process payment asynchronously
    try:
        # await payment_gateway.charge(payment.payment_method, order.total_amount)
        payment_success = True  # Simulate
        
        if payment_success:
            order = transition_order(order, OrderStatus.CONFIRMED)
            order.payment = {"method": payment.payment_method, "status": "success"}
        else:
            order = transition_order(order, OrderStatus.PAYMENT_FAILED)
    except Exception as e:
        order = transition_order(order, OrderStatus.PAYMENT_FAILED)
    
    orders_db[order_id] = order
    return order

@app.get("/orders/{order_id}")
async def get_order(order_id: str):
    """Get order status and details"""
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(404, "Order not found")
    return order

@app.delete("/orders/{order_id}")
async def cancel_order(order_id: str):
    """Cancel order (compensating transaction)"""
    order = orders_db.get(order_id)
    if not order:
        raise HTTPException(404, "Order not found")
    
    order = transition_order(order, OrderStatus.CANCELLED)
    
    # Compensating transactions
    # - Release reserved inventory
    # - Refund payment if charged
    
    orders_db[order_id] = order
    return {"message": "Order cancelled", "order": order}
```

### Key Takeaways
> - 💡 **State machine pattern** prevents invalid transitions (can't pay cancelled order)
> - 💡 **Each step is a separate resource** — clean REST semantics
> - 💡 **Idempotency keys** for payment — safe to retry without double charging
> - 💡 **Compensating transactions** for rollback (cancel = release inventory + refund)
> - 💡 **Saga pattern** for distributed transactions across services

---

## Question 35 — Notify Clients When Data Changes
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your dashboard needs to show live updates when data changes. Users refresh the page manually to see new data. What are your options and which do you choose?"*

### The Answer

```
NOTIFICATION STRATEGIES COMPARISON:

Strategy      │ Use When                    │ Complexity │ Scalability
──────────────┼─────────────────────────────┼────────────┼────────────
Polling       │ Simple, infrequent updates  │ Low        │ High
(every 30s)   │ Mobile/battery sensitive    │            │
──────────────┼─────────────────────────────┼────────────┼────────────
Long Polling  │ Need near-real-time,        │ Medium     │ Medium
              │ can't use WebSocket         │            │
──────────────┼─────────────────────────────┼────────────┼────────────
SSE           │ Server pushes to browser    │ Low-Medium │ High
(Recommended) │ Dashboard, notifications    │            │
              │ One-directional             │            │
──────────────┼─────────────────────────────┼────────────┼────────────
WebSocket     │ Bidirectional, chat,        │ High       │ Medium
              │ collaborative editing       │            │
──────────────┼─────────────────────────────┼────────────┼────────────
Webhooks      │ Third-party notifications   │ Medium     │ High
              │ B2B integrations            │            │
```

### Code Example — Server-Sent Events (SSE)

```python
import asyncio
import json
import time
from typing import AsyncGenerator
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

# Simple event bus (use Redis Pub/Sub in production)
subscribers: dict[str, list[asyncio.Queue]] = {}

async def event_generator(
    request: Request,
    topic: str
) -> AsyncGenerator[str, None]:
    """Generate SSE events for a topic"""
    # Create queue for this subscriber
    queue: asyncio.Queue = asyncio.Queue()
    
    if topic not in subscribers:
        subscribers[topic] = []
    subscribers[topic].append(queue)
    
    try:
        # Send initial connection event
        yield f"data: {json.dumps({'type': 'connected', 'topic': topic})}\n\n"
        
        while True:
            # Check if client disconnected
            if await request.is_disconnected():
                break
            
            try:
                # Wait for event with timeout (keep-alive)
                event = await asyncio.wait_for(queue.get(), timeout=30)
                yield f"data: {json.dumps(event)}\n\n"
            except asyncio.TimeoutError:
                # Send keep-alive comment to prevent connection timeout
                yield ": keep-alive\n\n"
    
    finally:
        # Cleanup
        if topic in subscribers and queue in subscribers[topic]:
            subscribers[topic].remove(queue)

@app.get("/events/{topic}")
async def subscribe_to_events(request: Request, topic: str):
    """SSE endpoint — browser EventSource connects here"""
    return StreamingResponse(
        event_generator(request, topic),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # Disable nginx buffering
            "Connection": "keep-alive",
        }
    )

async def publish_event(topic: str, event: dict):
    """Publish event to all subscribers of a topic"""
    if topic not in subscribers:
        return
    
    dead_queues = []
    for queue in subscribers[topic]:
        try:
            await queue.put(event)
        except Exception:
            dead_queues.append(queue)
    
    for q in dead_queues:
        subscribers[topic].remove(q)

@app.post("/orders/{order_id}/status")
async def update_order_status(order_id: str, status: str):
    """When order status changes, notify all watchers via SSE"""
    # Update database
    # await db.update_order_status(order_id, status)
    
    # Publish event to SSE subscribers
    await publish_event(
        topic=f"order.{order_id}",
        event={
            "type": "order_status_changed",
            "order_id": order_id,
            "status": status,
            "timestamp": time.time()
        }
    )
    
    return {"order_id": order_id, "status": status}

# Frontend JavaScript to connect:
"""
const eventSource = new EventSource('/events/order.123');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    if (data.type === 'order_status_changed') {
        updateOrderStatusUI(data.status);
    }
};

eventSource.onerror = () => {
    // EventSource auto-reconnects on error
    console.log('Reconnecting...');
};
"""
```

### Key Takeaways
> - 💡 **SSE is the sweet spot** for dashboards — simple, HTTP-based, auto-reconnect
> - 💡 **WebSocket only needed** when clients also send frequent messages to server
> - 💡 **Webhooks for B2B** — push events to external systems
> - 💡 **For production SSE**: use Redis Pub/Sub so multiple server instances can broadcast
> - 💡 **Set `X-Accel-Buffering: no`** to prevent nginx from buffering SSE responses

---

## Question 36 — Multi-Tenant SaaS API Data Isolation
🔴 Senior | ★★☆ Common

### The Scenario
> *"You're building a multi-tenant SaaS application. You need to ensure each company's data is completely isolated — Company A cannot access Company B's data. How do you design this?"*

### The Answer

```
MULTI-TENANCY MODELS:

Model           │ Isolation │ Cost      │ Complexity │ Best For
────────────────┼───────────┼───────────┼────────────┼────────────────
DB per tenant   │ Maximum   │ High      │ High       │ Healthcare,
                │           │           │            │ Finance (strict)
────────────────┼───────────┼───────────┼────────────┼────────────────
Schema per      │ High      │ Medium    │ Medium     │ Mid-size SaaS
tenant          │           │           │            │
────────────────┼───────────┼───────────┼────────────┼────────────────
Row-level       │ Medium    │ Low       │ Low        │ Most SaaS apps
security        │           │           │            │ (RLS in Postgres)
────────────────┼───────────┼───────────┼────────────┼────────────────
Application     │ Low       │ Lowest    │ Lowest     │ Simple apps
filtering       │           │           │            │ (risky!)
```

### Code Example — Row-Level Security + Tenant Middleware

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from sqlalchemy import event, text
from sqlalchemy.ext.asyncio import AsyncSession
from pydantic import BaseModel
from typing import Optional

app = FastAPI()

class TenantContext:
    """Thread-local tenant context"""
    def __init__(self, tenant_id: str, plan: str):
        self.tenant_id = tenant_id
        self.plan = plan

async def get_tenant(request: Request) -> TenantContext:
    """
    Extract tenant from request.
    Options: subdomain, JWT claim, API key lookup, header
    """
    # Method 1: From subdomain (acme.yourapp.com)
    host = request.headers.get("host", "")
    subdomain = host.split(".")[0]
    
    # Method 2: From JWT claim
    # token = request.headers.get("Authorization", "").replace("Bearer ", "")
    # payload = jwt.decode(token, SECRET_KEY, ...)
    # tenant_id = payload.get("tenant_id")
    
    # Lookup tenant in database
    tenant = await lookup_tenant(subdomain)
    if not tenant:
        raise HTTPException(404, "Tenant not found")
    
    return TenantContext(tenant_id=tenant["id"], plan=tenant["plan"])

async def get_db_with_tenant(
    tenant: TenantContext = Depends(get_tenant)
) -> AsyncSession:
    """
    Get database session with tenant isolation via PostgreSQL RLS.
    """
    async with AsyncSession(engine) as session:
        # Set tenant context for Row Level Security
        await session.execute(
            text("SET app.current_tenant_id = :tenant_id"),
            {"tenant_id": tenant.tenant_id}
        )
        yield session

# Models with tenant_id on every table
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column
from sqlalchemy import String, Integer, ForeignKey

class Base(DeclarativeBase):
    pass

class Product(Base):
    __tablename__ = "products"
    id: Mapped[int] = mapped_column(primary_key=True)
    tenant_id: Mapped[str] = mapped_column(String, index=True)  # Required!
    name: Mapped[str] = mapped_column(String)

# SAFE: Always include tenant_id in queries
from sqlalchemy import select

@app.get("/products")
async def list_products(
    tenant: TenantContext = Depends(get_tenant),
    db: AsyncSession = Depends(get_db_with_tenant)
):
    """Always filter by tenant_id"""
    result = await db.execute(
        select(Product).where(Product.tenant_id == tenant.tenant_id)
    )
    return result.scalars().all()

# PostgreSQL Row Level Security setup
"""
-- Enable RLS on all tenant tables
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Create policy: each user can only see their tenant's data
CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.current_tenant_id'));

-- Now even if someone forgets the .where(tenant_id=...) in app code,
-- PostgreSQL enforces it at the database level!
"""

async def lookup_tenant(subdomain: str) -> Optional[dict]:
    """Look up tenant by subdomain"""
    return {"id": "tenant_acme", "plan": "pro"}

from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
```

### Key Takeaways
> - 💡 **Row-Level Security (PostgreSQL)** is the recommended approach for most SaaS
> - 💡 **Always filter by tenant_id** in every query — never trust app-layer only
> - 💡 **Add tenant_id to every table** — index it for performance
> - 💡 **RLS as defense-in-depth** — catches bugs even if app forgets to filter
> - 💡 **Use subdomains** for tenant identification — easy to configure, clear UX

---

## Question 37 — API Needs Both Sync and Async Operations
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Some API operations complete in 100ms (sync). Others take 5 minutes (async: report generation, data export). How do you design an API that handles both patterns cleanly?"*

### The Answer

```
SYNC VS ASYNC API PATTERN:

SYNC (fast operations < 5 seconds):
  POST /users → 201 { "id": 1, "name": "John" }
  
ASYNC (slow operations > 5 seconds):
  POST /reports/generate → 202 Accepted { "job_id": "job_123", "status_url": "/jobs/job_123" }
  
  GET /jobs/job_123 → { "status": "processing", "progress": 45 }
  GET /jobs/job_123 → { "status": "completed", "result_url": "/reports/download/abc" }
  
  GET /reports/download/abc → [file download or JSON result]
```

### Code Example — Async Job Pattern

```python
import asyncio
import uuid
import time
from enum import Enum
from typing import Optional, Any
from fastapi import FastAPI, BackgroundTasks, HTTPException
from pydantic import BaseModel

app = FastAPI()

class JobStatus(Enum):
    QUEUED = "queued"
    PROCESSING = "processing"
    COMPLETED = "completed"
    FAILED = "failed"

class Job(BaseModel):
    job_id: str
    status: JobStatus
    progress: int = 0
    created_at: float
    completed_at: Optional[float] = None
    result: Optional[Any] = None
    error: Optional[str] = None

# In-memory job store (use Redis in production)
jobs: dict[str, Job] = {}

async def run_slow_job(job_id: str, params: dict):
    """Simulate a slow background task"""
    jobs[job_id].status = JobStatus.PROCESSING
    
    try:
        # Simulate work with progress updates
        for i in range(10):
            await asyncio.sleep(0.5)  # Simulate work
            jobs[job_id].progress = (i + 1) * 10
        
        # Store result
        jobs[job_id].status = JobStatus.COMPLETED
        jobs[job_id].completed_at = time.time()
        jobs[job_id].result = {"report_data": [1, 2, 3], "params": params}
        
    except Exception as e:
        jobs[job_id].status = JobStatus.FAILED
        jobs[job_id].error = str(e)

@app.post("/reports/generate", status_code=202)
async def generate_report(
    report_type: str,
    date_from: str,
    date_to: str,
    background_tasks: BackgroundTasks
):
    """
    Async endpoint: starts job, returns 202 Accepted immediately.
    202 = "Request accepted for processing, not yet complete"
    """
    job_id = str(uuid.uuid4())
    
    jobs[job_id] = Job(
        job_id=job_id,
        status=JobStatus.QUEUED,
        created_at=time.time()
    )
    
    # Start background processing
    params = {"type": report_type, "from": date_from, "to": date_to}
    background_tasks.add_task(run_slow_job, job_id, params)
    
    return {
        "job_id": job_id,
        "status": "queued",
        "status_url": f"/jobs/{job_id}",
        "estimated_completion_seconds": 30
    }

@app.get("/jobs/{job_id}")
async def get_job_status(job_id: str):
    """Poll for job status"""
    job = jobs.get(job_id)
    if not job:
        raise HTTPException(404, "Job not found")
    
    response = {
        "job_id": job.job_id,
        "status": job.status.value,
        "progress": job.progress,
        "created_at": job.created_at,
    }
    
    if job.status == JobStatus.COMPLETED:
        response["result"] = job.result
        response["completed_at"] = job.completed_at
        response["download_url"] = f"/reports/{job_id}/download"
    
    elif job.status == JobStatus.FAILED:
        response["error"] = job.error
    
    elif job.status == JobStatus.PROCESSING:
        # Suggest polling interval based on progress
        response["retry_after_seconds"] = max(2, 30 - int(job.progress / 4))
    
    return response

@app.get("/reports/{job_id}/download")
async def download_report(job_id: str):
    """Download completed report"""
    job = jobs.get(job_id)
    if not job or job.status != JobStatus.COMPLETED:
        raise HTTPException(404, "Report not found or not ready")
    
    # Return result or redirect to S3 pre-signed URL
    return job.result
```

### Key Takeaways
> - 💡 **202 Accepted** = "I received it but haven't processed it yet" (perfect for async)
> - 💡 **Return job_id + status_url** so client knows where to poll
> - 💡 **Suggest retry interval** in response to help clients poll efficiently
> - 💡 **Webhooks as alternative** to polling — push result when done
> - 💡 **Use Celery** for production background jobs (not BackgroundTasks for long jobs)

---

## Question 38 — Implement Search Across Multiple Entities
🟡 Mid | ★★☆ Common

### The Scenario
> *"Users want a global search that finds products, users, orders, and blog posts from a single search box. How do you design this?"*

### The Answer

### Code Example — Multi-Entity Search

```python
import asyncio
from typing import Optional
from fastapi import FastAPI, Query
from pydantic import BaseModel

app = FastAPI()

class SearchResult(BaseModel):
    entity_type: str
    id: str
    title: str
    description: Optional[str] = None
    score: float
    url: str

@app.get("/search")
async def global_search(
    q: str = Query(..., min_length=2, max_length=100),
    types: Optional[str] = Query(None, description="Comma-separated: products,users,orders"),
    limit: int = Query(10, le=50)
):
    """
    Global search across multiple entity types.
    Runs all searches in parallel for speed.
    """
    if len(q.strip()) < 2:
        raise HTTPException(400, "Query must be at least 2 characters")
    
    # Determine which entities to search
    search_types = set(types.split(",")) if types else {"products", "users", "orders"}
    
    # Run all searches in parallel
    search_tasks = {}
    if "products" in search_types:
        search_tasks["products"] = search_products(q, limit)
    if "users" in search_types:
        search_tasks["users"] = search_users(q, limit)
    if "orders" in search_types:
        search_tasks["orders"] = search_orders(q, limit)
    
    results = await asyncio.gather(*search_tasks.values(), return_exceptions=True)
    
    # Combine and sort by score
    all_results = []
    for entity_type, result in zip(search_tasks.keys(), results):
        if isinstance(result, Exception):
            print(f"Search failed for {entity_type}: {result}")
            continue
        all_results.extend(result)
    
    # Sort by relevance score
    all_results.sort(key=lambda x: x.score, reverse=True)
    
    return {
        "query": q,
        "total": len(all_results),
        "results": all_results[:limit]
    }

async def search_products(query: str, limit: int) -> list[SearchResult]:
    """Search products — in production, use Elasticsearch"""
    # Stub: return mock results
    return [
        SearchResult(
            entity_type="product",
            id="prod_1",
            title=f"Product matching '{query}'",
            description="Great product",
            score=0.95,
            url=f"/products/prod_1"
        )
    ]

async def search_users(query: str, limit: int) -> list[SearchResult]:
    return []

async def search_orders(query: str, limit: int) -> list[SearchResult]:
    return []

from fastapi import HTTPException
```

### Key Takeaways
> - 💡 **Elasticsearch** for full-text search at scale (SQL LIKE queries don't scale)
> - 💡 **Run searches in parallel** with asyncio.gather — not sequential
> - 💡 **Return entity_type** so frontend knows how to render each result
> - 💡 **Allow filtering by type** — users often want specific entity types
> - 💡 **Implement autocomplete** separately using a prefix trie or Elasticsearch suggest

---

## Question 39 — Handle File Processing Without Blocking
🟡 Mid | ★★☆ Common

### The Scenario
> *"Users upload images that need resizing and videos that need transcoding. These take 30 seconds to 5 minutes. How do you design this without blocking the API?"*

### The Answer

```
ASYNC FILE PROCESSING PIPELINE:

1. Client uploads file ──► API (< 1 second response with job_id)
2. File saved to S3
3. S3 triggers Lambda or Celery task
4. Worker processes file (resize/transcode)
5. Result saved to S3 with CDN URL
6. Client polls status OR receives webhook notification

┌────────┐   ┌─────┐   ┌────┐   ┌────────┐   ┌─────┐   ┌────┐
│ Client │──►│ API │──►│ S3 │──►│ Queue  │──►│ Worker│──►│ CDN│
│        │◄──│     │   │    │   │(Redis) │   │       │   │    │
│ job_id │   │     │   │    │   │        │   │(celery│   │    │
└────────┘   └─────┘   └────┘   └────────┘   └───────┘   └────┘
     │                                                       │
     └────────── GET /jobs/{id} ────────────► result_url ───┘
```

### Code Example — Async Image Processing

```python
from fastapi import FastAPI, UploadFile, BackgroundTasks
from celery import Celery
import uuid

app = FastAPI()
celery_app = Celery("tasks", broker="redis://localhost:6379/0")

@app.post("/images/upload")
async def upload_image(file: UploadFile, background_tasks: BackgroundTasks):
    """Upload image and start async processing"""
    job_id = str(uuid.uuid4())
    
    # Save original to S3/storage
    original_key = f"originals/{job_id}/{file.filename}"
    # await s3.upload(original_key, file)
    
    # Queue processing job
    process_image_task.delay(job_id, original_key)
    
    return {
        "job_id": job_id,
        "status": "processing",
        "status_url": f"/jobs/{job_id}"
    }

@celery_app.task
def process_image_task(job_id: str, s3_key: str):
    """Worker: resize image to multiple sizes"""
    from PIL import Image
    import io
    
    sizes = {"thumbnail": (150, 150), "medium": (800, 600), "large": (1920, 1080)}
    results = {}
    
    for size_name, dimensions in sizes.items():
        # Download, resize, re-upload
        # img = download_from_s3(s3_key)
        # resized = img.resize(dimensions)
        # upload_to_s3(f"processed/{job_id}/{size_name}.jpg", resized)
        results[size_name] = f"https://cdn.example.com/processed/{job_id}/{size_name}.jpg"
    
    # Update job status in Redis
    import redis
    r = redis.Redis()
    import json
    r.setex(f"job:{job_id}", 86400, json.dumps({
        "status": "completed",
        "urls": results
    }))
```

### Key Takeaways
> - 💡 **Return job_id immediately** — never make clients wait for processing
> - 💡 **Pre-signed S3 URLs** for uploads — offload upload bandwidth from your API
> - 💡 **Celery workers** can scale independently of API servers
> - 💡 **CDN for processed files** — serve from edge nodes for low latency
> - 💡 **Webhooks to notify** clients when processing completes

---

## Question 40 — API Gateway for 20 Microservices
🔴 Senior | ★★☆ Common

### The Scenario
> *"You have 20 microservices and clients need to call all of them. Managing 20 different URLs, auth systems, and rate limits is a nightmare. How do you design an API Gateway?"*

### The Answer

```
API GATEWAY ARCHITECTURE:

Clients (web/mobile/3rd party)
         │
         ▼
┌─────────────────────────────────────────────────┐
│              API GATEWAY                         │
│  ┌──────────┐ ┌──────────┐ ┌──────────────────┐ │
│  │   Auth   │ │  Rate    │ │    Routing       │ │
│  │  (JWT)   │ │ Limiting │ │  /users → svc_a  │ │
│  └──────────┘ └──────────┘ │  /orders → svc_b │ │
│  ┌──────────┐ ┌──────────┐ │  /products→svc_c │ │
│  │  Logging │ │ Circuit  │ └──────────────────┘ │
│  │ Tracing  │ │ Breaker  │                      │
│  └──────────┘ └──────────┘                      │
└──────────────────┬──────────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
 User Service  Order Service  Product Service
 (internal)    (internal)     (internal)
```

### Code Example — Simple API Gateway with FastAPI

```python
import httpx
import asyncio
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import Response
from typing import Optional

app = FastAPI(title="API Gateway")

# Service registry
SERVICES = {
    "users": "http://user-service:8001",
    "orders": "http://order-service:8002",
    "products": "http://product-service:8003",
    "payments": "http://payment-service:8004",
}

# Route configuration: path prefix → service
ROUTES = {
    "/api/v1/users": "users",
    "/api/v1/orders": "orders",
    "/api/v1/products": "products",
    "/api/v1/payments": "payments",
}

async def route_request(request: Request, service_name: str, path: str) -> Response:
    """Forward request to downstream service"""
    service_url = SERVICES.get(service_name)
    if not service_url:
        raise HTTPException(503, f"Service {service_name} not configured")
    
    target_url = f"{service_url}{path}"
    
    # Forward request with all headers
    headers = {
        k: v for k, v in request.headers.items()
        if k.lower() not in ["host", "content-length"]
    }
    
    # Add internal headers
    headers["X-Gateway-Request-ID"] = request.headers.get("X-Request-ID", "")
    headers["X-Forwarded-For"] = request.client.host
    
    async with httpx.AsyncClient(timeout=30.0) as client:
        try:
            body = await request.body()
            response = await client.request(
                method=request.method,
                url=target_url,
                headers=headers,
                content=body,
                params=request.query_params,
            )
            
            return Response(
                content=response.content,
                status_code=response.status_code,
                headers=dict(response.headers),
                media_type=response.headers.get("content-type")
            )
        
        except httpx.TimeoutException:
            raise HTTPException(504, f"Service {service_name} timed out")
        except httpx.ConnectError:
            raise HTTPException(503, f"Service {service_name} unavailable")

@app.api_route(
    "/api/v1/{service}/{path:path}",
    methods=["GET", "POST", "PUT", "PATCH", "DELETE"]
)
async def gateway_handler(
    service: str,
    path: str,
    request: Request,
):
    """Main gateway handler — routes all traffic"""
    # Auth check at gateway level (before forwarding)
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    if not token and request.url.path not in ["/api/v1/auth/login", "/health"]:
        raise HTTPException(401, "Authentication required")
    
    full_path = f"/{path}"
    return await route_request(request, service, full_path)

@app.get("/health")
async def health():
    """Check health of all downstream services"""
    async def check_service(name: str, url: str) -> dict:
        try:
            async with httpx.AsyncClient(timeout=2.0) as client:
                response = await client.get(f"{url}/health")
                return {"name": name, "status": "up", "latency_ms": 0}
        except Exception:
            return {"name": name, "status": "down"}
    
    results = await asyncio.gather(*[
        check_service(name, url) for name, url in SERVICES.items()
    ])
    
    all_up = all(r["status"] == "up" for r in results)
    return {
        "gateway": "up",
        "services": results,
        "overall": "healthy" if all_up else "degraded"
    }
```

### Key Takeaways
> - 💡 **Single entry point** for clients — they only need one URL
> - 💡 **Centralize auth, rate limiting, logging** at gateway level
> - 💡 **Kong, AWS API Gateway, or Traefik** for production (not custom code)
> - 💡 **Circuit breakers** per service prevent gateway from cascading failures
> - 💡 **BFF (Backend For Frontend)** pattern: separate gateways for web and mobile

---

*Next: [05 — Database & Data Management →](./05-database-data-management.md)*
