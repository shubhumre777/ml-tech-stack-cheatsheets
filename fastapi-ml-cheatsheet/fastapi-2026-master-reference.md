# FastAPI 2026 — Master API Reference

**Verified:** 11 November 2026  
**FastAPI:** 0.162.1 · **Python:** 3.11+ · **Pydantic:** v2 · **Runtime:** ASGI

This reference collects current FastAPI APIs for building typed, modular, production-ready HTTP and streaming services. Examples use current `Annotated` patterns and Pydantic v2.

## 1. Install and run

```bash
uv add fastapi "uvicorn[standard]" pydantic
uv run fastapi dev app/main.py
uv run fastapi run app/main.py
# Explicit production server:
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

## 2. App setup and route decorators

```python
from fastapi import FastAPI, status

app = FastAPI(
    title="Inference API",
    version="1.0.0",
    docs_url="/docs",
    redoc_url="/redoc",
    lifespan=lifespan,
    strict_content_type=True,
)

@app.get("/items")
@app.post("/items")
@app.put("/items/{item_id}")
@app.patch("/items/{item_id}")
@app.delete("/items/{item_id}")
@app.options("/items")
@app.head("/items")
@app.trace("/items")
async def endpoint():
    ...
```

Common operation options: `status_code`, `response_model`, `response_class`, `tags`, `summary`, `description`, `responses`, `dependencies`, `deprecated`, `name`, `operation_id`, and `include_in_schema`.

Use `app.api_route(path, methods=[...])` when one handler supports multiple HTTP methods.

Useful application methods: `include_router()`, `mount()`, `add_middleware()`, `add_exception_handler()`, and `openapi()`.

## 3. Routers and modular APIs

```python
from fastapi import APIRouter, Depends

router = APIRouter(
    prefix="/users",
    tags=["users"],
    dependencies=[Depends(require_user)],
    responses={404: {"description": "Not found"}},
)

@router.get("/{user_id}")
async def get_user(user_id: int):
    ...

app.include_router(router, prefix="/api/v1")
```

Organize routers by feature or domain. Put common prefixes, tags, response metadata, and dependencies on the router or on `include_router()`.

## 4. Request parameters — prefer Annotated

```python
from typing import Annotated
from fastapi import Path, Query, Header, Cookie, Body, Form, File, UploadFile

item_id: Annotated[int, Path(gt=0)]
q: Annotated[str | None, Query(min_length=2, max_length=60)] = None
limit: Annotated[int, Query(ge=1, le=100)] = 20
x_token: Annotated[str, Header(alias="X-Token")]
session: Annotated[str | None, Cookie()] = None
payload: Annotated[dict, Body(embed=True)]
title: Annotated[str, Form()]
upload: Annotated[UploadFile, File()]
```

Validation metadata includes `gt`, `ge`, `lt`, `le`, `min_length`, `max_length`, `pattern`, `alias`, `examples`, `description`, and `deprecated`.

For custom JSON Schema metadata, use `json_schema_extra={...}`. Do not pass arbitrary unknown keyword arguments to request parameter helpers.

## 5. Pydantic v2 request and response models

```python
from pydantic import BaseModel, Field

class ItemCreate(BaseModel):
    name: str = Field(min_length=1, max_length=120)
    price: float = Field(gt=0)
    tags: list[str] = Field(default_factory=list)

class ItemOut(BaseModel):
    id: int
    name: str
    price: float

@app.post(
    "/items",
    response_model=ItemOut,
    status_code=status.HTTP_201_CREATED,
)
async def create_item(data: ItemCreate) -> ItemOut:
    return ItemOut(id=1, **data.model_dump())
```

Pydantic v2 helpers: `model_validate()`, `model_dump()`, `model_dump_json()`, `model_copy()`, and `model_json_schema()`.

Response filtering options: `response_model_exclude_unset=True`, `response_model_exclude_defaults=True`, and `response_model_exclude_none=True`.

Prefer separate Create, Update, and public response models. Response models validate, document, serialize, and filter returned data. That filtering is an important protection against accidentally exposing private fields.

## 6. Dependency injection

```python
from typing import Annotated
from fastapi import Depends, Security

def get_db():
    with Session(engine) as db:
        yield db

DB = Annotated[Session, Depends(get_db)]

@app.get("/items")
def list_items(db: DB):
    ...

# Dependency signature:
Depends(dependency, use_cache=True, scope="function")

