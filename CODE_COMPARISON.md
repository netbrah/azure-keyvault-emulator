# Quick Reference: .NET vs Python Implementation

## Side-by-Side Code Comparison

### Example 1: Setting a Secret

#### Current .NET Implementation

```csharp
// Controller: SecretsController.cs
[HttpPut("{name}")]
[ProducesResponseType<SecretBundle>(StatusCodes.Status200OK)]
public async Task<IActionResult> SetSecret(
    [FromRoute] string name,
    [ApiVersion] string apiVersion,
    [FromBody] SetSecretRequest requestBody)
{
    var secret = await secretService.SetSecretAsync(name, requestBody);
    return Ok(secret);
}

// Service: SecretService.cs
public async Task<SecretBundle> SetSecretAsync(string name, SetSecretRequest secret, bool? managed = null)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(name);
    ArgumentNullException.ThrowIfNull(secret);

    if (await context.Secrets.AnyAsync(e => e.PersistedName == name && e.Deleted))
        throw new ConflictedItemException("Secret", name);

    var version = Guid.NewGuid().Neat();
    var secretUri = httpContextAccessor.BuildIdentifierUri(name, version, "secrets");

    var response = new SecretBundle
    {
        SecretIdentifier = secretUri,
        Value = secret.Value,
        Managed = managed,
        Attributes = secret.SecretAttributes,
        ContentType = secret.ContentType,
        Tags = secret.Tags
    };

    await context.Secrets.SafeAddAsync(name, version, response);
    await context.SaveChangesAsync();

    return response;
}

// Model: SecretBundle.cs
public class SecretBundle
{
    public string SecretIdentifier { get; set; }
    public string Value { get; set; }
    public bool? Managed { get; set; }
    public SecretAttributes Attributes { get; set; }
    public string ContentType { get; set; }
    public Dictionary<string, string> Tags { get; set; }
}
```

#### Python Equivalent (FastAPI)

```python
# Controller: secrets_router.py
from fastapi import APIRouter, Depends, Query
from models import SecretBundle, SetSecretRequest

router = APIRouter(prefix="/secrets", tags=["secrets"])

@router.put("/{name}")
async def set_secret(
    name: str,
    request_body: SetSecretRequest,
    api_version: str = Query(..., alias="api-version"),
    secret_service: SecretService = Depends(get_secret_service)
) -> SecretBundle:
    secret = await secret_service.set_secret(name, request_body)
    return secret

# Service: secret_service.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
import uuid

class SecretService:
    def __init__(self, session: AsyncSession, request: Request):
        self.session = session
        self.request = request
    
    async def set_secret(
        self, 
        name: str, 
        secret: SetSecretRequest, 
        managed: bool | None = None
    ) -> SecretBundle:
        if not name or not name.strip():
            raise ValueError("Name cannot be empty")
        if not secret:
            raise ValueError("Secret cannot be null")
        
        # Check for deleted secret with same name
        result = await self.session.execute(
            select(Secret).where(
                Secret.persisted_name == name,
                Secret.deleted == True
            )
        )
        if result.scalar_one_or_none():
            raise ConflictedItemException("Secret", name)
        
        # Generate version and URI
        version = str(uuid.uuid4()).replace('-', '')
        secret_uri = self._build_identifier_uri(name, version, "secrets")
        
        # Create response
        response = SecretBundle(
            id=secret_uri,
            value=secret.value,
            managed=managed,
            attributes=secret.attributes,
            content_type=secret.content_type,
            tags=secret.tags
        )
        
        # Save to database
        db_secret = Secret(
            persisted_name=name,
            version=version,
            data=response.model_dump_json()
        )
        self.session.add(db_secret)
        await self.session.commit()
        
        return response

# Model: models.py
from pydantic import BaseModel, Field
from typing import Optional

class SecretBundle(BaseModel):
    id: str = Field(..., alias="id")
    value: str
    managed: Optional[bool] = None
    attributes: SecretAttributes
    content_type: Optional[str] = Field(None, alias="contentType")
    tags: Optional[dict[str, str]] = None
    
    class Config:
        populate_by_name = True
```

