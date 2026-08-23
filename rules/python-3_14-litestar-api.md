---
type: "agent_requested"
description: "Python 3.14 + Litestar + Granian + msgspec coding guidelines"
---

# Litestar on Python 3.14: An Idiomatic, msgspec-First API Reference

Litestar is a batteries-included, class-first ASGI framework whose defining trait is that it is **msgspec-native**: request parsing, response serialization, and OpenAPI 3.1 generation all flow through msgspec's C-accelerated codecs, not Pydantic. On this stack you optimize for three things: (1) letting Litestar resolve types at runtime from real annotations — so annotations must stay importable, (2) keeping the async event loop unblocked so Granian's Rust/Tokio I/O layer can do its job, and (3) declaring everything with types so the layered DI and OpenAPI machinery has something to work with. Granian replaces the uvicorn/gunicorn stack with a single Rust binary; `litestar-granian` wires it into the `litestar run` CLI so you never invoke it manually in the common case.

The single most common way agents write wrong-but-plausible code here is by importing FastAPI/Pydantic/Starlette/uvicorn habits: reaching for `pydantic.BaseModel` instead of `msgspec.Struct`, `Depends()` instead of `Provide`, `APIRouter` instead of `Controller`/`Router`, `uvicorn.run()` instead of the Granian plugin, and — most destructively — letting a linter move runtime-needed imports into `if TYPE_CHECKING:` blocks, which breaks Litestar's and msgspec's runtime type resolution. This document shows the correct idiom once, densely, so you can pattern-match to good code.

---

## Stack snapshot

- **Research date:** 2026-08-22
- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories.

| Component | Version at research time | Role |
|---|---|---|
| Python | 3.14.7 (3.14.0 released 2025-10-07; 3.14.7 on 2026-08-05 per PEP 745) | Runtime; deferred annotations (PEP 649/749) |
| Litestar | 2.24.0 | ASGI framework |
| Granian | 2.8.1 | Rust/Tokio HTTP server (ASGI/RSGI/WSGI) |
| litestar-granian | 0.16.0 | Wires Granian into `litestar run` |
| msgspec | 0.21.x | Serialization + validation (Structs) |
| uv | 0.12.x | Package/project manager |
| Ruff | 0.16.x | Linter + formatter (one tool) |
| basedpyright | 1.39.x | Type checker (`recommended` mode) |
| pytest | 9.1.x | Test runner |
| pytest-asyncio | 1.4.x | Async test support (or anyio plugin) |
| Advanced Alchemy | 1.11.x | SQLAlchemy 2.x repository/service + Litestar plugin |
| Polyfactory | 3.3.x | Test data factories (msgspec-aware) |

Critical version-interaction: **Python 3.14 defers annotation evaluation by default (PEP 649/749).** Per the Python 3.14.0 release announcement, "PEP 649: The evaluation of annotations is now deferred." Litestar and msgspec both introspect annotations *at runtime* to build validators, DI graphs, and OpenAPI. This works because annotations remain resolvable lazily — but only if the referenced names are still importable at module scope. Do **not** add `from __future__ import annotations` (unnecessary on 3.14) and do **not** let tooling hide runtime-needed imports.

---

## Python 3.14 language baseline

Write modern typed Python and lean on 3.14 features that matter for a typed async API.

```python
# Modern typing: builtins as generics, X | None unions, no typing.Optional/List/Dict
def totals(rows: list[dict[str, int]], default: int | None = None) -> dict[str, int]:
    ...

# PEP 695 type parameter syntax (Python 3.12+) — no TypeVar boilerplate
class Repository[T]:
    def get(self, id: int) -> T | None: ...

def first[T](items: list[T]) -> T | None:
    return items[0] if items else None

# PEP 695 type aliases
type UserId = int
type JSONValue = str | int | float | bool | None | list["JSONValue"] | dict[str, "JSONValue"]
```

Key 3.14 features and how they interact with this stack:

- **Deferred annotation evaluation (PEP 649/749, Python 3.14).** Annotations are stored as lazy `annotate` functions and evaluated on demand via `annotationlib`. You get forward references "for free" without quoting and without `from __future__ import annotations`. Litestar/msgspec resolve these at app-construction time, so keep annotation names importable at runtime.
- **`Self` and `@override` (Python 3.12 / `typing.override` PEP 698).** Use `@override` on methods that override a base — basedpyright `recommended` reports missing overrides.

```python
from typing import Self, override
from litestar import Controller

class BaseController(Controller):
    def tag(self) -> Self:
        return self

class UserController(BaseController):
    @override
    def tag(self) -> Self:
        return self
```

