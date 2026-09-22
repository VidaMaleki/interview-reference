# FULL Interview Day Roadmap

## PHASE 0 — Before the clock starts

1. **Set git identity:** `git config --global user.name "..."` / `user.email "..."`, verify with `git config --global user.name`/`user.email`
2. **Sign into GitHub in VS Code** — Accounts icon (bottom-left), "Sign in with GitHub," authorize in browser
3. **Get the repo** — clone if given a URL, or create fresh on github.com → clone
4. **Open in VS Code** — `cd <folder>`, `code .` (if `code .` fails, open VS Code manually → File → Open Folder)
5. **Confirm git connected** — `git status` inside the project should say "nothing to commit, working tree clean," not "fatal: not a git repository"
6. **Confirm Copilot active** — bottom status bar icon not greyed out; test with a comment + Tab
7. **Log into DigitalOcean separately, in browser** — confirm dashboard + credits show
8. **One test commit + push** to confirm the whole pipeline works, then remove the test file

**If anything's broken:** *"Before I start, I'm noticing [specific issue] — could someone help me get this sorted before the timer starts?"* Normal, not a red flag.

---

## PHASE 1 — Clarify (5-10 min)
- Read the full prompt before touching anything
- Restate it in your own words out loud
- Ask 2-3 clarifying questions: exact input format? expected scale? anything ambiguous → state as an assumption if not answered
- **Watch for scope creep in your own head** — if a "what about X" idea isn't literally in the prompt (payment, carts, inventory, auth), name it as a "future consideration" in notes, don't chase it now

## PHASE 2 — Design (10 min)
- List every "noun" in the prompt → candidate models
- Draw relationships (which belongs to which) → foreign keys, always on the "many" side
- For each requirement bullet, name the HTTP method + URL
- **Pre-decided answers, don't re-derive:**
  - Framework: FastAPI + Python
  - Storage: SQLite (would use Postgres in production — name the reason if asked)
  - Skip unless required: auth, pagination, rate limiting, Alembic, UUID keys
- **Write "DECIDED" next to settled choices in your notes doc — don't re-open them later**

## PHASE 3 — Scaffold (10 min)
- venv FIRST: `python3 -m venv venv && source venv/bin/activate`
- Create files in dependency order: `database.py` → `models.py` → `schemas.py` → `main.py`
- Flat structure (no routes/ services/ folders) unless genuinely multiple distinct resources
- Scaffolding prompt template: *"Generate a FastAPI project using SQLite. Models: [fields+types]. Relationships: [FKs]. Routes (stubs only): [method+URL+description]. Include db setup, requirements.txt. No business logic — just structure. I'll write the logic myself."*

### database.py (copy exactly, just rename the .db file)
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

