---
title: NIXL Store
description: Build, test and extend NIXL Store from source.
---

This page covers the Rust workspace in `runtime/store`: building services, running tests, configuring nodes and working on the bindings. Run commands from `runtime/store` in your NIXL checkout. For component responsibilities and object lifetimes, start with [NIXL Store](/nixl/user-guide/nixl-store).

## Prerequisites

The services and Rust connector require Linux. The Python package targets Linux x86_64. The Rust services and direct connector API have no x86_64-specific build restriction, but aarch64 builds and runtime behavior are not yet validated by this guide. The workspace uses Rust edition 2024 and pins Rust 1.95.0. Install a C toolchain, `clang` and `libclang-dev` for bindgen. The types crate obtains protoc through `protoc-bin-vendored`; a separate system protoc installation is not required.

Python builds require CPython 3.12 with development headers and venv support. Install the project's `python/requirements-dev.txt` into a virtual environment for maturin and test tools. The extension's current package metadata selects Python 3.12.

NIXL Store is compatible with the NIXL version it is released with. Store and NIXL versions advance together; compatibility testing covers the colocated release, not every combination of versions. The CPU DRAM path uses UCX between data nodes and does not require a GPU. See [UCX](/nixl/user-guide/backend-selection/ucx) for transport setup.

## Building with NIXL

Build NIXL with UCX from the same checkout, following [Building NIXL from Source](/nixl/developer-guide/building-nixl-from-source). Use that build's libraries and plugins when running Store services and transfer tests. Set `LD_LIBRARY_PATH` to the installed library directory and `NIXL_PLUGIN_DIR` to its plugin directory.

Store has a standalone Cargo workspace at `runtime/store/Cargo.toml`, like `examples/rust/Cargo.toml`. It does not require changes to NIXL's root workspace. Its NIXL dependency points to the in-tree Rust bindings:

```toml
nixl-sys = { path = "../../src/bindings/rust" }
```

Keep `runtime/store/Cargo.lock` in sync with this dependency. Building and testing Store against the colocated bindings replaces the separate pinned-wheel download workflow.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `crates/types` | Shared types and generated Manager wire messages |
| `crates/rpc` | RPC transport and sessions |
| `crates/ipc` | Connector attachment and shared-memory protocol |
| `crates/allocator` | Slot allocation |
| `crates/logging` | Shared tracing setup |
| `crates/manager` | Metadata service library and binary |
| `crates/nsclient` | Node runtime library and binary |
| `crates/nsconnector` | Application library |
| `crates/python-bindings` | Native Python extension |
| `python/nixl_store` | Python package included by maturin |
| `bench` | Benchmark executable and launch scripts |

The Manager protocol lives in `crates/types/proto/manager.proto` and is compiled by prost-build. Component notes live in `docs/`. The Python binding guide lives in `crates/python-bindings/README.md`.

## Building

```bash
cargo build --workspace --locked
cargo build --locked -p nixl-store-manager
cargo build --release --locked
python3.12 -m venv .venv
.venv/bin/python -m pip install -r python/requirements-dev.txt
.venv/bin/maturin build --release --locked --compatibility linux --interpreter .venv/bin/python --out target/python-wheels
.venv/bin/python -m pip install --no-deps target/python-wheels/nixl_store-0.1.0-cp312-cp312-linux_x86_64.whl
```

Use debug builds for development and release builds for benchmarks. Service binaries land in `target/debug` or `target/release`. Maturin enables the extension's `extension-module` feature through `pyproject.toml`. For an editable development installation, activate the virtual environment and run `maturin develop --release --locked` instead of building and installing a wheel.

## Running from Source

Run these in separate terminals, with the runtime paths exported for the NSClient:

```bash
./target/debug/nixl-store-manager
```

```bash
./target/debug/nixl-store-nsclient --socket /tmp/nixl-store.sock
```

| Process | Option | Default |
| --- | --- | --- |
| Manager | `--listen` | `127.0.0.1:50051` |
| Manager | `--max-connections` | `128` |
| Manager | `--request-queues` | `1` |
| NSClient | `--config` | No file |
| NSClient | `--manager` | `127.0.0.1:50051` |
| NSClient | `--socket` | No default |
| NSClient | `--shutdown-grace` | `10` seconds |

Set the connector socket path with `--socket PATH` or the top-level `socket` key in the TOML file. Startup fails if neither is set. A command-line socket path overrides the file.

Manager placement policy, retained-generation limit and metadata shard counts are `ManagerConfig` settings, not command-line flags. Defaults are local-first placement, a retained-generation limit of two and sixteen shards for each metadata table.

Save this configuration as `nsclient.toml` in `runtime/store`:

