---
name: python-web-fastapi-django
description: Modern Python web development with FastAPI, Django, and DRF. Includes async patterns, Pydantic v2, and best practices for 2026.
---

# Python Web Stack: FastAPI + Django + DRF (2026)

## Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| **FastAPI** | Web framework | 0.115.x |
| **Django** | Web framework | 5.1.x |
| **Django REST Framework** | API toolkit | 3.15.x |
| **Pydantic** | Data validation | 2.9.x |
| **SQLModel** | ORM | 0.0.22 |
| **Async PostgreSQL** | Database driver | asyncpg 17.x |
| **Celery** | Task queue | 5.4.x |
| **Redis** | Cache/queue | Redis 5.x |
| **Testing** | pytest | 8.3.x |

## FastAPI Architecture

### Basic Structure
```python
# src/
# ├── main.py
# ├── api/
# │   ├── __init__.py
# │   ├── v1/
# │   │   ├── __init__.py
# │   │   ├── endpoints/
# │   │   │   ├── __init__.py
# │   │   │   ├── devices.py
# │   │   │   └── users.py
# │   │   └── router.py
# ├── core/
# │   ├── __init__.py
# │   ├── config.py
# │   └── security.py
# ├── models/
# │   ├── __init__.py
# │   └── device.py
# └── schemas/
#     ├── __init__.py
#     └── device.py

from fastapi import FastAPI, Depends
from fastapi.middleware.cors import CORSMiddleware

from api.v1.router import api_router
from core.config import settings

app = FastAPI(
    title=settings.PROJECT_NAME,
    description=settings.PROJECT_DESCRIPTION,
    version=settings.VERSION,
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.BACKEND_CORS_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["*"],
)

@app.get("/health", tags=["health"])
def health_check():
    """Health check endpoint"""
    return {"status": "healthy", "version": settings.VERSION}

app.include_router(api_router, prefix=settings.API_V1_STR)
```

### Pydantic v2 Models
```python
# src/models/device.py
from typing import Optional, List
from pydantic import BaseModel, Field, field_validator
from datetime import datetime
import re

class DeviceBase(BaseModel):
    """Base device schema with common fields"""
    name: str = Field(..., min_length=1, max_length=100)
    serial_number: str = Field(..., min_length=5, max_length=50)
    status: str = "offline"
    
    @field_validator('serial_number')
    @classmethod
    def validate_serial_number(cls, v: str) -> str:
        """Validate serial number format"""
        if not re.match(r'^[A-Z0-9]{5,50}$', v):
            raise ValueError('Serial number must be uppercase letters and numbers, 5-50 chars')
        return v

class DeviceCreate(DeviceBase):
    """Schema for creating a new device"""
    location_id: Optional[int] = None

class DeviceUpdate(BaseModel):
    """Schema for updating a device"""
    name: Optional[str] = Field(None, min_length=1, max_length=100)
    status: Optional[str] = None
    location_id: Optional[int] = None

class Device(DeviceBase):
    """Schema for device response"""
    id: int
    created_at: datetime
    updated_at: Optional[datetime] = None
    
    class Config:
        from_attributes = True  # Replaces model_validate with v2 syntax
```

### Database with SQLModel
```python
# src/database.py
from sqlmodel import SQLModel, create_engine, Session
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# For async operations
DATABASE_URL = "postgresql+asyncpg://user:password@localhost/dbname"
engine = create_async_engine(DATABASE_URL, echo=False)

# For sync operations (if needed)
sync_engine = create_engine(DATABASE_URL.replace("+asyncpg", ""), echo=False)
SyncSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=sync_engine)

async def get_async_session() -> AsyncSession:
    """Dependency for getting async database session"""
    async with AsyncSession(engine) as session:
        yield session

async def init_db():
    """Initialize database tables"""
    async with engine.begin() as conn:
        await conn.run_sync(SQLModel.metadata.create_all)
```

