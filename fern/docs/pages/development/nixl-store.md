---
title: NIXL Store
description: Build, test and extend NIXL Store from source.
---

This page covers the Rust workspace in `NB_Nixl-Store`: building services, running tests, configuring nodes and working on the bindings. Run commands from that checkout. For component responsibilities and object lifetimes, start with [NIXL Store](/nixl/user-guide/nixl-store).

## Prerequisites

Use Linux x86_64 for the complete service and Python workflow. The workspace uses Rust edition 2024 and pins Rust 1.95.0. Install a C toolchain, `clang` and `libclang-dev` for bindgen. The types crate obtains protoc through `protoc-bin-vendored`; a separate system protoc installation is not required.

Python builds require CPython 3.12 with development headers and venv support. Install the project's `python/requirements-dev.txt` into a virtual environment for maturin and test tools. The extension is not an arbitrary-version Python wheel: its current package metadata selects Python 3.12.

The pinned NIXL runtime is 1.5.0, matching the workspace's `nixl-sys` dependency. The fetched wheel includes UCX and supports the CPU DRAM path without CUDA or a GPU. UCX supplies the network transport; see [UCX](/nixl/user-guide/nixl-backends/ucx) for transport setup. A successful Rust compilation alone does not verify that runtime libraries and plugins can be loaded.

<!-- src: NB_Nixl-Store/Cargo.toml; rust-toolchain.toml; crates/types/build.rs; pyproject.toml; ci/fetch-nixl.sh -->

## Getting NIXL

```bash
ci/fetch-nixl.sh .nixl
export LD_LIBRARY_PATH="$PWD/.nixl/lib/.nixl_cu13.mesonpy.libs${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export NIXL_PLUGIN_DIR="$PWD/.nixl/lib/.nixl_cu13.mesonpy.libs/plugins"
```

The script downloads a pinned wheel and verifies its SHA-256 checksum before extracting libraries. It reuses a cached wheel only when the checksum matches. Keep the runtime version aligned with `nixl-sys` when updating dependencies.

For a local NIXL build, follow [Building NIXL from Source](/nixl/developer-guide/building-nixl-from-source). Point the runtime library and plugin paths to that build's installed libraries instead of the wheel extraction. Use a compatible NIXL version and include UCX. The store's default stub-based build loads NIXL at runtime; changing these paths changes the loaded implementation, not the store API.

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
| `crates/nsconnector-test-peer` | Service-free connector test executable |
| `crates/python-bindings` | Native Python extension |
| `python/nixl_store` | Python package included by maturin |
| `bench` | Benchmark executable and launch scripts |

The Manager protocol lives in `crates/types/proto/manager.proto` and is compiled by prost-build. Component notes live in `docs/`; the older Python design there has superseded signatures. The Python binding guide lives in `crates/python-bindings/README.md`. The IPC-backed node and connector workflow depends on Linux; do not infer whole-workspace portability from platform-independent metadata logic.

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
| NSClient | `--socket` | Required here or in TOML |
| NSClient | `--shutdown-grace` | `10` seconds |

Manager placement policy, retained-generation limit and metadata shard counts are `ManagerConfig` settings, not command-line flags. Defaults are local-first placement, a retained-generation limit of two and sixteen shards for each metadata table. Change code configuration deliberately rather than inventing runtime flags.

Copy `crates/nsclient/nsclient.example.toml` and pass it with `--config`. Command-line values override corresponding top-level keys. Configure `[staging]` classes, `[[global_segment]]` capacity and object bounds, `[orchestration]` admission and transfer progress, and `[listener]` attachment limits. Unknown keys are rejected.

Sizes accept byte counts or `KiB`, `MiB`, `GiB` strings. Durations accept `s`, `ms`, `us`; page sizes accept `standard`, `2MiB`, `1GiB`. Each staging class requires a count and size; each Global Segment requires a length. Check both staging and storage limits when choosing the largest supported object.

Wait for `Manager Global business service started` before attaching nodes. Confirm NSClient startup and attachment through its logs and a successful connector operation. Set `RUST_LOG=debug` to increase the tracing level; the default is `info`, using formatted tracing output. Redirect process output when you need persistent logs. The connector test peer is a substitute test endpoint, not a process to connect to an already running NSClient.

<!-- src: NB_Nixl-Store/crates/manager/src/main.rs; crates/manager/src/service.rs; crates/nsclient/src/main.rs; crates/nsclient/nsclient.example.toml; crates/logging/src/lib.rs -->

### Running Across Multiple Nodes

Bind the Manager to a reachable interface and point both NSClients at that IP and port. Each application still uses its node-local Unix socket. Permit the Manager TCP connection and the peer network traffic required by UCX. Do not assume opening the Manager port also permits payload transfers.

Write a key on node A, then read it on node B using the same namespace and compare its bytes. Keep the key until the consumer finishes. The [Quick Start](/nixl/user-guide/nixl-store/quick-start) demonstrates the operations; split its write and read across applications for this check.

### Benchmarking

Build release binaries, then run `bench/target.sh` on the storage node and `bench/initiator.sh --target IP` on the other node. The target runs Manager and storage; the initiator runs an NSClient without storage so transfers cross the network. Supply matching `--value-size` and `--keys` values.