### Example 2: RSA Key Generation

#### Current .NET Implementation

```csharp
// Factory: RsaKeyFactory.cs
using System.Security.Cryptography;

public static class RsaKeyFactory
{
    public static RSA CreateKey(int keySize)
    {
        var rsa = RSA.Create(keySize);
        return rsa;
    }
    
    public static string ExportPublicKey(RSA rsa)
    {
        var publicKey = rsa.ExportRSAPublicKey();
        return Convert.ToBase64String(publicKey);
    }
    
    public static JsonWebKey ToJsonWebKey(RSA rsa, string keyId)
    {
        var parameters = rsa.ExportParameters(includePrivateParameters: false);
        
        return new JsonWebKey
        {
            Kid = keyId,
            Kty = "RSA",
            N = Base64UrlEncoder.Encode(parameters.Modulus),
            E = Base64UrlEncoder.Encode(parameters.Exponent)
        };
    }
}
```

#### Python Equivalent

```python
# Factory: rsa_key_factory.py
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.backends import default_backend
import base64

class RsaKeyFactory:
    @staticmethod
    def create_key(key_size: int) -> rsa.RSAPrivateKey:
        """Create an RSA private key."""
        private_key = rsa.generate_private_key(
            public_exponent=65537,
            key_size=key_size,
            backend=default_backend()
        )
        return private_key
    
    @staticmethod
    def export_public_key(private_key: rsa.RSAPrivateKey) -> str:
        """Export public key as base64 string."""
        public_key_bytes = private_key.public_key().public_bytes(
            encoding=serialization.Encoding.DER,
            format=serialization.PublicFormat.SubjectPublicKeyInfo
        )
        return base64.b64encode(public_key_bytes).decode('utf-8')
    
    @staticmethod
    def to_json_web_key(
        private_key: rsa.RSAPrivateKey, 
        key_id: str
    ) -> dict:
        """Convert RSA key to JSON Web Key format."""
        public_key = private_key.public_key()
        public_numbers = public_key.public_numbers()
        
        # Convert to base64url encoding
        n = base64.urlsafe_b64encode(
            public_numbers.n.to_bytes(
                (public_numbers.n.bit_length() + 7) // 8, 
                byteorder='big'
            )
        ).decode('utf-8').rstrip('=')
        
        e = base64.urlsafe_b64encode(
            public_numbers.e.to_bytes(
                (public_numbers.e.bit_length() + 7) // 8,
                byteorder='big'
            )
        ).decode('utf-8').rstrip('=')
        
        return {
            "kid": key_id,
            "kty": "RSA",
            "n": n,
            "e": e
        }
```

### Example 3: Database Context

#### Current .NET Implementation (Entity Framework)

```csharp
// Context: VaultContext.cs
using Microsoft.EntityFrameworkCore;

public class VaultContext : DbContext
{
    public DbSet<Secret> Secrets { get; set; }
    public DbSet<Key> Keys { get; set; }
    public DbSet<Certificate> Certificates { get; set; }
    
    public VaultContext(DbContextOptions<VaultContext> options)
        : base(options)
    {
    }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Secret>(entity =>
        {
            entity.HasKey(e => e.Id);
            entity.Property(e => e.PersistedName).IsRequired();
            entity.Property(e => e.Version).IsRequired();
            entity.HasIndex(e => new { e.PersistedName, e.Version }).IsUnique();
        });
    }
}

// Extension: SafeGetAsync
public static async Task<TBundle> SafeGetAsync<TBundle, TAttributes>(
    this DbSet<T> set, 
    string name, 
    string version = "")
    where TBundle : class
    where TAttributes : class
{
    var query = set.Where(e => e.PersistedName == name && !e.Deleted);
    
    if (!string.IsNullOrEmpty(version))
        query = query.Where(e => e.Version == version);
    else
        query = query.OrderByDescending(e => e.Created).Take(1);
    
    var entity = await query.FirstOrDefaultAsync()
        ?? throw new NotFoundException($"Item {name} not found");
    
    return JsonSerializer.Deserialize<TBundle>(entity.Data);
}
```

