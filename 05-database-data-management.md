# 05 — Database & Data Management

> **Questions 41–50** | Handling data correctly, efficiently, and reliably

---

## Question 41 — Two Users Simultaneously Update Same Resource
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Two employees open the same customer record at the same time. Both make changes. One saves. The other saves too — and overwrites the first person's changes silently. How do you handle this?"*

### The Answer

```
THE LOST UPDATE PROBLEM:

Time  User A                    User B
──────────────────────────────────────────────────
1:00  Read: name="John Smith"
1:01                            Read: name="John Smith"
1:02  Edit: name="John S."
1:03  Save → DB: "John S."     ← Success
1:04                            Edit: email="new@ex.com"
1:05                            Save → DB: "John Smith" (overwrites A's change!)
                                
B's save LOSES A's name change! "John S." is now gone.

OPTIMISTIC LOCKING FIX:

1:00  A reads: { name: "John Smith", version: 5 }
1:01                            B reads: { name: "John Smith", version: 5 }
1:02  A saves: PUT /users/1 { name: "John S.", version: 5 }
1:03                   DB: version=5 → matches → UPDATE, version becomes 6
1:04                            B saves: PUT /users/1 { email: "new@ex.com", version: 5 }
1:05                   DB: version=5 → MISMATCH (now 6) → 409 CONFLICT!
                       B must re-read and merge changes
```

```mermaid
sequenceDiagram
    participant A as User A
    participant API as API
    participant DB as Database
    participant B as User B

    A->>API: GET /records/1
    API->>DB: SELECT * WHERE id=1
    DB-->>API: {data, version: 5}
    API-->>A: {data, version: 5}
    
    B->>API: GET /records/1
    API-->>B: {data, version: 5}

    A->>API: PUT /records/1 {changes, version: 5}
    API->>DB: UPDATE WHERE id=1 AND version=5
    DB-->>API: Updated (version now 6)
    API-->>A: 200 OK {version: 6}

    B->>API: PUT /records/1 {changes, version: 5}
    API->>DB: UPDATE WHERE id=1 AND version=5
    DB-->>API: 0 rows affected (version is 6!)
    API-->>B: 409 Conflict - Record was modified
    Note over B: Must re-read and resolve conflict
```

### Code Example — Optimistic Locking with ETags and Version Fields

```python
import hashlib
from fastapi import FastAPI, HTTPException, Header, Response
from pydantic import BaseModel
from typing import Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import Integer, String, select, update

app = FastAPI()

# Database model with version field
class CustomerRecord(Base):
    __tablename__ = "customers"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String)
    email: Mapped[str] = mapped_column(String)
    version: Mapped[int] = mapped_column(Integer, default=1)

class CustomerUpdate(BaseModel):
    name: Optional[str] = None
    email: Optional[str] = None
    version: int  # Client MUST provide the version they're updating

@app.get("/customers/{customer_id}")
async def get_customer(customer_id: int, response: Response, db: AsyncSession = Depends(get_db)):
    customer = await db.get(CustomerRecord, customer_id)
    if not customer:
        raise HTTPException(404, "Customer not found")
    
    # ETag for HTTP-level caching and concurrency control
    etag = hashlib.md5(f"{customer.id}:{customer.version}".encode()).hexdigest()
    response.headers["ETag"] = f'"{etag}"'
    
    return {
        "id": customer.id,
        "name": customer.name,
        "email": customer.email,
        "version": customer.version
    }

@app.put("/customers/{customer_id}")
async def update_customer(
    customer_id: int,
    update_data: CustomerUpdate,
    db: AsyncSession = Depends(get_db),
    if_match: Optional[str] = Header(None)  # ETag-based concurrency
):
    """
    OPTIMISTIC LOCKING: Update only if version matches.
    409 Conflict if another user already updated the record.
    """
    # Method 1: Version field in request body
    result = await db.execute(
        update(CustomerRecord)
        .where(
            CustomerRecord.id == customer_id,
            CustomerRecord.version == update_data.version  # ← Key check!
        )
        .values(
            name=update_data.name,
            email=update_data.email,
            version=update_data.version + 1  # Increment version
        )
        .returning(CustomerRecord)
    )
    
    updated = result.fetchone()
    
    if not updated:
        # Either record not found, or version mismatch
        existing = await db.get(CustomerRecord, customer_id)
        if not existing:
            raise HTTPException(404, "Customer not found")
        
        # Version mismatch — someone else updated it
        raise HTTPException(
            status_code=409,
            detail={
                "error": "Conflict",
                "message": "Record was modified by another user. Please refresh and retry.",
                "current_version": existing.version,
                "your_version": update_data.version,
            }
        )
    
    await db.commit()
    return {"id": customer_id, "version": update_data.version + 1}

# Pessimistic locking alternative (for critical operations)
@app.put("/customers/{customer_id}/pessimistic")
async def update_customer_pessimistic(
    customer_id: int,
    update_data: dict,
    db: AsyncSession = Depends(get_db)
):
    """
    PESSIMISTIC LOCKING: Lock the row during transaction.
    Use for high-contention resources (bank transfers, inventory).
    Slower but no conflicts possible.
    """
    async with db.begin():
        # Lock the row — other transactions wait until this one commits
        result = await db.execute(
            select(CustomerRecord)
            .where(CustomerRecord.id == customer_id)
            .with_for_update()  # SELECT ... FOR UPDATE (row-level lock)
        )
        customer = result.scalar_one_or_none()
        
        if not customer:
            raise HTTPException(404, "Customer not found")
        
        # Safe to update — row is locked
        customer.name = update_data.get("name", customer.name)
        customer.email = update_data.get("email", customer.email)
        
    # Lock released on commit
    return {"id": customer.id, "updated": True}

async def get_db():
    pass  # Placeholder

class Base:
    pass
```