- **`TypeIs` (PEP 742, Python 3.13) / `TypeGuard`.** Prefer `TypeIs` for narrowing that also narrows the negative branch.
- **Exception groups & `except*` (Python 3.11).** Relevant for structured concurrency (anyio task groups) which Litestar uses internally.
- **t-strings / template strings (PEP 750, Python 3.14).** A `t"..."` produces a `Template`, not a `str`. Useful for safe interpolation layers; do **not** pass a `Template` where a `str` is expected.
- **Free-threaded build (PEP 779, officially supported as of Python 3.14.0).** The 3.14 release notes state "PEP 779: Free-threaded Python is officially supported." Treat it as advanced; Granian ships free-threaded wheels (workers become threads in one interpreter) but explicitly warns against production use (see Granian section).
- **Subinterpreters (`concurrent.interpreters`, PEP 734).** Standard-library now, but not part of the normal request path here — Granian owns process/worker parallelism.
- **uuid v7 (`uuid.uuid7()`, Python 3.14)** — time-ordered UUIDs, ideal for DB primary keys (Advanced Alchemy also ships UUIDv7 bases).

---

## Application architecture: the layered app

Litestar is class-first and *layered*. Configuration (dependencies, guards, middleware, DTOs, `opt`, exception handlers, parameters) can be declared at four layers — **app → router → controller → handler** — and the most specific layer wins. This is the single biggest structural difference from FastAPI.

Use `Controller` classes to group routes sharing a path prefix, dependencies, or guards. Use function handlers for one-offs. Use `Router` to compose controllers under a common prefix.

```python
# app/controllers/users.py
from uuid import UUID

from litestar import Controller, get, post, patch, delete
from litestar.di import Provide
from litestar.params import Parameter

from app.domain.users import UserService, User, CreateUser


class UserController(Controller):
    path = "/users"
    tags = ["users"]
    dependencies = {"users": Provide(UserService.new)}  # controller-layer DI

    @get()
    async def list_users(
        self,
        users: UserService,
        limit: int = Parameter(default=20, ge=1, le=100),
    ) -> list[User]:
        return await users.list(limit=limit)

    @get("/{user_id:uuid}")
    async def get_user(self, user_id: UUID, users: UserService) -> User:
        return await users.get(user_id)

    @post()
    async def create_user(self, data: CreateUser, users: UserService) -> User:
        return await users.create(data)

    @delete("/{user_id:uuid}")
    async def delete_user(self, user_id: UUID, users: UserService) -> None:
        await users.delete(user_id)
```

The app factory is the composition root. Autodiscovery: the `litestar` CLI looks for an app in `app.py`, `asgi.py`, or `application.py` (or an app factory), and `--app path:obj` / `--app-dir` override it.

```python
# app/asgi.py
from litestar import Litestar
from litestar.config.cors import CORSConfig
from litestar.openapi import OpenAPIConfig
from litestar.openapi.plugins import ScalarRenderPlugin
from litestar_granian import GranianPlugin

from app.controllers.users import UserController
from app.config import settings, lifespan


def create_app() -> Litestar:
    return Litestar(
        route_handlers=[UserController],
        plugins=[GranianPlugin()],
        lifespan=[lifespan],
        cors_config=CORSConfig(allow_origins=settings.cors_origins),
        openapi_config=OpenAPIConfig(
            title="Example API",
            version="1.0.0",
            render_plugins=[ScalarRenderPlugin()],  # Scalar (added 2.8.0); default UI in 3.0
        ),
        debug=settings.debug,
    )


app = create_app()
```

Path parameter typing uses the `{name:type}` syntax (`:uuid`, `:int`, `:str`, `:float`, `:path`, `:date`, `:decimal`). Never do FastAPI-style bare `{user_id}` — the type suffix is required for parsing and OpenAPI. The OpenAPI UI plugins (`ScalarRenderPlugin`, `SwaggerRenderPlugin`, `RedocRenderPlugin`, `RapidocRenderPlugin`, `StoplightRenderPlugin`) were added in 2.8.0; Scalar becomes the sole default in 3.0.

---

## Data layer: msgspec Structs, not Pydantic

`msgspec.Struct` is the idiomatic model. Per msgspec's official benchmarks, Structs are "roughly 4x faster than standard classes/attrs/dataclasses, and 17x faster than pydantic" to create, "roughly 4x to 30x faster" for equality comparison, and "roughly 5x to 60x faster than the alternatives" for order comparison — and they are what Litestar serializes natively. Reach for a Struct first; reach for Pydantic only when integrating an existing Pydantic codebase.

```python
# app/domain/users.py
from typing import Annotated
from uuid import UUID

import msgspec
from msgspec import Struct, Meta, field


# Constrained scalar types via Annotated + Meta
Email = Annotated[str, Meta(min_length=5, max_length=254, pattern=r"[^@]+@[^@]+\.[^@]+")]
Age = Annotated[int, Meta(ge=0, le=130)]


class User(Struct, kw_only=True, frozen=True):
    id: UUID
    name: Annotated[str, Meta(min_length=1, max_length=100)]
    email: Email
    age: Age
    roles: list[str] = field(default_factory=list)


class CreateUser(Struct, kw_only=True, forbid_unknown_fields=True):
    name: Annotated[str, Meta(min_length=1, max_length=100)]
    email: Email
    age: Age
```

