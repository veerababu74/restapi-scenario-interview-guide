# 06 — Testing & Deployment

> **Questions 51–60** | Ensuring quality and smooth deployments

---

## Question 51 — 200 Endpoints, 0 Tests — Where to Start?
🟡 Mid | ★★★ Very Common

### The Scenario
> *"You join a company with a production FastAPI app, 200 endpoints, and zero automated tests. The team is afraid to deploy. How do you create a testing strategy from scratch?"*

### The Answer

```
TESTING PYRAMID:

                    ┌─────────────┐
                    │   E2E Tests │  ← Few, expensive, slow
                    │    (5-10)   │     "Critical user journeys"
                   ┌┴─────────────┴┐
                   │  Integration  │  ← Medium, test APIs end-to-end
                   │   Tests       │
                   │   (20-50)     │
                  ┌┴───────────────┴┐
                  │   Unit Tests    │  ← Many, cheap, fast
                  │   (100-200)     │     "Individual functions"
                  └─────────────────┘

STRATEGY — Start with highest business value:

Week 1: Critical paths (auth, payments, core business logic)
Week 2: Error handling and edge cases
Week 3: Integration tests for main flows
Month 2: Fill gaps, aim for 70% coverage
```

### Code Example — FastAPI Test Setup

```python
# tests/conftest.py
import asyncio
import pytest
from httpx import AsyncClient, ASGITransport
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

from app.main import app
from app.core.database import Base, get_db
from app.core.config import settings

# Use in-memory SQLite for tests (fast!)
TEST_DATABASE_URL = "sqlite+aiosqlite:///:memory:"

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for entire test session"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest.fixture(scope="session")
async def test_db_engine():
    engine = create_async_engine(TEST_DATABASE_URL, echo=False)
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.drop_all)
    await engine.dispose()

@pytest.fixture
async def db_session(test_db_engine):
    """Fresh database session for each test"""
    AsyncTestSession = sessionmaker(
        test_db_engine, class_=AsyncSession, expire_on_commit=False
    )
    async with AsyncTestSession() as session:
        yield session
        await session.rollback()  # Rollback after each test

@pytest.fixture
async def client(db_session):
    """HTTP client with overridden database dependency"""
    async def override_get_db():
        yield db_session
    
    app.dependency_overrides[get_db] = override_get_db
    
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test"
    ) as ac:
        yield ac
    
    app.dependency_overrides.clear()

@pytest.fixture
async def auth_client(client, db_session):
    """Authenticated HTTP client"""
    # Create test user
    response = await client.post("/auth/register", json={
        "email": "test@example.com",
        "password": "testpassword123",
        "name": "Test User"
    })
    
    # Login
    login_response = await client.post("/auth/login", json={
        "email": "test@example.com",
        "password": "testpassword123"
    })
    token = login_response.json()["access_token"]
    
    # Return client with auth header
    client.headers["Authorization"] = f"Bearer {token}"
    return client

# tests/test_users.py
import pytest
from httpx import AsyncClient

class TestUserRegistration:
    """Critical path: user registration"""
    
    @pytest.mark.asyncio
    async def test_register_success(self, client: AsyncClient):
        """Happy path: new user registers successfully"""
        response = await client.post("/auth/register", json={
            "email": "newuser@example.com",
            "password": "StrongPass123!",
            "name": "New User"
        })
        
        assert response.status_code == 201
        data = response.json()
        assert "user_id" in data
        assert data["email"] == "newuser@example.com"
        assert "password" not in data  # Never expose password!
    
    @pytest.mark.asyncio
    async def test_register_duplicate_email(self, client: AsyncClient):
        """Edge case: duplicate email returns 409"""
        user_data = {
            "email": "duplicate@example.com",
            "password": "StrongPass123!",
            "name": "User"
        }
        
        await client.post("/auth/register", json=user_data)  # First registration
        response = await client.post("/auth/register", json=user_data)  # Duplicate
        
        assert response.status_code == 409
        assert "email" in response.json()["detail"]["field"]
    
    @pytest.mark.asyncio
    async def test_register_weak_password(self, client: AsyncClient):
        """Validation: weak passwords rejected"""
        response = await client.post("/auth/register", json={
            "email": "user@example.com",
            "password": "123",  # Too short
            "name": "User"
        })
        
        assert response.status_code == 422  # Validation error

class TestUserAPI:
    """Test user CRUD operations"""
    
    @pytest.mark.asyncio
    async def test_get_own_profile(self, auth_client: AsyncClient):
        response = await auth_client.get("/users/me")
        assert response.status_code == 200
        assert "email" in response.json()
    
    @pytest.mark.asyncio
    async def test_cannot_access_other_user(self, auth_client: AsyncClient):
        """Security test: user can't access other user's data"""
        response = await auth_client.get("/users/999")
        assert response.status_code == 403
    
    @pytest.mark.asyncio
    async def test_unauthenticated_returns_401(self, client: AsyncClient):
        """Security test: protected endpoints require auth"""
        response = await client.get("/users/me")
        assert response.status_code == 401
```