#### Python Equivalent (SQLAlchemy)

```python
# Models: database.py
from sqlalchemy import Column, String, Boolean, DateTime, Index
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import declarative_base
from datetime import datetime

Base = declarative_base()

class Secret(Base):
    __tablename__ = "secrets"
    
    id = Column(String, primary_key=True)
    persisted_name = Column(String, nullable=False)
    version = Column(String, nullable=False)
    data = Column(String, nullable=False)  # JSON serialized
    deleted = Column(Boolean, default=False)
    created = Column(DateTime, default=datetime.utcnow)
    updated = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)
    
    __table_args__ = (
        Index('ix_name_version', 'persisted_name', 'version', unique=True),
    )

# Extensions: safe_get.py
from sqlalchemy import select, desc
from sqlalchemy.ext.asyncio import AsyncSession
from typing import TypeVar, Type
import json

T = TypeVar('T')

async def safe_get_async(
    session: AsyncSession,
    model: Type,
    bundle_class: Type[T],
    name: str,
    version: str = ""
) -> T:
    """Get a secret/key/certificate safely with version support."""
    query = select(model).where(
        model.persisted_name == name,
        model.deleted == False
    )
    
    if version:
        query = query.where(model.version == version)
    else:
        query = query.order_by(desc(model.created)).limit(1)
    
    result = await session.execute(query)
    entity = result.scalar_one_or_none()
    
    if not entity:
        raise NotFoundException(f"Item {name} not found")
    
    # Deserialize JSON data
    return bundle_class.model_validate_json(entity.data)

# Usage example
secret = await safe_get_async(
    session=session,
    model=Secret,
    bundle_class=SecretBundle,
    name="my-secret",
    version="abc123"
)
```

### Example 4: Middleware

#### Current .NET Implementation

```csharp
// Middleware: KeyVaultErrorMiddleware.cs
public class KeyVaultErrorMiddleware
{
    private readonly RequestDelegate _next;
    
    public KeyVaultErrorMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = 404;
            await WriteErrorResponse(context, "NotFound", ex.Message);
        }
        catch (ConflictedItemException ex)
        {
            context.Response.StatusCode = 409;
            await WriteErrorResponse(context, "Conflict", ex.Message);
        }
    }
    
    private async Task WriteErrorResponse(
        HttpContext context, 
        string code, 
        string message)
    {
        var error = new KeyVaultError
        {
            Error = new ErrorDetail
            {
                Code = code,
                Message = message
            }
        };
        
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(error);
    }
}
```

#### Python Equivalent (FastAPI)

```python
# Middleware: error_middleware.py
from fastapi import Request, status
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware

class KeyVaultErrorMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        try:
            response = await call_next(request)
            return response
        except NotFoundException as exc:
            return JSONResponse(
                status_code=status.HTTP_404_NOT_FOUND,
                content={
                    "error": {
                        "code": "NotFound",
                        "message": str(exc)
                    }
                }
            )
        except ConflictedItemException as exc:
            return JSONResponse(
                status_code=status.HTTP_409_CONFLICT,
                content={
                    "error": {
                        "code": "Conflict",
                        "message": str(exc)
                    }
                }
            )
        except Exception as exc:
            return JSONResponse(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                content={
                    "error": {
                        "code": "InternalError",
                        "message": "An internal error occurred"
                    }
                }
            )

# Application setup
from fastapi import FastAPI

app = FastAPI()
app.add_middleware(KeyVaultErrorMiddleware)
```

### Example 5: Program.cs / Main Application

#### Current .NET Implementation

```csharp
// Program.cs
using AzureKeyVaultEmulator.ApiConfiguration;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddConfiguredAuthentication();
builder.Services.AddControllers();
builder.Services.AddHttpContextAccessor();
builder.Services.AddSwaggerGen();
builder.Services.RegisterCustomServices();
builder.Services.AddVaultPersistenceLayer();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseMiddleware<KeyVaultErrorMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<VaultContext>();
await db.Database.MigrateAsync();

app.Run();
```

