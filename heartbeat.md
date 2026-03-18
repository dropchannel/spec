# Heartbeat

**Status:** Specified (introduced in dropchannel-py v0.6)  
**Prefix:** `heartbeat-`  
**Layer:** Protocol liveness — intra-pipeline observability  
**Related:** `observability.md`, `telemetry.md`, `channel-provider.md`, `protocol-registry.md`

---

## Purpose

The heartbeat protocol provides continuous, per-hop liveness signaling within a running
DropChannel pipeline. It operates independently of primary payload traffic and introduces
no coupling to primary slot semantics.

Heartbeat addresses two observability gaps present in Winch pipelines without it:

1. A stalled pipeline is indistinguishable from a pipeline with an offline participant.
2. In a multi-hop chain, there is no mechanism to identify which specific hop is
   responsible for a stall.

Both gaps are resolved entirely within the existing `ChannelProvider` interface via a
meta slot layer. No new storage operations are introduced. No out-of-band channels are
required.

**Heartbeat is not:**

- A payload transport. Heartbeat files carry no application data.
- A telemetry channel. Heartbeat operates within the pipeline's own storage positions;
  telemetry uses a dedicated side-channel. See `observability.md` for the distinction.
- An external monitoring dependency. Heartbeat state is readable by participants in the
  pipeline without any external tooling.

---

## Meta slots

Each node and client gains two additional storage namespaces alongside its existing
primary slots:

```
recv-side:  [ primary recv-slot ]  [ meta recv-slot ]
send-side:  [ primary send-slot ]  [ meta send-slot ]
```

Meta slots are logically separate namespaces within the same `ChannelProvider` position.
They hold only heartbeat files. There is no coupling, sequencing dependency, or shared
state between the primary and meta layers. Primary slot behavior is entirely unaffected
by any meta slot state.

Meta slot storage representations per provider are documented in `channel-provider.md`.

---

## Participant identity

Each node and client instance has a persistent UUID, generated on first startup and
reloaded on all subsequent startups. This UUID is the stable identity used in heartbeat
file naming and content.

UUID format: standard UUID4.

---

## Heartbeat file naming

```
heartbeat-node-{UUID}      # written by a node
heartbeat-client-{UUID}    # written by a client (endpoint)
```

One file per participant, keyed by UUID. Files are written to meta slots using
overwrite semantics — each write replaces the previous file at that name.

---

## Heartbeat content format

### Node heartbeat

```
{upstream_content}
Node_{UUID}:{timestamp}:{alive_s}
```

### Client heartbeat

```
Client_{UUID}:{timestamp}:{alive_s}:{status}:{status_s}
```

### Field definitions