Struct options you should use deliberately:

| Option | When to use |
|---|---|
| `kw_only=True` | Almost always for API models — avoids positional-arg fragility and lets you order fields freely |
| `frozen=True` | Immutable value objects / response models; makes them hashable |
| `forbid_unknown_fields=True` | Request bodies where extra keys should 400 rather than be silently dropped |
| `omit_defaults=True` | Response models where you want to shrink payloads by dropping default-valued fields |
| `rename="camel"` | Emit/accept camelCase JSON while keeping snake_case Python (also `"kebab"`, `"pascal"`, `"upper"`, `"lower"`, or a dict/callable) |
| `tag` / `tag_field` | Tagged unions (discriminated unions) for polymorphic payloads |
| `gc=False` | Hot-path structs with no reference cycles. Per msgspec's benchmarks this gives "the lowest GC pause (75x faster than standard classes!)" and reduces memory "by 16 bytes per instance" |

`Meta` constraints (validated at decode time): `gt`, `ge`, `lt`, `le`, `multiple_of` (numbers); `min_length`, `max_length`, `pattern` (strings/collections; `pattern` is str-only and unanchored — uses `re.search`); `tz` (datetime/time aware-vs-naive); plus schema metadata `title`, `description`, `examples`. Note: `pattern`/`min_length` cannot be set on an optional (`str | None`) directly — constrain the inner type.

**PATCH semantics** use `msgspec.UNSET` to distinguish "field absent" from "explicit null":

```python
from msgspec import Struct, UnsetType, UNSET

class UpdateUser(Struct, kw_only=True):
    name: str | UnsetType = UNSET
    email: str | None | UnsetType = UNSET  # UNSET = untouched, None = clear it

# During encode, UNSET fields are omitted; during decode, missing fields become UNSET.
```

Tagged unions for polymorphism:

```python
class Cat(Struct, tag="cat"):
    name: str
    lives: int = 9

class Dog(Struct, tag="dog"):
    name: str
    good_boy: bool = True

Pet = Cat | Dog  # msgspec discriminates on the "type" tag field automatically
```

Direct codec use when you need raw speed or non-handler serialization:

```python
import msgspec

encoder = msgspec.json.Encoder()
decoder = msgspec.json.Decoder(type=list[User])

raw: bytes = encoder.encode(user)                 # fast path, reuse the encoder
users: list[User] = decoder.decode(raw)           # typed, validated decode

# Lenient coercion / builtin conversion
msgspec.convert({"id": "...", "name": "x"}, type=User, strict=False)
msgspec.to_builtins(user)                          # Struct -> dict/list of builtins

# Custom types via hooks
def dec_hook(type_: type, obj: object) -> object:
    if type_ is complex:
        return complex(*obj)
    raise NotImplementedError

decoder = msgspec.json.Decoder(type=Thing, dec_hook=dec_hook)
```

`msgspec.msgpack`, `msgspec.yaml`, and `msgspec.toml` share the same API — use msgpack for internal service-to-service payloads. Use `msgspec.Raw` for zero-copy deferred parsing of sub-documents.

---

## DTOs: when a Struct alone is not enough

A Struct is your model. A **DTO** is a *transform* between your model and the wire — for excluding fields, renaming, partial updates, or serializing ORM objects. Litestar ships `MsgspecDTO`, `DataclassDTO`, `AttrsDTO`, and (via Advanced Alchemy) `SQLAlchemyDTO`. Use `dto=` for inbound, `return_dto=` for outbound; either can be set at any layer.

```python
from uuid import UUID
from litestar import post, patch
from litestar.dto import MsgspecDTO, DTOConfig, DTOData


class UserReadDTO(MsgspecDTO[User]):
    config = DTOConfig(exclude={"email"}, rename_strategy="camel")


class UserWriteDTO(MsgspecDTO[CreateUser]):
    config = DTOConfig(forbid_unknown_fields=True)


class UserPatchDTO(MsgspecDTO[User]):
    config = DTOConfig(exclude={"id"}, partial=True)  # all remaining fields optional


@post("/users", dto=UserWriteDTO, return_dto=UserReadDTO)
async def create_user(data: CreateUser) -> User:
    ...

@patch("/users/{user_id:uuid}", dto=UserPatchDTO)
async def patch_user(user_id: UUID, data: DTOData[User], users: UserService) -> User:
    # DTOData defers instantiation so you can merge onto an existing row
    existing = await users.get(user_id)
    return data.update_instance(existing)
```

`DTOConfig` knobs: `include` / `exclude` (dotted paths like `"address.street"` and `"children.0.email"` reach into nested models), `rename_fields` (per-field map), `rename_strategy`, `max_nested_depth`, `partial`. Decision: **no transform needed → return the Struct directly; transform needed → add a DTO.** Do not wrap every handler in a DTO by reflex — a plain Struct return is simpler and faster.

---

## Dependency injection: `Provide`, not `Depends`