### Key Takeaways
> - 💡 **Start with critical business paths** — auth, payments, core CRUD
> - 💡 **Use test database** (SQLite in-memory) — fast, isolated, no side effects
> - 💡 **Rollback after each test** — tests don't pollute each other
> - 💡 **Test security** — verify protected endpoints reject unauthenticated requests
> - 💡 **Target 70% coverage first** — 100% is diminishing returns

---

## Question 52 — Tests Pass Locally but Fail in CI/CD
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Your tests pass locally every time but randomly fail in GitHub Actions. Different failure messages each time. How do you debug this?"*

### The Answer

**Most common causes:**

```
LOCAL vs CI DIFFERENCES:

1. Environment variables
   Local: .env file with all vars set
   CI:    Missing env vars → null pointer errors

2. Database state
   Local: Previous test runs left data
   CI:    Clean database each run

3. Timing/async issues
   Local: Fast dev machine, tests finish before race condition
   CI:    Slower CI runner, race conditions surface

4. Dependency versions
   Local: requirements.txt not pinned → older versions
   CI:    pip installs latest → breaking changes

5. Test isolation
   Local: Tests run in same order, state leaks not noticed
   CI:    Parallel tests → state conflicts
```

### Code Example — Fixing Common CI Issues

```python
# pytest.ini (or pyproject.toml [tool.pytest.ini_options])
"""
[tool:pytest]
asyncio_mode = auto
timeout = 30  # Fail slow tests instead of hanging
markers =
    slow: marks tests as slow
    integration: marks integration tests
    unit: marks unit tests
"""

# Fix 1: Environment variable handling
# tests/conftest.py
import os
import pytest

@pytest.fixture(autouse=True)
def set_test_env(monkeypatch):
    """Ensure test environment variables are always set"""
    monkeypatch.setenv("DATABASE_URL", "sqlite+aiosqlite:///:memory:")
    monkeypatch.setenv("REDIS_URL", "redis://localhost:6379/1")  # Test DB
    monkeypatch.setenv("SECRET_KEY", "test-secret-key-not-for-production")
    monkeypatch.setenv("ENVIRONMENT", "test")

# Fix 2: Test isolation — reset state between tests
@pytest.fixture(autouse=True)
async def reset_redis(redis_client):
    """Flush test Redis before each test"""
    await redis_client.flushdb()
    yield
    await redis_client.flushdb()

# Fix 3: Mock time-dependent tests
from unittest.mock import patch
from datetime import datetime, timedelta

@pytest.mark.asyncio
async def test_token_expiry(client):
    """Test with controlled time"""
    # Create token that expires in 1 hour
    token = create_token(expires_in=3600)
    
    # Simulate time passing: mock datetime to return future time
    future_time = datetime.utcnow() + timedelta(hours=2)
    with patch("app.auth.datetime") as mock_dt:
        mock_dt.utcnow.return_value = future_time
        
        response = await client.get("/protected", headers={
            "Authorization": f"Bearer {token}"
        })
        assert response.status_code == 401  # Token should be expired

# Fix 4: Handle flaky async tests
import asyncio
import pytest

@pytest.mark.asyncio
async def test_websocket_connection(client):
    """WebSocket test with proper timeout"""
    try:
        async with asyncio.timeout(5.0):  # Fail after 5 seconds
            async with client.websocket_connect("/ws") as ws:
                await ws.send_json({"message": "hello"})
                response = await ws.receive_json()
                assert response["type"] == "message"
    except asyncio.TimeoutError:
        pytest.fail("WebSocket test timed out")

# Fix 5: Docker Compose for consistent test environment
# docker-compose.test.yml
"""
version: '3.8'
services:
  test:
    build: .
    command: pytest tests/ -v
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db/testdb
      REDIS_URL: redis://redis:6379/1
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
  
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
"""

def create_token(expires_in: int) -> str:
    return "test_token"
```