#### Python Equivalent

```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Import routers
from routers import secrets_router, keys_router, certificates_router
from middleware import KeyVaultErrorMiddleware
from database import Base, get_session
from auth import configure_auth

# Database setup
DATABASE_URL = "sqlite+aiosqlite:///./emulator.db"
engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: Create tables
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield
    # Shutdown: Clean up
    await engine.dispose()

# Create FastAPI app
app = FastAPI(
    title="Azure Key Vault Emulator",
    version="1.0.0",
    lifespan=lifespan
)

# Add middleware
app.add_middleware(KeyVaultErrorMiddleware)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Configure authentication
configure_auth(app)

# Include routers
app.include_router(secrets_router.router)
app.include_router(keys_router.router)
app.include_router(certificates_router.router)

# Dependency injection for database
async def get_db():
    async with async_session() as session:
        yield session

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=4997,
        ssl_keyfile="/certs/emulator.key",
        ssl_certfile="/certs/emulator.crt",
        reload=True  # Development only
    )
```

## Key Differences Summary

| Aspect | .NET | Python |
|--------|------|--------|
| **Web Framework** | ASP.NET Core | FastAPI |
| **Routing** | Attributes (`[HttpPut]`) | Decorators (`@router.put`) |
| **Dependency Injection** | Built-in DI container | FastAPI Depends() |
| **ORM** | Entity Framework Core | SQLAlchemy |
| **Models/DTOs** | Classes with properties | Pydantic models |
| **Async/Await** | `Task<T>`, `async/await` | `async def`, `await` |
| **Type System** | Compile-time checked | Runtime validation (Pydantic) |
| **Serialization** | System.Text.Json | pydantic JSON |
| **Cryptography** | System.Security.Cryptography | cryptography library |
| **Testing** | xUnit/NUnit | pytest |
| **Package Manager** | NuGet | pip/poetry/hatch |
| **Runtime** | .NET CLR | CPython interpreter |

## Lines of Code Comparison

Based on typical patterns:

| Component | .NET (Current) | Python (Estimated) | Ratio |
|-----------|----------------|-------------------|-------|
| Controllers | ~800 lines | ~600 lines | 0.75x |
| Services | ~1,500 lines | ~1,200 lines | 0.80x |
| Models | ~2,000 lines | ~1,500 lines | 0.75x |
| Database | ~800 lines | ~600 lines | 0.75x |
| Middleware | ~300 lines | ~250 lines | 0.83x |
| Tests | ~4,000 lines | ~3,500 lines | 0.88x |
| **Total** | **~15,000 lines** | **~11,500 lines** | **0.77x** |

**Conclusion**: Python version would be approximately 23% less code due to:
- Less boilerplate
- No explicit type declarations in many places
- More concise syntax
- Simpler dependency injection

## Complexity Comparison

| Task | .NET Complexity | Python Complexity | Winner |
|------|----------------|-------------------|--------|
| Web API Setup | Medium | Easy | 🐍 Python |
| Database ORM | Easy | Easy | 🤝 Tie |
| Type Safety | Easy (compile-time) | Medium (runtime) | 🔵 .NET |
| Async Operations | Easy | Easy | 🤝 Tie |
| Cryptography | Medium | Medium | 🤝 Tie |
| Testing | Easy | Easy | 🤝 Tie |
| Deployment | Medium | Easy | 🐍 Python |
| Performance | Excellent | Good | 🔵 .NET |
| Developer Experience | Good | Excellent | 🐍 Python |

## Conclusion

The code patterns between .NET and Python are remarkably similar, making conversion straightforward. Python would result in:
- ✅ ~23% less code
- ✅ Simpler deployment
- ✅ Better developer experience for Python devs
- ❌ Slightly less type safety
- ❌ Potentially lower performance

Both implementations would follow the same architecture and REST API spec, making them functionally equivalent.
