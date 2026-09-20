# Complete Reference — sensor-readings-api
Each section shows: the Copilot prompt to type, the resulting code with explanatory comments, and why key decisions were made.

## database.py

**Prompt to Copilot:** `# SQLAlchemy engine and session setup for a SQLite database called sensor_data.db`

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

SQLALCHEMY_DATABASE_URL = "sqlite:///./sensor_data.db"

# check_same_thread=False is SQLite-specific: SQLite normally restricts a
# connection to one thread, but FastAPI can use multiple threads per request
engine = create_engine(
    SQLALCHEMY_DATABASE_URL, connect_args={"check_same_thread": False}
)

# autocommit=False: changes only save when we explicitly call .commit()
# autoflush=False: don't auto-sync pending changes before every query
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Base is the parent class every SQLAlchemy model inherits from
Base = declarative_base()
```

## models.py

**Prompt to Copilot:** `# SQLAlchemy models: Sensor (id, name, location) and Reading (id, sensor_id foreign key to Sensor, value, unit, timestamp), one-to-many relationship`

```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey
from database import Base

class Sensor(Base):
    __tablename__ = "sensors"
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, nullable=False, unique=True)  # required, no two sensors share a name
    location = Column(String, nullable=True)  # optional field

class Reading(Base):
    __tablename__ = "readings"
    id = Column(Integer, primary_key=True, index=True)
    # foreign key lives on the "many" side — many readings, one sensor each
    sensor_id = Column(Integer, ForeignKey("sensors.id"), nullable=False, index=True)
    value = Column(Float, nullable=False)
    unit = Column(String, nullable=False)
    # DateTime, not String — needed for correct >=/<= time range filtering later
    timestamp = Column(DateTime, nullable=False)
```

## schemas.py

**Prompt to Copilot:** `# Pydantic schemas for Sensor and Reading: Create schemas for incoming requests, Response schemas for outgoing data built from database objects`

```python
from pydantic import BaseModel
from datetime import datetime
from typing import Optional

# Create schemas validate INCOMING data — no Config needed, not built from DB objects
class SensorCreate(BaseModel):
    name: str
    location: Optional[str] = None  # default value is what makes it skippable

# Response schemas shape OUTGOING data — need Config since they're built from
# real SQLAlchemy objects, not plain dicts
class SensorResponse(BaseModel):
    id: int
    name: str
    location: Optional[str] = None
    class Config:
        from_attributes = True  # lets Pydantic read .id, .name off a SQLAlchemy object

class ReadingCreate(BaseModel):
    value: float
    unit: str
    timestamp: datetime

class ReadingResponse(BaseModel):
    id: int
    sensor_id: int
    value: float
    unit: str
    timestamp: datetime
    class Config:
        from_attributes = True

class SummaryResponse(BaseModel):
    sensor_id: int
    min: Optional[float]   # Optional since a sensor with zero readings has no min/max/avg
    max: Optional[float]
    average: Optional[float]
    count: int
```

## main.py

**Prompt to Copilot (used incrementally, one route at a time):**
`# FastAPI app with SQLAlchemy database, Sensor and Reading models, CRUD routes` then, per route:
`# POST /sensors - create a sensor from SensorCreate schema, save to db, return it`
`# POST /sensors/{sensor_id}/readings - create a reading, 404 if sensor doesn't exist`
`# GET /sensors/{sensor_id}/readings - list readings, optional start/end time filters`
`# GET /sensors/{sensor_id}/readings/summary - min/max/avg/count via database aggregation`

