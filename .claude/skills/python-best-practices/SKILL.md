---
name: python-best-practices
description: Writing modern, Pythonic code — current type-hint syntax, dataclasses vs Pydantic, async, packaging, and tests
---

# Modern Python

## Type hints (3.12+ syntax)

```python
# Use built-in generics (3.9+), no need for typing.List, typing.Dict
def process_items(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}

# Union with | syntax (3.10+)
def find_user(user_id: int) -> User | None:
    ...

# Type parameter syntax (3.12+)
type Vector[T] = list[T]
type Matrix[T] = list[Vector[T]]

def first[T](items: list[T]) -> T:
    return items[0]

# TypedDict for structured dicts
from typing import TypedDict

class UserResponse(TypedDict):
    id: int
    name: str
    email: str
    active: bool
```

Type every function signature, run `mypy --strict` or `pyright` in CI, and keep `type: ignore` comments rare and justified.

## Dataclasses vs Pydantic

### Dataclasses — internal data, no validation needed
```python
from dataclasses import dataclass, field

@dataclass(frozen=True, slots=True)
class Point:
    x: float
    y: float

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

@dataclass
class Config:
    host: str = "localhost"
    port: int = 8080
    tags: list[str] = field(default_factory=list)
```

Reach for `frozen=True` on immutable value objects and `slots=True` when memory matters.

### Pydantic — external input, validation required
```python
from pydantic import BaseModel, Field, field_validator

class CreateUserRequest(BaseModel):
    model_config = {"strict": True}

    email: str = Field(max_length=255)
    name: str = Field(min_length=1, max_length=100)
    age: int = Field(ge=13, le=150)

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email format")
        return v.lower()
```

The split: dataclasses for domain models and internal structs; Pydantic wherever untrusted data crosses a boundary — API payloads, config files, external parsing.

## Async

```python
import asyncio
import httpx

async def fetch_user(client: httpx.AsyncClient, user_id: int) -> User:
    response = await client.get(f"/users/{user_id}")
    response.raise_for_status()
    return User(**response.json())

async def fetch_all_users(user_ids: list[int]) -> list[User]:
    async with httpx.AsyncClient(base_url="https://api.example.com") as client:
        tasks = [fetch_user(client, uid) for uid in user_ids]
        return await asyncio.gather(*tasks)

async def process_with_semaphore(items: list[str], max_concurrent: int = 10):
    semaphore = asyncio.Semaphore(max_concurrent)
    async def bounded_process(item: str):
        async with semaphore:
            return await process_item(item)
    return await asyncio.gather(*[bounded_process(i) for i in items])
```

Guidelines:
- Use `httpx`, not `requests`, when you need async HTTP
- Fan out with `asyncio.gather`; cap concurrency with `asyncio.Semaphore`
- Keep blocking I/O out of async functions — wrap legacy calls in `asyncio.to_thread`
- Manage connections and sessions with `async with`

## Project layout

```
my-project/
  src/
    my_project/
      __init__.py
      main.py
      models.py
      services/
        __init__.py
        user_service.py
      api/
        __init__.py
        routes.py
  tests/
    conftest.py
    test_models.py
    test_services/
      test_user_service.py
  pyproject.toml
```

A `src` layout stops the project root from leaking onto the import path and masking packaging bugs.

## pyproject.toml

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-project"
version = "1.0.0"
requires-python = ">=3.12"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-cov",
    "pytest-asyncio",
    "mypy",
    "ruff",
]

[project.scripts]
my-project = "my_project.main:cli"

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "RUF"]

[tool.mypy]
strict = true

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

Keep all tool config in `pyproject.toml`, and let Ruff replace the flake8 + isort + black stack — one tool, orders of magnitude faster.

## Virtual environments

```bash
# Use uv for fast dependency management
uv venv
uv pip install -e ".[dev]"

# Or standard venv
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
```

Always work inside a virtual environment, never install globally, and pin exact versions in a lockfile (`uv.lock`, or a `requirements.txt` frozen from `pip freeze`).

## Testing with pytest

```python
import pytest
from unittest.mock import AsyncMock, patch

@pytest.fixture
def user_service(db_session):
    return UserService(session=db_session)

async def test_create_user_returns_user_with_hashed_password(user_service):
    user = await user_service.create(email="test@example.com", password="secret")
    assert user.email == "test@example.com"
    assert user.password_hash != "secret"

async def test_create_user_rejects_duplicate_email(user_service):
    await user_service.create(email="test@example.com", password="secret")
    with pytest.raises(DuplicateEmailError):
        await user_service.create(email="test@example.com", password="other")

@pytest.fixture
def mock_http_client():
    client = AsyncMock(spec=httpx.AsyncClient)
    client.get.return_value = httpx.Response(200, json={"id": 1, "name": "Alice"})
    return client

async def test_fetch_user_parses_response(mock_http_client):
    user = await fetch_user(mock_http_client, user_id=1)
    assert user.name == "Alice"
    mock_http_client.get.assert_called_once_with("/users/1")
```

Share fixtures through `conftest.py`, cover variations with `pytest.mark.parametrize`, and use the `tmp_path` fixture for anything touching the filesystem.

## Pythonic idioms

```python
# Unpacking
first, *rest = items
x, y = point

# Comprehensions over map/filter
squares = [x**2 for x in numbers if x > 0]
lookup = {u.id: u for u in users}

# Context managers for resource cleanup
with open(path) as f:
    data = f.read()

# Walrus operator for assign-and-test
if (match := pattern.search(text)) is not None:
    process(match.group(1))

# Structural pattern matching (3.10+)
match command:
    case {"action": "move", "direction": d}:
        move(d)
    case {"action": "quit"}:
        sys.exit(0)
    case _:
        raise ValueError(f"Unknown command: {command}")
```

## Handling errors

```python
class AppError(Exception):
    def __init__(self, message: str, code: str):
        super().__init__(message)
        self.code = code

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str):
        super().__init__(f"{resource} {id} not found", "NOT_FOUND")

# Specific exceptions, never bare except
try:
    user = await get_user(user_id)
except NotFoundError:
    return {"error": "User not found"}, 404
except DatabaseError as e:
    logger.exception("Database error fetching user")
    return {"error": "Internal error"}, 500
```

Never write a bare `except:` — catch the narrowest exception that fits. Log with `logger.exception()` so the traceback comes along, and give your app its own exception hierarchy to branch on.