| Field            | Type    | Description |
|------------------|---------|-------------|
| `UUID`           | string  | Persistent UUID for this participant instance. |
| `timestamp`      | integer | Unix timestamp (seconds) at time of write. |
| `alive_s`        | integer | Seconds since this participant started its current session. |
| `status`         | string  | Client only. One of: `ready`, `busy`, `closing`. See [Client status](#client-status). |
| `status_s`       | integer | Client only. Seconds since the current status was set. |
| `upstream_content` | string | Node only. The full contents of the most recent non-self heartbeat file found on the relevant meta slot, or the literal string `HEARTBEAT_NOT_FOUND`. |

`upstream_content` is prepended to the node's own line, making each node's heartbeat
file a cumulative chain of all upstream participant heartbeats it has observed.

---

## Node heartbeat cycle

Each node runs a heartbeat cycle at interval `HEARTBEAT_INTERVAL` (HI), concurrently
with and independently of the primary slot state machine.

**Step 1 — Read send-side meta, write recv-side meta:**

```
Find any file matching heartbeat-node-* or heartbeat-client-*
on the send-side meta slot where filename != heartbeat-node-{UUID_self}

If found:   content = contents of that file
If not:     content = "HEARTBEAT_NOT_FOUND"

Write to recv-side meta slot:
  filename: heartbeat-node-{UUID_self}
  content:  content + "\n" + "Node_" + UUID_self + ":" + timestamp + ":" + alive_s
```

**Step 2 — Read recv-side meta, write send-side meta:**

```
Find any file matching heartbeat-node-* or heartbeat-client-*
on the recv-side meta slot where filename != heartbeat-node-{UUID_self}

If found:   content = contents of that file
If not:     content = "HEARTBEAT_NOT_FOUND"

Write to send-side meta slot:
  filename: heartbeat-node-{UUID_self}
  content:  content + "\n" + "Node_" + UUID_self + ":" + timestamp + ":" + alive_s
```

**Step 3 — Sleep HI. Repeat from Step 1.**

If more than one non-self heartbeat file is found on a given meta slot, the node
selects the most recently written and logs a warning. This condition should not arise
in a correctly configured pipeline.

---

## Client heartbeat

Clients write only their own `heartbeat-client-{UUID}` file. Clients do not read,
relay, or propagate any other heartbeat files — that is the node's responsibility.

Client heartbeats are **event-driven at status transitions**, then HI-driven until
the next transition:

- On each status transition: write a heartbeat immediately and reset the HI timer.
- Every HI interval thereafter while status is unchanged: write a heartbeat with the
  same status, updated `timestamp`, `alive_s`, and `status_s`.

Each client write goes to **both** recv-side and send-side meta slots.

### Client status

| Trigger | Status written |
|---------|----------------|
| Startup | `ready` |
| Returning to idle (was busy, no message available) | `ready` |
| New message pickup (including `busy` → `busy`) | `busy` |
| Shutdown | `closing` |

`status_s` resets to 0 on every transition, including `busy` → `busy`. During
HI-driven repeat writes, `status_s` increments normally, reflecting time spent
in the current status rather than time since last write.

The `closing` status is written once, immediately before process exit. No subsequent
HI-driven writes follow — the process exits immediately after the write.

---

## Startup purge

On startup, before beginning the heartbeat cycle, each participant MUST delete its own
heartbeat files from both meta slots:

- A node deletes all files matching `heartbeat-node-*` from both its recv-side and
  send-side meta slots.
- A client deletes all files matching `heartbeat-client-*` from both its recv-side
  and send-side meta slots.

Each participant purges only its own file type. Files written by adjacent participants
(e.g., a `heartbeat-client-*` file on a node's send-side meta) are left intact.

The purge ensures that stale heartbeat files from a previous session do not persist
across restarts. Recovery from the purge gap is bounded at 1×HI.

---

## Reading the heartbeat chain

Because nodes relay upstream content forward, a participant at any position in the
pipeline can read the full liveness state of all upstream participants from a single
meta slot file.

**Full chain healthy** — a node's send-side heartbeat file contains entries from all
upstream participants in order, oldest upstream first:

```
Client_AAA:1710000000:120:ready:45
Node_BBB:1710000001:119:
Node_CCC:1710000002:118:
```

**Broken chain** — a `HEARTBEAT_NOT_FOUND` prefix indicates the node immediately
upstream of the reading participant has not produced a heartbeat. Entries after
`HEARTBEAT_NOT_FOUND` are from the reading node itself:

```
HEARTBEAT_NOT_FOUND
Node_CCC:1710000002:118:
```

This identifies the stall location: the hop between the node writing `HEARTBEAT_NOT_FOUND`
and its upstream neighbor is where the chain breaks.

**`closing` status** — a client entry with `status: closing` indicates intentional
shutdown rather than unexpected disappearance. This is the distinguishable departure
signal that allows participants to differentiate a graceful shutdown from an offline
or crashed participant.

**Inferring abandonment vs. offline** — compare the `timestamp` in the most recent
heartbeat entry against the current time. If `now - timestamp > N × HI`, the upstream
participant has exceeded its expected emission interval and should be treated as
offline. The absence of a `closing` status in the last observed entry suggests
ungraceful departure.

---

## Security considerations

Heartbeat file content is **unencrypted**. Content consists solely of UUIDs,
timestamps, integer counters, the status strings `ready` / `busy` / `closing`, and
the sentinel `HEARTBEAT_NOT_FOUND`. No payload content and no routing secrets are
present in heartbeat files.

UUID exposure reveals only that a participant with that identifier exists and is
polling — information already implicit in the existence of the channel configuration.

Encryption of heartbeat content via a shared key distributed at channel setup time
(`DRIP_KEY`) is deferred to a future revision.

---

## Backward compatibility

A participant not implementing the heartbeat protocol (e.g., a v0.5 node in a v0.6
pipeline) will not write heartbeat files. It will appear as `HEARTBEAT_NOT_FOUND` at
its adjacent nodes — indistinguishable from an offline participant. The pipeline
continues to function; that hop is simply invisible to heartbeat diagnostics.

In a mixed pipeline, `HEARTBEAT_NOT_FOUND` cannot be used to distinguish an offline
participant from a non-participating one. Operators should note this limitation when
upgrading incrementally.

---

## Protocol registry entry

The following entry belongs in `protocol-registry.md`:

| Prefix        | Protocol | Description |
|---------------|----------|-------------|
| `heartbeat-`  | Meta slot | Per-hop liveness chain. Nodes relay upstream heartbeat content forward; clients write status signals. Operates on meta slots alongside primary payload slots. Plaintext. |