Litestar DI centers on `Provide`, declared in a `dependencies={}` mapping at any layer. The dependency name is the key; handlers/other dependencies receive it by matching parameter name. There is no `Depends()` default-value marker as in FastAPI.

```python
from litestar import get
from litestar.di import Provide


async def provide_db() -> AsyncSession: ...          # async callable

def provide_settings() -> Settings:                  # sync, non-blocking
    return settings


@get(
    "/report",
    dependencies={
        "db": Provide(provide_db),
        "cfg": Provide(provide_settings, sync_to_thread=False, use_cache=True),
    },
)
async def report(db: AsyncSession, cfg: Settings) -> dict[str, str]:
    ...
```

Rules that matter:

- **`sync_to_thread`** — for a *synchronous* dependency (or handler) you must decide: `sync_to_thread=True` offloads blocking work to a threadpool (keeps the loop free); `sync_to_thread=False` asserts it's non-blocking. Litestar warns if you omit it on a sync callable. Async callables need neither.
- **`use_cache=True`** — memoizes the return value *across requests* (naive cache, no kwarg comparison). Use for expensive singletons (a configured client). It does **not** mean "once per request" — dependencies are already computed once per request regardless.
- **Generator dependencies** provide teardown: `yield` the value, clean up after. Prefer them for sessions/connections.

```python
from collections.abc import AsyncGenerator

async def provide_session(db_engine: AsyncEngine) -> AsyncGenerator[AsyncSession, None]:
    async with AsyncSession(db_engine) as session:
        yield session
        # runs after the handler returns (commit/rollback/close)
```

- **`Dependency()` marker** documents/validates an injected value and can `skip_validation=True`.
- **Override in tests** by passing `dependencies=` to the test client (see Testing).

---

## Parameters, request bodies, and responses

Query/path/header/cookie params are declared as typed handler arguments; `Parameter` adds constraints and metadata. `Body` configures the request body (media type, examples). Bare body Structs are parsed from JSON by default.

```python
from typing import Annotated
from litestar import get, post
from litestar.enums import RequestEncodingType
from litestar.params import Parameter, Body


@get("/search")
async def search(
    q: Annotated[str, Parameter(min_length=1, max_length=64)],
    page: int = 1,
    x_trace: Annotated[str | None, Parameter(header="X-Trace-Id")] = None,
) -> list[User]:
    ...


@post("/upload")
async def upload(
    data: Annotated[dict, Body(media_type=RequestEncodingType.MULTI_PART)],
) -> None:
    ...
```

Responses: return the Struct/DTO directly for JSON. Use explicit response types when you need control.

```python
from litestar import get, post
from litestar.response import Response, Stream, File, Redirect, Template, ServerSentEvent
from litestar.status_codes import HTTP_201_CREATED
from litestar.background_tasks import BackgroundTask


@get("/download")
async def download() -> File:
    return File(path="/data/report.csv", filename="report.csv")


@post("/items", status_code=HTTP_201_CREATED)
async def make_item(data: CreateItem) -> Response[Item]:
    return Response(Item(...), background=BackgroundTask(notify, item_id))


@get("/events")
async def events() -> ServerSentEvent:
    async def gen():
        yield "tick"
    return ServerSentEvent(gen())
```

Pagination uses the built-in containers: `OffsetPagination[T]`, `ClassicPagination[T]`, `CursorPagination[K, T]` — return them from handlers for correctly-typed OpenAPI.

---

## Exceptions, guards, and authentication

Raise Litestar's HTTP exceptions; register handlers per layer via `exception_handlers`.

```python
from litestar import Request, Response
from litestar.exceptions import NotFoundException, ClientException, ValidationException
from litestar.plugins.problem_details import ProblemDetailsPlugin  # RFC 9457


async def not_found_handler(request: Request, exc: NotFoundException) -> Response:
    return Response({"detail": exc.detail}, status_code=exc.status_code)
```

**Guards** are authorization callables `(connection, handler) -> None` that raise on denial; attach at any layer and pass data via the `opt` dict.

```python
from litestar import post
from litestar.connection import ASGIConnection
from litestar.handlers.base import BaseRouteHandler
from litestar.exceptions import PermissionDeniedException


def requires_admin(connection: ASGIConnection, handler: BaseRouteHandler) -> None:
    if not connection.user.is_admin:
        raise PermissionDeniedException()


@post("/admin/users", guards=[requires_admin])
async def create_user(data: CreateUser) -> User: ...
```

**JWT auth** lives in `litestar.security.jwt`. Choose `JWTAuth`, `JWTCookieAuth`, or `OAuth2PasswordBearerAuth`; each is an init-plugin that installs middleware, wires OpenAPI security, and exposes a `.login()` helper. Provide a `retrieve_user_handler` and `exclude` public paths.

