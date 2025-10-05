# Domain-Driven Design Implementation Guide

## Overview

This guide provides step-by-step instructions for implementing Domain-Driven Design (DDD) + Clean Architecture + CQRS patterns in the Solidity Security Platform services.

## Architecture Layers

### 1. Domain Layer (Pure Business Logic)

The domain layer contains the core business logic and has zero external dependencies.

#### Domain Entities

Domain entities represent core business concepts with identity and lifecycle.

```python
# src/domain/entities/user.py
from datetime import datetime
from typing import Optional
from ..value_objects.email import Email
from ..value_objects.password import Password

class User:
    def __init__(
        self,
        user_id: int,
        email: Email,
        username: str,
        full_name: Optional[str] = None
    ):
        self._id = user_id
        self._email = email
        self._username = username
        self._full_name = full_name
        self._is_active = True
        self._created_at = datetime.utcnow()
        self._updated_at = None

    @property
    def id(self) -> int:
        return self._id

    @property
    def email(self) -> Email:
        return self._email

    @property
    def username(self) -> str:
        return self._username

    @property
    def is_active(self) -> bool:
        return self._is_active

    def deactivate(self) -> None:
        """Business rule: User can be deactivated"""
        self._is_active = False
        self._updated_at = datetime.utcnow()

    def update_profile(self, full_name: Optional[str]) -> None:
        """Business rule: User can update their profile"""
        self._full_name = full_name
        self._updated_at = datetime.utcnow()

    def can_create_project(self) -> bool:
        """Business rule: Only active users can create projects"""
        return self._is_active
```

#### Value Objects

Value objects represent concepts without identity that are defined by their attributes.

```python
# src/domain/value_objects/email.py
import re
from dataclasses import dataclass

@dataclass(frozen=True)
class Email:
    value: str

    def __post_init__(self):
        if not self._is_valid_email(self.value):
            raise ValueError(f"Invalid email format: {self.value}")

    @staticmethod
    def _is_valid_email(email: str) -> bool:
        pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
        return bool(re.match(pattern, email))

    def domain(self) -> str:
        return self.value.split('@')[1]
```

#### Domain Services

Domain services contain business logic that doesn't naturally fit into entities or value objects.

```python
# src/domain/services/auth_service.py
from typing import Optional
from ..entities.user import User
from ..value_objects.email import Email
from ..value_objects.password import Password

class AuthDomainService:
    def can_login(self, user: User, password: Password) -> bool:
        """Business rule: Only active users can login"""
        return user.is_active

    def generate_username_from_email(self, email: Email) -> str:
        """Business rule: Generate username from email local part"""
        local_part = email.value.split('@')[0]
        return local_part.lower().replace('.', '_')

    def is_password_strong_enough(self, password: Password) -> bool:
        """Business rule: Password strength requirements"""
        return (
            len(password.value) >= 8 and
            any(c.isupper() for c in password.value) and
            any(c.islower() for c in password.value) and
            any(c.isdigit() for c in password.value)
        )
```

#### Repository Interfaces

Repository interfaces define contracts for data access without implementation details.

```python
# src/domain/repositories/user_repository.py
from abc import ABC, abstractmethod
from typing import Optional, List
from ..entities.user import User
from ..value_objects.email import Email

class UserRepository(ABC):
    @abstractmethod
    async def save(self, user: User) -> User:
        """Save user to storage"""
        pass

    @abstractmethod
    async def find_by_id(self, user_id: int) -> Optional[User]:
        """Find user by ID"""
        pass

    @abstractmethod
    async def find_by_email(self, email: Email) -> Optional[User]:
        """Find user by email"""
        pass

    @abstractmethod
    async def find_by_username(self, username: str) -> Optional[User]:
        """Find user by username"""
        pass

    @abstractmethod
    async def list_active_users(self, limit: int = 100) -> List[User]:
        """List active users"""
        pass

    @abstractmethod
    async def delete(self, user_id: int) -> bool:
        """Delete user"""
        pass
```

### 2. Application Layer (Use Cases)

The application layer orchestrates domain objects to fulfill use cases.

#### Commands (Write Operations)

Commands represent actions that change system state.

```python
# src/application/auth/commands/register_command.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class RegisterUserCommand:
    email: str
    username: str
    password: str
    full_name: Optional[str] = None
```

#### Queries (Read Operations)

Queries represent data retrieval operations.

```python
# src/application/auth/queries/get_user_query.py
from dataclasses import dataclass
from typing import Optional

@dataclass
class GetUserByEmailQuery:
    email: str

@dataclass
class GetUserByIdQuery:
    user_id: int
```

#### Command Handlers

Command handlers execute business logic for write operations.

