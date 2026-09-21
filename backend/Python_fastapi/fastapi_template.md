# My FastAPI Interview Template

## Pre-decided, every time
- **FastAPI + Python** — automatic validation + docs, faster than Flask under time pressure
- **SQLite for the exercise** — zero setup, real persistence; would use Postgres in production for concurrent-write scale
- **Skip unless required:** auth, pagination, rate limiting, Alembic migrations, UUID keys
- **Clarify first:** exact input format? expected scale? anything ambiguous?

## Folder structure (flat — correct for a single-resource, few-endpoint build)
```
├── main.py              # FastAPI app, routes, dependency injection
├── models.py             # SQLAlchemy models (Sensor, Reading)
├── schemas.py             # Pydantic request/response schemas
├── database.py            # Engine, session, declarative base
├── requirements.txt
├── .env                  # Config (not committed)
├── .gitignore
├── Dockerfile
├── Makefile
├── conftest.py            # pytest root marker
├── tests/
│   └── test_main.py       # 5 isolated tests
└── .github/
    └── workflows/
        └── test.yml       # CI: runs pytest on push
```
*(Split into `routes/ models/ schemas/` folders, and consider a repository+service layer split, only once there are genuinely multiple distinct resources — not just more endpoints on one resource. Mention this as "how I'd evolve it," don't build it under time pressure.)*
```
├── main.py                  # App instance, includes routers, startup
├── config.py                 # Settings from environment variables
├── database.py
├── models/
│   ├── __init__.py
│   ├── sensor_model.py
│   └── reading_model.py
├── schemas/
│   ├── __init__.py
│   ├── sensor_schema.py
│   └── reading_schema.py
├── repositories/              # Raw database queries only — no decisions
│   ├── __init__.py
│   ├── sensor_repository.py
│   └── reading_repository.py
├── services/                  # Business logic, raises ValueError
│   ├── __init__.py
│   ├── sensor_service.py
│   └── reading_service.py
├── api/
│   └── routes/                # Thin layer — calls service, returns schema
│       ├── __init__.py
│       ├── sensor_routes.py
│       └── reading_routes.py
├── core/
│   └── logger.py

**Why this split matters at scale:**
- **Repository** isolates raw queries, so the service layer never touches SQL directly — easier to swap databases, easier to mock in unit tests
- **Service** holds business rules and raises plain `ValueError`s — keeps logic decoupled from HTTP concerns; routes translate errors to the right status code at the boundary
- **Routes** stay thin — just translate HTTP in, call the service, translate the result back out
```
## database.py
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

## models.py — SQLAlchemy (real tables)
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey
from database import Base

class Parent(Base):
    __tablename__ = 'parents'
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False)

class Child(Base):
    __tablename__ = 'children'
    id = Column(Integer, primary_key=True, index=True)
    parent_id = Column(Integer, ForeignKey('parents.id'), nullable=False, index=True)
    value = Column(Float, nullable=False)
    timestamp = Column(DateTime, nullable=False)
```

## schemas.py — Pydantic (API shapes)
```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

class ChildCreate(BaseModel):
    value: float
    timestamp: datetime

class ChildResponse(BaseModel):
    id: int
    value: float
    timestamp: datetime
    class Config:
        from_attributes = True
```

## main.py skeleton
```python
import logging
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from sqlalchemy import func
from database import engine, SessionLocal, Base
import models, schemas

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

Base.metadata.create_all(bind=engine)
app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_parent_or_404(id: int, db: Session):
    obj = db.query(models.Parent).filter(models.Parent.id == id).first()
    if not obj:
        raise HTTPException(status_code=404, detail=f"Parent {id} not found")
    return obj

@app.post("/parents")
def create_parent(data: schemas.SensorCreate, db: Session = Depends(get_db)):
    obj = models.Parent(**data.dict())
    db.add(obj); db.commit(); db.refresh(obj)
    logger.info(f"Created parent {obj.id}")
    return obj

@app.get("/parents/{id}/children")
def list_children(id: int, db: Session = Depends(get_db)):
    get_parent_or_404(id, db)
    return db.query(models.Child).filter(models.Child.parent_id == id).all()

@app.get("/parents/{id}/children/summary")
def summary(id: int, db: Session = Depends(get_db)):
    get_parent_or_404(id, db)
    min_v, max_v, avg_v, count = db.query(
        func.min(models.Child.value), func.max(models.Child.value),
        func.avg(models.Child.value), func.count(models.Child.value)
    ).filter(models.Child.parent_id == id).first()
    return {"min": min_v, "max": max_v, "average": avg_v, "count": count}
```

## Status codes
- **404** — referencing something that doesn't exist
- **422** — the submitted data itself is malformed
- **409** — conflict with existing state
- **200/201** — success (read / created)

## Test template — isolated with in-memory DB (avoids the bug I hit)
```python
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import StaticPool
from database import Base
from main import app, get_db

TEST_DATABASE_URL = "sqlite:///:memory:"
engine = create_engine(TEST_DATABASE_URL, connect_args={"check_same_thread": False}, poolclass=StaticPool)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

def override_get_db():
    db = TestingSessionLocal()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db
client = TestClient(app)

@pytest.fixture(autouse=True)
def setup_db():
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)

# 5 test categories minimum: happy path, validation failure (422),
# not-found (404), business-rule rejection, filter/query behavior
```
*(conftest.py at project root, even empty, is required for pytest to find main.py from tests/)*

## Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

## Makefile
```makefile
run:
	uvicorn main:app --reload
test:
	pytest
build:
	docker build -t app .
```

## requirements.txt — write minimal, don't blindly pip freeze
```
fastapi
uvicorn
sqlalchemy
pytest
httpx
python-dotenv
```

## Phase timing (3 hrs)
Clarify 5m → Design 10m → Scaffold 10m (venv first!) → Core logic 90m → Tests 20m → Logging/config 10m → Deploy 30m → Wrap-up 15m

## Review round structure
1. What I built (1-2 min)
2. Key decisions + tradeoffs, one at a time, with reasoning
3. What I'd add with more time (mention repository/service split, Postgres, auth, pagination)
4. Scaling answer: name the bottleneck → name the fix
5. What I deliberately skipped, with reasoning

## Notes doc — fill in as I build
```
Requirements (my words) / Clarifying Qs / Design decisions + why /
Skipped + why / Issues hit + fixes / What I'd add with more time
```