```python
import os
from typing import Any
from uuid import UUID

from litestar import Litestar, post, Response
from litestar.connection import ASGIConnection
from litestar.security.jwt import OAuth2PasswordBearerAuth, OAuth2Login, Token
from msgspec import Struct


class User(Struct):
    id: UUID
    email: str


async def retrieve_user_handler(
    token: Token, connection: ASGIConnection[Any, Any, Any, Any]
) -> User | None:
    return await lookup_user(token.sub)


oauth2_auth = OAuth2PasswordBearerAuth[User](
    retrieve_user_handler=retrieve_user_handler,
    token_secret=os.environ["JWT_SECRET"],
    token_url="/login",
    exclude=["/login", "/schema"],
)


@post("/login")
async def login(data: Credentials) -> Response[OAuth2Login]:
    user = await authenticate(data)
    return oauth2_auth.login(identifier=str(user.id))


app = Litestar(route_handlers=[login], on_app_init=[oauth2_auth.on_app_init])
```

The authenticated principal is available as `connection.user` / `request.user`.

---

## Lifecycle, state, and middleware

Use a `lifespan` async context manager for startup/shutdown resource management (preferred over `on_startup`/`on_shutdown`). Store shared resources on typed `app.state`.

```python
from collections.abc import AsyncGenerator
from contextlib import asynccontextmanager
from litestar import Litestar
from sqlalchemy.ext.asyncio import create_async_engine


@asynccontextmanager
async def lifespan(app: Litestar) -> AsyncGenerator[None, None]:
    engine = create_async_engine(settings.database_url)
    app.state.engine = engine
    try:
        yield
    finally:
        await engine.dispose()
```

Middleware: prefer config-driven built-ins (`CORSConfig`, `CSRFConfig`, `CompressionConfig`, `RateLimitConfig`, `AllowedHostsConfig`, logging middleware) over hand-rolled classes. For custom logic subclass `AbstractMiddleware` or wrap with `DefineMiddleware`.

```python
from litestar import Litestar
from litestar.config.compression import CompressionConfig
from litestar.middleware.rate_limit import RateLimitConfig

rate_limit = RateLimitConfig(rate_limit=("minute", 100))
app = Litestar(
    compression_config=CompressionConfig(backend="gzip"),
    middleware=[rate_limit.middleware],
)
```

Structured logging with structlog:

```python
from litestar import Litestar
from litestar.logging import StructLoggingConfig
from litestar.plugins.structlog import StructlogPlugin

app = Litestar(plugins=[StructlogPlugin()], logging_config=StructLoggingConfig())
```

Channels (websockets/pub-sub) use the `ChannelsPlugin` with an in-memory or Redis backend; simple socket endpoints use `@websocket_listener` / `WebsocketListener`.

---

## Database access: Advanced Alchemy

SQLAlchemy integration is a **separate package**, `advanced-alchemy` — not bundled in Litestar core. It provides audit base classes, async repository + service layers, a session dependency, and Alembic wiring. Import the Litestar plugin from `advanced_alchemy.extensions.litestar` (or the re-export at `litestar.plugins.sqlalchemy`).

```python
from advanced_alchemy.extensions.litestar import (
    SQLAlchemyAsyncConfig, SQLAlchemyPlugin, AsyncSessionConfig, base,
)
from advanced_alchemy.repository import SQLAlchemyAsyncRepository
from advanced_alchemy.service import SQLAlchemyAsyncRepositoryService
from litestar import Litestar
from sqlalchemy.orm import Mapped, mapped_column


class UserModel(base.UUIDAuditBase):          # id (UUID) + created_at/updated_at
    __tablename__ = "user_account"
    email: Mapped[str] = mapped_column(unique=True)
    name: Mapped[str]


class UserRepository(SQLAlchemyAsyncRepository[UserModel]):
    model_type = UserModel


class UserService(SQLAlchemyAsyncRepositoryService[UserModel]):
    repository_type = UserRepository


alchemy = SQLAlchemyPlugin(
    config=SQLAlchemyAsyncConfig(
        connection_string=settings.database_url,
        before_send_handler="autocommit",     # commit on 2xx, rollback otherwise
        session_config=AsyncSessionConfig(expire_on_commit=False),
        create_all=True,                       # dev only; use Alembic in prod
    ),
)

app = Litestar(route_handlers=[UserController], plugins=[alchemy])
```

Available audit bases: `UUIDBase`, `UUIDAuditBase`, `UUIDv7AuditBase`, `BigIntAuditBase`, `NanoidAuditBase`. The session is injected under the key `db_session` by default (configurable via `session_dependency_key`). Serialize ORM rows with `SQLAlchemyDTO` from `advanced_alchemy.extensions.litestar.dto`. Migrations run through the Advanced Alchemy CLI (Alembic integration), not raw `alembic`.

---

## Granian: the server

Granian is a single Rust/Tokio binary serving ASGI, RSGI, and WSGI, replacing the gunicorn+uvicorn+httptools stack. Litestar is ASGI, so always `--interface asgi`. In the common case you never call Granian directly — the plugin drives it (next section) — but you must understand its model and flags for production tuning.