```python
import logging
from datetime import datetime
from typing import Optional
from fastapi import FastAPI, Depends, HTTPException
from sqlalchemy.orm import Session
from sqlalchemy import func
from database import engine, SessionLocal, Base
import models, schemas

# Structured logging instead of print() - has levels, usable in real deployments
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Creates all tables fresh on startup - fine since there's no existing data to
# preserve; a real production app with live data would use Alembic migrations instead
Base.metadata.create_all(bind=engine)
app = FastAPI()

# Dependency: opens a session, hands it to the route, closes it after -
# even if the route raises an error, since 'finally' always runs
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Shared logic pulled out since 3 different routes need it - avoids repeating
# the same check-and-raise pattern in every route
def get_sensor_or_404(sensor_id: int, db: Session) -> models.Sensor:
    sensor = db.query(models.Sensor).filter(models.Sensor.id == sensor_id).first()
    if not sensor:
        # 404, not 422: the JSON is well-formed, it's referencing something that
        # doesn't exist - that's a "not found" situation, not a "malformed data" one
        raise HTTPException(status_code=404, detail=f"Sensor {sensor_id} not found")
    return sensor

@app.post("/sensors", response_model=schemas.SensorResponse)
def create_sensor(sensor: schemas.SensorCreate, db: Session = Depends(get_db)):
    db_sensor = models.Sensor(name=sensor.name, location=sensor.location)
    db.add(db_sensor)
    db.commit()
    db.refresh(db_sensor)  # pulls back the auto-generated id from the database
    logger.info(f"Created sensor {db_sensor.id}: {db_sensor.name}")
    return db_sensor  # FastAPI converts this through SensorResponse automatically

@app.post("/sensors/{sensor_id}/readings", response_model=schemas.ReadingResponse)
def create_reading(sensor_id: int, reading: schemas.ReadingCreate, db: Session = Depends(get_db)):
    get_sensor_or_404(sensor_id, db)  # this IS the "reject unregistered sensors" requirement
    db_reading = models.Reading(
        sensor_id=sensor_id, value=reading.value, unit=reading.unit, timestamp=reading.timestamp
    )
    db.add(db_reading)
    db.commit()
    db.refresh(db_reading)
    logger.info(f"Created reading for sensor {sensor_id}: {db_reading.value}{db_reading.unit}")
    return db_reading

@app.get("/sensors/{sensor_id}/readings", response_model=list[schemas.ReadingResponse])
def get_readings(
    sensor_id: int,
    start: Optional[datetime] = None,  # optional query params - filtering is a bonus, not required
    end: Optional[datetime] = None,
    db: Session = Depends(get_db),
):
    get_sensor_or_404(sensor_id, db)
    query = db.query(models.Reading).filter(models.Reading.sensor_id == sensor_id)
    if start:
        query = query.filter(models.Reading.timestamp >= start)
    if end:
        query = query.filter(models.Reading.timestamp <= end)
    return query.order_by(models.Reading.timestamp).all()

@app.get("/sensors/{sensor_id}/readings/summary", response_model=schemas.SummaryResponse)
def get_summary(
    sensor_id: int,
    start: Optional[datetime] = None,
    end: Optional[datetime] = None,
    db: Session = Depends(get_db),
):
    get_sensor_or_404(sensor_id, db)
    # Database calculates min/max/avg directly - never pull thousands of rows
    # into Python memory just to compute these
    query = db.query(
        func.min(models.Reading.value),
        func.max(models.Reading.value),
        func.avg(models.Reading.value),
        func.count(models.Reading.value),
    ).filter(models.Reading.sensor_id == sensor_id)
    if start:
        query = query.filter(models.Reading.timestamp >= start)
    if end:
        query = query.filter(models.Reading.timestamp <= end)

    min_val, max_val, avg_val, count = query.first()
    return schemas.SummaryResponse(
        sensor_id=sensor_id, min=min_val, max=max_val, average=avg_val, count=count
    )
```

## tests/test_main.py

**Prompt to Copilot:** `# pytest tests for the sensor API: valid sensor creation, missing required field (422), reading for unregistered sensor (404), creating and retrieving a reading, summary stats calculation`

