# ChannelProvider Interface

The `ChannelProvider` is the core abstraction of the DropChannel system. It defines the
complete set of operations any storage medium must support to carry blobs between
participants in a pipeline. All protocols and all participants
(endpoints, nodes) interact with storage exclusively through this interface.

A `ChannelProvider` implementation is transport-agnostic from the protocol's perspective.
Whether it writes to a cloud bucket, a synced folder, a local directory, or proxies
through an HTTP relay is an implementation detail invisible to the protocol layer.

---

## Operations

### Primary slot operations

**`write(channel_id, slot, data) → bool`**
Deposit a blob into a slot. Returns `True` if the blob was written successfully. Returns
`False` if the slot is already occupied (write guard). Implementations must not overwrite
an existing blob — the caller is responsible for checking the return value.

**`read(channel_id, slot) → bytes | None`**
Retrieve and delete a blob from a slot in a single operation (consume-on-read). Returns
the blob bytes if a blob is present, `None` if the slot is empty. Under the Tide
propagation protocol, only the terminating endpoint calls `read()` on a recv-slot. All
other participants use `peek()`.

**`peek(channel_id, slot) → bytes | None`**
Retrieve a blob from a slot without consuming it. Returns the blob bytes if present,
`None` if empty. The blob remains in the slot after this call. Used by nodes and
originating endpoints during the forward pass of the Tide propagation protocol.

**`exists(channel_id, slot) → bool`**
Returns `True` if a blob is present in the slot, `False` if empty. The primary polling
operation during steady-state node operation — one `exists()` call per poll cycle in
each steady-state mode.

**`delete(channel_id, slot) → None`**
Delete a blob from a slot. Idempotent — no error if the slot is already empty. Called
during the Tide ACK cascade by each node to clear its recv-slot after downstream
confirmation is received.

### Meta slot operations

Meta slots are a separate namespace within the same provider position, used exclusively
by the heartbeat observability layer. They do not share state with primary slots.

**`meta_write(channel_id, slot, filename, data) → None`**
Write (or overwrite) a named file in the meta slot namespace.

**`meta_read(channel_id, slot, filename) → bytes | None`**
Read a named file from the meta slot namespace. Returns `None` if not found.

**`meta_delete(channel_id, slot, filename) → None`**
Delete a named file from the meta slot namespace. Idempotent.

**`meta_list(channel_id, slot, prefix="") → list[str]`**
List filenames in the meta slot namespace, optionally filtered by prefix.

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

**Slot isolation.** Primary slot operations must not affect meta slot state, and vice versa.

**No cross-slot coupling.** No operation on one slot must cause a side effect on any other
slot. (The Tide ACK cascade is implemented by participants calling `delete()` explicitly,
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

## Slot Naming

Slot names are arbitrary strings agreed out-of-band between the two endpoint operators.
The provider treats them as opaque identifiers. The recommended convention is to name
slots after the direction of travel:

```
a-to-b    ← written by Endpoint A, read by Endpoint B
b-to-a    ← written by Endpoint B, read by Endpoint A
```

A slot name is invariant across all hops in a physical pipeline. Every participant —
endpoints and nodes alike — uses the same slot name on both their recv and send sides.