# OAuth2 scope-aware dependency:
Security(get_current_user, scopes=["items:read"])

# Override a dependency in tests:
app.dependency_overrides[get_db] = fake_get_db
```

Dependencies can be functions, classes, callable instances, nested dependencies, and yield-based resource providers. Use `Depends` for shared logic. Use `Security` when OAuth2 scopes should be represented in OpenAPI. FastAPI calls dependencies for you; do not call them manually.

## 7. Errors, status codes, and responses

```python
from fastapi import HTTPException, Request, Response
from fastapi.responses import JSONResponse, PlainTextResponse
from fastapi.responses import HTMLResponse, RedirectResponse
from fastapi.responses import FileResponse, StreamingResponse

raise HTTPException(status_code=404, detail="Not found")

@app.exception_handler(MyError)
async def handle_error(request: Request, exc: MyError):
    return JSONResponse(status_code=400, content={"detail": str(exc)})

response.headers["X-Trace-ID"] = trace_id
response.set_cookie(
    key="sid", value=token,
    httponly=True, secure=True, samesite="lax",
)
response.delete_cookie(key="sid")
```

Useful constants from `fastapi.status`: `HTTP_200_OK`, `HTTP_201_CREATED`, `HTTP_202_ACCEPTED`, `HTTP_204_NO_CONTENT`, `HTTP_400_BAD_REQUEST`, `HTTP_401_UNAUTHORIZED`, `HTTP_403_FORBIDDEN`, `HTTP_404_NOT_FOUND`, and `HTTP_422_UNPROCESSABLE_ENTITY`.

Prefer typed return values and `response_model` for normal JSON APIs. Use `JSONResponse` when direct response control is required.

## 8. Lifespan and application state

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, Request

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.model = load_model()
    yield
    app.state.model = None  # release resources

app = FastAPI(lifespan=lifespan)

@app.post("/predict")
async def predict(request: Request, data: Input):
    model = request.app.state.model
    ...
```

Code before `yield` runs at startup; code after it runs during shutdown. Use `app.state` for shared resources and `request.state` for per-request data. This is the recommended lifecycle mechanism; do not build new projects around legacy startup/shutdown event decorators.

## 9. Middleware and CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://app.example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.middleware("http")
async def timing(request: Request, call_next):
    response = await call_next(request)
    response.headers["X-App"] = "api"
    return response
```

Built-in ASGI middleware includes `HTTPSRedirectMiddleware`, `TrustedHostMiddleware`, and `GZipMiddleware`. Configure explicit production CORS origins, especially when credentials are enabled.

## 10. Authentication and API keys

```python
from fastapi.security import OAuth2PasswordBearer, HTTPBearer, APIKeyHeader
from fastapi import Depends, Security

oauth2 = OAuth2PasswordBearer(tokenUrl="auth/token")
Token = Annotated[str, Depends(oauth2)]

@app.get("/me")
async def me(token: Token):
    user = verify_token(token)  # signature, expiry, issuer, audience
    return user

user = Security(get_current_user, scopes=["items:read"])
```

Other security helpers include `HTTPBearer`, `HTTPBasic`, `APIKeyHeader`, `APIKeyCookie`, `APIKeyQuery`, and `OAuth2AuthorizationCodeBearer`. FastAPI integrates security definitions into OpenAPI; the application must still verify tokens, enforce authorization, hash passwords, protect secrets, and apply rate limits.

## 11. Files, forms, and static content

```python
from fastapi import UploadFile, File, Form

@app.post("/upload")
async def upload(file: Annotated[UploadFile, File()]):
    contents = await file.read()
    filename = file.filename
    mime_type = file.content_type
    await file.close()
    return {"filename": filename, "bytes": len(contents)}

title: Annotated[str, Form()]
return FileResponse(path, filename="report.csv")
```

Install `python-multipart` for form and file uploads. Prefer `UploadFile` over reading an entire large upload as `bytes`. Enforce upload size limits and validate file content; do not trust the supplied MIME type alone.

Use `StaticFiles(directory="static")` with `app.mount("/static", ...)` to serve static assets.

## 12. Background tasks and request context

```python
from fastapi import BackgroundTasks, Request

@app.post("/notify", status_code=202)
async def notify(tasks: BackgroundTasks):
    tasks.add_task(send_email, "user@example.com")
    return {"queued": True}
