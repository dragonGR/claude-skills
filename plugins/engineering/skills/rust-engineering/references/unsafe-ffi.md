# Unsafe code and FFI

Read this before writing or reviewing an `unsafe` block, an `unsafe fn`, an `unsafe impl`, code exported to C or imported from it, or anything built on raw pointers or `MaybeUninit`.

## What a SAFETY comment has to say

The comment names each precondition the unsafe operation requires and points at the code that upholds it. If it restates the operation, it proves nothing.

```rust
// Useless
// SAFETY: calling get_unchecked
unsafe { self.buf.get_unchecked(idx) }

// Useful
// SAFETY: `idx < self.len` is checked at the top of this function, and `self.len <=
// self.buf.len()` holds because `push` is the only writer of `len`.
unsafe { self.buf.get_unchecked(idx) }
```

An `unsafe fn` documents what callers must guarantee:

```rust
/// Reinterprets a frame header in place.
///
/// # Safety
///
/// `bytes` must be at least `HEADER_LEN` long and aligned to `align_of::<Header>()`.
pub unsafe fn header_ref(bytes: &[u8]) -> &Header { ... }
```

Review questions for each block:

- Does the invariant hold on every path into the block, including error paths and early returns above it?
- Can safe code elsewhere in the module break it? The soundness boundary is the module: if a `pub(crate)` field that the invariant depends on can be written from outside, the block is unsound even though it looks local.
- Does the block do more than one unsafe thing? Each needs its own justification. Smaller blocks make that visible.
- Could a panic between establishing the invariant and relying on it leave a value observable in a broken state (for example after `set_len` but before initialization)?

Tooling that helps: Clippy `undocumented_unsafe_blocks` (restriction) requires the comment; the edition 2024 default warning `unsafe_op_in_unsafe_fn` requires explicit `unsafe {}` blocks inside `unsafe fn` bodies, so each operation gets its own justification.

## Edition 2024 changes that touch unsafe code

- `extern` blocks must be written `unsafe extern "C" { ... }`. Items inside can be declared `safe fn` (callable without `unsafe`) or `unsafe fn`; unmarked items default to unsafe. Mark an import `safe` only if every possible argument is sound (`sqrt(f64)`), never when it takes pointers.
- `no_mangle`, `export_name` and `link_section` must be written `#[unsafe(no_mangle)]` and so on, because symbol collisions across linked libraries are unsound.
- `std::env::set_var` and `remove_var` are `unsafe`: calling them while other threads may read the environment is unsound on some platforms. Set environment before spawning threads (before building a multi-threaded runtime), or pass configuration explicitly.
- References to `static mut` are denied. Use atomics, `Mutex`, `OnceLock`/`LazyLock`, or `&raw mut` for FFI.

## Values that must never exist

Undefined behavior happens when an invalid value is created, not when it is used.

| Mistake | Why it is UB | Instead |
|---|---|---|
| `transmute::<u8, MyEnum>(b)`, `transmute::<u8, bool>(b)` | Only declared discriminants and `0`/`1` are valid | `MyEnum::try_from(b)`, `b != 0` after validating |
| `transmute::<[u8; 4], u32>(bytes)` | Works, but hides endianness | `u32::from_le_bytes(bytes)` or `from_be_bytes` |
| `mem::zeroed::<&T>()`, of `NonNull`, `Box`, `fn` pointers | These are never null | `Option<&T>`, `Option<NonNull<T>>` |
| `MaybeUninit::<u32>::uninit().assume_init()` | Uninitialized memory is not "some integer" | Write every field, then `assume_init` |
| `vec.set_len(n)` before writing elements | Exposes uninitialized `T` | Write through `spare_capacity_mut()`, then `set_len` |
| `str::from_utf8_unchecked(socket_bytes)` | `str` must be valid UTF-8 | `str::from_utf8`, or keep `&[u8]` |
| `&mut *ptr` while another reference to the same data is live | Aliasing violation | Restructure so one path owns access; Miri finds these |
| `slice::from_raw_parts(ptr, 0)` with null `ptr` | Pointer must be non-null and aligned even for length 0 | Check null and use `&[]` |