SQLALCHEMY_DATABASE_URL = "sqlite:///./app.db"
engine = create_engine(SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()
```

### models.py — CHECK: `from database import Base`, never `class Base(DeclarativeBase): pass` redefined locally
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

### schemas.py
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

## PHASE 4 — Core logic (90 min) — WRITE THIS YOURSELF
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
def create_parent(data: schemas.ParentCreate, db: Session = Depends(get_db)):
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
**CHECK: does every model that gets referenced (foreign key target) have its own CREATE endpoint?** Easy to build the "child" endpoints and forget the "parent" one.

**Status codes:** 404 = referencing something that doesn't exist | 422 = malformed data | 409 = conflict | 200/201 = success

**Aggregation:** always `func.min/max/avg/count` + `.group_by()` in the DB query — never pull all rows into Python and loop.

**`func` comes from `from sqlalchemy import func`** — never `db.func`.

## PHASE 5 — Tests (20 min)
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
```
**Two conftest.py files:** root one EMPTY (pytest path marker only), `tests/conftest.py` holds fixtures above.
**`test_main.py` needs:** `from conftest import client` at the top.
**5 categories minimum:** happy path, validation failure (422), not-found/business-rule (404), create-then-retrieve, aggregation correctness.
**Run:** `python3 -m pytest` (more reliable than bare `pytest` if path issues appear)

## PHASE 6 — Logging & config (10 min)
- `logger.info(...)` at key events (created X, updated Y)
- Config via `.env` + `python-dotenv`, with a fallback default: `os.getenv("VAR", "fallback")`

## PHASE 7 — Deploy (30 min)
1. Push to GitHub
2. DigitalOcean → Create → Apps → connect GitHub repo, branch main
3. Confirm Dockerfile detected (or Python auto-detect)
4. **Reduce to 1 container** (default sometimes shows 2)
5. Create App, wait for "Healthy"
6. Visit `<url>/docs` — if it 404s, try `<url>/redoc` instead (known CDN quirk, not a real bug) or `<url>/openapi.json` to confirm the app itself is fine
7. Re-run full manual test sequence against the live URL

**Dockerfile:**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```
**requirements.txt — write minimal by hand, don't blindly `pip freeze`** (caused a real deploy failure once — messy frozen list with unrelated packages):
```
fastapi
uvicorn
sqlalchemy
pytest
httpx
python-dotenv
```

## PHASE 8 — Wrap-up (15 min)
README: architecture line, design decisions + why, known limitations, what you'd add with more time. Final commit + push.

---

## THE 45-MIN REVIEW
1. What I built (1-2 min)
2. Key decisions + tradeoffs, one at a time
3. What I'd add with more time (repository/service split, Postgres, auth, pagination)
4. Scaling: name the bottleneck → name the fix
5. What I deliberately skipped, with reasoning
**When pushed back on:** restate → agree with reasoning, or push back with your own reasoning. Never silently comply.

## THE 30-MIN BEHAVIORAL
Real DO values: **Bold, Fast, Learning, Simple, Love, Community, Proud.** Have a story ready for each, plus: why DigitalOcean/AI infra, a difficult bug, ambiguity, disagreement with a teammate.

---

## Checklist — common pitfalls with this exact stack, check before you hit them
- [ ] Every model with a foreign key pointing to it needs its own CREATE endpoint — check this in Phase 2, not after tests fail
- [ ] `Base` imported from database.py in models.py, never redefined locally
- [ ] `func` from `sqlalchemy`, never `db.func`
- [ ] Two conftest.py, root empty, tests/ has the fixtures
- [ ] `test_main.py` imports `client` explicitly from conftest
- [ ] Once a design decision is made, write "DECIDED" in notes and move on — don't re-open the same debate twice
- [ ] Stay scoped to literally what's asked — no carts, payment, inventory, auth unless the prompt requires it
- [ ] `datetime.now(timezone.utc)` not `datetime.utcnow()` (deprecated) — import `timezone` separately; `datetime.UTC` doesn't exist on the class import, only the module

## Notes doc — fill in as I build

## Requirements (my words)
## Clarifying Qs / assumptions
## Design decisions + why

- Storage: "I used SQLite because [zero setup, real persistence, right for the time budget]. I'd use Postgres for real production concurrent-write scale."

- Scalability: "Right now this handles [current scope] fine. The first bottleneck at higher traffic would be [SQLite's concurrency limits / whatever's actually true]. I'd add [read replica / caching / connection pooling] first, then [next lever] if that wasn't enough."

- Metrics/observability: "I have structured logging now. At real scale, I'd add [Datadog/Prometheus] to track the four golden signals — latency, traffic, errors, saturation — with alerting so issues get caught before customers report them."

- Tech choices generally (any library/tool): "I chose [X] because [specific reason tied to the actual constraint — time, simplicity, what the prompt needed]. The alternative would be [Y], which I'd reach for when [different condition is true]."

- Skipped things (Docker, auth, pagination, etc.): "I [built/skipped] this because [reasoning tied to the actual requirements and time budget]. I'd add it if [the real trigger condition]."

The underlying shape, every single time: what you did → why, tied to a real constraint (not just "it's simpler") → what the alternative is → what condition would make you choose the alternative instead. That fourth part is what makes it sound like judgment rather than a memorized fact — you're naming when you'd flip your decision, not just defending the one you made.
```
Requirements (my words) / Clarifying Qs / Design decisions + why /
Skipped + why / Issues hit + fixes / What I'd add with more time
```
