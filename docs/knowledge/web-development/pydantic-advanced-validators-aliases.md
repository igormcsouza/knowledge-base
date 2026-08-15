---
tags:

- web-development
- pydantic
- validation
- fastapi
- python

---

# Pydantic Advanced: Validators, Aliases & Settings

Pydantic models are usually the actual request/response contract of a
FastAPI service — see [DDD & the Service Layer](../architecture/ddd-service-layer.md)
for where they sit relative to the domain logic underneath them. The basics
(type-annotated fields, automatic parsing) get you far, but real services
lean on Pydantic v2's more advanced surface constantly: custom validation
logic, field aliasing for APIs whose wire format doesn't match your Python
naming, computed fields, and settings management.

## Custom Validators

**`@field_validator`** runs against a single field. `mode="before"` runs
*before* Pydantic's own type coercion (so you receive the raw input,
whatever it is); `mode="after"` runs *after* coercion (so you receive the
already-typed, validated value):

```python
from pydantic import BaseModel, field_validator


class SignupRequest(BaseModel):
    email: str
    username: str

    @field_validator("email", mode="before")
    @classmethod
    def normalize_email(cls, value: str) -> str:
        return value.strip().lower()  # raw input — do this before type coercion

    @field_validator("username", mode="after")
    @classmethod
    def username_no_spaces(cls, value: str) -> str:
        if " " in value:
            raise ValueError("username cannot contain spaces")
        return value  # already coerced to str by this point
```

**`@model_validator`** runs against the *whole* model, once all fields have
been individually validated — the right place for checks that span more
than one field:

```python
from pydantic import BaseModel, model_validator


class DateRange(BaseModel):
    start: date
    end: date

    @model_validator(mode="after")
    def check_order(self) -> "DateRange":
        if self.end < self.start:
            raise ValueError("end must not be before start")
        return self
```

`mode="before"` on a `@model_validator` receives the raw input dict (useful
for reshaping data before field-level validation even starts); `mode="after"`
receives the already-constructed, field-validated model instance.

## Field Aliases

APIs frequently use a wire format that doesn't match idiomatic Python naming
— `camelCase` JSON against `snake_case` Python is the most common case.
`Field(alias=...)` maps between them:

```python
from pydantic import BaseModel, Field


class UserResponse(BaseModel):
    user_id: int = Field(alias="userId")
    full_name: str = Field(alias="fullName")
```

A plain `alias` is used for *both* parsing input and producing output. Often
you want different behavior in each direction — e.g. accept either the
Python name or the wire name on input, but always emit the wire name on
output — which is what `validation_alias` and `serialization_alias` are for:

```python
class UserResponse(BaseModel):
    user_id: int = Field(validation_alias="userId", serialization_alias="userId")
```

`populate_by_name` (in `model_config`) additionally allows the model to be
*constructed* using the Python attribute name even when an alias is set —
useful when your own code builds the model directly (not from parsed JSON)
and shouldn't have to remember the wire-format alias:

```python
from pydantic import BaseModel, ConfigDict, Field


class UserResponse(BaseModel):
    model_config = ConfigDict(populate_by_name=True)

    user_id: int = Field(alias="userId")

# both of these work now:
UserResponse(userId=42)      # from parsed JSON, using the alias
UserResponse(user_id=42)     # from your own code, using the Python name
```

## Computed Fields

`@computed_field` exposes a derived value as if it were a real field —
included in `.model_dump()` and serialized JSON, without being part of the
constructor's input:

```python
from pydantic import BaseModel, computed_field


class Order(BaseModel):
    unit_price: float
    quantity: int

    @computed_field
    @property
    def total(self) -> float:
        return self.unit_price * self.quantity
```

```python
order = Order(unit_price=9.99, quantity=3)
order.model_dump()
# {"unit_price": 9.99, "quantity": 3, "total": 29.97}
```

This keeps a response model from having to duplicate logic the caller could
otherwise get wrong — `total` is computed once, in one place, every time the
model is serialized.

## Custom Types via `Annotated` + Before/AfterValidator