```python
# src/application/auth/handlers/auth_command_handlers.py
from typing import Optional
from ...domain.entities.user import User
from ...domain.repositories.user_repository import UserRepository
from ...domain.services.auth_service import AuthDomainService
from ...domain.value_objects.email import Email
from ...domain.value_objects.password import Password
from ..commands.register_command import RegisterUserCommand

class RegisterUserCommandHandler:
    def __init__(
        self,
        user_repository: UserRepository,
        auth_service: AuthDomainService
    ):
        self._user_repository = user_repository
        self._auth_service = auth_service

    async def handle(self, command: RegisterUserCommand) -> User:
        # Validate email
        email = Email(command.email)

        # Check if user already exists
        existing_user = await self._user_repository.find_by_email(email)
        if existing_user:
            raise ValueError("User with this email already exists")

        # Validate password strength
        password = Password(command.password)
        if not self._auth_service.is_password_strong_enough(password):
            raise ValueError("Password does not meet strength requirements")

        # Create user entity
        user = User(
            user_id=0,  # Will be set by repository
            email=email,
            username=command.username,
            full_name=command.full_name
        )

        # Save user
        saved_user = await self._user_repository.save(user)
        return saved_user
```

#### Query Handlers

Query handlers execute data retrieval operations.

```python
# src/application/auth/handlers/auth_query_handlers.py
from typing import Optional
from ...domain.entities.user import User
from ...domain.repositories.user_repository import UserRepository
from ...domain.value_objects.email import Email
from ..queries.get_user_query import GetUserByEmailQuery, GetUserByIdQuery

class GetUserQueryHandler:
    def __init__(self, user_repository: UserRepository):
        self._user_repository = user_repository

    async def handle(self, query: GetUserByEmailQuery) -> Optional[User]:
        email = Email(query.email)
        return await self._user_repository.find_by_email(email)

    async def handle_by_id(self, query: GetUserByIdQuery) -> Optional[User]:
        return await self._user_repository.find_by_id(query.user_id)
```

### 3. Infrastructure Layer (Technical Implementation)

The infrastructure layer implements the interfaces defined in the domain layer.

#### Database Models

SQLAlchemy models for data persistence.

```python
# src/infrastructure/database/models/user_model.py
from sqlalchemy import Column, Integer, String, Boolean, DateTime
from sqlalchemy.sql import func
from .base_model import BaseModel

class UserModel(BaseModel):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String, unique=True, index=True, nullable=False)
    username = Column(String, unique=True, index=True, nullable=False)
    full_name = Column(String, nullable=True)
    hashed_password = Column(String, nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())
```

#### Repository Implementation

Concrete implementation of repository interfaces.

```python
# src/infrastructure/database/repositories/user_repository_impl.py
from typing import Optional, List
from sqlalchemy.orm import Session
from sqlalchemy import select

from ...domain.entities.user import User
from ...domain.repositories.user_repository import UserRepository
from ...domain.value_objects.email import Email
from ..models.user_model import UserModel

class SqlAlchemyUserRepository(UserRepository):
    def __init__(self, session: Session):
        self._session = session

    async def save(self, user: User) -> User:
        user_model = UserModel(
            email=user.email.value,
            username=user.username,
            full_name=user.full_name,
            is_active=user.is_active
        )

        self._session.add(user_model)
        await self._session.commit()
        await self._session.refresh(user_model)

        return self._model_to_entity(user_model)

    async def find_by_email(self, email: Email) -> Optional[User]:
        stmt = select(UserModel).where(UserModel.email == email.value)
        result = await self._session.execute(stmt)
        user_model = result.scalar_one_or_none()

        return self._model_to_entity(user_model) if user_model else None

    def _model_to_entity(self, model: UserModel) -> User:
        """Convert SQLAlchemy model to domain entity"""
        return User(
            user_id=model.id,
            email=Email(model.email),
            username=model.username,
            full_name=model.full_name
        )
```

### 4. Presentation Layer (API Interface)

The presentation layer handles HTTP requests and responses.

#### API Endpoints

FastAPI routers for HTTP endpoints.

```python
# src/presentation/api/v1/auth/router.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from ....application.auth.commands.register_command import RegisterUserCommand
from ....application.auth.handlers.auth_command_handlers import RegisterUserCommandHandler
from ....infrastructure.database.connection import get_db
from ....infrastructure.database.repositories.user_repository_impl import SqlAlchemyUserRepository
from ....domain.services.auth_service import AuthDomainService
from .schemas import UserCreateRequest, UserResponse

router = APIRouter(prefix="/auth", tags=["authentication"])

@router.post("/register", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def register_user(
    request: UserCreateRequest,
    db: Session = Depends(get_db)
):
    """Register a new user"""
    # Dependency injection
    user_repository = SqlAlchemyUserRepository(db)
    auth_service = AuthDomainService()
    handler = RegisterUserCommandHandler(user_repository, auth_service)

    # Create command
    command = RegisterUserCommand(
        email=request.email,
        username=request.username,
        password=request.password,
        full_name=request.full_name
    )

    try:
        # Execute command
        user = await handler.handle(command)

        # Return response
        return UserResponse(
            id=user.id,
            email=user.email.value,
            username=user.username,
            full_name=user.full_name,
            is_active=user.is_active
        )
    except ValueError as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=str(e)
        )
```