### Key Takeaways
> - 💡 **Pin all dependencies** with exact versions (`pip freeze > requirements.txt`)
> - 💡 **Use Docker Compose** for CI — identical environment every time
> - 💡 **`autouse=True` fixtures** reset state automatically between tests
> - 💡 **Mock time** for tests involving token expiry or scheduled events
> - 💡 **Add `pytest-timeout`** to catch hanging tests (async deadlocks)

---

## Question 53 — Test Endpoint Depending on Third-Party Payment API
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your checkout endpoint calls Stripe's payment API. You can't call Stripe in tests (costs money, unreliable, slow). How do you test this properly?"*

### The Answer

```
TESTING STRATEGIES FOR EXTERNAL APIs:

1. MOCK (httpx-mock, responses library)
   → Intercept HTTP calls, return fake responses
   → Fast, free, deterministic
   → Risk: mock may not match real API behavior

2. VCR (cassette recording)
   → Record real API responses once
   → Replay in tests
   → Fast after recording, realistic

3. Sandbox environments
   → Stripe test mode (free, realistic)
   → Use for integration tests only

4. Contract testing (Pact)
   → Define contract between your service and Stripe
   → Verify both sides match the contract
```

### Code Example — Mocking External APIs

```python
import pytest
import httpx
import pytest_httpx  # pip install pytest-httpx
from unittest.mock import AsyncMock, patch

# ---- Your payment service ----
class PaymentService:
    def __init__(self, http_client: httpx.AsyncClient):
        self.client = http_client
        self.stripe_url = "https://api.stripe.com/v1"
    
    async def charge_card(self, amount: int, payment_method_id: str) -> dict:
        """Charge card via Stripe"""
        response = await self.client.post(
            f"{self.stripe_url}/payment_intents",
            data={
                "amount": amount,
                "currency": "usd",
                "payment_method": payment_method_id,
                "confirm": True,
            },
            headers={"Authorization": "Bearer sk_test_..."}
        )
        response.raise_for_status()
        return response.json()

# ---- Tests with mocked Stripe ----
@pytest.mark.asyncio
async def test_successful_payment(httpx_mock):
    """Test payment processing with mocked Stripe response"""
    # Setup mock to return successful Stripe response
    httpx_mock.add_response(
        url="https://api.stripe.com/v1/payment_intents",
        method="POST",
        json={
            "id": "pi_test_123",
            "status": "succeeded",
            "amount": 2000,
            "currency": "usd",
        },
        status_code=200,
    )
    
    async with httpx.AsyncClient() as client:
        service = PaymentService(client)
        result = await service.charge_card(2000, "pm_card_visa")
    
    assert result["status"] == "succeeded"
    assert result["amount"] == 2000

@pytest.mark.asyncio
async def test_payment_card_declined(httpx_mock):
    """Test handling of declined card"""
    httpx_mock.add_response(
        url="https://api.stripe.com/v1/payment_intents",
        method="POST",
        json={
            "error": {
                "type": "card_error",
                "code": "card_declined",
                "message": "Your card was declined."
            }
        },
        status_code=402,
    )
    
    async with httpx.AsyncClient() as client:
        service = PaymentService(client)
        with pytest.raises(httpx.HTTPStatusError):
            await service.charge_card(2000, "pm_card_declined")

@pytest.mark.asyncio
async def test_payment_network_timeout(httpx_mock):
    """Test handling of Stripe timeout"""
    httpx_mock.add_exception(
        httpx.TimeoutException("Connection timed out")
    )
    
    async with httpx.AsyncClient() as client:
        service = PaymentService(client)
        with pytest.raises(httpx.TimeoutException):
            await service.charge_card(2000, "pm_card_visa")

# ---- Dependency injection for testability ----
# app/services/payment.py
from fastapi import FastAPI, Depends
import httpx

app = FastAPI()

def get_payment_client():
    return httpx.AsyncClient(
        base_url="https://api.stripe.com/v1",
        headers={"Authorization": f"Bearer {settings.STRIPE_SECRET_KEY}"}
    )

@app.post("/checkout")
async def checkout(
    payment_method_id: str,
    amount: int,
    client: httpx.AsyncClient = Depends(get_payment_client)
):
    """Checkout endpoint with injectable HTTP client"""
    try:
        result = await client.post("/payment_intents", data={...})
        return {"status": "paid", "payment_id": result.json()["id"]}
    except httpx.HTTPStatusError as e:
        raise HTTPException(402, "Payment declined")

class settings:
    STRIPE_SECRET_KEY = "sk_test_..."
```

