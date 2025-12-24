# Azure Key Vault Emulator - Python Conversion Feasibility Analysis

## Executive Summary

Converting the Azure Key Vault Emulator from .NET to Python is **technically feasible but represents a significant undertaking**. Based on the current codebase analysis:

- **Effort Estimate**: 4-6 weeks for a single experienced developer
- **Complexity**: Medium-High
- **Risk Level**: Medium
- **Recommendation**: Feasible with careful planning and phased approach

## Current State Analysis

### Codebase Statistics
- **Total C# Files**: 170 files
- **Total Lines of Code**: ~11,254 lines (src) + 4,035 lines (tests) = **15,289 total lines**
- **Main Components**:
  - 8 Controllers (REST API endpoints)
  - 11 Services (business logic)
  - 90+ Shared models and utilities
  - 25 Test files

### Technology Stack (Current .NET)
- **Framework**: ASP.NET Core Web API (.NET 10.0)
- **Database**: Entity Framework Core + SQLite
- **Authentication**: JWT Bearer tokens
- **Cryptography**: System.Security.Cryptography, X509 certificates
- **Dependencies**:
  - Microsoft.EntityFrameworkCore.Sqlite
  - Azure.Security.KeyVault.Certificates
  - Microsoft.IdentityModel.JsonWebTokens
  - Swashbuckle (Swagger/OpenAPI)

### API Coverage
The emulator currently implements the complete Azure Key Vault REST API:
- **Secrets API**: 8 endpoints (set, get, delete, backup, restore, versions, list)
- **Keys API**: 10+ endpoints (create, get, update, delete, encrypt, decrypt, sign, verify, etc.)
- **Certificates API**: 15+ endpoints (create, import, get, update, delete, merge, policies, issuers, contacts)
- **Deleted Items**: Separate controllers for soft-delete/recovery
- **Additional**: RNG endpoint, emulator control endpoints

## Python Technology Stack Recommendations

### 1. Web Framework Options

#### **FastAPI** (Recommended)
- ✅ Modern, high-performance async framework
- ✅ Built-in OpenAPI/Swagger generation
- ✅ Excellent async/await support
- ✅ Type hints and Pydantic validation
- ✅ Similar to ASP.NET Core's attribute routing
- ❌ Newer, smaller ecosystem than Flask

#### Flask (Alternative)
- ✅ Mature, well-documented
- ✅ Large ecosystem
- ❌ Sync-first (async support via extensions)
- ❌ Manual OpenAPI/Swagger setup

#### Django REST Framework (Not Recommended)
- ❌ Too heavyweight for this use case
- ❌ Opinionated ORM conflicts with emulator goals

### 2. Database ORM

#### **SQLAlchemy** (Recommended)
- ✅ Most mature Python ORM
- ✅ SQLite support out-of-the-box
- ✅ Async support (SQLAlchemy 2.0+)
- ✅ Similar to Entity Framework

#### Alternatives: Tortoise ORM, Peewee
- Both viable but smaller communities

### 3. Cryptography

#### **cryptography** library (Recommended)
- ✅ Industry standard for Python
- ✅ Comprehensive X.509 certificate support
- ✅ RSA, EC, AES implementations
- ✅ Used by Azure SDK itself

#### Additional: **PyJWT** for JWT tokens

### 4. Testing

#### **pytest** (Recommended)
- ✅ Industry standard
- ✅ Excellent async support
- ✅ Rich plugin ecosystem

#### Testcontainers Python
- ✅ Port of Java Testcontainers available

## Conversion Breakdown by Component

### 1. Controllers/Routes (Est: 3-5 days)

**Current**: 8 ASP.NET Core controllers
**Target**: FastAPI routers

**Example Conversion:**

