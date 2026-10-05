# Parsing untrusted input

Read this when Rust code turns bytes, request bodies, serde payloads, numbers, strings, file names or regex patterns from outside the process into typed values.

The goal at a boundary is a total function: every possible input produces either a valid domain value or a typed error, with no panic, no unbounded allocation and no silent default. Inner code then works with types that cannot hold invalid values and does not re-check.

## Length-prefixed frames

```rust
// Before: four panics on a short or malformed frame, and no cap on the declared length
fn parse_frame(buf: &[u8]) -> Frame {
    let len = u32::from_be_bytes(buf[0..4].try_into().unwrap()) as usize;
    let kind = FrameKind::from_u8(buf[4]).unwrap();
    let body = buf[5..5 + len].to_vec();
    Frame { kind, body }
}

// After
const LEN_PREFIX: usize = 4;
const HEADER_LEN: usize = LEN_PREFIX + 1;

/// Returns the frame and the number of bytes consumed.
pub fn parse_frame(buf: &[u8], max_body: usize) -> Result<(Frame, usize), FrameError> {
    let Some((len_bytes, rest)) = buf.split_first_chunk::<LEN_PREFIX>() else {
        return Err(FrameError::Incomplete);
    };
    let Some((&kind_byte, rest)) = rest.split_first() else {
        return Err(FrameError::Incomplete);
    };
    let len = usize::try_from(u32::from_be_bytes(*len_bytes)).map_err(|_| FrameError::TooLarge)?;
    if len > max_body {
        return Err(FrameError::TooLarge);
    }
    let kind = FrameKind::try_from(kind_byte)?;
    let body = rest.get(..len).ok_or(FrameError::Incomplete)?;
    Ok((Frame { kind, body: body.to_vec() }, HEADER_LEN + len))
}
```

Check the declared length against the limit before waiting for more bytes. A streaming decoder that buffers "until the frame is complete" without that check lets a peer announce 4 GiB and trickle bytes until the process runs out of memory. `max_body` comes from config. The same applies to element counts: a declared count of 10 million records must not become `Vec::with_capacity(10_000_000)` before any record has arrived; reserve at most `min(count, cap)` and let the vector grow as records actually parse.

## Bodies, uploads and decompression

```rust
fn read_limited(reader: impl Read, limit: u64) -> Result<Vec<u8>, UploadError> {
    let mut buf = Vec::new();
    reader.take(limit.saturating_add(1)).read_to_end(&mut buf)?;
    if u64::try_from(buf.len()).map_or(true, |n| n > limit) {
        return Err(UploadError::TooLarge);
    }
    Ok(buf)
}
```

Reading one byte past the limit distinguishes "exactly at the limit" from "over it". Tokio's `AsyncReadExt::take` works the same way. In axum, `DefaultBodyLimit` (2 MB unless changed) covers the `Bytes`, `String`, `Json` and `Form` extractors; a handler that consumes the body stream itself needs its own limit, for example `http_body_util::Limited`. Decompression needs a limit on the output, not the input: wrap the decoder in `take` as well, since a small gzip body can expand by orders of magnitude.

Binary formats that read length prefixes from the input (bincode and similar) need their size limit configured; the default may be unlimited. bincode itself is unmaintained (RUSTSEC-2025-0141, which `cargo deny check advisories` reports), and its 3.0.0 release contains only a `compile_error!`; new code should pick a maintained format.

## Serde at the boundary

```rust
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct CreateTransfer {
    pub to_account: AccountId,
    pub amount_minor: MinorUnits,
    pub currency: Currency,
    pub idempotency_key: IdempotencyKey,
}
```

The acting user and the source account come from the authenticated session, never from this struct. `deny_unknown_fields` turns a misspelled or unexpected key into a 400 instead of a silently ignored field. It is not supported together with `#[serde(flatten)]`, so privileged payloads should not flatten.

Validation lives in the field types, so an invalid value cannot be constructed anywhere:

```rust
#[derive(Debug, Clone, Deserialize)]
#[serde(try_from = "String")]
pub struct IdempotencyKey(String);

impl TryFrom<String> for IdempotencyKey {
    type Error = InvalidIdempotencyKey;

    fn try_from(raw: String) -> Result<Self, Self::Error> {
        let valid = (MIN_IDEMPOTENCY_KEY_LEN..=MAX_IDEMPOTENCY_KEY_LEN).contains(&raw.len())
            && raw.bytes().all(|b| b.is_ascii_alphanumeric() || b == b'-');
        if valid { Ok(Self(raw)) } else { Err(InvalidIdempotencyKey) }
    }
}
```

The error type needs `Display` for serde to report it.

### PATCH without fail-open defaults

```rust
// Before: {"session_ttl_secs": 900} also sets require_mfa = false and ip_allowlist = []
#[derive(Deserialize, Default)]
#[serde(default)]
pub struct SecurityPolicy {
    pub session_ttl_secs: u32,
    pub require_mfa: bool,
    pub ip_allowlist: Vec<IpNet>,
}
// handler: *stored = body;

// After
#[derive(Deserialize)]
#[serde(deny_unknown_fields)]
pub struct SecurityPolicyPatch {
    pub session_ttl_secs: Option<u32>,
    pub require_mfa: Option<bool>,
    pub ip_allowlist: Option<Vec<IpNet>>,
}

impl SecurityPolicy {
    pub fn apply(&mut self, patch: SecurityPolicyPatch) {
        if let Some(ttl) = patch.session_ttl_secs {
            self.session_ttl_secs = ttl;
        }
        if let Some(mfa) = patch.require_mfa {
            self.require_mfa = mfa;
        }
        if let Some(list) = patch.ip_allowlist {
            self.ip_allowlist = list;
        }
    }
}
```