Threading model (this differs fundamentally from uvicorn/gunicorn):

- **workers** — OS processes, each a full Python interpreter running the app.
- **runtime threads** (`--runtime-threads`) — Rust I/O threads per worker.
- **blocking threads** (`--blocking-threads`) — Rust threads for blocking ops (primarily WSGI).
- **runtime mode** (`--runtime-mode [auto|st|mt]`) — per the Granian README, "in st mode Granian will spawn N single-threaded Rust runtimes, while in mt mode Granian will spawn a single multi-threaded runtime with N threads." `st` is best with few workers; `mt` scales on many cores; `auto` decides.

Do not port uvicorn/gunicorn worker×thread formulas — Granian's architecture is different. For worker count, the Granian README advises: "matching the amount of CPU cores for the workers is generally the best starting point; on containerized environments like docker or k8s is best to have 1 worker per container though and scale your containers using the relevant orchestrator."

Direct invocation (reference — prefer the plugin):

```bash
granian --interface asgi \
  --host 0.0.0.0 --port 8000 \
  --workers 4 \
  --runtime-mode mt \
  --loop uvloop \
  --http auto \
  --backpressure 256 \
  --respawn-failed-workers \
  --access-log \
  app.asgi:app
```

Key flags (verified against Granian 2.8 `--help`): `--interface [asgi|asginl|rsgi|wsgi]`, `--http [auto|1|2]`, `--ws/--no-ws`, `--workers`, `--runtime-threads`, `--blocking-threads`, `--runtime-mode [auto|st|mt]`, `--loop [auto|asyncio|uvloop|rloop]`, `--backpressure`, `--backlog`, `--respawn-failed-workers`/`--respawn-interval`, `--workers-lifetime`, `--access-log`/`--access-log-fmt`, `--log-level`, `--ssl-certificate FILE`, `--ssl-keyfile FILE`. Note the exact spellings — it is `--runtime-threads` (not `--threads`), `--runtime-mode` (not `--threading-mode`), and `--ssl-certificate` (not `--ssl-certfile`). All flags have `GRANIAN_*` env equivalents. **Critical insight:** set `--backpressure` in production — without a bound, traffic spikes queue unboundedly and OOM the worker.

Never `async def` a handler and then call blocking sync I/O inside it — that stalls the worker's event loop and starves Granian's whole concurrency model. Push blocking calls to `sync_to_thread=True` dependencies or async drivers.

**Free-threaded caveat:** the Granian README states, "Warning: free-threaded Python support is still experimental and highly discouraged in production environments." Ship on the standard GIL build for production.

---

## litestar-granian: the plugin

`GranianPlugin` makes `litestar run` start Granian instead of uvicorn, while keeping Litestar's lifespan, signal handling, dev-reload, and logging integration. Per the plugin's README, "GranianPlugin() replaces Litestar's standard run command with a Granian-backed command. It does not change commands that start another ASGI server directly." This is the idiomatic way to run a Litestar app on Granian — **use the plugin, not a bare `granian` invocation**, and never mix the two (the plugin owns the server lifecycle).

```python
from litestar import Litestar, get
from litestar_granian import GranianPlugin

@get("/")
async def hello() -> dict[str, str]:
    return {"hello": "world"}

app = Litestar(route_handlers=[hello], plugins=[GranianPlugin()])
```

```bash
litestar --app app.asgi:app run                 # dev
litestar --app app.asgi:app run --reload        # dev with autoreload
litestar --app app.asgi:app run --wc 4 --host 0.0.0.0 --port 8000   # prod-style
```

The plugin surfaces Granian's tuning (workers, HTTP version, SSL, access logging) through the `litestar run` command and its `GRANIAN_*` env vars. Event-loop extras are opt-in: install `litestar-granian[uvloop]` (or `[rloop]`) and pass `--loop uvloop`. Other `litestar` CLI commands stay useful: `litestar routes`, `litestar schema` (export OpenAPI), `litestar version`.

---

## Tooling: uv

uv is the package/project manager — not pip, poetry, pdm, pipenv, or conda. Commit `uv.lock`. Use `[dependency-groups]` (PEP 735) for dev deps, **not** the legacy `[tool.uv] dev-dependencies` (deprecated, warns in current uv).

```toml
# pyproject.toml
[project]
name = "example-api"
version = "1.0.0"
requires-python = ">=3.14"
dependencies = [
    "litestar[standard]>=2.24",
    "granian>=2.8",
    "litestar-granian>=0.16",
    "msgspec>=0.21",
    "advanced-alchemy>=1.11",
    "uvloop>=0.21; sys_platform != 'win32'",
]

[dependency-groups]
dev = [
    "pytest>=9.1",
    "pytest-asyncio>=1.4",
    "pytest-cov",
    "polyfactory>=3.3",
    "ruff>=0.16",
    "basedpyright>=1.39",
]

[tool.uv]
required-version = ">=0.12"
package = false          # application, not a distributable library
```