```toml
manager = "127.0.0.1:50051"
socket = "/tmp/nixl-store.sock"
shutdown_grace = "10s"

[staging]
page_size = "standard"

[[staging.classes]]
count = 16
size = "64KiB"

[[global_segment]]
length = "2MiB"
min_object = 1
max_object = "64KiB"
alignment = 64

[orchestration]
shard_count = 4
max_sessions = 64
staging_wait_deadline = "60s"
transfer_deadline = "30s"

[listener]
handshake_deadline = "5s"
max_handshakes = 64
```

```bash
./target/debug/nixl-store-nsclient --config nsclient.toml
```

This example provides sixteen staging slots and one Global Segment. `[orchestration]` controls admission and transfer progress; `[listener]` controls connector attachment. The annotated `crates/nsclient/nsclient.example.toml` lists all supported keys and defaults. Command-line values override matching top-level keys. Unknown keys are rejected.

Sizes accept byte counts or `KiB`, `MiB`, `GiB` strings. Durations accept `s`, `ms`, `us`; page sizes accept `standard`, `2MiB`, `1GiB`. Each staging class requires a count and size; each Global Segment requires a length. Check both staging and storage limits when choosing the largest supported object.

Wait for `Manager Global business service started` before attaching nodes. Confirm NSClient startup and attachment through its logs and a successful connector operation. Set `RUST_LOG=debug` to increase the tracing level; the default is `info`. Redirect process output when you need persistent logs.

### Running Across Multiple Nodes

Bind the Manager to a reachable interface and point both NSClients at that IP and port. Each application still uses its node-local Unix socket. Control traffic between each NSClient and the Manager uses TCP only. Payload transfers use NIXL/UCX between NSClients on the data nodes. Permit the Manager TCP port and configure the data-node network for the selected UCX transport; the Manager does not participate in UCX payload transfers.

Write a key on node A, then read it on node B using the same namespace and compare its bytes. Keep the key until the consumer finishes. The [Quick Start](/nixl/user-guide/nixl-store/quick-start) demonstrates the operations; split its write and read across applications for this check.

### Benchmarking

Build release binaries, then run `bench/target.sh` on the storage node and `bench/initiator.sh --target IP` on the other node. The target runs Manager and storage; the initiator runs an NSClient without storage so transfers cross the network. Supply matching `--value-size` and `--keys` values.

`--warmup` excludes initial iterations, while `--threads` controls concurrent operations sharing a connector. The scripts share argument handling through `bench/params.sh`. Output reports mean and percentile call latency, operations per second and value-byte bandwidth. Setup and cleanup are excluded from each timed operation. Use `bench/README.md` for the full parameter table; benchmark results depend on the network and configuration and are not service guarantees.

## Testing

Run formatting and lint checks from `runtime/store`:

```bash
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked --no-deps -- -D warnings
```

For the complete workspace test suite, configure the colocated NIXL/UCX runtime paths first. Set `NIXL_STORE_REQUIRE_NIXL=1` so tests fail when NIXL is unavailable instead of skipping checks that require it:

```bash
export NIXL_STORE_REQUIRE_NIXL=1
cargo test --workspace --locked
scripts/test-python-e2e.sh
scripts/test-python-msrv.sh
```

Cargo runs unit and integration tests; append a test-name filter to select one. The NSClient binary integration test starts real services and exercises connector operations, callbacks and batches. Run it alone with `cargo test --locked -p nixl-store-nsclient --test binary`. Python end-to-end tests exercise the services through the bindings. The MSRV script checks the extension against the minimum supported Rust toolchain.

CI separates fast checks from native transfer coverage:

| Tier | Trigger | Coverage |
| --- | --- | --- |
| Fast gate - GitHub Actions | PR changes under `runtime/store/**` | Formatting, Clippy and unit tests from the Store workspace, without building NIXL/UCX or requiring `/build` |
| Native suite - Jenkins | `/build` | Store NIXL/UCX tests and full end-to-end coverage against prebuilt NIXL and UCX libraries |
| Cross-project smoke test | Regular `/build` on NIXL PRs | One Store end-to-end smoke test to detect integration breakage without the full Store suite |

The fast gate is planned in `.github/workflows/store-checks.yml`, with a feedback target of about two minutes. Native tests integrate with NIXL's Rust test harness. The CI toolchain uses Rust 1.95 for Store. Ordinary NIXL changes should use the Store smoke test rather than the entire Store suite.

## Bindings

Rust uses `Connector::connect(Options)`, `put`, `upsert`, `get`, `get_with`, `size_query`, `clear`, the batch methods, `register_auto_copy` and `shutdown`. An `Ok` operation result can still contain a store refusal; `Err(LocalFailure)` indicates local failure. Inspect the status and, for reads, the destination disposition.