### API Endpoints
```python
# src/api/v1/endpoints/devices.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession
from typing import List

from database import get_async_session
from schemas.device import Device, DeviceCreate, DeviceUpdate
from models.device import Device as DeviceModel
from services import device_service

router = APIRouter(prefix="/devices", tags=["devices"])

@router.get("/", response_model=List[Device])
async def read_devices(
    skip: int = 0,
    limit: int = 100,
    session: AsyncSession = Depends(get_async_session)
):
    """Read all devices"""
    devices = await device_service.get_devices(session, skip=skip, limit=limit)
    return devices

@router.get("/{device_id}", response_model=Device)
async def read_device(
    device_id: int,
    session: AsyncSession = Depends(get_async_session)
):
    """Read a specific device"""
    device = await device_service.get_device(session, device_id=device_id)
    if device is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Device not found"
        )
    return device

@router.post("/", response_model=Device, status_code=status.HTTP_201_CREATED)
async def create_device(
    device: DeviceCreate,
    session: AsyncSession = Depends(get_async_session)
):
    """Create a new device"""
    return await device_service.create_device(session, device=device)

@router.put("/{device_id}", response_model=Device)
async def update_device(
    device_id: int,
    device: DeviceUpdate,
    session: AsyncSession = Depends(get_async_session)
):
    """Update an existing device"""
    updated_device = await device_service.update_device(
        session, device_id=device_id, device=device
    )
    if updated_device is None:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Device not found"
        )
    return updated_device

@router.delete("/{device_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_device(
    device_id: int,
    session: AsyncSession = Depends(get_async_session)
):
    """Delete a device"""
    success = await device_service.delete_device(session, device_id=device_id)
    if not success:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Device not found"
        )
```

## Django + DRF Architecture

### Project Structure
```python
# myproject/
# ├── manage.py
# ├── myproject/
# │   ├── __init__.py
# │   ├── settings.py
# │   ├── urls.py
# │   └── wsgi.py
# ├── devices/
# │   ├── __init__.py
# │   ├── admin.py
# │   ├── apps.py
# │   ├── models.py
# │   ├── serializers.py
# │   ├── views.py
# │   └── urls.py
# └── users/
#     └── ...

# devices/models.py
from django.db import models
from django.contrib.auth.models import User

class Device(models.Model):
    STATUS_CHOICES = [
        ('offline', 'Offline'),
        ('online', 'Online'),
        ('maintenance', 'Maintenance'),
    ]
    
    name = models.CharField(max_length=100)
    serial_number = models.CharField(max_length=50, unique=True)
    status = models.CharField(
        max_length=20,
        choices=STATUS_CHOICES,
        default='offline'
    )
    location = models.CharField(max_length=255, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='devices')
    
    class Meta:
        indexes = [
            models.Index(fields=['serial_number']),
            models.Index(fields=['status']),
        ]
    
    def __str__(self):
        return f"{self.name} ({self.serial_number})"

# devices/serializers.py
from rest_framework import serializers
from .models import Device

class DeviceSerializer(serializers.ModelSerializer):
    """Serializer for Device model"""
    owner = serializers.ReadOnlyField(source='owner.username')
    
    class Meta:
        model = Device
        fields = [
            'id', 'name', 'serial_number', 'status', 
            'location', 'created_at', 'updated_at', 'owner'
        ]
        read_only_fields = ['id', 'created_at', 'updated_at', 'owner']

# devices/views.py
from rest_framework import viewsets, permissions
from .models import Device
from .serializers import DeviceSerializer

class DeviceViewSet(viewsets.ModelViewSet):
    """
    ViewSet for Device model.
    Provides cruds for devices with filtering.
    """
    queryset = Device.objects.all()
    serializer_class = DeviceSerializer
    permission_classes = [permissions.IsAuthenticated]
    filterset_fields = ['status', 'location']
    search_fields = ['name', 'serial_number']
    ordering_fields = ['created_at', 'updated_at', 'name']
    
    def perform_create(self, serializer):
        """Set owner to current user"""
        serializer.save(owner=self.request.user)
    
    def get_queryset(self):
        """Filter to show only user's devices"""
        return super().get_queryset().filter(owner=self.request.user)
```

