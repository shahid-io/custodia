# E0 Foundation

Get a runnable service, a database and a test harness in place so every later story
has somewhere to land. Keep these small: later epics deepen each piece.

---

## CUS-001 Settings and a running service

**Points:** 2 · **Depends on:** none · **Concepts:** Pydantic Settings, ASGI lifespan

**User story**
As a developer, I want the service to start with configuration loaded from the
environment, so that the same code runs locally, in tests and in containers.

**Acceptance criteria**
- [ ] `uvicorn app.main:app --reload` starts the service and `GET /health` returns `200 {"status": "ok"}`.
- [ ] Settings (app name, environment, database URL, Redis URL, log level) load from environment variables and a local `.env` file.
- [ ] Starting with a missing required setting fails fast with a clear validation error.
- [ ] `.env` is git-ignored; a `.env.example` documents every variable.

**Technical notes**
- `app/core/config.py`: a `Settings(BaseSettings)` class with `model_config = SettingsConfigDict(env_file=".env")`. Use `PostgresDsn` / `RedisDsn` types so bad URLs fail at startup.
- Expose settings through a cached function (`functools.lru_cache`) so tests can override it.
- `app/main.py`: use the `lifespan` context manager (not the deprecated `on_event`) as the place where engines and clients will be created later.

---

## CUS-002 Local infrastructure with Docker Compose

**Points:** 2 · **Depends on:** CUS-001 · **Concepts:** containers for dependencies

**User story**
As a developer, I want Postgres and Redis running locally with one command, so that
I can develop against real infrastructure instead of mocks.

**Acceptance criteria**
- [ ] `docker compose up -d` starts PostgreSQL 16+ and Redis 7+ with named volumes.
- [ ] Both services define health checks; `docker compose ps` shows them healthy.
- [ ] The default `.env.example` values connect to these containers.
- [ ] `app/core/database.py` creates an async engine and an `async_sessionmaker`, and `GET /health` is unaffected if you have not wired the DB check yet (that comes in CUS-506).

**Technical notes**
- Database URL form: `postgresql+asyncpg://user:pass@localhost:5432/custodia`.
- `async_sessionmaker(engine, expire_on_commit=False)`: explain to yourself why `expire_on_commit=False` matters in async code (lazy loads after commit would need I/O).
- Create the engine in `lifespan` and dispose of it on shutdown.

---

## CUS-003 Test harness

**Points:** 3 · **Depends on:** CUS-001 · **Concepts:** pytest, async tests, test pyramid

**User story**
As a developer, I want a test setup with unit, integration and e2e layers, so that I
can test each architectural layer at the right cost.

**Acceptance criteria**
- [ ] `pytest tests/unit` runs with no database or network.
- [ ] An e2e test calls `GET /health` through `httpx.AsyncClient` with `ASGITransport` and passes.
- [ ] Shared fixtures live in `tests/conftest.py`.
- [ ] Markers or directories make it possible to run each layer on its own.

**Technical notes**
- `pytest-asyncio` with `asyncio_mode = "auto"` is already configured in `pyproject.toml`.
- Write a reusable `client` fixture now; later stories add `app.dependency_overrides` to it.
- Integration tests get a real Postgres in CUS-208. Until then, keep them minimal.