```

Request fields include `method`, `url`, `headers`, `query_params`, `path_params`, `client`, `cookies`, `state`, and `app`.

`BackgroundTasks` run after the response in the same application process. Use a durable queue or worker system for heavy, long-running, retryable, or critical jobs.

## 13. WebSockets

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws")
async def socket(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            message = await ws.receive_text()
            await ws.send_json({"echo": message})
    except WebSocketDisconnect:
        pass
```

WebSocket methods include `accept()`, `receive_text()`, `receive_bytes()`, `receive_json()`, `send_text()`, `send_bytes()`, `send_json()`, and `close()`.

## 14. Streaming — SSE, JSON Lines, and bytes

### Server-Sent Events

```python
from fastapi.sse import EventSourceResponse, ServerSentEvent

@app.get("/events", response_class=EventSourceResponse)
async def events():
    yield {"token": "hello"}
    yield ServerSentEvent(
        data={"token": "!"},
        event="token",
        id="2",
    )
```

SSE supports `data`, `event`, `id`, `retry`, and `comment`. It uses `text/event-stream` and is suitable for one-way updates, notifications, and LLM token streams.

### JSON Lines

```python
from collections.abc import AsyncIterable

async def stream_rows() -> AsyncIterable[Item]:
    for item in items:
        yield item  # one JSON value per line
```

FastAPI serializes typed yielded items as JSON Lines (`application/jsonl`).

### Raw text or binary streaming

```python
from fastapi.responses import StreamingResponse

async def chunks():
    yield b"chunk-1\\n"
    yield b"chunk-2\\n"
```

Return a `StreamingResponse` for raw text, bytes, media chunks, or large files. JSON Lines and binary yield streaming were added in 0.134.0; SSE support was added in 0.135.0.

## 15. Testing

```python
from fastapi.testclient import TestClient

def test_health():
    with TestClient(app) as client:
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["ok"] is True
```

Use `TestClient`, `app.dependency_overrides`, pytest fixtures, and async tests with HTTPX `ASGITransport`. Use `with TestClient(app)` when the test must execute lifespan startup and shutdown.

## 16. 2026 operational notes

- **FastAPI 0.143.0:** OpenTelemetry exporter auto-configuration became opt-in. Enable it with `FASTAPI_OTEL_AUTO_CONFIGURE=true` or `FastAPI(telemetry={"auto_configure": True})`.
- **FastAPI 0.141.0:** `app.frontend(check_dir="auto")` was added for frontend-serving workflows.
- **FastAPI 0.146.0:** `FastAPI(root_path=...)` is deprecated; configure the server with `fastapi run --root-path /api/v1`.
- **FastAPI 0.145.0:** `openapi_prefix` was removed.
- **FastAPI 0.155.0:** arbitrary extra keyword arguments were removed from request parameter helpers. Use `json_schema_extra` for custom schema metadata.
- **FastAPI 0.132.0:** strict JSON Content-Type validation is enabled by default. Set `strict_content_type=False` only for clients that cannot send a valid JSON Content-Type.
- Prefer the `lifespan` context manager over legacy startup/shutdown event decorators.
- Do not use deprecated response helpers such as `ORJSONResponse` for ordinary typed JSON output.

## Official references

- [FastAPI release notes](https://fastapi.tiangolo.com/release-notes/)
- [API reference](https://fastapi.tiangolo.com/reference/)
- [Path operation configuration](https://fastapi.tiangolo.com/tutorial/path-operation-configuration/)
- [Request parameter models](https://fastapi.tiangolo.com/tutorial/query-param-models/)
- [Response models](https://fastapi.tiangolo.com/tutorial/response-model/)
- [Dependencies](https://fastapi.tiangolo.com/tutorial/dependencies/)
- [Lifespan](https://fastapi.tiangolo.com/advanced/events/)
- [Middleware](https://fastapi.tiangolo.com/advanced/middleware/)
- [Security](https://fastapi.tiangolo.com/tutorial/security/)
- [Upload files](https://fastapi.tiangolo.com/tutorial/request-files/)
- [WebSockets](https://fastapi.tiangolo.com/advanced/websockets/)
- [Server-Sent Events](https://fastapi.tiangolo.com/tutorial/server-sent-events/)
- [Stream JSON Lines](https://fastapi.tiangolo.com/tutorial/stream-json-lines/)
- [Testing](https://fastapi.tiangolo.com/tutorial/testing/)