### When to Use Which?
| Scenario | Strategy |
|----------|----------|
| Documents, profiles, low contention | Optimistic locking (version field) |
| Bank transfers, inventory | Pessimistic locking (SELECT FOR UPDATE) |
| Web APIs (HTTP) | ETag + If-Match headers |
| High-throughput, can retry | Optimistic with CAS (Compare-And-Swap) |

### Key Takeaways
> - 💡 **Optimistic locking** is best for most APIs — no blocking, just retry on conflict
> - 💡 **Return 409 Conflict** with clear message explaining what happened
> - 💡 **ETag headers** implement optimistic locking at the HTTP level
> - 💡 **Pessimistic locking** (`SELECT FOR UPDATE`) for critical financial operations
> - 💡 **Never silently overwrite** — always check version before update

---

## Question 42 — Write to Database and Message Queue Atomically
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your API needs to save an order to PostgreSQL AND publish an 'order.created' event to RabbitMQ/Kafka. If DB save succeeds but queue publish fails, the order exists but no email is sent. How do you guarantee both happen?"*

### The Answer

```
THE DUAL WRITE PROBLEM:

Option 1: Write DB then Queue:
  DB SAVE ✅ → Queue PUBLISH ❌ → Order exists, no event!

Option 2: Write Queue then DB:
  Queue PUBLISH ✅ → DB SAVE ❌ → Event sent, no order in DB!

Option 3: Two-Phase Commit:
  Slow, complex, most DBs don't support distributed 2PC

SOLUTION: Outbox Pattern
  Write to DB AND an "outbox" table in SAME transaction.
  A separate process reads the outbox and publishes to queue.
  If outbox publish fails, retry. DB and outbox are always in sync.
```

```mermaid
flowchart LR
    A[API Request] --> B[Begin Transaction]
    B --> C[INSERT INTO orders]
    B --> D[INSERT INTO outbox\nevent_type=order.created\npayload=...]
    C --> E[COMMIT]
    D --> E
    E --> F[Return 201 Created]
    
    G[Outbox Poller\nevery 1 second] --> H[SELECT FROM outbox\nWHERE published=false]
    H --> I[Publish to Kafka/RabbitMQ]
    I --> J[UPDATE outbox\nSET published=true]
    
    style E fill:#c8e6c9
    style I fill:#c8e6c9
```

### Code Example — Outbox Pattern

```python
import json
import asyncio
from datetime import datetime
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import Mapped, mapped_column
from sqlalchemy import Integer, String, Boolean, DateTime, Text, select

app = FastAPI()

# Outbox table in the SAME database as your orders
class OutboxEvent(Base):
    __tablename__ = "outbox_events"
    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    event_type: Mapped[str] = mapped_column(String(100))
    aggregate_id: Mapped[str] = mapped_column(String(100))  # e.g., order ID
    payload: Mapped[str] = mapped_column(Text)  # JSON
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    published: Mapped[bool] = mapped_column(Boolean, default=False)
    published_at: Mapped[datetime] = mapped_column(DateTime, nullable=True)
    retry_count: Mapped[int] = mapped_column(Integer, default=0)

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    user_id: Mapped[int] = mapped_column(Integer)
    total: Mapped[float]
    status: Mapped[str] = mapped_column(String, default="pending")

@app.post("/orders", status_code=201)
async def create_order(order_data: dict, db: AsyncSession = Depends(get_db_session)):
    """
    Create order + add to outbox in ONE transaction.
    Atomically: either both succeed or both fail.
    """
    async with db.begin():  # Single atomic transaction
        # 1. Create the order
        order = Order(
            user_id=order_data["user_id"],
            total=order_data["total"],
            status="pending"
        )
        db.add(order)
        await db.flush()  # Get the ID without committing
        
        # 2. Add event to outbox (same transaction!)
        outbox_event = OutboxEvent(
            event_type="order.created",
            aggregate_id=str(order.id),
            payload=json.dumps({
                "order_id": order.id,
                "user_id": order.user_id,
                "total": order.total,
                "timestamp": datetime.utcnow().isoformat()
            })
        )
        db.add(outbox_event)
        # Commit both together atomically
    
    # Don't publish to queue here — let the outbox poller do it
    return {"order_id": order.id, "status": "pending"}

# Outbox poller — runs as a separate background process
async def outbox_poller():
    """
    Polls outbox table and publishes events to message queue.
    Runs independently — retries until published.
    """
    while True:
        try:
            async with get_db_session() as db:
                # Get unpublished events
                result = await db.execute(
                    select(OutboxEvent)
                    .where(
                        OutboxEvent.published == False,
                        OutboxEvent.retry_count < 5
                    )
                    .order_by(OutboxEvent.id)
                    .limit(100)
                )
                events = result.scalars().all()
                
                for event in events:
                    try:
                        # Publish to message queue
                        await publish_to_kafka(
                            topic=event.event_type,
                            payload=json.loads(event.payload)
                        )
                        
                        # Mark as published
                        event.published = True
                        event.published_at = datetime.utcnow()
                    
                    except Exception as e:
                        event.retry_count += 1
                        print(f"Failed to publish event {event.id}: {e}")
                
                await db.commit()
        
        except Exception as e:
            print(f"Outbox poller error: {e}")
        
        await asyncio.sleep(1)  # Poll every second

async def publish_to_kafka(topic: str, payload: dict):
    """Publish event to Kafka"""
    # aiokafka.AIOKafkaProducer usage
    print(f"Publishing to {topic}: {payload}")

async def get_db_session():
    pass  # Placeholder

class Base:
    pass
```