```python
import pytest
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_create_sensor():
    # Happy path - valid input should succeed
    response = client.post("/sensors", json={"name": "Test Sensor", "location": "Room 1"})
    assert response.status_code == 200
    assert response.json()["name"] == "Test Sensor"

def test_create_sensor_missing_name():
    # Validation failure - name is required, Pydantic should reject this
    response = client.post("/sensors", json={"location": "Room 1"})
    assert response.status_code == 422

def test_create_reading_unregistered_sensor():
    # Business-rule rejection - this IS the "reject unregistered sensors" requirement
    response = client.post(
        "/sensors/99999/readings",
        json={"value": 23.5, "unit": "celsius", "timestamp": "2026-01-01T00:00:00"},
    )
    assert response.status_code == 404

def test_create_and_get_reading():
    # Create a real sensor first, then confirm a reading attaches to it correctly
    sensor_response = client.post("/sensors", json={"name": "Sensor A"})
    sensor_id = sensor_response.json()["id"]
    client.post(
        f"/sensors/{sensor_id}/readings",
        json={"value": 20.0, "unit": "celsius", "timestamp": "2026-01-01T00:00:00"},
    )
    response = client.get(f"/sensors/{sensor_id}/readings")
    assert response.status_code == 200
    assert len(response.json()) == 1

def test_summary_stats():
    # Confirms the database aggregation actually computes correctly
    sensor_response = client.post("/sensors", json={"name": "Sensor B"})
    sensor_id = sensor_response.json()["id"]
    for value in [10.0, 20.0, 30.0]:
        client.post(
            f"/sensors/{sensor_id}/readings",
            json={"value": value, "unit": "celsius", "timestamp": "2026-01-01T00:00:00"},
        )
    response = client.get(f"/sensors/{sensor_id}/readings/summary")
    data = response.json()
    assert data["min"] == 10.0
    assert data["max"] == 30.0
    assert data["average"] == 20.0
    assert data["count"] == 3
```

## requirements.txt
```
fastapi
uvicorn
sqlalchemy
pytest
httpx
```
*(httpx is needed by FastAPI's TestClient — add if tests error on import)*

## Dockerfile

**Prompt to Copilot:** `# Dockerfile for a FastAPI app using Python 3.11 slim, installs requirements.txt, runs with uvicorn on port 8080`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
# copying requirements.txt and installing BEFORE the rest of the code means
# Docker caches this step - rebuilds are faster if only app code changes later
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
	docker build -t sensor-readings-api .
```

## .gitignore
```
venv/
__pycache__/
*.pyc
.env
*.db
```

## .github/workflows/test.yml (CI/CD)
```yaml
name: Tests
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest
```

## README.md (fill in as you go, per your notes doc habit)
```markdown
# Sensor Readings API
REST API for ingesting and querying IoT sensor readings.

## Architecture
Client -> FastAPI routing -> route function -> SQLAlchemy -> SQLite

## Setup
1. python3 -m venv venv && source venv/bin/activate
2. pip install -r requirements.txt
3. uvicorn main:app --reload
4. Visit http://localhost:8000/docs

## Testing
pytest

## Design Decisions
- SQLite over Postgres: zero setup cost, real persistence, would swap to
  Postgres for production scale
- FastAPI over Flask: automatic validation + docs, faster under time constraint
- Two models (Sensor, Reading) with foreign key: avoids duplicating sensor
  info across every reading row
- Database-side aggregation for stats: avoids loading all rows into memory

## Known Limitations
- No authentication - out of scope for this exercise
- No pagination on readings list - would add if result sets could get large
- Auto-increment IDs, not UUIDs - simpler, no need for unguessable IDs here

## What I'd Add With More Time
- Message queue for high-volume ingestion spikes
- Postgres for real concurrent-write scale
- Rate limiting on submission endpoint
```

---
**Run order to confirm everything works:**
1. `uvicorn main:app --reload`
2. Visit `localhost:8000/docs` — create a sensor, then a reading, then check summary
3. `pytest` — confirm all 5 tests pass