### Key Takeaways
> - 💡 **pytest-httpx** intercepts all HTTP calls — no real network requests
> - 💡 **Test all scenarios**: success, declined, network error, timeout
> - 💡 **Dependency injection** makes services testable (inject mock client)
> - 💡 **Stripe test mode** for integration tests — uses real API, free, has test card numbers
> - 💡 **Never call payment API in unit tests** — slow, costs money, flaky

---

## Question 54 — Deployment Causes 30 Seconds Downtime
🟡 Mid | ★★★ Very Common

### The Scenario
> *"Every time you deploy your API, there's 30 seconds of downtime where users get 503 errors. Your server team says 'just deploy at 3AM'. How do you achieve true zero-downtime deployment?"*

### The Answer

```
ZERO-DOWNTIME DEPLOYMENT STRATEGIES:

BLUE-GREEN DEPLOYMENT:
                        ┌─────────────────┐
                        │  Load Balancer  │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
         (100% traffic)     (0% traffic)
              │                  │
        ┌─────▼──────┐     ┌─────▼──────┐
        │   Blue     │     │   Green    │
        │   (v1)     │     │   (v2)     │
        │  Running   │     │  Standby   │
        └────────────┘     └────────────┘

Deploy step:
1. Deploy v2 to Green (while Blue serves 100%)
2. Run health checks on Green
3. Switch LB: Blue=0%, Green=100%
4. If problems: switch back in seconds
```

```mermaid
sequenceDiagram
    participant LB as Load Balancer
    participant B as Blue (v1)
    participant G as Green (v2)
    participant Deploy as Deploy Process

    Note over B: Serving 100% traffic
    Deploy->>G: Deploy new version
    Deploy->>G: Health check /health
    G-->>Deploy: 200 OK (healthy)
    Deploy->>LB: Switch traffic to Green
    LB->>G: Forward traffic (100%)
    Note over B: Now idle
    Note over G: Serving 100% traffic
    Deploy->>B: Optional: keep as rollback
```

### Code Example — FastAPI Graceful Shutdown + Health Check

```python
import asyncio
import signal
from contextlib import asynccontextmanager
from fastapi import FastAPI, Response
from typing import Optional

# Track in-flight requests
in_flight_requests = 0
shutting_down = False

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Startup and shutdown lifecycle management"""
    global shutting_down
    
    # Startup
    print("Starting application...")
    await startup_database_pool()
    await startup_redis_connection()
    print("Application ready!")
    
    yield  # Application runs
    
    # Graceful shutdown
    shutting_down = True
    print("Shutdown initiated, waiting for in-flight requests...")
    
    # Wait up to 30 seconds for in-flight requests to complete
    for _ in range(300):  # 300 × 0.1s = 30s
        if in_flight_requests == 0:
            break
        await asyncio.sleep(0.1)
    
    print(f"Shutting down with {in_flight_requests} requests still pending")
    await shutdown_database_pool()
    await shutdown_redis_connection()
    print("Shutdown complete")

app = FastAPI(lifespan=lifespan)

@app.middleware("http")
async def track_requests(request, call_next):
    global in_flight_requests
    
    if shutting_down:
        return Response(
            content='{"error": "Service is shutting down, please retry"}',
            status_code=503,
            headers={"Retry-After": "5"},
            media_type="application/json"
        )
    
    in_flight_requests += 1
    try:
        return await call_next(request)
    finally:
        in_flight_requests -= 1

@app.get("/health")
async def health_check():
    """
    Kubernetes/Load Balancer health check.
    Returns 200 when ready to receive traffic.
    Returns 503 when shutting down.
    """
    if shutting_down:
        return Response(
            content='{"status": "shutting_down"}',
            status_code=503,
            media_type="application/json"
        )
    
    # Check critical dependencies
    health_status = {
        "status": "healthy",
        "database": await check_database(),
        "redis": await check_redis(),
    }
    
    all_healthy = all(v == "ok" for k, v in health_status.items() if k != "status")
    if not all_healthy:
        health_status["status"] = "unhealthy"
        return Response(
            content=str(health_status),
            status_code=503,
            media_type="application/json"
        )
    
    return health_status

@app.get("/health/ready")
async def readiness_check():
    """Kubernetes readiness probe — is app ready to receive traffic?"""
    if shutting_down:
        return Response(status_code=503)
    return {"ready": True}

@app.get("/health/live")
async def liveness_check():
    """Kubernetes liveness probe — is app alive (not hung)?"""
    return {"alive": True}

async def startup_database_pool():
    pass

async def startup_redis_connection():
    pass

async def shutdown_database_pool():
    pass

async def shutdown_redis_connection():
    pass

async def check_database() -> str:
    return "ok"

async def check_redis() -> str:
    return "ok"
```