### Key Takeaways
> - 💡 **Outbox pattern** solves dual-write atomically — DB + outbox in one transaction
> - 💡 **The poller retries** until published — at-least-once delivery guarantee
> - 💡 **Consumers must be idempotent** — they may receive the same event twice
> - 💡 **Clean up old outbox events** after N days to prevent table growth
> - 💡 **Alternative**: Debezium CDC (Change Data Capture) reads DB transaction log directly

---

## Question 43 — Soft Deletes vs GDPR Right to Be Forgotten
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your app uses soft deletes (is_deleted=true). A user requests GDPR 'right to be forgotten' — complete deletion of their data. But soft-deleted records still contain PII. How do you handle both?"*

### The Answer

```
SOFT DELETE: Mark as deleted (keep data)
  { id: 1, email: "john@ex.com", is_deleted: true }
  ↑ Still has PII! GDPR violation if user requests deletion!

GDPR DELETION OPTIONS:

Option 1: Hard delete (simple but loses history)
  DELETE FROM users WHERE id = 1

Option 2: Anonymization (keep records, remove PII)
  { id: 1, email: "deleted_1@anonymized.com", name: "Deleted User", is_deleted: true }
  ↑ Record exists (for order history, audit) but no identifiable info

Option 3: Pseudonymization (replace PII with tokens)
  { id: 1, email: "token_abc123", name: "token_def456" }
  ↑ Can de-anonymize if needed (legal hold), but not readable
```

### Code Example — GDPR Deletion Service

```python
import hashlib
import uuid
from datetime import datetime
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel

app = FastAPI()

class DeletionRequest(BaseModel):
    user_id: int
    reason: str = "gdpr_right_to_erasure"
    confirmation: str  # Must be "DELETE MY DATA"

@app.post("/users/{user_id}/gdpr/delete")
async def gdpr_delete_user(
    user_id: int,
    request: DeletionRequest,
    db=Depends(get_db_session)
):
    """
    GDPR Right to Erasure: Anonymize all user PII while preserving
    business records (orders, invoices) in anonymized form.
    """
    if request.confirmation != "DELETE MY DATA":
        raise HTTPException(400, "Confirmation text doesn't match")
    
    # Generate anonymization token (consistent per user for referential integrity)
    anon_token = hashlib.sha256(f"deleted_user_{user_id}".encode()).hexdigest()[:12]
    
    async with db.begin():
        # 1. Anonymize user record
        await db.execute(
            """
            UPDATE users SET
                email = :email,
                name = :name,
                phone = NULL,
                address = NULL,
                date_of_birth = NULL,
                gdpr_deleted_at = :deleted_at,
                gdpr_deletion_token = :token,
                is_deleted = true
            WHERE id = :user_id
            """,
            {
                "email": f"deleted_{anon_token}@anonymized.invalid",
                "name": "Deleted User",
                "deleted_at": datetime.utcnow(),
                "token": anon_token,
                "user_id": user_id
            }
        )
        
        # 2. Anonymize orders (keep for accounting, remove PII)
        await db.execute(
            """
            UPDATE orders SET
                shipping_address = :anon_address,
                customer_name = :anon_name,
                customer_email = :anon_email
            WHERE user_id = :user_id
            """,
            {
                "anon_address": "Address removed",
                "anon_name": "Deleted User",
                "anon_email": f"deleted_{anon_token}@anonymized.invalid",
                "user_id": user_id
            }
        )
        
        # 3. Delete actual PII tables (no business need to keep)
        await db.execute("DELETE FROM user_sessions WHERE user_id = :user_id", {"user_id": user_id})
        await db.execute("DELETE FROM user_preferences WHERE user_id = :user_id", {"user_id": user_id})
        await db.execute("DELETE FROM user_activity_log WHERE user_id = :user_id", {"user_id": user_id})
        
        # 4. Log the deletion for compliance
        await db.execute(
            """
            INSERT INTO gdpr_deletions (user_id, anonymization_token, deleted_at, reason)
            VALUES (:user_id, :token, :deleted_at, :reason)
            """,
            {
                "user_id": user_id,
                "token": anon_token,
                "deleted_at": datetime.utcnow(),
                "reason": request.reason
            }
        )
    
    return {
        "user_id": user_id,
        "status": "deleted",
        "deleted_at": datetime.utcnow().isoformat(),
        "message": "User data has been anonymized per GDPR requirements",
        "reference": anon_token  # For any follow-up
    }

async def get_db_session():
    pass
```

### Key Takeaways
> - 💡 **Soft deletes don't satisfy GDPR** — data is still accessible
> - 💡 **Anonymization is better than deletion** — preserves business records
> - 💡 **Identify all tables containing PII** before implementing
> - 💡 **Don't forget backups** — GDPR deletion must also apply to backup copies
> - 💡 **Log all GDPR deletions** for compliance audits

