# Items API

A stateless CRUD REST API built with [FastAPI](https://fastapi.tiangolo.com/). Every endpoint returns structured JSON in a consistent envelope.

## Features

- Full CRUD on an `items` resource (create, list, read, replace, partial update, delete)
- Consistent success and error response envelopes
- Pagination on list endpoints
- Input validation with per-field error details
- Stateless: no sessions, cookies, or per-client server state
- Storage hidden behind a repository interface, so the backend is swappable

## Requirements

- Python 3.9+

## Quick start

```bash
pip install fastapi uvicorn
uvicorn main:app --reload
```

The API runs at `http://localhost:8000`. Interactive docs are at `http://localhost:8000/docs` (Swagger UI) and `http://localhost:8000/redoc`.

## Data model

| Field         | Type            | Rules                    |
|---------------|-----------------|--------------------------|
| `id`          | string (UUID)   | Server-generated         |
| `name`        | string          | Required, 1-120 chars    |
| `description` | string or null  | Optional, max 1000 chars |
| `price`       | number          | Required, >= 0           |
| `tags`        | array of string | Optional, default `[]`   |
| `created_at`  | datetime (UTC)  | Server-generated         |
| `updated_at`  | datetime (UTC)  | Server-managed           |

## Endpoints

| Method | Path                     | Description                        | Success |
|--------|--------------------------|------------------------------------|---------|
| POST   | `/items`                 | Create an item                     | 201     |
| GET    | `/items?limit=&offset=`  | List items (paginated)             | 200     |
| GET    | `/items/{id}`            | Get one item                       | 200     |
| PUT    | `/items/{id}`            | Replace an item (all fields)       | 200     |
| PATCH  | `/items/{id}`            | Update only the fields provided    | 200     |
| DELETE | `/items/{id}`            | Delete an item                     | 200     |
| GET    | `/health`                | Health check                       | 200     |

List query parameters: `limit` (1-100, default 20) and `offset` (>= 0, default 0).

## Response format

**Success**

```json
{
  "data": {
    "id": "3f0c2b1e-7a44-4c8e-9d1a-2b5f6c9e0a11",
    "name": "Widget",
    "description": null,
    "price": 9.99,
    "tags": ["demo"],
    "created_at": "2026-09-28T10:15:30.123456Z",
    "updated_at": "2026-09-28T10:15:30.123456Z"
  }
}
```

List responses add pagination info:

```json
{
  "data": [ ... ],
  "meta": { "total": 42, "limit": 20, "offset": 0 }
}
```

**Error**

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed.",
    "details": [
      { "field": "price", "message": "Input should be greater than or equal to 0" }
    ]
  }
}
```

| Status | `code`             | When                          |
|--------|--------------------|-------------------------------|
| 404    | `not_found`        | Item ID does not exist        |
| 422    | `validation_error` | Invalid request body or query |
| 500    | `internal_error`   | Unexpected server error       |

## Examples

```bash
# Create
curl -X POST http://localhost:8000/items \
  -H 'Content-Type: application/json' \
  -d '{"name": "Widget", "price": 9.99, "tags": ["demo"]}'

# List
curl 'http://localhost:8000/items?limit=10&offset=0'

# Get
curl http://localhost:8000/items/<id>

# Replace (all fields required)
curl -X PUT http://localhost:8000/items/<id> \
  -H 'Content-Type: application/json' \
  -d '{"name": "Widget v2", "price": 12.50, "tags": []}'

# Partial update
curl -X PATCH http://localhost:8000/items/<id> \
  -H 'Content-Type: application/json' \
  -d '{"price": 7.25}'

# Delete
curl -X DELETE http://localhost:8000/items/<id>
```

## Statelessness and storage

The API keeps no session or client state; each request contains everything needed to process it, so any instance can serve any request.

The included `InMemoryItemRepository` is for **development and testing only**. Its data is lost on restart and is not shared between processes or instances. Before running multiple instances or deploying to production, implement the `ItemRepository` protocol against a real datastore (PostgreSQL, Redis, DynamoDB, etc.) and assign it to `repo` in `main.py`:

```python
class ItemRepository(Protocol):
    def create(self, data: ItemCreate) -> Item: ...
    def get(self, item_id: str) -> Optional[Item]: ...
    def list(self, limit: int, offset: int) -> tuple[list[Item], int]: ...
    def replace(self, item_id: str, data: ItemReplace) -> Optional[Item]: ...
    def patch(self, item_id: str, data: ItemPatch) -> Optional[Item]: ...
    def delete(self, item_id: str) -> bool: ...
```

## Project structure

```
.
├── main.py      # App, schemas, repository, endpoints, error handlers
└── README.md
```

## Not included

Authentication, rate limiting, filtering/sorting on list, and a persistent database are intentionally left out and are straightforward to add.
