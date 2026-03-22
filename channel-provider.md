# DockProvider Interface

The `DockProvider` is the core abstraction of the DropChannel system. It defines the
complete set of operations any storage medium must support to carry blobs between
participants in a pipeline. All protocols and all participants
(endpoints, Rafts) interact with storage exclusively through this interface.

A `DockProvider` implementation is transport-agnostic from the protocol's perspective.
Whether it writes to a cloud bucket, a synced folder, a local directory, or proxies
through an HTTP relay is an implementation detail invisible to the protocol layer.

---

## Operations

### Primary Waterway operations

**`write(channel, waterway, filename, data) → bool`**
Deposit a blob into a Waterway. Returns `True` if the blob was written successfully. Returns
`False` if the Waterway is already occupied (write guard). Implementations must not overwrite
an existing blob — the caller is responsible for checking the return value.

**`read(channel, waterway, filename) → bytes | None`**
Retrieve and delete a blob from a Waterway in a single operation (consume-on-read). Returns
the blob bytes if a blob is present, `None` if the Waterway is empty. Under the Tideway
propagation protocol, only the terminating endpoint calls `read()` on an Upper-side Waterway. All
other participants use `peek()`.

**`peek(channel, waterway, filename) → bytes | None`**
Retrieve a blob from a Waterway without consuming it. Returns the blob bytes if present,
`None` if empty. The blob remains in the Waterway after this call. Used by Rafts and
originating endpoints during the forward pass of the Tideway propagation protocol.

**`exists(channel, waterway, filename) → bool`**
Returns `True` if a blob is present in the Waterway, `False` if empty. The primary polling
operation during steady-state Raft operation — one `exists()` call per poll cycle in
each steady-state mode.

**`delete(channel, waterway, filename) → None`**
Delete a blob from a Waterway. Idempotent — no error if the Waterway is already empty. Called
during the Tideway ACK cascade by each Raft to clear its Upper-side Waterway after downstream
confirmation is received.

### Meta Waterway operations

Meta Waterways are a separate namespace within the same provider position, used exclusively
by the heartbeat observability layer. They do not share state with primary Waterways.

**`meta_write(channel, waterway, filename, data) → None`**
Write (or overwrite) a named file in the meta Waterway namespace.

**`meta_read(channel, waterway, filename) → bytes | None`**
Read a named file from the meta Waterway namespace. Returns `None` if not found.

**`meta_delete(channel, waterway, filename) → None`**
Delete a named file from the meta Waterway namespace. Idempotent.

**`meta_list(channel, waterway) → list[str]`**
List filenames in the meta Waterway namespace.

---

## Contracts

**Write guard.** `write()` must return `False` rather than overwriting an existing blob.
The check-before-write sequence has an inherent race window on shared storage — atomicity
is best-effort on all current providers. Atomic write guards (compare-and-swap, conditional
PUT) are a future concern.

**Idempotent delete.** `delete()` and `meta_delete()` must not raise if the target is
already absent.

**Consume-on-read.** `read()` must delete the blob as part of the same operation. A blob
that has been returned to the caller must not be returned again by a subsequent `read()`.

**Waterway isolation.** Primary Waterway operations must not affect meta Waterway state, and vice versa.

**No cross-Waterway coupling.** No operation on one Waterway must cause a side effect on any other
Waterway. (The Tideway ACK cascade is implemented by participants calling `delete()` explicitly,
not by the provider.)

---

## Provider Implementations

The Python reference implementation (`dropchannel-py`) provides the following:

| Provider key | Mechanism | Infrastructure required |
|---|---|---|
| `gcs` | Google Cloud Storage objects | GCS bucket |
| `httprelay` | HTTP relay server (proxies to GCS) | Cloud Run + GCS bucket |
| `dropbox` | Local filesystem via Dropbox sync | Shared Dropbox folder |
| `local` | Local filesystem, same machine | Local directory |

Adding a new provider requires: implementing this interface, registering the provider
in the factory, and documenting its required configuration. No changes to any protocol
implementation or participant code are needed.

---

## Waterway Naming

Waterway names are arbitrary strings agreed out-of-band between the two endpoint operators.
The provider treats them as opaque identifiers. The recommended convention is to name
Waterways after the direction of travel:

```
a-to-b    ← written by Endpoint A, read by Endpoint B
b-to-a    ← written by Endpoint B, read by Endpoint A
```

A Waterway name is invariant across all hops in a physical pipeline. Every participant —
endpoints and Rafts alike — uses the same Waterway name on both their Upper and Lower sides.