---

## Question 44 — Zero-Downtime Database Migrations
🔴 Senior | ★★★ Very Common

### The Scenario
> *"You need to rename a column `user_name` to `full_name` in a table with 10M rows. Last time you ran a migration, it locked the table and caused 5 minutes of downtime. How do you do this without downtime?"*

### The Answer

```
EXPAND-CONTRACT PATTERN (aka parallel change):

❌ WRONG WAY (causes downtime):
  ALTER TABLE users RENAME COLUMN user_name TO full_name;
  (Locks entire 10M row table for minutes!)

✅ RIGHT WAY (zero downtime):

Step 1 - EXPAND (deploy new code that writes to BOTH columns):
  ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
  New code: writes to user_name AND full_name

Step 2 - BACKFILL (migrate existing data in batches, no lock):
  UPDATE users SET full_name = user_name WHERE id BETWEEN 1 AND 10000;
  UPDATE users SET full_name = user_name WHERE id BETWEEN 10001 AND 20000;
  ... (batches of 10K, overnight)

Step 3 - SWITCH (deploy code that reads/writes full_name only):
  New code: reads from full_name, ignores user_name
  
Step 4 - CONTRACT (remove old column):
  ALTER TABLE users DROP COLUMN user_name;
  (Old column is now unused — safe to drop)
```

```mermaid
gantt
    title Zero-Downtime Migration Timeline
    section Deployment
    Deploy Step 1 (add column)    :d1, 2024-01-01, 1d
    Backfill data (background)    :d2, after d1, 3d
    Deploy Step 2 (read new col)  :d3, after d2, 1d
    Deploy Step 3 (drop old col)  :d4, after d3, 2d
    section Database
    Old column active             :active, 2024-01-01, 2024-01-06
    Both columns active           :active, 2024-01-02, 2024-01-04
    New column active             :active, 2024-01-04, 2024-01-08
```

### Code Example — Alembic Migration with Batched Backfill

```python
# Migration file: alembic/versions/001_rename_user_name.py

from alembic import op
import sqlalchemy as sa
from sqlalchemy.orm import Session

def upgrade():
    """Step 1: Add new column (fast — no table scan)"""
    # ADD COLUMN is fast, doesn't lock rows
    op.add_column('users', sa.Column('full_name', sa.String(255), nullable=True))
    
    # Add index concurrently (PostgreSQL) — no table lock!
    op.execute('CREATE INDEX CONCURRENTLY ix_users_full_name ON users (full_name)')

def downgrade():
    op.drop_column('users', 'full_name')

# Separate script: backfill.py (run as background job)
async def backfill_full_name(db, batch_size: int = 10000):
    """
    Backfill data in small batches.
    Safe to run while app is running — no locks!
    """
    max_id_result = await db.execute("SELECT MAX(id) FROM users")
    max_id = max_id_result.scalar()
    
    offset = 0
    batches_processed = 0
    
    while offset < max_id:
        await db.execute(
            """
            UPDATE users 
            SET full_name = user_name
            WHERE id > :offset AND id <= :limit
            AND full_name IS NULL
            """,
            {"offset": offset, "limit": offset + batch_size}
        )
        await db.commit()
        
        offset += batch_size
        batches_processed += 1
        
        # Progress logging
        progress = min(100, int(offset / max_id * 100))
        print(f"Backfill progress: {progress}% ({offset}/{max_id} rows)")
        
        # Small delay between batches to not overload database
        import asyncio
        await asyncio.sleep(0.1)
    
    print(f"Backfill complete! {batches_processed} batches processed")

# FastAPI app code that works during migration
from pydantic import BaseModel, model_validator
from typing import Optional

class User(BaseModel):
    id: int
    full_name: Optional[str] = None
    user_name: Optional[str] = None  # Old column (still exists during migration)
    
    @model_validator(mode="after")
    def coalesce_name(self):
        """During migration, accept either column"""
        if not self.full_name and self.user_name:
            self.full_name = self.user_name
        return self
    
    @property
    def name(self) -> str:
        """Always returns a name, regardless of migration state"""
        return self.full_name or self.user_name or ""
```

### Key Takeaways
> - 💡 **Expand-Contract (3 deployments)**: Add column → Backfill → Drop old column
> - 💡 **Batch backfills** in 10K row chunks to avoid locking
> - 💡 **`CREATE INDEX CONCURRENTLY`** in PostgreSQL — no table lock!
> - 💡 **Code must support both old and new schema** during transition
> - 💡 **Test migrations on staging** with production-sized data first

---

## Question 45 — Aggregate Data from SQL + MongoDB + Redis
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your user dashboard needs to show: user profile (PostgreSQL), recent activity (MongoDB), and online status (Redis). How do you aggregate this efficiently?"*

### The Answer

### Code Example — Repository Pattern + Data Aggregation