```python
# Current C# (SecretsController.cs)
[HttpPut("{name}")]
public async Task<IActionResult> SetSecret(
    [FromRoute] string name,
    [FromBody] SetSecretRequest requestBody)
{
    var secret = await secretService.SetSecretAsync(name, requestBody);
    return Ok(secret);
}

# Python FastAPI equivalent
@router.put("/secrets/{name}")
async def set_secret(
    name: str,
    request_body: SetSecretRequest,
    secret_service: SecretService = Depends(get_secret_service),
    api_version: str = Query(...)
) -> SecretBundle:
    secret = await secret_service.set_secret(name, request_body)
    return secret
```

**Complexity**: Low-Medium
- Route definitions map 1:1
- Async/await supported in both
- Pydantic models similar to C# DTOs

### 2. Services/Business Logic (Est: 5-8 days)

**Current**: 11 service classes
**Target**: Python async classes

**Main Services**:
- SecretService (~150 lines)
- KeyService (~200 lines)
- CertificateService (~438 lines) ⚠️ **Most Complex**
- CertificateBackingService (~236 lines)
- EncryptionService
- TokenService

**Challenges**:
- Certificate operations are complex (PKCS#12, X.509, PEM formats)
- Cryptographic operations must match .NET behavior exactly
- Key generation and management

**Complexity**: Medium-High

### 3. Models/DTOs (Est: 2-3 days)

**Current**: 90+ C# classes in Models directory
**Target**: Pydantic models

**Example Conversion:**

```python
# Current C#
public class SecretBundle
{
    public string SecretIdentifier { get; set; }
    public string Value { get; set; }
    public SecretAttributes Attributes { get; set; }
    public Dictionary<string, string> Tags { get; set; }
}

# Python Pydantic
from pydantic import BaseModel, Field
from typing import Optional

class SecretBundle(BaseModel):
    id: str = Field(..., alias="secretIdentifier")
    value: str
    attributes: SecretAttributes
    tags: Optional[dict[str, str]] = None
    
    class Config:
        populate_by_name = True
```

**Complexity**: Low
- Straightforward 1:1 mapping
- Pydantic handles validation similar to C#

### 4. Persistence Layer (Est: 3-4 days)

**Current**: Entity Framework Core + SQLite
**Target**: SQLAlchemy + SQLite

**Features to Port**:
- VaultContext database context
- Migrations (6 migration files exist)
- Custom query extensions (SafeGetAsync, SafeAddAsync)
- Soft delete support
- Version tracking

**Example Conversion:**

```python
# SQLAlchemy model
from sqlalchemy import Column, String, Boolean, DateTime
from sqlalchemy.ext.asyncio import AsyncSession

class Secret(Base):
    __tablename__ = "secrets"
    
    id = Column(String, primary_key=True)
    persisted_name = Column(String, nullable=False)
    version = Column(String, nullable=False)
    value = Column(String, nullable=False)
    deleted = Column(Boolean, default=False)
    created_at = Column(DateTime)
    updated_at = Column(DateTime)
```

**Complexity**: Medium
- Alembic for migrations (similar to EF migrations)
- Async queries require SQLAlchemy 2.0+

### 5. Middleware (Est: 2-3 days)

**Current**: 4 middleware classes
**Target**: FastAPI middleware or dependencies

**Middleware**:
- KeyVaultErrorMiddleware (exception handling)
- ClientRequestIdMiddleware (request tracking)
- RestoreDoubleSlashRerouteMiddleware (path handling)
- RequestDumpMiddleware (debugging)

**FastAPI Approach**:
```python
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class KeyVaultErrorMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            return await call_next(request)
        except KeyVaultException as e:
            return JSONResponse(
                status_code=e.status_code,
                content={"error": e.to_dict()}
            )
```

**Complexity**: Low-Medium

### 6. Authentication (Est: 2-3 days)

**Current**: JWT Bearer authentication with custom emulated tokens
**Target**: FastAPI OAuth2/JWT

**Features**:
- JWT token generation and validation
- Emulated Azure AD credentials
- Bearer token authentication

**Libraries**:
- PyJWT for token handling
- FastAPI security utilities

**Complexity**: Low-Medium

### 7. Cryptography (Est: 4-6 days) ⚠️ **Highest Risk**

**Current Operations**:
- RSA key generation (2048, 3072, 4096 bit)
- EC key generation (P-256, P-384, P-521)
- AES encryption/decryption
- Key wrapping/unwrapping
- HMAC operations
- X.509 certificate generation
- PKCS#12 handling
- PEM/DER format conversions

**Python Cryptography Library Mapping**:

```python
from cryptography.hazmat.primitives.asymmetric import rsa, ec
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.x509 import CertificateBuilder, Name

# RSA key generation
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)

# Certificate generation
cert = CertificateBuilder() \
    .subject_name(Name(...)) \
    .issuer_name(Name(...)) \
    .public_key(private_key.public_key()) \
    .serial_number(...) \
    .not_valid_before(datetime.utcnow()) \
    .not_valid_after(datetime.utcnow() + timedelta(days=365)) \
    .sign(private_key, hashes.SHA256())
```

**Challenges**:
- Must produce byte-identical outputs for interoperability
- Certificate chain validation
- PKCS#12 format handling

**Complexity**: High

### 8. Docker Configuration (Est: 1 day)

**Current**: Multi-stage .NET Docker build
**Target**: Python Docker image

**Example Dockerfile**:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1
ENV CERT_PATH=/certs/emulator.pfx
ENV CERT_PASSWORD=emulator

EXPOSE 4997

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "4997", \
     "--ssl-keyfile", "/certs/emulator.key", \
     "--ssl-certfile", "/certs/emulator.crt"]