For casting between plain-data structs and bytes, the `bytemuck` and `zerocopy` crates check layout requirements at compile time, which is safer than hand-written transmutes.

## Exporting Rust to C

```rust
#[repr(C)]
pub enum RsStatus {
    Ok = 0,
    InvalidArgument = 1,
    Internal = 2,
}

/// # Safety
///
/// `data` must point to `len` readable bytes in one allocation that stay unmodified for the
/// duration of the call, or may be null when `len` is 0. `out` must be valid for a `u64` write.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rs_checksum(data: *const u8, len: usize, out: *mut u64) -> RsStatus {
    let outcome = std::panic::catch_unwind(|| {
        if out.is_null() {
            return RsStatus::InvalidArgument;
        }
        let input: &[u8] = if len == 0 {
            &[]
        } else if data.is_null() {
            return RsStatus::InvalidArgument;
        } else {
            // SAFETY: `data` is non-null, and the caller guarantees `len` readable bytes in
            // a single allocation that outlive this call.
            unsafe { std::slice::from_raw_parts(data, len) }
        };
        let sum = checksum(input);
        // SAFETY: `out` is non-null (checked above); the caller guarantees it is aligned and writable.
        unsafe { out.write(sum) };
        RsStatus::Ok
    });
    outcome.unwrap_or(RsStatus::Internal)
}
```

The rules this template encodes:

- Every exported function body runs inside `catch_unwind`. Without it, a panic reaching the `extern "C"` boundary aborts the host process. `catch_unwind` does nothing under `panic = "abort"`, so a library meant to be embedded should not be built with it.
- Null and length are checked before building a slice, and `(NULL, 0)` is accepted as an empty input because C callers pass it.
- Only `#[repr(C)]` types, integers and raw pointers cross the boundary. Never `String`, `Vec`, `&str`, slices, or enums without a `repr`.
- A Rust `enum` received from C is not trustworthy: take an integer and convert with `TryFrom`.

### Handles and ownership

```rust
pub struct Engine { /* ... */ }

#[unsafe(no_mangle)]
pub extern "C" fn rs_engine_new() -> *mut Engine {
    std::panic::catch_unwind(|| Box::into_raw(Box::new(Engine::new())))
        .unwrap_or(std::ptr::null_mut())
}

/// # Safety
///
/// `engine` must be null or a pointer returned by `rs_engine_new` that has not been freed.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rs_engine_free(engine: *mut Engine) {
    if engine.is_null() {
        return;
    }
    // SAFETY: per the contract, the pointer came from Box::into_raw and is freed only here.
    drop(unsafe { Box::from_raw(engine) });
}
```

Memory crosses back to the allocator that created it. A pointer from `Box::into_raw` or `CString::into_raw` must be released through `Box::from_raw` or `CString::from_raw`, which is why the library exports `rs_engine_free` and `rs_string_free`; C calling `free()` on it is undefined behavior. The C side also must not change a returned string's length (writing a nul inside it) before handing it back. Simpler still for strings: let the caller pass a buffer and capacity, write into it, and return the required length.

`Engine`'s `Drop` runs inside `rs_engine_free`; if it can panic, wrap the drop in `catch_unwind` too.

### Callbacks

A C library that calls back into Rust with a `void *user_data` needs a trampoline:

```rust
unsafe extern "C" fn on_record(user: *mut c_void, rec: *const RawRecord) -> c_int {
    let outcome = std::panic::catch_unwind(|| {
        if user.is_null() || rec.is_null() {
            return CB_ERROR;
        }
        // SAFETY: `user` is the `&mut Collector` passed to `lib_scan` below, which does not
        // return until the scan (and therefore every callback) has finished.
        let collector = unsafe { &mut *user.cast::<Collector>() };
        // SAFETY: the library guarantees `rec` is valid for the duration of the callback.
        let rec = unsafe { &*rec };
        match collector.push(rec) {
            Ok(()) => CB_CONTINUE,
            Err(_) => CB_ERROR,
        }
    });
    outcome.unwrap_or(CB_ERROR)
}
```