```python
import asyncio
from typing import Optional
from fastapi import FastAPI

app = FastAPI()

# Repository pattern: one class per data source
class UserRepository:
    """PostgreSQL — structured user data"""
    
    async def get_user(self, user_id: int) -> Optional[dict]:
        # async with asyncpg connection
        return {
            "id": user_id,
            "name": "John Doe",
            "email": "john@example.com",
            "plan": "pro",
            "created_at": "2023-01-15"
        }

class ActivityRepository:
    """MongoDB — unstructured activity events"""
    
    async def get_recent_activity(self, user_id: int, limit: int = 10) -> list:
        # async with motor connection
        return [
            {"event": "login", "timestamp": "2024-01-10T10:00:00Z"},
            {"event": "purchase", "timestamp": "2024-01-09T15:30:00Z"},
        ]

class PresenceRepository:
    """Redis — real-time online status"""
    
    async def get_online_status(self, user_id: int) -> dict:
        # await redis.get(f"presence:{user_id}")
        return {"online": True, "last_seen": "2024-01-10T10:01:00Z"}

# Aggregate all in one API call
user_repo = UserRepository()
activity_repo = ActivityRepository()
presence_repo = PresenceRepository()

@app.get("/users/{user_id}/dashboard")
async def get_user_dashboard(user_id: int):
    """
    Aggregate from 3 data sources in PARALLEL.
    Total time = max(postgres_time, mongo_time, redis_time)
    Instead of postgres_time + mongo_time + redis_time
    """
    user, activity, presence = await asyncio.gather(
        user_repo.get_user(user_id),
        activity_repo.get_recent_activity(user_id, limit=10),
        presence_repo.get_online_status(user_id),
        return_exceptions=True  # Don't fail all if one source fails
    )
    
    return {
        "user": user if not isinstance(user, Exception) else None,
        "activity": activity if not isinstance(activity, Exception) else [],
        "presence": presence if not isinstance(presence, Exception) else {"online": False},
        "data_sources": {
            "postgres": "ok" if not isinstance(user, Exception) else "error",
            "mongodb": "ok" if not isinstance(activity, Exception) else "error",
            "redis": "ok" if not isinstance(presence, Exception) else "error",
        }
    }
```

### Key Takeaways
> - 💡 **Repository pattern** isolates each data source — easy to swap
> - 💡 **asyncio.gather** runs all queries in parallel — huge latency improvement
> - 💡 **return_exceptions=True** prevents one failed source from breaking everything
> - 💡 **CQRS**: Use fast read stores (Redis/MongoDB) for reads, SQL for writes
> - 💡 **Cache the aggregated result** in Redis to avoid hitting 3 DBs per request

---

## Question 46 — Read-Heavy API Overwhelming Primary Database
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your e-commerce API has 95% reads and 5% writes. The primary PostgreSQL database is at 90% CPU. Read queries are killing the database. How do you scale reads?"*

### The Answer

```
READ SCALING ARCHITECTURE:

           Writes (5%)               Reads (95%)
               │                          │
               ▼                          ▼
        ┌─────────────┐         ┌─────────────────────┐
        │   Primary    │         │    Cache Layer       │
        │  PostgreSQL  │──repl──►│  Redis (hot data)   │
        │   (writes)   │         └──────────┬──────────┘
        └─────────────┘                     │ Cache miss
                                            ▼
                               ┌─────────────────────────┐
                               │     Read Replicas        │
                               │ Replica 1  │  Replica 2  │
                               │ PostgreSQL │  PostgreSQL  │
                               └─────────────────────────┘
```

### Code Example — Read Replica Routing

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker
import random

app = FastAPI()

# Separate engines for primary and replicas
primary_engine = create_async_engine(
    "postgresql+asyncpg://user:pass@primary-db:5432/mydb",
    pool_size=10
)

replica_engines = [
    create_async_engine(
        "postgresql+asyncpg://user:pass@replica1:5432/mydb",
        pool_size=20  # More connections for reads
    ),
    create_async_engine(
        "postgresql+asyncpg://user:pass@replica2:5432/mydb",
        pool_size=20
    ),
]

PrimarySession = sessionmaker(primary_engine, class_=AsyncSession, expire_on_commit=False)

def get_replica_session() -> AsyncSession:
    """Round-robin load balancing across read replicas"""
    engine = random.choice(replica_engines)
    Session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
    return Session()

async def get_write_db() -> AsyncSession:
    """Always write to primary"""
    async with PrimarySession() as session:
        yield session

async def get_read_db() -> AsyncSession:
    """Read from replicas"""
    async with get_replica_session() as session:
        yield session

@app.get("/products")
async def list_products(
    db: AsyncSession = Depends(get_read_db)  # Uses replica
):
    """Read operation → replica"""
    result = await db.execute("SELECT * FROM products LIMIT 100")
    return result.fetchall()

@app.post("/products")
async def create_product(
    data: dict,
    db: AsyncSession = Depends(get_write_db)  # Uses primary
):
    """Write operation → primary"""
    # INSERT INTO products ...
    return {"created": True}

# Read-After-Write Consistency: after a write, use primary for a short period
import time
from fastapi import Request

@app.post("/orders")
async def create_order(
    request: Request,
    data: dict,
    db: AsyncSession = Depends(get_write_db)
):
    """Create order on primary"""
    order_id = "order_123"
    
    # Set cookie so next reads use primary (for read-after-write consistency)
    from fastapi.responses import JSONResponse
    response = JSONResponse(content={"order_id": order_id})
    response.set_cookie(
        "use_primary_until",
        str(int(time.time()) + 2),  # Use primary for 2 seconds
        httponly=True
    )
    return response
