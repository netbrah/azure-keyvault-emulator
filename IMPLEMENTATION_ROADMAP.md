# Python Implementation Roadmap

## Quick Start: Choose Your Path

### Path A: Python Client Library (Recommended First Step)
**Timeline**: 1-2 weeks | **Risk**: Low | **Value**: High

Start here to quickly add Python support without rewriting everything.

[Jump to Python Client Roadmap](#path-a-python-client-library-1-2-weeks)

### Path B: Full Python Port
**Timeline**: 6-8 weeks | **Risk**: Medium | **Value**: Very High (if needed)

Complete rewrite in Python for pure Python environment.

[Jump to Full Port Roadmap](#path-b-full-python-port-6-8-weeks)

---

## Path A: Python Client Library (1-2 weeks)

### Week 1: Core Implementation

#### Day 1-2: Project Setup & Secrets API

**Tasks**:
1. Initialize Python project
2. Set up package structure
3. Implement base EmulatorClient
4. Add Secrets operations
5. Basic error handling

**Deliverables**:
```
azure-keyvault-emulator-client/
├── pyproject.toml
├── README.md
├── src/
│   └── azure_keyvault_emulator_client/
│       ├── __init__.py
│       ├── client.py          # ✅ Base client
│       ├── secrets.py         # ✅ Secrets ops
│       ├── models.py          # ✅ Data models
│       └── exceptions.py      # ✅ Exceptions
└── tests/
    ├── __init__.py
    └── test_secrets.py        # ✅ Tests
```

**Code Example**:
```python
# src/azure_keyvault_emulator_client/client.py
import httpx
from typing import Optional

class EmulatorClient:
    """Azure Key Vault Emulator client."""
    
    def __init__(
        self,
        vault_url: str = "https://localhost:4997",
        api_version: str = "7.5",
        verify_ssl: bool = False
    ):
        self.vault_url = vault_url.rstrip('/')
        self.api_version = api_version
        self.client = httpx.AsyncClient(
            base_url=vault_url,
            verify=verify_ssl,
            headers={
                "Authorization": "Bearer emulator-token",
                "Content-Type": "application/json"
            }
        )
    
    async def set_secret(self, name: str, value: str, **kwargs) -> dict:
        """Set a secret in the vault."""
        response = await self.client.put(
            f"/secrets/{name}",
            params={"api-version": self.api_version},
            json={"value": value, **kwargs}
        )
        response.raise_for_status()
        return response.json()
    
    async def get_secret(self, name: str, version: str = "") -> dict:
        """Get a secret from the vault."""
        url = f"/secrets/{name}/{version}" if version else f"/secrets/{name}"
        response = await self.client.get(
            url,
            params={"api-version": self.api_version}
        )
        response.raise_for_status()
        return response.json()
    
    async def close(self):
        """Close the client connection."""
        await self.client.aclose()
    
    async def __aenter__(self):
        return self
    
    async def __aexit__(self, *args):
        await self.close()
```

**Testing**:
```python
# tests/test_secrets.py
import pytest
from azure_keyvault_emulator_client import EmulatorClient

@pytest.mark.asyncio
async def test_set_and_get_secret():
    async with EmulatorClient() as client:
        # Set secret
        result = await client.set_secret("test-secret", "test-value")
        assert result["value"] == "test-value"
        
        # Get secret
        secret = await client.get_secret("test-secret")
        assert secret["value"] == "test-value"
```

#### Day 3-4: Keys API

**Tasks**:
1. Implement Keys operations
2. Add cryptographic operation support
3. Write tests

**Deliverables**:
```python
# Add to client.py
async def create_key(self, name: str, key_type: str = "RSA", **kwargs) -> dict:
    """Create a new key."""
    response = await self.client.post(
        f"/keys/{name}/create",
        params={"api-version": self.api_version},
        json={"kty": key_type, **kwargs}
    )
    response.raise_for_status()
    return response.json()

async def get_key(self, name: str, version: str = "") -> dict:
    """Get a key from the vault."""
    url = f"/keys/{name}/{version}" if version else f"/keys/{name}"
    response = await self.client.get(
        url,
        params={"api-version": self.api_version}
    )
    response.raise_for_status()
    return response.json()

async def encrypt(self, name: str, algorithm: str, value: bytes) -> dict:
    """Encrypt data with a key."""
    import base64
    response = await self.client.post(
        f"/keys/{name}/encrypt",
        params={"api-version": self.api_version},
        json={
            "alg": algorithm,
            "value": base64.b64encode(value).decode()
        }
    )
    response.raise_for_status()
    return response.json()

async def decrypt(self, name: str, algorithm: str, ciphertext: bytes) -> dict:
    """Decrypt data with a key."""
    import base64
    response = await self.client.post(
        f"/keys/{name}/decrypt",
        params={"api-version": self.api_version},
        json={
            "alg": algorithm,
            "value": base64.b64encode(ciphertext).decode()
        }
    )
    response.raise_for_status()
    return response.json()
```

#### Day 5: Certificates API

**Tasks**:
1. Implement Certificates operations
2. Add policy management
3. Write tests

**Deliverables**:
```python
# Add to client.py
async def create_certificate(
    self, 
    name: str, 
    policy: Optional[dict] = None
) -> dict:
    """Create a certificate."""
    response = await self.client.post(
        f"/certificates/{name}/create",
        params={"api-version": self.api_version},
        json={"policy": policy or self._default_policy()}
    )
    response.raise_for_status()
    return response.json()

async def get_certificate(self, name: str, version: str = "") -> dict:
    """Get a certificate from the vault."""
    url = (f"/certificates/{name}/{version}" if version 
           else f"/certificates/{name}")
    response = await self.client.get(
        url,
        params={"api-version": self.api_version}
    )
    response.raise_for_status()
    return response.json()

def _default_policy(self) -> dict:
    """Get default certificate policy."""
    return {
        "key_props": {"exportable": True, "kty": "RSA", "key_size": 2048},
        "secret_props": {"contentType": "application/x-pkcs12"},
        "x509_props": {
            "subject": "CN=DefaultCert",
            "validity_months": 12
        }
    }
```

### Week 2: Polish & Release

#### Day 6-7: Enhanced Features

**Tasks**:
1. Add synchronous client option
2. Improve error handling
3. Add logging
4. Type hints throughout

**Deliverables**:
```python
# Sync client wrapper
from typing import Any
import asyncio

class SyncEmulatorClient:
    """Synchronous wrapper for EmulatorClient."""
    
    def __init__(self, *args, **kwargs):
        self._async_client = EmulatorClient(*args, **kwargs)
        self._loop = asyncio.new_event_loop()
    
    def set_secret(self, name: str, value: str, **kwargs) -> dict:
        return self._loop.run_until_complete(
            self._async_client.set_secret(name, value, **kwargs)
        )
    
    def get_secret(self, name: str, version: str = "") -> dict:
        return self._loop.run_until_complete(
            self._async_client.get_secret(name, version)
        )
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self._loop.run_until_complete(self._async_client.close())
        self._loop.close()
```

#### Day 8-9: Documentation

**Tasks**:
1. Write comprehensive README
2. Add API documentation
3. Create usage examples
4. Docker Compose example

**Deliverables**:

```markdown
# README.md

## Installation

```bash
pip install azure-keyvault-emulator-client
```

## Quick Start

### Async Usage
```python
from azure_keyvault_emulator_client import EmulatorClient

async with EmulatorClient("https://localhost:4997") as client:
    # Secrets
    await client.set_secret("my-secret", "my-value")
    secret = await client.get_secret("my-secret")
    
    # Keys
    await client.create_key("my-key", key_type="RSA")
    key = await client.get_key("my-key")
    
    # Certificates
    await client.create_certificate("my-cert")
    cert = await client.get_certificate("my-cert")
```

### Sync Usage
```python
from azure_keyvault_emulator_client import SyncEmulatorClient

with SyncEmulatorClient("https://localhost:4997") as client:
    client.set_secret("my-secret", "my-value")
    secret = client.get_secret("my-secret")
```

## Running the Emulator

```yaml
# docker-compose.yml
version: '3.8'
services:
  keyvault:
    image: jamesgoulddev/azure-keyvault-emulator:latest
    ports:
      - "4997:4997"
    volumes:
      - ./certs:/certs
    environment:
      - Persist=true
```

```bash
docker-compose up -d
```
```

#### Day 10: Package & Publish

**Tasks**:
1. Configure pyproject.toml
2. Build package
3. Publish to PyPI
4. Create release notes

**Deliverables**:

```toml
# pyproject.toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "azure-keyvault-emulator-client"
version = "0.1.0"
description = "Python client for Azure Key Vault Emulator"
authors = [
    {name = "Your Name", email = "you@example.com"}
]
readme = "README.md"
requires-python = ">=3.8"
dependencies = [
    "httpx>=0.24.0",
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

[project.urls]
Homepage = "https://github.com/james-gould/azure-keyvault-emulator"
Documentation = "https://github.com/james-gould/azure-keyvault-emulator"
Repository = "https://github.com/james-gould/azure-keyvault-emulator"

[project.optional-dependencies]
dev = [
    "pytest>=7.4.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.0.0",
    "mypy>=1.0.0",
    "ruff>=0.0.280",
]
```

**Publishing**:
```bash
# Build
python -m build

# Upload to PyPI
python -m twine upload dist/*
```

---

## Path B: Full Python Port (6-8 weeks)

### Phase 1: Foundation (Week 1)

#### Sprint Goals
- ✅ Project structure set up
- ✅ FastAPI application running
- ✅ Database models created
- ✅ Basic routing established
- ✅ Docker environment working

#### Tasks Breakdown

**Day 1: Project Initialization**
```bash
# Create project
mkdir azure-keyvault-emulator-python
cd azure-keyvault-emulator-python

# Initialize with poetry
poetry init
poetry add fastapi uvicorn sqlalchemy aiosqlite pydantic cryptography pyjwt

# Or with pip/requirements.txt
pip install fastapi uvicorn[standard] sqlalchemy[asyncio] \
    aiosqlite pydantic cryptography pyjwt
```

**Project Structure**:
```
azure-keyvault-emulator-python/
├── src/
│   └── azure_keyvault_emulator/
│       ├── __init__.py
│       ├── main.py                 # Application entry
│       ├── config.py               # Configuration
│       ├── database.py             # Database setup
│       ├── auth.py                 # Authentication
│       ├── models/                 # Pydantic models
│       │   ├── __init__.py
│       │   ├── secrets.py
│       │   ├── keys.py
│       │   └── certificates.py
│       ├── routers/                # API routes
│       │   ├── __init__.py
│       │   ├── secrets.py
│       │   ├── keys.py
│       │   └── certificates.py
│       ├── services/               # Business logic
│       │   ├── __init__.py
│       │   ├── secret_service.py
│       │   ├── key_service.py
│       │   └── certificate_service.py
│       ├── middleware/             # Middleware
│       │   ├── __init__.py
│       │   └── error_handler.py
│       ├── db_models/              # SQLAlchemy models
│       │   ├── __init__.py
│       │   └── vault_models.py
│       └── utils/                  # Utilities
│           ├── __init__.py
│           ├── crypto.py
│           └── token.py
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_secrets.py
│   ├── test_keys.py
│   └── test_certificates.py
├── Dockerfile
├── docker-compose.yml
├── pyproject.toml
└── README.md
```

**Day 2-3: Database Layer**

```python
# src/azure_keyvault_emulator/database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker, declarative_base
from sqlalchemy import Column, String, Boolean, DateTime, Index
from datetime import datetime

Base = declarative_base()

class SecretModel(Base):
    __tablename__ = "secrets"
    
    id = Column(String, primary_key=True)
    persisted_name = Column(String, nullable=False, index=True)
    version = Column(String, nullable=False)
    data = Column(String, nullable=False)  # JSON
    deleted = Column(Boolean, default=False)
    created = Column(DateTime, default=datetime.utcnow)
    updated = Column(DateTime, onupdate=datetime.utcnow)
    
    __table_args__ = (
        Index('ix_name_version', 'persisted_name', 'version', unique=True),
    )

# Similar models for Key and Certificate

DATABASE_URL = "sqlite+aiosqlite:///./emulator.db"
engine = create_async_engine(DATABASE_URL, echo=True)
async_session_maker = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

async def get_session() -> AsyncSession:
    async with async_session_maker() as session:
        yield session

async def init_db():
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
```

**Day 4-5: FastAPI Application**

```python
# src/azure_keyvault_emulator/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

from .database import init_db
from .routers import secrets_router, keys_router, certificates_router
from .middleware.error_handler import error_handler_middleware

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    await init_db()
    yield
    # Shutdown (cleanup if needed)

app = FastAPI(
    title="Azure Key Vault Emulator",
    version="1.0.0",
    lifespan=lifespan
)

# Add middleware
app.middleware("http")(error_handler_middleware)

# Include routers
app.include_router(secrets_router, prefix="/secrets", tags=["secrets"])
app.include_router(keys_router, prefix="/keys", tags=["keys"])
app.include_router(certificates_router, prefix="/certificates", tags=["certificates"])

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "azure_keyvault_emulator.main:app",
        host="0.0.0.0",
        port=4997,
        ssl_keyfile="/certs/emulator.key",
        ssl_certfile="/certs/emulator.crt"
    )
```

### Phase 2: Secrets API (Week 2)

**Implement all secrets endpoints with full tests**

### Phase 3: Keys API (Week 3)

**Implement all keys endpoints including crypto operations**

### Phase 4: Certificates API (Week 4)

**Implement all certificates endpoints including policies**

### Phase 5: Integration (Week 5)

**Full integration testing with Azure SDK clients**

### Phase 6: Release (Week 6)

**Documentation, Docker, CI/CD, release**

---

## Decision Matrix

Use this to choose your path:

| Criteria | Python Client | Full Port | Keep .NET |
|----------|--------------|-----------|-----------|
| **Time Available** | 1-2 weeks | 6-8 weeks | 0 |
| **Python Users?** | Some | Many | None |
| **Budget** | Low | Medium-High | None |
| **Risk Tolerance** | Low | Medium | None |
| **Maintenance** | Low | High | Low |

**Recommendation Flow**:
1. Do you need Python support? 
   - No → Keep .NET
   - Yes → Continue
2. Do you have 6+ weeks and need pure Python?
   - Yes → Full Port
   - No → Python Client
3. Can you run .NET in container?
   - Yes → Python Client
   - No → Full Port

## Success Criteria

### Python Client
- [x] Package published to PyPI
- [x] Works with emulator
- [x] >90% test coverage
- [x] Documentation complete
- [x] <100ms overhead

### Full Port
- [x] 100% REST API compatibility
- [x] All tests passing
- [x] Works with Azure SDKs
- [x] Docker image <500MB
- [x] Documentation complete
- [x] Performance within 2x of .NET

## Getting Help

- Review detailed feasibility: `PYTHON_CONVERSION_FEASIBILITY.md`
- See code examples: `CODE_COMPARISON.md`
- Read recommendations: `CONVERSION_RECOMMENDATION.md`
- Open issues for questions

---

**Next Step**: Choose Path A (Python Client) or Path B (Full Port) and begin Week 1!