### Kubernetes Deployment for Zero Downtime

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0    # Never have 0 available pods
      maxSurge: 1          # Add 1 extra pod during update
  template:
    spec:
      containers:
      - name: api
        image: myapp:v2
        readinessProbe:    # Wait until ready before sending traffic
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:     # Restart if hung
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        lifecycle:
          preStop:         # Give connections time to drain before kill
            exec:
              command: ["/bin/sleep", "10"]
      terminationGracePeriodSeconds: 60  # Wait 60s before force kill
```

### Key Takeaways
> - 💡 **Blue-green** = instant rollback capability (keep old version running)
> - 💡 **Rolling updates** = gradual replacement, no full downtime
> - 💡 **Health checks** = LB only sends traffic to healthy instances
> - 💡 **`preStop` hook** = give connections 10s to drain before pod is removed
> - 💡 **`terminationGracePeriodSeconds`** must be longer than your slowest request

---

## Question 55 — Manual Verification After Every Deployment
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your team manually tests 20 endpoints after every deployment. It takes 2 hours and requires 3 people. How do you automate post-deployment verification?"*

### The Answer

### Code Example — Smoke Tests + Auto-Rollback

```python
# scripts/smoke_tests.py
import asyncio
import sys
import httpx

BASE_URL = "https://api.production.com"

async def run_smoke_tests(base_url: str) -> bool:
    """Run critical smoke tests after deployment"""
    async with httpx.AsyncClient(base_url=base_url, timeout=10.0) as client:
        tests = [
            check_health(client),
            check_auth_login(client),
            check_public_endpoints(client),
            check_database_connectivity(client),
        ]
        
        results = await asyncio.gather(*tests, return_exceptions=True)
        
        passed = all(r is True for r in results)
        
        for test, result in zip(tests, results):
            status = "✅ PASS" if result is True else f"❌ FAIL: {result}"
            print(f"  {test.__name__}: {status}")
        
        return passed

async def check_health(client: httpx.AsyncClient) -> bool:
    response = await client.get("/health")
    assert response.status_code == 200, f"Health check failed: {response.status_code}"
    assert response.json()["status"] == "healthy"
    return True

async def check_auth_login(client: httpx.AsyncClient) -> bool:
    response = await client.post("/auth/login", json={
        "email": "smoketest@example.com",
        "password": "SmokeTestPass123!"
    })
    assert response.status_code == 200, f"Login failed: {response.status_code}"
    assert "access_token" in response.json()
    return True

async def check_public_endpoints(client: httpx.AsyncClient) -> bool:
    endpoints = ["/products", "/categories", "/health/metrics"]
    for endpoint in endpoints:
        response = await client.get(endpoint)
        assert response.status_code == 200, f"{endpoint} returned {response.status_code}"
    return True

async def check_database_connectivity(client: httpx.AsyncClient) -> bool:
    response = await client.get("/health/detailed")
    assert response.json()["database"] == "ok"
    return True

if __name__ == "__main__":
    base_url = sys.argv[1] if len(sys.argv) > 1 else BASE_URL
    success = asyncio.run(run_smoke_tests(base_url))
    sys.exit(0 if success else 1)

# CI/CD pipeline integration (.github/workflows/deploy.yml):
"""
- name: Run smoke tests
  run: python scripts/smoke_tests.py ${{ env.PRODUCTION_URL }}
  
- name: Auto-rollback on failure
  if: failure()
  run: |
    kubectl rollout undo deployment/api-deployment
    echo "🚨 Deployment rolled back due to smoke test failure"
"""
```

### Key Takeaways
> - 💡 **Smoke tests** verify critical paths work — not comprehensive, just "is it alive?"
> - 💡 **Auto-rollback** on smoke test failure — no human needed
> - 💡 **Dedicated smoke test user** in database — don't pollute production data
> - 💡 **Run in CI/CD pipeline** as the final step after deployment
> - 💡 **Alert on failure** via Slack/PagerDuty immediately

---

## Question 56 — Configuration for Dev/Staging/Prod Environments
🟢 Junior | ★★★ Very Common

### The Scenario
> *"Your API has hardcoded database URLs and API keys in the source code. Different environments use different values. Someone accidentally deployed dev config to production. How do you manage configuration properly?"*

### The Answer

### Code Example — Pydantic Settings (12-Factor App)

```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Optional
from functools import lru_cache