```

### Key Takeaways
> - 💡 **Read replicas** scale reads horizontally — add more as needed
> - 💡 **Route reads to replicas, writes to primary** — separation of concerns
> - 💡 **Redis cache** reduces load further — most reads never hit the DB
> - 💡 **Read-after-write consistency**: route to primary for N seconds after write
> - 💡 **Replica lag** is typically <1 second — acceptable for most use cases

---

## Question 47 — Full-Text Search with Autocomplete
🟡 Mid | ★★☆ Common

### The Scenario
> *"Users search for products. The current LIKE '%query%' search is slow on 1M+ products and doesn't support typos. How do you implement proper full-text search with autocomplete?"*

### The Answer

### Code Example — Elasticsearch Full-Text Search

```python
from fastapi import FastAPI, Query
from elasticsearch import AsyncElasticsearch
from typing import Optional

app = FastAPI()
es = AsyncElasticsearch(["http://localhost:9200"])

@app.get("/search/products")
async def search_products(
    q: str = Query(..., min_length=1),
    category: Optional[str] = None,
    min_price: Optional[float] = None,
    max_price: Optional[float] = None,
    page: int = 1,
    limit: int = 20
):
    """Full-text search with fuzzy matching, filters, and facets"""
    
    must_clauses = [
        {
            "multi_match": {
                "query": q,
                "fields": ["name^3", "description", "brand^2"],  # ^ = boost
                "fuzziness": "AUTO",  # Handle typos automatically
                "type": "best_fields"
            }
        }
    ]
    
    filter_clauses = []
    
    if category:
        filter_clauses.append({"term": {"category": category}})
    
    if min_price or max_price:
        price_filter = {"range": {"price": {}}}
        if min_price:
            price_filter["range"]["price"]["gte"] = min_price
        if max_price:
            price_filter["range"]["price"]["lte"] = max_price
        filter_clauses.append(price_filter)
    
    query = {
        "bool": {
            "must": must_clauses,
            "filter": filter_clauses
        }
    }
    
    result = await es.search(
        index="products",
        body={
            "query": query,
            "from": (page - 1) * limit,
            "size": limit,
            "highlight": {
                "fields": {"name": {}, "description": {"fragment_size": 150}}
            },
            "aggs": {
                "categories": {"terms": {"field": "category", "size": 10}},
                "price_ranges": {
                    "range": {
                        "field": "price",
                        "ranges": [
                            {"to": 50}, {"from": 50, "to": 100}, {"from": 100}
                        ]
                    }
                }
            }
        }
    )
    
    hits = result["hits"]["hits"]
    
    return {
        "total": result["hits"]["total"]["value"],
        "page": page,
        "results": [
            {
                "id": hit["_id"],
                "score": hit["_score"],
                **hit["_source"],
                "highlights": hit.get("highlight", {})
            }
            for hit in hits
        ],
        "facets": {
            "categories": result["aggregations"]["categories"]["buckets"],
            "price_ranges": result["aggregations"]["price_ranges"]["buckets"]
        }
    }

@app.get("/search/autocomplete")
async def autocomplete(q: str = Query(..., min_length=2)):
    """Autocomplete suggestions as user types"""
    result = await es.search(
        index="products",
        body={
            "suggest": {
                "product_suggest": {
                    "prefix": q,
                    "completion": {
                        "field": "name_suggest",  # Special completion field
                        "fuzzy": {"fuzziness": 1},
                        "size": 10
                    }
                }
            }
        }
    )
    
    suggestions = result["suggest"]["product_suggest"][0]["options"]
    return {
        "suggestions": [s["_source"]["name"] for s in suggestions]
    }
```

### Key Takeaways
> - 💡 **Elasticsearch** is purpose-built for full-text search — much faster than SQL LIKE
> - 💡 **Fuzziness: "AUTO"** handles typos automatically
> - 💡 **Faceted search** (aggregations) builds filter sidebars
> - 💡 **Boost fields** with `^N` — name matches more important than description
> - 💡 **Sync data**: use CDC (Debezium) or post-write hooks to keep ES in sync with DB

---

## Question 48 — High-Volume IoT Time-Series Data
🔴 Senior | ★☆☆ Rare

### The Scenario
> *"Your IoT platform receives 100,000 sensor readings per second from 50,000 devices. Each reading has device_id, timestamp, temperature, humidity, pressure. How do you handle this?"*

### The Answer

```
TIME-SERIES DATA CHALLENGES:

1. Write throughput: 100K inserts/second (regular PostgreSQL handles ~10K)
2. Query patterns: "last 24h average by device" (not typical OLTP)
3. Data retention: Keep raw data for 7 days, hourly averages for 1 year
4. Old data grows forever

SOLUTION STACK:
┌─────────────────────────────────────────────────────────────┐
│  Devices                                                     │
│  (50K devices)                                              │
└────────────────────────┬────────────────────────────────────┘
                         │ MQTT / HTTP
                         ▼
                  ┌─────────────┐
                  │  Message    │
                  │  Queue      │
                  │  (Kafka)    │
                  └──────┬──────┘
                         │
                  ┌──────▼──────┐
                  │  TimescaleDB│  ← PostgreSQL extension for time-series
                  │  or InfluxDB│  ← Purpose-built time-series DB
                  └─────────────┘
