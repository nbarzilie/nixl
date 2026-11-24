---
title: NIXL Store
description: NIXL Store is a distributed key-value store that uses NIXL to move data between applications.
---

## What is NIXL Store?

NIXL Store gives applications a shared, distributed key-value store that uses [NIXL](/nixl/getting-started/overview) for data transfers. It adds keys, commit-gated visibility and reference-based reclamation to registered memory. Applications can use it to stage objects between producers and consumers, including RL experience transfer and secondary-tier KV-cache pool management. Framework integration remains the application's responsibility.

Producers and consumers can run in different processes and on different nodes. They need a complete object even when production and consumption happen at different times. Keeping a registered pool in a node process lets applications share capacity without allocating and registering a private store for each process.

You connect through a small library API and identify objects by namespace and key. NIXL Store owns placement, visibility and reclamation. NIXL owns memory registration, connections and transfers. The store currently supports CPU DRAM over UCX. SSD support is planned for a future release.

## Key Capabilities

- **Key-value operations** - Put, upsert, get, clear and query an object's size within a connector's namespace.
- **Batch operations** - Submit multiple writes, reads, size queries or clears and receive ordered per-item results.
- **Commit-gated visibility** - A write becomes visible after its transfer completes and the store commits it. Readers never acquire a partially transferred slot.
- **Atomic replacement** - Upsert publishes a new slot while existing readers finish with the previous value.
- **Safe reclamation** - Clear removes a binding before reclaiming memory held by active readers.
- **Shared memory** - Applications attach to the node's pre-registered staging pool.
- **Application-controlled copy** - Use the built-in CPU copy, register a function or select a per-call write or read callback.
- **Rust and Python bindings** - Embed NSConnector in Rust or use the Python package on Linux x86_64 with CPython 3.12.

<Tip>
You do not register store memory or count remote readers yourself. The node process and Manager coordinate these lifetimes. See [Architecture](/nixl/user-guide/nixl-store/architecture).
</Tip>

<Note>
Values must currently fit the largest staging slot. Future versions will support direct transfers from application-owned registered memory and streaming through bounded staging memory.
</Note>

<div className="nixl-stack-diagram">
<div className="nixl-stack-layer"><div className="nixl-stack-layer-title">Application / Framework Adapter</div></div>
<div className="nixl-stack-agent-box">
<div className="nixl-stack-agent-layer nixl-stack-agent-nb"><div className="nixl-stack-agent-layer-title">NSConnector API - Rust / Python</div></div>
<div className="nixl-stack-agent-layer nixl-stack-agent-core"><div className="nixl-stack-agent-layer-title">NSClient</div><div className="nixl-stack-agent-chips"><span className="nixl-stack-chip">Staging</span><span className="nixl-stack-chip">Local Segment</span><span className="nixl-stack-chip">Global Segment</span><span className="nixl-stack-chip">NIXL Agent</span></div></div>
<div className="nixl-stack-agent-layer nixl-stack-agent-sb"><div className="nixl-stack-agent-layer-title">NIXL → UCX → CPU DRAM</div></div>
</div>
<div className="nixl-stack-agent-box"><div className="nixl-stack-agent-layer nixl-stack-agent-core"><div className="nixl-stack-agent-layer-title">Manager - NSClient control path</div><div className="nixl-stack-agent-chips"><span className="nixl-stack-chip">Placement</span><span className="nixl-stack-chip">Slot States</span><span className="nixl-stack-chip">Reference Counts</span><span className="nixl-stack-chip">Peer Catalog</span></div></div></div>
</div>

Payload bytes flow through NSClient staging and NIXL between nodes. The Manager handles metadata only.