class Settings(BaseSettings):
    """
    All configuration from environment variables.
    Never hardcode values — they come from .env files or CI/CD secrets.
    """
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False
    )
    
    # Environment
    ENVIRONMENT: str = "development"  # development | staging | production
    DEBUG: bool = False
    
    # Database
    DATABASE_URL: str  # Required — no default!
    DATABASE_POOL_SIZE: int = 10
    DATABASE_POOL_OVERFLOW: int = 20
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"
    
    # Security
    SECRET_KEY: str  # Required — no default!
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 15
    
    # External APIs
    STRIPE_SECRET_KEY: Optional[str] = None
    OPENAI_API_KEY: Optional[str] = None
    AWS_ACCESS_KEY_ID: Optional[str] = None
    AWS_SECRET_ACCESS_KEY: Optional[str] = None
    S3_BUCKET_NAME: Optional[str] = None
    
    # CORS
    CORS_ALLOWED_ORIGINS: str = "http://localhost:3000"
    
    @property
    def cors_origins_list(self) -> list[str]:
        return [o.strip() for o in self.CORS_ALLOWED_ORIGINS.split(",")]
    
    @property
    def is_production(self) -> bool:
        return self.ENVIRONMENT == "production"
    
    @property
    def is_development(self) -> bool:
        return self.ENVIRONMENT == "development"

@lru_cache()
def get_settings() -> Settings:
    """Cache settings — only loaded once"""
    return Settings()

settings = get_settings()

# .env.example (committed to repository — shows what's needed, no actual values)
"""
ENVIRONMENT=development
DEBUG=true
DATABASE_URL=postgresql+asyncpg://user:password@localhost:5432/mydb_dev
REDIS_URL=redis://localhost:6379/0
SECRET_KEY=your-secret-key-here-change-in-production
CORS_ALLOWED_ORIGINS=http://localhost:3000
STRIPE_SECRET_KEY=sk_test_...
"""

# .env (never committed — actual values for local dev)
# .gitignore includes: .env, .env.local, .env.*.local

# app/main.py
from app.core.config import settings, get_settings
from fastapi import FastAPI, Depends

def create_app():
    app = FastAPI(
        title="My API",
        debug=settings.DEBUG,
        docs_url="/docs" if not settings.is_production else None,
        redoc_url="/redoc" if not settings.is_production else None,
    )
    return app

# Use in dependency injection
@app.get("/admin/config")
async def show_config(settings: Settings = Depends(get_settings)):
    """Show non-sensitive config values (admin only)"""
    return {
        "environment": settings.ENVIRONMENT,
        "debug": settings.DEBUG,
        "database_pool_size": settings.DATABASE_POOL_SIZE,
        # NEVER return: SECRET_KEY, API keys, passwords!
    }
```

### Key Takeaways
> - 💡 **12-factor app**: config from environment, never from code
> - 💡 **Required settings** have no default — app fails fast if missing
> - 💡 **Commit `.env.example`**, never commit `.env`
> - 💡 **Use CI/CD secrets** for production (GitHub Secrets, Vault, AWS Parameter Store)
> - 💡 **Disable docs/debug in production** automatically based on ENVIRONMENT setting

---

## Question 57 — Test Suite Takes 45 Minutes
🟡 Mid | ★★☆ Common

### The Scenario
> *"Your test suite takes 45 minutes. Developers avoid running tests before committing because it's too slow. How do you speed it up?"*

### The Answer

### Code Example — Parallel Tests + Optimization

```python
# pytest.ini
"""
[tool:pytest]
# Run tests in parallel with pytest-xdist
addopts = -n auto --timeout=30

# Categorize tests
markers =
    unit: fast unit tests (< 1s each)
    integration: integration tests (1-10s each)  
    slow: slow tests (> 10s each)
    smoke: smoke tests for deployment verification
"""

# conftest.py
import pytest
from typing import Generator

# Expensive fixtures at session scope (created once, shared)
@pytest.fixture(scope="session")
async def db_engine():
    """Create DB engine ONCE for entire test session"""
    engine = create_async_engine("sqlite+aiosqlite:///:memory:")
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

# Cheap fixtures at function scope (fresh for each test)
@pytest.fixture
async def db(db_engine):
    """Fresh transaction for each test (rollback after)"""
    async with AsyncSession(db_engine) as session:
        async with session.begin_nested():  # Savepoint
            yield session
            await session.rollback()  # Never commits