Everyday commands:

```bash
uv sync                    # create/refresh the env from the lockfile
uv add litestar            # add a dependency (updates pyproject + lock + env)
uv add --group dev pytest  # add to a group
uv lock                    # regenerate the lockfile
uv run litestar run        # run inside the project env
uv run pytest              # run tests
uv python install 3.14     # install a managed interpreter
uv python pin 3.14         # write .python-version
```

`uv pip ...` is a compatibility escape hatch, not the project workflow. Multi-stage Docker with cache mounts:

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.14-bookworm-slim AS build
WORKDIR /app
ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project
COPY . .
RUN --mount=type=cache,target=/root/.cache/uv uv sync --frozen --no-dev

FROM python:3.14-slim-bookworm
WORKDIR /app
COPY --from=build /app /app
ENV PATH="/app/.venv/bin:$PATH"
CMD ["litestar", "--app", "app.asgi:app", "run", "--host", "0.0.0.0", "--port", "8000", "--wc", "4"]
```

Use `--frozen` in CI/Docker so a stale lockfile fails the build instead of silently resolving.

---

## Tooling: Ruff (lint + format)

Ruff is both linter and formatter — it replaces black, isort, flake8, pyupgrade, and bandit. **The single most important Ruff setting on this stack is the flake8-type-checking (TC) escape valve.** The TC rules move imports into `if TYPE_CHECKING:` to speed startup — but Litestar and msgspec resolve annotations at *runtime*, so a hidden import produces a `NameError` at app construction. Register your runtime-introspected base classes and decorators so Ruff leaves those imports at module scope.

```toml
[tool.ruff]
line-length = 100
target-version = "py314"
src = ["src", "app"]

[tool.ruff.lint]
select = [
    "E", "W", "F", "I", "N", "UP", "B", "A", "C4", "DTZ", "T20", "SIM",
    "TID", "TC", "RUF", "ASYNC", "S", "PT", "PL", "RET", "PTH", "EM", "TRY",
]
ignore = ["E501"]  # formatter owns line length

[tool.ruff.lint.flake8-type-checking]
# CRITICAL: keep runtime-introspected imports out of TYPE_CHECKING blocks
runtime-evaluated-base-classes = [
    "msgspec.Struct",
    "litestar.dto.MsgspecDTO",
    "sqlalchemy.orm.DeclarativeBase",
    "advanced_alchemy.base.UUIDAuditBase",
    "advanced_alchemy.base.UUIDBase",
]
runtime-evaluated-decorators = ["litestar.get", "litestar.post"]

[tool.ruff.lint.per-file-ignores]
"tests/**/*" = ["S101", "PLR2004"]   # allow assert & magic values in tests
"__init__.py" = ["F401"]             # re-exports

[tool.ruff.lint.isort]
known-first-party = ["app"]

[tool.ruff.format]
quote-style = "double"
docstring-code-format = true
```

Commands: `ruff check --fix` and `ruff format`. In CI use `ruff check` and `ruff format --check`. Because Ruff does not resolve base classes across files, you may still need an occasional `# noqa: TC001`; prefer registering the base class over sprinkling noqas.

---

## Tooling: basedpyright (`recommended` mode)

basedpyright is a pyright fork with stricter defaults and extra rules; it is the type checker here — **not mypy** (mypy is the adjacent-ecosystem choice this stack does not use). `typeCheckingMode = "recommended"` is basedpyright's own mode above `strict`. Per the basedpyright docs, "'recommended' enables all diagnostic rules as either 'warning' or 'error', but sets failOnWarnings to true so that all diagnostics will still cause a non-zero exit code when run in the CLI" — so it is essentially "all" but distinguishes runtime-crash risks (errors) from missing-annotation noise (warnings), while still failing CI on either.

```toml
[tool.basedpyright]
pythonVersion = "3.14"
typeCheckingMode = "recommended"
include = ["app", "tests"]
exclude = ["**/__pycache__", ".venv"]
venvPath = "."
venv = ".venv"

# Common softenings for a pragmatic API codebase under `recommended`:
reportMissingTypeStubs = false        # third-party libs without stubs
reportUnusedCallResult = false        # awaiting side-effecting coroutines is noisy otherwise
reportAny = false                     # relax if integrating dynamically-typed libs
```

Writing code that satisfies `recommended` without noise: annotate every return type (including `-> None`), use `@override` on overrides, avoid bare `Any`, and suppress specific diagnostics with `# pyright: ignore[reportRuleName]` (which requires the rule name — plain `# type: ignore` is flagged by `reportIgnoreCommentWithoutRule`). For adopting on an existing codebase, `basedpyright --writebaseline` records current diagnostics so CI only fails on new ones. basedpyright also exits with a non-zero code on invalid config (unlike pyright, which may silently ignore it). Run via `uv run basedpyright`.

---

## Testing: pytest + Litestar test clients

