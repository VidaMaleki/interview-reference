# Interview Day Roadmap

## Pre-decided (never re-derive these)
- **Language/Framework:** Python + FastAPI — automatic validation + docs, faster than Flask under time pressure
- **Storage:** SQLite now, would use Postgres in real production (concurrent writes, scale) — zero setup cost tradeoff
- **Skip unless required:** auth, pagination, rate limiting, Alembic migrations, UUID keys — name each skip + reason if asked
- **Clarifying questions to ask:** exact input format? expected scale? anything ambiguous to state as an assumption?

## Folder structure
```
main.py / models.py / schemas.py / database.py / requirements.txt
tests/test_main.py / .env / .gitignore / Dockerfile / Makefile / README.md
```
Split into `routes/ models/ schemas/` folders only once there are multiple DISTINCT resources — not just many endpoints on one resource.

## database.py template
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

engine = create_engine('sqlite:///app.db')
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
    parent_id = Column(Integer, ForeignKey('parents.id'))
    value = Column(Float)
    timestamp = Column(DateTime)
```
Foreign key always lives on the "many" side. Match column type to real meaning (DateTime not String for timestamps).

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
        from_attributes = True   # needed on Response schemas built from DB objects
```
Optional field = give it a default (`= None`), not a flag. Never `str = None` — use `str | None = None`.

## main.py skeleton
```python
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from database import engine, SessionLocal, Base
import models, schemas

Base.metadata.create_all(bind=engine)
app = FastAPI()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.post("/parents")
def create_parent(data: schemas.SensorCreate, db: Session = Depends(get_db)):
    obj = models.Parent(**data.dict())
    db.add(obj); db.commit(); db.refresh(obj)
    return obj

@app.get("/parents/{id}/children")
def list_children(id: int, db: Session = Depends(get_db)):
    parent = db.query(models.Parent).filter(models.Parent.id == id).first()
    if not parent:
        raise HTTPException(404, f"Parent {id} not found")
    return db.query(models.Child).filter(models.Child.parent_id == id).all()
```

## Status codes cheat sheet
- **404** — referencing something that doesn't exist (e.g., unregistered parent_id)
- **422** — the data itself is malformed (missing field, wrong type)
- **409** — conflict with existing state (duplicate, inactive resource)
- **200/201** — success (200 = OK/read, 201 = created)

## Aggregates — let the DB calculate, never pull all rows into Python
```python
from sqlalchemy import func
db.query(func.min(Model.value), func.max(Model.value), func.avg(Model.value))\
  .filter(...).first()
```

## Dockerfile
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```
Why: predictable, portable, works identically everywhere — even though App Platform can auto-detect Python without it.

## Makefile
```makefile
run:
	uvicorn main:app --reload
test:
	pytest
build:
	docker build -t app .
```

## GitHub Actions CI (.github/workflows/test.yml)
```yaml
name: Tests
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with: {python-version: '3.11'}
      - run: pip install -r requirements.txt
      - run: pytest
```

## Test categories (minimum 5)
1. Happy path (valid input, 200/201)
2. Validation failure (missing field, 422)
3. Not-found case (bad id, 404)
4. Business-rule rejection (the domain-specific rule in the prompt)
5. Filter/query behavior

## Notes doc — fill in as you build
```
## Requirements (my words)
## Clarifying Qs / assumptions
## Design decisions + why
## Skipped + why
## Issues hit + fixes
## What I'd add with more time
```

## Review-round structure (45 min)
1. What I built (1-2 min)
2. Key decisions + tradeoffs, one at a time
3. What I'd add with more time
4. Scaling answer ready: identify current bottleneck → name the fix
5. Name what you deliberately skipped, with reasoning

## Phase time budget (3 hrs)
Clarify 5m → Design 10m → Scaffold 10m → Core logic 90m → Tests 20m → Logging/config 10m → Deploy 30m → Wrap-up 15m