The modern, PEP 593-based pattern for reusable custom validation logic is
`Annotated` combined with `BeforeValidator`/`AfterValidator`, replacing the
older Pydantic v1 pattern of implementing `__get_validators__` on a custom
class:

```python
from typing import Annotated
from pydantic import BaseModel, BeforeValidator, AfterValidator


def strip_whitespace(value: str) -> str:
    return value.strip()


def must_be_positive(value: int) -> int:
    if value <= 0:
        raise ValueError("must be positive")
    return value


CleanStr = Annotated[str, BeforeValidator(strip_whitespace)]
PositiveInt = Annotated[int, AfterValidator(must_be_positive)]


class Product(BaseModel):
    name: CleanStr
    stock: PositiveInt
```

`CleanStr` and `PositiveInt` are now reusable, composable type aliases —
define the validation logic once, attach it to any field across any number
of models by annotation, instead of repeating a `@field_validator` in every
model that needs the same rule.

## Strict vs. Lax Mode

By default Pydantic is **lax**: it coerces reasonably-shaped input into the
declared type (`"42"` → `42` for an `int` field). **Strict mode** disables
that coercion and requires the input to already be the exact declared type:

```python
from pydantic import BaseModel, Field, ConfigDict


class StrictOrder(BaseModel):
    model_config = ConfigDict(strict=True)

    quantity: int  # "5" is now rejected — only an actual int passes


class LaxOrder(BaseModel):
    quantity: int = Field(strict=False)  # per-field override; "5" still coerces
```

Lax mode is usually right for parsing external input (form data, query
params, loosely-typed JSON from a third party) where some coercion is
expected and helpful. Strict mode earns its place for internal
service-to-service contracts, or anywhere silently accepting `"true"` for a
`bool` field would mask a client-side bug you'd rather catch immediately.

## Settings Management with `pydantic-settings`

`BaseSettings` (moved to the separate `pydantic-settings` package in v2) is
Pydantic applied to configuration instead of request bodies — fields load
from environment variables automatically, with the same validation you'd
apply to any other model:

```python
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class DatabaseSettings(BaseModel):
    host: str
    port: int = 5432


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_nested_delimiter="__", env_file=".env")

    database: DatabaseSettings
    debug: bool = False
    api_key: str = Field(alias="API_KEY")


settings = Settings()
```

With `env_nested_delimiter="__"`, nested config loads from prefixed
environment variables — `DATABASE__HOST=db.internal` and
`DATABASE__PORT=5433` populate `settings.database.host` and
`settings.database.port` automatically, no manual parsing required.

!!! tip "Pro Tip"
    Construct `Settings()` exactly once, at import/startup time, and pass it
    around (FastAPI's `Depends` works well here) rather than re-instantiating
    it per request — env var loading and validation is a fixed one-time cost,
    not something to repeat on every call.

## Summary

- `@field_validator` (single field, `before`/`after` coercion) and
  `@model_validator` (whole model, cross-field checks) cover custom
  validation logic beyond plain type annotations.
- `alias` vs. `validation_alias`/`serialization_alias` control input vs.
  output field naming independently; `populate_by_name` lets your own code
  still use the Python attribute name.
- `@computed_field` exposes a derived value in serialized output without
  making it a constructor argument.
- `Annotated[...] + BeforeValidator/AfterValidator` is the modern,
  composable way to build reusable custom types — the PEP 593 successor to
  `__get_validators__`.
- Lax mode coerces input (good for external/loosely-typed data); strict
  mode requires exact types (good for internal contracts).
- `pydantic-settings`' `BaseSettings` applies the same validation model to
  configuration, with automatic env var and nested-config loading.

## Related Articles

- [FastAPI Event Loop](fastapi-event-loop.md) — where these models sit in a
  FastAPI request: parsed before your route body runs, serialized after it
  returns.
- [DDD & the Service Layer](../architecture/ddd-service-layer.md) — Pydantic
  models are typically the request/response layer wrapping the service-layer
  logic underneath.
- [API Lifecycle & Design](api-lifecycle-design.md) — field aliasing is a
  common tool for keeping a wire-format contract stable while Python-side
  naming evolves.
