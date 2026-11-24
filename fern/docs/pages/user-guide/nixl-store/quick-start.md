---
title: Quick Start
description: Start the services and store and read a value through the Python bindings.
---

This walkthrough starts a Manager, one NSClient and a Python application on a Linux x86_64 node. It uses CPU DRAM and UCX. Run source commands from `runtime/store` in your NIXL checkout. Use separate terminals for the two services and the application.

## Install

Install Rust and the system prerequisites from the [Developer Guide](/nixl/developer-guide/nixl-store). Build the services, then fetch the pinned NIXL runtime libraries:

```bash
cargo build --workspace --locked
ci/fetch-nixl.sh .nixl
export LD_LIBRARY_PATH="$PWD/.nixl/lib/.nixl_cu13.mesonpy.libs${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export NIXL_PLUGIN_DIR="$PWD/.nixl/lib/.nixl_cu13.mesonpy.libs/plugins"
python3.12 -m venv .venv
.venv/bin/python -m pip install -r python/requirements-dev.txt
.venv/bin/maturin build --release --locked --compatibility linux --interpreter .venv/bin/python --out target/python-wheels
.venv/bin/python -m pip install --no-deps target/python-wheels/nixl_store-0.1.0-cp312-cp312-linux_x86_64.whl
```

Export the same library paths in the NSClient terminal. Installing the wheel installs the application library; it does not start either service. The fetched wheel supplies UCX for this CPU path.

## Start the Manager

```bash
./target/debug/nixl-store-manager
```

Wait for `Manager Global business service started`. The default control address is `127.0.0.1:50051`. Leave this terminal running while starting the NSClient.

## Start an NSClient

```bash
./target/debug/nixl-store-nsclient --manager 127.0.0.1:50051 --socket /tmp/nixl-store.sock
```

This uses the built-in memory configuration. The socket path is required, either here or in a configuration file. Run the application as the same operating-system user as the NSClient. Keep the socket path identical in both commands.

## Store and Read a Value

Pass the destination buffer directly to `get`. Its byte length bounds the copy. The default `copy=None` selects the registered copy function or the built-in CPU copy. See [Bindings](/nixl/developer-guide/nixl-store#bindings) for buffer ownership and per-call copy functions.

```bash
.venv/bin/python - <<'PYTHON'
import nixl_store as ns

store = ns.Connector("/tmp/nixl-store.sock", b"quick-start")
try:
    result = store.upsert(b"example", b"hello")
    assert result.status == ns.Status.COMPLETED
    destination = bytearray(5)
    result = store.get(b"example", destination)
    assert result.status == ns.Status.COMPLETED
    assert result.destination_state == ns.DestinationState.COMPLETE
    assert result.bytes_copied == 5
    assert destination == b"hello"
    print(destination.decode())
    assert store.clear(b"example").wait() == ns.Status.COMPLETED
finally:
    store.close()
PYTHON
```

The application prints `hello`. Upsert makes the example repeatable if an earlier run left the key behind. Use put when an existing binding should reject a write. Always inspect the returned status before consuming a read destination; an unsuccessful copy can modify it.

Rust applications use the `nsconnector` crate described in the Developer Guide.

## Teardown

The example waits for clear and closes its connector. Clear completion means the key is invisible to new readers; reclamation can finish later. Stop the NSClient with Ctrl-C, then stop the Manager with Ctrl-C in its terminal. Closing a connector alone does not clear the keys it wrote.