`Options` fixes the socket and namespace. Defaults include 64 active calls, 64 queued calls, 4096-byte keys, 1024-byte namespaces and 4096 batch items. Connect timeout is five seconds; operation and callback timeouts are thirty seconds. Microbatches default to 64 items and 65,536 encoded bytes.

Python applications connect with `nixl_store.Connector(socket, namespace)` and close the connection with `close()`. The namespace is fixed for that connector.

```text
put(key, value: Buffer, /, *, copy: Copy = None) -> PutResult
upsert(key, value: Buffer, /, *, copy: Copy = None) -> PutResult
get(key, dest: Buffer, /, *, copy: Copy = None) -> GetResult
put_many([PutItem(key, value: Buffer)], *, copy: Copy = None) -> list[PutResult]
upsert_many([PutItem(key, value: Buffer)], *, copy: Copy = None) -> list[PutResult]
get_many([GetItem(key, dest: Buffer)], *, copy: Copy = None) -> list[GetResult]
size_query_many(keys) -> list[SizeResult]
clear_many(keys) -> Future[list[Status]]
clear(key) -> Future[Status]
size_query(key) -> int
register_copy_fn(src_device, dst_device, fn: mem_copy_f | None) -> None
BufferRef.from_buffer(obj)
BufferRef.from_address(ptr, nbytes, device)
mem_copy_f(dst: BufferRef, src: BufferRef, nbytes) -> Status
```

A `Buffer` is a C-contiguous CPU buffer-protocol object or a `BufferRef`; reads require writable storage. `copy=None` uses the registered device-pair function or built-in copy. `copy=function` bypasses registration for that call. The function receives destination, source and byte count, and returns a status. It must finish synchronously, retain no callback buffer references and avoid reentry into the same connector.

Successful put and upsert calls block until the key is visible to all consumers. Reads write at most the destination buffer's byte length (`nbytes`). `PutResult` includes status and an optional placement descriptor. `GetResult` includes status, destination state and bytes copied when complete. `SizeResult` includes status and optional size. Size-query failure raises `StoreError` with `.status`; it is not atomic with a later read. Clear returns a Future that resolves when the binding is invisible.

Batches preserve input order with per-item results. Duplicate keys receive `DUPLICATE_KEY`; exceeding the batch limit raises `StoreError(INVALID_INPUT)` before submission. Overlapping read destinations raise `ValueError` without submission. These are admission rules, not a promise that a whole batch commits atomically.

| Status | Meaning |
| --- | --- |
| `COMPLETED` | Operation completed successfully |
| `MISSING_KEY` | No visible binding |
| `PUT_REJECT` | The key is already bound |
| `INSUFFICIENT_CAPACITY` | No suitable storage capacity |
| `VALUE_TOO_LARGE` | Value exceeds supported staging size |
| `TRANSFER_FAILED` | Payload transfer failed |
| `OUTCOME_UNKNOWN` | Final outcome cannot be established |
| `BUSY` | Admission capacity unavailable |
| `CLOSED` | Attachment closed |
| `DEADLINE` | Deadline reached |

Handle reads, asynchronous data methods, GPU buffers, placement-descriptor read hints and SSD support are future capabilities.

## Protocol and Compatibility

The Manager handshake carries a protocol version. Treat wire messages and descriptor layouts as compatibility boundaries: update both endpoints and their tests together. A version field does not make mixed-version deployments safe. Do not roll out a changed descriptor layout assuming an older process can interpret it.

## Extending NIXL Store

Start allocator changes in `crates/allocator` and placement changes in `crates/manager/src/placement.rs`. Preserve allocation alignment, ownership and free semantics. Test exhaustion and replacement as well as successful allocation. Add tests and benchmarks for changed behavior. Every optimization or modification should include benchmark results showing no performance degradation under comparable configurations.

Every extension must preserve visibility after commit, prevent reuse while referenced and keep payload bytes out of the Manager. Exercise failure paths where a transfer's final outcome is uncertain. Review protocol and public-contract changes against the component documentation in `runtime/store/docs/` and update the affected documents and tests together.

## Debugging

Enable `RUST_LOG=debug` and correlate service, request and operation identifiers across node and Manager logs. Inspect catalog and slot state through code-level diagnostics or tests.

For connection errors, check the Manager address, listening interface and route. For attachment errors, check the socket path, same-user admission and handshake timeout. For library loading failures, check both runtime environment variables. For capacity failures, compare Global Segment capacity and object limits with staging slot sizes.

## Contributing

File issues and pull requests in the [NIXL GitHub repository](https://github.com/ai-dynamo/nixl). Preserve SPDX headers, follow the workspace's Rust formatting and unsafe-code rules, and include tests for protocol and lifetime changes. For documentation in NIXL, follow the [Contribution Guide](/nixl/developer-guide/contribution-guide), including DCO sign-off.
