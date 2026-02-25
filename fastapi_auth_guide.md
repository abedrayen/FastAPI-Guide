# Building a Modular, Scalable Authentication System in FastAPI

A step-by-step guide for intermediate backend developers. This document covers JWT-based auth, RBAC, clean architecture, and production-ready patterns.

---

## How to Use This Guide

- **Who it’s for:** You are comfortable with Python, basic FastAPI (routes, request/response), and the idea of REST APIs. You may not have built a full auth system before.
- **How to read it:** Go section by section. Each section has a **What you’ll learn** and a **Key idea** so you know the goal before the code. Code blocks are followed by **Why this works** or **Step by step** so you see the reasoning.
- **How to practice:** Build the project folder and files as you go, or read the whole guide first and then implement. Run and test after each major section (e.g. after Section 4, after Section 10).

---

## Table of Contents

0. [Stack & design choices](#stack--design-choices-why-this-combo-is-solid)  
1. [Architecture Overview](#1-architecture-overview)
2. [Project Folder Structure](#2-project-folder-structure)
3. [Environment Configuration](#3-environment-configuration)
4. [Database Layer](#4-database-layer)
5. [Pydantic Schemas](#5-pydantic-schemas)
6. [Security Utilities](#6-security-utilities)
7. [Exception Handling](#7-exception-handling)
8. [Authentication Service](#8-authentication-service)
9. [Dependency Injection](#9-dependency-injection)
10. [API Routes](#10-api-routes)
11. [Role-Based Authorization (RBAC)](#11-role-based-authorization-rbac)
12. [Application Entry Point](#12-application-entry-point)
13. [Testing the API with Postman](#13-testing-the-api-with-postman)
14. [Testing Strategy](#14-testing-strategy)
15. [Extending the System](#15-extending-the-system)
16. [Production Deployment & Security](#16-production-deployment--security)

---

## Stack & design choices (why this combo is solid)

This guide uses a **single, consistent stack** that is production-proven and easy to maintain. No optional alternatives are mixed in.

| Layer | Choice | Why it’s solid |
|-------|--------|-----------------|
| **API** | FastAPI | Async-capable, automatic OpenAPI, built-in validation and dependency injection. |
| **Config** | Pydantic Settings + `.env` | One source of truth, validated at startup, type-safe. No scattered `os.getenv`. |
| **Validation / schemas** | Pydantic | Same as FastAPI’s native model; request/response validation and serialization in one place. |
| **Database** | PostgreSQL + SQLAlchemy (sync) | PostgreSQL: robust, ACID, good for auth and tokens. SQLAlchemy: standard Python ORM; sync keeps the guide simple; async is possible later via `async_database_url`. |
| **Driver** | psycopg2-binary | Official PostgreSQL adapter; binary wheel avoids compile issues. |
| **Passwords** | passlib + bcrypt | Industry standard; bcrypt is slow by design (good for passwords); passlib gives a stable API and rounds control. |
| **Tokens** | python-jose (JWT) | Implements JWT (RFC 7519); supports HS256 and RS256; stable and widely used. |
| **Algorithm** | HS256 | Symmetric signing; fine for one server or a small set of services sharing one secret. Use RS256 if you need asymmetric (e.g. many verifiers, one signer). |
| **Auth flow** | Access + refresh tokens | Short-lived access tokens limit exposure; refresh tokens allow re-issue without re-login; refresh stored in DB for revocation. |
| **Structure** | Routers → services → repositories | Clear separation: HTTP in routers, business logic in services, data access in repositories. Easy to test and extend. |

**What we don’t use here (and why we keep it simple):** No extra DI framework (FastAPI’s `Depends` is enough), no cache layer in the baseline (add later if needed), no async DB in the main guide (optional via `async_database_url`). This keeps the guide **one clear path** and avoids “maybe use X or Y” so you get a single, solid combo you can ship and extend.

---

## 1. Architecture Overview

**What you’ll learn:** How the pieces of an auth system fit together and how a single request flows from the client to the database and back.

**Key idea:** We split the app into clear layers. Each layer has one job: routers handle HTTP, services handle business logic, repositories handle data. That way you can change one part (e.g. database or auth method) without rewriting the rest.

### High-Level Architecture

The diagram below shows who talks to whom. The client only talks to the API; the API uses services; services use repositories and security utilities; repositories talk to the database.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT (Frontend / API Consumer)                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           FastAPI Application (main.py)                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Routers   │  │ Dependencies│  │  Exception  │  │  Middleware (CORS,   │  │
│  │  (auth,     │  │  (get_db,   │  │   Handlers  │  │  logging, etc.)     │  │
│  │   users)    │  │  get_current│  │             │  │                     │  │
│  │             │  │   _user)    │  │             │  │                     │  │
│  └──────┬──────┘  └──────┬──────┘  └─────────────┘  └─────────────────────┘  │
│         │                 │                                                      │
└─────────┼─────────────────┼─────────────────────────────────────────────────────┘
          │                 │
          ▼                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SERVICE LAYER                                       │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐   │
│  │  AuthService        │  │  UserService        │  │  TokenService       │   │
│  │  (login, register,  │  │  (CRUD, profile)    │  │  (create, refresh,  │   │
│  │   refresh)          │  │                     │  │   revoke)            │   │
│  └──────────┬──────────┘  └──────────┬──────────┘  └──────────┬──────────┘   │
│             │                         │                         │              │
└─────────────┼─────────────────────────┼─────────────────────────┼──────────────┘
              │                         │                         │
              ▼                         ▼                         ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  REPOSITORY / DATA LAYER          │  SECURITY UTILITIES                      │
│  ┌─────────────┐ ┌─────────────┐  │  ┌─────────────┐ ┌─────────────────────┐ │
│  │ UserRepo    │ │ TokenRepo   │  │  │ JWT (create,│ │ Password (hash,     │ │
│  │ (SQLAlchemy)│ │ (blacklist) │  │  │ verify)     │ │ verify with passlib)│ │
│  └──────┬──────┘ └──────┬──────┘  │  └─────────────┘ └─────────────────────┘ │
└─────────┼───────────────┼─────────┴──────────────────────────────────────────┘
          │               │
          ▼               ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DATABASE (PostgreSQL)                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Layer | Responsibility in one sentence |
|-------|----------------------------------|
| **Routers** | Receive HTTP request, validate body (via Pydantic), call one service method, return response. |
| **Dependencies** | Provide the DB session and the “current user” to routes that need them. |
| **Services** | Contain the “what to do”: e.g. “log in = find user, check password, create tokens”. |
| **Repositories** | Contain the “how to get/save data”: e.g. “get user by email” without exposing SQL. |
| **Security** | Hash passwords and create/verify JWTs in one place so every flow uses the same rules. |

### Request Flow (Login Example)

This sequence shows what happens when a user sends `POST /auth/login` with email and password. Follow the arrows: router → service → database and security, then the answer goes back.

```
  User                Router              AuthService           Security           DB
   │                     │                      │                    │              │
   │  POST /auth/login   │                      │                    │              │
   │────────────────────>│                      │                    │              │
   │                     │  login(credentials)  │                    │              │
   │                     │─────────────────────>│                    │              │
   │                     │                      │  get_user_by_email │              │
   │                     │                      │─────────────────────────────────>│
   │                     │                      │<─────────────────────────────────│
   │                     │                      │  verify_password   │              │
   │                     │                      │───────────────────>│              │
   │                     │                      │<───────────────────│              │
   │                     │                      │  create_tokens     │              │
   │                     │                      │───────────────────>│              │
   │                     │<─────────────────────│                    │              │
   │  { access, refresh }│                      │                    │              │
   │<────────────────────│                      │                    │              │
```

**Takeaway:** The router never talks to the database or to JWT directly. It only calls `AuthService.login()`. That makes the API easy to test and easy to extend (e.g. add “login with Google” in the same service without changing the route).

---

## 2. Project Folder Structure

**What you’ll learn:** Where to put each kind of file so the project stays clear as it grows.

**Key idea:** Group by “kind” (models, schemas, services) and keep shared, cross-cutting code in one place (`core/`, `db/`). That way, when you add a new feature (e.g. “posts”), you add a model, schema, repository, service, and router without touching auth.

```
project_root/
├── app/
│   ├── __init__.py
│   ├── main.py                    # FastAPI app, lifespan, router inclusion
│   ├── config.py                  # Pydantic Settings from .env
│   │
│   ├── core/                      # Shared, cross-cutting concerns
│   │   ├── __init__.py
│   │   ├── security.py            # JWT + password hashing
│   │   ├── exceptions.py          # Custom exceptions + handlers
│   │   └── dependencies.py        # get_db, get_current_user, RBAC
│   │
│   ├── db/                        # Database setup and session
│   │   ├── __init__.py
│   │   ├── base.py                # Declarative base, engine, SessionLocal
│   │   └── init_db.py             # Create tables, optional seed
│   │
│   ├── models/                    # SQLAlchemy ORM models (shared)
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── token.py               # Refresh token / blacklist if needed
│   │
│   ├── schemas/                   # Pydantic request/response schemas
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── token.py
│   │   └── common.py              # Pagination, message responses
│   │
│   ├── repositories/              # Data access layer
│   │   ├── __init__.py
│   │   ├── base.py                # Generic CRUD base
│   │   ├── user_repository.py
│   │   └── token_repository.py
│   │
│   ├── services/                  # Business logic
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── user_service.py
│   │   └── token_service.py
│   │
│   └── api/
│       ├── __init__.py
│       ├── v1/
│       │   ├── __init__.py
│       │   ├── router.py          # Aggregates all v1 routes
│       │   ├── auth.py            # /auth/login, register, refresh, logout
│       │   └── users.py           # /users/me, admin endpoints
│       └── deps.py                # API-level deps (optional re-export)
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                # Pytest fixtures (client, db, test user)
│   ├── test_auth_api.py
│   └── test_dependencies.py
│
├── .env.example
├── .env                            # Not committed; from .env.example
├── requirements.txt                # fastapi, uvicorn, sqlalchemy, psycopg2-binary, python-jose[cryptography], passlib[bcrypt], pydantic-settings
└── README.md
```

**Why this layout:**  
- **core/** holds security and dependencies so every feature uses the same rules (one place to fix a bug or change JWT expiry).  
- **repositories/** hide the database; later you can add caching or switch to another store without changing services.  
- **api/v1/** keeps versioning clear: when you need breaking changes, you add v2 and keep v1 working.

---

## 3. Environment Configuration

**What you’ll learn:** How to load configuration (database URL, secret key, timeouts) from the environment so you don’t hardcode secrets and can use different values in dev vs production.

**Key idea:** All config lives in one Pydantic `Settings` class that reads from `.env`. The app fails fast at startup if something required (like `SECRET_KEY`) is missing, and you get type-safe access everywhere (e.g. `settings.database_url`).

### `.env.example`

```env
# Application
APP_ENV=development
DEBUG=true
API_V1_PREFIX=/api/v1

# Database (PostgreSQL)
DATABASE_URL=postgresql://user:password@localhost:5432/appdb

# JWT
SECRET_KEY=your-super-secret-key-change-in-production-min-32-chars
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# Security
BCRYPT_ROUNDS=12
ALLOWED_ORIGINS=http://localhost:3000,http://127.0.0.1:8000
```

### `app/config.py`

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # App
    app_env: str = "development"
    debug: bool = False
    api_v1_prefix: str = "/api/v1"

    # Database (PostgreSQL)
    database_url: str = "postgresql://user:password@localhost:5432/appdb"

    # JWT
    secret_key: str
    algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

    # Security
    bcrypt_rounds: int = 12
    allowed_origins: str = "http://localhost:3000"

    @property
    def async_database_url(self) -> str:
        """For async SQLAlchemy with PostgreSQL, use the asyncpg driver."""
        return self.database_url.replace("postgresql://", "postgresql+asyncpg://", 1)


settings = Settings()
```

**Step by step:** (1) One place for all config, validation at startup, and type safety. (2) `extra="ignore"` means extra keys in `.env` are ignored when `.env` has keys you don’t define. (3) Use a strong `SECRET_KEY` in production (e.g. from a secret manager).

---

## 4. Database Layer

### `app/db/base.py`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, DeclarativeBase
from app.config import settings

# PostgreSQL: install psycopg2-binary (or psycopg2) for the default driver
engine = create_engine(
    settings.database_url,
    pool_pre_ping=True,
    pool_size=5,
    max_overflow=10,
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


class Base(DeclarativeBase):
    pass


def get_db():
    """Dependency: yield a DB session and close it after request."""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**Why this works:**  
- `get_db` is a **generator** (it uses `yield`). FastAPI runs the code before `yield` when the request starts and the code after `yield` (in `finally`) when the request ends, so the session is always closed even if an error occurs.  
- `pool_pre_ping=True` makes SQLAlchemy check that a connection is still alive before using it, which avoids errors when PostgreSQL closes idle connections.  
- `pool_size` and `max_overflow` limit how many connections are open at once so you don’t overload the database.

### `app/models/user.py`

This is the **User** table: one row per user. We store the **hashed** password, not the plain one. The `Role` enum is used later for RBAC (who can do what). The relationship to `RefreshToken` lets us list or revoke a user’s refresh tokens.

```python
from sqlalchemy import String, Boolean, DateTime, Enum as SQLEnum
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime
import enum

from app.db.base import Base


class Role(str, enum.Enum):
    ENGINEER = "engineer"
    SALER = "saler"
    ADMIN = "admin"


class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    full_name: Mapped[str | None] = mapped_column(String(255), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    role: Mapped[Role] = mapped_column(SQLEnum(Role), default=Role.ENGINEER)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    # Optional: for refresh token revocation
    refresh_tokens = relationship("RefreshToken", back_populates="user", cascade="all, delete-orphan")
```

**Field guide:**  
- `hashed_password`: we never store the raw password; only the result of bcrypt hashing.  
- `is_active`: lets you “disable” a user without deleting the row.  
- `role`: used by dependencies to allow or deny access (e.g. only `ADMIN` can hit certain endpoints; `ENGINEER`, `SALER`, `ADMIN` are the three roles).  
- `refresh_tokens`: SQLAlchemy relationship; “when this user is deleted, delete their refresh tokens too” (`cascade`).

### `app/models/token.py`

```python
from sqlalchemy import String, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column, relationship
from datetime import datetime

from app.db.base import Base


class RefreshToken(Base):
    """Store each issued refresh token so we can revoke it on logout or check it on refresh."""
    __tablename__ = "refresh_tokens"

    id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    user_id: Mapped[int] = mapped_column(ForeignKey("users.id"), nullable=False)
    token_jti: Mapped[str] = mapped_column(String(36), unique=True, nullable=False)  # JWT ID
    expires_at: Mapped[datetime] = mapped_column(DateTime, nullable=False)
    created_at: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    user = relationship("User", back_populates="refresh_tokens")
```

**Why store refresh tokens:** So we can **revoke** them (e.g. on logout or “log out all devices”). We store the token’s unique id (`jti`) and expiry; when the user sends a refresh token, we can check it’s not revoked before issuing new tokens.

### `app/db/init_db.py`

```python
from app.db.base import Base, engine
from app.models.user import User
from app.models.token import RefreshToken


def init_db():
    Base.metadata.create_all(bind=engine)
```

**What it does:** Creates all tables that exist on `Base` (and its registered models) in the database. Call this once at app startup (we do it in `main.py`’s lifespan). In production you’d typically use migrations (e.g. Alembic) instead of `create_all`.

---

## 5. Pydantic Schemas

**What you’ll learn:** How to describe the shape of request bodies and responses with Pydantic so FastAPI validates input and serializes output automatically.

**Key idea:** A **schema** is a class that defines fields and types. Use different schemas for different operations: e.g. `UserCreate` (includes password) for registration, `UserResponse` (no password) for “get me”. That way the API never accidentally returns a password.

### `app/schemas/common.py`

```python
from pydantic import BaseModel


class MessageResponse(BaseModel):
    """Use for endpoints that return a simple message (e.g. logout, delete)."""
    message: str
```

### `app/schemas/user.py`

```python
from datetime import datetime
from pydantic import BaseModel, EmailStr
from app.models.user import Role


class UserBase(BaseModel):
    email: EmailStr
    full_name: str | None = None


class UserCreate(UserBase):
    password: str
    role: Role  # User chooses: engineer, saler, or admin


class UserUpdate(BaseModel):
    full_name: str | None = None
    is_active: bool | None = None


class UserInDB(UserBase):
    id: int
    is_active: bool
    role: Role
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}


class UserResponse(UserInDB):
    """What we return to the client (exclude sensitive fields if needed)."""
    pass
```

### `app/schemas/token.py`

```python
from pydantic import BaseModel


class TokenPair(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int  # seconds


class TokenPayload(BaseModel):
    sub: str  # user id
    email: str
    role: str
    exp: int
    iat: int
    type: str  # "access" | "refresh"
    jti: str | None = None


class LoginRequest(BaseModel):
    email: str
    password: str


class RefreshRequest(BaseModel):
    refresh_token: str
```

**Schema roles in short:**  
- **UserCreate** is for registration: it includes `password` and `role` so the user chooses their role (engineer, saler, or admin) when signing up; we never return this schema.  
- **UserResponse** / **UserInDB** are for responses: they have no `password` field, so it can’t leak.  
- **TokenPair** is what we return after login/register/refresh. **TokenPayload** describes what we put inside the JWT (for documentation and internal checks).

---

## 6. Security Utilities

**What you’ll learn:** How we hash passwords (so we never store plain text) and how we create and verify JWTs (so the client can prove who they are on each request).

**Key idea:** All password and token logic lives in one module (`core/security.py`). We use **passlib + bcrypt** for passwords and **python-jose** for JWT—a solid, widely used combo. Every part of the app that needs to hash, verify, or issue tokens uses these functions so there is a single place to change algorithm or expiry.

**Concepts in one sentence:**  
- **Hashing (passwords):** We turn a password into a fixed string with bcrypt. We can check “does this password match this hash?” but we cannot get the password back from the hash.  
- **JWT (tokens):** A signed string that contains claims (e.g. user id, role, expiry). The server signs it with a secret; later it can verify the signature and read the claims without hitting the database.

### `app/core/security.py`

```python
from datetime import datetime, timedelta, timezone
from uuid import uuid4
from passlib.context import CryptContext
from jose import JWTError, jwt
from app.config import settings

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto", bcrypt__rounds=settings.bcrypt_rounds)


# ---- Passwords ----

def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)


# ---- JWT (exp/iat are Unix timestamps per RFC 7519) ----

def _create_token(
    subject: str,
    email: str,
    role: str,
    expires_delta: timedelta,
    token_type: str,
    jti: str | None = None,
) -> tuple[str, int]:
    now = datetime.now(timezone.utc)
    expire = now + expires_delta
    payload = {
        "sub": subject,
        "email": email,
        "role": role,
        "exp": int(expire.timestamp()),
        "iat": int(now.timestamp()),
        "type": token_type,
        "jti": jti or str(uuid4()),
    }
    encoded = jwt.encode(
        payload,
        settings.secret_key,
        algorithm=settings.algorithm,
    )
    return encoded, int(expires_delta.total_seconds())


def create_access_token(subject: str, email: str, role: str) -> tuple[str, int]:
    delta = timedelta(minutes=settings.access_token_expire_minutes)
    return _create_token(subject, email, role, delta, "access")


def create_refresh_token(subject: str, email: str, role: str) -> tuple[str, int, str]:
    delta = timedelta(days=settings.refresh_token_expire_days)
    jti = str(uuid4())
    token, _ = _create_token(subject, email, role, delta, "refresh", jti=jti)
    return token, int(delta.total_seconds()), jti


def decode_token(token: str) -> dict | None:
    try:
        payload = jwt.decode(
            token,
            settings.secret_key,
            algorithms=[settings.algorithm],
        )
        return payload
    except JWTError:
        return None
```

**Step by step:**  
- **Passwords:** `hash_password` is used when registering or changing password; `verify_password` is used at login. Bcrypt is slow on purpose to make brute-force attacks harder.  
- **Access token:** Short-lived (e.g. 30 minutes). The client sends it in the `Authorization: Bearer <token>` header on every protected request.  
- **Refresh token:** Long-lived (e.g. 7 days). The client uses it only to get a new access (and optionally refresh) token when the access token expires. We give it a unique `jti` so we can revoke it in the database.  
- **decode_token:** Used by dependencies to turn the Bearer token back into payload; returns `None` if the token is invalid or expired so we can return 401.

**Best practice:** We use `datetime.now(timezone.utc)` and store `exp`/`iat` as Unix timestamps (integers) so the JWT is RFC 7519–compliant and timezone-safe.

**Watch out:** Always use the **access** token in the `Authorization` header for protected routes. If you accidentally accept a refresh token there, an attacker who steals it could get new tokens for a long time. Our `get_current_user` checks `payload.get("type") == "access"` for this reason.

---

## 7. Exception Handling

**What you’ll learn:** How to define your own exceptions and turn them into consistent JSON error responses so the client always gets the same shape for errors.

**Key idea:** Instead of raising generic `HTTPException` everywhere, we raise custom exceptions (e.g. `UnauthorizedError`, `NotFoundError`). One handler per exception type converts them to the right status code and a `{"detail": "message"}` body. That keeps route code clean and ensures every 401 looks the same.

### `app/core/exceptions.py`

```python
from fastapi import Request, status
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

# Custom exceptions
class AppException(Exception):
    """Base for app-specific exceptions."""
    def __init__(self, message: str, status_code: int = status.HTTP_400_BAD_REQUEST):
        self.message = message
        self.status_code = status_code


class NotFoundError(AppException):
    def __init__(self, message: str = "Resource not found"):
        super().__init__(message, status.HTTP_404_NOT_FOUND)


class UnauthorizedError(AppException):
    def __init__(self, message: str = "Not authenticated"):
        super().__init__(message, status.HTTP_401_UNAUTHORIZED)


class ForbiddenError(AppException):
    def __init__(self, message: str = "Not allowed"):
        super().__init__(message, status.HTTP_403_FORBIDDEN)


class ConflictError(AppException):
    def __init__(self, message: str = "Conflict"):
        super().__init__(message, status.HTTP_409_CONFLICT)


# Handlers
async def app_exception_handler(request: Request, exc: AppException) -> JSONResponse:
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.message},
    )


async def validation_exception_handler(request: Request, exc: RequestValidationError) -> JSONResponse:
    return JSONResponse(
        status_code=status.HTTP_422_UNPROCESSABLE_ENTITY,
        content={"detail": exc.errors()},
    )
```

**When to use which:**  
- **401 Unauthorized:** “We don’t know who you are” (no or invalid token, wrong password).  
- **403 Forbidden:** “We know who you are but you’re not allowed to do this” (e.g. wrong role).  
- **404 Not Found:** Resource doesn’t exist.  
- **409 Conflict:** e.g. “Email already registered.”  
Validation errors (wrong body shape) are handled by `validation_exception_handler` and return 422 with a list of field errors.

---

## 8. Authentication Service

**What you’ll learn:** How to implement “register”, “login”, and “refresh” in one place by combining repositories and security utilities. The router will only call this service.

**Key idea:** The **auth service** doesn’t know about HTTP. It receives plain data (email, password, or refresh token), uses repositories to load/save data and security to hash/verify/issue tokens, and returns a `TokenPair` or raises an exception. That way you can reuse the same logic from different routes (e.g. web login vs OAuth callback) or tests.

### Repositories: what they are

A **repository** is a class that talks to the database for one “entity” (e.g. User). It has methods like `get(id)`, `get_by_email(email)`, `create(...)`. The service calls the repository instead of writing SQL or using the session directly. That way, if you later add caching or change the database, you only change the repository.

### `app/repositories/base.py`

```python
from typing import Generic, TypeVar, Type
from sqlalchemy.orm import Session
from app.db.base import Base

ModelType = TypeVar("ModelType", bound=Base)


class BaseRepository(Generic[ModelType]):
    def __init__(self, model: Type[ModelType], db: Session):
        self.model = model
        self.db = db

    def get(self, id: int) -> ModelType | None:
        return self.db.get(self.model, id)
```

**Why a base repository:** We use a generic `BaseRepository[ModelType]` so every entity (User, RefreshToken) gets a standard `get(id)` without repeating code. Each specific repository (e.g. `UserRepository`) adds its own methods like `get_by_email` or `create`.

### `app/repositories/user_repository.py`

```python
from app.models.user import User, Role
from app.repositories.base import BaseRepository


class UserRepository(BaseRepository[User]):
    def __init__(self, db):
        super().__init__(User, db)

    def get_by_email(self, email: str) -> User | None:
        return self.db.query(User).filter(User.email == email).first()

    def create(self, email: str, hashed_password: str, role: Role, full_name: str | None = None) -> User:
        user = User(email=email, hashed_password=hashed_password, full_name=full_name, role=role)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user
```

**What the auth service needs:** “Find user by email” (login, register check) and “create user” (register). We don’t put password hashing here—the service hashes the password and passes the hash to `create`.

### `app/repositories/token_repository.py`

```python
from datetime import datetime
from app.models.token import RefreshToken
from app.repositories.base import BaseRepository


class TokenRepository(BaseRepository[RefreshToken]):
    def __init__(self, db):
        super().__init__(RefreshToken, db)

    def add_token(self, user_id: int, jti: str, expires_at: datetime) -> RefreshToken:
        token = RefreshToken(user_id=user_id, token_jti=jti, expires_at=expires_at)
        self.db.add(token)
        self.db.commit()
        self.db.refresh(token)
        return token

    def is_revoked(self, jti: str) -> bool:
        return self.db.query(RefreshToken).filter(RefreshToken.token_jti == jti).first() is not None
```

**Usage:** When we issue a refresh token we call `add_token` to store its `jti` and expiry. On refresh we could call `is_revoked(jti)` to reject revoked tokens; for full logout you’d add a way to mark a token as revoked (e.g. a `revoked_at` column) and check it here.

### `app/services/auth_service.py`

The service implements three flows: **register** (create user + issue tokens), **login** (find user, verify password, issue tokens), **refresh** (verify refresh token, load user, issue new token pair). All three use the same `_issue_tokens` helper so token creation is in one place.

```python
from datetime import datetime, timedelta, timezone
from app.config import settings
from app.core.security import (
    hash_password,
    verify_password,
    create_access_token,
    create_refresh_token,
    decode_token,
)
from app.core.exceptions import UnauthorizedError, ConflictError
from app.repositories.user_repository import UserRepository
from app.repositories.token_repository import TokenRepository
from app.schemas.user import UserCreate
from app.schemas.token import TokenPair


class AuthService:
    def __init__(self, user_repo: UserRepository, token_repo: TokenRepository):
        self.user_repo = user_repo
        self.token_repo = token_repo

    def register(self, data: UserCreate) -> TokenPair:
        if self.user_repo.get_by_email(data.email):
            raise ConflictError("Email already registered")
        hashed = hash_password(data.password)
        user = self.user_repo.create(
            email=data.email,
            hashed_password=hashed,
            full_name=data.full_name,
            role=data.role,
        )
        return self._issue_tokens(user)

    def login(self, email: str, password: str) -> TokenPair:
        user = self.user_repo.get_by_email(email)
        if not user or not verify_password(password, user.hashed_password):
            raise UnauthorizedError("Invalid email or password")
        if not user.is_active:
            raise UnauthorizedError("Account is disabled")
        return self._issue_tokens(user)

    def refresh_tokens(self, refresh_token: str) -> TokenPair:
        payload = decode_token(refresh_token)
        if not payload or payload.get("type") != "refresh":
            raise UnauthorizedError("Invalid refresh token")
        # Optional: check token_repo.is_revoked(payload.get("jti"))
        user = self.user_repo.get(int(payload["sub"]))
        if not user or not user.is_active:
            raise UnauthorizedError("User not found or inactive")
        return self._issue_tokens(user)

    def _issue_tokens(self, user) -> TokenPair:
        access, access_exp = create_access_token(
            str(user.id), user.email, user.role.value
        )
        refresh, refresh_exp, jti = create_refresh_token(
            str(user.id), user.email, user.role.value
        )
        expires_at = datetime.now(timezone.utc) + timedelta(seconds=refresh_exp)
        self.token_repo.add_token(user.id, jti, expires_at)
        return TokenPair(
            access_token=access,
            refresh_token=refresh,
            expires_in=access_exp,
        )
```

**Takeaway:** The service never imports FastAPI or the database engine. It only uses repository and security functions. That makes it easy to unit-test (pass fake repositories) and to add new flows (e.g. OAuth) by calling `_issue_tokens(user)` after you’ve identified the user.

---

## 9. Dependency Injection

**What you’ll learn:** How FastAPI’s dependency system provides the database session and the “current user” to routes that need them, and how to enforce “only admins can call this” without repeating logic in every route.

**Key idea:** A **dependency** is a function (or callable) that FastAPI runs before your route. If your route declares “I need `CurrentUser`”, FastAPI runs `get_current_user` first: it reads the Bearer token, decodes it, loads the user from the DB, and either injects the user or raises 401. You don’t put that logic in every route—you declare “I need a logged-in user” and get it.

### `app/core/dependencies.py`

```python
from typing import Annotated
from fastapi import Depends, Header
from sqlalchemy.orm import Session
from app.db.base import get_db
from app.models.user import User, Role
from app.core.security import decode_token
from app.core.exceptions import UnauthorizedError, ForbiddenError
from app.repositories.user_repository import UserRepository

# Type aliases for cleaner signatures
DbSession = Annotated[Session, Depends(get_db)]


def get_user_repository(db: DbSession) -> UserRepository:
    return UserRepository(db)


def get_current_user(
    authorization: Annotated[str | None, Header()] = None,
    user_repo: UserRepository = Depends(get_user_repository),
) -> User:
    if not authorization or not authorization.startswith("Bearer "):
        raise UnauthorizedError("Missing or invalid authorization header")
    token = authorization.replace("Bearer ", "").strip()
    payload = decode_token(token)
    if not payload or payload.get("type") != "access":
        raise UnauthorizedError("Invalid or expired token")
    user_id = payload.get("sub")
    if not user_id:
        raise UnauthorizedError("Invalid token payload")
    user = user_repo.get(int(user_id))
    if not user or not user.is_active:
        raise UnauthorizedError("User not found or inactive")
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]


def require_roles(*allowed: Role):
    """Dependency factory: restrict access to given roles."""
    def role_check(user: User) -> User:
        if user.role not in allowed:
            raise ForbiddenError("Insufficient permissions")
        return user
    return Depends(role_check)


# Convenience: restrict endpoints to specific roles
RequireAdmin = require_roles(Role.ADMIN)
RequireEngineer = require_roles(Role.ENGINEER)
RequireSaler = require_roles(Role.SALER)
RequireEngineerOrAdmin = require_roles(Role.ENGINEER, Role.ADMIN)  # Example: use for endpoints that allow either role
```

**Step by step through `get_current_user`:**  
1. Read the `Authorization` header. If it’s missing or doesn’t start with `Bearer `, raise 401.  
2. Extract the token string and call `decode_token(token)`. If it returns `None` (invalid or expired), raise 401.  
3. Check that the payload has `type == "access"` (so someone can’t send a refresh token here).  
4. Get `sub` (user id) from the payload and load the user from the database. If not found or inactive, raise 401.  
5. Return the user object; FastAPI injects it into the route as `current_user`.  

**`require_roles`:** This is a **dependency factory**. You call `require_roles(Role.ADMIN)` and get a dependency that runs after `get_current_user` and checks `user.role in allowed`; if not, it raises 403. So a route that uses `Depends(RequireAdmin)` gets a user only if they are an admin. We have three roles: **engineer**, **saler**, and **admin**; you can combine them (e.g. `require_roles(Role.ENGINEER, Role.ADMIN)` for engineer-or-admin-only endpoints).

---

## 10. API Routes

**What you’ll learn:** How to expose the auth and user operations as HTTP endpoints while keeping route handlers short and delegating all logic to the service and dependencies.

**Key idea:** Each route should do only: (1) receive and validate input (Pydantic does this from the body), (2) get any dependencies (e.g. `AuthService` or `CurrentUser`), (3) call one service method or return the current user, (4) let FastAPI serialize the return value with the declared `response_model`.

### `app/api/v1/auth.py`

```python
from fastapi import APIRouter, Depends
from app.schemas.token import TokenPair, LoginRequest, RefreshRequest
from app.schemas.user import UserCreate, UserResponse
from app.core.dependencies import DbSession, CurrentUser
from app.repositories.user_repository import UserRepository
from app.repositories.token_repository import TokenRepository
from app.services.auth_service import AuthService

router = APIRouter(prefix="/auth", tags=["auth"])


def get_auth_service(
    db: DbSession,
) -> AuthService:
    return AuthService(
        user_repo=UserRepository(db),
        token_repo=TokenRepository(db),
    )


@router.post("/register", response_model=TokenPair)
def register(
    data: UserCreate,
    service: AuthService = Depends(get_auth_service),
):
    return service.register(data)


@router.post("/login", response_model=TokenPair)
def login(
    data: LoginRequest,
    service: AuthService = Depends(get_auth_service),
):
    return service.login(data.email, data.password)


@router.post("/refresh", response_model=TokenPair)
def refresh(
    data: RefreshRequest,
    service: AuthService = Depends(get_auth_service),
):
    return service.refresh_tokens(data.refresh_token)


@router.get("/me", response_model=UserResponse)
def me(current_user: CurrentUser):
    return current_user
```

### `app/api/v1/users.py`

```python
from fastapi import APIRouter, Depends
from app.schemas.user import UserResponse
from app.core.dependencies import CurrentUser, RequireAdmin, RequireEngineer, RequireSaler

router = APIRouter(prefix="/users", tags=["users"])


@router.get("/me", response_model=UserResponse)
def get_me(current_user: CurrentUser):
    return current_user


@router.get("/admin-only", response_model=UserResponse)
def admin_only(current_user: CurrentUser = Depends(RequireAdmin)):
    return current_user


@router.get("/engineer-only", response_model=UserResponse)
def engineer_only(current_user: CurrentUser = Depends(RequireEngineer)):
    return current_user


@router.get("/saler-only", response_model=UserResponse)
def saler_only(current_user: CurrentUser = Depends(RequireSaler)):
    return current_user
```

### `app/api/v1/router.py`

```python
from fastapi import APIRouter
from app.api.v1 import auth, users

api_router = APIRouter()
api_router.include_router(auth.router, prefix="")
api_router.include_router(users.router, prefix="")
```

**What each route does in plain language:**  
- `POST /auth/register`: body = email, password, full_name, role (engineer | saler | admin) → service creates user with chosen role and returns tokens.  
- `POST /auth/login`: body = email, password → service verifies and returns tokens.  
- `POST /auth/refresh`: body = refresh_token → service validates refresh token and returns new token pair.  
- `GET /auth/me`: needs `CurrentUser` → returns that user (no service call).  
- `GET /users/admin-only`: needs `CurrentUser` and `RequireAdmin` → only admins. Similarly `engineer-only` and `saler-only` for engineers and salers.

---

## 11. Role-Based Authorization (RBAC)

**What you’ll learn:** How “only admins can do X” is enforced using the same dependency system as “user must be logged in”.

**Key idea:** **Authentication** = “who are you?” (we answer this with the JWT and `get_current_user`). **Authorization** = “are you allowed to do this?” (we answer with `require_roles`). RBAC means we assign each user a **role**: **engineer**, **saler**, or **admin**. Each endpoint can require one or more roles; if the user’s role isn’t in the allowed list, we return 403.

Flow in code:

- **Authentication:** `get_current_user` (and thus `CurrentUser`) ensures a valid JWT and loads the user.
- **Authorization:** `require_roles(Role.ADMIN)` or `RequireAdmin` ensures the user has one of the allowed roles (engineer, saler, or admin).

Flow:

```
Request with Authorization: Bearer <access_token>
        │
        ▼
  get_current_user
        │
        ├── No/invalid token → 401 Unauthorized
        ├── User inactive → 401
        └── User OK → inject User
                        │
                        ▼
              require_roles(Admin) (if used)
                        │
                        ├── role not in allowed (e.g. Admin) → 403 Forbidden
                        └── OK → inject User, run endpoint
```

**Adding a new role:** (1) Add a new value to the `Role` enum in `app/models/user.py`, (2) run a database migration so the column accepts the new value, (3) assign the role to users as needed, (4) use `Depends(require_roles(Role.NEW_ROLE))` on any endpoint that should be restricted to that role.

---

## 12. Application Entry Point

**What you’ll learn:** How to assemble the FastAPI app: lifespan (create tables on startup), middleware (CORS), exception handlers, and the API router.

**Key idea:** `main.py` is the only place that “wires” everything together. It doesn’t contain business logic—it creates the app, registers handlers, and includes the router. When the app starts, the lifespan runs and we create the database tables; when a request comes in, it goes through middleware, then to the router, which uses the dependencies and services we built.

### `app/main.py`

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.config import settings
from app.core.exceptions import (
    AppException,
    app_exception_handler,
    validation_exception_handler,
)
from fastapi.exceptions import RequestValidationError
from app.db.init_db import init_db
from app.api.v1.router import api_router


@asynccontextmanager
async def lifespan(app: FastAPI):
    init_db()
    yield
    # Optional: close pool, flush logs


app = FastAPI(
    title="Auth API",
    version="1.0.0",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.allowed_origins.split(","),
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.add_exception_handler(AppException, app_exception_handler)
app.add_exception_handler(RequestValidationError, validation_exception_handler)

app.include_router(api_router, prefix=settings.api_v1_prefix)
```

**Run the app:** From the project root, run `uvicorn app.main:app --reload`. The `--reload` flag restarts the server when you change code. Open `http://127.0.0.1:8000/docs` to see the auto-generated Swagger UI and try the endpoints.

---

## 13. Testing the API with Postman

**What you’ll learn:** How to call every auth and user route from Postman: set base URL and body, then use the access token for protected routes.

**Key idea:** Public routes (register, login, refresh) need no token; you send JSON in the body. Protected routes (me, admin-only, etc.) require the `Authorization: Bearer <access_token>` header—you get the token from the login or register response.

### Setup

1. **Base URL:** All routes are under `http://127.0.0.1:8000/api/v1`. Create a Postman environment variable `base_url` = `http://127.0.0.1:8000/api/v1` and use `{{base_url}}` in request URLs, or type the full URL each time.
2. **Headers for JSON:** For any request with a body, set header `Content-Type: application/json` (Postman usually sets this when you pick “raw” + “JSON” in the Body tab).
3. **Start the app** so Postman can reach it: `uvicorn app.main:app --reload`.

---

### 1. Register — `POST /auth/register`

**Purpose:** Create a new user and get access + refresh tokens. The user **chooses their role** in the request body: `"engineer"`, `"saler"`, or `"admin"`.

| Setting | Value |
|--------|--------|
| Method | POST |
| URL | `http://127.0.0.1:8000/api/v1/auth/register` |
| Headers | `Content-Type: application/json` |
| Body | raw, JSON |

**Body (example):**

```json
{
  "email": "engineer@example.com",
  "password": "securepass123",
  "full_name": "Jane Engineer",
  "role": "engineer"
}
```

Use `"role": "engineer"`, `"role": "saler"`, or `"role": "admin"` depending on the type of user being registered.

**What to do:** Send the request. You should get **200** and a JSON body like:

```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "token_type": "bearer",
  "expires_in": 1800
}
```

**Copy the `access_token`** (and optionally the `refresh_token`) for the next steps. If you send the same email again you should get **409 Conflict**.

---

### 2. Login — `POST /auth/login`

**Purpose:** Get access + refresh tokens for an existing user.

| Setting | Value |
|--------|--------|
| Method | POST |
| URL | `http://127.0.0.1:8000/api/v1/auth/login` |
| Headers | `Content-Type: application/json` |
| Body | raw, JSON |

**Body (example):**

```json
{
  "email": "engineer@example.com",
  "password": "securepass123"
}
```

**What to do:** Send the request. You should get **200** and the same token shape as register. Copy `access_token` for protected routes. Wrong password or unknown email → **401 Unauthorized**.

---

### 3. Refresh tokens — `POST /auth/refresh`

**Purpose:** Get a new access (and optionally refresh) token using a valid refresh token.

| Setting | Value |
|--------|--------|
| Method | POST |
| URL | `http://127.0.0.1:8000/api/v1/auth/refresh` |
| Headers | `Content-Type: application/json` |
| Body | raw, JSON |

**Body (example):**

```json
{
  "refresh_token": "eyJhbGc..."
}
```

Paste the `refresh_token` you got from login or register. You should get **200** and a new token pair. Invalid or expired refresh token → **401**.

---

### 4. Get current user (auth) — `GET /auth/me`

**Purpose:** Return the user identified by the access token. **Requires authentication.**

| Setting | Value |
|--------|--------|
| Method | GET |
| URL | `http://127.0.0.1:8000/api/v1/auth/me` |
| Authorization | Type: **Bearer Token**. Token: paste the `access_token` from login/register. |

**What to do:** In Postman, open the **Authorization** tab, choose **Type: Bearer Token**, and paste the `access_token` into the “Token” field. Send the request. You should get **200** and your user object (id, email, full_name, role, etc.). No token or invalid token → **401**.

---

### 5. Get current user (users) — `GET /users/me`

Same as `/auth/me`: **GET** `http://127.0.0.1:8000/api/v1/users/me` with **Bearer Token** = `access_token`. **200** returns the same user object.

---

### 6. Admin-only — `GET /users/admin-only`

**Purpose:** Only users with role **admin** can call this. **Requires authentication + admin role.**

| Setting | Value |
|--------|--------|
| Method | GET |
| URL | `http://127.0.0.1:8000/api/v1/users/admin-only` |
| Authorization | Bearer Token = `access_token` of an **admin** user. |

**What to do:** Log in as a user that has `role: "admin"` (you may need to set this in the DB for a test user). Use that user’s `access_token`. You should get **200** and the user. If the token belongs to an engineer or saler → **403 Forbidden**. No/invalid token → **401**.

---

### 7. Engineer-only — `GET /users/engineer-only`

Same as admin-only but for role **engineer**. **GET** `http://127.0.0.1:8000/api/v1/users/engineer-only` with Bearer Token of an **engineer** user. Engineer → **200**; admin or saler → **403**; no token → **401**.

---

### 8. Saler-only — `GET /users/saler-only`

Same as above for role **saler**. **GET** `http://127.0.0.1:8000/api/v1/users/saler-only` with Bearer Token of a **saler** user. Saler → **200**; others → **403**; no token → **401**.

---

### Quick reference

| Route | Method | Auth | Body | Expected |
|-------|--------|------|------|----------|
| `/api/v1/auth/register` | POST | No | email, password, full_name, role | 200 + tokens |
| `/api/v1/auth/login` | POST | No | email, password | 200 + tokens |
| `/api/v1/auth/refresh` | POST | No | refresh_token | 200 + tokens |
| `/api/v1/auth/me` | GET | Bearer | — | 200 + user |
| `/api/v1/users/me` | GET | Bearer | — | 200 + user |
| `/api/v1/users/admin-only` | GET | Bearer (admin) | — | 200 + user (admin only) |
| `/api/v1/users/engineer-only` | GET | Bearer (engineer) | — | 200 + user (engineer only) |
| `/api/v1/users/saler-only` | GET | Bearer (saler) | — | 200 + user (saler only) |

**Tip:** Create one Postman request per route and save the `access_token` in an environment variable (e.g. from a test script on the login request: “Tests” tab → `pm.environment.set("access_token", pm.response.json().access_token);`). Then set Authorization to “Bearer Token” and value `{{access_token}}` so you don’t paste the token manually for each protected request.

---

## 14. Testing Strategy

**What you’ll learn:** How to test the auth API with pytest (automated tests): use a separate test database, override the `get_db` dependency so tests don’t touch the real DB, and write simple tests that call the client and assert on status codes and response body.

**Key idea:** We don’t run tests against the real database or real secrets. We point the app at a test database (e.g. `appdb_test`), and we override `get_db` so each test (or each test run) gets a clean state. Then we use the TestClient to send HTTP requests and assert on the response.

### `tests/conftest.py`

```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.db.base import Base, get_db
from app.models.user import User, Role
from app.core.security import hash_password

# Use a separate test database so we don't touch real data
# Create it first: createdb appdb_test
TEST_DATABASE_URL = "postgresql://user:password@localhost:5432/appdb_test"
engine = create_engine(TEST_DATABASE_URL, pool_pre_ping=True)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()


@pytest.fixture(scope="function")
def db_session():
    Base.metadata.create_all(bind=engine)
    session = TestingSessionLocal()
    try:
        yield session
    finally:
        session.close()
        Base.metadata.drop_all(bind=engine)


@pytest.fixture
def client(db_session):
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as c:
        yield c
    app.dependency_overrides.clear()


@pytest.fixture
def test_user(db_session):
    user = User(
        email="test@example.com",
        hashed_password=hash_password("password123"),
        full_name="Test User",
        role=Role.ENGINEER,
    )
    db_session.add(user)
    db_session.commit()
    db_session.refresh(user)
    return user
```

### `tests/test_auth_api.py`

```python
import pytest
from fastapi.testclient import TestClient


def test_register(client: TestClient):
    response = client.post(
        "/api/v1/auth/register",
        json={
            "email": "new@example.com",
            "password": "securepass123",
            "full_name": "New User",
            "role": "engineer",
        },
    )
    assert response.status_code == 200
    data = response.json()
    assert "access_token" in data
    assert data["token_type"] == "bearer"


def test_login_success(client: TestClient, test_user):
    response = client.post(
        "/api/v1/auth/login",
        json={"email": "test@example.com", "password": "password123"},
    )
    assert response.status_code == 200
    assert "access_token" in response.json()


def test_login_wrong_password(client: TestClient, test_user):
    response = client.post(
        "/api/v1/auth/login",
        json={"email": "test@example.com", "password": "wrong"},
    )
    assert response.status_code == 401


def test_me_requires_auth(client: TestClient):
    response = client.get("/api/v1/auth/me")
    assert response.status_code == 401
```

**How the fixtures work:**  
- `db_session`: Creates all tables at the start and drops them at the end so each test run starts with an empty DB (or use a transaction and rollback for speed).  
- `client`: Overrides `get_db` so that when the app handles a request, it uses our test engine/session instead of the real one.  
- `test_user`: Inserts one user with a known password so we can test login and protected endpoints.  

**Next step:** For routes that require auth, you can add a fixture that overrides `get_current_user` to return `test_user` so you don’t need to log in in every test.

---

## 15. Extending the System

**What you’ll learn:** Where to plug in OAuth, email verification, 2FA, or password reset without rewriting the auth system we built.

**Key idea:** Because we separated layers (router → service → repository) and centralized token issuance in `_issue_tokens`, new flows mostly add new code: a new service method, maybe a new route, and reuse the same tokens and RBAC. The table below says where each extension lives and how it connects.

| Extension | Where to add | Notes |
|-----------|--------------|--------|
| **OAuth (Google, GitHub)** | New `app/services/oauth_service.py`, new routes in `app/api/v1/auth.py` | Use `authlib` or `httpx`; create or link user, then call `_issue_tokens(user)`. |
| **Email verification** | New `app/models/user.py` field `email_verified`, `app/services/email_service.py`, task queue | On register, send link; add dependency `require_verified_email` that raises 403 if not verified. |
| **2FA (TOTP)** | New `app/models/user.py` field `totp_secret`, `app/core/security.py` (e.g. `pyotp`), new routes `/auth/2fa/enable`, `/auth/2fa/verify` | In `login`, if 2FA enabled return temp token or require `code`; after verify, issue normal tokens. |
| **Password reset** | New table `password_reset_tokens`, `app/services/password_reset_service.py`, email sending | Token in link; short-lived; after use delete token and update password. |

**Golden rule:** New auth methods (OAuth, 2FA) should still end by calling the same `_issue_tokens(user)` and returning `TokenPair`. Then the rest of the API (protected routes, RBAC) works without change.

---

## 16. Production Deployment & Security

**What you’ll learn:** Concrete steps to harden the app and run it safely in production (secrets, HTTPS, DB, logging, etc.).

**Key idea:** Development is permissive (e.g. broad CORS, long-lived tokens for convenience). Production should lock down: real secrets, HTTPS only, strict CORS, rate limiting on auth endpoints, and proper logging without leaking tokens or passwords.

Checklist:

- **Secrets:** Never commit `.env`. Use a secrets manager (e.g. AWS Secrets Manager, HashiCorp Vault) and inject `SECRET_KEY` and `DATABASE_URL` at runtime.
- **HTTPS only:** Enforce TLS; set `Secure` and `SameSite` on cookies if you use cookie-based refresh tokens.
- **Token storage:** Store access tokens in memory or short-lived storage; store refresh tokens in httpOnly cookies or secure storage and rotate on use.
- **CORS:** Set `allow_origins` to exact front-end origins, not `*` in production.
- **Rate limiting:** Add rate limiting (e.g. `slowapi` or reverse proxy) on `/auth/login` and `/auth/register` to mitigate brute force.
- **Password policy:** Enforce minimum length and complexity in `UserCreate` or a validator; keep `bcrypt_rounds` ≥ 12.
- **DB:** Use PostgreSQL with connection pooling (`pool_size`, `max_overflow`). For async, use `postgresql+asyncpg`. Run migrations (e.g. Alembic) instead of `create_all` in production. Create the database and user before first run (e.g. `createdb appdb`).
- **Logging:** Log auth failures (without passwords), token revocations, and role changes; use structured logging and avoid logging tokens.
- **Dependencies:** Pin versions in `requirements.txt`, run `pip audit` and update regularly.
- **Headers:** Add security headers (e.g. `X-Content-Type-Options`, `Strict-Transport-Security`) via middleware or reverse proxy.

---

## Summary

**What we built:** A full auth system with registration, login, refresh, and role-based access. We use three roles: **engineer**, **saler**, and **admin**. Every request that needs a user goes through the same dependency (`get_current_user`); every protected endpoint can optionally require one or more roles (`RequireAdmin`, `RequireEngineer`, `RequireSaler`, or custom combinations). Passwords are hashed, tokens are signed JWTs, and the database stores users and refresh tokens for revocation.

You now have a single Markdown guide that walks through:

- A **layered architecture** (routers → services → repositories → DB) with clear responsibilities.
- **JWT access + refresh** tokens and **passlib** for passwords.
- **Pydantic** for config and request/response schemas.
- **SQLAlchemy** models and repositories with **dependency injection** for DB and current user.
- **RBAC** via dependencies (`CurrentUser`, `RequireAdmin`, `RequireEngineer`, `RequireSaler`) with three roles: engineer, saler, admin.
- **Centralized exceptions** and **pytest**-based tests.
- **Extension points** for OAuth, email verification, and 2FA.
- **Production and security** best practices.

Save this file as `fastapi_auth_guide.md` and use it as a reference while implementing or refactoring your FastAPI auth system.