Use Litestar's own clients from `litestar.testing` — `TestClient` (sync), `AsyncTestClient`, and the `create_test_client` / `create_async_test_client` factories — not a raw httpx `ASGITransport`; the Litestar clients manage the app lifespan and an anyio blocking portal correctly. Generate test data with **Polyfactory's `MsgspecFactory`** (the Litestar-org library; `ModelFactory` is Pydantic-specific).

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
asyncio_mode = "auto"
addopts = "-ra --strict-markers --strict-config"
filterwarnings = ["error"]
xfail_strict = true
```

```python
# tests/conftest.py
from collections.abc import AsyncIterator
import pytest
from litestar import Litestar
from litestar.testing import AsyncTestClient
from app.asgi import create_app


@pytest.fixture
async def client() -> AsyncIterator[AsyncTestClient[Litestar]]:
    async with AsyncTestClient(app=create_app()) as c:
        yield c
```

```python
# tests/test_users.py
import msgspec
from polyfactory.factories.msgspec_factory import MsgspecFactory
from litestar.status_codes import HTTP_200_OK, HTTP_201_CREATED
from app.domain.users import CreateUser


class CreateUserFactory(MsgspecFactory[CreateUser]):
    __model__ = CreateUser


async def test_create_user(client) -> None:
    payload = CreateUserFactory.build()
    resp = await client.post("/users", content=msgspec.json.encode(payload))
    assert resp.status_code == HTTP_201_CREATED


async def test_list_users(client) -> None:
    resp = await client.get("/users")
    assert resp.status_code == HTTP_200_OK
```

Override dependencies per-test by passing `dependencies=` to `create_test_client`, which cleanly stubs services/DB without touching production wiring:

```python
from litestar.testing import create_test_client
from litestar.di import Provide

def test_with_fake_service() -> None:
    with create_test_client(
        route_handlers=[UserController],
        dependencies={"users": Provide(lambda: FakeUserService(), sync_to_thread=False)},
    ) as client:
        assert client.get("/users").status_code == 200
```

Note: with pytest-asyncio use `asyncio_mode = "auto"`; if you follow Litestar's anyio-based examples instead, mark tests with `pytest.mark.anyio` and provide an `anyio_backend` fixture returning `"asyncio"`. Run with `uv run pytest`.

---

## Project layout

```
app/
  asgi.py              # create_app() factory + app object (CLI autodiscovery target)
  config.py            # settings (env-driven), lifespan context manager
  controllers/         # Controller classes grouped by resource
  domain/              # msgspec Structs, services, business logic
  db/                  # Advanced Alchemy models, repositories, migrations
tests/
  conftest.py
pyproject.toml
uv.lock
.python-version        # "3.14"
Dockerfile
```

CI pipeline (order matters — fail fast on cheap checks):

```bash
uv sync --frozen
uv run ruff check
uv run ruff format --check
uv run basedpyright
uv run pytest --cov=app
```

Health check handler for orchestrators:

```python
from litestar import get
from litestar.status_codes import HTTP_200_OK

@get("/health", sync_to_thread=False)
def health() -> dict[str, str]:
    return {"status": "ok"}
```

---

## Anti-patterns to avoid

| Wrong (adjacent-ecosystem habit) | Right (this stack) |
|---|---|
| `from pydantic import BaseModel` for API models | `from msgspec import Struct` |
| `def handler(dep = Depends(get_dep))` | `dependencies={"dep": Provide(get_dep)}` + typed param |
| `from fastapi import APIRouter` | `litestar.Controller` / `litestar.Router` |
| `uvicorn.run("app:app")` / bare `granian ...` alongside the plugin | `GranianPlugin()` + `litestar run` |
| `@get("/users/{user_id}")` (untyped path param) | `@get("/users/{user_id:uuid}")` |
| Letting Ruff move Struct/handler imports into `if TYPE_CHECKING:` | register `runtime-evaluated-base-classes` / `-decorators` |
| `from __future__ import annotations` "to be safe" | omit it — 3.14 defers annotations natively |
| Blocking I/O inside `async def` handler | async driver, or `sync_to_thread=True` dependency |
| Sync handler/dep with no `sync_to_thread` | set `sync_to_thread=True/False` explicitly |
| `None` vs missing conflated in PATCH | `field: T \| None \| UnsetType = UNSET` |
| Granian `--threads` / `--threading-mode` | `--runtime-threads` / `--runtime-mode [auto\|st\|mt]` |
| `mypy` | `basedpyright` (`recommended`) |
| `black` + `isort` + `flake8` | `ruff format` + `ruff check` |
| `pip install` / `poetry add` / `requirements.txt` as source of truth | `uv add` + committed `uv.lock` |
| `[tool.uv] dev-dependencies = [...]` | `[dependency-groups] dev = [...]` |
| `ModelFactory` for msgspec test data | `MsgspecFactory` |
| raw `httpx.ASGITransport` in tests | `litestar.testing.AsyncTestClient` |
| plain `# type: ignore` | `# pyright: ignore[reportSpecificRule]` |