```

### Code Example — TimescaleDB Batch Inserts

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List
import asyncpg
from datetime import datetime

app = FastAPI()

class SensorReading(BaseModel):
    device_id: str
    timestamp: datetime
    temperature: float
    humidity: float
    pressure: float

# In-memory buffer for batch inserts
reading_buffer: List[SensorReading] = []
BUFFER_SIZE = 1000

@app.post("/readings/batch")
async def ingest_readings(readings: List[SensorReading]):
    """Batch endpoint: accept up to 1000 readings per request"""
    reading_buffer.extend(readings)
    
    if len(reading_buffer) >= BUFFER_SIZE:
        await flush_buffer()
    
    return {"accepted": len(readings)}

async def flush_buffer():
    """Bulk insert readings into TimescaleDB"""
    if not reading_buffer:
        return
    
    batch = reading_buffer.copy()
    reading_buffer.clear()
    
    conn = await asyncpg.connect("postgresql://user:pass@timescaledb/iot")
    try:
        await conn.executemany(
            """
            INSERT INTO sensor_readings (device_id, time, temperature, humidity, pressure)
            VALUES ($1, $2, $3, $4, $5)
            ON CONFLICT (device_id, time) DO NOTHING
            """,
            [(r.device_id, r.timestamp, r.temperature, r.humidity, r.pressure)
             for r in batch]
        )
    finally:
        await conn.close()

@app.get("/devices/{device_id}/stats")
async def get_device_stats(device_id: str, hours: int = 24):
    """Get hourly aggregates for a device"""
    conn = await asyncpg.connect("postgresql://user:pass@timescaledb/iot")
    try:
        # TimescaleDB time_bucket function for efficient aggregation
        rows = await conn.fetch(
            """
            SELECT
                time_bucket('1 hour', time) AS hour,
                AVG(temperature) AS avg_temp,
                MIN(temperature) AS min_temp,
                MAX(temperature) AS max_temp,
                AVG(humidity) AS avg_humidity
            FROM sensor_readings
            WHERE device_id = $1
              AND time > NOW() - INTERVAL '24 hours'
            GROUP BY hour
            ORDER BY hour DESC
            """,
            device_id
        )
        return [dict(row) for row in rows]
    finally:
        await conn.close()

# TimescaleDB setup SQL:
"""
-- Create hypertable (partitioned by time automatically)
SELECT create_hypertable('sensor_readings', 'time');

-- Retention policy: keep raw data 7 days only
SELECT add_retention_policy('sensor_readings', INTERVAL '7 days');

-- Continuous aggregate: pre-compute hourly averages
CREATE MATERIALIZED VIEW sensor_readings_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    device_id,
    AVG(temperature) AS avg_temp,
    AVG(humidity) AS avg_humidity
FROM sensor_readings
GROUP BY bucket, device_id;
"""
```

### Key Takeaways
> - 💡 **TimescaleDB** = PostgreSQL + time-series optimizations (familiar SQL!)
> - 💡 **Batch inserts** instead of one-by-one — 100x more efficient
> - 💡 **Continuous aggregates** pre-compute hourly/daily stats — instant queries
> - 💡 **Retention policies** automatically purge old raw data
> - 💡 **Kafka** between devices and DB — handles burst traffic, prevents DB overload

---

## Question 49 — N+1 Query Problem
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your API endpoint to list 100 orders with customer names takes 5 seconds. When you look at the logs, you see 101 SQL queries for one request — 1 for orders, then 1 for each customer. How do you fix the N+1 problem?"*

### The Answer

```
N+1 PROBLEM VISUALIZED:

❌ N+1 (current):
  Query 1: SELECT * FROM orders LIMIT 100     → 100 orders
  Query 2: SELECT * FROM users WHERE id = 1   → User for order 1
  Query 3: SELECT * FROM users WHERE id = 2   → User for order 2
  ...
  Query 101: SELECT * FROM users WHERE id = 100
  Total: 101 queries! (N orders + 1 for list)

✅ JOIN approach (1 query):
  SELECT orders.*, users.name FROM orders
  JOIN users ON orders.user_id = users.id
  LIMIT 100
  Total: 1 query!

✅ Eager loading approach (2 queries):
  Query 1: SELECT * FROM orders LIMIT 100
  Query 2: SELECT * FROM users WHERE id IN (1, 2, ..., 100)
  (In-memory join in Python)
  Total: 2 queries!
```

### Code Example — SQLAlchemy Eager Loading

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy.orm import selectinload, joinedload
from sqlalchemy import select, Integer, String, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship

app = FastAPI()

class User(Base):
    __tablename__ = "users"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    name: Mapped[str] = mapped_column(String)
    orders: Mapped[list["Order"]] = relationship(back_populates="user")

class Order(Base):
    __tablename__ = "orders"
    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    user_id: Mapped[int] = mapped_column(Integer, ForeignKey("users.id"))
    total: Mapped[float]
    user: Mapped["User"] = relationship(back_populates="orders")

# ❌ N+1 problem (100 orders = 101 queries):
@app.get("/orders/bad")
async def list_orders_bad(db: AsyncSession = Depends(get_db)):
    result = await db.execute(select(Order).limit(100))
    orders = result.scalars().all()
    
    return [
        {
            "id": o.id,
            "total": o.total,
            "user_name": (await db.get(User, o.user_id)).name  # ← N+1!
        }
        for o in orders
    ]