`CB_CONTINUE` and `CB_ERROR` are the library's own return codes. Never keep a reference derived from `rec` after the callback returns. If the library may invoke the callback from another thread, `Collector` must be `Send`, whatever the Rust types say. If invocations can overlap, two live `&mut Collector` is undefined behavior: cast to `&Collector` instead and synchronize inside it (a `Mutex`, or atomics).

## Importing C into Rust

Generate declarations with `bindgen` rather than writing them by hand; a wrong signature is silent UB. Wrap the raw API in a type that owns the resource:

```rust
#[repr(C)]
pub struct LzCtx {
    _data: (),
    _marker: PhantomData<(*mut u8, PhantomPinned)>,
}

unsafe extern "C" {
    fn lz_ctx_new() -> *mut LzCtx;
    fn lz_ctx_free(ctx: *mut LzCtx);
    fn lz_compress(ctx: *mut LzCtx, src: *const u8, src_len: usize, dst: *mut u8, dst_cap: usize) -> isize;
}

pub struct Compressor {
    ctx: NonNull<LzCtx>,
}

// SAFETY: liblz documents that a context may be used from any thread, but not from two
// threads at once. Moving it is sound (Send); sharing `&Compressor` across threads is not,
// so there is no Sync impl, and `compress` takes `&mut self`.
unsafe impl Send for Compressor {}

impl Compressor {
    pub fn new() -> Result<Self, LzError> {
        // SAFETY: lz_ctx_new has no preconditions and returns null on allocation failure.
        let raw = unsafe { lz_ctx_new() };
        NonNull::new(raw).map(|ctx| Self { ctx }).ok_or(LzError::Alloc)
    }

    pub fn compress(&mut self, src: &[u8], dst: &mut [u8]) -> Result<usize, LzError> {
        // SAFETY: `ctx` is live until Drop; `&mut self` gives exclusive use; `src` and `dst`
        // are valid for their lengths during the call, and liblz keeps neither pointer.
        let rc = unsafe {
            lz_compress(self.ctx.as_ptr(), src.as_ptr(), src.len(), dst.as_mut_ptr(), dst.len())
        };
        let written = usize::try_from(rc).map_err(|_| LzError::from_code(rc))?;
        if written > dst.len() {
            return Err(LzError::ContractViolation);
        }
        Ok(written)
    }
}

impl Drop for Compressor {
    fn drop(&mut self) {
        // SAFETY: `ctx` came from lz_ctx_new and is freed exactly once, here.
        unsafe { lz_ctx_free(self.ctx.as_ptr()) }
    }
}
```

The opaque struct with a `PhantomData<(*mut u8, PhantomPinned)>` marker is `!Send`, `!Sync` and `!Unpin`, so the raw type cannot be moved across threads by accident; the wrapper opts back in to exactly what the library allows. The return value from C is checked against the buffer size rather than trusted.

## Verifying unsafe code

- Miri: `cargo +nightly miri test` runs tests in an interpreter that reports out-of-bounds access, use-after-free, uninitialized reads, misalignment, invalid enum and bool values, data races and aliasing violations on the paths the tests execute. It cannot call most foreign functions, so keep the pure-Rust unsafe logic (buffers, parsers, pointer arithmetic) separable from the FFI calls and test it under Miri.
- Sanitizers for FFI paths: `RUSTFLAGS=-Zsanitizer=address cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu`. Sanitizers are nightly-only (rustc 1.98 has no stable flag). The explicit `--target` keeps `RUSTFLAGS` away from build scripts and proc macros, which otherwise get instrumented and usually break the build. ASan checks only code it instrumented, so build the C side with `-fsanitize=address` too (for the `cc` crate, `CFLAGS=-fsanitize=address`) if you want errors caught on both sides of the boundary.
- Fuzz any unsafe parser or decoder that sees external bytes (`cargo fuzz`), under a sanitizer.
- A test passing under plain `cargo test` is weak evidence for unsafe code: UB can behave correctly until an optimization or allocator change.
