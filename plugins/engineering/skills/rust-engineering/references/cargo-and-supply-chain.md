# Cargo, lints, features and supply chain

Read this when configuring lints or CI for a Rust crate or workspace, adding or changing feature flags, raising the MSRV, adding a dependency, or reviewing a dependency's build script or proc macro.

## Lints that catch the failure catalogue

Clippy's default groups miss most of the panic and truncation bugs because the lints that find them live in `restriction` and `pedantic`, which are opt-in. Configure them once in the workspace root (`[lints]` needs Rust 1.74+):

```toml
# Cargo.toml at the workspace root
[workspace.lints.rust]
unsafe_op_in_unsafe_fn = "deny"

[workspace.lints.clippy]
unwrap_used = "deny"
expect_used = "warn"
indexing_slicing = "warn"
string_slice = "warn"
arithmetic_side_effects = "warn"
cast_possible_truncation = "deny"
cast_sign_loss = "deny"
cast_possible_wrap = "deny"
undocumented_unsafe_blocks = "deny"
await_holding_lock = "deny"
```

```toml
# each member's Cargo.toml
[lints]
workspace = true
```

Where this is too strict for a crate (a CLI where `expect` on argument parsing is fine, test code, benchmarks), relax it in that crate or module with a stated reason rather than removing it globally (the `reason` field needs Rust 1.81):

```rust
#[allow(clippy::indexing_slicing, reason = "indices come from enumerate() over the same slice")]
```

When a lint group is enabled and one member lint is then overridden, give the group a lower `priority` so the specific setting wins: `pedantic = { level = "warn", priority = -1 }`.

Treat a new `#[allow]` without a reason in a diff as a finding to question, not a formality.

## CI commands

```sh
cargo fmt --all --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo test --workspace --locked
cargo hack check --workspace --each-feature --locked
cargo hack check --workspace --rust-version --locked
cargo deny check
```

- `--locked` fails the build if `Cargo.lock` would change, so CI tests exactly what was reviewed.
- `cargo hack --each-feature` checks every feature alone plus `--no-default-features` and the default set; `--feature-powerset --depth 2` adds pairs. Unlike plain cargo, it builds each package in its own invocation, which is what exposes a crate relying on features some other workspace member happened to enable.
- `cargo hack --rust-version` builds on the toolchain named in `package.rust-version`, which catches std APIs newer than the declared MSRV.
- Crates with `unsafe` code add a nightly job running `cargo +nightly miri test` on the tests that exercise it (see `unsafe-ffi.md`).

## Feature flags

Cargo builds each dependency once, with the union of all features any package in the build requested. Consequences:

- Features must be additive. `feature = "postgres"` and `feature = "sqlite"` that select different implementations of one type break the moment two crates in the graph pick different ones. If exclusivity is unavoidable, fail loudly with `#[cfg(all(feature = "a", feature = "b"))] compile_error!(...)`.
- A crate that uses `tokio::fs` must enable `tokio/fs` itself. If it compiles only because a sibling enables it, `cargo build -p` on that crate and every downstream user will fail.
- A feature that disables a security check (say, a crate's own `skip-signature-check` feature for tests) can be turned on by any crate in the graph, and then it is on for everyone. Keep such switches out of libraries, or make them runtime configuration that the application owns.
- `default-features = false` on a dependency does nothing if another crate in the graph enables defaults. Check with `cargo tree -e features -i <crate>`.

## MSRV and editions

- `package.rust-version` is the declared minimum. Using a std API newer than it (see the table in SKILL.md) is a breaking change for users on that toolchain.
- Edition 2024 requires Rust 1.85 and implies `resolver = "3"`, which prefers dependency versions whose own `rust-version` is compatible with yours. Older editions can opt in through `resolver.incompatible-rust-versions = "fallback"` in `.cargo/config.toml`.
- Moving an existing crate to edition 2024 changes behavior in places the compiler does not flag as errors: temporaries in `if let` scrutinees now drop before `else`, and tail-expression temporaries drop earlier. Run `cargo fix --edition`, then review lock guards and borrows in those positions by hand.

## Adding a dependency

Before adding a crate, check:

- Whether the workspace already depends on something that does the job (`cargo tree -i <candidate>`, and `cargo tree -d` for crates already present in two versions).
- The exact name on crates.io, the linked repository, recent releases and maintainers. Typosquats of popular crates exist.
- Whether it has a `build.rs` or is a proc macro, and what that code does.
- The features you need; disable defaults you do not.
- That `cargo deny check` passes with it: advisories, licenses, sources and bans.

### Reviewing a build script or proc macro

Both run with the permissions of whoever builds: a developer laptop with SSH keys and cloud credentials, or a CI runner with tokens in its environment. Things that need a reason:

- Network access (downloading binaries or sources at build time).
- Reading files outside the crate and `OUT_DIR`, or reading the environment beyond `CARGO_*` and documented variables.
- Writing outside `OUT_DIR`.
- Spawning processes other than the C compiler or `pkg-config` path the crate documents.
- Obfuscated or minified code, or embedded binaries.

Keep deploy and publish credentials out of the CI jobs that compile untrusted or newly updated dependencies; give those jobs only what building needs.

## cargo-deny

```toml
# deny.toml
[advisories]
yanked = "deny"
ignore = [
    # { id = "RUSTSEC-YYYY-NNNN", reason = "why this does not affect us, and until when" },
]

[licenses]
allow = ["MIT", "Apache-2.0"]

[bans]
multiple-versions = "warn"
wildcards = "deny"

[bans.build]
allow-build-scripts = [
    # { name = "ring" },
]

[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

- `advisories` checks the RustSec database for vulnerable, unmaintained and yanked crates. Every `ignore` entry carries a reason; an ignore without one is a finding.
- `licenses` denies anything not in `allow`. Adjust the list to the project's actual policy.
- `bans.build.allow-build-scripts`, when present, lists the crates allowed to have a build script; everything else fails. It turns "a transitive dependency grew a build.rs" into a reviewed change.
- `sources` with `unknown-registry` and `unknown-git` set to `deny` stops a dependency from being pulled from an unexpected git URL or registry.

`cargo audit` covers the advisories check alone and is fine where cargo-deny is not set up.

## Lock files and installs

- Applications commit `Cargo.lock` and build with `--locked`. Dependency updates (`cargo update`) go in their own reviewed change, not mixed into feature work.
- `cargo install` ignores the package's lock file and resolves fresh versions unless given `--locked`. Tooling installed in CI uses `--locked` and a pinned version.