With plain `Option<T>`, a missing key and an explicit `null` both become `None`. When the API must distinguish "leave unchanged" from "clear", use `Option<Option<T>>` with a custom deserializer (serde_with provides one); by default serde collapses both cases.

Changes to security settings usually deserve a re-authentication or an audit record; that is a product decision, but a PATCH endpoint that can switch MFA off should not be reachable with a stale session.

### Enums that select behavior

```rust
// Risky: the first variant that parses wins, and unknown fields are ignored,
// so an "internal" payout with a memo can deserialize as External
#[derive(Deserialize)]
#[serde(untagged)]
enum PayoutTarget {
    External { account: String },
    Internal { account: String, memo: String },
}

// Explicit
#[derive(Deserialize)]
#[serde(tag = "type", rename_all = "snake_case", deny_unknown_fields)]
enum PayoutTarget {
    External { account: ExternalAccount },
    Internal { account: AccountId, memo: String },
}
```

## Numbers

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Serialize, Deserialize)]
#[serde(transparent)]
pub struct MinorUnits(u64);

impl MinorUnits {
    pub fn checked_add(self, rhs: Self) -> Option<Self> {
        self.0.checked_add(rhs.0).map(Self)
    }

    pub fn checked_sub(self, rhs: Self) -> Option<Self> {
        self.0.checked_sub(rhs.0).map(Self)
    }

    pub fn checked_mul_qty(self, qty: u32) -> Option<Self> {
        self.0.checked_mul(u64::from(qty)).map(Self)
    }
}
```

No `Add` or `Sub` impl, so `+` on amounts does not compile. If the service handles more than one currency, the amount type carries the currency and the operations reject a mismatch. Conversions from storage types (`i64` from PostgreSQL `bigint`) go through `u64::try_from` and fail on negative values rather than wrapping.

Floats parsed from text accept more than people expect: `"NaN".parse::<f64>()`, `"inf"` and `"-infinity"` all succeed. JSON has no NaN literal, but query strings, CSV, form fields and binary formats do. Reject `!x.is_finite()` at the boundary for any float that feeds a comparison, a sort, a `Duration`, or a cast to an integer.

## Strings

```rust
// Panics when byte `max` falls inside a multi-byte character
let preview = &comment[..max];

// Rust 1.91+
let preview = if comment.len() <= max {
    comment
} else {
    &comment[..comment.floor_char_boundary(max)]
};

// Older MSRV
fn truncate_bytes(s: &str, max: usize) -> &str {
    if s.len() <= max {
        return s;
    }
    let mut end = max;
    while !s.is_char_boundary(end) {
        end -= 1;
    }
    &s[..end]
}
```

`is_char_boundary(0)` is always true, so the loop terminates. Both versions still split grapheme clusters (an emoji built from several code points can lose its second half). Byte limits suit storage and protocol fields. Limits a user sees ("50 characters") count grapheme clusters (the `unicode-segmentation` crate), because `chars().count()` counts code points and splits emoji and combining accents.

## File paths

```rust
fn upload_path(root: &Path, name: &str) -> Result<PathBuf, PathError> {
    let candidate = Path::new(name);
    let only_plain_names = candidate
        .components()
        .all(|c| matches!(c, Component::Normal(_)));
    if name.is_empty() || !only_plain_names {
        return Err(PathError::Invalid);
    }
    Ok(root.join(candidate))
}
```

`Path::join` with an absolute argument replaces the base entirely, and `..` walks out of it. Rejecting every component that is not `Normal` removes `/`, `..`, `.` and Windows drive prefixes. Symlinks inside the root can still point outside; if the directory may contain them, `canonicalize` the result and check `starts_with(canonical_root)` (which compares whole components), accepting that the check and the later open are not atomic. Better: store uploads under a generated id and keep the client's file name as metadata only.

## Regular expressions

Fixed `regex` patterns on untrusted input are not a ReDoS finding (see SKILL.md). What still needs care:

- A pattern supplied by a user can compile to a very large automaton (counted repetitions multiply). Build it with `RegexBuilder::size_limit` and cap the pattern length.
- Compile fixed patterns once (`LazyLock<Regex>` or a struct field), not per call.

## Hash maps keyed by input

SKILL.md covers HashDoS and iteration order. Fast unkeyed hashers are fine for keys the process generates itself.

## Secrets

```rust
pub struct ApiKey(String);

impl fmt::Debug for ApiKey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.write_str("ApiKey(<redacted>)")
    }
}

#[tracing::instrument(skip_all, fields(user_id = %user_id))]
async fn login(user_id: UserId, password: SecretString, db: &Db) -> Result<Session, AuthError> { ... }
```

`#[instrument]` records every argument by default, using `Debug` for types that are not primitives, so without `skip_all` or `skip(password)` the password lands in every span. `err` on `#[instrument]` logs the error's `Display`, so error messages must not embed secrets either. Compare tokens and MACs with `subtle::ConstantTimeEq` (`a.ct_eq(b).into()`) rather than `==`, which returns at the first differing byte.