```

**Complexity**: Low

### 9. Testing (Est: 5-7 days)

**Current**: 
- 25 test files
- 4,035 lines of test code
- Integration tests with Azure SDK clients
- TestContainers support

**Target**: pytest + pytest-asyncio

**Example Test Conversion**:

```python
@pytest.mark.asyncio
async def test_set_and_get_secret(secret_client):
    # Arrange
    secret_name = "test-secret"
    secret_value = "test-value"
    
    # Act
    await secret_client.set_secret(secret_name, secret_value)
    secret = await secret_client.get_secret(secret_name)
    
    # Assert
    assert secret.value == secret_value
```

**Complexity**: Medium
- Tests should work with both Python and original .NET emulator
- TestContainers Python module exists

### 10. Documentation (Est: 2-3 days)

**Updates Needed**:
- README.md with Python setup instructions
- Configuration documentation
- API documentation (auto-generated via FastAPI)
- Docker setup guide
- Migration guide from .NET version

**Complexity**: Low

## REST API Conformance

### Official Azure Key Vault REST API

The conversion should maintain 100% compatibility with:

**Official Documentation**:
- Secrets: https://learn.microsoft.com/en-us/rest/api/keyvault/secrets/
- Keys: https://learn.microsoft.com/en-us/rest/api/keyvault/keys/
- Certificates: https://learn.microsoft.com/en-us/rest/api/keyvault/certificates/

**API Versions Supported**:
- Current emulator targets: 7.4, 7.5
- Latest stable: 7.5 (2023-11-01)
- Preview: 2025-07-01 (mentioned in issue)

**Key Endpoints** (must maintain exact compatibility):

#### Secrets (Base: `/secrets`)
- `PUT /secrets/{name}` - Set secret
- `GET /secrets/{name}/{version}` - Get secret version
- `GET /secrets/{name}` - Get latest secret
- `DELETE /secrets/{name}` - Delete secret
- `GET /secrets` - List secrets
- `POST /secrets/{name}/backup` - Backup secret
- `POST /secrets/restore` - Restore secret
- `GET /secrets/{name}/versions` - List versions

#### Keys (Base: `/keys`)
- `POST /keys/{name}/create` - Create key
- `POST /keys/{name}/import` - Import key
- `GET /keys/{name}/{version}` - Get key version
- `GET /keys/{name}` - Get latest key
- `DELETE /keys/{name}` - Delete key
- `GET /keys` - List keys
- `POST /keys/{name}/encrypt` - Encrypt
- `POST /keys/{name}/decrypt` - Decrypt
- `POST /keys/{name}/sign` - Sign
- `POST /keys/{name}/verify` - Verify
- `POST /keys/{name}/wrapkey` - Wrap key
- `POST /keys/{name}/unwrapkey` - Unwrap key

#### Certificates (Base: `/certificates`)
- `POST /certificates/{name}/create` - Create certificate
- `POST /certificates/{name}/import` - Import certificate
- `GET /certificates/{name}/{version}` - Get certificate
- `GET /certificates/{name}` - Get latest certificate
- `DELETE /certificates/{name}` - Delete certificate
- `GET /certificates` - List certificates
- `PATCH /certificates/{name}/pending` - Update operation
- `GET /certificates/{name}/policy` - Get policy
- `PATCH /certificates/{name}/policy` - Update policy
- `POST /certificates/{name}/merge` - Merge certificate

**Response Format Consistency**:
All responses must match Azure's exact JSON schema, including:
- Field names (camelCase)
- Date formats (Unix timestamps or ISO 8601)
- Error response structure
- Pagination format (nextLink, value array)

## Development Approach

### Phase 1: Foundation (Week 1)
1. Set up FastAPI project structure
2. Implement basic routing and middleware
3. Set up SQLAlchemy models and migrations
4. Create Pydantic models for all DTOs
5. Configure Docker environment

### Phase 2: Core Secrets API (Week 2)
1. Implement SecretService
2. Implement SecretsController routes
3. Add basic authentication
4. Write tests for secrets operations
5. Verify Azure SDK compatibility

### Phase 3: Keys API (Week 2-3)
1. Implement cryptography utilities (RSA, EC)
2. Implement KeyService
3. Implement KeysController routes
4. Add encryption/decryption operations
5. Write tests for key operations

### Phase 4: Certificates API (Week 3-4)
1. Implement X.509 certificate utilities
2. Implement CertificateService
3. Implement CertificatesController routes
4. Handle certificate policies and issuers
5. Write tests for certificate operations

### Phase 5: Integration & Testing (Week 4-5)
1. Full integration testing with Azure SDK
2. TestContainers Python module
3. Performance testing
4. Docker image optimization
5. Documentation

### Phase 6: Cleanup & Release (Week 5-6)
1. Code review and refactoring
2. Final documentation
3. CI/CD pipeline setup
4. Release preparation
5. Migration guide

## Advantages of Python Version

### Benefits
1. **Broader Accessibility**: Python is more widely known than C#
2. **Simpler Deployment**: No .NET runtime required
3. **Faster Startup**: Python typically has faster cold start times
4. **Cross-Platform**: Even easier cross-platform deployment
5. **Community**: Larger data science/ML community could benefit
6. **Container Size**: Potentially smaller Docker images
7. **Integration**: Easier integration with Python-based tooling

### Considerations
1. **Performance**: .NET is generally faster for CPU-intensive operations
2. **Type Safety**: Python's type hints are not enforced at runtime
3. **Async Maturity**: Python async ecosystem is less mature than .NET's
4. **Maintenance**: Now maintaining two implementations (or deprecating .NET)

## Risk Assessment

### High Risks
1. **Cryptographic Compatibility**: Ensuring byte-for-byte compatibility with Azure Key Vault
   - **Mitigation**: Extensive testing with Azure SDK clients
   
2. **Certificate Handling**: Complex PKCS#12 and X.509 operations
   - **Mitigation**: Use well-tested cryptography library, comprehensive test suite

3. **Azure SDK Compatibility**: Python Azure SDK must work seamlessly
   - **Mitigation**: Integration tests with real Azure SDK clients

### Medium Risks
1. **Performance**: Python may be slower for crypto operations
   - **Mitigation**: Use native libraries, benchmark early

2. **Async Consistency**: SQLAlchemy async support is newer
   - **Mitigation**: Use SQLAlchemy 2.0+, thorough testing

### Low Risks
1. **REST API Structure**: Well-defined, straightforward to port
2. **Docker Deployment**: Standard Python containerization
3. **Testing**: pytest ecosystem is mature

## Effort Estimation

| Component | Days | Risk |
|-----------|------|------|
| Controllers/Routes | 3-5 | Low |
| Services | 5-8 | Medium |
| Models | 2-3 | Low |
| Persistence | 3-4 | Medium |
| Middleware | 2-3 | Low |
| Authentication | 2-3 | Low |
| Cryptography | 4-6 | **High** |
| Docker | 1 | Low |
| Testing | 5-7 | Medium |
| Documentation | 2-3 | Low |
| **Total** | **29-42 days** | **Medium** |

**Realistic Timeline**: 6-8 weeks for a single experienced developer

## Recommendation

### Should You Convert?

**YES, if:**
- ✅ You want broader community adoption (Python users)
- ✅ You're willing to maintain two versions or deprecate .NET
- ✅ You have 6-8 weeks of dedicated development time
- ✅ You need lighter deployment footprint
- ✅ Your primary user base prefers Python

**NO, if:**
- ❌ Current .NET version meets all needs
- ❌ Limited development resources
- ❌ Performance is critical (crypto operations)
- ❌ You don't want to maintain multiple implementations

### Hybrid Approach (Recommended)

Consider creating a Python client/wrapper that communicates with the existing .NET emulator via REST API:

**Advantages**:
- ✅ Reuse existing tested implementation
- ✅ Provide Python-friendly interface
- ✅ Much faster to implement (1-2 weeks)
- ✅ No risk of behavioral differences
- ✅ Single source of truth

**Python Client Example**:
```python
# azure_keyvault_emulator_client.py
import httpx
from typing import Optional