# ✅ Eager loading with selectinload (2 queries total):
@app.get("/orders/good")
async def list_orders_good(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order)
        .options(selectinload(Order.user))  # Load all users in 1 extra query
        .limit(100)
    )
    orders = result.scalars().all()
    
    return [
        {
            "id": o.id,
            "total": o.total,
            "user_name": o.user.name  # Already loaded!
        }
        for o in orders
    ]

# ✅ JOIN (1 query — most efficient for simple cases):
@app.get("/orders/join")
async def list_orders_join(db: AsyncSession = Depends(get_db)):
    result = await db.execute(
        select(Order, User.name)
        .join(User, Order.user_id == User.id)
        .limit(100)
    )
    rows = result.all()
    
    return [
        {"id": order.id, "total": order.total, "user_name": name}
        for order, name in rows
    ]

# ✅ Manual batching for async/non-ORM code:
@app.get("/orders/batch")
async def list_orders_batch(db: AsyncSession = Depends(get_db)):
    # Step 1: Fetch orders
    result = await db.execute(select(Order).limit(100))
    orders = result.scalars().all()
    
    # Step 2: Get all unique user IDs
    user_ids = list(set(o.user_id for o in orders))
    
    # Step 3: Fetch all users in ONE query
    users_result = await db.execute(
        select(User).where(User.id.in_(user_ids))
    )
    users = {u.id: u for u in users_result.scalars().all()}
    
    # Step 4: Join in Python
    return [
        {
            "id": o.id,
            "total": o.total,
            "user_name": users.get(o.user_id, User(name="Unknown")).name
        }
        for o in orders
    ]

async def get_db():
    pass

class Base:
    pass
```

### Key Takeaways
> - 💡 **`selectinload()`** = 2 queries (best for lists with many items)
> - 💡 **`joinedload()`** = 1 query (best for single items or small sets)
> - 💡 **Manual batching** = fetch IDs, then `WHERE id IN (...)` — works without ORM
> - 💡 **Use SQLAlchemy echo=True** during development to see query count
> - 💡 **Log slow queries** with `log_min_duration_statement = 100ms` in postgres.conf

---

## Question 50 — Validate Uniqueness Against Database State
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Two users try to register with the same email at the exact same time. Your app checks 'is email taken?' → both say no → both insert → duplicate email in database! How do you handle this?"*

### The Answer

```
RACE CONDITION IN REGISTRATION:

Time  Request A (email: john@ex.com)     Request B (email: john@ex.com)
──────────────────────────────────────────────────────────────────────
1ms   SELECT: email exists? → NO
2ms                                      SELECT: email exists? → NO
3ms   INSERT users (email: john@ex.com)  ← Success
4ms                                      INSERT users (email: john@ex.com) ← DUPLICATE!

SOLUTIONS:
1. UNIQUE constraint in DB (always!) → DB catches the duplicate
2. ON CONFLICT in INSERT → handle gracefully in code
3. Distributed lock (Redis) → prevent concurrent processing of same email
```

### Code Example — Handling Unique Constraint Violations

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, EmailStr
from sqlalchemy.exc import IntegrityError
from typing import Optional

app = FastAPI()

class UserCreate(BaseModel):
    email: EmailStr
    password: str
    name: str

@app.post("/register", status_code=201)
async def register(user: UserCreate, db=Depends(get_db)):
    """
    Register user with proper unique constraint handling.
    Database UNIQUE constraint is the final safety net.
    """
    # Optional app-level check (for better UX — not security)
    existing = await db.execute(
        "SELECT id FROM users WHERE email = $1", user.email
    )
    if existing:
        raise HTTPException(
            status_code=409,
            detail={"field": "email", "message": "Email already registered"}
        )
    
    # Insert — DB UNIQUE constraint will catch concurrent duplicates
    try:
        result = await db.execute(
            """
            INSERT INTO users (email, name, password_hash)
            VALUES ($1, $2, $3)
            RETURNING id
            """,
            user.email, user.name, hash_password(user.password)
        )
        return {"user_id": result["id"], "email": user.email}
    
    except IntegrityError as e:
        if "unique" in str(e).lower() and "email" in str(e).lower():
            raise HTTPException(
                status_code=409,
                detail={"field": "email", "message": "Email already registered"}
            )
        raise

# SQLAlchemy with ON CONFLICT:
from sqlalchemy import text

async def upsert_user(email: str, name: str, db):
    """Insert or update — handle duplicates gracefully"""
    await db.execute(
        text("""
            INSERT INTO users (email, name)
            VALUES (:email, :name)
            ON CONFLICT (email) DO UPDATE
            SET name = EXCLUDED.name,
                updated_at = NOW()
            RETURNING id
        """),
        {"email": email, "name": name}
    )

# Database schema (always add UNIQUE constraint):
"""
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,  ← Database-level enforcement
    name VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
"""

async def get_db():
    pass

def hash_password(password: str) -> str:
    import bcrypt
    return bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()
```

### Key Takeaways
> - 💡 **Database UNIQUE constraint is mandatory** — app-level check alone is not enough
> - 💡 **Catch IntegrityError** and return 409 with helpful field-level error message
> - 💡 **App-level check is for UX** (immediate feedback), not security
> - 💡 **ON CONFLICT DO UPDATE** for upsert operations
> - 💡 **Never show "email already exists"** for user enumeration protection — but for registration forms it's acceptable UX

---

*Next: [06 — Testing & Deployment →](./06-testing-deployment.md)*
