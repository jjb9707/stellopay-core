# Workspace Clippy policy

The contracts workspace is checked with the same command locally and in
Contracts CI:

```text
cargo clippy --workspace --all-targets -- -D warnings
```

The `--workspace` and `--all-targets` flags are intentional. They cover all
seven workspace crates, their library and binary targets, integration tests,
unit tests, and benchmark targets. A lint warning is a failed check; warnings
must not be hidden by a workspace-wide `allow`.

## Baseline and cleanup

The initial warning-free pass exposed several categories of work:

- Rustdoc continuation warnings in the RBAC interface documentation.
- An unnecessary numeric cast in payroll milestone handling.
- Deprecated Soroban test registration calls in contract and integration
  tests.
- needless borrows, avoidable `map_or` expressions, and avoidable `if let`
  error forwarding in payroll code.
- test-only unused imports, unused bindings, and assertions that did not
  assert a behavior.
- one legitimate complexity warning from the generated price-oracle
  configuration ABI.

All reported call sites were corrected or modernized. The cleanup does not
disable a lint group and does not add a crate-level `allow`. The few
function-level `dead_code` annotations in payroll document Soroban interface
traits consumed by the contract macro; they are narrow and do not suppress
the workspace lint gate. The existing test fixtures retain their own
pre-existing compatibility annotations where required by the SDK.

## Configuration

`clippy.toml` keeps the repository's existing complexity and MSRV settings.
The only threshold adjustment is `too-many-arguments-threshold = 11`. The
published `configure_pair` price-oracle ABI has ten flat configuration fields,
and Soroban generates a wrapper with that exact argument shape. The threshold
is therefore one above the current ABI rather than a general exemption for
large functions; newly authored functions with more arguments still fail.

## Verification

Before opening a pull request, run:

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace
cargo build --workspace --release --target wasm32-unknown-unknown
```

The release WASM comparison from this change is:

| Contract | Baseline | Current | Delta |
| --- | ---: | ---: | ---: |
| `multisig` | 41,686 | 41,686 | 0 |
| `price_oracle` | 220,399 | 219,946 | -453 (-0.206%) |
| `rbac` | 22,456 | 22,456 | 0 |
| `stello_pay_contract` | 189,699 | 189,018 | -681 (-0.359%) |

The size values are compared with `benchmarks/wasm_sizes.json`. Artifact
generation remains a separate CI step so a compiler failure and a size-policy
failure are visible independently.