#### Request/Response Schemas

Pydantic schemas for API validation.

```python
# src/presentation/api/v1/auth/schemas.py
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, EmailStr

class UserCreateRequest(BaseModel):
    email: EmailStr
    username: str
    password: str
    full_name: Optional[str] = None

class UserResponse(BaseModel):
    id: int
    email: str
    username: str
    full_name: Optional[str]
    is_active: bool

    class Config:
        from_attributes = True
```

## Testing Strategy

### Unit Testing

Test each layer in isolation.

```python
# tests/unit/domain/test_user_entity.py
import pytest
from src.domain.entities.user import User
from src.domain.value_objects.email import Email

def test_user_creation():
    email = Email("test@example.com")
    user = User(
        user_id=1,
        email=email,
        username="testuser"
    )

    assert user.id == 1
    assert user.email == email
    assert user.username == "testuser"
    assert user.is_active is True

def test_user_deactivation():
    email = Email("test@example.com")
    user = User(
        user_id=1,
        email=email,
        username="testuser"
    )

    user.deactivate()
    assert user.is_active is False

def test_can_create_project_when_active():
    email = Email("test@example.com")
    user = User(
        user_id=1,
        email=email,
        username="testuser"
    )

    assert user.can_create_project() is True

def test_cannot_create_project_when_inactive():
    email = Email("test@example.com")
    user = User(
        user_id=1,
        email=email,
        username="testuser"
    )

    user.deactivate()
    assert user.can_create_project() is False
```

### Integration Testing

Test layer interactions.

```python
# tests/integration/test_user_registration.py
import pytest
from sqlalchemy.orm import Session

from src.application.auth.commands.register_command import RegisterUserCommand
from src.application.auth.handlers.auth_command_handlers import RegisterUserCommandHandler
from src.infrastructure.database.repositories.user_repository_impl import SqlAlchemyUserRepository
from src.domain.services.auth_service import AuthDomainService

@pytest.mark.asyncio
async def test_user_registration_success(db_session: Session):
    # Arrange
    user_repository = SqlAlchemyUserRepository(db_session)
    auth_service = AuthDomainService()
    handler = RegisterUserCommandHandler(user_repository, auth_service)

    command = RegisterUserCommand(
        email="test@example.com",
        username="testuser",
        password="StrongPassword123",
        full_name="Test User"
    )

    # Act
    user = await handler.handle(command)

    # Assert
    assert user.id is not None
    assert user.email.value == "test@example.com"
    assert user.username == "testuser"
    assert user.full_name == "Test User"
    assert user.is_active is True

@pytest.mark.asyncio
async def test_user_registration_duplicate_email(db_session: Session):
    # Arrange
    user_repository = SqlAlchemyUserRepository(db_session)
    auth_service = AuthDomainService()
    handler = RegisterUserCommandHandler(user_repository, auth_service)

    # Create first user
    command1 = RegisterUserCommand(
        email="test@example.com",
        username="testuser1",
        password="StrongPassword123"
    )
    await handler.handle(command1)

    # Try to create second user with same email
    command2 = RegisterUserCommand(
        email="test@example.com",
        username="testuser2",
        password="StrongPassword123"
    )

    # Act & Assert
    with pytest.raises(ValueError, match="User with this email already exists"):
        await handler.handle(command2)
```

## Best Practices

### 1. Domain Layer Guidelines
- Keep domain entities focused on business logic
- Use value objects for concepts without identity
- Domain services for business logic that spans entities
- No external dependencies in domain layer

### 2. Application Layer Guidelines
- One command/query per use case
- Handlers orchestrate domain objects
- Validate input data before creating domain objects
- Use dependency injection for repositories

### 3. Infrastructure Layer Guidelines
- Implement repository interfaces from domain
- Keep database models separate from domain entities
- Use mapping functions to convert between layers
- Handle technical exceptions at this layer

### 4. Presentation Layer Guidelines
- Validate input with Pydantic schemas
- Convert between domain entities and API responses
- Handle application exceptions appropriately
- Keep controllers thin - delegate to application layer

## Getting Started

1. **Start with Domain Layer**: Define entities and value objects
2. **Add Repository Interfaces**: Define data access contracts
3. **Implement Application Layer**: Create commands, queries, and handlers
4. **Build Infrastructure**: Implement repositories and external services
5. **Create API Layer**: Build FastAPI endpoints and schemas
6. **Add Tests**: Unit tests for domain, integration tests for workflows

This architecture provides a solid foundation for building maintainable, testable, and scalable services in the Solidity Security Platform.