# Pydantic v2 and SQLAlchemy 2.x

Read this when writing Pydantic models or SQLAlchemy sessions and queries, or when migrating Pydantic v1 to v2 or SQLAlchemy 1.x to 2.x. Check the installed versions first; a project can run v2 while importing `pydantic.v1` in places.

## Pydantic v1 to v2

| v1 | v2 |
|---|---|
| `Model.parse_obj(d)` | `Model.model_validate(d)` |
| `Model.parse_raw(s)` | `Model.model_validate_json(s)` |
| `m.dict()`, `m.json()` | `m.model_dump()`, `m.model_dump_json()` |
| `m.copy(update=...)` | `m.model_copy(update=...)` |
| `Model.construct(...)` | `Model.model_construct(...)` |
| `Model.update_forward_refs()` | `Model.model_rebuild()` |
| `@validator`, `@root_validator` | `@field_validator`, `@model_validator` |
| `class Config:` | `model_config = ConfigDict(...)` |
| `orm_mode = True` | `from_attributes=True` |
| `Field(regex=...)` | `Field(pattern=...)` |

Behavior changes that break silently or only at runtime:

- **`Optional[X]` no longer implies a default.** `nickname: Optional[str]` is required in v2 (it may be `None`, but must be present). Clients that omitted it now get a validation error. Write `nickname: str | None = None` when absence is allowed.
- **No int or float to `str` coercion by default.** A client sending `"zip": 12345` for a `str` field now fails.
- **Lax mode still coerces numbers.** For an `int` field, `"123"` and `1.0` are accepted, `1.5` is rejected. Use `Field(strict=True)` or `ConfigDict(strict=True)` where coercion would hide a client bug.
- **Deprecated names still work with warnings.** A test run with warnings ignored hides mixed v1/v2 idioms; run with `-W error::DeprecationWarning` scoped to your package when migrating.

## Pydantic at trust boundaries

**Extra keys are ignored by default.** A client sending `{"is_admin": true}` to a model without that field gets no error, which hides client bugs and makes probing invisible. For privileged or state-changing input, `model_config = ConfigDict(extra="forbid")`.

**Validation bypasses.** `model_construct()` and `model_copy(update=...)` do not validate. The `update` dict is applied as is, including keys the model does not declare as client-writable.

```python
# Before: the body is checked, then the unchecked dict is what gets applied
@app.patch("/teams/{team_id}/settings")
def update_settings(team_id: int, body: dict, team: Team = Depends(team_admin)):
    TeamSettingsIn.model_validate(body)
    new = TeamSettingsOut.model_validate(team.settings).model_copy(update=body)
    team.settings = new.model_dump()     # body can carry "plan": "enterprise", "seat_limit": 10000

# After: only fields of the validated model, and only those the client sent
@app.patch("/teams/{team_id}/settings")
def update_settings(
    team_id: int,
    body: TeamSettingsIn,
    team: Team = Depends(team_admin),
    db: Session = Depends(get_db),
) -> TeamSettingsOut:
    changes = body.model_dump(mode="json", exclude_unset=True)
    team.settings = {**team.settings, **changes}
    db.commit()
    return TeamSettingsOut.model_validate(team.settings)
```

`exclude_unset=True` distinguishes "not sent" from "sent as null", so a PATCH that omits `timezone` does not overwrite it with `None`. Assign a new dict rather than mutating the loaded one: the ORM does not detect in-place changes to a plain `JSON` column.

**Separate models per direction and operation.** `UserCreate`, `UserUpdate`, `UserOut`. One model used for input and output lets a client set server-owned fields (`id`, `role`, `balance`) or read fields it should not (`password_hash`).

**Secrets.** `SecretStr` hides the value in `repr` and JSON output; `.get_secret_value()` at the point of use.

## SQLAlchemy 2.x session lifecycle

A `Session` is one unit of work and not thread-safe. An `AsyncSession` is one task's unit of work and not safe across concurrent tasks either.

```python
# Before: one session for the whole process
engine = create_engine(settings.database_url)
db = Session(engine)

# After: one session per request, closed on every path
SessionLocal = sessionmaker(engine)

def get_db() -> Iterator[Session]:
    with SessionLocal() as session:
        yield session
```

What goes wrong with a shared session:

- FastAPI runs `def` routes in a threadpool, so concurrent requests use the same session and connection at once: interleaved statements, one request committing or rolling back another's work, errors that appear only under load.
- After a failed flush the session must be rolled back. Until someone calls `rollback()`, every operation raises `PendingRollbackError`, so one bad request breaks all later ones on that worker.
- The identity map keeps returning objects loaded earlier, so one request sees another's stale state.

Transaction shape: `with Session(engine) as session, session.begin():` commits on success and rolls back on exception. With `sessionmaker`, `with SessionLocal.begin() as session:` does the same.

**Expire on commit.** `expire_on_commit=True` (the default) expires every loaded object at commit. Touching an attribute afterwards issues a new query, raises `DetachedInstanceError` if the session is closed, and under asyncio attempts implicit IO. Build response objects before commit or close, or configure `async_sessionmaker(engine, expire_on_commit=False)` for async code.

**Async specifics.** Lazy loading needs implicit IO, which asyncio cannot do on attribute access; it raises (`MissingGreenlet`). Eager-load what the response needs, or use `AsyncAttrs` and `await obj.awaitable_attrs.children`. `await engine.dispose()` on shutdown. From 2.1, `greenlet` is installed only with the `sqlalchemy[asyncio]` extra; without it, importing `sqlalchemy.ext.asyncio` fails with an `ImportError` naming that extra.

## N+1 and loading strategies

Default relationship loading is `lazy="select"`, one query per object per relationship on first access.

```python
# Before: 1 query for tickets + 1 per ticket for the assignee + 1 per ticket for tags
tickets = db.scalars(select(Ticket).where(Ticket.project_id == project.id).limit(page_size)).all()
return [{"id": t.id, "assignee": t.assignee.name, "tags": [tag.name for tag in t.tags]} for t in tickets]

# After: 2 queries regardless of page size (the many-to-one is joined into the first)
tickets = db.scalars(
    select(Ticket)
    .where(Ticket.project_id == project.id)
    .options(joinedload(Ticket.assignee), selectinload(Ticket.tags), raiseload("*"))
    .order_by(Ticket.updated_at.desc(), Ticket.id.desc())
    .limit(page_size)
).all()
```

- `selectinload` for one-to-many and many-to-many (one extra `IN` query, no row multiplication).
- `joinedload` for many-to-one. When joined-loading a collection, call `.unique()` on the result; 2.x raises otherwise.
- `raiseload("*")` in list queries, or `lazy="raise"` on relationships, turns a future N+1 into an error in tests instead of a slow endpoint in production.
- In tests for list endpoints, assert the number of statements (an event listener on `before_cursor_execute` counting calls) so regressions are caught.

## Other 2.x changes worth knowing

- `session.query(Model)` is legacy; new code uses `select()` with `session.scalars()` / `session.execute()`.
- `session.get(Model, pk)` replaces `query.get(pk)`.
- Raw strings are not accepted as SQL; they must be wrapped in `text()`, which then deserves the same injection review as any string SQL.
- 2.1 requires Python 3.11+, and a bare `postgresql://` URL now selects psycopg 3 instead of psycopg2. An image that has only psycopg2 installed fails at `create_engine` with `No module named 'psycopg'` after the upgrade; name the driver in the URL (`postgresql+psycopg://` or `postgresql+psycopg2://`) so the upgrade cannot switch it.