# Mark slow tests
@pytest.mark.slow
@pytest.mark.asyncio
async def test_full_checkout_flow():
    """Full integration test — marked as slow"""
    pass

@pytest.mark.unit
def test_password_validation():
    """Fast unit test — no DB needed"""
    assert validate_password("weak") == False
    assert validate_password("StrongPass123!") == True

def validate_password(password: str) -> bool:
    return len(password) >= 8

class Base:
    pass

# Run only fast tests during development
# pytest -m "not slow" --timeout=10
# Run all tests in CI
# pytest --timeout=60

# Makefile targets:
"""
test-fast:
    pytest -m "unit or smoke" -n auto --timeout=10

test-full:
    pytest -n auto --timeout=60 --cov=app --cov-report=html

test-ci:
    pytest -n auto --timeout=60 --cov=app --cov-fail-under=70
"""
```

### Key Takeaways
> - 💡 **`pytest-xdist` with `-n auto`** runs tests in parallel — huge speedup
> - 💡 **Session-scoped fixtures** for expensive setup (DB engine, app)
> - 💡 **Test markers** separate fast/slow tests — run fast locally, full in CI
> - 💡 **In-memory SQLite** instead of real PostgreSQL for unit tests
> - 💡 **Rollback instead of truncate** — much faster test isolation

---

## Question 58 — Test WebSocket Endpoints in FastAPI
🟡 Mid | ★★☆ Common

### The Scenario
> *"You have WebSocket endpoints for real-time features. How do you write automated tests for WebSocket connections, message sending, and receiving?"*

### The Answer

### Code Example — WebSocket Testing

```python
import asyncio
import pytest
from httpx import AsyncClient, ASGITransport
from fastapi.testclient import TestClient
from app.main import app

# Synchronous WebSocket testing (simpler)
def test_websocket_basic():
    """Test WebSocket with synchronous TestClient"""
    with TestClient(app) as client:
        with client.websocket_connect("/ws/room1?user_id=1") as ws:
            ws.send_json({"type": "message", "content": "Hello!"})
            data = ws.receive_json()
            assert data["type"] == "new_message"
            assert data["content"] == "Hello!"

def test_websocket_authentication():
    """Test that WebSocket requires auth"""
    with TestClient(app) as client:
        # No auth token — should be rejected
        with pytest.raises(Exception):
            with client.websocket_connect("/ws/room1") as ws:
                pass

def test_websocket_broadcast():
    """Test that messages are broadcast to all users in room"""
    with TestClient(app) as client:
        # Connect two users to same room
        with client.websocket_connect("/ws/room1?user_id=1") as ws1:
            with client.websocket_connect("/ws/room1?user_id=2") as ws2:
                # User 1 sends message
                ws1.send_json({"type": "message", "content": "Hi everyone!"})
                
                # User 2 should receive it
                message = ws2.receive_json()
                assert message["content"] == "Hi everyone!"
                assert message["sender_id"] == 1

def test_websocket_disconnect_handling():
    """Test clean disconnection handling"""
    with TestClient(app) as client:
        with client.websocket_connect("/ws/room1?user_id=1") as ws1:
            with client.websocket_connect("/ws/room1?user_id=2") as ws2:
                # User 1 disconnects
                pass  # ws1 context manager exits = disconnection
            
            # After disconnect, user 2 should receive notification
            data = ws2.receive_json()
            assert data["type"] == "user_left"

# For the WebSocket endpoint being tested:
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

test_app = FastAPI()
test_connections: dict = {}

@test_app.websocket("/ws/{room_id}")
async def websocket_endpoint(websocket: WebSocket, room_id: str, user_id: int = 1):
    await websocket.accept()
    
    if room_id not in test_connections:
        test_connections[room_id] = []
    test_connections[room_id].append((user_id, websocket))
    
    try:
        while True:
            data = await websocket.receive_json()
            
            # Broadcast to all in room
            for uid, ws in test_connections.get(room_id, []):
                if ws != websocket:
                    await ws.send_json({
                        "type": "new_message",
                        "content": data.get("content"),
                        "sender_id": user_id
                    })
    except WebSocketDisconnect:
        test_connections[room_id] = [
            (uid, ws) for uid, ws in test_connections.get(room_id, [])
            if ws != websocket
        ]
        for uid, ws in test_connections.get(room_id, []):
            await ws.send_json({"type": "user_left", "user_id": user_id})