`--warmup` excludes initial iterations, while `--threads` controls concurrent operations sharing a connector. The scripts share argument handling through `bench/params.sh`. Output reports mean and percentile call latency, operations per second and value-byte bandwidth. Setup and cleanup are excluded from each timed operation. Use `bench/README.md` for the full parameter table; benchmark results depend on the network and configuration and are not service guarantees.

## Testing

With runtime paths configured:

```bash
cargo build --locked -p nixl-store-nsconnector-test-peer
export NSCONNECTOR_TEST_PEER="$PWD/target/debug/nixl-store-nsconnector-test-peer"
export NIXL_STORE_REQUIRE_NIXL=1
cargo test --workspace --locked
cargo test --locked -p nixl-store-manager
.venv/bin/python -m pytest python/tests -q
scripts/test-python-e2e.sh
scripts/test-python-msrv.sh
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked --no-deps -- -D warnings
```

The NSClient binary integration test starts real services and exercises connector operations, callbacks and batches. Run it with `cargo test --locked -p nixl-store-nsclient --test binary`.

Cargo runs unit and integration tests; append a test-name filter to select one. Real transfer and binary tests require runtime NIXL/UCX libraries. Metadata-only unit tests and the Python tests using the explicit connector test peer do not need running store services. End-to-end Python tests exercise the real services. The MSRV script checks the Python extension against the minimum supported Rust toolchain.

CI runs format, strict Clippy, build and workspace test jobs on merge requests and the default branch. The configured runners use `perflab`, `os/linux`, `type/docker` tags. Formatting checks source layout; Clippy checks compiler and lint diagnostics, including documented unsafe blocks. Neither replaces transfer tests.

<!-- src: NB_Nixl-Store/.gitlab-ci.yml; scripts/test-python-e2e.sh; scripts/test-python-msrv.sh -->

## Bindings

Rust uses `Connector::connect(Options)`, `put`, `upsert`, `get`, `get_with`, `size_query`, `clear`, the batch methods, `register_auto_copy` and `shutdown`. An `Ok` operation result can still contain a store refusal; `Err(LocalFailure)` indicates local failure. Inspect the status and, for reads, the destination disposition.

`Options` fixes the socket and namespace. Defaults include 64 active calls, 64 queued calls, 4096-byte keys, 1024-byte namespaces and 4096 batch items. Connect timeout is five seconds; operation and callback timeouts are thirty seconds. Microbatches default to 64 items and 65,536 encoded bytes.

<!-- src: NB_Nixl-Store/crates/nsconnector/src/config.rs; crates/nsconnector/src/session.rs -->

Python applications connect with `nixl_store.Connector(socket, namespace)` and close the connection with `close()`. The namespace is fixed for that connector.

<!-- src: Python API signatures supplied by documentation author from the RFC; publication follows implementation landing. -->

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
| `PUT_REJECT` | Put cannot create the requested binding |
| `INSUFFICIENT_CAPACITY` | No suitable storage capacity |
| `VALUE_TOO_LARGE` | Value exceeds supported staging size |
| `TRANSFER_FAILED` | Payload transfer failed |
| `OUTCOME_UNKNOWN` | Final outcome cannot be established |
| `BUSY` | Admission capacity unavailable |
| `CLOSED` | Attachment closed |
| `DEADLINE` | Deadline reached |

Handle reads, asynchronous data methods, GPU buffers and placement-descriptor read hints are future capabilities. SSD support is also planned; no release date or persistence guarantee is implied.

<!-- src: NB_Nixl-Store/crates/types/src/manager_rpc.rs; crates/nsclient/tests/binary.rs -->

## Protocol and Compatibility

The Manager handshake carries protocol version `1`. Treat wire messages and descriptor layouts as compatibility boundaries: update both endpoints and their tests together. A version field does not make mixed-version deployments safe. Do not roll out a changed descriptor layout assuming an older process can interpret it.

## Extending NIXL Store

Start allocator changes in `crates/allocator` and placement changes in `crates/manager/src/placement.rs`. Preserve allocation alignment, ownership and free semantics. Test exhaustion and replacement as well as successful allocation. A new memory type or copy mode also needs registration, transfer, buffer validation and binding changes; an API enum alone does not implement support.

Every extension must preserve visibility after commit, prevent reuse while referenced and keep payload bytes out of the Manager. Exercise failure paths where a transfer's final outcome is uncertain. Review protocol and public-contract changes together before changing either side's assumptions.

## Debugging

Enable `RUST_LOG=debug` and correlate service, request and operation identifiers across node and Manager logs. Inspect catalog and slot state through code-level diagnostics or tests; do not assume a public administrative inspection command exists.

For connection errors, check the Manager address, listening interface and route. For attachment errors, check the socket path, same-user admission and handshake timeout. For library loading failures, check both runtime environment variables. For capacity failures, compare Global Segment capacity and object limits with staging slot sizes. Adding storage does not fix a value that cannot fit staging.

## Contributing

Use the source repository's issue and merge-request workflow. Preserve SPDX headers, follow the workspace's Rust formatting and unsafe-code rules, and include tests for protocol and lifetime changes. For documentation in NIXL, follow the [Contribution Guide](/nixl/developer-guide/contribution-guide), including DCO sign-off. Keep implementation claims tied to code and clearly label future contracts.