### URL Configuration
```python
# devices/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'devices', views.DeviceViewSet)

urlpatterns = [
    path('', include(router.urls)),
]

# myproject/urls.py
from django.urls import path, include

urlpatterns = [
    path('api/v1/', include('devices.urls')),
    path('api-auth/', include('rest_framework.urls')),
]
```

## Async Patterns & Performance

### Background Tasks with Celery
```python
# tasks.py
from celery import Celery
import asyncio
from database import get_async_session
from services import process_device_data

celery_app = Celery(
    'tasks',
    broker='redis://localhost:6379/0',
    backend='redis://localhost:6379/0'
)

@celery_app.task(bind=True)
def process_device_readings(self, device_id: int):
    """Process device readings in background"""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    
    try:
        session = loop.run_until_complete(get_async_session().__anext__())
        result = loop.run_until_complete(
            process_device_data(session, device_id)
        )
        return result
    finally:
        loop.close()

# In your FastAPI endpoint
from tasks import process_device_readings

@router.post("/{device_id}/process")
async def process_device(device_id: int):
    """Trigger background processing"""
    process_device_readings.delay(device_id)
    return {"message": "Processing started"}
```

### Caching
```python
# cache.py
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend
from fastapi_cache.decorator import cache
import redis.asyncio as redis

@cache(expire=60)  # Cache for 60 seconds
async def get_device_with_cache(device_id: int, session: AsyncSession):
    """Get device with caching"""
    return await device_service.get_device(session, device_id)
```

## Testing

### Pytest Configuration
```python
# conftest.py
import pytest
from fastapi.testclient import TestClient

from main import app
from database import get_async_session
from models.device import Device

@pytest.fixture(scope="module")
def client():
    """Create test client"""
    with TestClient(app) as client:
        yield client

@pytest.fixture
def test_device():
    """Create test device data"""
    return {
        "name": "Test Device",
        "serial_number": "TEST001",
        "status": "offline"
    }
```

### API Tests
```python
# tests/test_devices.py
def test_create_device(client, test_device):
    """Test device creation"""
    response = client.post("/devices/", json=test_device)
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == test_device["name"]
    assert "id" in data

def test_read_devices(client, test_device):
    """Test reading devices"""
    # First create a device
    client.post("/devices/", json=test_device)
    
    # Then read all devices
    response = client.get("/devices/")
    assert response.status_code == 200
    data = response.json()
    assert len(data) >= 1
```

## Docker Configuration

### Dockerfile
```dockerfile
# Dockerfile
FROM python:3.12-slim as builder

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/

# Production stage
FROM python:3.12-slim

WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*

# Copy from builder
COPY --from=builder /app /app

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### docker-compose.yml
```yaml
version: '3.8'

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data/
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  web:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src
    environment:
      DATABASE_URL: postgresql://user:password@db/myapp
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - db
      - redis

volumes:
  postgres_data:
```

## Common Patterns & Anti-Patterns

### Do
✅ Use Pydantic v2 with `from_attributes=True`  
✅ Use async/await for database operations  
✅ Implement pagination with `skip` and `limit`  
✅ Use FastAPI dependencies for authentication  
✅ Document endpoints with Pydantic schemas  
✅ Use Django ORM for complex queries  

### Don't
❌ Use sync database operations in async endpoints  
❌ Skip input validation  
❌ Hardcode configurations  
❌ Ignore error handling  
❌ Use `select *` in raw SQL queries  

## References

* [FastAPI Documentation](https://fastapi.tiangolo.com)
* [Django Documentation](https://docs.djangoproject.com)
* [Django REST Framework](https://www.django-rest-framework.org)
* [Pydantic v2](https://docs.pydantic.dev)
* [SQLModel](https://sqlmodel.tiangolo.com)

## 2026 Best Practices

* **FastAPI 0.115**: Latest async-first framework
* **Pydantic v2.9**: Better performance with Rust
* **Async First**: Use async/await throughout
* **Django 5.1**: Latest with async view support
* **SQLModel**: Modern ORM for async operations