```

### Key Takeaways
> - 💡 **FastAPI TestClient** supports WebSocket testing synchronously (simpler)
> - 💡 **Test the full flow**: connect, send, receive, disconnect
> - 💡 **Test broadcast** with multiple simultaneous connections
> - 💡 **Test disconnect handling** — ensure cleanup happens correctly
> - 💡 **Test authentication** — unauthenticated connections should be rejected

---

## Question 59 — Containerize and Deploy to Kubernetes
🔴 Senior | ★★☆ Common

### The Scenario
> *"Your team wants to move from bare-metal deployments to containers. How do you containerize your FastAPI app and deploy it to Kubernetes with best practices?"*

### The Answer

### Code Example — Optimized Dockerfile + K8s Deployment

```dockerfile
# Dockerfile - Multi-stage build for minimal image size
FROM python:3.11-slim as builder

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Install Python dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ---- Production stage ----
FROM python:3.11-slim

# Security: run as non-root user
RUN useradd --create-home appuser
WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
    CMD python -c "import httpx; httpx.get('http://localhost:8000/health').raise_for_status()"

EXPOSE 8000

# Use exec form to handle signals properly
CMD ["python", "-m", "uvicorn", "app.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "4", \
     "--no-access-log"]
```

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myregistry/api:v1.2.3  # Always use specific tag, never :latest
        ports:
        - containerPort: 8000
        
        # Resource limits prevent one pod from consuming all resources
        resources:
          requests:
            cpu: "250m"     # 0.25 CPU cores (minimum guaranteed)
            memory: "256Mi"
          limits:
            cpu: "500m"     # 0.5 CPU cores (maximum)
            memory: "512Mi"
        
        # Environment variables from Kubernetes secrets (not in code!)
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: database_url
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: api-secrets
              key: secret_key
        
        # Probes for zero-downtime deployment
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 5
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]  # Drain connections

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Scale up when CPU > 70%
```

### Key Takeaways
> - 💡 **Multi-stage builds** — smaller images (dev dependencies not in production)
> - 💡 **Run as non-root** — security best practice
> - 💡 **Resource limits** — prevents one pod from starving others
> - 💡 **Secrets from Kubernetes Secrets** — never in Dockerfile or configmap
> - 💡 **HPA** scales automatically based on CPU/memory — handles traffic spikes

---

## Question 60 — A/B Testing for New API Response Format
🔴 Senior | ★☆☆ Rare

### The Scenario
> *"You want to test a new JSON response format on 10% of traffic before rolling it out to everyone. How do you implement A/B testing in your API?"*

### The Answer

### Code Example — Feature Flags for A/B Testing

```python
import hashlib
import os
from fastapi import FastAPI, Request, Response
from typing import Optional

app = FastAPI()

def get_user_variant(user_id: str, experiment: str) -> str:
    """
    Deterministic variant assignment — same user always gets same variant.
    Uses hash so no need to store assignment in database.
    """
    hash_input = f"{user_id}:{experiment}"
    hash_value = int(hashlib.md5(hash_input.encode()).hexdigest(), 16)
    
    # 10% get variant B, 90% get variant A
    if hash_value % 100 < 10:
        return "B"
    return "A"

@app.get("/users/{user_id}")
async def get_user(user_id: str, request: Request):
    """Return different response formats based on A/B test assignment"""
    variant = get_user_variant(user_id, "response_format_v2")
    
    user_data = {"id": user_id, "first_name": "John", "last_name": "Doe", "email": "john@example.com"}
    
    # Add variant to response headers for tracking
    headers = {"X-Experiment-Variant": variant, "X-Experiment-Name": "response_format_v2"}
    
    if variant == "B":
        # New format: split name fields
        return Response(
            content=str({
                "id": user_data["id"],
                "first_name": user_data["first_name"],
                "last_name": user_data["last_name"],
                "email": user_data["email"],
            }),
            headers=headers
        )
    else:
        # Current format: combined name
        return Response(
            content=str({
                "id": user_data["id"],
                "name": f"{user_data['first_name']} {user_data['last_name']}",
                "email": user_data["email"],
            }),
            headers=headers
        )
```

### Key Takeaways
> - 💡 **Deterministic hash** — same user always gets same variant (consistent experience)
> - 💡 **Feature flags** (LaunchDarkly, Unleash) for production A/B testing
> - 💡 **Track metrics** per variant — error rate, latency, business metrics
> - 💡 **Graduate rollout** — 1% → 10% → 50% → 100%
> - 💡 **Always have a kill switch** — instantly revert all users to control

---

*Next: [07 — Real-World Integration →](./07-real-world-integration.md)*