class EmulatorClient:
    def __init__(self, base_url: str = "https://localhost:4997"):
        self.client = httpx.AsyncClient(
            base_url=base_url,
            verify=False  # Emulator uses self-signed cert
        )
    
    async def set_secret(self, name: str, value: str) -> dict:
        response = await self.client.put(
            f"/secrets/{name}",
            json={"value": value},
            params={"api-version": "7.5"}
        )
        return response.json()
```

## Technical Specification Checklist

If proceeding with full conversion, ensure:

- [ ] FastAPI 0.104+ (latest stable)
- [ ] Python 3.11+ (for performance)
- [ ] SQLAlchemy 2.0+ (async support)
- [ ] cryptography 41.0+ (latest security fixes)
- [ ] PyJWT 2.8+ (JWT handling)
- [ ] pytest 7.4+ (testing)
- [ ] uvicorn (ASGI server)
- [ ] Docker multi-stage build
- [ ] Black + isort (code formatting)
- [ ] mypy (type checking)
- [ ] OpenAPI 3.0+ schema generation
- [ ] Comprehensive test suite (>80% coverage)

## Azure REST API Research

### Official REST API Documentation

Based on the official Microsoft documentation, the Azure Key Vault REST API consists of:

#### API Version History
- **7.5** (2023-11-01) - Current production
- **7.4** (2023-02-01) - Previous stable
- **2025-07-01** (Preview) - Upcoming features

#### Complete Endpoint List

##### Secrets Operations (7.5)
```
Documented at: https://learn.microsoft.com/en-us/rest/api/keyvault/secrets/
```
- Set Secret: `PUT /secrets/{secret-name}?api-version=7.5`
- Get Secret: `GET /secrets/{secret-name}/{secret-version}?api-version=7.5`
- Update Secret: `PATCH /secrets/{secret-name}/{secret-version}?api-version=7.5`
- Delete Secret: `DELETE /secrets/{secret-name}?api-version=7.5`
- Get Deleted Secret: `GET /deletedsecrets/{secret-name}?api-version=7.5`
- Purge Deleted Secret: `DELETE /deletedsecrets/{secret-name}?api-version=7.5`
- Recover Deleted Secret: `POST /deletedsecrets/{secret-name}/recover?api-version=7.5`
- Backup Secret: `POST /secrets/{secret-name}/backup?api-version=7.5`
- Restore Secret: `POST /secrets/restore?api-version=7.5`
- List Secrets: `GET /secrets?api-version=7.5`
- List Secret Versions: `GET /secrets/{secret-name}/versions?api-version=7.5`
- List Deleted Secrets: `GET /deletedsecrets?api-version=7.5`

##### Keys Operations (7.5)
```
Documented at: https://learn.microsoft.com/en-us/rest/api/keyvault/keys/
```
- Create Key: `POST /keys/{key-name}/create?api-version=7.5`
- Import Key: `POST /keys/{key-name}/import?api-version=7.5`
- Get Key: `GET /keys/{key-name}/{key-version}?api-version=7.5`
- Update Key: `PATCH /keys/{key-name}/{key-version}?api-version=7.5`
- Delete Key: `DELETE /keys/{key-name}?api-version=7.5`
- Get Deleted Key: `GET /deletedkeys/{key-name}?api-version=7.5`
- Purge Deleted Key: `DELETE /deletedkeys/{key-name}?api-version=7.5`
- Recover Deleted Key: `POST /deletedkeys/{key-name}/recover?api-version=7.5`
- List Keys: `GET /keys?api-version=7.5`
- List Key Versions: `GET /keys/{key-name}/versions?api-version=7.5`
- List Deleted Keys: `GET /deletedkeys?api-version=7.5`
- Backup Key: `POST /keys/{key-name}/backup?api-version=7.5`
- Restore Key: `POST /keys/restore?api-version=7.5`
- Encrypt: `POST /keys/{key-name}/{key-version}/encrypt?api-version=7.5`
- Decrypt: `POST /keys/{key-name}/{key-version}/decrypt?api-version=7.5`
- Sign: `POST /keys/{key-name}/{key-version}/sign?api-version=7.5`
- Verify: `POST /keys/{key-name}/{key-version}/verify?api-version=7.5`
- Wrap Key: `POST /keys/{key-name}/{key-version}/wrapkey?api-version=7.5`
- Unwrap Key: `POST /keys/{key-name}/{key-version}/unwrapkey?api-version=7.5`
- Get Random Bytes: `POST /rng?api-version=7.5`
- Release Key: `POST /keys/{key-name}/{key-version}/release?api-version=7.5`
- Rotate Key: `POST /keys/{key-name}/rotate?api-version=7.5`

##### Certificates Operations (7.5)
```
Documented at: https://learn.microsoft.com/en-us/rest/api/keyvault/certificates/
```
- Create Certificate: `POST /certificates/{certificate-name}/create?api-version=7.5`
- Import Certificate: `POST /certificates/{certificate-name}/import?api-version=7.5`
- Get Certificate: `GET /certificates/{certificate-name}/{certificate-version}?api-version=7.5`
- Update Certificate: `PATCH /certificates/{certificate-name}/{certificate-version}?api-version=7.5`
- Delete Certificate: `DELETE /certificates/{certificate-name}?api-version=7.5`
- Get Deleted Certificate: `GET /deletedcertificates/{certificate-name}?api-version=7.5`
- Purge Deleted Certificate: `DELETE /deletedcertificates/{certificate-name}?api-version=7.5`
- Recover Deleted Certificate: `POST /deletedcertificates/{certificate-name}/recover?api-version=7.5`
- List Certificates: `GET /certificates?api-version=7.5`
- List Certificate Versions: `GET /certificates/{certificate-name}/versions?api-version=7.5`
- List Deleted Certificates: `GET /deletedcertificates?api-version=7.5`
- Get Certificate Policy: `GET /certificates/{certificate-name}/policy?api-version=7.5`
- Update Certificate Policy: `PATCH /certificates/{certificate-name}/policy?api-version=7.5`
- Get Certificate Operation: `GET /certificates/{certificate-name}/pending?api-version=7.5`
- Update Certificate Operation: `PATCH /certificates/{certificate-name}/pending?api-version=7.5`
- Delete Certificate Operation: `DELETE /certificates/{certificate-name}/pending?api-version=7.5`
- Merge Certificate: `POST /certificates/{certificate-name}/pending/merge?api-version=7.5`
- Get Certificate Issuer: `GET /certificates/issuers/{issuer-name}?api-version=7.5`
- Set Certificate Issuer: `PUT /certificates/issuers/{issuer-name}?api-version=7.5`
- Update Certificate Issuer: `PATCH /certificates/issuers/{issuer-name}?api-version=7.5`
- Delete Certificate Issuer: `DELETE /certificates/issuers/{issuer-name}?api-version=7.5`
- List Certificate Issuers: `GET /certificates/issuers?api-version=7.5`
- Get Certificate Contacts: `GET /certificates/contacts?api-version=7.5`
- Set Certificate Contacts: `PUT /certificates/contacts?api-version=7.5`
- Delete Certificate Contacts: `DELETE /certificates/contacts?api-version=7.5`
- Backup Certificate: `POST /certificates/{certificate-name}/backup?api-version=7.5`
- Restore Certificate: `POST /certificates/restore?api-version=7.5`

#### Response Format Standards

All endpoints return JSON with specific structure:

```json
{
  "value": [...],           // For list operations
  "nextLink": "...",        // For pagination
  "id": "https://...",      // Resource identifier
  "attributes": {           // Common attributes
    "enabled": true,
    "created": 1609459200,  // Unix timestamp
    "updated": 1609459200,
    "recoveryLevel": "Recoverable+Purgeable"
  },
  "tags": {...}            // User-defined tags
}
```

#### Error Response Format

```json
{
  "error": {
    "code": "BadParameter",
    "message": "The request parameter is invalid.",
    "innererror": {
      "code": "InvalidParameterValue"
    }
  }
}
```

#### Authentication
- **Required**: Bearer token in Authorization header
- **Format**: `Authorization: Bearer <token>`
- **Emulator**: Must accept any token format for development use

### Current Emulator Coverage

Based on code analysis, the current .NET emulator implements:

✅ **Fully Implemented**:
- All Secrets operations
- All Keys operations including crypto (encrypt, decrypt, sign, verify, wrap, unwrap)
- All Certificates operations including policies, issuers, contacts
- Soft delete and recovery for all resource types
- Backup and restore
- Pagination with skipToken
- JWT authentication

✅ **Additional Features**:
- SQLite persistence
- Docker support
- TestContainers module
- .NET Aspire integration
- Swagger/OpenAPI documentation

## Conclusion

Converting the Azure Key Vault Emulator to Python is **feasible and valuable** for expanding the user base. The estimated effort is **6-8 weeks** for a complete, tested implementation.

**Key Success Factors**:
1. Comprehensive testing with real Azure SDK clients
2. Strict adherence to REST API specifications
3. Careful cryptographic implementation
4. Maintaining feature parity with .NET version

**Recommended Next Steps**:
1. Validate stakeholder interest in Python version
2. Decide on maintenance strategy (two versions vs. deprecation)
3. Set up Python project structure
4. Begin with Phase 1 (Foundation) to validate approach
5. Implement Secrets API first as proof of concept

This conversion would make the emulator accessible to a much broader developer community while maintaining full compatibility with Azure's REST API specifications.
